---
title: "古い非同期 API を Promise でラップする"
---

# タスク発行

:::message
『タスクキューとマイクロタスクキュー』のチャプターでタスクとマイクロタスクの関係については詳しく解説したので、タスクの詳細についてはそちらのチャプターを参照してください。
:::

ここまで Promise チェーンでのマイクロタスク発行による非同期処理を見てきましたが、イベントループではステップ２に「すべてのマイクロタスクの実行」がありました。マイクロタスクの基礎ができたのでここからようやくタスクの解説になります。

最初に述べたように `setTimoue()` 関数は Web API の一部です。これを使わずに最初説明したのは、この関数に登録するコールバック関数がタスクキューへと追加されてしまうからです。

`setTimout(cb, delay)` の形で `cb` にはコールバック関数、`delay` には遅延時間をミリ秒が単位で指定することで、その時間が経ってからコールバック関数を実行できます。つまり、このコールバック関数は非同期的に実行されます。

```js
// timeout.js
console.log("[1] Sync process");

setTimeout(() => {
  console.log("[5] This line will be printed after 3000ms");
}, 3000); // 3000ミリ秒後に実行
setTimeout(() => {
  console.log("[4] This line will be printed after 2000ms");
}, 2000); // 2000ミリ秒後に実行
setTimeout(() => {
  console.log("[3] This line will be printed after 1000ms");
}, 1000); // 1000ミリ秒後に実行

console.log("[2] Sync process");
```

実行すると次の出力を得ます。

```sh
❯ deno run timeout.js
[1] Sync process
[2] Sync process
[3] This line will be printed after 1000ms
[4] This line will be printed after 2000ms
[5] This line will be printed after 3000ms
```

またもや可視化してみたので次のリンクから確認してください。

- [setTimeout.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gdGltZW91dC5qcwpjb25zb2xlLmxvZygiWzFdIFN5bmMgcHJvY2VzcyIpOwpzZXRUaW1lb3V0KCgpID0%2BIHsKICBjb25zb2xlLmxvZygiWzVdIFRoaXMgbGluZSB3aWxsIGJlIHByaW50ZWQgYWZ0ZXIgMzAwMG1zIik7Cn0sIDMwMDApOwpzZXRUaW1lb3V0KCgpID0%2BIHsKICBjb25zb2xlLmxvZygiWzRdIFRoaXMgbGluZSB3aWxsIGJlIHByaW50ZWQgYWZ0ZXIgMjAwMG1zIik7Cn0sIDIwMDApOwpzZXRUaW1lb3V0KCgpID0%2BIHsKICBjb25zb2xlLmxvZygiWzNdIFRoaXMgbGluZSB3aWxsIGJlIHByaW50ZWQgYWZ0ZXIgMTAwMG1zIik7Cn0sIDEwMDApOwpjb25zb2xlLmxvZygiWzJdIFN5bmMgcHJvY2VzcyIpOwo%3D)

タスクキューへと追加されるのが分かると思います。

# タスクベースの非同期 API について

MDN のドキュメントには以下のことが書かれています。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90

>理想的には、すべての非同期関数はプロミスを返すはずですが、**残念ながら API の中にはいまだに古いやり方で成功/失敗用のコールバックを渡しているものがあります**。顕著な例としては `setTimeout()` 関数があります。**古い様式であるコールバックとプロミスの混在は問題を引き起こします**。というのは、`saySomething()` が失敗したりプログラミングエラーを含んでいた場合に、そのエラーをとらえられないからです。setTimeout にその責任があります。**幸いにも setTimeout をプロミスの中にラップすることができます**。良いやり方は、問題のある関数をできる限り低い水準でラップした上で、直接呼び出さないようにすることです。
>(上記ページより引用)

非同期 API はコールバックのスタイルでタスクを発行するよりも、Promise を返してマイクロタスクを発行するものであることが望ましいという旨が読み取れます。そして、タスクを発行するタイプのコールバックスタイル API は Promise でラップすることでエラーハンドリングなどをよりやりやすくなります。

# Promisification

上で述べたように Promise インスタンスによってタスクベースの非同期処理(つまり古いタイプの非同期 API)をラップする手法は "Promisifying" または "Promisification" と呼ばれます。

https://ja.javascript.info/promisify

ただし、手動でやることは最近はあまりやらないらしいです。というのも単純に非同期 API 自体が Promise-based なものが出てきています。また、古いタイプの API をラップするのでも、例えば Node 環境では `promisifying` 関数というのが util module にあり、これを使うことで手動でラップすることなく Promisify できるようになっています。

Deno 環境などではほとんどすべての非同期 API が Promise インスタンスを返しますし、`setTimeout()` でさえも Promise で手動ラップする必要も実はなく、Promise を返すタイマーが Standard library の１つとして提供されています。

https://deno.land/std@0.137.0/async#delay

Node 環境でも Promise-based な非同期 API が色々提供されており、Promise を返すタイマー処理も存在しています。

https://nodejs.org/dist/latest-v18.x/docs/api/timers.html#timers-promises-api

とは言っても内部で、Promise でラップしていることもありますし、手動でラップする方法を学んでおいて損はないので解説します。単純に `new Promise()` でラップするだけです。

なるべく低水準でラップして直接的に呼び出さないようにします。

```js
const promiseTimer = (delay) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve();
    }, delay);
})};

promiseTimer(1000)
  .then(() => console.log("Timeout"))
  .then(() => console.log("Next action"))
  .then(() => console.log("Next action"))
  .catch((err) => console.error(err.message));
```

これで、特定時間が経過したら何かする、それが完了したらまた何かするというのがやりやすくなります。

Visualizer で可視化してみました。確認してみてください。

- [promiseTimer.js - JS Visualizer](https://www.jsv9000.app/?code=Y29uc3QgcHJvbWlzZVRpbWVyID0gKGRlbGF5KSA9PiB7CiAgcmV0dXJuIG5ldyBQcm9taXNlKChyZXNvbHZlKSA9PiB7CiAgICBzZXRUaW1lb3V0KCgpID0%2BIHsKICAgICAgcmVzb2x2ZSgpOwogICAgfSwgZGVsYXkpOwp9KX07Cgpwcm9taXNlVGltZXIoMTAwMCkKICAudGhlbigoKSA9PiBjb25zb2xlLmxvZygiVGltZW91dCIpKQogIC50aGVuKCgpID0%2BIGNvbnNvbGUubG9nKCJOZXh0IGFjdGlvbiIpKQogIC50aGVuKCgpID0%2BIGNvbnNvbGUubG9nKCJOZXh0IGFjdGlvbiIpKQogIC5jYXRjaCgoZXJyKSA9PiBjb25zb2xlLmVycm9yKGVyci5tZXNzYWdlKSk7)

はじめにタスクが発行されてタスクキューへと送られていますね。タスクキューからコールスタックへと送られて、`resolve()` が呼び出されます。それによって待機状態の Promoise が履行状態になるため、`.then()` メソッドのコールバックがマイクロタスクキューにマイクロタスクとして送られます。そしてマイクロタスクがコールスタックに積まれ実行されることで更にマイクロタスクが発生して、すべてのマイクロタスクが処理されてコールスタックが空になるとイベントループは終了します。

:::message
上の書き方は少し冗長なので、アロー関数での `return` を省略をして書くと次の様になります。

```js
const promiseTimer = (delay)
  => new Promise((resolve) => setTimeout(resolve, delay));
```
:::

Promise でラップして使わない場合にはコールバックをいくつもネストする必要がでてくるので、いわゆる **Callback Hell** になります。

```js
const msg = "Timeout!";

setTimeout((value) => {
  console.log(value);
  setTimeout((value) => {
    console.log(value);
    setTimeout((value) => {
      console.log(value);
      setTimeout((value) => {
        console.log(value);
      }, 1000, value);
    }, 1000, value);
  }, 1000, value);
}, 1000, msg);
```

また、Promise でラップすることによって、Async function にて `await` 式を使って完了を待てるようになります。

```js
// simplePromisify.js
const promiseTimer = (delay) => new Promise((resolve) => setTimeout(resolve, delay));

(async function doAsync() {
  await promiseTimer(1000);
  console.log("timeout!");
  await promiseTimer(1000);
  console.log("timeout!");
})();
// 即時実行関数
```

