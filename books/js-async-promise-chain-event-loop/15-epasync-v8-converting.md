---
title: "V8 エンジンによる async/await の内部変換"
cssclass: zenn
date: 2022-05-14
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: ch_V8 エンジンによる async/await の内部変換
---

# このチャプターについて

このチャプターでは、V8 エンジンによる async/await の内部変換コードから async/await の舞台裏を探索し、その挙動について理解するための解説を行います。

仕様を直接見るよりも、V8 エンジンでどうなっているかを見た方が分かりやすいので V8 からアプローチします。前のチャプターで見たとおり、async/await では若干謎の挙動が存在しています。V8 エンジンの内部変換コードを見ることでその謎は解決できます。

:::message
このチャプターは『[V8エンジンによる内部変換コードでasync/awaitの挙動を理解する](https://zenn.dev/estra/articles/asyncawait-v8-converting)』の記事と同じ内容になるので、すでに読まれた方はスキップしてもらって構いません。
:::

さて、『[V8 エンジンについて](e-epasync-v8-engine)』のチャプターで V8 エンジンについての予備知識はいれておきましたね。このチャプターでは、V8 公式のブログ記事とプレゼン動画を元に解説していきます。

# V8 エンジンによる内部変換コード

それでは、V8 開発チームの Maya Lekova 氏と Benedikt Meurer 氏によるプレゼン動画『Holding on to your Performance Promises』と、それに基づく V8 エンジン公式サイトのブログ記事『Faster async functions and promises』を元にして async/await の V8 エンジンでの内部変換コードを見ていきます。

ブログ記事だけだと分かりづらい部分があると感じたので、動画も一緒に視聴することをおすすめします。平易な英語なので比較的聞きやすいと思います。

https://v8.dev/blog/fast-async#await-under-the-hood

https://youtu.be/DFP5DKDQfOc

:::message
ブログ記事と動画において、「以前の ECMAScript の仕様では async/await のオーバーヘッド(余計な Promise インスタンスとマイクロタスクの生成)があったため、V8 エンジンでそれを改善した上で、**ECMAScript の仕様自体にその最適化をマージし**、V8 エンジンだけでなく**あらゆるエンジンで同じ最適化ができるようにした**」ということが述べられています。

https://github.com/tc39/ecma262/pull/1250

このチャプターではそれらを全部すっ飛ばして結論としての変換コードから見ていきますので、注意してください。細かい部分については元々の動画とブログを参考にしてください。
:::

では結論として、V8 エンジンでは次のような async/await を内部的に変換しています。

```js:シンプルな async 関数
async function foo(v) {
  const w = await v;
  return w;
}
```

変換後は以下のようになります(実際に公式ブログ記事に示されているものですが、疑似コード的なものであると考えられます)。

```js:V8エンジンによる変換コード
// 途中で一時停止できる関数として resumable (再開可能) のマーキング
resumable function foo(v) {
  implicit_promise = createPromise();
  // (0) async 関数の返り値となる Promise インスタンスを作成

  // (1) v が Promise インスタンスでないならラッピングする
  promise = promiseResolve(v);
  // (2) async 関数 foo を再開またはスローするハンドラのアタッチ
  performPromiseThen(
    promise,
    res => resume(«foo», res),
    err => throw(«foo», err));

  // (3) async 関数 foo を一時停止して implicit_promise を呼び出し元へと返す
  w = suspend(«foo», implicit_promise);
  // (4) w = のところから async 関数の処理再開となる

  // (5) async 関数で return していた値である w で最終的に implict_promise を解決する
  resolvePromise(implicit_promise, w);
}

// 内部で使う関数
function promiseResolve(v) {
  // v が Promise ならそのまま返す
  if (v is Promise) return v;
  // v が Promise でないならラッピングして返す
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

基本的なステップはコメントに書いた通りです。

- (0) V8 エンジンによって async 関数自体が実行を一時停止して後から再開できる関数として、reusable(再開可能)のマーキングをし、async 関数自体の返り値となる Promise インスタンスとして `implicit_promise` を作成します
- (1) await 式の評価対象について Promise インスタンスでないならラッピングして `promise` に代入します
- (2) `promise` が Settled になったときのハンドラを同期的にアタッチします
- (3) async 関数の処理を `suspend()` で一時停止して、Promise インスタンスである `implicit_promise` を呼び出し元へと返却します
- (4) `promise` が Settled となり次第、async 関数の処理を再開し、await 式の評価結果を `w` に代入するところから処理再開となります
- (5) 最終的に async 関数内部で `return` していた値で `implicit_promise` を resolve することで呼び出し元に返されていた Promise インスタンスが Settled となります

変換後のコードで普通の `return` が存在していないのは、`suspend()` の時点で呼び出し元である Caller へと Promise インスタンスとして `implicit_promise` を返してるからです。async 関数はどんなときでも、Promise インスタンスを返します。async 関数の処理が一時停止して、呼び出し元に制御が戻った時にすでに返り値として Promise インスタンスを用意していなければいけません。ただし、その時に返り値の Promise インスタンスが履行されている必要はなく、Pending 状態のままでいいのです。

再び、async 関数の処理が再開し、最終的に async 関数で `return w` としていた値 `w` で `implicit_promise` が解決されることで、呼び出し元に返ってきていた Promise インスタンスが Settled になり、その値 `w` を Promise chain などで利用できるようになります。

`implicit_promise = createPromise()` は後から解決される Promise インスタンス `implicit_promise` を作成し、`reoslvePromise(implicit_promise, w)` では作成したその Promise インスタンスを後から `w` で解決しています。細かい実装は分からないので、ここではそういうものだと考えてください。

ということで、上記コードの説明としてもう少しコメントを追加しておきたいと思います。`v` は `v = Promise.resolve(42)` というように値 `42` で既に履行されている Promise インスタンスとして想定します。

```js:V8エンジンによる変換コード
// 途中で一時停止できる関数として resumable (再開可能) のマーキング
// async 関数からは、susupend のところまで行った時点で処理を中断して Pending 状態の Promise インスタンス(implicit_promise)が呼び出し元に返される
// 通常の return は意味がない(generator の yield と同じ)
resumable function foo(v) {
  implicit_promise = createPromise();
  // async 関数の返り値となる promise インスタンスを作成
  // 非同期処理を一時停止(susupend)したときもこれが呼び出し元に返ってきている

  // １つの await 式 (必ず１つはマイクロタスクが生成される)
  // 1. v を promise でラップする
  promise = promiseResolve(v); // v がプロミスでないならラッピング
  // 2. foo を再開するハンドラのアタッチ
      // Promise.prototype.then() が裏側で行っていることと同じ
      // promise が Settled になったらマイクロタスクを発行
      // マイクロタスクは PromiseReactionJob で async 関数の処理再開を告げる
  performPromiseThen(
    promise,
    res => resume(«foo», res),
    err => throw(«foo», err));
    // アタッチしているだけでとりあえず次に進む
  // 3. foo (async 関数)を一時停止して implicit_promise を caller へと返す
  w = suspend(«foo», implicit_promise);
  // ここまでが１つの await で、foo のコンテキストを一旦ポップする
  // w には await 式の評価結果の値が代入される(yields 42 from the await)
  // w = のところに値が入り実行再開する(w には promise の履行値 42 が入る)

  resolvePromise(implicit_promise, w); // return する値 w (= 42)で resolve する
  // caller へ返していた Promise インスタンスが Settled になる
}

// 使う関数
function promiseResolve(v) {
  // v が promise ならそのまま返す
  if (v is Promise) return v;
  // そうでないならプロミスでラップして返す
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

基本的に `w = await v;` のように各 await 式ごとに次の部分が必要となります。`w =` のように代入しないなら単に `suspend(«foo», implicit_promise);` となり、そのポイントから処理再開となることは変わりません。

```js
  // (1) v が Promise インスタンスでないならラッピングする
  promise = promiseResolve(v);
  // (2) async 関数 foo を再開するハンドラのアタッチ
  performPromiseThen(
    promise,
    res => resume(«foo», res),
    err => throw(«foo», err));
  // (3) async 関数 foo を一時停止して implicit_promise を呼び出し元へと返す
  w = suspend(«foo», implicit_promise);
```

別のプレゼンの前資料である次のドキュメントから借用したコードで考えると次のようにもできます。

[Zero-cost async stack traces - Google ドキュメント](https://docs.google.com/document/d/13Sy_kBIJGP0XT34V1CV3nkWya4TwYx9L3Yv45LdGB6Q/)

```js:別の書き方
const .promise = @promiseResolve(x);
@performPromiseThen(.promise,
  res => @resume(.generator_object, res),
  err => @throw(.generator_object, err));
@yield(.generator_object, .outer_promise);
```

:::details ジェネレータ関数の yield
上のコードの書き方で `yield` というキーワードがでてきましたが、async 関数と `yield` キーワードが内部で利用できるジェネレータ関数には関係性があります。

まず、ジェネレータ関数では `yield` の数だけ関数の処理を一時停止して値を生み出すことができます。

```js:yieldSample.js
// ジェネレータ関数の定義
function* generatorFn(n) {
  n++;
  yield n;
  n *= 5;
  yield n;
  n = 0;
  yield n;
}
// ジェネレータオブジェクトをジェネレータ関数から取得
const generator = generatorFn(5);

// ジェネレータオブジェクトの next メソッドでイテレータリザルトを返す
console.log(generator.next());
// { value: 6, done: false }
console.log("関数を一時停止してなにか別の処理");
console.log(generator.next());
// { value: 30, done: false }
console.log("関数を一時停止してなにか別の処理");
console.log(generator.next());
// { value: 0, done: false }
console.log("関数を一時停止してなにか別の処理");
console.log(generator.next());
// { value: undefined, done: false }
console.log("ジェネレータ関数内のすべての処理を終了");
```

このスクリプトを実行すると次のような出力を得ます。
```sh
❯ deno run yieldSample.js
{ value: 6, done: false }
関数を一時停止してなにか別の処理
{ value: 30, done: false }
関数を一時停止してなにか別の処理
{ value: 0, done: false }
関数を一時停止してなにか別の処理
{ value: undefined, done: true }
ジェネレータ関数内のすべての処理を終了
```

async/await では最初の await 式でのみ暗黙的に async 関数から返される Promise インスタンスを `yield` していると考えることができます。それ以降は await 式による評価のたびに一時停止しますが、呼び出し元に値を返しません。最終的に async 関数内の処理がすべて完了すると async 関数内で `return` されている値で最初に返した Promise インスタンスを履行します。

あるいは async 関数内部でジェネレータが使われているとも考えることができます。実際、async/await が ECMAScript に導入されるまではこのジェネレータ関数と Promise インスタンスを組み合わせて async 関数のようなものつくっていたそうです。async 関数を使ったコードを Babel や TypeScript で古い JavaScript にトランスパイルする際にはジェネレータ関数と Promise インスタンスを組み合わせて実現しています。

- 参考: [async await - TypeScript Deep Dive 日本語版](https://typescript-jp.gitbook.io/deep-dive/future-javascript/async-await)

ジェネレータ関数について知らなければこのことについてはとりあえずは無視してもよいです。

ジェネレータ関数や `yield` については第４章の「[イテレータとイテラブルとジェネレータ関数](k-epasync-iterator-generator)」で解説します。
:::

## await 式は確実にマイクロタスクを１つ発行する

`performPromiseThen()` の箇所に注目してほしいのですが、これは `Promise.prototype.then()` が舞台裏でやっていることと本質的に同じとなります。

`peformPromiseThen()` に渡す引数である `promise` が Settled になることで、`then()` メソッドのコールバックのようにマイクロタスクが発行されます。このマイクロタスクは `PromiseReactionJob` と呼ばれています。

この `PromiseReactionJob` というマイクロタスクがマイクロタスクキューからコールスタックへと送られます。そのマイクロタスクによって更にコールスタック上で async 関数の関数実行コンテキストが再度プッシュされて積まれることで処理を再開できるようになっています。await 式ごとにこの `performPromiseThen()` の実行が必要となります。`then()` メソッドのようにマイクロタスクが発行されるので、Promise chain で考えれば理解できるはずです。

## await 式が２個ある場合

それでは、今までの内容を踏まえて、今度は await 式が２個ある場合を考えてみます。

```js:await式が２個ある async 関数
async function foo2(v, x) {
  await v;
  console.log("Microtask1");
  await x;
  console.log("Microtask2");
  return 42;
}
```

↓ V8 エンジンによる変換として考えられるコード。

```js:V8エンジンによる変換コード
resumable function foo2(v, x) {
  implicit_promise = createPromise();
  // async 関数の返り値となる promise インスタンスを作成

  // <<await v>>
  promise1 = promiseResolve(v);
  performPromiseThen(promise1,
    res => resume(«foo2», res),
    err => throw(«foo2», err));
  suspend(«foo2», implicit_promise);
  // 呼び出し元に implicit_promise を返す
  // 中断かつ処理再開のポイント

  console.log("Microtask1");

  // <<await x>>
  promise2 = promiseResolve(x);
  performPromiseThen(promise2,
    res => resume(«foo2», res),
    err => throw(«foo2», err));
  suspend(«foo2», implicit_promise);
  // implicit_promise はすでに返されているのでここでは一時停止するだけ
  // 中断かつ処理再開のポイント

  console.log("Microtask2");

  resolvePromise(implicit_promise, 42);
  // 最終的に return する値 42 で resolve する
}
```

# 色々なパターン

さて、基本的な変換が分かったので、もう少し深く潜ってみたいと思います。この変換を基本系に色々な async/await を考えてみます。

こちらの uhyo さんの記事で紹介されているような色々なパターンと、その速度(マイクロタスクをいくつ発行するか)についても考えてみましょう。

https://zenn.dev/uhyo/articles/return-await-promise

## 通常の関数で Promise を返す場合

まずは、比較対象として Promise インスタンスを返す通常の関数を考えてみましょう。

```js
// asyncSpeedY.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve()
  .then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

// 通常の関数で即時実行
(function returnPromise() {
  // return new Promise(resolve => {
  //   resolve();
  // });
  // どっちでも同じ
  return Promise.resolve();
  // マイクロタスクは発生しない
})().then(() => console.log("👦 [4] <2-Sync> MICRO: then after function"));

Promise.resolve()
  .then(() => console.log("👦 [5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <4-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

通常の関数なので V8 エンジンによる async/await の変換はありません。

`Promise.resolve().then()` によって同期的に(直ちに)マイクロタスクキューへコールバックがマイクロタスクとして発行されます。また、即時実行関数の中でも履行状態で作成される Promise インスタンスが返されるため、次の `then()` メソッドのコールバックが同期的に(直ちに)マイクロタスクキューへマイクロタスクとして発行されます。ということで、関数内部で余計なマイクロタスクは発生しません。

これを V8 エンジンで実行すると次のように予測が簡単な出力を得ます。Chrome、Node、Deno でやっても全部同じです。

```sh
# v8 コマンドで JavaScript ファイルを実行
❯ v8 asyncSpeedY.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👦 [4] <2-Sync> MICRO: then after function
👦 [5] <3-Sync> MICRO: then
👦 [6] <4-Async> MICRO: then
```

:::message
V8 エンジンでは Web API である `queueMicrotask()` は提供されないので、代わりに `Promise.resolve().then()` でマイクロタスクを発行しています。

通常 `Promise.resolve().then()` は `queueMicrotask()` よりもオーバーヘッドがあるので、マイクロタスクを発行するだけなら、`queueMicrotask()` を使用します。
:::

## await も return も無い場合

それでは次に、`await` 式も `return` も無い async 関数を考えてみましょう。次のようなシンプルに何もしない async 関数の変換はどうなるでしょうか?

```js:何もしない async 関数
async function empty() {}
```

V8 エンジンは次のように内部的に変換すると想定されます。

```js:V8_Converting
resumable function empty() {
  implicit_promise = createPromise();

  // await 式はないので中断しない

  resolvePromise(implicit_promise, undefined);
  // return する値はないので undefined で resolve する
  // 返される Promise インスタンスは直ちに履行状態となる(マイクロタスクは発生しない)
}
```

`await` がないので、各 await 式に必要ないつものコードはありません。そして、`return` している値も無いので、`return` する値は `undefined` となり、async 関数から返される Promise インスタンスは `undefined` で解決されます。

そして `peformPromiseThen()` が無いのでマイクロタスクは１つも発行されず、async 関数から返ってくる Promise インスタンスはただちに履行状態となります。

:::message
async 関数(Async function)はどんなときでも必ず Promise インスタンスを返します。
:::

それでは、次のコードの実行順番を予測します。

```js
// asyncSpeed1.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

// async 関数を即時実行
(async function empty() {})().then(() => console.log("👦 [4] <2-Sync> MICRO: then after async function"));

Promise.resolve()
  .then(() => console.log("👦 [5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <4-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

今回も即時実行で関数を実行します。async 関数からは Promise インスタンスが必ず返ってくるので、`then()` メソッドで Promise chain を構築できます。

それではマイクロタスクについて考えてみましょう。

まずは、`Promise.resolve().then()` で同期的にマイクロタスクが発行されて、その次の肝心の async function の即時実行でも、上の変換で見たように関数から返えされる Promise インスタンス自体は直ちに履行状態となるので、`then()` メソッドのコールバックがマイクロタスクとしてマイクロタスクキューに送られます。次の `Promise.resolve.then()` メソッドのコールバックも同期的にマイクロタスクを発行してキューへ送られます。

スクリプト評価による同期処理がすべて終わり、コールスタックからグローバルコンテキストがポップして破棄されることで、コールスタックが空になるので、マイクロタスクのチェックポイントとなります。マイクロタスクキューの先頭にあるものから順番にすべて処理されていきます。

３番目にマイクロタスクキューへ送られたコールバック `() => console.log("👦 [5] <3-Sync> MICRO: then")` が実行された時点で、元々の `Promise.reoslve().then()` で返ってくる Promise インスタンスが履行状態となるので、`Promise.resolve().then().then()` のコールバックがマイクロタスクキューに送られて直ちにコールスタックへと積まれて実行されます。

ということで、実行結果は次のようになります。

```sh
❯ v8 asyncSpeed1.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👦 [4] <2-Sync> MICRO: then after async function
👦 [5] <3-Sync> MICRO: then
👦 [6] <4-Async> MICRO: then
```

## await 42 の場合

次は、async 関数内で `await 42` だけをする場合を考えてみます。

:::message
42 という数字はマジックナンバーで、「任意の数値」を意味するケースとして利用されます。
:::

```js:foo4
async function foo4() {
  await 42;
}
```

↓ V8 エンジンによる内部変換として想定されるコード。

```js:V8_Converting
resumable function foo4() {
  implicit_promise = createPromise();

  // <- await 式
  promise = promiseResolve(42); // プロミスでないのでラップする
  // promise が Settled になったら処理再開のためのマイクロタスクを発行
  // すでに Settled となるので直ちにマイクロタスクを発行
  performPromiseThen(
    promise,
    res => resume(«foo4», res),
    err => throw(«foo4», err));
  // async 関数を一時停止して、呼び出し元に implicit_promise を返す
  suspend(«foo4», implicit_promise);
  // await 式 -> (再開処理だが特にやることはない)

  resolvePromise(implicit_promise, undefined);
  // 呼び出し元への返り値である implicit_promise に対して
  // return したものはなにもないので undefined で resolve する

  // 発生するマイクロタスクは合計１つ
  // (つまりthenのコールバックを起動できるまでマイクロタスク一個分)
}
function promiseResolve(v) {
  if (v is Promise) return v;
  // promise ではないのでラッピングする
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

await 式というのは通常は Promise インスタンスを評価し、Promise インスタンスの評価結果としてその履行値を返すという使いかたをしますが、Promise インスタンスでないものも評価できます。

その場合は、`promise = promiseResolve(42)` であるように、Promise インスタンスでない場合として新しい Promise でラッピングされます(await 式で評価する値自体で解決する Promise インスタンス)。

いずれにせよ `performPromiseThen()` を行うため、作成された Promise インスタンスが Settled になるまで待ち、Settled になった時点で async 関数の処理再開を告げるマイクロタスクを発行します。この場合は Promise インスタンスがすぐに履行状態になるので、同期的にマイクロタスクを直ちに発行します。

ということで、async 関数から返ってくる Promise インスタンスにチェーンする `then()` メソッドのコールバックの実行はマイクロタスク１回が実行されるまで待つ必要があります。

```js
// asyncSpeed8.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

// async function から返る Promise はマイクロタスク一個の実行で履行状態で then でマイクロタスク発行
(async function foo4() {
  await 42;
  // マイクロタスク一個だけ発行する
  // <2-Sync>
})().then(() =>
  console.log("👻 [5] <4-Async> MICRO: then after async function")
);

Promise.resolve()
  .then(() => console.log("👦 [4] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <5-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

ということで、今までの場合と違い async 関数の内部でマイクロタスクが一個だけ発行されるので、async 関数から返される Promise インスタンスが履行状態になるにはそのマイクロタスクが処理される必要があります。従って、chain している `then()` メソッドのコールバックがマイクロタスクとして発行されるタイミングが今までのようにすぐにではなく、ずれることになります。

従って、実行順番は次のようになります。

```sh
❯ v8 asyncSpeed8.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👦 [4] <3-Sync> MICRO: then
👻 [5] <4-Async> MICRO: then after async function
👦 [6] <5-Async> MICRO: then
```

## await Promise.resolve(42) の場合

今度は、すでに履行状態の Promise インスタンスを await してみましょう。

```js:fooZ
async function fooZ() {
  await Promise.resolve(42);
}
```

↓ V8 エンジンによる内部変換コードは次のようになると想定されます。

```js:V8_Converting
resumable function fooZ() {
  implicit_promise = createPromise();

  // <- await 式
  promise = promiseResolve(Promise.resolve(42)); // プロミスなのでそのまま返す
  // promise が Settled になったら処理再開のためのマイクロタスクを発行
  // すでに Settled となるので直ちにマイクロタスクを発行
  performPromiseThen(
    promise,
    res => resume(«fooZ», res),
    err => throw(«fooZ», err));
  // async 関数を一時停止して、呼び出し元に implicit_promise を返す
  suspend(«fooZ», implicit_promise); // await 式 ->
  // 再開処理だが特にやることはない

  resolvePromise(implicit_promise, undefined);
  // 呼び出し元への返り値である implicit_promise に対して
  // return したものはなにもないので undefined で resolve する

  // 発生するマイクロタスクは合計１つ
  // (つまりthenのコールバックを起動できるまでマイクロタスク一個分)
}
function promiseResolve(v) {
  // プロミスなのでそのまま返す 
  if (v is Promise) return v;
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

この場合は実は `await 42` と同じで、内部的にマイクロタスクを１つ発行することになります。そういう訳で、`await` で**何を評価しようが少なくともマイクロタスク１つが発行される**ことになります。各 await 式において最低でも１つマイクロタスクが発行されます。

ということで、次のように `Math.random() < 0.5` で 50% ずつの確率で分岐するコードでは実行結果は同じになります。

```js
// awaitPlainValue.js
const returnPromise = () => Promise.resolve();
console.log("🦖 [1] MAINLINE: Start");

(async () => {
  console.log("🦖 [2] MAINLINE: In async function");
  // どちらの場合でも同じ
  if (Math.random() < 0.5) {
    await 1;
    console.log("👦 [4] <1> MICRO: after await");
    await 2;
    console.log("👦 [6] <3> MICRO: after await");
    await 3;
    console.log("👦 [8] <5> MICRO: after await");
  } else {
    await returnPromise();
    console.log("👦 [4] <1> MICRO: after await");
    await returnPromise();
    console.log("👦 [6] <3> MICRO: after await");
    await returnPromise();
    console.log("👦 [8] <5> MICRO: after await");
  }
  return 4;
})().then(() => console.log("👦 [10] <7> MICRO: then cb after async func"));

Promise.resolve()
  .then(() => console.log("👦 [5] <2> MICRO: then cb"))
  .then(() => console.log("👦 [7] <4> MICRO: then cb"))
  .then(() => console.log("👦 [9] <6> MICRO: then cb"));

console.log("🦖 [3] MAINLINE: End");
```

実行順番は次のようになります(どちらの場合でも同じ)。

```sh
❯ v8 awaitPlainValu.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: In async function
🦖 [3] MAINLINE: End
👦 [4] <1> MICRO: after await
👦 [5] <2> MICRO: then cb
👦 [6] <3> MICRO: after await
👦 [7] <4> MICRO: then cb
👦 [8] <5> MICRO: after await
👦 [9] <6> MICRO: then cb
👦 [10] <7> MICRO: then cb after async func
```

## await promise chain の場合

次は、既に履行状態の Promise インスタンスではなく、履行するまで１つマイクロタスクが必要な Promise chain を await してみましょう。

```js:foo9
async function foo9() {
  await Promise.resolve("😭").then(value => console.log(value));
}
```

↓ V8 エンジンによる内部変換。

```js:V8_Converting
resumable function foo9() {
  implicit_promise = createPromise();

  // <- await 式
  promise = promiseResolve(Promise.resolve("😭").then(value => console.log(value)));
  // promise インスタンスなのでそのまま返す
  // promise が Settled になったら処理再開のためのマイクロタスクを発行
  // その前に一回はマイクロタスクが必要
  performPromiseThen(
    promise,
    res => resume(«foo9», res),
    err => throw(«foo9», err));
  suspend(«foo9», implicit_promise);
  // await 式 ->
  // ここまでで二回はマイクロタスクを使用している
  // 再開処理だが特にやることはない

  resolvePromise(implicit_promise, undefined);
  // 呼び出し元への返り値である implicit_promise に対して
  // return したものはなにもないので undefined で履行

  // 発生するマイクロタスクは合計2つ
  // (つまりthenのコールバックを起動できるまでマイクロタスク2個分)
}
function promiseResolve(v) {
  // promise インスタンスなのでそのまま返す
  if (v is Promise) return v;
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

何を await しようがマイクロタスクは確実に１個発行されますが、今回のケースでは、`promise` が Settled になるまでに１つマイクロタスクが必要となります。それが実行されてから、`promise` が Settled になりマイクロタスクが再び発行されるので、マイクロタスクは合計２つ必要となります(今までの場合よりも一個多い)。

実際のコードを考えてみましょう。ここまで来ると非常に予測が難しくなります。

```js
// asyncSpeed9.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

(async function foo9() {
  await Promise.resolve("😭").then((value) =>
    console.log("🦄 [4] <2-Sync> MICRO: then inside", value) // これが一回分
  );
  // async 関数から返される Promise インスタンスが履行するまで合計マイクロタスク２回分必要
  // <4-Async>
})().then(() => console.log("👻 [7] <6-Async> MICRO: then after async function"));

Promise.resolve()
  .then(() => console.log("👦 [5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <5-Async> MICRO: then"))
  .then(() => console.log("👦 [8] <7-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

`<>` で囲んである数字はマイクロタスクキューに追加される順番で、`Sync` は同期的にマイクロタスクキューに送られて、`Async` はその前のマイクロタスク実行後に非同期的にマイクロタスクキューに送られる場合となっています。

async 関数から返される Promise インスタンスが履行状態になるまでに内部発生するマイクロタスク２個分が実行される必要があるため、実行結果は次のようになります。

```sh
❯ v8 asyncSpeed9.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
🦄 [4] <2-Sync> MICRO: then inside 😭
👦 [5] <3-Sync> MICRO: then
👦 [6] <5-Async> MICRO: then
👻 [7] <6-Async> MICRO: then after async function
👦 [8] <7-Async> MICRO: then
```

というわけで、Promise chain を await するとチェーンの数だけマイクロタスクが必要となります。

## return 42 の場合

今度は、async 関数の中で何も await せずに単なる数値 `42` を返す async 関数を考えてみます。

```js:foo0
async function foo0() {
  return 42;
}
```

↓ V8 エンジンで内部変換されるコードは次のようになると想定されます。

```js:V8_Converting
resumable function foo0() {
  implicit_promise = createPromise();
  // <- await 式 が無いので中断しない ->
  resolvePromise(implicit_promise, 42);
  // 最終的に return する値 42 で resolve する
  // 内部ではマイクロタスクは１つも生成されない
}
```

await も return も無い場合と同じく、この場合はマイクロタスクが１つも発生しません。ということは、この async 関数から返される Promise インスタンスは同期的に(直ちに)履行状態となります。

```js
// asyncSpeed0.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

// async function から返る Promise は直ちに履行状態で then でマイクロタスク発行
(async function foo0() { return 42; })().then(() =>
  console.log("👻 [4] <2-Sync> MICRO: then after async function")
);

Promise.resolve()
  .then(() => console.log("👦 [5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <4-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

このコードの実行結果は次のようになります。前のケースと同じです。

```sh
❯ v8 asyncSpeed0.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👻 [4] <2-Sync> MICRO: then after async function
👦 [5] <3-Sync> MICRO: then
👦 [6] <4-Async> MICRO: then
```

## return await Promise.resolve(42) の場合

さて、そろそろ問題のケースに突入します。実際に問題となるのは、`return Promise.resolve(42)` の場合なのですが、その前に簡単な `return await Promise.resolve(42)` を考えてみます。

次のようなシンプルな async 関数を再び考えてみます。

```js:foo4
async function foo4() {
  return await Promise.resolve(42);
}
```

こままだと V8 の変換がしづらいので分解して考えてみましょう。次のコードは上と同じです。

```js:foo4
async function foo4() {
  const value = await Promise.resolve(42);
  return value;
}
```

この V8 エンジンによる内部変換コードは次のようになると想定されます。

```js:V8_Converting
resumable function foo4() {
  implicit_promise = createPromise();

  // <- await 式
  promise = promiseResolve(Promise.resolve()); // プロミスならそのまま返す
  // promise が Settled になったら処理再開のためのマイクロタスクを発行
  // すでに Settled なのですぐにマイクロタスクを発行
  performPromiseThen(
    promise,
    res => resume(«foo4», res),
    err => throw(«foo4», err));
  // async 関数を一時停止して、呼び出し元に implicit_promise を返す
  value = suspend(«foo4», implicit_promise); // await 式 ->
  // <- value = に await 式の評価結果が入るところから再開処理 ->
  // value には Promiseから取り出された値(この場合は 42)が入る

  resolvePromise(implicit_promise, value);
  // 呼び出し元への返り値である implicit_promise に対して
  // 元々 return していた値 value で resolve する
  // value は await 式で取り出された 42

  // 発生するマイクロタスクは合計１つ
  // (つまりthenのコールバックを起動できるまでマイクロタスク一個分)
}
// 使う関数
function promiseResolve(v) {
  // プロミスならそのまま返す
  if (v is Promise) return v;
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

この場合はマイクロタスクが１つですみます。後で説明しますが、実は `return Promise.resolve(42)` では１つでは済みません。

実際のコードで再び考えてみましょう。

```js
// asyncSpeed4.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

(async function foo4() {
  return await Promise.resolve();
  // 内部的にマイクロタスクが１つだけ生成される <2-Sync>
})().then(() => console.log("👻 [5] <4-ASync> MICRO: then after async function"));

Promise.resolve()
  .then(() => console.log("👦 [4] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <5-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

これを実行すると以下の結果を得ます。

```sh
❯ v8 asyncSpeed4.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👦 [4] <3-Sync> MICRO: then
👻 [5] <4-ASync> MICRO: then after async function
👦 [6] <5-Async> MICRO: then
```

## return Promise.resolve(42) の場合

さて、実はこれが一番やっかいなパターンです。結論から言うと、`return await Promise.resolve(42)` の場合はマイクロタスク１つで済んだのに、`return Promise.resolve(42)` の場合にはマイクロタスクが２つ発生します。

再び単純な async 関数を考えてみます。

```js:foo3
async function foo3() {
  return Promise.resolve(42);
}
```

↓ V8 エンジンによる内部変換コードとして想定されるコード。

```js:V8_Converting
resumable function foo3() {
  implicit_promise = createPromise();
  // suspend 時に呼び出し元に返される Promise インスタンス
  // <- await 式 なし ->
  resolvePromise(implicit_promise, Promise.resolve(42));
  // return する値 Promise.resolve(42) で implicit_promise を resolve する
  // この時に内部ではマイクロタスクが2つ生成される(resolve関数にPromiseを渡すから)
}
```

`resolvePromise()` の部分に注目してください。`implicit_promise` を `Promise.resolve(42)` という Promise インスタンスで resolve を試みています。`resolvePromise()` 自体は `resolve()` 関数とやっていることは同じです。

ECMAScript の仕様において `resolve()` 関数に渡された Promise の `then()` メソッドを呼ぶというマイクロタスクを発行するというように決まっています。

こちらについては、uhyo さんの記事で詳しく解説されています。

https://zenn.dev/uhyo/articles/return-await-promise

:::message alert
ここでいう、`resolve()` 関数とは Promise コンストラクタの引数となる executor 関数の引数として渡す `resolve` コールバック関数のことで、`Promise.resolve()` のことではないことに注意してください。

つまり、`resolve(Promise.resolve(42))` と `Promise.resolve(Promise.resolve(42))` は違います。`resolve()` 関数が特殊であり、`Promise.resolve()` の**引数が Promise インスタンスの場合は変換せずにそのまま返します**。
:::

この記事では V8 エンジンの内部変換で考えるので、上の記事にように通常の関数に戻して考えるのではなく、async 関数の内部変換後に起きることでそのまま考えてみます。

`resolvePromise()` という操作は以前に作成した Promise インスタンスに対して、コンストラクタ外部から resolve を起動して第二引数の値によって解決を試みるという操作ですが、基本的にはコンストラクタで `resolve()` するのと変わりません。

V8 内部変換で実際にどのようにしているかは分かりませんが、通常 Promise コンストラクタで resolve するところを外部から resolve を試みることができます。
https://stackoverflow.com/questions/26150232/resolve-javascript-promise-outside-the-promise-constructor-scope

つまり、`resolvePromise()` では以下のようなことを行っています。実際には Promise インスタンスは以前に作成したもので、外部から resolve していますが、分かりやすいようにあえてコンストラクタ関数で考えています。

```js:V8_convertingで考える
// Promise.resolve(42) で implciit_proise を resolve する
const implicit_promise = new Promise(resolve => {
  resolve(Promise.resolve(42));
});
```

`resolve()` 関数に渡された Promise の `then()` メソッドを呼ぶというマイクロタスクを発行するというように決まっているわけですから上のコードは次のようになります。分かりづらいですが、マイクロタスクを発行するために、`Promise.resolve().then()` が上から包んでいます。

```js
const implicit_promise = new Promise(resolve => {
  Promise.resolve().then(() => {
    Promise.resolve(42).then(resolve);
  });
});
```

ということで、`implicit_promise` が Settled になるまでに、上のコードではあきらかにマイクロタスクを２つ必要としています(`then()` メソッドのコールバック関数がマイクロタスクとして２回発行されます)。

またもや実際のコードで実行順番を考えてみましょう。

```js
// asyncSpeed3.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

(async function foo3() {
  return Promise.resolve(42);
  // 内部的にマイクロタスクが2つ必要となる
  // <2-Sync>
  // <4-Async>
})().then((data) => console.log("👻 [6] <6-Async> MICRO: then after async function", data));

Promise.resolve()
  .then(() => console.log("👦 [4] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [5] <5-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

`<>` で数字を囲んである部分がマイクロタスクとして発行されるのでマーキングしてあります。`<>` 内の数字がマイクロタスクとして発行される順番です。

```sh
❯ v8 asyncSpeed3.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👦 [4] <3-Sync> MICRO: then
👦 [5] <5-Async> MICRO: then
👻 [6] <6-Async> MICRO: then after async function 42
```

この場合、`return await Promise.resolve()` に比べて１つマイクロタスクが多く発生するので、このような結果となっています。`42` という値が Promise chain で値として繋げていることも分かりますね。

ここで注目すべきは、`return await Promise.resolve(42)` の場合と `return Promise.resolve(42)` の場合の違いです。

```js:foo4
async function foo4() {
  // return await Promise.resolve(42); // 同じ
  // マイクロタスクは内部で１つだけ発生
  const value = await Promise.resolve(42);
  return value; // 42 という値
}

async function foo3() {
  // マイクロタスクは内部で２つ発生
  return Promise.resolve(42);
}
```

V8 での async/await の内部変換では await 式ごとに次の箇所が必要となりました(以下は `foo4` の場合)。

```js:foo4におけるawait式の変換
  promise = promiseResolve(Promise.resolve()); // プロミスならそのまま返す
  // promise が Settled になったら処理再開のためのマイクロタスクを発行
  // すでに Settled なのですぐにマイクロタスクを発行
  performPromiseThen(
    promise,
    res => resume(«foo4», res),
    err => throw(«foo4», err));
  value = suspend(«foo4», implicit_promise);
```

そして、`promiseResolve()` という操作は次の内部で使用される関数でした。

```js
function promiseResolve(v) {
  // プロミスならそのまま返す
  if (v is Promise) return v;
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

お分かりだと思いますが、`promiseResolve()` では引数が Promise インスタンスならそのまま返します。いずれにせよ await 式では確実にマイクロタスクが１つ発生します。

`foo4` と `foo3` の V8 による変換コードを比較して考えると、いずれにせよ最初に async 関数の返り値となる `implicit_Promise` は作成します。await 式の有無によって、`foo4` の方にマイクロタスクが１つ発生して、`foo3` の方では await 式が無いのでマイクロタスクが発生していないため、この時点では `foo3` の方が優れているように見えます。

問題となるポイントは、最後の `resolvePromise()` の場所です。Promise の resolve に Promise インスタンスを使用しているかしていないかです。

仕様上、Promise インスタンスで resolve を試みるとマイクロタスクが２個発生します。

Promise インスタンス以外で resolve を試みるとマイクロタスクは発生せずに Promise インスタンスの状態が直ちに遷移します。ということで、途中までは `foo3` の方が余計なマイクロタスクを生成していないように思えましたが、最終的に async 関数の返り値となる `implicit_promise` を解決する際に余計なマイクロタスクが２つ生成されていまったので、`foo4` の方がマイクロタスクが少なく済みます。

### Promise インスタンスで resolve するということ

ある Promise インスタンスのコンストラクタで `resolve()` 関数や `Promise.resolve()` の引数として、Promise インスタンスを渡すと **Unwrapping** という現象がおき、引数として渡した Promise インスタンスの状態や履行値、拒否理由などを自身の状態と値として同化できます。

:::message
Unwrapping については『[resolve 関数と reject 関数の使い方](g-epasync-resolve-reject)』のチャプターで解説しました。
:::

ただし、この `Promise.resolve()` と `resolve()` の２つには注意すべき違いがあります。

上で見たようにまずは `resolve()` 関数に Promise インスタンスを渡した場合は注意が必要です。次のようなコードで Promise インスタンスで resolve を試みることでコードの実行順番が直感的に予測しずらくなります。

```js
// コードの参照元 : https://twitter.com/ferdaber/status/1098318363305099264?s=20&t=Qu2h-Aa0IhI5Lh-bxPkcOw
new Promise(resolve => {
  resolve('a');
}).then(console.log);
new Promise(resolve => {
  resolve(Promise.resolve('b'));
}).then(console.log)
Promise.resolve(Promise.resolve('c')).then(console.log);
```

このコードの実行の順番は次のようになります。

```sh
a
c
b
```

`Promise()` コンストラクタに引数として渡すコールバックである Executor 関数自体は同期的に実行されるので、直感的にはすべてのマイクロタスクが直ちにマイクロタスクキューへ送られて順番に処理されると考えて a → b → c の順番であると予測してしまいます。ですが、2 番目の `Promise()` コンストラクタ内部では `Promise.resolve('B')` という Promise インスタンスで resolve を試みているため、このような結果となります。

```js
new Promise(resolve => {
  resolve(Promise.resolve('b'));
}).then(console.log)
```

このコードは上で見たように ECMAScript の仕様において `resolve()` 関数に渡された Promise の `then()` メソッドを呼ぶというマイクロタスクを発行するというように決まっているわけですから、次のように変換できます。

```js
new Promise(resolve => {
  Promise.resolve().then(() => {
    Promise.resolve('b').then(resolve);
  });
}).then(console.log)
```

そういう訳で、この Promise インスタンスが履行状態となるまでにマイクロタスクが２個必要となり、出力順番は a → c → b となるわけです。

それでは、次の場合はどうなるでしょうか?

```js
// resolveWithPromise2.js
new Promise((resolve) => {
  resolve("Q");
}).then(value => console.log("[1]", value)); // <1-Sync>

Promise.resolve(Promise.resolve(Promise.resolve("I")))
  .then(value => console.log("[2]", value)) // <2-Sync>

new Promise((resolve) => {
  resolve(Promise.resolve("S")); // <3-Sync> <5-Async>
}).then(value => console.log("[5]", value)); // <7-Async>

Promise.resolve(Promise.resolve("U"))
  .then(value => console.log("[3]", value)) // <4-Sync>
  .then(() => console.log("[4]", "V")); // <6-Async>
```

`Promise.resolve(Promise.resolve(42))` の場合と `resolve(Promise.resolve(42))` の場合では話が違うので注意してください。

`Promise.resolve()` の引数に Promise インスタンスを渡すと**マイクロタスクは発生せずにそのまま引数の Promise インスタンスが返ってきます**。ということで、上のように `Promise.resolve()` 自体をいくらネストしようが内部でマイクロタスクは発生せずに直ちに履行状態となります。

従って、実行結果は次のようになります。

```sh
❯ v8 resolveWithPromise2.js
[1] Q
[2] I
[3] U
[4] V
[5] S
```

`Promise()` コンストラクタに渡す Executor 関数の引数である `resolve()` 関数が特殊ですので注意してください。

### どっちを使うべき?

スタックトレースの比較では `return await Promise.resolve(42)` (つまり `foo4`)の方が詳細に情報が表示されます。

これについては、azukiazusa さんの記事で解説されています。

https://zenn.dev/azukiazusa/articles/difference-between-return-and-return-await#%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9%E3%81%AE%E5%87%BA%E5%8A%9B

また、async stack trace については Masaki Hara さんの次の記事で詳細に解説されています。

https://zenn.dev/qnighy/articles/3a999fdecc3e81#%E9%9D%9E%E5%90%8C%E6%9C%9F%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9

具体的にどちらが優れているかというのは、それぞれ意見があると思いますが、マイクロタスクの発生が増加することで直感的に処理予測がしづらくなるので `return await` の方が個人的にはいいかなと思います。

## await async function の場合

基本形はすべてわかったので、少し応用を考えてみたいと思います。今度は await 式で async function (の返り値)を評価してみます。

```js:fooW
async function fooPrevious() {
  console.log("👍 MAINLINE: Sync process in async function!!");
  return await Promise.reslve(42);
}

async function fooNext() {
  let value = await fooPrevious();
  value++;
  return value;
}
```

await 式は基本的には Promise インスタンスを評価し履行値を取り出します。そして、async 関数はどんなときでも Promise インスタンスを返します。結局のところは `await Promise.resolve(42)` の場合や `await promise chain` の場合と同じです。

ということで、V8 エンジンによる内部変換として考えられるコードは以下のものとなります。

```js:V8_Converting
resumable function fooPrevious() {
  implicit_promise = createPromise();

  // 同期処理
  console.log("👍 MAINLINE: Sync process in async function!!");

  // < const value = await Promise.resolve(42); >
  promise = promiseResolve(Promise.resolve(42)); // Promise インスタンスをそのまま帰す
  performPromiseThen(promise,
    res => resume(«fooPrevious», res),
    err => throw(«fooPrevious», err)); // マイクロタスク１つ発生
  value = suspend(«fooPrevious», implicit_promise); // 履行値 42 が代入される
  // 処理再開ポイント
  resolvePromise(implicit_promise, value); // 42 で resolve
}
resumable function fooNext() {
  implicit_promise = createPromise();

  // < const value = await fooPrevious(); >
  promise = promiseResolve(fooPrevious()); // Promise インスタンスをそのまま返す
  performPromiseThen(promise,
    res => resume(«fooNext», res),
    err => throw(«fooNext», err)); // マイクロタスク１つ発生
  value = suspend(«fooNext», implicit_promise); // 履行値 42 が代入される
  // 処理再開ポイント
  value++;

  resolvePromise(implicit_promise, value); // 43 で resolve
}
function promiseResolve(v) {
  if (v is Promise) return v;
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

変換の原理自体は既に分かったので、上のように変換コードで一々考える必要もありません。単純にマイクロタスクがいくつ発生するかで考えます。

```js:fooW
async function fooPrevious() {
  console.log("👍 MAINLINE: Sync process in async function!!");
  return await Promise.reslve(42);
  // await 式ごとに確実にマイクロタスクが１つ発生するが、
  // 評価対象の Promise インスタンスにチェーンはないので１つですむ
  // microtask = 1
  // return Promise.resolve(42) ならマイクロタスク２つが発生する
}

async function fooNext() {
  // async 関数の返り値となる Promise インスタンスを評価して履行値を取り出す
  let value = await fooPrevious();
  // await 式ごとに確実にマイクロタスクが１つ発生する
  // micortask++
  value++;
  return value;
}
// 合計マイクロタスクが２つ発生する
```

ということで、マイクロタスクは２つ発生します。`fooNext()` をチェーンした場合には `then()` メソッドのコールバックがマイクロタスクキューへと発行されるのは async 関数の内部で生成されるマイクロタスク合計２個を実行した後になります。

また実際のコードで考えてみます。

```js
// asyncSpeedW.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [4] <1-Sync> MICRO: then"));

async function fooPrevious() {
  console.log("👍 [2] MAINLINE: Sync process in async function!!");
  return await Promise.resolve(42); // <2-Sync>
  // await 式ごとに確実にマイクロタスクが１つ発生するが、
  // 評価対象の Promise インスタンスにチェーンはないので１つですむ
  // microtask = 1
}

// 即時実行
(async function fooNext() {
  // async 関数の返り値となる Promise インスタンスを評価して履行値を取り出す
  let value = await fooPrevious();
  // await 式ごとに確実にマイクロタスクが１つ発生する
  // micortask++
  console.log("🦄 [6] <4-Async> MICRO: after await in async function");
  value++; 
  return value;
})().then((value) =>
  console.log("👻 [8] <5-Async> MICRO: then after async function:", value)
);

Promise.resolve()
  .then(() => console.log("👦 [5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [7] <6-Async> MICRO: then"));

console.log("🦖 [3] MAINLINE: End");
```

実行結果は次のようになります。

```sh
❯ v8 asyncSpeedW.js
🦖 [1] MAINLINE: Start
👍 [2] MAINLINE: Sync process in async function!!
🦖 [3] MAINLINE: End
👦 [4] <1-Sync> MICRO: then
👦 [5] <3-Sync> MICRO: then
🦄 [6] <4-Async> MICRO: after await in async function
👦 [7] <6-Async> MICRO: then
👻 [8] <5-Async> MICRO: then after async function: 43
```

## await Promise.reject(new Error("reason")) の場合

await 式で Rejected 状態の Promise インスタンスを評価すると、例外が throw されます。

try/catch で補足しない場合は async 関数内の処理がそこで終わり、以降の処理は実行されません。さらに、async 関数自体から返ってくる Promise インスタンスも Rejected 状態となるので、次のように chaining した場合は、`catch()` で例外が補足されます。

```js
(async function fooR() {
  await Promise.reject(new Error("reason"));
  console.log("これは実行されない");
})()
  .then(() => console.log("これは実行されないがマイクロタスクを発行"))
  .catch(err => console.log("これは実行される", err))
  .finally(() => console.log("これは実行される"));
```

`await Promise.reject(new Error("reason"));` は V8 によって次のように内部変換されます。

```js
  // Promise インスタンスならそのまま返す
  promise = promiseResolve(Promise.reject(new Error("reason")));
  // async 関数を再開またはスローするハンドラのアタッチ
  // Settled になったら throw を告げるマイクロタスクを発行
  performPromiseThen(
    promise,
    res => resume(«fooR», res),
    err => throw(«fooR», err)); // throw される
  suspend(«fooR», implicit_promise);
```

`peformPromiseThen()` で `promise` に対して Rejected 状態となったときのハンドラもアタッチしていたので、Rejected なら resume(再開) ではなく、throw を告げるマイクロタスクを発行します。

実際のコードでまた考えてみます。

```js
// promiseRejectionR.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

(async function fooR() {
  // await 式は確実にマイクロタスク１つ発生
  await Promise.reject(new Error("reason"));
  console.log("これは実行されない");
  // <2-Sync>
})()
  .then(() => console.log("👻 [(5)] <4-Async> 実行されないがマイクロタスクを発行 MICRO: [Fullfilled]"))
  .catch((err) => console.log("😭 [7] <6-Async> MICRO: [Rejected]", err.stack))
  .finally(() => console.log("👍 [9] <8-Async> MICRO: [Finally]"))

Promise.resolve()
  .then(() => console.log("👦 [4] <3-Sync> MICRO: then"))
  .then(() => console.log("🤪 [6] <5-Async> MICRO: then"))
  .then(() => console.log("🤪 [8] <7-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

Rejected 状態の Promise インスタンスにチェーンされている `then()` メソッドの**コールバック関数は実行されませんが、マイクロタスク自体は発行します**。ということで、実行順番は次のようになります。

:::message
『[catch メソッドと finally メソッド](h-epasync-catch-finally)』のチャプターで見たとおり、`catch()` メソッドや `then()` メソッドはコールバックが実行されないときでもマイクロタスクを発生させて、その連鎖的な処理によって Promise chain の実行となります。
:::

```sh
❯ v8 promiseRejectionR.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👦 [4] <3-Sync> MICRO: then
🤪 [6] <5-Async> MICRO: then
😭 [7] <6-Async> MICRO: [Rejected] Error: reason
    at fooR (promiseRejectionR.js:7:24)
    at promiseRejectionR.js:10:3
🤪 [8] <7-Async> MICRO: then
👍 [9] <8-Async> MICRO: [Finally]
```

async 関数では、try/catch/finally の構文が使用できますので、async 関数内で `await Promise.reject(new Error("reason"))` 以降の処理もできます。

```js
(async function fooRX() {
  try {
    await Promise.reject(new Error("reason"));
    console.log("これは実行されない");
    await Promise.resolve(42);
  } catch (err) {
    console.log("例外発生", err.stack);
  } finally {
    cosnole.log("最後に実行できる");
  }
})()
  .then(() => console.log("これは実行される"))
  .catch(err => console.log("これは実行されないがマイクロタスクを発行", err.stack))
  .finally(() => console.log("これは実行される"));
```

上のようなコードの場合、try/catch で例外は補足されており、async 関数自体から返ってくる Promise インスタンスは履行状態となるため、チェーンした `then()` メソッドのコールバックは実行されて、`catch()` メソッドのコールバックは実行されないことに注意してください。

V8 の内部変換で考えてみるとこんな感じでしょうか。

```js:V8_Converting
  try {
    // Promise インスタンスならそのまま返す
    promise = promiseResolve(Promise.reject(new Error("reason")));
    // async 関数を再開またはスローするハンドラのアタッチ
    // Settled になったら throw を告げるマイクロタスクを発行
    performPromiseThen(
      promise,
      res => resume(«fooRX», res),
      err => throw(«fooRX», err)); // throw される
    suspend(«fooRX», implicit_promise);
    // async 関数を一時停止して、呼び出し元に implicit_promise を返す

    // 実行されない
    console.log("これは実行されない");
    promise = promiseResolve(Promise.resolve(42));
    performPromiseThen(
      promise,
      res => resume(«fooRX», res),
      err => throw(«fooRX», err));
    suspend(«fooRX», implicit_promise);

  } catch (err) {
    // throw された例外を補足するところから再開
    console.log("例外発生", err.stack);
  } finally {
     cosnole.log("最後に実行できる");
  }
```

基本はすべて同じです。resume(再開) ではなく throw を告げるマイクロタスクが発行されることで、処理再開となるポイントでは throw された例外が補足されるところからとなります。

では実際のコードで実行順番を考えてみます。

```js
// promiseRejectionRX.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

(async function fooRX() {
  try {
    await Promise.reject(new Error("reason"));
    // マイクロタスク１つ発生
    console.log("これは実行されない");
    await Promise.resolve(42);
  } catch (err) {
    // <2-Sync>
    console.log("👹 [4] <2-Sync> MICRO: 例外発生", err.stack);
  } finally {
    console.log("👹 [5] <2-Sync> MICRO: 最後に実行");
  }
})()
  .then(() => console.log("👻 [6] <4-Async> MICRO: 実行される [Fullfilled]"))
  .catch((err) => console.log("😭 [(8)] <6-Async> MICRO: 実行されないがマイクロタスクを発行 [Rejected]", err.stack))
  .finally(() => console.log("👍 [10] <8-Async> MICRO: 最後に実行 [Finally]"));

Promise.resolve()
  .then(() => console.log("🤪 [6] <3-Sync> MICRO: then"))
  .then(() => console.log("🤪 [4] <5-Async> MICRO: then"))
  .then(() => console.log("🤪 [9] <7-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

今回は、async 関数内の try/catch によって例外補足されているため、async 関数から返ってくる Promise インスタンス自体は Fullfilled であり、チェーンされた `then()` メソッドのコールバックも実行されます。`catch()` メソッドのコールバックは実行されませんが、マイクロタスクは発行されるので注意してください。

実際に実行すると次の出力を得ます。

```sh
❯ v8 promiseRejectionRX.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👹 [4] <2-Sync> MICRO: 例外発生 Error: reason
    at fooRX (promiseRejectionRX.js:7:26)
    at promiseRejectionRX.js:17:3
👹 [5] <2-Sync> MICRO: 最後に実行
🤪 [6] <3-Sync> MICRO: then
👻 [6] <4-Async> MICRO: 実行される [Fullfilled]
🤪 [4] <5-Async> MICRO: then
🤪 [9] <7-Async> MICRO: then
👍 [10] <8-Async> MICRO: 最後に実行 [Finally]
```

# async/await の最適化

以上、async/await の挙動について、V8 エンジンの内部変換コードから解説を試みてみました。

最初に述べたよう ECMAScript の仕様自体が async/await の最適化(かつては V8 において `--harmony-await-optimization` というフラグで使用されていた機能)をマージしました。

https://github.com/tc39/ecma262/pull/1250

2017 年時点での async/await の仕様では、１つの await 式に２つの追加の Promise インスタンスと少なくとも３つのマイクロタスクが必要だったため非常に無駄が多かったですが、ECMAScript の仕様自体が最適化されたため、それを実装する**他の JavaScript エンジンでも同様に async/await の高速化をできるようになった**そうです。

:::message alert
V8 のブログ記事を見て node の version 8 から version 10 に更新すると async/await の挙動が変わるというのが最初紹介されていますが。これはバグが治ったというだけで、その時点での仕様では正しいのですが、この最適化がマージされたことによって再び version 8 と同じ挙動(実行順番)となりました。筆者もこれを読んだときは最初混乱しました。ブログ記事は構成が微妙で最後まで読まないとちゃんと理解できないようになっていますので注意してください。省略した `throwaway` Promise についてもそうです。参照する場合は最後までしっかり読まないと誤解するので気をつけてください。

参考:
[node.js - JS Promise's inconsistent execution order between nodejs versions - Stack Overflow](https://stackoverflow.com/questions/62032674/js-promises-inconsistent-execution-order-between-nodejs-versions)
:::

このように、async/await のオーバーヘッド(余計な Promise インスタンスとマイクロタスクの生成)を削減し最適化したことで async/await は高速化し、async stack trace による Debuggability(デバッグのしやすさ) の向上も伴って、async/await の機能は手書きの Promise に勝るようになったとのことです。

>**async/await outperforms hand-written promise code now**. The key takeaway here is that we significantly reduced the overhead of async functions — **not just in V8, but across all JavaScript engines, by patching the spec**.
>([Faster async functions and promises · V8](https://v8.dev/blog/fast-async)より引用、太字は筆者強調)

そして、開発者にも手書きの Promise よりも async/await の使用と V8 がネイティブに提供する Promise 実装を使用するように勧めています。

>And we also have some nice performance advice for JavaScript developers:
>- **favor async functions and await over hand-written promise code**, and
>- stick to the native promise implementation offered by the JavaScript engine to benefit from the shortcuts, i.e. avoiding two microticks for await.
>
>([Faster async functions and promises · V8](https://v8.dev/blog/fast-async)より引用、太字は筆者強調)

# async/await のまとめ

V8 の舞台裏を見ることで async/await の挙動が理解できたと思います。

もちろん async/await を理解できるようになるには、今までの知識として Promise とイベントループ、マイクロタスクの概念が必要不可欠です。ここまで学習してきたことによって async/await が理解できるようになったことを忘れないでください。

await 式によって async 関数内の実行フローが分割され制御が行ったり来たりしますが、それは Promise chain での連鎖的なマイクロタスク発行による逐次実行と同じです。async 関数では処理再開を告げるマイクロタスクとして `PromiseReactionJob` がコールスタックに積まれ、async 関数の関数実行コンテキストが再びプッシュされてコールスタックのトップになることで実行再開となります。

非同期処理の本質的な部分は**イベントループにおけるタスクとマイクロタスクの処理**です。

Promise chain も async/await も本質的には**イベントループにおけるマイクロタスクの連鎖的な処理**です。言うなれば **マイクロタスク連鎖(Microtask chain)** です。

V8 エンジンでは async/await の内部変換が行われており、これによって**最適化されたマイクロタスクの連鎖的処理**を実現しています。
