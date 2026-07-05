# invoice-builder プロジェクト引継ぎ

## 目的・背景

Tsuyoshiさん（非エンジニア）が、coconalaなどのフリーランスプラットフォーム経由で
中小企業向けに販売しているWebベースの請求書作成PWA「invoice-builder」の開発プロジェクト。

- 業務モデル：Tsuyoshiさんがクライアント（中小企業）ごとにExcelテンプレートをアップロード・設定し、
  専用URLを発行。クライアントはそのURLで請求書PDFを自分で生成できる。
- ゴール：毎月のExcelコピペ作業をなくすこと。

## 技術スタック

- プレーンHTML/JavaScript（ビルドツールなし）、GitHub Web UIで直接編集
- ホスティング：Vercel（GitHub main branchから自動デプロイ）
- データベース：Firebase Firestore
- 認証：Firebase Auth（Google login、Tsuyoshiさん専用の管理者アクセス）
- Excel処理：SheetJS（CDN）
- ZIP生成：JSZip（CDN）
- PDF生成：Google Cloud Run + Puppeteer

### リポジトリ・アカウント情報

- GitHubリポジトリ：`kireit0313-cloud/invoice-builder`
- 本番URL：`https://invoice-builder-iota-lac.vercel.app/`
- Cloud Runサービス：`invoice-pdf-server`（リポジトリ：`kireit0313-cloud/invoice-pdf-server`）
- Cloud Runローカルパス：`C:\Users\fokar\invoice-pdf-server\`
- Google アカウント/プロジェクト：kireit0313@gmail.com / idea-notebook-172e0
- 参考プロジェクト：Idea Loom（`kireit0313-cloud/idea-notebook`）— 同じスタックの参照実装

将来的に「estimate-builder」（見積書作成ツール）を別リポジトリとして計画中
（Excelインポート/エクスポート型、Firestore顧客保存なし、過去Excel再インポート対応）。

---

## 現在の状態（本セッション終了時点）

### ① docType（見積書／請求書）切り替えの完全削除【本セッション完了】

- 管理画面(index.html)・クライアント画面(client.html)の両方から、見積書／請求書切替UI・ロジックを完全撤去
- アプリは請求書専用となった
- Cloud Run側`index.js`は変更不要（`docType`が渡されなくても`'請求書'`扱いで動作する既存ロジックのため）

### ② 管理画面(index.html)の改善【本セッション完了】

- カード見出しの①②③④番号を全廃止（alert文の番号も削除）
- 「案件情報・送付先情報（自由項目）」→「自由項目記入欄（上段）」に名称変更（機能は従来通り、`projectFields`として保存）
- 新規カード「自由記入欄（下段）」を追加
  - 単一テキストエリア、Firestoreに`remarksLower`として保存
  - クライアント画面での新規取引先作成時、この値がデフォルト値として引き継がれる
  - **PDFでの「未記入なら非表示」表示ロジックはCloud Run側`index.js`が未対応（次回作業）**
- 登録済みクライアント一覧に「削除」ボタンを追加
  - 確認ダイアログ→取引先(partners)サブコレクションを先に全削除→クライアント本体を削除
  - 動作確認済み・問題なく削除できることを確認済み
- 「データのバックアップ」カードを追加
  - 全クライアント＋各取引先データをJSON形式でダウンロード（`invoice-builder-backup_yyyymmdd.json`）
- 「データの復元（クライアント単位）」カードを追加
  - バックアップJSONをアップロード→復元したいクライアントを選択→確認ダイアログ→復元
  - 復元時、対象クライアントの既存取引先を全削除してからバックアップの内容で書き戻す（完全ロールバック方式）
  - Firestore Timestampはbackup JSON内の`{seconds, nanoseconds}`から`Timestamp`オブジェクトに変換して復元
  - 削除後の個別復元、動作確認済み

### ③ クライアント画面(client.html)の改善【本セッション完了】

- 「自由記入欄（任意）」→「自由項目記入欄（上段）」に名称統一（管理画面との一貫性）
- 新規カード「自由記入欄（下段）」を追加
  - 明細入力カードの直後・アクションボタンの手前に配置（PDF上での表示位置＝明細の下、に合わせた配置）
  - `remarksLower`として取引先(partner)ごとにFirestoreに保存
  - 新規取引先作成時、クライアント設定の`remarksLower`をデフォルト値として継承
  - 内容保存／PDF出力／一括保存／一括出力の全経路でデータが失われないよう受け渡しを追加済み
  - `generatePDF()`のペイロードに`remarksLower`を追加済み（Cloud Run送信はOK、**表示ロジックは未実装**）
- ヘッダー表示の見直し
  - 取引先詳細画面：クライアント名を非表示にし、取引先名を主表示に変更
    （`headerTitle` = 取引先名、`headerSub` = 「請求書入力」のみ）
  - 取引先一覧画面：従来通りクライアント名を主表示のまま維持
    （`headerTitle` = クライアント名＋「様」、取引先一覧画面に戻った際に復元されるよう実装）
  - 一括編集モードは対象外（取引先一覧から開くため、元々クライアント名表示のまま）

動作確認：記入欄の追加・保存・ヘッダー表示、いずれも問題なし。

---

## 既知の残作業・次回優先事項

1. **【最優先】Cloud Run側`index.js`の対応**
   - クライアント画面から送信されている`remarksLower`を受け取り、PDF上で明細の下段に表示する処理を追加
   - `remarksLower`が空の場合はPDF上に何も表示しない（非表示）仕様
   - `index.js`はローカルファイルのため、Tsuyoshiさんが`findstr /n`で該当箇所を確認→スクリーンショット共有→Claudeが差分提示、の手順で進める
2. **【将来】estimate-builderの新規リポジトリ作成**（Excelインポート/エクスポート型）
3. **【保留】GitHub Actions自動デプロイ（Cloud Run向け）** — 現状は手動コピー＋`gcloud run deploy`
4. **【保留・優先度低】SendGridメール送信機能** — Tsuyoshiさんのアドレスからの送信になるため実用性の観点で保留中

---

## 学び・注意点（蓄積）

- **列マッピングは位置ベース**：単価・金額はどちらも「金額入力」型として自動判定されるため、種類ではなく列の並び順（インデックス）でマッピングする。
- **Cloud Run `index.js`整合性ルール**：`client.html`からCloud Runへ渡すデータを変更したら、必ず`index.js`側の受け取りロジックも同時に確認すること（過去に未定義変数によるCORSエラー風のバグの原因になった）。
- **`encodeURIComponent()`**：`Content-Disposition`ヘッダーの日本語ファイル名には必須。
- **`const`再宣言バグ**：分割代入した変数名をそのまま`const`で再宣言すると`SyntaxError`。分割代入側の変数名をリネームして回避。
- **ブラウザキャッシュ**：コードが反映されないように見える場合、まずハードリフレッシュ（`Ctrl+Shift+R`）を試す。
- **`onclick`属性への直接埋め込みリスク**：取引先名などをそのまま`onclick`属性に埋め込むと特殊文字でSyntaxErrorになる。`data-*`属性＋`el.dataset`で読み取る方式にする。
- **一括操作は明示的に引数を渡す**：`currentPartner`などのグローバル状態に依存する関数は一括処理ループ内で静かに失敗する。
- **Firestore設計**：最新1件を上書きするのみで履歴は残らない。再保存時は`createdAt`を保持すること。
- **Firestoreの`deleteDoc`はサブコレクションを自動削除しない**：親ドキュメントを消してもサブコレクション（`partners`）は残るため、明示的に全件削除してから親を削除する必要がある。
- **バックアップ／復元とFirestore Timestamp**：`JSON.stringify`でTimestampは`{seconds, nanoseconds}`相当の形に変換される。復元時は`new Timestamp(seconds, nanoseconds)`で明示的に戻す必要がある（そのままsetDocすると単なるMapフィールドになりTimestampとして扱われなくなる）。

---

## 開発の進め方（合意済みルール）

- **コード提供方式**：差分方式（差分の前後を明示し、GitHub Web UIで貼り替えられる形で提示）。変更範囲が広い場合のみファイル全体を提示。
- **ファイル編集**：GitHub Web UI上で直接。ローカルビルド環境なし。
- **GitHub読み込み**：`curl -s https://raw.githubusercontent.com/kireit0313-cloud/[repo]/main/[filename]`をbashツールで実行。行番号特定には`grep -n`。`web_fetch`はGitHub rawファイルには使えない。
- **ローカル限定ファイル**：Cloud Runの`index.js`はClaudeが直接読めないため、Tsuyoshiさんが`findstr /n`で確認しスクリーンショット共有。
- **コンテキスト継続**：`invoice-builder-CONTEXT.md`はセッション終了時に更新し、GitHub Web UIで手動コミット。`developer-profile-updated.md`は正しいスタック定義（プレーンHTML、React/Viteなし）を記載。
- **ターミナルコマンド**：一つずつ実行（まとめて貼らない）。
- **セッション開始の合言葉**：「invoice-builder-CONTEXT.mdとdeveloper-profile-updated.mdをGitHubから読み込んで」

---

## ツール・リソース一覧

- ホスティング：Vercel（GitHub main branchから自動デプロイ）
- データベース：Firebase Firestore
- 認証：Firebase Auth（Google login、Tsuyoshiさん専用）
- PDF生成：Google Cloud Run + Puppeteer（`invoice-pdf-server`）
- デプロイコマンド：`gcloud run deploy invoice-pdf-server --source . --region asia-northeast1 --allow-unauthenticated --memory 1Gi`（`C:\Users\fokar\invoice-pdf-server\`から実行）
- Excel処理：SheetJS（CDN）
- ZIP生成：JSZip（CDN）
- 参考プロジェクト：Idea Loom（`kireit0313-cloud/idea-notebook`）
