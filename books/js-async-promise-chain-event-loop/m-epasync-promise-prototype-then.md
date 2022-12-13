---
title: "Promise.prototype.then メソッドの仕様挙動"
cssclass: zenn
date: 2022-11-02
modified: 2022-12-14
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases:
  - Promise本『Promise.prototype.then メソッドの仕様挙動』
  - thenメソッドの勘違いの修正
---

# このチャプターについて

このチャプターでは、`Promise.prototype.then` メソッドについて欠落していた情報を解説します。

`then` メソッドについては、この [第２章](sec-02-epasync) で解説していましたが、情報の欠落と間違いがあったため、このチャプターでまとめて補足・修正しておきたいと思います。

誤りが波及しているチャプターは以下のものですが、これらのチャプターについても順次修正していく予定です。

- [Promise chain で値を繋ぐ](7-epasync-pass-value-to-the-next-chain)
- [then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)
- [Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)
- [コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)

それでは、さっそく、この本の解説で間違っていた点について例を出して解説していきます。

## 参考文献

こちらのチャプターの解説で使用する ECMASciript の仕様のコードは以下のブログ記事のシリーズのものを参考にさせていただきました。非常に分かりやすい解説されているので、自分自身で Promsie の仕様を実装して学びたい場合には是非参考にしてください。

https://humanwhocodes.com/blog/2020/09/creating-javascript-promise-from-scratch-constructor/

https://humanwhocodes.com/blog/2020/09/creating-javascript-promise-from-scratch-resolving-to-a-promise/

https://humanwhocodes.com/blog/2020/10/creating-javascript-promise-from-scratch-then-catch-finally/

また ECMAScript の仕様そのものを理解するためには以下の記事が参考になるので興味があれば参考にしてみてください。

https://timothygu.me/es-howto/

https://v8.dev/blog/understanding-ecmascript-part-1

# then メソッドのコールバックから返る値の違いによる挙動

結論から言うと、`Promise.prototype.then()` メソッドにわたす **コールバック関数から返る値の種類によって発生するマイクロタスクの数が異なる** ことが仕様から判明しました。

`.then(callback)` のように渡したコールバック関数 `callback` がマイクロタスクとして発行されるという話はそのままですが、`callback` から `.then` メソッドを持つオブジェクトが返ると、追加でマイクロタスクが発生します。

その原理については後で詳しく解説するので、発生する現象についてここから見ていきます。

まずは今まで行ってきた解説で完全に説明できる例です。

以下のコードでは、`then()` メソッドのコールバック関数で数値などの普通の値を返す２つの Promise chain を用意して競争させています。このコードの実行順序予測はこれまでの説明通りで、実際実行してみると予想通りになることがわかります。

```js:simpleValue.js
// <> は発生しているマイクロタスクの追跡順番
console.log("🦖 [A]");

Promise.resolve()
  .then(() => { // <1>
    console.log("💙 [B]")
    return 1;
  })
  .then((d) => { // <3>
    console.log("💙 [C]", d);
  });

Promise.resolve()
  .then(() => { // <2>
    console.log("💚 [D]");
    return 2;
  })
  .then((d) => { // <4>
    console.log("💚 [E]", d);
  })
  .then(() => { // <5>
    console.log("💚 [F]")
  });

console.log("🦖 [G]");

/* RESULT
❯ deno run simpleValue.js
🦖 [A]
🦖 [G]
💙 [B]
💚 [D]
💙 [C] 1
💚 [E] 2
💚 [F]
*/
```

さて、次にこれまでの解説では予測がうまくいかない例となるコードを出してみます。以下のコードでは、上とほぼ同じの２つの Promise chain を競争させますが、一方の chain では `then()` メソッドのコールバックで Promise オブジェクトを返し、もう片方では数値のような普通の値を返してみます。

上のコードと同じ実行順番になる予測してみると、出力の順序が予想通りにならないことがわかります。

```js:complexPattern.js
console.log("🦖 [A]");

Promise.resolve()
  .then(() => {
    console.log("💙 [B]")
    return Promise.resolve(1);
  })
  .then((d) => {
    console.log("💙 [C]", d);
  });

Promise.resolve()
  .then(() => {
    console.log("💚 [D]");
    return 2;
  })
  .then((d) => {
    console.log("💚 [E]", d);
  })
  .then(() => {
    console.log("💚 [F]")
  });

console.log("🦖 [G]");

/* RESULT
❯ deno run complexPattern.js
🦖 [A]
🦖 [G]
💙 [B]
💚 [D]
💚 [E] 2
💚 [F]
💙 [C] 1
*/
```

このように `then()` メソッドのコールバックから返る値の種類によってマイクロタスクの発生数が異なるため、現在行っている解説だとこのようなコードの実行順序が理解できなくなります。

これまで見てきた Promise chain の比較であれば、💙 と 💚  が順番に出力されてくるはずですが、そうはなりません。

この理由としては、最初に言ったとおり `then()` メソッドに渡すコールバックから Promise などの `.then()` メソッドを持つオブジェクト (**Thenable** と呼ばれるオブジェクト) が返されると、もとの `then()` メソッドから返される Promsie を解決するために通常の値を返す時に加えて追加で２つのコールバックがマイクロタスクとして発生するためです。

これまでの解説だと、上記のような普通の値を返す chain と Promise を返す chain の比較がなかったため、Promise を返している chain においても一見正しい解説のように見えてしまっているはずですが、コールバック関数で Promise を返している場合、**実際にはマイクロタスクが追加で２つ発生しています**。

実際、もう片方の chain の `then` コールバックから返る値を Promsie オブジェクトに変更すれば最初のコードと同じ出力順番にすることができます。

```js:fakePattern.js
console.log("🦖 [A]");

Promise.resolve()
  .then(() => {
    console.log("💙 [B]")
    return Promise.resolve(1);
  })
  .then((d) => {
    console.log("💙 [C]", d);
  });

Promise.resolve()
  .then(() => {
    console.log("💚 [D]");
    return Promise.resolve(2); // Promise にした
  })
  .then((d) => {
    console.log("💚 [E]", d);
  })
  .then(() => {
    console.log("💚 [F]")
  });

console.log("🦖 [G]");

/* RESULT
❯ deno run fakePattern.js
🦖 [A]
🦖 [G]
💙 [B]
💚 [D]
💙 [C] 1
💚 [E] 2
💚 [F]
*/
```

これは、発生するマイクロタスクの数をつじつま合わせして出力の順序を調整しただけで、実は最初のコードよりも発生するマイクロタスクが増加しています。

:::message alert
さきに言っておくと、この挙動の違いを明確に理解するには ECMAScript の仕様をしっかり見る必要があります。とはいえ、仕様自体が非常に解読困難で冗長なものでもあるので、掻い摘んで概要を説明していきたいと思います。
:::

# Thenable とは

まず始めに、Thenable について解説しておきます。

Thenable とは `.then` というメソッドを持つオブジェクトの総称です。例えば、Promise は `.then` メソッドを持つので Thenable であると言えます。また、自分で適当な挙動の `.then` メソッドを実装したオブジェクトの場合でも Thenable と言えます。つまり、**Thenable であるからと言って必ずしも Promise だとは限らない** ので注意してくだださい。

以下のように `then` メソッドが実装されていれば、await や promise chain で利用でき、実装されている `then` メソッドが実行されます。

```js
const thenable = {
  then: (onFulfilled) => {
    setTimeout(() => onFulfilled(42), 100);
  },
};

(async () => {
  const v = await thenable;
  console.log(v); // => 42
})();
```

このように `.then` メソッドを持っていれば、await 評価や Promise chain で promise オブジェクトと同じように扱えます。このような動作があるのは、Promsie 自体が元はそれを実装するコミュニティベースのライブラリがいくつかあり、あとになって仕様に導入されるようになった経緯があるからです。

つまり、Promise 以外の `.then` を持つオブジェクト (ECMAScirpt 実装ではない Promise) などがネイティブの Promise のように扱えるようにした仕組みが Thenable と言えます。

さて、実はそういったオブジェクトそのものを使いたいからこの概念の説明をしたわけではありません。問題である Promise.prototype.then の挙動について説明するのに必要なのでこの概念の解説をしています。

# 仕様の基礎

ここからは JavaScript の仕様である ECMAScript の基礎について解説していきます。最終的なぜそのような動作になるかを自分で確認できるように仕様の URL をワードに貼り付けておきます。

## 抽象操作とアルゴリズムステップ

まず ECMAScript の仕様には「**抽象操作 (Abstract Operation)**」というものが記述されています。

この抽象操作とは、ECMAScript 仕様の内部で利用される操作で、プログラマーからはアクセスできません。意味合いとしては「長い表記を省略するため」というのが大きく、V8 などの JS エンジン側で実装はまかされています。

仕様を理解するにはこの抽象操作によるアルゴリズムステップを理解していくことになります。

Promise 系列の仕様を理解するために必要な抽象操作は [Promise Abstract Operations](https://tc39.es/ecma262/#sec-promise-abstract-operations) の項目に記載されています。ただし、ここに記述されている抽象操作からは他の項目にある抽象操作も呼び出されるので理解するのには様々な操作をたどっていく必要があります。

例えば、Promsie の静的メソッドである [Promise.resolve](https://tc39.es/ecma262/#sec-promise.resolve) は内部的に大半の作業を `PromiseResolve` という抽象操作に委任しています。

![algorithm-steps](/images/js-async/img_ecma-algorithm-step.jpg)

このような各アルゴリズムステップを見て関係を理解するのは大変なので、コアとなる抽象操作については図でまとめておきました。それぞれの操作は以下のような呼び出し関係となっています。

![promise抽象操作](/images/js-async/PromiseSpec.excalidraw.png)

例えば、`Promise.prototype.catch` や `Promise.prototype.finally` といったプロトタイプメソッドは、実は大半の作業を `Promise.prototype.then` にまかせています。

## Job

図一番下の [HostEnqueuPromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob) という操作が最終的にマイクロタスクをマイクロタスクキューへとエンキューする操作です。

:::message
だたし、ECMAScript では「マイクロタスク」は [Job](https://tc39.es/ecma262/#job) と呼ばれるものとして扱われていることに注意してください。マイクロタスク自体はあくまで WHATWG 仕様に定義されているものです。
:::

Job というのは実際には内部的な仕様の型 (Specifiaction type) である抽象クロージャ ([Abstact Closure](https://tc39.es/ecma262/#sec-abstract-closure)) という型の値です。

Promise 関連の Job (マイクロタスク) を作る直接の注意操作は以下の２つとなります。そして、この２つがこれまでの問題の中心とります。

- [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob)
- [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob)

# CreateResolvingFunctions

さて、ここからは本格的に問題となる仕様の部分について触れていきます。

`Promise.prototype.then` のコールバックから返される値によって異なる挙動が起きますが、この問題を起こしている抽象操作の実体は [CreateResolvingFunctinos](https://tc39.es/ecma262/#sec-createresolvingfunctions) で作成される [Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions) という関数です。

仕様的にはこのような名前がついていますが、その正体はかなり身近なもので、`new Promise(executor)` という Promsie インスタンス作成時に渡すコールバック関数 `executor` の引数となる `resolve` 関数です。

```js
const p = new Promise((resolve, reject) => {
  //                  ^^^^^^^↑ これ
  resolve(42); // この関数が Promise Resolve Functions で作成される関数
});

p.then(console.log); // => 42
```

今までなんとなく `resolve()` という関数を使ってきましたが、この関数が「実際にやること」が定義されているのが Promise Resolve Functions です。

Promise コンストラクタを使ってプロミスインスタンスを作成すると、CreateResolvingFunction 抽象操作が呼び出され、解決用の関数である [Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions) と拒否用の関数である [Promise Reject Functions](https://tc39.es/ecma262/#sec-promise-reject-functions) が作成されます。そしてこれらことが今まで見てきた `resolve` 関数と `reject` 関数そのものです。

『[Promise の基本概念](a-epasync-promise-basic-concept)』や『[Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)』のチャプターでも説明しましたが、`reject` 関数に比べて、`resolve` 関数は非常に複雑です。これは Unwrapping の能力が関係しており、`reject` 関数では引数の `reason` に何を渡そうが拒否すると決まっていますが、`resolve` 関数では引数にわたす `resolution` (解決値) の値の種類よって処理内容が変わるので、複雑になっています。

`Promise.prototype.then` でもこの `resolve` 関数が実は使われており (図示してある)、コールバックから返される値が引数の `resolution` (解決値) として使われます。従って、`resolve` 関数の処理の場合分けを考える必要があるという話になります。

# then から resolve 関数が呼び出される仕組み

`Promise.prototype.then` から `resolve` 関数が呼び出されるというのは仕様を見てもかなり分かりづらいので補足しておきます。

`Promise.prototype.then` が呼び出されると PerformPromiseThen 抽象操作が呼び出される前に [Promise コンストラクタ](https://tc39.es/ecma262/#sec-promise-executor) によって新しい Promise インスタンスが作成されます。実際には以下のステップで発生しています。

> 3. Let C be ? [SpeciesConstructor](https://tc39.es/ecma262/#sec-speciesconstructor)(promise, [%Promise%](https://tc39.es/ecma262/#sec-promise-constructor)).
> 4. Let resultCapability be ? [NewPromiseCapability](https://tc39.es/ecma262/#sec-newpromisecapability)(C).

[NewPromiseCapability](https://tc39.es/ecma262/#sec-newpromisecapability) 抽象操作はコンストラクタを引数にとって、新しい Promise オブジェクトを作成し、内部フィールドにその参照を格納します。仕様のステップでは以下の場所で [Construct](https://tc39.es/ecma262/#sec-construct) 抽象操作を使ってコンストラクタを起動させます。

> 6. Let promise be ? [Construct](https://tc39.es/ecma262/#sec-construct)(C, « executor »).

そして、[Promise コンストラクタ](https://tc39.es/ecma262/#sec-promise-executor) の仕様の次のステップで CreateResolvingFunctions が呼び出され、`resolve` 関数と `reject` 関数が作成されています。

> 8. Let resolvingFunctions be [CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions)(promise).

その後 then メソッド内部の処理の続きとして [PerformPromiseThen](https://tc39.es/ecma262/#sec-promise.prototype.then) 抽象操作が呼び出され、仕様の以下のステップ 10-11 のステップで promise の状態によって処理が振り分けられますが、どちらの場合でも [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob) が呼び出されます。

> 10. Else if promise.\[\[PromiseState\]\] is fulfilled, then
>   a. Let value be promise.\[\[PromiseResult\]\].
>   b. Let fulfillJob be [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob)(fulfillReaction, value).
>   c. Perform [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)(fulfillJob.\[\[Job\]\], fulfillJob.\[\[Realm\]\]).
> 11. Else,
>   a. [Assert](https://tc39.es/ecma262/#assert): The value of promise.\[\[PromiseState\]\] is rejected.
>   b. Let reason be promise.\[\[PromiseResult\]\].
>   c. If promise.\[\[PromiseIsHandled\]\] is false, perform [HostPromiseRejectionTracker](https://tc39.es/ecma262/#sec-host-promise-rejection-tracker)(promise, "handle").
>   d. Let rejectJob be [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob)(rejectReaction, reason).
>   e. Perform [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)(rejectJob.\[\[Job\]\], rejectJob.\[\[Realm\]\]).

NewPromiseReactiobJob は上で説明したように Job (マイクロタスク) を作成するための操作です。

この NewPromiseReactionJob 操作を呼び起こすと、then 操作の最初で NewPromiseCapability 操作をつかって内部的に作り出した CreateResolvingFunctions の関数を呼び出したことになるため、`resolve` 関数の呼び出しが起きています。

実際に `resolve` 関数や `reject` 関数が呼び出されるのは仕様の以下のステップです。[PromiseCapability Record](https://tc39.es/ecma262/#sec-promisecapability-records) (promise の状態にアクセスできるようにしたヘルパーオブジェクトのようなもの) のフィールド `[[Resolve]]` と `[[Reject]]` にはそれぞれ `resolve` 関数と `reject` 関数が保持されており、[Call](https://tc39.es/ecma262/#sec-call) 抽象操作で関数の呼び出しを行っています。

> h. If handlerResult is an [abrupt completion](https://tc39.es/ecma262/#sec-completion-record-specification-type), then
>   i. Return ? [Call](https://tc39.es/ecma262/#sec-call)(promiseCapability.\[\[Reject\]\], undefined, « handlerResult.\[\[Value\]\] »).
> i. Else,
>   i. Return ? [Call](https://tc39.es/ecma262/#sec-call)(promiseCapability.\[\[Resolve\]\], undefined, « handlerResult.\[\[Value\]\] »).

このようにして仕様内部では `Promise.prototype.then` から `resolve` 関数と `reject` 関数が呼び出せるようになっています。

# Promise Resolve Functions

さて、いよいよ本題に入ります。CreateResolvingFunctions 抽象操作で作成される [Promise Resolve Function](https://tc39.es/ecma262/#sec-promise-resolve-functions) が `resolve` 関数の正体であり、冒頭の問題が発生する仕様箇所です。

この関数のアルゴリズムステップを理解できれば問題解決となります。

実際にこの操作で作成される関数を JavaScript で実装してみます。細かい部分は気にせずに重要な箇所だけ考えていき、関数として記述できる step.4 から 16 までを考えて、徐々に埋めてきましょう。

```js:resolve 関数
const resolve = (resoltion) => {
  // ...仕様のアルゴリズムステップを実装
}
```

step.1 から step.4 までは関数作成のためのただの準備です。

> 1. Let F be the [active function object](https://tc39.es/ecma262/#active-function-object).
> 2. [Assert](https://tc39.es/ecma262/#assert): F has a \[\[Promise\]\] internal slot whose value [is an Object](https://tc39.es/ecma262/#sec-object-type).
> 3. Let promise be F.\[\[Promise\]\].
> 4. Let alreadyResolved be F.\[\[AlreadyResolved\]\].

ただし、ECMAScript の仕様の読むために必要な情報があるのでいくつか解説しておきます。

ECMAScript のアルゴリズムステップにおける "Lex x be someValue" という記述は名前付きエイリアスの作成を意味しています。`someValue` という値に対して `x` という名前付きエイリアスを付けて、後続のアルゴリズムステップで扱いやすいようにしているだけです。上のステップでいえば、`F.[[Promise]]` と `F.[[AlreadyResolve]]` の値にそれぞれ `promise` と `alreadyResolve` という名前付きエイリアスを付けています。

ここにはないですが、"Set x to someOtherValue" という記述であれば、エイリアスが参照する値を修正していることになります。

"Assert:" で始まるステップについては、実装時になにかを行うためのステップではなく、暗黙的なアルゴリズムの条件をわざわざ明示化してくれているだけです。上記の step.2 ではアクティブな関数オブジェクトである `F` がオブジェクトを値として持つ `[[Promise]]` という内部スロットを備えているということを我々に明確にするために教えてくれています。

:::message
これらの内容は [Algorithm Conventions](https://tc39.es/ecma262/#sec-algorithm-conventions) の項目に記載されているので気になる場合には確認してください。
:::

内部スロットとは、ECMAScript におけるオブジェクトの内部状態を保持するために用意されているスロットのことです。`[[notation]]` のように仕様で表記されます。この表記法はコンテキストによって意味が変わるので注意してください。場合によっては、オブジェクトが仕様上持ちうる内部メソッドや [Record](https://tc39.es/ecma262/#sec-list-and-record-specification-type) (JS でいうオブジェクトのようなもの) のフィールドを指していることがあります。つまり、以下のような３つの事柄のどれかを指しています。

- Record のフィールド
- JavaScript オブジェクトの内部スロット
- JavaScript オブジェクトの内部メソッド

さて、step.1 から step.4 まではこんな感じですが、実は Promise Resolve Functinos のアルゴリズムステップの前にはこのような記述がありました。

> A promise resolve function is an anonymous built-in function that has [[Promise]] and [[AlreadyResolved]] internal slots.
>
> When a promise resolve function is called with argument resolution, the following steps are taken:

Promise Resole Function は `[[Promise]]` と `[[AlradyResolve]]` という内部スロットを持つ無名のビルトイン関数 (ECMAScript に備え付けの関数) であり、引数 `resolution` で呼び出されることで記述されているアルゴリズムステップを実行するとのことです。

さて、この引数 `resolution` が重要です。`resolve` 関数が呼び出されるのは以下のような形式となっていますね。

```js
resolve(resolution);
```

`new Promise(executor)` の形式であれば、`resolve()` 関数の引数として `resolution` にわたすのは新しくインスタンス化する Promise オブジェクトの内包させて履行させたい値でした。そして、Promise.prototype.then のコールバック関数で返す値もこの `resolution` として扱われて最終的に `resolve` 関数が呼び出されるわけです。

`resolve` 関数の実装では、この引数 `resolution` の値の種類によって処理が変わるので場合分けすることになります。その箇所は以下の仕様の step.7 から step.16 となります (step.5-6 は省略します)。

> 7. If [SameValue](https://tc39.es/ecma262/#sec-samevalue)(resolution, promise) is true, then
>   a. Let selfResolutionError be a newly created TypeError object.
>   b. Perform [RejectPromise](https://tc39.es/ecma262/#sec-rejectpromise)(promise, selfResolutionError).
>   c. Return undefined.
> 8. If resolution [is not an Object](https://tc39.es/ecma262/#sec-object-type), then
>   a. Perform [FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise)(promise, resolution).
>   b. Return undefined.
> 9. Let then be [Completion](https://tc39.es/ecma262/#sec-completion-ao)([Get](https://tc39.es/ecma262/#sec-get-o-p)(resolution, "then")).
> 10. If then is an [abrupt completion](https://tc39.es/ecma262/#sec-completion-record-specification-type), then
>   a. Perform [RejectPromise](https://tc39.es/ecma262/#sec-rejectpromise)(promise, then.\[\[Value\]\]).
>   b. Return undefined.
> 12. If [IsCallable](https://tc39.es/ecma262/#sec-iscallable)(thenAction) is false, then
>   a. Perform [FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise)(promise, resolution).
>   b. Return undefined.
> 13. Let thenJobCallback be [HostMakeJobCallback](https://tc39.es/ecma262/#sec-hostmakejobcallback)(thenAction).
> 14. Let job be [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob)(promise, resolution, thenJobCallback).
> 15. Perform [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)(job.\[\[Job\]\], job.\[\[Realm\]\]).
> 16. Return undefined.

ちょっと長いですが、やっていることは `resolve` 関数の引数 `resolution` の値によって処理を振り分けて、[FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise) か [RejectPromise](https://tc39.es/ecma262/#sec-rejectpromise) を使って promise オブジェクトの状態を pending から fulfilled または rejected へと遷移させ、関数自体から `undefined` を返して終了させています。

step.10 以降の処理は例外が起きた場合やその場合分けに適さなかった残りの場合の処理を記述していることになります。

`resolution` (解決値) として考えれられる値の種類は以下のようなケースが考えられ、実際それぞれのステップで場合分けされます。

1. 解決値が「元の promise そのもの」の場合 (step.7)
2. 解決値が「オブジェクトでない値」の場合 (step.8)
3. 解決値が「解決値がオブジェクト」の場合 (step.9)
  1. `then` プロパティがメソッドでない場合 (`undefind` である場合も含む) (step.12)
  2. `then` プロパティがメソッドの場合、つまりそのオブジェクトは Thenable である場合 (step.13-16)

それでは、まずは step.7 から step.11 を実装してみます。

```js:resolve 関数
const resolve = (resoltion) => {

  // step.7 (a/b/c) : 「元の promise そのもの」の場合では解決できないので直ちに拒否する
  if (Object.is(resolution, promise)) {
    const selfResolutionError = new TypeError("Cannot resolve to self");
    RejectPromise(promise, selfResolutionError);
    return undefined; // 関数終了
  }

  // stpe.8 (a/b) : 「オブジェクトではない値」の場合には直ちに履行する
  if (!isObject(resolution)) {
    // promise を fulfill する
    FulfillPromise(promise, resolution);
    return undefined; // 関数終了
  }

  let thenAction;
  try {
   // step.9/11
    thenAction = resolution.then;
    // 解決値の .then を取得
  } catch (thenError) {
    // step.10 (a/b) : then へのアクセスで例外が発生したら直ちに拒否する
    RejectPromise(promise, thenError);
    return undefined; // 関数終了
  }

  // ...
}
```

promsie オブジェクトの内部状態を実際に遷移させるのは [FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise) と [RejectPromise](https://tc39.es/ecma262/#sec-rejectpromise) という抽象操作を利用します。

この操作が実行されると、promise の内部状態を書き換えて、[TriggerPromiseReactions](https://tc39.es/ecma262/#sec-triggerpromisereactions) という抽象操作が実行されます。この操作内の [HostEnqueuPromieJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob) 操作によって、もし次の chain している `then` や `catch` などがあればそのコールバックをマイクロタスクとして発火させます。

ここまでで、解決値が「元の promise」の場合と「オブジェクトではない値」の場合のケースの処理がカバーできました。つまり `reslution` として渡される値が文字列や数値などの値である場合には直ちに履行することができることがわかります。そして、履行したら `return undefined` で `resolve` 関数自体の処理が終了するので、残りのアルゴリズムステップは無視できることになります。

残されたケースは `resolution` の値が「オブジェクトの場合」であり、さらに「then メソッドを持つかどうか」、つまり thenable であるかどうかの処理の振り分けを行う step.12 から step.16 までを実装します。

> 12. If [IsCallable](https://tc39.es/ecma262/#sec-iscallable)(thenAction) is false, then
>   a. Perform [FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise)(promise, resolution).
>   b. Return undefined.

step.12 では [IsCallable](https://tc39.es/ecma262/#sec-iscallable) という抽象操作を使って、オブジェクトの `.then` が呼び出し可能なメソッドであるかどうかを判定し、呼び出し不可能な場合には、`resolution` が `.then` メソッドを持たないオブジェクトであることがわかり、FulfillPromise を使って直ちに履行して `return undefined` で `resolve` 関数を終了させます。

そうではない場合には、呼び出し可能な `.then` メソッドを持つオブジェクトである、つまり Thenable であることが判明します。ここからがこのチャプターの中枢となります。

`resolution` が Thenable である場合の処理を step.13 から step.16 で行います。

> 13. Let thenJobCallback be [HostMakeJobCallback](https://tc39.es/ecma262/#sec-hostmakejobcallback)(thenAction).
> 14. Let job be [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob)(promise, resolution, thenJobCallback).

step.13 では、マイクロタスクとして実行するコールバック関数を [HostMakeJobCallback](https://tc39.es/ecma262/#sec-hostmakejobcallback) 抽象操作で作成します。返ってくるのは JobCallback Record というコールバック関数への参照を可能したデータです。step.14 ではこのデータから [NewPromiseThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) という抽象操作で、マイクロタスクの実体である Job を作成します。

> 15. Perform [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)(job.\[\[Job\]\], job.\[\[Realm\]\]).
> 16. Return undefined.

そして、[HostEnqueuPromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob) 操作によって実際にマイクロタスクとしてエンキューし、`return undefined` で `resolve` 関数を終了します。

これを JS で実装してみると以下のようになります。

```js:resolve 関数
const resolve = (resoltion) => {
  // ...仕様のアルゴリズムステップ(省略)

  let thenAction;
  try {
   // step.9/11
    thenAction = resolution.then;
    // 解決値の .then を取得
  } catch (thenError) {
    // step.10 (a/b) : then へのアクセスで例外が発生したら直ちに拒否する
    RejectPromise(promise, thenError);
    return undefined; // 関数終了
  }

  // step.12 : 「then が呼び出し可能なメソッドでない」場合には、直ちに履行
  if (!isCallable(thenAction)) {
    // ただのオブジェクトの場合もこれに含まれる
    FulfillPromise(promise, resolution);
    return undefined;
  }

  /*
  thenAction が呼び出し可能、つまり Thenable の場合には
  元の promise が解決できるようになるためには
  (つまりこの promise の状態が変わるためには)
  thenable が解決されるのを待つことになる
  thenable では複数の場合がある
  (1) promise オブジェクト
  (2) then メソッドを持つオブジェクト(promise以外)
  */
  // step.13 : コールバック関数を保持する JobCallback Record を host 定義の方法で作成する
  const thenJobCallback = HostMakeJobCallback(thenAction);
  // (1) : thenAction = then プロトタイプメソッド
  // (2) : thenAction = 実装された then メソッド

  // step.14
  const job = new PromiseResolveThenableJob(promise, resolution, thenJobCallback);
  // ここでは job は実行されず、ただ抽象クロージャが返ってくるだけ
  // 引数として渡した promise の解決はスケジューリングされた job で行われる(つまり、この関数が終わっても promise はまだ解決しない)

  // setp.15 : job(マイクロタスク)をエンキュー
  HostEnqueuePromiseJob(job);
  // step.16
  return undefined;
}
```

さて、`resolution` の値が Thenable の場合には NewPromiseThenableJob 操作によって作成される Job がマイクロタスクとして発行されることになります。

## 発生するマイクロタスクを考える

さて、実装をだいたい把握したところで、以下のような実際のコードで発生するマイクロタスクについて考えましょう。

```js
Promise.resolve(42)
  .then(x => {
    // このコールバック関数の実行で起きることを考える
    return Promise.resolve(x + 1);
  })
  .then(x => console.log(x));
```

`Proimse.resolve(42)` は履行した promise インスタンスなので、chain されている `.then()` メソッドのコールバック関数はただちに、マイクロタスクとして発行されます。このときのマイクロタスクの実体である Job は NewPromiseReactoinJob で作成されています。これが一個目のマイクロタスクです。

イベントループでコールバックがマイクロタスクが処理されるとき、`.then` メソッドの実行であらかじめ CreateResolutionFunctions で作成された `resolve` 関数が `resolution` として Thenable な値である `Promise.reslve(42 + 1)` を使って呼び出されます。

```js
// このような関数の実行が起きる
resolve(Promise.resolve(43));
```

これによって今まで実装してきたような `resolve` 関数が呼び出されます。場合分けで見たように `resolution` の値は Thenable (promise オブジェクトなので `.then` メソッドを持っている) なので、アルゴリズムステップの step.13 から step.16 が実行されますね。NewPromiseResolveThenableJob 操作で作成される Job がマイクロタスクとしてエンキューされます。これが二個目に発生するマイクロタスクとなります。

## NewPromiseResolveThenableJob

このマイクロタスクはコールスタック上にプッシュされて実行コンテキストとして処理されるわけですが、一体どのような処理になるか知りたいですね。そのためにはそのマイクロタスクを作成する [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) 操作で起きることを知る必要があります。

すこし大変ですが、アルゴリズムステップを見ていきましょう。

step.1 ではマイクロタスクとなる Job を作成します。

> 1. Let job be a new [Job](https://tc39.es/ecma262/#job) [Abstract Closure](https://tc39.es/ecma262/#sec-abstract-closure) with no parameters that captures promiseToResolve, thenable, and then and performs the following steps when called:
>   a. Let resolvingFunctions be [CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions)(promiseToResolve).
>   b. Let thenCallResult be [Completion](https://tc39.es/ecma262/#sec-completion-ao)([HostCallJobCallback](https://tc39.es/ecma262/#sec-hostcalljobcallback)(then, thenable, « resolvingFunctions.\[\[Resolve\]\], resolvingFunctions.\[\[Reject\]\] »)).
>   c. If thenCallResult is an [abrupt completion](https://tc39.es/ecma262/#sec-completion-record-specification-type), then
>     i. Return ? [Call](https://tc39.es/ecma262/#sec-call)(resolvingFunctions.\[\[Reject\]\], undefined, « thenCallResult.\[\[Value\]\] »).
>   d. Return ? thenCallResult.

Job の実体が抽象クロージャであることは説明しました。上のステップは step.a から step.d までのステップを行うような関数挙動を作成するというものだと考えてください。このステップが後のマイクロタスクとして処理される関数の挙動となります。

この抽象操作では、最終的には step.6 で Record として処理結果を返しますが step.2 から step.5 までは Realm 関連の処理ので省略します。

NewPromiseResolveThenableJob を JS のクラスで実装し、`resolve` 関数から以下のように呼び出せるようにしておきます。

```js
// resolve 関数内
// step.14
const job = new PromiseResolveThenableJob(
  promise,
  resolution,
  thenJobCallback
);
```

上記の引数 `promise` は実は CreateResolvingFunctions 操作の引数から渡ってきたものです。実装については詳細に触れませんが、CreateResolvingFunctions は以下のような関数です。内部で定義された `resolve` が今まで考えてきた `resolve` 関数です。

```js
function createResolvingFunctions(promise) {
  //                              ^^^^^^^^ この promise が PromiseResolveThenableJob へと渡される

  // step.1 : Record の作成
  const alreadyresolved = {
    value: false
  };

  // step.2 ~ 4 : 関数オブジェクトの作成
  const resolve = (resolution) => {
    //...
    // step.14
    const job = new PromiseResolveThenableJob(
      promise, // ここで渡される
      resolution,
      thenJobCallback
    );
    // ....
  };
  // step.5 ~ 6 : 関数の内部プロパティに参照させる
  resolve.alreadyResolved = alreadyResolved;
  resolve.promise = promise;

  // step.7 ~ 9 : 関数オブジェクトの作成
  const reject = (reason) => {
    //...
  };
  // step.10 ~ 11 : 関数の内部プロパティに参照させる
  reject.alreadResolved = alreadyResolved;
  reject.promise = promise;

  // step.12 : 最終的に resolve と reject の両方の関数オブジェクトを格納した Record を返す
  return {
    resolve,
    reject
  };
}
```

そして引数の `promise` 自体は Promise コンストラクタ関数で作成された promise であり、`then` メソッドから返る promise オブジェクトそのものです。

```js
Promise.resolve(42)
  .then(x => {
    // このコールバック関数の実行で起きることを考える
    return Promise.resolve(x + 1);
  }) // ← これから返る Promise が promise
  .then(x => console.log(x));
```

それがわかったところで再度 NewPromiseResolveThenableJob の実装について考えます。step.1 の a から d までの実装は以下のようになります。

```js
// step.1
class PromiseResolveThenableJob {
  constrcutor(
    promiseToResolve, // promise (元の未解決プロミス)
    thenable, // 解決値 (thenメソッドを持つ)
    then // JobCallback Record (thenメソッドを含むRecord)
  ) {

    // 返される抽象クロージャ(関数挙動)
    return () => {
      // step.a : 再度、解決関数を作成する
      const resolvingFunctions = createResolvingFunctions(promiseToResolve);
      const { resolve, reject } = resolvingFunctions;

      let thenCallResult;
      try {
        // step.b (ここが重要!)
        thenCallResult = thenable.then(resolve, reject);
      } catch (thenError) {
        // step.c
        return reject(thenError);
      }
      // step.d
      return thenCallResult;
    }
  }
}
```

`return () => {}` のように関数が返されるように実装していますが、仕様ではこのような振る舞いをキャプチャする抽象クロージャである Job がマイクロタスクとなるので、イベントループでマイクロタスクとして処理される関数自体がこのような関数となります。つまり、これが以下のコードで発生する２個目のマイクロタスクの実体です。

```js
Promise.resolve(42)
  .then(x => {
    // このコールバック関数の実行で起きることを考える
    return Promise.resolve(x + 1);
  })
  .then(x => console.log(x));
```

return される関数そのものが２個目のマイクロタスクであるわけですが、実際の処理がどのようなものかを説明すると、Thenable 用の解決関数である `resolve` と `reject` の作成を行い、さらにそれらを引数として `thenable.then(resolve, reject)` を起動します。このとき、`thenable` は `Promise.resolve(43)` という値であったので、履行しており、`resolve` 関数がマイクロタスクとして発行されます。これが３個目に発行されるマイクロタスクの正体です。

`resolve` 関数はすでに解説した `resolution` の値の種類によって処理が分かれる Promise Resolve Funciton です。このときの `resolution` は `thenable` (`Promise.resolve(43)`) が内包する値である `43` です。

`43` は通常の値なので、ただちに Fulfill されることになります。

```js:resolve 関数
const resolve = (resoltion) => {
  // ... 省略

  // stpe.8 (a/b) : 「オブジェクトではない値」の場合には直ちに履行する
  if (!isObject(resolution)) {
    // promise を fulfill する
    FulfillPromise(promise, resolution);
    return undefined; // 関数終了
  }

  // ...省略
}
```

さて、[FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise) 抽象操作では、TriggerPromiseReaction 抽象操作が実行されて、次に chain している `then` メソッドなどがあればそのコールバックをマイクロタスクとして発行します。

ということで、考えていた Promsie chain の `then` メソッドから返る Promise オブジェクトはマイクロタスクが３個実行されてからやっと解決したことになり、マイクロタスク４個目でコンソールに出力できることになります。

```js
Promise.resolve(42)
  .then(x => {
    return Promise.resolve(x + 1);
  }) // マイクロタスク３個が完了して次の then のコールバックを発行できるようになった
  .then(x => console.log(x));
  // マイクロタスク４個目でコンソール出力完了
```

# 解決値の違いによる挙動のまとめ

`resolve` 関数の引数である `reslution` (解決値) の違いによって `resolve` 関数の処理は場合分けされます。そして、それによって発生するマイクロタスクの数が異なることになりました。結果をまとめておきます。

解決値の値の種類 | (ステップ) 起こる処理
--|--
元の promise そのもの | (step.7) 例外を throw して直ちに拒否する
オブジェクトでない値 | (step.8) 解決対象の promise をその解決で直ちに履行する
Thenable ではないオブジェクト | (step.12)  promise をその解決値で直ちに履行する
Thenable なオブジェクト | (step.13-16) `thenable.then(resolve, reject)` を呼び出すためのマイクロタスクを発行する

解決値が promise の場合にはそれが settled になるまでにマイクロタスクが少なくとも確実に３個発生することに注意してください。つまり、Promise chain において `then` メソッドのコールバックで proimse オブジェクトを返すとマイクロタスクが３個発生することになります。

# Promise chain のネストをフラット化する弊害

この話に関連して、Promise chain のネストを行うことで通常発生するマイクロタスク３個からいくつか軽減することができます。

```js:nestVsFlat.js
const increment = (num) => {
  return Promise.resolve(num + 1)
};

// フラット化すると A を出力するまでマイクロタスクが 3個 かかる
increment(1)
  .then(num => increment(num)) // ３個可能消費
  .then(num => console.log("A", num));
  // ４個目のマイクロタスクで出力

// ネストしたままだと B を出力するまでマイクロタスクが 1個 かかる
increment(2)
  .then(num => { // １個目
    return increment(num)
      .then(num => console.log("B", num));
      // ２個目のマイクロタスクで出力
  });

/* RESULT
❯ deno run nestVsFlat.js
B 4
A 3
*/
```

フラット化していると処理の流れが見やすいですが、コールバックから返る値である Thenable の `.then` メソッドの起動がマイクロタスクとして発生してしまうので、無駄なマイクロタスクが発生することになります。

ただし、以下で説明するようにそもそも仕様最適化されている async/await が使えるので、async/await を使った方がマイクロタスクの発生をおさえ、さらに読みやすいコードが書けます。

# 仕様最適化の遺構

async/await と Promise chain でマイクロタスクの発生数が異なるという事象が起きますが、この Thenable の話が関与しています。

『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)』のチャプターで解説していますが、async/await は V8 エンジンサイドからの以下の PR で仕様自体の最適化がなされました。

https://github.com/tc39/ecma262/pull/1250/files

この最適化の中枢となる変更は以下の箇所です。

```diff
- 1. Let _promiseCapability_ be ! NewPromiseCapability(%Promise%).
+ 1. Let _promise_ be ? PromiseResolve(%Promise%, &laquo; _value_ &raquo;).
- 2. Perform ! Call(_promiseCapability_.[[Resolve]], *undefined*, &laquo; _promise_ &raquo;).
```

何が起きたかを掻い摘んで説明すると、[NewPromiseCapability](https://tc39.es/ecma262/#sec-newpromisecapability) 抽象操作と [Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions) の呼び出しがなくなり、 [PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve) 抽象操作がここに挿入されました。

`await thenable` という処理があったときには、 [PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve) 操作で await 式の評価対象が promise 以外の Thenable の場合だと、そのまま返すのではなく、一旦 promise でラップすることになりますが、promise オブジェクトだけは特別扱いして、そのまま返すようになりました。

これはつまり通常の値を評価するときと同じです。ただし、その後で thenable が持つ `.then` メソッドが実行されないと解決できないので、そのためのマイクロタスクが増加することになります。そして 解決時には [Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions) が起動して、[NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) が実行されます。

結局、仕様の最適化は async 関数の await 式の評価で promise を Thenable から引き離して、無駄な処理を削減するようにしたことが大きいです。

しかし、その一方で `Promise.prototype.then` の仕様ではコールバックから返される値が promise の場合を特別扱いしていません。つまり、仕様の最適化がされていないので、コールバックで promise を返した場合には async/await で発生するマイクロタスクよりも多くのマイクロタスクが発生してしまうことになります。
