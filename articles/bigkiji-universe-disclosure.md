---
title: "AIコーディングCLIが「何を送ったか」を承認してから走らせる — 開示マニフェストの実装"
emoji: "🔏"
type: "tech"
topics: ["ClaudeCode", "Electron", "セキュリティ", "AIエージェント", "個人開発"]
published: true
---

Claude Code や Codex に仕事を投げるとき、**そのプロセスが自分のマシンから何を持ち出したか**を、後から正確に言えるでしょうか。

私は言えませんでした。ログには「読んだファイル」の名前は出ます。でも、そのファイルの**どの行**が、**どのモデル**に、**どんな前処理を経て**渡ったのかは残りません。うまくいっている間は誰も困りません。困るのは、うまくいかなかった時にだけです。

この問題のために作ったデスクトップアプリを、Apache-2.0 で公開しました。

https://github.com/bigkijimon/bigkiji-universe

この記事は宣伝ではなく、**その中心にある「開示マニフェスト」という一つの仕掛け**の話です。他でも使える考え方だと思うので、実装ごと書きます。

## 何が問題だったのか

外部のAI CLIに仕事を任せる構成は、だいたいこうなります。

1. こちらがプロンプトを組む
2. CLIがファイルを読む
3. どこかのAPIへ送る
4. 結果が返る

このうち **2と3の境目に、人間が見られる断面がありません**。「送る直前の中身」は、送った後には存在しないからです。

「ログを取ればいい」と思うかもしれません。しかしログは*送った後の記録*です。送る前に止めて、中身を見て、それから許可する — という順番でなければ、判断の余地はありません。事後に読む記録と、事前に承認する対象は、別物です。

## 開示マニフェストという断面

やったことは単純です。**送る内容そのものをハッシュで封をして、そのハッシュを人間に承認させる。**

実装は `src/domain/pi-core/security/disclosure-manifest.js` の46行、うち封をする関数はこの12行です。

```js
function createDisclosureManifest({ runId, provider, purpose, policy, slices,
                                    redactions, estimatedTokens, payload,
                                    externalTools, model }) {
  const files = slices.map((item) => {
    const absolute = path.resolve(policy.vaultRoot, item.path);
    return {
      path: item.path.replace(/\\/g, '/'),
      ranges: item.ranges || [],
      sha256: fileHash(absolute),
    };
  });
  const base = {
    version: 2, runId, provider,
    model: String(model || ''),
    purpose: String(purpose || '').slice(0, 240),
    files,
    redactions: redactions.map(({ type, count }) => ({ type, count })),
    externalTools: normalizeExternalTools(externalTools),
    estimatedTokens: Number(estimatedTokens || 0),
    payloadHash: sha(String(payload || '')),
    policyHash: policy.security.policyHash,
  };
  return { ...base, disclosureHash: sha(JSON.stringify(base)) };
}
```

封じているものを一つずつ:

| 項目 | なぜ入っているか |
|---|---|
| `files[].sha256` + `ranges` | ファイル名だけでは足りない。**どの行が**渡ったかまで固定する |
| `model` | 「Opusにこれを読ませる」の承認が、別モデルの承認に化けないため |
| `payloadHash` | 組み上がった本文そのものの指紋 |
| `policyHash` | どのサンドボックス規則の下で作られたか |
| `redactions` | 何を何件伏せたか（種類と件数だけ。中身は当然入れない） |
| `externalTools` | 外部へ問い合わせる場合、その**問い合わせ文そのもの** |

これら全部を1つのJSONにして、さらにそのSHA-256を取ったのが `disclosureHash` です。

**人間が承認するのは、このハッシュです。**

### なぜ「ハッシュを承認する」のか

ここが一番言いたいところです。

普通の承認ボタンは「実行しますか？ はい/いいえ」です。でもそれは**押した瞬間の画面**への同意でしかありません。押してから実際に起動するまでの間に何かが変わっていたら、その同意は何を保証しているのでしょうか。

だから起動側は、承認されたハッシュを**もう一度検証してから**でないとプロセスを起こしません。

```js
if (policy.security?.policyHash !== task.disclosure?.policyHash)  throw new Error('STALE_SECURITY_POLICY');
if (!verifyDisclosureManifest(task.disclosure, policy, task.preparedPrompt)) throw new Error('STALE_DISCLOSURE_MANIFEST');
if ((task.disclosure.model || '') !== (task.model || ''))         throw new Error('STALE_MODEL_SELECTION');
```

検証側はファイルを読み直してハッシュを取り直します。

```js
function verifyDisclosureManifest(manifest, policy, payload = '') {
  if (!manifest || manifest.policyHash !== policy.security.policyHash) return false;
  if (!manifest.payloadHash || manifest.payloadHash !== sha(String(payload || ''))) return false;
  return manifest.files.every(
    (item) => fileHash(path.resolve(policy.vaultRoot, item.path)) === item.sha256
  );
}
```

つまり **承認してから起動するまでの間に1バイトでも動いていたら、実行されずに落ちます。** 「たぶん同じはず」で走らせない。ここだけは楽観を許さない設計にしました。

`STALE_MODEL_SELECTION` を別立てにしているのは、モデルの差し替えがいちばん静かに起きるからです。フォールバックで安いモデルに落ちるのは運用としては正しい。でもそれは**承認されていない実行**です。落として、もう一度承認を取ります。

## 送る量を先に減らす

マニフェストが正直であるためには、そもそも封じる対象が小さくないと意味がありません。丸ごと送ってハッシュを取っても、それは「全部送りました」の証明にしかならない。

前段のプルーナは既定でこう絞っています（`src/domain/pi-agent/context-pruner.js`）。

- ファイル 10本
- 48,000 文字
- 12,000 トークン

そして関連する箇所は**前後24行のスライス**で取ります。ファイル単位ではなく行範囲単位。マニフェストの `ranges` はここから来ます。

> ここで正直に書いておくと、**「何％削減できた」という数字はこのリポジトリに存在しません。** ベンチマークを置いていないので、READMEにも書いていません。書けることだけ書く、というのはこの手のツールでは特に大事だと思っています。

## 起動したプロセスに何を渡さないか

承認が通ったあと、子プロセスに渡す環境は最小にしています。

- 専用の `0700` な HOME と TMPDIR（使い捨て）
- **そのプロバイダの鍵だけ**。他社の鍵は渡らない
- Claude Code には `--strict-mcp-config` と空のMCP設定、`--disallowed-tools WebSearch,WebFetch,mcp__.*`
- Codex には `--ephemeral --ignore-user-config`、web_search 無効
- 実行はgitのworktreeで隔離。マージもコミットもpushもできない

ツールの入口には `PreToolUse` フックを噛ませて、web系と `mcp__*` を全部拒否、シェルは許可リストのみ（パイプ・リダイレクト・ネットワーク系バイナリは不可）。外部へ出る道は**ブローカー1本だけ**にしてあります。道が1本なら、そこだけ見張ればいい。

## 作ってみて分かったこと

**サンドボックスは、隠したいものだけでなく、要るものまで隠します。**

使い捨てHOMEを渡した結果、Claude Code が `Not logged in · Please run /login` と言い出しました。macOSの `security` はログインキーチェーンを `$HOME/Library/Keychains` から探すので、HOMEを差し替えた時点で**認証情報の在り処ごと**消えていたわけです。Codexは401。

「27回割り当てて有料実行ゼロ」の原因はこれでした。プロバイダが壊れていたのではなく、**一度も認証されていなかった**。

直し方は、鍵をばら撒くのではなく、そのCLIが起動に必要な1ファイルだけを名指しで貸すことでした。読み取り専用、タスクと同時に死ぬ。

## 公開して分かったこと（おまけ）

公開にあたってCIを直したら、**macOS以外では一度も通っていなかった**ことが分かりました。`npm ci` がロックファイルの不整合で落ちており、その裏に移植性のバグが積み上がっていました。

いちばん効いたのはこれです。**サンドボックスの判定が、比較の左辺と右辺で違う関数を使っていた。**

- 許可ルート側: `fs.realpathSync`
- 対象パス側: `fs.realpathSync.native`

macOSとLinuxでは同じ結果になります。Windowsでは違います。前者は8.3短縮名を展開せず、後者は展開する。結果、

```
許可: C:\Users\RUNNER~1\AppData\Local\Temp\...\project
対象: C:\Users\runneradmin\AppData\Local\Temp\...\project
```

が別物と判定され、**サンドボックス内の全読み取りが拒否**されていました。fail-closed なので穴ではありません。ただしWindowsではアプリが自分の作業ディレクトリを読めない。

教訓として一般化すると、**同じ場所を指す2つの綴りを比べていないか**は、パスを扱うコードで必ず疑う価値があります。そして今回それを見つけたのは私ではなく、**赤いまま放置されていたCI**でした。このリポジトリのCIは4日間赤で、その間ずっと「3OSで検査しています」と名乗っていました。通らない検査は、短くても無いのと同じです。

再発防止は、振る舞いのテストではなくソースの検査にしました。短縮名を持つプラットフォームでしか発火しない条件を、振る舞いで縛るのは無理があるからです。

```js
assert.doesNotMatch(sandboxSource, /fs\.realpathSync(?!\.native)/,
  'sandbox-policy must canonicalise through security-policy.canonical');
```

（この検査自体、最初は自分のコメントを拾って落ちました。コメント行を除いてから見ています。）

## まとめ

* **事後のログではなく、事前に承認できる断面を作る。** それがハッシュ1つなら、人間が見られる大きさになる
* **承認したものと実行するものが同一であることを、実行側が再検証する。** 「たぶん同じ」で走らせない
* **モデルの差し替えは承認の対象に含める。** 静かに起きるから
* **サンドボックスは要るものも隠す。** 認証が消えるところまで想像しておく
* **通らないCIは、無いのと同じ。** 直した瞬間に、隠れていたバグがまとめて出てくる

コードは全部読める状態にしてあります。設計判断の経緯は `docs/architecture.md` と `docs/v3/` に、通っていない検査は `docs/known-issues.md` に、隠さず書きました。

https://github.com/bigkijimon/bigkiji-universe

---

**環境**: Apple Silicon Mac / Node 24 / Electron 43。本体は Apache-2.0、依存はランタイム8つ。検査は61本（うち Linux は通過、Windows は未達 — 上記 known-issues 参照）。

**この記事のコード片は、すべて公開リポジトリの `a5b19be` 時点の実物です。** 引用元をそのまま貼っておくので、行数も含めて突き合わせられます。

- [`src/domain/pi-core/security/disclosure-manifest.js`](https://github.com/bigkijimon/bigkiji-universe/blob/a5b19bea0ad7c5003063082b4992ff58d6ab807a/src/domain/pi-core/security/disclosure-manifest.js) — マニフェスト生成と検証
- [`src/domain/pi-agent/task-runner.js`](https://github.com/bigkijimon/bigkiji-universe/blob/a5b19bea0ad7c5003063082b4992ff58d6ab807a/src/domain/pi-agent/task-runner.js) — 起動前の3つの陳腐化チェック
- [`src/domain/pi-agent/context-pruner.js`](https://github.com/bigkijimon/bigkiji-universe/blob/a5b19bea0ad7c5003063082b4992ff58d6ab807a/src/domain/pi-agent/context-pruner.js) — 既定値と±24行スライス
- [`docs/known-issues.md`](https://github.com/bigkijimon/bigkiji-universe/blob/a5b19bea0ad7c5003063082b4992ff58d6ab807a/docs/known-issues.md) — 通っていない検査

書いた人: **Uma**（[@bigkijimon](https://github.com/bigkijimon)）。Apple Silicon Mac 1台で、ローカルAIと外部CLIを併用した無人運転の仕組みを作っています。

