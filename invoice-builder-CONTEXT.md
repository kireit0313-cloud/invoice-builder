# invoice-builder プロジェクト引継ぎ

## 目的・背景

Tsuyoshiさん(非エンジニア)が、coconalaなどのフリーランスプラットフォーム経由で
中小企業向けに販売しているWebベースの請求書作成PWA「invoice-builder」の開発プロジェクト。

- 業務モデル：Tsuyoshiさんがクライアント(中小企業)ごとにExcelテンプレートをアップロード・設定し、
  専用URLを発行。クライアントはそのURLで請求書PDFを自分で生成できる。
- ゴール：毎月のExcelコピペ作業をなくすこと。

## 技術スタック

- プレーンHTML/JavaScript(ビルドツールなし)、GitHub Web UIで直接編集
- ホスティング：Vercel(GitHub main branchから自動デプロイ)
- データベース：Firebase Firestore(**invoice-builder専用プロジェクトに完全分離済み、詳細は下記**)
- 認証：Firebase Auth(Google login、Tsuyoshiさん専用の管理者アクセス)
- Excel処理：SheetJS(CDN)
- ZIP生成：JSZip(CDN)
- PDF生成：Google Cloud Run + Puppeteer

### リポジトリ・アカウント情報

- GitHubリポジトリ：`kireit0313-cloud/invoice-builder`
- 本番URL：`https://invoice-builder-iota-lac.vercel.app/`
- Cloud Runサービス：`invoice-pdf-server`(リポジトリ：`kireit0313-cloud/invoice-pdf-server`、GitHubにアップロード済み)
- Cloud Runローカルパス：`C:\Users\fokar\invoice-pdf-server\`
- Googleアカウント：kireit0313@gmail.com
- **Firebase/Firestoreプロジェクト(invoice-builder専用・新)**：`invoice-builder-4b77e`(Blazeプラン、リージョン`asia-northeast1`)
  - 旧`idea-notebook-172e0`との共有状態は解消済み(下記「Firebaseプロジェクト分離」参照)
- **Cloud Run稼働先GCPプロジェクト**：引き続き`idea-notebook-172e0`(Cloud Run自体はFirestoreと異なりプロジェクト移行スコープ外、将来のV2対応時に検討)
- 参考プロジェクト：Idea Loom(`kireit0313-cloud/idea-notebook`)— 同じスタックの参照実装、Firestoreは`idea-notebook-172e0`に専用化済み

将来的に「estimate-builder」(見積書作成ツール)を別リポジトリとして計画中
(Excelインポート/エクスポート型、Firestore顧客保存なし、過去Excel再インポート対応)。

---

## 現在の状態(本セッション終了時点)

### ① Firebaseプロジェクト完全分離【完了・最優先ブロッカー解消】

**背景：** 2026年7月6日頃、Idea Loomとinvoice-builderが同じFirebaseプロジェクト(`idea-notebook-172e0`)のFirestoreルールを1ファイルで共有していたため、ルール上書きでIdea Loom側の記述が消える事故が発生(復元済み)。再発防止のため、invoice-builder専用の新規Firebaseプロジェクトへ完全分離した。

**実施内容：**
- 新規Firebaseプロジェクト`invoice-builder-4b77e`を作成(Blazeプラン、Firestore `asia-northeast1`、Authentication Google login有効化)
- **GCPプロジェクト作成数の上限に遭遇**：個人アカウントのデフォルト上限に達していたため、(a) Googleの「Project Quota Increase」申請フォームから引き上げをリクエスト、(b) 並行して未使用の放置プロジェクト5個(`My First Project`×2、`Default Gemini Project`、`Generative Language Client`、名称なしプロジェクト)を削除。最終的にはクォータ引き上げ承認が下りてから新規プロジェクト作成に成功(削除した5個は30日間の削除保留状態のため、クォータには当面カウントされ続ける点に注意)
- `client.html`・`index.html`双方の`firebaseConfig`を新プロジェクトの値に書き換え(**2ファイルとも個別にfirebaseConfigを持つ構成のため、両方の書き換えが必要**)
- Firestoreセキュリティルールを新プロジェクトに移植。**移植時に設計バグを発見・修正**：取引先(`partners`)は`clients/{clientId}/partners/{partnerId}`のサブコレクションだが、ルールが`match /partners/{partnerId}`を`clients`ブロックの兄弟(sibling)として書いていたため、サブコレクションへの書き込みが一致せず拒否されていた。`match /partners/{partnerId}`を`match /clients/{clientId}`の内側にネストする形に修正して解消(詳細は下記「学び・注意点」参照)
- Authentication「承認済みドメイン」にVercel本番URL(`invoice-builder-iota-lac.vercel.app`)を追加
- バックアップJSON(アプリ内蔵の「データの復元」機能)からテストクライアントを復元し、動作確認
- PDF個別出力・一括出力ともに新プロジェクトで正常動作を確認
- 旧プロジェクト(`idea-notebook-172e0`)側のFirestoreルールから、invoice-builder専用部分(`clients`・`partners`)を削除し、Idea Loom専用(`users/{userId}/ideas/{ideaId}`のみ)に整理・公開済み

**現状：** invoice-builderとIdea Loomは、Firebase/Firestoreレベルで完全に独立したプロジェクトとなった。ルール上書き事故の再発リスクは解消。

### ② 依存バージョンの固定(再現性の確保)【2026/7/8完了】

「コードを変えていないのに、ある日突然動かなくなる」を防ぐための固定作業。以下まで到達済み。

- `package.json`の`puppeteer`をキャレット無しの正確指定`21.11.0`に固定してGitHubにコミット。
- `package-lock.json`(直接・間接すべての依存の正確バージョンを記録)を生成してGitHubにコミット。
  - アップロード時に、ファイル名がWindowsの短縮名(8.3形式)`PACKAG~2.JSO`に化けたが、GitHub Web UIで`package-lock.json`にリネームして修正済み(内容は正常)。
- 注意：`npm install`時にPuppeteer 21系の間接依存に既知の脆弱性が5件(high)検出される。これは古い版に固定していることが原因で、修正には本体アップグレード＋Chromium再検証が必要。急ぎではなく、将来のPuppeteerアップグレード時にまとめて対応する。

### ③ index.htmlから自由記入欄(上段・下段)を削除【2026/7/8完了・GitHub反映済み】

- 管理画面(`index.html`)の「自由項目記入欄(上段)」「自由記入欄(下段)」カードを、対応するJS(保存処理・編集復元・行追加関数)ごと削除。これらは取引先個別ページ(`client.html`)で取引先ごとに入力する運用に一本化。
- あわせて会社情報カードのタイトルを「会社情報・書類設定(PDFに記載する固定情報)」→「会社情報」に変更。
- 安全性確認済み：`client.html`は新規取引先作成時に`currentClient.projectFields || []` / `currentClient.remarksLower || ''`とガードしているため、`index.html`側に無くても空扱いで正常動作する。取引先ごとの自由記入欄(freeFields/remarksLower)は`client.html`に独立して存在し、PDFもそちら側の値から生成される。
- GitHubに反映済み(2026/7/8)。Vercelが自動デプロイ。

### ④ 取引先一覧UI改善・会社情報の項目統一【2026/7/9完了・デプロイ済み】

**client.html(クライアント画面)：**
- 取引先一覧：最上段のクライアント名を中央寄せ、戻るボタンは左端に絶対配置で固定。取引先名は左寄せのまま。
- 一括編集バーを常時表示に(未選択時はボタン無効、1件以上選択で有効化)。各行の「›」矢印を削除。削除「×」を赤い四角ボタン化。
- 取引先追加：敬称を「御中／様」に簡素化。締め日「手入力」を「取引先ページで入力」に改称(内部値は`手入力`のまま、詳細画面で日付欄に切替わる動作は維持)。
- ブラウザ標準の`alert`/`confirm`(ダイアログにドメイン名が出る)を全廃し、自前のトースト通知(`showToast`)＋確認モーダル(`showConfirm`)に置換(計13箇所)。※index.html側は本人希望で今回未対応。
- 振込先を「振込先の編集」独立カードに分離(PDFで会社情報と別位置に表示されるため)。
- **会社情報を項目リスト化**：社名以外(住所・TEL・登録番号・FAX・E-mail等)を「項目名＋内容」の並び替え可能リスト(▲▼)に統一。プリセットボタン(＋住所/TEL/FAX/E-mail/登録番号)付き。UIの並び順がそのままPDFの表示順。項目名を空欄にするとPDFで見出しなし(値のみ)表示。

**データ形式(新)：** `companyInfo.fields = [{label, value}, ...]`(並び順つき配列)。社名は`companyInfo.name`、振込先は`companyInfo.bankInfo`(fieldsとは別管理)。旧フィールド(`address`/`tel`/`invoiceNumber`/`customFields`)は読み込み時に自動でfieldsへ移行(住所は見出しなしにして従来の見た目を維持)。移行は`client.html`の`getCompanyFields()`と`index.js`の両方に実装。

**index.js(PDF)：** 会社情報ブロックを`company.fields`優先で描画(無ければ旧形式を自動変換)。label空欄は値のみ表示。→Cloud Run再デプロイ実施済み(2026/7/9)。

**index.html(管理画面)：** 会社情報入力を会社名のみに簡素化(住所・TEL・登録番号・振込先・追加項目の欄を削除)。保存を全上書きから「既存の`companyInfo`を保持し会社名のみ更新」に変更し、クライアント画面で設定したfields/振込先が管理画面の再保存で消えないよう修正。

---

## 過去の完了事項(前回セッションまで)

### client.html モバイル表示バグ修正

- 取引先追加エリア(`.add-partner-area`)に`flex-wrap: wrap`を追加し、スマホ幅でのはみ出しを解消

### Cloud Run `index.js`：remarksLower(自由記入欄・下段)のPDF表示対応

- `remarksLower`の分割代入追加、明細テーブル直後・合計欄直前に表示。未記入時は非表示。動作確認済み

### Cloud Run PDF生成の重大インシデントと対応

**症状：** OS標準Chromium(`apt-get install chromium`)のバージョンが自動更新され、Cloud Runサンドボックス環境と非互換になり「Failed to launch the browser process!」でPDF生成が完全停止。

**修正：** Puppeteer自身がダウンロードする動作確認済みChromiumを使う方式に変更。共有ライブラリ一式をDockerfileに追加。個別・一括とも復旧確認済み。

### 依存バージョンの固定化・Node.js移行

- Puppeteerを`^21.0.0`→`21.11.0`に正確固定(2026/7/8にGitHubへコミット済み)
- Node.jsベースイメージを`node:22-slim`に確定(2026/7/8)。**旧状態(`node:20-slim`)からの移行を完了。**

> **Node.js移行(2026/7/8完了)：** 以前ドキュメントは「Node.js 22へ移行」としていたが、実際のGitHub `Dockerfile`は`node:20-slim`のままだった(未コミット)。今回`FROM node:22-slim`に修正し、移行を完了させた。反映手順は下記「残作業／次アクション」を参照。

---

## 既知の残作業・次回優先事項

1. **【完了・2026/7/8】** ~~`package-lock.json`のGitHubへのコミット~~ → `package.json`のpuppeteer正確固定とあわせてコミット済み。
2. **【デプロイ待ち・優先】** Node.js 22移行と再現性の反映。
   - デプロイ元フォルダ`C:\Users\fokar\invoice-pdf-server\`を確認したところ、`Dockerfile`は既に`node:22-slim`だった(ローカルでは移行済み、GitHub未反映だっただけ)。
   - **重要な発見：** 同フォルダに`package-lock.json`が無かった。デプロイは`gcloud run deploy --source .`でこのフォルダを使うため、lockが無いと再現性が効かない。→ ミラーから`package-lock.json`をこのフォルダにコピー済み(2026/7/8)。これでデプロイ時も直接・間接依存が固定される。
   - GitHubの`Dockerfile`は`node:22-slim`にUpdate済み(2026/7/8)。`package-lock.json`もGitHubにあり。
   - **残手順（未実施）：** (a) `C:\Users\fokar\invoice-pdf-server\`で標準デプロイコマンド`gcloud run deploy`を実行、(b) デプロイ後に**個別・一括の両方でPDF生成を確認**(ポリシー遵守)。
3. **【完了・2026/7/8】** `index.html`(自由記入欄削除＋会社情報リネーム)をGitHubに反映済み。Vercelが自動デプロイ。
4. **【任意・低優先】** 削除済みGCPプロジェクト5個の30日後の完全削除確認(削除保留期間が明けるまで待機、特に作業不要)。
5. **【将来】** estimate-builderの新規リポジトリ作成(Excelインポート/エクスポート型)。
6. **【保留】** GitHub Actions自動デプロイ(Cloud Run向け) — 現状は手動`gcloud run deploy`。
7. **【保留・優先度低】** SendGridメール送信機能。
8. **【定期見直し】** バージョン管理ポリシーに沿った半年〜1年ごとの依存バージョン点検。

---

## バージョン管理ポリシー

| 項目 | 方針 |
|---|---|
| **Node.jsベースイメージ** | メジャーバージョンは明示的に固定(現在：`node:22-slim`、2026/7/8確定)。EOL(2027年4月30日)の半年前を目安に次のLTSへの移行を検討。 |
| **Puppeteerのバージョン** | `package.json`はキャレット(`^`)を使わず正確なバージョンで固定(現在：`21.11.0`)。`package-lock.json`も必ずGitHubにコミットする(2026/7/8コミット済み)。 |
| **Chromium** | Puppeteer自身がダウンロードする、そのバージョンに対応した組み合わせのみを使用。OS(apt)標準のChromiumパッケージは使わない |
| **依存バージョンの見直しタイミング** | 半年〜1年に1回、または「コードを変えていないのに急に動かなくなった」場合に真っ先に疑う項目として |
| **依存関係・ベースイメージ変更時の確認手順** | 変更後は必ず個別出力・一括出力の両方でPDF生成を確認してからGitHubにコミットする |
| **デプロイコマンド(標準形)** | 下記「ツール・リソース一覧」参照。`--execution-environment gen2`を必ず含める |

---

## 学び・注意点(蓄積)

- **Firestoreセキュリティルールのサブコレクション・ネスト必須**：`match /clients/{clientId}`と`match /partners/{partnerId}`を兄弟(sibling)ブロックとして書くと、`/partners/{partnerId}`はルート直下のコレクションにしか一致せず、実際のデータ構造である`clients/{clientId}/partners/{partnerId}`(サブコレクション)への書き込みはどのルールにも一致せずデフォルト拒否される。サブコレクションへのルールは、必ず親の`match`ブロックの内側にネストして書くこと。
- **GCPプロジェクト作成数には個人アカウントでも上限がある**：新規プロジェクト作成時に「アカウントで作成できるプロジェクト数の上限に達しました」というエラーが出た場合、(1) Googleの割り当て引き上げ申請フォーム(project quota increase)から申請、(2) 未使用の放置プロジェクト(Gemini API利用時などに自動生成される`Default Gemini Project`や`My First Project`)を削除、の2方向で対処できる。ただし削除したプロジェクトは30日間「削除保留」状態を経るため、即座にクォータの空きにはならない点に注意。
- **複数ファイルがそれぞれ独自の`firebaseConfig`を持つ構成に注意**：invoice-builderでは`client.html`と`index.html`が独立して`firebaseConfig`を保持しているため、Firebaseプロジェクトを切り替える際は両方を書き換える必要がある。片方だけ切り替えると、想定と異なるプロジェクトへの書き込み・認証エラーが発生し原因特定に時間がかかる。
- **列マッピングは位置ベース**：単価・金額はどちらも「金額入力」型として自動判定されるため、種類ではなく列の並び順(インデックス)でマッピングする。
- **Cloud Run `index.js`整合性ルール**：`client.html`からCloud Runへ渡すデータを変更したら、必ず`index.js`側の受け取りロジックも同時に確認すること。(なお`index.js`の`projectFields`・`remarksLower`は`client.html`の取引先ごとの値を受け取る。`index.html`側の削除とは無関係。)
- **`encodeURIComponent()`**：`Content-Disposition`ヘッダーの日本語ファイル名には必須。
- **`const`再宣言バグ**：分割代入した変数名をそのまま`const`で再宣言すると`SyntaxError`。分割代入側の変数名をリネームして回避。
- **ブラウザキャッシュ**：コードが反映されないように見える場合、まずハードリフレッシュ(`Ctrl+Shift+R`)を試す。認証プロジェクト切り替え直後のログインエラーにも有効。
- **`onclick`属性への直接埋め込みリスク**：取引先名などをそのまま`onclick`属性に埋め込むと特殊文字でSyntaxErrorになる。`data-*`属性＋`el.dataset`で読み取る方式にする。
- **一括操作は明示的に引数を渡す**：`currentPartner`などのグローバル状態に依存する関数は一括処理ループ内で静かに失敗する。
- **Firestore設計**：最新1件を上書きするのみで履歴は残らない。再保存時は`createdAt`を保持すること。
- **Firestoreの`deleteDoc`はサブコレクションを自動削除しない**：親ドキュメントを消してもサブコレクション(`partners`)は残るため、明示的に全件削除してから親を削除する必要がある。
- **バックアップ／復元とFirestore Timestamp**：`JSON.stringify`でTimestampは`{seconds, nanoseconds}`相当の形に変換される。復元時は`new Timestamp(seconds, nanoseconds)`で明示的に戻す必要がある。
- **CSS flexboxの折り返し**：横並びレイアウト(`display:flex`)は`flex-wrap: wrap`を指定しない限りモバイル幅で要素がはみ出す。
- **依存パッケージ・OSパッケージのバージョン固定は最優先事項**：`apt-get install`や`npm install`の未固定バージョン指定は、コードを一切変更していなくても「ある日突然動かなくなる」重大インシデントの原因になりうる。
- **「Failed to launch the browser process!」＋`inotify`/`dbus`/`NETLINK`エラーの読み方**：これらは起動失敗時の副次的な警告であることが多く、真因はPuppeteerとChromiumのバージョン不整合や共有ライブラリ不足であることが多い。
- **GitHubアップロード時のファイル名化けに注意**：Windowsからファイルをアップロードすると、名前が短縮名(8.3形式・例`PACKAG~2.JSO`)に化けることがある。アップロード後はファイル名が意図通りかを必ず確認し、化けていればGitHub上でリネームする。
- **`raw.githubusercontent.com`のCDNキャッシュ遅延**：コミット直後は反映が遅れることがある。確認にはクエリ付き(`?cb=...`)でキャッシュ回避するか、少し待って再取得する。
- **`index.html`の保存は`setDoc`でドキュメント全体を上書き(`merge`なし)**：`companyInfo`の一部だけを扱う画面では、保存前に既存ドキュメントを`getDoc`で読み、既存`companyInfo`を`{...existingCompany, name}`のように保持して部分更新すること。全上書きすると`client.html`で設定した`fields`や`bankInfo`が消える。
- **会社情報のPDF描画は`companyInfo.fields`(並び順つき配列)が正**：`index.html`と`client.html`と`index.js`の3つで形式を合わせること。旧`address`/`tel`/`invoiceNumber`/`customFields`は`fields`へ自動移行する経過措置がclient.htmlとindex.js両方に入っている。
- **Cloud Runデプロイは`--source .`のローカルフォルダを使う(GitHubではない)**：`gcloud run deploy invoice-pdf-server --source .`は`C:\Users\fokar\invoice-pdf-server\`のファイルをビルドする。GitHubの内容とこのフォルダは自動同期されないため、両方を最新に保つこと。特に`package-lock.json`はこのフォルダに存在しないと再現性が効かない(GitHubに置くだけでは不十分)。

---

## 開発の進め方(合意済みルール)

### 作業フォルダ(ローカルミラー)方式 ★2026/7/8導入

- デスクトップの`見積請求PWA開発`フォルダをClaudeに接続済み。この中に各リポジトリを複製(ミラー)して保持する。**このフォルダを「正(ソース・オブ・トゥルース)」とする。**
  - `見積請求PWA開発/invoice-builder/` … `index.html`・`client.html`・`invoice-builder-CONTEXT.md`・`developer-profile-updated.md`
  - `見積請求PWA開発/invoice-pdf-server/` … `index.js`・`Dockerfile`・`package.json`・`package-lock.json`
- **コード修正方式(トークン節約＆自動化)**：Claudeはフォルダ内のファイルを直接編集する。チャットに全文や大きな差分を貼り出さない。修正後は機械的な検証(識別子の有無・タグ収支・`node --check`等)を行う。
  - ※ 旧ルール「読み込み済みは全文渡し／未読は差分渡し」は、フォルダ接続前の運用。今後は上記「フォルダ直接編集」を基本とする。
- **GitHubへの反映**：Claudeが編集した「変更ファイルだけ」を、TsuyoshiがGitHub Web UIの「Upload files」でドラッグ＆ドロップしてコミットする(コピペ不要)。Vercelはmainへのpushで自動デプロイ。
- **同期のルール**：原則GitHub側を直接編集しない(ローカルミラーと食い違うため)。もし食い違ったら、その時だけGitHubから最新を取得してローカルを上書きする。
- **GitHub読み込み**：まずローカルミラーを`grep`等で参照する。GitHubから取得する必要がある場合は`web_fetch`で`raw.githubusercontent.com`を読む(bashの`curl`/`python`等でのURL取得は不可)。大きいファイルは一旦ファイルに保存してから検索する。
- **GitHub自動読み書きについて**：接続可能なGitHub連携(コネクタ)は現状の一覧に無く、gitでの自動commitも認証トークンが必要で(Claudeはトークンを入力できない)、現時点では「全自動コミット」はできない。上記のローカルミラー＋Web UIアップロードが現実的な最適解。

### その他の運用ルール

- **ファイル編集(Tsuyoshi側)**：ローカルビルド環境なし。GitHub Web UIのUpload/Editで反映。
- **コンテキスト継続と読み込み優先順位**：`invoice-builder-CONTEXT.md`と`developer-profile-updated.md`は、**接続フォルダ内(`見積請求PWA開発/invoice-builder/`)を「正」とする**。編集後は必ずGitHubにもUploadして同じ状態に保つ(GitHubは"いつでも読める予備＋オフサイトのバックアップ")。
  - **読み込み順**：セッション開始時は、フォルダが接続されていればフォルダ内のこの2ファイルを読む。フォルダが未接続のときだけGitHubから読む。
  - GitHubの2ファイルは削除しない(フォルダ未接続時・別PC・ローカル消失時の唯一の読み取り元＆バックアップになるため)。食い違いは「フォルダで編集→必ずGitHubへUpload」の徹底で防ぐ。
  - 会話が長くなったら新セッションを開始しCONTEXT.mdを読み込み直すと、トークンの積み上がりがリセットされる。
- **ターミナルコマンド**：一つずつ実行(まとめて貼らない)。
- **トラブル時の切り分け方針**：コード変更後にエラーが出た場合、まず「コード変更前の状態に戻しても同じエラーが再現するか」を確認し、コード起因かインフラ・環境起因かを切り分ける。
- **セッション開始の合言葉**：「デスクトップの見積請求PWA開発フォルダを接続して、invoice-builder-CONTEXT.mdとdeveloper-profile-updated.mdを読み込んで」
  - 補足：フォルダ接続はセッションごとに毎回必要（前回接続しても新セッションには引き継がれない。永続化する設定は現状なし）。合言葉に「フォルダを接続して」を含めると、接続要求→承認→読み込みが一続きで進む。

---

## ツール・リソース一覧

- ホスティング：Vercel(GitHub main branchから自動デプロイ)
- データベース：Firebase Firestore(`invoice-builder-4b77e`、Blazeプラン、`asia-northeast1`、invoice-builder専用)
- 認証：Firebase Auth(Google login、Tsuyoshiさん専用、`invoice-builder-4b77e`)
- PDF生成：Google Cloud Run + Puppeteer(`invoice-pdf-server`、Node.js 22 / Puppeteer 21.11.0固定・Puppeteer自前Chromium方式、GCPプロジェクトは引き続き`idea-notebook-172e0`)
- **デプロイコマンド(標準形)**：
  ```
  gcloud run deploy invoice-pdf-server --source . --region asia-northeast1 --allow-unauthenticated --memory 1Gi --execution-environment gen2
  ```
  実行場所：`C:\Users\fokar\invoice-pdf-server\`
