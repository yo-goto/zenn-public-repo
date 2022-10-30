---
title: "Promise chain で値を繋ぐ"
aliases: [ch_Promise chain で値を繋ぐ]
---

# このチャプターについて

前のチャプターを通して、Promise chain の基本的な動きが分かったと思います。ここからは値を Promise chain において値をつないでいく処理を考えてみたいと思います。

# 次のチェーンに値を繋ぐ

`then()` メソッドの引数のコールバックには入力として前の `then()` メソッドのコールバック内にて `return` した値を渡すことができます。

先程のコードを更に改造して、実際にテストしてみます。再び実行の順番を予想してみてください。

```js
// returnPromiseByFuncArg2AddChainValue.js
console.log("🦖 [A] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} Sync`);
    resolve(resolvedValue);
  });
};

returnPromise("🐵 1st Promise", "[B]")
  .then((value1) => {
    console.log("👦 [C]", value1);
    return "value from 1st then";
  })
  .then((value2) => {
    console.log("👦 [D]", value2);
    // return "value from 2nd then";
  })
  .then((value3) => {
    console.log("👦 [E]", value3);
  });

returnPromise("🐵 2nd Promise", "[F]")
  .then((value1) => {
    console.log("👦 [G]", value1);
    return "value from 1st then";
  })
  .then((value2) => {
    console.log("👦 [H]", value2);
    return "value from 2nd then";
  })
  .then((value3) => {
    console.log("👦 [I]", value3);
  });

console.log("🦖 [J] MAINLINE(End): Sync");
```

:::details 答え
答えは以下のようになります。

```sh
❯ deno run returnPromiseByFuncArg2AddChainValue.js
🦖 [A] MAINLINE(Start): Sync
👻 [B] Sync
👻 [F] Sync
🦖 [J] MAINLINE(End): Sync
👦 [C] 🐵 1st Promise
👦 [G] 🐵 2nd Promise
👦 [D] value from 1st then
👦 [H] value from 1st then
👦 [E] undefined
👦 [I] value from 2nd then
```

アルファベットに数字をつけてみると分かりやすくなります。

```js
// returnPromiseByFuncArg2AddChainValue-num.js
console.log("🦖 [1] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} Sync`);
    resolve(resolvedValue);
  });
};

returnPromise("🐵 1st Promise", "[2]")
  .then((value1) => {
    console.log("👦 [5]", value1);
    return "value from 1st then";
  })
  .then((value2) => {
    console.log("👦 [7]", value2);
    // return "value from 2nd then";
  })
  .then((value3) => {
    console.log("👦 [9]", value3);
  });

returnPromise("🐵 2nd Promise", "[3]")
  .then((value1) => {
    console.log("👦 [6]", value1);
    return "value from 1st then";
  })
  .then((value2) => {
    console.log("👦 [8]", value2);
    return "value from 2nd then";
  })
  .then((value3) => {
    console.log("👦 [10]", value3);
  });

console.log("🦖 [4] MAINLINE(End): Sync");
```
:::

動きは前のコードと同じなので解説はしません。JS Visualizer 9000 で可視化したものは以下です。

- [returnPromiseByFuncArg2AddChainValue.js](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUJ5RnVuY0FyZzJBZGRDaGFpblZhbHVlLW51bU5PLmpzCmNvbnNvbGUubG9nKCJbMV0gTUFJTkxJTkUoU3RhcnQpOiBTeW5jIFByb2Nlc3MiKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGAke29yZGVyfSBTeW5jIFByb2Nlc3NgKTsKICAgIHJlc29sdmUocmVzb2x2ZWRWYWx1ZSk7CiAgfSk7Cn07CgpyZXR1cm5Qcm9taXNlKCIxc3QgUHJvbWlzZSIsICJbMl0iKQogIC50aGVuKCh2YWx1ZTEpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCJbNV06IiwgdmFsdWUxKTsKICAgIHJldHVybiAidmFsdWUgZnJvbSAxc3QgdGhlbiI7CiAgfSkKICAudGhlbigodmFsdWUyKSA9PiB7CiAgICBjb25zb2xlLmxvZygiWzddOiIsIHZhbHVlMik7CiAgICAvLyByZXR1cm4gInZhbHVlIGZyb20gMm5kIHRoZW4iOwogIH0pCiAgLnRoZW4oKHZhbHVlMykgPT4gewogICAgY29uc29sZS5sb2coIls5XToiLCB2YWx1ZTMpOwogIH0pOwoKcmV0dXJuUHJvbWlzZSgiMm5kIFByb21pc2UiLCAiWzNdIikKICAudGhlbigodmFsdWUxKSA9PiB7CiAgICBjb25zb2xlLmxvZygiWzZdOiIsIHZhbHVlMSk7CiAgICByZXR1cm4gInZhbHVlIGZyb20gMXN0IHRoZW4iOwogIH0pCiAgLnRoZW4oKHZhbHVlMikgPT4gewogICAgY29uc29sZS5sb2coIls4XToiLCB2YWx1ZTIpOwogICAgcmV0dXJuICJ2YWx1ZSBmcm9tIDJuZCB0aGVuIjsKICB9KQogIC50aGVuKCh2YWx1ZTMpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCJbMTBdOiIsIHZhbHVlMyk7CiAgfSk7Cgpjb25zb2xlLmxvZygiWzRdIE1BSU5MSU5FKEVuZCk6IFN5bmMgUHJvY2VzcyIpOwo%3D)
- ⚠️ 注意: JS Visuzlizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク実行のタイミングについて誤解しないように注意してください

ポイントとしては、`return` 文をコメントアウトしてある `then()` コールバックの次の `then()` コールバックでは、渡されるはずの値がないので `undefined` となっている点です。何も `return` しない場合には次の `then()` メソッドのコールバックの入力値は `undefined` となるので注意してください。

# チェーンの最後まで値を繋ぐ

Promise chain で「値を繋ぐ」ことが理解しづらい場合には次のコードを考えてみます。このコードでは、`returnPromise()` 関数の第一引数として渡した文字列 `"1st Promise"` を Promise chain において `then()` メソッドのコールバックで毎回 `return` することがで最後まで値を繋げています。

```js:chainValue.js
// chainValue.js
console.log("🦖 [1] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} Sync`);
    resolve(resolvedValue);
  });
};

// 文字列 "🐵 1st Promise" で解決された後にその値を最後まで連鎖させる
returnPromise("🐵 1st Promise", "[2]")
  .then((value1) => {
    console.log("👦 [4]", value1); // 🐵 1st Promise
    return value1;
  })
  .then((value2) => {
    console.log("👦 [5]", value2); // 🐵 1st Promise
    return value2;
  })
  .then((value3) => {
    console.log("👦 [6]", value3); // 🐵 1st Promise
    return value3;
  })
  .then((value4) => {
    console.log("👦 [7]", value4); // 🐵 1st Promise
  });

console.log("🦖 [3] MAINLINE(End): Sync");
```

これを実行すると以下の出力を得ます。

```sh
❯ deno run chainValue.js
🦖 [1] MAINLINE(Start): Sync
👻 [2] Sync
🦖 [3] MAINLINE(End): Sync
👦 [4] 🐵 1st Promise
👦 [5] 🐵 1st Promise
👦 [6] 🐵 1st Promise
👦 [7] 🐵 1st Promise
```

value1 → value2 → value3 → vlaue4 というように値 `"1st Promise"` が最後まで連鎖できていることに注目してください。

`then()` メソッドのコールバック関数の引数に数値をつけたのは考えやすくするためで別に同じ `value` でもよいです。

```js
// chainValueName.js
console.log("🦖 [1] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} Sync`);
    resolve(resolvedValue);
  });
};

// 文字列 "🐵 1st Promise" で解決された後にその値を最後まで連鎖させる
returnPromise("🐵 1st Promise", "[2]")
  .then((value) => {
    console.log("👦 [4]", value); // 🐵 1st Promise
    return value;
  })
  .then((value) => {
    console.log("👦 [5]", value); // 🐵 1st Promise
    return value;
  })
  .then((value) => {
    console.log("👦 [6]", value); // 🐵 1st Promise
    return value;
  })
  .then((value) => {
    console.log("👦 [7]", value); // 🐵 1st Promise
  });

console.log("🦖 [3] MAINLINE(End): Sync");
```

