# 本をさがす — ISBN Book Finder

ISBNの手打ち入力またはスマホカメラのバーコードスキャンから書籍情報を検索するWebアプリ。  
GitHub Pages のみで動作する静的構成（サーバー不要・認証不要）。

公開URL: https://honsagashi.azimicat.dev/

---

## 概要

| 項目 | 内容 |
|---|---|
| データソース | [OpenBD API](https://openbd.jp/)（無料・認証不要） |
| バーコードスキャン | BarcodeDetector API（Android Chrome）／ ZXing-js（その他） |
| 永続化 | ブラウザの `localStorage` |
| 構成 | `index.html` 1ファイルのみ |
| ホスティング | GitHub Pages（HTTPS必須） |

---

## 機能仕様

### 検索タブ
- ISBN（10桁または13桁）を手打ち入力して検索
  - ハイフン・スペースは自動除去
  - Enter キーで検索実行
- カメラボタンでバーコードスキャン起動
  - EAN-13形式（ISBN-13）をリアルタイム認識
  - **枠内の領域のみ**をデコード対象とする（映像全体は検出しない）
  - 背面カメラを自動選択
  - ISBN以外のバーコードを検出した場合はステータスバーに通知
  - スキャン成功後、自動的に検索を実行

### 検索結果表示（OpenBD `summary` + `onix` + `hanmoto`）
| 表示項目 | OpenBDフィールド |
|---|---|
| 書影 | `onix.CollateralDetail.SupportingResource[ResourceContentType=01].ResourceVersion[0].ResourceLink` → `summary.cover`（フォールバック） |
| 書名・巻号 | `summary.title` / `summary.volume` |
| シリーズ名 | `summary.series` |
| 著者 | `summary.author`（カンマを除去して表示） |
| 出版社 | `summary.publisher` |
| 出版年月 | `summary.pubdate` |
| 内容紹介 | `onix.CollateralDetail.TextContent` → `hanmoto.kaisetsu105w` → `hanmoto.hanmotokarahitokoto`（優先順） |
| ISBN | `summary.isbn` |

- 書名は [版元ドットコム](https://www.hanmoto.com/) の書籍ページ（`https://www.hanmoto.com/bd/isbn/{ISBN}`）へのリンク
- 内容紹介が120字を超える場合は「続きを読む」で全文展開
- 書影が存在しない場合はプレースホルダーを表示

### 履歴タブ
- 検索成功のたびに自動記録（最大50件、`localStorage`）
- 同じISBNを再検索した場合は最新に更新（重複なし）
- タップで当該書籍の結果を検索タブに再表示
- 個別削除・一括削除に対応
- 記録からの経過時間を表示（「たった今」「N分前」「N時間前」「N日前」）

### お気に入りタブ
- 検索結果からワンタップで追加・解除
- `localStorage` に永続保存（件数上限なし）
- タップで検索タブに再表示
- 個別削除・一括削除に対応

---

## 技術構成

```
index.html
├── HTML構造
├── CSS（CSS変数によるテーマ管理、モバイルファースト）
└── JavaScript
    ├── OpenBD API呼び出し（fetch）
    ├── バーコードスキャン
    │   ├── BarcodeDetector API（Android Chrome・優先）
    │   └── ZXing-js（CDN・フォールバック）
    └── localStorage による履歴・お気に入り管理
```

### バーコードスキャンの動作
- **Android Chrome**: ブラウザ組み込みの `BarcodeDetector` API（Google MLKit）を使用。検出精度が高い。
- **その他（iOS Safari・PC等）**: ZXing-js 0.19.1 を使用。
- どちらの場合も、映像全体ではなく**画面上の枠（220×80px）に対応する映像ピクセル領域のみ**を切り出してデコードする。`object-fit: cover` によるスケール・オフセットを正確に計算して切り出し範囲を決定する。

### 使用ライブラリ（CDN）
| ライブラリ | 用途 | URL |
|---|---|---|
| ZXing-js 0.19.1 | バーコードスキャン（フォールバック） | `https://unpkg.com/@zxing/library@0.19.1/umd/index.min.js` |
| Noto Serif JP | 日本語フォント | Google Fonts |
| Shippori Mincho | 見出しフォント | Google Fonts |

### OpenBD API
```
GET https://api.openbd.jp/v1/get?isbn={ISBN}
```
- CORS対応・認証不要・無料
- レスポンスは配列。該当なしの場合 `[null]` が返る
- スキーマ: https://api.openbd.jp/v1/schema

---

## デプロイ手順（GitHub Pages）

### リポジトリ構成
```
your-repo/
└── index.html
```

### 設定手順
1. GitHubにリポジトリを作成
2. `index.html` をコミット・プッシュ
3. リポジトリの **Settings → Pages** を開く
4. Source を `Deploy from a branch` に設定
5. Branch: `main`、Folder: `/(root)` を選択して **Save**
6. 数分後に `https://{ユーザー名}.github.io/{リポジトリ名}/` で公開

---

## 注意事項

### HTTPS必須
カメラ（`getUserMedia` API）はセキュアコンテキスト（HTTPS）でのみ動作する。  
GitHub Pages はデフォルトでHTTPS提供のため問題なし。

ローカル開発時は以下のいずれかを使用:
- `python3 -m http.server 8080`（手打ち検索のみ動作、カメラ不可）
- VS Code の **Live Server**（設定で HTTPS 有効化）
- `npx serve` 等の HTTPS 対応ローカルサーバー

> `http://localhost` ではカメラが起動しない（手打ち検索は動作する）

### localStorageについて
- ブラウザのプライベートモードでは動作するが、ウィンドウを閉じると消去される
- ユーザーがブラウザのデータを削除すると履歴・お気に入りも消える
- 容量はブラウザにより異なるが通常5MB程度（書籍データのみなら数千件分に相当）

### OpenBDの収録範囲
- 日本国内の書籍が対象。雑誌・一部の専門書は未登録の場合がある
- 書影は出版社が登録した場合のみ存在する

---

## 今後の拡張候補

- 複数ISBNの一括検索（`/v1/get?isbn=ISBN1,ISBN2,...`）
- お気に入りリストのCSV/JSONエクスポート
- 読了・未読などのステータス管理
- 楽天ブックス・カーリルなど他APIとの併用
