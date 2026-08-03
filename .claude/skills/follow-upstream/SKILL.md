---
name: follow-upstream
description: このテンプレートの private fork を upstream に追従させるとき，またはアクションの破壊的変更を取り込むとき
---

# upstream への追従

このリポジトリは
[moneyforward-paypaysec-bridge-action](https://github.com/mpyw/moneyforward-paypaysec-bridge-action)
を呼ぶだけのテンプレート。実運用はこれの **private fork** で行う。

## 通常の追従

```bash
cd <あなたの private fork>
git remote add upstream https://github.com/mpyw/moneyforward-paypaysec-bridge-template.git  # 初回のみ
git fetch upstream
git merge upstream/main
git push origin main
```

"Use this template" で作った場合は履歴が繋がらないので merge できない。
**ミラーで作った fork** だけがこの方法で追従できる（README 参照）。

## アクションが破壊的変更を出したとき

`uses: …@v3` の **v が上がったとき**は，`sync.yml` を取り込むだけでは足りないことがある。

1. **secret 名が変わっていないか。** アクションの README の input 表と，
   自分の Secrets を突き合わせる
2. 変わっていたら，**新名で登録してから旧名を消す。** GitHub は secret の値を
   読み出せないのでリネームできず，登録し直しになる。順序を逆にすると，値が
   分からないまま消える

```bash
gh secret set NEW_NAME            # 値の入力を求められる
gh secret list                    # 新名が入ったことを確認してから
gh secret delete OLD_NAME
```

3. `sync.yml` の `uses:` のメジャーを上げる（merge で入るはず）
4. `gh workflow run sync.yml` で 1 回通す

## 取り込んだあと必ず見るもの

```bash
gh workflow run sync.yml
gh run watch
```

ログの読み方:

```
→ planned: create=0 update=1 unchanged=4 delete=0 (nothing written yet)
✓ created=0 updated=1 unchanged=4 deleted=0
```

- **`planned:` の行はまだ何も書いていない。** 拒否のチェックはこの後に走る。
  `planned:` だけ出て `✓` が出ていなければ，**書き込みは起きていない**
- `delete` が出たら止まって確認する。売却したなら正しい。していなければ読み取りが
  壊れている。ログの 8 行のうち，銘柄数が減っているカテゴリを見る

## フォークで確認すること

- **Actions タブが有効か。** ミラー push で作った private リポジトリでは無効なことが
  ある。無効だと cron も `workflow_dispatch` も動かない
- **デフォルトブランチが `main` か。** cron はデフォルトブランチのワークフローしか見ない
- **public になっていないか。** 実行ログには銘柄名が出る（金額と secret はマスク
  されるが銘柄名はされない）
