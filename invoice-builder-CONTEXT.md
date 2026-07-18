# invoice-builder プロジェクト引継ぎ（スリム版・2026/7/18再編）

> **開発の正はこのファイル。** セッション別の詳細履歴（①〜㊽）は`invoice-builder-CONTEXT-アーカイブ.md`（実装経緯が必要なときだけgrepで部分参照）。
> 販売・広告の正は`販売広告/00-販売CONTEXT.md`、ココナラの進捗は`販売広告/ココナラ/00-進捗メモ.md`、リリース進捗は`release-roadmap.md`。
> 編集後は必ずGitHubにもUploadして同期する（アーカイブ含む）。

## 目的・背景

Tsuyoshiさん（非エンジニア）が中小企業向けに販売するWebベースの請求書作成PWA。クライアント（=購入企業）ごとに専用URLを発行し、顧客はURLを開いて明細入力→インボイス対応の請求書PDFを自分で生成。毎月のExcelコピペ作業をなくすのがゴール。ココナラで販売中（2026/7/18公開）。

## 技術スタック・インフラ

- **フロント**：素のHTML/JS（ビルドなし）。`index.html`=管理画面（Tsuyoshi専用）、`client.html`=顧客ページ。ホスティング=Vercel（mainへpushで自動デプロイ）
- **DB/認証**：Firebase Firestore + Auth（Google login）。プロジェクト=`invoice-builder-4b77e`（専用・Blaze・`asia-northeast1`）
- **PDF生成**：Cloud Run + Puppeteer（`invoice-pdf-server`、Node22 / Puppeteer 21.11.0固定・自前Chromium。稼働GCPプロジェクトは`idea-notebook-172e0`のまま）
- **ライブラリ**：JSZip（ZIP出力）。SheetJSは全廃済み
- **リポジトリ**：`kireit0313-cloud/invoice-builder`・`kireit0313-cloud/invoice-pdf-server`
- **本番URL**：https://invoice-builder-iota-lac.vercel.app/ （管理ログインは`/?login`で表示）
- **Googleアカウント**：kireit0313@gmail.com（管理者。ルールとUIの両方で本人メールにピン留め）
- **デプロイ**：両リポジトリともGitHubへUpload/コミットだけで自動反映（invoice-pdf-serverはGitHub Actions→`gcloud run deploy`。Secret `GCP_SA_KEY`、SA `github-deploy@...`。コマンド標準形はアーカイブ⑦参照）。Actions確認: https://github.com/kireit0313-cloud/invoice-pdf-server/actions
- 参考実装：Idea Loom（`kireit0313-cloud/idea-notebook`、Firestoreは`idea-notebook-172e0`に分離済み）

## 仕様サマリー（現行）

### データ構造（Firestore）

- `clients/{clientId}`：clientIdは`crypto.randomUUID()`（秘密URL方式）。主フィールド：
  - `clientName`（顧客ページ上部の「〇〇様」表示）／`companyInfo.name`（PDF発行元社名）
  - `companyInfo.fields = [{label, value}, ...]`（並び順つき会社情報。label空欄=値のみ表示。ラベル`〒`はPDFでコロンなし）
  - `companyInfo.bankInfo`（振込先）／`companyInfo.seal = {enabled, mode:'auto'|'image', imageData}`（角印）
  - `companyInfo.theme = {mode:'gray'|'color', color:'#hex'}`（PDF配色）／`companyInfo.invoiceTitle = '請求書'|'ご請求書'`
  - `columns = [{name, enabled}×6]`（明細項目名・位置固定：0日付 1内容 2数量 3単価 4金額 5備考。税率は固定で対象外。enabledは数量/単価/備考のみ有効。旧データは全ONとして互換）
  - `maxPartners`（任意int64。取引先上限の個別上書き。既定50はclient.htmlのコード側）
- `clients/{clientId}/partners/{partnerId}`：`partnerName, honorific, issueDay, issueDate, invoiceNo, lineItems, freeFields(上段), remarksLower(下段)`。**最新1回分のみ保存**（履歴なし・前月値を引き継いで上書き）
- バックアップJSON：取引先単位`{_type:'invoice-builder-partner-backup', _version:1, ...}`。一括=同スキーマをZIPに複数格納（個別復元と互換）。管理画面の全体バックアップ=全クライアント+partners込み。復元時はTimestampを`new Timestamp(seconds, nanoseconds)`で戻す

### 共通項目の設計

- 会社情報・振込先・印鑑・配色・PDFタイトル・明細項目名は**すべて顧客ページ（client.html）で編集**。管理画面は「クライアント名・会社名・デフォルト税率・URL発行」のみ
- 共通項目はクライアント文書に1組だけ=取引先ごとに記憶されない（個別出力時に変更→保存→出力の運用は可能。一括出力は全取引先に同一適用）
- 保存はすべて`{companyInfo(またはcolumns), updatedAt}`のmerge書き込み（未認証update許可範囲内）

### PDF（index.js）

- デザイン=案B（左アクセントバー方式・囲み排除）。CSS変数`--c-brand`／`--c-band-bg`／`--c-subbar`で配色。カラー時は明るい色を相対輝度で自動暗補正。**タイトル黒固定・角印朱色固定**
- 明細表：`table-layout:fixed`・基準列幅15/31/6/14/14/6/14%。`colFlags`でOFF列を除外し残り幅を100%へ再按分。`border-collapse:separate`（2ページ目の下線二重化対策）
- 複数ページ：余白は`page.pdf`の`margin`側（bodyのpaddingでは2ページ目以降が端切れ）。`thead`反復・`tr{page-break-inside:avoid}`・合計/振込先ブロックも分断回避
- 角印：枠68px固定・Yuji Syuku（Google Fonts、`document.fonts.ready`待ち+3秒打ち切り）・右詰め列配分・長音符90°回転。**client.htmlのプレビュー（`sealAutoPreviewHtml`）と同一ロジック必須**
- 振込先は`docType !== '見積書'`で表示。文字サイズは2026/7/15に全体拡大済み（詳細アーカイブ㊺。さらに上げる場合は列幅再確認）
- 長すぎる区切りなし文字列はChromiumが全体縮小する仕様（折り返しCSSは本人希望で未対応・項目分割で回避）

### セキュリティ・認証

- 管理者判定：Firestoreルール`isAdmin()`（`email == 'kireit0313@gmail.com' && email_verified`）+ index.htmlのUIガード（本人以外は即signOut）。**メール変更時は両方更新**
- 未認証（顧客）update：`hasOnly(['companyInfo','projectFields','updatedAt','columns'])`かつ社名・clientName・clientId改変禁止。`partners`は既知clientId配下で全開放（秘密URL前提・許容済みリスク）
- ルール本文のバックアップ=`firestore.rules`（**自動デプロイなし**。適用はFirebaseコンソールに貼り付け→公開）
- Safariの管理ログインは非対応で確定（PC/Chrome/Edge運用。顧客ページは認証なしのため全ブラウザOK。経緯はアーカイブ㉚）
- XSS対策：表示は`escHtml`+`data-*`方式（onclick直埋め禁止）。全入力欄に`maxlength`。PDF系エラーは日本語トースト・一括出力は部分失敗を通知

### 上限・制限

- 取引先上限：既定50件（コード側）。クライアント個別は`maxPartners`をコンソールで設定
- ZIP内同名ファイルは連番リネームで一意化（JSZipは黙って上書きするため）

## 現在の状態（2026/7/18）

- **機能実装・フェーズ1（E2Eテスト）・フェーズ2（セキュリティ）・フェーズ3（運用・法務）すべて完了**。利用規約・プライバシーポリシー掲示済み、バックアップ体制・ロールバック手順文書化済み（`バックアップ体制.md`・`ロールバック手順.md`）
- **ココナラで販売公開中**（2026/7/18〜）：https://coconala.com/services/4311730
- **残り＝フェーズ4（初回納品前）**：スモークテスト＋納品フロー実測。**初回注文が入ったら納品前に必ず実施**（`販売広告/商品資産/デモデータ/デモサイト構築ガイド.md`のSTEP5が兼用）。進捗の正は`release-roadmap.md`

### 低優先の残作業

- Cloud Runデプロイ高速化（現状`--source`フルビルドで約13分→イメージビルド+キャッシュ方式で2〜4分。方針合意済み・アーカイブ7-2）
- 一括「復元」（ZIP→全取引先一括）／SendGridメール送信／削除済みGCPプロジェクト5個の完全削除確認
- estimate-builder（見積書ツール・別リポジトリ）構想。新規立ち上げ手順テンプレはアーカイブ末尾参照
- 依存バージョン点検（半年〜1年ごと）

## 開発の進め方（合意済みルール）

- **ローカルミラー方式**：接続フォルダ`見積請求PWA開発/開発/`配下の`invoice-builder/`・`invoice-pdf-server/`が「正」。Claudeが直接編集→変更ファイルだけをTsuyoshiさんがGitHub Web UIでUpload/コミット→自動デプロイ。GitHub側は直接編集しない
- チャットに全文・大差分を貼らない。編集後は機械的検証（`node --check`・タグ収支・grep）
- 1ステップずつ進め、各ステップをスクリーンショット等で実機確認。専門用語はかみ砕く
- GitHubからの取得は`web_fetch`で`raw.githubusercontent.com`（CDNキャッシュ注意：`?cb=`でキャッシュ回避）。bashのcurl等は不可
- **セッション開始の合言葉（用途別・2026/7/18更新）**：
  - 開発作業：「デスクトップの見積請求PWA開発フォルダを接続して、開発/invoice-builder/invoice-builder-CONTEXT.mdを読み込んで」
  - 販売・広告：「〜接続して、販売広告/00-販売CONTEXT.mdとココナラ/00-進捗メモ.mdを読み込んで」（開発CONTEXT不要）
  - 納品対応：販売側2ファイル＋`release-roadmap.md`
  - フォルダ接続はセッションごとに毎回必要
- 変更ファイルと反映先の対応：`index.html`・`client.html`・`terms.html`等→invoice-builderリポジトリ（Vercel自動）／`index.js`・`Dockerfile`等→invoice-pdf-serverリポジトリ（Actions自動デプロイ・**反映後は個別+一括PDFの実機確認必須**）
- トラブル時は「変更前に戻して再現するか」でコード起因かインフラ起因かを切り分け

## バージョン管理ポリシー

| 項目 | 方針 |
|---|---|
| Nodeベースイメージ | `node:22-slim`固定。EOL（2027/4/30）半年前に次LTS検討 |
| npm依存 | `^`なし正確固定（puppeteer 21.11.0／express 4.22.2）+`package-lock.json`コミット+Dockerfileは`npm ci --omit=dev` |
| Chromium | Puppeteer自前DL版のみ（apt版禁止） |
| 見直し | 半年〜1年ごと、または「変えてないのに動かない」時に最優先で疑う |
| 依存変更時 | 個別+一括PDFの実機確認をしてからコミット |

## 学び・注意点（蓄積）

- **Firestoreルール**：サブコレクションは親`match`の内側にネスト必須（兄弟に書くと拒否）。`deleteDoc`はサブコレクションを消さない。管理者判定は`request.auth != null`でなくメール/UIDに絞る。ルール変更は「未認証側の許可を変えないか」を先に確認すると公開先行できる。`Missing or insufficient permissions`は拒否が効いている正常サイン
- **複数ファイルが個別に`firebaseConfig`を持つ**：プロジェクト切替時は全HTMLを書き換える
- **`setDoc`全上書き注意**：一部の項目だけ扱う画面では既存doc取得→該当キーのみ更新（他画面の設定が消える）
- **Cloud Run整合**：client.html→index.jsへ渡すデータを変えたら受け取り側も同時確認。`Content-Disposition`の日本語ファイル名は`encodeURIComponent`必須
- **UI**：`onclick`直埋め禁止（`data-*`+dataset）。一括処理はグローバル状態でなく引数渡し。`flex-wrap`忘れでスマホはみ出し。ブラウザ標準`alert/confirm`は使わない（自前トースト/モーダル）。確認ダイアログにインフラ用語を出さない
- **日付**：表示・ファイル名用はローカル時刻で生成（`toISOString()`はUTCで午前中に前日になる）。記録用タイムスタンプはUTC ISOでよい
- **キャッシュ**：反映されない→まずハードリフレッシュ（Ctrl+Shift+R）。rawのCDNキャッシュは`?cb=`で回避
- **【環境依存・重要】bashマウントの末尾切れ**：接続フォルダをbash側から読むと末尾が切れて見えることがある（ホストのRead/Editは正常）。検証はホスト側ツールを正とし、大きいファイルの`node --check`は変更部だけスクラッチパッドに切り出して行う。bashの`cp`でミラー内ファイルを複製した場合は必ずmd5等で完全性確認
- **成果物は1つに1役割**：PDF=請求書、JSON=復元。復元機能を体裁ファイルに兼ねさせない
- **既定値はDBでなくコードに持たせる**（個別上書き値だけDBへ）
- **Firebase Auth**：`authDomain`をアプリと別ドメインのままSafariで使うと`missing initial state`。ポップアップ方式のまま`authDomain`だけ変えると全ブラウザで壊れる（詳細アーカイブ㉚）
- **Puppeteer PDF**：ページ余白は`page.pdf`のmarginで。`scale:1`でも折り返せない幅超えがあると全体縮小される。Webフォントは`fonts.ready`待ち+タイムアウト打ち切り
- **GitHubアップロード**：Windowsからのアップでファイル名が8.3形式に化けることがある→アップ後に名前確認
- 縦書きCSS・角印配置の詳細な学びはアーカイブ「学び・注意点」参照（writing-modeのheight挙動・正方形詰め余白など）

## フォルダ構成（2026/7/18再編）

```
見積請求PWA開発/
├── 開発/
│   ├── invoice-builder/      ← ミラー(GitHub同期)。CONTEXT・アーカイブ・roadmap・規約草案・firestore.rules・テストデータ等も同居
│   ├── invoice-pdf-server/   ← ミラー(GitHub同期)
│   └── 資料/                 ← 旧サマリーPDF等の保管（同期不要）
└── 販売広告/
    ├── 00-販売CONTEXT.md     ← 販売の正（チャネル横断）
    ├── 商品資産/             ← マニュアルv7・体験ガイド・デモデータ・デモ画像・印影ツール
    └── ココナラ/             ← 00-進捗メモ・01〜06・素材（出品画像）
```
