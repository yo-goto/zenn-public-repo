---
title: "Promise チェーンで値を繋ぐ"
---

# 次のチェーンに値を繋ぐ
さて、Promsise チェーンの基本的な動きが分かったと思います。

ここからは値を Promise チェーンにおいて値をつないでいく処理を考えてみたいと思います。
then()` メソッドの引数のコールバックには入力として前の `then()` メソッドのコールバック内にて `return` した値を渡すことができます。

先程のコードを更に改造して、実際にテストしてみます。再び実行の順番を予想してみてください。

```js
// returnPromiseByFuncArg2AddChainValue.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[B]")
  .then((value1) => {
    console.log("[C] This line is Asynchronously executed");
    console.log("Resolved value: ", value1);
    return "Resolved value passing to the next then callback";
  })
  .then((value2) => {
    console.log("[D] This line is Asynchronously executed");
    console.log("Resolved value: ", value2);
    return "Resolved value passing to the next then callback";
  })
  .then((value3) => {
    console.log("[E] This line is Asynchronously executed");
    console.log("Resolved value: ", value3);
    // return "Resolved value passing to the next then callback";
  })
  .then((value4) => {
    console.log("[F] This line is Asynchronously executed");
    console.log("Resolved value: ", value4);
  });
returnPromise("2nd Promise", "[G]")
  .then((value1) => {
    console.log("[H] This line is Asynchronously executed");
    console.log("Resolved value: ", value1);
    return "Resolved value passing to the next then callback";
  })
  .then((value2) => {
    console.log("[I] This line is Asynchronously executed");
    console.log("Resolved value: ", value2);
    return "Resolved value passing to the next then callback";
  })
  .then((value3) => {
    console.log("[J] This line is Asynchronously executed");
    console.log("Resolved value: ", value3);
    // return "Resolved value passing to the next then callback";
  })
  .then((value4) => {
    console.log("[K] This line is Asynchronously executed");
    console.log("Resolved value: ", value4);
    return "Resolved value passing to the next then callback";
  });

console.log("[L] Sync process");
```

:::details 答え
答えは、「A → B → G → L → C → H → D → I → E → J → F → K」となります。

```sh
❯ deno run returnPromiseByFuncArg2AddChainValue.js
[A] Sync process
[B] This line is Synchronously executed
[G] This line is Synchronously executed
[L] Sync process
[C] This line is Asynchronously executed
Resolved value:  1st Promise
[H] This line is Asynchronously executed
Resolved value:  2nd Promise
[D] This line is Asynchronously executed
Resolved value:  Resolved value passing to the next then callback
[I] This line is Asynchronously executed
Resolved value:  Resolved value passing to the next then callback
[E] This line is Asynchronously executed
Resolved value:  Resolved value passing to the next then callback
[J] This line is Asynchronously executed
Resolved value:  Resolved value passing to the next then callback
[F] This line is Asynchronously executed
Resolved value:  undefined
[K] This line is Asynchronously executed
Resolved value:  undefined
```

アルファベットに数字をつけてみると分かりやすくなります。

```js
// returnPromiseByFuncArg2AddChainValue.js
console.log("[A-1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[B-2]")
  .then((value1) => {
    console.log("[C-5] This line is Asynchronously executed");
    console.log("Resolved value: ", value1);
    return "Resolved value passing to the next then callback";
  })
  .then((value2) => {
    console.log("[D-7] This line is Asynchronously executed");
    console.log("Resolved value: ", value2);
    return "Resolved value passing to the next then callback";
  })
  .then((value3) => {
    console.log("[E-9] This line is Asynchronously executed");
    console.log("Resolved value: ", value3);
    // return "Resolved value passing to the next then callback";
  })
  .then((value4) => {
    console.log("<F-11> This line is Asynchronously executed");
    console.log("Resolved value: ", value4); // undefined
  });
returnPromise("1st Promise", "[G-3]")
  .then((value1) => {
    console.log("[H-6] This line is Asynchronously executed");
    console.log("Resolved value: ", value1);
    return "Resolved value passing to the next then callback";
  })
  .then((value2) => {
    console.log("[I-8] This line is Asynchronously executed");
    console.log("Resolved value: ", value2);
    return "Resolved value passing to the next then callback";
  })
  .then((value3) => {
    console.log("[J-10] This line is Asynchronously executed");
    console.log("Resolved value: ", value3);
    // return "Resolved value passing to the next then callback";
  })
  .then((value4) => {
    console.log("[K-12] This line is Asynchronously executed");
    console.log("Resolved value: ", value4); // undefined
  });

console.log("[L-4] Sync process");
```
:::

動きは前のコードと同じなので解説はしません。JS Visualizer 9000 で可視化したものは以下です。

- [returnPromiseByFuncArg2AddChainValue.js](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUJ5RnVuY0FyZzJBZGRDaGFpblZhbHVlLmpzCmNvbnNvbGUubG9nKCdbQS0xXSBTeW5jIHByb2Nlc3MnKTsKY29uc3QgcmV0dXJuUHJvbWlzZSA9IChyZXNvbHZlZFZhbHVlLCBvcmRlcikgPT4gewogIHJldHVybiBuZXcgUHJvbWlzZSgocmVzb2x2ZSkgPT4gewogICAgY29uc29sZS5sb2coYFske29yZGVyfV0gVGhpcyBsaW5lIGlzIFN5bmNocm9ub3VzbHkgZXhlY3V0ZWRgKTsKICAgIHJlc29sdmUocmVzb2x2ZWRWYWx1ZSk7CiAgfSk7Cn07CnJldHVyblByb21pc2UoJzFzdCBQcm9taXNlJywgJ0ItMicpCiAgLnRoZW4oKHZhbHVlMSkgPT4gewogICAgY29uc29sZS5sb2coJ1tDLTVdIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCcpOwogICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZTEpOwogICAgcmV0dXJuICdSZXNvbHZlZCB2YWx1ZSBwYXNzaW5nIHRvIHRoZSBuZXh0IHRoZW4gY2FsbGJhY2snOwogIH0pCiAgLnRoZW4oKHZhbHVlMikgPT4gewogICAgY29uc29sZS5sb2coJ1tELTddIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCcpOwogICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZTIpOwogICAgcmV0dXJuICdSZXNvbHZlZCB2YWx1ZSBwYXNzaW5nIHRvIHRoZSBuZXh0IHRoZW4gY2FsbGJhY2snOwogIH0pCiAgLnRoZW4oKHZhbHVlMykgPT4gewogICAgY29uc29sZS5sb2coJ1tFLTldIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCcpOwogICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZTMpOwogIH0pCiAgLnRoZW4oKHZhbHVlNCkgPT4gewogICAgY29uc29sZS5sb2coJzxGLTExPiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQnKTsKICAgIGNvbnNvbGUubG9nKCdSZXNvbHZlZCB2YWx1ZTogJywgdmFsdWU0KTsKICB9KTsKcmV0dXJuUHJvbWlzZSgnMXN0IFByb21pc2UnLCAnRy0zJykKICAudGhlbigodmFsdWUxKSA9PiB7CiAgICBjb25zb2xlLmxvZygnW0gtNl0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlMSk7CiAgICByZXR1cm4gJ1Jlc29sdmVkIHZhbHVlIHBhc3NpbmcgdG8gdGhlIG5leHQgdGhlbiBjYWxsYmFjayc7CiAgfSkKICAudGhlbigodmFsdWUyKSA9PiB7CiAgICBjb25zb2xlLmxvZygnW0ktOF0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlMik7CiAgICByZXR1cm4gJ1Jlc29sdmVkIHZhbHVlIHBhc3NpbmcgdG8gdGhlIG5leHQgdGhlbiBjYWxsYmFjayc7CiAgfSkKICAudGhlbigodmFsdWUzKSA9PiB7CiAgICBjb25zb2xlLmxvZygnW0otMTBdIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCcpOwogICAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZTMpOwogIH0pCiAgLnRoZW4oKHZhbHVlNCkgPT4gewogICAgY29uc29sZS5sb2coJ1tLLTEyXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQnKTsKICAgIGNvbnNvbGUubG9nKCdSZXNvbHZlZCB2YWx1ZTogJywgdmFsdWU0KTsKICAgIH0pOwogICAgCmNvbnNvbGUubG9nKCdbTC00XSBTeW5jIHByb2Nlc3MnKTsKLy8gRW5kCg%3D%3D)

ポイントとしては、`return` 文をコメントアウトしてある `then()` コールバックの次の `then()` コールバックでは、渡されるはずの値がないので `undefined` となっている点です。何も `return` しない場合には次の `then()` メソッドのコールバックの入力値は `undefined` となるので注意してください。

# チェーンの最後まで値を繋ぐ
Promise チェーンで「値を繋ぐ」ことが理解しづらい場合には次のコードを考えてみます。このコードでは、`returnPromise()` 関数の第一引数として渡した文字列 `"1st Promise"` を Promise チェーンにおいて `then()` メソッドのコールバックで毎回 `return` することがで最後まで値を繋げています。


```js
// chainValue.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

// 文字列 "1st Promise" で解決された後にその値を最後まで連鎖させる
returnPromise("1st Promise", "[2]")
  .then((value1) => {
    console.log("[4] This line is Asynchronously executed");
    console.log("Resolved value: ", value1); // 1st Promise
    return value1;
  })
  .then((value2) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value2); // 1st Promise
    return value2;
  })
  .then((value3) => {
    console.log("[6] This line is Asynchronously executed");
    console.log("Resolved value: ", value3); // 1st Promise
    return value3;
  })
  .then((value4) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value4); // 1st Promise
  });

console.log("[3] Sync process");
```

これを実行すると以下の出力を得ます。

```sh
❯ deno run chainValue.js
[1] Sync process
[2] This line is Synchronously executed
[3] Sync process
[4] This line is Asynchronously executed
Resolved value:  1st Promise
[5] This line is Asynchronously executed
Resolved value:  1st Promise
[6] This line is Asynchronously executed
Resolved value:  1st Promise
[7] This line is Asynchronously executed
Resolved value:  1st Promise
```

value1 → value2 → value3 → vlaue4 というように値 `"1st Promise"` が最後まで連鎖できていることに注目してください。

`then()` メソッドのコールバック関数の引数に数値をつけたのは考えやすくするためで別に同じ `value` でもよいです。

```js
// chainValueName.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};
returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
    return value;
  })
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
    return value;
  })
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
    return value;
  })
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
  });

console.log("[3] Sync process");
```

