---
title: "Promise チェーンはネストさせない"
---

:::message alert
このチャプターの内容は『イベントループの概要と注意点』のチャプターに基づいた古いものであり、イベントループの説明自体は間違っていませんが、分かりづらい部分があるので注意してください。
:::

# Promise チェーンをネストしてみる
前のチャプターでは、`then()` メソッドのコールバックにおいて、Promise インスタンスを `return` した場合は、「Promise インスタンスが `resolve` された値が次の `then()` メソッドのコールバック関数の引数として渡される」という話でした。

だだし、次のように場合はどうなるでしょうか? 
`return` しているものが `returnPromise().then(cb)` となっています。

```js
// promiseNest.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "[6]")
      .then((value) => {
        console.log("[9] This line is Asynchronously executed");
        console.log("Resolved value: ", value);
        return "from [9] callback";
      });
  })
  .then((value) => {
    console.log("[11] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
returnPromise("3rd Promise", "[3]")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("4th Promise", "[8]")
      .then((value) => {
        console.log("[10] This line is Asynchronously executed");
        console.log("Resolved value: ", value);
        return "from [10] callback";
      });
  })
  .then((value) => {
    console.log("[12] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[4] Sync process");
```

基本的には今までの流れと代りません。また圧縮して書いてみます。

```js
console.log("[1] Sync process");
const returnPromise = (resolvedValue, order) => {...};
returnPromise("1st Promise", "2").then(cb1).then(cb2);
returnPromise("3rd Promise", "3").then(cb3).then(cb4);
console.log("[4] Sync process");
```

今までと同じところまで流れを書いてみます。
1. ステップ１+ステップ２「同期処理を実行する」
  1. `console.log("[1] Sync process")` が同期処理される
  2. `returnPromise("1st Promise", "2")` が同期処理されて `then(cb1)` のコールバック `cb1` が直ちに Microtask queue へと送られる
  3. `returnPromise("3rd Promise", "3")` が同期処理されて `then(cb3)` のコールバック `cb3` が直ちに Microtask queue へと送られる
  4. `console.log("[4] Sync process")` が同期処理される
  5. Event Loop の次のステップ3へ移行
2. 「Microtask queue のすべての Microtask を実行する」
  6. `cb1` が実行される

:::message alert
Event loop のステップ１はステップ２と同質のものであり、「スクリプトの評価」は実質的に Task(Macrotask) として扱われるので、これが終わると、Event loop は次のステップ３「すべての Microtask の実行」へと移行します。

詳しくは、[Event loop の概要と注意点](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/2-epasync-event-loop) のチャプターを確認してください。
:::

この `cb1` の実行が問題です。`return returnPromise("2nd Promise", "6")` に注目すると、`returnPromise()` 関数は直ちに履行状態になる Promise インスタンスを返すので、`then(callbackNext)` のコールバック `callbackNext` が直ちに Microtask queue へと送られます。現時点の Microtask queue には次のコールバック関数がマイクロタスクとして追加されています。

```sh:Microtask queue
<-- cb3
```

この時点での `cb1` に注目してみます。

```js
returnPromise("1st Promise", "2")
  .then((value) => {
    // これが cb1 
    // 上から下に実行されていく
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);

    return returnPromise("2nd Promise", "6").then(callbackNext);
    // 返される Promise インスタンスが履行状態なので then のコールバック関数が直ちに Microtask queue へと送られる
  }) // ここで返される Promise チェーンはまだ待機状態
  .then(cb2);
```

さて、ここが混乱しやすいところですが、Promise チェーンにおいて `then()` メソッドのコールバック関数内で、`return` によって Promise インスタンスを返した場合はその解決値(resolve された値)が次の `then()` メソッドのコールバック関数の引数として渡されます。

いまコールバック関数内で `return` しているのは `returnPromise("2nd Promise", "6")` ではなく、`returnPromise("2nd Promise", "6").then(callbackNext)` なので、`then(callbackNext)` で返される Promise インスタンスの resolve した値が `then(cb2)` のコールバック関数 `cb2` の引数として渡されるはずです。

ですが、今の時点では `callbackNext` はキューへ送られていて実行されていないので、`then(callbackNext)` か返される Promise インスタンスは待機(pending)状態です。つまり、待機状態の Promise インスタンスを返してしまっています。

`then()` メソッドのコールバック関数内で待機状態の Promise インスタンスを返した場合はそれが解決されない限り、その `then()` メソッドから返ってくる Promise インスタンスも待機状態のままとなります。

ここで考えるのは親のコールバック関数 `cb1` を登録していた `then(cb1)` から返される Promise インスタンスです。この `cb1` から返される Promise インスタンスが解決されない `then(cb1)` から返される Promise インスタンスの状態が履行状態にはならず待機状態のもままで、次の `then()` メソッドのコールバック関数を Microtask queue へと送ることができません。

それはそれとして、`callbackNext` が Microtask queue へと送られた後、Event Loop のステップは「Microtask queue のすべての Microtask を実行する」の状態にあります。現時点で Microtask queue の先頭にあるタスクはコールバック関数 `cb3` なので、このコールバックが次に実行されます。

```sh:Microtask queue
<-- cb3 <-- callbackNext
```

コールバック関数 `cb3` に注目してみます。

```js
returnPromise("3rd Promise", "3")
  .then((value) => {
    // これが cb3 
    // 上から下に実行されていく
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);

    return returnPromise("4th Promise", "8").then(callbackNext2);
    // 返される Promise インスタンスが履行状態なので then のコールバック関数が直ちに Microtask queue へと送られる
  }) // ここで返される Promise チェーンはまだ待機状態
  .then(cb4);
```

全く同じように、`callbackNext2` が直ちに Microtask queue へと追加されます。

```sh:Microtask queue
<-- callbackNext <-- callbackNext2
```

`returnPromise("3rd Promise", "3").then(cb3)` から返される Promise インスタンスはまだ待機状態となります。Event Loop の状態もそのままなので、再び Microtask queue の先頭のマイクロタスクが実行されます。

```js
returnPromise("1st Promise", "2")
  .then((value) => {
    // cb1
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "6")
      .then((value) => {
        // callbackNext
        console.log("[9] This line is Asynchronously executed");
        console.log("Resolved value: ", value);
        return "from [9] callback";
      });
  })
  .then((value) => {
    // cb2
    console.log("[11] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
```

`callbackNext` が実行されると `cb1` コールバック内において結局、`"from [9] callback"` という文字列で解決された Promise インスタンスが結局 `return` されたことになります。つまり、`return Promise.resolve("from [9] callback")` と同じです。

次のように書きましたが、`then()` メソッドのコールバック関数内で待機状態の Promise が解決されて履行(Fullfilled)状態になったので、その `then()` メソッドから返ってくる Promise インスタンスも解決されて履行状態となります。

>`then()` メソッドのコールバック関数内で待機状態の Promise インスタンスを返した場合はそれが解決されない限り、その `then()` メソッドから返ってくる Promise インスタンスも待機状態のままとなります。

`returnPromise("1st Promise", "2").then(cb1)` から返される Promise インスタンスが待機状態から履行状態に移行したので、次の `then(cb2)` のコールバック関数 `cb2` が Microtask queue へと直ちに送られます。

```sh:Microtask queue
<-- callbackNext2 <-- cb2
```

そして、再び Microtask queue の先頭にあるマイクロタスクであるコールバック関数 `callbackNext2` が実行されて同じのように、`returnPromise("3rd Promise", "3").then(cb3)` から返される Promise インスタンスが履行状態となり、次の `then(cb4)` のコールバック関数 `cb4` が Microtask queue へと直ちに送られます。

```sh:Microtask queue
<-- cb2 <-- cb4
```

そして、同じように、`cb2`、`cb4` の順番で実行されて終わります。出力は次のようになります。

```js
❯ deno run promiseNest.js
[1] Sync process
[2] This line is (A)Synchronously executed
[3] This line is (A)Synchronously executed
[4] Sync process
[5] This line is Asynchronously executed
Resolved value:  1st Promise
[6] This line is (A)Synchronously executed
[7] This line is Asynchronously executed
Resolved value:  3rd Promise
[8] This line is (A)Synchronously executed
[9] This line is Asynchronously executed
Resolved value:  2nd Promise
[10] This line is Asynchronously executed
Resolved value:  4th Promise
[11] This line is Asynchronously executed
Resolved value:  from [9] callback
[12] This line is Asynchronously executed
Resolved value:  from [10] callback
```

分かりにくいですが、結局普通の Promise チェーンと同じ出力の順番になります。JS Visuzalizer で可視化してみたので実際にそうなることを確認してみてください。

- [promiseNest.js - JS Visuzalizer](https://www.jsv9000.app/?code=Ly8gcHJvbWlzZU5lc3QuanMKY29uc29sZS5sb2coJ1sxXSBTeW5jIHByb2Nlc3MnKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGBbJHtvcmRlcn1dIFRoaXMgbGluZSBpcyAoQSlTeW5jaHJvbm91c2x5IGV4ZWN1dGVkYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pOwp9OwoKcmV0dXJuUHJvbWlzZSgnMXN0IFByb21pc2UnLCAnMicpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygnWzVdIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCcpOwogICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7CiAgICByZXR1cm4gcmV0dXJuUHJvbWlzZSgnMm5kIFByb21pc2UnLCAnNicpCiAgICAgIC50aGVuKCh2YWx1ZSkgPT4gewogICAgICAgIGNvbnNvbGUubG9nKCdbOV0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICAgICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7CiAgICAgICAgcmV0dXJuICdmcm9tIFs5XSBjYWxsYmFjayc7CiAgICAgIH0pOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygnWzExXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQnKTsKICAgIGNvbnNvbGUubG9nKCdSZXNvbHZlZCB2YWx1ZTogJywgdmFsdWUpOwogIH0pOwpyZXR1cm5Qcm9taXNlKCczcmQgUHJvbWlzZScsICczJykKICAudGhlbigodmFsdWUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCdbN10gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlKTsKICAgIHJldHVybiByZXR1cm5Qcm9taXNlKCc0dGggUHJvbWlzZScsICc4JykKICAgICAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICAgICAgY29uc29sZS5sb2coJ1sxMF0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICAgICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7CiAgICAgICAgcmV0dXJuICdmcm9tIFsxMF0gY2FsbGJhY2snOwogICAgICB9KTsKICB9KQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coJ1sxMl0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCdbNF0gU3luYyBwcm9jZXNzJyk7)

# Promise チェーンのネストはアンチパターン
このように `then()` メソッドをネストさせるようなやり方は特に意味がある場合を覗いて、流れがわかりづらくなってしまうので通常は避けるべきアンチパターンとなります。このネストはフラットにでき、Promise チェーンはなるべくネストが浅くなるようにフラットにするのが推奨されます。

実際にネストを解消してみます。

```diff js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "[6]")
-     .then((value) => {
-       console.log("[9] This line is Asynchronously executed");
-       console.log("Resolved value: ", value);
-       return "from [9] callback";
-     });
  })
+ .then((value) => {
+   console.log("[9] This line is Asynchronously executed");
+   console.log("Resolved value: ", value);
+   return "from [9] callback";
+ })
  .then((value) => {
    console.log("[11] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
returnPromise("3rd Promise", "[3]")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("4th Promise", "[8]")
-     .then((value) => {
-       console.log("[10] This line is Asynchronously executed");
-       console.log("Resolved value: ", value);
-       return "from [10] callback";
-     });
  })
+ .then((value) => {
+   console.log("[10] This line is Asynchronously executed");
+   console.log("Resolved value: ", value);
+   return "from [10] callback";
+ })
  .then((value) => {
    console.log("[12] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[4] Sync process");
```

結果はこのようになり、ネストした状態のものよりも圧倒的に見やすく、流れが分かりなりました。

```js
// promiseNestShallow.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "[6]")
  })
  .then((value) => {
    console.log("[9] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return "from [9] callback";
  })
  .then((value) => {
    console.log("[11] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
returnPromise("3rd Promise", "[3]")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("4th Promise", "[8]")
  })
  .then((value) => {
    console.log("[10] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return "from [10] callback";
  })
  .then((value) => {
    console.log("[12] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[4] Sync process");
```

出力結果は全く同じになります。

```sh
❯ deno run promiseNestShallow.js
[1] Sync process
[2] This line is (A)Synchronously executed
[3] This line is (A)Synchronously executed
[4] Sync process
[5] This line is Asynchronously executed
Resolved value:  1st Promise
[6] This line is (A)Synchronously executed
[7] This line is Asynchronously executed
Resolved value:  3rd Promise
[8] This line is (A)Synchronously executed
[9] This line is Asynchronously executed
Resolved value:  2nd Promise
[10] This line is Asynchronously executed
Resolved value:  4th Promise
[11] This line is Asynchronously executed
Resolved value:  from [9] callback
[12] This line is Asynchronously executed
Resolved value:  from [10] callback
```

Promise チェーンはこのようにネストさせずに流れを見やすくします。

# fetch の例
これまでの話でどんな意味があるかを非同期 API の具体例で見てます。

`fetch()` メソッドは Web API の一部である Fetch API のメソッドであり、Promise-based な非同期 API、つまり「**処理の結果として Promise を返す API**」です。

`fetch()` メソッドは Web API であり、ECMAScript の一部ではないので注意してください。

:::message
`fetch()` メソッドの例は『then メソッドのコールバックで非同期処理』のチャプターで解説しましたので、そちらを参考にしてください。
:::

