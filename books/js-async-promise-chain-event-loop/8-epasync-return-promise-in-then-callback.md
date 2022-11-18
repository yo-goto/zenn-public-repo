---
title: "then メソッドのコールバックで Promise インスタンスを返す"
cssclass: zenn
date: 2022-04-17
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: ch_then メソッドのコールバックで Promise インスタンスを返す
---

# このチャプターについて

このチャプターは短いですが、Promise chain を理解する上で重要なので１つのチャプターとして独立させています。チャプターは飛びますが、『[コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)』のチャプターでも使う知識なので注意してください。

# then メソッドのコールバックで Promise インスタンスを返す

前のチャプターのコードでは `then()` メソッドのコールバックにおいて `return` で返却していたのは `"Resolved value passing to the next then callback"` という文字列でした。

`return` する値は文字列に関わらず数値や真偽値でも良いのですが、Promise インスタンスを返した場合はどうなるでしょうか？

その答えは「**渡された Promise インスタンスの `resolve` に使われた値が次の `then()` メソッドのコールバック関数の引数として渡される**」です。実際に `then()` メソッドのコールバック関数において新しい Promise インスタンスを返してみます。今までのコードをまた流用します。

```js
// returnPromiseFromThenCallback.js
console.log("🦖 [1] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    // 非同期で実行される場合もあるのでテキストを変更した
    resolve(resolvedValue);
  });
};

returnPromise("🐵 1st Promise", "[2]")
  .then((value) => {
    console.log("👦 [5] async:", value);
    return returnPromise("🐵 2nd Promise", "[6]");
    // resolve される値は "2nd Promise" で
    // これが次の then() のコールバック関数の入力として渡される
  })
  .then((value) => {
    console.log("👦 [9] async:", value);
  });
returnPromise("🐵 3rd Promise", "[3]")
  .then((value) => {
    console.log("👦 [7] async:", value);
    return returnPromise("🐵 4th Promise", "[8]");
    // resolve される値は "4th Promise" で
    // これが次の then() のコールバック関数の入力として渡される
  })
  .then((value) => {
    console.log("👦 [10] async:", value);
  });

console.log("🦖 [4] MAINLINE(End): Sync");
```

今まで必ず同期処理として呼ばれていた `returnPromise()` 関数ですが、今回は `then()` メソッドのコールバックで呼び出しているものもあるのでそれらは非同期的に実行されます。従って出力されるテキストを一部変更しました。

これを実行すると、次のような出力になります。

```sh
❯ deno run returnPromiseFromThenCallback.js
🦖 [1] MAINLINE(Start): Sync
👻 [2] (a)sync
👻 [3] (a)sync
🦖 [4] MAINLINE(End): Sync
👦 [5] async: 🐵 1st Promise
👻 [6] (a)sync
👦 [7] async: 🐵 3rd Promise
👻 [8] (a)sync
👦 [9] async: 🐵 2nd Promise
👦 [10] async: 🐵 4th Promise
```

Promise インスタンスを `then()` メソッドのコールバック関数内で `return` したの実行の順番がどうなるか不安になったかもしれませんが、今回の `returnPromise()` 関数の場合は、ただちに履行状態の Promise インスタンスが返ってくるので普通の値を返す場合とまったく同じになります。JS Visuzalizer で可視化したのでまた確認してみてください。

- [returnPromiseFromThenCallback.js](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUZyb21UaGVuQ2FsbGJhY2suanMKY29uc29sZS5sb2coIlsxXSBNQUlOTElORShTdGFydCk6IHN5bmMiKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGAke29yZGVyfSAoYSlzeW5jYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pOwp9OwoKcmV0dXJuUHJvbWlzZSgiMXN0IFByb21pc2UiLCAiWzJdIikKICAudGhlbigodmFsdWUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCJbNV0gYXN5bmM6IiwgdmFsdWUpOwogICAgcmV0dXJuIHJldHVyblByb21pc2UoIjJuZCBQcm9taXNlIiwgIls2XSIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiWzldIGFzeW5jOiIsIHZhbHVlKTsKICB9KTsKcmV0dXJuUHJvbWlzZSgiM3JkIFByb21pc2UiLCAiWzNdIikKICAudGhlbigodmFsdWUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCJbN10gYXN5bmM6IiwgdmFsdWUpOwogICAgcmV0dXJuIHJldHVyblByb21pc2UoIjR0aCBQcm9taXNlIiwgIls4XSIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiWzEwXSBhc3luYzoiLCB2YWx1ZSk7CiAgfSk7Cgpjb25zb2xlLmxvZygiWzRdIE1BSU5MSU5FKEVuZCk6IHN5bmMiKTsK)
- ⚠️ 注意: JS Visuzlizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク実行のタイミングについて誤解しないように注意してください
