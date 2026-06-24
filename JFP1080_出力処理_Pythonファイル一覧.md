# JFP1080 出力に関する Python ファイル一覧と処理内容

**作成日:** 2026-06-24
**対象帳票:** JFP1080 事業別予算査定一覧表（当初）
**プロジェクト:** `/mnt/c/claude/svg-editor/`

---

## 1. 出力パイプライン概要

JFP1080（事業別予算査定一覧表・当初）の出力（印刷 PDF 生成）に関わる Python ファイルを、データの流れ順に示す。

```
[業務システム] → POST /api/v1/external/JFP1080_事業別予算査定一覧表(当初)
    ① project_assessment.py            (入力検証スキーマ)
    ② print_external.py                (JFP1080専用エンドポイント)
    ③ print.py                         (汎用 create_print_job：出力統括)
    ④ data_prep.py                     (形式判定 → 変換ディスパッチ)
    ⑤ data_prep_report_transformer.py  (report_type ルーター)
    ⑥ JFP1080.py                       (JFP1080専用変換 → formData)
        ├ table_lines.py               (ReportConfig 定義・共通ユーティリティ)
        └ base.py                      (和暦/日付整形)
    ⑦ pdf_converter_client.py          (formData + SVGバインディング → pdf-converterへ)
        → PDFバイナリ返却
```

---

## 2. ファイル一覧と役割

| #  | ファイル（`server/app/` 起点） | 役割 |
|----|------|------|
| ①  | `schemas/print_external_payload/project_assessment.py` | JFP1080 受信ペイロードの **Pydantic スキーマ**。ツリー構造（事業>施策>施策内訳）と各査定額の型を定義 |
| ①' | `schemas/print_external_payload/__init__.py` | 各帳票スキーマと `JFP1080_CODE_ID` 定数を集約エクスポート |
| ②  | `api/v1/external/print_external.py` | code_id ごとの **型付き外部 API エンドポイント**。`POST /{JFP1080_CODE_ID}` を公開し汎用処理へ委譲 |
| ③  | `api/v1/external/print.py` | **汎用 `create_print_job`**。出力処理全体の統括（データ準備→バインディング解決→PDF生成→返却） |
| ④  | `services/data_prep.py` | **`apply_data_prep`**。入力形式を判定し適切な変換ルートへ振り分ける |
| ⑤  | `services/data_prep_report_transformer.py` | **`transform_report`**。`report_type` を見て帳票専用関数へディスパッチするルーター |
| ⑥  | `services/data_prep_reports/JFP1080.py` | **JFP1080 専用変換ロジック**。ツリーを再帰展開し 11 列の `formData` を生成 |
| ⑥a | `services/data_prep_reports/table_lines.py` | `ReportConfig` データクラス定義と共通ユーティリティ（数値整形・改定率など） |
| ⑥b | `services/data_prep_reports/base.py` | 和暦変換 (`to_wareki`)・帳票日付整形などの基盤ユーティリティ |
| ⑥c | `services/data_prep_reports/__init__.py` | 全帳票モジュール（JFP1080 等）の集約 import |
| ⑦  | `services/pdf_converter_client.py` | `formData` と SVG バインディングを **pdf-converter サービスへ HTTP 送信**し PDF を取得 |

> **補足（重複実装）**
> `server/data-prep/app/services/report_transformer.py` と `server/data-prep/app/main.py` にほぼ同等の変換ロジックが **別マイクロサービスとして** 存在する。
> ただし実際の印刷経路（`print.py`）は **in-process の `app.services.data_prep`** を呼ぶため、上表⑤が稼働系。`server/data-prep/` 配下は分離サービス版。

---

## 3. 各ファイルの処理：「何を」「どのように」

### ① `project_assessment.py`（入力スキーマ）

| 何を | どのように |
|------|-----------|
| 受信 JSON の構造検証 | `ProjectAssessmentInitialPrintRequest` で `report_type="project_budget_assessment"` 固定・`data`・`context`・`sync` を型定義 |
| 事業の階層ツリー | `ProjectAssessmentInitialNode`（`code/name/level/requested_amount/assess_1..4/assess_final/prev_year/children`）を **自己再帰型** で定義（`model_rebuild()` で前方参照解決） |
| データ本体 | `ProjectAssessmentInitialData.tree` を「事業(level=1)ルート群」として保持 |

### ② `print_external.py`（専用エンドポイント）

| 何を | どのように |
|------|-----------|
| JFP1080 の HTTP 受け口 | `@router.post(f"/{JFP1080_CODE_ID}")` で **URL=code_id 完全一致** のエンドポイントを定義 |
| ペイロード整形 | `_new_payload(request)` ＝ `req.model_dump(exclude={"sync"})` で `report_type+data+context` 形に変換 |
| 汎用処理へ委譲 | `_to_response(JFP1080_CODE_ID, payload, sync, db, api_key)` → `create_print_job` を呼ぶ |
| API 仕様 | `_PDF_RESPONSES`（200=PDF / 202=ジョブ受付 / 404 / 500）を Swagger に宣言 |

### ③ `print.py`（出力統括 `create_print_job`）

| 何を | どのように |
|------|-----------|
| データ準備 | `apply_data_prep(json_data, code_id)` を呼び、`formData` 化と `code_id` 解決（`"auto"` 時は解決値を採用） |
| テンプレート解決 | `service.resolve_print_binding(code_id, allow_head_fallback=True)` で `svgbinding.binding_json` を取得（無ければ 404） |
| 出力モード分岐 | `sync=true` → 同期 PDF 生成、バッチ指定時 → `_handle_batch_print`、非同期 → Celery `print_pdf_task.delay`（202＋status_url） |
| PDF 返却 | `convert_to_pdf(...)` 結果を `media_type="application/pdf"` で `Response` 返却（ファイル名は `{code_id}.pdf`） |

### ④ `data_prep.py`（`apply_data_prep`）

| 何を | どのように |
|------|-----------|
| 入力形式の判定 | `json_data["formData"]` を取り出し、3 形式を判別 |
| report_type 形式 | `is_report_format(form_data)` が真 → `transform_report(form_data)` で `(code_id, formData)` 取得し返す（JFP1080 はここ） |
| 統一フォーマット | `fixed+tables` 形式なら `transform_to_formdata()` で変換 |
| formData 直渡し | 上記いずれでもなければそのまま（数値整形のみ）返す |

### ⑤ `data_prep_report_transformer.py`（`transform_report`）

| 何を | どのように |
|------|-----------|
| 入力分解 | `report_type / context / data` に分解 |
| ルーティング | `_REGISTRY.get(report_type)` で `_Entry`（config＋resolve_code_id＋transform）取得、未登録なら `ValueError` |
| code_id 解決 | `entry.resolve_code_id(config, context)` |
| 本変換 | `entry.transform(config, context, data)`（JFP1080 は `JFP1080.transform`） |
| 整合性保証 | 起動時 `_assert_rules()` で「1 code_id=1 config」「transform 関数の共用禁止」を強制 |

### ⑥ `JFP1080.py`（JFP1080 専用変換）★出力の中核

| 何を | どのように |
|------|-----------|
| 帳票設定 | `PROJECT_BUDGET_ASSESSMENT = ReportConfig(...)`：タイトル「事業別予算査定一覧表(当初)」、`code_id_map={"INITIAL": "JFP1080_..."}`、11 列の `item_fields` |
| code_id 解決 | `resolve_code_id`：`context["plan_type"]`（既定 `INITIAL`）で code_id を引く |
| 数値整形 | `_fmt_int`：空→`""`、0→`"0"`、負値→`△{絶対値,}`（三角＋カンマ）、正値→カンマ区切り |
| 階層展開 | `_build(node, depth)` を `tree` に再帰適用。`depth` 分の全角空白でインデントしたラベル（`code + name`）を生成 |
| 査定額の連鎖補完 | `assess_1=要求額`, `assess_2=assess_1`… のように **未入力時は前段の値を引き継ぐ**（`a1=assess_1 or req`, `a2=assess_2 or a1`…） |
| 列マッピング | col1=ラベル, col2=要求額, col3〜7=1次〜最終査定, col8=前年度, **col9=最終査定−前年度（差額）**, col10/11=空 |
| 子要素 | `children` があれば `row["items"]` に再帰結果を格納（ネスト表） |
| 最終成形 | `{"formData": {title, subtitle, fiscal_year, report_date, items}}` を返す＋`logger.info("JFP1080変換完了: 事業=%d")` |

### ⑥a `table_lines.py`

| 何を | どのように |
|------|-----------|
| 帳票設定の型 | `@dataclass ReportConfig`（`title/subtitle_map/code_id_map/item_fields/default_plan_type`） |
| 汎用 code_id 解決 | `resolve_code_id(config, context)`：`plan_type` でマップ参照（JFP1080 は独自版を使用） |
| 共通計算 | `revision_rate(curr, prev)`：改定率 `(当−前)/前×100` を四捨五入整数で算出（ゼロ除算回避） |

### ⑥b `base.py`

| 何を | どのように |
|------|-----------|
| 和暦変換 | `to_wareki(fiscal_year)` で年号付き表記を生成 |
| 日付整形 | `format_report_date` / `to_wareki_date` で帳票日付を整形 |
| 時刻基準 | `now_jst` / `to_jst` で JST を提供 |

### ⑦ `pdf_converter_client.py`

| 何を | どのように |
|------|-----------|
| PDF 生成委譲 | `convert_to_pdf(binding_json, json_data)`：`binding_json`（SVGバインディング）＋`formData` を JSON 化 |
| 外部送信 | `httpx.post(...)` で pdf-converter サービスへ送り、PDF バイナリを取得（接続/ステータス例外を送出） |

---

## 4. まとめ

JFP1080 の出力は、**「型検証(①) → エンドポイント(②) → 出力統括(③) → 形式判定(④) → ルーティング(⑤) → JFP1080専用変換(⑥) → PDF変換委譲(⑦)」** という 7 段のパイプラインで構成される。

JFP1080 固有のロジックは **⑥ `JFP1080.py` に集約** され、他は全帳票共通の基盤・ルーティング層である。⑥の肝は次の 4 点。

1. **3階層ツリーの再帰展開**（事業 > 施策 > 施策内訳）
2. **査定額の前段引き継ぎ補完**（未入力時に前段の値を継承）
3. **△＋カンマの数値整形**
4. **最終査定−前年度の差額算出**（col9）

> **関連論点**
> `docs/report/` 配下に JFP1080 の仕様書ドラフト（`仕様_JFP1080_入出力処理フロー` 等）と確認依頼（`確認依頼_JFP1080_査定額0円の補完問題_5W1H.md`）が存在する。
> 「査定額0円の補完問題」は、⑥の「前段引き継ぎ補完」ロジック（`a1 = assess_1 or req` 等）と直接関係する未確定論点である。
