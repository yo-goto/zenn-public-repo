---
title: "Promise chain から async 関数へ"
cssclass: zenn
date: 2022-05-14
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『Promise chain から async 関数へ』
---

# このチャプターについて

Prosmise chain について詳しくなった所でようやく async/await に入ることができます。このチャプターでは、async/await を学習する上での注意点や意識すべき点などについて解説します。

:::message
イベントループと絡めた具体的な挙動については次のチャプターを確認してください。このチャプターで理解しにくい点がある場合にも次のチャプターの内容を見てから戻ってくるのもいいかもしれません。
:::

# async/await とは

async/await とは文字通り async 関数と await 式という２つを組み合わせることで Promise chain をより分かりやすく書くことができるように ES2017 で導入された ECMAScript の構文の通称です。

まずは読み方ですが、asynchronous という英単語の発音記号は `/eɪsɪ́ŋkr(ə)nəs/` なので、`async` の読み方は"エイシンク"が正しいということになりますが、ゴリラさん([@gorilla0513](https://twitter.com/gorilla0513))の Twitter アンケートによると意外にアシンク派が多いようです。

https://twitter.com/gorilla0513/status/1589809249848537088?s=20&t=9PPhBYSWCGtEJdOzJXL6-A

await の発音記号は `/əwéɪt/` なので、「エイシンク/アウェイト」あるいは「アシンク/アウェイト」となりますが、どちらも使われているようなので周りで使われている呼び方に合わせれば良いでしょう。

さて、注意点として、非同期処理の主役は async/await ではなく、**あくまで Promise が主役である**ことを忘れないでください。async/await を使ってできることは Promise による非同期処理の利便性を高める(具体的には制御フローが見やすくなったり、let 宣言や for ループ、try/catch などが使えるようになる)ことです。async/await 自体が Promise のシステムに基づいた上で Promise インスタンスそのものを扱っているので、**Promise について理解できていないと async/await のほとんどのことが理解できていないことになってしまいます**。

そういう訳で、初学者にとって「**async/await によって Promise を意識することなく非同期処理ができるようになった**」という文言はトラップとなる可能性が実は高いです。むしろ、Promise を扱っていることをしっかり意識しておかないと後々混乱しやすい構文であると個人的には感じています。

# async 関数

async/await には async 関数と await 式という２人の登場人物がいます。まずは async 関数について見ていきましょう。

async 関数は「非同期関数」と呼ばれることが多々あります。MDN 日本語版でもその呼び方がなされていますね。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/async_function

ただし、『[非同期 API と環境](f-epasync-asynchronous-apis#非同期という概念)』のチャプターで解説したとおり、`setTimeout()` などの非同期 API も非同期関数と呼ばれることがあります。両者は同じ呼称で同一視すべきではないのでこの本では「非同期関数」と言ったら async 関数のことを指すようにしていますが、実は**非同期関数という呼称自体も学習において誤解の原因となることがあります**(それについては後述します)。ちなみに、MDN 英語版での非同期関数のページでは "asynchronous function" ではなく "async function" という用語がタイトルになっています(この２つは若干ニュアンスが違うので受ける印象も異なってきます)。

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function

それ故に非同期関数という呼称はなるべく避けて「**async 関数**」という言葉を多用してきます。

それでは、async 関数が MDN でどのように説明されているか見てみましょう。

> An async function is a function declared with the async keyword, and the await keyword is permitted within it. The async and await keywords enable asynchronous, promise-based behavior to be written in a cleaner style, avoiding the need to explicitly configure promise chains.
> ([async function - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) より引用)

「async 関数は `async` キーワードを使って宣言された関数であり、関数内部での `await` キーワードの使用が認められている。`async` と `await` のキーワードを使用することで、よりクリーンなスタイルで、明示的に Promise chain を構成することなく Promise-based な非同期の振る舞いを書けるようにする」という旨が記載されています。

そして、async 関数は**どんな時も必ず Promise インスタンスを返す関数**であり、return される値が明示的に Promise ではない場合も Promise インスタンスでラップされて返却されることも言及されています。

> **Async functions always return a promise**. If the return value of an async function is not explicitly a promise, **it will be implicitly wrapped in a promise**.
> ([async function - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) より引用、太字は筆者強調)

それでは具体的なコードを見ていきます。async 関数の定義には `async` キーワードを頭に付けて関数宣言を行います。そして、Promise インスタンスを返すわけですから、Promise chain が可能です。

```js:通常の関数定義
// returnStringInAsync.js
async function alwaysReturnPromise() {
  return "👾 Promise でラップされる文字列";
}

// Promise インスタンスを返すので Promise chain が可能
alwaysReturnPromise()
  .then((value) => console.log("🍓 履行値:", value))
  .catch((reason) => console.error("🥦 拒否理由:", reason))
  .finally(() => console.log("👍 チェーン最後に実行"));
```

ちゃんと Promise chain が出来ていますね。

```sh:実行結果
# V8 エンジンで実行してみる
❯ v8 returnStringInAsync.js
🍓 履行値: 👾 Promise でラップされる文字列
👍 チェーン最後に実行
```

アロー関数で定義する場合は次のようになります。

```js:アロー関数での定義
// returnNothingInAsync.js
const alwaysReturnPromise = async () => {
  // 何もしない
};

// 必ず Promise インスタンスが返ってくるので Promise chain が可能
alwaysReturnPromise()
  .then((value) => console.log("🍓 履行値:", value)) // undefined
  .catch((reason) => console.error("🥦 拒否理由:", reason))
  .finally(() => console.log("👍 チェーン最後に実行"));
```

async 関数内部で何も `return` しなくても、必ず Promise インスタンスが返ってきます。この場合、返ってくる Promise インスタンスは履行値が `undefined` の履行状態になります。

```sh:実行結果
❯ v8 returnNothingInAsync.js
🍓 履行値: undefined
👍 チェーン最後に実行
```

## async 関数の即時実行

また、async 関数はその特性上、即時実行(IIFE)の形式で利用することも多いので、次の書き方を覚えておくとよいです。

```js
// async 版の無名関数の即時実行
(async function() {
  // 処理内容
})();

// async 版のアロー関数の即時実行
(async () => {
  // 処理内容
})();
```

即時実行関数はスコープ汚染しないなどの特徴はありますが、普通に関数を定義してすぐ呼び出すのと大した差はないです。

```js
// アロー関数での async 関数の定義
const Fn = async () => {
  // ...処理内容
};
Fn(); // すぐ呼び出し

// 上とやっていることと大して変わりない
(async () => {
  // ...処理内容
})();
```

:::message
このチャプターでは、分かりやすくするために即時実行時の際に無名関数でよいものもあえて名前をつけている場合があります。
:::

## async 関数内部は同期処理

『[コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async#同期か非同期か)』のチャプターで見たように **Promise コンストラクタ内部は同期処理**であり、`then()` で chain してはじめて非同期になりました。

```js
console.log("🦖 [1] sync");

const p = new Promise((resolve) => {
  // コンストラクタ関数に渡すコールバック(Executor 関数)内部は同期処理
  console.log("🦄 [2] sync");
  resolve();
});
// then で chain してはじめて非同期処理となる
p.then(() => console.log("👻 [4] async"));

console.log("🦖 [3] sync");

/* 出力結果
🦖 [1] sync
🦄 [2] sync
🦖 [3] sync
👻 [4] async
*/
```

async 関数でもまったく同じです。この事実も混乱の原因となっていますが、実は **async 関数の内部は同期処理**となります。以下のコードでは即時実行を使って書いていますが、通常の async 関数でもまったく同じで関数内部は基本的には同期処理です。内部的に await 式を使わない async 関数はほとんど何の役にも立ちませんが、説明の都合上あえてやっています。

```js
console.log("🦖 [1] sync");

// async 版のアロー関数での即時実行
(async () => {
  // async 関数内部は同期処理
  console.log("🦄 [2] sync");
})();

console.log("🦖 [3] sync");

/* 出力結果
🦖 [1] sync
🦄 [2] sync
🦖 [3] sync
*/
```

MDN の説明でもあったように、`async` と `await` の２つのキーワードがあってはじめて非同期の振る舞いを記述できます。

> **The async and await keywords enable asynchronous, promise-based behavior** to be written in a cleaner style, avoiding the need to explicitly configure promise chains.
> ([async function - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) より引用、太字は筆者強調)

『[非同期 API と環境](f-epasync-asynchronous-apis#非同期という概念)』のチャプターでも、async 関数が「非同期」の性質を発揮するのは async 関数内部に await 式がある時のみと言っていましたね。実際、**最初の await 式の前は同期処理であり**、最初の await 式の時点からはじめて非同期となります。async 関数内部は await 式によって分割されており、最初の await 式までは同期的に処理されることが MDN でも明言されています。つまり、await 式が存在していなければ同期的に完了することになります。

> The body of an async function can be thought of **as being split by zero or more await expressions**. **Top-level code, up to and including the first await expression (if there is one), is run synchronously. In this way, an async function without an await expression will run synchronously**. If there is an await expression inside the function body, however, the async function will always complete asynchronously.
> ([async function - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) より引用、太字は筆者強調)

:::message
「非同期関数」という呼称で考えてしまうと「必ず非同期になる」というような誤解を招く可能性があるので「async 関数」という呼び方にした方が良いわけです。微妙な違いですが、async 関数なら「`async` キーワードが付いた関数」程度に捉えることができます。
:::

```js
console.log("🦖 [1] sync");

(async () => {
  console.log("🦄 [2] sync");
  // ここまでは同期処理!!

  // await 式の時点から非同期となる
  // Promise.resolve(1).then(() => console.log("👻 [4] async")) とほぼ同じ
  await Promise.resolve(1);
  console.log("👻 [4] async");
})();
// await 式があれば async 関数は完了せずにその外の処理が行われるので「非同期」の現象が起きている
console.log("🦖 [3] sync");

/* 出力結果
🦖 [1] sync
🦄 [2] sync
🦖 [3] sync
👻 [4] async
*/
```

async/await は Promise chain で書き直せますし、その逆もできます。Promise chain なら `then()` のコールバックを書いてはじめて非同期になるように async 関数なら await 式があってはじめて非同期になります。そして、**await 式のたびに `then()` をしているのと同じことになります**。

```js
console.log("🦖 [1] sync");

(async () => {
  console.log("🦄 [2] sync");
  // ここまでは同期処理!!

  // await のたびに then しているのと同じ
  await Promise.resolve(1);
  console.log("👻 [4] async");
  await Promise.resolve(2);
  console.log("👻 [5] async");
})();

console.log("🦖 [3] sync");

/* 出力結果
🦖 [1] sync
🦄 [2] sync
🦖 [3] sync
👻 [4] async
👻 [5] async
*/
```

上のコードの一部を実際に Promise chain で書き直すと次のようになります(※ 完全に元のコードと同等に書き直したわけではないので注意してください)。

```js
console.log("🦖 [1] sync");

(async () => {
  console.log("🦄 [2] sync");
  // ここまでは同期処理!!

  Promise.resolve(1)
    .then(() => {
      console.log("👻 [4] async");
      return Promise.resolve(2);
    })
    .then(() => {
      console.log("👻 [5] async");
    });
})();

console.log("🦖 [3] sync");

/* 出力結果
🦖 [1] sync
🦄 [2] sync
🦖 [3] sync
👻 [4] async
👻 [5] async
*/
```

このように、async/await と Promise chain は共存できます。そして、Promise が理解できた上で async/await が分かると、共存させたり、相互に書き直すことができるようになります。

:::message alert
『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)』のチャプターなどを見てもらえればわかりますが、実は上の２つのコードは同等という訳ではありません。イメージしやすいように書いているだけなので注意してください。

Promise chain で一部書き直したコードの方は async 関数内部で Promise 処理を `return` していないので副作用が生じていますし、気づきにくいですが発生するマイクロタスクの数が異なっています。Promise chain と async/await を発生するマイクロタスクの数を完全に同じにした上で同じ実行順序に書き直すのは相当に内部を理解していないと困難です(最適化されている async/await 側に対して Promise chain で追加発生するマイクロタスクを無理矢理に追加したりしないと実現できないため、元々の async/await のコードに無駄な処理をつけ足すことになる)。

これについては『[番外編 - 仕様の比較と発展](n-epasync-promise-spec-compare)』のチャプターで解説しています。
:::

# await 式

async 関数の基本については解説したので、もう１人の登場人物である await 式について詳しく見ていきましょう。

さて、async 関数では内部で await 式を使って「**非同期処理の完了を待つ**」ことができると解説されることが多いですが、この「待つ」は非常に混乱させるワードなので注意してください。

そもそも「非同期処理の完了を待つ」という文言は単体だと非同期 API などについての情報が抜け落ちてしまっているのでここでの「非同期処理」はカッコが必要です。

結論としては、**「完了を待つ」という考え方自体がそもそもよろしくない**のですが、ちょっと考えてみましょう。

## どんな作業の完了を「待つ」のか

例えば、お馴染みの非同期 API `fetch()` は Promise-based API ですから Promise インスタンスを返してきましたね。`fetch()` は引数に渡した URL からデータを取得してくれます。ですが、ネットワーキングでリクエストを投げてレスポンスを受け取るまでには時間がかかります。通常は、こんな時間のかかる処理をメインスレッドで行っていたら無駄な待ち時間が発生してしまいますね。というわけで、データ取得は API を介してその作業を委任された環境(environment)がバックグラウンドで行ってくれていました。

:::message
環境と非同期 API についての話は『[非同期 API と環境](f-epasync-asynchronous-apis)』のチャプターで解説したので、詳しくはそちらを参照してください。
:::

そして、`fetch()` は「非同期処理」の説明の代表となるように await 式での解説でもよく使われます。また、await 式は「非同期処理の完了を待つ」という話でしたね。

次のコードを見ると違和感があるはずです。一方の async 関数は非同期 API である `fech()` を await していますが、もう一方の async 関数は Promise chain を await しています。

```js
async function returnRespnose(url) {
  const response = await fetch(url);
  return response;
}
async function returnText(url) {
  const text = await fetch(url).then(res => res.text());
  return text;
}
```

`const response = await fetch(url)` の場合は非同期処理の完了を待つというよりも、環境に委任した並列的作業を待つという感じになるので、「非同期処理を待つ」とは違った印象を受けるのが自然ですね。ただし、`const text = await fetch(url).then((res) => res.text())` というような場合は `then()` のコールバック関数が完了するのを待っていますし、実際には Promise chain で最終的に返ってくる Promise インスタンスが解決されるの待つので、明らかに「非同期処理を待つ」と言えます。

```js
async function returnRespnose(url) {
  const response = await fetch(url);
  // 環境に委任した並列的作業の完了を待つ
  return response;
}
async function returnText(url) {
  const text = await fetch(url).then(res => res.text());
  // 環境に委任した並列的作業の完了後にメインスレッドでデータと共に通知させた非同期のコールバック関数の完了を待つ
  return text;
}
```

「環境がバックグラウンドで並列的に作業する非同期 API の処理の完了」を直接的に待つのか、「イベントループで並行(concurrent)に処理される非同期処理(Promise chain で登録されたコールバック関数や aysnc 関数)の完了」を待つのかで印象がかなり変わってくるため、非同期の学習ではここで勘違いや混乱などが起きることが多いです(個人的にはそうでした)。

通常は「非同期処理」という１つの単語に様々な要素が含まれるので内訳を意識しておかないと混乱します。

そういう訳で、await 式の捉え方は「非同期処理の完了を待つ」というよりも、「**Promise インスタンスを評価して値を取り出す**」というようにした方がよいです。評価して値を取り出せるようになるには評価対象の Promise インスタンスの状態が Pending から Settled (値がすでに内包された状態)になる必要があります。つまり、値を取り出すための「評価」という作業によって結果的に Promise 処理の完了を待っていることになります。

基本的には await 式は Promise インスタンスを評価して、その履行値を取り出すという処理です(後で解説しますが、await 式では単なる数値や文字列などの値を評価することも可能です)。そもそも、await 自体は演算子であり、`await ~` というように右辺を評価して評価値を返します。「await 式」が await 演算子の右辺を評価して値を返すのは当然といえばそうですが、気づにくいことなので注意してください。

https://jsprimer.net/basic/statement-expression/#expression

むしろ、これが理解できていないと `setTimeout()` を await 式で待つというようなミスを犯してしまいます。`setTimeout()` API による何らかの処理を行ってから次の処理を行いたい場合は、以前のチャプターでみたように `setTimeout()` を Promise でラッピングする Promisification をする必要があります。

```js
// Promisification
function promiseTimer(delay) {
  return new Promise((resolve) => setTimeout(resolve, delay));
}

// アロー関数で即時実行
(async () => {
  await promiseTimer(3000);
  // Promise インスタンスを返すので await で意図通りになる
  console.log("after 3000ms");
})();
```

## 「待つ」は async 関数内の処理の一時中断

環境が代行する非同期 API の並列的作業の完了を「待つ」ことについて先に言及してしまいましたが、そもそも await 式の「待つ」の意味は**実際にそこですべての処理が停止させているわけではない**のでかなり注意する必要があります。

「待つ」という言葉から直感的に想起するのは「そこで完全に処理が止まる」ということですが、そんなことが起きてしまったらメインスレッドを「ブロッキング」することになり、『[非同期 API と環境](f-epasync-asynchronous-apis)』のチャプターで説明した「非同期処理(というテーマ)」の目的である「環境が並列的にバックグラウンドで作業している間もメインスレッドをブロッキングすることなく別の作業を続けられるようにすること」が崩れてしまうことになります。

async/await は Promise chain を変形することで書くことができますし、内部的にも Promise の処理に基づいています(これについては『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)』のチャプターで解説します)。そして、**Promise chain を学習してきましたが、ブロッキングなんて起きていませんでしたよね**。await 式の「待つ」は非同期 API の作業を起点とした一連の作業「A(非同期 API の作業) したら B(コールバック関数) する、B したら C(コールバック関数) する」という逐次処理を行う時に、 A が終わっていないのに B はできないので、非同期 API の作業を環境が終わらせるまで順番的に A の次に行いたい B という作業を行わないで、**別の作業をメインスレッドで続ける**ということを意味します。「非同期 API の並列的作業である A がバックグラウンドで環境が処理している間は、その async 関数内の処理は一時的に中断させて、別のことをメインスレッドで行う」というのが「async/await でできること」であり「やりたいこと」です。

```js:immediate.js
(async () => {
  console.log("😎 async 関数の処理を開始します");
  const url = "https://api.github.com/zen";

  // 非同期 API の作業を起点にした一連の作業
  const response = await fetch(url); // 作業 A (データの取得: 非同期 API による並列的作業)
  // A が終わってから B を行うので一時中断して関数の外へ
  const text = await response.text(); // 作業 B (データの抽出: 非同期 API による並列的作業)
  // B が終わってから C を行うので一時中断して関数の外へ
  const message = "🦄 Github says... " + text; // 作業 C (データの加工)

  console.log("👻 async 関数の処理を終了します");
  return message;
})().then((message) => console.log(message)); // 一連の作業結果として得られるテキストを出力

// 環境が fetch をやってくれている間もメインスレッドで別のこと(console.log の起動)ができる
console.log(
  "👉 作業 A を起動させて await で一時中断されたらグローバルコンテキストにあるこれが実行される"
);
```

「待つ」ために行うことは「Stop(完全停止)」ではなく「**Suspend(一時中断)**」です。await 式で非同期 API から返ってくる Promise インスタンスを評価する際には(評価対象の Promise インスタンスが Settled になり、マイクロタスクが発行されてイベントループにおいてそのマイクロタスクがコールスタックのトップになるまで) async 関数内の処理を一時中断して、**関数の外の別の処理を行います**。つまり、メインスレッドをブロッキングすることなく、別の処理を行うわけです。上のコードの場合は非同期の即時実行関数なので `await fetch()` で一時中断したらそのまま呼び出し元のグローバルコンテキストに制御が戻り `console.log()` が実行されます。実際にこのコードを実行すると次のような出力を得ます。

```sh
❯ deno run -A immediate.js
😎 async 関数の処理を開始します
👉 作業 A を起動させて await で一時中断されたらグローバルコンテキストにあるこれが実行される
👻 async 関数の処理を終了します
🦄 Github says... Keep it logically awesome.
```

「待つ」という言葉は説明するのには便利な言葉ですが、**単体では情報が不足しすぎています**。「待つ」や「await」という単語に惑わされないでください。実は async/await に慣れると「完了を待つ」という言葉をコードを書く際の思考時に使うことで、**非同期処理を考える上での色々な情報を省略できて便利に感じてくる**のですが、初学者の視点ではこの単語は確実にトラップとなります。実際、非同期 API 処理については、環境がバックグラウンドで行う並列的作業であること(同時に複数のことをやっていること)を知らないと「いつどこで完了する」のかが分からず、「完了を待つ」の意味を「ブロッキングする」と混同して解釈してしまうことになるので注意してください[^いつどこで完了する]。非同期 API の処理は環境側でバックグラウンドに行われて「完了」します。直接的に非同期 API を await するのではなく、async 関数や Promise chain などを await した際にはそれらの実行コンテキストがイベントループにおいて並行(concurrent)で処理された際にも「完了」となります。

  [^いつどこで完了する]: 予備知識があまり無い状態で async/await から学んでしまうとこの現象の起きる可能性が高いと予想しています。「async/await は非同期処理を同期的に書けるようにした」という文言を見かけたことがあると思いますが、「非同期 API による並列的作業」や「関数の外へ制御を戻す」といった情報が抜け落ちている場合には「ブロッキングするのか？」というように混乱させられることになります。

:::message
中断した後にどうやって再開するのか疑問に思われるかもしれませんが、その手段はもう知っています。イベントループでのマイクロタスクの処理です。『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)』のチャプターで説明しますが、await 式は「待っている」作業が完了するとマイクロタスクを発行します。このマイクロタスクがイベントループにおいて Promise chain で見たのと同じように処理される訳です。
:::

ということで実際にはすべての処理が完全停止するわけではないので、 async 関数を単体で考えてもほとんど意味がないわけです。『[コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async)』のチャプターでも非同期処理を考えるときは必ず同期処理と一緒に考えないといけないということは言いましたが、**非同期処理そのものの意味が生じるのは他のコードとの関係性があってのこと**です。

話を戻しますが、`fetch()` API は Promise-based な非同期 API であり、`fetch()` メソッドからは Promise インスタンスが処理の結果として返ってきます。そして、`await fetch(url).then(res => res.text())` のように Promise chain を await 式で評価する場合でも、結局は Promise chain から返ってくる Promise インスタンスを評価していますね。

本質的には async/await と Promise chain は全く同じです。Promise のシステムに基づき Promise インスタンスを扱います。Promise インスタンスを介してマイクロタスクを連続で発行し、それらがイベンループ上で連鎖的に処理されることで逐次処理を実現します。

ということで、以下のコードで async/await と Promise chain の両方を書いていますが、意味はほほとんど同じです。

```js:relAsyncAwait.js
const url = "https://api.github.com/zen";

console.log("[1] 🦖 同期: タイミングがずれない");

(async () => {
  console.log("[2] 👻 💙 同期: タイミングがずれない");
  const response = await fetch(url);
  // 環境に委任した並列作業が終わってから次の行の処理にすすみたいので、
  //一旦この関数内の処理は一時的に停止して次(関数外の別の処理)に進む
  console.log("[4] 🦄 💙 非同期: タイミングがずれる");
  const text = await response.text();
  console.log("[5] 🐱 Github Philosophy:", text);
})();

console.log("[3] 🦖 同期: タイミングがずれない");
```

Promise chain のところは分かりやすいようにブロックで囲んでいます。

```js:relPromiseChain.js
const url = "https://api.github.com/zen";

console.log("[1] 🦖 同期: タイミングがずれない");

{
  // わかりやすくするために敢えてブロックにしている
  console.log("[2] 👻 💚 同期:  タイミングがずれない");
  fetch(url)
    .then((response) => {
      console.log("[4] 🦄 💚 非同期: タイミングがずれる");
      return response.text();
    })
    .then((text) => console.log("[5] 🐱 Github Philosophy:", text));
}

console.log("[3] 🦖 同期: タイミングがずれない");
```

Promise chain でブロッキングが起きていなかった様に async/await でもブロッキングは起きません。「待つ」間には別の処理がメインスレッドで実行されています。実際に上の２つのコードを実行すると出力順番はまったく同じようになります。

```sh
# async/await バージョン
❯ deno run -A relAsyncAwait.js
[1] 🦖 同期: タイミングがずれない
[2] 👻 💙 同期: タイミングがずれない
[3] 🦖 同期: タイミングがずれない
[4] 🦄 💙 非同期: タイミングがずれる
[5] 🐱 Github Philosophy: Mind your words, they are important.
# Promise chain バージョン
❯ deno run -A relPromiseChain.js
[1] 🦖 同期: タイミングがずれない
[2] 👻 💚 同期:  タイミングがずれない
[3] 🦖 同期: タイミングがずれない
[4] 🦄 💚 非同期: タイミングがずれる
[5] 🐱 Github Philosophy: Mind your words, they are important.
```

## Promise chain を async/await で書き直す

async/await は Promise chain で書き直せるのでシンタックスシュガーであると言われます。実際にはそれ以上のもの(発生するマイクロタスクの数が少なかったり、デバッグなどで得られる効能が Promise chain よりも高いなどの性質がある)ですが、現時点では Promise chain と同等なものであると考えてください。本質的なメカニズムは同じです。

例えば次のコードを考えてみましょう。

```js
const githubApi = "https://api.github.com/zen";
const fetchData = (url) => {
  return fetch(url)
    .then((response) => {
      if (!response.ok) {
        throw new Error("Error");
      }
      console.log(`[3] 👦 MICRO: got data from "${url}"`);
      return response.text();
    });
};

console.log("[1] 🦖 MAINLINE: Start");

fetchData(githubApi)
  .then((data) => {
    console.log("[4] 👦 MICRO: 取得データ", data);
  })
  .catch((error) => {
    console.error(error);
  });

console.log("[2] 🦖 MAINLINE: End");
```

Promise chain であろうと async/await であろうと、マイクロタスクを連鎖的に発行してイベントループで処理されるのは同じです。

上記の Promise chain は次のように async/await で書き直すことができます。

```js
const githubApi = "https://api.github.com/zen";
const fetchData = async (url) => {
  const response = await fetch(url);
  if (!response.ok) throw new Error("Error");
  console.log(`[3] 👦 MICRO: got data from "${url}"`);
  const text = await response.text();
  return text;
};

console.log("[1] 🦖 MAINLINE: Start");

fetchData(githubApi)
  .then((data) => {
    console.log("[4] 👦 MICRO: 取得データ", data);
  })
  .catch((error) => {
    console.error(error);
  });

console.log("[2] 🦖 MAINLINE: End");
```

async/await では try/catch が使用できるので、 async 関数内で例外を捕捉できるようにして、さらに後続のチェーンも全部書き直します。

```js
const githubApi = "https://api.github.com/zen";
const fetchData = async (url) => {
  try {
    const response = await fetch(url);
    if (!response.ok) throw new Error("Error");
    console.log(`[3] 👦 MICRO: got data from "${url}"`);
    const text = await response.text();
    return text;
  } catch (error) {
    console.error(error);
  }
};

console.log("[1] 🦖 MAINLINE: Start");

// アロー関数かつ即時実行
(async () => {
  const data = await fetchData(githubApi);
  console.log("[4] 👦 MICRO: 取得データ", data);
})

console.log("[2] 🦖 MAINLINE: End");
```

## 一時中断した後に制御を呼び出し元に戻す

制御の流れを考える上で少し補足しておきます。await 式の評価に伴い async 関数内の処理を一時的に中断するという話でしたが、１回目の中断によって制御はその async 関数の呼び出し元へと戻ります。async 関数が即時実行なら、大抵はグローバルコンテキスト(分かりやすくグローバルスコープと読み替えてもよいです)で呼び出していることが多いので、await 式の評価時にそのままグローバルコンテキストへと制御が戻ります。

```js
console.log("[1]");

// アロー関数での async 関数の即時実行
(async () => {
  console.log("[2]");
  // 非同期 API である fetch の起動命令を行い関数の外へ(メソッドから返ってくる Promise インスタンスは Pending 状態)
  await fetch(url); // 環境がバックグラウンドで並列的に処理を進める
  // 環境が fetch 処理を完了すると評価対象の Promise インスタンスが Settled となりこの関数の処理再開を告げるマイクロタスクがキューへと送られる
  // マイクロタスクがコールスタックのトップに配置されることで処理再開となり次の行が実行される
  console.log("[4]");
})();

// await で中断後にはこのグローバルコンテキストへと制御が戻り次の行を実行する
console.log("[3]"); // 環境が API 処理を行っている間もメインスレッドで実行できる
```

async 関数は別の async 関数から呼び出されることも多いです。その場合にも、呼び出した async 関数内の await 式評価によって処理が一時的に中断した際には呼び出し元の async 関数へと制御が戻ります。その際に async 関数から返ってくる Promise インスタンスはまだ Pending 状態です。

現段階でいきなり理解するのは難しいかもしれませんが、そのようなケースで制御がどのように移動していいるかを追ってみると次のようになります。`console.log()` 内の数字を追うことで制御の流れがなんとなく分かると思います。実際に実行すると数字の順番にコンソールへ出力が行われます(コードそのものに特殊な意味はありません)。

```js
async function fn1() {
  console.log("[3]"); // fn1() が呼び出された時に最初に実行される
  await fetch(url); // 非同期 API の起動(完了するまで一旦制御を呼び出し元へ戻す)

  // 環境が fetch を完了させたらここからこの関数内の処理が再開する
  console.log("[5]");
  // この処理が完了することで fn1() が完了することになるので呼び出し元の fn2() 内での await 式の評価が完了する
}
async function fn2() {
  console.log("[2]"); // fn2() が呼び出された時に最初に実行される
  await fn1(); // 別の async 関数 fn1() の呼び出し
  // fn1() を呼び出した後で await fetch() によってこの関数に制御が戻るが、
  // ここでも await 式による評価で関数内の処理を一時中断して呼び出し元のグローバルコンテキストへと制御を戻す

  // fn1() から返される Promise インスタンスが Settled となり
  // await 式の評価が完了するのでここからこの関数内の処理を再開する
  console.log("[6]");
}

// グローバルコンテキスト
console.log("[1]");
fn2(); // async 関数の呼び出し

// グローバルコンテキストへと制御が戻ってきたので次の行を実行
console.log("[4]");
// グローバルコンテキストの処理はこれしかないので、環境がバックグラウンドで行っている
// fetch が完了するまで待機して、それが完了したら fn1() で中断した途中の処理から再開する
```

async/await ではなく Promise を返す通常の関数と Promise chain で書き直すと次のようになります。この形態なら何が起きているか分かりやすいでしょう。

```js
function fn1() {
  console.log("[3]");
  return fetch(url) // Promise を返すので chain できる
    .then(() => console.log("[5]"));
}
function fn2() {
  console.log("[2]");
  return fn1() // Promise を返すので chain できる
    .then(() => console.log("[6]"));
}

// グローバルコンテキスト
console.log("[1]");
fn2(); // Promise を返す関数の呼び出し

// グローバルコンテキストへと制御が戻ってきたので次の行を実行
console.log("[4]");
// グローバルコンテキストの処理はこれしかないので、環境がバックグラウンドで行っている
// fetch が完了するまで待機して、それが完了したら fn1() で中断した途中の処理(then で chain して登録されたコールバック)から再開する
```

`fn1()` 関数を書くのをやめて１つの Promise chain へ繋げるようにすると次のようになります。

```js
function fn2() {
  console.log("[2]");
  console.log("[3]");
  return fetch(url) // Promise を返すので chain できる
    .then(() => console.log("[5]"))
    .then(() => console.log("[6]"));
}

// グローバルコンテキスト
console.log("[1]");
fn2(); // Promise を返す関数の呼び出し

// グローバルコンテキストへと制御が戻ってきたので次の行を実行
console.log("[4]");
// グローバルコンテキストの処理はこれしかないので、環境がバックグラウンドで行っている
// fetch が完了するまで待機して、それが完了したら fn2() で中断した途中の処理(then で chain して登録されたコールバック)から再開する
```

関数すら書くのをやめて最初から `fetch()` という非同期 API を起点にした Promise chain で書いてしまうと次のようになります。

```js
// グローバルコンテキスト
console.log("[1]");

// 関数で書くこともやめた
console.log("[2]");
console.log("[3]");
fetch(url) // Promise を返すので chain できる
  .then(() => console.log("[5]"))
  .then(() => console.log("[6]"));

console.log("[4]");
// グローバルコンテキストの処理はこれしかないので、環境がバックグラウンドで行っている
// fetch が完了するまで待機して、それが完了したら chain しているコールバックがマイクロタスクキューへと送られる
```

極論を言ってしまえば、async/await は「所詮この程度のことを扱いやすいように書き直している」だけです。現時点では Promise chain の方が理解しやすいと思いますが、async 関数という単位でレイヤー化して、内部で `Promise.all()` などの静的メソッドを await して並列化を行ったり for ループでの反復処理や try-catch を使って例外処理を書けるようになることで Promise が関与する処理の利便性が向上します。

# Callback hell → Promise chain → async/await

より俯瞰的な視点で Promise chain から async 関数への変形を見てみます。

次の JSConf EU で行われた Shelley Vohr 氏による『Asynchrony: Under the Hood』の公演動画を 23:36 ~ のところから視聴してみてください。Callback hell → Promise chain → async/await の変形が視覚的に示されていて変形のイメージをつかめます。

https://youtu.be/SrNQS8J67zc

:::message
ここではシンプルに Promise chain から async 関数へと変形でき同じものであることをイメージできることが重要です。最初は変換を意識してみると理解しやすいですが、実際に使用する際には Promise を扱っていることだけ意識しておけばいいと思います。
:::

以下、動画で示されているコード変形について見ていきます。
Promise が無かった時代、非同期処理はコールバックベース API などで次のように、「A したら B する、B したら C する」というような逐次的な処理を行っていました。

ただし、見て分かるようにコールバックではネストが増えていくにつれてコードを把握するのが困難になります。このような形式は忌避の意味合いを込めて "**Callback hell**" (コールバック地獄)と呼ばれています。

```js:Callback hell
getData(a => {
  getMoreData(a, b => {
    getMoreData(b, c => {
      getMoreData(c, d => {
        getMoreData(d, e => {
          console.log(e);
        })
      })
    })
  })
});
```

そのネストの形状から "**Pyramid of doom**" (破滅のピラミッド)とも呼ばれます。

https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)

ES2015 での Promise 機能の登場により、Callback hell を作らずに、非同期の逐次的な処理を Promise chain で実現できるようになりました。

この時点では Callback hell から Promise chain へと変わったことで、**そもそもタスクベースの逐次処理からマイクロタスクベースの逐次処理へと変わっている**ことに注意してください。`getData()` と `getMoreData()` は Promise を返してくる非同期処理であると認識してください。

```js:Promise chain
getData()
  .then(a => getMoreData(a))
  .then(b => getMoreData(b))
  .then(c => getMoreData(c))
  .then(d => getMoreData(d))
  .then(e => console.log(e));
```

さらに、ループや try/catch などの古典的で平凡な処理を非同期処理と共にでできるように **Promise のシステムに基づいた拡張的な機能**として ES2017 で登場した async/await により、上の Promise chain を下のように変形できるようになりました。async/await は Promise に基づいているので、Promise chain と同じ様に Promise の状態に応じて連鎖的にマイクロタスクを発行している点に注意してください。Promise chain の時と同じ様に、`getData()` と `getMoreData()` は Promise を返してくる非同期処理であると認識してください。

```js:async/await
(async () => {
  const a = await getData(a);
  const b = await getMoreData(b);
  const c = await getMoreData(c);
  const d = await getMoreData(d);
  const e = console.log(e);
})();
```

Callback hell で見たようにコールバックスタイルのものと Promise chain はタスクとマイクロタスクというように裏の仕組み自体が違うのに対して、Promise chain と async/await は同じく、Promise インスタンスを使用してマイクロタスクを発行するものであることを意識することが重要です。

# async/await は Promise に基づき Promise を扱う

繰り返しますが、async/await は Promise chain の(ほぼ)シンタックスシュガーであるため、Promise インスタンスを取り扱っているという意識を持つことが重要です。

async/await で Promise を意識するための重要なポイントは２つあります。

- async 関数はどんなときでも必ず Promise インスタンスを返す
- await 式は Promise インスタンスを評価して値を取り出す(Promise インスタンス以外の値を評価する場合は一旦 Promise でラッピングして評価する)

以下の項目でそれぞれを確認しますが、原理については次のチャプターで詳しく説明します。

##  async 関数はどんなときでも必ず Promise インスタンスを返す

async 関数はどんなときでも必ず Promise インスタンスを返します。例えば、次のように何もしない関数を定義した場合でも、これを実行すると Promise インスタンスが返ってきます。

```js
async function empty() {}
```

async 関数からは必ず Promise インスタンスが返ってくるので、返り値である Promise インスタンスに対して今まで通り Promise chain が構築できます。

```js
// 即時実行
(async function empty() {})()
  .then(data => console.log(data)); // undefined が出力される
```

この場合、async 関数からは同期的に履行状態の Promise インスタンスが返ってくるので、チェーンされた `then()` メソッドのコールバック関数が直ちにマイクロタスクキューへとマイクロタスクとして発行されます。

async 関数内で `return` された値が返り値の Promise インスタンスの解決値となります。返り値と async 関数から返ってくる Promise インスタンスの状態、そのインスタンスが持つ値についての基本的な関係は以下となります。

`return` された値 | async 関数から返却される Promise の最終的状態 | 返却される Promise が持つ値 |
--|--|--
通常の値 | 履行状態 | 元々の値
履行状態の Promise | 履行状態 | `return` した Promise の履行値
拒否状態の Promise | 拒否状態 | `return` した Promise の拒否理由

`return` で Promise インスタンスを返した場合は Promise chain において `then()` メソッドで Promise インスタンスを返した場合と同じ様に、async 関数から返ってくる Promise インスタンス自体の状態も `return` した Promise インスタンスと同じになり、Promise が持つ値も `return` した Promise インスタンスの履行値や拒否理由となります。

何も `return` しない場合には、async 関数から返ってくる Promise インスタンスは同期的に `undefined` で履行されます(なぜこうなるのかは次のチャプターで説明します)。

実際、async 関数で返却する値としての Promise インスタンスには特に興味がなく await 式を利用した一連の逐次処理がしたいから async 関数を利用するというのは十分にありえます。即時実行するケースなどは大体そういう場合が多いのではないでしょうか。

```js
// await 式が使いたいから async function を書く
(async function justFetch() {
  const url = "https://api.github.com/zen";
  const response = await fetch(url);
  const text = await response.text();
  console.log(text);
  // undefined で履行される Promise インスタンスが返る
  // この関数から返ってくるものには興味がない
})();
```

ちなみに、永遠に Pending 状態の Promise インスタンスを `return` した場合などは今までの Promise chain と同じように、次の `then()` メソッドのコールバックなどを実行できません(`then()` メソッドはチェーンしている Promise インスタンスの状態が遷移した時に一度だけ実行されるので、永遠に Pending 状態なら実行されることがありません)。

```js
(async function pendingPromise() {
  return new Promise(() => {
    // resolve も reject もしない
  }); // 永遠に Pending 状態
})()
  .then(data => console.log("実行されない", data))
  .catch(err => console.log("実行されない", err))
  .finally(() => console.log("実行されない"));
```

## await 式は Promise インスタンスを評価して値を取り出す

await 式では「非同期処理の完了を待つ」というよりも、「**Promise インスタンスを評価して値を取り出す**」という側面の方が重要です。

await 式による評価を意識するために次のコードを考えます。

```js
const myPromise = new Promise(resolve => {
  resolve(42);
});

(async function increment() {
  let value = await myPromise; // 履行値である 42 という値を取り出す
  value++; // 値として取り出したことでインクリメントできる
  return value;
})()
  .then(data => console.log("インクリメント", data)) // 43
  .catch(err => console.log("実行されない", err))
  .finally(() => console.log("最後に実行"));
```

上のコードでは、明示的にコンストラクタで作成された Promise インスタンスである `myPromise` を async 関数内の await 式で評価して履行値である `42` という数値を取り出しています。そして、インクリメントをして `return` で返却しています。

async 関数から返ってくるのは Promise インスタンスなので、チェーンされた `then()` メソッドのコールバック関数の入力として値を取り出すことができます(これは『[コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)』のチャプターでみましたね)。従って、上のコードでコメントしてあるようにインクリメントした値である `43` という数値を出力できます。

もちろん、Rejected 状態の Promise インスタンスを評価することも可能です。ただし、Rejected 状態の Promise インスタンスを await 式で評価した場合は async 関数内のそれ以降の処理はスキップされます。

```js
// awaitRejectPromise.js
(async function increment() {
  const value = await Promise.reject("reason"); 
  console.log("これは実行されない");
  return value;
})()
  .then((data) => console.log("これは実行されない", data))
  .catch((err) => console.log("実行される", err))
  .finally(() => console.log("最後に実行される"));
```

この場合 async function から返ってくる Promise インスタンス自体が拒否状態となります。従って、チェーンしている `then()` メソッドのコールバックは実行されずに、`catch()` メソッドによって例外として補足されてコールバックが実行されます。

V8 で実行すると以下の出力を得ます。

```sh
❯ v8 awaitRejectPromise.js
実行される reason
最後に実行される
```

これは async 関数内で例外を throw した場合と同じです。それ以降の処理はスキップされて、async 関数から返ってくる Promise インスタンスは拒否状態となり、`catch()` メソッドで拒否理由と共に例外が補足されます。

```js
// throwErrorInAsync.js
(async function increment() {
  throw new Error("例外発生");

  console.log("これは実行されない");
  return 42; // 意味がない
})()
  .then((data) => console.log("これは実行されない", data))
  .catch((err) => console.log("実行される", err))
  .finally(() => console.log("最後に実行される"));
```

従って、上のコードを実行すると以下の出力を得ます。

```sh
❯ v8 throwErrorInAsync.js
実行される Error: 例外発生
最後に実行される
```

async 関数では古典的な例外補足の方法として try/catch/finally を使用できます。

```js
// awaitRejectPromise-kai.js
(async function increment() {
  let value = "defalut value";
  try {
    value = await Promise.reject("reason");
    console.log("😭 これは実行されない");
  } catch (err) {
    console.log("👹 実行される:", err);
  } finally {
    console.log("🦄 最後に実行される");
  }
  return value;
})()
  .then((data) => console.log("😅 これは実行される:", data))
  .catch((err) => console.log("😭 実行されない", err))
  .finally(() => console.log("🦄 最後に実行される"));
```

このようにした場合、try/catch で例外補足するため、async 関数から返ってくる Promise インスタンスは履行状態となります。したがって、チェーンしている `then()` メソッドのコールバックは実行でき、逆に `catch()` メソッドのコールバックは実行されません。

```sh
❯ v8 awaitRejectPromise-kai.js
👹 実行される: reason
🦄 最後に実行される
😅 これは実行される: defalut value
🦄 最後に実行される
```

async 関数内で try/catch/finally を使えば、今までのようにチェーンする必要はなくなるので、チェーン部分はなくしても良いでしょう。

```js
(async function increment() {
  let value = "defalut value";
  try {
    value = await Promise.reject("reason");
    console.log("😭 これは実行されない");
  } catch (err) {
    console.log("👹 実行される:", err);
  } finally {
    console.log("🦄 最後に実行される");
  }
  return value;
})();
```

## await 式は Promise インスタンスでないのものも評価できる

ここまで見てきたように、await 式は基本的には Promise インスタンスを評価するものですが、Promise インスタンスでない単なる値も評価できてしまいます(そのようなことをする意味自体はあまりない)。

そのような場合に何が起きるかというと、例外を発生させずに、await 式で評価する値を一旦 Promise インスタンスでラッピングしてから、値を取り出します。実はこれによって無駄なマイクロタスクと Promise インスタンスが生成されるので、コードを書く上では基本的にやる意味がないです。

```js
(async function increment() {
  let value = await 42;
  // 一旦 Promise インスタンスでラッピングされて履行値 42 がとりだされる
  value++;
  return value;
})()
  .then(data => console.log("インクリメント", data)) // 43
  .catch(err => console.log("実行されない", err))
  .finally(() => console.log("最後に実行"));
```

なぜこのようなことが起きるのかというと、[Await(value)](https://tc39.es/ecma262/#await) 操作の以下の仕様でそうするように決まっているからです。これは `Promise.prototype.then` メソッドに渡すコールバック関数から通常の値が返されたときに `then` メソッドからは常に新しい Promise インスタンスが返るのと同じような話です。コールバックで何を返そうが Promise が返されるのと同じで、await で何を評価しようが Promise として処理されるようになっています。

> - 2. Let promise be ? [PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve)([%Promise%](https://tc39.es/ecma262/#sec-promise-constructor), value).

具体的に裏でどのようなことが起きているのかは次のチャプターで確認します。

とにかく、await 式は基本的には Promise インスタンスを評価して値を取り出すものであると意識するのが重要です。
