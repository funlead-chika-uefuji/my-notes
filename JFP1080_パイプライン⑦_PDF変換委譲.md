# JFP1080 出力パイプライン⑦ — PDF 変換委譲（pdf_converter_client）

**作成日:** 2026-06-24
**対象帳票:** JFP1080 事業別予算査定一覧表（当初）
**プロジェクト:** `/mnt/c/claude/svg-editor/`

---

## 1. Python ファイル名と処理概要

**ファイル名:** `server/app/services/pdf_converter_client.py`

**処理概要:**
PDF 生成を担う **pdf-converter マイクロサービスの HTTP クライアント**。⑥で生成した `formData` と、外部 SVG バインディング（`binding_json`）を結合して pdf-converter に POST 送信し、PDF バイナリを取得する。PDF 描画ロジックを別サービスに一元化するための薄いクライアント層。描画エンジンや接続先は環境変数で切り替え可能。

---

## 2. ファイル内の処理項目一覧

| 処理プロジェクト名（定義名） | 処理概要 | 処理内容 |
|------|---------|---------|
| モジュール docstring | 役割宣言 | 「PdfGeneratorService の代わりに pdf-converter を HTTP で呼ぶ」と明記 |
| `PDF_CONVERTER_URL` | 接続先設定 | 環境変数。既定 `http://print_subsystem_pdf_converter:8000` |
| `PDF_CONVERTER_TIMEOUT` | タイムアウト設定 | 環境変数。既定 `120`（秒） |
| `PDF_RENDERER` | 描画エンジン選択 | 環境変数。既定 `reportlab`（新・軽量）／`cairosvg`（旧・安定）に即切替可能 |
| `convert_to_pdf(binding_json, json_data, output_filename=None)` ★主経路 | PDF 生成委譲 | `{"binding_json", "json_data", "renderer"}`（＋任意 `output_filename`）の payload を組み、`httpx.post(f"{URL}/convert", json=payload, timeout=...)` で送信。`raise_for_status()` 後に `resp.content`（PDF バイナリ）を返す。完了を `logger.info` に記録 |
| `save_pdf(pdf_bytes, pdf_storage_path, filename)` | PDF 保存 | 保存先ディレクトリを作成し、PDF バイナリをファイルに書き出してパスを返す（非同期/バッチ経路で使用） |

---

## 3. 補足

- JFP1080 の同期出力では、③ `create_print_job` が `convert_to_pdf(binding_json=svgbinding.binding_json, json_data=<⑥のformData>, output_filename=...)` を呼び、戻りの PDF バイナリをそのまま HTTP レスポンスとして返す。
- 接続不可・エラー時は `httpx.ConnectError` / `httpx.HTTPStatusError` を送出し、③側で `500 PDF_GENERATION_FAILED` に変換される。
- **タイトル/明細ブロックの高さなどレイアウトは `binding_json` 側**にあり、本クライアントは中身を加工せずそのまま転送する。
