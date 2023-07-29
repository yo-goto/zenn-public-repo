---
title: "catch メソッドと finally メソッド"
cssclass: zenn
date: 2022-05-14
modified: 2023-01-25
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『catch メソッドと finally メソッド』
---

## このチャプターについて

このチャプターでは今まで解説していなかった `catch()` メソッド、`finally()` メソッドについて、そしてそれらメソッドが発行するマイクロタスクについて解説していきたいと思います。

## Promise のプロトタイプメソッド

### then-catch-finally

`then()` 以外の Promise のプロトタイプメソッドとして `catch()` と `finally()` メソッドが挙げられます。これは try/catch/finally に対応しており、同様の考え方で使うことができます。

```js
new Promise((resolve, reject) => {
  if (Math.random() < 0.5) {
    resolve(42);
  } else {
    reject(new Error("例外発生"));
  }
}).then(data => console.log(data))
  .catch(err => console.log(err))
  .finally(() => console.log("最後に実行される"));
```

上のように `reject()` 関数によって Promise インスタンスが Rejected 状態になった場合、チェーンしている `then()` メソッドのコールバックは実行されずに、`catch()` メソッドのコールバックが実行されて例外を捕捉します。

逆に、`resolve()` 関数によって Promise インスタンスが Fulfilled 状態になった場合には、`catch()` メソッドのコールバックは実行されません。

一方、`finally()` メソッドは Promise インスタンスが Fulfilled 状態でも Rejected 状態でも関係なく、登録しているコールバックが実行されます。

### 常に新しい Promise インスタンスが返ってくる

`then()` メソッドからは常に新しい Promise インスタンスが返ってきたように、`catch()` メソッドと `finally()` メソッドでも新しい Promise インスタンスが返ってきますので、チェーンできます。

言ったように、`finally()` メソッドに値は繋げませんので、上のコードを実行すると `undefined` が出力されます。

```sh
❯ v8 catchFinally.js
👹 エラー: Error: 例外発生
👻 これは実行される
😭 データ: undefined
👦 最後に実行される
❯ v8 catchFinally.js
🤟 データ: 42
👻 これは実行される
😭 データ: undefined
👦 最後に実行される
```

`catch()` メソッドによって返ってくる Promise インスタンスは履行状態で返ってきますので、次の `then()` メソッドのコールバックを実行できます。

### finally メソッド

`finally` メソッドは少しクセがあるのでいくつか注意点を解説しておきます。

#### コールバックは入力値を取れない

`finally()` メソッドの **コールバック関数は一切引数をとらない** ということが特徴的です。`finally` の前に chain していた値を `finally` のコールバック関数で利用することはできません。

```js
// catchFinally.js
new Promise((resolve, reject) => {
  if (Math.random() < 0.5) {
    resolve(42);
  } else {
    reject(new Error("例外発生"));
  }
})
  .then((data) => console.log("🤟 データ:", data))
  .catch((err) => console.log("👹 エラー:", err))
  .then(() => {
    console.log("👻 これは実行される");
    return 42;
  })
  .finally((data) => {
    console.log("😭 データ:", data); // undefined
    console.log("👦 最後に実行される");
  });
```

#### 値をつなぐことはできる

前のメソッドからの入力値は取れませんが、`finally` のコールバックから値を返すと次の chain へつなぐことができます。

```js
Promise.resolve(42)
  .finally(x => {
    console.log(x); // => undefined
    return 33;
  })
  .then(console.log); // => 33
```

#### 返される Promise は chain 元の Promise の値で解決される

以下のように `Promise.resolve(42)` の後に chain した `finally` メソッドから更に `then` を chain するとコールバック関数の入力値として `42` が渡ります。

```js
Promise.resolve(42)
  .finally(() => {
    console.log("FINALLY");
  }) // この promise は 42 で解決される
  .then(x => {
    console.log("THEN:", x);
    // => THEN: 42
  });
```

これはつまり、`42` で解決された promise で `finally()` を呼び出すと、同じく `42` で解決される promise になることを示しています。これらは２つの異なる promise ですが、同じ値に解決されます。

同じ様に、`Promise.reject` で値 `42` を理由にしても次の `catch` につなぐことができます。

```js
Promise.reject(42)
  .finally(() => {
    console.log("FINALLY");
  }) // この promise は 42 で拒否される
  .catch(x => {
    console.log("CATCH:", x);
    // => CATCH: 42
  });
```

`finally()` から返された promise は元の promise と同じ理由で拒否されます。

このように `finally()` で Promise の拒否理由を受け渡すことができるということは、`finally()` のハンドラを追加してもプロミスの拒否を処理したことにはならない、ということになるので注意してください。つまり、`catch` で捕捉していない場合には未処理の拒否が残ることになります。

:::message alert
拒否されたプロミスが `finally()` ハンドラしか持たない場合、JavaScript ランタイムは依然として未処理のプロミス拒否に関するメッセージを出力します。そのメッセージを回避するためには、`then()` や `catch()` で拒否ハンドラを追加する必要があります。
:::

また、上のコードの中間に `then` を挿入しても、`then()` のコールバックは実行されません (実際には後述する `x => { throw x; }` のような thrower 関数がマイクロタスクとして発行されてイベントループで処理されています)。

```js
Promise.reject(42)
  .finally(() => {
    console.log("FINALLY"); // => FINALLY
  })
  .then(x => { // 無視される
    console.log("THEN", x);
    return 2;
  })
  .catch(x => {
    console.log("CATCH:", x); // => CATCH: 42
  });
```

## コールバックは実行されなくてもマイクロタスクは発生する

### 発生するマイクロタスク

重要なこととして、`catch()` メソッドや `then()` メソッドは登録してあるコールバックが実行されないときでも実はマイクロタスクが発行されます。この原理については後で解説しますが、その現象そのものについて確認しておきましょう。

例えば、`Promise.reject()` で拒否状態の Promise インスタンスに `then()` と `catch()` メソッドをチェーンしてみます。

実行順番はどうなるでしょうか？

```js
// catchMicrotask-1.js
console.log("🦖 [A-1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [B-3] <1-Sync> MICRO: then"));

Promise.reject(new Error("Exception"))
  .then(() => console.log("👻 [C-4] <2-Sync> MICRO: then callback"))
  .then(() => console.log("👻 [D-6] <4-Sync> MICRO: then callback"))
  .catch((err) => console.log("👹 [E-8] <6-Async> MICRO: catch callback", err))
  .finally(() => console.log("🦄 [F-10] <8-async> MICRO: finally callback"));

Promise.resolve()
  .then(() => console.log("👦 [G-5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [H-7] <5-Async> MICRO: then"))
  .then(() => console.log("👦 [I-9] <7-Async> MICRO: then"));

console.log("🦖 [J-2] MAINLINE: End");
```

既に拒否状態の Promise インスタンスに対しては `then()` メソッドのコールバックは実行されずに、`catch()` メソッドのコールバックによって例外捕捉されます。ただし、マイクロタスクは発行されます。

実行順番は次のようになります。

```sh
❯ v8 catchMicrotask-1.js
🦖 [A-1] MAINLINE: Start
🦖 [J-2] MAINLINE: End
👦 [B-3] <1-Sync> MICRO: then
👦 [G-5] <3-Sync> MICRO: then
👦 [H-7] <5-Async> MICRO: then
👹 [E-8] <6-Async> MICRO: catch callback Error: Exception
👦 [I-9] <7-Async> MICRO: then
🦄 [F-10] <8-async> MICRO: finally callback
```

もし、`then()` メソッドによってマイクロタスクが発行されていなければ、次のコードの `catch()` メソッドのコールバックの実行順番は上のコードとまったく同じになるはずですが、そうはなりません。

```js
// catchMicrotask-2.js
console.log("🦖 [A-1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [B-3] <1-Sync> MICRO: then"));

Promise.reject(new Error("Exception"))
  .catch((err) => console.log("👹 [E-4] <2-Async> MICRO: catch callback", err))
  .then(() => console.log("👻 [C-6] <4-Sync> MICRO: then callback"))
  .then(() => console.log("👻 [D-8] <6-Sync> MICRO: then callback"))
  .finally(() => console.log("🦄 [F-10] <8-async> MICRO: finally callback"));

Promise.resolve()
  .then(() => console.log("👦 [G-5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [H-7] <5-Async> MICRO: then"))
  .then(() => console.log("👦 [I-9] <7-Async> MICRO: then"));

console.log("🦖 [J-2] MAINLINE: End");
```

実際に実行すると `catch()` メソッドのコールバックの実行順番が先程よりも早くなっていることが分かります。

```sh
❯ v8 catchMicrotask-2.js
🦖 [A-1] MAINLINE: Start
🦖 [J-2] MAINLINE: End
👦 [B-3] <1-Sync> MICRO: then
👹 [E-4] <2-Async> MICRO: catch callback Error: Exception
👦 [G-5] <3-Sync> MICRO: then
👻 [C-6] <4-Sync> MICRO: then callback
👦 [H-7] <5-Async> MICRO: then
👻 [D-8] <6-Sync> MICRO: then callback
👦 [I-9] <7-Async> MICRO: then
🦄 [F-10] <8-async> MICRO: finally callback
```

同様に `catch()` メソッドもコールバックが実行されなくてもマイクロタスクが発生します。

```js
// catchMicrotask-3.js
console.log("🦖 [A-1] MAINLINE: Start");
Promise.resolve().then(() => console.log("👦 [B-3] <1-Sync> MICRO: then"));

Promise.resolve()
  .catch((err) => console.log("👹 [C-4] <2-Async> MICRO: catch callback", err))
  .catch((err) => console.log("👹 [D-6] <4-Async> MICRO: catch callback", err))
  .then(() => console.log("👻 [E-8] <6-Sync> MICRO: then callback"))
  .finally(() => console.log("🦄 [F-10] <8-async> MICRO: finally callback"));

Promise.resolve()
  .then(() => console.log("👦 [G-5] <3-Sync> MICRO: then"))
  .then(() => console.log("👦 [H-7] <5-Async> MICRO: then"))
  .then(() => console.log("👦 [I-9] <7-Async> MICRO: then"));

console.log("🦖 [J-2] MAINLINE: End");
```

ということで実行順番は次のようになります。もし `catch()` メソッドがマイクロタスクを発行しなければ、`[E-8]` の実行順番は `[G-5]` よりも先に来るはずですが、そうはなりません。

```sh
❯ v8 catchMicrotask-3.js
🦖 [A-1] MAINLINE: Start
🦖 [J-2] MAINLINE: End
👦 [B-3] <1-Sync> MICRO: then
👦 [G-5] <3-Sync> MICRO: then
👦 [H-7] <5-Async> MICRO: then
👻 [E-8] <6-Sync> MICRO: then callback
👦 [I-9] <7-Async> MICRO: then
🦄 [F-10] <8-async> MICRO: finally callback
```

### catch と finally は then メソッドを利用する

上記のような登録してあるコールバック関数が実行されなくても、マイクロタスクが必ず実行される理由は、`catch` メソッドと `finally` メソッドが内部的に `then` メソッドを利用していることと更にもう一つ理由があります。

まずは `catch` と `finally` が `then` (`Promise.prototype.then`) を利用していることを確認しておきましょう。これについて詳しくは『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで解説しますが、ECMAScript 仕様を見てます。

[Promise.prototype.catch(onRejected)](https://tc39.es/ecma262/#sec-promise.prototype.then) のアルゴリズムステップは引数 `onRejected` (コールバック) を取って以下のように実行されます。

> - 1. Let promise be the this value.
> - 2. Return ? [Invoke](https://tc39.es/ecma262/#sec-invoke)(promise, "then", « undefined, onRejected »).

２行目では [Invoke](https://tc39.es/ecma262/#sec-invoke) という抽象操作を利用して、`promise["then"](undefined, onRejected)` を起動しています。`"then"` はただのブラケット記法です。つまり、`catch` メソッドの呼び出しは実際には `promise.then(undefined, onRejected)` を呼び出しているということになります。

[Promise.prototype.finally(onFinally)](https://tc39.es/ecma262/#sec-promise.prototype.finally) のアルゴリズムステップはもっと複雑ですが、アルゴリズムステップの以下の最終行では `catch` と同じように [Invoke](https://tc39.es/ecma262/#sec-invoke) 抽象操作を使って、`promise.then(thenFinally, catchFinally)` を呼び出します。

> - 7. Return ? [Invoke](https://tc39.es/ecma262/#sec-invoke)(promise, "then", « thenFinally, catchFinally »).

`thenFinally` と `catchFinally` というコールバック関数は `finally` メソッドに渡すコールバック関数 `onFinally` が呼び出し可能な関数であるときにはそのまま `onFinally` が利用されます。

逆に `onFinally` が呼び出し可能なメソッドではない文字列や `undefined` のときには内部的に作成される関数で自動的に置換されます。`finally` と `then` ではこのような自動的な関数の置換が発生しており、これが登録していないコールバックとなってマイクロタスクとして処理されています。`catch` も内部的に `then` を使っているのでこの置換が発生しています。

### identity 関数と thrower 関数

`then(undefined, undefined)` のようにコールバック関数を未定義で呼び出すと上記で説明したような関数の置換が自動的に発生します。

`then` メソッドは `then(onFulfilled, onRejected)` というフォーマットですが、`onFulfilled` に自動置換される関数は identity 関数で、`onRejected` に自動置換される関数は thrower 関数と呼ばれます。

identity 関数は日本語では「恒等関数」とも呼ばれ、`(x) => x` のように引数をそのまま return するような関数です。一方、thrower 関数は `(x) => { throw x; }` のように引数をそのまま throw するような関数です。昔の仕様ではこれらの関数のことが明確に言及されていましたが、実際の仕様的には架空の関数であり、PromiseReactionJob 内部から動作を決定できるようにするためとの理由で ECMAScript の仕様から以下の PR で言及部分が削除されてしまいました。

https://github.com/tc39/ecma262/pull/584

この PR での顕著な変更点は以下の箇所です。Identity と Thrower の言及が削除されています。

```diff
- The function that should be applied to the incoming value, and whose return value will govern what happens to the derived promise. If [[Handler]] is `"Identity"` it is equivalent to a function that simply returns its first argument. If [[Handler]] is `"Thrower"` it is equivalent to a function that throws its first argument as an exception.
+ The function that should be applied to the incoming value, and whose return value will govern what happens to the derived promise. If [[Handler]] is *undefined*, a function that depends on the value of [[Type]] will be used instead.
```

identity 関数と thrower 関数の説明は仕様の外での解説でよく利用されるもので、MDN の [Promise.prototype.then()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then) のページでは上で解説したような関数でそのまま解説されています。

![identity-thower](/images/js-async/img_identity-thrower-functions.jpg)*[Promise.prototype.then() - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then) より*

ただし、現在の仕様からは上記の identity 関数と thrower 関数の記述が削除されてしまったので、仕様内で identity などをいくら検索しても正確にヒットすることはありません。(※ `finally` メソッドについては別途 identity に相当する valueThunk 関数と thrower 関数があり、そちらで引っかかることがあるのでややこしい)。

:::details 仕様解説
『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで解説していますが、実は identity 関数と thrower 関数の挙動実体は [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob) で作成される抽象クロージャであり、その挙動の主要な部分は [CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions) と呼ばれる操作で作成される `resolve` 関数と `reject` 関数です。これは `new Promise(executor)` で Promise インスタンスを作成するときに `executor` 関数の引数として渡す `resolve` 関数と `reject` 関数そのものです。それらが呼び出される箇所は [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob) の以下のステップです。

> - h. If handlerResult is an [abrupt completion](https://tc39.es/ecma262/#sec-completion-record-specification-type), then
>   - i. Return ? [Call](https://tc39.es/ecma262/#sec-call)(promiseCapability.\[\[Reject\]\], undefined, « handlerResult.\[\[Value\]\] »).
> - i. Else,
>   - i. Return ? [Call](https://tc39.es/ecma262/#sec-call)(promiseCapability.\[\[Resolve\]\], undefined, « handlerResult.\[\[Value\]\] »).
:::

仕様について解説してもここでは何を言ってるのか分かりづらいと思うので、内部置換されるコールバック関数についてはそのまま `(x) => x` という identity 関数と `(x) => { throw x; }` という thrower 関数であると考えておけばよいです。関数の実体が気になる場合には [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob) と [CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions) 操作の仕様を確認するようにしてください。

それでは上記の identity 関数と thrower 関数で自動置換されるというのはどのようなことかイメージできるようにサンプルを使って確認します。

まず `then` メソッドの場合ですが、`then(onFulfilled, onRejected)` の呼び出しで引数となるコールバック関数が両方とも省略されて、`undefined` になっている場合には `onFulfilled` は identity 関数に置換され、`onRejected` は thrower 関数に置換されます。

```js
Promise.resolve(42)
  .then() // then(x => x, x => { throw x; })
  .then(x => console.log(x)); // => 42
```

上記の Promise chain で一個目の `then()` の引数は省略されており、コールバック関数は両者とも置換されて、`then(x => x, x => { throw x; })` として呼び出されます。

`Promise.resolve(42)` は最初から履行しているので、chain している `then` メソッドの履行用のコールバック関数 `x => x` がマイクロタスクとして発行されることになります。これによって、次の `then` のコールバック関数の引数として値 `42` を渡してコンソール出力することが可能となります。

:::message alert
**JS Visualizer について**

JS Visualizer ではこのような内部置換されたコールバック関数として発行されるマイクロタスクは視覚化されないので注意してください。したがって、以下のようなコードでは実行順序がなぜそうなるか理解できないケースとなります。

```js
console.log("[1] Sync");

// 内部置換されるコールバック関数のマイクロタスクは可視化されない
Promise.resolve(42)
  .then() // then(x => x, x = { throw x; })
  .then() // then(x => x, x = { throw x; })
  .then(function A(x) { console.log("[4]", x); })

// 内部置換されるコールバック関数のマイクロタスクは可視化されない
Promise.resolve(88)
  .then() // then(x => x, x = { throw x; })
  .then(function B(x) { console.log("[3]", x); })

console.log("[2] Sync");
```

👉 [JS Visualizer code](https://www.jsv9000.app/?code=Y29uc29sZS5sb2coIlsxXSBTeW5jIik7CgpQcm9taXNlLnJlc29sdmUoNDIpCiAgLnRoZW4oMzMpIC8vIHRoZW4oeCA9PiB4LCB4ID0geyB0aHJvdyB4OyB9KQogIC50aGVuKDU1KSAvLyB0aGVuKHggPT4geCwgeCA9IHsgdGhyb3cgeDsgfSkKICAudGhlbihmdW5jdGlvbiBBKHgpIHsgY29uc29sZS5sb2coIls0XSIsIHgpOyB9KQoKUHJvbWlzZS5yZXNvbHZlKDg4KQogIC50aGVuKDMzKSAvLyB0aGVuKHggPT4geCwgeCA9IHsgdGhyb3cgeDsgfSkKICAudGhlbihmdW5jdGlvbiBCKHgpIHsgY29uc29sZS5sb2coIlszXSIsIHgpOyB9KQoKY29uc29sZS5sb2coIlsyXSBTeW5jIik7Cg%3D%3D)
:::

次に `then` の第一引数に `55` という数値を渡した場合にどうなるかを考えます。

```js
Promise.resolve(42)
  .then(55) // then(x => x, x => { throw x; })
  .then(x => console.log(x)); // => 42
```

実はコールバック関数が呼び出し可能ではない場合にも `x => x` という identity 関数に置換されます。`55` という数値は関数として呼び出せないのでこの数値自体が無視されて `x => x` という関数に置換されます。したがって、元の Promise が持つ履行値 `42` という値が Promise で連鎖して、コンソール出力される数値は `42` となります。

次に chain 元の Promise を `Promise.reject` を使って始めから拒否状態として二番目のメソッドを `catch` としてみます。

```js
Promise.reject(42)
  .then() // then(x => x, x => { throw x; })
  .catch(x => console.log(x)); // => 42
```

先程の例と同じ用に `then` のコールバックは両者ともに省略されて `undefined` なので、関数の置換が起きて `then(x => x, x => { throw x; })` として呼び出されます。

chain 元の Promise インスタンスは拒否理由 `42` で拒否されているため、`.then()` では拒否用のコールバック関数として内部置換された `x => { throw x; }` の thrower 関数がマイクロタスクとして発行されます。値 `42` が例外として throw されますが、`.catch(x => console.log(x))` で捕捉されて次のマイクロタスクとなる `x => console.log(x)` でコンソールに例外値 `42` が出力されます。

次に一番目の `then` メソッドを `catch` にしてみます。`catch(onRejected)` は内部的に `then(undefined, onRejected)` を呼び出すので、`undefined` は置換されて結局 `then(x => x, x => {throw x; })` が呼び出されます。

```js
Promise.reject(42)
  .catch() // then(x => x, x => { throw x; })
  .catch(x => console.log(x)); // => 42
```

chain 元の Promise インスタンスは拒否理由 `42` で拒否されているため、`.catch()` では拒否用のコールバック関数として内部置換された `x => { throw x; }` の thrower 関数がマイクロタスクとして発行されます。値 `42` が例外として throw されますが、`.catch(x => console.log(x))` で捕捉されて次のマイクロタスクとなる `x => console.log(x)` でコンソールに例外値 `42` が出力されます。

コールバック関数が実行されていないように見えたとしても、Promise chain において履行値や拒否理由の伝達が可能となっているのは、このように内部置換された identity 関数や thrower 関数がマイクロタスクとして発行されてイベントループで処理されているからです。
