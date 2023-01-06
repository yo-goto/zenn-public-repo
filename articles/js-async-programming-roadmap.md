---
title: "JSの非同期処理を理解するために必要だった知識と学習ロードマップ"
published: true
cssclass: zenn
emoji: "🦄"
type: "idea"
topics: [javascript, 非同期処理, ロードマップ, node, deno]
date: 2022-04-08
modified: 2022-11-16
url: "https://zenn.dev/estra/articles/js-async-programming-roadmap"
tags: [" #type/zenn #JavaScript/async  "]
aliases:
  - 記事『JSの非同期処理を理解するために必要だった知識と学習ロードマップ』
  - 非同期処理学習のロードマップ
---

# はじめに

JavaScript の非同期処理を学習してみて「ある程度自信を持って理解できたと言える」状態に到達したので、その感想とまとめの学習ロードマップとその中でどのような知識が必要になるかを紹介したいと思います。

この記事自体は後から別の記事で参照するかもしれませんが、~~具体的な話の無い気軽な内容なので流し読み程度に見てもらえるといいかもしれません~~(非同期処理の具体的な話も後半から盛り込みました)。あるいは、自分が実際に学習してきた道筋に基づいているのでショートカットとして参考にしてもらったり、使えるリソースなどの情報が共有できると思います。

もしくは「**JavaScript 初心者が非同期処理を理解できるようになるまでの道筋**」というストーリーで１つのサンプルとして見ていただけるといいかもしれません。

この記事自体に間違いや修正、更新があれば適宜修正していきます。~~また、リアルタイムで学習を進めている部分もあるので結構頻繁に更新しています~~。

:::details ChangeLog
大きな変更のみをトラッキングしています。
- 2022/11/16
  - 本の内容を反映させた追記・修正を追加
- 2022/05/21
  - 構成を修正
  - 「V8 エンジンから考える」の項目を追加
- 2022/04/30
  - 「イベントループの共通性質」の項目を追加
  - 「ロードマップのまとめ」の項目を追加
- 2022-04-28
  - 「Event loop への解像度を上げる」の項目を追加
  - 全体の項目を修正してアップデート
  - 「難しさの原因」の項目を追記
- 2022-04-17
  - 「お知らせ」の項目を追加
  - 「追記: 非同期処理機能を俯瞰する」の項目を追加
  - 内容の微修正
- 2022-04-14
  - ランタイムの補足を追加
  - 「環境」と「非同期処理の仕組み」に関してのまとめを追記
  - Web Workers についての記述を追加
- 2022-04-13
  - 「追記: API の機能を提供する環境について学ぶ」の項目を追加
  - Web APIs / Runtime APIs の項目が抜けていたので追加
  - await 式の補足を追加
:::

# (注意) 本の方でより分かりやすく説明しています

非同期処理の学習ロードマップなどと銘打って記事を書いたからには、自分も具体的な非同期処理についての解説やアウトプットをすべきだと考えたので、主に Promise チェーンについての解説をするために Zenn の Book として「**イベントループとプロミスチェーンで学ぶJavaScriptの非同期処理**」を公開しました。

自分のアウトプットを兼ねており、内容としては記事と同質なので無料公開しています。

https://zenn.dev/estra/books/js-async-promise-chain-event-loop

:::message alert
追記 2022-11-16:  
上記の本を書く過程で勘違いしていたり、理解しきれていなかった部分がいくつもあったことが発覚しました。この記事でも間違っている箇所を修正しましたが、本ではロードマップ的なプロセス自体を重要度によって再構成して、より分かりやすく解説しているので、そちらを参照してもらえると良いと思います。
:::

# 雑感

:::message alert
この項目は完全に個人の意見や感想なので飛ばしてもらっても構いません。
:::

## 非同期処理の感想

まず非同期処理について学んでみて、「かなり難しく、学習に時間がかかる」という印象を受けました。イメージとしては**学習前予想時の4倍**くらい難しいです。理解するために必要な情報の量が学習想定を遥かに超える上に、学習におけるトラップが非常に多いため、勘違いしたり、いつまでも謎が解明されないという現象が長く起きます。

非同期処理そのものについては"制御の流れ"が掴みづらく、バックグラウンドで何が起こっているかを把握しないと自分で書こうとしたときにまったく予測ができません。非同期処理の流れ自体が掴みづらいので、同期処理と非同期処理が混じっているときは更に予測しずらくなります。加えて、非同期処理には種類があり、**その種類ごとに実行のタイミングが実は異なる**ので、それらが入り交じることで更に予測が難しくなっています。

そもそも、「同期」と「非同期」という言葉に騙されそうになります。

局所的には Promise の概念が掴みづらく、 Promise チェーンでは特にコールバック関数がキーになるのですが、アンチパターンを学ぶことでようやく使えるレベルの理解に到達できたと感じました。

また、async/await についても、そもそも Promise チェーンを理解していないと使い物にならず、**その名前から明らかに非同期処理の主役なのではないかと感じられる**一方で、実体は異なるので「これは騙されるな」という印象を受けました。async/await と Promise の関係が把握できていないと致命的に混乱することになります。async 関数が Promise インスタンスを返すこと、await 式が Promise の評価値を返し、**async 関数内の実行フローを分割すること**、同期処理と非同期処理を行ったりきたりすること、それと**逐次的に処理を行う**ことなどとの違いなど、ここらへんではトラップとなるポイントがいくつもあります。

「Promise とか await とかっては実は非同期処理ではないのでは？😵‍💫」というような疑問が途中湧き上がったりもしますが、個人的にはその疑問を解消するところがターニングポイントになるなと感じました。

基本的には、解説記事を読んで理解したつもりになっても、制御の流れをコードから想像してみたり、実際に手を動かさないと理解はかなり厳しいと感じました。ドキュメントだけで理解するのはかなり困難なので映像や可視化ツールを使うことで比較的つかみやすくなり、実際にローカル環境で実行してみることで理解でき、だんだんと制御の流れが推測できるようになってきます。

## 難しさの原因 (追記)

これは**初学者の視点からの話**ですが、多くの非同期処理の解説では「**真実ではあるが、現実のすべてを語っているわけではない**」というパターンが多く、それゆえに非同期処理について勘違いしてしまったり、いつまでも理解できないという現象がおきがちだと感じました(これについては初学者向けの情報としての側面があるからという理由もあるかもしれません)。

:::message
もちろん、現実に起こるすべてのことを語れば良いのかという話もありますが、それはそれで問題で、いきなり仕様や実装の話をされて理解することは非常に困難です。あるレベルまでいかないと仕様や実装のことは理解できません。なので、適切なレベルでの抽象化とちょうどよい塩梅の情報の取捨選択で一本の道筋が分かるようなものがあると良いなと感じました。
:::

例えば、「JavaScript はシングルスレッドで実行される」という命題は真ですが、現実に起こることのすべてを語っているわけではないです。この命題に固執すると非同期処理の仕組みを理解できません。

関連して、非同期処理と並列処理を混同するのはありがちで、「非同期処理は並列処理ではない」という命題も真ですが、これに固執すると「実行した処理の完了を待たずに次の処理に行く」も理解できなくなってしまいます。「**じゃあ、その処理はどこでいつ完了するんだ?**」という疑問をいだきませんでしたか？

これついては、実際には「API を介して環境に委任した作業はバックグラウンドで並列的に行われ、それが完了したら何かをメインスレッドで非同期的に行う」というような両方が組み合わさって起きているというように解釈をしないと非同期処理そのものを行う目的について混乱するはめになります。

その他にも「Promise は同期処理」など言葉が不足した文言が存在しており、「`Promise()` コンストラクタ関数に渡す `executor` 関数というコールバックの処理自体は同期的に実行されるが、Promise チェーンにおける `then()` メソッドのコールバックは非同期的に実行される」といったように異なる２つの情報を対比させないと誤解したり混乱したりしがちなパターンも多くあるなと感じました。

もう１つ、難しさの原因として考えられるのは「**学習のハードル自体が非常に高いこと**」です。

JavaScript の非同期処理についてはその難しさの性質と必要な情報量の多さのために、**色々なリソースから断片的な情報を拾い集めて繋いでいく必要があります**。この作業が非同期処理については格段に多く、ほとんどは英語の情報からかき集めてくる必要があります。追記の項目でも書きましたが、非同期処理の話題で目につくような Promise や async/await といった言語の機能だけでは**どうあがいても理解できない仕組み**になっており、そのことについて触れられているリソースもそこまで多くないので、「理解できているようで実はまったく理解できていない」状態になる可能性が非常に高いです。言語機能のみを見ていても、情報が不足しすぎていて何を理解していないかが分からないので、どこまでいっても非同期処理が理解できないようになっています。

「仕様を見ればよいのでは?」という結論に至るわけですが、初心者で制御予測ができる程度の非同期処理の基礎を理解するのに最初から「仕様」を見ようと思う人も読みとける人もほとんどいません。しかも、JavaScript の非同期処理に関わる Event loop などの仕様は ECMAScript にはなく別のところにあり、しかもブラウザとランタイム環境で実装がそれぞれ異なるため、かなり頑張って調べないと到達できないようになっています。

# 学習ロードマップ

## ECMAScript の基本を学習

非同期処理というカテゴリについては(そもそも JavaScript は)、多くの有用なリソースがインターネット上で公開されているので最大限それらを活用して学習をすすめていきます。

まずは非同期処理の前に外すことのできない必要な知識として以下の事柄を知っているか確認します。
- 式(Expression)
- 関数式
- コールバック関数
- 無名関数
- アロー関数と省略形
- 即時実行関数

このあたりの知識が他のドキュメントや解説を見る時に**暗黙的に必要**になってきます。特にコールバック関数とアロー関数は重要です。async function を使用する際には即時実行関数がよくでてきます。await を理解する際にも式の概念がキーとなるので重要です。非同期処理は JavaScript の中でも比較的高度な内容であり、こういった知識が前提とされた状態で話が進んでいきますので理解しておくのを推奨します。不安がある場合には、azu 氏の "JavaScript Primer" で「[式と文](https://jsprimer.net/basic/statement-expression/)」と「[関数](https://jsprimer.net/basic/function-declaration/)」の項目を読んでおくのをおすすめします。

以上が理解できているなら、まず「非同期処理の基礎」を一から学ぶために "JavaScript Primer" の次の項目を読みすすていくのをおすすめします。"JavaScript Primer" では、「JavaScript の非同期処理の歴史的な発展」が俯瞰でき、順を追って非同期処理の必要な知識が一通り学習することができるため、自分も非常にお世話になりました。

https://jsprimer.net/basic/async/

## 制御予測のための基礎知識を学ぶ

非同期処理の基礎的な知識を大体網羅して「完全に理解した」と思ってコードを実際に書こうとしても(非同期処理の制御が予測できず)かけないことに気づくので次に非同期処理のバックグラウンドでの動きを学習します。

まずは非同期処理の流れを理解するのに必要な用語知識を確認しておきます。

- concurrent(並行)/parallel(並列)
- asynchronous(非同期)/synchronous(同期)
- sequential(逐次的)
- blocking/non-blocking
- single-thread/multi-thread
- Node.js や V8 engine についての基礎知識

上記の単語については、基礎的な学習を終えていればある程度の解像度で理解できているはずですが、解像度を上げることが必要になってきます。おそらく "synchronous" に振り回されるので、"sequential" と "blocking" を意識できるように注視しておくと良いと思います(自分は途中で「同期的」という概念が分からなくなり非常に混乱しました)。

blocking/non-blocking については Node.js 公式サイトの次のページを一読しておくと理解が進みます。

https://nodejs.org/ja/docs/guides/blocking-vs-non-blocking/

blocking の例として Node.js の同期 API (Synchronous API) についても知っておくとよいです。同期 API は Node.js の Event loop と後続の JavaScript の処理を、その API の処理が完了するまでブロックします。これらの API は `readFileSync` など名前の最後が `Sync` で終わります。

次に非同期処理の流れを予測できるようになるため必要な知識と道具を獲得します。

- Event Loop(イベントループ)
- Call Stack(コールスタック)
- Heap(ヒープ)
- **Web APIs** / Runtime APIs
- Task(タスク) と Task Queue(タスクキュー)
- Microtask(マイクロタスク) と Microtask Queue(マイクロタスクキュー)

これらの用語は非同期処理の種類を認識して、バックグラウンドでの流れをある程度の解像度で理解するのに必要になってきます。非同期処理の流れ(実行順番)を把握・予測するためには必ず理解したほうがよいです。これらのアイテム無しでは進むことができません(Event loop なしで非同期処理の制御予測をするのは**不可能**です)。

まず、Call Stack と Event Loop を理解するために JSConf EU での Philip Roberts 氏の講演動画「What the heck is the event loop anyway?」を見ることを強くオススメします。

@[youtube](8aGhZQkoFbQ)

この動画は多くの人がオススメしたり、様々な記事で引用されています。非常に分かりやすく説明されており、非同期処理の片方(Task)を理解するのに役立ちます。2014 年の動画ですが、JSConf の動画は他にもいくつか見ましたが、~~この動画以上に分かりやすく説明しているものはありません~~。

追記:「What the heck is the event loop anyway?」は Event loop と非同期処理を理解するための**入り口となる動画**として非常に素晴らしいのですが、JSConf での他の関連する動画もこの動画と同じかそれ以上に視聴する価値があることに気づきました。この点については追記の項目で記載します。

:::message
Chrome などのブラウザ環境と Node.js や Deno の環境では JavaScript エンジンとして同じ V8 エンジンを採用してますが、イベントループの実装はそれぞれ違います。それぞれの環境でイベントループを実装するのに使われているライブラリは以下のものとなります。

- [Libevent](https://libevent.org) (Chrome): an event notification library
- [Libuv](https://libuv.org) (Node.js) : a multi-platform support library with a focus on asynchronous I/O
- [Tokio](https://tokio.rs) (Deno) : an asynchronous Rust runtime

V8 エンジンが主に担当しているのは、Heap と Call Stack です。V8 のソースコードには setTimeout や DOM、HTTP request といった Web APIs は含まれていません(実は `setTimeout()` はタイマー機能のついていない Web API もどきが提供されています)。

上記の動画で解説されているのは、ブラウザ環境での Event Loop です。Promise チェーンや async/await といった言語としての非同期処理を理解するレベルでの抽象化なら、Node.js でも Deno でも同じように考えて大丈夫です(もちろん開発レベルで必要なら I/O とかタイマーの振る舞いは違う可能性があるので具体的に知る必要はあると思いますが)。ただし、Web APIs の部分はブラウザ特有のものなので、Node.js や Deno ではそれに対応するものとして Runtime APIs で置き換えて考えます。
:::

動画で概要を理解したら同氏が開発した JavaScript の可視化ツールである "Loupe" で実際に `setTimeout()` 関数のコールバック関数やクリックイベントなどがどのように動くか実験してみてください。

http://latentflip.com/loupe/

![js loupe image](/images/js-async/img_loupe-js-async.jpg)

Loupe では２種類ある非同期処理の種類の内の１つ(Task: タスク)しかサポートされていないので、次に Github 社のエンジニアである Andrew Dillon 氏が開発した "JavaScript Visualizer 9000" を使用して Promise 関連の非同期処理(Microtask: マイクロタスク)を理解するために使用します。

https://www.jsv9000.app

JavaScript Visualizer 9000 ではサンプルがいくつも用意されており、それらを理解するだけでも非常に有用です。二種類の非同期処理についてサポートされているので自分が書いてみた非同期処理を実際にこのアプリで実行してみて非同期処理の流れが予測どおりか確認していくことで非同期処理の流れが読めるようになってきます。ただし、async/await はサポートされていないのですそこは注意してください。

![js visualizer 9000](/images/js-async/img_js-visualizer-9000.jpg)

JavaScript Visualizer 9000 を使って、Task と Micortask の組み合わせに慣れてください。ここでは重要なこととして、 ~~Task Queue (Task Queue) は実は**キューではなくセット**であることを理解しておく必要があります。複数の `setTimeout()` 関数を色々な delay 時間を指定して実行してみれば理解できますが、あきらかにキューではないです。セットであるというのも、そもそも仕様として定義されています~~。(~~完全な間違い~~)

追記 2022-11-16: `setTimeout()` 関数のタスクキューへのタスク挿入順序自体が Visualizer 側の実装ミスと考えられます。タスクキューは Set というデータ型であると仕様で明記されていました。

> [Task queues](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue) are [sets](https://infra.spec.whatwg.org/#ordered-set), not [queues](https://infra.spec.whatwg.org/#queue), because the [event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) grabs the first [_runnable_](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task-runnable) [task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task) from the chosen queue, instead of [dequeuing](https://infra.spec.whatwg.org/#queue-dequeue) the first task.
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3) より引用)

追記: これは、「~~Task queues(という集合)はただのキューの集合ではなく、セットである~~」ということを意味していて、~~単一の "Task queue" は名前の通りキューであると考えられます。実はこれ非常に重要な事柄でした~~。

追記 2022-11-16: 上の追記で訂正した内容はかなり間違っていたことが発覚しました。タスクキューは Queue というデータ構造ではなく Set というデータ構造です。これは HTML Standard ではなく、それらの仕様が基づく用語や概念を定義している [Infra Standard](https://infra.spec.whatwg.org/) というページに記載されています。Queue と Set は List というデータ構造のサブセットであり、Set は順序集合([ordered set](https://ja.wikipedia.org/wiki/%E9%A0%86%E5%BA%8F%E9%9B%86%E5%90%88#%E5%89%8D%E9%A0%86%E5%BA%8F%E3%83%BB%E5%8D%8A%E9%A0%86%E5%BA%8F%E3%83%BB%E5%85%A8%E9%A0%86%E5%BA%8F))と呼ばれるものです。Set は List の一部であり、順序がついているのでほとんどは Queue と同じ動作になるのですが、細かい部分は異なります。

とにかく、JavaScript Visualizer 9000 は非同期処理を理解するまで長い付き合いになったので非常にオススメです。これが無かったら非同期処理を理解できなかったかもしれません。

:::message alert
JavaScript Visualizer 9000 を使って可視化することによって、マイクロタスクやタスクが Call stack と Event Loop でどう動くかを予測できるようになりますが、JavaScript Visualizer 9000 ではいくつかの要素が欠けている、または実装ミスと考えられる部分がありますので注意してください。

例えば、Web APIs の要素が無いため、Web API による並列的作業についてや、タスクキューとの関係性、複数のタスクキューが存在していることに気づけ無い可能性が高いです。

なぜ **Web APIs** の要素を省いたのかはわかりません。実はこれが省略されているために、Visualizer 9000 で複数の `setTimeout()` 関数を色々な delay 時間を指定して実行してタスクの動きを見ると「キュー」と思えない挙動に見えてしまうことに気づきました。基本的な流れを理解するのには問題ないですが、注意してください。

Visualizer 9000 では簡単にコードが共有できるので実際に以下のリンクから実行して挙動を見てみてください↓
- [manySetTimeout.js](https://www.jsv9000.app/?code=Ly8gbWFueVNldFRpbWVvdXQuanMKc2V0VGltZW91dChmdW5jdGlvbiBiKCkgewogIGNvbnNvbGUubG9nKCJiOiBhZnRlciA1MDBtcyIpOwp9LCA1MDApOwpzZXRUaW1lb3V0KGZ1bmN0aW9uIGMoKSB7CiAgY29uc29sZS5sb2coImM6IGFmdGVyIDBtcyIpOwp9LCAwKTsKc2V0VGltZW91dChmdW5jdGlvbiBhKCkgewogIGNvbnNvbGUubG9nKCJhOiBhZnRlciAxMDAwbXMiKTsKfSwgMTAwMCk7CnNldFRpbWVvdXQoZnVuY3Rpb24gZCgpIHsKICBjb25zb2xlLmxvZygiZDogYWZ0ZXIgMTAwbXMiKTsKfSwgMTAwKTsKCmZ1bmN0aW9uIGQoKSB7CiAgY29uc29sZS5sb2coImQ6IHN5bmMgZnVuY3Rpb24gZXhlY3V0ZWQgaW1tZWRpYXRlbHkiKTsKfQoKZCgpOwo%3D)

更に、スクリプトの評価によるコールスタック上へのグローバルコンテキストのプッシュとポップが可視化されていないため、最初のマイクロタスク・タスクの実行タイミングを誤解する可能性があります。細かいことについては Book の方に記載しています。
:::

## Promise チェーンの構築のアンチパターンを学ぶ

さて、ここまでくれば非同期処理がかなり理解できているはずです。きっと「非同期処理、完全に理解した😼」状態ですね。

それでは、テストとしてやっすん氏の次の動画でクイズをといてみてください。

@[youtube](T-_0Pc5P12U)

いかかですか?
自分の場合は「Promise とか async/await のこと全然わかってねえじゃん...😭」となりました。

動画をみてもらったら分かると思うのですが、Promise と `await` 式の関係が分かっていないと 1 番目と 2 番目のクイズが解けませんし、そして Promise チェーンが理解できていないと最後のクイズがとけません。

なので、ここからは実際に Promise チェーンを構築することに注力して学習をすすめていきます。また、Promise まわりの細かい疑問を調べ尽くして解消していきます。

- Microtask と Task の両方を念頭に Promise チェーンを構築できるようにする
- 同期処理と非同期処理を混在させた Promise チェーンを構築できるようにする
- Promise チェーンで値を次のチェーンへと渡し、エラーハンドリングが正しく行えるようなチェーン構築をできるようにする

ここで async 関数と `await` のことも考えてしまうとかなり混乱するので Promise チェーンのことに注力した方がいいかもしれません。

この段階では azu 氏の "JavaScript Promise の本" のコラムなどが非常に有用で、大変お世話になりました。

https://azu.github.io/promises-book/#promise-is-always-async

また、mdn のドキュメントが色々な疑問を解消してくれたので主に次のページを含む非同期処理に関連するページを読んでいくのがいいと思います。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises

上記の mdn の解説でも Promise に関してのアンチパターンが紹介されていますが、下記のページにおいていくつもの Promise アンチパターンが記載されているので１つずつ理解していきます。

https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html

アンチパターンとその理由を理解することによって Promise の曖昧だったところが非常にクリアになっていくのが分かります。

さて、ここまで来れば基礎的な Promise チェーンがある程度構築できるようになったと思います。自分もこの時点で非同期 API などの使い方もある程度わかるようになりました。

## async/await の本格的な学習

ここから本格的に async/await を理解するための学習が始まります。特に `await` がやっかいです。async 関数の内部での await によって「これは同期処理なのか、非同期処理なのか」がよくわからなくなってきます。JavaScript Visualizer 9000 では async/await が使えず可視化できないため、手元でコードを書いて実行していくことをメインにして理解していきます。

自分の場合は、いままでやってきた Promies の考え方と噛み合わなかったり、Promise との組み合わせ方を理解するまでに時間がかかりました。これは Promise が理解しきれていなかったことを意味しています。**複数の `await` 処理の入った複数の async 関数の処理を Promise チェーンで実現できるようになることが大切です**。Promise チェーンなら JavaScript Visualizer 9000 で可視化できますので。

また、最初は async/await によって Promise がいらなくなってしまうのかとも思いましたが、そんなことは無かったです。これについては、以下の複数の Promise を扱うことのできる静的メソッドを学習することで単純な async/await でやるよりも**効率的に目的をはたせる**ことを理解できれば体感できるはずです。

- [Promise.all()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
- [Promise.race()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/race)
- [Promise.allSettled()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
- [Promise.any()](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/any)

mdn で各メソッドの解説や次の記事を読むのをオススメします。それぞれの使い用途や違いなどがあきらかになると async/await でできること、単純な `await` 式にのみ注力してしまうとやりづらいことが逆に分かってくるはずです。

https://okapies.hateblo.jp/entry/2020/12/13/154311

更にこれによって async/await が Promise を代替するのではなく、Promise ベースの非同期処理の利便性を一部向上させる Promise というシステム自体に基づいた拡張的な機能であり、**Promise と一緒に使っていくもの**であるということが認識できると思います。

そもそも、`await` 式は「右辺の Promise インスタンスが Settled 状態(Fullfilled または Rejected 状態)になるまで非同期処理の完了を待ち、Promise インスタンスの状態が変化したらその Promise インスタンスの**評価結果を値として返す**」ものなので、Promise を扱っていると認識してしないとやっすん氏の動画で紹介されたミス(タスクを作成する `setTimeout` 関数を Promise でラップすることなくそのまま `await` してしまうなど)を犯してしまいます。

:::message
`await` 式では async 関数の内部で非同期処理の完了を待ちますが、その"待っている"というのは実際にすべての処理が止まっているわけではなく、async 関数の呼び出し元に一旦制御が戻って別の処理を再開し、`await` で待っていた Promise インスタンスが解決した後に再び async 関数に戻って `await` 式の後にある次の処理を再開するということを意味しています。

async 関数の中の処理は 1 つ 1 つ処理が終わってから順番に処理が進むので、「逐次的(順次的)な処理」がおこなわれていますが、`await` 式に出会うたびに、一度制御を呼び出し元に戻して別の同期処理などを実行しつつ、`await` で待っていた Promise が解決されたらコールスタックの状況を見て再び制御が async 関数に戻るので、実際に制御の流れは「同期的」ではありません。これは Promise チェーンでも同じ状況となります。`then()` のコールバック関数はマイクロタスクキューへと一旦送られ、制御は呼び出し元に戻ります。乱暴な言い方ですが、async/await は本質的には Promise チェーンの書き方を変えているだけです。

もし「同期的」に処理が進んでいるなら、async 関数の内部で `await` 式によってメインスレッドが専有されることで「ブロッキング」が発生し、その間は一切の処理が進まないことになります。もちろん、そんなことは起きていません。そんなことが起きるなら、そもそも「非同期処理」を行うメリットなんてないですよね。

Prmoise チェーンや `await` 式を利用している async 関数の制御が「同期的」であると混同してしまうのは、 Promise チェーンや async 関数内の処理のみについて考えれば確かにステップごとに「逐次的(順次的)」に処理が進んでいるからです。しかし、**そのステップの間では別の処理が進んでいます**。

「同期」や「非同期」という言葉に振り回されることによって、これを理解できるようになるためにかなり時間がかかりました。"blocking(ブロッキング)" と "sequential(逐次的)" という言葉に注視しておくと良いと言ったのはこのためです。

「Promise とか await とかっては実は非同期処理ではないのでは？😵‍💫」という冒頭の疑問はこれで解消できるはずです。

async/await では無いですが２つの Promise チェーンを同時に実行するのを可視化してみると制御が行ったり来たりしつつもステップごとに実行している様が理解できると思います。Visualizer で可視化してみたので確認してみてください。
[doublePromiseChain.js - JS Visualizer 9000](https://www.jsv9000.app/?code=Ly8gZG91YmxlUHJvbWlzZUNoYWluLmpzCgovLyB0aGluayBhd2FpdCB3aXRoIHByb21pc2VzCi8vIHJlbW92ZSBuZXN0aW5nLCBhbHdheSBlbmRzIHdpdGggY2F0Y2goKSBtZXRob2QKZnVuY3Rpb24gcmV0dXJuUHJvbWlzZUZuQSgpIHsKICBjb25zb2xlLmxvZygiLS0%2BIEVudGVyIHJldHVyblByb21pc2VGbkEiKTsKICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDEwMCkKICAgIC50aGVuKGZ1bmN0aW9uIGExKGRhdGEpIHsKICAgICAgY29uc29sZS5sb2coIi0tPiBFbnRlciB0aGVuIGNhbGxiYWNrOiBhMSIsIGRhdGEpOwogICAgICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDEwMSk7CiAgICB9KQogICAgLnRoZW4oZnVuY3Rpb24gYjEoZGF0YSkgewogICAgICBjb25zb2xlLmxvZygiLS0%2BIEVudGVyIHRoZW4gY2FsbGJhY2s6IGIxIiwgZGF0YSk7CiAgICAgIHJldHVybiBQcm9taXNlLnJlc29sdmUoMTAyKTsKICAgIH0pCiAgICAudGhlbihmdW5jdGlvbiBjMShkYXRhKSB7CiAgICAgIGNvbnNvbGUubG9nKCItLT4gRW50ZXIgdGhlbiBjYWxsYmFjazogYzEiLCBkYXRhKTsKICAgICAgcmV0dXJuIFByb21pc2UucmVzb2x2ZSgxMDMpOwogICAgfSkKICAgIC50aGVuKGZ1bmN0aW9uIGQxKGRhdGEpIHsKICAgICAgY29uc29sZS5sb2coIi0tPiBFbnRlciB0aGVuIGNhbGxiYWNrOiBkMSIsIGRhdGEpOwogICAgICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDEwNCk7CiAgICB9KQogICAgLmNhdGNoKChlKSA9PiBjb25zb2xlLmVycm9yKGUpKTsKfQoKZnVuY3Rpb24gcmV0dXJuUHJvbWlzZUZuQigpIHsKICBjb25zb2xlLmxvZygiLS0%2BIEVudGVyIHJldHVyblByb21pc2VGbkIiKTsKICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDIwMCkKICAgIC50aGVuKGZ1bmN0aW9uIGEyKGRhdGEpIHsKICAgICAgY29uc29sZS5sb2coIi0tPiBFbnRlciB0aGVuIGNhbGxiYWNrOiBhMiIsIGRhdGEpOwogICAgICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDIwMSk7CiAgICB9KQogICAgLnRoZW4oZnVuY3Rpb24gYjIoZGF0YSkgewogICAgICBjb25zb2xlLmxvZygiLS0%2BIEVudGVyIHRoZW4gY2FsbGJhY2s6IGIyIiwgZGF0YSk7CiAgICAgIHJldHVybiBQcm9taXNlLnJlc29sdmUoMjAyKTsKICAgIH0pCiAgICAudGhlbihmdW5jdGlvbiBjMihkYXRhKSB7CiAgICAgIGNvbnNvbGUubG9nKCItLT4gRW50ZXIgdGhlbiBjYWxsYmFjazogYzIiLCBkYXRhKTsKICAgICAgcmV0dXJuIFByb21pc2UucmVzb2x2ZSgyMDMpOwogICAgfSkKICAgIC50aGVuKGZ1bmN0aW9uIGQyKGRhdGEpIHsKICAgICAgY29uc29sZS5sb2coIi0tPiBFbnRlciB0aGVuIGNhbGxiYWNrOiBkMiIsIGRhdGEpOwogICAgICByZXR1cm4gUHJvbWlzZS5yZXNvbHZlKDIwNCk7CiAgICB9KQogICAgLmNhdGNoKGUgPT4gY29uc29sZS5lcnJvcihlKSk7Cn0KCnJldHVyblByb21pc2VGbkEoKQogIC50aGVuKChkYXRhKSA9PiB7CiAgICBjb25zb2xlLmxvZygiQWZ0ZXIgZXhlY3V0aW5nIEZuQSIsIGRhdGEpOwogIH0pOwpyZXR1cm5Qcm9taXNlRm5CKCkKICAudGhlbigoZGF0YSkgPT4gewogICAgY29uc29sZS5sb2coIkFmdGVyIGV4ZWN1dGluZyBGbkEiLCBkYXRhKTsKICB9KTsKCgo%3D)
:::

"JavaScript Primer" に戻ってくると、「Promise と async/await は一緒に使っていくものである」ということを再発見できます。

> このようにAsync Functionやawait式は既存のPromise APIと組み合わせて利用できます。 Async Functionも内部的にPromiseの仕組みを利用しているため、両者は対立関係ではなく共存関係になります。
> (https://jsprimer.net/basic/async/#relationship-promise-async-function より引用)

Promise の静的メソッドと `await` 式を組み合わせる例として、複数のリソース非同期的に取得する際に async 関数内で `await promise1; await promise2; await promise3;` のように１つずつ取得が完了するのを待つのではなく、`await Promise.allSettled([promise1, promise2, promise3]);` として複数の非同期処理を並行して走らせることでリソースの取得時間を削減できるなどが挙げられます。

つまり、"async/await" ではなく、"**Promise with async/await**" 的なノリであることを理解できます。

また、モダンな JavaScript/TypeScript ランタイムである Deno では Node.js と比較したときの特徴として「Deno での非同期アクションはすべて Promise を返す」という点をあげています。

> All async actions in Deno return a promise. Thus Deno provides different APIs than Node.
> ([Introduction - Deno Manual](https://deno.land/manual#comparison-to-nodejs) より引用)

"Async/await-based" や "Async/await-first" ではなく、"Promise-based "や "Promise-first" などの言葉を聞いたことがあると思います。その理由は "Promise" が常に非同期処理の主役だからです。

それが理解できたら、今度こそ async/await の強みを理解できる段階となります。

https://developers.google.com/web/fundamentals/primers/async-functions#%E4%BE%8B_%E3%83%AC%E3%82%B9%E3%83%9D%E3%83%B3%E3%82%B9%E3%81%AE%E3%82%B9%E3%83%88%E3%83%AA%E3%83%BC%E3%83%9F%E3%83%B3%E3%82%B0

非同期処理のループなどを構築する際に、Promise チェーンでは複雑になってしまうコード("スマート"な書き方のコード)を、async 関数内では `while` ループや `for` ループといった"シンプルで読みやすい平凡な書き方"へと持ち込めるようになります。

逆に、コールバック関数に async 関数を使う時は注意する必要があったり、どのタイミングでどれを await させるかなどの難しさも分かってきますが。

それが出来たら、仕上げに "Top-level await" を学びます。Top-level await で `fetch` などの非同期 API が簡単に使えるからと言って、いきなり Top-level await を学んでしまった場合は **通常の async/await における「同期と非同期」が分からなくなってしまい**、非常に混乱することが予測されるので、この最後の段階で学ぶことをオススメします。

https://qiita.com/uhyo/items/0e2e9eaa30ec2ff05260

さて、ここまでくれば、「非同期処理の基礎が身についた」といえる状態になったのではないでしょうか。もちろん、非同期処理のすべてを学びきった訳ではないので、今後も学習は続きます(非同期イテレータ `for await...of` やジェネレータなどはまだ自分もわかりません)。

~~以上が自分なりに「ある程度、非同期処理が理解できた言える状態」まで到達するのに辿ってきたロードマップです。~~
追記に色々記載しましたが、これでもまだ理解しきれていない部分が多くあったので、まだ話は続きます。

## API の機能を提供する環境について学ぶ (追記)

実は、非同期処理のための「ECMAScript」の言語機能だけを見ていても、JavaScript における非同期処理の仕組みは理解できませんでした。

:::message
追記 2022-11-16: この話が非同期処理の理解の上でもっと重要であり、トラップとなる話題であると気づきました。
:::

JavaScript とうのは常に「実行する環境」があり、その**環境によって利用できる機能が異なる**ので、ECMAScript(JavaScript の言語仕様)における非同期処理の機能を考えるよりも、「環境と結びついたものとしての JavaScript の非同期処理」、ひいては「環境そのもの」のことを考えないと非同期処理を理解できません。

というのも、`setTimeout()` API のタイマー処理や `fetch()` API のデータフェッチという「根本的な処理」を実際に「並列的に」行ってくれているのは次のような環境そのものだからです。

- Web APIs を提供するブラウザ環境 (Chrome の場合)
- Runtime APIs を提供するランタイム環境 (Node.js や Deno の場合)

https://www.freecodecamp.org/news/async-await-javascript-tutorial/

> It turns out that it is **the environment** that takes on the work, and the way to get the environment to do that work, is to use functionality that belongs to the environment. For example fetch or setTimeout in the browser environment.
> (上記ページより引用、太字は筆者強調)

従って、ECMAScript の言語としての「Promise や async/await キーワード」や「シングルスレッドと並行(concurrent)処理」を考えていても現実に起こる「非同期処理の仕組み」についての謎が永遠に解けません。

例えば、`fetch()` メソッドは ECMAScript の範疇ではなく、Web API(Web Platform API) であり、実際にその機能を提供するのはブラウザ環境やランタイム環境です。

Chrome といったモダンなブラウザ環境を代表として考えると、ブラウザはそもそも様々なタスクを行っているので、"シングルスレッド"などではなく、"マルチプロセスアーキテクチャ"であり、様々なタスクの責務を持つ分離したプロセスの中に複数のスレッドを持つ形になっています。

https://developer.chrome.com/blog/inside-browser-part2/

Chrome では以下のような複数のプロセスが作成されます。

- Browser process
- Renderer process
- GPU process
- Plugin process
- Extensions process
- Utility process

`setTimeout(cb, delay)` の例では、指定した時間が経過後にコールバックを Task queue へと追加し、Event Loop によって Call stack へと運んでメインスレッドで実行しているので、これを「シングルスレッド」で実行されていると解釈することは可能ですが、「現実のタイマー処理」はメインスレッドそのもので行われているわけではないです。もしタイマー処理がメインスレッドで行われているならそれこそ**ブロッキングが発生するはず**ですから。

`setTimeout()` メソッドはブラウザ環境やランタイム環境が提供する API であり、その「コールバックをキューに追加するまでの時間を図る」という処理自体は環境の別のところで行われいるはずです。

コールバック関数自体はメインスレッドで処理されるが、コールバックをキューに追加するまでのタイマーによる時間計測作業は別の場所で行われています。「非同期処理=非同期的なタイミングで実行される処理=コールバック」と考えるなら、コールバック関数の処理という非同期処理そのものは並行(concurrent)に処理されていますが、非同期処理を行うまでのタイミングを図っているタイマーの処理は環境が「並列的」に行っていることになります。

また、`fetch(url)` もネットワークの接続をしてデータを取得しているのに**メインスレッドをブロッキングしないでバックグラウンドで行うことができる**のは、「現実のデータ取得処理」がメインスレッドで行われていないからです。`fetch(url)` そのものの処理が環境の別スレッドで並列(parallel)に行われてる一方で、`fetch(url).then(cb)` の `cb` コールバックについて「非同期処理=非同期的なタイミングで実行される処理=コールバック」として考えるなら、たしかにコールバック関数の処理という非同期処理そのものは並行(concurrent)に処理されています。

実際、JavaScript の非同期処理でよく語られるシングルスレッドは現実のブラウザ環境(Chrome)では「Renderer Process の Main thread」のことであり、`fetch()` メソッドのネットワーク接続やリクエストなどの処理を行っているのは「Browser process の Network thread」です。

`await` 式の前後や Promise チェーンのコールバック自体は確かに「非同期処理=非同期的なタイミングで実行される処理」であり、シングルスレッドで並行(concurrent)に処理されていますが、環境が提供する機能である API の処理自体はメインスレッドとは別のところで並列的に行われています。

「並列と並行が同時に起きている」、「シングルスレッドであり、マルチスレッドである」という文字にすると頭が混乱するような話ですが、**対立する概念が同居して「非同期処理」の仕組みを実現している**と考えないとつじつまが合わないので、こうなります。

:::message alert
すべての非同期 API 処理が別厳密な並列(parallel)処理だとは限りません。ここでいう「並列的」や「並列」という言葉は「メインスレッド外で同時に別のことが起きている」という程度にとらえてください。
:::

この結論に至った経緯として、次の文献がメインとして参考になりました。

https://www.telerik.com/blogs/angular-basics-introduction-processes-threads-web-ui-developers

そして Event Loop と Call Stack を理解するための動画として紹介した『What the heck is the event loop anyway?』では「**一度に複数のことができるのはブラウザがランタイム以上のものであるからで、ブラウザから提供される Web APIs は実質的にスレッドである**」ということが実は語られていました。この動画では「ブラウザ環境」のことについても語られているので、Promise チェーンなどが理解できた後も何度見返してもよいと思います。

> Right, so I've been kind of partially lying do you and telling you that JavaScript can only do one thing at one time. That's true the JavaScript Runtime can only do one thing at one time. It can't make an AJAX request while you're doing other code. It can't do a setTimeout while you're doing another code. **The reason we can do things concurrently is that the browser is more than just the Runtime**. So, remember this diagram, **the JavaScript Runtime can do one thing at a time, but the browser gives us these other things, gives us these we shall APIs, these are effectively threads**, you can just make calls to, and those pieces of the browser are aware of this concurrency kicks in.
> (以下の書き起こしページから引用、太字は筆者強調)

https://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html

:::message
補足: Philip Roberts 氏が言う "Runtime" とは Chrome ブラウザ環境の JS エンジンである V8 のことを言っています。Deno や Node は V8 を含んだ"環境"であり、色々な API を提供するので、ここではブラウザと等価なものとして考えてください。

それぞれ公式サイトでの文言。
- [Deno is a simple, modern and secure runtime for JavaScript and TypeScript that uses V8 and is built in Rust.](https://deno.land)
- [Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine.](https://nodejs.org/en/)

また、引用の "we shall APIs" はおそらく書き起こしの間違いで、実際の動画では "Web APIs" と言っています。
:::

↓ 書き起こしが実際の言葉と若干違っているので動画で確認すると 11:48 ~ のところです。
[イベントループとは一体何ですか？ | Philip Roberts | JSConf EU](https://youtu.be/8aGhZQkoFbQ?t=708)

もう一度流れをまとめると、ブラウザ環境において `setTimeout()` と `fetch()` は **Web API** です(ランタイム環境では Runtime API)。ECMAScript 言語の一部ではありません。両方とも自分で定義した関数を実行しているのではなく API の呼び出し(API call)を行っています。API を介して「時間を図る」、「リソースを取得する」といったタスクが「環境」へ委任(Delegate)され、そのタスクを「環境そのもの」がバックグラウンドで並列的に行います。「環境に委任されたタスクが完了したら何かをメインスレッドで行う」という**コールバック関数の形**で登録しておいた別のタスク(=非同期処理)は、委任されたタスクを終わらせた時点で環境から Task queue や Microtask queue へと送られ、Event Loop によって Call stack へと運ばれた時点で JavaScript のコードとして実行されます。これが JavaScript における非同期処理の仕組みです。

結局の所、非同期処理の仕組みを理解するには「Promise や async/await といった言語機能の概念とその使い方」だけでなく「**API を提供する環境**」のことを知ることが必要だった、という訳です。

本来時間のかかるインターネットを介したデータ取得(`fetch()`)や I/O (`Deno.readFile()` や `fsPromise.readFile()`)などの処理は基本的に環境が API として用意してくれています。もちろんすべての API が時間のかかる処理ではなく、例えば、`URL()` なども Web API なので環境の提供する機能ですが、通信や I/O に比べてすぐに処理が終了するので、非同期ではなく同期的なものとなっています。

:::message
"blocking" の用語のところで紹介しましたが、Node や Deno の環境では時間のかかる処理の完了を待って、あえてメインスレッドの Event Loop をブロッキングする同期 API (Synchronous APIs) を提供しています。Node なら `fs.readFileSync()`、Deno なら `Deno.readFileSync()` などが該当します。用途としては、「スニペット程度のコードだからわざわざ非同期処理にするまでもない」というときや、他の何らかの理由であえて非同期処理にしたくない状況などで使用したりします。

> The synchronous APIs perform all operations synchronously, **blocking the event loop until the operation completes or fails**.
> ([File system | Node.js v17.9.0 Documentation](https://nodejs.org/api/fs.html#synchronous-api) より引用、太字は筆者強調)
:::

API を介して時間のかかる処理は環境へ委任されますが、もし環境が API として用意していないような時間がかかるタスク(メインスレッドを長時間専有してしまうような処理)を**自分で定義した同期関数**などで行う場合は、Web Workers でメインスレッドから分離した別スレッドを作成して、そのスレッドで真に並列(parallel)な処理として走らせることで**メインスレッドの Event Loop をブロッキングせずに済ませる**ことができます。いわゆる「オフロード」というやつです。もちろん Web Workers も Web API です。

https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers

## 非同期処理機能を俯瞰する (追記)

ECMAScript に取り入れられた非同期処理に関するものとして async/await の前に **Generator** という機能がありました。現時点で「非同期処理」の説明に Generator が使われていることはほとんど見かけませんが、async/await の機能の前にこれが導入されていたということを認識しておくといいかもしれません。

ここまで来て、色々な非同期処理の知識を得たと思いますが、ECMAScript の「非同期処理」の俯瞰を行うために Electron のコアエンジニアである Shelley Vohr 氏が JSConf EU で行った講演動画である『Asynchrony: Under the Hood』を見ておくといいと思います。

https://youtu.be/SrNQS8J67zc

Philip Roberts 氏の動画よりも新しいので、ECMAScript の機能として新しく追加された Promise のためのキューである Microtask queue が入った状態の Event loop の話をしています。ですが、最初にこれを見ても多分理解できないと思うので、やはり「What the heck is the event loop anyway?」から視聴するのをオススメします。

以下の項目に触れているため、「非同期処理」の全体を俯瞰できます。

- Callback hell
- Promise chain
- Generator
- async/await

最後には、Callback hell → Promise chain → async/await の変形がみれるので async/await と Promise チェーンの書き換えが分からない人は見るといいかもしれません。

とは言っても非同期処理機能の歴史的な発展については azu 氏の JavaScript Primer でもかなり分かりやすく解説されているので、こちらを理解した上で視聴するのをオススメします。

https://jsprimer.net/basic/async/#sync-processing

## Event loop への解像度を上げる (追記)

Event loop と Call stack を理解するための動画としてはじめの方に紹介した『What the heck is the event loop anyway?』と非同期処理の全体を俯瞰するための動画として紹介した『Asynchrony: Under the Hood』、Event loop がどのように動くか可視化するためのツールである『JavaScript Visualizer 9000』と色々なリソースを紹介してきました。しかし、学習を進めていく内に分かったこととして、これらだけでは Event loop への解像度が足らず非同期処理の制御を予測しきれない部分が**多くありました**。

そもそも、ブラウザ環境とランタイム環境での Event loop の実装は違うので、実際にはその辺を細かく理解する必要があったわけですが、その辺りの話を補うためのリソースも紹介しておきます。

:::message
イベントループの話は非常に煩雑で、ブラウザ環境ではレンダリング、 Node 環境ではフェーズと追加のマイクロタスクキューとタイマーなど、それぞれ考えるべき話があるためかなり複雑になります。そのためこの段階では擬似コードなどを使ってイベントループを理解していくのがよいです。

また、理解を詰めるために今までの学習よりも**多くの断片的なリソースが必要となりました**。

基本的には JSConf のいくつかの動画が非常に参考になりました。これらの動画は最初視聴した際には単純に分かりにくく、『What the heck is the event loop anyway?』が最も理解しやすいと感じていましたが、単に理解度が足りていなかっただけだと分かりました。

難易度的には JSConf.Asia 2018 での Jake Archibald 氏の講演動画である『In The Loop』が結構難しいと思います。
:::

まず、JSConf EU 2018 での Erin Zimmer 氏が行った講演動画である『Further Adventures of the Event Loop』ではブラウザ環境での Event loop を擬似コードによって解説してくれています。

@[youtube](u1kqx6AenYw)

この動画では Event loop への解像度を飛躍的に上げてくれます。この動画のおかげで Event loop について今まで理解しきれなかった点について理解できました。

- Task queue が複数存在し、どのキューを選ぶかはブラウザが行うこと
- Event loop 内にネストされたループ(マイクロタスクを完全に処理するループ)があること
- 最初の Task がスクリプトの評価(Script Evaluation)になること

次のような擬似コードによるイベントループをスモールステップで理解できました(実際には Chrome ブラウザは C++ で書かれているため、理論的な疑似コードとなります)。JS Visualizer を使っているだけでは理解しづらかった部分が非常にクリアになりました。

```js:最も単純なイベントループ
// event loop v1
while (true) {
  task = taskQueue.pop();
  execute(task);
}
```

上の単純なループから始まり、最終的には以下のレンダリングパイプラインまで考えたブラウザ環境のイベントループが理解できます。

```js:ブラウザ環境のイベントループ
// event loop v5
// 無限ループ
while (true) {
  // 複数ある Task queue から１つを選択
  queue = getNextQueue();
  task = queue.pop();
  execute(task);
  // Task は１つのみ実行する

  // Microtask queue が完全に空になるまで処理する
  while (micortaskQueue.hasTasks()) {
    doMicrotask();
  }

  // 前の描画更新から 16 ミリ秒ほど経っていれば再描画(60fps)
  if(isReapintTime()) {
    animationTasks = animationQueue.copyTasks();
    // 現時点においてキューにあるものすべてを実行する
    // Rendering pipeline 直前に確実に処理する
    for(task in animationTasks) {
      doAnimationTask(task);
    }
    // 再描画
    repaint();
  }
}
```

Rendering pipeline については以下のページを参考にしてください。
https://developer.chrome.com/blog/renderingng-architecture/

ただ、このコードでは実は情報が不足しており、Web API `requestAnimationFrame(cb)` (通称 "rAF")から発行されるコールバックの中で `queueMicrotask()` や `promise.then()` などでマイクロタスクが生成されてしまった場合には１つのアニメーションタスクの後にそれらも完全に処理されるというルールがあります。

例えば、rAF のコールバックでマイクロタスクが生成されるような次のコードを考えます。

```js
requestAnimationFrame(() => {
  console.log('rAF 1');
  Promise.resolve().then(() => {
    console.log('promise in rAF 1');
  });
});
requestAnimationFrame(() => {
  console.log('rAF 2');
  Promise.resolve().then(() => {
    console.log('promise in rAF 2');
  });
});
```

これを Chrome ブラウザ環境で実行すると、以下のように１つのアニメーションタスクが実行されたらマイクロタスクが処理されるような出力を得ます。

```sh
rAF 1
promise in rAF 1
rAF 2
promise in rAF 2
```

rAF から発行されるコールバックがタスクであるということは、whatwg の仕様で名言されておらず、W３C ワーキンググループの古い仕様で示唆されているというよく分からない状況になっているそうです。とにかく、rAF のコールバックは animation task source から供給されるタスクとして処理されます。

https://404forest.com/2017/07/18/how-javascript-actually-works-eventloop-and-uirendering/
https://github.com/whatwg/html/issues/2637

つまり、「**単一タスクを実行したら、すべてのマイクロタスクを処理する**」といういつものルールがここでも適用されます。考え方としては、「**Call stack が空になったらマイクロタスクが処理される**」というように捉えた方が理解しやすいかもしれません。というわけで、ブラウザ環境におけるイベントループの擬似コードは最終的に以下のようになります。

```js:ブラウザ環境のイベントループ
// event loop v6
// 無限ループ
while (true) {
  // 複数ある Task queue から１つを選択
  queue = getNextQueue();
  task = queue.pop();
  execute(task);
  // Task は１つのみ実行する

  // Microtask queue が完全に空になるまで処理する
  while (micortaskQueue.hasTasks()) {
    doMicrotask();
  }

  // 前の描画更新から 16 ミリ秒ほど経っていれば再描画(60fps)
  if(isReapintTime()) {
    animationTasks = animationQueue.copyTasks();
    // 現時点においてキューにあるものすべてを実行する
    // Rendering pipeline 直前に確実に処理する
    for(task in animationTasks) {
      doAnimationTask(task);
      // Microtask queue が完全に空になるまで処理する
      while (micortaskQueue.hasTasks()) {
        doMicrotask();
      }
    }
    // 再描画
    repaint();
  }
}
```

次の動画は『Further Adventures of the Event Loop』の別の場所での講演の動画です。JSCnof の短い時間で語られていた内容がより詳しく語られており、「API の機能を提供する環境について学ぶ」の項目で記載したような内容について冒頭で説明されているので頭が整理されます。視聴回数も非常に少なく、あまり知られていない動画ですが、**この動画はかなりおすすめです**。

@[youtube](2qDNgBgKsXI)

この動画では、Rendering pipeline の直前に実行する別のタスクのためのキューとして Animation Frame callback queue というもの存在しており、そのキューにアニメーション用のタスクを発火する `requestAnimationFrame` という API があることが理解できます。これによって、ブラウザ環境でのレンダリングとフレームを考慮したイベントループの仕組みと、それぞれのタスクが遂行されるタイミングについて細かく理解できます。

関連して、次の MDN の記事を読んでおくことで、実行コンテキストと Call stack への理解が深まります。特に Global exectution context を理解することで最初のマイクロタスク実行のタイミングについて納得できます。これらの記事を読んでみて、個人的には、**マイクロタスクよりもタスクの方が理解の上で重要である**と感じました。

https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide

https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth

ブラウザのイベントループについては次の記事も理解に役立ちます。
https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/

JSConf.Asia 2018 での Jake Archibald 氏の講演動画である『In The Loop』ではレンダリングの話とアニメーションの話がより詳細に語られています。他の動画とは違った Event loop のイメージ(メンタルモデル)を使うので難易度が高いです。『Further Adventures of the Event Loop』を視聴してメンタルモデルがずらされないようにしてから見るのをおすすめします。

@[youtube](cCOL7MC4Pl0)

平均 16.7 ミリ秒(60 fps) の間に 1 回起きるレンダリング更新を考慮した上でタスクやアニメーション用のタスクをどうやって制御するか、Task(`setTimeout()`), Microtask(`queueMicrotask()`), animationTask(`requestAnimationFrame()`) でそれぞれ無限ループを作成したときになにがおこるかなどが理解できます。

ブラウザ環境については分かりましたが、Node 環境ではどうなるでしょうか?

『Further Adventures of the Event Loop』』では、さらにブラウザ環境から node のイベントループへと移行して解説してくれるのでどのようにそれぞれ異なるかという点がクリアになります。

```js:Node 環境のイベントループ
// 待ち状態の Task がある限りループする(なくなったら止まる)
while (tasksAreWaiting()) {
  queue = getNextQueue();
  // 次の phase (キュー) を選択

  // 各 phase (キュー) でタスクが存在している限りすべて処理する
  // 実はコールバック(task)の処理回数の最大数に制限があり、それに到達すると次のphaseに移行する
  while (queue.hasTasks()) {
    task = queue.pop();
    execute(task);

    // １つの Task を処理したら、すべての Micotasks を処理する
    // Microtask queue は２つ存在するが
    // NextTick queue の方が promise のキューよりも先に処理される
    while (nextTickQueue.hasTasks()) {
      doNextTickTask();
    }
    while (microTaskQueue.hasTasks()) {
      doPromiseTask();
    }
  }
}
```

実際にはこのコードでは Node のイベントループを理解しきれないので、より解像度をあげるために次の動画がおすすめです。

@[youtube](PNa9OMajw9w)

この動画で Bert Belder 氏によって説明されている資料は本人が公開している次の Google Drive 上ある PDF から閲覧できます。Bert Belder 氏は現在は Deno の開発に関わっているみたいですね。

https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view?resourcekey=0-lR-GaBV1Bmjy086Fp3J4Uw

Node にはフェーズが存在しているため Event loop の全体はこうなっています(上記ページより引用)。

![event loop 1](/images/js-async/img_node-event-loop-1.jpg)*各フェーズを含むイベントループ全体 (上記ページより引用)*

上図での黄色い小さい箱が、Call stack で実行される JavaScript コードです。その黄色いボックス内部について拡大して見ているのが下図で、その内部はコールバック(つまり Task) を実行した後にマイクロタスクをキューが完全に空にするまで処理するためのループとなっています。

![event loop 2](/images/js-async/img_node-event-loop-2.jpg)*各フェーズで単一タスクが実行された後のマイクロタスクの処理 (上記ページより引用)*

ただ、この動画は 2016/09/25 に公開されたのものです。従って、この動画で解説されている Node のバージョンは最大でも `6.6.0` です。つまりその時点での Node のイベントループの全体像となります。Node は v10 から v11 になるタイミングでマイクロタスク処理のタイミングがブラウザと同じタイミングにするという変更があったため、現在のバージョンと話が異なってしまっています。実際、紹介した図は v11 以上でそのまま解釈すると大筋は変わりませんが、マイクロタスクのタイミングの解釈は正確にはできません。

https://github.com/nodejs/node/pull/22842

https://github.com/nodejs/help/issues/3512

v11 未満では、phase のキューにあるすべてタスクを処理してから、マイクロタスクを空にするようになっていましたが、v11 以上では単一タスクを処理したらすべてマイクロタスクを処理するように変更されています(つまりブラウザのイベントループの仕様に近づきました)。

実際、次のように v11 以上か未満で実行順番が異なってきます。

```js
setTimeout(() => {
  console.log(1);
  Promise.resolve().then(() => {
    console.log(2);
  });
}, 100);
setTimeout(() => {
  console.log(3);
  Promise.resolve().then(() => {
    console.log(4);
  });
}, 100);
// version v11 未満の出力順番 [1 3 2 4]
// version v11 以上の出力順番 [1 2 3 4]
```

次の IBM のコースの動画では Node の Event loop について今まで見たこと無いレベルの解像度で解説してくれています(限定公開されているため URL を知らないと Youtube から直接検索してもたどり着くことができません)。

@[youtube](X9zVB9WafdE)

Node の公式ドキュメントの解説と合わせて IBM の Node に関するチュートリアルドキュメントを読むと理解が深まります。

https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#event-loop-explained

https://developer.ibm.com/tutorials/learn-nodejs-the-event-loop/#why-you-need-to-understand-the-event-loop

Node 環境では、Phase というものが存在しており、各フェーズに対応するタスクキュー(または準ずるもの)があり、一連の Phase をすべて経過することで Event loop の一周となることを理解します。

こちらも合わせて読むと Node のイベントループへの理解が深まります。
https://blog.hiroppy.me/entry/nodejs-event-loop

Node 環境は歴史的経緯から、Promise の機能を取り入れる前に、もう１つのマイクロタスクキューとして `nextTickQueue` を導入しています。そのキューにマイクロタスクを発行する `process.nextTick()` という API は現在では非推奨とはいかないまでも、代わりにデファクトスタンダードな API として `queueMicrotask()` を使用するように推奨しています。また、`setTimeout(cb, 0)` と `setImmediate(cb)` のタイマー処理を並行すると呼び出されるタイミングによっては非決定になることがあることも理解できます。

また、次の Zenn での非同期処理について記事では、イベントループについて学びはじめたときには知識不足とメンタルモデルが構築ができていなかったため、理解できませんでしたが、この段階になって読み直すと非常に頭がクリアになりました。

https://zenn.dev/qnighy/articles/345aa9cae02d9d

この記事から得られた擬似コードを以前の node の擬似コードに組み合わせることで、生成されたマイクロタスクが完全に空になることを保証するようなコードをつくることができます。

```js:Node 環境のイベントループ
// 待ち状態の Task がある限りループする(なくなったら止まる)
while (tasksAreWaiting()) {
  queue = getNextQueue();
  // 次の phase (キュー) を選択

  // 各 phase (キュー) でタスクが存在している限りすべて処理する
  while (queue.hasTasks()) {
    // 実はコールバック(task)の処理回数の最大数(システム依存)に制限があり、それに到達すると次のphaseに移行する
    if (queue.arriveMaxTasks()) break;
    task = queue.pop();
    execute(task);

    // １つの Task を処理したら、すべての Micotasks を処理する
    // Microtask queue は２つ存在するが
    // NextTick queue の方が promise のキューよりも先に処理される
    do {
      while (nextTickQueue.hasTasks()) {
        doNextTickTask();
      }
      while (microTaskQueue.hasTasks()) {
        doPromiseTask();
      }
    } while (nextTickQueue.hasTasks());
    // microTaskQueue から来たマイクロタスクが process.nextTick を使用して新しいマイクロタスクを作成した場合も完全に空になるまで処理する
  }
}
```

`nextTickQueue` のループが `microTaskQueue` のループを包んでいることがこれで理解できます。

![event loop 2](/images/js-async/img_node-event-loop-2.jpg)

ちなみに、前の図中にあったプロセス終了時 `process#exit` の地点においてはもはやイベントループに戻ることができないので、次のようなコードで `proecss.on('exit', callback)` があった際にコールバック内部で別の task を発行してもそれらは実行できません。マイクロタスクだけは実行できます。`socket.on("close", callback)` などのコールバックは Close callbasks phase で実行されるのでこれと勘違いしないようにしてください。

```js
// process の終了時に実行される
process.on("exit", () => {
  console.log("Exit: Process will exit with this code");
  // マイクロタスクは実行できるがタスクはもはやこの段階で実行できない
  queueMicrotask(() => {
    // これは実行できる
    console.log("MICROTASK: by queueMicrotask");
  });

  // この段階ではプロセスの終了をとめるためにタスクを発行しても止めることは出来ない
  // 確実に終了する
  setImmediate(() => {
    // これは実行されない
    console.log("CHECK phase task: by setImmediate");
  });
  setTimeout(() => {
    // これは実行されない
    console.log("TIMERS phase task: by setTimeout");
  });
});
```

Node でのイベントループを実際に実装するのに使われているライブラリは Libuv (**Unicorn Velociraptor**) です。従って、次の Libuv の解説動画と合わせることで、C 言語のコードでのイベントループ全体像をつかむことができます。C については詳しくありませんが、概要はつかむことができました。

@[youtube](sGTRmPiXD4Y)

いままで擬似コードで理解していましたが、実装はこうなっています。

```cpp:Node 環境のイベントループ中心部
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout, r, ran_pending;

  r = uv__loop_alive(loop);
  if (r!)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv_update_time(loop);
    // イベントループ自体の時間の更新(タイマーはこの時間との差分を取る)
    uv_run_timers(loop); // Timers phase
    ran_pending = uv__run_pending(loop); // Pending I/O callbacks phase
    uv__run_idle(loop); // Idle phase
    uv__run_prepare(loop); // Prepare phase

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv_io_poll(loop, timeout); // Poll phase
    uv_run_check(loop); // Check phase
    uv_run_closing_handles(loop); // Close callbacks phase

    if (mode == UV_RUN_ONCE) {
      uv_update_time(loop);
      uv_run_timers(loop);
    }

    // 待機中の Task が残っているかどうか確認する
    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  loop->stop_flag = 0;

  return r;
}
```

実際のコードはリポジトリ上だとこちらにあります。

https://github.com/nodejs/node/blob/c61870c376e2f5b0dbaa939972c46745e21cdbdd/deps/uv/src/unix/core.c#L378-L426

また、EventEmitter API については、イベントループ自体とは直接的には関係なく、イベントが発火された瞬間にイベントリスナーが登録された順番で同期的に呼び出されます。これはイベント発火を行う `emitter.emit(evnentName)` API 自体がそのように同期的にリスナーを呼び出すということです。

> **Synchronously calls each of the listeners registered for the event named eventName**, in the order they were registered, passing the supplied arguments to each.
> ([Events: emitter.emit(eventName[, ...args]) | Node.js v18.0.0 Documentation](https://nodejs.org/api/events.html#emitteremiteventname-args) より引用、太字は筆者強調)

もしイベント通知後に非同期的になにかしたいなら、リスナーのコールバック内部で `setTimeout()` や `setImmediate()` などのメソッドでスケジューリングして実現できます。

> The `EventEmitter` **calls all listeners synchronously in the order in which they were registered**. This ensures the proper sequencing of events and helps avoid race conditions and logic errors. When appropriate, **listener functions can switch to an asynchronous mode of operation using the `setImmediate()` or `process.nextTick()` methods**:
> ([Events: Asynchronous vs. synchronous | Node.js v18.0.0 Documentation](https://nodejs.org/api/events.html#asynchronous-vs-synchronous) より引用、太字は筆者強調)

Node については以上になります。

一方、Deno 環境のイベントループについてはほとんど情報がなく公式ドキュメントが更新されるのを待っています(変更が多いので、かなり時間がかかりそう)。Rust の Future という Promise と似た機能で Tokio ランタイムを使ってイベントループを実現しているらしいですが、現時点ではあまり情報がありません。どうやら polyfill として Node の nextTickQueue も使えるため、node との互換性は可能な限り確保されているそうです。

Rust で書かれたイベントループの最初の tick はリポジトリ上では次の場所にあります。

https://github.com/denoland/deno/blob/68bf43fca7990d4e623b66243c2840ca7f0c3628/core/runtime.rs#L840-L1001

Dynamic import やら、色々新しい処理が含まれており、なかなか理解できません。加えて色々な変更が頻繁に起こるそうで、ドキュメントの整備はまだ難しいらしいです。Deno についてはドキュメントが公開されるまでは、細かい部分については実装をみてなんとか理解するしかないという感じになりそうです。

## イベントループの共通性質 (追記)

とはいえ、**非同期処理を予測するための**、環境にとらわれないイベントループの共通性質は存在しており、次のものとなります。

> 「**Task １回の実行につき、すべての Microtask を処理する**」

このようにタスクとマイクロタスクの関係性を捉えること。これが非同期処理を予測するために最も重要で、かつ最終的な結論となります。

:::message
これが非同期処理において理解するべき結論であると考えられます。この本質がつかめれば、あとは環境に特有な非同期 API の使い方や、ECMAScript に導入される新しい非同期処理のシンタックスなどを覚えて、このモデルを元に考えればよいだけです。

この記事では公開した後から大きな追記を追加してきましたが、これ以上の本質的な部分はおそらくでてこないと思うので、修正以外の更新を終わりたいと思います。これ以上は、本質というよりも、単純に利用時の応用や注意点になると思いますし、細かい部分については Book の方などで書いていきたいと思います。
:::

ここまで、それぞれの環境でのイベントループの解像度を高めましたが、非同期処理を予測する上でのイベントループの本質的部分(**タスクとマイクロタスクの関係性**)は同じものであると想定します(実装がそうすると期待します)。

というのも Node や Deno の環境ではブラウザの実行モデルに可能な限り近づこうとしていることが issue などでよみとれます。そして、重要なこととして Chromium, Node, Deno はどれも JavaScript の V8 エンジンを搭載しています。

V8 エンジン自体は~~イベントループを実装しませんが~~、**マイクロタスクキューを所有しています**。

https://v8docs.nodesource.com/node-16.13/db/d08/classv8_1_1_microtask_queue.html

:::message alert
V8 エンジンはイベントループを実装しないと以前に書いていましたが、正確には違いましたので修正いたします。詳細については追記の「V8 エンジンから考える」を見てください。
:::

Node や Deno はイベントループをそれぞれ異なるライブラリ(Libuv, Tokio)を使って実装していますが、**マイクロタスクのチェックポイント**(マイクロタスクをいつ実行するか)については V8 を埋め込む側(Node や Deno)が決めることができます。それゆえに Node では `nextTickQueue` にあるマイクロタスクを完全に処理してから `microTaskQueue` にあるマイクロタスクを処理するように調整しており、v10 と v11 でマイクロタスクのチェックポイントを変更してブラウザにより近いイベントループ(whatwg の仕様)に変更できています。

マイクロタスクのチェックポイントは「**Call stack (Execution context stack) が空になったら**」ということが HTML Standard の「Spin the event loop」の項目で書かれているので、ランタイム環境もこれに従うように実装する、と想定します(Event loop は HTML 仕様しか存在せず、ブラウザで動く JavaScript の再利用性を可能な限り高めるため)。

https://html.spec.whatwg.org/multipage/webappapis.html#spin-the-event-loop

![spin the event loop](/images/js-async/img_spin-the-event-loop.jpg)

該当部分。

> 4. Empty the [JavaScript execution context stack](https://tc39.es/ecma262/#execution-context-stack).
> 5. [Perform a microtask checkpoint](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint).

MDN でも明確に「**タスクが終了して実行コンテキストが空になるたび、マイクロタスクキューにあるマイクロタスクが次々に実行される**」と記載されていますね。

> Each time a task exits, and the execution context stack is empty, each microtask in the microtask queue is executed, one after another.
> ([In depth: Microtasks and the JavaScript runtime environment - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth) より引用)

HMTL 仕様にブラウザ環境とランタイム環境は従うはずですから、タスクとマイクロタスクの関係性は V8 を埋め込んでいるこれらの環境では同じです(同じモデルになるように実装するはずと想定します、バグなど以外は**実際そうなっています**)。

V8 のドキュメントでもマイクロタスクは各タスクの終了時に実行され、マイクロタスクキューはイベントループに戻る前に常に空にされると記載されていました。

> On a high level there are tasks and microtasks in JavaScript. Tasks handle events like I/O and timers, and execute one at a time. Microtasks implement deferred execution for async/await and promises, and **execute at the end of each task**. **The microtask queue is always emptied before execution returns to the event loop**.
> ([Faster async functions and promises · V8](https://v8.dev/blog/fast-async#tasks-vs.-microtasks) より引用、太字は筆者強調)

そして、次の図が非同期処理の学習において、イベントループで理解すべき本質的な事象の図となっています(理解の上で非常に重要であり、非同期処理を予測するためのメンタルモデルの核心になります)。

![microtasks vs tasks](/images/js-async/img_microtasks-vs-tasks.png)*イベントループの共通部分とみなせる (上記ページより引用)*

`setTimeout()` や  `setImmediate()` は環境の提供する非同期 API であり、それらはタスクを発行し、Promise や await の処理はマイクロタスクを発行し、**単一タスクが実行された後にすべてのマイクロタスクを処理します**。これを別の言い方で言うと「**コールスタックが空になったらマイクロタスクを処理する**」となります。ブラウザ環境とランタイム環境の大きな違いは**レンダリングの作業があるかないか**です。

:::message
Node 環境では Promise が入る前にタスクを発行するコールバックベース(イベントベース)の API を基本として開発したこともあって、それらから発行されるタスクを仕分ける複数のタスクキューにプライオリティを設けるために Phase 概念を導入したと考えられます(個人的な推測です)。

Deno 環境ではタイマー以外は Promise based な API を基本としたために Node の Phase に相当するものは実質 Timers のみとなっています(これについては Discord のヘルプでコミッターに聞きました)。

ブラウザ環境でも Web API で Promise を返さない非同期 API (コールバックを渡してタスクのみを発行するタイプ)は古いタイプの API であるということが示唆されています。

> **理想的には、すべての async 関数はプロミスを返すはずですが、残念ながら API の中にはいまだに古いやり方で成功/失敗用のコールバックを渡しているものがあります**。顕著な例としては `setTimeout()` 関数があります。
> ([プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90) より引用)

`FileReader.readAsText()` などの Web API もイベントベースなので、より新しい Promise-based API として `Blob.text()` などが登場してきています。

https://developer.mozilla.org/ja/docs/Web/API/Blob/text

実は Node でも Promise-based な API を提供しはじめており、Promise based な Timer API (Promise インスタンスを返す `setTimeout()`, `setImmediate()`, `setInterval()`) なども存在しています。

https://nodejs.org/api/timers.html#timers-promises-api

マイクロタスクを発行する Promise の仕組みが非同期処理の要になってくる(というか、もうなってる?)ことは間違い無さそうです。ただし、それはタスクがなくなるということを意味しているわけではありません。`<script>` タグなどの評価はタスクですし、ユーザーインタラクションによるイベントもなくなりません。
:::

## V8 エンジンから考える (追記)

V8 エンジンはイベントループを実装しないと以前に書いていましたが、正確には違ったので追記修正します。

V8 エンジンは**実はデフォルトのイベントループを保有しており**、マイクロタスクキュー１つと少なくともタスクキューを１つ所有しています。レンダリングや phase などが無いので、最もシンプルなものとして次の疑似コードが V8 エンジンのデフォルトイベントループとして考えられます。

```js:V8エンジンのデフォルトイベントループ
while (tasksAreWaiting()) {
  // すくなくても１つ以上のタスクキューから(環境定義のルールで)１つのタスクキューを選ぶ
  queue = getNextQueue();
  // 単一タスクを処理する
  task = queue.pop();
  execute(task);

  // １つのマイクロタスクキューにあるすべてのマイクロタスクを処理する
  while (micortaskQueue.hasTasks()) {
    doMicrotask();
  }
}
```

タスクキューが実際にいくつあろうがどれか１つを選択することには違いないので `getNextQueue()` で単一のタスクキューを選び取ります。

ということで、V8 エンジンなどの JavaScript エンジンから考えることでイベントループの本質的な部分についても理解できます。

V8 エンジンを実際に環境に埋め込む際には V8 の API を使用してマイクロタスクのチェックポイントや複数のタスクキューを定めます。Node、Deno、Chrome などの実行環境では Libuv, Tokio, Libevent と Blink などのライブラリによって非同期 I/O やレンダリングの機構、複数タスクキューのスケジューリングなどの仕組みを挿入して環境独自のイベントループを実装しています。

ちなみに V8 エンジンでのコードテストは [jsvu](https://github.com/GoogleChromeLabs/jsvu) で V8 を**ローカルインストールすることでテストできるのでイベントループが存在しており、このようになっていることを確認できます**。

また、async/await についても V8 エンジンが内部的に変換しているコードを考えることで挙動について簡単に理解できます。

```js:シンプルな async 関数
async function foo(v) {
  const w = await v;
  return w;
}
```

V8 エンジンは上の async 関数を内部的に以下のようなコードへと変換しています。

```js:V8エンジンによる変換コード
// 途中で一次中断できる関数として resumable (再開可能) のマーキング
resumable function foo(v) {
  implicit_promise = createPromise();
  // (0) async 関数の返り値となる Promise インスタンスを作成

  // (1) v が Promise インスタンスでないならラッピングする
  promise = promiseResolve(v);
  // (2) async 関数 foo を再開またはスローするハンドラのアタッチ
  performPromiseThen(
    promise,
    res => resume(«foo», res),
    err => throw(«foo», err));

  // (3) async 関数 foo を一次中断して implicit_promise を呼び出し元へと返す
  w = suspend(«foo», implicit_promise);
  // (4) w = のところから async 関数の処理再開となる

  // (5) async 関数で return していた値である w で最終的に implict_promise を解決する
  resolvePromise(implicit_promise, w);
}

// 内部で使う関数
function promiseResolve(v) {
  // v が Promise ならそのまま返す
  if (v is Promise) return v;
  // v が Promise でないならラッピングして返す
  promise = createPromise();
  resolvePromise(promise, v);
  return promise;
}
```

詳細については、次の記事で書きました。

https://zenn.dev/estra/articles/asyncawait-v8-converting

# ロードマップのまとめ

- (0) 非同期処理を理解するために暗黙的に必要とされる ECMAScript のシンタックスを学ぶ
- (1) Promise や async/await といった ECMAScript の機能について理解する
- (2) Promise チェーンの構築と await 式との関係性について学ぶ
- (3) イベントループの概要とタスク/マイクロタスクについて学ぶ
- (4) **非同期 API を提供する環境について学ぶ**
- (5) (イベントループの解像度を上げて、各環境での振る舞いについて学ぶ)
- (6) イベントループの本質的な部分を捉える
- (7) V8 エンジンからイベントループや async/await の挙動を考える

:::message alert
申し訳ないですが、リアルタイムの学習内容を反映させた後半の (4) からは単純に自分の学習時系列になっているだけなので注意してください。
:::

ふりかえってみると以下の３点が重要でした。

(A) 言語機能である Promise と async/awit の関係をしっかり把握し、Promise チェーンや await 式で実行フローが分割されて制御が行ったり来たりしていることに気づくことが重要です。「同期」や「非同期」という言葉に惑わされずに、「**ブロッキング**」と「**逐次的な実行**」で考えます。

(B) 環境と絡めた非同期処理の仕組みの理解と、その目的の解釈も重要です。
「環境が提供する機能である API を介して時間のかかる処理を環境に委任し、それが完了したら JavaScript のメインスレッドにその処理結果となるデータと共に通知させてその作業に関連する何か別の作業をコールバックの形で行う」というのが非同期処理の全体的な仕組みであり、「環境が並列的にバックグラウンドで作業している間もメインスレッドで別の作業を続けられるようにすること」が非同期処理の大きな目的となります。

(C) そして非同期処理を予測するための概念と処理のモデルを手に入れることが重要です。
**イベントループの中でタスクとマイクロタスクを処理するためのモデルが頭の中で完成すること**が非常に重要です。イベントループの解像度をあげるためには色々必要な情報が多いですが、本質的な部分は非常にシンプルであり、あらゆる環境で共通であると期待できます(これが分かるために色々な情報が必要だったわけですが)。

非同期処理の学習をする際にはこれらの点に気をつけるとショートカットできると思うので、参考にしてみてください。あと「学習前予想時の４倍は難しい」ということがあながち嘘でもないことが分かったかもしれません。なるべく、ショートカットしてください😅

# 参考文献とツールのまとめ

非同期処理を理解するのに必要不可欠な道具
- [Loupe](http://latentflip.com/loupe/)
- [JS Visualizer 9000](https://www.jsv9000.app)
- すぐに JS を実行できるランタイム環境(Deno or Node)

非同期処理の基礎理解
- [JavaScript Primer - 迷わないための入門書](https://jsprimer.net/)
- [What the heck is the event loop anyway? | Philip Roberts | JSConf EU - YouTube](https://www.youtube.com/watch?v=8aGhZQkoFbQ&t=804s)
- [mdn](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Asynchronous)

勘違いを正す
- [JavaScriptのasync/await 理解してますか？ 説明できますか？ クイズに答えてもらって良いですか？ - YouTube](https://www.youtube.com/watch?v=T-_0Pc5P12U&t=36s)

ある程度自信を持って理解したと言えるようになる
- [JavaScript Promiseの本](https://azu.github.io/promises-book/)のコラム
- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)
- [MDN](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Asynchronous)

環境における非同期処理の仕組みについて理解する
- [Inside look at modern web browser (part 2) - Chrome Developers](https://developer.chrome.com/blog/inside-browser-part2/)
- [Angular Basics: Introduction to Processes, Threads—Web UI](https://www.telerik.com/blogs/angular-basics-introduction-processes-threads-web-ui-developers)
- [What the heck is the event loop anyway? | Philip Roberts | JSConf EU - YouTube](https://www.youtube.com/watch?v=8aGhZQkoFbQ&t=804s)

現代の非同期処理の機能を俯瞰する
- [Asynchrony: Under the Hood - Shelley Vohr - JSConf EU - YouTube](https://www.youtube.com/watch?v=SrNQS8J67zc)
- [JavaScript Primer - 迷わないための入門書](https://jsprimer.net/)

イベントループの解像度を上げて共通性質を捉える
- [YOW! Perth 2018 - Erin Zimmer - Adventures with the JavaScript Event Loop #YOW!Perth - YouTube](https://www.youtube.com/watch?v=2qDNgBgKsXI&list=TLGGeQ1PY_e4jiIyOTA0MjAyMg)
- [JavaScriptの非同期処理をじっくり理解する (1) 実行モデルとタスクキュー](https://zenn.dev/qnighy/articles/345aa9cae02d9d)
