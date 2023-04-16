---
title: "Promise の静的メソッドと並列化"
cssclass: zenn
date: 2022-06-26
modified: 2023-03-31
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『Promise の静的メソッドと並列化』
---

## このチャプターについて

「いつ await すべきか分からない」というのが一般的に async/await での難しいポイントになりますが、すでに Promise インスタンスを評価して値を取り出すという await 式の特徴については学んでいるため、「なぜ await するべきか」についてはそこまで難しくは感じないでしょう。難しさを感じるとしたら、await の配置 (タイミング) による効率化やループなどに関わる部分だと思います。

このチャプターでは await 式の配置による制御の話題へ入る前に、テーマの１つとして Promise の静的メソッド (Static method) による複数の Promise-based API の処理の並列化を考えます。すでに非同期 API については色々と見てきましたが、すでにお馴染み Promise-based API である `fetch()` メソッドを使用してもう一度 await 式について考えを巡らせておきます。

## 非同期 API による並列化

『[同期 API とブロッキング](f-epasync-synchronus-apis)』のチャプターで解説したとおり、Promise chain や async/await は非同期 API を起点にした一連の作業の「実行と完了」の順番を担保するものです。そして、非同期 API のおかげで同時に複数のことができますが、競合するような複数の非同期 API 処理は同時に行ってはならなず、順番を決めて行うようにすべきであるということを解説しました。

その一方で関連性のない複数の作業については同時に行っても問題は無いということも述べました。

つまり、関連のない複数操作を並列化 (非同期 API 処理は同時に複数実行できる) させて、効率化を測ることができます。複数の Promise 処理を１つずつ await するのではなく、処理を起動した後で `Promise.allSettled()` などの静的メソッドでまとめて await する (`await Promise.allSettled([...promises])`) ことで並列化できます。

ただし、非同期 API について理解した上で「並列化」という言葉の意味を考える必要があります。「非同期 API を使うことで同時に複数のことができる」ということを知っていないと、この静的メソッドでの「並列化」が何を意味しているのか分からず、並行や非同期と混同して混乱することになります。

例えば、async 関数そのものを並列化していると思っていても **実際に並列化できるのは内部的に含まれている非同期 API の処理** であり、API 以外の付随する処理はイベントループによって並行 (concurrent) に実行されるので、「並列化」という言葉が意味していることを勘違いしてしまうケースが多いです。この場合には、「並列」と「非同期」と「並行」の複数概念が同時に必要となり、俯瞰的に組み合わせて理解する必要があります。メインスレッドでは常になんらかの処理 (コールバック関数など) が実行されており、複数の非同期 API 処理は環境がバックグラウンドで並列的に行っていることに注意してください。

:::message alert
ここで使う「並列化」という言葉は複数スレッドが関わる厳密な「並列 (parallel) 処理」のことではなく、「同時に複数のことができる」という性質の意味で使っています。実際に並列 (parallel) と言っても良いものもあるのですが、そう言い切れないものがあるので注意してください。
:::

例えば次のコードでは、リソースを取得する順番に特に意味がない (依存関係はない) ので複数の `fetch()` API から始まる一連の作業を並列化しています。「一連の作業を並列化する」といっても内部的には非同期 API が並列化されています。

以下のコードで出てくる `urls` 変数は URL 文字列の配列として考えてください。

```js
(async () => {
  // 並列的に複数の fetch API を起動
  // Promise インスタンスの配列を作成して Promise.allSettled にわたす
  const promises = [
    fetch(urls[0]).then(response => response.text()).then(text => console.log(text)),
    fetch(urls[1]).then(response => response.text()).then(text => console.log(text)),
    fetch(urls[2]).then(response => response.text()).then(text => console.log(text)),
  ];
  // chain の部分のコールバックは並行的(concurrent)にスケジューリング
  // 「並列化」できるのはあくまで非同期 API の処理であり、そのおかげ
  // ある非同期 API 処理が終わっていないときも別の JS コードや他の非同期 API 処理が実行できる

  // まとめて await させることで時間短縮して効率化
  await Promise.allSettled(promises);
  // 上の処理の完了が担保されてからコンソールにメッセージを出力
  console.log("⭐️ 並列化した複数フェッチがすべて終了しました");
})();
```

まとめて await させるというのはすべての並列化されたデータフェッチが完了してから次に行いたいなにかの作業があるので、そこで async 関数の処理を一次中断させているということです。await 処理によって並列化が行われている訳ではなく、非同期 API の起動自体がそもそも並列化しており、「すべてのデータフェッチが終わってから `console.log()` を行いたいので `Promise.allSettled()` を await 評価する箇所で async 関数内の処理を一時中断するよう指定している」という意図のコードとして解釈してください。

もしも次のように１つずつ await してしまったら、非同期 API による同時に複数できることのメリットを活かしきれていないことになり、上のコードよりも時間がかかることで効率が悪くなってしまいます。この場合はいわゆる「直列」となります。リソースの取得順番になにか意味があるならこのようなコードでも良いですが、そうでないなら無駄なので上のように並列化します。

```js
(async () => {
  await fetch(urls[0]).then(response => response.text()).then(text => console.log(text));
  // 上の fetch から始める一連の処理がすべて終わってから次の fetch を行う
  await fetch(urls[1]).then(response => response.text()).then(text => console.log(text));
  // 上の fetch から始める一連の処理がすべて終わってから次の fetch を行う
  await fetch(urls[2]).then(response => response.text()).then(text => console.log(text));
  // 上の fetch から始める一連の処理がすべて終わってからコンソールにメッセージを出力
  console.log("すべてのフェッチが終了しました");
})();
```

並列化ができるのは非同期 API の性質のおかげであり、環境がバックグラウンドで並列的に処理してくれているからです。環境は裏側でいくつかスレッドを使っていたり、ポーリングの仕組みによってタイマーなどはまとめて管理され有効期限が切れた複数タイマーのコールバックをまとめてタスクキューへと送信していたりします。

いきなり少し複雑なものを出したのは次のような単なる非同期 API 単体でのやり方との違いを理解するためです。この場合なら複数の非同期 API を並列化していることが一目瞭然で分かりやすいです。

`Promise.all([fetch(urls[0]),fetch(urls[1]),fetch(urls[2])])` の部分で非同期 API の並列化を行っています。もちろん `fetch()` メソッドの起動自体は１つずつですが、すべて起動した後は時間的にしばらく処理が継続するので、「並列」となっています。

```js
console.log("🐶 並列化した複数フェッチの開始");
Promise.all([
  fetch(urls[0]), // promise-based API
  fetch(urls[1]), // promise-based API
  fetch(urls[2]), // promise-based API
]).then(([res1, res2, res3]) => {
    console.log("⭐️ 並列化した複数フェッチがすべて終了しました");
    const promises = [
      res1.text(), // promise-based API
      res2.text(), // promise-based API
      res3.text(), // promise-based API
    ];
    return Promise.all(promises);
  })
  .then(([text1, text2, text3]) => {
    console.log("text1:", text1);
    console.log("text2:", text2);
    console.log("text3:", text3);
  });
```

Promise chain の `then()` メソッドの入力には配列の分割代入を使用していることに注意してください。ただ、このような書き方は今ではあまり一般的ではないと思われます。

async/await が使える今では冗長だったり、例外処理を try/catch で分かりやすく閉じ込めてしまいたいことがあるので Promise chain ではなくて同じ部分を次のように async 関数にまとめてしまった方がわかりやすく、コントロールもしやすいことがあります。

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
})();
```

ただ、async 関数内部のコードや関数の起動時にはいつでも Promise chain は使えるので冗長であれば必要に応じて部分的に chain させて短縮化しても良いでしょう。

上のコードのように、配列の `map()` メソッドなどを使って async 関数を複数起動させて返ってくる Promise インスタンスの配列を作ることもできます。取得したいリソース同士の依存関係がなければこの時点で await する必要なく、`Promise.allSetteld()` や `Promise.all()` などでまとめ上げてしまいます。

繰り返しますが「並列化」できているのは `Promise.allSettled()` や `Promise.all()` とは一切関係無く `fetch()` という「非同期 API」そのものの性質のおかげです。次のように `Promise.all()` を書かなくても関係なく、そのまま実行して「並列化」となります。

```js
// 並列起動(実際には１つずつ起動しているが起動後にバックグラウンドで処理が時間的に継続するので並列化となる)
fetch(urls[0]);
fetch(urls[1]);
fetch(urls[2]);
```

ということで次のように Promise chain を並列化していると思っていても、内部的な非同期 API 処理がバックグラウンドで並列化されるだけで、chain 部分のコールバックは今まで見てきたようにマイクロタスクとしてイベントループで連鎖的に並行 (concurrent) で処理されます。

```js
// 非同期 API の並列起動
fetch(urls[0]).then(response => response.text()).then(text => console.log(text));
fetch(urls[1]).then(response => response.text()).then(text => console.log(text));
fetch(urls[2]).then(response => response.text()).then(text => console.log(text));
// chain によって登録されているコールバック関数はマイクロタスクとしてイベントループで連鎖的に並行処理される
```

先程定義したような async 関数 `fetchThenConsole()` も次のように３つを順番に起動したときには、内部の await 式で一時的断中しても非同期 API がバックグラウンドで継続するために async 関数そのものが「並列化」しているように誤解してしまうことが多いですが、実際には複数の非同期 API 処理がバックグラウンドで継続し、API 本体以外の付随処理は Promise chain と同じ様に最初の await 式以降の await 式で分割された処理フローがマイクロタスクとして連鎖的にイベントループで処理されます。

```js
// async 関数を順番に起動していく(実質的には非同期 API を順番に起動)
fetchThenConsole(urls[0]); // 内部の await 式で一時中断して呼び出し元のコンテキストに制御が戻る↓
fetchThenConsole(urls[1]); // 内部の await 式で一時中断して呼び出し元のコンテキストに制御が戻る↓
fetchThenConsole(urls[2]); // 内部の await 式で一時中断して呼び出し元のコンテキストに制御が戻る↓
// 並列化しているのはあくまで内部の非同期 API
// await 式で分割された処理フローはイベントループで連鎖的に処理される
```

これも意図があって順序付けてしまいたいなら効率は悪くなりますが await 式で制御します。

非同期 API を await で実行・完了を順序付けてやるよりも、並列化することで時間短縮できる場合が多いです。順序付けて `fetch()` からなる chain を実行する `fRSequential()` と並列化する `fAParallel()` 順番に実行するコードを Chrome ブラウザ環境で考えてみます (開発者ツールでどのようなネットワーキングが起きているか確かめるため)。

```js:index.js
async function fRSeuquential(resources) {
  const results = [];
  for (let i = 0; i < resources.length; i++) {
    const texts = await fetch(resources[i]).then((response) => response.text());
    results.push(texts);
  }
  return results;
}

async function fAParallel(resources) {
  const promises = resources.map((resource) =>
    fetch(resource).then((response) => response.text())
  );
  const texts = await Promise.all(promises);
  return texts;
}

const testUrls2 = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];

(async () => {
  console.log("Start: fRSeuquentialの処理開始");
  const startSequential = Date.now();
  await fRSeuquential(testUrls2)
    .then((results) => console.log(results))
    .then(() => {
      const endSequential = Date.now();
      const time = Math.floor(endSequential - startSequential);
      console.log(`End: ${time}[ms]かかりました`);
    });
  console.log("fRSeuquentialが終了しました");

  console.log("Start: fAParallelの処理開始");
  const startParallel = Date.now();
  await fAParallel(testUrls2)
    .then((results) => console.log(results))
    .then(() => {
      const endParallel = Date.now();
      const time = Math.floor(endParallel - startParallel);
      console.log(`End: ${time}[ms]かかりました`);
    });
  console.log("fAParallelが終了しました");
})();
```

適当な HTML ファイルのスクリプトタグで `src="./index.js"` を指定した上でローカルサーバーを立てて読み込ませます。開発者ツール (dev tool) のネットワークの項目を確認するとブラウザ環境では大体１つの fetch に次のような時間がかかっているおり、待機時間がかなりを占めていることが分かります。

![time_to_fetch](/images/js-async/img_devtool_time_to_fetch.jpg)

そして順序付けて行ったデータフェッチと並列化したデータフェッチでは、並列化している方の効率が良くなっていることが分かりやすく図示されていますね。

![prallel_fetch](/images/js-async/img_devtool_parallel_fetch.jpg)

`fetch()` の例で見てきたように、非同期 API つまり **Non-blocking である API ならすべて並列化できます**。並列化でどの程度の速度がでるか (どの程度効率的にできるか) は環境や並列化する処理の数によるところが大きいとは思いますが、理論的には非同期 API を次のように１つずつ起動することで並列化となります。『[同期 API とブロッキング](f-epasync-synchronus-apis)』のチャプターで見た `Deno.writeTextFile()` などは Promise を返す非同期 API なので次のように並列化できます。ただしパスは競合しないように異なるものを指定しています。

```js
// 並列起動(実際には１つずつ起動しているが起動後にバックグラウンドで処理が時間的に継続するので並列化となる)
Deno.writeTextFile(path1, inputData); // Non-blocking
Deno.writeTextFile(path2, inputData); // Non-blocking
Deno.writeTextFile(path3, inputData); // Non-blocking

// ブロッキングしないので API 処理に関与しないこの後の処理は関係なく実行処理できる...
```

`await Promise.all([...promises])` はあくまで「すべての並列化処理が完了してから次に行いたい処理を行う」ということを指示しているだけです。

```js
(async () => {
  await Promise.all([
    Deno.writeTextFile(path1, inputData), // Promiseが返る
    Deno.writeTextFile(path2, inputData), // Promiseが返る
    Deno.writeTextFile(path3, inputData), // Promiseが返る
  ]);
  // 次の行はすべての並列化した Promise 処理が完了してから行いたいから await 式の評価を行った
  console.log("⭐️ 並列化した非同期 API 処理がすべて終了しました");
})();
```

同期 API (Blocking API) ならブロッキングを意図的に起こすので、もちろん並列起動はできずに１つずつしか実行・完了できません。名前の最後に `Sync` が付いている Deno の同期 API である `Deno.writeTextFileSync()` は１つずつ完了を待って次の行を実行します。

```js
// 同期 API (Blocking API) なので意図的に一つのことしかできないようにしている
Deno.writeTextFileSync(path1, inputData);
// 上の処理が完了してから次の処理を実行する
Deno.writeTextFileSync(path2, inputData);
// 上の処理が完了してから次の処理を実行する
Deno.writeTextFileSync(path3, inputData);

// ブロッキングするのでこの後の処理は上の処理が終わらない限り実行処理できない...
```

処理の流れは分かりやすいですが、１つずつ完了を待って処理するのであきらかに効率が悪いです。非同期 API のように並列化できないことよりも、**同期 API を使っている間はメインスレッドで別の JavaScript コードを実行できない** ということがデメリットとしては大きそうです。

## Promise Combinator による「合成」

並列化や順序付けにまつわる話を見てきましたが、ここで Promise の静的メソッドについての基本的な話に戻ります。

`Promise.all()` や `Promise.allSettled()` などの Promise オブジェクトの静的メソッドは Promise インスタンスの配列を引数に受け取ります (厳密には Iterable オブジェクト)。そして複数の Promise 処理を「合成」することから、"**Promise combinator**" と呼ばれています。

https://v8.dev/features/promise-combinators

最も分かりやすいものが `Promise.all()` です。この静的メソッドは Promise インスタンスの配列を受けとり、自身も Promise インスタンスを返します。配列内のすべての Promise インスタンスが履行状態となった場合に `Promise.all()` から返る Promise インスタンスも履行状態となります。

```js
Promise.all([pormise1, promise2, promise3])
  .then([value1, value2, value3] => {
    //  ^^^^^^^^^^^^^^^^^^^^^^^ 引数での配列の分割代入
    console.log({ value1 });
    console.log({ value2 });
    console.log({ value3 });
  })
```

`Promise.all()` の返り値となる Promise インスタンスには引数として渡した配列内の複数 Promise インスタンスの結果、つまり履行値の配列が格納されています。分かりやすいように次のように最初から履行している Promise インスタンスを `Promise.resolve()` で作成してみます。

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

`Promise.all()` からは返ってくるのは Promise インスタンスなのでもちろん chain できます。そしてその Promise の履行値は `[ 1, 2, 3 ]` という数値の配列であることが分かります。`then()` で chain した際に入力として受け取る `data` は `Promise.all()` から返る Promise インスタンスの履行値です。

こんな感じで複数の Promise インスタンスをまとめ上げて、すべてが履行状態になった後でなにかしたいと言う時にはこの `Promise.all()` メソッドを使用します。このような使い方がいわゆる「Promise の合成 (Composition)」と呼ばれる行為です。

「合成 (Composition)」という用語ですが、実際にやっていることは複数の Promise インスタンスを別の Promise インスタンスで包みこみ、内部の Promise インスタンスの状態によって包んでいる Promise インスタンスの状態が履行か拒否に決まるだけです。「合成」という言葉にそこまで特別な意味はないので、あまりこだわる必要はないです。逆にこだわりすぎると「直列」などの言葉に変に惑わされることになるので、単に複数の Promise 処理が終わってからなにか関連する処理がしたいので実用上は「**複数の Promise 処理をまとめあげる**」程度の認識で十分だと思います。

ただし、実際に **まとめあげるのは Promise インスタンスです**。これを認識しておくことで、配列に何を渡すか混乱せずにすみます。以下のように異なるタイプの処理でもすべて Promise インスタンスが返っているものしか渡していません。

```js
Promise.all([
  promiseAPI(),  // Promise が返る
  fetch(url),    // Promise が返る
  asyncFunc(),   // Promise が返る
  (async () => {})(), // Promise が返る
  promiseChain(),     // Promise が返る
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

async/await で書くなら次のような感じで、「複数の Promise 処理がすべて履行してから `console.log()` を行いたい」という意図で `Promise.all()` を await します。

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

await 式の特徴である Promise インスタンスを評価して値を取り出すという性質を利用すれば複数のリクエストのレスポンスをまとめて抽出できます。

```js
const reponses = await Promise.all([
  fetch(urls[0]),
  fetch(urls[1]),
  fetch(urls[2]),
]);
```

あるいは分割代入でそれぞれ抽出します。

```js
const [res1, res2, res3] = await Promise.all([
  fetch(urls[0]),
  fetch(urls[1]),
  fetch(urls[2]),
]);
```

あるいは chain させたり async 関数に一連の処理を閉じ込めて抽象化した操作からデータを引き出すこともできます。

```js
const [text, json] = await Promise.all([
  fetch(urls[0]).then(res => response.text()),
  returnJson(),
]);
```

## Promise Combinator の種類

Promise combinator は ES2022 の時点ですでに４つの種類が存在しています。４つは便宜的に対応関係にあると考えることができるので次のように分けられます。

- `Promise.allSettled()` vs `Promise.all()`
- `Promise.race()` vs `Promise.any()`

MDN のドキュメントには以下のように Promise の静的メソッドやプロトタイプメソッドが網羅されています。

![mdn での説明](/images/js-async/img_promise-methods-in-mdn.png =550x)*MDN のドキュメントより*

Promise combinator は `Promise.resolve()` や `Promise.reject()` と同じくビルトインオブジェクトである Promise の静的メソッドです。

引数となる複数の Promise 処理に対して何をしたいのかという目的によってこれらのメソッドを使い分けます。この４つの静的メソッドはすべて複数の Promise 処理をまとめあげて並列し、**返り値として Promise インスタンスが返ってきます**。ただし、引数として渡す配列内の Promise の状態によって返り値の Promise インスタンスの状態がどうなるか変わります。

### Promise.allSettled vs Promise.all

複数の Promise 処理すべてに興味があり、Promise 処理同士に依存関係があり１つも失敗したくないという場合には、`Promise.all()` を使用します。`fetch()` で考えるとすべてのリソースフェッチに成功する必要があるなら、この `Promise.all()` を利用します。複数の Promise 処理の成功や失敗に興味がなく、とりあえずすべての処理を行ってからなにかしたいという場合には `Promise.allSettled()` を使用します。複数の Promise 処理がすべて Settled になったら返り値の Promise インスタンスが履行状態になります。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all

『[古い非同期 API を Promise でラップする](12-epasync-wrapping-macrotask)』のチャプターで解説した Promisification した `setTimeout()` によるタイマーを使って比較してみましょう。

次の `xTimer()` という関数は、デフォルトで履行状態になる Promise インスタンスを返しますが、`success` に `false` を渡すと拒否状態の Promise インスタンスが返ります。

```js:xTimer.js
export default function xTimer(time, success = true) {
  return new Promise((resolve, rejetct) =>
    setTimeout(() => {
      if (success) {
        resolve(`${time}msで履行`);
      } else {
        rejetct(`${time}msで拒否`);
      }
    }, time)
  );
}
```

まずは `Promise.allSettled()` ですが、引数に Promise インスタンスの配列を受け取り、それらのインスタンスの状態に関わらず返ってくる Promise インスタンスは履行状態となります。

```js:relAllSettled.js
import xTimer from "./xTimer.js";

const sTimes = {
  id1: { time: 200, success: false },
  id2: { time: 100, success: false },
  id3: { time: 300, success: true },
};
const promises = Object.keys(sTimes).map((id) => {
  return xTimer(sTimes[id].time, sTimes[id].success);
});
Promise.allSettled(promises)
  .then((vals) => console.log(vals))
  .catch((err) => console.error(err))
  .finally(() => console.log("処理終了"));

/*
❯ deno run relAllSettled.js
[
  { status: "rejected", reason: "200msで拒否" },
  { status: "rejected", reason: "100msで拒否" },
  { status: "fulfilled", value: "300msで履行" }
]
処理終了
*/
```

`Promise.allSettled()` から返ってくる Promise インスタンスは次のような値を内部にもちます。await 式で評価したり、then のコールバックの入力となる値はこのようになっています。

```js
[
  { status: "rejected", reason: "200msで拒否" },
  { status: "rejected", reason: "100msで拒否" },
  { status: "fulfilled", value: "300msで履行" }
]
```

ただし、`Promise.allSettled()` から返る Promise インスタンスが Settled になるためには引数の配列に渡す Promise インスタンスは Settled となる必要があります。次のような Promise インスタンスを渡してしまうと `Promise.allSettled()` は解決せず Settled となりません。配列内のすべての Promise インスタンスが履行または拒否状態となる必要があります。

```js
const p = new Promise(() => {});
// Settled にならず永遠に Pending である Promise インスタンス
```

次のように１つでもそのような Promise があると `Promise.all()` は Settled にならないので chain しているコールバックは実行できません。

```js
Promise.allSettled([
  new Promise(() => {}), // 永遠に Pending
  Promise.resolve(1),
]).then(data => console.log(data));
// 実行しても何も起きないコード
```

一方、`Promise.all()` も Promise の配列を引数として受け取ります。配列内のすべての Promise インスタンスが履行状態になると、返り値の Promise インスタンスも履行状態となります。配列内の Promise インスタンスが１つでも拒否状態になると返り値の Promise インスタンスも拒否状態となります。

```js:relAll.js
import xTimer from './xTimer.js';

const sTimes = {
  id1: { time: 200, success: false },
  id2: { time: 100, success: false },
  id3: { time: 300, success: true },
};
const promises = Object.keys(sTimes).map((id) => {
  return xTimer(sTimes[id].time, sTimes[id].success);
});
Promise.all(promises)
  .then((vals) => console.log(vals))
  .catch((err) => console.error(err))
  .finally(() => console.log("処理終了"));

/*
❯ deno run relAll.js
100msで拒否
処理終了
*/
```

次のように１つでも Settled とならないような Promise があると `Promise.all()` から返る Promise インスタンスは履行・拒否のどちらの状態にも移行しません。

```js
Promise.all([
  new Promise(() => {}), // 永遠に Pending
  Promise.resolve(1),
]).then(data => console.log(data));
// 実行しても何も起きないコード
```

ということで、使い分け次のようなケースとなります。

- `Promise.allSettled()` を使用するのは、複数の非同期処理の絡む作業が互いに依存せずに正常に完了する場合や各プロミスの結果を常に知りたい場合に使用。
- `Promise.all()` を使用するのは、複数の非同期処理の絡む作業が互いに依存している場合やタスクのいずれかが拒否されたときにすぐに拒否したい場合。

### Promise.race vs Promise.any

複数の Promise 処理すべてには興味がなく、対象となるものの内のどれか１つの Promise インスタンスが Settled になるかどうかに興味があり、履行か拒否には興味がない場合には `Promise.race()` を使用します。

一方、履行されたものに興味があり、最初に履行したものを取り出したい場合には `Promise.any()` を使用します。

この２つのメソッドも複数の Promise 処理を合成して並列化しますが、１つの処理が条件を満たした時点で他の処理については考える必要がなくなり、完了、つまり返り値の Promise インスタンスが履行状態となります。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/any

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/race

`Promise.race()` も Promise インスタンスの配列を引数にとり、返り値となる Promise インスタンスは配列内の一番最初に完了した、つまり Settled となった Promise インスタンスと同じ状態になります。履行状態なら返り値のインスタンスも履行状態となり、拒否状態なら返り値のインスタンスも拒否状態となります。

`Promise.all()` などと同じく複数の `xTimer()` を並列化させます。タイマーの id2 が 100 ミリ秒で最も早く Settled となります。

状態は `success` が `false` なので拒否状態で完了して、`Promise.race()` から返る Promise インスタンスも拒否状態となります。

```js:relRace.js
import xTimer from "./xTimer.js";

const sTimes = {
  id1: { time: 200, success: false },
  id2: { time: 100, success: false },
  id3: { time: 300, success: true },
};
const promises = Object.keys(sTimes).map((id) => {
  return xTimer(sTimes[id].time, sTimes[id].success);
});
Promise.race(promises)
  .then((val) => console.log(val))
  .catch((err) => console.error(err))
  .finally(() => console.log("処理終了"));

/*
❯ deno run relRace.js
100msで拒否
処理終了
*/
```

一方、`Promise.any()` も Promise インスタンスの配列を引数にとり、一番最初に履行状態となった Promise インスタンスの状態に連鎖して `Promise.any()` から返る Promise インスタンスも履行状態となります。

```js:relAny.js
import xTimer from "./xTimer.js";

const sTimes = {
  id1: { time: 200, success: false },
  id2: { time: 100, success: false },
  id3: { time: 300, success: true },
};
const promises = Object.keys(sTimes).map((id) => {
  return xTimer(sTimes[id].time, sTimes[id].success);
});
Promise.any(promises)
  .then((val) => console.log(val))
  .catch((err) => console.error(err))
  .finally(() => console.log("処理終了"));

/*
❯ deno run relAny.js
300msで履行
処理終了
*/
```
