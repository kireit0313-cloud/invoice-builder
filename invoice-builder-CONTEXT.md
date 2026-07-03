# 見積書・請求書メーカー 開発引継ぎドキュメント

**最終更新：2026年7月3日（ステップ13-4完了）**

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

### Firestore設計（確定）

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

clients/{clientId}/partners/{partnerId}  ← 取引先サブコレクション（ステップ13-2で新設）
  partnerName: string                    ← 取引先名
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

## PDF仕様

### レイアウト
- タイトル：docTypeに応じて「請求書」or「見積書」
- 左上：取引先名（御中）＋発行日
- 右上：会社情報（社名・住所・TEL・インボイス番号・customFields）＋印鑑欄
- 中央：明細テーブル（6列）
- 右下：税率ごとの内訳＋合計
- 最下部：振込先（請求書のみ表示）
- 左下：自由記入欄（freeFields）※取引先ごとに個別

### 発行日について（課題・後回し）
- 現状：自動で作成日（今日の日付）が入る
- 見積書はこれでよい
- 請求書は月末・25日締めなど固定日が多く、自由記入が望ましい
- **将来課題として保留**

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
        ├→ 取引先を1件タップ → 明細入力画面（先月データがあれば自動インポート）
        │     └→ PDF出力・保存 → Firestoreに自動保存（次月の先月データになる）
        └→ チェックボックスで複数選択 → 「一括編集モードで開く」
              └→ 一括編集画面（取引先ごとにカード表示・明細編集可）
                    └→ 一括出力（PDF＋Excel）ZIP → Firestoreに各取引先の明細を自動保存
```

### 取引先一覧画面
- 取引先の一覧表示（最終更新日付き）
- 取引先の新規追加・削除
- チェックボックスで複数選択 → 一括編集モードで開く
- 会社情報の編集（社名以外・Firestore保存）

### 明細入力画面（個別）
- 自由記入欄（取引先ごとに項目名・内容を自由追加）
- 明細テーブル（6列＋税率）への入力
- 先月データの自動インポート（PDF保存済みの場合）
- 税率ボタン（10%・8%・0%）、その他は数値直入力
- 金額自動計算（数量×単価）or 手入力
- 税率ごとの合計内訳を自動表示
- Excelダウンロード（SheetJS）
- PDF出力（Cloud Run + Puppeteer）→ Firestoreへ自動保存

### 一括編集モード（ステップ13-3で追加）
- チェックした取引先の明細を取引先ごとのカードで縦に並べて表示
- 各カードは折りたたみ可能（▲▼）
- 先月データを自動インポートした状態で開く
- 行追加・金額修正・税率変更など個別に編集可能
- 「一括出力（PDF＋Excel）ZIP」で一括生成＋Firestoreへ各取引先の明細を保存（ステップ13-4で更新）
- 明細行数に上限なし

### 重要実装メモ
- onclick属性に取引先名を直接埋め込むとSyntaxErrorになる
  → data-pid属性にIDのみ格納し、el.dataset.pidで取得する方式を採用
- 一括編集モードのテーブルIDは `betbl_{partnerId}`、合計エリアIDは `betot_{partnerId}` の規則

---

## 完了済みステップ

- [x] ステップ1〜11：基盤構築（認証・Firestore・Excel読込・PDF生成・Cloud Run）
- [x] ステップ12：PDFデザインのカスタマイズ（全項目完了）
- [x] ステップ13-1：セキュリティ強化
  - clientIdをcrypto.randomUUID()でランダム化（推測・使い回し対策）
  - Firestoreルール：get（公開）/ list（認証必須）/ update（クライアント側は社名等変更不可）
  - partners サブコレクション設計確定
- [x] ステップ13-2：取引先管理機能（client.html全面リニューアル）
  - 取引先一覧・追加・削除
  - 明細入力画面（先月データ自動インポート）
  - 自由記入欄（取引先ごと）
  - PDF出力→Firestoreへ自動保存
  - ZIP一括PDF出力（JSZip）
  - 会社情報編集（社名以外・クライアント側から）
- [x] ステップ13-3：一括編集モード
  - チェックした取引先の明細を一覧表示（取引先ごとのカード・折りたたみ可）
  - 各カードで明細を直接編集可能
  - まとめてPDF出力（ZIP）＋Firestoreへ自動保存
  - 「そのままZIP出力」ボタンは削除（一括編集モード経由で代替）
- [x] ステップ13-4：一括編集モードにExcel同梱
  - 「全PDF出力（ZIP）」→「一括出力（PDF＋Excel）ZIP」にボタン名変更
  - ZIP内にPDFに加えてExcel（明細フォーマット）も同梱
  - 出力中はボタンテキストを「出力中...」に変更し、完了後に元に戻す

---

## 次のステップ候補

### メール送信機能（要検討）
- SendGridを使いPDFを添付してメール送信する機能
- Cloud Run（invoice-pdf-server）に `/send-email` エンドポイントを追加する方式
- **差出人はTsuyoshi所有アドレスになる**（クライアント自身のメールアドレスからは送れない）
- クライアントが自分のメールで取引先に送るユースケースには不向き
  → 現状のPDF出力→メールソフトに添付フローで十分な可能性あり
- オプション機能（UIにボタン追加）として実装する方向で検討中
- **次回セッション開始時に方針を決定してから着手**

### 発行日の自由記入（保留）
- 請求書：月末・25日締めなど固定日が多い → 自由記入フィールドが望ましい
- 見積書：作成日（今日の日付）でよい
- 現状は自動で今日の日付が入っており、暫定的に許容

### Excelインポート（見送り）
- 運用開始後は取引先がFirestoreに蓄積されるため出番がほぼない
- 初期セットアップ時の取引先登録はclient.htmlの「＋追加」ボタンで手動登録で対応
- **現時点では実装しない**

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
