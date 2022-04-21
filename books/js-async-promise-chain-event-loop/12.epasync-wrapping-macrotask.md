---
title: "古い非同期APIをPromiseでラップする[作成中]"
---

ここまで Promise チェーンでの Microtask 発行による非同期処理を見てきましたが、Event Loop ではステップ 2 に「Macrotask の実行」がありました。Microtask の基礎ができたのでここからようやく Macrotask の解説になります。

最初に述べたように `setTimoue()` 関数は Web API の一部です。これを使わずに最初説明したのは、この関数に登録するコールバック関数が Macrotask queue へと追加されてしまうからです。

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

Macrotask queue へと追加されるのが分かると思います。

つづく。。。
