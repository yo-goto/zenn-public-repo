---
title: "Thenable とネストについて"
cssclass: zenn
date: 2022-11-20
modified: 2022-11-21
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『Thenable とネストについて』
---

# このチャプターについて

Promise chain についての解説の間違いがあったため、このチャプターでは新しい内容と、その間違いについて修正して解説します。

:::message alert
誤った情報を記載して大変申し訳ございません。

具体的には以下の複数のチャプターにおいて解説が間違っている箇所があると考えられるため、このチャプターの内容について正確に記述できた時点で徐々にそれらも修正していく予定です。

影響の波及があるチャプター。
- [複数の Promise を走らせる](5-epasync-multiple-promises)
- [then メソッドは常に新しい Promise を返す](6-epasync-then-always-return-new-promise)
- [Promise chain で値を繋ぐ](7-epasync-pass-value-to-the-next-chain)
- [then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)
- [Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)
- [コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)

関連して構成し直せる可能性のあるチャプター。
- [Promise の基本概念](a-epasync-promise-basic-concept)
- [resolve 関数と reject 関数の使い方](g-epasync-resolve-reject)
- [catch メソッドと finally メソッド](h-epasync-catch-finally)
:::

# Thenable とは

JavaScript における Promise の実装は ECMASript 仕様に入る前にコミュニティベースでライブラリとして実装されてきたという歴史があります。

ECMAScript 仕様の Promise は元々 Promise/A+ という仕様から由来しているため、こちらを参照することで Promsie について理解できることがいくつかあります。

> 1.2.  “thenable” is an object or function that defines a `then` method.
> (https://promisesaplus.com より)
