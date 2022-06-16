---
title: "同期 API とブロッキング"
aliases: [ch_同期 API とブロッキング]
---

# このチャプターについて

このチャプターでは、非同期 API と対になっている同期 API について解説します。同期 API を理解することで非同期 API の有用性と難しさについて理解できます。また、ソースコードの配置と実行タイミングの関係性について考えることができる有用なケースとなります。

# 同期 API は意図的にブロッキングする

非同期 API と環境について前のチャプターで解説しました。非同期 API のおかげで環境がバックグラウンドで API 処理をしてくれている間も、同時に JavaScript コードを実行できるようになっているのは、実際のところ非同期 API がメインスレッドを(専有しないように)ブロッキングしないように**デザインされている**からです。

https://nodejs.org/ja/docs/guides/blocking-vs-non-blocking/

その一方で、Node 環境や Deno 環境では意図的にブロッキングを起こすようにデザインされた「**同期 API(Synchronous API)**」が存在しています。`console.log()` などの Web APIs(Web Platform APIs) は置いておいて、そういった API は名前の最後が `Sync` で終わるケースのものとして提供されています(I/O 関連の処理など)。

- 非同期 API (Non-blocking)
  - Node:
    - [fs.writeFile](https://nodejs.org/dist/v18.2.0/docs/api/fs.html#fswritefilefile-data-options-callback) (Callback-based API)
    - [fsPromises.writeFile](https://nodejs.org/dist/v18.2.0/docs/api/fs.html#fspromiseswritefilefile-data-options) (Promise-based API)
  - Deno: [Deno.writeFile](https://doc.deno.land/deno/stable/~/Deno.writeFile) (Promise-based API)
- 同期 API (Blocking) 
  - Node: [fs.wirteFileSync](https://nodejs.org/dist/v18.2.0/docs/api/fs.html#fswritefilesyncfile-data-options)
  - Deno: [Deno.wirteFileSync](https://doc.deno.land/deno/stable/~/Deno.writeFileSync)

同期 API を使うことで**ソースコードの配置と処理順番が完全に一致するようにできます**。つまり、コード配置と実行順序がずれてしまう難しい非同期処理を考える必要がなくなります。ただし、同時に複数のことができる非同期 API のメリットを捨て去ることになります。

例えば、Deno 環境においてテキストファイルにデータを書きこんだ後に読みこんでコンソールに出力することを同期 API(`Deno.writeTextFileSync` と `Deno.readTextFileSync`)と非同期 API(`Deno.writeTextFile` と `Deno.readTextFile`)のそれぞれで考えてみます。

```js:apiSync.js(同期APIを利用したコード)
// apiSync.js
const path = "./tests/helloSync.txt";
const inputData = "Hello Synchronously Zenn!";

// 書いた順番に実行される(ソースコードの配置順番どおりにしている)
console.log("[1]");

Deno.writeTextFileSync(path, inputData); // blocking
// この処理が完了してから次の処理に進む
const data = Deno.readTextFileSync(path); // blocking
// この処理が完了してから次の処理に進む
console.log("[2]", data);

console.log("[3]");
```

同期 API は意図的にメインスレッドをブロッキングするようにデザインされているので、ソースコードを上から下に書いたとおりに実行し、一行ずつ完了するのを待って次の行に移動します。実際に上のコードを実行すると次の出力を得ます。

```sh
# read と write のパーミッションが必要なので --allow-all で代用
❯ deno run --allow-all apiSync.js
[1]
[2] Hello Synchronously Zenn!
[3]
```

そのため、最後の `console.log()` がファイルの書き込み・読み出しの完了が終わってから実行されています。

一方、非同期 API を利用した場合はどうなるでしょうか。Deno の非同期 API である `Deno.writeTextFile()` と `Deno.readTextFile()` はそれぞれ Promise インスタンスを返してきますので、Promise チェーンが構築できます(API の名前の最後に `Sync` がついていないことに注意してください)。

```js:apiAsync.js(非同期APIを利用したコード)
// apiAsync.js
const path = "./tests/helloAsync.txt";
const inputData = "Hello Asynchronously Zenn!";

// 書いた順番に実行されない(ソースコードの配置順番どおりにしていない)
console.log("[1]");

// この処理の完了を待たずに次の処理に進む
Deno.writeTextFile(path, inputData) // non-blocking
  .then(() => Deno.readTextFile(path)) // non-blocking
  .then((data) => console.log("[3]", data));

// 上の処理の完了を待たずにコンソールに出力
console.log("[2]");
```

非同期 API はメインスレッドをブロッキングしないようにデザインされているので、ソースコードは上から下に書いたとおりには実行されません。その代わりに、環境が時間のかかる非同期 API の処理を裏で行っている間も同時に別のことができます。実際に上のコードを実行すると次の出力を得ます。

```sh
# read と write のパーミッションが必要なので --allow-all で代用
❯ deno run --allow-all apiAsync.js
[1]
[2]
[3] Hello Asynchronously Zenn!
```

そのため、ファイルの書き込み・読み出しの完了を待たずに最後の `console.log()` を実行できています。

アプリケーションではなく書き捨てのスクリプトや簡単なテストではこの「同期 API」が役立ちます。**書いた順番通りに実行されるので明らかに処理の流れが分かりやすい**からです。ただし、一度に複数のことができる非同期 API のメリットを捨てることになるので、明らかに非同期 API より時間がかかることになります。要するに非同期 API は「**効率が良い**」ということです。

# コード配置のための書き方

結局のところ非同期処理そのものは非同期 API が登場しない限り出番がありません。そして「**同時に複数のことをやりたい**」がためにわざわざ難しい非同期 API を使います。同時に複数のことをやっているので、バックグラウンドでの API 処理が完了したら別の作業を非同期的にメインスレッドで行うための「ソースコードの書き方」が必要になります。そのシンタックス(書き方)が Callback hell や Promise chain、async/await です。

```js:Promise chain
// [1] -> [2] -> [3] という順番で実行されることを保証する
Deno.writeTextFile(path, inputData) // [1]
  .then(() => Deno.readTextFile(path)) // [2]
  .then((data) => console.log("[3]", data)); // [3]
```

```js:async/await
// 上のコードを async/await で書き換えた
(async function writeAndRead() {
  // [1] -> [2] -> [3] という順番で実行されることを保証する
  await Deno.writeTextFile(path, inputData); // [1]
  const data = await Deno.readTextFile(path); // [2]
  console.log("[3]", data); // [3]
})(); // 即時実行
```

もちろん、現実的にはエラーハンドリングが付き纏うので完全な保証ではないです。上の例も説明のために例外処理を省いていますので注意してください。

非同期 API を起点にした一連の作業が特定順序で実行されることを保証するための書き方とその仕組みを学ぶということが非同期処理の学習です。つまり、「逐次(sequential)処理」をどうやって書いて、どういうメカニズムでその処理が実現されているのかを知ることが重要ということです。

