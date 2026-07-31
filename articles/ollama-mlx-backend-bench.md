---
title: "Ollama 0.30.8はMLXランナーを内蔵するがGGUFは通らない — M1 Max 64GB実測"
emoji: "🧮"
type: "tech"
topics: ["ollama", "mlx", "applesilicon", "llm"]
published: true
---

「Ollamaが新しくMLXバックエンドに対応して、Apple SiliconでのローカルLLM推論が速くなった」という話をたまに見かける。手元はM1 Max 64GBのMac、Ollamaは0.30.8。じゃあ自分の環境でも恩恵があるのか確かめようとして、まず引っかかった。**そもそも自分が普段 `ollama pull` して `ollama run` しているモデルは、本当にMLXランナーを通っているのか？** 確かめずにtok/sだけ測っても、比較する土台が無い。バイナリ解析とログ突合で調べたら、答えは「通っていなかった」だった。

## 1. そもそも「MLXバックエンド」はOllamaのどこにあるのか

まずOllamaのバイナリ自体にMLX関連のコードが本当に入っているのか、`strings`で見てみる。

```
$ ollama --version
ollama version is 0.30.8

$ strings $(which ollama) | grep -o ".\{20\}mlx-engine.\{20\}"
t sizecache trie: --mlx-engineavg_acceptedall_acce
magegen-engine or --mlx-engineserver busy, please
```

バイナリの中には他にも `starting mlx runner` `mlx runner is ready` `MLX engine initialized` `mlx decode first token` `MLX not available: %w` といった文字列が実在する。つまり0.30.8には、GPUの推論をllama.cpp系ランナーではなくMLXで処理する専用のコードパスが**確かに埋め込まれている**。ここまでは事実。

次に、それをユーザーが明示的に選べるのか。`ollama serve --help` が公開している環境変数一覧を見る。

```
$ ollama serve --help
Environment Variables:
      OLLAMA_DEBUG                  ...
      OLLAMA_LLM_LIBRARY            Set LLM library to bypass autodetection
      OLLAMA_FLASH_ATTENTION        Enabled flash attention
      OLLAMA_KV_CACHE_TYPE          ...
      （MLXを名指しする環境変数は無い）
```

`--mlx-engine` はOllama本体が内部でランナーのサブプロセスを起動するときに渡すフラグであって、`ollama serve` や `ollama run` の`--help`には出てこない。つまり「MLXを使うにはこの環境変数を立てる」という公式な操作方法は、少なくとも`--help`の範囲には存在しない。ランナーの選択はOllama側の自動判定に委ねられている。

## 2. 実際にgenerateしてログを追ったら、llama-serverだった

コードの存在と、実際に呼ばれるかは別の話なので、手元の2モデルで試した。dense系の小さいモデルとMoE系の大きいモデル、アーキテクチャが違えば挙動も変わるかもしれないと思ったからだ。

```
$ ollama show qwen3.6:latest
architecture   qwen35moe
parameters     36.0B
quantization   Q4_K_M

$ echo "Explain what a Kalman filter does in one paragraph." | ollama run qwen3.6:latest --verbose
...
prompt eval rate:     157.20 tokens/s
eval count:           534 token(s)
eval rate:            58.91 tokens/s
```

generate直後に `~/.ollama/logs/server.log` の末尾を見ると、こう出ている。

```
time=...20:02:41... msg="llama-server started in 8.33 seconds"
time=...20:02:41... source=sched.go:729 msg="loaded runners" count=1
```

ソースは `llama_server.go`。`ollama ps` でも `qwen3.6:latest 29 GB 100% GPU` と出るだけで、mlxという文字はどこにも出ない。念のため9.7Bのdenseモデルでも同じ手順を繰り返した。

```
$ ollama show qwen3.5:latest
architecture   qwen35
parameters     9.7B
quantization   Q4_K_M

$ echo "Explain what a Kalman filter does in one paragraph." | ollama run qwen3.5:latest --verbose
...
eval rate:            40.18 tokens/s
```

こちらもログのソースは同じく `llama_server.go`。2モデル・2アーキテクチャ（dense/MoE）のどちらも、MLXランナーへは一度も分岐しなかった。

![M1 Max 64GB / Ollama 0.30.8での生成速度比較。qwen3.5:latest(9.7B dense)が40.18 tok/s、qwen3.6:latest(36.0B MoE)が58.91 tok/s。両モデルともllama-serverが処理。](https://raw.githubusercontent.com/bigkijimon/zenn-content/main/images/ollama-mlx-backend-bench__tokps-chart.png)

2モデルとも `ollama pull` した標準GGUF(Q4_K_M)。単発測定であり複数回の中央値ではない。


ここで分かったのは「MLXの方が速いか遅いか」ではなく、**そもそも比較の土俵に立っていなかった**ということだ。`ollama pull` で降ってくる標準GGUFモデルを `ollama run` するという、ほとんどのユーザーがやっている操作では、MLXランナーは呼ばれていない。

## 3. 外のベンチマークは何を測っていたのか

「Ollama MLX」で検索すると出てくる比較記事の一つ、llmcheck.netには「MLX 82 tok/s vs Ollama 70 tok/s」という数字が載っている。ただしこれは**M5 Max・Qwen4.1 32B-A3B**での計測であり、**M1 Maxの実測行は1本も存在しない**（このURLの中身自体はこちらで直接検証していない。編成便が事前に確認した要約を出典として引用している）。

つまり「MLXバックエンドでtok/sが上がる」という主張自体は他のハードウェア・他のモデル形式での話であって、M1 Max 64GBで`ollama pull`したGGUFモデルを回している人には、今のところ当てはまる話ではない、というのが今回の実測から言えることだ。

## 4. 再現できる学びと、次に確かめること

自分の環境でMLXランナーが実際に発火しているか確かめたいなら、3行で足りる。

```
# 1. コードパスの実在確認
strings $(which ollama) | grep -i "mlx runner"

# 2. 生成を1回実行
echo "test prompt" | ollama run  --verbose

# 3. 直後にログのソースを確認（mlxが出るかllama_server.goが出るか）
tail -n 40 ~/.ollama/logs/server.log | grep -iE "loaded runners|llama_server.go|mlx"
```

今回試していないのが、Hugging Faceのsafetensorsを直接pullするケース(`ollama pull hf.co/mlx-community/...`)だ。MLXネイティブの量子化フォーマットで持ってくれば、GGUF経由とは違ってMLXランナーが選ばれる可能性がある。ここは追加のダウンロードが要る検証なので、今回は「未検証」として切り分けておく。数字が無いところを埋めない、というのがこのブログの方針でもある。

結論: Ollama 0.30.8のバイナリには確かにMLXランナーが存在する。しかし少なくとも手元のM1 Max 64GBで、標準的な`ollama pull`→`ollama run`という経路を通す限り、2モデル・2アーキテクチャのどちらでもMLXランナーは呼ばれず、llama-server（Metal GPU 100%）が処理していた。「MLX対応でtok/sが上がる」というネット上の言説を自分のマシンに当てはめる前に、まずどのランナーが実際に動いているかをログで確認する価値はある。

