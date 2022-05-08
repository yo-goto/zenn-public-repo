---
title: "V8エンジンによる変換コードでasync/awaitの挙動を理解する"
emoji: "👻"
type: "tech"
topics: [V8, ECMAScript, JavaScript, 非同期処理, promise]
published: true
date: 2022-05-08
url: "https://zenn.dev/estra/articles/asyncawait-v8-converting"
aliases: [記事_V8エンジンによる変換コードでasync/awaitの挙動を理解する]
tags: [" #V8 #JavaScript/async  "]
---

# はじめに
JavaScript の「非同期処理」ってやっぱり難しくないですか？

自分も色々試行錯誤しましたが、「完全に理解した🤓」→「やっぱり何も分からん😭」っていう無限ループの中で徐々に理解を深めていくしかないようです。

非同期処理を理解するには非同期 API を提供する環境のことやイベントループ、マイクロタスクなどの仕組みについて理解する必要があります。

そして環境に埋め込まれた JavaScript Engine のことも理解する必要があります。

今回の記事では、JavaScript Engine の１つである V8 が内部で変換するコードから async/await の挙動を理解するための解説を試みたいと思います。

:::message
async/await は特に Promise チェーンが理解できていないと雰囲気で使うしかなくなってしまうので Promise の基礎と Promise チェーンを理解しておくのをおすすめします。
:::

今回の記事は Zenn の Book の方で公開している『イベントループとプロミスチェーンで学ぶ JavaScript の非同期処理』で収録する予定の前記事となります。非同期処理の理解に欠かすことの出来ない Promise チェーンとイベントループについて解説しているので興味がある方は是非確認してみてください。この記事を理解する上でも役立つと思います。

https://zenn.dev/estra/books/js-async-promise-chain-event-loop

# 参考文献
今回記事を書くにあたって参照した文献になります。

https://zenn.dev/uhyo/articles/return-await-promise

https://zenn.dev/azukiazusa/articles/difference-between-return-and-return-await

https://v8.dev/blog/fast-async#await-under-the-hood


# V8 エンジンとは

まずは V8 エンジンが何かを確認しておきましょう。

https://v8.dev

>V8 is Google’s open source high-performance JavaScript and WebAssembly engine, written in C++. It is used in Chrome and in Node.js, among others. It implements ECMAScript and WebAssembly, and runs on Windows 7 or later, macOS 10.12+, and Linux systems that use x64, IA-32, ARM, or MIPS processors. V8 can run standalone, or can be embedded into any C++ application.
>(上記公式ページより引用)

V8 は **Google が提供するオープンソースの JavaScript エンジン**であり WebAssembly エンジンでもあります。C++ で書かれており、主に Chrome ブラウザ(正確には Chrome のオープンソース部分である Chromium)で利用されています。

V8 エンジンの凄いところは、Chrome だけでなく、Node, Deno といったランタイム環境やクロスプラットフォームのデスクトップアプリケーションを開発するための Electron などで利用されている JavaScript エンジンであるということです。

JavaScript の実行環境において JavaScript エンジンである V8 エンジンが担当している役割は以下のように様々です。

- JavaScript コードをコンパイルして実行: コンパイラ
- 関数呼び出しの特定順序で実行できるようにする: コールスタック
- オブジェクトのメモリアロケーションの管理: メモリヒープ
- 使用されなくなったオブジェクトのメモリ解放: ガベージコレクション
- JavaScript におけるすべてのデータ型、演算子、オブジェクト、関数の提供

参考文献
https://hackernoon.com/javascript-v8-engine-explained-3f940148d4ef

V8 は DOM については一切感知しませんし、Web API も(ごく一部を覗いて)提供しませんので、それらは V8 を埋め込む環境によって実装されて提供される必要があります。**V8 エンジンはデフォルトのイベントループとタスクキュー/マイクタスクキューを保有しています**が、環境は独自のイベントループを実装し、複数のタスクキューを設けて、マイクロタスクのチェックポイントを定めることができます。

V8 エンジンは GoogleChromeLabs が提供する jsvu(**JavaScript engine Version Updater**)を使ってインストールでき、ソースからビルドすることなく利用できます。V8 エンジンはローカル環境においてスタンドアロンで実行できるため、ECMAScript の実装について簡単にローカルでテストできます。

https://github.com/GoogleChromeLabs/jsvu

# V8 エンジンによる内部変換コード

さて、V8 エンジンについての予備知識を頭に入れたところで本題に入りましよう。今回は開発チームの Maya Lekova 氏と Benedikt Meurer 氏によるプレゼン動画と、それに基づく V8 エンジン公式サイトのブログを参考に async/await の V8 エンジンでの内部変換コードを見ていきます。ブログだけだと分かりづらい部分があるので、動画も一緒に視聴することをおすすめします。

https://v8.dev/blog/fast-async#await-under-the-hood

@[youtube](DFP5DKDQfOc)

:::message
ブログや動画では、以前の ECMAScript の仕様では async/await のオーバーヘッドがあったため、V8 エンジンでそれを改善したた上で、ECMAScript の仕様自体にその最適化をマージし、V8 エンジンだけでなく、あらゆるエンジンで最適化ができるようにしたことが述べられています。

今回の記事ではそれらを全部すっ飛ばして結論としての変換コードから見ていきますので、注意してください。細かい部分については動画とブログを参考にしてください。
:::

では結論として、V8 エンジンでは次のような async/await を内部的に変換しています。

```js:シンプルな非同期関数
async function foo(v) {
  const w = await v;
  return w;
}
```

変換後は以下のようになります。

```js:V8エンジンによる変換コード
// 途中で一時停止できる関数として resumable (再開可能) のマーキング
resumable function foo(v) {
  implicit_promise = createPromise(); 
  // (0) 非同期関数の返り値となる Promise インスタンスを作成
  
  // (1) v が Promise インスタンスでないならラッピングする
  promise = promiseResolve(v);
  // (2) 非同期関数 foo を再開するハンドラのアタッチ
  performPromiseThen(
    promise,
    res => resume(«foo», res),
    err => throw(«foo», err));

  // (3) 非同期関数 foo を一時停止して implicit_promise を呼び出し元へと返す
  w = suspend(«foo», implicit_promise); 
  // (4) w = のところから非同期関数の処理再開となる

  // (5) 非同期関数で return していた値である w で最終的に implict_promise を解決する
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

基本的なステップはコメントアウトした通りになります。

- (0) 非同期関数自体の返り値となる Promise インスタンスとして `implicit_promise` を作成する
- (1) await 式の評価対象について Promise インスタンスでないならラッピングして `promise` に代入します
- (2) `promise` が Settled になったときのハンドラを同期的にアタッチします
- (3) 非同期関数の処理を `suspend()` で一時停止して、Promsie インスタンスである `implicit_promise` を呼び出し元へと返却します
- (4) `promise` が Settled となり次第、非同期関数の処理を再開し、await 式の評価結果を `w` に代入するところから処理再開となります
- (5) 最終的に非同期関数内部で `return` していた値で `implicit_promise` を resolve することで呼び出し元に返されていた Promise インスタンスが Settled となります

変換後のコードで `return` が内のは、`suspend()` の時点で呼び出し元である Caller へと Promise インスタンスとして `implicit_promise` を返してるからです。非同期関数はどんなときでも、Promise インスタンスを返します。非同期関数の処理が一時停止して、呼び出し元に制御が戻った時にすでに返り値として Promise インスタンスを用意していなければいけません。ただし、その時に返り値の Promise インスタンスが履行されている必要はなく、Pending 状態のままでいいのです。

再び、非同期関数の処理が再開し、最終的に非同期関数で `return w` としていた値 `w` で `implicit_promise` が解決されることで、呼び出し元に返ってきていた Promise インスタンスが Setteld になり、その値 `w` を Promise チェーンなどで利用できるようになります。

`implicit_promise = createPromise()` は後から解決される Promise インスタンス `implicit_promise` を作成し、`reoslvePromise(implicit_promise, w)` では作成したその Promise インスタンスを後から `w` で解決しています。細かい実装は分からないので、ここではそういうものだと考えてください。

ということで、上記コードの説明としてもう少しコメントアウトしたいと思います。`v` は `v = Promise.resolve(42)` というように値 `42` で既に履行されている Promise インスタンスとして想定します。

```js:V8エンジンによる変換コード
// 途中で一時停止できる関数として resumable (再開可能) のマーキング
// 非同期関数からは、susupend のところまで行った時点で処理を中断して Pending 状態の Promise インスタンス(implicit_promise)が呼び出し元に返される
// 通常の return は意味がない(generator の yeild ぽい)
resumable function foo(v) {
  implicit_promise = createPromise(); 
  // 非同期関数の返り値となる promise インスタンスを作成
  // 非同期処理を一時停止(susupend)したときもこれが呼び出し元に返ってきている
  
  // １つの await 式 (必ず１つはマイクロタスクが生成される)
  // 1. v を promise でラップする
  promise = promiseResolve(v); // v がプロミスでないならラッピング
  // 2. foo を再開するハンドラのアタッチ
      // Promise.prototype.then() が裏側で行っていることと同じ
      // promise が Settled になったらマイクロタスクを発行
      // マイクロタスクは PromiseReactionJob で非同期関数の処理再開を告げる
  performPromiseThen(
    promise,
    res => resume(«foo», res),
    err => throw(«foo», err));
    // アタッチしているだけでとりあえず次に進む
  // 3. foo (非同期関数)を一時停止して implicit_promise を caller へと返す
  w = suspend(«foo», implicit_promise); 
  // ここまでが１つの awiat で、foo のコンテキストを一旦ポップする
  // w には await 式の評価結果の値が yeild され代入される(yields 42 from the await)
  // w = のところに値が入り実行再開する(w には promise の履行値 42 が入る)

  resolvePromise(implicit_promise, w); // return する値 w (= 42)で resolve する
  // caller へ返していた Promise インスタンスが Setteld になる
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
  // (2) 非同期関数 foo を再開するハンドラのアタッチ
  performPromiseThen(
    promise,
    res => resume(«foo», res),
    err => throw(«foo», err));
  // (3) 非同期関数 foo を一時停止して implicit_promise を呼び出し元へと返す
  w = suspend(«foo», implicit_promise); 
```

プレゼンの前資料である次のドキュメントから借用したコードで考えると次のようにもできます。

[Zero-cost async stack traces - Google ドキュメント](https://docs.google.com/document/d/13Sy_kBIJGP0XT34V1CV3nkWya4TwYx9L3Yv45LdGB6Q/)

```js:別の書き方
const .promise = @promiseResolve(x);
@performPromiseThen(.promise,
  res => @resume(.generator_object, res),
  err => @throw(.generator_object, err));
@yield(.generator_object, .outer_promise);
```

## await 式は確実にマイクロタスクを１つ発行する

`performPromiseThen()` の箇所に注目してほしいのですが、これは `Promsise.prototype.then()` が舞台裏でやっていることと本質的に同じとなります。

`peformPromiseThen()` に渡す引数である `promise` が Settled になることで、`then()` メソッドのコールバックのようにマイクロタスクが発行されます。

このマイクロタスクがマイクロタスクキューから選択されて、その実行コンテキストがコールスタックへと積まれることで、非同期関数の処理再開をできるようになっています。await 式ごとにこの `performPromiseThen()` の実行が必要となります。`then()` メソッドのようにマイクロタスクが発行されるので、Promise チェーンで考えれば理解できるはずです。

:::message
Generator で考えやすい人はそちらで考えるいいと思います。僕は Generator には詳しくないので、頭の中にある Promise チェーンとイベントループをベースモデルにして理解しています。
:::

## await 式が２個ある場合

それでは、今までの内容を踏まえて、今度は await 式が２個ある場合を考えてみます。

```js:await式が２個ある非同期関数
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
  // 非同期関数の返り値となる promise インスタンスを作成
  
  // <<await v>>
  promise = promiseResolve(v); 
  performPromiseThen(promise,
    res => resume(«foo2», res),
    err => throw(«foo2», err));
  suspend(«foo2», implicit_promise); 
  // 中断かつ処理再開のポイント
  
  console.log("Microtask1");

  // <<await x>>
  promise = promiseResolve(x);
  performPromiseThen(promise,
    res => resume(«foo2», res),
    err => throw(«foo2», err));
  suspend(«foo2», implicit_promise);
  // 中断かつ処理再開のポイント

  console.log("Microtask2");

  resolvePromise(implicit_promise, 42); 
  // 最終的に return する値 42 で resolve する
}
```

# 色々なパターン

さて、基本的な変換が分かったので、もう少し深く潜ってみたいと思います。この変換を基本系に色々な async/await を考えてみます。

こちらの uhyo 氏の記事で紹介されているような色々なパターンと、その速度(マイクロタスクをいくつ発行するか)についても考えてみましょう。

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

aysnc/await ではないので V8 エンジンでの変換はありません。

`Promise.resolve().then()` によって同期的に直ちにマイクロタスクキューへコールバックがマイクロタスクとして発行されます。また、即時実行関数内でも履行状態で作成される Promise インスタンスが返されるため、次の `then()` メソッドのコールバックが同期的に直ちにマイクロタスクキューへマイクロタスクとして発行されます。ということで、関数内部で余計なマイクロタスクは発生しません。

これを V8 エンジンで実行すると次のように予測が簡単な出力を得ます。Chrome、Node、Deno でやっても全部同じです。

```sh
❯ v8 asyncSpeedY.js
🦖 [1] MAINLINE: Start
🦖 [2] MAINLINE: End
👦 [3] <1-Sync> MICRO: then
👦 [4] <2-Sync> MICRO: then after function
👦 [5] <3-Sync> MICRO: then
👦 [6] <4-Async> MICRO: then
```

:::message
jsvu でインストールした `v8` コマンドに引数としてスクリプト名を渡すことでローカル実行できます。
:::

## await も return も無い場合

それでは次に、`await` 式も `return` も何も無い非同期関数を考えてみましょう。次のようなシンプルに何もしない非同期関数の変換はどうなるでしょうか?

```js:何もしない非同期関数
async function empty() {}
```

V8 エンジンは次のように内部的に変換すると想定されます。

```js:V8_Converting
resumable function empty() {
  implicit_promise = createPromise(); 

  // <- await 式 ->
  // <- 再開処理 ->

  resolvePromise(implicit_promise, undefined); 
  // return する値はないので undefined で resolve する
  // 返される Promise インスタンスは直ちに履行状態となる(マイクロタスクは発生しない)
}
```

`await` がないので、各 await 式に必要ないつものコードはありません。そして、`return` している値も無いので、`return` する値は `undefined` となり、非同期関数から返される Promise インスタンスは `undefined` で解決されます。

そして `peformPromiseThen()` が無いのでマイクロタスクは１つも発行されず、非同期関数から返ってくる Promise インスタンスはただちに履行状態となります。

:::message
非同期関数(Async funciton)はどんなときでも必ず Promise インスタンスを返します。
:::

それでは、次のコードの実行順番を予測します。

```js
// asyncSpeed1.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

// 非同期関数を即時実行
(async function empty() {})().then(() => console.log("👦 [4] <2-Sync> MICRO: then after async function"));

Promise.resolve()
  .then(() => console.log("👦 [5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <4-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

今回も即時実行で関数を実行します。非同期関数からは Promise インスタンスが必ず返ってくるので、`then()` メソッドで Promise チェーンを構築できます。

マイクロタスクについて考えてみましょう。
まずは、`Promise.resolve().then()` で同期的にマイクロタスクが発行されて、その次の肝心の async function の即時実行でも、上の変換で見たように関数から返えされる Promise インスタンス自体は直ちに履行状態となるので、`then()` メソッドのコールバックがマイクタスクとしてマイクタスクキューに送られます。次の `Promise.resolve.then()` メソッドのコールバックも同期的にマイクロタスクを発行してキューへ送られます。

スクリプト評価による同期処理がすべて終わり、コールスタックからグローバルコンテキストがポップして破棄されることで、コールスタックが空になるので、マイクロタスクのチェックポイントとなります。マイクロタスクキューの先頭にあるものから順番にすべて処理されていきます。

３番目にマイクロタスクキューへ送られたコールバック `() => console.log("👦 [5] <3-Sync> MICRO: then")` が実行された時点で、元々の `Promise.reoslve().then()` で返ってくる Promise インスタンスが履行状態となるので、`Promise.resolve().then().then()` のコールバックがマイクロタスクキューに送られて直ちにコールスタックへと積まれて実行されます。

:::message
マイクロタスクとコールスタックについて説明すると長くなるので、詳細については『イベントループとプロミスチェーンで学ぶ JavaScript の非同期処理』を参照してください。
:::

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

## await null だけの場合

次は、非同期関数内で `await null` だけをする場合を考えてみます。

```js
async function foo4() {
  await null;
}
```

↓ V8 エンジンによる内部変換として想定されるコード。

```js:V8_Converting
resumable function foo4() {
  implicit_promise = createPromise(); 

  // <- await 式
    promise = promiseResolve(null); // プロミスでないのでラップする
    // promise が Settled になったら処理再開のためのマイクロタスクを発行
    // すでに Setteld となるので直ちにマイクロタスクを発行
    performPromiseThen(
      promise,
      res => resume(«foo4», res),
      err => throw(«foo4», err));
    // 非同期関数を一時停止して、呼び出し元に implicit_promise を返す
    suspend(«foo4», implicit_promise); // awiat 式 ->
  // 再開処理だが特にやることはない ->

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

await 式というのは通常は Promise インスタンスを評価し、Promsie インスタンスの評価結果としてその履行値を返すという使いかたをしますが、Promise インスタンスでないものも評価できます。

その場合は、`promise = promiseResolve(null)` であるように、Promise インスタンスでない場合として新しい Promise でラッピングされます(await 式で評価する値自体で解決する Promise インスタンス)。

いずれにせよ `performPromiseThen()` を行うため、作成された Promise インスタンスが Setteld になるまで待ち、Setteld になった時点で非同期関数の処理再開を告げるマイクロタスクを発行します。この場合は Promsie インスタンスがすぐに履行状態になるので、同期的にマイクロタスクを直ちに発行します。

ということで、非同期関数から返ってくる Promise インスタンスにチェーンする `then()` メソッドのコールバックの実行はマイクロタスク１回が実行されるまで待つ必要があります。

```js
// asyncSpeed8.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

// async function から返る promsie はマイクロタスク一個の実行で履行状態で then でマイクロタスク発行
(async function foo4() {
  await null;
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

といことで、いままでの場合と違い非同期関数内部でマイクロタスクが一個だけ発行されるので、非同期関数から返される Promise インスタンスが履行状態になるにはそのマイクロタスクが実行される必要があります。従って、チェーンしている `then()` メソッドのコールバックがマイクロタスクとして発行されるタイミングが今までのようにすぐにではなく、ずれることになります。

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

## await Promise.resolve() の場合

今度は、すでに履行状態の Promise インスタンスを await してみましょう。

```js
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
    // すでに Setteld となるので直ちにマイクロタスクを発行
    performPromiseThen(
      promise,
      res => resume(«fooZ», res),
      err => throw(«fooZ», err));
    // 非同期関数を一時停止して、呼び出し元に implicit_promise を返す
    suspend(«fooZ», implicit_promise); // awiat 式 ->
  // 再開処理だが特にやることはない ->

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

この場合は実は `await null` と同じになりなり、内部的にマイクロタスクを１つ発行することになります。そういう訳で、`await` で**何を評価しようが少なくともマイクロタスク１つが発行される**ことになります。各 await 式において最低でも１つマイクロタスクが発行されます。

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

## await promise chian の場合

次は、既に履行状態の Promise インスタンスではなく、履行するまで１つマイクロタスクが必要な Promise チェーンを await してみましょう。

```js
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
  // awiat 式 ->
  // ここまでで二回はマイクロタスクを使用している
  // 再開処理だが特にやることはない ->

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

何を await しようがマイクロタスクは確実に１個発行されますが、今回のケースでは、`promise` が Setteld になるまでに１つマイクロタスクが必要となります。それが実行されてから、`promise` が Settled になりマイクロタスクが再び発行されるので、マイクロタスクは合計２つ必要となります(今までの場合よりも一個多い)。

実際のコードを考えてみましょう。ここまで来ると非常に予測が難しくなります。

```js
// asyncSpeed9.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

(async function foo9() {
  await Promise.resolve("😭").then((value) =>
    console.log("🦄 [4] <2-Sync> MICRO: then inside", value)
  );
  // マイクロタスク２回分必要
  // <4-Async>
})().then(() => console.log("👻 [7] <6-Async> MICRO: then after async function"));

Promise.resolve()
  .then(() => console.log("👦 [5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <5-Async> MICRO: then"))
  .then(() => console.log("👦 [8] <7-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

`<>` で囲んである数字はマイクロタスクキューに追加される順番で、`Sync` は同期的にマイクロタスクキューに送られて、`Async` はその前のマイクロタスク実行後に非同期的にマイクロタスクキューに送られる場合となっています。

非同期関数から返される Promsie インスタンスが履行状態になるまでに内部発生するマイクロタスク２個分が実行される必要があるため、実行結果は次のようになります。

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

## return 42 の場合

今度は、何も await せずに単なる数値 `42` を返す非同期関数を考えてみます。

:::message
42 という数字はマジックナンバーで、「任意の数値」を意味するケースがよくあります。
:::

```js
async function foo0() {
  return 42;
}
```

↓ V8 エンジンで内部変換されるコードは次のようになると想定されます。

```js:V8_Converting
resumable function foo0() {
  implicit_promise = createPromise(); 
  // <- await 式 なし ->
  resolvePromise(implicit_promise, 42); 
  // 最終的に return する値 42 で resolve する
  // 内部ではマイクロタスクは１つも生成されない
}
```

前のケースと同じで、マイクロタスクは１つも発生しません。ということは、この非同期関数から返される Promise インスタンスは同期的に直ちに履行状態となります。

```js
// asyncSpeed0.js
console.log("🦖 [1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [3] <1-Sync> MICRO: then"));

// async function から返る promsie は直ちに履行状態で then でマイクロタスク発行
(async function foo0() { return 42; })().then(() =>
  console.log("👻 [4] <2-Sync> MICRO: then after async function")
);

Promise.resolve()
  .then(() => console.log("👦 [5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [6] <4-Async> MICRO: then"));

console.log("🦖 [2] MAINLINE: End");
```

このコードの実行結果は次のようになります。前のケースと同じです。

```js
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

次のようなシンプルな非同期関数を再び考えてみます。

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
    // 非同期関数を一時停止して、呼び出し元に implicit_promise を返す
    value = suspend(«foo4», implicit_promise); // awiat 式 ->
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

再び単純な非同期関数を考えてみます。

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
  // return する値 Promise.resolve で implicit_promise を resolve する
  // この時に内部ではマイクロタスクが2つ生成される(resolve関数にPromiseを渡したから)
}
```

`resolvePromise()` の部分に注目してください。`implicit_promise` を `Promise.resolve(42)` という Promise インスタンスで resolve を試みています。`resolvePromise()` 自体は `resolve()` 関数とやっていることは同じです。

ECMA の仕様において `resolve()` 関数に渡された Promise の `then()` メソッドを呼ぶというマイクロタスクを発行するというように決まっています。

こちらについては、uhyo さんの記事で詳しく解説されています。

https://zenn.dev/uhyo/articles/return-await-promise

:::message alert
ここでいう、`resolve()` 関数とは Promise コンストラクタの引数となる executor 関数の引数として渡す `resolve` コールバック関数のことで、`Promise.resolve()` のことではないことに注意してください。

つまり、`resolve(Promise.resolve(42))` と `Promise.resolve(Promise.resolve(42))` は違います。`resolve()` 関数が特殊であり、`Promise.resolve()` の**引数が Promsie インスタンスの場合は変換せずにそのまま返します**。
:::

この記事では、V8 エンジンの内部変換で考えるので、uhyo さんの記事にように通常の関数に戻して考えるのでは、変換後に起きることで考えてみます。

`resolvePromise()` という操作は以前に作成した Promise インスタンスに対して、コンストラクタ外部から resolve を起動して第二引数の値によって解決を試みるという操作ですが、基本的にはコンストラクタで `resolve()` するのと変わりません。

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

ということで、`implicit_promise` が Settled になるまでに、上のコードではあきらかにマイクタスクを２つ必要としています(`then()` メソッドのコールバック関数がマイクロタスクとして２回発行される)。

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

この場合、`return await Promise.resolve()` に比べて１つマイクロタスクが多く発生するので、k のような結果となっています。

`42` という値が Promise チェーンで値として繋げていることも分かりますね。

# おわり

書くのに結構時間がかかって疲れたのでここらへんで一旦終わりたいと思います😅。もう少し複雑なケースもかけたら追記します。

