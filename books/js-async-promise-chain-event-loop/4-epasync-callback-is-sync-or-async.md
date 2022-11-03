---
title: "コールバック関数の同期実行と非同期実行"
cssclass: zenn
date: 2022-04-17
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: ch_コールバック関数の同期実行と非同期実行
---

# このチャプターについて

:::message alert
このチャプターでの古い内容に基づいた解説を新しい解説で置き換えました。実行コンテキストとマイクロタスクのチェックポイントを使って解説しています。
:::

このチャプターではコールバック関数の誤解しやすい点について解説しておきます。

# 同期か非同期か

『[Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)』のチャプターで Promise インスタンスの基本的な作成方法が分かったところで重要なことを解説します。

`new Promise(executor)` の `Promise()` コンストラクタ関数の引数として渡した `executor` 関数ですが、このコールバック関数は「**同期的に**」実行されます。次のコードでは完全に上から下へ順番にコードが実行されていきます。

```js:executorIsSync.js
// executorIsSync.js
console.log('🦖 [1] MAINLINE: Sync');

const promise = new Promise((resolve) => {
  console.log('👻 [2] これは同期的に実行される');
  resolve('🍎 解決値');
});

console.log("🦖 [3] MAINLINE: Sync");
```

ちなみに **"非同期処理"について考える時には、必ず"同期処理"と一緒に考えないと意味がない** ので、考えたい当該部分のコードを同期的に実行される `console.log()` で囲んでいます。

:::message
この本では、スクリプトの記述開始と終了ポイントには必ずコンソール出力する文字列に `MAINLINE` と書くようにしています。ちなみに `MAINLINE` は [IBM のチュートリアル](https://developer.ibm.com/tutorials/learn-nodejs-the-event-loop/) から取ってきた単語です。また、グローバルスコープで呼び出される `console.log()` でも `MAINLINE` の文字列をいれています。
:::

これを実行すると次のように出力されます。

```sh
❯ deno run executorIsSync.js
🦖 [1] MAINLINE: Sync
👻 [2] これは同期的に実行される
🦖 [3] MAINLINE: Sync
```

Promise は「**非同期処理の結果**を表現するビルトインオブジェクト」ですが、このように Promise コンストラクタに渡すコールバック関数は「**同期的に**」実行されます。つまり、完全に上から下へ行を移動するように実行されています。

今度は、上のコードに少し追加したものを考えてみます。「非同期処理」であるプロミスチェーン(Promise chain)です。

```js:thenCallbackIsAsync.js
// thenCallbackIsAsync.js
console.log('🦖 [1] MAINLINE: Sync');

const promise = new Promise((resolve) => {
  console.log('👻 [2] Sync');
  resolve('Resolved!');
});

// Promise chain
promise.then((value) => {
  console.log('👦 [4] Async');
  console.log('👦 [5] Resolved value:', value);
});

console.log('🦖 [3] MAINLINE: Sync');
```

さて、結果はコードに書いてあるのでもう分かっていると思いますが、これを実行すると次のような出力になります。

```sh
❯ deno run thenCallbackIsAsync.js
🦖 [1] MAINLINE: Sync
👻 [2] Sync
🦖 [3] MAINLINE: Sync
👦 [4] Async
👦 [5] Resolved value: Resolved!
```

Promise インスタンスは `then()`/`catch()`/`finally()` などの**プロトタイプメソッド**が使用できます。これによって、その Promise インスタンスの**状態が変化した後で**メソッドの引数として渡したコールスタック関数が「**非同期的に**」実行されることを保証できます。

今回の場合、`new Promise(executor)` で作成した Promise インスタンスである `promise` は、コールバック関数である `executor` が同期的に実行されて、すぐさま `resolve()` 関数に出会い実行されるので、ただちに `Promise` インスタンスの状態が履行(Fullfilled)状態になります。

コードの行を順番に下へ行くと `promise.then(cb)` に出会いますが、ここではコールバックである `cb` は Promise インスタンスが Fullfilled 状態になった時点でマイクロタスクキューへと送られます。この時点で Promise インスタンスである `promise` は履行(Fullfilled)状態なので、直ちにコールバック関数がマイクロタスクキューへと送られます。

しかし、マイクロタスクキューにあるこのコールバック関数はすぐに実行されません。コードの実行を考える上で、イベントループではスクリプトの評価によるすべての同期処理が最初のタスクとなり、その最中はコールスタック上に匿名のグローバルコンテキストが一番下に積まれている訳です。

コードの行をまた下に行くと、`console.log` に出会うので同期的にそれを実行します。この実行が終わった時点で、すべての同期処理が終わり、グローバルコンテキストがコールスタック上からポップします。これによってコールスタックが空になり、「**マイクロタスクのチェックポイント**」です。別の言い方では「**単一タスクが完了したら、すべてのマイクロタスクを処理する**」です。

というわけで、マイクロタスクキューにあるすべてのマイクロタスクを空にするまで処理します。

コールバック関数がマイクロタスクとして１つ発行されており、マイクロタスクキューには実行されるのを待っているマイクロタスクが１つあるので、直ちにそれを実行します。それによって、`"👦[4]Async"` がログに出力されて、その後に `"👦[5]Resolved value: Resolved!"` がログへ出力されます。

実際どのようにマイクロタスクが動くかを JS Visualizer 9000 で可視化してみたので以下のページから確認してみてください。

- [thenCallbackIsAsync.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gdGhlbkNhbGxiYWNrSXNBc3luYy5qcwpjb25zb2xlLmxvZygiWzFdIFN5bmMgcHJvY2VzcyIpOwoKY29uc3QgcHJvbWlzZSA9IG5ldyBQcm9taXNlKHJlc29sdmUgPT4gewogIGNvbnNvbGUubG9nKCJbMl0gVGhpcyBsaW5lIGlzIFN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICByZXNvbHZlKCJSZXNvbHZlZCEiKTsKfSk7Cgpwcm9taXNlLnRoZW4odmFsdWUgPT4gewogIGNvbnNvbGUubG9nKCJbNF0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7Cn0pOwoKY29uc29sZS5sb2coIlszXSBTeW5jIHByb2Nlc3MiKTsK)
- ⚠️ 注意: JS Visuzlizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク実行のタイミングについて誤解しないように注意してください

このように Promise chain において `.then()` メソッドのコールバックは Promise インスタンスがすでに履行(Fullfilled)状態であっても一旦はマイクロタスクキューへと送られてしまうので、どんなときでもそのコールバックの実行は非同期的になってしまいます。

まとめると、次の２つは対比的な実行となります。

- `Promise()` コンストラクタの引数として渡すコールバック関数(`executor`)は「**同期的に**」実行される
- `then()` メソッドの引数として渡すコールバック関数は「**非同期的に**」実行される

これに気付いていないと「Promise は同期的に実行される」とか「Promise chain は非同期的に実行される」とかの**言葉に惑わされて混乱する**ことになります。

# コールバック関数はいつ実行される?

このことについて、少し一般化して考えてみます。

そもそもコールバック関数の処理が同期的に行われるか、非同期的に行われるかというのはコールバック関数そのものの問題ではなく、そのコールバック関数を引数として受け取って使う方の問題です。

例えば、次のコードでは、コールバック関数として渡す `myFunc` はそれを引数として受け取る側である `syncCall()` 関数によって同期的に実行されます。

```js:whatIsCallbackFn-basic.js
// whatIsCallbackFn-basic.js
const myFunc = ([order, pattern, funcName]) => {
  console.log(`👻 ${order} This line is ${pattern} executed by ${funcName}`);
};

const syncCall = (callback, order) => {
  callback([order, "Synchronously", syncCall.name]);
};

console.log("🦖 [1] MAINLINE: Sync");
syncCall(myFunc, "[2]");
console.log("🦖 [3] MAINLINE: Sync");
```

ちなみに `myFunc` 関数の引数のところでは、「引数における配列の分割代入」を使用しています。また、関数内部ではテンプレートリテラルを使っていることに注意してください。

これを実行すると、コールバック関数は同期的に実行されていることが分かります。

```sh
❯ deno run whatIsCallbackFn-basic.js
🦖 [1] MAINLINE: Sync
👻 [2] Sync by syncCall
🦖 [3] MAINLINE: Sync
```

逆に、コールバック関数が非同期的に実行される場合ももちろんあります。この本ではしばらくの間 `setTimeout()` という非同期 API を使わないで説明する縛りをしていますが、次のように非同期 API にコールバック関数を渡す場合や、この後で説明する `then()` メソッドの引数としてコールバック関数を渡した場合は非同期に実行されます。

```js:whatIsCallbackFn.js
// whatIsCallbackFn.js
const myFunc = ([order, pattern, funcName]) => {
  console.log(`👻 ${order} This line is ${pattern} executed by ${funcName}`);
};

const syncCall = (callback, order) => {
  callback([order, "Synchronously", syncCall.name]);
};
const asyncAPICall = (callback, order) => {
  setTimeout(callback, 1000, [order, "Asynchrouously", asyncAPICall.name])
};
const thenCall = (callback, order) => {
  return Promise.resolve([order, "Asynchronously", thenCall.name])
    .then(callback);
};

console.log("🦖 [1] MAINLINE: Sync");
asyncAPICall(myFunc, "[7]");
console.log("🦖 [2] MAINLINE: Sync");
thenCall(myFunc, "[6]");
console.log("🦖 [3] MAINLINE: Sync");
syncCall(myFunc, "[4]");
console.log("🦖 [5] MAINLINE: Sync");
```

このコードを実行すると以下の出力を得ます。

```sh
❯ deno run whatIsCallbackFn.js
🦖 [1] MAINLINE: Sync
🦖 [2] MAINLINE: Sync
🦖 [3] MAINLINE: Sync
👻 [4] Sync by syncCall
🦖 [5] MAINLINE: Sync
👻 [6] Async by thenCall
👻 [7] This line is Asynchrouously executed by asyncAPICall
```

詳細について今は理解できなくても大丈夫です。とにかく「**コールバック関数がいつ実行されるかはコールバック関数を受け取る方の問題**」であるということを認識しておきましょう。

コールバック関数についての基礎は mdn のドキュメントで確認してください。

https://developer.mozilla.org/ja/docs/Glossary/Callback_function
