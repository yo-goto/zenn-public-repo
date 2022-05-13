---
title: "コールバック関数の同期実行と非同期実行"
---

:::message alert
このチャプターの内容は『イベントループの概要と注意点』のチャプターに基づいた古いものであり、イベントループの説明自体は間違っていませんが、分かりづらい部分があるので注意してください。
:::

# 同期か非同期か
では、Promise インスタンスの基本的な作成方法が分かったところで重要なことを解説します。

`new Promise(executor)` の `Promise()` コンストラクタ関数の引数として渡した `executor` 関数ですが、このコールバック関数は「**同期的に**」実行されます。

```js:executorIsSync.js
console.log("[1] Sync process");

const promise = new Promise(resolve => {
  console.log("[2] これは同期的に実行されます");
  resolve("解決値");
});

console.log("[3] Sync process");
```

ちなみに **"非同期処理"について考える時には、必ず"同期処理"と一緒に考えないと意味がない** ので、同期的に実行される `console.log()` で囲んでいます。

これを実行すると次のように出力されます。

```sh
❯ deno run executorIsSync.js
[1] Sync process
[2] これは同期的に実行されます
[3] Sync process
```

Promise は「**非同期処理の結果**を表現するビルトインオブジェクト」ですが、このように Promise コンストラクタに渡すコールバック関数は「**同期的に**」実行されます。つまり、完全に上から下へ行を移動するように実行されています。

今度は、上のコードに少し追加したものを考えてみます。「非同期処理」である Promise チェーンです。

```js:thenCallbackIsAsync.js
// thenCallbackIsAsync.js
console.log("[1] Sync process");

const promise = new Promise(resolve => {
  console.log("[2] This line is Synchronously executed");
  resolve("Resolved!");
});

promise.then(value => {
  console.log("[4] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[3] Sync process");
```

さて、結果はコードに書いてあるのでもう分かっていると思いますが、これを実行すると次のような出力になります。

```sh
❯ deno run thenCallbackIsAsync.js
[1] Sync process
[2] This line is Synchronously executed
[3] Sync process
[4] This line is Asynchronously executed
Resolved value:  Resolved!
```

Promise インスタンスは `then()` と `catch()` と `finally()` などの**プロトタイプメソッド**が使用できます。これによって、その Promise インスタンスの**状態が変化した後で**メソッドの引数として渡したコールスタック関数が「**非同期的に**」実行されることを保証できます。

今回の場合、`new Promise(executor)` で作成した Promise インスタンスである `promise` は、コールバック関数である `executor` が同期的に実行されて、すぐさま `resolve()` 関数にであい実行されるので、ただちに `Promise` インスタンスの状態が履行(Fullfilled)状態になります。

コードの行を順番に下に行くと `promise.then(cb)` に出会いますが、ここではコールバックである `cb` は Promise インスタンスが Fullfilled 状態になった時点で Microtask queue へと送られます。この時点で Promise インスタンスである `promise` は履行(Fullfilled)状態なので、直ちにコールバック関数が Microtask queue へと送られます。

しかし、Microtask queue にあるこのコールバック関数はすぐに実行されません。Event Loop ではまず Call stack が完全に空になるまで同期的に実行が続きます(Event loop のステップ１「スクリプトの評価」とステップ２「単一の Task(Mcarotask)の実行」)。

コードの行をまた下に行くと、`console.log` に出会うので同期的にそれを実行します。この実行が終わった時点で Call stack に積むものは何もなく完全に空の状態になったので、Event Loop が次のステップへと移行して Microtask queue に存在しているマイクロタスクをすべて実行します。

:::message alert
Event loop のステップ１はステップ２と同質のものであり、「スクリプトの評価」は実質的に Task(Macrotask) として扱われるので、これが終わると、Event loop は次のステップ３「すべての Microtask の実行」へと移行します。

詳しくは、[Event loop の概要と注意点](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/2-epasync-event-loop) のチャプターを確認してください。
:::

マイクロタスクは現在１つあるので直ちにそれを実行します。それによって、"[4]This line is Asynchronously executed" がログに出力されて、その後に "Resolved value:  Resolved!" がログに出力されます。

実際にどのようにマイクロタスクが動くかを JS Visualizer 9000 で可視化してみたので以下のページから確認してみてください。

- [thenCallbackIsAsync.js - JS Visualizer 9000](https://www.jsv9000.app/?code=Ly8gdGhlbkNhbGxiYWNrSXNBc3luYy5qcwpjb25zb2xlLmxvZygiWzFdIFN5bmMgcHJvY2VzcyIpOwoKY29uc3QgcHJvbWlzZSA9IG5ldyBQcm9taXNlKHJlc29sdmUgPT4gewogIGNvbnNvbGUubG9nKCJbMl0gVGhpcyBsaW5lIGlzIFN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICByZXNvbHZlKCJSZXNvbHZlZCEiKTsKfSk7Cgpwcm9taXNlLnRoZW4odmFsdWUgPT4gewogIGNvbnNvbGUubG9nKCJbNF0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7Cn0pOwoKY29uc29sZS5sb2coIlszXSBTeW5jIHByb2Nlc3MiKTsK)

このように Promise チェーンにおいて `.then()` メソッドのコールバックは Promise インスタンスがすでに履行(Fullfilled)状態であっても一旦は Microtask queue へと送られてしまうので、どんなときでもそのコールバックの実行は非同期的になってしまいます。

まとめると、次の２つは対比的な実行となります。

- `Promise()` コンストラクタの引数として渡すコールバック関数(`executor`)は「**同期的に**」実行される
- `then()` メソッドの引数として渡すコールバック関数は「**非同期的に**」実行される

これに気付いていないと「Promise は同期的に実行される」とか「Promise チェーンは非同期的に実行される」とかの**言葉に惑わされて混乱する**ことになります。

# コールバック関数はいつ実行される?
このことについて、少し一般化して考えてみます。

そもそもコールバック関数の処理が同期的に行われるか、非同期的に行われるかというのはコールバック関数そのものの問題ではなく、そのコールバック関数を引数として受け取って使う方の問題です。

例えば、次のコードでは、コールバック関数として渡す `myFunc` はそれを引数として受け取る側である `syncCall()` 関数によって同期的に実行されます。

```js
// wahtIsCallbackFn-basic.js
const myFunc = ([order, pattern, funcName]) => {
  console.log(`${order} This line is ${pattern} executed by ${funcName}`);
};

const syncCall = (callback, order) => {
  callback([order, "Synchronously", syncCall.name]);
};

console.log("[1] Sync process");
syncCall(myFunc, "[2]");
console.log("[3] Sync process");
```

ちなみに `myFunc` 関数の引数のところでは、「引数における配列の分割代入」を使用しています。また、関数内部ではテンプレートリテラルを使っていることに注意してください。

これを実行すると、コールバック関数は同期的に実行されていることが分かります。

```sh
❯ deno run whatIsCallbackFn-basic.js
[1] Sync process
[2] This line is Synchronously executed by syncCall
[3] Sync process
```

逆に、コールバック関数が非同期的に実行される場合ももちろんあります。この本ではしばらくの間 `setTimeout()` という非同期 API を使わないで説明する縛りをしていますが、次のように非同期 API にコールバック関数を渡す場合や、この後で説明する `then()` メソッドの引数としてコールバック関数を渡した場合は非同期に実行されます。

```js
// wahtIsCallbackFn.js
const myFunc = ([order, pattern, funcName]) => {
  console.log(`${order} This line is ${pattern} executed by ${funcName}`);
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

console.log("[1] Sync process");
asyncAPICall(myFunc, "[7]");
console.log("[2] Sync process");
thenCall(myFunc, "[6]");
console.log("[3] Sync process");
syncCall(myFunc, "[4]");
console.log("[5] Sync process");
```

このコードを実行すると以下の出力を得ます。

```sh
❯ deno run whatIsCallbackFn.js
[1] Sync process
[2] Sync process
[3] Sync process
[4] This line is Synchronously executed by syncCall
[5] Sync process
[6] This line is Asynchronously executed by thenCall
[7] This line is Asynchrouously executed by asyncAPICall
```

詳細について今は理解できなくても大丈夫です。とにかく「**コールバック関数がいつ実行されるかはコールバック関数を受け取る方の問題**」であるということを認識しておきましょう。

コールバック関数についての基礎は mdn のドキュメントで確認してください。

https://developer.mozilla.org/ja/docs/Glossary/Callback_function

