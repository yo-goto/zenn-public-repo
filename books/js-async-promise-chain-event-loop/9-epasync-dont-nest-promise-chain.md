---
title: "Promise chain はネストさせない"
cssclass: zenn
date: 2022-04-17
modified: 2023-01-06
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『Promise chain はネストさせない』
---

# このチャプターについて

このチャプターでは、Promise chain におけるネストについて、アンチパターンとしての話と、原理的な話を行います。

:::message alert
このチャプターの解説は『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで解説した「内容の間違い」の影響を以前まで受けていましたが、現在は内容を修正・補足しました。
:::

# Promise chain をネストしてみる

前のチャプターでは、`then()` メソッドのコールバックにおいて、Promise インスタンスを `return` した場合「Promise インスタンスの `resolve` に使われた値は次の `then()` メソッドのコールバック関数の引数として渡される」という話でした。

一方、次のように場合はどうなるでしょうか？
`return` しているものが `returnPromise().then(cb)` となっています。

```js
// promiseNest.js
console.log('🦖 [1] Sync');

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 [${order}] (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise('1st Promise', '2')
  .then((value) => {
    console.log('👦 [5] Async');
    console.log('👦 Resolved value: ', value);

    return returnPromise('2nd Promise', '6')
      .then((value) => {
        console.log('👦 [9] Async');
        console.log('👦 Resolved value: ', value);
        return 'from [9] callback';
      });
  })
  .then((value) => {
    console.log('👦 [11] Async');
    console.log('👦 Resolved value: ', value);
  });

returnPromise('3rd Promise', '3')
  .then((value) => {
    console.log('👦 [7] Async');
    console.log('👦 Resolved value: ', value);

    return returnPromise('4th Promise', '8')
      .then((value) => {
        console.log('👦 [10] Async');
        console.log('👦 Resolved value: ', value);
        return 'from [10] callback';
      });
  })
  .then((value) => {
    console.log('👦 [12] Async');
    console.log('👦 Resolved value: ', value);
  });

console.log('🦖 [4] Sync');
```

基本的には今までの流れと代りません。また圧縮して書いてみます。

```js
console.log("🦖 [1] Sync");
const returnPromise = (resolvedValue, order) => {...};

returnPromise("1st Promise", "2").then(cb1).then(cb2);

returnPromise("3rd Promise", "3").then(cb3).then(cb4);

console.log("🦖 [4] Sync");
```

今までと同じところまで流れを書いてみます。
1. 最初のタスク「スクリプトの評価」による「すべての同期処理の実行」
  1. `console.log("🦖 [1] Sync")` が同期処理される
  2. `returnPromise("1st Promise", "2")` が同期処理されて `then(cb1)` のコールバック `cb1` が直ちがマイクロタスクキューへと送られる
  3. `returnPromise("3rd Promise", "3")` が同期処理されて `then(cb3)` のコールバック `cb3` が直ちにマイクロタスクキューへと送られる
  4. `console.log("🦖 [4] Sync")` が同期処理される
  5. イベントループの次のステップへ移行 (同期処理がすべて終わり、グローバルコンテキストがポップしてコールスタックが空になったので、「マイクロタスクのチェックポイント」となる)
2. 「マイクロタスクキューのすべてのマイクロタスクを実行する」
  6. `cb1` が実行される

この `cb1` の実行が問題です。`return returnPromise("2nd Promise", "6")` に注目すると、`returnPromise()` 関数は直ちに履行状態になる Promise インスタンスを返すので、`then(callbackNext)` のコールバック `callbackNext` が直ちにマイクロタスクキューへと送られます。`cb1` が実行される時点でのマイクロタスクキューには `cb3` コールバック関数がマイクロタスクとして追加されており、先頭にある状態です。

```sh:マイクロタスクキュー
(先頭) <-- cb3
```

マイクロタスクキューの先頭にある `cb1` について考えてみます。

```js
returnPromise("1st Promise", "2")
  .then((value) => {
    // これが cb1
    // 上から下に実行されていく
    console.log("👦 [5] Async");
    console.log("👦 Resolved value: ", value);

    return returnPromise("2nd Promise", "6").then(callbackNext);
    // 返される Promise インスタンスが履行状態なので then のコールバック関数が直ちにマイクロタスクキューへと送られる

    // 🔥 この then メソッドから返される Promise を解決するために追加で２つのマイクロタスクが発生する
    // ただちに追加のマイクロタスクである一つ目がエンキューされることに注意
  }) // ここで返される Promise chain はまだ待機状態
  .then(cb2);
```

コールバック関数内の処理が上から下の順番に実行されていきます。

コールバック関数内で Promise オブジェクト (Promise chain) を返却していますが、`returnPromise("2nd Promise", "6")` は直ちに履行するので、chain されている `then` メソッドに登録されたコールバック関数 `callbackNext` が直ちにマイクロタスクとしてエンキューされます。この時点でのマイクロタスクキューの状態は次のようになります。

```sh:マイクロタスクキュー
(先頭) <-- cb3 <-- callbackNext
```

`then` メソッドのコールバック内で Promise オブジェクトを返すと追加のマイクロタスクが２つ発生しました。これについては『[then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)』と『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで解説しました。追加のマイクロタスクは連鎖的に２つ発生するわけですが、一つ目の追加のマイクロタスクはこの `cb1` の実行時に直ちにマイクロタスクキューへとエンキューされます。このマイクロタスクは `extraA-1` という名前にしておきましょう。

従って、マイクロタスクとして `cb1` が処理完了した時点でのマイクロタスクキューの状態は以下のようになります。

```sh:マイクロタスクキュー
(先頭) <-- cb3 <-- callbackNext <-- extraA-1
```

マイクロタスクキューについてはこのようなことになりますが、別の観点でも考えてみましょう。

混乱しやすいところですが、Promise chain において `then()` メソッドのコールバック関数内で、`return` によって Promise インスタンスを返した場合はその Promise インスタンスの内包する値、つまり解決値 (resolve された値) が次の `then()` メソッドのコールバック関数の引数として渡されます。

いまコールバック関数内で `return` しているのは `returnPromise("2nd Promise", "6")` ではなく、`returnPromise("2nd Promise", "6").then(callbackNext)` なので、`then(callbackNext)` で返される Promise インスタンスの resolve した値が `then(cb2)` のコールバック関数 `cb2` の引数として渡されるはずです。

ですが、今の時点では `callbackNext` はキューへ送られていて実行されていないので、`then(callbackNext)` か返される Promise インスタンスは待機状態です。つまり、待機状態の Promise インスタンスを返してしまっています。

`then()` メソッドのコールバック関数内で待機状態の Promise インスタンスを返した場合はそれが解決されない限り、その `then()` メソッドから返ってくる Promise インスタンスも待機状態のままとなります。

ここで考えるのは親のコールバック関数 `cb1` を登録していた `then(cb1)` から返される Promise インスタンスです。この `cb1` から返される Promise インスタンスが解決されないままだと `then(cb1)` から返される Promise インスタンスの状態が履行状態にはならず待機状態のもままで、次の `then(cb2)` メソッドのコールバック関数 `cb2` をマイクロタスクキューへと送ることができません。

それはそれとして、`callbackNext` と `extraA-1` がマイクロタスクキューへと送られた時にはイベントループのステップは「マイクロタスクキューのすべてのマイクロタスクを実行する」の状態にあります。現時点でマイクロタスクキューの先頭にあるタスクはコールバック関数 `cb3` なので、このコールバックが次に実行されます。

```sh:マイクロタスクキュー
(先頭) <-- cb3 <-- callbackNext <-- extraA-1
```

それでは次に実行されるはずのコールバック関数 `cb3` に注目してみます。

```js
returnPromise("3rd Promise", "3")
  .then((value) => {
    // これが cb3
    // 上から下に実行されていく
    console.log("👦 [7] Async");
    console.log("👦 Resolved value: ", value);

    return returnPromise("4th Promise", "8").then(callbackNext2);
    // 返される Promise インスタンスが履行状態なので then のコールバック関数が直ちにマイクロタスクキューへと送られる

    // 🔥 この then メソッドから返される Promise を解決するために追加で２つのマイクロタスクが発生する
    // ただちに追加のマイクロタスクである一つ目がエンキューされることに注意
  }) // ここで返される Promise chain はまだ待機状態
  .then(cb4);
```

全く同じように、`callbackNext2` とコールバック関数内で Promise オブジェクトが返されたことにより発生する２つの追加のマイクロタスクの１つ目である `extraB-1` が直ちにマイクロタスクキューへと追加されます。

```sh:マイクロタスクキュー
(先頭) <-- callbackNext <-- extraA-1 <-- callbackNext2 <-- extraB-1
```

`returnPromise("3rd Promise", "3").then(cb3)` から返される Promise インスタンスはまだ待機状態となります。イベントループの状態もそのままなので、再びマイクロタスクキューの先頭にあるマイクロタスク `callbackNext` が実行されます。

```sh:マイクロタスクキュー
(先頭) <-- extraA-1 <-- callbackNext2 <-- extraB-1
```

`callbackNext` が実行されることで、コールバック関数 `cb1` から返却していた Promise オブジェクトが解決することになります。これによって `then(cb1)` から返る Promise オブジェクトを解決する準備が完了し、さらに追加発生してた `extraA-1` が次のマイクロタスクキューの先頭になり実行されることで、追加発生する２つ目のマイクロタスク (名前は `extraA-2` としておきます) がマイクロタスクキューへと発行されます。

```sh:マイクロタスクキュー
(先頭) <-- callbackNext2 <-- extraB-1 <-- extraA-2
```

先頭のマイクロタスクは `callbackNext2` となり、これが実行されます。

```js
returnPromise("3rd Promise", "3")
  .then((value) => { // cb3
    console.log("👦 [7] Async");
    console.log("👦 Resolved value: ", value);

    return returnPromise("4th Promise", "8")
      .then((value) => { // callbackNext2
        console.log('👦 [10] Async');
        console.log('👦 Resolved value: ', value);
        return 'from [10] callback';
      });
    // 返される Promise インスタンスが履行状態なので then のコールバック関数が直ちにマイクロタスクキューへと送られる

    // 🔥 この then メソッドから返される Promise を解決するために追加で２つのマイクロタスクが発生する(ただちに追加のマイクロタスクである一つ目がエンキューされることに注意)
    // extraB-1
    // extraB-2
  }) // ここで返される Promise chain はまだ待機状態
  .then(cb4);
```

`callbackNext` のときと同じ様に `callbackNext2` が実行されることで、コールバック関数 `cb3` から返却していた Promise オブジェクトが解決することなります。これによって `then(cb3)` から返る Promise オブジェクトを解決する準備が完了します。

現時点のマイクロタスクキューの状態は次のようになります。

```sh:マイクロタスクキュー
(先頭) <-- extraB-1 <-- extraA-2
```

さらに追加発生していた `extraB-1` が次のマイクロタスクキューの先頭になり実行されることで、追加発生する２つ目のマイクロタスク (名前は `extraB-2` としておきます) がマイクロタスクキューへと発行されます。

```sh:マイクロタスクキュー
(先頭) <-- extraA-2 <-- extraB-2
```

そして、次のマイクロタスクキュー先頭にある `extraA-2` が実行されることで、`then(cb1)` から返る Promise オブジェクトが解決して履行します。

```js
returnPromise("1st Promise", "2")
  .then((value) => { // cb1
    console.log("👦 [5] Async");
    console.log("👦 Resolved value: ", value);

    return returnPromise("2nd Promise", "6")
      .then((value) => { // callbackNext
        console.log("👦 [9] Async");
        console.log("👦 Resolved value: ", value);
        return "from [9] callback";
      });
      // extraA-1 --> extraA-2
  }) // 解決されて履行したので次の cb2 がマイクロタスクとして発行される
  .then((value) => {
    // cb2
    console.log("👦 [11] Async");
    console.log("👦 Resolved value: ", value);
  });
```

`then(cb1)` の次に chain している `then(cb2)` のコールバック関数 `cb2` がようやくマイクロタスクとして発行されます。

```sh:マイクロタスクキュー
(先頭) <-- extraB-2 <-- cb2
```

次にマイクロタスクキュー先頭にある `extraB-2` が実行されることで、同じ様に `then(cb3)` から返る Promise オブジェクトが解決して履行します。

```js
returnPromise("3rd Promise", "3")
  .then((value) => { // cb3
    console.log("👦 [7] Async");
    console.log("👦 Resolved value: ", value);

    return returnPromise("4th Promise", "8")
      .then((value) => { // callbackNext2
        console.log('👦 [10] Async');
        console.log('👦 Resolved value: ', value);
        return 'from [10] callback';
      });
      // extraB-1 --> extraB-2
  }) // 解決されて履行したので次の cb4 がマイクロタスクとして発行される
  .then((value) => { // cb4
    console.log('👦 [12] Async');
    console.log('👦 Resolved value: ', value);
  });
```

現時点でのマイクロタスクキューの状態は次のようになります。

```sh:マイクロタスクキュー
(先頭) <-- cb2 <-- cb4
```

そして、同じように、`cb2`、`cb4` の順番で実行されて終わります。ということで、出力は次のようになります。

```sh
❯ deno run promiseNest.js
🦖 [1] Sync
👻 [2] (a)sync
👻 [3] (a)sync
🦖 [4] Sync
👦 [5] Async
👦 Resolved value: 1st Promise
👻 [6] (a)sync
👦 [7] Async
👦 Resolved value: 3rd Promise
👻 [8] (a)sync
👦 [9] Async
👦 Resolved value: 2nd Promise
👦 [10] Async
👦 Resolved value: 4th Promise
👦 [11] Async
👦 Resolved value: from [9] callback
👦 [12] Async
👦 Resolved value: from [10] callback
```

それでは全体を再度みて、発生するマイクロタスクを直接的に追跡しておきましょう。

```js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: どちらの promise chain かの識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log('🦖 [1] Sync');

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 [${order}] (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise('1st Promise', '2')
  .then((value) => { // <1-a[1]>
    console.log('👦 [5] Async');
    console.log('👦 Resolved value: ', value);

    // ここでは <3-a[2]> と <4-a[3]> の２つのマイクロタスクが直ちに発行されることに注意
    return returnPromise('2nd Promise', '6')
      .then((value) => { // <3-a[2]>
        console.log('👦 [9] Async');
        console.log('👦 Resolved value: ', value);
        return 'from [9] callback';
      });
    // 🔥 promise を返しているため追加のマイクロタスクが２つ発生
    // <4-a[3]> returnPromise('2nd Promise', '6').then(cb).then(resolve, reject) の呼び出し
    // ↪ <7-a[4]> resolve 関数の実行
  })
  .then((value) => { // <9-a[5]>
    console.log('👦 [11] Async');
    console.log('👦 Resolved value: ', value);
  });

returnPromise('3rd Promise', '3')
  .then((value) => { // <2-b[1]>
    console.log('👦 [7] Async');
    console.log('👦 Resolved value: ', value);

    // ここでは <5-b[2]> と <6-a[3]> の２つのマイクロタスクが直ちに発行されることに注意
    return returnPromise('4th Promise', '8')
      .then((value) => { // <5-b[2]>
        console.log('👦 [10] Async');
        console.log('👦 Resolved value: ', value);
        return 'from [10] callback';
      });
    // 🔥 promise を返しているため追加のマイクロタスクが２つ発生
    // <6-b[3]> returnPromise('4th Promise', '8').then(cb).then(resolve, reject) の呼び出し
    // ↪ <8-b[4]> resolve 関数の実行
  })
  .then((value) => { // <10-b[5]>
    console.log('👦 [12] Async');
    console.log('👦 Resolved value: ', value);
  });

console.log('🦖 [4] Sync');
```

:::message alert
このコードを可視化したものは以下となりますが、『[then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)』のチャプターで解説した追加発生するマイクロタスクが可視化されていないので気をつけてください。

👉 [promiseNest.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gcHJvbWlzZU5lc3QuanMKY29uc29sZS5sb2coJ1sxXSBTeW5jIHByb2Nlc3MnKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGBbJHtvcmRlcn1dIFRoaXMgbGluZSBpcyAoQSlTeW5jaHJvbm91c2x5IGV4ZWN1dGVkYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pOwp9OwoKcmV0dXJuUHJvbWlzZSgnMXN0IFByb21pc2UnLCAnMicpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygnWzVdIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCcpOwogICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7CiAgICByZXR1cm4gcmV0dXJuUHJvbWlzZSgnMm5kIFByb21pc2UnLCAnNicpCiAgICAgIC50aGVuKCh2YWx1ZSkgPT4gewogICAgICAgIGNvbnNvbGUubG9nKCdbOV0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICAgICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7CiAgICAgICAgcmV0dXJuICdmcm9tIFs5XSBjYWxsYmFjayc7CiAgICAgIH0pOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygnWzExXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQnKTsKICAgIGNvbnNvbGUubG9nKCdSZXNvbHZlZCB2YWx1ZTogJywgdmFsdWUpOwogIH0pOwpyZXR1cm5Qcm9taXNlKCczcmQgUHJvbWlzZScsICczJykKICAudGhlbigodmFsdWUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCdbN10gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlKTsKICAgIHJldHVybiByZXR1cm5Qcm9taXNlKCc0dGggUHJvbWlzZScsICc4JykKICAgICAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICAgICAgY29uc29sZS5sb2coJ1sxMF0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICAgICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7CiAgICAgICAgcmV0dXJuICdmcm9tIFsxMF0gY2FsbGJhY2snOwogICAgICB9KTsKICB9KQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coJ1sxMl0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCdbNF0gU3luYyBwcm9jZXNzJyk7)
- ⚠️ 注意: JS Visualizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク実行のタイミングについて誤解しないように注意してください
:::

# Promise chain のネストはアンチパターン

このように `then()` メソッドをネストさせるようなやり方は特に意味がある場合を除いて、流れがわかりづらくなってしまうので通常は避けるべきアンチパターンとなります。このネストはフラットにでき、Promise chain はなるべくネストが浅くなるようにフラットにするのが推奨されます。

実際にネストを解消してみます。

```diff js
console.log("🦖 [1] Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("👦 [5] Async");
    console.log("👦 Resolved value: ", value);

    return returnPromise("2nd Promise", "[6]")
-     .then((value) => {
-       console.log("👦 [9] Async");
-       console.log("👦 Resolved value: ", value);
-       return "from [9] callback";
-     });
  })
+ .then((value) => {
+   console.log("👦 [9] Async");
+   console.log("👦 Resolved value: ", value);
+   return "from [9] callback";
+ })
  .then((value) => {
    console.log("👦 [11] Async");
    console.log("👦 Resolved value: ", value);
  });

returnPromise("3rd Promise", "[3]")
  .then((value) => {
    console.log("👦 [7] Async");
    console.log("👦 Resolved value: ", value);

    return returnPromise("4th Promise", "[8]")
-     .then((value) => {
-       console.log("👦 [10] Async");
-       console.log("👦 Resolved value: ", value);
-       return "from [10] callback";
-     });
  })
+ .then((value) => {
+   console.log("👦 [10] Async");
+   console.log("👦 Resolved value: ", value);
+   return "from [10] callback";
+ })
  .then((value) => {
    console.log("👦 [12] Async");
    console.log("👦 Resolved value: ", value);
  });

console.log("🦖 [4] Sync");
```

結果はこのようになり、ネストした状態のものよりも圧倒的に見やすく、流れが分かりなりました。

```js
// promiseNestShallow.js
console.log('🦖 [1] Sync');

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 [${order}] (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise('1st Promise', '2')
  .then((value) => {
    console.log('👦 [5] Async');
    console.log('👦 Resolved value: ', value);
    return returnPromise('2nd Promise', '6');
  })
  .then((value) => {
    console.log('👦 [9] Async');
    console.log('👦 Resolved value: ', value);
    return 'from [9] callback';
  })
  .then((value) => {
    console.log('👦 [11] Async');
    console.log('👦 Resolved value: ', value);
  });

returnPromise('3rd Promise', '3')
  .then((value) => {
    console.log('👦 [7] Async');
    console.log('👦 Resolved value: ', value);
    return returnPromise('4th Promise', '8');
  })
  .then((value) => {
    console.log('👦 [10] Async');
    console.log('👦 Resolved value: ', value);
    return 'from [10] callback';
  })
  .then((value) => {
    console.log('👦 [12] Async');
    console.log('👦 Resolved value: ', value);
  });

console.log('🦖 [4] Sync');
```

出力結果は全く同じになります。

```sh
❯ deno run promiseNestShallow.js
🦖 [1] Sync
👻 [2] (a)sync
👻 [3] (a)sync
🦖 [4] Sync
👦 [5] Async
👦 Resolved value: 1st Promise
👻 [6] (a)sync
👦 [7] Async
👦 Resolved value: 3rd Promise
👻 [8] (a)sync
👦 [9] Async
👦 Resolved value: 2nd Promise
👦 [10] Async
👦 Resolved value: 4th Promise
👦 [11] Async
👦 Resolved value: from [9] callback
👦 [12] Async
👦 Resolved value: from [10] callback
```

Promise chain はこのようにネストさせずに流れを見やすくします。

:::message
実はネストをフラット化することで発生するマイクロタスクの順番が前後しますが、実用上は何も問題ありません。この話題については『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで詳しく解説します。
:::
