---
title: "【実測】M1 MacのComfyUIで実写がボヤける原因は「KSampler一発出し」だった — RealVisXL 2パスHires+4x-UltraSharpに組み替えた全ノード構成"
emoji: "🖼️"
type: "tech"
topics: ["ComfyUI", "StableDiffusion", "AppleSilicon", "SDXL"]
published: true
---

ComfyUI で実写を出すと「ディテールが甘い」「顔が崩れる」「解像度を上げると破綻する」。原因の大半は **単発 KSampler 一発出し**です。この記事は、それを **2 パス Hires fix + FreeU_V2 + 4x-UltraSharp** に組み替えて M1 Mac（ComfyUI `:8000`）で実際に回した **全ノード・全パラメータ（seed 含む）**を、出力 **2688×3456** まで再現できる形で公開します。

持ち帰ってほしい 1 点だけ先に書くと、**「ベースを小さく出して段階的に拡大し、2 パス目の denoise を 1.0→0.42 で締める」二段構え**が、ローカル実写の破綻を消す再現可能な最小構成です。**本記事は 2 パス Hires の「土台」までを扱います。顔固定（IPAdapter / PuLID）はモデル在庫済みですが本記事では未使用＝次段**とし、ここは後半で正直に線引きします。

> 先に断っておくと、本記事は**設定と目視比較の実録**です。生成所要時間・VRAM / ユニファイドメモリ消費・it/s・単発版との定量スコアは**未計測**で、数字は出しません。出す数字はすべて WF JSON の実値と output PNG の実測ピクセルだけです。

## 問題：単発 KSampler 一発出しだと実写がボヤける・崩れる

SDXL 実写モデルを入れて、Checkpoint → KSampler → VAEDecode → Save の最短経路で出す。チュートリアルの多くがこの形です。これで出るのは「ベース解像度のまま、肌がのっぺりして、拡大すると破綻する」画像でした。実際に単発版で確認できた破綻は、**肌がプラスチックのようにのっぺりして毛穴・質感が消える**、**等倍以上に拡大すると髪の毛の束や輪郭のエッジがボヤけて溶ける**、という 2 点です。この状態から拡大処理をかけても、ボケた情報を引き伸ばすだけでディテールは戻りませんでした。

この構成は社内のデザイン部門ドクトリンでも**明文で禁止**されています。実写・図解のワークフロー方針を定めた正本には、こうあります。

> 禁止：教科書的な単発 KSampler 一発出し（プロ多段 = Hires fix 2 パス + IPAdapter / PuLID 顔固定 + 4x-UltraSharp 必須）。
>
> *— 社内デザイン部門の実写ワークフロー正本より（「単発 KSampler 一発出しは禁止」）*

なぜ一発出しがダメなのか。SDXL は 1024px 級で学習されていて、いきなり最終解像度を狙うと構図もディテールも安定しません。かといって高解像度の EmptyLatent を一発で叩くと、破綻する（同じ顔が二重に出る等）か、M1 のメモリを一気に食います。この 2 つを同時に回避するのが 2 パス Hires fix です。差は後述の比較図とクロップ画像で見ます。

## 解法の全体像：2 パス Hires fix + FreeU_V2 + 4x-UltraSharp

単発をやめて足したのは 3 つの武器です。

1. **2 パス化**：ベースを小さめ（896×1152）に出し、latent を 1.5 倍してから 2 パス目でディテールを足す。
2. **FreeU_V2**：UNet のスキップ接続を調整してディテールを底上げする地味な一手。
3. **4x-UltraSharp**：ピクセル空間で 4 倍に持ち上げ、そこから 0.5 倍に戻してディテールを凝縮する。

モデルは用途で切り替えます。**実写は RealVisXL_V5.0_fp16、図解・英字ものは flux1-dev**。SDXL 系は図解の細い線・文字が苦手なので、実写と図解を同じモデルで押し通さないのが判断基準です（本記事は実写＝RealVisXL のライン）。

![単発KSamplerの短い直線フロー（左・ダメ版）と、FreeU_V2・2パスHires・4x-UltraSharpを挟んだ長いフロー（右・良い版）を並べたノード構成比較図。右は最終2688×3456に到達する。](https://raw.githubusercontent.com/bigkijimon/zenn-content/main/images/ai-image-diary-1__ai-image-diary-1-flow-compare.png)
*図①（検証済み・本記事の視覚検証の主役）：単発 KSampler（左・ダメ版）と 2 パス Hires 実構成（右・良い版）のノードフロー比較。ノード名・パラメータは `PRO_RealVisXL_2pass.json` の実値。*

## 全ノード構成と実パラメータ（そのまま再現できる）

実際に回した `PRO_RealVisXL_2pass.json` のノード列を、上から順に全部実値で並べます。同じ値で組めば同じチェーンになります（API 形式 JSON と、:8000 へ投入する Python スクリプト `pro_gen_realvisxl_2pass.py` は同一パラメータで一致）。

| # | class_type | 主要 inputs（実値） |
|---|---|---|
| 1 | CheckpointLoaderSimple | RealVisXL_V5.0_fp16.safetensors |
| 2 | FreeU_V2 | b1=1.3, b2=1.4, s1=0.9, s2=0.2 |
| 3 | CLIPTextEncode (pos) | cinematic editorial portrait, 85mm lens, detailed realistic skin texture with pores, film grain … |
| 4 | CLIPTextEncode (neg) | worst quality, blurry, deformed hands, plastic skin, waxy, cartoon, 3d render … |
| 5 | EmptyLatentImage | width=896, height=1152, batch_size=1 |
| 6 | **KSampler ①** | seed=770077, steps=**32**, cfg=**5.5**, sampler=dpmpp_2m_sde, scheduler=karras, denoise=**1.0** |
| 7 | LatentUpscaleBy | method=nearest-exact, scale_by=**1.5** |
| 8 | **KSampler ②（Hires fix）** | seed=770077, steps=**20**, cfg=5.5, dpmpp_2m_sde / karras, denoise=**0.42** |
| 9 | VAEDecode | latent → pixel |
| 10 | UpscaleModelLoader | 4x-UltraSharp.pth |
| 11 | ImageUpscaleWithModel | 4x-UltraSharp で ×4 |
| 12 | ImageScaleBy | method=area, scale_by=**0.5** |
| 13 | SaveImage | filename_prefix=PRO_RealVisXL_2pass |

2 つの KSampler の差分だけ抜き出すとこうです。ここが 2 パスの肝です。

```python
# 1パス目：ゼロから構図を作る（denoise = 1.0）
KSampler(seed=770077, steps=32, cfg=5.5,
         sampler_name="dpmpp_2m_sde", scheduler="karras", denoise=1.0)

LatentUpscaleBy(upscale_method="nearest-exact", scale_by=1.5)

# 2パス目 = Hires fix：構図は壊さずディテールだけ足す（denoise = 0.42）
KSampler(seed=770077, steps=20, cfg=5.5,
         sampler_name="dpmpp_2m_sde", scheduler="karras", denoise=0.42)
```

プロンプトの要点は「肌の質感を明示的に呼び、ネガティブでプラスチック肌・3D レンダ調を殺す」ことです。

```text
positive: … detailed realistic skin texture with pores, film grain, 85mm lens …
negative: worst quality, blurry, deformed hands, plastic skin, waxy, cartoon, 3d render …
```

### 解像度チェーン：実値から算出（output 実測と一致）

各ノードの scale 値を掛けていくと最終解像度が出ます。これが output PNG の実測ピクセルと完全に一致します。

![896×1152からlatent×1.5で1344×1728、4x-UltraSharpで5376×6912、area×0.5で最終2688×3456へ至る解像度チェーンの段階図。](https://raw.githubusercontent.com/bigkijimon/zenn-content/main/images/ai-image-diary-1__ai-image-diary-1-res-chain.png)
*図②（検証済み）：解像度チェーン。896×1152 →(latent×1.5)→ 1344×1728 →(4x-UltraSharp)→ 5376×6912 →(area×0.5)→ **2688×3456**。この計算値は output PNG の sips 実測ピクセルと一致する。*

## なぜこの刻み方なのか（実値から読める設計意図）

パラメータの「なぜ」を、憶測でなく実値から言える範囲で書きます。

### denoise 1.0 → 0.42 の意味

1 パス目は `denoise=1.0`＝完全にノイズから構図を作る。2 パス目は `denoise=0.42`＝入力（拡大済み latent）を 42% だけ壊してディテールを足す。ここを高くしすぎると 1 パス目の構図・顔が変わってしまい、低すぎるとただのボケた拡大になる。0.42 は「構図を保ったままディテールを乗せる」実運用値です。

### 4 倍に上げてから 0.5 倍で締める理由

4x-UltraSharp で一度 5376×6912 まで持ち上げ、`area` 法で 0.5 倍に戻す。最初から実効 2 倍を狙わずに**わざと過剰解像度を作ってから縮小する**のは、ディテールを面積平均で凝縮してエッジのジャギやアップスケーラ特有のノイズを均すためです。実効倍率は 2 倍ですが、経路が違うと締まり方が変わります。

### ベースを 896×1152 に抑え latent×1.5 で刻む二段構え

いきなり大きな EmptyLatent を叩かず、896×1152 → latent×1.5 と刻む。M1 のユニファイドメモリを一気に食わせない設計だと読めます。**ただし OOM 回避の因果は設計上の推定で、VRAM / メモリ消費は未計測**です。ここは「そう組んである」という事実と、「たぶんこの理由」という推定を分けて書いておきます。

## 【前言撤回】ドクトリンは「顔固定必須」と言うが、実 WF に顔固定ノードは無い

この記事の背骨です。前掲のドクトリンは「プロ多段 = Hires fix 2 パス + **IPAdapter / PuLID 顔固定** + 4x-UltraSharp 必須」と書いています。ところが、**実際に回した `PRO_RealVisXL_2pass.json` には IPAdapter / PuLID ノードが一つも入っていません**。入っているのは FreeU_V2 + 2 パス Hires + 4x-UltraSharp だけです。

つまり「顔固定必須」と自分で書いておきながら、土台の WF は顔固定なしで回していた。これは隠さず段階として書いた方が再現性が上がると判断しました。立て付けはこうです。

- 顔固定（IPAdapter-FaceID / PuLID）は**別ラインの武器**。まず土台の 2 パス Hires だけで、実写の破綻（ボケ・のっぺり・拡大破綻）がどこまで消えるかを見るのが今回の到達点。
- 顔固定モデルは在庫済みで、次段として繋げられる（`models/ipadapter`・`models/pulid`・`models/insightface` ディレクトリが実在）。

「ドクトリン通りに全部盛りできていない」ことを到達点として明示する方が、読者が自分の環境で段階的に組むときに迷いません。全部盛りの完成形をいきなり見せるより、土台がどこまで効くかの切り分けが実用的です。

## 実出力と再現できている証拠

回した実物です。output PNG 2 枚の実測ピクセルが、WF の計算チェーン（図②）と一致します。これが「再現できている」ことの証拠です。

| 実物 / 依存 | 実測値 |
|---|---|
| PRO_RealVisXL_2pass_00001_.png（Jul 13） | sips 実測 **2688×3456**・約 11.8MB |
| PRO_RealVisXL_2pass_00002_.png（Jul 14） | sips 実測 **2688×3456**・約 11.8MB |
| RealVisXL_V5.0_fp16.safetensors | 約 6.94GB |
| 4x-UltraSharp.pth | 約 67MB |
| 基盤 | ComfyUI :8000 launchd 常駐 / base_path=`/path/to/ComfyUI` |

![単発KSampler出力（左）と2パスHires+4x-UltraSharp出力（右）の等倍クロップ比較。同一seed=770077・同一プロンプト・同一クロップ位置。左は輪郭・髪・肌がボヤけ、右は髪の一本・肌の質感までシャープ。](https://raw.githubusercontent.com/bigkijimon/zenn-content/main/images/ai-image-diary-1__ai-image-diary-1-compare.png)
*単発 KSampler 出力（左・denoise1.0・Hiresなし・896×1152を素朴に拡大）vs 2 パス Hires+4x-UltraSharp 出力（右）の等倍クロップ（**同一 seed=770077・同一プロンプト・同一クロップ位置**）。左は輪郭・髪・肌がボヤけ、右は髪の一本・肌の質感までシャープに解像しています。あわせて図①のノード構成図と図②の実出力解像度チェーン（output PNG 実測 2688×3456 と一致）も参照。差は目視比較のみで語ります（定量スコアは未計測）。*

## 正直に言う：測っていないこと

誤読を防ぐため、数字を出さない範囲を明記します。

- **生成所要時間**：未計測。投入スクリプトのポーリングは「3 秒 × 最大 120 回 = 360 秒」の**タイムアウト上限**であって実測時間ではありません。だから「約○分」とは書きません。
- **VRAM / ユニファイドメモリ消費**：未計測（ログなし）。
- **it/s・サンプリング速度**：未計測。
- **単発版との定量比較スコア**：未計測。品質差は出力画像の目視比較でのみ語ります。

## まとめ：再現できる学び

1. ローカル実写の破綻の主因は単発 KSampler。**2 パス Hires fix（denoise 1.0 → 0.42）**が最小の効く一手。
2. **FreeU_V2 + 4x-UltraSharp（×4 して ×0.5 で締める）**でディテールを凝縮する。
3. M1 では**ベース 896×1152 → latent×1.5** と刻み、いきなり大解像度の latent を作らない。
4. **seed=770077 を含め全パラメータを公開**＝同じ値で誰でも再現でき、output 実測 2688×3456 と一致する。
5. 顔固定（IPAdapter / PuLID）は土台の**次段**。土台の 2 パスだけでどこまで破綻が消えるかを段階で見るのが誠実で、実装しやすい。

次段は、在庫済みの顔固定ライン（IPAdapter-FaceID / PuLID）をこの 2 パス土台に接続します。本記事はその土台の全公開まで。同じ値で組んで、まず土台の効きを自分の目で確かめてみてください。
