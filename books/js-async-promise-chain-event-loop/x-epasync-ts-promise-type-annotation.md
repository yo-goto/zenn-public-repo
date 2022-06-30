---
title: "Promise の型注釈"
---

## TypeScript の非同期処理
TypeScript で Promise の型注釈を考えてみます。

async function では Promise を扱っているということを意識することが非常に重要なので、TypeScript における型注釈を通して考えてみると非常に有用です。

TypeScript についてはオープンソースで公開されている『サバイバル TypeScript』が非常に参考になります。自分もお世話になっています。

ここでは以下のページを参考に Promise の型注釈および Async function における型注釈を考えてみます。

https://typescriptbook.jp/reference/promise-async-await

## Promise の型注釈
Promise の型注釈を理解する上ではジェネリクスの概念と型引数・型変数を理解する必要があります。

Promise インスタンスを返す関数は次のように型注釈を書きます。

```ts
function returnPromise(): Promise<string> {
  return new Promise(resolve => {
    resolve("解決値が文字列の場合は <string> にする");
  });
}
```

