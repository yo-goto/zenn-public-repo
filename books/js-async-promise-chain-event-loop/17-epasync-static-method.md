---
title: "Promise の静的メソッドと並列化"
aliases: [ch_Promise の静的メソッドと並列化]
---

# このチャプターについて

「いつ await すべきか分からない」というのが、一般的に async/await での難しいポイントになりますが、すでに Promise インスタンスを評価して値を取り出すという await 式の特徴については学んでいるため、「await する必要性の判断」や「なぜ await するべきか」についてはそこまで難しくは感じないでしょう。難しさを感じるとしたら、await の配置(タイミング)による効率化やループなどに関わる部分だと思います。

つまり、「非同期処理の学習」の後半戦では await 式の配置がキーになるでしょう。await 式が内部で必要なコールバック関数は必然的に async 化する必要があったりするので、await 式の配置によって async キーワードが引きずられます。await 式の配置によって効率的に順序付けする、あるいは意図的に実行順序付けしないことが重要です。

このチャプターでは await 式の配置を考えるテーマの１つとして Promise の静的メソッド(Static method)による複数の Promise-based API の処理の並列化を考えます。すでに非同期 API については色々と見てきましたが、すでにお馴染み Promise-based API である `fetch()` メソッドを使用してもう一度 await 式について考えを巡らせておきます。

# 非同期 API による並列化

『[同期 API とブロッキング](f-epasync-synchronus-apis)』のチャプターで解説したとおり、Promise chain や async/await は非同期 API を起点にした一連の作業の「実行と完了」の順番を担保するものです。そして、非同期 API のおかげで同時に複数のことができますが、競合するような複数の非同期 API 処理は同時に行ってはならなず、順番を決めて行うようにすべきであるということを解説しました。

その一方で、関連性のない複数の作業については同時に行っても問題は無いということも言いました。

つまり、関連のない複数操作は並列化(非同期 API 処理は同時に複数実行できる)させて、効率化を測ることができます。複数の Promise 処理を１つずつ await するのではなく、処理を起動した後で `Promise.allSettled()` などの静的メソッドでまとめて await する(`await Promise.allSettled([...promises])`)ことで並列化できます。

ただし、「非同期 API を使うことで同時に複数のことができる」ということを知っていないとこの静的メソッドでの「並列化」が何を意味しているのか分からず、並行や非同期と混同し混乱することになります。例えば、async 関数そのものを並列化していると思っていても、実際に並列化できるのは内部的に含まれている非同期 API の処理で API 以外の付随する処理はイベントループによって並行的に実行されるので「並列化」という言葉が意味していることを勘違いしてしまうケースが多いです。この場合には、「並列」と「非同期」と「並行」の複数概念が同時に必要となり、俯瞰的に組み合わせて理解する必要があります。メインスレッドでは常になんらかの処理(コールバック関数など)が実行されており、複数の非同期 API 処理は環境がバックグラウンドで行っていることを忘れないようにしてください。

次のコードの例では、リソース間の取得順番に意味がない(依存関係がない)ので複数の `fetch()` API から始まる一連作業を並列化しています。以下のコードで出てくる `urls` 変数は URL 文字列の配列として考えてください。

```js
(async () => {
  // 並列的(parallel)に複数の fetch API を起動
  // Promise インスタンスの配列を作成して Promise.allSettled にわたす
  const promises = [
    fetch(urls[0]).then(response => response.text()).then(text => console.log(text)),
    fetch(urls[1]).then(response => response.text()).then(text => console.log(text)),
    fetch(urls[2]).then(response => response.text()).then(text => console.log(text)),
  ];
  // chain の部分は並行的(concurrent)にスケジューリング
  // 「並列化」できるのはあくまで非同期 API の処理であり、そのおかげ
  // ある非同期 API 処理が終わっていないときも別の JS コードや他の非同期 API 処理が実行できる

  // まとめて await させることで時間短縮して効率化
  await Promise.allSettled(promises); 
  // 上の処理の完了が担保されてからコンソールにメッセージを出力
  console.log("⭐️ 並列化した複数フェッチがすべて終了しました");
})();
```

まとめて await させるというのはすべての並列化されたデータフェッチが完了してから次に行いたいなにかの作業があるので、そこで async 関数の処理を一時停止させているということです。await によって並列化しているわけではなく、起動自体がそもそも並列化しており、すべてのデータフェッチが終わってから `console.log()` を行いたいので `Promise.allSettled()` の部分で一時的に停止するように指定しているという様に解釈してください。

もしも次のように１つずつ await してしまったら、非同期 API による同時に複数できることのメリットを活かしきれていないことになるので上のコードよりも時間がかかり効率が悪くなってしまいます。この場合はいわゆる「直列」となります。

リソースの取得順番になにか意味があるならこのようなコードでも良いですが、そうでないなら無駄なので上のように並列化します。

```js
(asycn () => {
  await fetch(urls[0]).then(response => response.text()).then(text => console.log(text));
  // 上の fetch から始める一連の処理がすべて終わってから次の fetch を行う
  await fetch(urls[1]).then(response => response.text()).then(text => console.log(text));
  // 上の fetch から始める一連の処理がすべて終わってから次の fetch を行う
  await fetch(urls[2]).then(response => response.text()).then(text => console.log(text));
  // 上の fetch から始める一連の処理がすべて終わってからコンソールにメッセージを出力
  console.log("すべてのフェッチが終了しました");
})();
```

同期 API では意図的にブロッキングして同時に１つのことしかやらないようにしているため、並列化はもちろんできません。並列化ができるのは非同期 API そのものが環境がブロッキングすることなくバックグラウンドで並列化できるようにしていくれているからです。環境が裏側でいくつかスレッドを使っていたり、ポーリングの仕組みによってタイマーなどはまとめて管理され有効期限が切れた複数タイマーのコールバックをまとめてタスクキューへと送信していたりします。

いきなり少し複雑なものを出したのは次のような単なる非同期 API 単体でのやり方との違いを理解するためです。この場合なら複数の非同期 API を並列化していうことが一目瞭然で分かりやすいです。

`Promise.all([fetch(urls[0]),fetch(urls[1]),fetch(urls[2])])` の部分で非同期 API の並列化を行っています。もちろん `fetch()` メソッドの起動自体は１つずつですが、すべて起動した後は時間的にしばらく処理が継続するので、「並列」となっています。

```js
console.log("🐶 並列化した複数フェッチの開始");
Promise.all([
  fetch(urls[0]), 
  fetch(urls[1]),
  fetch(urls[2])
]).then(([res1, res2, res3]) => {
    console.log("⭐️ 並列化した複数フェッチがすべて終了しました");
    const promises = [
      res1.text(), 
      res2.text(), 
      res3.text()
    ];
    return Promise.all(promises);
  })
  .then(([text1, text2, text3]) => {
    console.log("text1:", text1);
    console.log("text2:", text2);
    console.log("text3:", text3);
  });
```

Promise chain の `then()` メソッドの入力には配列の分割代入を使用していることに注意してください。ただ、このような書き方はあまり一般的ではないと思われます。

async/await が使える今では冗長だったり、例外処理を try/catch で分かりやすく閉じ込めてしまいたいことがあるので Promise chain ではなくて同じ部分を async 関数にまとめてしまった方がわかりやすく、コントロールもしやすいことがあります。

```js
async function fetchThenConsole(url) {
  try {
    const response = await fetch(url);
    const text = await response.text();
    console.log(text);
  } catch(err) {
    console.log(err);
  }
}
(async () => {
  // Promise インスタンスの配列を作成(urls は配列)
  const promises = urls.map(url => fetchThenConsole(url));
  await Promise.allSettled(promises);
  // 上の処理の完了が担保されてからコンソールにメッセージを出力
  console.log("⭐️ 並列化した複数フェッチがすべて終了しました");
})
```

ただ、async 関数内部のコードや関数の起動時にはいつでも Promise chain は使えるので冗長であれば必要に応じて部分的に chain させて短縮化しても良いでしょう。

上のコードのように、配列の `map()` メソッドなどを使って async 関数を複数起動させて返ってくる Promise インスタンスの配列を作ることもできます。取得したいリソース同士の依存関係がなければこの時点で await する必要なく、`Promise.allSetteld()` や `Promise.all()` などでまとめ上げてしまいます。

# Promise Combinator による「合成」

`Promise.all()` や `Promise.allSettled()` などの Promise オブジェクトの静的メソッドは Promise インスタンスの配列を引数に受け取ります(厳密には Iterable オブジェクト)。そして複数の Promise 処理を「合成(Composition)」することから、"**Promise combinators**" と呼ばれています。

https://v8.dev/features/promise-combinators

分かりやすいものが `Promise.all()` です。この静的メソッドは Promise インスタンスの配列を受けとり、自身も Prosmise インスタンスを返します。配列内のすべての Promise インスタンスが履行状態になった場合に `Promise.all()` から返る Promise インスタンスも履行状態となります。

```js
Promise.all([pormise1, promise2, promise3])
  .then([value1, value2, value3] => {
    console.log({ value1 });
    console.log({ value2 });
    console.log({ value3 });
  })
```

`Promise.all()` の返り値となる Promise インスタンスには引数として渡した配列内の複数 Promise インスタンスの結果、つまり履行値の配列が格納されています。分かりやすいように次のように直ちに履行する Promise インスタンスを `Promise.reoslve()` で作成してみます。

```js
const promiseArray = [
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3),
];
Promise.all(promiseArray)
  .then(data => {
    console.log(data);
    // => [ 1, 2, 3 ]
  });
```

`Promise.all()` からは返ってくるのは Promise インスタンスなのでもちろん chian できます。そしてその Promise の履行値は `[ 1, 2, 3 ]` という数値の配列であることが分かります。`then()` で chain した際に入力として受け取る `data` は `Promise.all()` から返る Promise インスタンスの履行値です。

こんな感じで複数の Promise インスタンスをまとめ上げて、すべてが履行状態になった後でなにかしたいと言う時にはこの `Promise.all()` メソッドを使用します。このような使い方がいわゆる「Promise の合成(Composition)」と呼ばれる行為です。

「合成(Composition)」という用語がありますが、実際にやっていることは複数の Promise インスタンスを別の Promise インスタンスで包みこみ、内部の Promise インスタンスの状態によって包んでいる Promise インスタンスの状態が履行か拒否に決まるだけです。「合成」という言葉にそこまで特別な意味はないので、あまりこだわる必要はないです。逆にこだわりすぎると「直列」とかの言葉に変に惑わされることになるので、単に複数の Promise 処理が終わってからなにか関連する処理がしたいので「**複数の Promise 処理をまとめあげる**」程度の認識で十分だと思います。

**まとめあげるのは Promise インスタンスです**。これを認識しておくことで、配列に何を渡すか混乱せずにすみます。以下のように異なるタイプの処理でもすべて Promsise インスタンスが返っているものしか渡していません。

```js
Promise.all([
  promiseAPI(),  // Promise が返る
  fetch(url), // Promise が返る
  asyncFunc(), // Promise が返る
  (async () => {})(), // Promise が返る
  promiseChain(), // Promise が返る
  Promise.resolve().then().then(), // Promise が返る
]).then(() => console.log("すべてのPromise処理が履行しました"));
```

結局は次のように Promise インスタンスをまとめているだけです。

```js
Promise.all([
  promise1, // promiseAPI(),  
  promise2, // fetch(url), 
  promise3, // asyncFunc(), 
  promise4, // (async () => {})(), 
  promise5, // promiseChain(), 
  promise6, // Promise.resolve().then().then(), 
]).then(() => console.log("すべてのPromise処理が履行しました"));
```

async/await で書くなら次のような感じで、「複数の Promise 処理がすべて履行してから `console.log()` を行いたい」という意図で `Promsie.all()` を await します。

```js
(async () => {
  const promises = [
    promise1,
    promise2,
    promise3,
    promise4,
    promise5,
    promise6,
  ];
  await Promise.all(promises);
  console.log("すべてのPromise処理が履行しました")
})();
```

# Promise combinator の種類

Promise combinator は ES2022 の時点ですでに４つの種類が存在しています。４つは便宜的に対応関係にあると考えることができるので次のように対応付けられます。

- `Promise.allSettled()` vs `Promise.all()` : 対象の複数の Promise 処理同士に依存関係があるかどうか
- `Promise.race()` vs `Promise.any()` : Promise 処理の成功や失敗に興味があるかどうか

MDN のドキュメントには以下のように Promise の静的メソッドやプロトタイプメソッドが網羅されています。
![mdnでの説明](/images/js-async/img_promise-methods-in-mdn.png)

Promise combinator は `Promise.resolve()` や `Promise.reject()` と同じく静的メソッドです。

引数となる複数の Promise 処理に対して何をしたいのかという目的によってこれらのメソッドを使い分けます。この４つの静的メソッドはすべて複数の Promise 処理をまとめあげて並列化しますが、引数として渡す配列内の Promise の状態によって返り値の Promise インスタンスの状態がどうなるか変わります。

複数の Promise 処理すべてに興味があり、Promise 処理同士に依存関係があり１つも失敗したくないという場合には、`Promise.all()` を使用します。`fetch()` で考えるとすべてのリソースフェッチに成功する必要があるなら、この `Promise.all()` を利用します。複数の Promise 処理の成功や失敗に興味がなく、とりあえずすべての処理を行ってからなにかしたいという場合には `Promise.allSettled()` を使用します。複数の Promise 処理がすべて Settled になったら返り値の Promise インスタンスが履行状態になります。

複数の Promise 処理すべてには興味がなく、対象となるものの内のどれか１つの Promise インスタンスが Settled になるかどうかに興味があり、履行か拒否には興味がない場合には `Promise.race())` を使用します。履行されたものに興味があり、最初に履行したものを取り出したい場合には `Promise.any()` を使用します。この２つのメソッドも複数の Promise 処理を合成して並列化しますが、１つのものが条件を満たした時点で他の処理については考える必要がなくなり、完了、つまり返り値の Promise インスタンスが履行状態となります。


<!-- 
# await で制御する

ここでは次の "JSONPlaceholder" というフリーで使える fake API サービスを使って JSON データを複数取得することを考えます。

https://jsonplaceholder.typicode.com

>JSONPlaceholder is a free online REST API that you can use whenever you need some fake data. It can be in a README on GitHub, for a demo on CodeSandbox, in code examples on Stack Overflow, ...or simply to test things locally.
>([JSONPlaceholder - Free Fake REST API](https://jsonplaceholder.typicode.com/) より引用)

```js
const urls = {

};
```

 -->

