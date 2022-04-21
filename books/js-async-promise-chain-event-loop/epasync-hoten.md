---
title: "補填"
---

チャプター１０から移植した。

次のコードでは、今までのコードでメインとなる Promise チェーンを１つにした上で、`returnPromise()` 関数内で Promise チェーンを行うように改造しました。つまり Promise チェーンをネストさせています。

Promise インスタンスを返す処理は常に `return` するべきですが、このコードではあえて `return` させていません。

さて実行順番とアルファベット `[A-H]` の出力順番はどうなるでしょうか？予測してみてください。

```js
// promiseShouldBeReturnedNest.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order, nextOrder) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
    // ↓ ここにチェーンを追加してみる
  }).then((value) => {
    console.log(`${nextOrder} Additional nested chain`);
    return value;
  });
};

returnPromise("1st Promise", "[B]", "[C]")
  .then((value) => {
    console.log("[D] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // ここで敢えて return しないとどういう実行順番になるか?
    returnPromise("2nd Promise", "[E]", "[F]");
  })
  .then((value) => {
    console.log("[G] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[H] Sync process");
```

これはかなり難しいです。

:::details 答え
答えは、「A → B → H → C → D → E → F → G」となります。

数字付きで実際に出力してみるとこうなります。
```sh
❯ deno run promiseShouldBeReturnedNest-wrong.js
[A-1] Sync process
[B-2] This line is (A)Synchronously executed
[H-3] Sync process
[C-4] Additional nested chain
[D-5] This line is Asynchronously executed
Resolved value:  1st Promise
[E-6] This line is (A)Synchronously executed
[F-7] Additional nested chain
[G-8] This line is Asynchronously executed
Resolved value:  undefined
```
:::

最後の出力である `Resovled value` のところが `undefined` になっているので、値 `"2nd Promsie"` が繋げていないことがわかります。

そして問題なのは実行順番です。次のような順番で実行されると予想したのではないでしょうか？(~~実際には違います~~)

正解!

```js
// promiseShouldBeReturnedNest.js
console.log("[A-1] Sync process");

const returnPromise = (resolvedValue, order, nextOrder) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  }).then((value) => {
    console.log(`${nextOrder} Additional nested chain`);
    return value;
  });
};

returnPromise("1st Promise", "[B-2]", "[C-4]")
  .then((value) => {
    console.log("[D-5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // ここで敢えて return しないとどういう実行順番になるか?
    returnPromise("2nd Promise", "[E-6]", "[F-7]");
  })
  .then((value) => {
    console.log("[G-8] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[H-3] Sync process");
```

次の項目で説明しますが、Promise インスタンスを返す処理については `return` しないと、その処理が完了してから次の `then()` メソッドのコールバック処理を行うということが保証できません。ただし、上の場合は実はできている。Microtask の実行ステップであるためにマイクロタスクキューにいれられたすぐに実行される。

`new Prommise(executor)` の引数であるコールバック関数 `executor` 自体の処理は同期的に行わたとしても、その `executor` の中で非同期処理があったり、上の `returnPromise()` 関数の場合のように Promise チェーンがネストされていた場合には、`return` をつけないとそれらの処理は後回しになってしまいます。
