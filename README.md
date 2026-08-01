# moneyforward-paypaysec-bridge-template

MoneyForward が PayPay 証券に非対応なので、保有銘柄を毎営業日スクレイピングして
MoneyForward の手入力口座に反映する。そのためのフォーク用リポジトリ。

銘柄ごとに 1 つの資産として登録される。

```
[米国株] テスト電機                  評価額 456,789 円 / 取得 400,000 円
[米国株] テスト商事                  評価額 234,567 円 / 取得 200,000 円
[ミニ] テスト電機                    評価額   3,210 円 / 取得   3,500 円
[投信ミ] テスト・グローバル・ファン…  評価額 345,678 円 / 取得 300,000 円
[投信ミ] テストAIファンド            評価額   5,432 円 / 取得   5,800 円
```

数字と銘柄名は例。

中身は
[moneyforward-paypaysec-bridge-action](https://github.com/mpyw/moneyforward-paypaysec-bridge-action)
を呼ぶワークフローが 1 つあるだけ。**アクション側をクローンする必要はない。**

---

## ⚠ 最初に: private にしてからフォークする

このリポジトリを普通に Fork すると、**あなたの fork も public になる。**
GitHub の Fork ボタンは public → private の変換ができない。

`sync.yml` 自体に秘密は無いが、**public リポジトリでは Actions の実行ログが
誰にでも見える**。このジョブのログには読み取った銘柄名が出る。secrets は
マスクされ、金額もプログラムがマスクするが、**銘柄名はマスクされない。**
public のまま動かすと、あなたの保有銘柄が公開される。

### private fork の作り方

```bash
# 1. 空の private リポジトリを作る
gh repo create <あなた>/moneyforward-paypaysec-bridge-private --private

# 2. ミラーで中身を移す
git clone --bare https://github.com/mpyw/moneyforward-paypaysec-bridge-template.git
cd moneyforward-paypaysec-bridge-template.git
git push --mirror https://github.com/<あなた>/moneyforward-paypaysec-bridge-private.git
cd .. && rm -rf moneyforward-paypaysec-bridge-template.git

# 3. 作業用にクローンし、upstream を張る
git clone https://github.com/<あなた>/moneyforward-paypaysec-bridge-private.git
cd moneyforward-paypaysec-bridge-private
git remote add upstream https://github.com/mpyw/moneyforward-paypaysec-bridge-template.git
```

以後、更新は `git fetch upstream && git merge upstream/main` で取り込める。

**"Use this template" ボタンでも private リポジトリは作れる**が、履歴が繋がらないので
`upstream` を張っても merge できない。追従したいならミラーのほうを使う。

### フォーク直後に確認すること

- **Actions タブが有効になっているか。** ミラー push で作った private リポジトリでは
  無効なことがある。無効だと cron も `workflow_dispatch` も動かない
- **デフォルトブランチが `main` か。** cron と `workflow_dispatch` は
  デフォルトブランチのワークフローしか見ない

---

## セットアップ

[SETUP.md](./SETUP.md) に A〜D の 4 ステップがある。要約:

1. **MoneyForward に手入力資産を 1 つ作る**（ブラウザ）。URL の `account_id_hash` をメモ
2. **Google Cloud で Gmail API を有効化し、OAuth クライアント（デスクトップ）を作る**（ブラウザ）
3. **Gmail 資格情報を発行する**（ターミナル）:
   ```bash
   # client_secret.json を置いたディレクトリで
   go run github.com/mpyw/moneyforward-paypaysec-bridge-action/cmd/mfpp@v1 \
     gmail authorize
   ```
4. **secrets を 6 つ登録する**（ターミナル）:
   ```bash
   for name in PAYPAY_SEC_USERNAME PAYPAY_SEC_PASSWORD MF_EMAIL MF_PASSWORD MF_ASSET_ID; do
     gh secret set "$name"
   done
   gh secret set GMAIL_CREDENTIALS_JSON < gmail-credentials.json
   ```

動作確認:

```bash
gh workflow run sync.yml
gh run watch "$(gh run list --workflow sync.yml --limit 1 --json databaseId --jq '.[0].databaseId')"
```

### 前提

**PayPay 証券と MoneyForward の登録メールアドレスが、同じ Gmail の受信箱に
届くこと。** OTP を Gmail API で読む。転送でも受信はできるが、OTP メールは
**送信者アドレスで絞っている**ので、転送で `From:` が書き換わる経路では動かない。

---

## 何が書き換わるか

**`MF_ASSET_ID` で指定した手入力口座の中身は、このジョブが管理する。
保有銘柄に対応しないエントリは削除される。** 他の口座には触らない。

既存の資産を指定しないこと。**新しく 1 つ作って、それを渡す。**

安全側の制限が 3 つある:

- **読まなかったカテゴリからは削除しない。** 8 ページのどれか 1 つでも読めなければ
  実行全体が失敗するので、削除計画が立つ時点で全ページが検証済み。そのうえで、
  台帳にあってこの実行が一度も見なかったカテゴリの行は「古い」ではなく「未検証」
  なので、消さずに落とす
- **売却で銘柄が減るのは正常系。** 何銘柄減っても、そのカテゴリを読めていれば
  そのまま反映される
- ページが読み込み中のプレースホルダを返している間は値を採用しない。
  投資信託ページは非同期ロード中に全項目 0 円を表示し、それは整合性チェックを
  すべて通ってしまう

## 頻度を上げないこと

両サービスとも、短時間にログインを繰り返すと **OTP メールの送信自体を止める。**
5 回程度で止まった実績がある（同日中に復活した）。失敗しても連打しない。

## License

MIT. [LICENSE](./LICENSE) を参照。
