---
title: "Promise chain と async/await の仕様比較"
cssclass: zenn
date: 2023-01-09
modified: 2023-01-09
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases:
  - "Promise本『Promise chain と async/await の仕様比較』"
---

# このチャプターについて

このチャプターでは、async/await と Promise chain の仕様や挙動の比較を行い、今後の Promise の ECMAScrip 仕様の発展などについて解説していきます。

# 仕様最適化の遺構

## async/await の最適化

async/await と Promise chain でマイクロタスクの発生数が異なるという事象が起きますが、`resolution` として使われる Thenable の話が関与しています。

『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)』のチャプターで解説していますが、async/await は V8 エンジンサイドからの以下の PR で仕様自体の最適化がなされました。

https://github.com/tc39/ecma262/pull/1250/files

この最適化の中枢となる変更は以下の箇所です。

```diff
- 1. Let _promiseCapability_ be ! NewPromiseCapability(%Promise%).
+ 1. Let _promise_ be ? PromiseResolve(%Promise%, &laquo; _value_ &raquo;).
- 2. Perform ! Call(_promiseCapability_.[[Resolve]], *undefined*, &laquo; _promise_ &raquo;).
```

この箇所で何が起きたかを説明すると、[NewPromiseCapability](https://tc39.es/ecma262/#sec-newpromisecapability) 抽象操作と [Promise Resolve Function](https://tc39.es/ecma262/#sec-promise-resolve-functions) の呼び出しがなくなり、 [PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve) 抽象操作がここに挿入されました。

`await thenable` という処理があったときには、 [PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve) 操作で await 式の評価対象が Promise 以外の Thenable の場合だと一旦 Promise でラップすることになります (これは通常の値を評価するときとまったく同じです)。ただし、Promise オブジェクトだけは特別扱いして **そのまま返す** ようになりました。実際の PromiseResolve 操作の仕様のステップが以下の部分です。

> - 1. If [IsPromise](https://tc39.es/ecma262/#sec-ispromise)(x) is true, then
>   - a. Let xConstructor be ? [Get](https://tc39.es/ecma262/#sec-get-o-p)(x, "constructor").
>   - b. If [SameValue](https://tc39.es/ecma262/#sec-samevalue)(xConstructor, C) is true, return x.

PromiseResolve 操作から呼び出される [IsPromise](https://tc39.es/ecma262/#sec-ispromise) という抽象操作の判定によって Promise オブジェクトかどうかを判定しています。この操作では以下のアルゴリズムステップで引数 `x` がオブジェクトでなかったり、`[[PromiseState]]` という内部スロットを持たなければ `false` 値を返して Promise オブジェクトでないことを判定します。

> - 1. If x [is not an Object](https://tc39.es/ecma262/#sec-object-type), return false.
> - 2. If x does not have a \[\[PromiseState\]\] internal slot, return false.
> - 3. Return true.

結局、仕様の最適化は async 関数の await 式の評価で Promise を Thenable 全体から引き離して、無駄な処理を削減するようにしたことが大きいです。

## Promise.prototype.then の未最適化部分

しかし、その一方で `Promise.prototype.then` の仕様ではコールバックから返される値が Promise の場合を特別扱いしていません。

`return thenable` という処理が `then()` メソッドのコールバック関数であると、Thenable が持つ `then` メソッドが実行されて解決されるまで、その `then()` メソッドから返る Promise オブジェクトが解決できないので、その前に [Promise Resolve Function](https://tc39.es/ecma262/#sec-promise-resolve-functions) が起動して、[NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) が実行されてマイクロタスクが増加することになります。一方 async/await では `await thenable` での Thenable が Promise である場合にはそもそも NewPromiseResolveThenableJob 操作が発生しません (※ Promise 以外の Thenable だと発生しますし、async 関数本体の最後で `return thenable` 処理がある時も発生します)。最適化前の仕様では `await promise` という処理があれば `then` メソッドのコールバックで `return promise` した場合と同じく常に NewPromiseResolveThenableJob が実行されて追加のマイクロタスクが発生していましたが、このための無駄な Promise のラッピングとそれを解決するために発生する PromiseResolveThenableJob は ResolvePromise 操作を使うようにした最適化で削減されました。

つまり、`Promise.prototype.then` の方の仕様は async/await であったような最適化がされていないので、コールバック関数で Promise を返している場合には async/await で発生するマイクロタスクよりも多くのマイクロタスクが発生してしまうことになります。

## async/await と Promise chain の比較

『[Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)』のチャプターで「`Promise.resolve` と `executor` 関数の `resolve` 関数は同じようなものであるが、完全に等価ではない」と述べました。`resolve` は引数 `resolution` に Promise を取るとマイクロタスクが追加発生する一方で、`Promise.resolve` は引数が Promise だとそのまま返します。この違いによって２つを競争させたときには `Promise.resolve` を使った方がマイクロタスクの発生が少ないため先に終了できます。

```js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: どちらの promise chain かの識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
new Promise(resolve => {
  resolve(Promise.resolve("A"));
  // 🔥 引数が Promise なら追加で２つのマイクロタスクが発生
  // <1-a[1]> Promise.reoslve("A").then(resolve, reject) の呼び出し
  // ↪ <3-a[2]> resolve 関数の実行
}).then(console.log); // <4-a[3]>
//      ^^^^^^^^^^^ ３個目のマイクロタスクで出力

// こちらが先に終了する
Promise.resolve(Promise.resolve("B")) // 引数が Promise ならそのまま返す
  .then(console.log); // <2-b[1]>
  //   ^^^^^^^^^^^^ １個目のマイクロタスクで出力

/* RESULT
❯ deno run pResolveExeResolve.js
B
A
*/
```

Promise chain と async/await の違いはこのような `resolve` 関数と `Promise.resolve` 関数のどちらを使用するかという話に帰結します。最適化の結果として `Promise.resolve` で使用されている PromiseResolve 操作が async/await で使われるようになったのでマイクロタスクが減ったわけです。

:::message
実際には `Promise.resolve` 関数は[PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve) 抽象操作の以下のステップで `resolve` 関数の呼び出しを行っているため、`resolve` 関数を内部で利用している特殊なラッパーと言えます。

> 3. Perform ? [Call](https://tc39.es/ecma262/#sec-call)(promiseCapability.\[\[Resolve\]\], undefined, « x »).
:::

例えば、以下のようにまったく同じ処理を Promise chain と async/await で記述したときには仕様が最適化されている async/await の方が先に終了します。

```js:countMt.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: promise chain (p) か async/await (a) か
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [1] G: sync");

// 合計で4つのマイクロタスクが発生
Promise.resolve(1)
  .then((x) => { // <1-p[1]>
    console.log("💙 [3] P: async", x);
    return Promise.resolve(2);
    // 🔥 promise を返すので追加のマイクロタスクが２個発生
    // <3-p[2]> Promise.resolve(2).then(resolve, reject) の呼び出し
    // ↪ <5-p[3]> resolve 関数の実行
  })
  .then((y) => { // <6-p[4]>
    console.log("💙 [6] P: async", y);
  });

// 合計で２つのマイクロタスクが発生
(async () => {
  const x = await Promise.resolve(1);
  // <2-a[1]>
  console.log("💚 [4] A: async", x);
  const y = await Promise.resolve(2);
  // <4-a[2]>
  console.log("💚 [5] A: async", y);
})();

console.log("🦖 [2] G: sync");

/* RESULT
❯ deno run countMt.js
🦖 [1] G: sync
🦖 [2] G: sync
💙 [3] P: async 1
💚 [4] A: async 1
💚 [5] A: async 2
💙 [6] P: async 2
*/
```

[Promise chain のネストをフラット化する弊害](#promise-chain-のネストをフラット化する弊害) の項目で見たように、ネストをそのままにすれば以下のように出力順番を調整できますが、結局発生しているマイクロタスクの合計では async/await よりも Promise chain の方が多くなります。

```js:countMtX.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: promise chain (p) か async/await (a) か
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [1] G: sync");

// 合計で4つのマイクロタスクが発生
Promise.resolve(1)
  .then((x) => { // <1-p[1]>
    console.log("💙 [3] P: async", x);
    return Promise.resolve(2)
      .then((y) => { // <3-p[2]>
        console.log("💙 [5] P: async", y);1
      });
    // 🔥 promise を返すので追加のマイクロタスクが２個発生
    // <4-p[3]> promise.then(resolve, reject) の呼び出し
    // ↪ <6-p[4]> resolve 関数の実行
  });

// 合計で2つのマイクロタスクが発生
(async () => {
  const x = await Promise.resolve(1);
  // <2-a[1]>
  console.log("💚 [4] A: async", x);
  const y = await Promise.resolve(2);
  // <5-a[2]>
  console.log("💚 [6] A: async", y);
})();

console.log("🦖 [2] G: sync");

/* RESULT
❯ deno run countMtX.js
🦖 [1] G: sync
🦖 [2] G: sync
💙 [3] P: async 1
💚 [4] A: async 1
💙 [5] P: async 2
💚 [6] A: async 2
*/
```

やってることの意味合いは同じですが、発生するマイクロタスクの数が異なることからも Promise chain と async/await は厳密にはシンタックスシュガーではないということが分かります。

## 根本的な仕様最適化のプロポーザル

現在 Promise 関連で追加発生する余計なマイクロタスクは Thenable のための [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) 操作に集約されるといっても過言ではありません。

NewPromiseResolveThenableJob 操作は [Promise Resolve Function](https://tc39.es/ecma262/#sec-promise-resolve-functions) 操作である `resolve` 関数から呼び出されるので、この仕様から NewPromiseResolveThenableJob 操作をうまく除去できれば、この操作に依存している async 関数本体の `return promise` や `Promise.prototype.then` において追加発生するマイクロタスクを１つ除去できるはずです。

:::message
async 関数内部で `return promise` という処理があると２つの追加のマイクロタスクが発生してしまうことは、『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting#return-promiseresolve42-の場合)』のチャプターで解説しています。
:::

そして、それに対しての抜本的な最適化を行うようにする仕様プロポーザルが以下となります。

https://github.com/tc39/proposal-faster-promise-adoption

このプロポーザル自体がかなり最近作成されたもので、まだ [Stage 1](https://tc39.es/process-document/) なので時間はかかりますが、これがマージされれば、このチャプターなど今まで理解に苦しめられていた追加のマイクロタスクの発生が軽減されます。
