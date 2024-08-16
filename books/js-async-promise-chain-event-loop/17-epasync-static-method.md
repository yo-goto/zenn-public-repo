---
title: "Promise の静的メソッド"
cssclass: zenn
date: 2022-06-26
modified: 2024-08-16
AutoNoteMover: disable
tags: type/zenn/book, JavaScript/async
aliases: Promise本『Promise の静的メソッド』
---

## このチャプターについて

Promise の静的メソッドはいくつか種類がありますが、その中でも並列化の合成に関わるものは Promise combinator と呼ばれます。次のチャプターでは具体的に combinator と並列化についての解説しましすが、このチャプターでは、combinator 以外の静的メソッドについて改めて解説しておきます。

最近ではさらにいくつかの新しい静的メソッドも追加されているのでそれらの使い方についても解説します。

## Promise.reject

`Promise.reject()` は Promise インスタンスを生成して、その Promise インスタンスを拒否するメソッドです。このメソッドは、引数に渡された値を拒否値として持つ Promise インスタンスを生成します。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/reject

次に説明する `Promise.resolve()` とは異なり、Unwrapping の能力が無いため、引数が Promise オブジェクトの場合であっても、その引数の Promise を解体せず、その Promise オブジェクト自体を拒否理由として扱います。

```js
const promise = Promise.reject(Promise.resolve(42));

promise
  .then(val => console.log("[1]", val))
  .catch(err => console.log("[2]", err));

/* 実行結果
[2] Promise { 42 }
*/
```

## Promise.resolve

`Promise.resolve()` は Promise インスタンスを生成して、その Promise インスタンスを解決するメソッドです。このメソッドは、引数に渡された値を解決値として持つ Promise インスタンスを生成します。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve

`Promise.fulfilled()` ではなく、`Promise.resolve()` という名前になっていることに注意してください。つまり単に Promise オブジェクトを「履行(fulfillment)」しようとするのではなく「解決(resolution)」を行おうとします。

そして、Unwrapping の能力を持つため、引数の値によって処理内容が変わり、既存の Promise オブジェクトが引数として渡された場合にはそのままその Promise オブジェクトを返します。

```js
// プレーンな値(42)で解決
const p1 = Promise.resolve(42); // => 42で履行

// 履行状態のPromiseオブジェクトで解決
const p2 = Promise.resolve(Promise.resolve(42)); // 42で履行

// 拒否状態のPromiseオブジェクトで解決
const p3 = Promise.resolve(Promise.reject("Fail")); // Failで拒否

// 待機状態のPromiseオブジェクトで解決
const p4 = Promise.resolve(new Promise()); // 待機のまま
```

## Promise.withResolvers

『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting#return-await-promiseresolve42-の場合)』のチャプターにおいて、以下のようなコードを見ました。

```js:Promiseを外から解決・拒否する方法
// Executor関数のコールバックを外部管理するための変数
let promiseResolve, promiseReject;

const promise = new Promise((resolve, reject) => {
  // resolve関数を外部の変数に束縛
  promiseResolve = resolve;
  // reject関数を外部の変数に束縛
  promiseReject = reject;
}).then(() => console.log("resolve完了"));

setTimeout(() => {
  console.log("Start");
  promiseResolve();
  console.log("End");
}, 1000);

/* 出力結果
Start
End
resolve完了
*/
```

このように外部から Promise オブジェクトを解決・拒否する方法は、[Promise.withResolvers](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers) という ES2024 で追加された新しい Promise の静的メソッドでも実現可能となりました。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers

この `Promise.withResolvers()` メソッドを使うことで Promise の解決と拒否を Promise インスタンスの生成後に自由に外部から制御できます。上のコードと同等のことを `Promise.withResolvers()` を使って書くと次のようになります。

```js
const { promise, resolve, reject } = Promise.withResolvers();

// Promiseが解決後に実行されるコールバックをthenメソッドで登録
promise.then((value) => console.log("resolve完了", value));

setTimeout(() => {
  console.log("Start");
  resolve("履行値");
  console.log("End");
}, 1000);

/* 出力結果
Start
End
resolve完了 履行値
*/
```

## Promise.try

`Promise.try()` メソッドは現在 [stage3 の ECMAScript proposal](https://github.com/tc39/proposal-promise-try) です。

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/try

`Promise.try()` を使うことで、以下のような処理を統一的なインターフェースで扱えるようになります。

```js
const promiseTry = (func) => {
  return new Promise((resolve, reject) => {
    try {
      resolve(func());
    } catch (error) {
      reject(error);
    }
  });
};
```

引数の `func` は `new Promise(executor)` の `executor` 関数と同じ様に同期的に実行されます。その関数ではプレーンな値やPromiseオブジェクトが返されたり、例外が投げられたりすることができます。`Promise.try()` ではそれぞれの場合で上記コードのように解決が行われる Promise オブジェクトが返されます。

```js
const f1 = () => {
  return 42;
};
const f2 = () => {
  return Promise.resolve(42);
};
const f3 = () => {
  throw new Error("Fail");
};

const p1 = Promise.try(f1); // 42で履行されたPromiseオブジェクトが返る
const p2 = Promise.try(f2); // 42で履行されたPromiseオブジェクトが返る
const p3 = Promise.try(f3); // 例外で拒否されたPromiseオブジェクトが返る
```

要するに抽象化された `func` 関数内の処理結果を Promise でラップしたものが返されます。
