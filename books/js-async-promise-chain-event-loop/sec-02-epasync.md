---
title: "第２章 - Promise インスタンスと連鎖"
cssclass: zenn
date: 2022-06-16
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
alieases: [EPAsync 第３章]
---

# この章について

第２章では Promise インスタンスと Promise chain について見ていきます。Promise chain は非同期処理の処理予測を行うためのメンタルモデルを構築する上で基礎となる非常に重要な概念です。Promise chain の具体的なコード実行を通じて非同期処理の実行順序を予測できるようにこの章で訓練していきます。

この章の内容をしっかり学ぶことで、[第３章](sec-03-epasync)の async/await を理解できるようになります。

:::message
この本の執筆経緯的にも Promise chain についての解説である第２章をはじめに執筆しており難易度としては簡単なので、[第１章](sec-01-epasync)の内容が難しい場合には第２章から読み始めるのも一つの手です。
:::

:::message alert
この章のチャプターで大きな間違いが見つかりました。

`then()` メソッドのコールバックから返る値の種類によってマイクロタスクの発生数が異なるため、現在行っている解説だと以下のようなコードの実行順序が理解できなくなります。

```js
console.log("🦖 [1]");

Promise.resolve()
  .then(() => {
    // <1-sync>
    console.log("💙 [3]")
    // return Promise.resolve(2);
    return new Promise(resolve => {
      resolve(2);
    });
    // <3-async> <5-async>
  })
  .then((d) => {
    // <7-async>
    console.log("💙 [7]", d);
  });

Promise.resolve()
  .then(() => {
    // <2-sync>
    console.log("💚 [4]");
    return 1;
  })
  .then((d) => {
    // <4-async>
    console.log("💚 [5]", d);
  })
  .then(() => {
    // <6-async>
    console.log("💚 [6]")
  });


console.log("🦖 [2]");

/* RESULT
❯ deno run simpleDiff.js
🦖 [1]
🦖 [2]
💙 [3]
💚 [4]
💚 [5] 1
💚 [6]
💙 [7] 2
*/
```

この理由としては、`then()` メソッドに渡すコールバックから Promise などの Thenable オブジェクト(`.then()` メソッドを持つオブジェクト) が返されると、もとの `then()` メソッドから返される Promsie を解決するために通常の値を返す時に加えて追加で２つのコールバックがマイクロタスクとして発生します。

現在の解説だと、上記のような普通の値を返す chain と Promise を返す chain の比較がないため、一見正しい解説のように見えてしまっているはずですが、コールバック関数で Promise を返している場合、実際にはマイクロタスクが追加で２つ発生しています。

この挙動についての間違いから以下の解説のチャプターが影響を受けます。

- [複数の Promise を走らせる](5-epasync-multiple-promises)
- [then メソッドは常に新しい Promise を返す](6-epasync-then-always-return-new-promise)
- [Promise chain で値を繋ぐ](7-epasync-pass-value-to-the-next-chain)
- [then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)
- [Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)
- [コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)

さらに関連して Promise chain のネストをフラット化することでマイクロタスクの発生数が異なる現象が起きるようです。

```js:nestVsFlat.js
const increment = (num) => {
  return Promise.resolve(num + 1)
};

// フラット化すると A を出力するまでマイクロタスクが 3個 かかる
increment(1)
  .then(num => increment(num))
  .then(num => console.log("A", num));

// ネストしたままだと B を出力するまでマイクロタスクが 1個 かかる
increment(2)
  .then(num => {
    return increment(num)
      .then(num => console.log("B", num));
  });

/* RESULT
❯ deno run nestVsFlat.js
B 4
A 3
*/
```

その挙動についての詳細な解説を行うためには仕様の理解が必要なため現在調査中です。間違った情報を公開してしまい申し訳ございません。とりあえずのところ、この話題以外の async/await といった他の解説には影響は無いようです。

調査中の内容は以下のブログ記事で追跡してまとめています。

- [JavaScript thenメソッドのコールバックについての勘違い](https://publish.obsidian.md/ankiyorihajimeyo/TSJS/JavaScript+then%E3%83%A1%E3%82%BD%E3%83%83%E3%83%89%E3%81%AE%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6%E3%81%AE%E5%8B%98%E9%81%95%E3%81%84)
- [JavaScript Promise chainをネストする弊害](https://publish.obsidian.md/ankiyorihajimeyo/TSJS/JavaScript+Promise+chain%E3%82%92%E3%83%8D%E3%82%B9%E3%83%88%E3%81%99%E3%82%8B%E5%BC%8A%E5%AE%B3)

挙動の詳細が理解できしだい、新しいチャプターを追加し、波及しているチャプターの内容を順次修正してきます。
:::

# チャプター

- [Promise の基本概念](a-epasync-promise-basic-concept)
- [Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)
- [コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async)
- [resolve 関数と reject 関数の使い方](g-epasync-resolve-reject)
- [複数の Promise を走らせる](5-epasync-multiple-promises)
- [then メソッドは常に新しい Promise を返す](6-epasync-then-always-return-new-promise)
- [Promise chain で値を繋ぐ](7-epasync-pass-value-to-the-next-chain)
- [then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)
- [Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)
- [コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)
- [アロー関数で return を省略する](11-epasync-omit-return-by-arrow-shortcut)
- [catch メソッドと finally メソッド](h-epasync-catch-finally)
- [古い非同期 API を Promise でラップする](12-epasync-wrapping-macrotask)
- [イベントループは内部にネストしたループがある](13-epasync-loop-is-nested)
