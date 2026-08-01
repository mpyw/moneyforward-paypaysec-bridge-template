# セットアップ手順

あなたが手で順番にやる作業のチェックリスト。ワークフローは設定済みなので、
下記を消化すれば自動同期が動き始める。

## 前提

**PayPay 証券と MoneyForward の登録メールアドレスが、同じ Gmail の受信箱に
届くこと。** このツールは OTP を Gmail API で読む。

転送でも受信自体はできるが、OTP メールは**送信者アドレスで絞っている**ので、
転送で `From:` が書き換わる経路では動かない。別アドレスで登録している場合は、
先に各サービス側で登録アドレスを変更する。

Google Workspace ではなく個人 Gmail を想定している。サービスアカウントでは
個人の受信箱を読めないため (ドメイン全体の委任は Workspace 専用)。

> **推奨: OTP 専用の Google アカウントを作る。**
> `GMAIL_CREDENTIALS_JSON` は失効しない受信箱の読み取り鍵になる。専用アカウントに
> しておけば、それが漏れたときに読まれる範囲が OTP メールだけになる。無料。
> やるなら、両サービスの登録アドレス変更を先に済ませること。

---

## 進捗チェックリスト

- [ ] **A.** MoneyForward に手入力口座を 1 つ作成 + account_id_hash をメモ
- [ ] **B.** Google Cloud で Gmail API を有効化し、OAuth クライアント (デスクトップ) を作成
- [ ] **C.** `go run …/cmd/mfpp@v1 debug gmail authorize` で資格情報を発行
- [ ] **D.** `gh secret set` で 6 つの secret を登録

A・B はブラウザ、C・D はターミナル。

---

## A. MoneyForward 手入力資産の作成

PayPay 証券は MF 上では **1 つの手入力資産** にまとめる（Web 残高 + ミニアプリ残高を合算した値を書き込む）。

1. https://moneyforward.com/ にログイン
2. 「資産」→「手入力資産を追加」→ 1 つ作成:
   - 名前例: `PayPay 証券`
   - カテゴリ: 株式 (または現金・預金など、お好みで)
   - 評価額: **何でもいいので 1 件入力** (例: 現在の実残高 or 仮に 1 円)
     - **重要**: 0 件の状態だと show_manual ページが alert で挫かれて自動更新が走らない
     - 任意の初期値を 1 件登録すれば、以降は workflow が rollover で上書き更新する
3. **資産 ID を確認**: 作成した資産の詳細ページ URL の末尾セグメント
   - URL 例: `https://moneyforward.com/accounts/show_manual/<ASSET_ID>`
   - その `<ASSET_ID>` をメモ（D で `MF_ASSET_ID` に入れる）

---

## B. Gmail API の OAuth クライアント作成

OTP メールは Gmail API から直接読む。個人 Gmail はサービスアカウントでは読めない
(ドメイン全体の委任は Workspace 専用) ので、ユーザーの refresh token が要る。

1. Google Cloud プロジェクトで **Gmail API を有効化**
2. **OAuth 同意画面**を設定
   - User type: External (個人 Gmail なので Internal は選べない)
   - アプリ名・サポートメール・デベロッパー連絡先
   - スコープに `https://www.googleapis.com/auth/gmail.readonly` を追加
   - **公開ステータスを「本番環境」にする**
     - ⚠ 「テスト」のままだと **refresh token が 7 日で失効**し、cron が 1 週間後に
       静かに死ぬ。制限付きスコープなので未確認アプリの警告は出るが、自分の
       アカウントで使う分には通せる
3. **認証情報 → OAuth クライアント ID → デスクトップ** を作成し、JSON を
   `client_secret.json` として作業ディレクトリに置く (gitignore 済み)

## C. Gmail 資格情報の発行

このリポジトリに Go のコードは無い。アクション側をクローンせず直接叩く。

```bash
# client_secret.json を置いたディレクトリで
go run github.com/mpyw/moneyforward-paypaysec-bridge-action/cmd/mfpp@v1 \
  debug gmail authorize
```

ブラウザが開いて同意を求められ、`gmail-credentials.json` ができる。

`gcloud auth application-default login` は使わない。gcloud は ADC 発行時に
`cloud-platform` スコープを強制するので、CI に置く資格情報が漏れたときの影響が
「メール読み取り」から「Google Cloud プロジェクト全体の操作」に跳ね上がる。

確認:

```bash
go run github.com/mpyw/moneyforward-paypaysec-bridge-action/cmd/mfpp@v1 \
  debug gmail check     # どのメールボックスが開くか
```

## D. secrets を登録

`gh secret set` は値をローカルで sealed box 暗号化してから送るので、平文が
GitHub に渡ることはない。引数に値を書かず、対話入力か stdin を使う（コマンド
履歴と `ps` に残さないため）。

| キー | 入力値 |
|---|---|
| `PAYPAY_SEC_USERNAME` | PayPay 証券のログイン ID |
| `PAYPAY_SEC_PASSWORD` | PayPay 証券パスワード |
| `MF_EMAIL` | MoneyForward ログインメール |
| `MF_PASSWORD` | MF パスワード |
| `MF_ASSET_ID` | A でメモした手入力口座の account_id_hash |
| `GMAIL_CREDENTIALS_JSON` | C で作った `gmail-credentials.json` の中身 |

```bash
for name in PAYPAY_SEC_USERNAME PAYPAY_SEC_PASSWORD MF_EMAIL MF_PASSWORD MF_ASSET_ID; do
  gh secret set "$name"     # 値の入力を求められる
done
gh secret set GMAIL_CREDENTIALS_JSON < gmail-credentials.json
```

確認（名前と更新日時だけが出る。値は GitHub からも読み出せない）:

```bash
gh secret list
```

このリポジトリを action として他リポジトリから使う場合は、そちらの
Settings → Secrets and variables → Actions で同じ 6 つを設定する。

---

## 全部終わったら

- 平日 JST 15:30 になると `sync.yml` の cron が動いて初回同期が実行される
- 即時テストしたければ:
  ```bash
  gh workflow run sync.yml
  ```
- Actions タブで実行ログ確認:
  ```bash
  gh run list --workflow sync.yml --limit 5
  gh run view <run-id> --log
  ```

自分の fork の中で実行すること。`--repo` は要らない。

### 1 日に何度も叩かないこと

両サービスとも、短時間にログインを繰り返すと **OTP メールの送信自体を止める**。
2026-08-01 に 5 回試行して両方止まった (同日中に復活)。失敗しても連打しない。

## 失敗したときの読み方

workflow が異常終了すると GitHub から通知メールが来る。ログの読み方:

| メッセージ | 意味 | すること |
|---|---|---|
| `login failed at fetch-otp` | OTP メールが 5 分以内に来なかった | 叩きすぎ。翌日まで待つ |
| `login failed at await-challenge` + `(the browser was on …)` | 認証情報を拒否された、または想定外のページ | URL を見る。`/login/` に留まっていればパスワード |
| `the page was still loading after 20s` | 非同期ロードが終わらなかった | 一時的なら再実行。続くならサイト側の変更 |
| `評価額合計 is N but …` | 3 ルートが食い違った | セレクタがずれた。`mfpp debug paypaysec probe --url …` |
| `the total is N but nothing was listed under it` | 合計はあるのに銘柄行が無い | 保有銘柄セクションの描画失敗 |
| `deletes too much of the ledger` | 削除が多すぎる | **口座は無傷**。ログの `section=` と 銘柄数 を見て、読めなかったページを特定する |
| `reported no error but …` | 書き込んだのに反映されていない | 併記される「the service said」を読む |
| `more than one entry named` | MF 側に同名行がある | 手で片方を消す |

**どれも口座を壊す前に止まる。** 書き込みは 1 件ごとに読み戻して検証し、削除は
台帳の 1/3 を超えると拒否する。「もっともらしい間違った数字を黙って記録する」
経路は、2026-08-01 の敵対的レビューで見つかった 7 件を含めて塞いである。

