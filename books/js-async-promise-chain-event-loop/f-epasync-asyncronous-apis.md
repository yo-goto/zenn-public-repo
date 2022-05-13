---
title: "非同期 API と環境"
---

# このチャプターについて
イベントループや Promise チェーンの話へ入る前に、非同期処理で大切な「API と環境」の話をしておきます。

というのも、自分自身がこの話の大切さに気づくのに非常に時間がかかったからです。非同期処理の学習には多くのトラップがあり、様々な誤解をしがちです。

:::message
非同期処理の学習は以下のように解説によく使われる文言を疑ったり、慎重に考える必要があります。

- 「JavaScript はシングルスレッド」
- 「非同期処理は並列処理じゃない」
- 「Promise は同期処理」
- 「async/await は Promise を意識することなく非同期処理が書ける」
- 「await は非同期処理の完了を待つ」

間違いではないが、現実のすべてを語っていない文言。情報が不足しすぎている文言。
個人の主張である文言。異なる情報を対比させないと誤解を招く文言など、トラップが盛りだくさんです。

初学者の視点だと**これらの言葉に惑わされる可能性が非常に高い**ので(自分自身が実際に惑わされた経験があります)、注意してください。
:::

ですが、非同期処理そのものの目的「**なぜ非同期処理をするのか**」についてぶれてしまうと土台が崩れてしまいます。このチャプターでは API と環境の話を通して非同期処理を行う目的を考え、その全体的な仕組みを掴みます。

# API を提供する環境
非同期処理の学習において見逃されがちなものの１つとして "**API**" が挙げられます。

実は非同期処理の基本的な仕組みを理解するためにはこの "API" の話を欠かすことができません。

非同期処理というのは JavaScript の中でも比較的高度な話題であり、そのカテゴリの中だけでも広範囲な内容や知識を扱うために、Promise や async/await といった ECMAScript(JavaScript の言語コア)の機能に目が行きがちです。そして Promise を扱う解説記事などでは、Promise 解説のために `setTimeout()` という API をいきなり使ってしまってしまっていることがよくあるため、この API が実際には何をやっていているのかということに目がいかないことがあります。

非同期処理では、ECMAScript の機能だけでなく、「非同期 API + コールバック関数」や「`await` 式 + 非同期 API」という形の処理が多いです。兎にも角にも、JavaScript の言語機能に非同期 API を組み合わせることが非同期処理のベーシックな使い方となります。

そういう訳で、非同期処理を理解するには非同期の Web API やランタイム API などを絶対に欠かすことはできませんし、その API 群を提供する**環境に注目すること**が重要となります。ECMAScript の目につく Promise や async/await といった言語機能だけ見ていても非同期処理の仕組みは理解できないので注意してください。

# 非同期処理の目的
そもそも「非同期処理の目的」とはなんでしょうか？

非同期的に何かをわざわざする必要が何であるのでしょうか？

ここで、非同期 Web API である `fetch()` メソッドについて考えてみましょう。`fetch()` は引数に URL を指定して実行することでネットワーク接続をしてデータを取得できます。しかし、ネットワーク接続というのはリクエストを投げて応答があるまで時間がかかるものです。

JavaScript はシングルスレッド言語であり、単一コールスタックに実行コンテキストが作成されて実行されていきます。ということはネットワーク接続という時間のかかる処理をメインスレッドで行ってしまったら、その最中に何もできませんね。また、取得したデータを使って何かをしたい場合もデータが取得できるまではそのデータを使った処理は何もできません。

特にブラウザ環境ではレンダリング処理やユーザーインタラクションなどから発火されるイベントの処理もメインスレッド(UI スレッド)で行われています。もしそのような時間のかかる処理もメインスレッドでやってしまったら、ユーザーがテキスト選択やボタンクリックといった操作も何もできない時間ができてしまいます。

そういう訳で `fetch()` でのネットワーク接続など時間がかかる処理そのものは JavaScript エンジン(コールスタックを管理するコンポーネント)自体は直接関与しません。そのエンジンを埋め込んでいる環境が代わりに並列的(parallel)にバックグラウンドで行ってくれます。

データが取得できたらコールスタックにコールバック関数の形で**取得データと共に通知させて**、そのデータを使う処理をその時点で行えるようにします。これが非同期処理です。

非同期処理は並列処理とよく勘違いされるために、「**非同期処理は並列処理ではない**」という注意書きや、「**JavaScript はシングルスレッド**」ということが強調されがちです。

これらの命題は確かに真実ではありますが、**現実のすべてではありません**。

`fetch(url)` の他にも非同期 API として `setTimeout(callback, delay)` という指定時間が経過したら登録しておいたコールバック関数を実行するというものがあります。これはコードをスケジューリングする訳ですから**平行(concurrent)処理**であるとみなせます。指定時間が経過するまでの間、**メインスレッドでは別のことができます**。ですが「時間を図る」という処理をしつつ別のことができるのはなぜでしょうか？

「時間を測る」ということは明らかに作業です。時間を図る作業がメインスレッドで行われていたらメインスレッドはブロッキングされてその間は何もできません。もしそうならタイマーの処理などほぼ意味がありませんね。というわけで、時間を測るという作業は API を介して環境が代わりにバックグラウンドで並列的(parallel)に行ってくれます。

さて、JavaScript はシングルスレッドで実行されるはずでしたが、このように非同期 API を介して複数のことができる環境が JavaScript エンジンだけではなく色々なものを機能として持っているからです。そして**環境はシングルスレッドなどではありません**。

例えば、Chrome といったモダンなブラウザ環境を代表として考えると、ブラウザはそもそも様々なタスクを行っているので、"シングルスレッド"などではなく、マルチプロセスアーキテクチャであり、様々なタスクの責務を持つ分離したプロセスの中に複数のスレッドを持つ形になっています。

https://developer.chrome.com/blog/inside-browser-part2/

Chrome では実際に以下のような複数のプロセスが作成されます。
- Browser process
- Renderer process
- GPU process
- Plugin process
- Extensions process
- Utility process

実際、JavaScript の非同期処理でよく語られるシングルスレッドは現実のブラウザ環境(Chrome)では「Renderer Process の Main thread」のことであり、`fetch(url)` のネットワーク接続やリクエストなどの処理を行っているのは「Browser process の Network thread」です。

そもそも最初に見てもらった『What the heck is the event loop anyway?』では「**一度に複数のことができるのはブラウザがランタイム以上のものであるからで、ブラウザから提供される Web APIs は実質的にスレッドである**」ということが実は語られていました。

>Right, so I've been kind of partially lying do you and telling you that JavaScript can only do one thing at one time. That's true the JavaScript Runtime can only do one thing at one time. It can't make an AJAX request while you're doing other code. It can't do a setTimeout while you're doing another code. **The reason we can do things concurrently is that the browser is more than just the Runtime**. So, remember this diagram, **the JavaScript Runtime can do one thing at a time, but the browser gives us these other things, gives us these we shall APIs, these are effectively threads**, you can just make calls to, and those pieces of the browser are aware of this concurrency kicks in.
>(以下の書き起こしページから引用)

https://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html

:::message
**補足** 

Philip Roberts 氏が言う "Runtime" とは Chrome ブラウザ環境の JS エンジンである V8 のことを言っています。Deno や Node は V8 を含んだ"環境"であり、色々な API を提供するので、ここではブラウザと等価なものとして考えてください。

それぞれ公式サイトでの文言。
- [Deno is a simple, modern and secure runtime for JavaScript and TypeScript that uses V8 and is built in Rust.](https://deno.land)
- [Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine.](https://nodejs.org/en/)

また、引用の "we shall APIs" はおそらく書き起こしの間違いで、実際の動画では "Web APIs" と言っています。
:::

ここまでくれば非同期処理の仕組みや目的がなんとなく分かると思います。

「**環境が並列的にバックグラウンドで作業している間もメインスレッドをブロッキングすることなく別の作業を続けられるようにすること**」が非同期処理の大きな目的となります。

そして「**環境が提供する機能である API を介して時間のかかる処理を環境に委任し、それが完了したら JavaScript のメインスレッドにその処理結果となるデータと共に通知させてその作業に関連する何か別の作業をコールバックの形で行う**」というのが非同期処理の全体的な仕組みとなります。

これを理解することで、非同期処理の学習にありがちな誤解を解くことができます。

非同期処理そのものは確かに並列処理ではありませんが、「**API を介して環境に委任した作業はバックグラウンドで並列に行われ、それが完了したら何かをメインスレッドで非同期的に行う**」というような両方が組み合わさって起きているという仕組みを理解する必要があります。

結果を取得できるまで時間のかかる並列的な作業などをバックグラウンドで行いたいから非同期処理という形でその完了を通知させるわけです。

もちろん環境が提供できる API には限りがありますし、自分で定義した時間のかかるような独自の処理を行いたい場合には普通の API ではどうにもなりません。しかし、Web worker API を使うことでメインスレッドではなく別のスレッドに分離してそのような時間のかかる処理を自分自身で本当に並列的に行なえます。

https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers

