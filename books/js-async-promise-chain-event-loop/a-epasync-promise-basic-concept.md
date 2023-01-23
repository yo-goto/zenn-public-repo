---
title: "Promise の基本概念"
cssclass: zenn
date: 2022-05-03
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『Promise の基本概念』
---

# このチャプターについて

このチャプターでは Promise の基本的な概念と用語を紹介しておきます。Promise の知識自体は他の解説やドキュメントなどで目にしていると思うので、簡単な説明自体は省いて本質的な部分のみにフォーカスして解説します。

Promise のコードについての具体的な解説は『[Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)』のチャプターで行います。

:::message alert
このチャプターでの解説では `then()` メソッドを持つ Promise ライクなオブジェクト "Thenable" については混乱をさけるために意図的に省いているので注意して下さい。Thenable については『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで ECMAScript 仕様と共に解説しています。
:::

# State と Fate

紹介する用語はこちらのドキュメントを参考にしています。以下で解説する用語は筆者の解釈が混じっていますが、このドキュメント自体は MDN のお墨付きなので信用してください(というか ES6 仕様のドラフトなので、正統な Promise についての根本的なコンセプトが記述されています)。

https://github.com/domenic/promises-unwrapping/blob/master/docs/states-and-fates.md

このドキュメントは以下のリポジトリの一部ですが、このリポジトリ自体は Promise についての ECMAScript 仕様の礎となったものなので、含まれる他のドキュメントなども後々見ていくと理解の助けとなる部分が多いです。

https://github.com/domenic/promises-unwrapping#promise-resolve-functions

プロポーザルのドラフトという形式であり、ECMAScript 全体から切り離されて Promise のみにフォーカスされているため、Promise 仕様にまつわる操作がどのようなものであるのかが簡単に俯瞰できます。ただし、ES6 時代の古いものではあるという認識は必要なので注意してください。

## State

Promise インスタンスには次の３つの状態(**State**)があり、それぞれに排他的となっています(同時に１つの状態しか取りえないようになっています)。

- Pending(待機状態)
- Fulfilled(履行状態)
- Rejected(拒否状態)

**Settled**(決定状態、不変状態)は実際の状態ではなく、Pending(待機状態)であるかないかを言い表すための言葉です。**Pending でなければ Settled だと言えます**。

複数の Promise の完了を待つことができる Promise の静的メソッド `Promise.allSettled()` などの意味もこれです。Fulfilled か Rejected かは気にせず、とりあえず Settled になっているかだけかだけに注意を払います。

重要なこととして、Settled になった Promise インスタンスの**状態は二度と変わりません**。

:::details 仕様上の補足
Promise の State 概念は ECMAScript 仕様上では [Promise インスタンスが持つ内部スロット](https://tc39.es/ecma262/#table-internal-slots-of-promise-instances)の一つである `[[PromiseState]]` で管理されています。
:::

## Fate

Promise インスタンスには２つの運命(**Fate**)があり、それぞれに排他的になっています(同時に１つの運命しか取りえないようになっています)。

- Resolved(解決済み)
- Unresolved(未解決)

Resolved(解決済み) の Promise インスタンスは他の Promise インスタンスに従ってロックイン(**他の Promise インスタンスの状態に自身の状態が左右される状況**)されているか、Settled になっている場合を指します。

Resolved でなければ Unresolved であり、対象の Promise インスタンスに対して resolve や reject を試みることでその Promise の状態に影響を及ぼすことできる場合には、その Promise インスタンスは Unresolved です。

Fate の概念は非常に分かりづらいので State と共に図にしてみました。

![promiseStateFate](/images/js-async/img_promiseStateFate.jpg)*Promise の State と Fate の関係図*

中央の `?` が書かれた Promise インスタンスは他の Promise インスタンスに従っています。つまり、他の Promise インスタンスで resolve されて、`Promise.resolve(promise)` や `promise.then(callback)` において `.then(callback)` メソッドで返ってくる Promise インスタンスなどが該当します。このインスタンスは Pending 状態であっても、従っている Promise インスタンスが Settled になることで連鎖的に状態(State)が遷移するため、このインスタンスの運命(Fate)は Resolved となっています。

左の黄色の Promise インスタンスに対して何かしらの操作(resolve や reject) を行うとそのインスタンスの状態に影響があるので、この Promise インスタンスは Unresolved といえます。実はこれを知っていたとしても活用するような場面自体がすくないので、とりあえず Fate については「ふーん」ぐらいの気分で理解しておいて必要になったらちゃんと理解するぐらいの気持ちで十分です。

Unresolved な Promise インスタンスは必然的に Pending 状態です。上で述べたように、Pending 状態である Promise インスタンスのすべてが Unresolved ではないことに注意してください。

:::details 仕様上の補足
Resolved であるかどうかということを示す Promise の Fate 概念は ECMAScript 仕様上では [CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions) 抽象操作によって作成される `resolve` 関数と `reject` 関数が持つ `[[AlreadyResolved]]` という内部スロットに真偽値で追跡されています。この値が `true` なら Fate は Resolved であり、`false` なら Fate は Unresolved ということになります。
:::

## 注意

翻訳した日本語で考えてしまうと用語がややこしくなるので、なるべくオリジナルの英単語を使って理解した方が良いです。特に日本語の "解決する" などの単語には注意を払った方が良いでしょう。

とは言っても基本的に英語でもややこしく、"resolve" と言ったときに、単に fulfill(履行状態にするという意味) にすることと同じ意味で言っている場合があったりします。従って、"resolve" を考えるときは "resolve with ~" というように **何で resolve するのか** を考えると理解しやすくなります。"resolve with a plain value" というように、単なる値で resolve するなら fulfill であり、これは単に履行状態にするという意味で捉えることができます。

動詞の意味がややこしくなる理由は、動詞の元となる実際の `resolve()` 関数の挙動が `reject()` 関数に比べて複雑であり、引数の値の種類によって処理的な場合分けがあるからです。`resolve(promise)` というように Promise インスタンスで resolve を試みると unwrap という現象が起きて、その従っている Promise インスタンスの状態に同化します。逆に、`reject(promise)` は unwrap を行わずシンプルに Rejected 状態に遷移するような操作となっています。

:::details 仕様上の補足
`resolve` 関数の挙動について仕様的に何が起きるのかについては『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで解説していますが、簡易的に説明すると `resolve` 関数の挙動を定義しているのは ECMAScript 仕様上の [Promise Resolve Functions](https://tc39.es/ecma262/#sec-promise-resolve-functions) であり、`reject` 関数の挙動は [Promise Reject Functions](https://tc39.es/ecma262/#sec-promise-reject-functions) という項目です。

![resolveとrejectの仕様比較](/images/js-async/img_spec-diff-resolve-reject.jpg)

Promise Reject Functions の仕様が短いのに対して Promise Resolve Functions の仕様は長く複雑になっています。そして Promise Resolve Functions つまり `resolve` 関数はその引数の値の種類によって処理が場合分けされるようになっています。
:::

仕様が理解できるようになってくると分かることですが、このように解決(Resolution) によって起きることは図にある通り履行(Fulfillment)や引数に取った Promise オブジェクトの状態に応じて拒否(Rejection)を起こすなど多数の場合があるので注意してください。

:::message
Unwrapping については『[resolve 関数と reject 関数の使い方](g-epasync-resolve-reject)』で解説しています。そちらを参照してください。
:::

難しいですが、Fate の概念は他の文章を読むときには役立つことがあります。または、`Promise.fulfill()` というメソッドではなく `Promise.resolve()` というメソッドが存在している理由の理解に役立ちます。あとは、`resolve()` や `Promise.resolve()` の挙動・意味をしっかり理解しようとすると必要になってきます。

動詞や名詞などの意味合いの違いは次の Stack overflow の解答がわかりやすいです。上の図もこちらに記載されているものを参考に作成しました。

https://stackoverflow.com/a/29269515/18328461
