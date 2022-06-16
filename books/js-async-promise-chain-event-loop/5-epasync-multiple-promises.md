---
title: "複数の Promise を走らせる"
aliases: [ch_複数の Promise を走らせる]
---

# このチャプターについて

:::message alert
このチャプターでの古い内容に基づいた解説を新しい解説で置き換えました。実行コンテキストとマイクロタスクのチェックポイントを使って解説しています。
:::

非同期処理の学習で最も重要なことは「**制御の流れがつかめるようになること**」です。このチャプターからは、実行の順番を予測できるようになるために１つずつ処理を複雑にして Promise チェーンを解説していきます。

制御の流れを掴む上では、複数の Promise 処理を起動させてどのように処理されるかを理解することが肝になると個人的には考えています。非同期処理を単体で考えるのではなく同期処理と一緒に考える必要があるように、Promise 処理を理解するには、「単一の処理」ではなく「複数の処理」に注目して考えることが必要です。

# Promise インスタンスを返す関数
ここまで `new Promise(executor)` で Promise インスタンスを作成してきました。

今度は Promise インスタンスを返す関数をアロー関数式を使用して定義して先程のコードを改造してみます。これで Promise インスタンスを何回も作成できるようになります。また引数を渡せるようにしてその引数で Promise を解決するようにします。

```js:returnPromiseByFunc.js
// returnPromiseByFunc.js
console.log('🦖 [1] MAINLINE: Sync process');

const returnPromise = (resolveValue) => {
  return new Promise((resolve) => {
    console.log('👻 [2] This line is Synchronously executed');
    resolve(resolveValue);
  });
};

returnPromise('Resolved by function').then((value) => {
  console.log('👦 [4] This line is Asynchronously executed');
  console.log('👦 [5] Resolved value: ', value);
});

console.log('🦖 [3] MAINLINE: Sync process');
```

これは前のコードではそのまま Promise インスタンスを作成していたのを Promise インスタンスを返す関数に置き換えただけなので実行結果は以前と同じになります(履行値だけ違う)。

```sh
❯ deno run returnPromiseByFunc.js
🦖 [1] MAINLINE: Sync process
👻 [2] This line is Synchronously executed
🦖 [3] MAINLINE: Sync process
👦 [4] This line is Asynchronously executed
👦 [5] Resolved value:  Resolved by function
```

具体的には `returnPromise()` 関数は同期的に実行されて、内部の `Promise()` コンストラクタの引数であるコールバック関数もそのまま同期的に実行されます。

複数の Promise インスタンスを作成して、複数の Promise チェーンを同時に走らることを考えます。`reutrnPromise()` 関数は何度も起動させたいので、`[2]` 番目に出力される行を書き換えて引数で指定できるようにします。テンプレートリテラルで表現します(まだ複数 Promise は作成しません)。

```js:returnPromiseByFuncArg.js
// returnPromiseByFuncArg.js
console.log('🦖 [1] MAINLINE: Sync process');

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 [${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise('First Promise', 2).then((value) => {
  console.log('👦 [4] This line is Asynchronously executed');
  console.log('👦 [5] Resolved value: ', value);
});

console.log('🦖 [3] MAINLINE: Sync process');
```

テンプレートリテラルで書き換えただけなので実行結果は同じになります。

```sh
❯ deno run returnPromiseByFuncArg.js
🦖 [1] MAINLINE: Sync process
👻 [2] This line is Synchronously executed
🦖 [3] MAINLINE: Sync process
👦 [4] This line is Asynchronously executed
👦 [5] Resolved value:  First Promise
```

# 複数の Promise 処理を走らせる

それでは本題に入るための準備ができたのでが、実際に複数の Promise インスタンスを作成してみて実行の順番がどうなるかを見てみます。`[]` で囲まれた文字の順番がどのように出力されるか、ここからは自分で出力の順番を予想してみてください。

```js:returnPromiseByFuncArg2.js
// returnPromiseByFuncArg2.js
console.log('🦖 [A] MAINLINE: Sync process');

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 [${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise('1st Promise', 'B').then((value) => {
  console.log('👦 [C] This line is Asynchronously executed');
  console.log('👦 Resolved value: ', value);
});
returnPromise('2nd Promise', 'D').then((value) => {
  console.log('👦 [E] This line is Asynchronously executed');
  console.log('👦 Resolved value: ', value);
});

console.log('🦖 [F] MAINLINE: Sync process');
```

実行順番がどうなるか分かりましたか?

:::details 答え
答えは、「A → B → D → F → C → E」となります。

```sh
❯ deno run returnPromiseByFuncArg2.js
🦖 [A] MAINLINE: Sync process
👻 [B] This line is Synchronously executed
👻 [D] This line is Synchronously executed
🦖 [F] MAINLINE: Sync process
👦 [C] This line is Asynchronously executed
👦 Resolved value:  1st Promise
👦 [E] This line is Asynchronously executed
👦 Resolved value:  2nd Promise
```
:::

なぜこうなるのか考えてみます。

まずコードは上から下に実行されていきます。コードの実行において、イベントループの最初のタスクは「スクリプトの評価」です。コールスタック上には一番下にグローバルコンテキストが積まれた状態で、すべての同期処理が実行されていきます。

`returnPromise("1st Promise", "B")` は**同期処理**です。関数の中を見ても、`Promise()` コンストラクタの引数である `executor` 関数の中も同期的に実行されます。

`executor` 関数内ですぐに `resolve()` が呼び出されるので Promise インスタンスは直ちに履行状態へと移行します。`returnPromise("1st Promise", "B")` でここまでは同期的に実行されていることに注意してください。

`returnPromise("1st Promise", "B")` で返ってくる Promise インスタンスはすでに履行状態なので、直ちに `then()` メソッドの引数であるコールバック関数がマイクロタスクキューへと送られます。しかし、そのコールバックはまだ実行されません。イベントループはまだ最初のタスクを実行している途中であり、同期処理をすべて実行します。

ということで次に実行される同期処理は  `returnPromise("2nd Promise", "D")` となります。これも**同期処理**です。先ほどと同じように関数内の `Promise()` コンストラクタの引数である `executor` 関数の中も同期的に実行されます。

また同じように `returnPromise("2nd Promise", "D")` で返ってくる Promise インスタンスはもすでに履行状態なので、直ちに `then()` メソッドの引数であるコールバック関数がマイクロタスクキューへと送られます。もちろんこのコールバックもまだ同期処理が残っているため、まだ実行されません。

最後の同期処理である `console.log("[F] Sync process");` が次に実行されます。

ここまでで出力されるログは次のようになっていることを確認してください。

```sh
❯ deno run returnPromiseByFuncArg2.js
🦖 [A] MAINLINE: Sync process
👻 [B] This line is Synchronously executed
👻 [D] This line is Synchronously executed
🦖 [F] MAINLINE: Sync process

# ...この先はどうなる?
```

同期処理の実行がすべて完了したので、イベントループは次の段階に移行します。同期処理がすべて完了したため、コールスタックに積まれていたグローバルコンテキストがポップします。

これによってコールスタックが空になり、「**マイクロタスクのチェックポイント**」です。別の言い方では「**単一タスクが完了したら、すべてのマイクロタスクを処理する**」です。というわけで、マイクロタスクキューにあるすべてのマイクロタスクを空にするまで処理します。

マイクロタスクキューは「キュー」なので一番古いタスク(先に入れられたタスク)から実行していきます。最初にキューへと追加されたのは `returnPromise("1st Promise", "B").then(cb)` のコールバック関数 `cb` です。従って、出力の続きは次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2.js
🦖 [A] MAINLINE: Sync process
👻 [B] This line is Synchronously executed
👻 [D] This line is Synchronously executed
🦖 [F] MAINLINE: Sync process
👦 [C] This line is Asynchronously executed
👦 Resolved value:  1st Promise

# ...この先はどうなる?
```

そして、次にキューに追加されたのは `returnPromise("2nd Promise", "D").then()` のコールバック関数でした。従ってそのコールバック関数が実行されることで結局出力は次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2.js
🦖 [A] MAINLINE: Sync process
👻 [B] This line is Synchronously executed
👻 [D] This line is Synchronously executed
🦖 [F] MAINLINE: Sync process
👦 [C] This line is Asynchronously executed
👦 Resolved value:  1st Promise
👦 [E] This line is Asynchronously executed
👦 Resolved value:  2nd Promise
```

順番をアルファベットから数字に直してみるとこのようになります。

```js
// doubleThenCallback.js
console.log("🦖 [1] MAINLINE: Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 [${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "2").then((value) => {
  console.log("👦 [5] This line is Asynchronously executed");
  console.log("👦 Resolved value: ", value);
});
returnPromise("2nd Promise", "3").then((value) => {
  console.log("👦 [6] This line is Asynchronously executed");
  console.log("👦 Resolved value: ", value);
});

console.log("🦖 [4] MAINLINE: Sync process");
```

↓ JS Visuzalizer 9000 で実際に可視化してみたので確認してくみてください。

- [doubleThenCallback.js - JS Visuzalizer 9000](https://www.jsv9000.app/?code=Ly8gZG91YmxlVGhlbkNhbGxiYWNrLmpzCmNvbnNvbGUubG9nKCdbMV0gU3luYyBwcm9jZXNzJyk7Cgpjb25zdCByZXR1cm5Qcm9taXNlID0gKHJlc29sdmVkVmFsdWUsIG9yZGVyKSA9PiB7CiAgcmV0dXJuIG5ldyBQcm9taXNlKChyZXNvbHZlKSA9PiB7CiAgICBjb25zb2xlLmxvZyhgWyR7b3JkZXJ9XSBUaGlzIGxpbmUgaXMgU3luY2hyb25vdXNseSBleGVjdXRlZGApOwogICAgcmVzb2x2ZShyZXNvbHZlZFZhbHVlKTsKICB9KTsKfTsKcmV0dXJuUHJvbWlzZSgnMXN0IFByb21pc2UnLCAnMicpLnRoZW4oKHZhbHVlKSA9PiB7CiAgY29uc29sZS5sb2coJ1s1XSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQnKTsKICBjb25zb2xlLmxvZygnUmVzb2x2ZWQgdmFsdWU6ICcsIHZhbHVlKTsKfSk7CnJldHVyblByb21pc2UoJzJuZCBQcm9taXNlJywgJzMnKS50aGVuKCh2YWx1ZSkgPT4gewogIGNvbnNvbGUubG9nKCdbNl0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkJyk7CiAgY29uc29sZS5sb2coJ1Jlc29sdmVkIHZhbHVlOiAnLCB2YWx1ZSk7Cn0pOwoKY29uc29sZS5sb2coJ1s0XSBTeW5jIHByb2Nlc3MnKTsK)
- ⚠️ 注意: JS Visuzlizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク・タスクの実行タイミングについて誤解しないように注意してください

