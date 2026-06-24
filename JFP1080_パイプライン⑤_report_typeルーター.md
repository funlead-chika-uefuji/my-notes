# JFP1080 出力パイプライン⑤ — report_type ルーター（transform_report）

**作成日:** 2026-06-24
**対象帳票:** JFP1080 事業別予算査定一覧表（当初）
**プロジェクト:** `/mnt/c/claude/svg-editor/`

---

## 1. Python ファイル名と処理概要

**ファイル名:** `server/app/services/data_prep_report_transformer.py`（関数 `transform_report`）

**処理概要:**
`report_type` を見て適切な変換パターン＋設定にディスパッチする **ルーター**。自身は変換ロジックを持たず、`_REGISTRY`（report_type → 設定＋専用関数の対応表）を引いて帳票固有関数へ委譲する。起動時アサーションで「1 code_id = 1 config」「transform 関数の共用禁止」を強制し、帳票間のロジック混線を防ぐ。JFP1080 は `report_type="project_budget_assessment"` で `JFP1080.transform` に委譲される。

---

## 2. ファイル内の処理項目一覧

| 処理プロジェクト名（定義名） | 処理概要 | 処理内容 |
|------|---------|---------|
| モジュール docstring / 絶対ルール | 設計制約の宣言 | ①1 code_id=1 config ②transform 関数の共用禁止 ③transform 内のマッピング分岐禁止 を明記 |
| import 群 | 帳票モジュール取り込み | `data_prep_reports` から全帳票モジュール（`JFP1080` 含む）と `ReportConfig` を import |
| `_Entry`（dataclass, frozen） | レジストリ要素の型 | `config: ReportConfig` ＋ `resolve_code_id` ＋ `transform` の3点組を保持 |
| `_REGISTRY` | report_type 対応表 | report_type をキーに `_Entry` を登録。JFP1080 は `"project_budget_assessment": _Entry(config=JFP1080.PROJECT_BUDGET_ASSESSMENT, resolve_code_id=JFP1080.resolve_code_id, transform=JFP1080.transform)` |
| `_assert_rules()` ＋ 即時実行 | 起動時整合性検証 | 全 _Entry を走査し「code_id_map が1件」「code_id 重複なし」「config インスタンス共用なし」「transform 関数共用なし」を検証。違反は `ValueError` |
| `_PASSTHROUGH_REGISTRY` | パススルー登録（現在空） | 変換を業務システム側で行う帳票用の空 dict |
| `get_registered_handlers()` | 登録一覧の取得 | `{report_type: [code_id, ...]}` を返す（_REGISTRY 側は必ず1要素） |
| `is_report_format(data)` | 形式判定 | `"report_type"` `"context"` `"data"` の3キーが揃えば DB-close 形式と判定（④が使用） |
| `transform_report(data)` ★中核 | ルーティング本体 | 下表の手順で `(code_id, formData)` を返す |

### `transform_report(data)` の内部処理

| 処理プロジェクト名 | 処理概要 | 処理内容 |
|------|---------|---------|
| 入力分解 | 引数の展開 | `report_type` / `context` / `data`（report_data）に分解 |
| レジストリ参照 | ハンドラ取得 | `entry = _REGISTRY.get(report_type)`、`None` なら `ValueError("未対応の report_type")` |
| code_id 解決 | 出力先 ID 決定 | `code_id = entry.resolve_code_id(entry.config, context)` |
| 本変換 | formData 生成 | `form_data = entry.transform(entry.config, context, report_data)`（JFP1080 は `JFP1080.transform`） |
| ログ出力 | トレース記録 | `logger.info` で report_type → code_id・formData キー一覧を記録 |
| 返却 | 結果タプル | `return code_id, form_data` |

---

## 3. 補足

- JFP1080 は `_REGISTRY` の `"project_budget_assessment"` エントリ経由で、設定 `PROJECT_BUDGET_ASSESSMENT` と専用関数 `JFP1080.transform` に確実に結び付く。
- 「絶対ルール」により JFP1080 用 config / transform は他帳票と共有されないことが起動時に保証される。
