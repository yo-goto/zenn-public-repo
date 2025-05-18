---
title: "それぞれのイベントループ"
cssclass: zenn
date: 2022-05-06
modified: 2024-08-14
AutoNoteMover: disable
tags: type/zenn/book, JavaScript/async
aliases:
  - Promise本『それぞれのイベントループ』
  - イベントループの疑似コード
---

## このチャプターについて

イベントループには HTML 仕様がありますが、それぞれの実行環境で少しずつ異なる部分があります。このチャプターでは各実行環境においてイベントループがどのようになっているか、擬似コードを使って理解していきます。

## イベントループの共通性質

各実行環境でのイベントループの仕組みについての解説へ入る前に、イベントループの共通性質について述べておきます。これは非同期処理の予測を行うための核心でもあるので注意を払ってください。

イベントループ(Event loop)は前のチャプターで説明したとおり、ECMAScript の範疇ではなく、HTML Living standard や実装するランタイム環境で独自に定められています。

従って、最終的には各環境ごとにイベントループがどのようになっているかということを認識する必要があります。とはいっても、タスク(Task)とマイクロタスク(Microtask)の考え方はどの環境でも基本的に同じです。

> 「**単一タスク(Task)が実行された後にすべてのマイクロタスク(Microtask)を処理する**」

『[JSの非同期処理を理解するために必要だった知識と学習ロードマップ](https://zenn.dev/estra/articles/js-async-programming-roadmap)』の追記にて、ブラウザ環境(Chrome)とランタイム環境(Node, Deno)のイベントループについての調査をまとめましたが、結局のところ本質的な部分は同じであり、ブラウザ環境が実装すべき HTML 仕様のイベントループにランタイム環境も近づくことが期待できます。そして、実際にそうなっています。

https://html.spec.whatwg.org/multipage/webappapis.html#spin-the-event-loop

ブラウザ環境と各ランタイム環境での捉え方が異なる部分は、マイクロタスクのところではなく、タスクキューのプライオリティの振り分け方であると考えられます。何をタスク(Task)として扱っているかを見極めることが重要になります。そして、それぞれの環境が提供する非同期 API がタスクを発行するものかマイクロタスクを発行するものか(Promise を返す)を見極めます。

:::message
Node 環境では Promise が入る前にタスクを発行するコールバックベース(イベントベース)の API を基本として開発したこともあって、それらから発行されるタスクを仕分ける複数のタスクキューにプライオリティを設けるために Phase 概念を導入したと考えられます(個人的な推測です)。

Deno 環境ではタイマー以外は Promise based な API を基本としたために Node の Phase に相当するものは実質 Timers のみとなっています(これについては Discord のヘルプでコミッターに聞きました)。

ブラウザ環境でも Web API で Promise を返さない非同期 API (コールバックを渡してタスクのみを発行するタイプ)は古いタイプの API であるということが[示唆されています](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)。

イベントベースの古い `FileReader.readAsText()` などの Web API に取って代わる新しい Promise-based API として `Blob.text()` なども登場してきています[^Blob.text]。

  [^Blob.text]: https://developer.mozilla.org/ja/docs/Web/API/Blob/text

実は Node でも Promise-based な API を提供しはじめており、Promise based な Timer API (Promise インスタンスを返す `setTimeout()`, `setImmediate()`, `setInterval()`) なども存在しています[^Promise-based-Timer]。

  [^Promise-based-Timer]: https://nodejs.org/api/timers.html#timers-promises-api

マイクロタスクを発行する Promise の仕組みが非同期処理の要になってくる(というか、もうなってる？)ことは間違い無さそうです。ただし、それはタスクがなくなるということを意味しているわけではありません。`<script>` タグなどの評価はタスクですし、ユーザーインタラクションによるイベントもなくなりません。
:::

「**単一タスク(Task)が実行された後にすべてのマイクロタスク(Microtask)を処理する**」という共通性質は理解の上で非常に重要ですが、これよりも実用的にマイクロタスクを捉えることのできるものがあります。

> 「**コールスタック(Call stack)が空になったらマイクロタスクを処理する**」

マイクロタスクの実行タイミングは「Microtask checkpoint」と呼ばれ、イベントループにおいてこのチェックポイントはコールスタックが空となった時に必ず実行されるように仕様で定義されています。Node や Deno といったランタイム環境では「ブラウザ環境のイベントループの仕様に近づき、マイクロタスクはコールスタックが空になったら実行されるように実装するはずである」と期待できます(そもそもイベントループの仕様は HTML 仕様しか存在せず、ブラウザで動く JavaScript の再利用性を可能な限り高めるためことが期待されます)。

https://github.com/nodejs/node/pull/22842

https://github.com/denoland/deno/issues/11731

ということで、この２つの考え方が非同期処理の理解と予測を行うためのモデルとなります。

- 「**単一タスク(Task)が実行された後にすべてのマイクロタスク(Microtask)を処理する**」
- 「**コールスタックが空になったらマイクロタスクを処理される**」

そして、重要なこととして、Chrome, Node, Deno, (Electron) といった環境は共通して V8 エンジンを JavaScript エンジンとして採用しています。V8 エンジンについて知っておくことで JavaScript の処理モデルが理解しやすくなります。V8 のドキュメントやブログなどは非常に有用なのでいくつか読んでおくと良いです(ECMAScript の新機能の使い方なども紹介されています)。

https://v8.dev/blog/fast-async#tasks-vs.-microtasks

V8 エンジンの上記ブログポストで示されているこの図が非同期処理の学習において、イベントループで理解すべき本質的な事象の図となっています(理解の上で非常に重要であり、非同期処理を予測するためのメンタルモデルの核心になります)。

![microtasks vs tasks](/images/js-async/img_microtasks-vs-tasks.png)*([Faster async functions and promises · V8](https://v8.dev/blog/fast-async#tasks-vs.-microtasks)より引用)*

:::message
今は理解できなくても構いませんので、後から参照してもらって大丈夫です。ですが、この項目の説明がこの本において理解すべき最も重要な話になるので注意してください。
:::

ブログ記事では次のようにも語られています。

> On a high level there are tasks and microtasks in JavaScript. Tasks handle events like I/O and timers, and execute one at a time. Microtasks implement deferred execution for async/await and promises, and **execute at the end of each task**. **The microtask queue is always emptied before execution returns to the event loop**.
> ([Faster async functions and promises · V8](https://v8.dev/blog/fast-async#tasks-vs.-microtasks)より引用、太字は筆者強調)

非同期処理の仕組みの核心として、`setTimeout()` や  `setImmediate()` は環境の提供する非同期 API であり、それらはタスクを発行する一方、Promise や await の処理はマイクロタスクを発行し、**単一タスクが実行された後にすべてのマイクロタスクを処理します**。これを別の言い方で言うと「**コールスタックが空になったらマイクロタスクを処理する**」となります。ブラウザ環境とランタイム環境の大きな違いは**レンダリングの作業があるかないか**です。

## イベントループのメンタルモデル

`setTimeout` や Promise chain がイベントループでどのように動くかをイメージできることは非常に重要です。頭の中にイベントループのモデルが完成することで非同期処理の制御の流れを把握し、予測できるようになります。ただし、このモデルを「完成」させるには多くの情報が必要であり、適宜修正していく必要があります。

修正していく必要があるのは、インターネット上にあふれるモデル化された図像(視覚化されたイメージ)などが基本的に「何かが足りていない」あるいは「何かが間違っている」というものが非常に多く、筆者自身もかなり多くの画像を見てきましたが実行順序を理解するために必要な要素がすべて揃っているものは**殆ど存在していない**のではないかという感想です。イベントループ自体も進化しているところがあるので古いバージョンの図像がそのままいまでも使われていることがよくあります。

基本的には『What the heck is the event loop anyway?』で解説されている図像を基準にしてけば良いと思うのですが、(動画自体が古いこともあって)これも古いもので Promise のために挿入されたマイクロタスクキューの要素がありません。ちなみにこの動画で紹介されている Philip Roberts 氏が自身で作った Loupe という視覚化ツールで `setTimeout` などの動きを学ぶことができます。

http://latentflip.com/loupe/

![loupe](/images/js-async/img_loupe-js-async.jpg)

そしてこの本の途中で何度か使用している JavaScript Visualizer 9000 では、Promise のためのマイクロタスクキューの要素が追加されたイベントループを表現している、Loupe の後継といっていいレベルの完成度を誇っています。

https://www.jsv9000.app

![visualizer](/images/js-async/img_js-visualizer-9000.jpg)

ですが、この本の冒頭書いたように実装ミスと思われる部分や Web APIs の要素がないのでこれ単体でメンタルモデルにしてしまうのが非常に危ういのです(勘違いする可能性のある部分がいくつかあるため)。

Loupe であった Web API の要素(複数同時に実行できるため実質的にスレッドが存在していることを)を Visualizer 9000 に追加してモデルを修正しないといけません。グローバルコンテキストについては Loupe でも Visualizer でも無いためこの要素も追加する必要があります。

ベースとして Loupe と JS Visualizer の両方を組み合わせた上で足りない要素をモデルに追加したり、実装ミスである部分を修正していくのが最短だと考えています。そのために擬似コードやそれぞれの要素に注視しながらモデルを構築します。

## イベントループの仕組み

共通性質やメンタルモデルの注意点について頭に入れておきましたので、より具体的にイベントループがどのようになっているかを見ておきましょう。イベントループについてはテキストで理解するよりも動画で理解した方がいい場合もあるので、こちらの動画を参考にしてください。

https://youtu.be/2qDNgBgKsXI

この動画は、[JSConf EU 2018 での Erin Zimmer 氏が行った講演動画である『Further Adventures of the Event Loop』](https://youtu.be/u1kqx6AenYw)の別の場所での講演動画です。JSConf の短い時間で語られていた内容がより詳細に語られており、非常にわかりやすいので是非視聴することをおすすめします。

それではこの動画を参考にしてイベントループについて考えていきたいと思います。イベントループの本質は上述したように実はかなりシンプルですが、それが理解できるようになるまでは疑似コードで考えた方が分かりやすいです。

### レンダリングエンジンとメッセージループ

ブラウザ環境でのイベントループの実装について少し話しておきます。

Chrome ブラウザ環境のイベントループの実装については実はほとんど情報がありません。Chrome ブラウザで利用されている JavaScript エンジンである V8 はデフォルトのイベントループを提供していますが、基本的に外部からプラグインして実装するようになっています。

Chrome ブラウザ環境では Libevent というライブラリが実際のイベントループの実装に使われているようです。

https://libevent.org

イベントループはブラウザ環境(Chrome) の文脈ではメッセージループ(Message loop)と呼ばれることがあり、「Message loop はレンダリングエンジンである Blink (Chrome の場合)が実装した上でそれをイベントループとしている」という情報もあるのですが、Libevent との情報をあわせると Task queues から単一のタスクキューを選択して Task のスケジューリングをするのが Blink で、実際のイベントループは Libevent が使用されていると考えられます。要するに協調して動作しているわけです。

https://docs.google.com/document/d/11N2WTV3M0IkZ-kQlKWlBcwkOkKTCuLXGVNylK5E2zvc/edit

> The main purpose of the scheduler is to decide which task gets to execute on the main thread at any given time. To enable this, the scheduler provides higher level replacements for the APIs that are used to post tasks on the main thread.
> ([Blink Scheduler](https://docs.google.com/document/d/11N2WTV3M0IkZ-kQlKWlBcwkOkKTCuLXGVNylK5E2zvc/edit) より引用)

コードベース上にあるドキュメントは以下です。
- [Blink Scheduler](https://chromium.googlesource.com/chromium/src/+/master/third_party/blink/renderer/platform/scheduler/README.md)
- [Scheduling docs](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/platform/scheduler/links.md)

Chrome ブラウザ環境では、環境実装のルールとしてどのようにタスクキューを優先するかを Blink scheduler によって選択させているようです。内部的にどのような順位になっているかを知りたい場合は上記ドキュメントを参照してください。

https://docs.google.com/a/google.com/document/d/1SWpjgtwIaL_hIcbm6uGJKZ8o8R9xYre-yG0VDOjFBxU/edit

https://nhiroki.jp/2017/12/10/javascript-parallel-processing#1-%E3%83%AC%E3%83%B3%E3%83%80%E3%83%AA%E3%83%B3%E3%82%B0%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%B3%E3%81%A8-javascript-%E3%81%AE%E5%AE%9F%E8%A1%8C%E3%83%A2%E3%83%87%E3%83%AB

レンダリングエンジンである Blink は実は Chrome (Chromium) の中で非常に大きな役割を持っています。Blink は JavaScript エンジンである V8 エンジンを管理する上位コンポーネントとしてブラウザタブ内のコンテンツレンダリングに関わる全てを実装しており、具体的には以下のようなことを行っています。

- DOM、CSS、WebIDL などを含む Web プラットフォーム仕様の実装
- V8 エンジンを埋め込み JavaScript を実行
- 基盤となるネットワークスタックからのリソースリクエスト
- DOM ツリーの構築
- スタイルとレイアウトの計算
- Chrome コンポジターの埋め込みとグラフィックの描写

Blink が実際に行っていることについては以下の Google ドキュメントに概要が記載されています。

https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg/

Web API のインターフェース仕様を記述する WebIDL などを実装していることから、Blink エンジン内で Promise-based API である `fetch` API などの実装も行われています。`fetch` API の実装はコードベース的には以下の場所にあります。

https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/fetch/

Chrome (Chromium) ブラウザが提供する API 機能の実装は主に以下の２つのディレクトリ内に存在しています。C++ コードを読めるならこれらのディレクトリにあるコードから実装を理解できます。

- [third_party/blink/renderer/core/](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/)
- [third_party/blink/renderer/modules/](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/)

話をイベントループに戻すと、ブラウザ環境でのイベントループでは、Node や Deno といったランタイム環境にはない、Blink 等の**レンダリングエンジンの存在があるため、レンダリングの作業そのものを考慮する必要があります**。逆にランタイム環境では、レンダリングのタスクが存在しないためシンプルになりますが、Node ではタスクの優先度をより細かくするなどの違いがあります。

:::message
イベントループの共通性質で述べたように、ブラウザ環境とランタイム環境のイベントループの大きな違いは「**レンダリングの作業があるかないか**」です。
:::

実装に関する細かいことは置いておいて、イベントループの疑似コードについて考えていきます。

### MDN のイベントループ

まずは、MDN のドキュメントでイベントループがどのように語られているか見てみます。イベントループの疑似コードは次のようになっています。

```js:MDN のイベントループ
while (queue.waitForMessage()) {
  queue.processNextMessage()
}
```

ドキュメントには以下のように解説されています。

> `queue.waitForMessage()` waits synchronously for a message to arrive (if one is not already available and waiting to be handled).
> ([The event loop - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop#event_loop) より引用)

待ち状態のタスク(メッセージ)がある限りそのタスクを処理しつづけるというループになっていますが、これは簡略化しすぎているので、非同期処理については何も分かりません。ですが、イベントループというものは本質的には、このようにタスクを処理するための半無限ループであることを理解しておくとよいです。そして、「**メッセージの通知**」を待つためのループであることを覚えておいてください。

:::message
ブラウザ環境の場合はタブなどを閉じない限りは無限にループしますが、ランタイム環境でコンソールからファイルを実行した場合には処理待ちのタスクが無くなり次第プログラムは終了、つまりイベントループから脱出します。
:::

### ブラウザ環境のイベントループ

ここで、レンダリングを含めたブラウザ環境でのイベントループを知るために、『Further Adventures of the Event Loop』の内容を主に参考にして考えてみます。

ブラウザ環境において、HTML ファイルに次のようなスクリプトがあった場合、ブラウザはスクリプトをパースして評価し、同期処理の部分は最初のタスク(Task)として実行されます。

:::message alert
「最初のタスク」といっても本当に最初ではありませんので注意してください。スクリプトの実行を考える上での「最初のタスク」となります。
:::

```html
<script>
  // <- 最初の Task
  const foo = bar;
  foo.doSomething();

  document.body.addEventListener('keydown', (event) => {
    // <- イベントを受信したら発火される Task
    if (event.key === 'PageDown') {
      location.href = "/#/36";
    }
    console.log("Event is triggered");
    // イベントを受信したら発火される Task ->
  });
  // 最初の Task ->
</script>
```

`addEventListener()` は Web API のメソッドであり、そのコールバック部分はブラウザがキーダウンイベントを受け取った際に、別のタスクとして実行されます。つまり、タスクキュー(Task queue)へと送られた後に、コールスタックへと運ばれて JavaScript として実行されるわけです。

```js:イベントループ v1
// 無限ループ
while (true) {
  task = taskQueue.pop();
  execute(task);
}
```

上述したとおり、タブを閉じるなどしない限りブラウザ環境では無限にループが回り続けます。

#### 複数のタスクキュー

『[タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)』のチャプターで解説した通り、HTML 仕様においてイベントループは１つ以上のタスクキューを持ちます。

> An event loop has one or more task queues. A task queue is a set of tasks.
> ([Event loops | HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3) より引用)

そして以下のものがタスクとして扱われます。

- イベント(Event)
- パース(Parsing)
- コールバック(Callbacks)
- リソースの使用(Using a resource)
- DOM 操作への反応(Reacting to DOM manipulation)

そしてタスクキューで重要なルールは以下のものでした。

- (1) 複数あるタスクキューはどの順番に処理されるか決められていない
- (2) 同一のタスクキュー内に存在しているタスクは到着した順番に処理される
- (3) 同一の供給源から来たタスクは同じタスクキューへと送られる

実際、`setTimeout()` API のコールバックなどとマウスクリックによって発火するイベントによるコールバックなどは別のタスクソースから別々のタスクキューへと送られます。上述した通り、レンダリングエンジンである Blink がタスクキューを選択してタスクをスケジューリングします。基本的にはユーザーインタラクションやマウスクリックなどのイベントを優先的に処理するようになっています。

そういった具体的な優先度は考慮せずに、とりあえず複数のタスクキューが存在しており、タスクキューから１つタスクをとって実行するモデルをイベントループの疑似コードで考えますと次のようになります。

```js:イベントループ v2
// 無限ループ
while (true) {
  queue = getNextQueue();
  // 複数ある Task queue から１つを選択
  task = queue.pop();
  execute(task);
  // task はループにつき一個のみが処理される
}
```

これで複数のタスクキューからタスクキューを選んで単一タスクを処理するモデルができましたが、肝心のマイクロタスクがありません。

#### マイクロタスクキュー

マイクロタスクキューは単一タスクが処理された後にすべてのマイクロタスクが処理されるという話でしたから、イベントループにはマイクロタスクをすべて処理するためのループが必要となります。

```js:イベントループ v3
// 無限ループ
while (true) {
  queue = getNextQueue();
  // 複数ある Task queue から１つを選択
  task = queue.pop();
  execute(task);
  // task はループにつき一個のみが処理される

  // すべての Microtask を処理するためのループ
  while (microtaskQueue.hasTasks()) {
    doMicrotask();
  }
}
```

これで Promise chain などでマイクロタスクからマイクロタスクが生み出されたとしても、すぐに処理されるようになりました。

従って、次のようにマイクロタスクを無限に生み出すループを作成して実行すると何もできなくなります。テキスト選択やボタンクリックもできなくなります。

```js
function microTaskLoop() {
  Promise.resolve().then(microTaskLoop);
}

microTaskLoop();
```

タスクの場合、そのようなことは起きません。

```js
function taskLoop() {
  setTimeout(taskLoop, 0);
}

taskLoop();
```

クリックやテキスト選択も可能です。これは上述したとおり、レンダリングエンジンが `setTimeout()` のコールバックが送られるタスクキューとは別に、ユーザーインタラクション用のタスクキューに送られたタスクを優先して処理するようにタスクキューを選択するからです。

JSConf.Asia での Jake Archibald 氏による講演動画『In The Loop』において、タスク、マイクロタスク、アニメーションタスク(`requestAnimationFrame()` API のコールバック)で無限ループを作成した場合の比較を行っているので、詳しくはこちらを参考してください。

https://youtu.be/cCOL7MC4Pl0

#### レンダリングパイプライン

ブラウザ環境ではランタイム環境には無いレンダリング更新の作業があります。

タスクキューはレンダリングパイプライン(**Rendering pipeline**)の混雑の中で機能しています。レンダリングパイプラインはブラウザウィンドウの描画の責務を持ちます。DOM の変更や CSS のスタイル更新を行った場合、このレンダリングパイプラインによって画面に表示されます。

レンダリングパイプラインとは、簡単に言えばレンダリングの更新のための色々な処理のことです。以下のようなステップでレンダリング更新が行われます。

![Rendering Pipeline](/images/js-async/img_renderingPipeline.jpg)*[Rendering Performance](https://web.dev/rendering-performance/)を参考に筆者作成*

図中の最初の rAF は非同期 Web API である `requestAnimationFrame()` のコールバック関数となります。

https://web.dev/rendering-performance/#2.-js-css-greater-style-greater-paint-greater-composite

もう少し広範囲の視点で考えるとレンダリングパイプラインは HTML ファイルからスクリーン上までのピクセル描写までの工程と言えます。Blink のドキュメントでは以下のような工程として説明されています。Paint 後の Composite 作業については Main thread から同じ Renderer process 内の Compositor thread に移行して描画まで処理が行われます。

![Blink内のレンダリングパイプライン](/images/js-async/img_rendering-pipeline-in-blink.jpg)*[How Blink works - Google ドキュメント](https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg/edit#) より引用*

レンダリングパイプラインの各工程についてのより具体的な解説は以下のページやスライドなどから参考にしてください。

https://developer.chrome.com/blog/renderingng-architecture/

https://docs.google.com/presentation/d/1boPxbgNrTU0ddsc144rcXayGA_WF53k96imRH8Mp34Y/edit#slide=id.p

JavaScript はシングルスレッド言語であり、ブラウザ環境でユーザーの JavaScript コードはメインスレッド(Main thread)で実行されます。上述した通り、ブラウザ環境ではレンダリングの作業があります。そしてレンダリングのための作業もメインスレッドで行われます。従ってその作業を行っている間はそのスレッドで JavaScript を実行できません。

レンダリング更新は 1/60 秒で四捨五入して 16.7 ミリ秒 (60fps) 平均で行われます。つまり上記のレンダリングパイプラインが 16.7 ミリ秒ごとに以下の図のようにメインスレッドで発生します。

![Rendering pipeline 60 fps](/images/js-async/img_renderingPipelineFrames.jpg)*[In The Loop](https://www.youtube.com/watch?v=cCOL7MC4Pl0)を参考に筆者作成*

:::message
レンダリングの更新の間隔は 60Hz のリフレッシュレートモニターで平均 16.7 ミリ秒となりますが、一般的にブラウザは W3C の勧告に従って、レンダリング更新をディスプレイのリフレッシュレートに合わせて行います。したがって、リフレッシュレートが 144Hz や 240Hz ならレンダリング更新はそれぞれ一秒間に 144 回の更新、240 回の更新となり、rAF のコールバック関数もそれに合わせた分だけ呼び出されることになります。

参考: [Window.requestAnimationFrame() - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Window/requestAnimationFrame)
:::

各レンダリングパイプラインの発生までの間隔でユーザーの JavaScript コード(タスクとマイクロタスク)をメインスレッドで実行できます。ただし、16.7 ミリ秒以上かかるタスクなどがあればレンダリング更新がおくれてしまいフレームが落ちることになるので注意する必要があります。

結局、イベントループの疑似コードは次のようになります。

```js:イベントループ v4
// 無限ループ
while (true) {
  queue = getNextQueue();
  task = queue.pop();
  execute(task);
  // task はループにつき一個のみが処理される

  // すべての Microtask を処理するためのループ
  while (microtaskQueue.hasTasks()) {
    doMicrotask();
  }

  // 前の描画更新から 16.7 ミリ秒ほど経っていれば再描画
  if(isRepaintTime()) repaint();
}
```

#### requestAnimationFrame API

先に出てきましたが、レンダリングパイプライン直前にのみ確実に実行できる `requestAnimationFrame()` という Web API が存在しています。長いので "rAF" と略されて表記される事が多いです。

https://developer.mozilla.org/ja/docs/Web/API/Window/requestAnimationFrame

この API では、引数として渡したコールバック関数はレンダリング更新のタイミングでしか実行できません。アニメーションなどに使うことができます。

```js
requestAnimationFrame(() => {
  // animationTask
});
```

実はこの Web API 用に "Animation Frame Callback Queue" というタスクキューが存在しています。このタスクキューに送られるタスクはアニメーションタスクなどと呼ばれることもあります。rAF のコールバックがタスクとなることは HTML 仕様では明確に述べられていませんが、W3C ワーキンググループの古い仕様で読み取れる部分があります。いずれにせよ便宜的にタスクとして考えます。

https://github.com/whatwg/html/issues/2637

このアニメーションタスクはタスクでありマイクロタスクではありません。イベントループの一周において、その時点で Animation Frame Callback Queue にある分のタスクすべてを処理します。処理された rAF のアニメーションタスクから更にアニメーションタスクが生み出されてしまった場合は、次のループで処理されることに注意してください。

という訳でイベントループの疑似コードは次のように更新されます。

```js:イベントループ v5
// 無限ループ
while (true) {
  queue = getNextQueue();
  task = queue.pop();
  execute(task);
  // task はループにつき一個のみが処理される

  // すべての Microtask を処理するためのループ
  while (microtaskQueue.hasTasks()) {
    doMicrotask();
  }

  // 前の描画更新から 16.7 ミリ秒ほど経っていれば再描画
  if(isRepaintTime()) {
    animationTasks = animationQueue.copyTasks();
    // 現時点においてキューにあるものすべてを実行する
    for(task in animationTasks) {
      doAnimationTask(task);
    }
    // 再描画
    repaint();
  }
}
```

ただしアニメーションタスクはタスクなので、コールバック関数内でマイクロタスクが生み出されてしまった場合は「単一タスクを実行したらすべてのマイクロタスクを処理する」というルールがここでも適用されます。

例えば、次のように rAF のコールバックからマイクロタスクを生み出すコードを考えます。

```js
requestAnimationFrame(() => {
  console.log('rAF 1');
  Promise.resolve().then(() => {
    console.log('promise in rAF 1');
  })
  .then(() => {
    console.log('promise in rAF 2');
  });
});
requestAnimationFrame(() => {
  console.log('rAF 2');
  Promise.resolve().then(() => {
    console.log('promise in rAF 3');
  });
});
```

このコードをブラウザ環境で実行すると、以下のように１つのアニメーションタスクが実行されたらマイクロタスクをすべて処理するというような出力を得ます。

```sh
rAF 1
promise in rAF 1
promise in rAF 2
rAF 2
promise in rAF 3
```

ということで、イベントループの疑似コードは次のように更新されます。

```js:イベントループ v6
// 無限ループ
while (true) {
  // 複数ある Task queue から１つを選択
  queue = getNextQueue();
  task = queue.pop();
  execute(task);
  // Task は１つのみ実行する

  // Microtask queue が完全に空になるまで処理する
  while (microtaskQueue.hasTasks()) {
    doMicrotask();
  }

  // 前の描画更新から 16.7 ミリ秒ほど経っていれば再描画
  if(isRepaintTime()) {
    animationTasks = animationQueue.copyTasks();
    // 現時点においてキューにあるものすべてを実行する
    // Rendering pipeline 直前に確実に処理する
    for(task in animationTasks) {
      doAnimationTask(task);
      // Microtask queue が完全に空になるまで処理する
      while (microtaskQueue.hasTasks()) {
        doMicrotask();
      }
    }
    // 再描画
    repaint();
  }
}
```

#### 最終的なイベントループ

結局のところ、最終的なイベントループは次のようになります。

```js:ブラウザ環境のイベントループ(v6)
while (true) {
  queue = getNextQueue();
  task = queue.pop();
  execute(task);

  while (microtaskQueue.hasTasks()) {
    doMicrotask();
  }

  if(isRepaintTime()) {
    animationTasks = animationQueue.copyTasks();
    for(task in animationTasks) {
      doAnimationTask(task);
      while (microtaskQueue.hasTasks()) {
        doMicrotask();
      }
    }
    repaint();
  }
}
```

所詮は疑似コードなので、上記以外にもいろいろな書き方がありえます。ブラウザ環境のイベントループについてのより仕様に近い疑似コードは以下のリポジトリの解説などが参考になります。

https://github.com/atotic/event-loop

### Web workers のイベントループ

Web Worker API を使うことで、環境が API として提供していないような時間のかかる同期関数などを別スレッドで並列に走らせることができます。

https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers

そして、Web worker はメインスレッドとは別にそれぞれ独自のイベントループ、コールスタック、タスクキュー、マイクロタスクキューなどを持ちます。

ブラウザ環境と比べて次のような特長があります。

- Script タグが無い
- ユーザーインタラクションが無い
- DOM 操作が無く、アニメーションも無い

というわけで、ブラウザのメインスレッドのイベントループよりも簡単です。

Web worker では post-message event の送受信を行い、ブラウザウィンドウのメインスレッドとコミュニケーションを取ります。マイクロタスクキューもあるので Promise が使えます。疑似コードは以下のようになります。

```js:Web worker のイベントループ
// 無限ループ
while (true) {
  queue = getNextQueue();
  task = queue.pop();
  execute(task);

  while (micortaskQueue.hasTasks()) {
    doMicrotask();
  }
}
```

非常にシンプルです。いつもの「単一タスクを処理したらすべてマイクロタスクを処理する」というルールです。

### Node 環境のイベントループ

Node 環境では、直感に反してブラウザ環境よりもシンプルなものとなります。以下のように、ブラウザ環境では存在していたものが Node 環境には存在しません。

- スクリプトのパースイベントが存在しない
- やっかいなユーザーインタラクションが存在しない(マウスクリックなど)
- Animation frame callback が存在しない
- レンダリングパイプラインが存在しない

Node 環境のイベントループの実装は、Libuv (**Unicorn Velociraptor**) という非同期ランタイムのライブラリが担当しています。Node のイベントループの特長は以下となります。

- Promise 用のものに加えてもう１つ `nextTickQueue` というマイクロタスクキューが存在していおり、そのキューにあるマイクロタスクは Promise のマイクロタスクキューよりも先に処理される
- 複数のタスクキューが存在しており、各タスクキューはそれぞれのフェーズ(Phase)に結びついており、イベントループはそれぞれの 6 つのフェーズを経ることでイベントループの一周となる
- `setImmediate()` という Web 非互換のタイマーがあり、I/O サイクル内では `setTimeout()` のコールバックとの比較では `setImmediate()` のコールバックの方が先に処理される

Node 環境のイベントループに上で述べたようにフェーズ(Phase)という概念が導入されています。このフェーズはそれぞれをタスクキューであると考えてください(実際にはキューでないものもある)。

https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#event-loop-explained

![Node phases](/images/js-async/img_nodePhase.jpg)*[上記ページ](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)より引用*

フェーズ(Phase)は上図のようになっており、イベントループはこの６つのフェーズ(タスクキュー)をすべて経ることでイベントループ一周とします。ただ上の図は非常に分かりづらいのでもう少し情報が必要でしょう。

Node のイベントループについては以下の Node Interactive EUROPE での Bert Belder 氏による講演動画で詳しく解説されています。

https://youtu.be/PNa9OMajw9w

この動画で Bert Belder 氏によって説明されている資料は本人が公開している次の Google Drive 上ある PDF から閲覧できます。Bert Belder 氏は Deno の開発に関わっているみたいですね。

https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view?resourcekey=0-lR-GaBV1Bmjy086Fp3J4Uw

この講演で紹介されているイベントループの全体像は以下のようになっています。

![Node-event-loop](/images/js-async/img_node-event-loop-1.jpg)*[2016 Node Interactive.pdf](https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view?resourcekey=0-lR-GaBV1Bmjy086Fp3J4Uw) より引用*

上図での黄色い小さい箱が、Call stack で実行される JavaScript コードです。その黄色いボックス内部について拡大して見ているのが下図で、その内部はコールバック(つまり Task) を実行した後にマイクロタスクをキューが完全に空にするまで処理するためのループとなっています。

![Node event loop2](/images/js-async/img_node-event-loop-2.jpg)*[2016 Node Interactive.pdf](https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view?resourcekey=0-lR-GaBV1Bmjy086Fp3J4Uw)より引用*

ただし、この動画は 2016/09/25 に公開されたのものです。従って、この動画で解説されている Node のバージョンは最大でも v6.6.0 です。つまりその時点での Node のイベントループの全体像となります。Node は v10 から v11 になるタイミングでマイクロタスク処理のタイミングがブラウザと同じタイミングにするという変更があったため、現在のバージョンと話が異なってしまっています。実際、紹介した図は v11 以上でそのまま解釈すると大筋は変わりませんが、マイクロタスクのタイミングの解釈は正確にはできません。

https://github.com/nodejs/node/pull/22842

https://github.com/nodejs/help/issues/3512

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

これの意味していることは、「Node 環境はブラウザ環境のイベントループに近づいた」ということです。つまり、「**単一タスクを実行したらすべてのマイクロタスクを処理する**」という重大ルールが適用できるということです。これによってブラウザ環境と基本的に同じものとして考えることができます。

従って、Node 環境のイベントループの疑似コードは以下のように考えることができます。

```js:Node 環境のイベントループ(v1)
// Node event loop v1
// 待機状態のタスクがなくなったらイベントループを脱出する
while (tasksAreWaiting()) {
  queue = getNextQueue();
  // phase (タスクキュー) を１つ選択

  // phase (タスクキュー) でタスクが存在しているかどうか
  while (queue.hasTasks()) {
    task = queue.pop();
    execute(task);

    // １つの Task を処理したら、すべての Microtasks を処理する
    while (microTaskQueue.hasTasks()) {
      doPromiseTask();
    }
  }
}
```

しかし、これでもまだ情報が足りていません。

Node 環境にはもう１つのマイクロタスクキューである `nextTickQueue` が存在していることに気をつけてください。この `nextTickQueue` にマイクロタスクを送るには `process.nextTick()` API を利用します。ただし、Node 環境は歴史的経緯から、Promise の機能を取り入れる前に、`nextTickQueue` を導入しました。`process.nextTick()` API は現在では非推奨とはいかないまでも、今では代わりにデファクトスタンダードな API として `queueMicrotask()` を使用するように推奨しています。

つまり、Node 環境ではマイクロタスクのためのキューが以下の二種類となります。

- nextTickQueue
- microTaskQueue

`nextTickQueue` にあるマイクロタスクは Promise 用のマイクロタスクキューよりも先に処理されます。例えば、次のコードを考えます。

```js
// queueMicroVsNextTick.js
console.log("[1] 🦖 MAINLINE: Start");

(function main() {
  setTimeout(() => {
    console.log("[7] ⏰ TIMERS: Task")
  });
  Promise.resolve().then(() => {
    console.log("[4] 👦 MICRO: [microTaskQueue] then");
  });
  queueMicrotask(() => {
    console.log("[5] 👦 MICRO: [microTaskQueue] queueMicrotask");
  });
  process.nextTick(() => {
    console.log("[3] 👦 MICRO: [nextTickQueue] process.nextTick");
    queueMicrotask(() => {
      console.log("[6] 👦 MICRO: [microTaskQueue] queueMicrotask");
    });
  });
})();

console.log("[2] 🦖 MAINLINE: End");
```

`Promise.resolve().then()` のコールバックと `queueMicrotask()` API のコールバックは同じ `microTaskQueue` へと送られる一方、`process.nextTick()` API のコールバックは `nextTickQueue` へと送られます。

単一タスクが実行されたら、すべてのマイクロタスクが処理されるわけですから、もし `process.nextTick()` のコールバックで `microTaskQueue` へのマイクロタスクを生み出した場合もそれらすべてのマイクロタスクを処理します。

従って、上記コードの出力は次のようになります。

```sh
❯ node queueMicroVsNextTick.js
[1] 🦖 MAINLINE: Start
[2] 🦖 MAINLINE: End
[3] 👦 MICRO: [nextTickQueue] process.nextTick
[4] 👦 MICRO: [microTaskQueue] then
[5] 👦 MICRO: [microTaskQueue] queueMicrotask
[6] 👦 MICRO: [microTaskQueue] queueMicrotask
[7] ⏰ TIMERS: Task
```

より短いコードで見てみると次のようになります。

```js:nodeTimer.js
setImmediate(() => console.log("[6]")); // コールバックをタスクとして発行
setTimeout(() => console.log("[5]")); // コールバックをタスクとして発行
queueMicrotask(() => console.log("[3]")); // コールバックをマイクロタスクとして発行
Promise.resolve().then(() => console.log("[4]")); // コールバックをマイクロタスクとして発行
process.nextTick(() => console.log("[2]")); // コールバックをマイクロタスクとして発行
console.log("[1]"); // 同期処理
```

```sh
❯ node nodetimer.js
[1]
[2]
[3]
[4]
[5]
[6]
```

さらに注意点として、１つのフェーズでは特定数の Task が実行されて、次のフェーズに行きます。すべてではなく特定数(最大制限)があるのは、１つのフェーズのタスク(Task)が多すぎると次のフェーズにいつまでも移行できなくなるからです。では実行されずに残されたタスクはどうなるかというと一旦保留にして、次のイベントループにおいて実行されます。ただし、タスクだけは常に完全にキューが空になるまで実行されます。

従って、Node 環境のイベントループの疑似コードは最終的に次のようになります。

```js:Node 環境のイベントループ(v2)
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

    // １つの Task を処理したら、すべての Microtasks を処理する
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

ちなみに以下の図にあるプロセス終了時 `process#exit` の地点においてはもはやイベントループに戻ることができないので、次のようなコードで `process.on('exit', callback)` があった際にコールバック内部で別のタスクを発行してもそれらは実行できません。マイクロタスクだけは実行できます。ただし、`socket.on("close", callback)` などのコールバックは Close callbacks phase で実行されるのでこれと勘違いしないようにしてください。

![Node event loop](/images/js-async/img_node-event-loop-1.jpg)*[2016 Node Interactive.pdf](https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view?resourcekey=0-lR-GaBV1Bmjy086Fp3J4Uw)より引用*

例えば、次のコードで `process.on("exit", callback)` の引数として渡したコールバック関数において、マイクロタスクを処理することはできますが、タスクは処理できないので注意してください。

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

Node のイベントループの詳細については以下の IBM のチュートリアルがおすすめです。両方合わせて参考にしてください。

https://www.youtube.com/watch?list=TLGGmD0fij1sF90wNTA1MjAyMg&v=X9zVB9WafdE&feature=emb_imp_woyt

https://developer.ibm.com/tutorials/learn-nodejs-the-event-loop/#why-you-need-to-understand-the-event-loop

#### イベントループの実装

Node でのイベントループを実際に実装するのに使われているライブラリは Libuv でした。次の NodeConf EU で開催された講演動画である『A deep dive into libuv』において Libuv のイベントループについての解説がされています。この講演で紹介されている C 言語コードでのイベントループの全体像をつかむことができます。C 言語については詳しくなくても、ある程度の雰囲気はつかむことができると思います。

https://youtu.be/sGTRmPiXD4Y

これまで擬似コードで理解していましたが、実装は次のようになっており、Phase のサイクルがコードで実現されていることが分かります。また、タイマーでのポーリング機構で時間のデルタを取るという処理のための時間更新処理が含まれています。

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

実際のコードはリポジトリ上だとこちらにあります。現時点(2023/02/08)での最新版イベントループです。Node のイベントループの実装について詳しく知りたい場合にはこのファイルを調査してください。

https://github.com/nodejs/node/blob/28bf0317c2e2e745f38275d930e3ead9e301ee1a/deps/uv/src/unix/core.c#L382-L448

#### タイマーの比較

Phase では Timers が最初で Check よりも早く実行されるはずですが、遅延時間を指定しない `setTimeout()` と `setImmediate()` の比較をそれら単体でやると実行順序は不定となります。

```js:timeout_vs_immediate.js
setTimeout(() => console.log("timeout"));
setImmediate(() => console.log("immediate"));
```

実際に複数回実行すると次のようにどちらのケースも起きます。

```sh
❯ node timeout_vs_immediate.js
timeout
immediate
❯ node timeout_vs_immediate.js
immediate
timeout
```

公式ドキュメントにはそれらのオーダーは呼び出されたコンテキストに依存すると書いてあります。メインモジュール内で両方呼び出された場合にはそれらのタイミングはプロセスのパフォーマンスに束縛されます。

> The order in which the timers are executed will vary depending on the context in which they are called. If both are called from within the main module, then timing will be bound by the performance of the process (which can be impacted by other applications running on the machine).
> ([The Node.js Event Loop, Timers, and process.nextTick() | Node.js](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#setimmediate-vs-settimeout) より引用)

I/O サイクル(Callback-based API の `fs` のメソッドのコールバックの中など)で呼び出すと決定論的になり、`setImmediate()` のコールバックが常に先に実行されます。

```js
const fs = require('fs');

fs.readFile("./test.md", () => {
  setImmediate(() => console.log("immediate"));
  setTimeout(() => console.log("timeout"));
});

// 実行すると常に同じ順番になる(決定論的)
// ❯ node ioCycle.js
// immediate
// timeout
```

ちなみに、Node の `setTimeout()` API は遅延時間を指定しない場合はデフォルトで `1` として設定されます。また、`1` 以下の数値を指定しても `1` としてセットされます。

> When delay is larger than 2147483647 or less than 1, the delay will be set to 1. Non-integer delays are truncated to an integer.
> ([Timers | Node.js v18.2.0 Documentation](https://nodejs.org/dist/v18.2.0/docs/api/timers.html#settimeoutcallback-delay-args) より引用)

ということで、絶対に１ミリ秒以上かかります。

ちなみに WHATWG 仕様には次のように、正確に指定した時間通りの実行が保証されるわけではないと記載されています。

> This API does not guarantee that timers will run exactly on schedule. Delays due to CPU load, other tasks, etc, are to be expected.

また、`setTimeout()` は `queueMicrotask()` のようにコールバックを直ちにキューに送るという処理ではなく、あくまで「**遅延時間が経過したら**」コールバックをタスクキューに送るという処理なので注意してください。

同期処理やマイクロタスクを使った処理が１つでもあると、この結果は大抵 Phase の順番通りになり、Timers のコールバックの方が先に処理されます。とはいえ、タイマーなので不定性がつきまといます。

```js:normalTimeComparison.js
setImmediate(() => console.log("immediate"));
setTimeout(() => console.log("timeout"));
// process.nextTick(() => console.log("nextTick"));
console.log("sync process");

// ❯ node normalTimeComparison.js
// sync process
// timeout
// immediate
```

筆者の環境ではタイマーの遅延時間を `3` にすると `setImmediate()` のコールバックが先に実行されます。

タイマーの比較はあまり気にしすぎると混乱するので軽く流しても大丈夫です。`setImmediate()` の正しい使い方はこのような比較ではなく、I/O イベントのコールバック直後に実行するクリーンアップなどに利用します。

> Schedules the "immediate" execution of the callback after I/O events' callbacks.

### Deno 環境のイベントループ

Deno は Node に比べてかなりシンプルです。Promise-based API を非同期 API の基軸にして、マイクロタスクの仕組みに集約しているためタスクをサイクルして使うフェーズ概念そのものがありません。Node よりもシンプルに考えることができます。Node のフェーズに相当するものは Timer 系の処理(`setTimeout()` や `setInterval()`)のみです。

Deno 環境のイベントループの実装は Tokio という Rust 言語のための非同期ランタイムが担当しており、JavaScript の Promise などの仕組みは Rust における Future という別の非同期処理の仕組みによって実現されています。ただし、Deno のイベントループについてはほとんど情報がなく、公式ドキュメントが更新されるのを待っています(変更が多いので、かなり時間がかかりそうです)。

ですが、今まで見てきたとおり、「**単一タスクが完了したら、すべてのマイクロタスクを処理する**」というループであることは変わりません(バグや現在取組中の issue 以外は)。

ということで基本的に V8 エンジンのデフォルトイベントループと同じとして考えてよいです。

```js:V8エンジンのデフォルトイベントループ
while (tasksAreWaiting()) {
  task = taskQueue.pop();
  execute(task);

  while (microtaskQueue.hasTasks()) {
    doMicrotask();
  }
}
```

ただし、Node 互換モードがあるので、そのモードを使用する際には、Node の nextTickQueue などをポリフィルを使って使用できることに注意してください。

#### イベントループの実装

一応実装がどこにあるかも示しておきます。現時点(2023/02/08) 最新の Rust で書かれたイベントループの実行関数と tick (一周の処理) はリポジトリ上では次の場所にあります。Deno のイベントループの実装について詳しく調べたい場合はこのファイルを調査してください。

https://github.com/denoland/deno/blob/161a4fea47d134fa6077910b794883ffb6e9b0c5/core/runtime.rs#L1238-L1438

Rust が分からなくても雰囲気はつかめます。おそらく ops は Operations の略称ですし、実装では Node 互換モードのための NextTick を枯らすための処理があることが分かります。また、コメントで Dynamic import の解決や Unhandled Rejection に関しても記述されています。実行順番的には Dynamic import の解決 → NextTick を枯らす → Microtask を枯らす → Promise Rejection をチェックする、というようになっていますね。

イベントループの基本動作メカニズム(タスクとマイクロタスクの関係)については理解できても、これらの細かい箇所は筆者もまだ理解しきれていない技術の部分であり、最終的にはこれらの部分を含めて理解するということになります(この本では未記載の箇所であり、将来的には全貌を解明するつもりです)。

## 非同期処理を考える上でのイベントループ

各環境におけるイベントループについての共通性質を再度まとめておきます。

ブラウザ環境でも、ランタイム環境でもイベントループの共通性質として言えることは、以下となります。

- イベントループは１つ以上のタスクキューを持ち、単一のマイクロタスクキューを持つ(Node 以外)
- 開始時のスクリプト評価(すべての同期処理)はコードの実行を考える上での最初のタスクになる
- 単一タスクが実行された後にすべてのマイクロタスクを処理する(コールスタックが空になったらマイクロタスクのチェックポイントが実行される)

V8 エンジンのデフォルトイベントループで見たようにイベントループの基本形はこのようになります。

```js:V8エンジンのデフォルトイベントループ
while (tasksAreWaiting()) {
  // すくなくても１つ以上のタスクキューから(環境定義のルールで)１つのタスクキューを選ぶ
  queue = getNextQueue();
  // 単一タスクを処理する
  task = queue.pop();
  execute(task);

  // １つのマイクロタスクキューにあるすべてのマイクロタスクを処理する
  while (microtaskQueue.hasTasks()) {
    doMicrotask();
  }
}
```

概念的には、あとは環境実装のルールに従ってこの形に色々つけていくだけです。
