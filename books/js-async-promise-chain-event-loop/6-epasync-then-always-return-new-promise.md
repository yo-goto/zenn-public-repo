---
title: "then メソッドは常に新しい Promise を返す"
cssclass: zenn
date: 2022-04-17
modified: 2023-03-31
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『then メソッドは常に新しい Promise を返す』
---

## このチャプターについて

このチャプターでは、Promise chain での注意点である `then()` メソッドの特長について解説しておきます。

## then メソッドから返ってくる Promise インスタンス

前のチャプターから続いて、`then()` メソッドをそれぞれもう 1 つずつ増やしてみます。

次のコードについても出力の順番を予測してみてください。

```js:returnPromiseByFuncArg2AddChain.js
// returnPromiseByFuncArg2AddChain.js
console.log("🦖 [A] Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 [${order}] Sync`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "B")
  .then((value) => {
    console.log("👦 [C] Async");
    console.log("👦 Resolved value: ", value);
  })
  .then(() => {
    console.log("👦 [D] Async");
  });
returnPromise("2nd Promise", "E")
  .then((value) => {
    console.log("👦 [F] Async");
    console.log("👦 Resolved value: ", value);
  })
  .then(() => {
  console.log("👦 [G] Async");
  });

console.log("🦖 [H] Sync");
```

:::details 答え
答えは、「A → B → E → H → C → F → D → G」となります。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
🦖 [A] Sync
👻 [B] Sync
👻 [E] Sync
🦖 [H] Sync
👦 [C] Async
👦 Resolved value: 1st Promise
👦 [F] Async
👦 Resolved value: 2nd Promise
👦 [D] Async
👦 [G] Async
```
:::

正解できましたか？それでは、なぜこうなるのかを解説してみます。

準備としてコールバック関数などを `cb1` というように省略表記をしてコードを圧縮して書くと次のようになります。

```js
console.log("🦖 [A] Sync");
const returnPromise = (resolvedValue, order) => {...};
returnPromise("1st Promise", "B").then(cb1).then(cb2);
returnPromise("2nd Promise", "E").then(cb3).then(cb4);
console.log("🦖 [H] Sync");
```

前のコードと考え方は同じです。まずはイベントループにおいて最初のタスクである「スクリプトの評価」で「すべての同期処理の実行」が行われます。コールスタックの一番下にグローバルコンテキストが積まれた状態で同期処理がどんどん行われていきます。

- (1) `console.log("🦖 [A] Sync")` が同期処理される
- (2) `returnPromise("1st Promise", "B")` が同期処理されて返される Promise インスタンスが直ちに履行 (Fulfilled) 状態になるので、`returnPromise("1st Promise", "B").then(cb)` のコードバック関数 `cb` が直ちにマイクロタスクキューへと送られます。

さて、ここまでは前のコードと同じですね。

ここで重要なのは「**`then()` メソッドは常に新しい Promise インスタンスを返す**」ということです。

- `returnPromise("1st Promise", "B")` によって返ってくる Promise インスタンスを promise1 とします
- `returnPromise("1st Promise", "B").then(cb1)` 、つまり `promise1.then(cb1)` によって返ってくる Promise インスタンスを `promise2` とします

この２つは **全く別の Promise インスタンス** となります。

Promise chain において、各 `then()` メソッドにおいて返ってくる Promise インスタンスは **それぞれ別のモノ** であるということを意識してください。

## Promise インスタンスの状態

ここで話は代わりますが、Promise インスタンスというものはそれぞれ「状態 (State)」を持ってましたね。

- Pending(待機状態)
- Fulfilled(履行状態)
- Rejected(拒否状態)

:::message
Promise の状態 (State) と運命 (Fate) などの基本概念については、『[Promise の基本概念](a-epasync-promise-basic-concept)』のチャプターを参照してください。
:::

`Promise.resolve()` や `Promise.reject()` などの静的メソッドで状態を決めて初期化しない限り、Promise インスタンスは基本的に待機 (pending) 状態から始まります。Promise chain では `then()` メソッドで返ってくる Promise インスタンスの状態が待機状態から履行状態へと変わった時点で次の `then()` メソッドで登録したコールバックがマイクロタスクキューへと送られます。

そして、`then(cb)` で返ってくる Promise インスタンスが履行状態へと移行するのは登録されているコールバック `cb` の実行が完了した時点です。

従って、`returnPromise("1st Promise", "B").then(cb1)` で返ってくる Promise インスタンスはイベントループのこの時点で登録しているコールバック `cb1` がマイクロタスクキューへと送られただけで処理は完了していませんので、まだ待機状態となります。

`then(cb1)` で返ってくる Promise インスタンスが待機状態なので、`returnPromise("1st Promise", "B").then(cb1).then(cb2)` で登録したコールバック `cb2` はまだマイクロタスクキューへと送られません。このまま待機させておきます。

そして、そのまま次の処理へと進みます。次の行は `returnPromise("2nd Promise", "E").then(cb3).then(cb4)` なので、まったく同じことが置きます。

```js
console.log("🦖 [A] Sync");
const returnPromise = (resolvedValue, order) => {...};
returnPromise("1st Promise", "B").then(cb1).then(cb2);
returnPromise("2nd Promise", "E").then(cb3).then(cb4);
console.log("🦖 [H] Sync");
```

1. `returnPromise("2nd Promise", "E")` が同期的に実行されて直ちに履行 (Fulfilled) 状態となった Promise インスタンスが返ってくるので、`then(cb3)` で登録されているコールバック関数 `cb3` が直ちにマイクロタスクキューへと送られます
2. `then(cb3)` で返ってくる別の Promise インスタンスはまだ待機状態なので `then(cb4)` のコールバック関数 `cb4` はまだキューへ送られずにそのまま待機となります
3. 次の処理に進み、`console.log("[H] Sync")` が実行されます

これでイベントループにおいてコード実行の最初のタスクである「スクリプトの評価」における「すべて同期処理の実行」が終わりました。出力はこの時点で次のようになっています。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
🦖 [A] Sync
👻 [B] Sync
👻 [E] Sync
🦖 [H] Sync

# ...この先はどうなる?
```

:::message
**以前の解説**: 最初のタスクの実行が終わり、イベントループは次のステップに移行します。「マイクロタスクキューにあるすべてのマイクロタスクの実行」です。
:::

いつもどおり、すべての同期処理が終わったため、グローバルコンテキストがポップして、コールスタックが空になったので「マイクロタスクのチェックポイント」となります。別の言い方では「**単一タスクが完了したら、すべてのマイクロタスクを処理する**」です。というわけで、マイクロタスクキューにあるすべてのマイクロタスクを空にするまで処理します。

先にキューへと送られた `cb1` が実行されます。`then(cb1)` で登録したコールバック `cb1` の実行が完了したので `then(cb1)` で返ってくる Promise インスタンスが履行状態へと移行します。Promise インスタンスの状態が履行状態へと移行したことで、さらに `then(cb1).then(cb2)` に登録していたコールバック関数 `cb2` が直ちにマイクロタスクキューへと送られます。

続いて次にマイクロタスクキュー内にあるマイクロタスクが実行されます。`cb1` の後には `cb3` が順番としてキューに送られていたので `cb3` が直ちに実行されます。`cb1` のときと同じように `then(cb3)` で返ってくる Promise インスタンスの状態が待機状態から履行状態へと移行します。`then(cb3)` で返ってくる Promise インスタンスの状態が履行状態へと変わったことで、後続の `then(cb4)` で登録していたコールバック関数 `cb4` が直ちにマイクロタスクキューへと送られます。

この時点での出力はこのようになっています。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
🦖 [A] Sync
👻 [B] Sync
👻 [E] Sync
🦖 [H] Sync
👦 [C] Async
👦 Resolved value: 1st Promise
👦 [F] Async
👦 Resolved value: 2nd Promise

# ...この先はどうなる?
```

この時点のステップは「マイクロタスクキューにあるすべてのマイクロタスクの実行」であり、マイクロタスクキューにマイクロタスクが存在し続ける限りそれらは実行されます。マイクロタスクキュー内にはいまだ `cb2` と `cb4` が順番に存在しているのでそれらも順番に実行されていきます。

従って、最終的な出力は次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
🦖 [A] Sync
👻 [B] Sync
👻 [E] Sync
🦖 [H] Sync
👦 [C] Async
👦 Resolved value: 1st Promise
👦 [F] Async
👦 Resolved value: 2nd Promise
👦 [D] Async
👦 [G] Async
```

言葉で説明すると非常に長くなってしまいましたがこのような結果となります。
実際に JS Visualizer 9000 で可視化してみたので確認してみてください。

- [returnPromiseByFuncArg2AddChain.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUJ5RnVuY0FyZzJBZGRDaGFpbi5qcwpjb25zb2xlLmxvZygiW0FdIFN5bmMgcHJvY2VzcyIpOwpjb25zdCByZXR1cm5Qcm9taXNlID0gKHJlc29sdmVkVmFsdWUsIG9yZGVyKSA9PiB7CiAgcmV0dXJuIG5ldyBQcm9taXNlKChyZXNvbHZlKSA9PiB7CiAgICBjb25zb2xlLmxvZyhgWyR7b3JkZXJ9XSBUaGlzIGxpbmUgaXMgU3luY2hyb25vdXNseSBleGVjdXRlZGApOwogICAgcmVzb2x2ZShyZXNvbHZlZFZhbHVlKTsKICB9KTsKfTsKCnJldHVyblByb21pc2UoIjFzdCBQcm9taXNlIiwgIkIiKQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coIltDXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWUpOwogIH0pCiAgLnRoZW4oKCkgPT4gewogICAgY29uc29sZS5sb2coIltEXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICB9KTsKcmV0dXJuUHJvbWlzZSgiMm5kIFByb21pc2UiLCAiRSIpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0ZdIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCIpOwogICAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7CiAgfSkKICAudGhlbigoKSA9PiB7CiAgY29uc29sZS5sb2coIltHXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICB9KTsKCmNvbnNvbGUubG9nKCJbSF0gU3luYyBwcm9jZXNzIik7CgovLyBFbmQ%3D)
- ⚠️ 注意: JS Visualizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク実行のタイミングについて誤解しないように注意してください

## Promise の状態を確かめる

実際に Promise インスタンスを `console.log()` でそのまま出力してみて状態がどのようになっているかを確認してみます。

次のコードでは、`console.log()` の引数として直接 Promise インスタンスを渡しています。どのような出力が得られるでしょうか？

```js
// consolePromise.js
console.log("[Fulfilled status]", new Promise(resolve => resolve("Resolved")));

console.log("[Fulfilled status]", Promise.resolve("Resolved"));

console.log("[Pending status]", Promise.resolve("Resolved but").then(value => console.log(value)));

console.log("[Rejected status]", Promise.reject("Rejected"))
```

１つずつどうなるかを考えてみます。

`new Promise(executor)` では、`executor` 関数自体は「同期的」に実行されるという話でした。この場合は内部で直ちに `resolve()` 関数が呼ばれるので、作成した Promise インスタンスは履行 (Fulfilled) 状態となります。従って、コンソールに出力される Promise インスタンスは履行状態のものとなります。というわけで次の出力をまずは得ます。

```sh
❯ deno run consolePromise.js
[Fulfilled status] Promise { "Resolved" }
# ...
```

では、次の `Promise.resolve()` ですが、これは以下のように `new Promise()` で作成するのと大体は同じものでした。

```js
const promise1 = Promise.resolve("Promise履行時の値");
// この２つは大体同じ
const promise2 = new Promise(res => {
  res("Promise履行時の値");
});
```

従って、２番目の出力は１番目の出力と同じになります。

```sh
❯ deno run consolePromise.js
[Fulfilled status] Promise { "Resolved" }
[Fulfilled status] Promise { "Resolved" }
# ...
```

３番目が肝心です。`then()` メソッドで返ってくる Promise インスタンスは `Promise.resolve()` で返ってくる Promise インスタンスとは別物であり、Promise chain において前の Promise インスタンスが待機状態から履行状態に移行して初めてコールバック関数をマイクロタスクキューへと送ります。そして、マイクロタスクキューへと送られたコールバック関数が Call stack へと送られて実行が完了して初めてそのコールバックをキューに送った `then()` メソッドから返ってくる Promise インスタンスが履行状態へと移行します。

`console.log("[Pending status]", Promise.resolve("Resolved but").then(callback))` は同期的に実行されますが、この時点において、`Promise.resolve()` 自体から返ってくる Promise インスタンスが履行状態であったとしても `Promise.resolve().then(callback)` から最終的に返ってくる Promise インスタンスは待機状態であり、出力される Promise インスタンスは待機状態のものとなります。

従って、３番目の出力は次のようになります。

```sh
❯ deno run consolePromise.js
[Fulfilled status] Promise { "Resolved" }
[Fulfilled status] Promise { "Resolved" }
[Pending status] Promise { <pending> }
# ...
```

待機 (Pending) 状態の Promise インスタンスを出力すると、このように `Promise { <pending> }` が表示されます。履行 (Fulfilled) 状態の Promise インスタンスは `Promise { 解決された値 }` というように出力されていますね。

`Promise.resolve().then(callback)` の `callback` ですが、現時点ではイベントループのステップは「スクリプトの評価」で、同期処理をすべて完了していません。コールバックの中身は `value => console.log(value)` というものなので、コンソールへ出力がなされますが、マイクロタスクキューへと送られるこのコールバックはイベントループの「スクリプトの評価」のステップが完了した後に実行されます。

つまり、グローバルコンテキストがコールスタックからポップして、コールスタックが空となった時にマイクロタスクのチェックポイントですから、その時点からマイクロタスクが処理されます。

４番目では、`console.log("[Rejected status]", Promise.reject("Rejected"))` が実行されます。`Promise.reject()` については、`Promise.resolve()` の時と同じです。「[Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)」のチャプターで説明したように以下の２つはほとんど同じでした。

```js
const promise = Promise.reject("Promise拒否時の理由");
// ２つは大体同じ
const promise = new Promise((_, rej) => {
  rej("Promise拒否時の理由");
});
```

`Promise.resolve()` と同じように作成する Promise インスタンスは直ちに拒否 (Rejected) 状態になります。従って、４番目の出力は以下のようになります。

```sh
❯ deno run consolePromise.js
[Fulfilled status] Promise { "Resolved" }
[Fulfilled status] Promise { "Resolved" }
[Pending status] Promise { <pending> }
[Rejected status] Promise { <rejected> "Rejected" }
# ...
```

拒否 (Rejected) 状態の Promise インスタンスを出力すると `Promise { <rejected> 拒否された理由 }` が表示されます。今回の場合は、理由 (reason) として "Rejected" という文字列を `Promise.reject()` の引数として渡しているのでこのような出力が得られました。

これで、イベントループの最初のステップ「スクリプトの評価」が終わりました。いつもどおり、すべての同期処理が終わったため、グローバルコンテキストがポップして、コールスタックが空になったので「マイクロタスクのチェックポイント」となります。別の言い方では「**単一タスクが完了したら、すべてのマイクロタスクを処理する**」です。というわけで、マイクロタスクキューにあるすべてのマイクロタスクを空にするまで処理します。

:::message
**以前の解説**: 最初のタスクの実行が終わり、イベントループは次のステップに移行します。「マイクロタスクキューにあるすべてのマイクロタスクの実行」です。
:::

ここまで来て初めて３番目の出力の際に `then(cb)` でキューへ送ったコールバック関数がコールスタックへと送られて実行されます。従って、５番目の出力は次のようになります。

```sh
❯ deno run consolePromise.js
[Fulfilled status] Promise { "Resolved" }
[Fulfilled status] Promise { "Resolved" }
[Pending status] Promise { <pending> }
[Rejected status] Promise { <rejected> "Rejected" }
Resolved but
# ...
```

この "Resolved but" という文字列は `Promise.resolve("Resolved but").then(value => console.log(value))` で履行状態の Promise の解決値が Promise chain で `value` として繋がれているので、このタイミングでその値が出力されています。

最後に `Promise.reject()` を使って拒否状態にした Promise インスタンスについてエラー補足などを行っていなかったので、未補足であるとして Deno の場合は最後に次のような出力が行われます。

```sh
❯ deno run consolePromise.js
[Fulfilled status] Promise { "Resolved" }
[Fulfilled status] Promise { "Resolved" }
[Pending status] Promise { <pending> }
[Rejected status] Promise { <rejected> "Rejected" }
Resolved but
error: Uncaught (in promise) Rejected
```

ちなみに Node で実行した場合は次のような出力になります。

```sh
❯ node consolePromise.js
[Fulfilled status] Promise { 'Resolved' }
[Fulfilled status] Promise { 'Resolved' }
[Pending status] Promise { <pending> }
[Rejected status] Promise { <rejected> 'Rejected' }
Resolved but
node:internal/process/promises:288
            triggerUncaughtException(err, true /* fromPromise */);
            ^

[UnhandledPromiseRejection: This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). The promise rejected with the reason "Rejected".] {
  code: 'ERR_UNHANDLED_REJECTION'
}

Node.js v18.2.0
```
