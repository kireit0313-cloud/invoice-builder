# invoice-builder 本番リリース ロードマップ／TODO

> 作成：2026/7/11。進捗の正は本ファイル、機能仕様の正は `invoice-builder-CONTEXT.md`。
> 完了項目は `[x]` にし、必要なら日付・メモを添える。

---

## フェーズ0：残作業の反映【完了】

- [x] ㉒㉓をGitHubへUpload（`index.html`・`client.html`＝Vercel自動反映、`invoice-pdf-server/index.js`＝GitHub Actions自動デプロイ）
- [x] 反映後、明細列を数種OFFにしたパターンで個別PDF・一括ZIPの見た目を実機確認

---

## フェーズ1：機能E2Eテスト（品質担保）【2026/7/12 実施・ほぼ完了】

> 実機E2Eを `フェーズ1-機能E2Eテスト-チェックリスト.md` で実施。テスト1〜7合格、テスト8は前回確認済み（CONTEXT⑱）。
> 途中で3件修正：①バックアップ/PDFファイル名の日付が1日ずれ（`todayStr()`のUTC→ローカル化・client.html）、②個別画面の「取引先一覧」戻るボタンを取引先名の横へ寄せた（client.html・視認性改善）、③2ページ目で明細見出し下線とグレー行下線がかぶる（表を`border-collapse:separate`化・index.js）。いずれもGitHub反映・実機確認済み。

- [x] 通し操作：クライアント登録 → URL発行 → 取引先追加 → 明細入力 → 個別PDF／一括ZIP → バックアップ／復元（2026/7/12）
- [x] 明細列ON/OFF：数量・単価・備考の8組合せでPDF崩れ・幅按分を確認（2026/7/12・全8組合せ✅）
- [x] 角印：字数2〜13・アルファベット混在・長音符（ローソン等）・画像アップロード各モード（2026/7/12）
- [x] 角印 × カラーテーマ（グレー／5プリセット／任意色）の組合せ確認（2026/7/12）
- [x] 複数ページ：明細28行の確認用バックアップJSONで2ページ送り・余白・見出し反復を確認（2026/7/12・下線かぶりも修正済み）
- [x] 環境差：Chrome／Edge ✅。**Safari＝管理画面ログイン不可（下記フェーズ2の項目で対応）**。client.html（顧客ページ）はログイン不要のためSafariでも影響なし（2026/7/12）
- [x] スマホ表示・ハードリフレッシュ後の反映（2026/7/12・Android実機＋iPhoneデバイスモードで崩れなし）

---

## フェーズ2：堅牢性・セキュリティ（リスク回避）

- [ ] Firestoreルールの権限境界：他クライアントのclientId／partnerIdに書き込めないか、未認証で読めないか
- [ ] 入力バリデーション：空欄・異常値・極端に長い文字列・特殊文字（`onclick`のdata-*化を再点検）
- [ ] 例外処理：Cloud RunのPDF生成失敗・ネットワーク断時にトーストで止まるか（内部エラーを生で見せない）
- [ ] 認証：Auth承認済みドメイン、管理画面がTsuyoshi以外からアクセスできないこと
- [ ] **【要対応・2026/7/12発見】Safariで管理画面ログイン不可（`Unable to process request due to missing initial state`）**。原因＝認証ドメイン（`invoice-builder-4b77e.firebaseapp.com`）がアプリ（`invoice-builder-iota-lac.vercel.app`）と別ドメインで、Safariのストレージ分離（ITP）で認証の初期状態が読めない。方式をリダイレクトに変えるだけでは不可。対応＝Vercelで`/__/auth/*`等をfirebaseapp.comへ中継するrewrite（`vercel.json`）を追加し、`authDomain`をvercelドメインに変えて同一ドメイン化（index.html・client.html両方のfirebaseConfig）。反映後は全ブラウザで**ログインのみ**再確認。影響は管理画面のみ・PC運用で回避可（顧客のclient.htmlはログイン不要で無影響）。
- [ ] HTTPS：Vercel／Cloud Runの自動HTTPSを確認のみ（HTTP→HTTPSリダイレクト含む）

---

## フェーズ3：運用・法務・再現性

- [ ] 依存固定の再点検：Node22／Puppeteer21.11.0／自前Chromium
- [ ] Google Fonts読込安定性（角印Yuji Syukuの `document.fonts.ready` 待ちの実効性）
- [ ] 法務：利用規約・プライバシーポリシー（顧客データを預かる立場。顧客先URLへの掲示要否を検討）
- [ ] バックアップ体制：Firestoreエクスポート方針の確立（コードはGitHubがオフサイト保管）
- [ ] ロールバック手順の明文化：Vercel＝過去Deploymentへ即戻し、Cloud Run＝Actions再実行／前リビジョン切替

---

## フェーズ4：リリース当日

- [ ] スモークテスト：本番URLで代表クライアント1件を通しで疎通確認
- [ ] 初回納品フローの確認：テンプレ設定〜URL発行〜顧客操作の想定時間（15〜30分/件）を実測

---

## 非該当・簡略化（このプロジェクトでは不要／優先度低）

- 決済フロー：なし
- SMTP／メール認証（SPF/DKIM/DMARC）：SendGridは保留中のため対象外
- メンテナンス画面：管理者1人・顧客は自分のURLのみで影響小
- PageSpeed／アセット圧縮：静的HTMLで軽量、優先度低
- SQLインジェクション：Firestoreのため非該当
