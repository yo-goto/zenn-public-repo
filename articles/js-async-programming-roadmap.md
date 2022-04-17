---
title: "JSの非同期処理を理解するために必要だった知識と学習ロードマップ"
emoji: "🦄"
type: "idea"
topics: [javascript, 非同期処理, ロードマップ]
published: true
date: 2022-04-08
url: "https://zenn.dev/estra/articles/js-async-programming-roadmap"
aliases: [記事_JSの非同期処理を理解するために必要だった知識と学習ロードマップ]
tags: [" #type/zenn #JavaScript/async  "]
---

## はじめに
JavaScript の非同期処理を学習してみて「ある程度自信を持って理解できたと言える」状態に到達したので、その感想と、まとめの学習ロードマップとその中でどのような知識が必要になるかを紹介したいと思います。

この記事自体は後から別の記事で参照するかもしれませんが、~~具体的な話の無い気軽な内容なので~~、流し読み程度に見てもらえるといいかもしれません(非同期処理の具体的な話も少し盛り込みました)。あとは、自分が実際に学習してきた道筋に基づいているのでショートカットとして参考にしてもらったり、使えるリソースなどの情報が共有できるかもしれません。

もしくは「JavaScript 初心者が非同期処理を理解できるようになるまで」というストーリーで１つのサンプルとして見ていただけるといいかもしれません。

具体的な非同期処理についての知見はいずれちょこちょこアウトプットとして書いていこうかなと思います。この記事自体に間違いや修正、更新があれば適宜修正していきます。

:::details ChangeLog
- 2022-04-13
  - 「追記: ECMAScript のみに注目しても理解できない」の項目を追加
  - Web APIs / Runtime APIs の項目が抜けていたので追加
  - await 式の補足を追加
- 2022-04-14
  - ランタイムの補足を追加
  - 「環境」と「非同期処理の仕組み」に関してのまとめを追記
  - Web Workers についての記述を追加
- 2022-04-17
  - 「お知らせ」の項目を追加
  - 「追記: 非同期処理機能の俯瞰」の項目を追加
  - 内容の微修正
:::

## お知らせ
非同期処理の学習ロードマップなどと銘打って記事を書いたからには、自分も具体的な非同期処理についての解説やアウトプットをすべきだと考えたので、主に Promise チェーンについての解説をするために Zenn の Book として「**イベントループとプロミスチェーンで学ぶJavaScriptの非同期処理**」を公開しました。

自分のアウトプットを兼ねており、内容としては記事と同質なので無料公開しています。興味があれば読んでみてください!

https://zenn.dev/estra/books/js-async-promise-chain-event-loop

## 非同期処理の感想
まず非同期処理について学んでみて、「かなり難しく、学習に時間がかかる」という印象を受けました。イメージとしては**学習前予想時の３倍**は難しいです。

全体としては、非同期処理の"制御の流れ"が掴みづらく、バックグラウンドで何が起こっているかを把握しないと自分で書こうとしたときにまったく予測ができません。非同期処理の流れ自体が掴みづらいので、同期処理と非同期処理が混じっているときは更に予測しずらくなります。加えて、非同期処理には種類があり、**その種類ごとに実行のタイミングが実は異なる**ので、それらが入り交じることで更に予測が難しくなっています。

そもそも、「同期」と「非同期」という言葉に騙されそうになります。

局所的には Promise の概念が掴みづらく、 Promise チェーンでは特にコールバック関数がキーになるのですが、アンチパターンを学ぶことでようやく使えるレベルの理解に到達できたと感じました。

また、async/await についても、そもそも Promise チェーンを理解していないと使い物にならず、**その名前から明らかに非同期処理の主役なのではないかと感じられる**一方で、実体は異なるので「これは騙されるな」という印象を受けました。async/await と Promise の関係が把握できていないと致命的に混乱することになります。非同期関数が Promise インスタンスを返すこと、await 式が Promise の評価値を返し、**非同期関数内の実行フローを分割すること**、同期処理と非同期処理を行ったりきたりすること、それと**逐次的に処理を行う**ことなどとの違いなど、ここらへんではトラップとなるポイントがいくつもあります。

「Promise とか await とかっては実は非同期処理ではないのでは？😵‍💫」というような疑問が途中湧き上がったりもしますが、個人的にはその疑問を解消するところがターニングポイントになるなと感じました。

基本的には、解説記事を読んで理解したつもりになっても、制御の流れをコードから想像してみたり、実際に手を動かさないと理解はかなり厳しいと感じました。ドキュメントだけで理解するのはかなり困難なので映像や可視化ツールを使うことで比較的つかみやすくなり、実際にローカル環境で実行してみることで理解でき、だんだんと制御の流れが推測できるようになってきます。

## 学習ロードマップ
非同期処理というカテゴリについては(そもそも JavaScript は)、多くの有用なリソースがインターネット上で公開されているので最大限それらを活用して学習をすすめていきます。

まずは非同期処理の前に外すことのできない必要な知識として以下の事柄を知っているか確認します。
- 式(Expression)
- 関数式
- コールバック関数
- 無名関数
- アロー関数と省略形
- 即時実行関数

このあたりの知識が他のドキュメントや解説を見る時に**暗黙的に必要**になってきます。特にコールバック関数とアロー関数は重要です。async function を使用する際には即時実行関数がよくでてきます。await を理解する際にも式の概念がキーとなるので重要です。非同期処理は JavaScript の中でも比較的高度な内容であり、こういった知識が前提とされた状態で話が進んでいきますので理解しておくのを推奨します。

不安がある場合には、azu 氏の "JavaScript Primer" で「式と文」と「関数」の項目を読んでおくのをおすすめします。

https://jsprimer.net/basic/statement-expression/

https://jsprimer.net/basic/function-declaration/

以上が理解できているなら、まず「非同期処理の基礎」を一から学ぶために "JavaScript Primer" の次の項目を読みすすめましょう(ただし、いきなりすべて理解しようとするのは困難なのである程度の理解度で最後まで進めていき、細かい点については後から参照していく形がよいと思います)。"JavaScript Primer" では、「JavaScript の非同期処理の歴史的な発展」が俯瞰でき、エラーファーストコールバック → Promise → async/await というように順を追って非同期処理の必要な知識が一通り学習できます。

https://jsprimer.net/basic/async/

https://jsprimer.net/use-case/ajaxapp/

:::message
ちなみに、自分が非同期処理を「本格的に」学習しはじめたのは JavaScript Primer をほぼ読破した後であり、上で挙げた前提となる知識や非同期処理の導入は JavaScript Primer を読んで既に完了済みという前提があるので注意してください。

これ以降の話は、そういった基礎については理解できたが(理解できたと思っていたが)、実際に非同期処理を書いたり、人のソースコードを読んだりしようとした際に「**書けない・読めない・分からない**」という状況に陥ったためにさらなる理解を求めて学習した内容となります。
:::

ドキュメントを読み終えて「完全に理解した」と思ってコードを実際に書こうとしても(非同期処理の制御が予測できず)かけないことに気づくので次に非同期処理のバックグラウンドでの動きを学習します。

まずは非同期処理の流れを理解するのに必要な用語知識を確認しておきます。
- concurrent(並行)/parallel(並列)
- asynchronous(非同期)/synchronous(同期)
- sequential(逐次的)
- blocking/non-blocking
- single-thread/multi-thread
- Node.js や V8 engine についての基礎知識

上記の単語については、ここまでくればある程度の解像度で理解できているはずですが、いろんな記事を見てもここらへんの言葉って割と定義があいまいなので、なるべく解像度をあげておくとよいと思います。多分 "synchronous" に振り回されるので、"sequential" と "blocking" を意識できるように注視しておくと良いと思います(自分は途中で「同期的」という概念が分からなくなり非常に混乱しました)。

blocking/non-blocking については Node.js 公式サイトの次のページを一読しておくと理解が進みます。

https://nodejs.org/ja/docs/guides/blocking-vs-non-blocking/

blocking の例として Node.js の同期 API (Synchronous API) についても知っておくとよいです。同期 API は Node.js の Event loop と後続の JavaScript の処理を、その API の処理が完了するまでブロックします。これらの API は `readFileSync` など名前の最後が `Sync` で終わります。

次に非同期処理の流れを予測できるようになるため必要な知識と道具を獲得します。
- Event Loop
- Call Stack
- Heap
- **Web APIs** / Runtime APIs
- Microtask と Microtask Queue
- Macrotask と Macrotask Queue

これらの用語は非同期処理の種類を認識して、バックグラウンドでの流れをある程度の解像度で理解するのに必要になってきます。非同期処理の流れ(実行順番)を把握・予測するためには必ず理解したほうがよいです。これらのアイテム無しでは進むことができません。

まず、Call Stack と Event Loop を理解するために JSConf EU での Philip Roberts 氏の講演動画「What the heck is the event loop anyway?」を見ることを強くオススメします。
@[youtube](8aGhZQkoFbQ)

この動画は色々な人がオススメしたり、多くの記事で引用されています。非常に分かりやすく説明されており、非同期処理の片方(Macrotask)を理解するのに役立ちます。2014 年の動画ですが、JSConf の動画は他にもいくつか見ましたが、この動画以上に分かりやすく説明しているものはありません。

:::message
Chrome などのブラウザ環境と Node.js や Deno の環境では JavaScript エンジンとして同じ V8 エンジンを採用してますが、Event Loop の実装はそれぞれ違います。
- [Blink](https://www.chromium.org/blink/) (Chrome): Rendering engine
- [Tokio](https://tokio.rs) (Deno) : an asynchronous Rust runtime 
- [libuv](https://libuv.org) (Node.js) : a multi-platform support library with a focus on asynchronous I/O

V8 エンジンが担当しているのは、Heap と Call Stack です。V8 のソースコードには setTimeout や DOM、HTTP request といった Web APIs は含まれていません。

上記の動画で解説されているのは、ブラウザ環境での Event Loop です。Promise チェーンや async/await といった言語としての非同期処理を理解するレベルでの抽象化なら、Node.js でも Deno でも同じように考えて大丈夫です(もちろん開発レベルで必要なら I/O とかタイマーの振る舞いは違う可能性があるので具体的に知る必要はあると思いますが)。ただし、Web APIs の部分はブラウザ特有のものなので、Node.js や Deno ではそれに対応するものとして Runtime APIs で置き換えて考えます。
:::

動画で概要を理解したら同氏が開発した JavaScript の可視化ツールである "Loupe" で実際に `setTimeout()` 関数のコールバック関数やクリックイベントなどがどのように動くか実験してみてください。

http://latentflip.com/loupe/

![js loupe image](/images/js-async/img_loupe-js-async.jpg)

Loupe では２種類ある非同期処理の種類の内の１つ(Macrotask: マクロタスク)しかサポートされていないので、次に Github 社のエンジニアである Andrew Dillon 氏が開発した "JavaScript Visualizer 9000" を使用して Promise 関連の非同期処理(Microtask: マイクロタスク)を理解するために使用します。

https://www.jsv9000.app

JavaScript Visualizer 9000 ではサンプルがいくつも用意されており、それらを理解するだけでも非常に有用です。二種類の非同期処理についてサポートされているので自分が書いてみた非同期処理を実際にこのアプリで実行してみて非同期処理の流れが予測どおりか確認していくことで非同期処理の流れが読めるようになってきます。ただし、async/await はサポートされていないのですそこは注意してください。

![js visualizer 9000](/images/js-async/img_js-visualizer-9000.jpg)

JavaScript Visualizer 9000 を使って、Macrotask と Micortask の組み合わせに慣れてください。ここでは重要なこととして、 ~~Macrotask Queue (Task Queue) は実は**キューではなくセット**であることを理解しておく必要があります。複数の `setTimeout()` 関数を色々な delay 時間を指定して実行してみれば理解できますが、あきらかにキューではないです。セットであるというのも、そもそも仕様として定義されています~~。(おそらく間違い)

https://html.spec.whatwg.org/multipage/webappapis.html#task-queue

>Task queues are sets, not queues, because step one of the event loop processing model grabs the first runnable task from the chosen queue, instead of dequeuing the first task.  
>(上記ページより引用)

追記: これは、おそらく「Task queues(という集合)はただのキューの集合ではなく、セットである」ということを意味していて、単一の "Task queue" は名前の通りキューであると考えられます。

とにかく、JavaScript Visualizer 9000 は非同期処理を理解するまで長い付き合いになったので非常にオススメです。これが無かったら非同期処理を理解できなかったかもしれません。

:::message alert
JavaScript Visualizer 9000 を使って可視化することによって、マイクロタスクやマクロタスクが Call stack と Event Loop でどう動くかを予測できるようになりますが、JavaScript Visualizer 9000 では Web APIs の要素が無いため、実はもう一歩踏み込む必要があります。

なぜ **Web APIs** の要素を省いたのかはわかりません。実はこれが省略されているために、Visualizer 9000 で複数の `setTimeout()` 関数を色々な delay 時間を指定して実行してマクロタスクの動きを見ると「キュー」と思えない挙動に見えてしまうことに気づきました。基本的な流れを理解するのには問題ないですが、注意してください。

Visualizer 9000 では簡単にコードが共有できるので実際に以下のリンクから実行して挙動を見てみてください↓
[manySetTimeout.js](https://www.jsv9000.app/?code=Ly8gbWFueVNldFRpbWVvdXQuanMKc2V0VGltZW91dChmdW5jdGlvbiBiKCkgewogIGNvbnNvbGUubG9nKCJiOiBhZnRlciA1MDBtcyIpOwp9LCA1MDApOwpzZXRUaW1lb3V0KGZ1bmN0aW9uIGMoKSB7CiAgY29uc29sZS5sb2coImM6IGFmdGVyIDBtcyIpOwp9LCAwKTsKc2V0VGltZW91dChmdW5jdGlvbiBhKCkgewogIGNvbnNvbGUubG9nKCJhOiBhZnRlciAxMDAwbXMiKTsKfSwgMTAwMCk7CnNldFRpbWVvdXQoZnVuY3Rpb24gZCgpIHsKICBjb25zb2xlLmxvZygiZDogYWZ0ZXIgMTAwbXMiKTsKfSwgMTAwKTsKCmZ1bmN0aW9uIGQoKSB7CiAgY29uc29sZS5sb2coImQ6IHN5bmMgZnVuY3Rpb24gZXhlY3V0ZWQgaW1tZWRpYXRlbHkiKTsKfQoKZCgpOwo%3D)
:::

さて、ここまでくれば非同期処理がかなり理解できているはずです。きっと「非同期処理、完全に理解した😼」状態ですね。
では、テストとしてやっすん氏の次の動画でクイズをといてみてください。

@[youtube](T-_0Pc5P12U)

いかかですか?
自分の場合は「俺、Promise とか async/await のこと全然わかってねえじゃん...😭」となりました。

動画をみてもらったら分かると思うのですが、Promise と `await` 式の関係が分かっていないと 1 番目と 2 番目のクイズが解けませんし、そして Promise チェーンが理解できていないと最後のクイズがとけません。なので、ここからは実際に Promise チェーンを構築することに注力して学習をすすめていきます。また、Promise まわりの細かい疑問を調べ尽くして解消していきます。

- Microtask と Macrotask の両方を念頭に Promise チェーンを構築できるようにする
- 同期処理と非同期処理を混在させた Promise チェーンを構築できるようにする
- Promise チェーンで値を次のチェーンへと渡し、エラーハンドリングが正しく行えるようなチェーン構築をできるようにする

ここで Async function と `await` のことも考えてしまうとかなり混乱するので Promise チェーンのことに注力した方がいいかもしれません。

この段階では azu 氏の "JavaScript Promise の本" のコラムなどが非常に有用でした。

https://azu.github.io/promises-book/#promise-is-always-async

また、mdn のドキュメントが色々な疑問を解消してくれたので主に次のページを含む非同期処理に関連するページを読んでいくのがいいと思います。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises

上記の mdn の解説でも Promise に関してのアンチパターンが紹介されていますが、下記のページにおいていくつもの Promise アンチパターンが記載されているので１つずつ理解していきます。

https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html

アンチパターンとその理由を理解することによって Promise の曖昧だったところが非常にクリアになっていくのが分かります。

さて、ここまで来れば基礎的な Promise チェーンがある程度構築できるようになったと思います。自分もこの時点で非同期 API などの使い方もある程度わかるようになりました。

ここから本格的に async/await を理解するための学習が始まります。特に `await` がやっかいです。非同期関数の内部での await によって「これは同期処理なのか、非同期処理なのか」がよくわからなくなってきます。JavaScript Visualizer 9000 では async/await が使えず可視化できないため、手元でコードを書いて実行していくことをメインにして理解していきます。

自分の場合は、いままでやってきた Promies の考え方と噛み合わなかったり、Promise との組み合わせ方を理解するまでに時間がかかりました。これは Promise が理解しきれていなかったことを意味しています。**複数の `await` 処理の入った複数の非同期関数の処理を Promise チェーンで実現できるようになることが大切です**。Promise チェーンなら JavaScript Visualizer 9000 で可視化できますので。

また、最初は async/await によって Promise がいらなくなってしまうのかとも思いましたが、そんなことは無かったです。これについては、以下の複数の Promise を扱うことのできる静的メソッドを学習することで単純な async/await でやるよりも**効率的に目的をはたせる**ことを理解できれば体感できるはずです。

- [Promise.all()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
- [Promise.race()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/race)
- [Promise.allSettled()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) 
- [Promise.any()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/any)

mdn で各メソッドの解説や次の記事を読むのをオススメします。それぞれの使い用途や違いなどがあきらかになると async/await でできること、単純な `await` 式にのみ注力してしまうとやりづらいことが逆に分かってくるはずです。

https://okapies.hateblo.jp/entry/2020/12/13/154311

更にこれによって async/await が Promise を代替するのではなく、Promise ベースの非同期処理の利便性を一部向上させる Promise というシステム自体に基づいた拡張的な機能であり、**Promise と一緒に使っていくもの**であるということが認識できると思います。

そもそも、`await` 式は「右辺の Promise インスタンスが Settled 状態(Fullfilled または Rejected 状態)になるまで非同期処理の完了を待ち、Promise インスタンスの状態が変化したらその Promise インスタンスの**評価結果を値として返す**」ものなので、Promise を扱っていると認識してしないとやっすん氏の動画で紹介されたミス(マクロタスクを作成する `setTimeout` 関数を Promise でラップすることなくそのまま `await` してしまうなど)を犯してしまいます。

:::message
`await` 式では非同期関数の内部で非同期処理の完了を待ちますが、その"待っている"というのは実際にすべての処理が止まっているわけではなく、非同期関数の呼び出し元に一旦制御が戻って別の処理を再開し、`await` で待っていた Promise インスタンスが解決した後に再び非同期関数に戻って `await` 式の後にある次の処理を再開するということを意味しています。

非同期関数の中の処理は 1 つ 1 つ処理が終わってから順番に処理が進むので、「逐次的(順次的)な処理」がおこなわれていますが、`await` 式に出会うたびに、一度制御を呼び出し元に戻して別の同期処理などを実行しつつ、`await` で待っていた Promise が解決されたらコールスタックの状況を見て再び制御が非同期関数に戻るので、実際に制御の流れは「同期的」ではありません。これは Promise チェーンでも同じ状況となります。`then()` のコールバック関数はマイクロタスクキューへと一旦送られ、制御は呼び出し元に戻ります。乱暴な言い方ですが、async/await は本質的には Promise チェーンの書き方を変えているだけです。

もし「同期的」に処理が進んでいるなら、非同期関数の内部で `await` 式によってメインスレッドが専有されることで「ブロッキング」が発生し、その間は一切の処理が進まないことになります。もちろん、そんなことは起きていません。そんなことが起きるなら、そもそも「非同期処理」を行うメリットなんてないですよね。

Prmoise チェーンや `await` 式を利用している Async function の制御が「同期的」であると混同してしまうのは、 Promise チェーンや非同期関数内の処理のみについて考えれば確かにステップごとに「逐次的(順次的)」に処理が進んでいるからです。しかし、**そのステップの間では別の処理が進んでいます**。

「同期」や「非同期」という言葉に振り回されることによって、これを理解できるようになるためにかなり時間がかかりました。"blocking(ブロッキング)" と "sequential(逐次的)" という言葉に注視しておくと良いと言ったのはこのためです。

「Promise とか await とかっては実は非同期処理ではないのでは？😵‍💫」という冒頭の疑問はこれで解消できるはずです。

async/await では無いですが２つの Promise チェーンを同時に実行するのを可視化してみると制御が行ったり来たりしつつもステップごとに実行している様が理解できると思います。Visualizer で可視化してみたので確認してみてください。
[doublePromiseChain.js - JS Visualizer 9000](https://www.jsv9000.app/?code=Ly8gZG91YmxlUHJvbWlzZUNoYWluLmpzCgovLyB0aGluayBhd2FpdCB3aXRoIHByb21pc2VzCi8vIHJlbW92ZSBuZXN0aW5nLCBhbHdheSBlbmRzIHdpdGggY2F0Y2goKSBtZXRob2QKZnVuY3Rpb24gcmV0dXJuUHJvbWlzZUZuQSgpIHsKICBjb25zb2xlLmxvZygiLS0%2BIEVudGVyIHJldHVyblByb21pc2VGbkEiKTsKICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDEwMCkKICAgIC50aGVuKGZ1bmN0aW9uIGExKGRhdGEpIHsKICAgICAgY29uc29sZS5sb2coIi0tPiBFbnRlciB0aGVuIGNhbGxiYWNrOiBhMSIsIGRhdGEpOwogICAgICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDEwMSk7CiAgICB9KQogICAgLnRoZW4oZnVuY3Rpb24gYjEoZGF0YSkgewogICAgICBjb25zb2xlLmxvZygiLS0%2BIEVudGVyIHRoZW4gY2FsbGJhY2s6IGIxIiwgZGF0YSk7CiAgICAgIHJldHVybiBQcm9taXNlLnJlc29sdmUoMTAyKTsKICAgIH0pCiAgICAudGhlbihmdW5jdGlvbiBjMShkYXRhKSB7CiAgICAgIGNvbnNvbGUubG9nKCItLT4gRW50ZXIgdGhlbiBjYWxsYmFjazogYzEiLCBkYXRhKTsKICAgICAgcmV0dXJuIFByb21pc2UucmVzb2x2ZSgxMDMpOwogICAgfSkKICAgIC50aGVuKGZ1bmN0aW9uIGQxKGRhdGEpIHsKICAgICAgY29uc29sZS5sb2coIi0tPiBFbnRlciB0aGVuIGNhbGxiYWNrOiBkMSIsIGRhdGEpOwogICAgICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDEwNCk7CiAgICB9KQogICAgLmNhdGNoKChlKSA9PiBjb25zb2xlLmVycm9yKGUpKTsKfQoKZnVuY3Rpb24gcmV0dXJuUHJvbWlzZUZuQigpIHsKICBjb25zb2xlLmxvZygiLS0%2BIEVudGVyIHJldHVyblByb21pc2VGbkIiKTsKICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDIwMCkKICAgIC50aGVuKGZ1bmN0aW9uIGEyKGRhdGEpIHsKICAgICAgY29uc29sZS5sb2coIi0tPiBFbnRlciB0aGVuIGNhbGxiYWNrOiBhMiIsIGRhdGEpOwogICAgICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDIwMSk7CiAgICB9KQogICAgLnRoZW4oZnVuY3Rpb24gYjIoZGF0YSkgewogICAgICBjb25zb2xlLmxvZygiLS0%2BIEVudGVyIHRoZW4gY2FsbGJhY2s6IGIyIiwgZGF0YSk7CiAgICAgIHJldHVybiBQcm9taXNlLnJlc29sdmUoMjAyKTsKICAgIH0pCiAgICAudGhlbihmdW5jdGlvbiBjMihkYXRhKSB7CiAgICAgIGNvbnNvbGUubG9nKCItLT4gRW50ZXIgdGhlbiBjYWxsYmFjazogYzIiLCBkYXRhKTsKICAgICAgcmV0dXJuIFByb21pc2UucmVzb2x2ZSgyMDMpOwogICAgfSkKICAgIC50aGVuKGZ1bmN0aW9uIGQyKGRhdGEpIHsKICAgICAgY29uc29sZS5sb2coIi0tPiBFbnRlciB0aGVuIGNhbGxiYWNrOiBkMiIsIGRhdGEpOwogICAgICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDIwNCk7CiAgICB9KQogICAgLmNhdGNoKGUgPT4gY29uc29sZS5lcnJvcihlKSk7Cn0KCnJldHVyblByb21pc2VGbkEoKQogIC50aGVuKChkYXRhKSA9PiB7CiAgICBjb25zb2xlLmxvZygiQWZ0ZXIgZXhlY3V0aW5nIEZuQSIsIGRhdGEpOwogIH0pOwpyZXR1cm5Qcm9taXNlRm5CKCkKICAudGhlbigoZGF0YSkgPT4gewogICAgY29uc29sZS5sb2coIkFmdGVyIGV4ZWN1dGluZyBGbkEiLCBkYXRhKTsKICB9KTsKCgo%3D)
:::

"JavaScript Primer" に戻ってくると、「Promise と async/await は一緒に使っていくものである」ということを再発見できます。

https://jsprimer.net/basic/async/#relationship-promise-async-function

>このようにAsync Functionやawait式は既存のPromise APIと組み合わせて利用できます。 Async Functionも内部的にPromiseの仕組みを利用しているため、両者は対立関係ではなく共存関係になります。  
>(上記ページより引用)

Promise の静的メソッドと `await` 式を組み合わせる例として、複数のリソース非同期的に取得する際に Async function 内で `await promise1; await promise2; await promise3;` のように１つずつ取得が完了するのを待つのではなく、`await Promise.allSettled([promise1, promise2, promise3]);` として複数の非同期処理を並行して走らせることでリソースの取得時間を削減できるなどが挙げられます。

つまり、"async/await" ではなく、"**Promise with async/await**" 的なノリであることを理解できます。

また、モダンな JavaScript/TypeScript ランタイムである Deno では Node.js と比較したときの特徴として「Deno での非同期アクションはすべて Promise を返す」という点をあげています。

>All async actions in Deno return a promise. Thus Deno provides different APIs than Node.  
>([Introduction - Deno Manual](https://deno.land/manual#comparison-to-nodejs) より引用)

"Async/await-based" や "Async/await-first" ではなく、"Promise-based "や "Promise-first" などの言葉を聞いたことがあると思います。その理由は "Promise" が常に非同期処理の主役だからです。

それが理解できたら、今度こそ async/await の強みを理解する段階になると思います。

https://developers.google.com/web/fundamentals/primers/async-functions#%E4%BE%8B_%E3%83%AC%E3%82%B9%E3%83%9D%E3%83%B3%E3%82%B9%E3%81%AE%E3%82%B9%E3%83%88%E3%83%AA%E3%83%BC%E3%83%9F%E3%83%B3%E3%82%B0

非同期処理のループなどを構築する際に、Promise チェーンでは複雑になってしまうコード("スマート"な書き方のコード)を、Async function 内では `while` ループや `for` ループといった"シンプルで読みやすい平凡な書き方"へと持ち込めるようになります。

逆に、コールバック関数に Async function を使う時は注意する必要があったり、どのタイミングでどれを await させるかなどの難しさも分かってきますが。

それが出来たら、仕上げに "Top-level await" を学びます。Top-level await で `fetch` などの非同期 API が簡単に使えるからと言って、いきなり Top-level await を学んでしまった場合は **通常の async/await における「同期と非同期」が分からなくなってしまい**、非常に混乱することが予測されるので、この最後の段階で学ぶことをオススメします。

https://qiita.com/uhyo/items/0e2e9eaa30ec2ff05260

さて、ここまでくれば、「非同期処理の基礎が身についた」といえる状態になったのではないでしょうか。もちろん、非同期処理のすべてを学びきった訳ではないので、今後も学習は続きます(非同期イテレータ `for await...of` やジェネレーターなどはまだ自分もわかりません)。

以上が自分なりに「ある程度、非同期処理が理解できた言える状態」まで到達するのに辿ってきたロードマップです。各項目の間で時間が空いたりして結構無駄が多かったかもしれませんが、**非同期処理への納得感**を達成するにはこれぐらいは必要だったかなと思います。

実際は紹介していないもっと多くのリソースからも学習しているので自分の中での最短はこんな感じのイメージになりました。次の記事からは学びで得た具体的な知見をいくつか絞って書いていこうかなと思います。記事を書いている途中で加速的に理解が進んだり、理解するために欠けていたピースがハマるので非常にオススメです。

## 追記: ECMAScriptのみに注目しても理解できない
実は、非同期処理のための「ECMAScript」の言語機能だけを見ていても、JavaScript における非同期処理の仕組みは理解できません。

:::message
実はこの記事を書いてから違和感があり、再度調べ直した結果として理解できました😇
間違いがあれば指摘していただけると助かります。
:::

"JavaScript" の境界線がどこまでなのかははっきりしませんが、JavaScript は常に実行する環境があり、その**環境によって利用できる機能が異なる**ので、ECMAScript(JavaScript の言語コア)における非同期処理の機能を考えるよりも、「環境と結びついたものとしての JavaScript の非同期処理」、ひいては「環境そのもの」のことを考えないと非同期処理を理解できません。

というのも、`setTimeout()` のタイマー処理や `fetch()` のネットワーク接続をしてデータ取得するという「根本的な処理」を実際に「マルチスレッドの並列(parallel)処理で」行ってくれているのは次のものだからです。

- Web APIs を提供するブラウザ環境 (Chrome の場合)
- Runtime APIs を提供するランタイム環境 (Node.js や Deno の場合)

https://www.freecodecamp.org/news/async-await-javascript-tutorial/

>It turns out that it is **the environment** that takes on the work, and the way to get the environment to do that work, is to use functionality that belongs to the environment. For example fetch or setTimeout in the browser environment.
>(上記ページより引用)

従って、ECMAScript の言語としての「Promise や async/await キーワード」や「シングルスレッドと並行(concurrent)処理」を考えていても現実に起こる「非同期処理の仕組み」についての謎が永遠に解けません。

例えば、`fetch()` メソッドは ECMAScript の一部ではなく、Web APIs(Web Platform APIs) の一部である Fetch API のメソッドであり、実際にその機能を提供するのはブラウザ環境やランタイム環境です。

Chrome といったモダンなブラウザ環境を代表として考えると、ブラウザはそもそも様々なタスクを行っているので、"シングルスレッド"などではなく、マルチプロセスアーキテクチャであり、様々なタスクの責務を持つ分離したプロセスの中に複数のスレッドを持つ形になっています。

https://developer.chrome.com/blog/inside-browser-part2/

Chrome では以下のような複数のプロセスが作成されます。
- Browser process
- Renderer process
- GPU process
- Plugin process
- Extensions process
- Utility process

`setTimeout(cb, delay)` の例では、指定した時間後にコールバックを Macrotask queue へと追加し、Event Loop によって Call stack へと運んでメインスレッドで実行しているので、これを「シングルスレッド」で実行されていると解釈することは可能ですが、「現実のタイマー処理」はメインスレッドで行われているわけではないです。もしタイマー処理がメインスレッドで行われているならそれこそ**ブロッキングが発生するはず**ですから。
`setTimeout()` メソッドはブラウザ環境やランタイム環境が提供する API であり、その「コールバックをキューに追加するまでの時間を図る」という処理自体は環境の別スレッドで行わているはずです。

コールバック自体はメインスレッドで処理されるが、コールバックをキューに追加するためのタイマー処理は別のスレッドで行われており、「非同期処理=非同期的なタイミングで実行される処理=コールバック」と考えるなら、コールバック関数の処理という非同期処理そのものは並行(concurrent)に処理されていますが、非同期処理のタイミングを図っているタイマーの処理は環境の別スレッドであり、並列(parallel)に実行していることになります。

また、`fetch(url)` もネットワークの接続をしてデータを取得しているのに**メインスレッドをブロッキングしないでバックグラウンドで行うことができる**のは、「現実のデータ取得処理」がメインスレッドで行われていないからです。`fetch(url)` そのものの処理が環境の別スレッドで並列(parallel)に行われてる一方で、`fetch(url).then(cb)` の `cb` コールバックについて「非同期処理=非同期的なタイミングで実行される処理=コールバック」として考えるなら、たしかにコールバック関数の処理という非同期処理そのものは並行(concurrent)に処理されています。

実際、JavaScript の非同期処理でよく語られるシングルスレッドは現実のブラウザ環境(Chrome)では「Renderer Process の Main thread」のことであり、`fetch()` メソッドのネットワーク接続やリクエストなどの処理を行っているのは「Browser process の Network thread」です。

`await` 式の前後や Promise チェーンのコールバック自体は確かに「非同期処理=非同期的なタイミングで実行される処理」であり、シングルスレッドで並行(concurrent)に処理されていますが、それを Micortask queue や Macrotask queue へと追加するための処理自体は並列(parallel)でマルチスレッドで起きています。

「並列(parallel)処理と並行(concurrent)処理が同時に起きている」、「シングルスレッドであり、マルチスレッドである」という文字にすると頭が混乱するような話ですが、**対立する概念が同居して「非同期処理」の仕組みを実現している**と考えないとつじつまが合わないので、こうなります。

この結論に至った経緯として、次の文献がメインとして参考になりました。

https://www.telerik.com/blogs/angular-basics-introduction-processes-threads-web-ui-developers

そして Event Loop と Call Stack を理解するための動画として紹介した「What the heck is the event loop anyway?」では「**一度に複数のことができるのはブラウザがランタイム以上のものであるからで、ブラウザから提供される Web APIs は実質的にスレッドである**」ということが実は語られていました。この動画では「ブラウザ環境」のことについても語られているので、Promise チェーンなどが理解できた後も何度見返してもよいと思います。

>Right, so I've been kind of partially lying do you and telling you that JavaScript can only do one thing at one time. That's true the JavaScript Runtime can only do one thing at one time. It can't make an AJAX request while you're doing other code. It can't do a setTimeout while you're doing another code. **The reason we can do things concurrently is that the browser is more than just the Runtime**. So, remember this diagram, **the JavaScript Runtime can do one thing at a time, but the browser gives us these other things, gives us these we shall APIs, these are effectively threads**, you can just make calls to, and those pieces of the browser are aware of this concurrency kicks in.
>(以下の書き起こしページから引用)

https://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html

:::message
補足: Philip Roberts 氏が言う "Runtime" とは Chrome ブラウザ環境の JS エンジンである V8 のことを言っています。Deno や Node は V8 を含んだ"環境"であり、色々な API を提供するので、ここではブラウザと等価なものとして考えてください。

それぞれ公式サイトでの文言。
- [Deno is a simple, modern and secure runtime for JavaScript and TypeScript that uses V8 and is built in Rust.](https://deno.land)
- [Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine.](https://nodejs.org/en/)

また、引用の "we shall APIs" はおそらく書き起こしの間違いで、実際の動画では "Web APIs" と言っています。
:::

↓ 書き起こしが実際の言葉と若干違っているので動画で確認すると 11:48 のところです。
[イベントループとは一体何ですか？ | Philip Roberts | JSConf EU](https://youtu.be/8aGhZQkoFbQ?t=708)

最後にもう一度流れをまとめますと、ブラウザ環境において `setTimeout()` と `fetch()` は **web API** です(ランタイム環境では Runtime API)。ECMAScript 言語の一部ではありません。両方とも自分で定義した関数を実行しているのではなく API の呼び出しを行っています。API を介して「時間を図る」、「リソースを取得する」といったタスクが「環境」へ委任(Delegate)され、そのタスクを「環境そのもの」がバックグラウンドで並列(parallel)に行います。「環境に委任されたタスクが完了したら何かをメインスレッドで行う」という**コールバック関数の形**で登録しておいた別のタスク(=非同期処理)は、委任されたタスクを終わらせた時点で環境から Macrotask queue や Microtask queue へと送られ、Event Loop によって Call stack へと運ばれた時点で JavaScript のコードとして実行されます。これが JavaScript における非同期処理の仕組みです。

結局の所、非同期処理の仕組みを理解するには「Promise や async/await といった言語機能の概念とその使い方」だけでなく「**API を提供する環境**」のことを知ることが必要だった、という訳です。

本来時間のかかるインターネットを介したデータ取得(`fetch()`)や I/O (`Deno.readFile()` や `fsPromise.readFile()`)などの処理は基本的に環境が API として用意してくれています。もちろんすべての API が時間のかかる処理ではなく、例えば、`URL()` なども Web API なので環境の提供する機能ですが、通信や I/O に比べてすぐに処理が終了するので、非同期ではなく同期的なものとなっています。

:::message
"blocking" の用語のところで紹介しましたが、Node や Deno の環境では時間のかかる処理の完了を待って、あえてメインスレッドの Event Loop をブロッキングする同期 API (Synchronous APIs) を提供しています。Node なら `fs.readFileSync()`、Deno なら `Deno.readFileSync()` などが該当します。用途としては、「スニペット程度のコードだからわざわざ非同期処理にするまでもない」というときや、他の何らかの理由であえて非同期処理にしたくない状況などで使用したりします。

>The synchronous APIs perform all operations synchronously, **blocking the event loop until the operation completes or fails**.
> ([File system | Node.js v17.9.0 Documentation](https://nodejs.org/api/fs.html#synchronous-api) より引用)
:::

API を介して時間のかかる処理は環境へ委任されますが、もし環境が API として用意していないような時間がかかるタスク(CPU-intensitve でメインスレッドを長時間専有してしまうような処理)を自分で定義した同期関数などで行う場合は、Web Workers でメインスレッドから分離した別スレッドを作成して、そのスレッドで並列(parallel)に処理を走らせることで**メインスレッドの Event Loop をブロッキングせずに済ませる**ことができます。もちろんこれも Web API です。

https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers

## 追記: 非同期処理機能の俯瞰
ECMAScript に取り入れられた非同期処理に関するものとして async/await の前に **Generator** という機能がありました。現時点で「非同期処理」の説明に Generator が使われていることはほとんど見かけませんが、async/await の機能の前にこれが導入されていたということを認識しておくといいかもしれません。

ここまで来て、色々な非同期処理の知識を得たと思いますが、ECMAScript の「非同期処理」の俯瞰を行うために Electron のコアエンジニアである Shelley Vohr 氏が JSConf EU で行った講演動画である「Asynchrony: Under the Hood」を見ておくといいと思います。

@[youtube](SrNQS8J67zc)

Philip Roberts 氏の動画よりも新しいので、ECMAScript の機能として新しく追加された Promise のためのキューである Microtask queue が入った状態の Event loop の話をしています。ですが、最初にこれを見ても多分理解できないと思うので、やはり「What the heck is the event loop anyway?」から視聴するのをオススメします。

以下の項目に触れているため、「非同期処理」の全体を俯瞰できます。

- Callback hell
- Promise chain
- Generator
- async/await

最後には、Callback hell → Promise chain → async/await の変形がみれるので async/await と Promise チェーンの書き換えが分からない人は見るといいかもしれません。

とは言っても非同期処理機能の歴史的な発展については azu 氏の JavaScript Primer でもかなり分かりやすく解説されているので、こちらを理解した上で視聴するのをオススメします。

https://jsprimer.net/basic/async/#sync-processing

## 参考文献とツールのまとめ

非同期処理を理解するのに必要不可欠な道具。
- [Loupe](http://latentflip.com/loupe/)
- [JS Visualizer 9000](https://www.jsv9000.app)
- すぐに JS を実行できるランタイム環境(Deno or Node)

非同期処理の基礎を理解するために必要な基本リソース。
- [JavaScript Primer - 迷わないための入門書](https://jsprimer.net/)
- [What the heck is the event loop anyway? | Philip Roberts | JSConf EU - YouTube](https://www.youtube.com/watch?v=8aGhZQkoFbQ&t=804s)
- [mdn](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Asynchronous)

分かってきたと思ったころに分かっていないことを確認するのに役立ったリソース。
- [JavaScriptのasync/await 理解してますか？ 説明できますか？ クイズに答えてもらって良いですか？ - YouTube](https://www.youtube.com/watch?v=T-_0Pc5P12U&t=36s)

ある程度自信を持って理解したと言えるようになるためのリソース。
- [JavaScript Promiseの本](https://azu.github.io/promises-book/)のコラム
- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)
- [mdn](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Asynchronous)(初歩にして真髄的なとこがある)

環境における非同期処理の仕組みについて理解するためのリソース。
- [Inside look at modern web browser (part 2) - Chrome Developers](https://developer.chrome.com/blog/inside-browser-part2/)
- [Angular Basics: Introduction to Processes, Threads—Web UI](https://www.telerik.com/blogs/angular-basics-introduction-processes-threads-web-ui-developers)
- [What the heck is the event loop anyway? | Philip Roberts | JSConf EU - YouTube](https://www.youtube.com/watch?v=8aGhZQkoFbQ&t=804s)

現代の非同期処理の機能を俯瞰するためのリソース。
- [Asynchrony: Under the Hood - Shelley Vohr - JSConf EU - YouTube](https://www.youtube.com/watch?v=SrNQS8J67zc)

