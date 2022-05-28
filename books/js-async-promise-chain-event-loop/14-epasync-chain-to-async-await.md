---
title: "Promise チェーンから非同期関数へ"
---

# このチャプターについて
Prosmise チェーンが分かったことでようやく async/await に入ることができます。このチャプターでは、async/await を学習する上での注意点や意識すべき点などについて解説します。

:::message
イベントループと絡めた具体的な挙動については次のチャプターを確認してください。このチャプターで理解しにくい点がある場合は次のチャプターの内容を見てから戻ってくるのもいいかもしれません。
:::

# Async function
まずは注意点として、非同期処理の主役は async/await ではなく、**あくまで Promise が主役である**ことを忘れないでください。async/await を使ってできることは Promise による非同期処理の利便性を高める(具体的には制御フローが見やすくなったり、let 宣言や for ループ、try/catch などが使えるようになる)ことです。

そういう訳で、初学者にとって「**async/await によって Promise を意識することなく非同期処理ができるようになった**」という文言はトラップとなる可能性が実は高いです。むしろ、Promise を扱っていることを意識しておかないと後々混乱しやすい構文であると個人的には感じています。

:::message
asynchronous の発音記号は `/eɪsɪ́ŋkr(ə)nəs/` ですので、`async` の読み方は日本語だと"エイシンク"になるはずですが、アシンクでもエイシンクでも好きな方を使ってください。
:::

まず、非同期関数(Async function)は**どんな時も必ず Promise インスタンスを返す関数**です。

非同期関数の定義には `async` キーワードを使用します。そして、Promise インスタンスを返すわけですから、Promise チェーンが可能です。

```js:通常の関数定義
// returnStringInAsync.js
async function alwaysReturnPromise() {
  return "👾 Promise でラップされる文字列";
}

alwaysReturnPromise()
  .then((value) => console.log("🍓 履行値:", value))
  .catch((reason) => console.error("🥦 拒否理由:", reason))
  .finally(() => console.log("👍 チェーン最後に実行"));
```

ちゃんと Promise チェーンが出来ていますね。

```sh:実行結果
# V8 エンジンで実行してみる
❯ v8 returnStringInAsync.js
🍓 履行値: 👾 Promise でラップされる文字列
👍 チェーン最後に実行
```

アロー関数だとこんな感じになります。

```js:アロー関数での定義
// returnNothingInAsync.js
const alwaysReturnPromise = async () => {
  // 何もしない
};

alwaysReturnPromise()
  .then((value) => console.log("🍓 履行値:", value)) // undefined
  .catch((reason) => console.error("🥦 拒否理由:", reason))
  .finally(() => console.log("👍 チェーン最後に実行"));
```

Async function 内部で何も `return` しなくても、必ず Promise インスタンスが返ってきます。この場合、返ってくる Promise インスタンスは履行値が `undefined` の履行状態になります。

```sh:実行結果
❯ v8 returnNothingInAsync.js
🍓 履行値: undefined
👍 チェーン最後に実行
```

# await 式

非同期関数では内部で `await` 式を使って「**非同期処理の完了を待つ**」ことができます。この「待つ」ですが、非常に混乱させるワードなので注意してください。

例えば、お馴染みの非同期 API `fetch()` は Promise インスタンスを返してきましたね。`fetch()` は引数に渡した URL からデータを取得してくれます。ですが、ネットワーキングでリクエストを投げてレスポンスを受け取るまでには時間がかかります。通常は、こんな時間のかかる処理をメインスレッドで行っていたら無駄な待ち時間が発生してしまいますね。というわけで、データ取得は API を介してその作業を委任された環境(environment)がバックグラウンドで行ってくれます。

:::message
環境と非同期 API についての話は『非同期 API と環境』のチャプターで解説したので、詳しくはそちらを参照してください。
:::

とはいえ、`const response = await fetch(url)` の場合は非同期処理の完了を待つというよりも、環境に委任した並列作業を待つという感じになるので、「非同期処理を待つ」とは少し違った印象を受けるのが自然ですね。ただし、`const text = await fetch(url).then((res) => res.text())` というような場合は Promise チェーンで最終的に返ってくる Promise インスタンスが解決されるの待つので、明らかに「非同期処理を待つ」と言えます。

```js
async function test(url) {
  const response = await fetch(url);
  // 環境に委任した並列作業の完了を待つ
  const text = await fetch(url).then(res => res.text());
  // 環境に委任した並列作業の完了後にメインスレッドでデータと共に通知させた非同期のコールバック関数の完了を待つ
}
```

非同期処理を待つのか、非同期 API の完了を待つのかで印象がかなり変わってくるため、非同期の学習ではここで勘違いや混乱が起きることが多いです(個人的にはそうでした)。

そういう訳で、await 式の捉え方は「非同期処理の完了を待つ」というよりも、「Promise インスタンスを評価して値を取り出す」というようにした方がよいです。

基本的には await 式は Promise インスタンスを評価して、その履行値を取り出すという処理です(後で解説しますが、await 式では単なる数値や文字列などの値を評価することも可能です)。

むしろ、これが理解できていないと `setTimeout()` を await 式で待つというようなミスを犯してしまいます。`setTimeout()` API による何らかの処理を行ってから次の処理を行いたい場合は、以前のチャプターでみたように `setTimeout()` を Promise でラッピングする Promisification をする必要があります。

さて、話を戻しますが、`fetch()` API は Promise-based な非同期 API であり、`fetch()` メソッドからは Promise インスタンスが処理の結果として返ってきます。そして、Promise チェーンを await する場合でも、結局は Promise チェーンから返ってくる Promise インスタンスを評価していますね。

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

async/await は Promise チェーンのシンタックスシュガーであると言われます。実際にはそれ以上のもの(デバッグなどで得られる効能が Promise チェーンよりも高いなどの性質がある)ですが、現時点では Promise チェーンと完全に同等であると考えてください。

Promise チェーンも async/await もマイクロタスクを連鎖的に発行してイベントループで処理されるのは同じです。

上記の Promise チェーンは async/await で書き直すことができます。

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

async/await では try/catch が使用できるので、非同期関数内で例外を捕捉できるようにして、さらに後続のチェーンも全部書き直します。

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

# Callback hell → Promise chain → async/await
いきなり謎の変形をしてしまったのでより簡単で分かりやすい Promise チェーンから非同期関数への変形を見てみます。

次の動画の 23:36 ~ のところから視聴してみてください。Callback hell → Promise chain → async/await の変形が視覚的に示されていて変形のイメージをつかめます。

@[youtube](SrNQS8J67zc)

:::message
ここではシンプルに Promise チェーンから非同期関数へと変形でき同じものであることをイメージできることが重要です。
最初は変換を意識してみると理解しやすいですが、実際に使用する際には Promise を扱っていることだけ意識しておけばいいと思います。
:::

以下、動画で示されているコード変形について見ていきます。
Promise が無かった時代、非同期処理はコールバックベース API などで次のように、「A したら B する、B したら C する」というような逐次的な処理を行っていました。

ただし、見て分かるようにコールバックではネストが増えていくにつれてコードを把握するのが困難になります。このような形式を **Callback hell** と言います。

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

Promise の登場により、Callback hell を作らずに、非同期の逐次的な処理を Promise チェーンで実現できるようになりました。

この時点では Callback hell から Promise chain へと変わったことで、**そもそもタスクベースの逐次処理からマイクロタスクベースの逐次処理へと変わっている**ことに注意してください。`getData()` と `getMoreData()` は Promise を返してくる非同期処理であると認識してください。

```js:Promise chain
getData()
  .then(a => getMoreData(a))
  .then(b => getMoreData(b))
  .then(c => getMoreData(c))
  .then(d => getMoreData(d))
  .then(e => console.log(e));
```

さらに、ループや try/catch などの古典的で平凡な処理を非同期処理と共にでできるように **Promise のシステムに基づいた拡張的な機能**として登場した async/await により、上の Promise チェーンを下のように変形できるようになりました。async/await は Promise に基づいているので、Promise チェーンと同じ様に Promise の状態に応じて連鎖的にマイクロタスクを発行している点に注意してください。Promise チェーンの時と同じ様に、`getData()` と `getMoreData()` は Promise を返してくる非同期処理であると認識してください。

```js:async/await
(async () => {
  const a = await getData(a);
  const b = await getMoreData(b);
  const c = await getMoreData(c);
  const d = await getMoreData(d);
  const e = console.log(e);
})();
```

Callback hell で見たようにコールバックスタイルのものと Promise チェーンはタスクとマイクロタスクというように裏の仕組み自体が違うのに対して、Promise チェーンと async/await は同じく、Promise インスタンスを使用してマイクロタスクを発行するものであることを意識することが重要です。

# async/await は Promise に基づき Promise を扱う

繰り返しますが、async/await は Promise チェーンのシンタックスシュガーであるため、Promise インスタンスを取り扱っているという意識を持つことが重要です。

async/await で Promise を意識するための重要なポイントは２つあります。

- 非同期関数(async function)はどんなときでも必ず Promise インスタンスを返す
- await 式は Promise インスタンスを評価して値を取り出す(Promise インスタンス以外の値を評価する場合は一旦 Promise でラッピングして評価する)

以下の項目でそれぞれを確認しますが、原理については次のチャプターで詳しく説明します。

## 非同期関数はどんなときでも必ず Promise インスタンスを返す

非同期関数(async function)はどんなときでも必ず Promise インスタンスを返します。例えば、次のように何もしない関数を定義した場合でも、これを実行すると Promise インスタンスが返ってきます。

```js
async function empty() {}
```

非同期関数からは必ず Promise インスタンスが返ってくるので、返り値である Promise インスタンスに対して今まで通り Promise チェーンが構築できます。

```js
// 即時実行
(async function empty() {})()
  .then(data => console.log(data)); // undefined が出力される
```

この場合、非同期関数からは同期的に履行状態の Promise インスタンスが返ってくるので、チェーンされた `then()` メソッドのコールバック関数が直ちにマイクロタスクキューへとマイクロタスクとして発行されます。

非同期関数内で `return` された値が返り値の Promise インスタンスの解決値となります。返り値と非同期関数から返ってくる Promise インスタンスの状態、そのインスタンスが持つ値についての基本的な関係は以下となります。

`return` された値 | 非同期関数から返却される Promise の最終的状態 | 返却される Promise が持つ値 |
--|--|--
通常の値 | 履行状態 | 元々の値
履行状態の Promise | 履行状態 | `return` した Promise の履行値
拒否状態の Promise | 拒否状態 | `return` した Promise の拒否理由

`return` で Promise インスタンスを返した場合は Promise チェーンにおいて `then()` メソッドで Promise インスタンスを返した場合と同じ様に、非同期関数から返ってくる Promise インスタンス自体の状態も `return` した Promise インスタンスと同じになり、Promise が持つ値も `return` した Promise インスタンスの履行値や拒否理由となります。

何も `return` しない場合には、非同期関数から返ってくる Promise インスタンスは同期的に `undefined` で履行されます(なぜこうなるのかは次のチャプターで説明します)。

永遠に Pending 状態の Promise インスタンスを `return` した場合などは今までの Promise チェーンと同じように、次の `then()` メソッドのコールバックなどを実行できません(`then()` メソッドはチェーンしている Promise インスタンスの状態が遷移した時に一度だけ実行されるから、永遠に Pending 状態なら絶対に実行されない)。

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

上のコードでは、明示的にコンストラクタで作成された Promise インスタンスである `myPromise` を非同期関数内で await 式で評価して履行値である `42` という数値を取り出しています。そして、インクリメントをして `return` で返却しています。

非同期関数から返ってくるのは Promise インスタンスなので、チェーンされた `then()` メソッドのコールバック関数の入力として値を取り出すことができます(これは『then メソッドのコールバックで非同期処理』のチャプターでみましたね)。従って、上のコードでコメントしてあるようにインクリメントした値である `43` という数値を出力できます。

もちろん、Rejected 状態の Promise インスタンスを評価することも可能です。ただし、Rejected 状態の Promise インスタンスを await 式で評価した場合は非同期関数内のそれ以降の処理はスキップされます。

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

これは非同期関数内で例外を throw した場合と同じです。それ以降の処理はスキップされて、非同期関数から返ってくる Promise インスタンスは拒否状態となり、`catch()` メソッドで拒否理由と共に例外が補足されます。

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

非同期関数(async function)では古典的な例外補足の方法として try/catch/finally を使用できます。

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

このようにした場合、try/catch で例外補足するため、非同期関数から返ってくる Promise インスタンスは履行状態となります。したがって、チェーンしている `then()` メソッドのコールバックは実行でき、逆に `catch()` メソッドのコールバックは実行されません。

```sh
❯ v8 awaitRejectPromise-kai.js
👹 実行される: reason
🦄 最後に実行される
😅 これは実行される: defalut value
🦄 最後に実行される
```

非同期関数内で try/catch/finally を使えば、今までのようにチェーンする必要はなくなるので、チェーン部分はなくしても良いでしょう。

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

そのような場合に何が起きるかというと、await 式で評価する値を一旦 Promise インスタンスでラッピングしてから、値を取り出します。実はこれによって無駄なマイクロタスクと Promise インスタンスが生成されるので、本当にやる意味がないです。

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

なぜこのようなことが起きるのかというと、仕様でそうするように決まっているからとして言えません。裏でどのようなことが起きているのかは次のチャプターで確認します。

とにかく、await 式は基本的には Promise インスタンスを評価して値を取り出すものであると意識するのが重要です。

