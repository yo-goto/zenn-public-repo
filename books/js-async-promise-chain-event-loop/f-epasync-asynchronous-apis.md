---
title: "非同期 API と環境"
aliases: [ch_非同期 API と環境]
---

# このチャプターについて
このチャプターでは非常に重要な話をしますが、それを理解するための前提知識がいくつか必要になってきますので、第２章や第３章のチャプターで解説する知識を使っています。分からない部分についてはすぐに理解できなくてもいいので読み進めて後から戻ってきてください。

それでは、イベントループや Promise chain の話へ入る前に、非同期処理で大切な「API と環境」の話をしておきます。なぜこの話を先にしておくかというと、自分自身がこの話の大切さに気づくのに非常に時間がかかったからです。

そもそも、非同期処理の学習には多くのトラップがあり、様々な誤解をしがちです。ですが、非同期処理そのものの目的「なぜ非同期処理をするのか」についてぶれてしまうと土台が崩れてしまいます。このチャプターでは API と環境の話を通して非同期処理を行う目的を考え、その全体的な仕組みを掴みます。

:::message
**非同期処理の学習では言葉に惑わされないことが重要**

「非同期」や「同期」、「await」といった単語レベルでもそうですが、言葉に惑わされる(混乱させられる)ケースが非常に多いと感じました。実際、非同期処理の学習では以下のような解説によく使用される文言に対して疑ったり、慎重に考える必要があります。

- 「JavaScript はシングルスレッドで実行される」
- 「非同期処理は並列処理ではない」
- 「Promise は同期処理」
- 「async/await は Promise を意識することなく非同期処理が書ける」
- 「await は非同期処理の完了を待つ」

**間違いではないが現実のすべてを語っていない文言**、情報が不足しすぎており異なる情報を対比させないと誤解を招く文言、個人の主張である文言など、トラップが盛りだくさんです💣

初学者の視点だと**これらの言葉に惑わされる可能性が非常に高い**ので、注意してください(自分自身が実際に惑わされた経験があります)。
:::

# 非同期処理の解説で見過ごされがちな話
非同期処理の学習において見逃されがちなものの１つとして "API" が挙げられます。実は非同期処理の基本的な仕組みを理解するためにはこの "API" の話を欠かすことができません。

:::message alert
非同期処理の学習において、**最もトラップとなるポイント**がこの話だと考えています。それ故、このチャプターを最初に持ってきました。
:::

非同期処理というのは JavaScript の中でも比較的高度な話題であり、そのカテゴリの中だけでも広範囲な内容や知識を扱うために、Promise や async/await といった ECMAScript(JavaScript の言語コア)の機能に目が行きがちです。そして Promise を扱う解説記事などでは、Promise 解説のために `setTimeout()` という API をいきなり使ってしまってしまっていることがよくあるため、この API が実際には何をやっていているのかということに目がいかないことがあります。

非同期処理では、ECMAScript の機能だけでなく、「非同期 API + コールバック関数」や「`await` 式 + 非同期 API」という形の処理が多いです。このように ECMAScript の言語機能(コールバック関数、Promise, async/await など)に非同期 API (`fetch()` や `setTimeout()` など)を組み合わせることが「非同期処理」のベーシックな考え方・使い方となります。

そういう訳で、非同期処理を理解するには非同期の Web APIs や Runtime APIs などを絶対に欠かすことはできませんし、その API の機能を提供する**環境に注目すること**が重要となります。非同期処理の話題で目につく Promise や async/await といった ECMAScript の言語機能だけを見ていても非同期処理の仕組みや実行順序について理解できないので注意してください。この話が非同期処理の学習において最大の罠であるのは、**Promise や async/await の話だけを追っても「非同期処理についての謎」が永遠に解けないようになっているからです**。

# API の機能を提供する環境について
そもそもの話をしますが、JavaScript には常に**実行する環境**(**environment**)があり、その環境によって利用できる機能が異なります。環境による相違点として顕著なものが環境の提供する機能である API です。

一方で、JavaScript は ECMAScript という仕様に基づいて動作が定められているため、実行する環境が異なっても共通する動作があります。別のチャプターで解説しますが、実際には環境に埋め込まれた JavaScript エンジンが ECMAScript を実装しています。例えば Chrome ブラウザ環境に埋め込まれている V8 という JavaScript エンジンは、Node や Deno といったランタイム環境にも利用されています。クライアントサイド(ブラウザ環境)やサーバーサイド(ランタイム環境)で JavaScript が同じように使えるのはこの JavaScript エンジンのおかげです。より正確には "ECMAScript エンジン" とでも呼べる代物ですね。

https://jsprimer.net/basic/introduction/#javascript-ecmascript

API は ECMAScript とはほとんど関係なく環境が独自に定義して提供しているものです。ただし、使い勝手が同じになるように `setTimeout()` や  `console.log()` など様々な環境で同じ名前で同じ使い勝手の API が提供されている場合がよくあります。実際には機能的にそれぞれ微妙に違うことがあります(自分がそうなのですが、この事実に初学者は最初気づけずに混乱します)。

重要なので再確認しますが、API というのは**環境に特有の機能**であり、ECMAScript の一部ではありません。

>It turns out that the way we farm out work in JavaScript is **to use environment-specific functions and APIs. And this is a source of great confusion in JavaScript**.
>
>**JavaScript always runs in an environment**.
>
>Often, that environment is the browser. But it can also be on the server with NodeJS. But what on earth is the difference?
>
>The difference – and this is important – is that the browser and the server (NodeJS), functionality-wise, **are not equivalent. They are often similar, but they are not the same**.
>([Async Await JavaScript Tutorial – How to Wait for a Function to Finish in JS](https://www.freecodecamp.org/news/async-await-javascript-tutorial/) より引用、太字は筆者強調)

この「環境」ですが、例えば、この本を見ている Chrome や Safari などのブラウザも環境の１つです。ブラウザ環境以外には、Node や Deno などのサーバーサイドで使えるランタイム環境などもあげられます。

- ブラウザ環境 : Chrome, Safari, Firefox など
- ランタイム環境 : Node, Deno など

API にも色々な種類と分け方がありますが、ブラウザによって提供されるものは MDN では **Web APIs** として呼ばれています。この名前はかなり混乱を生じさせるものです。実態としては **Browser APIs** と呼ぶべきで、ブラウザ環境によって提供される API のことを指している場合が多いです。大抵の場合は、Web APIs といったらブラウザ環境が提供する機能のことだと考えてください。

https://developer.mozilla.org/ja/docs/Web/API

ちなみに「Web API」から直感的に想起する Web 上で提供されている API(何らかの Web サービスから提供される API)は MDN では「[サードパーティ API(Third-party APIs)](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Client-side_web_APIs/Third_party_APIs#restful_api_%E2%80%94_nytimes)」として呼ばれています。

ランタイム環境で提供される API は便宜的にまとめて Runtime APIs として呼称します。

Web APIs も Runtime APIs にも同期型の API と非同期型の API が存在しています。例えば、`setTimeout()` が非同期の API です。Node, Deno, Chrome などの実行環境に関わらず同じ名前の API として `setTimeout()` が提供されており、使い勝手はほとんど同じですが、それらは環境が独自に定義していることに注意してください。

:::message
ちなみに Deno ランタイム環境で提供される Runtime APIs は大きく以下の２要素で成り立っています。

- 1. [Web Platform APIs](https://deno.land/manual/runtime/web_platform_apis): `fetch()` のような Web APIs と同じ名前、同じ使い勝手になるように提供されている API 群
- 2. Deno global: 非同期 I/O などに関する Deno 環境での完全独自定義の API 群(global の `Deno` ネームスペースに含まれる `Deno.readFile()` というように `Deno` から始まる名前の関数など)

参考: [The Runtime | Manual | Deno](https://deno.land/manual/runtime)
:::

JavaScript は元々ブラウザ環境でのみ使われる言語だったので、ブラウザ環境が提供する Web API の方が歴史が長い訳です。従って、Node や Deno といったランタイム環境ではこの Web API を同じ名前で同じ使い勝手で開発者が使えるように用意してくれています。Console API の `console.log()` メソッドが分かりやすいでしょう。この `console.log()` は基本的にどんな環境でも使えます。ランタイム環境はブラウザ環境で走る JavaScript との互換性を高めて、JavaScript プログラマーがある程度同じ使い勝手のもとして推論して使えるように準備してくれている訳です。

ただし、ブラウザ環境とランタイム環境で Web APIs(Web Platform APIs) の互換性がある程度あったとしても、ブラウザ環境に無いような API(Deno なら Deno globals) についてはそれぞれのランタイム環境が独自に定義しているために基本的には互換性はありません。Node や Deno ではそういった API の名前や設計思想が異なる点がありますからね。

そして、最近になってランタイム環境間での Web API の互換性を高める目的で [WinterCG(Web-interoperable Runtimes Community Group)](https://wintercg.org)という団体が発足しました。サーバーランタイム(Deno / Node.js) とエッジランタイム(Cloudflare Workers / Deno)の環境間で Web Platform API (`fetch()` や `setTimeout()` や `queueMicrotask()` など)の互換性を高めようというのが目的です。開発者は今後、より互換性の高い JavaScript で開発ができるようになると期待できます。

>The ultimate goal of the group is to promote **runtimes supporting a comprehensive unified API surface** that JavaScript developers can rely on **regardless of the runtime they are using**: be it browsers, servers, embedded applications, or edge runtimes.
>
>The members of the group want to provide a space to better coordinate between browser vendors and other implementors on how Web Platform APIs can be best implemented and used outside of browsers.
>([What are we trying to do? | WinterCG](https://wintercg.org/faq#what-are-we-trying-to-do) より引用、太字は筆者強調)

# 「非同期処理」の目的と仕組み
環境と API についての予備知識を入れたので、このチャプターの本題である「非同期処理の目的」について考えてみましょう。

そもそも「非同期処理」ですが、わざわざ(**実行のタイミングをずらして**)非同期的に何かをする必要がなぜあるのでしょうか？

:::message alert
「非同期」の概念そのものについてはこのチャプターの最後で解説します。今は実行のタイミングがコードの配置とずれてしまう処理のことを指していると考えてください。また、「同期」もこの時点ではソースコードの配置と実行のタイミングがずれないものであると考えてください。
:::

ここで、非同期 API である `fetch()` メソッドについて考えてみましょう。`fetch()` は引数に URL を指定して実行することでネットワーク接続をしてデータを取得できます。しかし、ネットワーク接続というのはリクエストを投げて応答があるまで時間がかかるものです。

ところで、JavaScript はシングルスレッド言語であり、プログラムは**シングルスレッド(単一のスレッド)で実行**されます。シングルスレッドで実行されるとはつまり、**単一コールスタック(single callstack)で実行される** ということです(これについては別のチャプター『[コールスタックと実行コンテキスト](b-epasync-callstack-execution-context)』で解説します)。この「シングルスレッド」ですが、ブラウザ環境の文脈で言えば JavaScript は(UI スレッドとも呼ばれる)**メインスレッド**で実行されます。

JavaScript は「シングルスレッド(単一のスレッド)で実行される」わけですから、この"メインスレッド"とやらを長時間専有してしまうような処理があると、その間は何もできなくなってしまうわけです。実際に長時間メインスレッドを専有することを「**ブロッキング(blocking)**」と言います。

`fetch()` の話に戻りますが、ということはネットワーク接続という時間のかかる処理をメインスレッドで行ってしまったら、その最中には何もできなくなってしまいます。ブラウザならクリックやテキスト選択といった操作も何もできません。また、取得したデータを使って何かをしたい場合もデータが取得できるまではそのデータを使った処理は何もできません。

ブラウザ環境ではレンダリング処理やユーザーインタラクションなどから発火されるイベントの処理もメインスレッド(= UI スレッド)で行われています。もし、`fetch()` によるデータ取得をメインスレッドでやってしまったら、それこそブロッキングが起きるはずで、ブロッキングが起きている間は、ユーザーがテキスト選択やボタンクリックといった操作も何もできない時間ができてしまいます(なぜなら、そのような**ユーザー操作によるイベントもメインスレッドで処理されるからです**)。

メインスレッド(Main thread)については実際に MDN で次のように説明されています。

>メインスレッドは、ブラウザーがユーザーのイベントや描画を処理するところです。既定では、ブラウザーは単一のスレッドを使用してページ内のすべての JavaScript を、レイアウト、再フロー、ガベージコレクションなどと同様に実行します。つまり、実行に時間がかかる JavaScript 関数がスレッドをブロックし、ページが反応しなくなり、使い勝手が悪くなります。
>([Main thread (メインスレッド) - MDN Web Docs 用語集: ウェブ関連用語の定義 | MDN](https://developer.mozilla.org/ja/docs/Glossary/Main_thread) より引用)

そういう訳で `fetch()` でのネットワーク接続など時間がかかる処理そのものは JavaScript エンジン(コールスタックを管理するコンポーネント)自体は直接関与しません。そのエンジンを埋め込んでいる環境が代わりに並列的にバックグラウンドで行ってくれます。

データが取得できたらコールスタックにコールバック関数の形で**取得データと共に通知させて**、そのデータを使う処理をその時点で行えるようにします。これが非同期処理です。

非同期処理は並列処理とよく勘違いされるために、「**非同期処理は並列処理ではない**」という注意書きや、「**JavaScript はシングルスレッドで実行される**」ということが強調されがちです。

これらの命題は確かに真実ではありますが、**現実のすべてを語ってはいません**。

`fetch(url)` の他にも非同期 API として上で挙げた `setTimeout(callback, delay)` という指定時間が経過したら登録しておいたコールバック関数を実行するというものがあります。これは、第一引数に渡したコールバック関数が指定時間後に実行されるようにスケジューリングする訳ですから、**並行(concurrent)処理**であるとみなせます。そして、指定時間が経過するまでの間、**メインスレッドでは別のことができます**。ですが「時間を図る」という処理をしつつ別のことができるのはなぜでしょうか？

例えば、次のコードでは、タイマーを順番に起動しますが。タイマーの「時間を図る」という処理が終わっていないにも関わらず次のタイマーを起動できたり、コンソールに出力したりできるのは何故でしょうか？

```js:parallelTimer
// コールバック関数をアロー関数で定義
setTimeout(() => {
  console.log("[A] 並列的なタイマー");
}, 1000);
// 上のタイマー処理が終わっていないのに次のタイマーを起動
setTimeout(() => {
  console.log("[B] 並列的なタイマー");
}, 1500);
// 上のタイマー処理が終わっていないのに次のタイマーを起動
setTimeout(() => {
  console.log("[C] 並列的なタイマー");
}, 500);
// 上のタイマー処理が終わっていないのにコンソールへ出力
console.log("タイマーを起動しているのにコンソールに出力ができてしまう");
```

実際にこのコードを実行すると次の出力を得ます。

```sh
❯ deno run parallelTimer.js
タイマーを起動しているのにコンソールに出力ができてしまう
[C] 並列的なタイマー
[A] 並列的なタイマー
[B] 並列的なタイマー
```

`setTimeout()` については『[タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues#settimeout-api)』で詳しく解説しますが、このようにタイマーによって「時間を測る」ということは明らかに作業です。時間を図る作業がメインスレッドで行われていたら**メインスレッドはブロッキングされてその間は何もできません**。もしそうならタイマーの処理などほぼ意味がありませんね。

```js
// コールバック関数をアロー関数で定義
setTimeout(() => {
  console.log("[2] 1000ミリ秒測るタイマー");
}, 1000);
// もしブロッキングされていたら次の処理はタイマーが終わってからでないと実行できない
consle.log("[1] 実際には時間を図りつつ他のことができる"); 

/* コンソールへの出力は次のようになる
[1] 実際には時間を図りつつ他のことができる
[2] 1000ミリ秒測るタイマー
*/
```

現実では、時間を図りつつ他の JavaScript コードを処理しているわけで、**同時に複数のことをやっている**わけです。

というわけで、時間を測るという作業やインターネットを介したデータ取得などの行為は実は API を介して**環境が代わりにバックグラウンドで並列的に行ってくれます**。つまり、 API の呼び出しは**環境へ作業を委任する(delegate)という行為**だった訳です。

:::message
次の freecodecamp の記事からの引用を見てもらえばわかると思いますが、例えばジャグリングをしていて両手がふさがっている最中に新しくボールを追加しようとしたら、誰かに手伝ってもらう必要があります。一人では同時に複数のことができなくても、誰かに作業を委任(delegate)して手伝ってもらうことで、効率よく作業をすすめることができます。

>Let's circle back to the juggling example. What would happen if you wanted to add another ball? Instead of six balls, you wanted to juggle seven balls. That's might be a problem.
>
>You don't want to stop juggling, because it's just so much fun. But you can't go and get another ball either, because that would mean you'd have to stop.
>
>The solution? **Delegate the work to a friend or family member**. They aren't juggling, so they can go and get the ball for you, then toss it into your juggling at a time when your hand is free and you are ready to add another ball mid-juggle.
>(中略)
>**It turns out that it is the environment that takes on the work, and the way to get the environment to do that work, is to use functionality that belongs to the environment**. For example fetch or setTimeout in the browser environment.
>([Async Await JavaScript Tutorial – How to Wait for a Function to Finish in JS](https://www.freecodecamp.org/news/async-await-javascript-tutorial/) より引用、太字は筆者強調)

そして、JavaScript の文脈で言えば、作業を手伝ってくれるのが「環境」というわけです。実際にジャグリングしているのは「Runtime(JavaScript エンジン)」であり、その作業を手伝ってくれるのが「環境(のどこかのコンポーネント)」です。
:::

さて、JavaScript はシングルスレッドで実行される[^シングルスレッド実行]はずでしたが、このように非同期 API を介して**同時に複数のことができる**のは、**環境が JavaScript エンジンだけではなく色々なものを機能として持っているからです**。そして**環境はシングルスレッドなどではありません**。

  [^シングルスレッド実行]: JavaScript の非同期処理において「シングルスレッドでの実行」がやたら強調される背景は他のプログラミング言語での非同期の仕組みとの比較から来ている可能性が高いです。

例えば、Chrome といったモダンなブラウザ環境を代表として考えると、ブラウザはそもそも様々なタスクを行っているので、シングルスレッドなどではなく、**マルチプロセスアーキテクチャ**であり、様々なタスクの責務を持つ分離したプロセスの中に複数のスレッドを持つ形になっています。

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
>(以下の書き起こしページから引用、太字は筆者強調)

https://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html

:::message
**補足** 

Philip Roberts 氏が言う "Runtime" とは Chrome ブラウザ環境の JS エンジンである V8 のことを言っています。Deno や Node は V8 を含んだ"環境"であり、色々な API を提供するので、ここではブラウザと等価なものとして考えてください。

それぞれ公式サイトでの文言。
- [Deno is a simple, modern and secure runtime for JavaScript and TypeScript that uses V8 and is built in Rust.](https://deno.land)
- [Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine.](https://nodejs.org/en/)

また、引用の "we shall APIs" はおそらく書き起こしの間違いで、実際の動画では "Web APIs" と言っています。
:::

ここまでくれば非同期処理の全体的な仕組みや目的がなんとなく分かると思います。

「**環境が並列的にバックグラウンドで作業している間もメインスレッドをブロッキングすることなく別の作業を続けられるようにすること**」が「非同期処理(というテーマ)」の大きな目的となります。正確には「**非同期 API の目的**」ですが、一般的にくくられてしまう大きなテーマである「非同期処理(というテーマ)」の目的はこれです。次のチャプターで解説しますが、非同期 API に付随して結果として非同期的に処理されるコードが非同期処理ですが、これ自体は単なる「結果」であって「目的」ではありません。

そして「**環境が提供する機能である API を介して時間のかかる処理を環境に委任し、それが完了したら JavaScript のメインスレッドにその処理結果となるデータと共に通知させてその作業に関連する何か別の作業をコールバックの形で行う**」というのが「非同期処理(というテーマ)」の全体的な仕組みとなります。

これを理解することで、非同期処理の学習にありがちな誤解を解くことができます。

非同期処理そのものは確かに並列処理ではありませんが、「**API を介して環境に委任した作業はバックグラウンドで並列に行われ、それが完了したら何かをメインスレッドで非同期的に行う**」というように、環境全体では「非同期 API」+「非同期処理」として「並列」と「非同期」の**両方が組み合わさって起きている**という仕組みを理解する必要があります。

:::message alert
実際には「並列(parallel)」とは言い切れないパターンがあったりしますが、**JavaScript (JS エンジン)の視点で考えれば別のところで同時に起きていること**を理解するには「並列」として便宜的に考えないと理解が進まないというジレンマがあります。厳密な定義の並列(parallel)を使って一般的に説明しようとすると使えなくなってしまうので、お茶を濁すしかないというのが実情な気がしています。

つまり、「同時に複数のことをできる」からと言って必ずしも複数スレッドを利用した「並列(parallel)」ではない訳です。『What the heck is the event loop anyway?』でも「**一度に複数のことができるのはブラウザがランタイム以上のものであるからで、ブラウザから提供される Web APIs は実質的にスレッドである**」という話がされていましたが、あくまで「実質的に」という注釈がつきます。Node などでは複数のスレッドをセットにしたスレッドプールを利用することがあるので実際「並列(parallel)」と言ってもよいものやポーリング際にまとめて有効期限の切れたタイマーを一括処理するなどがあるのでかなりややこしい話になっています。基本的に JS の文脈で語られる「並列」という言葉は信用せずに流す程度がいいかもしれません。

例えば、`Promise.all()` などの静的メソッドの話題で「並列化」という言葉をよく聞くと思いますが、その実体が何なのか意識されずに「並列化」という言葉が気軽に使われているケースが多いので、読み手側も意識しすぎてしまうと混乱することになります。これについては『[Promise の静的メソッドと並列化](17-epasync-static-method)』のチャプターで解説します。

この本ででてくる「並列的」という言葉は複数スレッドが絡む厳密な「並列(parallel)」ではなく、「**複数の作業が時間的に同時にされている**」ということを指しますので注意してください。厳密な並列(parallel)を指す場合には「並列(parallel)処理」と記載するようにします。

英語でもそうなのですが、専門用語と一般的通念としての言葉がまじって語られる場合がよくあります。例えば「Concurrently」という副詞は必ずしも「並行(concurrent)処理」を表すのではなく単に「同時」という意味に過ぎない場合などがよくあります。
:::

結果を取得できるまで時間のかかる作業などを「並列的に」バックグラウンドで行いたいから非同期処理という形でその完了を通知させるわけです。

もちろん環境が提供できる API には限りがありますし、自分で定義した時間のかかるような独自の処理を行いたい場合には普通の API ではどうにもなりません。しかし、Web worker API (これも Web API)を使うことでメインスレッドではなく別のスレッドに分離してそのような時間のかかる処理を自分自身で本当に並列(parallel)なものとして「並列(parllel)処理」を行うことができます。

https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers

Deno 環境でも同一名の API を使用できます。マニュアルには複数スレッドで独立させてプログラムを走らせることが可能と記載されています。

>Workers can be used to **run code on multiple threads**. Each instance of Worker is **run on a separate thread**, dedicated only to that worker.
>([Workers | Manual | Deno](https://deno.land/manual/runtime/workers#workers) より引用、太字は筆者強調)

# 「非同期処理」の内訳

だいたいの話は分かったと思いますので、解釈の話としてまとめておきます。

非同期 API の実態は環境がバックグラウンドで並列的に行う機能であり、その処理結果から取得できるデータをメインスレッド(コールスタック)に通知させます。

分かりやすい例では、通知させるのは `fetch(url)` から取得してきたデータ `Response` ですね。この `Response` のデータはそのままでは使えないので、色々処理を施すわけです。`Response` オブジェクトから使いたいテキストデータや JSON データを取得するとか。この**非同期 API の処理結果に対してなにかする作業**が実行タイミングのずれてしまういわゆる非同期処理です。下のように `then()` メソッドに書かれたコールバック形式の処理がソースコードの配置と実行タイミングがずれるのでこれは非同期処理であると断言できます。

```js
console.log("[1]");

// [非同期 API] の起動(githubのAPIからデザイン思想について取得)
fetch("https://api.github.com/zen")
  .then((response) => { // [非同期に実行されるコールバック関数] 
    // (取得したデータからtextを取得)
    return response.text();
  }) 
  .then((text) => { // [非同期に実行されるコールバック関数] 
    // (textをコンソール出力)
    console.log("[3]", text);
  }); 

console.log("[2]");

// コンソールへの出力は次のようになる
/* [1]
 * [2]
 * [3] It's not fully shipped until it's fast.
*/
```

`fetch()` API からは返り値として Promsie インスタンスが返ってきます。詳しくは『[コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async)』のチャプターで解説しますが、この Promise インスタンスに対して `then()` メソッドをつなげていく形式を **Promise chain** と呼びます。

Promise chain も非同期処理の範疇に含まれますが、このように「非同期処理」というテーマの範疇に色々なものが含まれていること自体が非同期処理の学習における混乱要因の１つだと考えています。実際に「非同期処理」を語る際には以下のように複数のカテゴリに分解できます。

- (1) `setTimeout()` や `fetch()` などの非同期 API
  👉 さらにタイプがありそれぞれで考え方や実行タイミングが異なる
- (2) `then()`、`catch()`、`finally()` メソッドや上記の非同期 API に登録して非同期的に実行されるコールバック関数(実際にはコールバック関数の実行コンテキスト)
- (3) `await` 式の前後の処理(実際には async 関数の実行コンテキスト)

(2) と (3) は近いのですが、実行コンテキストの考え方が異なるので厳密に同じであるとは言えません。そして、(1) は他の２つと比べて全く異なりますし、(1) の中でも種類があります。そういう訳で、「非同期処理」という単語でまとめて統一的に理解しようとすると**内部的な説明の辻褄が合わなくなって訳がわからなくなってしまう**わけです。Promise の話をしていたとおもっていたら全然違う非同期 API の話だったみたいなことになります(本当に全然違うので説明するためのロジックに矛盾が起きます)。「非同期処理」という１つの単語で理解するのはかなりキツイく、最初から上のように種類があるということを認識して分割して理解した方が良いです。

(1) の「非同期 API」は非同期処理かどうかというと、最初は**むしろ並列処理**としてみなした方が理解しやすいです。ただし、上で述べたように厳密な並列(parallel)処理であると信じてはいけません。あくまで非同期 API に「同時に複数のことができる」性質があり「並列的な作業」を行える程度のものであると考えてください。

:::message
厳密に並列ではないのに並列的なものであると理解した方がいいのは、入れない場合に比べて弊害が少ないと個人的に考えているからです。これは実際に自分が「並列」概念を入れないとむしろ混乱して理解が進まなかったためです。並列の厳密な定義は非同期処理についてある程度理解してからでも良いと思います。
:::

一方、(2) と (3) は厳密な並行(concurrent)処理となります。この辺りが本当にやっかいなことに複雑です。

具体的な説明をすると、非同期 API の起動自体はソースコードの配置に沿ってタイミングをずらさずに(同期的に)開始しますが、時間的にはバックグラウンドで処理を(並列的に)継続しつつ、メインスレッドでは別の JS コードの処理ができています。そして、非同期 API の結果から生じたデータに対して後で何かするというのが非同期処理です。ソースコードの配置に沿った順番とはずれたタイミングで(非同期に)処理されます。

```js
// 上から下に一行ずつ実行されるのが基本でそこからずれるのが非同期処理

cnsole.log("[1] 同期実行: ソースコードの配置では最初");

// 非同期 API の起動(タイマーの起動自体)は同期的(タイミングがずれない)だが、その後もバックグラウンドで並列的に「時間を測る処理」がすすむ
setTimeout(() => {
  // ソースコードの配置上の順番と処理順番がずれる
  console.log("[3] 非同期実行: これはコールバック関数");
  // 指定時間が経過したらこのコールバック関数[非同期処理]を実行するように伝える
}, 1000);

// API 処理が継続的に裏で実行されつつも、次のコードが実行できる
console.log("[2] 同期実行: ソースコードの配置では最後");

// コンソールへの出力は次のようになる
/* [1] 同期実行: ソースコードの配置では最初
 * [2] 同期実行: ソースコードの配置では最後
 * [3] 非同期実行: これはコールバック関数
*/
```

API を介した作業を環境が代わりに並列的に行っている間もメインスレッドで別の作業を続けられるようにするのが非同期処理の大きな目的(単体で見れば、非同期 API の目的といった方が正しい)でした。

「非同期処理」をよりマクロ的に分類するなら、次のようになります(上で挙げた１~３のカテゴリを更に抽象化した分類です)。

- (A) 非同期 API(環境の機能): `setTimeout()` や `fetch()` といったグローバルに利用できるメソッド 
  👉 **起動後に環境によって並列的にバックグラウンドで処理される**
- (B) 非同期のシンタックスで書かれた関数(ECMAScript の関数): 非同期 API の処理結果を使った後になにか処理する作業であり、非同期 API に渡す Callback や Promise chain、async/await で記述される
  👉 **イベントループの機構によって並行(concurrent)にメインスレッドで処理される**

一般的に「非同期処理」の名目で解説されるのは (B) の方です。(A) の話か (B) の話なのかでかなり違うという事実があるので気をつける必要があります(初学者はこれに気づけないケースが多いと思います)。(A) が存在していることに注意を払わないで (B) に注視してしまうことで、大きな勘違いと混乱を生むことになります。

しかも**現実的には (A) と (B) が単体で使われることがなく、両方組み合わされて使われるので非常にややこしいことになっています**。つまり、(B) 非同期のシンタックスで書かれた関数の中で (A) 非同期 API が利用されているような絡み合った状態になることが普通です。逆に (A) の引数に (B) というコールバック関数を場合もあります。例えば、`setTimeout(callback, delay)` の `callback` は (B) であり、`setTimeout()` 自体が (A) です。

つまり、両方が一体となって処理を実現していくわけです。

JavaScript はこのように ECMAScript と環境実装の API(実行環境の固有機能) を組み合わせたものとして考えます(`console.log()` も API ですし、JavaScript は API 無しでは大したことはできません)。

:::message alert
「非同期処理」という単語でひとまとめにして認識しようとするとややこしくなるので、この本では以後は「非同期 API (A) かそれ以外 (B)」という認識で非同期 API (環境の機能)なら明確に「非同期 API」であると呼称します。非同期 API ではないもの(Promise chain や async/await など)ならまとめて「非同期処理」として呼称するようにします(可能な限り詳細に呼称するようにします)。
:::

話を戻すと、非同期 API の処理は環境が並列的に行ってくれるので、非同期 API の処理結果を使ってコールバック関数の形でなにか処理したい場合には他の同期処理とはタイミングをずらす必要がありますね(本質的には (A) 非同期 API を使いたいがために (B) 非同期のシンタックスの関数を書く必要があるという話になります)。

以下のコードでは、`fetch()` API でデータ取得しながら `setTimeout()` API で指定した遅延時間 1000 ミリ秒を計測し、コンソールにもテキスト出力しています。**環境の機能である非同期 API(Non-blocking API)を使用しているから同時に複数のことができます**。

:::message
非同期 API は Node の文脈では **Non-blocking API** (ブロッキングしない API)と呼ばれることが多いですが、`fetch()` や `setTimeout()` などの非同期の Web API も Non-blocking API です。
:::

「**同時に複数のことをやることで効率的な処理がしたい**」という欲求を叶えるために非同期 API を使用し、後続の関連する処理を制御するために非同期処理のシンタックスである Callback や Promise chain, async/await を使用します。

```js
console.log("sync"); // 環境が提供する API(同期的に実行される)

// 環境が提供する非同期 API(並列的にデータ取得を行い、取得成功して安定化したらメインスレッドへ chain した then メソッドで登録しておいたコールバックを取得データとともに通知して実行する)
fetch("https://api.github.com/zen") // Promise インスタンスが返ってくる
  .then((response) => { // コールバック関数(非同期的に実行される)
    return response.text()
  }) 
  .then((text) => { // コールバック関数(非同期的に実行される)
    console.log(text)
  });

// 環境が提供する非同期 API(並列的にタイマーを走らせ、指定時間が経過したらメインスレッドへコールバックを通知して実行する)
setTimeout(() => { // コールバック関数(非同期的に実行される)
  console.log("async code");
}, 1000);

console.log("sync"); // 環境が提供する API(同期的に実行される)
```

注意点として、`fetch()` や `setTimeout()` は「非同期関数」と呼称されることがあります。これらは環境から提供される関数であり非同期 API であるため「非同期関数」と呼ぶことは間違いではないですが、誤解を招きやすい表現となっています。例えば、`async` キーワードを使って宣言された async function のことも「非同期関数」と呼ぶので非常に紛らわしいことになります。`setTimeout()` と async function を**同じ呼び方で同一視することは絶対にできない**ので分けて考えるべきです。

この本では「非同期関数」といったら `async` キーワードを使って宣言・定義された関数のことを指し、ECMAScript の範疇ではなく環境の機能ならば「非同期 API」として呼んでいるので注意してください。async function のこともなるべく「async 関数」と呼ぶようにします。

# 「非同期 API」の種類

「非同期処理」というのは複数の要素が絡み合っていることが分かった訳ですが、その要素の１つである「非同期 API」にも更に種類があります。上のコードであげたように `fetch()` と `setTimeout()` は非同期 API ですが、裏の立脚する仕組みが異なります。

`setTimeout()` はタスクを発行しますが、`fetch()` は Promise インスタンスを返すため(間接的に)マイクロタスクを発行します。

:::message alert
後述しますが、`setTimeout()` に対してより正確に対比できる API は `fetch()` ではなく `queueMicrotask()` です。`setTimeout()` はコールバックをタスクキューへと発行し、`queueMicrotask()` はコールバックをマイクロタスクキューへと発行します。

```js
console.log("[1] 🦖 sync");
setTimeout(() => { // 非同期的に実行されるコールバック関数
  console.log("[4] 🐵 async code");
});
queueMicrotask(() => { // 非同期的に実行されるコールバック関数
  console.log("[3] 🐶 async code");
});
console.log("[2] 🦖 sync");

/* コンソールへの出力は次のようになる
[1] 🦖 sync
[2] 🦖 sync
[3] 🐶 async code
[4] 🐵 async code
*/
```
:::

「非同期処理(というテーマ)」を実現するための機構として、タスクは古い方式で、マイクロタスクは新しい方式です。マイクロタスクは Promise 処理のために導入された新しい機構であり、`fetch()` API はこのマイクロタスクの仕組みに立脚した非同期 API となります。`fetch()` は処理結果となる値を入れ込んだ Promise インスタンスを返してきます。これは **Promise-based API** と呼ばれるモダンな非同期 API の仕組みです。

>Many modern Web APIs are promise-based, including WebRTC, Web Audio API, Media Capture and Streams, and many more.
>([How to use promises - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises#conclusion) より引用)

Mdn でも、非同期 API の理想は Promise インスタンスを返す関数(つまり、Promise-based API)であると示唆されています。

>理想的には、すべての非同期関数はプロミスを返すはずですが、残念ながら API の中にはいまだに古いやり方で成功/失敗用のコールバックを渡しているものがあります。顕著な例としては `setTimeout()` 関数があります。
>([プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90) より引用)

実際、Node よりも後発であるモダンなランタイム環境である Deno では、基本的にすべての非同期アクションは Promise インスタンスを返します。Promise-based API を基軸に開発されている訳です。

>All async actions in Deno return a promise. Thus Deno provides different APIs than Node.
>([Introduction | Deno Manual](https://deno.land/manual/introduction#comparison-to-nodejs) より引用)

さて、非同期 API についてまとめておきましょう。

Promise-based API は Promise インスタンスを返すため、Promise を介した逐次処理などに使われるコールバックはマイクロタスクとして発行されます。この API は Promise 関連の処理に使われる裏の仕組みであるマイクロタスクに立脚しているため、「**マイクロタスクベースの非同期 API**」とでも呼べる代物です。

コールバックをタスクとして発行するタイプである旧式の非同期 API は立脚する仕組みがタスクであるため、「**タスクベースの非同期 API**」と呼べますね。

ということで、非同期 API にはそれが立脚する仕組みから便宜的に次のように分けることができます。

- (1) タスクベースの非同期 API
  - コールバックをタスクとして発行する
  - 例: `setTimeout()` や `setInerval()` など
- (2) マイクロタスクベースの非同期 API
  - (A) コールバックをマイクロタスクとして発行する非同期 API
    - 例: `queueMicrotask()` や `MutationObserver()` など
  - (B) Promise-based API
    - Promise インスタンスを返し、そのインスタンスの中に API の処理結果(取得データなど)を入れておく
    - 例: `fetch()` や `Blob.text()` など

:::message alert
『[タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)』のチャプターで解説しますが、コールバック関数をそのままマイクロタスクとして発行する非同期 API として `queueMicrotask()` と `MutationObserver()` があるので注意してください。Promise-based API は Promise インスタンスを返しますが、マイクロタスクが利用されるのは返されるインスタンスに `.then(cb)` でチェーンして、履行状態に移行した後で発行されるコールバックに対してなので、マイクロタスクをそのまま直接発行するわけではないです。
:::

非同期処理の学習でのトラップとして、非同期 API にはこのように種類があることを認識する必要があります。「非同期処理」に分類があったのに「非同期 API」にも分類がある訳なので非常に複雑になっています。上の２つのタイプの非同期 API は立脚する裏の仕組みが異なるため、**それぞれ実行タイミングや考え方が異なります**。これを認識し、処理の順番を予測できるように必要なのが、**イベントループ**やタスクキュー・マイクロタスクキューの概念です。

# 「非同期」という概念

このチャプターの締めとして、改めて「非同期」という概念を確認しておきます。インターネット上のどこを探してもフワフワしている概念なのですが、「非同期(asynchronous)」の概念が MDN でどのように説明されているかを見ると次のように記載されています。

>The term asynchronous refers to two or more objects or events not existing or happening at the same time (**or multiple related things happening without waiting for the previous one to complete**).
>([Asynchronous - MDN Web Docs Glossary: Definitions of Web-related terms | MDN](https://developer.mozilla.org/en-US/docs/Glossary/Asynchronous) より引用、太字は筆者強調)

上の文章は MDN の日本語版と英語版で解釈がなぜか食い違っているように思えるのでオリジナルの英語版を持ってきています。太字以外を見ないようにしてください(共通部分以外の前にある文は信頼できないですし、混乱することになるので)。「**複数の関連する事象が前の事象の完了を待たずに起きる**」というのが個人的には唯一信頼できる定義だと考えています。

これによって「非同期 API」が同時に複数のことができる性質を持っているにも関わらずその呼称が「並列 API(Parallel API)」ではなく、「非同期 API(Asynchronous API)」になっている説明が付きます。

あるいは nodejs.dev での定義を見てみると次のように説明されています(この定義は話半分に聞いてください)。

>Asynchronous means that things can happen independently of the main program flow.
>([JavaScript Asynchronous Programming and Callbacks](https://nodejs.dev/learn/javascript-asynchronous-programming-and-callbacks) より引用)

なるほど、非同期(Asynchronous)とは「**メインのプログラムフローから独立して物事が起きうること**」とのことです。これも「複数の関連する処理が前の処理の完了を待たずに起きる」の言い換えに近いですね。

理解しやすく今までの JavaScript の文脈で具体的な情報を補うと、ある処理が単一スレッドとしてのメインスレッド(メインのプログラムフロー)の外、つまり JavaScript エンジンの外、つまり環境のどこかでバックグラウンドで実行されることが「非同期」であるということです。

上の説明はすこし拡大解釈ですが、「非同期(Asynchronous)」とはまさに「**非同期 API の性質そのもの**」あるいは「**非同期 API を使用することで起きる事象そのもの**」を指していたわけです。

「非同期処理」の学習における混乱の最大の原因が「非同期処理」という用語そのものにあります。

この本でも長らく勘違いしていたのですが、非同期処理のテーマで語られる Promise や async 関数の処理そのものは必ずしも「非同期(asynchronous)」の性質を発現しません。async 関数(非同期関数)という名前なのにです。それらはある特定の条件において「非同期」の性質が発揮されます。

async 関数については『[Promise chain から async 関数へ](14-epasync-chain-to-async-await)』のチャプターで詳しく解説しますが、async 関数というのは内部に await 式がある時に限って「非同期」となります(以下のコードでは `console.log()` の引数の番号が実行した際の出力の順番になります)。
```js
console.log("[1]");
// async 関数の即時実行(アロー関数での書き方)
(async () => {
  console.log("[2]");
  await 1;
  console.log("[4]");
})();
// await 式があれば async 関数は完了せずにその外の処理が行われるので「非同期」の現象が起きている
console.log("[3]");
```

:::details 即時実行関数の補足
`(async () => {})();` という上の書き方は即時実行関数の書き方になります。即時実行にはいくつか書き方があるので注意してください。

```js
// 無名関数の即時実行
(function() {
  // 処理内容
})();

// 非同期の無名関数の即時実行
(async function() {
  // 処理内容
})();

// アロー関数の即時実行
(() => {
  // 処理内容
})();

// 非同期のアロー関数の即時実行
(async () => {
  // 処理内容
})(); 
```

即時実行関数は普通に関数を定義してすぐ呼び出すのと大した差はないですが、コード量も減って分かりやすく記述しやすいのでこの書き方を多用しています。

```js
// アロー関数での async 関数の定義
const Fn = async () => {
  // 処理内容
};
Fn(); // すぐ呼び出し

// 上とやっていることと大して変わりない
(async () => {
  // 処理内容
})(); 
```

関数式やアロー関数については別途『[Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func#関数式とアロー関数の補足)』のチャプターで解説しています。
:::

次のように await 式が関数内部に無ければそのまま完了してから関数の次の処理が行われるので「同期」です。
```js
console.log("[1]");
(async () => {
  console.log("[2]");
  1;
  console.log("[3]");
})();
// await 式が無ければ async 関数が完了してからその外の処理が行われるのでこの場合は「同期」である
console.log("[4]");
```

Promise も後続の `then()` メソッドがある場合、つまり Promise chain となっている場合に限って「非同期」となります。
```js
console.log("[1]");
new Promise(resolve => {
  console.log("[2]");
  resolve();
}).then(() => console.log("[4]"));
// Promise chain の処理は完了せずにその外の処理が行われるので「非同期」の現象が起きている
console.log("[3]");  
```

単一の Promise インスタンスをそのまま生成するだけでは「非同期」にはなりません。
```js
console.log("[1]");
new Promise(resolve => {
  console.log("[2]");
  resolve();
  console.log("[3]")
});
// Promise の処理が完了してからその外の処理が行われるのでこの場合は「同期」である
console.log("[4]");  
```

「非同期(asynchronous)」の性質を常に発現しうるのは「非同期 API」ですが、その処理を単体で視た時に実際に「非同期」になっているかは分かりません。例えば、次のような短いスクリプトファイルであれば後続の処理が無いのでこの処理を見ても「非同期」であるとは言えません。

```js:これしか書かれていないスクリプトファイル
// githubからデザイン思想を取得してコンソールに出力するだけ
fetch("https://api.github.com/zen")
  .then(response => response.text());
  .then(text => console.log(text));
// コード配置と実行・完了のタイミングがズレていない(上から下に行われていく)
```

「非同期」の現象が起きていると言えるのは他のコードとの関係性があってのことです。次のように `fetch()` から始まる一連の Promise chain の後に何らかの処理が書かれていれば、これを実行した際には「非同期」であると言えます。

```js:これしか書かれていないスクリプトファイル
fetch("https://api.github.com/zen")
  .then(response => response.text());
  .then(text => console.log(text));
// 上の処理の完了を待たずにその外の処理が行われるので「非同期」の現象が起きている
console.log("上の処理の完了を待たずにコンソールに出力");
```

「非同期」という概念が掴みづらいのは、このように状況によってその現象が起きたり起きなかったりするためです。

「非同期」という性質を持った「処理」がそれ単体で存在する訳ではありません。他のコードとの関係性があって始めて成り立つ事象を指し示す概念であり、非同期 API についても「**その性質を発現しうる**」という解釈が妥当です。MDN の定義「複数の関連する事象が前の事象の完了を待たずに起きる」をベースにして考えるとこうなります。

次のチャプターで詳細に解説しますが、現実的には Callback や Promise chain、async/await は非同期 API の絡む処理を行う際に必要なシンタックスであり、状況に応じて「非同期」の性質を発現しますが、単体で利用することはほとんどなく、非同期 API を起点にした一連の後続処理の順序を制御するための方法として利用されます。

このチャプターの冒頭で「非同期処理」とは「実行のタイミングがずれてしまう処理」であるという旨を書きましたが、それは部分的な解釈に過ぎませんでした。本質的には「複数の関連する事象が前の事象の完了を待たずに起きる」が「非同期」の正しい解釈であり、非同期 API を使う際に書かざる負えない Promise chain などのシンタックスによって付随的にでてくる「実行タイミングがコードの配置とずれる処理」は「非同期」の一部であるという訳です。

```js:これしか書かれていないスクリプトファイル
fetch("https://api.github.com/zen") // 起動させて環境がバックグラウンドで行う API 処理[非同期 API]
  .then(response => response.text()); // 実行タイミングが配置とずれる[thenに登録されたコールバック関数]
  .then(text => console.log(text)); // 実行タイミングが配置とずれる[thenに登録されたコールバック関数]
// 上の処理の完了を待たずにその外の処理が行われるので「非同期」の現象が起きている
console.log("上の処理の完了を待たずにコンソールに出力");
```

そしてこの付随的にタイミングがずれる処理が実際に行われるための機構がイベントループです。

このような訳で、実は「非同期」の概念を最初から直接的に理解しようとすると、様々な要因で深みにハマってしまい理解するのが難しいです。色々な要素が関わってくる上に、状況に応じて「非同期」か「同期」か変わってくるので、正しく解説しようとしたり、正しく理解しようとするほど初学者にとって理解が困難になる概念になっています。矛盾しているようですが、直接的に正しく理解しない方が最初は理解が進みます。

「非同期 API」についても、これこそが「非同期」の性質を発現しうるものだった訳なのですが、これも最初は「並列(parallel)」であるとあえて誤解した方が理解が進みます。とは言え、誤解していること自体を知っておくことは重要です。

:::message
>非同期処理そのものは確かに並列処理ではありませんが、「**API を介して環境に委任した作業はバックグラウンドで並列に行われ、それが完了したら何かをメインスレッドで非同期的に行う**」というように、環境全体では「非同期 API」+「非同期処理」として「並列」と「非同期」の**両方が組み合わさって起きている**という仕組みを理解する必要があります。

と言いましたが、実際にはすべて「非同期」の概念にまつわる話であった訳です。最初の説明では内訳が複雑なため「非同期 API = 並列」かつ「`then()` のコールバックや `await` 式の前後 = 実行タイミングがコード配置とずれてしまう非同期処理」とみなしていました(これについては筆者の勘違いもありました😅)。

「非同期処理」という用語を今ある知識で完璧に説明するなら、非同期処理とは「**複数の関連する事象が前の事象の完了を待たずに起きる非同期という現象を引き起こす可能性のある API 処理やその実行に付随して起きる後続の関連する実行タイミングがコード配置とズレてしまう可能性のある処理のこと、あるいはそれらすべてをまとめて認識するための呼称**」であると言えます。「可能性がある」と言っているのは状況次第で同期になることもあるからです。

こんな定義だと紛らわしく扱いずらいので、上で述べたようにこの本では非同期 API (環境の機能)なら明確に「非同期 API」であることを強調して呼称し、それ以外の非同期にまつわる処理やシンタックス(Promise chain や async/await など)はまとめて「非同期処理」として呼称します(可能な場合はなるべく詳細に呼称するようにします)。

- (A) 非同期 API (環境の機能):
  👉「環境」によってバックグラウンドで「並列的に」処理されるため結果的に「同時に複数のことができて効率がよい」
- (B) 非同期処理 (Promise chain や async/await などの非同期に関するあらゆる処理やシンタックスまとめて呼称): 
  👉 イベントループの機構によってメインスレッドで「並列(concurrent)に」処理されるため実行や完了のタイミングがコード配置とズレる「ECMAScript」の関数に主にフォーカス
:::

「非同期」を直接的に理解できるようになるためには、上記のように非同期のシンタックスや環境にまつわる色々な知識が一旦必要となるので「ブロッキング」といった周辺の概念や対立する「同期」の概念から理解することで外形を掴んでいった方がよいです。

ということで、基本的な思考経路はシンプルに以下のものとなります。

>**ブロッキングするなら同期。同期でないなら非同期。実行タイミングがコード配置とズレてしまうものも非同期とみなす。**

とりあえずはこの考え方に乗ることで物事をシンプルにして理解できます。

