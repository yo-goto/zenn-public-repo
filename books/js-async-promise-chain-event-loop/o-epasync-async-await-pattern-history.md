---
title: "async/await Pattern の歴史的発展"
cssclass: zenn
date: 2023-03-29
modified: 2023-03-29
url: "https://zenn.dev/estra/articles/m-epasync"
AutoNoteMover: disable
tags: [" #type/zenn/book #JavaScript/async  "]
aliases: Promise本『async/await Pattern の歴史的発展』
---

## async/await Pattern

これまで JavaScript や TypeScript における async/await のシンタックスとイベントループなどの非同期処理の背景となる機構を見てきましたが、実は async/await というシンタックスは JavaScript 特有のものではりません。

他のプログラミング言語でもこの構文は存在しており、一般には「async/await Pattern」と呼ばれます。このパターンは JavaScript だけではなく、C#、C++、Python、Kotlin、Rust、Swift、Zig などの多くの言語で非同期の振る舞いを記述するための機能として提供されています。

https://en.wikipedia.org/wiki/Async/await

## 歴史的な発展の経緯

async/await のパターンが一般化するまで、C# をはじめとする多くの言語で非同期処理を記述する方法が試行錯誤された歴史が存在しています。実際この async/await パターン (シンタックス) は JavaScript (ECMAScript) に導入されるはるか前に、C# 言語で生まれました。また、C# と TypeScript はいずれも Microsoft から提供されており、言語開発者は同じ [Anders Hejlsberg](https://en.wikipedia.org/wiki/Anders_Hejlsberg) 氏であり、async/await のシンタックスも ECMAScript の仕様に導入される前に TypeScript で先に実装されたという経緯もあるので、これらの言語の間には深い関係があります。

やくもけ ([@yakumokech](https://twitter.com/yakumokech?s=20)) さんの以下の動画でその歴史について分かりやすく解説されていたので、ぜひ参考にしてみてください。

https://youtu.be/3Od9rhxlS-E

動画の中で語られていますが、C# の [Task](https://learn.microsoft.com/ja-jp/dotnet/api/system.threading.tasks.task?view=net-8.0) インスタンスと `ContinueWith` メソッド、それらに近い Promise インスタンスと `then` メソッドとの関係や、Haskell などの関数型言語の影響も受けている経緯など非常に勉強になります。

特に最初からイベントループやスレッドプールが存在しない Python や Rust などの言語では非同期処理を利用できる実行環境を明示して async/await を利用する必要があるそうです。実際、Deno ランタイムのバックで利用されている [Tokio](https://tokio.rs) は Rust 用のサードパーティランタイムであり、Rust で非同期処理を利用できるようにしています (ちなみに Rust の非同期処理は Promise オブジェクトではなく Future オブジェクトを利用します)。

このように他のプログラミング言語での async/await パターンについて知ることで、JS の async/await を超えて「非同期処理」とはなにか、どのように記述すればいいのかについて、複数の言語間で比較・相対化・一般化して学習することができます。
