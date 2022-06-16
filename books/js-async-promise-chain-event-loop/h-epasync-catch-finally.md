---
title: "catch メソッドと finally メソッド"
aliases: [ch_catch メソッドと finally メソッド]
---

# このチャプターについて
このチャプターでは今まで解説していなかった `catch()` メソッド、`finally()` メソッドについて、そしてそれらメソッドが発行するマイクロタスクについて解説していきたいと思います。

# Promise のプロトタイプメソッド
`then()` 以外の Promise のプロトタイプメソッドとして `catch()` と `finally()` メソッドが挙げられます。これは try/catch/finally に対応しており、同様の考え方で使うことができます。

```js
new Promise((resolve, reject) => {
  if (Math.random() < 0.5) {
    resolve(42);
  } else {
    reject(new Error("例外発生"));
  }
}).then(data => console.log(data))
  .catch(err => console.log(err))
  .finally(() => console.log("最後に実行される"));
```

上のように `reject()` 関数によって Promise インスタンスが Rejected 状態になった場合、チェインしている `then()` メソッドのコールバックは実行されずに、`catch()` メソッドのコールバックが実行されて例外を捕捉します。

逆に、`resolve()` 関数によって Promise インスタンスが Fullfilled 状態に場合には、`catch()` メソッドのコールバックは実行されません。

一方、`finally()` メソッドは Promise インスタンスが Fullfilled 状態であろうと Rejected 状態であろうと登録しているコールバックが実行されます。ただし、`finally()` メソッドの**コールバック関数は一切引数をとらない**ので、チェーンで値を繋ぐことはできません。

```js
// catchFinally.js
new Promise((resolve, reject) => {
  if (Math.random() < 0.5) {
    resolve(42);
  } else {
    reject(new Error("例外発生"));
  }
})
  .then((data) => console.log("🤟 データ:", data))
  .catch((err) => console.log("👹 エラー:", err))
  .then(() => {
    console.log("👻 これは実行される");
    return 42;
  })
  .finally((data) => {
    console.log("😭 データ:", data); // undefined
    console.log("👦 最後に実行される");
  });
```

`then()` メソッドからは常に新しい Promise インスタンスが返ってきたように、`catch()` メソッドと `finall()` メソッドでも新しい Promise インスタンスが返ってきますので、チェーンできます。

言ったように、`finally()` メソッドに値は繋げませんので、上のコードを実行すると `undefined` が出力されます。

```sh
❯ v8 catchFinally.js
👹 エラー: Error: 例外発生
👻 これは実行される
😭 データ: undefined
👦 最後に実行される
❯ v8 catchFinally.js
🤟 データ: 42
👻 これは実行される
😭 データ: undefined
👦 最後に実行される
```

`catch()` メソッドによって返ってくる Promise インスタンスは履行状態で返ってきますので、次の `then()` メソッドのコールバックを実行できます。

# コールバックは実行されなくてもマイクロタスクは発生する

重要なこととして、`catch()` メソッドや `then()` メソッドはコールバックが実行されないときでも実はマイクロタスクが発行されます。

例えば、`Promise.reject()` で拒否状態の Promise インスタンスに `then()` と `catch()` メソッドをチェーンしてみます。

実行順番はどうなるでしょうか?

```js
// catchMicrotask-1.js
console.log("🦖 [A-1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [B-3] <1-Sync> MICRO: then"));

Promise.reject(new Error("Exception"))
  .then(() => console.log("👻 [C-4] <2-Sync> MICRO: then callback"))
  .then(() => console.log("👻 [D-6] <4-Sync> MICRO: then callback"))
  .catch((err) => console.log("👹 [E-8] <6-Async> MICRO: catch callback", err))
  .finally(() => console.log("🦄 [F-10] <8-async> MICRO: finally callback"));

Promise.resolve()
  .then(() => console.log("👦 [G-5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [H-7] <5-Async> MICRO: then"))
  .then(() => console.log("👦 [I-9] <7-Async> MICRO: then"));

console.log("🦖 [J-2] MAINLINE: End");
```

既に拒否状態の Promise インスタンスに対しては `then()` メソッドのコールバックは実行されずに、`catch()` メソッドのコールバックによって例外補足されます。ただし、マイクロタスクは発行されます。

実行順番は次のようになります。

```sh
❯ v8 catchMicrotask-1.js
🦖 [A-1] MAINLINE: Start
🦖 [J-2] MAINLINE: End
👦 [B-3] <1-Sync> MICRO: then
👦 [G-5] <3-Sync> MICRO: then
👦 [H-7] <5-Async> MICRO: then
👹 [E-8] <6-Async> MICRO: catch callback Error: Exception
👦 [I-9] <7-Async> MICRO: then
🦄 [F-10] <8-async> MICRO: finally callback
```

もし、`then()` メソッドによってマイクロタスクが発行されていなれば、次のコードの `catch()` メソッドのコールバックの実行順番は上のコードとまったく同じになるはずですが、そうはなりません。

```js
// catchMicrotask-2.js
console.log("🦖 [A-1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [B-3] <1-Sync> MICRO: then"));

Promise.reject(new Error("Exception"))
  .catch((err) => console.log("👹 [E-4] <2-Async> MICRO: catch callback", err))
  .then(() => console.log("👻 [C-6] <4-Sync> MICRO: then callback"))
  .then(() => console.log("👻 [D-8] <6-Sync> MICRO: then callback"))
  .finally(() => console.log("🦄 [F-10] <8-async> MICRO: finally callback"));

Promise.resolve()
  .then(() => console.log("👦 [G-5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [H-7] <5-Async> MICRO: then"))
  .then(() => console.log("👦 [I-9] <7-Async> MICRO: then"));

console.log("🦖 [J-2] MAINLINE: End");
```

実際に実行すると `catch()` メソッドのコールバックの実行順番が先程よりも早くなっていることが分かります。

```sh
❯ v8 catchMicrotask-2.js
🦖 [A-1] MAINLINE: Start
🦖 [J-2] MAINLINE: End
👦 [B-3] <1-Sync> MICRO: then
👹 [E-4] <2-Async> MICRO: catch callback Error: Exception
👦 [G-5] <3-Sync> MICRO: then
👻 [C-6] <4-Sync> MICRO: then callback
👦 [H-7] <5-Async> MICRO: then
👻 [D-8] <6-Sync> MICRO: then callback
👦 [I-9] <7-Async> MICRO: then
🦄 [F-10] <8-async> MICRO: finally callback
```

同様に `catch()` メソッドもコールバックが実行されなくてもマイクロタスクが発生します。

```js
// catchMicrotask-3.js
console.log("🦖 [A-1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [B-3] <1-Sync> MICRO: then"));

Promise.resolve()
  .catch((err) => console.log("👹 [C-4] <2-Async> MICRO: catch callback", err))
  .catch((err) => console.log("👹 [D-6] <4-Async> MICRO: catch callback", err))
  .then(() => console.log("👻 [E-8] <6-Sync> MICRO: then callback"))
  .finally(() => console.log("🦄 [F-10] <8-async> MICRO: finally callback"));

Promise.resolve()
  .then(() => console.log("👦 [G-5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [H-7] <5-Async> MICRO: then"))
  .then(() => console.log("👦 [I-9] <7-Async> MICRO: then"));

console.log("🦖 [J-2] MAINLINE: End");
```

ということで実行順番は次のようになります。もし `catch()` メソッドがマイクロタスクを発行しな k れば、`[E-8]` の実行順番は `[G-5]`   よりも先に来るはずですが、そうはなりません。
 
```sh
❯ v8 catchMicrotask-3.js
🦖 [A-1] MAINLINE: Start
🦖 [J-2] MAINLINE: End
👦 [B-3] <1-Sync> MICRO: then
👦 [G-5] <3-Sync> MICRO: then
👦 [H-7] <5-Async> MICRO: then
👻 [E-8] <6-Sync> MICRO: then callback
👦 [I-9] <7-Async> MICRO: then
🦄 [F-10] <8-async> MICRO: finally callback
```

