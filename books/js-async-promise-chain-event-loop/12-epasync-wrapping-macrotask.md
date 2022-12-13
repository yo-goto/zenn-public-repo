---
title: "古い非同期 API を Promise でラップする"
cssclass: zenn
date: 2022-04-17
modified: 2022-12-04
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases:
  - Promise本『古い非同期 API を Promise でラップする』
  - Promisify
  - Promisification
---

# このチャプターについて

ここまで Promise chain でのマイクロタスク発行による非同期処理を見てきました。「単一タスクを実行したら、すべてのマイクロタスクを処理する」というのがイベントループの重大ルールです。

このチャプターでは、タスクによる非同期処理と、マイクロタスクをかませることで逐次処理やエラーハンドリングをやりやすくする "Promisification" という手法について解説しておきます。

:::message alert
『[タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)』のチャプターでタスクとマイクロタスクの関係については詳しく解説したので、タスクの詳細についてはそちらのチャプターを参照してください。
:::

# タスク発行

『[非同期 API と環境](f-epasync-asynchronous-apis)』のチャプターで説明したとおり、`setTimeout()` は非同期の Web API です。

`setTimout(cb, delay)` の形で `cb` にはコールバック関数、`delay` には遅延時間をミリ秒が単位で指定することで、その時間が経ってからコールバック関数を実行できます。つまり、このコールバック関数は非同期的に実行されます。

この処理はコードのスケジューリングをするわけですが、処理の実態は「並列的に環境側で指定した時間を計測して、時間経過後に登録したコールバック関数をタスクキューへ送信する」というものです。登録したコールバック関数は実際に処理されるタイミングは、イベントループの機構においてタスクキューからコールスタックへと配置されてスタック上でトップとなった瞬間です。

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
- ⚠️ 注意: タスクキューへのタスクを入れるタイミングに実装ミスと思われる部分があるので注意してください (タイマーの指定時間が経過した順番にタスクキューへ入れられるはずのところが、タイマーの起動順番にタスクキューへと入れられてしまっています)

タスクキューへと追加されるのが分かると思います。

# タスクベースの非同期 API について

MDN のドキュメントでは、非同期 API の理想はコールバックのスタイルでタスクを発行するよりも、[Promise を返してマイクロタスクを発行するものであることが望ましいという旨](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90) が読み取れます。

そして、タスクを発行するタイプの Callback-based API を Promise でラップすることでエラーハンドリングなどが行いやすくなります。

# Promisification

上で述べたように Promise インスタンスによってタスクベースの非同期処理 (つまり古いタイプの非同期 API) をラップする手法は "Promisifying" または "Promisification" と呼ばれます。

https://ja.javascript.info/promisify

:::message alert
この手法を非同期処理の学習のはじめに知ってしまうと非常に混乱することになるので注意してください。

というのも、Promisification では非同期処理の内訳として登場する複数のものが絡めて利用されているためです。`setTimeout()` と Promise のそれぞれタスクキューとマイクロタスクキューという別々の機構を利用し、この手法ではタスクとマイクロタスクを順番に消費して連鎖的に処理を実現するということをやるので、コード上の見た目に比べて相当複雑なことをしています。具体的な Promisification の処理のフローでは「API を起動して環境に並列的作業を行わせ、それが完了後にイベントループにタスクを送り、そのタスクが処理される際に Promise を解決してマイクロタスクを発火させることで後続の関連処理を順番に行わせる」ということをやります。各要素について順番に理解できていれば難しくないですが、はじめからこれをまとめて理解しようとすると後々混乱してしまう可能性があります。

さらに `Promise()` コンストラクタに渡す Executor 関数内部の処理は『[コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async)』のチャプターで見たとおり実は「**同期処理**」なのですが、`setTimeout()` を使用してしまうことで非同期処理であると勘違いすることになります。
:::

ただし、最近は手動でやることはあまりやらないらしいです。というのも、単純に非同期 API 自体 Promise-based であるものが出てきています。また、古いタイプの API をラップするのでも、例えば Node 環境では `promisifying` 関数というのが util module にあり、これを使うことで手動ラップすることなく Promisification ができるようになっています。

Deno 環境などではほとんどすべての非同期 API が Promise インスタンスを返しますし、`setTimeout()` でさえも Promise で手動ラップする必要も実はなく、Promise を返すタイマーが std の１つとして提供されています。

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

```js:pyramidDoom.js
setTimeout((value) => {
  console.log(value, "[1]");
  setTimeout((value) => {
    console.log(value, "[2]");
    setTimeout((value) => {
      console.log(value, "[3]");
      setTimeout((value) => {
        console.log(value, "[4]");
        setTimeout((value) => {
          console.log(value, "[5] 頂点");
        }, 1000, value);
      }, 1000, value);
    }, 1000, value);
  }, 1000, value);
}, 1000, "タイムアウト");

/* 出力結果
❯ deno run pyramidDoom.js
タイムアウト [1]
タイムアウト [2]
タイムアウト [3]
タイムアウト [4]
タイムアウト [5] 頂点
*/
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

実際に Deno では標準モジュール (std) の `async` モジュールで提供される `delay()` 関数はこのように作られています。リポジトリでは次の場所に存在しています。上で定義したものより緻密に作られていますが内部的には `setTimeout()` をちゃんと使っていることが分かります。

https://github.com/denoland/deno_std/blob/0.145.0/async/delay.ts

https://doc.deno.land/https://deno.land/std@0.145.0/async/mod.ts/~/delay
