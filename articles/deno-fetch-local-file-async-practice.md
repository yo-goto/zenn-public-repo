---
title: "Denoのローカルfetchで非同期処理の練習"
emoji: "🧗‍♂️"
type: "tech"
topics: [Deno, fetch, 非同期処理, javascript]
published: true
date: 2022-04-06
url: "https://zenn.dev/estra/articles/deno-fetch-local-file-async-practice"
aliases: [記事_Denoのローカルfetchで非同期処理の練習]
tags: [" #deno #JavaScript/WebAPI/Fetch #type/zenn  "]
---

## fetchを練習したい
非同期処理について学んでいると、`fetch` などの処理を見かけることがよくあります。

非同期処理をする API などの学習として `fetch` の練習などをしようとしても、`fetch` する対象データについては、公開されている API などを探したり、回数制限などを気にしたりする必要があるので、手軽に試すことが中々難しいです。また Node.js などの環境では標準で使うことができず、`node-fetch` などの外部ライブラリを `npm install` する必要があります。

## Deno Fetch
そこで、手軽に練習できるものとして、 Deno のビルトイン API として提供さている `fetch` に注目しました。

https://deno.land/manual@v1.20.4/runtime/web_platform_apis#fetch-api

モダンな JavaScript/TypeScript のランタイムである Deno では web 互換な Web API を使うことができます。Web API である Fetch API を使い勝手をそのままに使えるようになっています。

Fetch API 以外にも web 互換な API が多くあり、以下の公式ブログポストでどれくらい互換性があるのかについての説明がなされています。

https://deno.com/blog/every-web-api-in-deno#fetch-request-response-and-headers

Deno の `fetch` の良い点として、ローカルファイルをローカルサーバーなどをたてることなく使える点があげられます。

Deno v1.16 から  `file:` スキームでの `fetch` が行えるようにサポートされたので、サーバーなしで簡単にあつかえるようになりました。

:::message
ここで紹介する Deno の機能は次のバージョンにおいてです。
```sh
❯ deno -V
deno 1.20.4
```
:::

## 使い方
Deno では絶対ファイルパスのみをサポートしているので、`fetch("./some.json")` のような相対パスによる `fetch` は機能しません。

なので、こちらも webAPI である URL API の `URL()` コンストラクタと `import.meta` を使用することで、絶対ファイルパスを作成します。

https://doc.deno.land/deno/stable/~/ImportMeta
https://doc.deno.land/deno/stable/~/URL
https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/import.meta
https://developer.mozilla.org/ja/docs/Web/API/URL

`URL()` コンストラクタの第一引数に相対パス、第二引数にベース URL となるものを入れてパスを構築できます。

`import.meta.url` によってこのモジュールの URL が取得できます。

```js
// このファイルの場所から相対パス
const relativeUrl = "./testTextFile/textForFetch.txt";
const localUrl = new URL(relativeUrl, import.meta.url).toString();
```

これによって、`localUrl` に `file:///Users/roshi/Development/Testing/understanding-async/deno-async/testTextFile/textForFetch.txt` のような `file:` で始まる絶対ファイル URL を得ることができます。

`toString()` を使用して文字列化をしている理由は、Deno の `fetch()` の引数として URL オブジェクトを入れると Deno のリンターに「URL オブジェクトは非推奨なので文字列またはなどを使用しろ」と注意されるためです。

あとは、相対パスで指定した場所に適当なテキストファイルなど用意しておきます。

```txt:testTextFile/textForFetch.txt

In laboris aliquip pariatur aliqua officia veniam quis aliquip. Dolor eu magna reprehenderit pariatur pariatur labore officia. Sit irure et excepteur dolor. Minim tempor nisi nulla veniam mollit. Esse elit aute reprehenderit id minim non et anim non id. Quis sunt elit labore officia voluptate cillum incididunt labore mollit ea adipisicing dolor eiusmod. Veniam cupidatat mollit occaecat mollit ullamco.

```

あとは `fetch()` メソッドの第一引数に URL 文字列を渡して、Promise チェーンで逐次的に処理を行います。

```js
const relativeUrl = "./testTextFile/textForFetch.txt";
const localUrl = new URL(relativeUrl, import.meta.url).toString();

console.log("sync process 1");

fetch(localUrl)
  .then(response => {
    if (!response.ok) {
      throw new Error("Error");
    }
    console.log(`got data from "${localUrl}"`);
    return response.text();
  })
  .then(data => {
    console.log(data);
  })
  .catch(error => {
    console.error(error.message);
  })

console.log("sync process 2");
```

実際に実行してみます。Deno はデフォルトでファイル読み書きなどに対してのセキュリティを設けているので `deno run` で実行する際にはパーミッションフラグが必要となります。

今回はファイルの read を行いたいため、`--allow-read` フラグを実行するファイル名の前に入力して実行します(ファイル名の後だとコマンドライン引数として認識され、パーミッションフラグとしては使えなくなってしまうので注意してください)。

```js
❯ deno run --allow-read denoFetchLocal.js
sync process 1
sync process 2
got data from "file:///Users/roshi/Development/Testing/understanding-async/deno-async/testTextFile/textForFetch.txt"

In laboris aliquip pariatur aliqua officia veniam quis aliquip. Dolor eu magna reprehenderit pariatur pariatur labore officia. Sit irure et excepteur dolor. Minim tempor nisi nulla veniam mollit. Esse elit aute reprehenderit id minim non et anim non id. Quis sunt elit labore officia voluptate cillum incididunt labore mollit ea adipisicing dolor eiusmod. Veniam cupidatat mollit occaecat mollit ullamco.

```

無事に `fetch` でファイル取得をできました。

非同期処理や Promise チェーンの基礎については azu さんの『JavaScript Primer
迷わないための入門書』が大変分かりやすくおすすめです。

https://jsprimer.net/basic/async/

https://jsprimer.net/use-case/ajaxapp/promise/

