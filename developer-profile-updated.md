# 開発者プロフィール（2026年7月3日改訂版）

> このプロフィールは開発着手前に作成した構想メモです。
> 開発の進捗によって内容が変わってきますので、内容の詳細（機能仕様・進捗）は `invoice-builder-CONTEXT.md` を参照してください。
> 開発は `invoice-builder-CONTEXT.md`の内容を正として進めて行きます。

---

## 開発体制

- 非エンジニアのTsuyoshiが、Claudeと協働でステップバイステップ実装する
- ファイル操作はGitHubのWebUI経由（ローカルにリポジトリをクローンしない）
- 各ステップ完了後、スクリーンショットで動作確認してから次に進む

## 既存インフラ

- ホスティング：Vercel（GitHubのmainブランチにpushで自動反映）
- DB・認証：Firebase（Firestore／Auth）
- バックエンド：Cloud Run（PDF生成サーバー、ローカルPCから`gcloud`で個別デプロイ）
- 参照プロジェクト：Idea Loom（`kireit0313-cloud/idea-notebook`）── 同じ技術スタックで稼働済み

## 技術スタック（確定・変更なし）

| レイヤー | 採用技術 |
|---|---|
| フロントエンド | **素のHTML + JavaScript**（ビルドツールなし） |
| ホスティング | Vercel |
| バックエンド | Cloud Run |
| DB | Firebase Firestore |
| 認証 | Firebase Auth |
| Excel処理 | SheetJS（ブラウザ内処理） |
| PDF生成 | Puppeteer（Cloud Run） |
| メール送信 | SendGrid（月100通無料枠・将来実装） |

> React + Viteでの構築は構想段階で検討したが、GitHub WebUIでの直接編集という
> 運用に合わせて**素のHTML方式へ意図的に転換済み**。以後この方針を維持する。

## このプロジェクトの目的

自営業・中小企業向けの見積書・請求書作成PWA。coconala等のクラウドソーシングでの受注販売を想定した納品モデル。

### 納品フロー

1. Tsuyoshiが顧客のExcelテンプレートを読み込み、設定作業を行う（15〜30分/件）
2. 顧客専用URLを発行して納品
3. 顧客はURLにアクセスして入力・PDF出力するだけ
4. 顧客側のランニングコストなし

## 開発ルール（更新）

1. 一度に大きく進めず、1ステップずつ確認しながら進める
2. コードは差分方式で行う
   → GitHub WebUIでの貼り替え作業がしやすいように張替の位置を示す。行数が分かればなおよい。
3. 専門用語は都度かみ砕いて説明する
4. 迷ったときはIdea Loomの構成を参考にする
5. 各ステップ完了後にスクリーンショットで確認する
6. GitHubのファイル取得は`curl`コマンドで行う（`web_fetch`は制限あり）
   例：`curl -s https://raw.githubusercontent.com/kireit0313-cloud/invoice-builder/main/client.html`
7. **新しいセッションの開始時は`invoice-builder-CONTEXT.md`を読み込んでから作業を始める**

## 機能要件・開発フェーズ・競合ポジションについて

開発内容は実装の進行とともに頻繁に更新されるため、このプロフィールでは扱わず
**`invoice-builder-CONTEXT.md`に一元化**します。プロフィール文書とCONTEXT.mdの
二重管理はしません。