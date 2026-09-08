---
title: "「コミットにセッションURLが既定で付く」がHN135pt — 自社412コミットで確認したら0本だった"
emoji: "🔎"
type: "tech"
topics: ["claudecode", "git", "ai", "automation"]
published: true
---

「Claude Codeがコミットメッセージにセッションリンクを既定で付ける」という話がHacker Newsのフロントページで135pt・167コメントを集めていた。既定でON、しかも警告なし——そう聞いたら、自分のリポジトリがどうなっているか気になる。実際に自社の実運用リポジトリ2本・合計412コミットをgrepしてみたら、該当するリンクは1本も無かった。

## 話題になっていたこと

発端は GitHub issue `anthropics/claude-code#66504`「Claude Session URL appended to commit messages and PR descriptions by default」。2026-08-31時点でHacker Newsのフロントページに135pt・コメント167という規模で載っていた（Hacker Newsのfront_page APIで観測）。

issue自体の本文は今回読んでいないため、ここでは「この話題がHNで大きく取り上げられていた」という観測事実だけを使う。中身の主張を孫引きすることはしない。

## 自分のリポジトリではどうなっているか確認した

話を鵜呑みにする前に、手元の実運用リポジトリで確認した。使ったのはこの1行だけだ。

```
git log --all --format="%B" | grep -c "claude.ai/code"
```

対象は日常的にClaude Codeで開発している2つのリポジトリ。結果は次の通り。

リポジトリ総コミット数Co-Authored-By: Claude を含むclaude.ai/code を含む

HS_Web1341100
upclass-app2782460
合計4123560


AI著者つきコミットは356本ある。にもかかわらず、セッションURLを含むものは2リポジトリ合わせて0本だった。

## 抑止設定は触っていない、それでも0本だった

issueによれば、この機能を止める設定は `.claude/settings.json` の `attribution.commit` だという（これはissueの要約から引用する事実であり、自分で検証した挙動ではない）。両リポジトリのローカル設定と、グローバルの `~/.claude/settings.json` を確認したが、どちらにも `attribution` というキーは存在しなかった。つまり「意図的に切っていたから付いていなかった」わけではない。

さらに、HS_Web の `Co-Authored-By` 行をユニーク化すると、複数のモデル世代にまたがっていた。

```
Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>
```

Fable 5からOpus 5まで、複数のモデルにまたがって一貫してセッションURLは0本だった。単一バージョンの一時的な状態ではなさそうに見える。

## 検証していないことは検証していないと書く

ここで断定できることと、できないことを分けておく。

- **分かったこと**: 抑止設定を一度も触っていない環境で、412コミット中セッションURLは0本だった。

- **分からないこと**: なぜ0本なのかは特定していない。導入バージョンがissueの記載と一致しているか、headless実行（`claude -p`）では挙動が異なるのか、プランや地域による差があるのか、あるいはissueがクローズされた後に既定の挙動が変わったのか——このどれも今回は検証していない。

「既定でこうなる」という話をHNの点数だけで信じず、自分のリポジトリで同じコマンドを一度実行してみる価値はある。

## 読者が今すぐ試せること

自分のリポジトリで同じことを確かめるには、これだけでいい。

```
cd <対象リポジトリ>
git log --all --format="%B" | grep -c "claude.ai/code"
git log --all --format="%B" | grep -c "Co-Authored-By: Claude"
```

前者が0で後者が0より大きければ、今回の自社リポジトリと同じ状況——AI著者つきコミットはあるのに、セッションURLは付いていない、ということになる。原因の特定はまだ済んでいないが、まずは自分の環境の実態を数字で持っておくことが、この手の話題を追いかけるときの最初の一歩になる。
