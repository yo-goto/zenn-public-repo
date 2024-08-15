---
title: "Promise コンストラクタと Executor 関数"
cssclass: zenn
date: 2022-04-21
modified: 2024-08-14
AutoNoteMover: disable
tags: type/zenn/book, JavaScript/async
aliases: Promise本『Promise コンストラクタと Executor 関数』
---

## このチャプターについて

『[Promise の基本概念](a-epasync-promise-basic-concept)』のチャプターでは抽象的な概念についてしか触れていなかったので、ここからはコード上での Promise について、インスタンスの作成方法などを通じて触れていきます。

## Promise オブジェクト

まず、Promise とは「**非同期処理の結果を表現するビルトインオブジェクト**」であり、モダンな非同期処理ではこの Promise オブジェクトを介して非同期処理を行うのがベターです。

Promise オブジェクトは `fetch()` といった非同期 API (ECMAScript の一部ではなくブラウザやランタイムの環境が提供する機能)の処理の結果として返されるパターンが多いですが、Promise そのものは**ビルトインオブジェクト**であり、ECMAScript (JavaScript の言語コア) の一部であることを忘れないようにしてください。

また、"Promise API" という言葉がありますが、これは Promise インスタンスを返すタイプの非同期 API である "Promise-based API" のことを指しており、Promise 自体が API であるわけではないので注意してください。他の解説によっては、Promise の静的メソッドである `Promise.all()` などを指している場合もあります。

:::message alert
この本ではそうしませんが、他の解説では Promise API といっているときは JavaScript エンジンの文脈で話していることがあり、その場合に `Promise.all` や `Promise.allSettled`、また `Promise(executor)` コンストラクタなどについて Promise API と言っている場合あるので注意してください。
:::

## Promise コンストラクタ

コード上で `Promise()` はコンストラクタ関数であり、`new` 演算子と併用して使用することで Promise オブジェクト(Promise インスタンス)を生成できます。Promise オブジェクトを作成する際には、`Promise()` コンストラクタには **Executor関数** と呼ばれるコールバックを引数として渡します。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise

`deno` コマンドで [REPL 環境](https://ja.wikipedia.org/wiki/REPL) を立ち上げて少しテストしてみると次のようになります。

```sh
❯ deno
Deno 1.20.4
exit using ctrl+d or close()
# deno REPL で見てみる
> const promise = new Promise(resolve => resolve(1))
undefined
> typeof promise
"object"
> promise instanceof Promise
true
# Promise.resolve() でつくっても同じ
> const np = Promise.resolve(1);
undefined
> np instanceof Promise
true
```

以降、各 Promise オブジェクトについて、コンストラクタ関数から作成されることや、関数やメソッドから返ってくるということを意識するために "Promise インスタンス" という言葉を多用していきます。

Promise インスタンスの作成は `new Promise(executor)` が基本形です。コールバック関数として引数に渡す `executor` 自身は引数を２つ受け取ります。

次のコードでは `executor` がコールバック関数であることに注目するため、あえて Promise コンストラクタの外で定義してみると次のようになります。

```js
function executor(resolve, reject) {
  // 以下の処理は適当に形式をあわせて書いているだけです
  if (Math.random() < 0.5) { // 適当な条件
    resolve("Promise履行時の値");
    // resolve 関数は Promise インスタンスを履行(Fulfilled)状態にしたい時に呼び出す
  } else {
    reject("Promise拒否時の理由");
    // reject 関数は Promise インスタンスを拒否(Rejected)状態にしたい時に呼び出す
  }
}

// あえてコールバックをコンストラクタの外で定義している
const promise = new Promise(executor);
```

:::message alert
『[非同期 API と環境](f-epasync-asynchronous-apis)』のチャプターで解説したとおり、非同期 API には種類があり、`setTimeout()` は**タスクベースの非同期 API** です。

基本的に非同期処理の解説では上記のコンストラクタ関数内で `setTimeout()` などを使っていくことが一般的だと思いますが、タスクベースの非同期 API と本来的にはマイクロタスクベースの非同期 API のために存在する Promise を絡めて考えてしまうと混乱することになるので、最初はあえて使わずに説明してきます。ちなみに `console.log()` 自体も Web API ですがこちらは非同期処理とは関係ないので使用します。

実は Promise コンストラクタ内で `setTimeout()` などのタスクベースの非同期 API を利用することで、後のチャプターで解説する "[Promisification](12-epasync-wrapping-macrotask)" という手法になるのですが、これを最初に知ってしまうと学習を進めてく過程で**混乱する可能性が高いので**注意してください。
:::

JavaScript では「関数は値」なのでこのように関数を他の値のように引数として渡すことができます。「コールバック関数」はこのように他の関数に引数として渡される関数のことを指します。

`Promise()` コンストラクタの引数として渡されるコールバック関数である `executor` の引数である `resolve` と  `reject` もコールバック関数です。慣習的に `resolve` や `reject` となっていますが実際には名前は何でも OK です。`executor` の中において、`resolve()` 関数は Promise インスタンスを履行(Fulfilled)状態にしたいという時に呼び出し、`reject()` は Promise インスタンスを拒否(Rejected)状態にしたいという時に呼び出します。

この２つの関数はクセがあるので注意してください。(後述)。

`executor` は基本的には無名関数(匿名関数)でアロー関数の省略形などがよく使われるので注意してください。ここから、徐々に変形していきます。

まずは、`Promise()` コンストラクタの中でコールバック関数を無名関数として定義してみます。

```js
const promise = new Promise(function (resolve, reject) {
  if (Math.random() < 0.5) {
    resolve("Promise履行時の値");
  } else {
    reject("Promise拒否時の理由");
  }
});
```

次はコールバック関数をアロー関数に変形します。この形式が多くの解説記事で見られるような一般的な形になります。

```js
const promise = new Promise((resolve, reject) => {
  if (Math.random() < 0.5) {
    resolve("Promise履行時の値");
  } else {
    reject("Promise拒否時の理由");
  }
});
```

:::message
アロー関数についての補足をこのページの下で行っています。
:::

`executor` 関数の第二引数である `reject` は**省略可能なので書かない場合もよくあります**。拒否状態などを気にせず、履行状態のみを考えます(実際には、`executor` 関数の中でエラーが発生すると Promise インスタンスは自動的に拒否(Rejected)状態へと移行します)。

```js
// reject 関数を省略
const promise = new Promise((resolve) => {
  resolve("Promise履行時の値");
});
```

さらにアロー関数は引数が１つのときにカッコを省略できるので次のように文字数を少なくして書けます。

```js
const promise = new Promise(resolve => {
  resolve("Promise履行時の値");
});
```

`resolve()` 関数の名前は何でも良かったので、名前を短くして文字数をもっと減らしてみます。

```js
const promise = new Promise(res => {
  res("Promise履行時の値");
});
```

これで最初の書き方よりもかなり楽に書けていることが分かります。さすがに `res` というような書き方はあまりしないと思いますが、この先にでてくるものとの差異を明らかにするため、わざとやっています。

アロー関数の `return` 省略を使って、もっと文字数を減らしてみます。

```js
const promise = new Promise(res => res("Promise履行時の値"));
```

これはアロー関数の省略形の中でも最も短い形式となっています。ですが、実は上のコードは `res => {return res("Promise履行時の値")}` の省略形となっています。`return` に注意してください。

アロー関数の省略は以下のようにでき、下の３つのコートはすべて等価です。従って、`return` を上のように省略できています。

```js
(a) => {
  return a + 100;
}
// 等価
(a) => a + 100;
// 等価
a => a + 100;
```

ここまで、`new Promise(executor)` というコードをなるべく短く書けるように省略してきましたが、実は上記のコードと同じようなことを `Promise()` コンストラクタ関数を使用せずに `Promise.resolve()` という Promise の**静的メソッド**(static method)を使って実現できます。

```js
const promise1 = Promise.resolve("Promise履行時の値");
// この２つは大体同じ
const promise2 = new Promise(res => res("Promise履行時の値"));
```

`executor` 関数の引数である `res` 関数と静的メソッドである `Promise.resolve()` は別物であることに注目してください。

:::message
`Promise.resolve()` と `resolve()` 関数(上の例では `res()`)が別モノであるということが実は async/await の挙動などで関わってくるので注意してください。特に `resolve()` 関数はかなり特殊で再帰性などの特徴が関わります。つまり、**完全に等価ではない**ので、「大体同じ」として扱っています。
:::

この `Promise.resolve()` は最も文字数が少なく書けるので、Promise オブジェクトの初期化やテストコードを書く際に活用できる便利なショートカットとして覚えてください。実際に Promise オブジェクトを作成する際には `new Promise(executor)` が基本となります。

さて、`executor` 関数の引数は２つありました。`resolve` (上の例では `res`) と `reject` です。`reject` 自体は第二引数であり、省略できたので上記のように短く書くために無視してきましたが、これでは不公平なので `reject` についても省略形で書けるようにします。次のコードでは、`executor` 関数の中で `reject()` 関数のみを書いて Promise インスタンスを拒否状態にしています。

```js
const promise = new Promise((resolve, reject) => {
  reject("Promise拒否時の理由");
});
```

`reject` がそのまま省略可能であったのに対して、`reject` を使いたい場合に `resolve` 関数が省略できないのは第一引数だからです。**第一引数がないのに第二引数は書けません**。

ただし、上述の通り `resolve` と `reject` について名前は何でも良いのでそれを利用して次のように字数が減るように書くことができます。

```js
const promise = new Promise((_, rej) => {
  rej("Promise拒否時の理由");
});
```

アンダースコア(`_`)という記号によってあえて使わない `resolve` 関数の名前を最も短い一文字にしています。この書き方は別に推奨というわけではないですが、こうやってできるということを認識するために書いています。

さて、`resolve` でやったようにアロー関数のさらなる省略形でもっと文字数を減らしてみます。

```js
const promise1 = new Promise((_, rej) => rej("Promise拒否時の理由"));
// ２つは等価
const promise2 = new Promise((_, rej) => {
  return rej("Promise拒否時の理由");
});
```

かなり文字数が減りましたね。予想できると思いますが、これらのコードと同じことを Promise オブジェクトの静的メソッドである `Promise.reject()` を利用してもっと短く書くことが可能です。

```js
const promise1 = new Promise((_, rej) => rej("Promise拒否時の理由"));
// この２つは大体同じ
const promise2 = Promise.reject("Promise拒否時の理由");
```

この `Promise.reject()` も初期化やテストなどで活用できる便利なショートカットとして使えますが、基本は `new Promise(executor)` です。

:::message
実際には、`resolve` と `reject` の両方とも `executor` の中で使用しないなら両方とも省略することが可能です。そういった場面は特に意味がないので使わないと思いますが、一応できるということを示しておきます。

```js
// executorBothEmit.js
// resolve も reject も引数として渡さないで省略する
const promise = new Promise(() => {
  console.log("hello zenn");
});

promise.then(() => console.log("not executed"));
console.log("Promise status:", promise);
```

これを実行すると以下の出力を得ます。

```sh
❯ deno run executorBothEmit.js
hello zenn
Promise status: Promise { <pending> }
```

`resolve` や `reject` を呼び出さないので、Promise インスタンスは永遠に待機(Pending)状態であり、`then()` メソッドで登録しておいたコールバック関数は実行されませんし、エラーも捕捉されません。
:::

## 関数式とアロー関数の補足

アロー関数について触れましたが、少し補足します。

まずアロー関数の前に通常の関数宣言とは別の方法で関数を定義する「関数式」について触れておきます。`function` キーワードを使用した関数宣言は次のようになります。

```js
function myFunc() {
  console.log("arg1:", arg1);
  console.log("arg2:", arg2);
}

myFunc("hello", "zenn");
/* 出力
 * arg1: hello
 * arg2: zenn
 */
```

通常の関数宣言は上書きでてきしまうので、`const` 宣言を使った変数への代入という形で関数の定義を行います。これが「関数式」です。関数式の使い方は次のようになります。

```js
// helloZenn.js
const myFunc = function (arg1, arg2) {
  console.log("arg1:", arg1);
  console.log("arg2:", arg1);
};

myFunc("hello", "zenn");
/* 出力
 * arg1: hello
 * arg2: zenn
 */
```

関数を変数 `myFunc` に代入しているわけです。呼び出し時には `()` をつけることで関数の実行となります。`()` をつけなければただの値として評価されます。例えば次のコードでは、`()` をつけずに `console.log()` の引数として渡しています。

```js
// helloZenn-value.js
const myFunc = function (arg1, arg2) {
  console.log("arg1:", arg1);
  console.log("arg2:", arg1);
};

console.log(myFunc); // => [Function: myFunc]
```

これを実行すると次の出力を得ます。

```sh
❯ deno run helloZenn-value.js
[Function: myFunc]
```

このように、JavaScript では「**関数は値**」であり、関数を他の値と同じように変数に代入したりコールバック関数として引数に渡すことができるようになっています。コールバック関数として引数に渡す場合は `()` をつけずに渡します。`()` をつけてしまった場合には関数実行の結果としてその関数の返却する返り値が引数として渡されるので注意してください。

関数式で関数の定義をするメリットとしてははじめに言ったように `const` 宣言で **関数の上書きを出来ないようにしています**。

:::message
**関数式と関数宣言の使い分け**

関数宣言では、`function` キーワードが頭にあるので見やすかったり、巻き上げ(hoisting)があるためファイル全体で定義した関数が使えるという利点があります。その一方、関数式は定義した行以降でしかその関数を使えず、他人が見て分かりづらい場合もありますが、グローバルスコープを汚染することなくコールバックなどで使い捨てることなどができます。結局は両方を使い分けるのが良さそうです。

参考: [When to use a function declaration vs. a function expression](https://www.freecodecamp.org/news/when-to-use-a-function-declarations-vs-a-function-expression-70f15152a0a0/)
:::

ではここでアロー関数を使って関数式を書き換えてみます。`function` キーワードを取り払って、アロー記号 `=>` をつけます。

```js
// helloZenn-allow.js
const myFunc = (arg1, arg2) => {
  console.log("arg1:", arg1);
  console.log("arg2:", arg1);
};

myFunc("hello", "zenn");
/* 出力
 * arg1: hello
 * arg2: zenn
 */
```

これだけです。さらに関数内のコードが単一の式の場合は `return` を省略できます。このようなアロー関数の省略形を使って書くと次のように書けます。

```js
// 全部同じ意味
const increment0 = (num) => {
  return num + 1;
}
// 単一の式のみなので `return` を省略できる
const increment1 = (num) => num + 1;
// 引数が１つのみなので括弧を省略できる
const increment2 = num => num + 1;
```

ただし、次のコードのように返り値としてオブジェクトリテラルを返す場合には注意が必要です。

```js
const returnObject = () => {
  return { key: "value" };
}
```

アロー関数の省略形として次の書き方は誤りです。

```js
// この書き方は誤り
const returnObject = () => { key: "value" };
// こうなってしまっている
const returnObject = () => {
  key: "value"
};
```

この場合はオブジェクトリテラルを `()` でくくることで `return` を省略した形で書くことができます。

```js
// 正しい書き方
const returnObjectOmit = () => ({ key: "value" });
```

参考
https://typescriptbook.jp/reference/functions/arrow-functions

:::message
余談ですが、アロー関数の省略形を知っておくと JSX でのアロー関数について理解できます。

React Component をアロー関数で書くと次のようになります。

```jsx
const MyComponent = () => {
  return <h1>Hello, world</h1>;
};
```

これをアロー関数の省略形で `return` を省略できます。

```jsx
const MyComponent = () => <h1>Hello, world</h1>;
```

ただしネストがある場合には `()` で囲んであげます。

```jsx
const MyComponent = () => (
  <div>
    <h1>Hello, world</h1>
    <h1>Hello, Zenn</h1>
  </div>
);
// これは次と同じ意味
const MyComponentReturn = () => {
  return (
    <div>
      <h1>Hello, world</h1>
      <h1>Hello, Zenn</h1>
    </div>
  );
};
```

この書き方はイテレーションを使用して子要素をネストさせるパターンなどでよく使用されます。

```jsx
function Home() {
  const name = ["Tarou", "Hanako", "Kaoru"];
  return (
    <div className="container">
      <div className="mylist">
        <ul>
          {names.map(name => (
              <li key={name}>{name}</li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```
:::

あとはコールバック関数にアロー関数を渡す際などでたまに見かける書き方として、引数をあえて「使用しないアンダースコア１つ」にして `()` を書かずに文字数を少なくするというものがあります。

```js
const zeroParamFn = _ => {
  console.log("使わない引数をあえて１個のアンダースコアにする");
};
Promise.resolve()
  .then(_ => console.log("コールバックでたまに見かける書き方"));
setTimeout(_ => {
  console.log("推奨でないという話も聞く");
}, 3000);
```

これは、ただの「書き方のスタイル」で引数が０個のときのアロー関数の省略形 `() => {...}` よりも文字数が少なく出来るというだけのものです。

参考
https://stackoverflow.com/questions/41085189/using-underscore-variable-with-arrow-functions-in-es6-typescript

アロー関数と関数式の説明は以上になります。
実際にはアロー関数を使うことで通常の関数宣言や関数式と異なる挙動がいくつかありますが、その説明は別の記事や本で補足してください。

https://qiita.com/suin/items/a44825d253d023e31e4d

https://typescriptbook.jp/reference/functions/function-expression-vs-arrow-functions

以降、Promise インスタンスを返す関数や async/await を使った非同期関数などが登場しますが、このアロー関数を使って定義する場合があるので注意してください。
