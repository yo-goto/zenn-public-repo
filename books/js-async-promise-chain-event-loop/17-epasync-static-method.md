---
title: "Promise の静的メソッド"
aliases: [ch_Promise の静的メソッド]
---

# このチャプターについて

「いつ await すべきか分からない」というのが、一般的に async/await での難しいポイントになりますが、すでに Promise インスタンスを評価して値を取り出すという await 式の特徴については学んでいるため、そこまで難しくは感じないでしょう。

このチャプターでは Promise の静的メソッドをつかって複数の Promise-based API を並列化することを考えます。その際にどの地点で await すべきを考えてみます。

# await で制御する

すでに非同期 API については色々と見てきました。Promsise の静的メソッドを考える前に、すでにお馴染み Promise-based API である `fetch()` メソッドを使用してもう一度 await 式について考えを巡らせておきましょう。

ここでは次の "JSONPlaceholder" というフリーで使える fake API サービスを使って JSON データを複数取得することを考えます。

https://jsonplaceholder.typicode.com

>JSONPlaceholder is a free online REST API that you can use whenever you need some fake data. It can be in a README on GitHub, for a demo on CodeSandbox, in code examples on Stack Overflow, ...or simply to test things locally.
>([JSONPlaceholder - Free Fake REST API](https://jsonplaceholder.typicode.com/) より引用)

```js
const urls = {

}
```


# Promise.all

# Promise.any

# Promise.race

# Promise.settled



