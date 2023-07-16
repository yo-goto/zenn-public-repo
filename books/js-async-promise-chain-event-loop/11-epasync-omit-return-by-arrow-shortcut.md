---
title: "アロー関数で return を省略する"
cssclass: zenn
date: 2022-04-17
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『アロー関数で return を省略する』
---

## このチャプターについて

非同期処理を理解するために必ずしも必要でありませんが、実用上必要となるのはアロー関数であり、その省略形の話は知っておかないと損なので１つのチャプターとして独立させておきます。

アロー関数については、『コールバック関数の同期実行と非同期実行』で解説してあったので Promise chain でそれを使ってみたいと思います。

## アロー関数で return を省略する

アロー関数の省略形について、もう一度確認しておきます。アロー関数の省略は以下のようにでき、下の３つのコードはすべて等価となります。

```js
(a) => {
  return a + 100;
}
// 等価
(a) => a + 100;
// 等価
a => a + 100;
```

アロー関数の省略形を使用することで、`return` を書かずに `return` させることができます。これは他人のソースコードを読んだりするときに役立ったり、文字数を減らして書くのに役立ちます。

次のように、Promise インスタンスを返す処理は `then()` メソッドのコールバック関数においてアロー関数の省略形を使って書くことができます。

```js
// promiseShouldBeReturned.js
console.log("🦖 [1] Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};

// アロー関数の省略形を使って return を省略する
returnPromise("1st Promise", "[2]")
  .then(() => returnPromise("2nd Promise", "[6]"))
  .then((value) => console.log("👦 Resolved value: ", value));
// 同じ意味
returnPromise("3rd Promise", "[3]")
  .then(() => {
    // Promise インスタンスについては必ず return するようにする
    return returnPromise("4th Promise", "[8]");
  })
  .then((value) => console.log("👦 Resolved value: ", value));

console.log("🦖 [4] Sync");
```

`.then((value) => console.log("👦 Resolved value: ", value));` については、`console.log()` の返り値は `undefined` となるので、次の `then()` メソッドのコールバックに値を渡す必要がなければやっても大丈夫です。

値を繋ぐ際には、この「アロー関数の省略形」を意識しておくとよいです。次のコードは『[Promise chain で値を繋ぐ](7-epasync-pass-value-to-the-next-chain)』のチャプターの最後に見ましたが、ちょっと `console.log()` を抜いて改造してみます。

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

↓ このコードから `then()` メソッドのコールバック関数内の `console.log()` をなくして値だけを繋いでみます。

```js
// chainValueNameArrow.js
console.log("🦖 [1] Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};
returnPromise("1st Promise", "[2]")
  .then((value) => value)
  .then((value) => value)
  .then((value) => value)
  .then((value) => console.log("👦 [last] Resolved value: ", value));
  // 文字列 "1st Promise" を最後までつなげる

console.log("🦖 [3] Sync");
```

コード自体に特別な意味は無いですが、アロー関数の省略形でこのようなことができるということを意識するためにやっています。これを実行すると次の出力を得ます。値が最後まで連鎖できていることがわかります。

```sh
❯ deno run chainValueNameArrow.js
🦖 [1] Sync
👻 [2] (a)sync
🦖 [3] Sync
👦 [last] Resolved value: 1st Promise
```
