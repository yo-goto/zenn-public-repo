---
title: "Top-level await"
cssclass: zenn
date: 2022-05-28
modified: 2022-11-14
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『Top-level await』
---

# このチャプターについて

このチャプターでは、async/await の学習においてトラップとなる Top-level await について簡単に解説しておきます。

Top-level await は新しい機能(ES2022 で導入)であり、await 式が async 関数の外側で使えるようになります。ただし、初学者がこの機能について学んでしまうことで、**「同期」と「非同期」の概念が分からなくなってしまいます**(個人的な経験です)。したがって、async 関数について理解できてから学習するようにしてください。

# Top-level await とは

通常、await 式は async 関数内でのみしか利用できません(かつてはそうでした)。Top-level await の導入によってその制限は一部緩和されました。

まずは、次のような async 関数について考えてみます。

```js:async 関数
// simpleAsyncAwait.js
console.log("🦖 [1] MAINLINE: Start");
const url = "https://api.github.com/zen";

(async function () {
  console.log("🦖 [2] SYNC: In async function");
  try {
    const response = await fetch(url);
    // fetch() は成功か失敗に関わらず、リクエストに対する Response に解決する Promise インスタンスを返す
    const text = await response.text();
    // response.text() はレスポンスの本文をテキスト表現で解決する Promise インスタンスを返す
    console.log("👦 [4] MICRO: Github Philosophy>>", text);
  } catch (error) {
    console.log(error); 
  }
})();

console.log("🦖 [3] MAINLINE: End");
```

Top-level await では上のコードのように async 関数を定義することなく、ファイル直下に次のように書くことができます。これにより、Promise を返す非同期 API などから処理結果の値を簡単にとりだすことができます。

```js:Top-level await
// simpleTopLevelAwait.js
console.log("🦖 [A] MAINLINE: Start");
const url = "https://api.github.com/zen";

console.log("🦖 [B] MAINLINE: Middle");
try {
  const response = await fetch(url); // Respnose オブジェクトを取り出す
  const text = await response.text(); // テキストデータを取り出す
  console.log("👦 [C] MICRO: Github Philosophy>>", text);
} catch (error) {
  console.log(error); 
}

console.log("🦖 [D] MAINLINE: End");
```

ただし、Top-level await が使えるのは、JavaScript モジュール(ECMAScript モジュール)でのみなので注意してください。モジュールについての詳細は次の V8 や MDN のドキュメントを参照してください。

https://v8.dev/features/modules

https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Modules

Deno では最初から単一ファイルで Top-level await が使用できますが、Node ではバージョン v18.1.0 の時点では単一ファイルで使用すると `SyntaxError: await is only valid in async functions and the top level bodies of modules
` というエラーを吐き出します。ファイル拡張子を `.mjs` にしたり、`package.json` に `type: module` を追加することで利用できます。

# Top-level await の実行順序

Deno ランタイム環境において、上記２つのコードの実行順序を考えてみましょう。まず、async 関数の場合の `simpleAsyncAwait.js` は今までの知識で予測がつきます。`fetch()` でネットワーク接続をするので、Deno で実行するには `--allow-net` のフラグが必要になります。

```sh
❯ deno run --allow-net simpleAsyncAwait.js
🦖 [1] MAINLINE: Start
🦖 [2] SYNC: In async function
🦖 [3] MAINLINE: End
👦 [4] MICRO: Github Philosophy>> Avoid administrative distraction.
```

今まで通りですね。

一方、Top-level await の場合はこうなりません。実は、**Top-level await を使用しているファイル全体が１つの大きな async 関数のように機能します**。

https://v8.dev/features/top-level-await

>Top-level await enables developers to use the await keyword outside of async functions. **It acts like a big async function** causing other modules who import them to wait before they start evaluating their body.
>([上記ページ](https://v8.dev/features/top-level-await)より引用、太字は筆者強調)

:::message alert
Top-level await を使用したモジュール自体を `import` する他のモジュールはそれ自体のコードの評価を開始する前に待機することとなり、Top-level await が導入される前に比べて、モジュールの実行順序が複雑になります。
:::

というわけで、**このファイルのみを考えると**ファイル全体が async 関数と同じ様になるので、同期実行であった部分が async 関数内の処理と同じになります。

```js:Top-level await
// simpleTopLevelAwait.js
console.log("🦖 [A] MAINLINE: Start");
const url = "https://api.github.com/zen";

console.log("🦖 [B] MAINLINE: Middle");
try {
  const response = await fetch(url);
  const text = await response.text();
  console.log("👦 [C] MICRO: Github Philosophy>>", text);
} catch (error) {
  console.log(error); 
}

// このファイル全体が１つの大きな async 関数となるので await 式の評価が終わってから実行される
console.log("🦖 [D] MAINLINE: End");
```

そんなわけで、実行順序は次のようになります。

```sh
❯ deno run --allow-net simpleTopLevelAwait.js
🦖 [A] MAINLINE: Start
🦖 [B] MAINLINE: Middle
👦 [C] MICRO: Github Philosophy>> Responsive is better than fast.
🦖 [D] MAINLINE: End
```

Deno では Top-level await が何もせずに最初から使えてしまうため、この実行順序について非常に混乱しました。Top-level await を含む単一ファイルを実行すると、**await 処理が同期的に実行されて見えるため**(実際このファイル単体の実行でみれば同期的に実行されていると言える)、「await 式は同期実行される😵‍💫？」という混乱が個人的にありました。

async 関数の解説でも、たまに関数定義を書くのを省いて次のように書いてしまっている場合があります。初学者がこれを見ると非常に混乱するので気をつけてください。

```js
// top-level await か async/await で後の話が変わってくる
try {
  const response = await fetch(url);
  const text = await response.text();
  console.log("👦 [C] MICRO: Github Philosophy>>", text);
} catch (error) {
  console.log(error);
}
```

ファイル全体が大きな１つの async 関数のように振る舞うことを認識できれば同じように考えることができるのですが、これに気づかないと訳がわからなくなってしまいます。

# モジュールの実行順序への影響

ここで解説したように単一ファイル内での実行順序が変わるだけでなく、モジュールの処理順序が非常に複雑になりますので注意してください。Top-level await は思った以上に複雑な機能です。モジュールの実行順序へ与える影響については次の記事を参考にしてください。Top-level await が使えるメリットについても解説されています。

https://qiita.com/uhyo/items/0e2e9eaa30ec2ff05260

この内容については詳しく解説できるレベルの理解ではないので、注意書き程度にとどめておきます。非同期処理の学習に終わりはありません😱
