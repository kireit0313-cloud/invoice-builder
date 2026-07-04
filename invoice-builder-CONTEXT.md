# 見積書・請求書メーカー 開発引継ぎドキュメント

**最終更新：2026年7月4日（ステップ14-2完了：PDFフォーマット修正／明細ラベルのクライアント別カスタマイズ方針決定）**

---

## プロジェクト概要

自営業・中小企業向けの見積書・請求書作成PWA。
coconala等のクラウドソーシングでの受注販売を想定した納品モデル。

### 納品フロー
1. TsuyoshiがExcelテンプレートを読み込み、設定作業（15〜30分/件）
2. 顧客専用URLを発行して納品
3. 顧客はURLにアクセスして入力・PDF出力するだけ
4. 顧客側のランニングコストなし

---

## リポジトリ・URL情報

| 項目 | 内容 |
|------|------|
| GitHubリポジトリ | https://github.com/kireit0313-cloud/invoice-builder（Public） |
| 公開URL（Vercel） | https://invoice-builder-iota-lac.vercel.app/ |
| 参照プロジェクト | https://github.com/kireit0313-cloud/idea-notebook |

---

## 技術スタック

| レイヤー | 採用技術 |
|---------|---------|
| フロントエンド | HTML + JavaScript（素のHTML方式、ビルドツールなし） |
| ホスティング | Vercel（GitHubのmainブランチにpushで自動反映） |
| バックエンド | Cloud Run（invoice-pdf-server デプロイ済み） |
| DB | Firebase Firestore |
| 認証 | Firebase Auth（Googleログイン） |
| Excel読み込み | SheetJS（ブラウザ内処理、.xlsx/.xls両対応） |
| PDF生成 | Puppeteer on Cloud Run（デプロイ済み・稼働中） |
| 一括ZIP出力 | JSZip（CDN読み込み） |
| メール送信 | SendGrid 月100通無料枠（将来実装・要検討） |

---

## 【重要】製品ライン分割の方針（2026年7月4日決定・継続）

現在の`invoice-builder`は**請求書専用システムとして完成させる**方針に確定。
見積書は**別システム・別リポジトリ**として今後新規開発する（Phase 2）。

### 請求書システム（このリポジトリ・invoice-builder）
- 取引先は固定・少数（10〜30件程度）
- 毎月同じ取引先に請求 → Firestoreでの前回データ自動引き継ぎが有効
- 取引先ごとに締め日を設定し、発行日を自動計算
- **現状のFirestore設計のまま継続**

### 見積書システム（将来・別リポジトリ：estimate-builder（仮称））
- 顧客は都度発生・数百件も想定（個人向け商売の場合）
- Firestoreに顧客マスタを保存しない設計とする
- リピート顧客への対応は「過去のExcelデータをアップロード→インポート→編集→PDF出力」の方式
- フェーズ1（請求書システムの完成）を優先し、その後着手

### docType切り替えについて（未着手）
- 請求書/見積書切り替えUIは削除し請求書専用にする作業が残っている

---

## 【本日決定】明細項目名のクライアント別カスタマイズ方針（重要）

### 背景・経緯
- 本来の設計思想：クライアントから受け取るExcelの列名（業種によって「作業日」「日付」「施術日」など異なる）をそのまま使い、明細テーブル・PDF・Excel出力に反映することで、クライアントごとの専用ツールというカスタマイズ感を出す狙いがあった。
- しかし現状、Excelアップロード時に管理画面（index.html）で`columns`（列名・型）を検出・保存する機能はあるが、**client.html側の明細テーブル／PDF／Excel出力のどこにもこの`columns`データが反映されておらず、常に固定の6項目（作業日・サービス名・数量・単価・金額・備考）がハードコードされている**という設計と実装の乖離が発覚した。

### 検討した選択肢と結論
- 「列の並び順・列数もクライアントによって完全可変にする」案も検討したが、以下の理由で見送り：
  - 実装コストが非常に大きい（データ構造・入力画面・金額自動計算ロジック・PDF・Excel出力すべてに影響）
  - 請求書明細で6項目以外に必要になりそうな項目（単位・現場名・時刻・型番など）は、ほとんどが既存の「サービス名」「備考」への自由記入で代用可能と判断
- **最終方針：列の数・並び順は現状の6列で固定のまま、列の「表示ラベル（名前）」だけをクライアントごとにカスタマイズ可能にする**
  - 例：「作業日」→「日付」「施術日」「訪問日」など、業種に応じて呼び方だけ変更できる
  - データ構造（`workDate`, `service`, `quantity`, `unitPrice`, `amount`, `notes`等のキー名）は変更しない。表示ラベルのみ`columns`（Firestoreに保存済みの`{name, type}`配列）から取得する
  - 税率列は今まで通りExcelの列とは無関係にシステム側で常に自動追加する固定列（今回変更なし・確認済み）

### 未着手（次回セッションの最優先タスク）
1. `client.html`の明細テーブルヘッダー（個別入力画面・一括編集画面の両方）を、ハードコードされた「作業日」「サービス名」等から、`currentClient.columns`の`name`を使った表示に変更する
2. Excel出力のヘッダー行（`xlsData`の1行目、個別・一括両方）も同様に`columns`の`name`を反映する
3. PDF出力（index.js）のテーブルヘッダーも、client.htmlから渡された列ラベルを使うように変更する（現状は「日付」「内容」等が固定でハードコードされている）
4. `columns`が未設定（旧仕様のまま／Excel未アップロード）の場合のフォールバック処理を用意する（デフォルトラベル：作業日・サービス名・数量・単価・金額・備考）
5. 管理画面（index.html）の列設定テーブルは既存のまま活用できる見込み（`columns`の保存自体は既にできている）

---

## Firebase設定

Idea LoomのFirebaseプロジェクトを共用

```javascript
const firebaseConfig = {
  apiKey: "AIzaSyATf8RjZ7y_wjiiPJ1VR9KXktbDPXuMpFo",
  authDomain: "idea-notebook-172e0.firebaseapp.com",
  projectId: "idea-notebook-172e0",
  storageBucket: "idea-notebook-172e0.firebasestorage.app",
  messagingSenderId: "682338340449",
  appId: "1:682338340449:web:598a4db429e148990e6420"
};
```

### Firestore設計（確定・本日更新）

```
clients/{clientId}/                      ← clientIdはcrypto.randomUUID()で生成（新規登録時）
  clientName: "フジタリフォーム"
  clientId: "フジタリフォーム"           ← 表示用（IDとは別）
  columns: [{ name, type }]              ← Excel取り込み時の列名・型（★現状表示に未反映。次回対応）
  defaultTaxRate: 10
  docType: "請求書" or "見積書"
  companyInfo: {
    name,                                ← 社名（変更不可）
    address, tel, invoiceNumber, bankInfo,
    customFields: [{ label, value }]     ← 業種対応の追加項目（クライアント側で編集可）
  }
  projectFields: [{ label, value }]      ← 管理画面側の自由項目（旧称・管理画面でのみ使用）
  createdBy: uid
  createdAt: timestamp                   ← 初回のみ記録（再保存で変わらない）
  updatedAt: timestamp

clients/{clientId}/partners/{partnerId}  ← 取引先サブコレクション
  partnerName: string                    ← 取引先名
  honorific: "御中" | "様"               ← 敬称（本日追加。取引先登録時に選択。未設定時は"御中"にフォールバック）
  issueDay: "末日" | "25" | "20" | "15" | "10" | "手入力"  ← 締め日パターン
  issueDate: string                      ← 直近保存された発行日（表示文字列、例:"2026年7月31日"）
  lineItems: [...]                       ← 前回PDF保存時の明細（次月インポート元）
  freeFields: [{ label, value }]         ← 取引先ごとの自由記入欄（旧projectFields相当）
  createdAt: timestamp
  updatedAt: timestamp
```

### Firestoreセキュリティルール（現在適用中・変更なし）

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /clients/{clientId} {
      allow get: if true;
      allow list: if request.auth != null;

      allow create, delete: if request.auth != null;
      allow update: if request.auth != null
        || (
          request.resource.data.diff(resource.data).affectedKeys()
            .hasOnly(['companyInfo', 'projectFields', 'updatedAt'])
          && request.resource.data.companyInfo.name == resource.data.companyInfo.name
          && request.resource.data.clientName == resource.data.clientName
          && request.resource.data.clientId == resource.data.clientId
        );

      match /partners/{partnerId} {
        allow get: if true;
        allow list: if true;
        allow write: if true;
      }
    }
  }
}
```

承認済みドメイン：invoice-builder-iota-lac.vercel.app（追加済み）

---

## 明細データ仕様（現状・列ラベルのみ今後カスタマイズ予定）

### 6列フォーマット（縦持ち：1行＝1明細、列の数・順番は固定）

| 列名（現状の表示ラベル） | 形式 | 備考 |
|------|------|------|
| 日付（旧：作業日） | 自由記述 | 「6/2」「6/2〜6/4」どちらも可。表示ラベルは今後クライアントごとに変更可能にする予定 |
| 内容（旧：サービス名） | テキスト | 同上 |
| 数量 | 数値（任意） | 空欄可 |
| 単価 | 数値（任意） | 空欄可 |
| 金額 | 数値 | 数量×単価で自動計算。どちらか空欄なら手入力 |
| 備考 | テキスト | 空欄運用でも実害なしと判断し、項目として維持 |

### 税率（明細行ごと・Excelの列とは無関係の固定システム列）
- 0%以上の整数を自由入力（10・8・0はボタンで即入力）
- 0%：税込み・対象外扱い（消費税を追加計算しない）
- デフォルト値はFirestoreのdefaultTaxRateから取得

---

## 発行日仕様（確定・変更なし）

- 取引先ごとに**締め日パターン**を登録：「末日」「25日」「20日」「15日」「10日」「手入力」
- PDF出力時に締め日パターンから当月の発行日を自動計算（例：末日→2026年7月31日）
- 個別ページで発行日を確認・「変更」ボタンで手入力上書き可能
- 「内容保存」ボタンで発行日・自由記入欄・明細をまとめてFirestoreに保存
- 一括編集モードでは発行日は**表示のみ**（編集不可）

---

## 敬称（御中／様）仕様（本日実装）

- 取引先の新規登録時に、敬称を選択：「御中（法人）」または「様（個人）」
- Firestoreの`partners/{partnerId}.honorific`に保存
- 個別PDF出力・一括PDF出力の両方で、取引先ごとの`honorific`を`generatePDF()`の引数として明示的に渡す方式に修正（修正前は一括モードで`currentPartner`という個別画面用の変数を参照していたため反映されないバグがあった）
- 未設定の場合は"御中"にフォールバック

---

## PDF仕様（本日改修）

### レイアウト（本日変更）
- タイトル：docTypeに応じて「請求書」or「見積書」
- **左：取引先名（敬称付き）＋「下記の通りご請求申し上げます。」の文言**（本日追加）
- **右上：請求日＋会社情報（社名・住所・TEL・インボイス番号・customFields）＋印鑑欄**（発行日を右上に移動）
- **税込合計金額を枠線付きで大きく強調表示**（本日追加。表示は「¥○○○」のみ、「円」の重複表記は削除済み）
- 中央：明細テーブル（6列。列幅を変更：内容の幅を拡大、備考・税率の幅を縮小）
  - ヘッダー表記：「日付」「内容」「数量」「単価」「金額」「税率」「備考」（旧：作業日・サービス名。※次回、クライアント別カスタマイズ対応予定）
- 右下：税率ごとの内訳＋合計
- **振込先を枠線・背景色付きで大きく強調表示**（本日改修。請求書のみ表示）
- 左下：自由記入欄（freeFields）※取引先ごとに個別

### 今後の拡張予定（バックログ）
- PDFの横向き（landscape）レイアウト対応
- 明細テーブルの下にも自由記入欄を追加できるようにする
- 明細ラベルのクライアント別カスタマイズ（上記「本日決定」セクション参照・次回最優先）

---

## Cloud Run情報

| 項目 | 内容 |
|------|------|
| サービス名 | invoice-pdf-server |
| リージョン | asia-northeast1（東京） |
| URL | https://invoice-pdf-server-682338340449.asia-northeast1.run.app |
| プロジェクト | idea-notebook-172e0 |
| ローカルソース | C:\Users\fokar\invoice-pdf-server\ |

デプロイコマンド：
```
cd invoice-pdf-server
gcloud run deploy invoice-pdf-server --source . --region asia-northeast1 --allow-unauthenticated --memory 1Gi
```

重要：GitHubではなくローカルPC管理。変更時はローカルに上書き保存後にgcloudでデプロイ。

### 本日のindex.js修正内容
1. PDFのHTMLテンプレート全面改修（レイアウト・スタイル変更、上記「PDF仕様」参照）
2. 分割代入に`honorific`（敬称）を追加
3. **バグ修正**：`honorific`という変数名を2箇所で`const`宣言してしまい`SyntaxError: Identifier 'honorific' has already been declared`が発生 → 分割代入側を`honorific: honorificInput`にリネームして解消
4. 税込合計金額表示から重複していた「円」表記を削除

**教訓：分割代入で受け取った変数名と、その後に自分で新しく`const`宣言する変数名が重複しないよう注意する。**

---

## ファイル構成

```
invoice-builder/（GitHub管理）
├── index.html    ← Tsuyoshi管理画面
└── client.html   ← クライアント向け入力画面

C:\Users\fokar\invoice-pdf-server\（ローカル管理）
├── index.js      ← PDF生成サーバー本体
├── package.json
└── Dockerfile
```

---

## 管理画面（index.html）の機能

1. クライアント名入力
   - 新規作成時：crypto.randomUUID()でランダムなIDを自動生成
   - 編集中は「既存クライアントを編集中です」バッジ＋「新規登録に戻る」ボタンを表示
2. 会社情報・書類設定（請求書/見積書・社名・住所・TEL・インボイス番号・振込先・追加項目）
3. 案件情報・送付先情報（独立した自由項目）
4. デフォルト税率設定
5. Excelテンプレートアップロード（列自動判定：列名・型を検出し`columns`として保存。★次回、この情報をclient.html側の表示に反映させる作業を行う）
6. 列設定確認・修正（「列名（Excelの1行目）」「自動判定された種類」の一覧表示）
7. 設定を保存する（Firestore上書き保存・createdAt保持）
8. 登録済みクライアント一覧（URLコピー・編集ボタン）

---

## クライアント画面（client.html）の機能

### 画面フロー
```
起動時
  └→ 取引先一覧画面
        ├→ 取引先を1件タップ → 明細入力画面（先月データがあれば自動インポート、発行日は締め日から自動計算）
        │     └→ 内容保存（発行日・自由記入欄・明細を保存） or PDF出力・Excel出力
        └→ チェックボックスで複数選択 → 「一括編集モードで開く」
              └→ 一括編集画面（取引先ごとにカード表示・明細のみ編集可、発行日は表示のみ）
                    ├→ 全件内容保存（明細をまとめて保存）
                    └→ 一括出力（PDF＋Excel）ZIP → Firestoreに各取引先の明細を自動保存
```

### 取引先一覧画面
- 取引先の一覧表示（最終更新日付き）
- 取引先の新規追加時に**敬称プルダウン（本日追加：御中／様）**と**締め日プルダウン**を選択
- 取引先の削除
- チェックボックスで複数選択 → 一括編集モードで開く
- 会社情報の編集（社名以外・Firestore保存）

### 明細入力画面（個別）
- **発行日欄**：締め日パターンから自動計算された発行日を表示。「変更」ボタンで手入力に切替可能
- 自由記入欄（取引先ごとに項目名・内容を自由追加）
- 明細テーブル（6列＋税率）への入力。**ヘッダー表記は現状「作業日」「サービス名」のまま**（PDF側の表記のみ本日「日付」「内容」に変更済みで、入力画面とPDFで表記が一致していない状態。次回のラベルカスタマイズ対応時に統一する）
- 先月データの自動インポート（PDF保存済みの場合）
- 税率ボタン（10%・8%・0%）、その他は数値直入力
- 金額自動計算（数量×単価）or 手入力
- 税率ごとの合計内訳を自動表示
- アクションボタン：**内容保存（グレー）／Excel出力（緑 #1D6F42）／PDF出力（赤 #D32F2F）** の順に配置

### 一括編集モード
- チェックした取引先の明細を取引先ごとのカードで縦に並べて表示
- 各カードは折りたたみ可能（▲▼）
- 各カード上部に発行日を**表示のみ**
- 先月データを自動インポートした状態で開く
- 行追加・金額修正・税率変更など明細のみ個別に編集可能
- 「全件内容保存」ボタン：チェックした全取引先の明細をまとめてFirestoreに保存
- 「一括出力（PDF＋Excel）ZIP」で一括生成＋Firestoreへ各取引先の明細を保存
- **本日修正**：`generatePDF()`呼び出し時に各取引先の`honorific`を明示的に渡すよう修正（修正前は敬称が反映されないバグがあった）

### 重要実装メモ
- onclick属性に取引先名を直接埋め込むとSyntaxErrorになる
  → data-pid属性にIDのみ格納し、el.dataset.pidで取得する方式を採用
- 一括編集モードのテーブルIDは `betbl_{partnerId}`、合計エリアIDは `betot_{partnerId}` の規則
- `bulkGetLineItems()`が返すオブジェクトのキーは`date`（`workDate`ではない）。個別ページの`getLineItems()`とキー名の対応関係に注意
- **`generatePDF()`関数はhonorificを引数として受け取る方式に変更済み**（`currentPartner`のようなグローバル変数への依存をなくし、個別・一括どちらの呼び出しからも正しい取引先のhonorificが渡るようにした）

---

## 完了済みステップ

- [x] ステップ1〜11：基盤構築（認証・Firestore・Excel読込・PDF生成・Cloud Run）
- [x] ステップ12：PDFデザインのカスタマイズ（全項目完了）
- [x] ステップ13-1：セキュリティ強化
- [x] ステップ13-2：取引先管理機能（client.html全面リニューアル）
- [x] ステップ13-3：一括編集モード
- [x] ステップ13-4：一括編集モードにExcel同梱
- [x] ステップ14-1：発行日の自由記入機能
- [x] **ステップ14-2（本日完了）：PDFフォーマット修正・敬称機能**
  - 請求日を右上に移動
  - 取引先ごとに敬称（御中／様）を選択できるように追加（個別・一括PDF両方に反映、バグ修正込み）
  - 税込合計金額を枠線付きで大きく強調表示（「円」の重複表記を削除）
  - 「下記の通りご請求申し上げます。」の文言を追加
  - 明細テーブルの列幅調整（内容の幅を拡大、備考・税率を縮小）
  - PDF上のラベルを「作業日→日付」「サービス名→内容」に変更（※入力画面側は未変更。次回、クライアント別カスタマイズ対応時に統一予定）
  - 振込先を枠線・背景色付きで大きく強調表示
  - **設計方針決定**：明細項目の列数・並び順は固定のまま、表示ラベルのみクライアントごとにカスタマイズ可能にする方針を確定（詳細は「本日決定」セクション参照）

---

## 次のステップ候補

### 【最優先・次回セッション開始点】明細ラベルのクライアント別カスタマイズ実装
「本日決定」セクションの未着手タスク1〜4を参照。具体的には：
1. `client.html`の明細テーブルヘッダー（個別・一括両方）を`currentClient.columns`の`name`から取得して表示
2. Excel出力のヘッダー行も同様に対応
3. PDF出力（index.js）のテーブルヘッダーもclient.htmlから渡されたラベルを使用
4. `columns`未設定時のフォールバック処理（デフォルト：作業日・サービス名・数量・単価・金額・備考）

### docType切り替えUIの削除（請求書専用化）
- 未着手。フェーズ1の一環として今後対応

### PDFの横向き（landscape）対応
- 未着手。バックログ

### 明細テーブル下の自由記入欄追加
- 未着手。バックログ

### 見積書システム（Phase 2・別リポジトリ）
- 請求書システム（このリポジトリ）の完成後に着手

### メール送信機能（要検討・保留）
- 優先度は低い

---

## 開発ルール

1. 一度に大きく進めず、1ステップずつ確認しながら進める
2. コードは**差分方式**で提示する（GitHub WebUIでの貼り替え作業がしやすいよう、貼り替え位置を明示する）
3. 専門用語は都度かみ砕いて説明する
4. 迷ったときはIdea Loomの構成を参考にする
5. 各ステップ完了後にスクリーンショットで確認する
6. GitHubのファイル取得はcurlコマンドで行う（web_fetchは制限あり）
   例：`curl -s https://raw.githubusercontent.com/kireit0313-cloud/invoice-builder/main/client.html`
7. **新しいセッションの開始時はinvoice-builder-CONTEXT.mdを読み込んでから作業を始める**
8. **client.html側でCloud Runに新しいデータを渡す変更をした場合、必ずindex.js側の受け取り処理も同時に確認・修正すること**
9. GitHub WebUIでの編集中に「commit失敗（他のコミットが割り込んだ）」エラーが出た場合は、一度ファイルを開き直して最新版に対して差分を当て直す
10. **新しい変数をconstで宣言する際、分割代入等で既に使われている変数名と重複していないか確認する**（本日、`honorific`の重複宣言でSyntaxErrorが発生した教訓）
11. **大きな設計変更（データ構造に影響するもの）を実装する前に、スコープと選択肢を整理してユーザーに確認してから着手する**（本日、可変列システムの要否を確認した上で、固定列＋ラベルカスタマイズという現実的な方針に着地した経緯を踏まえる）
