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
- Cloud Runサービス：`invoice-pdf-server`（リポジトリ：`kireit0313-cloud/invoice-pdf-server`、**GitHubにアップロード済み・Claudeが直接curlで読める**）
- Cloud Runローカルパス：`C:\Users\fokar\invoice-pdf-server\`
- Google アカウント/プロジェクト：kireit0313@gmail.com / idea-notebook-172e0
- 参考プロジェクト：Idea Loom（`kireit0313-cloud/idea-notebook`）— 同じスタックの参照実装

将来的に「estimate-builder」（見積書作成ツール）を別リポジトリとして計画中
（Excelインポート/エクスポート型、Firestore顧客保存なし、過去Excel再インポート対応）。

---

## 現在の状態（本セッション終了時点）

### ① client.html モバイル表示バグ修正【本セッション完了】

- 取引先追加エリア（`.add-partner-area`）が `display:flex` で折り返し設定がなく、
  スマホ幅で入力欄＋プルダウン2つ＋ボタンがはみ出す不具合を修正
- `.add-partner-area` に `flex-wrap: wrap` を追加
- `.add-partner-input` を `flex: 1 1 100%` にし、入力欄が常に1行目単独で配置されるよう変更
- スマホでは入力欄が1行目、プルダウン・ボタンが2行目以降に折り返される表示に

### ② Cloud Run `index.js`：remarksLower（自由記入欄・下段）のPDF表示対応【本セッション完了】

- `remarksLower` をリクエストボディの分割代入に追加
- `remarksLowerBlockHtml` を生成するロジックを追加（未記入なら非表示、改行は`<br>`に変換）
- `.remarks-lower` のCSSスタイルを追加（`.project-info` と同系統のデザイン）
- 明細テーブル（`</table>`）の直後・合計欄（`<div class="totals">`）の直前に表示位置を設定
- 動作確認済み：記入時は明細下に表示、空欄時は非表示

### ③ Cloud Run PDF生成の重大インシデントと対応【本セッション完了】

**症状：** 上記②のデプロイ後、個別・一括PDF出力が「Failed to launch the browser process!」で完全に失敗するようになった。

**切り分けの過程：**
- `index.js` を完全に修正前の状態に戻しても同じエラーが再現 → コード変更が原因ではないと判明
- Puppeteer起動オプションへの `--disable-dev-shm-usage` `--disable-gpu` 追加 → 効果なし
- `headless: 'new'` → `headless: true` → 効果なし
- Dockerfileのベースイメージを `node:20-bookworm-slim` に変更 → 効果なし
- `--execution-environment gen2` でのデプロイ → 効果なし
- `--no-zygote` `--single-process` 等の追加 → 効果なし

**根本原因：** Dockerfileが `apt-get install chromium` でOS(Debian)標準のChromiumパッケージを使う構成になっており、Puppeteer側は「PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true」でこれを流用する設定だった。OS標準Chromiumはバージョンが固定されておらず、`apt-get update` のたびに最新版に変わりうる。あるタイミングでOS標準Chromiumのバージョンが上がり、Cloud Runのサンドボックス実行環境と非互換になって起動不能になった。表示されていた`inotify`・`dbus`・`NETLINK socket`関連のエラーは起動失敗に付随する副次的な警告であり、直接の原因ではなかった。

**最終的な修正：**
- Dockerfileから `PUPPETEER_SKIP_CHROMIUM_DOWNLOAD` `PUPPETEER_EXECUTABLE_PATH` の指定を削除
- 代わりにPuppeteer自身に、そのバージョンに対して動作確認済みのChromiumを`npm install`時に自動ダウンロードさせる方式に変更
- Chromium実行に必要な共有ライブラリ一式をDockerfileで明示的にインストール（`libnss3` `libgbm1` `fonts-liberation` 等、[Puppeteer公式トラブルシューティング](https://pptr.dev/troubleshooting)準拠）
- Puppeteer起動オプションに `--disable-dev-shm-usage` `--disable-gpu` `--no-zygote` `--single-process` 等を追加し、サンドボックス環境での安定性を強化
- 個別・一括PDF出力ともに復旧確認済み

### ④ 依存バージョンの固定化・Node.js移行【本セッション完了】

- **Node.js 20 → 22 へ移行**：Node.js 20は2026年4月30日にEOL（サポート終了）済みと判明したため、`Dockerfile` の `FROM node:20-slim` を `FROM node:22-slim` に変更しデプロイ・動作確認済み
- **Puppeteerのバージョン固定**：`package.json` の `"puppeteer": "^21.0.0"`（キャレット付き＝自動で新しいバージョンに変わりうる）を `"puppeteer": "21.11.0"`（正確なバージョン固定）に変更し、Cloud Buildのたびに異なるPuppeteer/Chromiumの組み合わせが入るリスクを排除

---

## 既知の残作業・次回優先事項

1. **【推奨】`package-lock.json` のGitHubへのコミット**
   - `package.json` の固定だけでは、Puppeteerの間接的な依存パッケージのバージョンにわずかなブレが残る。`package-lock.json` を必ずコミットして完全な再現性を確保する
2. **【将来】estimate-builderの新規リポジトリ作成**（Excelインポート/エクスポート型）
3. **【保留】GitHub Actions自動デプロイ（Cloud Run向け）** — 現状は手動 `gcloud run deploy`
4. **【保留・優先度低】SendGridメール送信機能**
5. **【定期見直し】バージョン管理ポリシーに沿った半年〜1年ごとの依存バージョン点検**（下記「バージョン管理ポリシー」参照）

---

## バージョン管理ポリシー（本セッションで新設）

今回、コードを一切変更していないのにOS標準Chromiumのバージョンが勝手に変わり、PDF生成が突然停止する重大インシデントが発生したことを受けて、以下の方針を定める。

| 項目 | 方針 |
|---|---|
| **Node.jsベースイメージ** | メジャーバージョンは明示的に固定（現在：`node:22-slim`）。EOLの半年前を目安に次のLTSへの移行を検討する。Node.js 22のEOLは2027年4月30日予定 |
| **Puppeteerのバージョン** | `package.json` はキャレット(`^`)を使わず正確なバージョンで固定する（現在：`21.11.0`）。`package-lock.json` も必ずGitHubにコミットする |
| **Chromium** | Puppeteer自身がダウンロードする、そのバージョンに対応した組み合わせのみを使用する。OS(apt)標準のChromiumパッケージは使わない |
| **依存バージョンの見直しタイミング** | 半年〜1年に1回、または「コードを変えていないのに急に動かなくなった」場合に真っ先に疑う項目として |
| **依存関係・ベースイメージ変更時の確認手順** | 変更後は必ず個別出力・一括出力の両方でPDF生成を確認してからGitHubにコミットする |
| **デプロイコマンド（標準形）** | 下記「ツール・リソース一覧」参照。`--execution-environment gen2` を必ず含める |

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
- **CSS flexboxの折り返し**：横並びレイアウト（`display:flex`）は `flex-wrap: wrap` を指定しない限りモバイル幅で要素がはみ出す。フォーム的な横並びUIを追加する際は必ずモバイル幅での見た目を確認する。
- **依存パッケージ・OSパッケージのバージョン固定は最優先事項**：`apt-get install` や `npm install` の未固定バージョン指定（キャレット等）は、コードを一切変更していなくても「ある日突然動かなくなる」重大インシデントの原因になりうる。特にPuppeteer/Chromiumのようにブラウザバイナリに依存する構成では、OS標準パッケージではなくツール自身が管理する動作確認済みバイナリを使うこと。
- **「Failed to launch the browser process!」＋`inotify`/`dbus`/`NETLINK`エラーの読み方**：これらのエラーメッセージ自体は起動失敗時に付随する副次的な警告であることが多く、直接の原因を示しているとは限らない。真因はPuppeteerとChromiumのバージョン不整合や、共有ライブラリ不足であることが多い。

---

## 開発の進め方（合意済みルール）

- **コード提供方式（更新）**：
  - すでに会話内で全文を読み込み済みのファイルを修正する場合 → **検証済みの完成版ファイルをまるごと渡す**（ダウンロードしてそのまま上書き保存できる形）
  - まだ会話内で読み込んでいないファイルを修正する場合 → 従来通り**差分方式**（トークン節約のため）
- **ファイル編集**：GitHub Web UI上で直接、またはClaudeが渡した完成版ファイルで上書き。ローカルビルド環境なし。
- **GitHub読み込み**：`curl -s https://raw.githubusercontent.com/kireit0313-cloud/[repo]/main/[filename]`をbashツールで実行。行番号特定には`grep -n`。`web_fetch`はGitHub rawファイルには使えない。
- **`invoice-pdf-server`（index.js・Dockerfile・package.json等）はGitHubにアップロード済み**：ローカル限定ではなくなったため、Claudeが直接curlで読み込める。`findstr`でのスクリーンショット共有手順は基本的に不要になった。
- **コンテキスト継続**：`invoice-builder-CONTEXT.md`はセッション終了時に更新し、GitHub Web UIで手動コミット。`developer-profile-updated.md`は正しいスタック定義（プレーンHTML、React/Viteなし）を記載。
- **ターミナルコマンド**：一つずつ実行（まとめて貼らない）。
- **トラブル時の切り分け方針**：コード変更後にエラーが出た場合、まず「コード変更前の状態に戻しても同じエラーが再現するか」を確認し、コード起因かインフラ・環境起因かを切り分ける。
- **セッション開始の合言葉**：「invoice-builder-CONTEXT.mdとdeveloper-profile-updated.mdをGitHubから読み込んで」

---

## ツール・リソース一覧

- ホスティング：Vercel（GitHub main branchから自動デプロイ）
- データベース：Firebase Firestore
- 認証：Firebase Auth（Google login、Tsuyoshiさん専用）
- PDF生成：Google Cloud Run + Puppeteer（`invoice-pdf-server`、Node.js 22 / Puppeteer 21.11.0固定・Puppeteer自前Chromium方式）
- **デプロイコマンド（標準形・更新）**：
  ```
  gcloud run deploy invoice-pdf-server --source . --region asia-northeast1 --allow-unauthenticated --memory 1Gi --execution-environment gen2
  ```
  （`C:\Users\fokar\invoice-pdf-server\`から実行）
- Excel処理：SheetJS（CDN）
- ZIP生成：JSZip（CDN）
- 参考プロジェクト：Idea Loom（`kireit0313-cloud/idea-notebook`）
