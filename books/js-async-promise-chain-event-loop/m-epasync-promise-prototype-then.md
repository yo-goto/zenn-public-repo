---
title: "Promise.prototype.then の仕様挙動"
cssclass: zenn
date: 2022-11-02
modified: 2023-01-09
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases:
  - Promise本『Promise.prototype.then の仕様挙動』
  - thenメソッドの勘違いの修正
---

## このチャプターについて

このチャプターでは、`Promise.prototype.then` メソッドについて欠落していた情報を解説します。

`then` メソッドについては、この [第２部](sec-02-epasync) で解説していましたが、情報の欠落と間違いがあったため、このチャプターでまとめて補足・修正しておきたいと思います。

誤りが波及しているチャプターは以下のものですが、これらのチャプターについても修正致しました。

- 『[then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)』(修正済み)
- 『[Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)』(修正済み)
- 『[コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)』(修正済み)

:::message
他の波及チャプターを修正したため、このチャプターは仕様の解説を含む番外編という位置づけになりました。このチャプターにおける解説の結論を含む簡易バージョンは『[then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)』のチャプターに設けており、さらに結果をまとめた表を『[解決値の違いによる挙動のまとめ](#解決値の違いによる挙動のまとめ)』の項目に設けているので、現時点で難しすぎる場合にはその箇所だけ見て飛ばしてもらっても構いません。
:::

### 参考文献

こちらのチャプターの解説で使用する ECMASciript の仕様のコードは [ESLint](https://eslint.org) の開発者である Nicholas C. Zakas 氏のブログ記事シリーズから参考にさせていただきました。非常に分かりやすく解説されているので、自分自身で Promise の仕様を実装して学びたい場合には是非参考にしてください。

- [Creating a JavaScript promise from scratch, Part 1: Constructor - Human Who Codes](https://humanwhocodes.com/blog/2020/09/creating-javascript-promise-from-scratch-constructor/)
- [Creating a JavaScript promise from scratch, Part 2: Resolving to a promise - Human Who Codes](https://humanwhocodes.com/blog/2020/09/creating-javascript-promise-from-scratch-resolving-to-a-promise/)
- [Creating a JavaScript promise from scratch, Part 3: then(), catch(), and finally() - Human Who Codes](https://humanwhocodes.com/blog/2020/10/creating-javascript-promise-from-scratch-then-catch-finally/)
- [Creating a JavaScript promise from scratch, Part 4: Promise.resolve() and Promise.reject() - Human Who Codes](https://humanwhocodes.com/blog/2020/10/creating-javascript-promise-from-scratch-promise-resolve-reject/)

また、上のシリーズで実装された Promise のライブラリ (Pledge) は以下のリポジトリで公開されています。このライブラリをローカルインストールしてライブラリ内部を `console.log` などを入れて改造してみて実行することでメカニズムの理解が容易となります。

https://github.com/humanwhocodes/pledge

また ECMAScript の仕様そのものを理解するためには以下の記事が参考になるので興味があれば参考にしてみてください。

- [Understanding the ECMAScript spec, part 1](https://v8.dev/blog/understanding-ecmascript-part-1)
- [How to Read the ECMAScript Specification](https://timothygu.me/es-howto/)

それでは、さっそく、この本の解説で間違っていた点について例を出して解説していきます。

## then メソッドのコールバックから返る値の違いによる挙動

結論から言うと、`Promise.prototype.then()` メソッドにわたす **コールバック関数から返る値の種類によって発生するマイクロタスクの数が異なる** ことが仕様から判明しました。

`then(callback)` のように渡したコールバック関数 `callback` がマイクロタスクとして発行されるという話はそのままですが、`callback` から `then` メソッドを持つオブジェクトが返ると、追加でマイクロタスクが発生します。

以下のように `then` のコールバックで通常の値が返されている場合には合計２個のマイクロタスクが発生し、その２個目のマイクロタスクでコンソールへの出力となります。

```js
Promise.resolve(42)
  .then(x => x + 1) // 通常の値 43 が返されている
  .then(x => console.log(x));
  //   ^^^^^^^^^^^^^^^^^^^^
  //  マイクロタスク２個目で出力
```

一方、以下のように `then` のコールバックで Promise オブジェクトが返されている場合には合計 4 個のマイクロタスクが発生し、その 4 個目のマイクロタスクでコンソールへの出力となります。

```js
Promise.resolve(42)
  .then(x => Promise.resolve(x + 1)) // Promise オブジェクトが返されている
  .then(x => console.log(x));
  //   ^^^^^^^^^^^^^^^^^^^^
  //  マイクロタスク4個目で出力
```

その原理については後で詳しく解説するので、発生するマイクロタスクの数が異なることを２つを競争させる実際のサンプルコードを使って見ていきます。

まずは今まで行ってきた解説で完全に説明できる例です。

以下のコードでは、`then()` メソッドのコールバック関数で数値などの普通の値を返す２つの Promise chain を用意して競争させています。このコードの実行順序予測はこれまでの説明通りで、実際実行してみると予想通りになることがわかります。

```js:simpleValue.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: どちらの promise chain かの識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [A]");

Promise.resolve()
  .then(() => { // <1-a[1]>
    console.log("💙 [B]")
    return 1;
  })
  .then((d) => { // <3-a[2]>
    console.log("💙 [C]", d);
  });

Promise.resolve()
  .then(() => { // <2-b[1]>
    console.log("💚 [D]");
    return 2;
  })
  .then((d) => { // <4-b[2]>
    console.log("💚 [E]", d);
  })
  .then(() => { // <5-b[3]>
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
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: どちらの promise chain かの識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [A]");

Promise.resolve()
  .then(() => { // <1-a[1]>
    console.log("💙 [B]")
    return Promise.resolve(1);
    // <3-a[2]> <5-a[3]>
  })
  .then((d) => { // <7-a[4]>
    console.log("💙 [C]", d);
  });

Promise.resolve()
  .then(() => { // <2-b[1]>
    console.log("💚 [D]");
    return 2;
  })
  .then((d) => { // <4-b[2]>
    console.log("💚 [E]", d);
  })
  .then(() => { // <6-b[3]>
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

この理由としては、最初に言ったとおり `then()` メソッドに渡すコールバックから Promise などの `then()` メソッドを持つオブジェクト (**Thenable** と呼ばれるオブジェクト) が返されると、もとの `then()` メソッドから返される Promise を解決するために通常の値を返す時に加えて追加で２つのコールバックがマイクロタスクとして発生するためです。

これまでの解説だと、上記のような普通の値を返す chain と Promise を返す chain の比較がなかったため、Promise を返している chain においても一見正しい解説のように見えてしまっているはずですが、コールバック関数で Promise を返している場合、**実際にはマイクロタスクが追加で２つ発生しています**。

実際、もう片方の chain の `then` コールバックから返る値を Promise オブジェクトに変更すれば最初のコードと同じ出力順番にすることができます。

```js:fakePattern.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: どちらの promise chain かの識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [A]");

Promise.resolve()
  .then(() => { // <1-a[1]>
    console.log("💙 [B]")
    return Promise.resolve(1);
    // <3-a[2]> <5-a[3]>
  })
  .then((d) => { // <9-a[4]>
    console.log("💙 [C]", d);
  });

Promise.resolve()
  .then(() => { // <2-b[1]>
    console.log("💚 [D]");
    return Promise.resolve(2); // Promise にした
    // <4-b[2]> <6-b[3]>
  })
  .then((d) => { // <8-b[4]>
    console.log("💚 [E]", d);
  })
  .then(() => { // <10-b[5]>
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
さきに言っておくと、この挙動の違いを明確に理解するには ECMAScript の仕様をしっかり見る必要があります。とはいえ、仕様自体が非常に解読困難で冗長なものでもあるので、概要を説明していきたいと思います。

ただし、概要だけでもかなり長くなるので結果をまとめた表を [解決値の違いによる挙動のまとめ](#解決値の違いによる挙動のまとめ) の見出しにまとめたので、そちらから見てもらっても構いません。
:::

## Thenable とは

まず始めに、Thenable について解説しておきます。

Thenable とは `then` というメソッドを持つオブジェクトの総称です。例えば、Promise は `then` メソッドを持つので Thenable であると言えます。また、自分で適当な挙動の `then` メソッドを実装したオブジェクトの場合でも Thenable と言えます。つまり、**Thenable であるからと言って必ずしも Promise だとは限らない** ので注意してくだださい。

以下のように `then` メソッドが実装されていれば、await や Promise chain で利用でき、実装されている `then` メソッドが実行されます。

```js
const thenable = {
  then: (onFulfilled) => {
    setTimeout(() => onFulfilled(42), 100);
  },
};

(async () => {
  const v = await thenable;
  // 内包する値 42 を抽出できる
  console.log(v); // => 42
})();
```

このように `then` メソッドを持っていれば、await 式の評価や Promise chain で promise オブジェクトと同じように扱えます。このような動作があるのは、Promise 自体が元はそれを実装するコミュニティベースのライブラリがいくつかあり、あとになって仕様に導入されるようになった経緯があるからです。

つまり、Promise 以外の `then` を持つオブジェクト (ECMAScirpt 実装ではない Promise) などがネイティブの Promise のように扱えるようにした仕組みが Thenable と言えます。

さて、実はそういったオブジェクトそのものを使いたいからこの概念の説明をしたわけではありません。問題である `Promise.prototype.then` の挙動について説明するのに必要なのでこの概念の解説をしています。

## 仕様の基礎

ここからは JavaScript の仕様である ECMAScript の基礎について解説していきます。最終的なぜそのような動作になるかを自分で確認できるように仕様の URL をワードに貼り付けておきます。

### 抽象操作とアルゴリズムステップ

まず ECMAScript の仕様には「**抽象操作 (Abstract Operation)**」というものが記述されています。

抽象操作とは ECMAScript 仕様の内部で利用される関数であり、JavaScript から直接呼び出すことはできません。意味合いとしては単純に仕様の編集者が何回も同じことを書かないように「長い表記を省略して完結に記述できるようにする」というのが大きいです。

ECMAScript 仕様のプロトタイプメソッドや静的メソッド、抽象操作についてはそれぞれ「アルゴリズムステップ (Algorithm steps)」というものが定義されています。アルゴリズムステップが仕様が定義する操作の挙動を表現しているため、仕様を理解するためには各操作のアルゴリズムステップを理解していくことになります。

アルゴリズムステップ自体は以下のようなリストで表現されています。また、１つのアルゴリズムステップは連続したサブステップに細分化されることがあり、サブステップは字下げされて、それ自体がさらに字下げされたサブステップに分割されることがあります。

![アルゴリズムムステップ](/images/js-async/img_ecmascript-algorithm-steps.jpg)*[https://tc39.es/ecma262/#sec-algorithm-conventions](https://tc39.es/ecma262/#sec-algorithm-conventions) より*

このようなアルゴリズムステップの表現は HTML 仕様でもまったく同じです。

さて、Promise 系列の仕様を理解するために必要な抽象操作は [Promise Abstract Operations](https://tc39.es/ecma262/#sec-promise-abstract-operations) の項目に記載されています。ただし、ここに記述されている抽象操作からは他の項目にある抽象操作も呼び出されるので、細かく理解するのには様々な操作をたどっていく必要があります。

例えば Promise の静的メソッドである [Promise.resolve](https://tc39.es/ecma262/#sec-promise.resolve) は内部的に大半の作業を `PromiseResolve` という抽象操作に委任しています。以下のように ECMA 仕様において、`Promise.resolve` 内部のアルゴリズムステップから抽象操作が呼び出されています。

![algorithm-steps](/images/js-async/img_ecma-algorithm-step.jpg)*[https://tc39.es/ecma262/#sec-promise.resolve](https://tc39.es/ecma262/#sec-promise.resolve) より*

プロトタイプメソッドや静的メソッドの仕様を見てみると、このようにそれぞれのアルゴリズムステップ内で抽象操作が呼び出され、さらにその抽象操作から別の抽象操作が呼び出されるという依存関係が構築されているので、仕様挙動を理解するためにはそれぞれの操作をたどっていき関係性を把握していく必要があります。そのように各アルゴリズムステップを見て関係をたどって理解するのはかなり大変な作業なので、コアとなる抽象操作については簡易的な図でまとめておきました。それぞれの操作は以下のような呼び出し関係となっています。

![promise抽象操作](/images/js-async/PromiseSpec.excalidraw.png)

例えば `Promise.prototype.catch` や `Promise.prototype.finally` といったプロトタイプメソッドは、実は大半の作業を `Promise.prototype.then` にまかせており、さらに `Promise.prototype.then` は PerformPromiseThen という操作に多くの作業をまかせています。また、`Promise.prototype.then` や `Promise.resolve` や Await などの多くの操作から NewPromiseCapability 抽象操作が呼び出されて Promise オブジェクトの作成が行われています。

### ECMAScript の V8 実装

ちなみに、ECMAScript 仕様を直接見て各操作感の関係を辿るのが億劫なら、V8 エンジン側での実装を直接見て理解するのも一つの手です。『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)』のチャプターで言ったとおり、ECMAScript の V8 実装は [V8 Torque](https://v8.dev/docs/torque) (TypeScript ライクな V8 エンジンの開発専用の言語) や C++ で GitHub リポジトリの [builtins](https://github.com/v8/v8/tree/main/src/builtins) の場所に記載されています。

ECMASciript の Promise にまつわる抽象操作の項目である Promise Abstract Operations は同じ名前のファイルとして [v8/src/builtins/promise-abstract-operations.tq](https://github.com/v8/v8/blob/main/src/builtins/promise-abstract-operations.tq) で記述されています(`.tq` 拡張子は **T**or**q**ue 言語で書かれたファイルです)。仕様に定義されている抽象操作が同じ名前の関数としてそのまま定義されていることが分かります。

https://github.com/v8/v8/blob/a760f03a6e99bf4863d8d21c5f7896a74a0a39ea/src/builtins/promise-abstract-operations.tq#L150-L263

仕様内のアルゴリズムステップの記述がコメントされており、各操作が TypeScript ライクに記述されているので場合によっては仕様よりも V8 実装を見たほうが理解しやすいです。

ただし、GitHub のリポジトリはミラーリポジトリであり、GitHub 上では Torque のシンタックスハイライト等も無いので、実際のコードベースを調べるときは Chromium Code Search を利用するのが良いです。

https://source.chromium.org/chromium/chromium/src/+/main:v8/src/builtins/promise-abstract-operations.tq

### Job とマイクロタスク

図一番下の [HostEnqueuPromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob) という操作が最終的にマイクロタスクをマイクロタスクキューへとエンキューする操作です。

:::message
だたし、ECMAScript では「マイクロタスク」は [Job](https://tc39.es/ecma262/#job) と呼ばれるものとして扱われていることに注意してください。マイクロタスク自体はあくまで WHATWG の HTML 仕様に定義されているものです。
:::

Job というのは実際には内部的な仕様の型 (Specifiaction type) である抽象クロージャ ([Abstact Closure](https://tc39.es/ecma262/#sec-abstract-closure)) という型の値です。

抽象クロージャの作成時には指定されたアルゴリズムステップと複数の値のコレクションをキャプチャします。簡単にいえば、抽象クロージャは関数の挙動を示すものであり、実際に [CreateBuiltinFunction](https://tc39.es/ecma262/#sec-createbuiltinfunction) という抽象操作は抽象クロージャを引数にしてその抽象クロージャが記述した振る舞いを持つ関数オブジェクトを作成します。

当該の HostEnqueuePromiseJob 操作は [ECMAScript 仕様](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)では部分的な要件しか定義されていません。というのも、Host から名前が始まる抽象操作は [Host hook](https://tc39.es/ecma262/#host-hook) と呼ばれ、このタイプの操作は ECMAScript の仕様外で全体や部分が定義されるものだからです。

ECMAScript 仕様の側では Host hook は以下のように述べられています。

> A host hook is an abstract operation that is defined in whole or in part by an external source. All [host hooks](https://tc39.es/ecma262/#host-hook) must be listed in Annex [D](https://tc39.es/ecma262/#sec-host-layering-points).
> (https://tc39.es/ecma262/#host-hook より引用)

HTML 仕様の側では Host hook は以下のように述べられています。

> The JavaScript specification contains a number of [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined) abstract operations, that vary depending on the host environment. This section defines them for user agent hosts.
> (https://html.spec.whatwg.org/multipage/webappapis.html#javascript-specification-host-hooks より引用)

これらの操作は「ホスト環境(host environment)」によって異なる実装となることが述べられていますね。「ホスト(Host)」自体は Host hook などの [D Host Layer Points](https://tc39.es/ecma262/#sec-host-layering-points) に列挙された機能を定義する外部ソースであり、すべての Web ブラウザ実装の集合や WHATWG の HTML 仕様もホストに相当します。ホスト環境は Node や Chrome などの特定の環境を指します。

Host hook である HostEnqueuePromiseJob 操作の ECMA 仕様は以下のように job と realm を引数にとって job を将来のある時点でスケジューリングするという操作であることが決まっていますが、簡素なもので要件などのみが定義されています。

![hostenqueuepromisejob ecma spec](/images/js-async/img_hostEnqueuPromiseJob-ecma.jpg)*[https://tc39.es/ecma262/#sec-hostenqueuepromisejob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob) より*

Host hook なので ECMAScript の外部ソースである(ホストである) HTML 仕様でも以下のように定義されています。ECMAScript 仕様では「Job をスケジューリング」するという抽象的な操作であったものが HTML 仕様ではイベントループにおける「マイクロタスクのキューイング操作」としてより具体的に定義されていることが分かりますね。

![hostenqueuepromisejob html spec](/images/js-async/img_hostEnqueuePromiseJob-html.jpg)*[https://html.spec.whatwg.org/webappapis.html#hostenqueuepromisejob](https://html.spec.whatwg.org/webappapis.html#hostenqueuepromisejob) より*

:::message
HTML 仕様側に ECMAScript 仕様側の抽象操作への記述が大量にあることからも、結局は両方の仕様を見ないと全体像がつかめないという罠があります。
:::

Promise 関連の Job (マイクロタスク) を作る直接の抽象操作は以下の２つとなります。そして、この２つがこれまでの問題の中心とります。

抽象操作 | 説明
--|--
[NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob) | Promise の解決時に発行される Job (マイクロタスク) を作成する操作
[NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) | 解決値が Thenable の場合に上記の Job から更に追加の Job (マイクロタスク) を作成する操作

[NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob) で作成される PromiseReactionJob と呼ばれる Job (Abstract closure) がこれまで扱ってきたイベントループで処理されるマイクロタスクの実体となります。Promise chain だけでなく、async/await でもこの Job は発行されます。

一方で、async/await の仕様の最適化で一部削減されたものの Promise chain においてはいまだに仕様に完全に残っている特殊な追加のマイクロタスクを発生させる操作が [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) となります。この操作によって当該の問題となっている追加発生するマイクロタスクが生成されることになります。

従って、この NewPromiseResolveThenableJob 抽象操作の呼び出しが行われる経路を特定し、この操作によって作成される Job (マイクロタスク) がどのような挙動を発生させているのかを理解することで問題を解決できます。

図に記載していますが、NewPromiseResolveThenableJob 抽象操作を直接的に呼び出しているのは、[CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions) という抽象操作によって作成される [Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions) です。

まずは Promise Resolve Functions を作成する CreateResolvingFunctions 抽象操作から見ていきましょう。

## CreateResolvingFunctions

それでは、ここから本格的に問題となる仕様の部分について実際に触れていきます。

`Promise.prototype.then` のコールバックから返される値の種類によって異なる処理が発生しますが、この問題を起こしている抽象操作の実体は [CreateResolvingFunctinos](https://tc39.es/ecma262/#sec-createresolvingfunctions) で作成される [Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions) という関数です。

仕様的にはこのような名前がついていますが、その正体はかなり身近なもので、`new Promise(executor)` という Promise インスタンス作成時に渡すコールバック関数である `executor` の引数となる `resolve` 関数です。

```js
const p = new Promise((resolve, reject) => {
  //               これ↑^^^^^^^
  resolve(42); // この関数が Promise Resolve Functions で作成される関数
});

p.then(console.log); // => 42
```

今まではなんとなく `resolve()` という関数を使ってきましたが、この関数が「実際にやること」が定義されているのが Promise Resolve Functions の仕様です。

Promise コンストラクタを使ってプロミスインスタンスを作成すると、CreateResolvingFunction 抽象操作が呼び出され、解決用の関数である [Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions) と拒否用の関数である [Promise Reject Functions](https://tc39.es/ecma262/#sec-promise-reject-functions) が作成されます。そしてこれらことが今まで見てきた `resolve` 関数と `reject` 関数そのものであり実体です。

『[Promise の基本概念](a-epasync-promise-basic-concept)』や『[Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)』のチャプターでも説明しましたが、`reject` 関数に比べて、`resolve` 関数は非常に複雑です。これは Unwrapping の能力が関係しており、`reject` 関数では引数の `reason` (拒否理由) として何を渡そうが拒否すると決まっていますが、`resolve` 関数では引数の `resolution` (解決値) の値の種類よって処理内容が変わるため複雑になっています。

![resolveとrejectの仕様](/images/js-async/img_spec-diff-resolve-reject.jpg)

`Promise.prototype.then` でも **この `resolve` 関数が実は使われており** (呼び出し図に注目)、コールバックから返される値が引数の `resolution` (解決値) として使われます。従って、`resolve` 関数の処理の場合分けを考える必要があるという話になります。

## then から resolve 関数が呼び出される仕組み

`Promise.prototype.then` から `resolve` 関数が呼び出されるというのは仕様を見てもかなり分かりづらいので補足しておきます。

`Promise.prototype.then` が呼び出されると PerformPromiseThen 抽象操作が呼び出される前に [Promise コンストラクタ](https://tc39.es/ecma262/#sec-promise-executor) によって新しい Promise インスタンスが作成されます。実際には仕様の以下のステップで発生しています。

> - 3. Let C be ? [SpeciesConstructor](https://tc39.es/ecma262/#sec-speciesconstructor)(promise, [%Promise%](https://tc39.es/ecma262/#sec-promise-constructor)).
> - 4. Let resultCapability be ? [NewPromiseCapability](https://tc39.es/ecma262/#sec-newpromisecapability)(C).

[NewPromiseCapability](https://tc39.es/ecma262/#sec-newpromisecapability) 抽象操作はコンストラクタを引数にとって、新しい Promise オブジェクトを作成し、内部フィールドにその参照を格納します。仕様のステップでは以下の場所で [Construct](https://tc39.es/ecma262/#sec-construct) 抽象操作を使ってコンストラクタを起動させます。

> - 6. Let promise be ? [Construct](https://tc39.es/ecma262/#sec-construct)(C, « executor »).

そして、[Promise コンストラクタ](https://tc39.es/ecma262/#sec-promise-executor) の仕様の次のステップで CreateResolvingFunctions が呼び出され、`resolve` 関数と `reject` 関数が作成されています。

> - 8. Let resolvingFunctions be [CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions)(promise).

その後 then メソッド内部の処理の続きとして [PerformPromiseThen](https://tc39.es/ecma262/#sec-promise.prototype.then) 抽象操作が呼び出され、仕様の以下のステップ 10-11 のステップで promise の状態によって処理が振り分けられますが、どちらの場合でも [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob) が呼び出されます。

> - 10. Else if promise.\[\[PromiseState\]\] is fulfilled, then
>   - a. Let value be promise.\[\[PromiseResult\]\].
>   - b. Let fulfillJob be [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob)(fulfillReaction, value).
>   - c. Perform [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)(fulfillJob.\[\[Job\]\], fulfillJob.\[\[Realm\]\]).
> - 11. Else,
>   - a. [Assert](https://tc39.es/ecma262/#assert): The value of promise.\[\[PromiseState\]\] is rejected.
>   - b. Let reason be promise.\[\[PromiseResult\]\].
>   - c. If promise.\[\[PromiseIsHandled\]\] is false, perform [HostPromiseRejectionTracker](https://tc39.es/ecma262/#sec-host-promise-rejection-tracker)(promise, "handle").
>   - d. Let rejectJob be [NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob)(rejectReaction, reason).
>   - e. Perform [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)(rejectJob.\[\[Job\]\], rejectJob.\[\[Realm\]\]).

NewPromiseReactiobJob は上で説明したように Job (マイクロタスク) を作成するための操作です。

この NewPromiseReactionJob 操作を呼び起こすと、then 操作の最初で NewPromiseCapability 操作をつかって内部的に作り出した CreateResolvingFunctions の関数を呼び出したことになるため、`resolve` 関数の呼び出しが起きています。

実際に `resolve` 関数や `reject` 関数が呼び出されるのは仕様の以下のステップです。[PromiseCapability Record](https://tc39.es/ecma262/#sec-promisecapability-records) (promise の状態にアクセスできるようにしたヘルパーオブジェクトのようなもの) のフィールド `[[Resolve]]` と `[[Reject]]` にはそれぞれ `resolve` 関数と `reject` 関数が保持されており、仕様内の以下のステップで [Call](https://tc39.es/ecma262/#sec-call) 抽象操作を使って関数呼び出しを行います。

> - h. If handlerResult is an [abrupt completion](https://tc39.es/ecma262/#sec-completion-record-specification-type), then
>   - i. Return ? [Call](https://tc39.es/ecma262/#sec-call)(promiseCapability.\[\[Reject\]\], undefined, « handlerResult.\[\[Value\]\] »).
> - i. Else,
>   - i. Return ? [Call](https://tc39.es/ecma262/#sec-call)(promiseCapability.\[\[Resolve\]\], undefined, « handlerResult.\[\[Value\]\] »).

このようにして仕様内部では `Promise.prototype.then` から CreateResolvingFunctions 操作で作成した `resolve` 関数と `reject` 関数が呼び出せるようになっています。

:::message
**ギュメ記号について**  

`« »` は「[ギュメ](https://ja.wikipedia.org/wiki/%E6%8B%AC%E5%BC%A7)」と呼ばれる記号です。この [ギュメ記号](https://tc39.es/ecma262/#sec-list-and-record-specification-type) は ECMAScript では、仕様内の List 型 (仕様型) を表現するためのリテラル記法として利用されます。例えば、`« 1, 2 »` は、2 つの要素を持ち、それぞれが特定の値で初期化されたリスト値を定義している。新しい空の List は `« »` と表現することができます。

[Call](https://tc39.es/ecma262/#sec-call) のような抽象操作の引数で `aurgumentList` と定義されていれば、呼び出し側ではこのギュメ記号を使った List 型リテラルで引数を表現していることが多いです。
:::

## Promise Resolve Functions

さて、いよいよ本題に入ります。CreateResolvingFunctions 抽象操作で作成される [Promise Resolve Function](https://tc39.es/ecma262/#sec-promise-resolve-functions) が `resolve` 関数の正体であり、冒頭の問題が発生する仕様箇所です。

この関数のアルゴリズムステップを理解できれば問題解決となります。

実際にこの操作で作成される関数を JavaScript で実装してみます。細かい部分は気にせずに重要な箇所だけ考えていき、関数として記述できる step.4 から 16 までを考えて、徐々に埋めてきましょう。

```js:resolve 関数
const resolve = (resoltion) => {
  // ...仕様のアルゴリズムステップを実装
}
```

step.1 から step.4 までは関数作成のためのただの準備です。

> - 1. Let F be the [active function object](https://tc39.es/ecma262/#active-function-object).
> - 2. [Assert](https://tc39.es/ecma262/#assert): F has a \[\[Promise\]\] internal slot whose value [is an Object](https://tc39.es/ecma262/#sec-object-type).
> - 3. Let promise be F.\[\[Promise\]\].
> - 4. Let alreadyResolved be F.\[\[AlreadyResolved\]\].

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

> A promise resolve function is an anonymous built-in function that has \[\[Promise\]\] and \[\[AlreadyResolved\]\] internal slots.
>
> When a promise resolve function is called with argument resolution, the following steps are taken:

Promise Resole Function は `[[Promise]]` と `[[AlradyResolve]]` という内部スロットを持つ無名のビルトイン関数 (ECMAScript に備え付けの関数) であり、引数 `resolution` で呼び出されることで記述されているアルゴリズムステップを実行するとのことです。

実はこの引数 `resolution` が非常に重要です。`resolve` 関数が呼び出されるのは以下のような形式となっていますね。

```js
resolve(resolution);
```

`new Promise(executor)` の形式であれば、`resolve()` 関数の引数として `resolution` に渡すのは新しくインスタンス化する Promise オブジェクトの内包させて履行させたい値でした。

```js
const p = new Promise(resolve => {
  resolve(42); // 履行値を reslution として渡す
});
```

そして、Promise.prototype.then のコールバック関数で返す値もこの `resolution` として扱われて最終的に `resolve` 関数が呼び出されるわけです。

```js
Promise.resolve(42)
  .then(x => {
    return x + 1;
    //     ^^^^^
    // resolution として resolve 関数に渡される値 43
  })
```

`resolve` 関数の実装では、この引数 `resolution` の値の種類によって処理が変わるので場合分けすることになります。その箇所は以下の仕様の step.7 から step.16 となります (step.5-6 は省略します)。

> - 7. If [SameValue](https://tc39.es/ecma262/#sec-samevalue)(resolution, promise) is true, then
>   - a. Let selfResolutionError be a newly created TypeError object.
>   - b. Perform [RejectPromise](https://tc39.es/ecma262/#sec-rejectpromise)(promise, selfResolutionError).
>   - c. Return undefined.
> - 8. If resolution [is not an Object](https://tc39.es/ecma262/#sec-object-type), then
>   - a. Perform [FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise)(promise, resolution).
>   - b. Return undefined.
> - 9. Let then be [Completion](https://tc39.es/ecma262/#sec-completion-ao)([Get](https://tc39.es/ecma262/#sec-get-o-p)(resolution, "then")).
> - 10. If then is an [abrupt completion](https://tc39.es/ecma262/#sec-completion-record-specification-type), then
>   - a. Perform [RejectPromise](https://tc39.es/ecma262/#sec-rejectpromise)(promise, then.\[\[Value\]\]).
>   - b. Return undefined.
> - 12. If [IsCallable](https://tc39.es/ecma262/#sec-iscallable)(thenAction) is false, then
>   - a. Perform [FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise)(promise, resolution).
>   - b. Return undefined.
> - 13. Let thenJobCallback be [HostMakeJobCallback](https://tc39.es/ecma262/#sec-hostmakejobcallback)(thenAction).
> - 14. Let job be [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob)(promise, resolution, thenJobCallback).
> - 15. Perform [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)(job.\[\[Job\]\], job.\[\[Realm\]\]).
> - 16. Return undefined.

ちょっと長いですが、やっていることは `resolve` 関数の引数 `resolution` の値によって処理を振り分けて、[FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise) か [RejectPromise](https://tc39.es/ecma262/#sec-rejectpromise) を使って promise オブジェクトの状態を pending から fulfilled または rejected へと遷移させ、関数自体から `undefined` を返して終了させています。

step.10 以降の処理は例外が起きた場合やその場合分けに適さなかった残りの場合の処理を記述していることになります。

`resolution` (解決値) として考えれられる値の種類は以下のようなケースが考えられ、実際それぞれのステップで場合分けされます。

- 1. 解決値が「元の promise そのもの」の場合 (step.7)
- 2. 解決値が「オブジェクトでない値」の場合 (step.8)
- 3. 解決値が「解決値がオブジェクト」の場合 (step.9)
  - 1. `then` プロパティがメソッドでない場合 (`undefind` である場合も含む) (step.12)
  - 2. `then` プロパティがメソッドの場合、つまりそのオブジェクトは Thenable である場合 (step.13-16)

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
    // 解決値の then を取得
  } catch (thenError) {
    // step.10 (a/b) : then へのアクセスで例外が発生したら直ちに拒否する
    RejectPromise(promise, thenError);
    return undefined; // 関数終了
  }

  // ...
}
```

promise オブジェクトの内部状態を実際に遷移させるのは [FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise) と [RejectPromise](https://tc39.es/ecma262/#sec-rejectpromise) という抽象操作を利用します。

この操作が実行されると、promise の内部状態を書き換えて、[TriggerPromiseReactions](https://tc39.es/ecma262/#sec-triggerpromisereactions) という抽象操作が実行されます。この操作内の [HostEnqueuPromieJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob) 操作によって、もし次の chain している `then` や `catch` などがあればそのコールバックをマイクロタスクとして発火させます。

ここまでで、解決値が「元の promise」の場合と「オブジェクトではない値」の場合のケースの処理がカバーできました。つまり `reslution` として渡される値が文字列や数値などの値である場合には直ちに履行することができることがわかります。そして、履行したら `return undefined` で `resolve` 関数自体の処理が終了するので、残りのアルゴリズムステップは無視できることになります。

残されたケースは `resolution` の値が「オブジェクトの場合」であり、さらに「then メソッドを持つかどうか」、つまり thenable であるかどうかの処理の振り分けを行う step.12 から step.16 までを実装します。

> - 12. If [IsCallable](https://tc39.es/ecma262/#sec-iscallable)(thenAction) is false, then
>   - a. Perform [FulfillPromise](https://tc39.es/ecma262/#sec-fulfillpromise)(promise, resolution).
>   - b. Return undefined.

step.12 では [IsCallable](https://tc39.es/ecma262/#sec-iscallable) という抽象操作を使って、オブジェクトの `then` が呼び出し可能なメソッドであるかどうかを判定し、呼び出し不可能な場合には、`resolution` が `then` メソッドを持たないオブジェクトであることがわかり、FulfillPromise を使って直ちに履行して `return undefined` で `resolve` 関数を終了させます。

そうではない場合には、呼び出し可能な `then` メソッドを持つオブジェクトである、つまり Thenable であることが判明します。ここからがこのチャプターの中枢となります。

`resolution` が Thenable である場合の処理を step.13 から step.16 で行います。

> - 13. Let thenJobCallback be [HostMakeJobCallback](https://tc39.es/ecma262/#sec-hostmakejobcallback)(thenAction).
> - 14. Let job be [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob)(promise, resolution, thenJobCallback).

step.13 では、マイクロタスクとして実行するコールバック関数を [HostMakeJobCallback](https://tc39.es/ecma262/#sec-hostmakejobcallback) 抽象操作で作成します。返ってくるのは JobCallback Record というコールバック関数への参照を可能したデータです。step.14 ではこのデータから [NewPromiseThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) という抽象操作で、マイクロタスクの実体である Job を作成します。

> - 15. Perform [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)(job.\[\[Job\]\], job.\[\[Realm\]\]).
> - 16. Return undefined.

そして、[HostEnqueuPromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob) 操作によって実際にマイクロタスクとしてエンキューし、`return undefined` で `resolve` 関数を終了します。

これを JS で実装してみると以下のようになります。

```js:resolve 関数
const resolve = (resoltion) => {
  // ...仕様のアルゴリズムステップ(省略)

  let thenAction;
  try {
   // step.9/11
    thenAction = resolution.then;
    // 解決値の then を取得
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

`resolution` の値が Thenable の場合には NewPromiseThenableJob 操作によって作成される Job がマイクロタスクとして発行されることになります。

### 発生するマイクロタスクを考える

さて、`resolve` 関数の実装をだいたい把握したところで、以下のような実際のコードで発生するマイクロタスクについて考えましょう。

```js
Promise.resolve(42)
  .then(x => {
    // このコールバック関数の実行で起きることを考える
    return Promise.resolve(x + 1);
    //     ^^^^^^^^^^^^^^^^^^^^^^
    // resolution として resolve 関数に渡される値
  })
  .then(x => console.log(x));
```

`Promise.resolve(42)` は履行した promise インスタンスなので、chain されている `then()` メソッドのコールバック関数は直ちにマイクロタスクとして発行されます。このときのマイクロタスクの実体である Job は NewPromiseReactoinJob 抽象操作で作成されています。これが一個目のマイクロタスクです。

イベントループでコールバックがマイクロタスクが処理されるとき、`then` メソッドの実行であらかじめ CreateResolutionFunctions で作成された `resolve` 関数が Thenable な値である `Promise.resolve(42 + 1)` を引数 `resolution` として使って呼び出されます。

```js
// このような関数の実行が起きる
resolve(Promise.resolve(43));
```

これによって今まで実装してきたような `resolve` 関数が呼び出されます。場合分けで見たように `resolution` の値は Thenable (Promise オブジェクトは `then` メソッドを持っている) なので、アルゴリズムステップの step.13 から step.16 が実行されます。これによって NewPromiseResolveThenableJob 操作で作成される Job がマイクロタスクとしてエンキューされます。これが二個目に発生するマイクロタスクとなります。

### NewPromiseResolveThenableJob

このマイクロタスクはコールスタック上にプッシュされて実行コンテキストとして処理されるわけですが、一体どのような処理になるか知りたいですね。そのためにはそのマイクロタスクを作成する [NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob) 操作で起きることを知る必要があります。

すこし大変ですが、アルゴリズムステップを見ていきましょう。

step.1 ではマイクロタスクとなる Job を作成します。

> - 1. Let job be a new [Job](https://tc39.es/ecma262/#job) [Abstract Closure](https://tc39.es/ecma262/#sec-abstract-closure) with no parameters that captures promiseToResolve, thenable, and then and performs the following steps when called:
>   - a. Let resolvingFunctions be [CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions)(promiseToResolve).
>   - b. Let thenCallResult be [Completion](https://tc39.es/ecma262/#sec-completion-ao)([HostCallJobCallback](https://tc39.es/ecma262/#sec-hostcalljobcallback)(then, thenable, « resolvingFunctions.\[\[Resolve\]\], resolvingFunctions.\[\[Reject\]\] »)).
>   - c. If thenCallResult is an [abrupt completion](https://tc39.es/ecma262/#sec-completion-record-specification-type), then
>     - i. Return ? [Call](https://tc39.es/ecma262/#sec-call)(resolvingFunctions.\[\[Reject\]\], undefined, « thenCallResult.\[\[Value\]\] »).
>   - d. Return ? thenCallResult.

Job の実体が抽象クロージャであることは説明しました。上のステップは step.a から step.d までのステップを行うような関数挙動を作成するというものだと考えてください。このステップが後のマイクロタスクとして処理される関数の挙動となります。

この抽象操作では、最終的には step.6 で Record として処理結果を返しますが step.2 から step.5 までは Realm 関連の処理なので省略します。

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

そして引数の `promise` 自体は Promise コンストラクタ関数で作成された Promise であり、`then` メソッドから返る Promise オブジェクトそのものです。

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

ということで、考えていた Promise chain の `then` メソッドから返る Promise オブジェクトはマイクロタスクが３個実行されてからやっと解決したことになり、マイクロタスク４個目でコンソールに出力できることになります。

```js
Promise.resolve(42)
  .then(x => { // １個目のマイクロタスク
    return Promise.resolve(x + 1);
  }) // ここで更に発生する2個のマイクロタスクが完了して次の then のコールバックを発行できるようになった
  .then(x => console.log(x));
  // 4個目のマイクロタスクでコンソール出力
```

ちなみに Promise ではない Thenable オブジェクトの場合にはマイクロタスクが Promise の場合のときと比べて１つ減ったりします。
それは `thenable.then(resolve, reject)` が呼び出されるときに、この `then` は `Promise.prototype.then` のネイティブメソッドでなく、以下のように別に実装されているとコールバック関数をマイクロタスクとして発行せずに直ちに呼び出すからです。

```js
Promise.resolve(42)
  .then(x => { // <1>
    const thenable = {
      then(onFulfilled) {
        onFulfilled(x + 1);
      },
    };
    return thenable;
    // <2> thenable.then(resolve, reject) を呼び出すマイクロタスク
  })
  .then(x => console.log(x)); // <3>
  // 3個目のマイクロタスクでコンソール出力
```

もしも Promise と同じマイクロタスクの数にしたい場合には Thenable オブジェクトの `then` メソッド内部で `queueMircotask` API を使ってコールバックをマイクロタスクを発行するようにすれば良いわけです。

```js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: Promise (p) か Thenable (t) か
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
Promise.resolve(42)
  .then(x => { // <1-p[1]>
    console.log("💙 [1]")
    return Promise.resolve(x + 1);
    // <3-p[2]> Promise.resolev(43).then の呼び出し
    // <5-p[3]> resolve 関数の実行
  })
  .then(x => { // <7-p[4]>
    console.log("👻 [3] PROMISE", x)
  });

Promise.resolve(42)
  .then(x => { // <2-t[1]>
    console.log("💚 [2]")
    const thenable = {
      then(onFulfilled) {
        queueMicrotask(() => { // コールバック関数をマイクロタスクとして発行する
          onFulfilled(x + 1);
        });
      }
    };
    return thenable;
    // <4-t[2]> thenable.then の呼び出し
    // <6-t[3]> () => { resolve(42 + 1); } の実行
  })
  .then(x => { // <8-t[4]>
    console.log("🦄 [4] THENABLE", x)
  });

/* RESULT
❯ deno run nonPromiseThenable2.js
💙 [1]
💚 [2]
👻 [3] PROMISE 43
🦄 [4] THENABLE 43
*/
```

## 解決値の違いによる挙動のまとめ

`resolve` 関数の引数である `reslution` (解決値) の違いによって `resolve` 関数の処理は場合分けされます。そして、それによって発生するマイクロタスクの数が異なることになりました。結果をまとめておきます。

解決値の値の種類 | (ステップ) 起こる処理
--|--
元の promise そのもの | (step.7) 例外を throw して解決対象の promise を直ちに拒否する
オブジェクトでない値 | (step.8) 解決対象の promise をその解決値で直ちに履行する
Thenable ではないオブジェクト | (step.12) 解決対象の promise をその解決値で直ちに履行する
Thenable なオブジェクト | (step.13-16) `thenable.then(resolve, reject)` を呼び出すためのマイクロタスクが発行され、Thenable が Promise の場合にはさらにそのマイクロタスクから `resolve` 関数が追加のマイクロタスクとして発行されて、最終的に解決対象の promise が履行される

解決値が Promise オブジェクトの場合にはそれが settled になるまでに追加でマイクロタスクが２個発生することに注意してください。つまり、Promise chain において `then` メソッドのコールバックで Promise オブジェクトを返すとマイクロタスクが合計で３個発生することになります。

```js
/* <n> は発生しているマイクロタスクの追跡順番 */
Promise.resolve(42)
  .then(x => { // <1>
    return Promise.resolve(x + 1);
    // 🔥 コールバックから Promise を返しているので追加のマイクロタスクが２つ発生
    // <2> Promise.resolve(43).then(resolve, reject) の呼び出し
    // <3> resolve 関数の実行
  })
  .then(x => { // <4>
    console.log(x);
    // コールバックからは何も返していないので直ちに undefined で履行し、追加のマイクロタスクは発生しない
  });
// 合計で４個のマイクロタスクが発生している
```

上のコードでは、まず `x => { return Promise.resolve(x + 1);}` という `then` のコールバック関数が１個目のマイクロタスクとして発行されます。その時、コールバック関数の返り値が Promise オブジェクトなので追加のマイクロタスクが発生します。

具体的には `Promise.resolve(x + 1).then(resolve, reject)` の呼び出し自体を行う関数が２個目のマイクロタスクとして発行されてイベントループで処理されると、その `resolve` 関数が３個目のマイクロタスクとして発行されます。それが実行されると、`then` 自体から返る Promise オブジェクト自体 (元の未解決 Promise オブジェクト) が解決し、chain してある `then` のコールバック `x => console.log(x)` が４個目のマイクロタスクとして発行されます。

## 解決値が Promise chain の場合

解決値が Promise chain であった場合にはどうなるでしょうか。`then()` メソッドからは常に新しい Promise オブジェクトが返るので、もちろんこれは解決値が Promise オブジェクトである場合に相当します。ただし、発生するマイクロタスクの発生順番については混乱しやすいので注意してください。

比較しやすいように、まずは解決値が単純な `Promise.resolve(1)` で生成されただけの Promise の場合を考えてみます。

```js
// simpleReturnPromise.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: Promise chain の識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [1]");

Promise.resolve(1)
  .then(x => console.log("👻 [3]")) // <1-a[1]>
  .then(x => console.log("👻 [5]")) // <3-a[2]>
  .then(x => console.log("👻 [6]")) // <5-a[3]>
  .then(x => console.log("👻 [7]")) // <7-a[4]>
  .then(x => console.log("👻 [9] S: 終了")) // <9-a[5]>

Promise.resolve(2)
  .then(x => { // <2-b[1]>
    console.log("💙 [4]");
    const p1 = Promise.resolve(x + 1);
    return p1;
    // -> resolve(Promise.resolve(43))
    // 🔥 promise を返すので追加のマイクロタスクが２個発生
    // <4-b[2]> p1.then(reoslve, reject) の呼び出し
    // ↪ <6-b[3]> resolve 関数の実行
  })
  .then(x => console.log("💙 [8] P: 終了", x)); // <8-b[4]>

console.log("🦖 [2]");

/* RESULT
❯ deno run simpleReturnPromise.js
🦖 [1]
🦖 [2]
👻 [3]
💙 [4]
👻 [5]
👻 [6]
👻 [7]
💙 [8] P: 終了 3
👻 [9] S: 終了
*/
```

上記コードの以下の箇所に注目します。マイクロタスク `<2-b[1]>` がイベントループにおいて処理されるときに、追加発生するマイクロタスクの１つ目である PromiseResolveThenableJob がマイクロタスクキューへとただちにエンキューされます。

```js
Promise.resolve(2)
  .then(x => { // <2-b[1]>
    console.log("💙 [4]");
    const p1 = Promise.resolve(x + 1);
    return p1;
    // -> resolve(Promise.resolve(43))
    // 🔥 promise を返すので追加のマイクロタスクが２個発生
    // <4-b[2]> p1.then(reoslve, reject) の呼び出し
    // ↪ <6-b[3]> resolve 関数6-b[3]> resolve() の実行の実行
  })
  .then(x => console.log("💙 [8] P: 終了", x)); // <8-b[4]>
```

次に `Promise.resolve(2).then(callback)` によって最後の `then` メソッドから返る Promise オブジェクトが解決値となる場合を考えてみます。コールバック関数の `return` の値として直接書いても良いですが、分かりやすくするためにあえて `p2` という変数で定義しておきます。

```js
// simpleReturnPromiseChain.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: promise chain の識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [1]");

Promise.resolve(1)
  .then(x => console.log("👻 [3]")) // <1-a[1]>
  .then(x => console.log("👻 [5]")) // <3-a[2]>
  .then(x => console.log("👻 [7]")) // <6-a[3]>
  .then(x => console.log("👻 [8]")) // <8-a[4]>
  .then(x => console.log("👻 [10] A 終了")) // <10-a[5]>

Promise.resolve(2)
  .then(x => { // <2-b[1]>
    console.log("💙 [4]");

    const p2 = Promise.resolve(2).then(x => { //  <4-b[2]>
      console.log("💚 [6]");
      return x;
    });

    return p2;
    // -> resolve(p2)
    // 🔥 promise を返すので追加のマイクロタスクが２個発生
    // <5-b[3]> p2.then(resolve, reject) の呼び出し
    // ↪ <7-b[4]> resolve の実行
  })
  .then(x => console.log("💙 [9] B 終了", x)); // <9-b[5]>

console.log("🦖 [2]");

/* RESULT
❯ deno run simpleReturnPromiseChain.js
🦖 [1]
🦖 [2]
👻 [3]
💙 [4]
👻 [5]
💚 [6]
👻 [7]
👻 [8]
💙 [9] B 終了 2
👻 [10] A 終了
*/
```

上記コードの以下の箇所に注目します。マイクロタスク `<2-b[1]>` がイベントループにおいて処理されるときに注意してください。`p2` は `Promise.resolve(2)` から始まる Promise chain であり、`p2` 自体は最後の `then` メソッドから返る Promise オブジェクトを参照しています。マイクロタスク `<2-b[1]>` が処理されるとき、p2 の Promise chain の `then` メソッドのコールバックが直ちにマイクロタスク `<4-b[2]>` として発行されます(`Promise.resolve(2)` が最初から履行しているため)。

```js
Promise.resolve(2)
  .then(x => { // <2-b[1]>
    console.log("💙 [4]");

    const p2 = Promise.resolve(2).then(x => { //  <4-b[2]>
      console.log("💚 [6]");
      return x;
    }); // ← この then から返る Promise オブジェクトが p2 であり、p2.then メソッドが thenAction となる

    return p2;
    // -> resolve(p2)
    // 🔥 promise を返すので追加のマイクロタスクが２個発生
    // <5-b[3]> p2.then(resolve, reject) の呼び出し
    // ↪ <7-b[4]> resolve の実行
  })
  .then(x => console.log("💙 [9] B 終了", x)); // <9-b[5]>
```

コールバック最後で `return p2` という処理が実行されることで、内部的には `resolve(p2)` という解決関数に解決値 `p2` が渡されて実行されることになります。解決値 `p2` は Promise オブジェクトであるため、追加のマイクロタスクが発生します。１つ目のマイクロタスクが NewPromiseResolveThenableJob 操作で作成される PromiseResolveThenableJob です。このマイクロタスクは `p2.then(resolve, reject)` を呼び出す処理です。`p2` は `Promise.resolve(2).then(callback)` という Promise chain を代入していましたが、`then` から返る新しい Promise オブジェクトを実際には参照しているので、さらにその Promise オブジェクトは `then` メソッドを持っています。この `then` を実行するための処理が１つ目の追加のマイクロタスクであることに注意してください。このマイクロタスクも `<2-b[1]>` マイクロタスクの処理時において直ちに発行されるため、順番的には `<4-b[2]>` の次である `<5-b[3]>` となります(🔥 この順番は分かりづらいので注意してください)。

`then` メソッドのコールバック関数の戻り値は内部的には `resolve` 関数の引数である解決値 `resolution` として渡されるという話でした。上記の解決値による違いは結局は `resolve` 関数の引数が問題なので、以下のように直接的に `resolve` 関数で考えてみるのもよいでしょう。

```js
// simpleResolveRel.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: promise chain の識別 (a or b)
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
console.log("🦖 [1]");

new Promise(resolve => {
  const p1 = Promise.resolve("A");
  resolve(p1);
  // 🔥 追加のマイクロタスクが２つ発生
  // <1-a[1]> p1.then(resolve, reject) の呼び出し
  // ↪ <4-a[2]> resolve 関数の実行
}).then(y => {
    // <6-a[3]>
    console.log("💙 [4]", y);
    return 55;
  })
  .then(x => console.log("💙 [6] A: 終了", x)); // <8-a[4]>

new Promise(resolve => {
  // この中で直ちにマイクロタスクが２つ発生することに注意
  const p2 = Promise.resolve("B").then(y => { // <2-b[1]>
    console.log("💚 [3]", y);
    return 55;
  });

  resolve(p2);
  // 🔥 追加のマイクロタスクが２つ発生
  // <3-b[2]> p2.then(resolve, reject) の呼び出し
  // ↪ <5-b[3] resolve 関数の実行
}).then(x => console.log("💚 [5] B: 終了", x));// <7-b[4]>

console.log("🦖 [2]");

/* RESULT
❯ deno run simpleResolveRel.js
🦖 [1]
🦖 [2]
💚 [3] B
💙 [4] A
💚 [5] B: 終了 55
💙 [6] A: 終了 55
*/
```

上のコードで発生するマイクロタスクの順番を図示してみると以下のようになります。

![マイクロタスクの順番](/images/js-async/img_microtask-enqueu-sec.png)

## Promise chain のネストをフラット化する弊害

『[Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)』のチャプターで Promise chain はなるべくネストさせずにフラットにするようにと解説しましたが、上記の `then` のコールバックから返される値の種類によって発生するマイクロタスク数が異なることに影響を受けて Promise chain のネストをフラット化するとコンソール出力が行われるマイクロタスクが遅延します。

```js
// nestVsFlat.js
/* <n-t[m]> は発生しているマイクロタスクの追跡順番
  n: 全体のマイクロタスクのカウント
  t: フラット(f) か ネスト(n) か
  m: それぞれの処理の中でのマイクロタスクのカウント
*/
const increment = (num) => {
  return Promise.resolve(num + 1)
};

// フラット化すると A を出力するまでマイクロタスクを3個消費する
increment(1)
  .then(num => increment(num)) // ３個消費 <1-f[1]> <3-f[2]> <6-f[3]>
  .then(num => console.log("A", num)); // <8-f[4]>
  // ４個目のマイクロタスクで出力
  // chain 全体で４個のマイクロタスクが発生

// ネストしたままだと B を出力するまでマイクロタスクを1個消費する
increment(1)
  .then(num => { // <2-n[1]>
    return increment(num)
      .then(num => console.log("B", num));
      //    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ <4-n[2]>
      // ２個目のマイクロタスクで出力
      // 追加のマイクロタスクが２個発生
      // <5-n[3]> <7-n[4]>
  });
// chain 全体で４個のマイクロタスクが発生

/* RESULT
❯ deno run nestVsFlat.js
B 3
A 3
*/
```

フラット化していると処理の流れが見やすいですが、コールバックから返る値である Thenable の `then` メソッドの起動がマイクロタスクとして発生してしまうので、コンソール出力を行うコールバックのマイクロタスクが実行されるまでに追加でマイクロタスクが発生することになります。それぞれの Promise chain で発生するマイクロタスクの合計は同じですが出力が起きるためのマイクロタスクの発生順序が異なります。とはいっても、このような挙動の違いは理解の上では必要ですが、実用上なにかの問題があるというわけではないのでそこまで気にする必要はありません。

また、『[Promise chain と async/await の仕様比較](n-epasync-promise-spec-compare)』のチャプターで説明するようにそもそも仕様最適化されている async/await が使えるのであれば、async/await を使った方がマイクロタスクの発生を抑制した上でさらに読みやすいコードが書けます。

