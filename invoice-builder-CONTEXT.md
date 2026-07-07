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
- Cloud Runサービス：`invoice-pdf-server`(リポジトリ：`kireit0313-cloud/invoice-pdf-server`、GitHubにアップロード済み・Claudeが直接curlで読める)
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

### ① Firebaseプロジェクト完全分離【本セッション完了・最優先ブロッカー解消】

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

- Node.js 20(EOL)→ 22へ移行
- Puppeteerを`^21.0.0`→`21.11.0`に正確固定

---

## 既知の残作業・次回優先事項

1. **【推奨】`package-lock.json`のGitHubへのコミット**(Puppeteerの間接依存バージョンのブレを完全に排除するため、未完了)
2. **【任意・低優先】削除済みGCPプロジェクト5個の30日後の完全削除確認**(削除保留期間が明けるまで待機、特に作業不要)
3. **【将来】estimate-builderの新規リポジトリ作成**(Excelインポート/エクスポート型)
4. **【保留】GitHub Actions自動デプロイ(Cloud Run向け)** — 現状は手動`gcloud run deploy`
5. **【保留・優先度低】SendGridメール送信機能**
6. **【定期見直し】バージョン管理ポリシーに沿った半年〜1年ごとの依存バージョン点検**

---

## バージョン管理ポリシー

| 項目 | 方針 |
|---|---|
| **Node.jsベースイメージ** | メジャーバージョンは明示的に固定(現在：`node:22-slim`)。EOLの半年前を目安に次のLTSへの移行を検討。Node.js 22のEOLは2027年4月30日予定 |
| **Puppeteerのバージョン** | `package.json`はキャレット(`^`)を使わず正確なバージョンで固定(現在：`21.11.0`)。`package-lock.json`も必ずGitHubにコミットする(未完了) |
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
- **Cloud Run `index.js`整合性ルール**：`client.html`からCloud Runへ渡すデータを変更したら、必ず`index.js`側の受け取りロジックも同時に確認すること。
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

---

## 開発の進め方(合意済みルール)

- **コード提供方式**：
  - すでに会話内で全文を読み込み済みのファイルを修正する場合 → **検証済みの完成版ファイルをまるごと渡す**
  - まだ会話内で読み込んでいないファイルを修正する場合 → **差分方式**(トークン節約のため)
- **ファイル編集**：GitHub Web UI上で直接、またはClaudeが渡した完成版ファイルで上書き。ローカルビルド環境なし。
- **GitHub読み込み**：`curl -s https://raw.githubusercontent.com/kireit0313-cloud/[repo]/main/[filename]`をbashツールで実行。行番号特定には`grep -n`。GitHub API(`api.github.com`)はレート制限にかかりやすいため、可能な限り`raw.githubusercontent.com`を直接使う。
- **`invoice-pdf-server`(index.js・Dockerfile・package.json等)はGitHubにアップロード済み**：Claudeが直接curlで読み込める。
- **コンテキスト継続**：`invoice-builder-CONTEXT.md`はセッション終了時に更新し、GitHub Web UIで手動コミット。`developer-profile-updated.md`は正しいスタック定義(プレーンHTML、React/Viteなし)を記載。
- **ターミナルコマンド**：一つずつ実行(まとめて貼らない)。
- **トラブル時の切り分け方針**：コード変更後にエラーが出た場合、まず「コード変更前の状態に戻しても同じエラーが再現するか」を確認し、コード起因かインフラ・環境起因かを切り分ける。
- **セッション開始の合言葉**：「invoice-builder-CONTEXT.mdとdeveloper-profile-updated.mdをGitHubから読み込んで」

---

## ツール・リソース一覧

- ホスティング：Vercel(GitHub main branchから自動デプロイ)
- データベース：Firebase Firestore(`invoice-builder-4b77e`、Blazeプラン、`asia-northeast1`、invoice-builder専用)
- 認証：Firebase Auth(Google login、Tsuyoshiさん専用、`invoice-builder-4b77e`)
- PDF生成：Google Cloud Run + Puppeteer(`invoice-pdf-server`、Node.js 22 / Puppeteer 21.11.0固定・Puppeteer自前Chromium方式、GCPプロジェクトは引き続き`idea-notebook-172e0`)
- **デプロイコマンド(標準形)**：
