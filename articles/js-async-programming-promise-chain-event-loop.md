---
title: "PromiseチェーンとEvent Loopで学ぶ非同期処理"
emoji: "🔗"
type: "tech"
topics: [javascript, 非同期処理]
published: false
date: 2022-04-09
url: "https://zenn.dev/estra/articles/js-async-programming-promise-chain-event-loop"
aliases: [記事_PromiseチェーンとEvent Loopで学ぶ非同期処理]
tags: [" #JavaScript/async #type/zenn  "]
---

# 追記: Book化します

:::message
内容的に追記していくことが多く、文章の量もまだ増える予定なので記事ではなく本(Book)の方が内容としては向いていると思われるので、Book の方で実験的にだしてみました。こちらの記事はしばらくしたらクローズするので注意してください。
:::

https://zenn.dev/estra/books/js-async-promise-chain-event-loop

# はじめに
https://zenn.dev/estra/articles/js-async-programming-roadmap

前回の記事でロードマップを書いたからにはアウトプットをしないといけないので、今回の記事では非同期処理の真髄と言っても過言ではない Promise チェーンと Event Loop から非同期処理の重要であるポイント(自分が重要と感じたポイント)を解説してみたいと思います。

:::message
この記事内で使用する JavaScript の実行は次の Deno ランタイム環境で行っています。

```sh
❯ deno -V
deno 1.20.4
```
:::

Promise そのものの基本はあまり説明しません。今回は例外処理なども気にせずにただ Promise チェーンを構築して Event Loop でどうなるかを予測し、JS Visualizer 9000 で実際に可視化していきます。目的は非同期処理の制御の流れを理解し、予測できるようになることです。

ただし、次のものを理解できていることを前提とします。

- Event Loop
- Call Stack

この用語や仕組みについて理解できていない場合は、JSConf EU での Philip Roberts 氏の講演動画「What the heck is the event loop anyway?」を是非見るようにしてください。

@[youtube](8aGhZQkoFbQ)

# Event Loop の基礎
まず、Event Loop ステップについて後で使用するので説明しておきます。

Event Loop は Macrotask queue と Microtask queue 内にある Macrotask/Microtask を処理するためのループアルゴリズムです。Event Loop は次に実行する Macrotask/Microtask を選択して、実行するために Call Stack へと配置します。

Event Loop の配置は次の４つのステップから構成されます。

1. スクリプトの評価: スクリプト内の関数を Call Stack が空になるまで同期的に実行する
2. Macrotask の実行: Macrotask queue から最も古いマクロタスクを選択して Call Stack が空になるまで実行します
3. すべての Microtask の実行: Microtask queue から最も古いマイクロタスクを選択して Call Statck が空になるまで実行します
4. UI レンダリング: UI をレンダリングしてステップ 2 に戻ります(ブラウザの場合)

参考
https://www.jsv9000.app

# PromiseコンストラクタとExecutor関数

まずは、Promise とは「**非同期処理の結果を表現するビルトインオブジェクト**」であり、モダンな非同期処理ではこの Promise オブジェクトを介して非同期処理を行うのがベターです。

まず、コード上で `Promise()` はコンストラクタ関数であり、`new` 演算子と併用して使用することで Prosmise オブジェクト(Promise インスタンス)を生成できます。Promsie オブジェクトを作成する際には、`Promise()` コンストラクタには **Executor関数** と呼ばれるコールバックを引数として渡します。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise

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
```

Promise インスタンスの作成は `new Promise(executor)` が基本形です。コールバックとして引数に渡す `executor` 自身は引数を２つ受け取ります。次のコードでは、`executor` がコールバック関数であることに注目するため、あえて Promise コンストラクタの外で定義してみますと次のようになります。

基本的に Promise の解説では `setTimeout()` を使っていくのが割と一般的だと思いますが、`setTimeout()` 関数は **Web API** であることを意識して**ここではあえて使わずに説明してきます**。

```js
function executor(resolve, reject) {
  // この中には `setTimeout` などの処理を書くのが一般的です
  // 以下の処理は適当に形式をあわせて書いているだけです。
  const condition = true; // 適当な条件
  const value = "Promise履行時の値";
  const reason = "Promise拒否時の理由";
  if (condition) {
    resolve(value);
    // resolve 関数は Promise インスタンスを履行(Fullfilled)状態にしたい時に呼び出す
    // このコードでは 拒否ではなく履行状態になる
  } else {
    reject(reason);
    // reject 関数は Promise インスタンスを拒否(Rejected)状態にしたい時に呼び出す
  }
}

// あえてコールバックをコンストラクタの外で定義している
const promise = new Promise(executor);
```

JavaScript では「関数は値」なのでこのように関数を他の値のように引数として渡すことができます。「コールバック関数」はこのように他の関数に引数として渡される関数のことを指します。

`Promise()` コンストラクタの引数として渡されるコールバック関数である `executor` の引数である `resolve` と  `reject` もコールバック関数です。慣習的に `resolve` や `reject` となっていますが実際には名前は何でも OK です。`executor` の中において、`resolve()` 関数は Promise インスタンスを履行(Fullfilled)状態にしたいときに呼び出し、`reject()` は Promise インスタンスを拒否(Rejected)状態にしたい時に呼び出します。

この２つの関数はクセがあるので注意します(後述)。

`executor` は基本的には無名関数でアロー関数の省略形などがよく使われるので注意してください。ここから、徐々に変形していきます。

まずは、`Promise()` コンストラクタの中で無名関数として定義してみます。

```js
const promise = new Promise(function (resolve, reject) {
  const condition = true;
  const value = "Promise履行時の値";
  const reason = "Promise拒否時の理由";
  if (condition) {
    resolve(value);
  } else {
    reject(reason);
  }
});
```

次はアロー関数に変形します。この形式が色んな記事で見られるような一般的な形になります。

```js
const promise = new Promise((resolve, reject) => {
  const condition = true;
  const value = "Promise履行時の値";
  const reason = "Promise拒否時の理由";
  if (condition) {
    resolve(value);
  } else {
    reject(reason);
  }
});
```

`executor` 関数の第二引数である `reject` は**省略可能なので書かない場合もよくあります**。拒否状態とかを気にせずに履行状態のみを考えます(実際には、`executor` 関数の中でエラーが発生すると Promise インスタンスは自動的に拒否(Rejected)状態へと移行します)。

```js
// reject 関数を省略
const promise = new Promise((resolve) => {
  resolve("Promise履行時の値");
});
```

さらにアロー関数は引数が 1 つのときにカッコを省略できるので次のように文字数を少なくして書けます。

```js
const promise = new Promise(resolve => {
  resolve("Promise履行時の値");
});
```

`resolve()` 関数は名前は何でも良かったのでもっと文字数を減らしてみます。

```js
const promise = new Promise(res => {
  res("Promise履行時の値");
});
```

これで最初の書き方よりもかなり楽に書けていることが分かります。さすがに `res` というような書き方はあまりしないと思いますが、この先にでてくるものとの差異を明らかにするためにわざとやっています。

もっと文字数を減らしてみます。

```js
const promise = new Promise(res => res("Promise履行時の値"));
```

これはアロー関数の省略形の中でも最も短い形式となっています。ですが、実は上のコードは `res => {return res("Promise履行時の値")}` の省略形となっています。ここまでする必要は特にないですが、あとで別の場所で使用するので一応変形してみました。`return` が入っていることに注目してください。この `return` については後述します。

アロー関数の省略は以下のようにでき、下の３つのコートはすべて等価です。

```js
(a) => {
  return a + 100;
}
// 等価
(a) => a + 100;
// 等価
a => a + 100;
```

ここまで、`new Promise(executor)` というコードをなるべく短く書けるように省略してきましたが、実は上記のコードと同じようなことを `Promise()` コンストラクタ関数を使用せずに `Promise.resolve()` という Promise オブジェクトの**静的メソッド**を使って実現できます。

```js
const promise = Promise.resolve("Promise履行時の値");
// この２つは等価
const promise = new Promise(res => {
  res("Promise履行時の値");
});
```

`executor` 関数の引数である `res` 関数と静的メソッドである `Promise.resolve()` は別物であることに注目してください。この `Promise.resolve()` は最も文字数が少なく書けるので、Promise オブジェクトの初期化やテストコードを書く際に活用できる便利なショートカットとして覚えてください。実際に Promise オブジェクトを作成する際には `new Promise(excutor)` が基本となります。

さて、`executor` 関数の引数は２つありました。`resolve` (`res`) と `reject` です。`reject` 第二引数で省略できたので上記のように短く書くために無視してきましたが、これでは不公平なので `reject` についても省略形で書けるようにします。次のコードでは、`executor` 関数の中で `reject()` 関数のみを書いて Promsie インスタンスを拒否状態にしています。

```js
const promise = new Promise((resolve, reject) => {
  reject("Promise拒否時の理由");
});
```

`reject` が省略可能であったのに対して、`resolve` 関数が省略できないのは第一引数だからです。第一引数がないのに第二引数は書けません。

ただ、言ったとおり `resolve` と `reject` は名前は何でも良いのでそれを利用して次のようになるべく文字数が減るように書くことができます。

```js
const promise = new Promise((_, rej) => {
  rej("Promise拒否時の理由");
});
```

アンダーバーという記号を使って使わない `resolve` 関数の名前を最も短い一文字にしています。
実際にはこんな書き方は滅多にしないと思いますが、こうやってできるということを認識するために書いています。

さて、`resolve` でやったようにアロー関数のさらなる省略形でもっと文字数を減らしてみます。

```js
const promise = new Promise((_, rej) => rej("Promise拒否時の理由"));
// ２つは等価
const promise = new Promise((_, rej) => {
  return rej("Promise拒否時の理由");
});
```

かなり文字数が減りましたね。予想できると思いますが、これらのコードと同じことを Promise オブジェクトの静的メソッドである `Promise.reject()` を利用してもっと短く書くことが可能です。

```js
const promise = Promise.reject("Promise拒否時の理由");
// ２つは等価
const promise = new Promise((_, rej) => {
  rej("Promise拒否時の理由");
});
```

この `Promise.reject()` も初期化やテストなどで活用できる便利なショートカットとして使えますが、基本は `new Promise(executor)` です。

# コールバック関数の同期実行と非同期実行

では、Promise インスタンスの基本的な作成方法が分かったところで重要なことを解説します。

`new Promise(executor)` の `Promise()` コンストラクタの引数として渡した `executor` 関数ですが、このコールバック関数は「**同期的に**」実行されます。

```js:executorIsSync.js
console.log("[1] Sync process");

const promise = new Promise(resolve => {
  console.log("[2] これは同期的に実行されます");
  resolve("解決値");
});

console.log("[3] Sync process");
```

ちなみに「非同期処理」について考える時には、必ず同期処理と一緒に考えないと意味がないので、同期的に実行される `console.log()` で囲んでいます。

これを実行すると次のように出力されます。

```sh
❯ deno run executorIsSync.js
[1] Sync process
[2] これは同期的に実行されます
[3] Sync process
```

Promise は「**非同期処理の結果**を表現するビルトインオブジェクト」ですが、このように Promise コンストラクタに渡すコールバック関数は「**同期的に**」実行されます。つまり、完全に上から下へ行を移動するように実行されています。

今度は、上のコードに少し追加したものを考えてみます。「非同期処理」である Promise チェーンです。

```js:thenCallbackIsAsync.js
// thenCallbackIsAsync.js
console.log("[1] Sync process");

const promise = new Promise(resolve => {
  console.log("[2] This line is Synchronously executed");
  resolve("Resolved!");
});

promise.then(value => {
  console.log("[4] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[3] Sync process");
```

さて、結果はコードに書いてあるのでもう分かっていると思いますが、これを実行すると次のような出力になります。

```sh
❯ deno run thenCallbackIsAsync.js
[1] Sync process
[2] This line is Synchronously executed
[3] Sync process
[4] This line is Asynchronously executed
Resolved value:  Resolved!
```

Promise インスタンスは `then()` と `catch()` と `finally()` などの**プロトタイプメソッド**が使用できます。これによって、その Promise インスタンスの**状態が変化した後で**メソッドの引数として渡したコールスタック関数が「**非同期的に**」実行されることを保証できます。

今回の場合、`new Promise(executor)` で作成した Promise インスタンスである `promise` は、コールバック関数である `executor` が同期的に実行されて、すぐさま `resolve()` 関数にであい実行されるので、ただちに `Promise` インスタンスの状態が履行(Fullfilled)状態になります。

コードの行を順番に下に行くと `promise.then(cb)` に出会いますが、ここではコールバックである `cb` は Promise インスタンスが Fullfilled 状態になった時点で Microtask queue へと送られます。この時点で Promise インスタンスである `promise` は履行(Fullfilled)状態なので、直ちにコールバック関数が Microtask queue へと送られます。

しかし、Microtask queue にあるこのコールバック関数はすぐに実行されません。Event Loop ではまず Call stack が完全に空になるまで同期的に実行が続きます。

コードの行をまた下に行くと、`console.log` に出会うので同期的にそれを実行します。この実行が終わった時点で Call stack に積むものは何もなく完全に空の状態になったので、Event Loop が次のステップへと移行して Microtask queue に存在しているマイクロタスクをすべて実行します。

マイクロタスクは現在 1 つあるので直ちにそれを実行します。それによって、"[4] This line is Asynchronously executed" がログに出力されて、その後に "Resolved value:  Resolved!" がログに出力されます。

実際にどのようにマイクロタスクが動くかを JS Visualizer 9000 で可視化してみたので以下のページから確認してみてください。

[thenCallbackIsAsync.js - JS Visualizer 9000](https://www.jsv9000.app/?code=Ly8gdGhlbkNhbGxiYWNrSXNBc3luYy5qcwpjb25zb2xlLmxvZygiPDE%2BIFN5bmMgcHJvY2VzcyIpOwoKY29uc3QgcHJvbWlzZSA9IG5ldyBQcm9taXNlKHJlc29sdmUgPT4gewogIGNvbnNvbGUubG9nKCI8Mj4gVGhpcyBsaW5lIGlzIFN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICByZXNvbHZlKCJSZXNvbHZlZCEiKTsKfSk7Cgpwcm9taXNlLnRoZW4odmFsdWUgPT4gewogIGNvbnNvbGUubG9nKCI8ND4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7Cn0pOwoKY29uc29sZS5sb2coIjwzPiBTeW5jIHByb2Nlc3MiKTsK)

このように Promise チェーンにおいて `.then()` メソッドのコールバックは Promise インスタンスがすでに履行(Fullfilled)状態であっても一旦は Microtask queue へと送られてしまうので、どんなときでもそのコールバックの実行は非同期的になってしまいます。

まとめると、次の２つは対比的な実行となります。

- `Promise()` コンストラクタの引数として渡すコールバック関数(`executor`)は「**同期的に**」実行される
- `then()` メソッドの引数として渡すコールバック関数は「**非同期的に**」実行される

これに気付いていないと「Promise は同期的に実行される」とか「Promsie チェーンは非同期的に実行される」とかの**言葉に惑わされて混乱する**ことになります。

# 複数のPromiseを走らせる
非同期処理で最も重要なことは「制御の流れ」がつかめるようになることです。実行の順番を予測できるようになるために 1 つずつ処理を複雑にして Promise チェーンを解説していきたいと思います。

ここまで `new Promise(executor)` で Promise インスタンスを作成してきました。今度は Promise インスタンスを返す関数をアロー関数式を使用して定義して先程のコードを改造してみます。これで Promise インスタンスを何回も作成できるようになります。また引数を渡せるようにしてその引数で Promise を解決するようにします。

```js:returnPromiseByFunc.js
console.log("[1] Sync process");

const returnPromise = (resolveValue) => {
  return new Promise(resolve => {
    console.log("[2] This line is Synchronously executed");
    resolve(resolveValue);
  });
};

returnPromise("Resolved by function").then(value => {
  console.log("[4] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[3] Sync process");
```

これは前のコードではそのまま Promise インスタンスを作成していたのを Promise インスタンスを返す関数に置き換えただけなので実行結果は以前と同じになります(解決値だけ違う)。

```sh
❯ deno run returnPromiseByFunc.js
[1] Sync process
[2] This line is Synchronously executed
[3] Sync process
[4] This line is Asynchronously executed
Resolved value:  Resolved by function
```

具体的には `returnPromise()` 関数は同期的に実行されて、内部の `Promise()` コンストラクタの引数であるコールバック関数もそのまま同期的に実行されます。

さて、それではこの `returnPromise()` 関数を使用して複数の Promise インスタンスを作成してみます。`reutrnPromise()` 関数は何度も起動させたいので、`[2]` 番目に出力される行を書き換えて引数で指定できるようにします。テンプレートリテラルで表現します。

```js:returnPromiseByFuncArg.js
// returnPromiseByFuncArg.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("First Promise", 2).then((value) => {
  console.log("[4] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[3] Sync process");
```

テンプレートリテラルで書き換えただけなので実行結果は同じになります。

```sh
❯ deno run returnPromiseByFuncArg.js
[1] Sync process
[2] This line is Synchronously executed
[3] Sync process
[4] This line is Asynchronously executed
Resolved value:  First Promise
```

それでは準備ができたのでが実際に複数の Promise インスタンスを作成してみて実行の順番がどうなるかを見てみます。`<>` で囲まれた文字の順番がどのように出力されるか、ここからは自分で出力の順番を予想してみてください。

```js:returnPromiseByFuncArg2.js
// returnPromiseByFuncArg2.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("First Promise", "B").then((value) => {
  console.log("[C] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});
returnPromise("First Promise", "D").then((value) => {
  console.log("[E] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[F] Sync process");
```

実行順番がどうなるか分かりましたか?

:::details 答え
答えは、「A → B → D → F → C → E」となります。

```sh
❯ deno run returnPromiseByFuncArg2.js
[A] Sync process
[B] This line is Synchronously executed
[D] This line is Synchronously executed
[F] Sync process
[C] This line is Asynchronously executed
Resolved value:  First Promise
[E] This line is Asynchronously executed
Resolved value:  First Promise
```
:::

なぜこうなるのか考えてみます。

まずコードは上から下に実行されていきます。Event Loop では最初のステップで同期処理すべて実行されていきます。`returnPromise("First Promise", "B")` は**同期処理**です。関数の中を見ても、`Promise()` コンストラクタの引数である `executor` 関数の中も同期的に実行されます。

`executor` 関数内ですぐに `resolve()` が呼び出されるので Promise インスタンスは直ちに履行(Fullfilled)状態へと移行します。`returnPromise("First Promise", "B")` でここまでは同期的に実行されていることに注意してください。

`returnPromise("First Promise", "B")` で返ってくる Promise インスタンスはすでに履行状態なので、直ちに `then()` メソッドの引数であるコールバック関数が Microtask queue へと送られます。しかし、そのコールバックはまだ実行されません。Event Loop の今のステップでは、Call Stack に積まれるものが完全になくなるまで、スクリプトを評価して同期的処理をすべて実行します。

ということで次に実行される同期処理は  `returnPromise("First Promise", "D")` となります。これも**同期処理**です。先ほどと同じように関数内の `Promise()` コンストラクタの引数である `executor` 関数の中も同期的に実行されます。

また同じように `returnPromise("First Promise", "D")` で返ってくる Promise インスタンスはもすでに履行状態なので、直ちに `then()` メソッドの引数であるコールバック関数が Microtask queue へと送られます。もちろんこのコールバックもまだ同期処理が残っているため、まだ実行されません。

最後の同期処理である `console.log("[F] Sync process");` が次に実行されます。

ここまでで出力されるログは次のようになっていることを確認してください。

```sh
❯ deno run returnPromiseByFuncArg2.js
[A] Sync process
[B] This line is Synchronously executed
[D] This line is Synchronously executed
[F] Sync process

# ...この先はどうなる?
```

同期処理の実行がすべて完了したので、Event Loop は次のステップへと移行します。「同期処理の実行」の次は「Macrotask queue にある Macrotask の実行」を行います。しかし、Macrotask を作成するような処理は行っていないので Macrotask queue にタスクは存在していません。従って、Event Loop は再び次のステップへと移行します。「Macrotask queue にある Macrotask の実行」の次は「Microtask queue にあるすべての Microtask の実行」を行います。

Microtask queue はキューなので一番古いタスク(先に入れられたタスク)から実行していきます。最初にキューへと追加されたのは `returnPromise("First Promise", "B").then()` のコールバック関数です。従って、出力の続きは次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2.js
[A] Sync process
[B] This line is Synchronously executed
[D] This line is Synchronously executed
[F] Sync process
[C] This line is Asynchronously executed
Resolved value:  First Promise

# ...この先はどうなる?
```

そして、次にキューに追加されたのは `returnPromise("First Promise", "D").then()` のコールバック関数でした。従ってそのコールバック関数が実行されることで結局出力は次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2.js
[A] Sync process
[B] This line is Synchronously executed
[D] This line is Synchronously executed
[F] Sync process
[C] This line is Asynchronously executed
Resolved value:  First Promise
[E] This line is Asynchronously executed
Resolved value:  First Promise
```

順番をアルファベットから数字に直してみるとこのようになります。

```js
// doubleThenCallback.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("First Promise", "2").then((value) => {
  console.log("[5] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});
returnPromise("First Promise", "3").then((value) => {
  console.log("[6] This line is Asynchronously executed");
  console.log("Resolved value: ", value);
});

console.log("[4] Sync process");
```

↓ JS Visuzalizer 9000 で実際に可視化してみたので確認してくみてください。

[doubleThenCallback.js - JS Visuzalizer 9000](https://www.jsv9000.app/?code=Ly8gZG91YmxlVGhlbkNhbGxiYWNrLmpzCmNvbnNvbGUubG9nKCI8MT4gU3luYyBwcm9jZXNzIik7Cgpjb25zdCByZXR1cm5Qcm9taXNlID0gKHJlc29sdmVkVmFsdWUsIG9yZGVyKSA9PiB7CiAgcmV0dXJuIG5ldyBQcm9taXNlKChyZXNvbHZlKSA9PiB7CiAgICBjb25zb2xlLmxvZyhgPCR7b3JkZXJ9PiBUaGlzIGxpbmUgaXMgU3luY2hyb25vdXNseSBleGVjdXRlZGApOwogICAgcmVzb2x2ZShyZXNvbHZlZFZhbHVlKTsKICB9KTsKfTsKCnJldHVyblByb21pc2UoIkZpcnN0IFByb21pc2UiLCAiMiIpLnRoZW4oKHZhbHVlKSA9PiB7CiAgY29uc29sZS5sb2coIjw1PiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKfSk7CnJldHVyblByb21pc2UoIkZpcnN0IFByb21pc2UiLCAiMyIpLnRoZW4oKHZhbHVlKSA9PiB7CiAgY29uc29sZS5sb2coIjw2PiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKfSk7Cgpjb25zb2xlLmxvZygiPDQ%2BIFN5bmMgcHJvY2VzcyIpOw%3D%3D)

# thenメソッドは常に新しいPromiseを返す

では `then()` メソッドをそれぞれもう 1 つずつ増やしてみてみます。次のコードについても出力の順番を予測してみてください。

```js
// returnPromiseByFuncArg2AddChain.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("First Promise", "B")
  .then((value) => {
    console.log("[C] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  })
  .then(() => {
    console.log("[D] This line is Asynchronously executed");
  });
returnPromise("First Promise", "E")
  .then((value) => {
    console.log("[F] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  })
  .then(() => {
    console.log("[G] This line is Asynchronously executed");
  });

console.log("[H] Sync process");
```

:::details 答え
答えは、「A → B → E → H → C → F → D → G」となります。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
[A] Sync process
[B] This line is Synchronously executed
[E] This line is Synchronously executed
[H] Sync process
[C] This line is Asynchronously executed
Resolved value:  First Promise
[F] This line is Asynchronously executed
Resolved value:  First Promise
[D] This line is Asynchronously executed
[G] This line is Asynchronously executed
```
:::

さて、正解できましたか?それでは、なぜこうなるのか、解説してみます。

準備としてコールバック関数などを省略してコードを簡略化してみます。

```js
console.log("[A] Sync process");
const returnPromise = (resolvedValue, order) => {...};
returnPromise("First Promise", "B").then(cb1).then(cb2);
returnPromise("First Promise", "E").then(cb3).then(cb4);
console.log("[H] Sync process");
```

前のコードと考え方は同じです。まずは Event Loop の最初のステップである「同期処理の実行」が行われます。

- (1) `console.log("[A] Sync process")` が同期処理される
- (2) `returnPromise("First Promise", "B")` が同期処理されて返される Promise インスタンスが直ちに履行(Fullfilled)状態になるので、`returnPromise("First Promise", "B").then(cb)` のコードバック関数 `cb` が直ちに Microtask queue へと送られます。

さて、ここまでは前のコードと同じですね。

ここでは「**`then()` メソッドは常に新しい Promise インスタンスを返す**」ということが重要です。

- `returnPromise("First Promise", "B")` によって返ってくる Promise インスタンスを promise1 とします
- ``returnPromise("First Promise", "B").then(cb1)` 、つまり `promise1.then(cb1)` によって返ってくる Promise インスタンスを `proimse2` とします

このふたつは全く別の Promise インスタンスとなります。Promise インスタンスは状態を持っていますね。

- 待機(pending)状態
- 不変(Setteled)状態
  - 履行(Fullfilled)状態
  - 拒否(Rejected)状態

Promise インスタンスは基本的に待機(pending)状態から始まります。Promise チェーンでは `then()` メソッドで返ってくる Promise インスタンスの状態が待機(pending)状態から履行(Fullfilled)状態へと変わった時点で次の `then()` メソッドで登録したコールバックが Microtask queue へと送られます。

そして、`then(cb)` で返ってくる Promise インスタンスが履行状態へと移行するのは登録されているコールバック `cb` が実行が完了した時点です。

従って、`returnPromise("First Promise", "B").then(cb1)` で返ってくる Promise インスタンスは Event Loop のこの時点では登録しているコールバック `cb1` が Microtask queue へと送られただけで処理は完了していませんので、まだ待機(pending)状態となります。

`then(cb1)` で返ってくる Promise インスタンスが待機状態なので、`returnPromise("First Promise", "B").then(cb1).then(cb2)` で登録したコールバック `cb2` はまだ Microtask queue へと送られません。このまま待機させておきます。

そして、そのまま次の処理へと進みます。次の行は `returnPromise("First Promise", "E").then(cb1).then(cb2)` なので、まったく同じことが置きます。

```js
console.log("[A] Sync process");
const returnPromise = (resolvedValue, order) => {...};
returnPromise("First Promise", "B").then(cb1).then(cb2);
returnPromise("First Promise", "E").then(cb3).then(cb4);
console.log("[H] Sync process");
```

1. `returnPromise("First Promise", "E")` が同期的に実行されて直ちに履行(Fullfilled)状態となった Promise インスタンスが返ってくるので、`then(cb3)` で登録されているコールバック関数 `cb3` が直ちに Microtask queue へと送られます
2. `then(cb3)` で返ってくる別の Promise インスタンスはまだ待機(pending)状態なので `then(cb4)` のコールバック関数 `cb4` はまだキューへ送られずにそのまま待機となります
3. 次の処理に進み、`console.log("[H] Sync process")` が実行されます

これで Event Loop の最初のステップである「同期処理の実行」が終わりました。出力はこの時点で次のようになっています。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
[A] Sync process
[B] This line is Synchronously executed
[E] This line is Synchronously executed
[H] Sync process

# ...この先はどうなる?
```

Event Loop の最初のステップである「同期処理の実行」の次は「Macrotask queue にある Macrotask の実行」を行います。しかし、Macrotask を作成するような処理は行っていないので Macrotask queue にタスクは存在していません。従って、Event Loop は再び次のステップへと移行します。「Macrotask queue にある Macrotask の実行」の次は「Microtask queue にあるすべての Microtask の実行」を行います。

先にキューへと送られた `cb1` が実行されます。`then(cb1)` で登録したコールバック `cb1` の実行が完了したので `then(cb1)` で返ってくる Promise インスタンスが履行(Fullfilled)状態へと移行します。Promise インスタンスの状態が履行状態へと移行したことで、さらに `then(cb1).then(cb2)` で登録していたコールバック関数  `cb2` が直ちに Microtask queue へと送られます。

続いて次に Microtask queue 内にあるマイクロタスクが実行されます。`cb1` の後には `cb3` が順番としてキューに送られていたので `cb3` が直ちに実行されます。`cb1` のときと同じように `then(cb3)` で返ってくる Promsie インスタンスの状態が待機(pending)状態から履行(Fullfilled)状態へと移行します。`then(cb3)` で返ってくる Promise インスタンスの状態が履行(Fullfilled)状態へと変わったことで、後続の `then(cb4)` で登録していたコールバック関数 `cb4` が直ちに Microtask queue へと送られます。

この時点での出力はこのようになっています。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
[A] Sync process
[B] This line is Synchronously executed
[E] This line is Synchronously executed
[H] Sync process
[C] This line is Asynchronously executed
Resolved value:  First Promise
[F] This line is Asynchronously executed
Resolved value:  First Promise

# ...この先はどうなる?
```

この時点のステップは「Microtask queue にあるすべての Microtask の実行」であり、Microtask queue にマイクロタスクが存在し続ける限りそれらは実行されます。いまだに `cb2` と `cb4` が順番に Microtask queue に存在しているのでそれらも順番に実行されていきます。

従って、最終的な出力は次のようになります。

```sh
❯ deno run returnPromiseByFuncArg2AddChain.js
[A] Sync process
[B] This line is Synchronously executed
[E] This line is Synchronously executed
[H] Sync process
[C] This line is Asynchronously executed
Resolved value:  First Promise
[F] This line is Asynchronously executed
Resolved value:  First Promise
[D] This line is Asynchronously executed
[G] This line is Asynchronously executed
```

言葉で説明すると非常に長くなってしまいましたがこのような結果となります。
実際に JS Visualizer 9000 で可視化してみたので確認してみてください。

- [returnPromiseByFuncArg2AddChain.js - JS Visualizer 9000](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUJ5RnVuY0FyZzJBZGRDaGFpbi5qcwpjb25zb2xlLmxvZygiPEE%2BIFN5bmMgcHJvY2VzcyIpOwoKY29uc3QgcmV0dXJuUHJvbWlzZSA9IChyZXNvbHZlZFZhbHVlLCBvcmRlcikgPT4gewogIHJldHVybiBuZXcgUHJvbWlzZSgocmVzb2x2ZSkgPT4gewogICAgY29uc29sZS5sb2coYDwke29yZGVyfT4gVGhpcyBsaW5lIGlzIFN5bmNocm9ub3VzbHkgZXhlY3V0ZWRgKTsKICAgIHJlc29sdmUocmVzb2x2ZWRWYWx1ZSk7CiAgfSk7Cn07CgpyZXR1cm5Qcm9taXNlKCJGaXJzdCBQcm9taXNlIiwgIkIiKQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coIjxDPiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWUpOwogIH0pCiAgLnRoZW4oKCkgPT4gewogICAgY29uc29sZS5sb2coIjxEPiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICB9KTsKcmV0dXJuUHJvbWlzZSgiRmlyc3QgUHJvbWlzZSIsICJFIikKICAudGhlbigodmFsdWUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCI8Rj4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICB9KQogIC50aGVuKCgpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCI8Rz4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgfSk7Cgpjb25zb2xlLmxvZygiPEg%2BIFN5bmMgcHJvY2VzcyIpOwo%3D)

# Promiseチェーンで値を繋ぐ
さて、Promsise チェーンの基本的な動きが分かったと思います。

ここからは値を Promise チェーンにおいて値をつないでいく処理を考えてみたいと思います。
then()` メソッドの引数のコールバックには入力として前の `then()` メソッドのコールバック内にて `return` した値を渡すことができます。

先程のコードを更に改造して、実際にテストしてみます。再び実行の順番を予想してみてください。

```js
// returnPromiseByFuncArg2AddChainValue.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("First Promise", "B")
  .then((value1) => {
    console.log("[C] This line is Asynchronously executed");
    console.log("Resolved value: ", value1);
    return "Resolved value passing to the next then callback";
  })
  .then((value2) => {
    console.log("[D] This line is Asynchronously executed");
    console.log("Resolved value: ", value2);
    return "Resolved value passing to the next then callback";
  })
  .then((value3) => {
    console.log("[E] This line is Asynchronously executed");
    console.log("Resolved value: ", value3);
    // return "Resolved value passing to the next then callback";
  })
  .then((value4) => {
    console.log("[F] This line is Asynchronously executed");
    console.log("Resolved value: ", value4);
  });
returnPromise("First Promise", "G")
  .then((value1) => {
    console.log("[H] This line is Asynchronously executed");
    console.log("Resolved value: ", value1);
    return "Resolved value passing to the next then callback";
  })
  .then((value2) => {
    console.log("[I] This line is Asynchronously executed");
    console.log("Resolved value: ", value2);
    return "Resolved value passing to the next then callback";
  })
  .then((value3) => {
    console.log("[J] This line is Asynchronously executed");
    console.log("Resolved value: ", value3);
    // return "Resolved value passing to the next then callback";
  })
  .then((value4) => {
    console.log("[K] This line is Asynchronously executed");
    console.log("Resolved value: ", value4);
    return "Resolved value passing to the next then callback";
  });

console.log("[L] Sync process");
```

:::details 答え
答えは、「A → B → G → L → C → H → D → I → E → J → F → K」となります。

```sh
❯ deno run returnPromiseByFuncArg2AddChainValue.js
[A] Sync process
[B] This line is Synchronously executed
[G] This line is Synchronously executed
[L] Sync process
[C] This line is Asynchronously executed
Resolved value:  First Promise
[H] This line is Asynchronously executed
Resolved value:  First Promise
[D] This line is Asynchronously executed
Resolved value:  Resolved value passing to the next then callback
[I] This line is Asynchronously executed
Resolved value:  Resolved value passing to the next then callback
[E] This line is Asynchronously executed
Resolved value:  Resolved value passing to the next then callback
[J] This line is Asynchronously executed
Resolved value:  Resolved value passing to the next then callback
[F] This line is Asynchronously executed
Resolved value:  undefined
[K] This line is Asynchronously executed
Resolved value:  undefined
```

アルファベットに数字をつけてみると分かりやすくなります。

```js
// returnPromiseByFuncArg2AddChainValue.js
console.log("<A-1> Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("First Promise", "B-2")
  .then((value1) => {
    console.log("<C-5> This line is Asynchronously executed");
    console.log("Resolved value: ", value1);
    return "Resolved value passing to the next then callback";
  })
  .then((value2) => {
    console.log("<D-7> This line is Asynchronously executed");
    console.log("Resolved value: ", value2);
    return "Resolved value passing to the next then callback";
  })
  .then((value3) => {
    console.log("<E-9> This line is Asynchronously executed");
    console.log("Resolved value: ", value3);
    // return "Resolved value passing to the next then callback";
  })
  .then((value4) => {
    console.log("<F-11> This line is Asynchronously executed");
    console.log("Resolved value: ", value4);
    // undefined となる
  });
returnPromise("First Promise", "G-3")
  .then((value1) => {
    console.log("<H-6> This line is Asynchronously executed");
    console.log("Resolved value: ", value1);
    return "Resolved value passing to the next then callback";
  })
  .then((value2) => {
    console.log("<I-8> This line is Asynchronously executed");
    console.log("Resolved value: ", value2);
    return "Resolved value passing to the next then callback";
  })
  .then((value3) => {
    console.log("<J-10> This line is Asynchronously executed");
    console.log("Resolved value: ", value3);
    // return "Resolved value passing to the next then callback";
  })
  .then((value4) => {
    console.log("<K-12> This line is Asynchronously executed");
    console.log("Resolved value: ", value4);
    // undefined となる
  });

console.log("<L-4> Sync process");
```
:::

動きは前のコードと同じなので解説はしません。JS Visualizer 9000 で可視化したものは以下です。

- [returnPromiseByFuncArg2AddChainValue.js](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUJ5RnVuY0FyZzJBZGRDaGFpblZhbHVlLmpzCmNvbnNvbGUubG9nKCI8QS0xPiBTeW5jIHByb2Nlc3MiKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGA8JHtvcmRlcn0%2BIFRoaXMgbGluZSBpcyBTeW5jaHJvbm91c2x5IGV4ZWN1dGVkYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pOwp9OwoKcmV0dXJuUHJvbWlzZSgiRmlyc3QgUHJvbWlzZSIsICJCLTIiKQogIC50aGVuKCh2YWx1ZTEpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCI8Qy01PiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWUxKTsKICAgIHJldHVybiAiUmVzb2x2ZWQgdmFsdWUgcGFzc2luZyB0byB0aGUgbmV4dCB0aGVuIGNhbGxiYWNrIjsKICB9KQogIC50aGVuKCh2YWx1ZTIpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCI8RC03PiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWUyKTsKICAgIHJldHVybiAiUmVzb2x2ZWQgdmFsdWUgcGFzc2luZyB0byB0aGUgbmV4dCB0aGVuIGNhbGxiYWNrIjsKICB9KQogIC50aGVuKCh2YWx1ZTMpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCI8RS05PiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWUzKTsKICAgIC8vIHJldHVybiAiUmVzb2x2ZWQgdmFsdWUgcGFzc2luZyB0byB0aGUgbmV4dCB0aGVuIGNhbGxiYWNrIjsKICB9KQogIC50aGVuKCh2YWx1ZTQpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCI8Ri0xMT4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlNCk7CiAgfSk7CnJldHVyblByb21pc2UoIkZpcnN0IFByb21pc2UiLCAiRy0zIikKICAudGhlbigodmFsdWUxKSA9PiB7CiAgICBjb25zb2xlLmxvZygiPEgtNj4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlMSk7CiAgICByZXR1cm4gIlJlc29sdmVkIHZhbHVlIHBhc3NpbmcgdG8gdGhlIG5leHQgdGhlbiBjYWxsYmFjayI7CiAgfSkKICAudGhlbigodmFsdWUyKSA9PiB7CiAgICBjb25zb2xlLmxvZygiPEktOD4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlMik7CiAgICByZXR1cm4gIlJlc29sdmVkIHZhbHVlIHBhc3NpbmcgdG8gdGhlIG5leHQgdGhlbiBjYWxsYmFjayI7CiAgfSkKICAudGhlbigodmFsdWUzKSA9PiB7CiAgICBjb25zb2xlLmxvZygiPEotMTA%2BIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCIpOwogICAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZTMpOwogICAgLy8gcmV0dXJuICJSZXNvbHZlZCB2YWx1ZSBwYXNzaW5nIHRvIHRoZSBuZXh0IHRoZW4gY2FsbGJhY2siOwogIH0pCiAgLnRoZW4oKHZhbHVlNCkgPT4gewogICAgY29uc29sZS5sb2coIjxLLTEyPiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWU0KTsKICB9KTsKCmNvbnNvbGUubG9nKCI8TC00PiBTeW5jIHByb2Nlc3MiKTsK)

ポイントとしては、`return` 文をコメントアウトしてある `then()` コールバックの次の `then()` コールバックでは、渡されるはずの値がないので `undefined` となっている点です。何も `return` しない場合には次の `then()` メソッドのコールバックの入力値は `undefined` となるので注意してください。

# thenコールバックでPromiseインスタンスを返す
今のコードでは `then()` メソッドのコールバックにおいて `return` して返却したのは `"Resolved value passing to the next then callback"` という文字列でした。

`return` する値は、数値でも真偽値でも文字列での何でも良いのですが、Promise インスタンスを返した場合はどうなるでしょうか?

答えは、「**その Promise インスタンスが `resolve` された値が次の `then()` メソッドのコールバック関数の引数として渡される**」です。実際に `then()` メソッドのコールバック関数において新しい Promise インスタンスを返してみます。今までのコードをまた流用します。

```js
// returnPromiseFromThenCallback.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is (A)Synchronously executed`);
    // 非同期で実行される場合もあるのでテキストを変更した
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "2")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "6");
    // resolve される値は "2nd Promise" で、これが次の then() のコールバック関数の入力として渡される
  })
  .then((value) => {
    console.log("[9] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
returnPromise("3rd Promise", "3")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("4th Promise", "8");
    // resolve される値は "5th Promise" で、これが次の then() のコールバック関数の入力として渡される
  })
  .then((value) => {
    console.log("<10> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
  
console.log("[4] Sync process");
```

今まで必ず同期処理として呼ばれていた `returnPromise()` 関数ですが、今回は `then()` メソッドのコールバックで呼び出しているものもあるのでそれらは非同期的に実行されます。従って出力されるテキストを一部変更しました。

これを実行すると、次のような出力になります。

```sh
❯ deno run returnPromiseFromThenCallback.js
[1] Sync process
[2] This line is (A)Synchronously executed
[3] This line is (A)Synchronously executed
[4] Sync process
[5] This line is Asynchronously executed
Resolved value:  1st Promise
[6] This line is (A)Synchronously executed
[7] This line is Asynchronously executed
Resolved value:  3rd Promise
[8] This line is (A)Synchronously executed
[9] This line is Asynchronously executed
Resolved value:  2nd Promise
<10> This line is Asynchronously executed
Resolved value:  4th Promise
```

Promise インスタンスを `then()` メソッドのコールバック関数内で `return` したの実行の順番がどうなるか不安になったかもしれませんが、今回の `returnPromise()` 関数の場合は、ただちに履行(Fullfilled)状態の Promise インスタンスが返ってくるので普通の値を返す場合とまったく同じになります。JS Visuzalizer 9000 で可視化したのでまた確認してみてください。

- [returnPromiseFromThenCallback.js](https://www.jsv9000.app/?code=Ly8gcmV0dXJuUHJvbWlzZUZyb21UaGVuQ2FsbGJhY2suanMKY29uc29sZS5sb2coIjwxPiBTeW5jIHByb2Nlc3MiKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGA8JHtvcmRlcn0%2BIFRoaXMgbGluZSBpcyAoQSlTeW5jaHJvbm91c2x5IGV4ZWN1dGVkYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pOwp9OwoKcmV0dXJuUHJvbWlzZSgiMXN0IFByb21pc2UiLCAiMiIpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiPDU%2BIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCIpOwogICAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7CiAgICByZXR1cm4gcmV0dXJuUHJvbWlzZSgiMm5kIFByb21pc2UiLCAiNiIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiPDk%2BIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCIpOwogICAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7CiAgfSk7CnJldHVyblByb21pc2UoIjNyZCBQcm9taXNlIiwgIjMiKQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coIjw3PiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWUpOwogICAgcmV0dXJuIHJldHVyblByb21pc2UoIjR0aCBQcm9taXNlIiwgIjgiKTsKICB9KQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coIjwxMD4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCI8ND4gU3luYyBwcm9jZXNzIik7Cg%3D%3D)

# Promiseチェーンはネストさせない

だだし、次のように場合はどうなるでしょうか? 
`return` しているものが `returnPromise().then(cb)` となっています。

```js
// promiseNest.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "2")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "6")
      .then((value) => {
        console.log("[9] This line is Asynchronously executed");
        console.log("Resolved value: ", value);
        return "from [9] callback";
      });
  })
  .then((value) => {
    console.log("<11> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
returnPromise("3rd Promise", "3")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("4th Promise", "8")
      .then((value) => {
        console.log("<10> This line is Asynchronously executed");
        console.log("Resolved value: ", value);
        return "from <10> callback";
      });
  })
  .then((value) => {
    console.log("<12> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[4] Sync process");
```

基本的には今までの流れと代りません。また圧縮して書いてみます。

```js
console.log("[1] Sync process");
const returnPromise = (resolvedValue, order) => {...};
returnPromise("1st Promise", "2").then(cb1).then(cb2);
returnPromise("3rd Promise", "3").then(cb3).then(cb4);
console.log("[4] Sync process");
```

今までと同じところまで流れを書いてみます。
1. 「同期処理を実行する」
  1. `console.log("[1] Sync process")` が同期処理される
  2. `returnPromise("1st Promise", "2")` が同期処理されて `then(cb1)` のコールバック `cb1` が直ちに Microtask queue へと送られる
  3. `returnPromise("3rd Promise", "3")` が同期処理されて `then(cb3)` のコールバック `cb3` が直ちに Microtask queue へと送られる
  4. `console.log("[4] Sync process")` が同期処理される
  5. Event Loop の次のステップへ移行
2. 「Macrotask queue の Macrotask を実行する」
  6.  何も無いので Event Loop の次のステップへ
3. 「Microtask queue のすべての Microtask を実行する」
  7. `cb1` が実行される

この `cb1` の実行が問題です。`return returnPromise("2nd Promise", "6")` に注目すると、`returnPromise()` 関数は直ちに履行状態になる Promise インスタンスを返すので、`then(callbackNext)` のコールバック `callbackNext` が直ちに Microtask queue へと送られます。現時点の Microtask queue には次のコールバック関数がマイクロタスクとして追加されています。

```sh:Microtask queue
<-- cb3 <--
```

この時点での `cb1` に注目してみます。

```js
returnPromise("1st Promise", "2")
  .then((value) => {
    // これが cb1 
    // 上から下に実行されていく
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);

    return returnPromise("2nd Promise", "6").then(callbackNext);
    // 返される Promise インスタンスが履行状態なので then のコールバック関数が直ちに Microtask queue へと送られる
  }) // ここで返される Promise チェーンはまだ待機状態
  .then(cb2);
```

さて、ここが混乱しやすいところですが、Promsie チェーンにおいて `then()` メソッドのコールバック関数内で、`return` によって Promise インスタンスを返した場合はその解決値(resolve された値)が次の `then()` メソッドのコールバック関数の引数として渡されます。

いまコールバック関数内で `return` しているのは `returnPromise("2nd Promise", "6")` ではなく、`returnPromise("2nd Promise", "6").then(callbackNext)` なので、`then(callbackNext)` で返される Promsie インスタンスの resolve した値が `then(cb2)` のコールバック関数 `cb2` の引数として渡されるはずです。

ですが、今の時点では `callbackNext` はキューへ送られていて実行されていないので、`then(callbackNext)` か返される Promise インスタンスは待機(pending)状態です。つまり、待機状態の Promise インスタンスを返してしまっています。

`then()` メソッドのコールバック関数内で待機状態の Promise インスタンスを返した場合はそれが解決されない限り、その `then()` メソッドから返ってくる Promise インスタンスも待機状態のままとなります。

ここで考えるのは親のコールバック関数 `cb1` を登録していた `then(cb1)` から返される Promise インスタンスです。この `cb1` から返される Promise インスタンスが解決されない `then(cb1)` から返される Promise インスタンスの状態が履行状態にはならず待機状態のもままで、次の `then()` メソッドのコールバック関数を Microtask queue へと送ることができません。

それはそれとして、`callbackNext` が Microtask queue へと送られた後、Event Loop のステップは「Microtask queue のすべての Microtask を実行する」の状態にあります。現時点で Microtask queue の先頭にあるタスクはコールバック関数 `cb3` なので、このコールバックが次に実行されます。

```sh:Microtask queue
<-- cb3 <-- callbackNext
```

コールバック関数 `cb3` に注目してみます。

```js
returnPromise("3rd Promise", "3")
  .then((value) => {
    // これが cb3 
    // 上から下に実行されていく
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);

    return returnPromise("4th Promise", "8").then(callbackNext2);
    // 返される Promise インスタンスが履行状態なので then のコールバック関数が直ちに Microtask queue へと送られる
  }) // ここで返される Promise チェーンはまだ待機状態
  .then(cb4);
```

全く同じように、`callbackNext2` が直ちに Microtask queue へと追加されます。

```sh:Microtask queue
<-- callbackNext <-- callbackNext2
```

`returnPromise("3rd Promise", "3").then(cb3)` から返される Promise インスタンスはまだ待機状態となります。Event Loop の状態もそのままなので、再び Microtask queue の先頭のマイクロタスクが実行されます。

```js
returnPromise("1st Promise", "2")
  .then((value) => {
    // cb1
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "6")
      .then((value) => {
        // callbackNext
        console.log("[9] This line is Asynchronously executed");
        console.log("Resolved value: ", value);
        return "from [9] callback";
      });
  })
  .then((value) => {
    // cb2
    console.log("<11> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
```

`callbackNext` が実行されると `cb1` コールバック内において結局、`"from [9] callback"` という文字列で解決された Promise インスタンスが結局 `return` されたことになります。つまり、`return Promsie.resolve("from [9] callback")` と同じです。

次のように書きましたが、`then()` メソッドのコールバック関数内で待機状態の Promsie が解決されて履行(Fullfilled)状態になったので、その `then()` メソッドから返ってくる Promise インスタンスも解決されて履行状態となります。

>`then()` メソッドのコールバック関数内で待機状態の Promise インスタンスを返した場合はそれが解決されない限り、その `then()` メソッドから返ってくる Promise インスタンスも待機状態のままとなります。

`returnPromise("1st Promise", "2").then(cb1)` から返される Promise インスタンスが待機状態から履行状態に移行したので、次の `then(cb2)` のコールバック関数 `cb2` が Microtask queue へと直ちに送られます。

```sh:Microtask queue
<-- callbackNext2 <-- cb2
```

そして、再び Microtask queue の先頭にあるマイクロタスクであるコールバック関数 `callbackNext2` が実行されて同じのように、`returnPromise("3rd Promise", "3").then(cb3)` から返される Promise インスタンスが履行状態となり、次の `then(cb4)` のコールバック関数 `cb4` が Microtask queue へと直ちに送られます。

```sh:Microtask queue
<-- cb2 <-- cb4
```

そして、同じように、`cb2`、`cb4` の順番で実行されて終わります。出力は次のようになります。

```js
❯ deno run promiseNest.js
[1] Sync process
[2] This line is (A)Synchronously executed
[3] This line is (A)Synchronously executed
[4] Sync process
[5] This line is Asynchronously executed
Resolved value:  1st Promise
[6] This line is (A)Synchronously executed
[7] This line is Asynchronously executed
Resolved value:  3rd Promise
[8] This line is (A)Synchronously executed
[9] This line is Asynchronously executed
Resolved value:  2nd Promise
<10> This line is Asynchronously executed
Resolved value:  4th Promise
<11> This line is Asynchronously executed
Resolved value:  from [9] callback
<12> This line is Asynchronously executed
Resolved value:  from <10> callback
```

分かりにくいですが、結局普通の Promsie チェーンと同じ出力の順番になります。JS Visuzalizer で可視化してみたので実際にそうなることを確認してみてください。

- [promiseNest.js - JS Visuzalizer 9000](https://www.jsv9000.app/?code=Ly8gcHJvbWlzZU5lc3QuanMKY29uc29sZS5sb2coIjwxPiBTeW5jIHByb2Nlc3MiKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIpID0%2BIHsKICByZXR1cm4gbmV3IFByb21pc2UoKHJlc29sdmUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKGA8JHtvcmRlcn0%2BIFRoaXMgbGluZSBpcyAoQSlTeW5jaHJvbm91c2x5IGV4ZWN1dGVkYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pOwp9OwoKcmV0dXJuUHJvbWlzZSgiMXN0IFByb21pc2UiLCAiMiIpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiPDU%2BIFRoaXMgbGluZSBpcyBBc3luY2hyb25vdXNseSBleGVjdXRlZCIpOwogICAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7CiAgICByZXR1cm4gcmV0dXJuUHJvbWlzZSgiMm5kIFByb21pc2UiLCAiNiIpCiAgICAgIC50aGVuKCh2YWx1ZSkgPT4gewogICAgICAgIGNvbnNvbGUubG9nKCI8OT4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICAgICAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7CiAgICAgICAgcmV0dXJuICJmcm9tIDw5PiBjYWxsYmFjayI7CiAgICAgIH0pOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiPDExPiBUaGlzIGxpbmUgaXMgQXN5bmNocm9ub3VzbHkgZXhlY3V0ZWQiKTsKICAgIGNvbnNvbGUubG9nKCJSZXNvbHZlZCB2YWx1ZTogIiwgdmFsdWUpOwogIH0pOwpyZXR1cm5Qcm9taXNlKCIzcmQgUHJvbWlzZSIsICIzIikKICAudGhlbigodmFsdWUpID0%2BIHsKICAgIGNvbnNvbGUubG9nKCI8Nz4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICAgIHJldHVybiByZXR1cm5Qcm9taXNlKCI0dGggUHJvbWlzZSIsICI4IikKICAgICAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICAgICAgY29uc29sZS5sb2coIjwxMD4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICAgICAgY29uc29sZS5sb2coIlJlc29sdmVkIHZhbHVlOiAiLCB2YWx1ZSk7CiAgICAgICAgcmV0dXJuICJmcm9tIDwxMD4gY2FsbGJhY2siOwogICAgICB9KTsKICB9KQogIC50aGVuKCh2YWx1ZSkgPT4gewogICAgY29uc29sZS5sb2coIjwxMj4gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCI8ND4gU3luYyBwcm9jZXNzIik7Cg%3D%3D)


このように `then()` メソッドをネストさせるようなやり方は特に意味がある場合を覗いて、流れがわかりづらくなってしまうので通常は避けるべきアンチパターンとなります。このネストはフラットにでき、Promise チェーンはなるべくネストが浅くなるようにフラットにするのが推奨されます。

実際にネストを解消してみます。

```diff js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "2")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "6")
-     .then((value) => {
-       console.log("[9] This line is Asynchronously executed");
-       console.log("Resolved value: ", value);
-       return "from [9] callback";
-     });
  })
+ .then((value) => {
+   console.log("[9] This line is Asynchronously executed");
+   console.log("Resolved value: ", value);
+   return "from [9] callback";
+ })
  .then((value) => {
    console.log("<11> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
returnPromise("3rd Promise", "3")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("4th Promise", "8")
-     .then((value) => {
-       console.log("<10> This line is Asynchronously executed");
-       console.log("Resolved value: ", value);
-       return "from <10> callback";
-     });
  })
+ .then((value) => {
+   console.log("<10> This line is Asynchronously executed");
+   console.log("Resolved value: ", value);
+   return "from <10> callback";
+ })
  .then((value) => {
    console.log("<12> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[4] Sync process");
```

結果はこのようになり、ネストした状態のものよりも圧倒的に見やすく、流れが分かりなりました。

```js
// promiseNestShallow.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "2")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("2nd Promise", "6")
  })
  .then((value) => {
    console.log("[9] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return "from [9] callback";
  })
  .then((value) => {
    console.log("<11> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });
returnPromise("3rd Promise", "3")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return returnPromise("4th Promise", "8")
  })
  .then((value) => {
    console.log("<10> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    return "from <10> callback";
  })
  .then((value) => {
    console.log("<12> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[4] Sync process");
```

出力結果は全く同じになります。

```sh
❯ deno run promiseNestShallow.js
[1] Sync process
[2] This line is (A)Synchronously executed
[3] This line is (A)Synchronously executed
[4] Sync process
[5] This line is Asynchronously executed
Resolved value:  1st Promise
[6] This line is (A)Synchronously executed
[7] This line is Asynchronously executed
Resolved value:  3rd Promise
[8] This line is (A)Synchronously executed
[9] This line is Asynchronously executed
Resolved value:  2nd Promise
<10> This line is Asynchronously executed
Resolved value:  4th Promise
<11> This line is Asynchronously executed
Resolved value:  from [9] callback
<12> This line is Asynchronously executed
Resolved value:  from <10> callback
```

Promise チェーンはこのようにネストさせずに流れを見やすくします。

# returnする代りに副作用を使用しない
さて、今まで `then()` メソッドのコールバック関数内にて返すものとしては次のパターンでした。

- (1) 文字列や数値などの通常の値を `return` する
  - 直ちに次の `then()` メソッドのコールバックが Microtask queue へと追加されて、コールバック関数の引数には `return` した値が渡される
- (2) Promsie インスタンスを `return` する
  - 待機状態ならそれが解決してから次の `then()` メソッドのコールバックが Microtask queue へと追加され、`resolve` した値がコールバック関数の引数に渡される
  - 履行状態なら直ちに次の `then()` メソッドのコールバックが Microtask queue へと追加され、`resolve` した値がコールバック関数の引数に渡される
- (3) 何も `return` しない
  - 直ちに次の `then()` メソッドのコールバックが Microtask queue へと追加されて、コールバック関数の引数は `undefined` となる

(2) と (3) を混同してしまう場合に気をつけてください。`then()` メソッドのコールバック関数で Promise を使った非同期処理を行う場合には必ず Promise インスタンスを `return` するようにしてください。

```js
// promiseShouldBeReturned.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "2")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // return しない場合は副作用となり値が渡らず解決されるまえに次に行ってしまう
    returnPromise("2nd Promise", "6")
  })
  .then((value) => {
    // この value は undefined となる
    console.log("[9] This line is Asynchronously executed");
    console.log("Resolved value: ", value); // undefined が表示される
  });
returnPromise("3rd Promise", "3")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // Promise インスタンスについては必ず return するようにする
    return returnPromise("4th Promise", "8")
  })
  .then((value) => {
    console.log("<10> This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[4] Sync process");
```

これを実行すると次の出力を得ます。`undefined` となっているところに注目してください。

```sh
❯ deno run promiseShouldBeReturned.js
[1] Sync process
[2] This line is (A)Synchronously executed
[3] This line is (A)Synchronously executed
[4] Sync process
[5] This line is Asynchronously executed
Resolved value:  1st Promise
[6] This line is (A)Synchronously executed
[7] This line is Asynchronously executed
Resolved value:  3rd Promise
[8] This line is (A)Synchronously executed
[9] This line is Asynchronously executed
Resolved value:  undefined
<10> This line is Asynchronously executed
Resolved value:  4th Promise
```

# アロー関数でreturnを省略する
さて、「Promise コンストラクタと Executor 関数」の項目において、アロー関数の省略形を解説しました。

アロー関数の省略は以下のようにでき、下の３つのコートはすべて等価です。

```js
(a) => {
  return a + 100;
}
// 等価
(a) => a + 100;
// 等価
a => a + 100;
```

アロー関数の省略形を使用することで、`return` を書かずに `return` させることができます。これは人のソースコードを読んだりするときに役立ったり、文字数を減らして書くのに役立ちます。

次のように、Promise インスタンスを返す処理は `then()` メソッドのコールバック関数のおいてアロー関数の省略形を使って書くことができます。

```js
// promiseShouldBeReturned.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`[${order}] This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

// アロー関数の省略形を使って return を省略する
returnPromise("1st Promise", "2")
  .then(() => returnPromise("2nd Promise", "6"))
  .then((value) => console.log("Resolved value: ", value));
// 同じ意味
returnPromise("3rd Promise", "3")
  .then(() => {
    // Promise インスタンスについては必ず return するようにする
    return returnPromise("4th Promise", "8")
  })
  .then((value) => console.log("Resolved value: ", value));

console.log("[4] Sync process");
```

`.then((value) => console.log("Resolved value: ", value));` については、`console.log()` の返り値は `undefined` となるので、次の `then()` メソッドのコールバックに値を渡す必要がなければやっても大丈夫です。

# 続き
かなり長くなってしまったので、続きは次の記事につづくか、後から追記する形になります。



