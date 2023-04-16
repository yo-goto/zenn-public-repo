---
title: "async/await Pattern の歴史的発展"
cssclass: zenn
date: 2023-03-29
modified: 2023-04-16
url: "https://zenn.dev/estra/articles/o-epasync-async-await-pattern-history"
AutoNoteMover: disable
tags: [" #type/zenn/book #JavaScript/async  "]
aliases: Promise本『async/await Pattern の歴史的発展』
---

## async/await Pattern

これまで JavaScript における async/await のシンタックスとイベントループなどの非同期処理の背景となる機構を見てきましたが、実は async/await というシンタックスは JavaScript 特有のものではありません。

他のプログラミング言語でもこの構文は存在しており、一般には「async/await Pattern」と呼ばれます。このパターンは JavaScript だけではなく、C#、C++、Python、Kotlin、Rust、Swift、Zig などの多くの言語で非同期の振る舞いを記述するための機能として提供されています。

https://en.wikipedia.org/wiki/Async/await

## 歴史的な発展

async/await のパターンが一般化するまで、多くのプログラミング言語で非同期処理を記述する方法が試行錯誤された歴史が存在しています。実際この async/await パターン (シンタックス) は C# という言語で誕生した後の数年後に JavaScript (ECMAScript) を始めとする様々な言語で導入されました。

実は C# と TypeScript はいずれも Microsoft から提供されている上に言語開発者も同じ [Anders Hejlsberg](https://en.wikipedia.org/wiki/Anders_Hejlsberg) 氏であるという驚愕の事実があります。また、async/await のシンタックスも ECMAScript の仕様に導入される前に TypeScript で先に実装されたという経緯もあるので、これらの言語の間には明らかに深い関係があります。

このような async/await パターンの発展の歴史については、やくもけ ([@yakumokech](https://twitter.com/yakumokech?s=20)) さんの以下の動画にて分かりやすく解説されていたので、ぜひ参考にしてみてください。

https://youtu.be/3Od9rhxlS-E

動画の中で語られていますが、C# の [Task](https://learn.microsoft.com/ja-jp/dotnet/api/system.threading.tasks.task?view=net-8.0) オブジェクトと `ContinueWith` メソッド、それらに近い Promise オブジェクトと `then` メソッドとの関係や、Haskell などの関数型言語の影響も受けている経緯など非常に勉強になり、面白いです。

特に素の状態ではイベントループやスレッドプールが存在しない Python や Rust などの言語では、非同期処理を利用できる実行環境を明示して async/await のシンタックスを利用する必要があるそうです。実際、Deno ランタイムのバックで利用されている [Tokio](https://tokio.rs) は Rust 用のサードパーティランタイムであり、このランタイムによって Rust で非同期処理を利用できるようにしています (ちなみに Rust の非同期処理は Promise オブジェクトではなく類似した概念である Future オブジェクトを利用し、await キーワードの配置も前置ではなく後置となります)。

https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html

このように他のプログラミング言語での async/await パターンについて知ることで、JS の async/await を超えて「非同期処理」とはなにか、どのように記述すればいいのかについて、相対化・一般化して学習することができます。
