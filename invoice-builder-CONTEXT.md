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
- Excel処理：SheetJS(CDN) — **管理画面`index.html`のテンプレート取り込みのみで使用**。`client.html`の出力Excelは廃止したためSheetJS読み込みも`client.html`からは削除済み(2026/7/10②)。
- ZIP生成：JSZip(CDN) — `client.html`のPDF一括出力ZIP・一括バックアップZIPで使用
- PDF生成：Google Cloud Run + Puppeteer

### リポジトリ・アカウント情報

- GitHubリポジトリ：`kireit0313-cloud/invoice-builder`
- 本番URL：`https://invoice-builder-iota-lac.vercel.app/`
- Cloud Runサービス：`invoice-pdf-server`(リポジトリ：`kireit0313-cloud/invoice-pdf-server`、GitHubにアップロード済み)
- **デプロイ方式：GitHub Actions自動デプロイ(2026/7/10導入)**。`main`へのpushで`.github/workflows/deploy.yml`が`gcloud run deploy`を自動実行する。ターミナル操作・手動`gcloud`は不要。
  - 旧デプロイ元フォルダ`C:\Users\fokar\invoice-pdf-server\`は役割終了。**2026/7/10に削除済み**(古いコードで誤デプロイする事故を防ぐため)。以後この端末ローカルフォルダは使わない。
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

**データ形式(2026/7/10②追加)：**
- `companyInfo.seal = { enabled, mode:'auto'|'image', imageData }`（クライアント単位・角印）
- `companyInfo.theme = { mode:'gray'|'color', color:'#hex' }`（クライアント単位・PDF配色）
- `partners/{id}.invoiceNo`（取引先単位・請求書番号。任意、空欄非表示）
- バックアップJSON：`{ _type:'invoice-builder-partner-backup', _version:1, exportedAt, clientId, partnerId, partner:{ partnerName, honorific, issueDay, issueDate, invoiceNo, lineItems, freeFields, remarksLower } }`。個別=単体JSON、一括=同スキーマの単体JSONをZIPに複数格納（互換）。

**index.js(PDF)：** 会社情報ブロックを`company.fields`優先で描画(無ければ旧形式を自動変換)。label空欄は値のみ表示。→Cloud Run再デプロイ実施済み(2026/7/9)。

**index.html(管理画面)：** 会社情報入力を会社名のみに簡素化(住所・TEL・登録番号・振込先・追加項目の欄を削除)。保存を全上書きから「既存の`companyInfo`を保持し会社名のみ更新」に変更し、クライアント画面で設定したfields/振込先が管理画面の再保存で消えないよう修正。

### ⑤ 一括編集モードのUI改修【2026/7/9完了】

`client.html`の一括編集モード。**GitHub反映・要デプロイなし(client.htmlのみ)。**

- カードの区切りを明確化：枠線を濃く(`#CBD5E0`)＋影を追加し、取引先ごとのカードがはっきり分かれるように。
- カードヘッダーを青寄りの配色(`#6699CC`)＋取引先名・折りたたみ表示を白抜きに。
- 取引先名の横に敬称(御中／様)を表示。`p.honorific`(未設定時は御中)。
- 取引先名の横の「◯行」カウント表示を削除。
- スマホ(≤480px)で一括操作バー(進捗テキスト＋全件保存＋一括出力)が崩れる問題を、縦積み＋ボタン全幅にして解消。
- **一括編集の仕様確認**：一括モードは明細のみ編集可。自由記入欄(上段`freeFields`・下段`remarksLower`)は画面に出さないが、保存・出力時は既存値を保持(消えない)。請求書では自由記入欄が毎月変わることは想定薄いため、現状維持で確定。

### ⑥ 印鑑(角印)機能を新規実装【2026/7/9完了】

請求書PDFの会社情報の下に印鑑を表示する機能。**クライアント単位**の設定。`client.html`＋`index.js`(両フォルダ)を変更。

- **データ形式**：`companyInfo.seal = { enabled:bool, mode:'auto'|'image', imageData:dataURL }`。クライアントごとにFirestoreへ保存。画像は300pxにリサイズしてbase64で保持(追加インフラ不要)。
- **client.html**：会社情報エリアに「印鑑(角印)」カードを追加。オンオフ・自動生成/画像アップロード切替・ライブプレビュー・「画像を削除」。関数：`buildSealEditHTML`/`saveSealInfo`/`handleSealUpload`(canvasでリサイズ→base64)/`sealAutoPreviewHtml`ほか。
- **index.js(PDF)**：従来の空円プレースホルダー(`.stamp-box`「印」)を**廃止**。`seal.enabled`のとき、`mode:'image'`なら`<img>`、それ以外は社名から自動生成角印を描画。sealが未設定のクライアントは印鑑なしで出力(＝旧・空円は表示されなくなる)。**印鑑は各クライアントで設定を保存して初めて有効になる。**
- **自動生成角印の作り**：Claude API等は不使用、純CSS。朱色(`#C0392B`)の四角枠に社名を`writing-mode: vertical-rl`＋`text-orientation: upright`で縦組み・右→左に配置。`text-align: start`＋`height:100%`で各列を上端そろえ(上揃え)。
- **サイズ(自動拡大方式で確定)**：文字数に応じて枠を拡大。列数`grid=ceil(√文字数)`、枠`box=grid*18+12`px(**上限80px**、下限44px)、フォント`font=min(15, floor((box-12)/grid)-3)`px。**マス目(18px)とフォント(マス目より小さめ)を分離**しているのは、完全平方数の文字数(例「株式会社GMO商会」=9文字=3×3)で枠と文字が重なるのを防ぐため。
- **法的確認**：請求書に押印は法的義務がなく、角印(認印相当)は物理でも電子生成でも請求書の有効性は変わらない。∴アプリ生成の角印でOK。※実印＋印鑑証明が要る契約書は別。
- **未対応(仕様確定)**：下段の余白は「正方形＋上揃え＋任意長の社名」では原理的に避けられない。中央寄せは縦書きの`height`の性質上ほぼ効かない(数px)ため見送り、上端固定で確定。将来どうしても気になれば「枠を長方形にして文字量に合わせる」案があるが角印らしさは減る。

### ⑦ Cloud RunのGitHub Actions自動デプロイを導入【2026/7/10完了】

「デプロイのたびにターミナルを起動して`gcloud run deploy`を手打ちする」作業を廃止し、**GitHubへコミットするだけで本番反映**される状態にした。

- **仕組み**：`invoice-pdf-server`リポジトリに`.github/workflows/deploy.yml`を追加。`main`へのpushをトリガーに、GitHub Actionsが①リポジトリをチェックアウト→②サービスアカウントキーでGoogle Cloud認証→③`gcloud run deploy invoice-pdf-server --source . --project idea-notebook-172e0 --region asia-northeast1 --allow-unauthenticated --memory 1Gi --execution-environment gen2`を実行。コマンド内容は従来の手動デプロイと完全に同一。
- **認証**：GCPにデプロイ専用サービスアカウント`github-deploy@idea-notebook-172e0.iam.gserviceaccount.com`を作成(ロール：編集者＋サービスアカウントユーザー)。そのJSONキーをGitHubリポジトリの**Secret `GCP_SA_KEY`**に登録。キーレス方式(Workload Identity連携)ではなくキー方式を採用した理由は、非エンジニアがGCP/GitHubの画面クリックだけで完結でき、ターミナル操作が要らないため。
- **デプロイ元が変わった点に注意**：デプロイのソースは**GitHubリポジトリ**になった(旧：ローカル`C:\Users\fokar\invoice-pdf-server\`)。これに伴い、後述の「`index.js`の2フォルダ同時書き込み」ルールと「セッション開始時に2フォルダ接続」は**廃止**(下記「開発の進め方」を更新済み)。
- **確認済み**：初回デプロイがActionsで成功(緑)、本番で個別PDF・一括ZIP出力ともに正常動作を確認(ポリシー遵守)。
- **セキュリティ注意**：`GCP_SA_KEY`は本番デプロイ権限を持つ資格情報。GitHub Secretは表示不可＝正常。ローカルにダウンロードしたJSONキーは登録後に削除する。万一漏洩したらGCPで該当キーを無効化し再発行→Secret更新。

---

## 2026/7/10 セッション②：PDFデザイン刷新・請求書番号・Excel廃止とバックアップ移行

> このセッションで変更したファイルは **`invoice-pdf-server/index.js`** と **`invoice-builder/client.html`** の2つ（`index.html`は未変更）。すべてGitHubへアップロード済み前提。

### ⑧ 請求書PDFのレイアウト変更【index.js】

- **振込先を最下部へ移動**：並びを「合計→自由記入欄(下段)→振込先」に変更（従来は合計→振込先→自由記入欄）。
- **振込先ボックスの角丸を廃止**（`border-radius:0`）。
- **取引先名（`.client-name`）を下線内で中央寄せ**（`text-align:center`）。縦位置は右側の会社住所あたりに合わせて少し下げた（`.client-block`を`justify-content:flex-start; padding-top:30px`）。左右位置は従来のまま。

### ⑨ 色テーマ（グレー／カラー）機能【index.js＋client.html】

- **データ形式**：`companyInfo.theme = { mode:'gray'|'color', color:'#hex' }`（クライアント単位）。
- **index.js**：`:root`にCSS変数を出力し、グレー時は従来配色、カラー時のみ会社カラーを適用。適用先＝取引先名の下線・税込合計欄・明細見出し行（塗り＋白文字）・件名/備考/振込先ボックス・合計行・お振込先見出し。**タイトル「請求書」は黒固定**（会社カラーにしない）。**角印は朱色のまま**。明るすぎる色は白文字が読めるよう相対輝度で自動的に暗く補正（`lum>0.32`の間、黒を混ぜる）。うすい背景色`tintBg`（白90%混合）・うすい枠`tintBorder`（白65%混合）。
- **自由項目上段/下段は白抜き**（メリハリ）：カラー時＝白地＋会社カラー枠＋色ラベル（`--c-free-*`変数）、グレー時＝従来のうすいグレー。合計欄・振込先の塗りボックスとハッキリ差がつく。
- **client.html**：カード「PDFの配色（カラー）」を追加。グレー/「カラー選択」ラジオ＋プリセット5色（ネイビー/グリーン/レンガ/パープル/チャコール）＋カラーピッカー＋カラーコード直入力＋プレビュー。`normalizeHex`で3桁hex展開・不正値拒否。

### ⑩ 明細テーブルの固定幅化【index.js】

- **`table-layout: fixed`** に変更。列幅は指定どおり固定され、長い内容でも列が伸びず枠内で折り返す（内容駆動autoで1件の長い日付が列を広げる/改行する問題を解消）。
- 列幅：日付15% / 内容31%(広め) / 数量6% / 単価14% / 金額14% / 税率6% / 備考14%。
- **見出しは中央寄せ**（`th text-align:center`）、値セル（`td.num`）は右寄せのまま。見出しフォント11px＋`word-break:keep-all`＋わずかな字間詰めで「単価（税抜）」等が1行に収まる。`td`に`overflow-wrap:anywhere`。
- 検証：Noto Sans CJK JPでピクセル実測。日付「12/25～12/28」≈81px≤内寸93px、金額「¥9,000,000」≈65px≤86px、見出し「単価（税抜）」≈65px≤88px＝すべて1行。

### ⑪ 請求書番号（invoiceNo）【index.js＋client.html】

- **取引先単位の任意手入力欄**（会社固定でも自由項目でもない、専用欄）。空欄なら非表示。**請求日の上**に「請求書番号：〇〇」と表示（入力画面・PDFとも文字色は黒`#1A202C`）。
- データ：`partners/{id}.invoiceNo`。明細と同様に前月値を引き継ぐ運用（毎月そこだけ書き換え）。
- 反映先：個別（`generatePDF`に`invoiceNo`引数追加、保存/出力に反映）、**一括編集モードでも編集可**（各カードに`beinv_${p.id}`入力欄。全件保存・一括出力に反映）。
- UI上、発行日カードを「発行日・請求書番号」に改称。

### ⑫ 出力Excelを廃止し、JSONバックアップ/復元へ移行【client.html】

- **設計判断**：PDF＝請求書（見せる/渡す）、**JSONバックアップ＝過去データの保管・復元**、と役割を分離。1枚のExcelに体裁と復元を兼ねさせない。開発段階で本番ユーザーがいないため移行コストほぼゼロ。
- **廃止**：個別「Excel出力」ボタン＋`exportExcel`関数、一括ZIP内のxlsx生成、`client.html`のSheetJS読み込み（`XLSX`）を削除（**JSZipはZIP用に残置**）。一括ボタン文言は「一括出力（PDF）ZIP」。**管理画面`index.html`のExcel取り込み（テンプレート設定）は別物なので残す**（今回未変更）。
- **取引先単位バックアップ**：`exportBackup`＝画面表示中の内容を`_type:'invoice-builder-partner-backup'`のJSONで書き出し（ファイル名`バックアップ_取引先_日付.json`）。`triggerRestore`/`handleRestoreFile`＝JSONを選択→確認→**フォームに復元（保存するまで確定しない）**。復元は取引先の識別情報（名前・敬称・締め日）は保持し、明細・自由項目・備考・請求書番号・発行日だけ差し替え。
- **名前不一致の警告**：バックアップの取引先名と開いている取引先名が違うときだけ、赤系（danger）で強めの確認を表示（一致時は通常確認）。誤ファイルの取り違え防止。**確認文言に「Firestore」は出さず**「『内容保存』ボタンを押すまで確定しません」と表現。
- **一括バックアップ**：一括ページに「一括バックアップ（ZIP）」ボタン（`bulkBackupExport`）。取引先ごとのJSON（個別復元と同一スキーマ＝そのまま個別ページで復元可）をZIPにまとめて書き出し。※一括「復元」は未実装（必要なら将来追加）。

### ⑬ ボタンの配色統一・影・スマホ対応【client.html】

- **配色統一**（個別・一括共通）：保存＝アクセント青`.btn-save`、PDF＝赤`.btn-pdf`、バックアップ/復元＝アウトライン`.btn-outline`。**全ボタンに影**（案B）。並びは「バックアップ→(復元)→保存→PDF」。
- **スマホ崩れ修正**：個別アクションは`@media(max-width:480px)`で2列グリッド（`.actions .btn { flex:1 1 calc(50% - 4px) }`）。一括バーは縦積み全幅（`.bulk-actions-bar .btn { width:100% }`）。`.actions`基本に`flex-wrap:wrap`。

### ⑭ 「共通項目」バッジ【client.html】

- 会社情報・振込先・印鑑・PDFの配色の4カード見出しに「共通項目」バッジ（`.common-badge`、薄青ピル）を追加。取引先ごとの項目と、全取引先共通の設定を視覚的に区別。

### 仕様の確定メモ（このセッションで決めたこと）

- **PDF出力前の確認画面は入れない**：入力画面が実質プレビューを兼ね、再出力も軽い（上書きされるだけ）ため。将来必要なら「軽い確認モーダル（描画コストなし）」を追加する余地あり。全画面プレビューはテンプレート二重管理になるため非推奨。
- **Excel取り込みの拡張余地**：管理画面の取り込みはSheetJSで、`accept`と文言を広げれば`.ods`/`.xlsm`/`.xlsb`/`.xls`/`.csv`等に対応可（読み込み処理は変更不要）。Googleスプレッドシートは「ダウンロード→xlsx/csv→アップロード」が実務解、直API連携は非推奨。CSVのみShift-JIS対策が必要。今回は未実装（将来検討）。

---

## 2026/7/11 セッション③：PDFデザイン案B刷新・振込先拡大・複数ページ対応・取引先上限

> このセッションで変更したファイルは **`invoice-pdf-server/index.js`** と **`invoice-builder/client.html`** の2つ（`index.html`は未変更）。index.js＝デザイン刷新・振込先拡大・複数ページ対応（Cloud Run再デプロイ要）、client.html＝取引先の登録上限（Vercel自動反映）。旧`idea-notebook-172e0`のFirestore置き去り`clients`データも削除して整理済み。

### ⑮ 請求書PDFのデザイン刷新（囲みを減らす）【index.js・反映済み】

- **背景**：全体的に囲み（ボックス）が多く、特に上段・下段の自由記入欄の枠がデザイン性を落としていた。印刷を考慮し「シンプルは保ちつつデザイン性を上げる」方針。4案（罫線ミニマル／左アクセントライン／レターヘッド／見出しキャプション型）を比較HTMLで提示し、**案B（左アクセントライン）を採用**。
- **採用デザイン（案B）**：各ブロックを「囲み」から「左の縦アクセントバー」に置換。
  - 取引先名：下線 → 左アクセントバー（`border-left:4px var(--c-brand)`）＋左寄せ。
  - 税込合計：囲み → 左バー＋薄い地色の帯（`--c-band-bg`）。ラベル・金額は`--c-brand`色。
  - **自由記入欄（上段`project-info`・下段`remarks-lower`）：囲みを全廃**し、薄い左バー（`--c-subbar`）のみに。
  - 明細テーブル：外枠・縦罫線を廃止し、ヘッダー下線（`2px var(--c-brand)`）＋各行の下線（`1px #E2E8F0`）だけの抜けた表に。列幅（15/31/6/14/14/6/14）は既存の実測済み値を維持。
  - 合計（grand）行：上線を`var(--c-brand)`、文字は黒。
  - 振込先（`bank-info`）：囲み → 薄い左バー＋小見出し（11px・`--c-brand`）。
- **カラーテーマ変数を案B用に再設計**（旧`--c-underline`/`--c-hl-*`/`--c-th-bg`/`--c-box-*`/`--c-free-*`/`--c-total-line`等は廃止）：
  - `--c-brand`（主アクセント：縦バー・合計帯・見出し・明細ヘッダー下線・合計上線）
  - `--c-band-bg`（税込合計帯の背景＝薄い地色）／`--c-subbar`（補助セクションの左バー＝薄色）
  - グレー時：`--c-brand:#2C3E50` / `--c-band-bg:#F4F6F8` / `--c-subbar:#CBD5E0`。カラー時：`brand`＝会社カラー（明るい色は相対輝度で自動暗補正）、`band-bg`＝白90%混合、`subbar`＝白65%混合。**「請求書」タイトルは黒固定、角印は朱色固定**（従来どおりテーマ非依存）。
  - `companyInfo.theme = { mode, color }` のデータ形式は不変。`client.html`の配色ピッカーはそのまま使える（client.html・index.html変更なし）。
- **参考ファイル（接続フォルダ`invoice-builder/`に保存）**：`pdf-design-proposals.html`（4案比較）、`pdf-design-B-color-preview.html`（案Bのカラー反映プレビュー：グレー＋プリセット5色）。実装の元にしたモック。
- **確認済み**：デプロイ後、本番で個別PDF・一括ZIPの見た目を確認（グレー・会社カラー）。

### ⑯ 振込先の文字を1.2倍に拡大【index.js・反映済み】

- 見やすさ向上のため、`.bank-info-title` を11px→**13.2px**、`.bank-info-body` を13.5px→**16.2px**に変更（1.0〜1.6倍を0.2刻みで比較し1.2倍を採用）。左バー・余白・他要素のサイズは据え置き。
- 参考ファイル：`invoice-builder/pdf-bank-size-samples.html`（1.0〜1.6倍の比較モック）。

### ⑰ 複数ページ（A4超え）対応【index.js・反映済み】

- **背景**：明細が増えA4 1枚を超えたときの2ページ目のレイアウトが甘かった（特に余白）。運用注意だけに頼らず、印刷CSSで安全化する方針。
- **ページ余白をページ単位へ移設**：従来は`body`の`padding:40px`で余白を作っていたが、これは複数ページ時に**2ページ目以降の上下余白が消える**（端切れ）原因だった。`body`を`padding:0; width:100%`にし、余白は`page.pdf`の`margin:{top/bottom/left/right:'40px'}`へ移動。**全ページに均一な余白**が付く。内側の実効幅（≒714px）は従来と同じなので、実測済みの列幅（15/31/6/14/14/6/14）はそのまま有効。
- **改ページCSS**（`</style>`直前に追加）：
  - `thead { display: table-header-group; }` … 明細見出し行を各ページ先頭に自動反復（Chromium既定の明示）。
  - `tr { page-break-inside: avoid; }` … 行を改ページ境界で上下に割らない（セル内2行折り返しの行も丸ごと送る）。
  - `.total-highlight, .totals, .remarks-lower, .bank-info { page-break-inside: avoid; }` … 各まとまりを途中で分断しない。
- **収容目安**：明細が1行に収まる内容で1ページ約15行前後（角印・自由記入欄・備考が埋まると14行程度、少なければ20行程度）。セル内で2行折り返す明細が混じるとさらに減る。超過分はChromiumが自動改ページ。
- **確認用**：`invoice-builder/pdf-2page-sample.html`（2ページ送りの見え方）、`invoice-builder/バックアップ_サンプル_2ページ確認用.json`（明細28行＝確実に2ページになる復元用データ。`_type:'invoice-builder-partner-backup'`準拠）。

### ⑱ 取引先の登録上限【client.html・反映済み】

- **目的**：無尽蔵な取引先登録の予防と整頓（Firestoreは読み書き件数＋保存容量の従量課金。ただし取引先1件は数KBで保存コストは実質軽微。上限はコスト対策というより暴走予防）。
- **仕様**：`addPartner`で追加前に件数チェック。`const maxPartners = Number(currentClient?.maxPartners) > 0 ? Number(currentClient.maxPartners) : 50;`。`partners.length >= maxPartners`ならトースト表示して追加中止。
  - **既定50件はコードに持たせる**（Firestoreには入れない）。**特定クライアントだけ**上げ下げしたいときは、そのclient文書に`maxPartners`（型`int64`）をFirebaseコンソールで手入力する運用。頻度が低い前提でUI改修（index.htmlへの入力欄追加）は見送り。
  - 強制レベルは**UI上のガードのみ**（Firestore直叩きは技術上バイパス可能だが、非エンジニアのエンド顧客には実質十分）。厳格化が必要なら将来カウンタ＋セキュリティルールで対応。
- **確認済み**：`maxPartners:3`を一時設定し4件目でブロックされることを確認→フィールド削除で既定50へ復帰。

### ⑲ 旧プロジェクトのFirestore置き去りデータを削除【整理】

- Firebase分離（新`invoice-builder-4b77e`へ移行）後、旧`idea-notebook-172e0`側に残っていた`clients`コレクション（移行前の置き去りデータ）を削除。Idea Loomは`users/{userId}/ideas/{ideaId}`を使用しており`clients`とは別系統のため影響なし。Cloud Run（idea-notebook側）とも無関係。削除後もinvoice-builderは新プロジェクト単独で正常動作を確認。

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
2. **【完了・2026/7/9】** ~~Node.js 22移行と再現性の反映~~ → 反映済み。
   - デプロイ元フォルダ`C:\Users\fokar\invoice-pdf-server\`の`Dockerfile`は`node:22-slim`。同フォルダに`package-lock.json`をコピー済み(2026/7/8)で、デプロイ時も直接・間接依存が固定される。GitHubの`Dockerfile`・`package-lock.json`も反映済み(2026/7/8)。
   - `C:\Users\fokar\invoice-pdf-server\`で標準デプロイコマンド`gcloud run deploy`を実行し、Node.js 22移行と再現性を本番反映。**デプロイ後に個別・一括の両方でPDF生成を確認済み**(ポリシー遵守)。
3. **【完了・2026/7/8】** `index.html`(自由記入欄削除＋会社情報リネーム)をGitHubに反映済み。Vercelが自動デプロイ。
4. **【任意・低優先】** 削除済みGCPプロジェクト5個の30日後の完全削除確認(削除保留期間が明けるまで待機、特に作業不要)。
5. **【将来】** estimate-builderの新規リポジトリ作成(Excelインポート/エクスポート型)。
6. **【完了・2026/7/10】** ~~GitHub Actions自動デプロイ(Cloud Run向け)~~ → 導入済み。`main`へのpushで自動デプロイ。手動`gcloud run deploy`は不要になった(詳細は上記「⑦」)。
7. **【保留・優先度低】** SendGridメール送信機能。
8. **【定期見直し】** バージョン管理ポリシーに沿った半年〜1年ごとの依存バージョン点検。
9. **【完了・2026/7/10②】** ~~PDFデザイン刷新(振込先移動・角丸・取引先名・色テーマ)／請求書番号／出力Excel廃止＋JSONバックアップ移行／ボタン統一・スマホ対応~~ → 反映済み(詳細は上記「2026/7/10 セッション②」)。
10. **【将来・任意】** 一括「復元」(ZIPを選ぶと全取引先を一括で戻す)。現状は一括バックアップのみで、復元は取引先単位。
11. **【将来・任意】** 管理画面のExcel取り込みの対応形式拡張(`.ods`/`.xlsm`/`.xlsb`/`.xls`/`.csv`等。`accept`と文言の変更でSheetJSがそのまま読める。CSVのみShift-JIS対策が必要)。

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
- **~~Cloud Runデプロイは`--source .`のローカルフォルダを使う~~ → 2026/7/10改定：デプロイ元はGitHubリポジトリ**：GitHub Actions自動デプロイ導入により、`gcloud run deploy --source .`はGitHub Actions上でチェックアウトした**リポジトリの中身**をビルドするようになった。よって`package.json`・`package-lock.json`・`Dockerfile`等は**GitHubリポジトリを最新に保つこと**が再現性の条件(ローカルフォルダは無関係。旧デプロイ元`C:\Users\fokar\invoice-pdf-server\`は削除済み)。
- **縦書き(`writing-mode: vertical-rl`)の`height`は「縦の長さ」ではない**：縦書きでは`height`が文字の流れる方向(インライン方向)のサイズとして扱われる。そのため`height:auto`でも要素が枠の縦いっぱいに広がり、flexの`align-items:center`による上下中央寄せがほぼ効かない(数pxしか動かない)。縦位置を厳密に制御するのは難しいと理解しておく。角印は`text-align:start`＋`height:100%`で「各列を上端そろえ(上揃え)」に確定。
- **縦書きの列内配置は継承`text-align`の影響を受ける**：親(`.company-side`は右寄せ)から継承した`text-align`が、縦書きの各列内での文字位置を左右する。短い列が中央/端に寄る現象は、要素に`text-align: start`を明示して解消する。
- **正方形に任意長テキストを詰めると余白は原理的に残る**：完全平方数の文字数(9・16など)は枠にぴったり詰まり余白ゼロ→枠線と重なる。逆にそれ以外は最後の列や下段に必ず空きが出る。角印では「枠のマス目(18px)」と「フォント(マス目より小さめ・最大15px)」を分離し、常に余白が残るようにして重なりを防いだ。下段余白の完全解消は正方形を捨てない限り不可。
- **PDF(Puppeteer)とプレビュー(client.html)は同じ計算式を3ファイルで揃える**：角印のサイズ・フォント計算は`index.js`(両フォルダ)と`client.html`の`sealAutoPreviewHtml`で同一ロジックにすること。片方だけ変えるとプレビューとPDFがずれる。
- **【要注意・環境依存】接続フォルダのbashマウント(Linux側)が古い/末尾切れになることがある**：本セッションで`invoice-pdf-server/index.js`に対しbash(`/sessions/.../mnt/...`)側が411行で末尾切れの状態を示し続けた(ホストのRead/Write/Editでは完全な451行)。この状態で**bashの`cp`でミラーへコピーすると、切れたファイルでミラーを破損させた**。教訓：(1)`index.js`の2フォルダ同期はbash `cp`に頼らず、ホストのEdit/Writeで両フォルダへ個別に書く。(2)構文チェックは、切れて見えるときはスクラッチパッド(`outputs/`)に全文を書いて`node --check`する。(3)実デプロイ/GitHubアップロードはホストのWindowsファイルを使うので、ホスト側が完全なら実害はない。
- **(2026/7/10 セッション②で再確認) bashマウント末尾切れは client.html でも発生**：ホストの`Edit`は成功しているのにbash側`node --check`が「Unexpected end of input」になる場合、ほぼこの末尾切れ。対処：(a)ホストの`Read`/`Grep`で完全性を確認、(b)危険な追加ロジックだけスクラッチパッド(`outputs/`)に切り出して`node --check`＋スタブ実行、(c)全体の`node --check`は諦めてよい（ホスト側が正なら実害なし）。ブラウザ未導入・Puppeteer/Chromiumのダウンロードもタイムアウトするため、PDFの見た目確認は「実テンプレートを再現するnodeハーネスでHTML生成→構造チェック」＋「アップロード済みPDFは`pdftoppm`でPNG化して`Read`」で行った。
- **表の列幅は`table-layout: fixed`で固定するのが安定**：既定の`auto`は内容駆動で、長い1件（例：日付に`12/3,4,5,6,7,8,9`）が列を広げたり改行したりして表全体が崩れる。`fixed`＋`width%`指定＋`overflow-wrap:anywhere`にすると、列は指定幅で固定され長い内容は枠内で折り返す。見出しの1行維持はフォント縮小＋`word-break:keep-all`。**折り返し可否はPIL(Pillow)＋Noto Sans CJK JPで`getlength`実測**して列内寸と比較すると確実（ブラウザ無しでも検証できる）。
- **PDFの色テーマはCSS変数で切替、明るい色は相対輝度で自動補正**：`:root`に`--c-*`変数を出力し、`index.js`のスタイルは変数参照に統一。グレー/カラーの切替は変数セットの差し替えだけ。会社カラーが明るいと白文字/白地文字が読めないので、相対輝度`lum>0.32`の間だけ黒を混ぜて暗くしてから使う。うすい背景/枠は白との混合で生成。**角印(朱色)とタイトル(黒)はテーマに引っ張られない固定色**にする（役割が違うため）。
- **成果物は「1つに1役割」を守ると設計がすっきりする**：出力Excelは“昔の請求書Excel”でも“正確な復元元”でもない中途半端な立ち位置だった。PDF＝請求書、JSON＝復元、と分離。復元用は往復が壊れにくいJSONにし、体裁優先のファイルに復元機能を兼ねさせない。バックアップJSONは`_type:'invoice-builder-partner-backup'`で判定し、一括バックアップ(ZIP)も同一スキーマの単体JSONを詰めて個別復元と互換にする。
- **確認ダイアログにインフラ用語(「Firestore」等)を出さない**：エンド顧客(ITリテラシー低)には伝わらない。「『内容保存』ボタンを押すまで確定しません」のように操作で説明する。破壊的操作の取り違え対策は、致命的なときだけ強め(danger)の確認を出す（例：復元で取引先名が不一致のときだけ警告）。
- **PuppeteerのPDFで複数ページ時、`body`のpaddingは1・最終ページにしか効かない**：`margin:0`＋`body{padding}`だと2ページ目以降の上下余白が消えて端切れになる。ページ余白は`body`ではなく`page.pdf`の`margin`（または`@page{margin}`）で持たせると全ページ均一になる。あわせて`tr{page-break-inside:avoid}`で行の分断を防ぎ、`thead{display:table-header-group}`で見出し行を各ページ反復（Chromium既定だが明示推奨）。合計・振込先など「まとまり」にも`page-break-inside:avoid`。
- **Firestoreコンソールに「number」型は無い**：数値は`int64`（整数）または`double`で入れる。型を選ぶと値の入力欄が数値用に切り替わる。コード側が`Number()`で読むならどちらでも可。
- **親ドキュメント削除の抜け殻はコンソールでグレー斜体表示**：`deleteDoc`はサブコレクションを消さないため、`partners`が残る親IDが「フィールドの無いドキュメント」として一覧に斜体で残る。無害だが完全に消すには中のサブコレクションも削除する。コレクション丸ごとはコンソールの「コレクションを削除」で（数件なら抜け殻が残っていないか各docを開いて確認）。
- **既定値はDBでなくコードに持たせる設計**：取引先上限の既定50はFirestoreに入れず`client.html`のフォールバック値（`... : 50`）で保持。DBに置くのは「クライアント個別の上書き値`maxPartners`」だけ。全体既定を変えるときはコードを直す。設定を持たない大多数のクライアントに余計なフィールドが増えず、一覧もすっきりする。

---

## 開発の進め方(合意済みルール)

### 作業フォルダ(ローカルミラー)方式 ★2026/7/8導入

- デスクトップの`見積請求PWA開発`フォルダをClaudeに接続済み。この中に各リポジトリを複製(ミラー)して保持する。**このフォルダを「正(ソース・オブ・トゥルース)」とする。**
  - `見積請求PWA開発/invoice-builder/` … `index.html`・`client.html`・`invoice-builder-CONTEXT.md`・`developer-profile-updated.md`
  - `見積請求PWA開発/invoice-pdf-server/` … `index.js`・`Dockerfile`・`package.json`・`package-lock.json`

- **~~`index.js`の2フォルダ同時書き込み(2026/7/9導入)~~ → 2026/7/10廃止**：GitHub Actions自動デプロイ導入により、デプロイ元がローカルフォルダからGitHubリポジトリに移ったため、この運用は不要になった。デプロイ元フォルダ`C:\Users\fokar\invoice-pdf-server\`も削除済み。
  - **現行ルール(2026/7/10〜)**：`index.js`・`Dockerfile`等の編集先は**ミラー1箇所のみ**＝`C:\Users\fokar\Desktop\見積請求PWA開発\invoice-pdf-server\`。編集後、変更ファイルをGitHubへUpload/コミットすれば、GitHub Actionsが自動でCloud Runへデプロイする。セッション開始時に接続するフォルダも`見積請求PWA開発`の**1つだけ**でよい。
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
- **セッション開始の合言葉(2026/7/10更新)**：「デスクトップの見積請求PWA開発フォルダを接続して、invoice-builder-CONTEXT.mdとdeveloper-profile-updated.mdを読み込んで」
  - 補足：フォルダ接続はセッションごとに毎回必要（前回接続しても新セッションには引き継がれない。永続化する設定は現状なし）。合言葉に「フォルダを接続して」を含めると、接続要求→承認→読み込みが一続きで進む。
  - **接続は`見積請求PWA開発`の1つだけでよい(2026/7/10〜)**：GitHub Actions自動デプロイ導入で、デプロイ元がGitHubに移りローカルの`C:\Users\fokar\invoice-pdf-server\`は削除済みのため、旧「2フォルダ接続」は不要になった。

---

## ツール・リソース一覧

- ホスティング：Vercel(GitHub main branchから自動デプロイ)
- データベース：Firebase Firestore(`invoice-builder-4b77e`、Blazeプラン、`asia-northeast1`、invoice-builder専用)
- 認証：Firebase Auth(Google login、Tsuyoshiさん専用、`invoice-builder-4b77e`)
- PDF生成：Google Cloud Run + Puppeteer(`invoice-pdf-server`、Node.js 22 / Puppeteer 21.11.0固定・Puppeteer自前Chromium方式、GCPプロジェクトは引き続き`idea-notebook-172e0`)
- **デプロイ方式(標準・2026/7/10〜)**：GitHub Actions自動デプロイ。`invoice-pdf-server`リポジトリの`main`へUpload/コミットすると`.github/workflows/deploy.yml`が下記コマンドを自動実行する。**手動操作・ターミナルは不要**。実行状況は https://github.com/kireit0313-cloud/invoice-pdf-server/actions で確認。
  ```
  gcloud run deploy invoice-pdf-server --source . --project idea-notebook-172e0 --region asia-northeast1 --allow-unauthenticated --memory 1Gi --execution-environment gen2
  ```
  - ワークフロー定義：`invoice-pdf-server/.github/workflows/deploy.yml`(ミラーにも保持)。認証はGitHub Secret `GCP_SA_KEY`(SA `github-deploy@idea-notebook-172e0.iam.gserviceaccount.com`)。
  - フォールバック(手動)：Actionsが使えない緊急時のみ、上記コマンドを最新のローカルクローンで実行してもよい。ただし通常はGitHub経由に一本化する。

---

## 新規プロジェクト立ち上げ時の標準環境セットアップ手順(テンプレート)

> invoice-builderで確立した構成を、今後の新規プロジェクト(estimate-builder等)でもそのまま踏襲するための手順書。
> 基本方針：**非エンジニアが画面クリックだけで完結でき、ターミナル作業を極力なくす**。素のHTML/JS＋Vercel＋Firebase＋Cloud Run(必要な場合)＋GitHub Actions自動デプロイ。

### A. リポジトリとホスティング(フロントエンド)

1. GitHubで新規リポジトリを作成(`kireit0313-cloud/プロジェクト名`)。
2. 素のHTML/JS(ビルドツールなし)で構成。編集はGitHub Web UIまたはローカルミラー＋Upload。
3. Vercelでリポジトリをインポートし、`main`ブランチ自動デプロイを有効化。
4. 参照実装が要るときはIdea Loom/invoice-builderの同スタック構成を流用。

### B. Firebase(DB・認証) ※プロジェクトごとに完全分離する

1. **プロジェクトごとに専用のFirebaseプロジェクトを新規作成**(既存プロジェクトのFirestoreルールを共有しない。共有すると別アプリのルールを上書きする事故が起きる)。Blazeプラン、リージョンは`asia-northeast1`推奨。
2. Firestore・Authentication(Google login等)を有効化。Authの「承認済みドメイン」にVercel本番URLを追加。
3. `firebaseConfig`は**HTMLファイルごとに個別に持つ構成**になりがちなので、複数ファイルある場合は全ファイルに同じ設定を入れる。
4. Firestoreセキュリティルール：**サブコレクションのルールは必ず親の`match`ブロックの内側にネスト**する(兄弟に書くと拒否される)。
5. GCPプロジェクト作成上限に注意。上限エラー時は「割り当て引き上げ申請」＋「未使用プロジェクト削除(削除は30日保留)」で対処。

### C. Cloud Run(サーバー処理が必要な場合。PDF生成等)

1. サーバー用リポジトリを作成。`Dockerfile`＋`package.json`＋`index.js`。
2. **バージョンは必ず固定**：Node.jsベースイメージはメジャー明示(例`node:22-slim`)、npm依存は`^`を使わず正確指定＋`package-lock.json`をコミット。Puppeteer利用時はOS(apt)標準Chromiumを使わず、Puppeteer自前ダウンロードのChromiumに対応した組み合わせのみ使用。
3. 初回だけ手動`gcloud run deploy ... --execution-environment gen2`で疎通確認。以後はD.の自動デプロイへ。

### D. GitHub Actions自動デプロイ(Cloud Run向け・invoice-builderで確立した本命手順)

1. **デプロイ専用サービスアカウントを作成**：GCPコンソール→IAMと管理→サービスアカウント→作成。名前例`github-deploy`。ロールは**編集者＋サービスアカウントユーザー**の2つ(画面クリックだけで確実に動く最小構成。より厳格に絞るなら run.admin / cloudbuild.builds.editor / artifactregistry / storage 等に分解)。
2. そのSAの**JSONキーを作成・ダウンロード**(キー方式を採用。キーレスのWorkload Identity連携はコマンド操作が要るため非エンジニア向けでない)。
3. GitHubの対象リポジトリ→Settings→Secrets and variables→Actions→**New repository secret**。Name=`GCP_SA_KEY`、値にJSON全文を貼る。登録後ローカルのJSONは削除。
4. リポジトリに`.github/workflows/deploy.yml`を追加(下記ひな型。`サービス名`・`プロジェクトID`・リージョン・リソース指定だけ差し替え)。`main`へのpushで自動デプロイ。
5. Actionsタブで緑=成功を確認し、成果物(PDF等)の主要機能を実機確認してから完了とする。

**`deploy.yml`ひな型：**
```yaml
name: Deploy to Cloud Run
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - uses: google-github-actions/setup-gcloud@v2
      - run: |
          gcloud run deploy サービス名 \
            --source . \
            --project プロジェクトID \
            --region asia-northeast1 \
            --allow-unauthenticated \
            --memory 1Gi \
            --execution-environment gen2
```

### E. 開発ワークフロー(共通)

- ローカルミラー(接続フォルダ)を「正」とし、Claudeが直接編集→変更ファイルをGitHub Web UIでUpload/コミット。Vercel/Cloud Runは自動デプロイ。
- 接続フォルダはプロジェクト1つ分でよい(デプロイ元ローカルフォルダは自動デプロイ化により不要)。
- コンテキストは`プロジェクト名-CONTEXT.md`に一元化し、接続フォルダ内を正・GitHubを予備バックアップとする。
- 依存/ベースイメージ変更後は必ず主要機能を実機確認してからコミット。半年〜1年ごと、または「変えてないのに急に動かない」時に依存固定を最優先で点検。
