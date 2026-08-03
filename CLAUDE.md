# moneyforward-paypaysec-bridge-template

PayPay 証券の残高を MoneyForward の手入力資産に反映するワークフローの，フォーク元。

**このリポジトリにコードは無い。** 中身は
[moneyforward-paypaysec-bridge-action](https://github.com/mpyw/moneyforward-paypaysec-bridge-action)
を呼ぶ `sync.yml` と，利用者向けの文書だけ。

```
.github/workflows/sync.yml   # cron 平日 JST 15:30。アクションを uses: で呼ぶ
.github/dependabot.yml       # SHA 固定するなら必要
README.md                    # private fork の作り方と，何が書き換わるか
SETUP.md                     # A〜D の手順
```

## 編集するときの注意

- **実行は本体では走らない。** `sync.yml` に
  `if: github.repository != 'mpyw/moneyforward-paypaysec-bridge-template'` が入っている。
  外すと secrets を持たない本体で毎平日 cron が失敗し，public に赤が付き続ける
- **SHA 固定の例にリテラルを書かない。** 以前は書いてあって，リリースのたびに
  書き換える運用が 2 回連続で漏れ，例が「実在する銘柄を消すバグ入りの版」を指したまま
  残った。いまは引くコマンドを載せてある
- **secret 名はアクション側の input 名と 1 対 1 に対応する**（input を大文字にして
  ハイフンを `_` に）。アクション側にテストがあるので，こちらが勝手にずらすと合わなくなる
- **アクションのメジャーが上がったら README の移行案内ではなく，最新の姿だけを書く。**
  移行元が存在しない移行表は読者を混乱させるだけ

## 文書に書いてある「なぜ」を消さない

利用者が判断するために要る情報が入っている。要約すると:

- **private fork でないと保有銘柄が公開される。** 実行ログに銘柄名が出る
- **`MONEYFORWARD_ASSET_ID` の口座は丸ごと管理される。** 対応しない行は削除される。
  だから既存の資産ではなく新規に作らせている
- **OAuth 同意画面を「テスト」のままにすると 7 日で死ぬ。** 原因から遠く離れて壊れる
- **`@v3` は動くタグ。** 固定したいなら SHA-1，ただし DOM 変更時に修正が届かなくなる

## 追従

`.claude/skills/follow-upstream` に手順がある。
