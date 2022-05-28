---
title: "Top-level await"
---

# このチャプターについて

このチャプターでは、async/await の学習においてトラップとなる Top-level await について簡単に解説します。

Top-level await は比較的新しい機能であり、await 式が非同期関数(async funciton)の外側で使えるようになります。ただし、初学者がこの機能について学んでしまうことで、**「同期」と「非同期」の概念が分からなくなってしまいます**(個人的な経験です)。したがって、非同期関数について理解できてから学習するようにしてください。

# Top-level await とは

通常、await 式は非同期関数(async function)内でのみしか利用できません(かつてはそうでした)。Top-level await の導入によってその制限は一部緩和されました。

```js:非同期関数
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

Top-level await では非同期関数を定義することなく、ファイル直下に次のように書くことができます。これにより、Promise を返す非同期 API などから処理結果の値を簡単にとりだすことができます。

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

console.log("🦖 [D] MAINLINE: End");
```

ただし、Top-level await は使えるのは、JavaScritpt モジュール(ECMAScript モジュール)でのみなので注意してください。モジュールについての詳細は次の V8 のドキュメントを参照してください。

https://v8.dev/features/modules

Deno では最初から単一ファイルで Top-level await が使用できますが、Node ではバージョン v18.1.0 の時点では単一ファイルで使用すると `SyntaxError: await is only valid in async functions and the top level bodies of modules
` というエラーを吐き出します。ファイル拡張子を `.mjs` にしたり、`package.json` に `type: module` を追加することで利用できます。

# Top-level await の実行順序

Deno ランタイム環境において、上記２つのコードの実行順序を考えてみましょう。まず、非同期関数の場合の `simpleAsyncAwait.js` は今までの知識で予測がつきます。`fetch()` でネットワーク接続をするので、Deno で実行するには `--allow-net` のフラグが必要になります。

```sh
❯ deno run --allow-net simpleAsyncAwait.js
🦖 [1] MAINLINE: Start
🦖 [2] SYNC: In async function
🦖 [3] MAINLINE: End
👦 [4] MICRO: Github Philosophy>> Avoid administrative distraction.
```

今まで通りですね。

一方、Top-level await の場合はそうはいきません。**Top-level await を使用しているファイル全体が１つの大きな非同期関数のように機能します**。

https://v8.dev/features/top-level-await

:::message alert
さらに、このモジュール自体を `import` する他のモジュールがそれ自体のコードの評価を開始する前に待機することになり、Top-level await が導入される前に比べて、モジュールの実行順序が複雑になります。
:::

というわけで、**このファイルのみを考えると**ファイル全体が非同期関数と同じ様になるので、同期実行であった部分が非同期関数内の処理と同じになります。

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

// このファイル全体が１つの大きな非同期関数となるので await 式の評価が終わってから実行される
console.log("🦖 [D] MAINLINE: End");
```

というわけで、実行順序は次のようになります。

```sh
❯ deno run --allow-net simpleTopLevelAwait.js
🦖 [A] MAINLINE: Start
🦖 [B] MAINLINE: Middle
👦 [C] MICRO: Github Philosophy>> Responsive is better than fast.
🦖 [D] MAINLINE: End
```

Deno では Top-level await が何もせずに最初から使えてしまうため、この実行順序について非常に混乱しました。Top-level await を含む単一ファイルを実行すると、**同期的に実行されているように見えるため**、「await 式は同期実行される🫠？」という混乱が実際にありました。

非同期関数の解説でも、たまに Async function を書くのを省いて次のように書いてしまっている場合があります。解説者本人は分かっているからよいかもしれませんが、初学者がこれを見ると非常に混乱しますので、絶対に async function の定義を省いてはいけないと考えています。

```js
// top-level await か async/await で話が変わってくる
try {
  const response = await fetch(url);
  const text = await response.text();
  console.log("👦 [C] MICRO: Github Philosophy>>", text);
} catch (error) {
  console.log(error); 
}
```

# モジュールの実行順序への影響

ここで解説したように単一ファイル内での実行順序が変わるだけでなく、モジュールの処理順序が非常に複雑になりますので注意してください。Top-level await は思った以上に複雑な機能です。モジュールの実行順序へ与える影響については次の記事を参考にしてください。Top-level await が使えるメリットについても解説されています。

https://qiita.com/uhyo/items/0e2e9eaa30ec2ff05260

この内容については詳しく解説できるレベルの理解ではないので、注意書き程度にとどめておきます。非同期処理の学習に終わりはありません😱

