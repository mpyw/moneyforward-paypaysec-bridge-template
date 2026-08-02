# セットアップ手順

あなたが手で順番にやる作業のチェックリスト。ワークフローは設定済みなので、
下記を消化すれば自動同期が動き始める。

## 用意するもの

| | 何に使うか |
|---|---|
| GitHub アカウント | ワークフローを動かす。無料枠で足りる |
| [GitHub CLI (`gh`)](https://cli.github.com/) | リポジトリ作成と Secret 登録 |
| [Go](https://go.dev/dl/) | Gmail の資格情報を発行する C の 1 コマンドだけ |
| `git` | フォークを作る |
| ブラウザ | Google の同意画面。**手元で開ける必要がある**（SSH 越しでは完結しない） |

```bash
# macOS (Homebrew)
brew install gh go

# 認証（ブラウザが開く）
gh auth login
```

Go を使うのは **C の 1 回だけ**。以降の同期は GitHub のランナー側で動くので、
手元に Go を残しておく必要はない。バージョンは 1.21 以降なら、必要なものを
自動で取ってくる。

---

## 前提

**PayPay 証券と MoneyForward の登録メールアドレスが、同じ Gmail の受信箱に
届くこと。** このツールは OTP を Gmail API で読む。

転送でも受信自体はできるが、OTP メールは**送信者アドレスで絞っている**ので、
転送で `From:` が書き換わる経路では動かない。別アドレスで登録している場合は、
先に各サービス側で登録アドレスを変更する。

Google Workspace ではなく個人 Gmail を想定している。サービスアカウントでは
個人の受信箱を読めないため (ドメイン全体の委任は Workspace 専用)。

> [!TIP]
> **OTP 専用の Google アカウントを作ることを勧める。**
> `GMAIL_CREDENTIALS` は失効しない受信箱の読み取り鍵になる。専用アカウントに
> しておけば、それが漏れたときに読まれる範囲が OTP メールだけになる。無料。
> やるなら、両サービスの登録アドレス変更を先に済ませること。

---

## 進捗チェックリスト

- [ ] **A.** MoneyForward に手入力口座を 1 つ作成 + ASSET_ID をメモ
- [ ] **B.** Google Cloud で Gmail API を有効化し、OAuth クライアント（デスクトップ）を作成
- [ ] **C.** `go run …/cmd/mfpp@v3 gmail authorize` で資格情報を発行
- [ ] **D.** `gh secret set` で 6 つの secret を登録

A・B はブラウザ、C・D はターミナル。

---

## A. MoneyForward 手入力資産の作成

PayPay 証券は MF 上では **1 つの手入力資産** にまとめる（Web 残高 + ミニアプリ残高を合算した値を書き込む）。

1. https://moneyforward.com/ にログイン
2. 「資産」→「手入力資産を追加」→ 1 つ作成:
   - 名前例: `PayPay 証券`
   - カテゴリ: 株式 (または現金・預金など、お好みで)
   - **中身は空のままでよい。** 初回実行が銘柄を作る
3. **資産 ID を確認**: 作成した資産の詳細ページ URL の末尾セグメント
   - URL 例: `https://moneyforward.com/accounts/show_manual/<ASSET_ID>`
   - その `<ASSET_ID>` をメモ（D で `MONEYFORWARD_ASSET_ID` に入れる）

> [!WARNING]
> **既存の資産を指定しないこと。** この資産の中身はジョブが管理し、保有銘柄に
> 対応しない行は削除される。新しく 1 つ作って、それを渡す。

---

## B. Google Cloud で Gmail API の OAuth クライアントを作る

OTP メールは Gmail API から直接読む。**個人 Gmail はサービスアカウントでは読めない**
（ドメイン全体の委任は Google Workspace 専用）ので、あなた自身の同意で発行した
refresh token が要る。無料。課金の有効化も不要。

> [!NOTE]
> Google Cloud コンソールの「OAuth 同意画面」は **Google Auth Platform**
> （Branding / Audience / Data Access / Clients）に再編された。以下はその名前で
> 書いてある。UI は変わるので、名前が合わなければ「同じことをしている場所」を探すこと。

### B-1. プロジェクトを用意する

https://console.cloud.google.com/ で新規プロジェクトを作る（既存のものでもよい）。
名前は何でもよい。以降の設定はすべてこのプロジェクトに紐づく。

### B-2. Gmail API を有効化する

**APIs & Services → Library → "Gmail API" → Enable**

有効化しないと、あとで同意フローが通ってもトークンで API を呼べない。

### B-3. Branding — 同意画面に出る文言

**Google Auth Platform → Branding**

| 項目 | 何を入れるか |
|---|---|
| App name | 何でもよい。同意画面に出るだけ |
| User support email | 自分のアドレス |
| Developer contact information | 同上 |

ロゴもホームページ URL も要らない。審査に出さないので、埋まっていればよい。

### B-4. Audience — External と「本番環境」

**Google Auth Platform → Audience**

- **User type: External**。個人 Gmail では Internal を選べない（Workspace 専用）
- **Publishing status を "In production" にする**

> [!WARNING]
> **"Testing" のままにしないこと。** Testing の外部アプリが発行した refresh token は
> **7 日で失効する。** つまりセットアップの 1 週間後に cron が静かに死ぬ。発行から
> 遠く離れて壊れるので、知らないと原因に辿り着けない。

### B-5. Data Access — スコープはひとつだけ

**Google Auth Platform → Data Access → Add or remove scopes**

```
https://www.googleapis.com/auth/gmail.readonly
```

**これ以外は足さない。** CI に置く資格情報が漏れたときの影響を「このメールボックスの
読み取り」に留めるため。送信も削除もラベル操作もできない権限にしておく。

`gmail.readonly` は Google の分類では制限付きスコープなので、審査を通していない
アプリでは同意画面の前に警告が出る（B-7）。

### B-6. Clients — デスクトップアプリのクライアント

**Google Auth Platform → Clients → Create client**

- **Application type: Desktop app**

Desktop を選ぶ理由は、同意フローがループバック
（`http://127.0.0.1:<ランダムポート>`）で完結し、**リダイレクト URI を登録しなくて
よい**から。Web application を選ぶと URI 登録が要るうえ、ポートが実行ごとに変わる
この方式と噛み合わない。

作成後に JSON をダウンロードし、**作業ディレクトリに `client_secret.json` として
置く**。

```bash
mv ~/Downloads/client_secret_*.json ./client_secret.json
chmod 600 client_secret.json
```

`.gitignore` 済み。**C を一度実行したら、このファイルはもう要らない**
（発行された `gmail-credentials.json` の中に client_id と client_secret が入る）。

### B-7. 同意画面で出る警告について

C を実行するとブラウザが開き、**「Google はこのアプリを確認していません」**と
表示される。審査を通していない自作アプリなので正常。

**詳細 → （安全ではないページ）に移動** で進める。

> [!NOTE]
> 未審査のアプリには**プロジェクト単位で生涯 100 ユーザーの上限**がある
> （クライアント ID を作り直してもリセットされない）。使うのが自分ひとりなら
> 一生かからないので、気にしなくてよい。

### B-8. やってはいけないこと

```bash
gcloud auth application-default login   # ← 使わない
```

gcloud は ADC 発行時に `cloud-platform` スコープを強制する。漏洩時の影響が
「メール読み取り」から「**Google Cloud プロジェクト全体の操作**」に跳ね上がる。
このツールは ADC へフォールバックしない実装になっている。

### 取り消したくなったら

https://myaccount.google.com/permissions で連携を解除すれば、refresh token は
即座に無効化される。再発行は C をもう一度実行するだけ。

## C. Gmail 資格情報の発行

このリポジトリに Go のコードは無い。アクション側をクローンせず、モジュールを
直接実行する。

```bash
# B-6 で client_secret.json を置いたディレクトリで
go run github.com/mpyw/moneyforward-paypaysec-bridge-action/cmd/mfpp@v3 \
  gmail authorize
```

起きること:

1. ローカルにループバックのサーバが立ち、ブラウザが開く
2. Google アカウントを選ぶ（**OTP が届くアカウント**を選ぶこと）
3. B-7 の「確認していません」警告 → **詳細 → 移動**
4. 「Gmail のメールの表示」の同意 → 許可
5. ブラウザがループバックに戻り、`gmail-credentials.json` ができる

疎通確認。**どのメールボックスが開いたか**を表示するので、アカウントを選び間違えて
いないかはここで分かる:

```bash
go run github.com/mpyw/moneyforward-paypaysec-bridge-action/cmd/mfpp@v3 \
  gmail check
```

終わったら、もう使わないほうを消しておく:

```bash
chmod 600 gmail-credentials.json
rm client_secret.json          # D 以降は不要
```

> [!CAUTION]
> `gmail-credentials.json` に入っている refresh token は**失効しない**。
> 実質的に、そのメールボックスへの恒久的な読み取り鍵。パスワードと同じ扱いをする。
> 漏れたら https://myaccount.google.com/permissions で連携を解除し、C をやり直す。

## D. secrets を登録

`gh secret set` は値をローカルで sealed box 暗号化してから送るので、平文が
GitHub に渡ることはない。引数に値を書かず、対話入力か stdin を使う（コマンド
履歴と `ps` に残さないため）。

| キー | 入力値 |
|---|---|
| `PAYPAYSEC_USERNAME` | PayPay 証券のログイン ID |
| `PAYPAYSEC_PASSWORD` | PayPay 証券パスワード |
| `MONEYFORWARD_EMAIL` | MoneyForward ログインメール |
| `MONEYFORWARD_PASSWORD` | MF パスワード |
| `MONEYFORWARD_ASSET_ID` | A でメモした手入力口座の ASSET_ID |
| `GMAIL_CREDENTIALS` | C で作った `gmail-credentials.json` の中身 |

```bash
for name in PAYPAYSEC_USERNAME PAYPAYSEC_PASSWORD MONEYFORWARD_EMAIL MONEYFORWARD_PASSWORD MONEYFORWARD_ASSET_ID; do
  gh secret set "$name"     # 値の入力を求められる
done
gh secret set GMAIL_CREDENTIALS < gmail-credentials.json
```

確認（名前と更新日時だけが出る。値は GitHub からも読み出せない）:

```bash
gh secret list
```

このリポジトリを action として他リポジトリから使う場合は、そちらの
Settings → Secrets and variables → Actions で同じ 6 つを設定する。

---

## 全部終わったら

- 平日 JST 15:30 の cron が動いて初回同期が実行される。
  **定刻には来ない。** GitHub の scheduler は混雑で 1〜3 時間ずれる。大引け後という
  要件は満たすので、遅れること自体は問題ない
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
| `deletes entries from categories this run did not read` | 台帳にあるカテゴリを一度も読んでいない | **口座は無傷**。ログの 8 行を見て、どのカテゴリが出ていないかを特定する。売却で銘柄が減っただけならこのエラーにはならない |
| `reported no error but …` | 書き込んだのに反映されていない | 併記される「the service said」を読む |
| `more than one entry named` | MF 側に同名行がある | 手で片方を消す |

**どれも口座を壊す前に止まる。** 書き込みは 1 件ごとに読み戻して検証し、削除は
その銘柄のカテゴリを実際に読めた実行でしか行わない。「もっともらしい間違った
数字を黙って記録する」経路は、敵対的レビューで見つかった 7 件を含めて塞いである。

