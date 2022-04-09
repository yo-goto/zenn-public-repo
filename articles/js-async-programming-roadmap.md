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

この記事自体は後から別の記事で参照するかもしれませんが、具体的な話の無い気軽な内容なので、流し読み程度に見てもらえるといいかもしれません。あとは、自分が実際に学習してきた道筋に基づいているのでショートカットとして参考にしてもらったり、使えるリソースなどの情報が共有できるかもしれません。

具体的な非同期処理についての知見はいずれちょこちょこアウトプットとして書いていこうかなと思います。

## 非同期処理の感想
まず非同期処理について学んでみて、「かなり難しく、学習に時間がかかる」という印象を受けました。

全体としては、非同期処理の"制御の流れ"が掴みづらく、バックグラウンドで何が起こっているかを把握しないと自分で書こうとしたときにまったく予測ができません。非同期処理の流れ自体が掴みづらいので、同期処理と非同期処理が混じっているときは更に予測しずらくなります。加えて、非同期処理には種類があり、その種類ごとに実行のタイミングが実は異なるので、それらが入り交じることで更に予測が難しくなっています。

そもそも、「同期」と「非同期」という言葉に騙されそうになります。

局所的には Promise の概念が掴みづらく、 Promise チェーンでは特にコールバック関数がキーになるのですが、アンチパターンを学ぶことでようやく使えるレベルの理解に到達できたと感じました。

また、async/await についても、そもそも Promise チェーンを理解していないと使い物にならず、その名前から明らかに非同期処理の主役なのではないかと感じられる一方で、実体は異なるので「これは騙されるな」という印象を受けました。async/await と Promise の関係が把握できていないと致命的に混乱することになります。非同期関数が Promise インスタンスを返すこと、await 式が Promise の評価値を返し、非同期関数内の実行フローを分割すること、同期処理と非同期処理を行ったりきたりすること、それと逐次的に処理を行うことなどとの違いなど、ここらへんではトラップとなるポイントがいくつもあります。

「Promise とか await とかっては実は非同期処理ではないのでは？😵‍💫」というような疑問が途中湧き上がったりもしますが、個人的にはその疑問を解消するところがターニングポイントになるなと感じました。

基本的には、解説記事を読んで理解したつもりになっても、制御の流れをコードから想像してみたり、実際に手を動かさないと理解はかなり厳しいと感じました。ドキュメントだけで理解するのはかなり困難なので映像や可視化ツールを使うことで比較的つかみやすくなり、実際にローカル環境で実行してみることで理解でき、だんだんと制御の流れが推測できるようになってきます。

## 学習ロードマップ
非同期処理というカテゴリについては(そもそも JavaScript は)、多くの有用なリソースがインターネット上で公開されているので最大限それらを活用して学習をすすめていきます。

まずは非同期処理の前にはずせない必要な知識として以下の事柄を知っているか確認します。
- 式(Expression)
- 関数式
- コールバック関数
- 無名関数
- アロー関数と省略形
- 即時実行関数

このあたりの知識が他のドキュメントや解説を見る時に暗黙的に必要になってきます。特にコールバック関数とアロー関数は重要です。async function を使用する際には即時実行関数がよくでてきます。await を理解する際にも式の概念がキーとなるので重要です。非同期処理は JavaScript の中でも比較的高度な内容であり、こういった知識が前提とされた状態で話が進んでいきますので理解しておくのを推奨します。

不安がある場合には、JavaScript Primer で「式と文」と「関数」の項目を読んでおくのをおすすめします。

https://jsprimer.net/basic/statement-expression/

https://jsprimer.net/basic/function-declaration/

以上が理解できているなら、まず非道処理の基礎を一から学ぶために JavaScript Primer の次の項目を読みすすめましょう。

https://jsprimer.net/basic/async/

https://jsprimer.net/use-case/ajaxapp/

ただし、ドキュメントを読み終えて「完全に理解した」と思ってコードを実際に書こうとしてもかけないことに気づくので次に非同期処理のバックグラウンドでの動きを学習します。

まずは非同期処理の流れを理解に必要な用語知識を確認しておきます。
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
- Microtask と Microtask Queue
- Macrotask と Macrotask Queue

これらの用語は非同期処理の種類を認識して、バックグラウンドでの流れをある程度の解像度で理解するのに必要になってきます。非同期処理の流れ(実行順番)を把握・予測するためには必ず理解したほうがよいです。

まず、Call Stack と Event Loop を理解するために JSConf EU での Philip Roberts 氏の講演動画を見ることを強くオススメします。
@[youtube](8aGhZQkoFbQ)

この動画は色々な人がオススメしたり、多くの記事で引用されています。非常に分かりやすく説明されており、非同期処理の片方(Macrotask)を理解するのに役立ちます。

動画で概要を理解したら同氏が開発した JavaScript の可視化ツールである "Loupe" で実際に `setTimeout()` 関数のコールバック関数などがどのように動くか実験してみてください。

http://latentflip.com/loupe/

![js loupe image](/images/js-async/img_loupe-js-async.jpg)

Loupe では２種類ある非同期処理の種類の内の１つ(Macrotask: マクロタスク)しかサポートされていないので、次に Github 社のエンジニアである Andrew Dillon 氏が開発した "JavaScript Visualizer 9000" を使用して Promise 関連の非同期処理(Microtask: マイクロタスク)を理解するために使用します。

https://www.jsv9000.app

JavaScript Visualizer 9000 ではサンプルがいくつも用意されており、それらを理解するだけでも非常に有用です。二種類の非同期処理についてサポートされているので自分が書いてみた非同期処理を実際にこのアプリで実行してみて非同期処理の流れが予測どおりか確認していくことで非同期処理の流れが読めるようになってきます。ただし、async/await はサポートされていないのですそこは注意してください。

![js visualizer 9000](/images/js-async/img_js-visualizer-9000.jpg)

JavaScript Visualizer 9000 は非同期処理を理解するまで長い付き合いになったのでかなりオススメです。

ここまでくれば非同期処理がかなり理解できているはずです。きっと「非同期処理、完全に理解した」状態ですね。
では、テストとしてやっすん氏の動画でクイズをといてみてください。

@[youtube](T-_0Pc5P12U)

いかかですか?
自分の場合は「俺、Promise とか async/await のこと全然わかってねえじゃん...」となりました。

なので、ここからは実際に Promise チェーンを構築することに注力して学習をすすめていきます。また、Promise まわりの細かい疑問を調べ尽くして解消していきます。

- Microtask と Macrotask の両方を念頭に Promise チェーンを構築できるようにする
- 同期処理と非同期処理を混在させた Promise チェーンを構築できるようにする
- Promise チェーンで値を次のチェーンへと渡し、エラーハンドリングが正しく行えるようなチェーン構築をできるようにする

ここで Async function と `await` のことも考えてしまうとかなり混乱するので Promise チェーンのことに注力した方がいいかもしれません。

この段階では "JavaScript Promise の本" のコラムなどが非常に有用でした。また、mdn のドキュメントが色々な疑問を解消してくれたので主に次のページを含む非同期処理に関連するページを読んでいくのがいいと思います。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises

上記の mdn の解説でも Promise に関数るアンチパターンが紹介されていますが、下記のページにおいていくつもの Promise アンチパターンが記載されているので１つずつ理解していきます。

https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html

Promise の曖昧だったところが非常にクリアになっていくのが分かります。

さて、ここまで来れば基礎的な Promise チェーンがある程度構築できるようになったと思います。自分もこの時点で非同期 API などの使い方もある程度わかるようになりました。

さて、ようやく async/await です。特に `await` はやっかいです。非同期関数内部での await によって「これは同期処理なのか、非同期処理なのか」がよくわからなくなってきます。JavaScript Visualizer 9000 では async/await が使えないので可視化できないですので、手元でコードを書いて実行していくことをメインにして理解していきます。

自分の場合は、いままでやってきた Promies の考え方と噛み合わなかったり、Promise との組み合わせ方を理解するまでに時間がかかりました。

また、最初は async/await によって Promise がいらなくなってしまうのかとも思いましたが、そんなことは無かったです。これについては、以下の複数の Promise を扱うことのできる静的メソッドを学習することで async/await でやるよりも効率的に目的をはたせることを理解できれば体感できるはずです。

- `Promise.all()`
- `Promise.race()`
- `Promise.allSettled()` 
- `Promise.any()`

mdn で各メソッドの解説や次の記事を読むのをオススメします。それぞれの使い用途や違いなどがあきらかになると async/await でできること、やりづらいことが逆に分かってくるはずです。

https://okapies.hateblo.jp/entry/2020/12/13/154311

更にこれによって async/await が Promise を代替するのではなく、Promise ベースの非同期処理の利便性を一部向上させる Promise というシステム自体に基づいた拡張的な機能であり、Promise と一緒に使っていくものであるということが理解できると思います。

つまり、"async/await" ではなく、"Promise with async/await" 的なノリであることを理解できます。

たとえばモダンな JavaScript/TypeScript ランタイムである Deno では Node.js と比較したときに特徴として「Deno での非同期アクションはすべて Promise を返す」という点をあげています。

>All async actions in Deno return a promise. Thus Deno provides different APIs than Node.

"Async/await-based" や "Async/await-first" ではなく、"Promise-based "や "Promise-first" などの言葉を聞いたことがあると思います。その理由は "Promise" が常に非同期処理の主役だからです。

さてここまでくれば、「非同期処理の基礎が身についた」といえる状態になったのではないでしょうか。
いやー、長かったですね。あとは、解説記事なり実際になにか作ったりしてアウトプットしていくだけです。がんばって「async/await 完全に理解した」ひいては「非同期処理、完全に理解した」という状態にしていきましょう。

以上が自分なりに「ある程度、非同期処理が理解できた言える状態」まで到達するのに辿ってきたロードマップです。各項目の間で時間が空いたりして結構無駄が多かったかもしれませんが、非同期処理への納得感を達成するにはこれぐらいは必要だったかなと思います。実際は紹介していないもっと多くのリソースからも学習しているので自分の中での最短はこんな感じのイメージになりました。次の記事からは学びで得た具体的な知見をいくつか絞って書いていこうかなと思います。

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
- [JavaScript Promiseの本](https://azu.github.io/promises-book/)
- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)
- [mdn](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Asynchronous)(初歩にして真髄的なとこがある)

