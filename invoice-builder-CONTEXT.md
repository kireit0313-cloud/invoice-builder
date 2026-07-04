# 見積書・請求書メーカー 開発引継ぎドキュメント

**最終更新：2026年7月4日（ステップ14-1完了：発行日の自由記入機能）**

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

## 【重要】製品ライン分割の方針（2026年7月4日決定）

現在の`invoice-builder`は**請求書専用システムとして完成させる**方針に確定。
見積書は**別システム・別リポジトリ**として今後新規開発する（Phase 2）。

coconala等での出品も、**請求書用・見積書用を別商品として出品する**方向で検討中。

### 請求書システム（このリポジトリ・invoice-builder）
- 取引先は固定・少数（10〜30件程度）
- 毎月同じ取引先に請求 → Firestoreでの前回データ自動引き継ぎが有効
- 取引先ごとに締め日を設定し、発行日を自動計算（新機能・本日実装）
- **現状のFirestore設計のまま継続**

### 見積書システム（将来・別リポジトリ：estimate-builder（仮称））
- 顧客は都度発生・数百件も想定（個人向け商売の場合）
- **Firestoreに顧客マスタを保存しない設計**とする
- リピート顧客への対応は「過去のExcelデータをアップロード→インポート→編集→PDF出力」の方式
- 会社情報（発行者側）のみFirestoreに保持し、顧客データは保存しない
- 着手時期はフェーズ1（請求書システムの完成）を優先し、その後着手

### docType切り替えについて
- 請求書/見積書切り替えUIは**シンプルに切り分ける方針で確定**
- 今後、invoice-builderから`docType`切り替え機能を削除し請求書専用にする作業が残っている（未着手）

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
  columns: [{ name, type }]
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
  issueDay: "末日" | "25" | "20" | "15" | "10" | "手入力"  ← 締め日パターン（本日追加）
  issueDate: string                      ← 直近保存された発行日（表示文字列、例:"2026年7月31日"）（本日追加）
  lineItems: [...]                       ← 前回PDF保存時の明細（次月インポート元）
  freeFields: [{ label, value }]         ← 取引先ごとの自由記入欄（旧projectFields相当）
  createdAt: timestamp
  updatedAt: timestamp
```

### Firestoreセキュリティルール（現在適用中）

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

## 明細データ仕様（確定）

### 6列フォーマット（縦持ち：1行＝1明細）

| 列名 | 形式 | 備考 |
|------|------|------|
| 作業日 | 自由記述 | 「6/2」「6/2〜6/4」どちらも可 |
| サービス名 | テキスト | |
| 数量 | 数値（任意） | 空欄可 |
| 単価 | 数値（任意） | 空欄可 |
| 金額 | 数値 | 数量×単価で自動計算。どちらか空欄なら手入力 |
| 備考 | テキスト | |

### 税率（明細行ごと）
- 0%以上の整数を自由入力（10・8・0はボタンで即入力）
- 0%：税込み・対象外扱い（消費税を追加計算しない）
- デフォルト値はFirestoreのdefaultTaxRateから取得

---

## 発行日仕様（本日実装・確定）

- 取引先ごとに**締め日パターン**を登録：「末日」「25日」「20日」「15日」「10日」「手入力」
- PDF出力時に締め日パターンから当月の発行日を自動計算（例：末日→2026年7月31日）
- 個別ページで発行日を確認・「変更」ボタンで手入力上書き可能
- 「内容保存」ボタンで発行日・自由記入欄・明細をまとめてFirestoreに保存
- 一括編集モードでは発行日は**表示のみ**（編集不可）。個別ページで保存した`issueDate`をそのまま使用
- 一括編集モードに「全件内容保存」ボタンを追加（各取引先の発行日・明細をまとめて保存）

### 設計判断の理由
- 請求書は取引先ごとに締め日（月末・25日など）が異なるケースが多いため、取引先登録時に締め日を設定する方式にした
- 一括編集モードでは発行日・自由記入欄の編集を行わない方針とした（個別ページで管理する運用に統一し、シンプルさを優先）

---

## PDF仕様

### レイアウト
- タイトル：docTypeに応じて「請求書」or「見積書」
- 左上：取引先名（御中）＋発行日（issueDateを反映・本日更新）
- 右上：会社情報（社名・住所・TEL・インボイス番号・customFields）＋印鑑欄
- 中央：明細テーブル（6列）
- 右下：税率ごとの内訳＋合計
- 最下部：振込先（請求書のみ表示）
- 左下：自由記入欄（freeFields）※取引先ごとに個別

### 発行日について（解消済み）
- 従来は自動で作成日（今日の日付）が入るのみだったが、**本日、取引先ごとの締め日ベースで自由記入可能に変更**
- ファイル名のContent-Dispositionヘッダーに発行日（日本語文字列）を使う際は`encodeURIComponent`が必須（本日のバグ修正）

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

### 本日のindex.js修正内容（バグ修正）
1. `app.post('/generate-pdf', ...)`内の分割代入に`issueDate`を追加
   ```javascript
   const { clientName, items, docType, companyInfo, projectFields, issueDate: issueDateInput } = req.body;
   ```
2. 未定義変数`body`参照エラーを修正
   ```javascript
   const issueDate = issueDateInput || todayStr();
   ```
3. Content-Dispositionヘッダーの発行日部分に`encodeURIComponent`を追加（日本語文字列がヘッダーに直接入るとエラーになるため）
   ```javascript
   res.setHeader('Content-Disposition', `attachment; filename="${encodeURIComponent(clientName)}_${encodeURIComponent(issueDate.replace(/\//g, '-'))}.pdf"`);
   ```

**教訓：client.html側の変更でCloud Runに新しいデータ（issueDateなど）を渡す場合、必ずindex.js側の受け取り処理（分割代入・ヘッダー生成など）も合わせて確認・修正すること。**

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
5. Excelテンプレートアップロード（列自動判定）
6. 列設定確認・修正
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
- 取引先の新規追加時に**締め日プルダウンを選択**（本日追加：末日／25日／20日／15日／10日／手入力）
- 取引先の削除
- チェックボックスで複数選択 → 一括編集モードで開く
- 会社情報の編集（社名以外・Firestore保存）

### 明細入力画面（個別）
- **発行日欄**（本日追加）：締め日パターンから自動計算された発行日を表示。「変更」ボタンで手入力に切替可能
- 自由記入欄（取引先ごとに項目名・内容を自由追加）
- 明細テーブル（6列＋税率）への入力
- 先月データの自動インポート（PDF保存済みの場合）
- 税率ボタン（10%・8%・0%）、その他は数値直入力
- 金額自動計算（数量×単価）or 手入力
- 税率ごとの合計内訳を自動表示
- アクションボタン（本日変更）：**内容保存（グレー）／Excel出力（緑 #1D6F42）／PDF出力（赤 #D32F2F）** の順に配置
  - 「内容保存」：発行日・自由記入欄・明細をFirestoreに保存（ページ遷移しても消えないように）
  - 「Excel出力」：SheetJSでダウンロード
  - 「PDF出力」：Cloud Run + Puppeteer → PDF生成・ダウンロード・Firestoreへ自動保存

### 一括編集モード（発行日・自由記入欄は編集不可の方針に確定）
- チェックした取引先の明細を取引先ごとのカードで縦に並べて表示
- 各カードは折りたたみ可能（▲▼）
- 各カード上部に発行日を**表示のみ**（取引先の登録済みissueDateまたは締め日からの自動計算値）
- 先月データを自動インポートした状態で開く
- 行追加・金額修正・税率変更など明細のみ個別に編集可能
- 「全件内容保存」ボタン（本日追加）：チェックした全取引先の明細をまとめてFirestoreに保存
- 「一括出力（PDF＋Excel）ZIP」で一括生成＋Firestoreへ各取引先の明細を保存
- 明細行数に上限なし

### 重要実装メモ
- onclick属性に取引先名を直接埋め込むとSyntaxErrorになる
  → data-pid属性にIDのみ格納し、el.dataset.pidで取得する方式を採用
- 一括編集モードのテーブルIDは `betbl_{partnerId}`、合計エリアIDは `betot_{partnerId}` の規則
- `bulkGetLineItems()`が返すオブジェクトのキーは`date`（`workDate`ではない）。個別ページの`getLineItems()`とキー名の対応関係に注意（個別は`workDate`、一括は`date`で統一されていないため、今後リファクタ時は要注意）

---

## 完了済みステップ

- [x] ステップ1〜11：基盤構築（認証・Firestore・Excel読込・PDF生成・Cloud Run）
- [x] ステップ12：PDFデザインのカスタマイズ（全項目完了）
- [x] ステップ13-1：セキュリティ強化
- [x] ステップ13-2：取引先管理機能（client.html全面リニューアル）
- [x] ステップ13-3：一括編集モード
- [x] ステップ13-4：一括編集モードにExcel同梱
- [x] **ステップ14-1（本日完了）：発行日の自由記入機能**
  - 取引先登録時に締め日パターン（末日／25日／20日／15日／10日／手入力）を選択
  - 個別ページで発行日の確認・手入力変更・保存
  - 「内容保存」ボタンを個別・一括両方に追加（発行日・自由記入欄・明細のFirestore保存）
  - 一括ページの発行日は表示のみに変更（編集は個別ページに統一）
  - アクションボタンの並び順・配色を変更（内容保存／Excel出力（緑）／PDF出力（赤））
  - index.js側のバグ修正（issueDate未受信、body未定義エラー、Content-Dispositionヘッダーのencode漏れ）

---

## 次のステップ候補

### 【最優先】PDFの明細欄等フォーマットの修正（次回セッションで着手）
- 具体的な修正点は次回セッション開始時にヒアリングして着手する

### docType切り替えUIの削除（請求書専用化）
- index.html・client.html双方から見積書/請求書切り替えの概念を削除し、請求書専用に簡素化する
- 未着手。フェーズ1の一環として今後対応

### 見積書システム（Phase 2・別リポジトリ）
- 過去のExcelデータをインポートして見積書作成できるようにする方針
- Firestoreに顧客を蓄積しない設計（都度Excelインポート）
- 請求書システム（このリポジトリ）の完成後に着手

### メール送信機能（要検討・保留）
- SendGridを使いPDFを添付してメール送信する機能
- **差出人はTsuyoshi所有アドレスになる**（クライアント自身のメールアドレスからは送れない）という制約から、現状のPDF出力→メールソフトに添付フローで十分な可能性あり
- 優先度は低い

---

## 開発ルール

1. 一度に大きく進めず、1ステップずつ確認しながら進める
2. コードは**差分方式**で提示する
   - GitHub WebUIでの貼り替え作業がしやすいよう、貼り替え位置（前後のコード・行数）を明示する
3. 専門用語は都度かみ砕いて説明する
4. 迷ったときはIdea Loomの構成を参考にする
5. 各ステップ完了後にスクリーンショットで確認する
6. GitHubのファイル取得はcurlコマンドで行う（web_fetchは制限あり）
   例：`curl -s https://raw.githubusercontent.com/kireit0313-cloud/invoice-builder/main/client.html`
7. **新しいセッションの開始時はinvoice-builder-CONTEXT.mdを読み込んでから作業を始める**
8. **client.html側でCloud Runに新しいデータを渡す変更をした場合、必ずindex.js側の受け取り処理も同時に確認・修正すること**（本日の教訓：issueDate追加時にindex.js未修正でエラーが多発した）
9. GitHub WebUIでの編集中に「commit失敗（他のコミットが割り込んだ）」エラーが出た場合は、一度ファイルを開き直して最新版に対して差分を当て直す。複数の差分をまとめて適用する場合は、時間を空けずに一気にコミットする
