---
name: gys-screenshot-to-ppt-wireframe
description: 公営企業会計システム刷新プロジェクト (ぎょうせい/GYS × ファンリード/FL) の既存画面スクショ(画像)を、開発が同一ファイル粒度で受け取りNext.js実装に進められるPPTワイヤー(.pptx)に変換する4フェーズ標準ワークフロー。画像→①項目抽出→②DS突合→③PPTワイヤー生成(python-pptx)→④実装引き継ぎ。「画面スクショをワイヤーに」「画像からPPTワイヤー」「{画面}のワイヤーPPT」「nextjs向けワイヤー」「旧画面をワイヤー化」「STD/BFP/BFD系をワイヤーに」「帳票条件設定画面のワイヤー」等のキーワードで必ず発動。新規1枚から量産まで対応。
metadata:
  type: workflow
  project: GYS (公営企業会計システム新版)
  version: 1.0
  baseline-sample: STD3290 対前年度比較表（帳票出力 条件選択画面）
  derived-from: gys-old-to-new-screen (Figma経路) を 画像→PPT に転換
---

# GYS 画面スクショ → PPTワイヤー 標準ワークフロー

公営企業会計システム (ぎょうせい/GYS × ファンリード/FL) で、**既存画面のスクショ画像を、開発(安部・石倉・大塚さん)がそのままNext.js実装に使える「PPTワイヤー(.pptx) + 構造化md」に変換する4フェーズワークフロー** を実行する。

旧スキル `gys-old-to-new-screen` の **HTMLプロト→Figma還元** を **PPTワイヤー(python-pptx)** に置換したもの。Figma/HTMLは生成しない。

## 中核思想

- **入力 = 既存画面スクショ**（旧BizBrowser画面 `gys_bizbrouser_画面集/` や現行サイトのキャプチャ）。
- **成果物 = PPTワイヤー + md**。HTML/Figmaは作らない。PPTが正式ハンドオフ。
- **正本 = python script**（`build_wireframe.py` + `screen.json`）。.pptxは生成物。テキスト差分でレビュー可（旧スキルのバイナリHTML問題を解消）。
- **ワイヤー = グレーボックス + 注釈レイヤー**（各ボックスに `⟦shared-component名⟧ token:DSトークン` を併記）。配色は実装で適用、レビューは構造に集中。

## 必須リファレンス (起動時に読む)

| 種別 | パス |
|---|---|
| 描画エンジン | `~/.claude/skills/gys-screenshot-to-ppt-wireframe/gys_wireframe_lib.py` |
| ベースラインサンプル | `~/Desktop/GYS/wireframe-条件選択画面/`（screen.json + build_wireframe.py + .pptx） |
| 旧画面スクショ集 | `~/Desktop/GYS/gys_bizbrouser_画面集/`（STD/BFP/BFD/BRD系 1300枚超） |
| 既存build流儀 | `~/Desktop/GYS/build_*_ppt.py`（rgb/box/tbox idiom） |
| shared-components | `~/Desktop-backup/projects/gys-yosanhensei/gyousei-shared-components/src/components/` |
| デザインガイドライン (Box) | ID `2176447218566`（入江由佳 / v1.2 / 2026-03-15） |
| Yosanhensei-Frontend | `~/Desktop-backup/projects/gys-yosanhensei/Yosanhensei-Frontend/` |

## 4フェーズ概要

```
[画面スクショ画像] → ①項目抽出 → ②DS突合 → ③PPTワイヤー生成 → ④実装引き継ぎ
```

### Phase 1: 画像→項目抽出
- 入力: スクショ画像（複数可。`~/Desktop/GYS/gys_bizbrouser_画面集/{画面ID}.png` 等）。任意でBox帳票仕様(`372679775768`配下)を相互参照。
- 手段: 画像を Read(vision) でUI要素を列挙 ＋ Pillow で実寸/概略座標を測り、各要素の **bbox（比 %）** を控える。
- 出力: `~/Desktop/GYS/YYYYMMDD_{画面ID}_{画面名}_旧システム抽出.md`（旧スキルと同名）＋ 要素表（項目名・bbox・型・グループ）。

### Phase 2: デザインガイドライン照合（旧スキルから流用）
- 入力: Phase 1抽出物 ＋ ガイドライン(Box `2176447218566`) ＋ shared-components ls ＋ `tailwind.config.js`。
- 手段: 各要素 → shared-component（AmountField/DualInputField/SectionCard等）＋ DSトークンへマッピング。
- 出力: `..._旧→新DS突合表.md`（列に「→ ワイヤーボックス名 / コンポ / トークン / kind」を追加）。
- 必ず立てる: **オープン質問 L1–L5**（後述）。スクショでL1レイアウトは一部自明化されるが、L3/L5は業務確認が残る。

### Phase 3: PPTワイヤー生成（新核）
`wireframe-{画面名}/` を作り:
- **`screen.json`** … 画面データ（項目 + bbox + コンポ + トークン + kind）。スキーマは `gys_wireframe_lib.py` 冒頭docstring参照。
- **`build_wireframe.py`** … 正本（テキスト）。`sys.path` にスキルdirを足し `from gys_wireframe_lib import build_from_json` を呼ぶ。
- **`{画面名}_wireframe_v0.N.pptx`** … 生成物。

```python
import sys; sys.path.insert(0, "~/.claude/skills/gys-screenshot-to-ppt-wireframe")
from gys_wireframe_lib import build_from_json
build_from_json("screen.json", "条件選択画面_wireframe_v0.1.pptx", legend=True)
```

- キャンバス 1366×768 を 16:9 スライドにマップ。**4点セット枠**（Header 72h / Sidebar 280w / Footer / Main 1086w：Breadcrumb→Title+IDバッジ→Tab→Content）はエンジンが自動描画。
- `screen.json` の box 座標は **コンテンツ領域（タブ下）に対する % (0-100)**。`kind`: field / button / table / section / radio / checkbox / select / stepper / text。
- 各ボックスに `component` と `token` を入れると注釈レイヤー `⟦…⟧ token:…` が自動付与。
- 版管理: スクリプト＆json が正本。再ビルドで `v0.N+1`。1画面=1スライド、状態/タブ違いは同デッキ内スライド追加。
- 目視確認: LibreOffice で PDF化→`pdftoppm -png` してレンダリングを Read（後述）。

### Phase 4: 実装引き継ぎ
- 出力: `..._実装引き継ぎ_{開発担当}.md`
  - shared-component import 一覧
  - 新規コンポ候補（SectionCard / StepperWithUnit / DualInputField / 2カラム比較 等）
  - .pptx パス
  - 未解決オープン質問（L3/L5系）
  - Ignite UI マッピング・Next.js実装メモ
- 安部・石倉・大塚さんへ Markdown＋PPTで引き渡し。

### 🔴 必須レイアウト4点セット (島村さんFB 2026-05-19反映 / エンジンが自動描画)

実装側コードで必ず import される:
```jsx
import Header from '../../components/Header';
import Sidebar from '../../components/Sidebar';
import Footer from '../../components/Footer';
import Breadcrumb from '../../components/Breadcrumb';
```
`gys_wireframe_lib.add_screen_slide()` がこの4点セット枠を自動で描く。content の box はその内側(Main領域)に配置される。

## 共通リファレンス値 (絶対遵守 / 旧スキルから全流用)

### カラートークン (gyousei-shared-components/tailwind.config.js) — `gys_wireframe_lib.TOKENS`

| トークン | 値 | 用途 |
|---|---|---|
| primary | `#2295F9` | メインアクセント |
| primary-light | `#DBEBF5` | ヘッダー背景・アクションボタン |
| primary-dark | `#1B6DB4` | ボーダー・フォーカス状態・注釈色 |
| border | `#cad2d9` | shared-components 共通ボーダー |
| bg-input | `#f5f7fa` | 入力エリア (§4.1.1) |
| bg-unit | `#EEEEEE` | InputWithUnit 単位部分 |
| bg-readonly | `#F8F7F7` | DualInputField readonly |
| notice-bg | `#D3EEFF` | テーブル content-bg / 通知バー |
| menu-bg | `#F1F9FF` | メニュー背景 |

ワイヤー本体は greyscale。色は注釈テキスト `token:bg-input` で示す（実装で適用）。

### フォント (§3.1.3)
- **Noto Sans JP**（28px=タイトル / 16-20px=小見出し / 13-14px=本文 / 10px=メニュー補助）

### 画面基準
- 解像度 1366×768 (§2.2) / ボタン高 大46・中36・小26 (§3.2.17) / 金額=桁カンマ+「円」+右揃え (§3.2.8) / 入力エリア背景グレー `#f5f7fa` (§4.1.1)

### 命名規則 (§3.1.5 用語集)
| 旧 | 新 | 理由 |
|---|---|---|
| 終了ボタン | **キャンセルボタン** | 用語集に「終了」未定義、「キャンセル」が該当 |
| 閉じるボタン | **×ボタン** | ボタン名では使わない |
| 「印刷」 | そのまま (PDF表示の意) | ✅ |
| 「出力」 | そのまま (ローカルDL の意) | ✅ |

### shared-components 主要コンポ (30+件) — 直接 import 可
```
header, footer, breadcrumb, heading, button (3階層),
input, select, checkbox, radio, alert,
input-with-unit, dual-input-field,
amount-field, budget-field, date-field, date-range-field,
form, form-field, form-label-with-required,
card, badge, dialog, modal, dropdown-menu,
collapsible-section, debtor-search-dialog,
container, empty-state, error-boundary
```
未収載（新規追加候補・注釈に `(新規)` を付ける）: SectionCard / StepperWithUnit / DualInputField(code+name) / A列B列 2カラム比較。

### Ignite UI for React (Yosanhensei-Frontend のみ採用)
`@infragistics/igniteui-react-{core,grids,inputs,layouts}` v19.5.2

| ワイヤー kind | shared-component | Ignite UI |
|---|---|---|
| field | input | `IgrInput` |
| select | select | `IgrSelect` / `IgrCombo` |
| stepper | input-with-unit | `IgrNumberInput` |
| radio | radio | `IgrRadioGroup` + `IgrRadio` |
| button | button(main) | `IgrButton variant="contained" color="primary"` |

**まず shared-components の既存コンポを使う**。Ignite UI は補完用。

## Box ファイルID
| 用途 | Box ID |
|---|---|
| デザインガイドライン | `2176447218566` |
| 帳票仕様確認設計書フォルダ | `372679775768` |
| サンプル: 予算査定一覧科目別 | `2183630001029` |

## オープン質問テンプレート (L1-L5)
Phase 2-3 で必ず立てる:
- **L1**: A列/B列のような並列レイアウト = どのパターン？（2カラム並列/タブ/アコーディオン/テーブル風）
- **L2**: スピナー（旧Windows風 ▲▼）を新DSでどう対応？（StepperWithUnit / IgrNumberInput / Select代替）
- **L3**: ラジオ+数値入力の組み合わせの意味（業務側確認）
- **L4**: コード+名称ペアの命名/採用（DualInputField）
- **L5**: 動的依存ロジック（grayout等）の再現方針

## 量産時の注意点
1. 同時並行 **3画面まで**（context窓）。Figma MCPは使わないのでレート制約は無い。
2. DS突合表は **共有Drive一元管理**。
3. L3/L5系 オープン質問は **画面横断で集約** してから業務側へ。
4. `gys_wireframe_lib.py` は **不変**。画面固有は `screen.json` に閉じる（単一責任）。
5. 命名変更（終了→キャンセル等）は **画面横断で同じ判断**。
6. 複数画面を1デッキにまとめたい場合は `screen.json` の `tabs` を画面分並べる。

## PPT生成のコツ / トラブルシューティング

| 症状 | 対処 |
|---|---|
| `ModuleNotFoundError: pptx` | `python-pptx` 要インストール（環境は pyenv 3.11.8 に導入済 v1.0.2） |
| 目視したい | `soffice --headless --convert-to pdf` → `pdftoppm -png -r 90 -f 2 -l 2 deck.pdf out` → 生成PNGを Read |
| ボックスが枠外/重なる | `screen.json` の座標は **コンテンツ領域%**。y は タブ下=0、フッター手前=100。フィールドは y≈19 から始める |
| テーブル列がずれる | box に `"cols":[...]`, `"rows":N` を指定 |
| 注釈が出ない | box に `component` / `token` を入れる |
| 日本語が豆腐 | フォントは Noto Sans JP（`gys_wireframe_lib.JP`）。環境にNoto未導入ならLibreOfficeが代替フォントで描画 |

## 起動時の確認事項
「{画面名} をワイヤーに」「STD/BFP系をPPTワイヤーに」「帳票条件設定画面のワイヤー」等の依頼を受けたら先に確認:
1. **対象画面のスクショ画像パス**（`gys_bizbrouser_画面集/{ID}.png` か、別途提供か）
2. **どこから着手か**（Phase 1全部 or 既存抽出md流用でPhase 3から）
3. **業務確認の窓口**（島村さん固定、特定なら指定）
4. **緊急度・期限**
5. **既存の DS突合表/抽出mdがあるか**（`~/Desktop/GYS/` 配下確認）

## ファイル命名規約
```
~/Desktop/GYS/
├── YYYYMMDD_{画面ID}_{画面名}_旧システム抽出.md      ← Phase1
├── YYYYMMDD_{画面ID}_旧→新DS突合表.md              ← Phase2
├── wireframe-{画面名}/
│   ├── screen.json            ← 画面データ
│   ├── build_wireframe.py     ← 正本(テキスト)
│   └── {画面名}_wireframe_v0.N.pptx  ← 生成物
└── YYYYMMDD_{画面名}_実装引き継ぎ_{開発担当}.md       ← Phase4
```
（`gys_wireframe_lib.py` はスキルdir常駐。各wireframeフォルダからは sys.path 経由で import）

## 担当者一覧 (2026-05-19時点)
| 役割 | 氏名 | 領域 |
|---|---|---|
| プロジェクトリード | 池田（ファンリード） | 全体 |
| 業務 | 島村さん | 帳票仕様・業務観点 |
| 業務 (帳票専門) | 枇杷木さん | 帳票出力フォーマット |
| 設計リード | 安部さん・石倉さん (ファンリード) | 実装難度・DS方針 |
| 実装 | 大塚さん (ファンリード, 石倉配下) | shared-components実装 |
| デザインガイドライン作者 | 入江由佳さん (ファンリード) | UIデザイン v1.2 |
| 帳票仕様書作者 | 杉山拓也さん (ファンリード) | 帳表仕様確認設計書 |

## 関連スキル
| スキル | 用途 |
|---|---|
| `gys-old-to-new-screen` | Figma還元が要る案件（本スキルの前身・共存） |
| `frontend-design` | （任意）HTMLで詳細詰めたい時のみ |

## 改訂履歴
| 版 | 日付 | 改訂者 | 内容 |
|---|---|---|---|
| 1.0 | 2026-06-23 | 池田 + Claude (Opus 4.8 1M) | 初版。gys-old-to-new-screen の Figma経路を 画像→PPTワイヤー(python-pptx) に転換。STD3290 条件選択画面でベースライン実証 |
