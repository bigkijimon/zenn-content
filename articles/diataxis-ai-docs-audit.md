---
title: "AIが無人で書き溜めた社内ドキュメント1,355本をDiátaxisで仕分けたら、3割が自社のドキュメントですらなかった"
emoji: "🗂"
type: "tech"
topics: ["Diataxis", "Ollama", "ローカルLLM", "ドキュメント", "ClaudeCode"]
published: true
---

Claude Codeに無人でドキュメントを書かせ続けて数か月経つと、「うちのナレッジベースは今どんな型に偏っているのか」が気になった。答え合わせに [Diátaxis](https://diataxis.fr/)（Tutorials / How-to guides / Technical reference / Explanation の4分類フレームワーク）を使い、自社リポジトリの Markdown をローカルLLMで全数分類した。結果は「tutorial型が0.0%」という綺麗な数字になったが、それより先に踏んだ罠のほうが再現性が高い教訓だった——**「何本あるか」を数える前に、その母数が本当に自分たちが書いたものかを疑う必要があった。**

## 母数を数える前に、母数が汚れていた

まず素直に数えた。

```
# $VAULT = 自社ドキュメントの置き場（各自の環境に読み替える）
find "$VAULT" -name "*.md" \
  -not -path "*/graphify-out/*" -not -path "*/node_modules/*" \
  -not -path "*/.next*/*" -not -path "*/_archive/*" | wc -l
→ 1355
```

1,355本。この数字をそのままDiátaxisに投げるところだった。だが内訳をディレクトリ単位で眺めると、同じルールファイルが違うパスに何度も出てくる。犯人は Claude Code 用スキルパッケージ `vercel-react-best-practices` のファイル75本(ルール72本+README/SKILL/AGENTS各1)で、1つのアプリ配下に5箇所——`skills/` `data/skills/` `agent/skills/` `.tabnine/agent/skills/` `.agents/skills/`——別々にインストールされていた。

```
diff -q skills/vercel-react-best-practices/rules/rendering-hoist-jsx.md \
        agent/skills/vercel-react-best-practices/rules/rendering-hoist-jsx.md
diff -q skills/vercel-react-best-practices/rules/rendering-hoist-jsx.md \
        .tabnine/agent/skills/vercel-react-best-practices/rules/rendering-hoist-jsx.md
# 差分なし。5箇所ともバイト単位で同一ファイル
```

ベンダー配布のスキルファイルとElectronアプリのビルド同梱docsをまとめて弾くと、438本(1,355本の32.3%)が「自分たちが書いたのではなく、ツールが持ち込んだもの」だった。

```
find "$VAULT" -name "*.md" ... | grep -E "/skills/|/dist/|\.tabnine" | wc -l
→ 438
```

残った**917本**が、実際にBigKiji自身(主にClaude Code)が無人で書き溜めた自社コーパスの実体だ。ここから先はこの917本を対象にする。

## Diátaxisという物差し

[diataxis.fr](https://diataxis.fr/) はドキュメントを4つの需要に分ける。原文の定義を借りると "Diátaxis identifies four distinct needs, and four corresponding forms of documentation"。4分類は Tutorials(学習)・How-to guides(特定タスクの遂行)・Technical reference(情報の参照)・Explanation(理解と背景)で、これを「実践的⇔理論的」「学習志向⇔参照志向」の2軸が分ける。

## ローカルLLMで917本を仕分ける

Claudeのトークンは使わず、Ollama上の `qwen3.5:latest` に投げた。各ファイル冒頭500文字を抜粋し、5択(tutorial/howto/reference/explanation/other)を1単語で返させる。最初の実装は空応答しか返ってこなかった。

```
curl http://localhost:11434/api/generate -d '{
  "model": "qwen3.5:latest",
  "prompt": "Classify ... Answer with exactly one word:",
  "options": {"num_predict": 10}
}'
→ {"response": "", "thinking": "Thinking Process:\n\n1. Analyze the Request...", "done_reason": "length"}
```

qwen3.5はデフォルトで思考トークンを吐くhybrid-thinkingモデルで、`num_predict`の予算をthinkingだけで使い切り、本題の1単語に到達する前に打ち切られていた。Ollama 0.30.8では `"think": false` を渡すとthinkingを止めて即答させられる。

```
{"model": "qwen3.5:latest", "prompt": "...", "think": false,
 "options": {"temperature": 0, "num_predict": 10}}
→ {"response": "explanation"}
```

917本の逐次分類にかかった時間は**1373.9秒(約22.9分)、平均1.498秒/件**(スクリプトの実測タイマー)。GPUはOllama単独稼働でComfyUI等の他ジョブは走らせていない状態。

## 結果 — 欠けていたのは"tutorial"、ゼロ件

![無人生成コーパス917本のDiátaxis内訳。reference 47.2%、explanation 26.9%、howto 21.4%、other 4.5%、tutorial 0.0%の横棒グラフ](https://raw.githubusercontent.com/bigkijimon/zenn-content/main/images/diataxis-ai-docs-audit__diataxis-breakdown.png)

n=917（全1,355本からベンダー配布物・ビルド成果物438本を除外）。分類=qwen3.5:latest(Ollama, think:false)。要点: 自社コーパスはreference/explanationに偏り、tutorialは0件——読者がAIエージェント自身であることの反映。


分類件数割合

reference43347.2%
explanation24726.9%
howto19621.4%
other414.5%
tutorial00.0%


最大はreference(47.2%)、最小はtutorialでゼロ。念のため「はじめに」「Step 1」「チュートリアル」等のキーワードを含む36本を別途スキャンしたが、その中にもtutorial判定は1本も出なかった——分類器の癖ではなく、実際にそういう内容がなかったということだ。組織の「骨格」にあたるドキュメント(部署ごとのナレッジ地図`MOC_*`6本・各部署の追記式ログ`INDEX.md`20本・全体入口`000_START_HERE.md`)も個別に確認したところ、MOC_*はreference4本/explanation2本、INDEX.mdは20本中18本がreferenceで残り2本がhowto——どちらにもtutorialは1本も無く、骨格そのものの設計意図がtutorial型を持っていなかった。

理由は分かりやすい。この917本の主な読み手はAIエージェント自身であって、初心者の人間ではない。エージェントは「手順を追って学ぶ」必要がなく、「今必要な事実を1回で引く(reference)」か「なぜそうなっているかを把握する(explanation)」かのどちらかを求める。tutorialが生まれない構造は、無人ドキュメント生成が悪いのではなく、読者の性質を正しく反映した結果だった。

## 分類パイプライン自身にもバグがあった

初回実行では917本中22本が問答無用で"other"になっていた。原因はPythonの定番の罠で、パスの先頭2文字("./")を剥がすつもりで書いた `path.lstrip("./")` が、実際には「先頭から'.'か'/'に含まれる文字を続く限り削る」という文字集合ベースの処理だった。

```
# 誤り: "./.pi-subagents/artifacts/x.md" → "pi-subagents/artifacts/x.md"
#       (先頭の "./" だけでなく、隠しディレクトリの "." まで一緒に剥がれる)
full = os.path.join(root, path.lstrip("./"))

# 修正: プレフィックスの明示的な除去に置き換え
full = os.path.join(root, path[2:] if path.startswith("./") else path)
```

`.pi-subagents/` `.pi/` `.claude/` 配下の隠しディレクトリだけがこの罠を踏み、ファイルが見つからず例外→機械的に"other"へフォールバックしていた。修正して22本を再分類すると、"other"のまま残ったのは3本だけで、残り19本はreference/explanation/howtoへ再分配された。上記の表はこの修正後の最終値である。

## 学び

◆ドキュメント型の分析結果 
無人ドキュメント生成は「参照」と「説明」と「作業ログ」を量産するのは得意だが、「学習コンテンツ」を自然には生まない。読み手がAIエージェント自身である社内コーパスでは、それはむしろ健全な偏りだ。

◆実務的な教訓(母数チェック) 
母数を数える段階で32.3%がベンダー配布物というノイズだったことのほうが、実務上はよほど効く教訓だった。再現する手順は3行に収まる。

```
# 1. 母数を数える(除外条件を明示)
find . -name "*.md" -not -path "*/node_modules/*" -not -path "*/.next*/*" | wc -l
# 2. ベンダー/ビルド成果物を機械的に弾く
... | grep -E "/skills/|/dist/|\.tabnine" | wc -l
# 3. 残った母数だけをローカルLLMで仕分ける(think:false を忘れない)
```

1,350本という数字を鵜呑みにして分類を始めていたら、Diátaxisの結果そのものより先に「ベンダーのReactルールが自社ドキュメントの3割を占めている」という誤った印象を記事にするところだった。数える前に、数える対象を疑う。ローカルLLMは無料で使い倒せるからこそ、その前段の母数チェックにも同じくらい時間をかける価値がある。
