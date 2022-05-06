---
title: "Promise の基本概念"
---

# State と Fate
ここでは Promise の基礎的な概念と用語を紹介しておきます。
紹介する用語はこちらのドキュメントを参考にしています。以下で解説する用語は自分の解釈が混じっていますが、このドキュメント自体は Mdn のお墨付きなので信用してください(ES6 仕様のドラフトです)。

https://github.com/domenic/promises-unwrapping/blob/master/docs/states-and-fates.md

不安な場合にはオリジナルを参照してじっくり考えてみてください。

## State
Promise インスタンスには次の３つの状態(**State**)があり、それぞれに排他的となっています(同時に１つの状態しか取りえないようになっています)。

- Pending(待機状態)
- Fullfilled(履行状態)
- Rejected(拒否状態)

**Settled**(決定状態、不変状態)は実際の状態ではなく、Pending(待機状態)であるかないかを言い表すための言葉です。**Pending でなければ Settled だと言えます**。

複数の Promise の完了を待つことができる Promise の静的メソッド `Promise.allSettled()` などの意味もこれです。Fullfilled か Rejected かは気にせずとりあえず Settled になっているかだけかだけに注意を払います。

重要なこととして、Settled になった Promise インスタンスの**状態は二度と変わりません**。

## Fate
Promise インスタンスには２つの運命(**Fate**)があり、それぞれに排他的になっています(同時に１つの運命しか取りえないようになっています)。

- Resolved(解決済み)
- Unresolved(未解決)

Resolved(解決済み) の Promise インスタンスは他の Promise インスタンスに従ってロックイン(**他の Promise インスタンスの状態に自身の状態が左右される状況**)されているか、Settled になっている場合を指します。

Resolved でなければ Unresolved であり、対象の Promise インスタンスに対して resolve や reject を試みることでその Promise の状態に影響を及ぼすことできる場合には、その Promise インスタンスは Unresolved です。

Fate の概念は非常に分かりづらいので図にしてみました。

![promiseStateFate](/images/js-async/img_promiseStateFate.jpg)*Promise の State と Fate の関係図*

中央の `?` が書かれた Promise インスタンスは他の Promise インスタンスに従っています。つまり、他の Promise インスタンスで resolve されて、`Promise.resolve(promise)` や `promise.then(callback)` において `.then(callback)` メソッドで返ってくる Promise インスタンスなどが該当します。このインスタンスは Pending 状態であっても、従っている Promise インスタンスが Settled になることで連鎖的に状態(State)が遷移するため、このインスタンスの運命(Fate)は Resolved となっています。

左の黄色の Promise インスタンスに対して何かしらの操作(resolve や reject) を行うとそのインスタンスの状態に影響があるので、この Promise インスタンスは Unresolved といえます(非常に分かりづらいですね😅)。

Unresolved な Promise インスタンスは必然的に Pending 状態です。上で述べたように、Pending 状態である Promise インスタンスのすべてが Unresolved ではないことに注意してください。

## 注意

翻訳した日本語で考えてしまうと用語がややこしくなるので、なるべくオリジナルの英単語を使って理解した方が良いです。特に日本語の "解決する" などの単語には注意を払った方が良いでしょう。

基本的には英語でもややこしく、"resolve" と言ったときに、単に fullfill(履行状態にするという意味) にすることと同じ意味で言っている場合があったりします。従って、"resolve" を考えるときは "resolve with ~" というように **何で resolve するのか** を考えると理解しやすくなります。"resolve with a plain value" というように、単なる値で resolve するなら fullfill であり、これは単に履行状態にするという意味で捉えることができます。

動詞の意味がややこしくなる理由は、動詞の元となる実際の `resolve()` メソッドの挙動が `reject()` に比べて複雑で、再帰性が関与してくるからです。

`resolve(promise)` というように Promise インスタンスで resolve を試みると unwrap という現象が起きて、その従っている Promsie インスタンスの状態に同化します。逆に、`reject(promise)` は unwrap ができないため単純に Rejected 状態に遷移します。

:::message
Unrapping については『resolve と reject の使い方』で解説しています。そちらを参照してください。
:::

難しいですが、Fate の概念は他の文章を読むときに役立ちます。または、`Promise.fullfill()` というメソッドが存在せずに `Promise.resolve()` というメソッドが存在している理由の理解に役立ちます。

あとは、`resolve()` や `Promise.resolve()` の挙動・意味をしっかり理解しようとすると必要になってきます。

動詞や名詞などの意味合いの違いは次の Stack overflow の解答がわかりやすいです。上の図もこちらに記載されているものを参考に作成しました。

https://stackoverflow.com/a/29269515/18328461

