---
title: "複数の Promise を走らせる"
---

非同期処理で最も重要なことは「制御の流れ」がつかめるようになることです。実行の順番を予測できるようになるために 1 つずつ処理を複雑にして Promise チェーンを解説していきたいと思います。

ここまで `new Promise(executor)` で Promise インスタンスを作成してきました。今度は Promise インスタンスを返す関数をアロー関数式を使用して定義して先程のコードを改造してみます。これで Promise インスタンスを何回も作成できるようになります。また引数を渡せるようにしてその引数で Promise を解決するようにします。

```js:returnPromiseByFunc.js
console.log("[1] Sync process");

const returnPromise = (resolveValue) => {
  return new Promise(resolve => {
    console.log("[2] This line is Synchronously executed");
    resolve(resolveValue);
  });
};

returnPromise("Resolved by function").then(value => {
  console.log("[4] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[3] Sync process");
```

これは前のコードではそのまま Promise インスタンスを作成していたのを Promise インスタンスを返す関数に置き換えただけなので実行結果は以前と同じになります(解決値だけ違う)。

```sh
❯ deno run returnPromiseByFunc.js
[1] Sync process
[2] This line is Synchronously executed
[3] Sync process
[4] This line is Asynchronously executed
Resolved value:  Resolved by function
```

具体的には `returnPromise()` 関数は同期的に実行されて、内部の `Promise()` コンストラクタの引数であるコールバック関数もそのまま同期的に実行されます。

さて、それではこの `returnPromise()` 関数を使用して複数の Promise インスタンスを作成してみます。`reutrnPromise()` 関数は何度も起動させたいので、`[2]` 番目に出力される行を書き換えて引数で指定できるようにします。テンプレートリテラルで表現します。

```js:returnPromiseByFuncArg.js
// returnPromiseByFuncArg.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("First Promise", "[2]").then((value) => {
  console.log("[4] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[3] Sync process");
```

テンプレートリテラルで書き換えただけなので実行結果は同じになります。

```sh
❯ deno run returnPromiseByFuncArg.js
[1] Sync process
[2] This line is Synchronously executed
[3] Sync process
[4] This line is Asynchronously executed
Resolved value:  First Promise
```

それでは準備ができたのでが実際に複数の Promise インスタンスを作成してみて実行の順番がどうなるかを見てみます。`[]` で囲まれた文字の順番がどのように出力されるか、ここからは自分で出力の順番を予想してみてください。

```js:returnPromiseByFuncArg2.js
// returnPromiseByFuncArg2.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[B]").then((value) => {
  console.log("[C] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});
returnPromise("2nd Promise", "[D]").then((value) => {
  console.log("[E] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[F] Sync process");
```

実行順番がどうなるか分かりましたか?

:::details 答え
答えは、「A → B → D → F → C → E」となります。

```sh
❯ deno run returnPromiseByFuncArg2.js
[A] Sync process
[B] This line is Synchronously executed
[D] This line is Synchronously executed
[F] Sync process
[C] This line is Asynchronously executed
Resolved value:  1st Promise
[E] This line is Asynchronously executed
Resolved value:  2nd Promise
```
:::

なぜこうなるのか考えてみます。

まずコードは上から下に実行されていきます。Event Loop では最初のステップ１で同期処理すべて実行されていきます。`returnPromise("1st Promise", "B")` は**同期処理**です。関数の中を見ても、`Promise()` コンストラクタの引数である `executor` 関数の中も同期的に実行されます。

:::message alert
Event loop のステップ１はステップ２と同質のものであり、「スクリプトの評価」は実質的に Task(Macrotask) として扱われるので、これが終わると、Event loop は次のステップ３「すべての Microtask の実行」へと移行します。

詳しくは、[Event loop の概要と注意点](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/epasync-event-loop) のチャプターを確認してください。
:::

`executor` 関数内ですぐに `resolve()` が呼び出されるので Promise インスタンスは直ちに履行(Fullfilled)状態へと移行します。`returnPromise("1st Promise", "B")` でここまでは同期的に実行されていることに注意してください。

`returnPromise("1st Promise", "B")` で返ってくる Promise インスタンスはすでに履行状態なので、直ちに `then()` メソッドの引数であるコールバック関数が Microtask queue へと送られます。しかし、そのコールバックはまだ実行されません。Event Loop の今のステップでは、Call Stack に積まれるものが完全になくなるまで、スクリプトを評価して同期的処理をすべて実行します。

ということで次に実行される同期処理は  `returnPromise("2nd Promise", "D")` となります。これも**同期処理**です。先ほどと同じように関数内の `Promise()` コンストラクタの引数である `executor` 関数の中も同期的に実行されます。

また同じように `returnPromise("2nd Promise", "D")` で返ってくる Promise インスタンスはもすでに履行状態なので、直ちに `then()` メソッドの引数であるコールバック関数が Microtask queue へと送られます。もちろんこのコールバックもまだ同期処理が残っているため、まだ実行されません。

最後の同期処理である `console.log("[F] Sync process");` が次に実行されます。

ここまでで出力されるログは次のようになっていることを確認してください。

```sh
❯ deno run returnPromiseByFuncArg2.js
[A] Sync process
[B] This line is Synchronously executed
[D] This line is Synchronously executed
[F] Sync process

# ...この先はどうなる?
```

同期処理の実行がすべて完了したので、Event Loop は次のステップへと移行します。~~「同期処理の実行」の次は「Macrotask queue にある Macrotask の実行」を行います。しかし、Macrotask を作成するような処理は行っていないので Macrotask queue にタスクは存在していません。従って、Event Loop は再び次のステップへと移行します。~~

:::message alert
Event loop のステップ１はステップ２と同質のものであり、「スクリプトの評価」は実質的に Task(Macrotask) として扱われるので、これが終わると、Event loop は次のステップ３「すべての Microtask の実行」へと移行します。

詳しくは、[Event loop の概要と注意点](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/epasync-event-loop) のチャプターを確認してください。
:::

最初の Task(Macrotask) の実行が終わり、Event loop は次のステップ「Microtask queue にあるすべての Microtask の実行」(ステップ４)を行います。

Microtask queue はキューなので一番古いタスク(先に入れられたタスク)から実行していきます。最初にキューへと追加されたのは `returnPromise("1st Promise", "B").then()` のコールバック関数です。従って、出力の続きは次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2.js
[A] Sync process
[B] This line is Synchronously executed
[D] This line is Synchronously executed
[F] Sync process
[C] This line is Asynchronously executed
Resolved value:  1st Promise

# ...この先はどうなる?
```

そして、次にキューに追加されたのは `returnPromise("2nd Promise", "D").then()` のコールバック関数でした。従ってそのコールバック関数が実行されることで結局出力は次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2.js
[A] Sync process
[B] This line is Synchronously executed
[D] This line is Synchronously executed
[F] Sync process
[C] This line is Asynchronously executed
Resolved value:  1st Promise
[E] This line is Asynchronously executed
Resolved value:  2nd Promise
```

順番をアルファベットから数字に直してみるとこのようになります。

```js
// doubleThenCallback.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[2]").then((value) => {
  console.log("[5] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});
returnPromise("2nd Promise", "[3]").then((value) => {
  console.log("[6] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[4] Sync process");
```

↓ JS Visuzalizer 9000 で実際に可視化してみたので確認してくみてください。

- [doubleThenCallback.js - JS Visuzalizer 9000](https://www.jsv9000.app/?code=Ly8gZG91YmxlVGhlbkNhbGxiYWNrLmpzCmNvbnNvbGUubG9nKCdbMV0gU3luYyBwcm9jZXNzJyk7Cgpjb25zdCByZXR1cm5Qcm9taXNlID0gKHJlc29sdmVkVmFsdWUsIG9yZGVyKSA9PiB7CiAgcmV0dXJuIG5ldyBQcm9taXNlKChyZXNvbHZlKSA9PiB7CiAgICBjb25zb2xlLmxvZyhgWyR7b3JkZXJ9XSBUaGlzIGxpbmUgaXMgU3luY2hyb25vdXNseSBleGVjdXRlZGApOwogICAgcmVzb2x2ZShyZXNvbHZlZFZhbHVlKTsKICB9KTsKfTsKcmV0dXJuUHJvbWlzZSgnMXN0IFByb21pc2UnLCAnMicpLnRoZW4oKHZhbHVlKSA9PiB7CiAgY29uc29sZS5sb2coJ1s1XSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQnKTsKICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlKTsKfSk7CnJldHVyblByb21pc2UoJzJuZCBQcm9taXNlJywgJzMnKS50aGVuKCh2YWx1ZSkgPT4gewogIGNvbnNvbGUubG9nKCdbNl0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7Cn0pOwoKY29uc29sZS5sb2coJ1s0XSBTeW5jIHByb2Nlc3MnKTsK)

