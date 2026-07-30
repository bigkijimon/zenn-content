# zenn-content
Zenn/技術ブログ公開用（articles/ + images/）

## デプロイの投稿数上限（2026-07-30 実測）

Zenn Connect の初回連携時、`articles/` に7本溜めた状態で手動デプロイしたところ
**2本だけ公開され、5本が「投稿数の上限に達したためデプロイされませんでした」** となった。
直後にもう一度手動デプロイしても **0本** ＝ **上限はデプロイ単位ではなく時間単位**。

- 実測値: 1回のウィンドウで **2本**（公式FAQ https://zenn.dev/faq#rate-limit は数値を明示していない）
- 対処: 日を変えて手動デプロイを押し直す。残り本数 ÷ 2 が必要な回数の目安。
- 押したあと `bigkiji_blog_publish.py reconcile --apply` を回すと、公開された分が
  `公開記事レジストリ.csv` と `.published.json` に自動で取り込まれる（`.zenn-pending` も外れる）。

## 連携前に必ず通す検証

`bigkiji_publish_api.py` の `zenn_validate()` が push 前に title(70文字)・emoji(1文字)・
topics(1〜5)・slug を確認する。**手動デプロイは1本でも弾かれると全体が中断**するので、
1本の制限超過が他の記事の公開をまとめてブロックする（2026-07-30に実際に起きた）。
