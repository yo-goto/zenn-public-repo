---
title: "then メソッドのコールバックで Promise インスタンスを返す"
cssclass: zenn
date: 2022-04-17
modified: 2024-08-14
AutoNoteMover: disable
tags: type/zenn/book, JavaScript/async
aliases: Promise本『then メソッドのコールバックで Promise インスタンスを返す』
---

## このチャプターについて

このチャプターは短いですが、Promise chain を理解する上で重要なので１つのチャプターとして独立させています。チャプターは飛びますが、『[コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)』のチャプターでも使う知識なので注意してください。

:::message alert
このチャプターの解説は『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで解説した「内容の間違い」の影響を以前まで受けていましたが、現在は内容を修正・補足しました。
:::

## then メソッドのコールバックで Promise インスタンスを返す

前のチャプターのコードでは `then()` メソッドのコールバックにおいて `return` で返却していたのは `"Resolved value passing to the next then callback"` という文字列でした。

`return` する値は文字列に関わらず数値や真偽値でも良いのですが、Promise インスタンスを返した場合はどうなるでしょうか？

その答えは「**渡された Promise インスタンスの `resolve` に使われた値が次の `then()` メソッドのコールバック関数の引数として渡される**」です。実際に `then()` メソッドのコールバック関数において新しい Promise インスタンスを返してみます。今までのコードをまた流用しますが、まずは Promise chain を１つにして考えてみます。

```js:rP1.js
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
    console.log("👦 [4] async:", value);
    return returnPromise("🐵 2nd Promise", "[5]");
    // resolve される値は "2nd Promise" で
    // これが次の then() のコールバック関数の入力として渡される
  })
  .then((value) => {
    console.log("👦 [6] async:", value);
  });

console.log("🦖 [3] MAINLINE(End): Sync");
```

今まで必ず同期処理として呼ばれていた `returnPromise()` 関数ですが、今回は `then()` メソッドのコールバックで呼び出しているものもあるのでそれらは非同期的に実行されます。従って出力されるテキストを一部変更しました。

これを実行するとこれまでの Promise chain と同じように作業 A → B → C というように順序が担保されて `then` に登録されたコールバックが実行されます。

```sh
❯ deno run rp1.js
🦖 [1] MAINLINE(Start): Sync
👻 [2] (a)sync
🦖 [3] MAINLINE(End): Sync
👦 [4] async: 🐵 1st Promise
👻 [5] (a)sync
👦 [6] async: 🐵 2nd Promise
```

ここまでは予想通りだと思いますが、内部的に発生するマイクロタスクについて注意することがあるので解説します。問題を簡単にするために簡素化した以下のコードで考えます。

```js:Promise を返す場合
Promise.resolve(42)
  .then(x => {
    return Promise.resolve(x + 1);
  })
  .then(x => { // 43 が入力値として渡る
    console.log(x);
    //          ^ : 43
  });
```

`then` のコールバック関数で Promise インスタンスを返した時に、その Promise インスタンスが内包する値 (履行値や拒否理由) は `then` や `catch` などのコールバックの入力値として渡りますが、数値や文字列などの通常の値をコールバックから返した時に比べて追加で２つマイクロタスクが発生することになります。

まずは、通常の値を返した場合に発生するマイクロタスクを追跡してみます。

```js:通常の値を返す場合
/* <n> は発生しているマイクロタスクの追跡順番 */
Promise.resolve(42)
  .then(x => { // <1> このコールバック関数が１つ目のマイクロタスク
    return x + 1;
  })
  .then(x => { // <2> このコールバック関数が２つ目のマイクロタスク
    console.log(x);
  });
```

この場合には単純に `then` に登録してあるコールバック関数だけがマイクロタスクとして発行されています。

コールバックから Promise が返される場合に発生するマイクロタスクを追跡すると、以下のように追加でマイクロタスクが２つ発生しています。

```js:Promise を返す場合
/* <n> は発生しているマイクロタスクの追跡順番 */
Promise.resolve(42)
  .then(x => { // <1> このコールバック関数が１つ目のマイクロタスク
    return Promise.resolve(x + 1);
    // <2> Promise.resolve(43).then(resolve, reject) の呼び出し
    // <3> resolve の実行
  })
  .then(x => { // <4> このコールバック関数が４つ目のマイクロタスク
    console.log(x);
  });
```

`return Promise.resolve(x + 1)` という処理で、Promise インスタンスが返されると、そのコールバックが登録されている `then` メソッド自体から返る Promise インスタンスを解決するために必要なマイクロタスクが２つ発生してしまいます。このマイクロタスクの実体は追加の１つ目についてはコールバックから返した Promise インスタンスの `then` メソッドの呼び出しで、追加の２つ目については呼び出された `then` メソッドの `resolve` 関数となります。

この段階では何を言っているかのよくわからない思いますが、とにかく、`then` メソッドのコールバックで Promise を返すと追加のマイクロタスクが２つ発生するということだけ覚えておいてください。この挙動についての詳細は『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで解説します。

このように普通の場合に比べて Promise を返すコールバックの実行ではマイクロタスクが追加で２つ発生することから以下のような両方のケースの Promise chain を競争させると通常の値を返している方の Promise chain の処理の方が早く終ることになります。

```js:valueType.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: どちらの promise chain かの識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [1]");

Promise.resolve(1)
  .then(x => { // <1-a[1]>
    return Promise.resolve(x + 1);
    // <3-a[2]>
    // <5-a[3]>
  })
  .then(x => console.log("🔥 [4]", x)); // <6-a[1]>

Promise.resolve(1)
  .then(x => { // <2-b[1]>
    return x + 1;
  })
  .then(x => console.log("🦄 [3]", x)); // <4-b[1]>

console.log("🦖 [2]");
```

実行結果は以下で、先に実行したはずの Promise chain (コールバックで Promise を返している方) が後に完了しています。

```sh
❯ deno run valueType.js
🦖 [1]
🦖 [2]
🦄 [3] 2
🔥 [4] 2
```

これを踏まえて練習用にコールバックから Promise を返すタイプの Promise chain を２つ競争させるコードを用意しました。この実行順序を予測してみてください。

```js
// returnPromiseFromThenCallback.js
console.log("🦖 [A] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise("🐵 1st Promise", "[B]")
  .then((value) => {
    console.log("👦 [C] async:", value);
    return returnPromise("🐵 2nd Promise", "[D]");
  })
  .then((value) => {
    console.log("👦 [E] async:", value);
  });

returnPromise("🐵 3rd Promise", "[F]")
  .then((value) => {
    console.log("👦 [G] async:", value);
    return returnPromise("🐵 4th Promise", "[H]");
  })
  .then((value) => {
    console.log("👦 [I] async:", value);
  });

console.log("🦖 [J] MAINLINE(End): Sync");
```

このままスクロールせずに実行結果を予測してください。

:::details 実行結果
これを実行すると、次のような出力になります。

```sh
❯ deno run returnPromiseFromThenCallback.js
🦖 [A] MAINLINE(Start): Sync
👻 [B] (a)sync
👻 [F] (a)sync
🦖 [J] MAINLINE(End): Sync
👦 [C] async: 🐵 1st Promise
👻 [D] (a)sync
👦 [G] async: 🐵 3rd Promise
👻 [H] (a)sync
👦 [E] async: 🐵 2nd Promise
👦 [I] async: 🐵 4th Promise
```
:::

さて、発生するマイクロタスクを追跡してみると出力順序が理解できるはず。

```js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: どちらの promise chain かの識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [A-1] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise("🐵 1st Promise", "[B-2]")
  .then((value) => { // <1-a[1]>
    console.log("👦 [C-5] async:", value);
    return returnPromise("🐵 2nd Promise", "[D-6]");
    // Promise を返しているため発生するマイクロタスクは追加で２つ発生する
    // <3-a[2]>
    // <5-a[3]>
  })
  .then((value) => { // <7-a[4]>
    console.log("👦 [E-9] async:", value);
  });

returnPromise("🐵 3rd Promise", "[F-3]")
  .then((value) => { // <2-b[1]>
    console.log("👦 [G-7] async:", value);
    return returnPromise("🐵 4th Promise", "[H-8]");
    // Promise を返しているため発生するマイクロタスクは追加で２つ発生する
    // <4-b[2]>
    // <6-b[3]>
  })
  .then((value) => { // <8-b[4]>
    console.log("👦 [I-10] async:", value);
  });

console.log("🦖 [J-4] MAINLINE(End): Sync");
```

両者ともコールバックで Promise を返しているため、競争させても同じ数のマイクロタスクが発生するので交互に出力が起きるだけです。

```sh
❯ deno run returnPromiseFromThenCallback.js
🦖 [A-1] MAINLINE(Start): Sync
👻 [B-2] (a)sync
👻 [F-3] (a)sync
🦖 [J-4] MAINLINE(End): Sync
👦 [C-5] async: 🐵 1st Promise
👻 [D-6] (a)sync
👦 [G-7] async: 🐵 3rd Promise
👻 [H-8] (a)sync
👦 [E-9] async: 🐵 2nd Promise
👦 [I-10] async: 🐵 4th Promise
```

今度は通常の値を返すコールバックの Promise chain と Promise を返す Promise chain で競争させてみます。再び出力順番を予想してみてください。

```js
// race2chain.js
console.log("🦖 [A] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise("🐵 1st Promise", "[B]")
  .then((value) => {
    console.log("👦 [C] async:", value);
    return returnPromise("🐵 2nd Promise", "[D]");
  })
  .then((value) => {
    console.log("👦 [E] async:", value);
  });

returnPromise("🐵 3rd Promise", "[F]")
  .then((value) => {
    console.log("👦 [G] async:", value);
    return "Normal value"
  })
  .then((value) => {
    console.log("👦 [H] async:", value);
  });

console.log("🦖 [I] MAINLINE(End): Sync");
```

このままスクロールせずに実行結果を予測してください。

:::details 実行結果
これを実行すると、次のような出力になります。

```sh
❯ deno run race2chain.js
🦖 [A] MAINLINE(Start): Sync
👻 [B] (a)sync
👻 [F] (a)sync
🦖 [I] MAINLINE(End): Sync
👦 [C] async: 🐵 1st Promise
👻 [D] (a)sync
👦 [G] async: 🐵 3rd Promise
👦 [H] async: Normal value
👦 [E] async: 🐵 2nd Promise
```
:::

さて、これもマイクロタスクの追跡を行ってみます。

```js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: どちらの promise chain かの識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [A-1] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise("🐵 1st Promise", "[B-2]")
  .then((value) => { // <1-a[1]>
    console.log("👦 [C-5] async:", value);
    return returnPromise("🐵 2nd Promise", "[D-6]");
    // Promise を返しているため発生するマイクロタスクは追加で２つ発生する
    // <3-a[2]>
    // <5-a[3]>
  })
  .then((value) => { // <6-a[4]>
    console.log("👦 [E-9] async:", value);
  });

returnPromise("🐵 3rd Promise", "[F-3]")
  .then((value) => { // <2-b[2]>
    console.log("👦 [G-7] async:", value);
    return "Normal value"
    // 通常の値を返しているため発生するマイクロタスクは１つだけ
  })
  .then((value) => { // <4-b[3]>
    console.log("👦 [H-8] async:", value);
  });

console.log("🦖 [I-4] MAINLINE(End): Sync");
```

実行結果は以下のようになります。

```sh
❯ deno run race2chain.js
🦖 [A-1] MAINLINE(Start): Sync
👻 [B-2] (a)sync
👻 [F-3] (a)sync
🦖 [I-4] MAINLINE(End): Sync
👦 [C-5] async: 🐵 1st Promise
👻 [D-6] (a)sync
👦 [G-7] async: 🐵 3rd Promise
👦 [H-8] async: Normal value
👦 [E-9] async: 🐵 2nd Promise
```

:::message alert
**JS Visualizer について**  

このチャプターの以前の解説では JS Visualizer のコードで実行順番の把握を確認してもらっていましたが、実は JS Visualizer では**可視化されないマイクロタスク**(上記で解説した２つの追加のマイクロタスク)があることが発覚しました。

出力結果自体は変わりませんが、追加で発生する可視化されないマイクロタスクがあることを知らないとなぜそのような実行順序になるか理解できないケースがあるので気をつけて使用してください。

👉 [JS Visualizer Code](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUZyb21UaGVuQ2FsbGJhY2suanMKY29uc29sZS5sb2coIlsxXSBNQUlOTElORShTdGFydCk6IHN5bmMiKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGAke29yZGVyfSAoYSlzeW5jYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pOwp9OwoKcmV0dXJuUHJvbWlzZSgiMXN0IFByb21pc2UiLCAiWzJdIikKICAudGhlbigodmFsdWUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCJbNV0gYXN5bmM6IiwgdmFsdWUpOwogICAgcmV0dXJuIHJldHVyblByb21pc2UoIjJuZCBQcm9taXNlIiwgIls2XSIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiWzldIGFzeW5jOiIsIHZhbHVlKTsKICB9KTsKcmV0dXJuUHJvbWlzZSgiM3JkIFByb21pc2UiLCAiWzNdIikKICAudGhlbigodmFsdWUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCJbN10gYXN5bmM6IiwgdmFsdWUpOwogICAgcmV0dXJuIHJldHVyblByb21pc2UoIjR0aCBQcm9taXNlIiwgIls4XSIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiWzEwXSBhc3luYzoiLCB2YWx1ZSk7CiAgfSk7Cgpjb25zb2xlLmxvZygiWzRdIE1BSU5MSU5FKEVuZCk6IHN5bmMiKTsK)
:::
