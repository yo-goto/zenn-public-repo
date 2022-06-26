---
title: "then メソッドのコールバックで Promise インスタンスを返す"
aliases: [ch_then メソッドのコールバックで Promise インスタンスを返す]
---

# このチャプターについて
このチャプターは短いですが、Promise チェーンを理解する上で重要なので１つのチャプターとして独立させています。チャプターは飛びますが、『[コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)』のチャプターでも使う知識なので注意してください。

# then メソッドのコールバックで Promise インスタンスを返す

前のチャプターのコードでは `then()` メソッドのコールバックにおいて `return` で返却していたのは `"Resolved value passing to the next then callback"` という文字列でした。

`return` する値は、数値でも真偽値でも文字列での何でも良いのですが、Promise インスタンスを返した場合はどうなるでしょうか？

答えは、「**その Promise インスタンスが `resolve` された値が次の `then()` メソッドのコールバック関数の引数として渡される**」です。実際に `then()` メソッドのコールバック関数において新しい Promise インスタンスを返してみます。今までのコードをまた流用します。

```js
// returnPromiseFromThenCallback.js
console.log('🦖 [1] Sync process');

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 [${order}] This line is (A)Synchronously executed`);
    // 非同期で実行される場合もあるのでテキストを変更した
    resolve(resolvedValue);
  });
};

returnPromise('1st Promise', '2')
  .then((value) => {
    console.log("👦 [5] This line is Asynchronously executed");
    console.log("👦 Resolved value: ", value);
    return returnPromise("2nd Promise", "6");
    // resolve される値は "2nd Promise" で、これが次の then() のコールバック関数の入力として渡される
  })
  .then((value) => {
    console.log('👦 [9] This line is Asynchronously executed');
    console.log('👦 Resolved value: ', value);
  });
returnPromise('3rd Promise', '3')
  .then((value) => {
    console.log("👦 [7] This line is Asynchronously executed");
    console.log("👦 Resolved value: ", value);
    return returnPromise("4th Promise", "8");
    // resolve される値は "5th Promise" で、これが次の then() のコールバック関数の入力として渡される
  })
  .then((value) => {
    console.log('👦 [10] This line is Asynchronously executed');
    console.log('👦 Resolved value: ', value);
  });

console.log('🦖 [4] Sync process');
```

今まで必ず同期処理として呼ばれていた `returnPromise()` 関数ですが、今回は `then()` メソッドのコールバックで呼び出しているものもあるのでそれらは非同期的に実行されます。従って出力されるテキストを一部変更しました。

これを実行すると、次のような出力になります。

```sh
❯ deno run returnPromiseFromThenCallback.js
🦖 [1] Sync process
👻 [2] This line is (A)Synchronously executed
👻 [3] This line is (A)Synchronously executed
🦖 [4] Sync process
👦 [5] This line is Asynchronously executed
👦 Resolved value:  1st Promise
👻 [6] This line is (A)Synchronously executed
👦 [7] This line is Asynchronously executed
👦 Resolved value:  3rd Promise
👻 [8] This line is (A)Synchronously executed
👦 [9] This line is Asynchronously executed
👦 Resolved value:  2nd Promise
👦 [10] This line is Asynchronously executed
👦 Resolved value:  4th Promise
```

Promise インスタンスを `then()` メソッドのコールバック関数内で `return` したの実行の順番がどうなるか不安になったかもしれませんが、今回の `returnPromise()` 関数の場合は、ただちに履行状態の Promise インスタンスが返ってくるので普通の値を返す場合とまったく同じになります。JS Visuzalizer で可視化したのでまた確認してみてください。

- [returnPromiseFromThenCallback.js](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUZyb21UaGVuQ2FsbGJhY2suanMKY29uc29sZS5sb2coJ1sxXSBTeW5jIHByb2Nlc3MnKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGBbJHtvcmRlcn1dIFRoaXMgbGluZSBpcyAoQSlTeW5jaHJvbm91c2x5IGV4ZWN1dGVkYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pOwp9OwoKcmV0dXJuUHJvbWlzZSgnMXN0IFByb21pc2UnLCAnMicpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygnWzVdIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCcpOwogICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7CiAgICByZXR1cm4gcmV0dXJuUHJvbWlzZSgnMm5kIFByb21pc2UnLCAnNicpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygnWzldIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCcpOwogICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7CiAgfSk7CnJldHVyblByb21pc2UoJzNyZCBQcm9taXNlJywgJzMnKQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coJ1s3XSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQnKTsKICAgIGNvbnNvbGUubG9nKCdSZXNvbHZlZCB2YWx1ZTogJywgdmFsdWUpOwogICAgcmV0dXJuIHJldHVyblByb21pc2UoJzR0aCBQcm9taXNlJywgJzgnKTsKICB9KQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coJ1sxMF0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCdbNF0gU3luYyBwcm9jZXNzJyk7Cg%3D%3D)
- ⚠️ 注意: JS Visuzlizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク実行のタイミングについて誤解しないように注意してください

