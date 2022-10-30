---
title: "古い非同期 API を Promise でラップする"
aliases: [
  ch_古い非同期 API を Promise でラップする,
  Promisify,
  Promisification
]
---

# このチャプターについて

ここまで Promise chain でのマイクロタスク発行による非同期処理を見てきました。「単一タスクを実行したら、すべてのマイクロタスクを処理する」というのがイベントループの重大ルールです。

このチャプターでは、タスクによる非同期処理と、マイクロタスクをかませることで逐次処理やエラーハンドリングをやりやすくする "Promisification" という方法について解説しておきます。

:::message alert
『[タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)』のチャプターでタスクとマイクロタスクの関係については詳しく解説したので、タスクの詳細についてはそちらのチャプターを参照してください。
:::

# タスク発行

この本の冒頭で述べたように `setTimoue()` 関数は Web API の一部です。これをなるべく使わずに最初説明したのは、この関数に登録するコールバック関数がタスクキューへと追加されてしまうからです。

`setTimout(cb, delay)` の形で `cb` にはコールバック関数、`delay` には遅延時間をミリ秒が単位で指定することで、その時間が経ってからコールバック関数を実行できます。つまり、このコールバック関数は非同期的に実行されます。

```js
// timeout.js
console.log('🦖 [1] MAINILNE: Sync');
setTimeout(() => {
  console.log('⏰ [5] TIMERS: 3000ms でタイムアウト');
}, 3000); // 3000ミリ秒後に実行したい(3000ミリ秒後にタスクキューへ発行)
setTimeout(() => {
  console.log('⏰ [4] TIMERS: 2000ms でタイムアウト');
}, 2000); // 2000ミリ秒後に実行したい(2000ミリ秒後にタスクキューへ発行)
setTimeout(() => {
  console.log('⏰ [3] TIMERS: 1000ms でタイムアウト');
}, 1000); // 1000ミリ秒後に実行したい(1000ミリ秒後にタスクキューへ発行)
console.log('🦖 [2] MAINILNE: Sync');
```

実行すると次の出力を得ます。

```sh
❯ deno run timeout.js
🦖 [1] MAINILNE: Sync
🦖 [2] MAINILNE: Sync
⏰ [3] TIMERS: 1000ms でタイムアウト
⏰ [4] TIMERS: 2000ms でタイムアウト
⏰ [5] TIMERS: 3000ms でタイムアウト
```

Visualizer で可視化してみたので次のリンクから確認してください。

- [setTimeout.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gdGltZW91dC5qcwpjb25zb2xlLmxvZygiWzFdIFN5bmMgcHJvY2VzcyIpOwpzZXRUaW1lb3V0KCgpID0%2BIHsKICBjb25zb2xlLmxvZygiWzVdIFRoaXMgbGluZSB3aWxsIGJlIHByaW50ZWQgYWZ0ZXIgMzAwMG1zIik7Cn0sIDMwMDApOwpzZXRUaW1lb3V0KCgpID0%2BIHsKICBjb25zb2xlLmxvZygiWzRdIFRoaXMgbGluZSB3aWxsIGJlIHByaW50ZWQgYWZ0ZXIgMjAwMG1zIik7Cn0sIDIwMDApOwpzZXRUaW1lb3V0KCgpID0%2BIHsKICBjb25zb2xlLmxvZygiWzNdIFRoaXMgbGluZSB3aWxsIGJlIHByaW50ZWQgYWZ0ZXIgMTAwMG1zIik7Cn0sIDEwMDApOwpjb25zb2xlLmxvZygiWzJdIFN5bmMgcHJvY2VzcyIpOwo%3D)
- ⚠️ 注意: JS Visuzlizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク・タスク実行のタイミングについて誤解しないように注意してください
- ⚠️ 注意: タスクキューへのタスクを入れるタイミングに実装ミスと思われる部分があるので注意してください(タイマーの指定時間が経過した順番にタスクキューへ入れられるはずのところが、タイマーの起動順番にタスクキューへと入れられてしまっています)

タスクキューへと追加されるのが分かると思います。

# タスクベースの非同期 API について

MDN のドキュメントには以下のことが書かれています。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90

>理想的には、すべての非同期関数はプロミスを返すはずですが、**残念ながら API の中にはいまだに古いやり方で成功/失敗用のコールバックを渡しているものがあります**。顕著な例としては `setTimeout()` 関数があります。**古い様式であるコールバックとプロミスの混在は問題を引き起こします**。というのは、`saySomething()` が失敗したりプログラミングエラーを含んでいた場合に、そのエラーをとらえられないからです。setTimeout にその責任があります。**幸いにも setTimeout をプロミスの中にラップすることができます**。良いやり方は、問題のある関数をできる限り低い水準でラップした上で、直接呼び出さないようにすることです。
>([上記ページ](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)より引用、太字は筆者強調)

非同期 API はコールバックのスタイルでタスクを発行するよりも、Promise を返してマイクロタスクを発行するものであることが望ましいという旨が読み取れます。そして、タスクを発行するタイプのコールバックスタイル API は Promise でラップすることでエラーハンドリングなどをよりやりやすくなります。

# Promisification

上で述べたように Promise インスタンスによってタスクベースの非同期処理(つまり古いタイプの非同期 API)をラップする手法は "Promisifying" または "Promisification" と呼ばれます。

https://ja.javascript.info/promisify

ただし、最近は手動でやることはあまりやらないらしいです。というのも、単純に非同期 API 自体 Promise-based であるものが出てきています。また、古いタイプの API をラップするのでも、例えば Node 環境では `promisifying` 関数というのが util module にあり、これを使うことで手動ラップすることなく Promisification ができるようになっています。

Deno 環境などではほとんどすべての非同期 API が Promise インスタンスを返しますし、`setTimeout()` でさえも Promise で手動ラップする必要も実はなく、Promise を返すタイマーが Standard library の１つとして提供されています。

https://deno.land/std@0.145.0/async#delay

Node 環境でも Promise-based な非同期 API が色々提供されており、Promise を返すタイマー処理も存在しています。

https://nodejs.org/dist/v18.2.0/docs/api/timers.html#timers-promises-api

とは言っても内部で、Promise でラップしていることもありますし、手動でラップする方法を学んでおいて損はないので解説します。単純に `new Promise()` によってラップするだけです。

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
  .then(() => console.log("Next action"));
```

これで、特定時間が経過したら何かする、それが完了したらまた何かするというのがやりやすくなります。

Visualizer で可視化したので、確認してみてください。

- [promiseTimer.js - JS Visualizer](https://www.jsv9000.app/?code=Y29uc3QgcHJvbWlzZVRpbWVyID0gKGRlbGF5KSA9PiB7CiAgcmV0dXJuIG5ldyBQcm9taXNlKChyZXNvbHZlKSA9PiB7CiAgICBzZXRUaW1lb3V0KCgpID0%2BIHsKICAgICAgcmVzb2x2ZSgpOwogICAgfSwgZGVsYXkpOwp9KX07Cgpwcm9taXNlVGltZXIoMTAwMCkKICAudGhlbigoKSA9PiBjb25zb2xlLmxvZygiVGltZW91dCIpKQogIC50aGVuKCgpID0%2BIGNvbnNvbGUubG9nKCJOZXh0IGFjdGlvbiIpKQogIC50aGVuKCgpID0%2BIGNvbnNvbGUubG9nKCJOZXh0IGFjdGlvbiIpKQogIC5jYXRjaCgoZXJyKSA9PiBjb25zb2xlLmVycm9yKGVyci5tZXNzYWdlKSk7)
- ⚠️ 注意: JS Visuzlizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク・タスク実行のタイミングについて誤解しないように注意してください

はじめにタスクが発行されてタスクキューへと送られていますね。タスクキューからコールスタックへと送られて、`resolve()` が呼び出されます。それによって待機状態の Promise は履行状態となるため、`.then()` メソッドのコールバックがマイクロタスクキューにマイクロタスクとして送られます。そしてマイクロタスクがコールスタックに積まれ実行されることで更にマイクロタスクが発生して、すべてのマイクロタスクが処理されてコールスタックが空になるとイベントループは終了します。

:::message
上の書き方は少し冗長なので、アロー関数での `return` を省略をして書くと次の様になります。

```js
const promiseTimer = (delay)
  => new Promise((resolve) => setTimeout(resolve, delay));
```
:::

Promise でラップして使わない場合にはコールバックをいくつもネストする必要がでてくるので、いわゆる **Callback Hell** になります。その形から "Pyramid of doom" とも呼ばれます。

```js
setTimeout((value) => {
  console.log(value);
  setTimeout((value) => {
    console.log(value);
    setTimeout((value) => {
      console.log(value);
      setTimeout((value) => {
        console.log(value);
        setTimeout((value) => {
          console.log(value, "ピラミッドのてっぺん");
        }, 1000, value);
      }, 1000, value);
    }, 1000, value);
  }, 1000, value);
}, 1000, "タイムアウト");
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

例えば、これで何が嬉しいかというと次の関連する処理まで一時的に時間を置きたいという要望を叶えることができます。上で定義した `promiseTimer()` を `sleep()` という名前で再び定義して考えてみます。複数回のリクエストをおこなような操作を対象となるサーバーに負荷をかけないように時間間隔を置くことである程度分割して行うようにしたい場合、この `sleep()` によって async 関数内の次の処理を指定時間以上あけるようにできます。

```js
function sleep(time) {
  // resolve の名前は何でもよいので短い r にしておく
  return new Promise(r => setTimeout(r, time)); 
}
(async () => {
  await multipleFetch(); // 複数の fetch を行う async 関数だとする
  await sleep(3000).then(() => console.log("3秒以上経過したから再度リクエスト"));
  await multipleFetch();
  await sleep(3000).then(() => console.log("3秒以上経過したから再度リクエスト"));
  await multipleFetch();
  console.log("async 関数内のすべての処理が終了しました");
})();
```

async 関数の外側で何らかの別の処理が走っている可能性もありイベントループのマイクロタスクキューで別の待ちタスクなどがあれば、３秒よりももっと多くの時間遅延します。

実際に Deno では標準モジュール(std)の `async` モジュールで提供される `delay()` 関数はこのように作られています。リポジトリでは次の場所に存在しています。上で定義したものより緻密に作られていますが内部的には `setTimeout()` をちゃんと使っていることが分かります。

https://github.com/denoland/deno_std/blob/0.145.0/async/delay.ts

https://doc.deno.land/https://deno.land/std@0.145.0/async/mod.ts/~/delay

