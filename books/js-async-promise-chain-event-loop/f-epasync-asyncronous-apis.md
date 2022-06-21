---
title: "非同期 API と環境"
aliases: [ch_非同期 API と環境]
---

# このチャプターについて
イベントループや Promise チェーンの話へ入る前に、非同期処理で大切な「API と環境」の話をしておきます。というのも、自分自身がこの話の大切さに気づくのに非常に時間がかかったからです。

そもそも、非同期処理の学習には多くのトラップがあり、様々な誤解をしがちです。

ですが、非同期処理そのものの目的「なぜ非同期処理をするのか」についてぶれてしまうと土台が崩れてしまいます。このチャプターでは API と環境の話を通して非同期処理を行う目的を考え、その全体的な仕組みを掴みます。

:::message
**非同期処理の学習では言葉に惑わされないことが重要**

「非同期」や「同期」といった単語レベルでもそうですが、言葉に惑わされる(混乱させられる)ケースが非常に多いと感じました。実際、非同期処理の学習では以下のような解説によく使用される文言に対して疑ったり、慎重に考える必要があります。

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
- ランタイム環境: Node, Deno など

API にも色々な種類と分け方がありますが、ブラウザによって提供されるものは MDN では **Web APIs** として呼ばれています。この名前はかなり混乱を生じさせるものです。実態としては **Browser APIs** と呼ぶべきで、ブラウザ環境によって提供される API のことを指している場合が多いです。大抵の場合は、Web APIs といったらブラウザ環境が提供する機能のことだと考えてください(非同期 API とか Web API とか名前から直感的に想起されるもものと実体がそぐわないために混乱するケースが多いです)。

https://developer.mozilla.org/ja/docs/Web/API

ランタイム環境で提供される API は便宜的に Runtime APIs として呼称します。

Web APIs も Runtime APIs にも同期型の API と非同期型の API が存在しています。例えば、`setTimeout()` が非同期の API です。Node, Deno, Chrome などの実行環境に関わらず同じ名前の API として `setTimeout()` が提供されており、使い勝手はほとんど同じですが、それらは環境が独自に定義していることに注意してください。

:::message
ちなみに Deno ランタイム環境で提供される Runtime APIs は大きく以下の２要素で成り立っています。

- 1. [Web Platform APIs](https://deno.land/manual/runtime/web_platform_apis): `fetch()` のような Web APIs と同じ名前、同じ使い勝手になるように提供されている API 群
- 2. Deno global: 非同期 I/O などに関する Deno 環境での完全独自定義の API 群(global の `Deno` ネームスペースに含まれる `Deno.readFile()` というように `Deno` から始まる名前の関数など)

参考: [The Runtime | Manual | Deno](https://deno.land/manual/runtime)
:::

ただし、最近になってランタイム環境間での API の互換性を高める目的で [WinterCG(Web-interoperable Runtimes Community Group)](https://wintercg.org)という団体が発足しました。サーバーランタイム(Deno / Node.js) とエッジランタイム(Cloudflare Workers / Deno)の環境間で Web Platform API (`fetch()` や `setTimeout()` や `queueMicrotask()` など)の互換性を高めようというのが目的です。開発者は今後、より互換性の高い JavaScript で開発ができるようになると期待できます。

>The ultimate goal of the group is to promote **runtimes supporting a comprehensive unified API surface** that JavaScript developers can rely on **regardless of the runtime they are using**: be it browsers, servers, embedded applications, or edge runtimes.
>
>The members of the group want to provide a space to better coordinate between browser vendors and other implementors on how Web Platform APIs can be best implemented and used outside of browsers.
>([What are we trying to do? | WinterCG](https://wintercg.org/faq#what-are-we-trying-to-do) より引用、太字は筆者強調)

# 「非同期処理」の目的と仕組み
環境と API についての予備知識を入れたので、このチャプターの本題である「非同期処理の目的」について考えてみましょう。

そもそも「非同期処理」ですが、わざわざ(**実行のタイミングをずらして**)非同期的に何かをする必要がなぜあるのでしょうか？

ここで、非同期 API である `fetch()` メソッドについて考えてみましょう。`fetch()` は引数に URL を指定して実行することでネットワーク接続をしてデータを取得できます。しかし、ネットワーク接続というのはリクエストを投げて応答があるまで時間がかかるものです。

ところで、JavaScript はシングルスレッド言語であり、プログラムは**シングルスレッドで実行**されます。シングルスレッドで実行されるとはつまり、**単一コールスタックで実行される**ということです(これについては別のチャプターで解説します)。この「シングルスレッド」ですが、ブラウザ環境の文脈で言えば JavaScript は(UI スレッドとも呼ばれる)メインスレッドで実行されます。

JavaScript は「シングルスレッドで実行される」わけですから、この"メインスレッド"とやらを長時間専有してしまうような処理があると、その間は何もできなくなってしまうわけです。実際に長時間メインスレッドを専有することを「**ブロッキング(blocking)**」と言います。

`fetch()` の話に戻りますが、ということはネットワーク接続という時間のかかる処理をメインスレッドで行ってしまったら、その最中にはクリックやテキスト選択といった操作は何もできませんね。また、取得したデータを使って何かをしたい場合もデータが取得できるまではそのデータを使った処理は何もできません。

ブラウザ環境ではレンダリング処理やユーザーインタラクションなどから発火されるイベントの処理もメインスレッド(= UI スレッド)で行われています。もし、`fetch()` による実際のデータの取得をメインスレッドでやってしまったら、それこそブロッキングが起きるはずで、ブロッキングが起きている間は、ユーザーがテキスト選択やボタンクリックといった操作も何もできない時間ができてしまいます(なぜなら、そのような**ユーザー操作によるイベントもメインスレッドで処理されるからです**)。

そういう訳で `fetch()` でのネットワーク接続など時間がかかる処理そのものは JavaScript エンジン(コールスタックを管理するコンポーネント)自体は直接関与しません。そのエンジンを埋め込んでいる環境が代わりに並列的(parallel)にバックグラウンドで行ってくれます。

データが取得できたらコールスタックにコールバック関数の形で**取得データと共に通知させて**、そのデータを使う処理をその時点で行えるようにします。これが非同期処理です。

非同期処理は並列処理とよく勘違いされるために、「**非同期処理は並列処理ではない**」という注意書きや、「**JavaScript はシングルスレッド**」ということが強調されがちです。

これらの命題は確かに真実ではありますが、**現実のすべてを語ってはいません**。

`fetch(url)` の他にも非同期 API として上で挙げた `setTimeout(callback, delay)` という指定時間が経過したら登録しておいたコールバック関数を実行するというものがあります。これは、第一引数に渡したコールバック関数が指定時間後に実行されるようにスケジューリングする訳ですから、**平行(concurrent)処理**であるとみなせます。そして、指定時間が経過するまでの間、**メインスレッドでは別のことができます**。ですが「時間を図る」という処理をしつつ別のことができるのはなぜでしょうか？

「時間を測る」ということは明らかに作業です。時間を図る作業がメインスレッドで行われていたら**メインスレッドはブロッキングされてその間は何もできません**。もしそうならタイマーの処理などほぼ意味がありませんね。現実では、時間を図りつつ他の JavaScript コードを処理しているわけで、**同時に複数のことをやっている**わけです。

というわけで、時間を測るという作業やインターネットを介したデータ取得などの行為は API を介して**環境が代わりにバックグラウンドで並列的(parallel)に行ってくれます**。つまり、 API の呼び出しは**環境へ作業を委任する(delegate)という行為**だった訳です。

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

さて、JavaScript はシングルスレッドで実行されるはずでしたが、このように非同期 API を介して**同時に複数のことができる**のは、**環境が JavaScript エンジンだけではなく色々なものを機能として持っているからです**。そして**環境はシングルスレッドなどではありません**。

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

「**環境が並列的にバックグラウンドで作業している間もメインスレッドをブロッキングすることなく別の作業を続けられるようにすること**」が "非同期処理" の大きな目的となります。正確には「**非同期 API の目的**」ですが、一般的にくくられてしまう大きなテーマである「非同期処理」の目的はこれです。

そして「**環境が提供する機能である API を介して時間のかかる処理を環境に委任し、それが完了したら JavaScript のメインスレッドにその処理結果となるデータと共に通知させてその作業に関連する何か別の作業をコールバックの形で行う**」というのが "非同期処理" の全体的な仕組みとなります。

これを理解することで、非同期処理の学習にありがちな誤解を解くことができます。

非同期処理そのものは確かに並列処理ではありませんが、「**API を介して環境に委任した作業はバックグラウンドで並列に行われ、それが完了したら何かをメインスレッドで非同期的に行う**」というように、環境全体では「非同期 API」+「非同期処理」として「並列」と「非同期」の**両方が組み合わさって起きている**という仕組みを理解する必要があります。

結果を取得できるまで時間のかかる並列的な作業などをバックグラウンドで行いたいから非同期処理という形でその完了を通知させるわけです。

もちろん環境が提供できる API には限りがありますし、自分で定義した時間のかかるような独自の処理を行いたい場合には普通の API ではどうにもなりません。しかし、Web worker API (これも Web API)を使うことでメインスレッドではなく別のスレッドに分離してそのような時間のかかる処理を自分自身で本当に並列的に行うことができます。

https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers

Deno 環境でも同一名の API を使用できます。マニュアルには複数スレッドで独立させてプログラムを走らせることが可能と記載されています。

>Workers can be used to **run code on multiple threads**. Each instance of Worker is **run on a separate thread**, dedicated only to that worker.
>([Workers | Manual | Deno](https://deno.land/manual/runtime/workers#workers) より引用、太字は筆者強調)

# 非同期 API は非同期処理か？

だいたいの話は分かったと思いますので、解釈の話としてまとめておきます。

非同期 API の実態は環境がバックグラウンドで並列的に行う機能であり、その処理結果から取得できるデータをメインスレッド(コールスタック)に通知させます。

分かりやすい例では、通知させるのは `fetch(url)` から取得してきたデータ `Response` ですね。この `Response` のデータはそのままでは使えないので、色々処理を施すわけです。`Response` オブジェクトから使いたいテキストデータや JSON データを取得するとか。この**非同期 API の処理結果に対してなにかする作業**がいわゆる**非同期処理**です。

```js
console.log("[1]");

// [非同期 API] の起動(githubのAPIからデザイン思想について取得)
fetch("https://api.github.com/zen")
  .then(response => response.text()) // [非同期処理] 取得したデータからtextを取得
  .then(text => console.log("[3]", text)); // [非同期処理] textをコンソール出力

console.log("[2]");

// コンソールへの出力は次のようになる
/* [1]
 * [2]
 * [3] It's not fully shipped until it's fast.
*/
```

非同期 API は非同期処理かどうかというと、**むしろ並列処理**です。非同期 API の起動自体はソースコードの配置に沿ってタイミングをずらさずに(同期的に)開始しますが、時間的にはバックグラウンドで処理を継続しつつ、メインスレッドでは別の JS コードの処理ができています。そして、非同期 API の結果から生じたデータに対して後で何かするというのが非同期処理です。ソースコードの配置に沿った順番とはずれたタイミングで(非同期に)処理されます。

```js
// 上から下に一行ずつ実行されるのが基本でそこからずれるのが非同期処理

cnsole.log("[1] 同期実行: ソースコードの配置では最初");

// 非同期 API の起動(タイマーの起動自体)は同期的(タイミングがずれない)だが、その後も並列的に「時間を測る処理」がすすむ
setTimeout(() => {
  // ソースコードの配置上の順番と処理順番がずれる
  console.log("[3] 非同期実行: これはコールバック関数");
  // 指定時間が経過したらこのコールバック関数[非同期処理]を実行するように伝える
}, 1000);

// API が継続的に裏で実行されつつも、次のコードが実行できる
console.log("[2] 同期実行: ソースコードの配置では最後");

// コンソールへの出力は次のようになる
/* [1] 同期実行: ソースコードの配置では最初
 * [2] 同期実行: ソースコードの配置では最後
 * [3] 非同期実行: これはコールバック関数
*/
```

API を介した作業を環境が代わりに並列的に行っている間もメインスレッドで別の作業を続けられるようにするのが非同期処理の大きな目的(単体で見れば、非同期 API の目的といった方が正しい)でした。

分かりやすく考えるなら、便宜的に次のように分類できます。

- 非同期 API(環境の機能): 環境が代行しバックグラウンドで処理する作業
- 非同期処理(ECMAScript の関数): 非同期 API の処理結果を使ってなにか処理する作業

JavaScript は上のように ECMAScript と環境実装の API(実行環境の固有機能) を組み合わせたものとして考えます(`console.log()` も API ですし、JavaScript は API 無しでは大したことはできません)。
非同期 API の処理は環境が並列的に行ってくれるので、非同期 API の処理結果を使ってコールバック関数の形でなにか処理したい場合には他の同期処理とはタイミングをずらす必要がありますね。

```js
console.log("sync"); // 環境が提供する API(同期的に実行される)

// 環境が提供する非同期 API(並列的にデータ取得を行い、安定化したらメインスレッドへコールバックを取得データとともに通知して実行する)
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

# 非同期 API の種類

後のチャプターでも解説しますが、非同期 API には実は種類があります。上のコードであげたように `fetch()` と `setTimeout()` は非同期 API ですが、裏の仕組みは異なります。

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
```
:::

非同期処理の方式として、タスクは古い方式で、マイクロタスクは新しい方式です。マイクロタスクは Promise 処理のために導入された新しい機構であり、`fetch()` API はこのマイクロタスクの仕組みに立脚した非同期 API となります。`fetch()` は処理結果となる値を入れ込んだ Promise インスタンスを返してきます。これは **Promise-based API** と呼ばれるモダンな非同期 API の仕組みです。

>Many modern Web APIs are promise-based, including WebRTC, Web Audio API, Media Capture and Streams, and many more.
>([How to use promises - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises#conclusion) より引用)

Mdn でも、非同期 API の理想は Promise インスタンスを返す関数(つまり、Promise-based API)であると示唆されています。

>理想的には、すべての非同期関数はプロミスを返すはずですが、残念ながら API の中にはいまだに古いやり方で成功/失敗用のコールバックを渡しているものがあります。顕著な例としては `setTimeout()` 関数があります。
>([プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90) より引用)

実際、Node よりも後発であるモダンなランタイム環境である Deno では、基本的にすべての非同期アクションは Promise インスタンスを返します。Promise-based API を基軸に開発されている訳です。

>All async actions in Deno return a promise. Thus Deno provides different APIs than Node.
>([Introduction | Deno Manual](https://deno.land/manual/introduction#comparison-to-nodejs) より引用)

さて、非同期 API についてまとめておきましょう。

Promise-based API は Promise インスタンスを返すため、Promise を介した逐次処理などに使われるコールバックはマイクロタスクとして発行されます。この API は Promise 関連の処理に使われる裏の仕組みであるマイクロタスクに立脚しているため、「マイクロタスクベースの API」とでも呼べる代物です。

コールバックをタスクとして発行するタイプである旧式の非同期 API は立脚する仕組みがタスクであるため、「タスクベースの非同期 API」と呼べますね。

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
『タスクキューとマイクロタスクキュー』のチャプターで解説しますが、コールバック関数をそのままマイクロタスクとして発行する非同期 API として `queueMicrotask()` と `MutationObserver()` があるので注意してください。Promise-based API は Promise インスタンスを返しますが、マイクロタスクが利用されるのは返されるインスタンスに `.then(cb)` でチェーンして、履行状態に移行した後で発行されるコールバックに対してなので、マイクロタスクをそのまま発行するわけではないです。
:::

非同期処理の学習でのトラップとして、非同期 API にはこのように種類があることを認識する必要があります。２つのタイプの非同期 API は立脚する裏の仕組みが異なるため、**それぞれ実行タイミングや考え方が異なります**。これを認識し、処理の順番を予測できるように必要なのが、イベントループの概念です。

