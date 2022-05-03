---
title: "then メソッドは常に新しい Promise を返す"
---

# then メソッドから返ってくる Promise インスタンス
では `then()` メソッドをそれぞれもう 1 つずつ増やしてみてみます。次のコードについても出力の順番を予測してみてください。

```js:returnPromiseByFuncArg2AddChain.js
// returnPromiseByFuncArg2AddChain.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[B]")
  .then((value) => {
    console.log("[C] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  })
  .then(() => {
    console.log("[D] This line is Asynchronously executed");
  });
returnPromise("2nd Promise", "[E]")
  .then((value) => {
    console.log("[F] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  })
  .then(() => {
    console.log("[G] This line is Asynchronously executed");
  });

console.log("[H] Sync process");
```

:::details 答え
答えは、「A → B → E → H → C → F → D → G」となります。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
[A] Sync process
[B] This line is Synchronously executed
[E] This line is Synchronously executed
[H] Sync process
[C] This line is Asynchronously executed
Resolved value:  1st Promise
[F] This line is Asynchronously executed
Resolved value:  2nd Promise
[D] This line is Asynchronously executed
[G] This line is Asynchronously executed
```
:::

正解できましたか？それでは、なぜこうなるのかを解説してみます。

準備としてコールバック関数などを `cb1` というように省略表記をしてコードを圧縮して書くと次のようになります。

```js
console.log("[A] Sync process");
const returnPromise = (resolvedValue, order) => {...};
returnPromise("1st Promise", "B").then(cb1).then(cb2);
returnPromise("2nd Promise", "E").then(cb3).then(cb4);
console.log("[H] Sync process");
```

前のコードと考え方は同じです。まずは Event Loop の最初のステップである「スクリプトの評価」で「同期処理の実行」が行われます。

- (1) `console.log("[A] Sync process")` が同期処理される
- (2) `returnPromise("1st Promise", "B")` が同期処理されて返される Promise インスタンスが直ちに履行(Fullfilled)状態になるので、`returnPromise("1st Promise", "B").then(cb)` のコードバック関数 `cb` が直ちに Microtask queue へと送られます。

さて、ここまでは前のコードと同じですね。

ここでは「**`then()` メソッドは常に新しい Promise インスタンスを返す**」ということが重要です。

- `returnPromise("1st Promise", "B")` によって返ってくる Promise インスタンスを promise1 とします
- ``returnPromise("1st Promise", "B").then(cb1)` 、つまり `promise1.then(cb1)` によって返ってくる Promise インスタンスを `proimse2` とします

このふたつは全く別の Promise インスタンスとなります。

Promise チェーンにおいて、各 `then()` メソッドにおいて返ってくる Promise インスタンスはそれぞれ別のモノであるということを意識してください。

# Promise インスタンスの状態
急に話は代わりますが、Promise インスタンスというものはそれぞれ「状態」を持ってましたね。

- 待機(pending)状態
- 不変(Setteled)状態: a.k.a 決定状態
  - 履行(Fullfilled)状態
  - 拒否(Rejected)状態

`Promise.resolve()` や `Promise.reject()` などの静的メソッドで状態を決めて初期化しない限り、Promise インスタンスは基本的に待機(pending)状態から始まります。Promise チェーンでは `then()` メソッドで返ってくる Promise インスタンスの状態が待機(pending)状態から履行(Fullfilled)状態へと変わった時点で次の `then()` メソッドで登録したコールバックが Microtask queue へと送られます。

そして、`then(cb)` で返ってくる Promise インスタンスが履行状態へと移行するのは登録されているコールバック `cb` が実行が完了した時点です。

従って、`returnPromise("1st Promise", "B").then(cb1)` で返ってくる Promise インスタンスは Event Loop のこの時点では登録しているコールバック `cb1` が Microtask queue へと送られただけで処理は完了していませんので、まだ待機(pending)状態となります。

`then(cb1)` で返ってくる Promise インスタンスが待機状態なので、`returnPromise("1st Promise", "B").then(cb1).then(cb2)` で登録したコールバック `cb2` はまだ Microtask queue へと送られません。このまま待機させておきます。

そして、そのまま次の処理へと進みます。次の行は `returnPromise("2nd Promise", "E").then(cb1).then(cb2)` なので、まったく同じことが置きます。

```js
console.log("[A] Sync process");
const returnPromise = (resolvedValue, order) => {...};
returnPromise("1st Promise", "B").then(cb1).then(cb2);
returnPromise("2nd Promise", "E").then(cb3).then(cb4);
console.log("[H] Sync process");
```

1. `returnPromise("2nd Promise", "E")` が同期的に実行されて直ちに履行(Fullfilled)状態となった Promise インスタンスが返ってくるので、`then(cb3)` で登録されているコールバック関数 `cb3` が直ちに Microtask queue へと送られます
2. `then(cb3)` で返ってくる別の Promise インスタンスはまだ待機(pending)状態なので `then(cb4)` のコールバック関数 `cb4` はまだキューへ送られずにそのまま待機となります
3. 次の処理に進み、`console.log("[H] Sync process")` が実行されます

これで Event Loop の最初のステップである「スクリプトの評価」において「同期処理の実行」が終わりました。出力はこの時点で次のようになっています。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
[A] Sync process
[B] This line is Synchronously executed
[E] This line is Synchronously executed
[H] Sync process

# ...この先はどうなる?
```

~~Event Loop の最初のステップである「スクリプトの評価」の次は「Macrotask queue にある Macrotask の実行」を行います。しかし、Macrotask を作成するような処理は行っていないので Macrotask queue にタスクは存在していません。従って、Event Loop は再び次のステップへと移行します。~~

:::message alert
Event loop のステップ１はステップ２と同質のものであり、「スクリプトの評価」は実質的に Task(Macrotask) として扱われるので、これが終わると、Event loop は次のステップ３「すべての Microtask の実行」へと移行します。

詳しくは、[Event loop の概要と注意点](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/2-epasync-event-loop) のチャプターを確認してください。
:::

最初の Task(Macrotask) の実行が終わり、Event loop は次のステップ「Microtask queue にあるすべての Microtask の実行」(ステップ４)を行います。

:::message
Macrotask(マクロタスク)については大分後で解説するのでここでは気にしないでください。
:::

先にキューへと送られた `cb1` が実行されます。`then(cb1)` で登録したコールバック `cb1` の実行が完了したので `then(cb1)` で返ってくる Promise インスタンスが履行(Fullfilled)状態へと移行します。Promise インスタンスの状態が履行状態へと移行したことで、さらに `then(cb1).then(cb2)` で登録していたコールバック関数  `cb2` が直ちに Microtask queue へと送られます。

続いて次に Microtask queue 内にあるマイクロタスクが実行されます。`cb1` の後には `cb3` が順番としてキューに送られていたので `cb3` が直ちに実行されます。`cb1` のときと同じように `then(cb3)` で返ってくる Promsie インスタンスの状態が待機(pending)状態から履行(Fullfilled)状態へと移行します。`then(cb3)` で返ってくる Promise インスタンスの状態が履行(Fullfilled)状態へと変わったことで、後続の `then(cb4)` で登録していたコールバック関数 `cb4` が直ちに Microtask queue へと送られます。

この時点での出力はこのようになっています。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
[A] Sync process
[B] This line is Synchronously executed
[E] This line is Synchronously executed
[H] Sync process
[C] This line is Asynchronously executed
Resolved value:  1st Promise
[F] This line is Asynchronously executed
Resolved value:  2nd Promise

# ...この先はどうなる?
```

この時点のステップは「Microtask queue にあるすべての Microtask の実行」であり、Microtask queue にマイクロタスクが存在し続ける限りそれらは実行されます。いまだに `cb2` と `cb4` が順番に Microtask queue に存在しているのでそれらも順番に実行されていきます。

従って、最終的な出力は次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
[A] Sync process
[B] This line is Synchronously executed
[E] This line is Synchronously executed
[H] Sync process
[C] This line is Asynchronously executed
Resolved value:  1st Promise
[F] This line is Asynchronously executed
Resolved value:  2nd Promise
[D] This line is Asynchronously executed
[G] This line is Asynchronously executed
```

言葉で説明すると非常に長くなってしまいましたがこのような結果となります。
実際に JS Visualizer 9000 で可視化してみたので確認してみてください。

- [returnPromiseByFuncArg2AddChain.js - JS Visualizer 9000](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUJ5RnVuY0FyZzJBZGRDaGFpbi5qcwpjb25zb2xlLmxvZygiW0FdIFN5bmMgcHJvY2VzcyIpOwpjb25zdCByZXR1cm5Qcm9taXNlID0gKHJlc29sdmVkVmFsdWUsIG9yZGVyKSA9PiB7CiAgcmV0dXJuIG5ldyBQcm9taXNlKChyZXNvbHZlKSA9PiB7CiAgICBjb25zb2xlLmxvZyhgWyR7b3JkZXJ9XSBUaGlzIGxpbmUgaXMgU3luY2hyb25vdXNseSBleGVjdXRlZGApOwogICAgcmVzb2x2ZShyZXNvbHZlZFZhbHVlKTsKICB9KTsKfTsKCnJldHVyblByb21pc2UoIjFzdCBQcm9taXNlIiwgIkIiKQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coIltDXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWUpOwogIH0pCiAgLnRoZW4oKCkgPT4gewogICAgY29uc29sZS5sb2coIltEXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICB9KTsKcmV0dXJuUHJvbWlzZSgiMm5kIFByb21pc2UiLCAiRSIpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0ZdIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCIpOwogICAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7CiAgfSkKICAudGhlbigoKSA9PiB7CiAgY29uc29sZS5sb2coIltHXSBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICB9KTsKCmNvbnNvbGUubG9nKCJbSF0gU3luYyBwcm9jZXNzIik7CgovLyBFbmQ%3D)

# Promise の状態を確かめる
実際に Promise インスタンスを `console.log()` でそのまま出力してみて状態がどのようになっているかを確認してみましょう。

次のコードでは、`console.log()` の引数として直接 Promise インスタンスを渡しています。どのような出力が得られるでしょうか?

```js
// consolePromise.js
console.log("[Fullfilled status]", new Promise(resolve => resolve("Resolved")));

console.log("[Fullfilled status]", Promise.resolve("Resolved"));

console.log("[Pending status]", Promise.resolve("Resolved but").then(value => console.log(value)));

console.log("[Rejcted status]", Promise.reject("Rejected"))
```

１つずつどうなるかを考えてみます。

`new Promise(executor)` では、`executor` 関数自体は「同期的」に実行されるという話でしが。この場合は内部で直ちに `resolve()` 関数が呼ばれるので、作成した Promise インスタンスは履行(Fullfilled)状態となります。従って、コンソールに出力される Promise インスタンスは履行状態のものとなります。というわけで次の出力をまずは得ます。

```sh
❯ deno run consolePromise.js
[Fullfilled status] Promise { "Resolved" }
# ...
```

では、次の `Promise.resolve()` ですが、これは以下のように `new Promsie()` で作成するのと等価なものでした。

```js
const promise = Promise.resolve("Promise履行時の値");
// この２つは等価
const promise = new Promise(res => {
  res("Promise履行時の値");
});
```

従って、２番目の出力は１番目の出力と同じになります。

```sh
❯ deno run consolePromise.js
[Fullfilled status] Promise { "Resolved" }
[Fullfilled status] Promise { "Resolved" }
# ...
```

３番目が肝心です。`then()` メソッドで返ってくる Promise インスタンスは `Promise.resolve()` で返ってくる Promise インスタンスとは別物であり、Promise チェーンにおいて前の Promise インスタンスが待機状態から履行状態に移行して初めてコールバック関数を Microtask queue へと送ります。そして、Microtask queue へと送られたコールバック関数が Call stack へと送られて実行が完了して初めてそのコールバックをキューに送った `then()` メソッドから返ってくる Promise インスタンスが履行状態へと移行します。

`console.log("[Pending status]", Promise.resolve("Resolved but").then(callback))` は同期的に実行されますが、この時点において、`Promise.resolve()` 自体から返ってくる Promise インスタンスが履行状態であったとしても `Promise.resolve().then(callback)` から最終的に返ってくる Promise インスタンスは待機状態であり、出力される Promise インスタンスは待機状態のものとなります。

従って、３番目の出力は次のようになります。

```sh
❯ deno run consolePromise.js
[Fullfilled status] Promise { "Resolved" }
[Fullfilled status] Promise { "Resolved" }
[Pending status] Promise { <pending> }
# ...
```

待機(Pending)状態の Promise インスタンスを出力すると、このように `Promise { <pending> }` が表示されます。履行(Fullfilled)状態の Promise インスタンスは `Promise { 解決された値 }` というように出力されていますね。

`Promise.resolve().then(callback)` の `callback` ですが、現時点で Event loop のステップは「スクリプトの評価」で、同期処理をすべて完了していません。コールバックの中身は `value => console.log(value)` というものなので、コンソールへ出力がなされますが、Microtask queue へと送られるこのコールバックは Event loop の「スクリプトの評価」のステップが完了した後に実行されます。


４番目では、`console.log("[Rejcted status]", Promise.reject("Rejected"))` が実行されます。`Promise.resject()` については、`Promise.resolve()` の時と同じです。「Promise コンストラクタと Executor 関数」のチャプターで説明したように以下の２つは等価でした。

```js
const promise = Promise.reject("Promise拒否時の理由");
// ２つは等価
const promise = new Promise((_, rej) => {
  rej("Promise拒否時の理由");
});
```

`Promise.resolve()` と同じように作成する Promise インスタンスは直ちに拒否(Rejected)状態になります。従って、４番目の出力は以下のようになります。

```sh
❯ deno run consolePromise.js
[Fullfilled status] Promise { "Resolved" }
[Fullfilled status] Promise { "Resolved" }
[Pending status] Promise { <pending> }
[Rejcted status] Promise { <rejected> "Rejected" }
# ...
```

拒否(Rejected)状態の Promise インスタンスを出力すると `Promise { <rejected> 拒否された理由 }` が表示されます。今回の場合は、理由(reason)として "Rejected" という文字列を `Promise.reject()` の引数として渡しているのでこのような出力が得られました。

これで、Event loop の最初のステップ「スクリプトの評価」が終わったので、~~次の「Macrotask の実行」のステップに移りますが、Macrotask は無いので、次のステップ「Microtask の実行」ステップに移行します。~~

:::message alert
Event loop のステップ１はステップ２と同質のものであり、「スクリプトの評価」は実質的に Task(Macrotask) として扱われるので、これが終わると、Event loop は次のステップ３「すべての Microtask の実行」へと移行します。

詳しくは、[Event loop の概要と注意点](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/2-epasync-event-loop) のチャプターを確認してください。
:::

最初の Task(Macrotask) の実行が終わり、Event loop は次のステップ「Microtask queue にあるすべての Microtask の実行」(ステップ４)を行います。

ここまで来て初めて３番目の出力の際に `then(cb)` でキューに送ったコールバック関数が Call stack へと送られて実行されます。従って、５番目の出力は次のようになります。

```sh
❯ deno run consolePromise.js
[Fullfilled status] Promise { "Resolved" }
[Fullfilled status] Promise { "Resolved" }
[Pending status] Promise { <pending> }
[Rejcted status] Promise { <rejected> "Rejected" }
Resolved but
# ...
```

この "Resolved but" という文字列は `Promise.resolve("Resolved but").then(value => console.log(value))` で履行状態の Promise の解決値が Promise チェーンで `value` として繋がれているので、このタイミングでその値が出力されています。

最後に `Promise.reject()` で拒否状態にした Pormise インスタンスについてエラー補足などを行っていなかったので、未補足であるとして Deno の場合は最後に次のような出力が行われます。

```sh
❯ deno run consolePromise.js
[Fullfilled status] Promise { "Resolved" }
[Fullfilled status] Promise { "Resolved" }
[Pending status] Promise { <pending> }
[Rejcted status] Promise { <rejected> "Rejected" }
Resolved but
error: Uncaught (in promise) Rejected
```

ちなみに Node で実行した場合は次のような出力になります。

```sh
❯ node consolePromise.js
[Fullfilled status] Promise { 'Resolved' }
[Fullfilled status] Promise { 'Resolved' }
[Pending status] Promise { <pending> }
[Rejcted status] Promise { <rejected> 'Rejected' }
Resolved but
node:internal/process/promises:279
            triggerUncaughtException(err, true /* fromPromise */);
            ^

[UnhandledPromiseRejection: This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). The promise rejected with the reason "Rejected".] {
  code: 'ERR_UNHANDLED_REJECTION'
}

Node.js v17.8.0
```

