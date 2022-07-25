---
title: "イベントループの概要と注意点"
aliases: [ch_イベントループの概要と注意点]
---

# このチャプターについて
このチャプターでは、イベントループの各ステップと、それについてのいくつかの注意点などについて解説します。イベントループについては、『[What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ)』を見ていればなんとなくイメージがついていると思いますが、まだ未視聴の場合は動画を視聴してからこのチャプターを読んでください。

:::message alert
**注意**

**こちらのチャプターは以前に作成したイベントループに関しての古い解説とそれに対しての訂正・注釈・補足となります**。このチャプターの解説はそういった理由からイベントループを理解するために不十分なところが多いので、理解できない部分があっても気にしないでください。

イベントループについては別のチャプターで更新された知見に基づいた詳細な解説を行っており、『[それぞれのイベントループ](c-epasync-what-event-loop)』を中心とした周辺のチャプターに基づきます。内容的に重複する部分がありますが、ここに記載されている内容も理解するために必要なところがあるので削除していません。ただし、古い知見の観点からの解説ではありますが、**完全に古い状態というわけではなく、ある程度分かりやすいように修正もほどこしています**。

また、誤解や間違いを直すための試行錯誤の過程が一部見て取れると思うので「そういう資料」であると思って、メッセージの項目に注意して読みすすめてください。
:::

# イベントループの各ステップ
まずはイベントループの各ステップについて説明しておきます。

ただし、このループはブラウザ環境の場合であることに注意してください。実装はそれぞれ違いますし、ブラウザ環境であるレンダリングのステップなどは Node や Deno ではありません。Node のイベントループにあるいくつかのフェーズはもちろんブラウザ環境にはありません。

という訳で、ある程度の抽象度でイベントループを理解したら**各環境でのループの実装について調べる必要があります**(これについては『[それぞれのイベントループ](c-epasync-what-event-loop)』のチャプターで詳しく解説しています)。

:::message
Chrome などのブラウザ環境と Node.js や Deno の環境では JavaScript エンジンとして同じ V8 エンジンを採用してますが、イベントループの実装はそれぞれ違います。それぞれのイベントループを実装するために利用されているライブラリは以下のものとなります。

- [Libevent](https://libevent.org) (Chrome): an event notification library
- [Libuv](https://libuv.org) (Node.js) : a multi-platform support library with a focus on asynchronous I/O
- [Tokio](https://tokio.rs) (Deno) : an asynchronous Rust runtime 

V8 エンジンが担当しているのは、Heap と Call Stack です。V8 のソースコードには setTimeout や DOM、HTTP request といった Web APIs は含まれていません(別のチャプターで後述しますが、タイマー機能の無い `setTimeout()` もどきなどは入っています)。

『[What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ)』の動画で解説されているのは、ブラウザ環境でのイベントループです。Web APIs の部分はブラウザ特有のものなので、Node.js や Deno ではそれに対応するものとして Runtime APIs で置き換えて考えてください。また、動画で解説されているイベントループは Promise のシンタックスが ECMAScript に導入される以前のものですので注意してください。
:::

イベントループはタスクキューとマイクロタスクキュー内にあるタスク/マイクロタスクを処理するための**ループアルゴリズム**です。イベントループは次に実行するタスクあるいはマイクロタスクを選択して、コールスタック(Call Stack)へと配置します。コールスタックに配置されたタスクやマイクロタスクが実際に処理されることで非同期処理が実現します。

:::message
イベントループ自体は ECMAScript で仕様定義されていません。従って ECMAScript の言語機能ではなく、環境側で考慮すべきものとなります。具体的なイベントループの仕様は whatwg の「HTML Living Standard」で言及されており、ブラウザ環境はこれに基づいています。これとは別に Node.js は少しばかり複雑なイベントループを持っていますが、基本的には、ブラウザ環境のイベントループ仕様に近づこうとします。これについても『[それぞれのイベントループ](c-epasync-what-event-loop)』のチャプターで解説します。
:::

[JS Visualizer](https://www.jsv9000.app) ではイベントループは基本的に次の４つのステップから構成されると記載されています。

- (1) スクリプトの評価: 関数本体であるかのように(例えば `main()` のように)、スクリプトを同期的に実行し、コールスタックが空になるまで実行する。
- (2) **単一の**タスクの実行: タスクキューから最も古いタスクを選択してコールスタックが空になるまで実行する。
- (3) **すべての**マイクロタスクの実行: マイクロタスクキューから最も古いマイクロタスクを選択してコールスタックが空になるまで実行し、マイクロタスクキューが空になるまで繰り返す。
- (4) UI のレンダリング更新: UI をレンダリング更新してステップ２に戻る(ブラウザ環境のみで、Node や Deno では存在しない)。

JS Visualizer を使い始めた当初は理解できていませんでしたが、上の４つのステップにおいてステップ(1)は実質ステップ(2)と同じであり、「スクリプトの評価」自体が**最初のタスク**になっています。文字だけの説明だと理解するのは難しいので、イベントループは擬似コードで理解したほうがよいです。

:::message alert
ステップ(1)が実質的にステップ(2)であることを理解できなかった理由は、「**最初のタスク**」が何になるかが分からなかったことと、「グローバルコンテキスト」と「マイクロタスクのチェックポイント」についての知識が不足していたためです。実質的に最初のタスクとなるのはスクリプトの評価であり実際にコールスタック上で処理されているのはグローバルコンテキストです。グローバルコンテキストがコールスタックからポップすることでコールスタックが空になるのでマイクロタスクのチェックポイントとなります。

ちなみに、ステップ(1)で「関数本体であるかのように」と書いていますが、これがコールスタックに積まれるグローバルコンテキストのことを指しています。

現時点では、イベントループについて上のように４つのステップで考える必要は**特にない**と考えています。重要なのは、「**単一タスクを実行したら、すべてのマイクロタスクを処理する**」です。これについては『[それぞれのイベントループ](c-epasync-what-event-loop)』のチャプターで詳しく解説しています。
:::

# イベントループへの誤解

以下はブラウザ環境における疑似コードでのイベントループのループの仕組みです。

:::message alert
『[それぞれのイベントループ](c-epasync-what-event-loop)』でより詳しく様々な環境のイベントループについて新しい疑似コードで解説しました。ブラウザ環境の疑似コードは一部代わりますが、本質は変わらないので大丈夫です。もとの疑似コードも残しておきます。

イベントループの疑似コードには色々パターンがありますが、所詮は疑似コードなので、物によっては部分的に省略されている部分などがあり、理解するための解像度が足りていないものなどあるので注意してください。
:::

```js
while (eventLoop.waitForTask()) {
  const taskQueue = eventLoop.selectTaskQueue()
  if (taskQueue.hasNextTask()) {
    taskQueue.processNextTask()
  }

  const microtaskQueue = eventLoop.microTaskQueue
  while (microtaskQueue.hasNextMicrotask()) {
    microtaskQueue.processNextMicrotask()
  }

  if (shouldRender()) {
    applyScrollResizeAndCSS()
    runAnimationFrames()
    render()
  }
}
```

擬似コードですが理解するための補足として次のように考えます。
- `eventLoop.selectTaskQueue()` は複数個存在しているタスクキューから１つのタスクキューを選ぶ処理
- `taskQueue.hasNextTask()` そのタスクキューにタスクが存在しているかどうかの真偽値
- `taskQueue.processNextTask()` はタスクの処理
- `microtaskQueue.hasNextMicrotask()` はマイクロタスクキューにマイクロタスクがあるかどうかの真偽値
- `microtaskQueue.processNextMicrotask()` はマイクロタスクの処理
- `shouldRender()` はレンダリングを更新すべきかの真偽値

疑似コードには**いくつかパターンがある**のですが、次のサイトのページから拝借しています。疑似コードについてはこのページを見てもらうとよく理解できるはずです。イベントループの全貌についてもかなり解像度で理解できると思います。

https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/

そして自分がこの本を書き始めた時点で**誤解・勘違いしていた重要な点**について挙げておきます。

- 最初のタスクは「**スクリプトの評価**」となる
- **ループはネストしている**(イベントループの各ループにおいて、タスクは一個の処理なのにマイクロタスクの処理は完全に終わるまで続く)
- **タスクキューは複数個あり**、そこから１つのキューを選択して１つのタスクを実行する
- レンダリングの更新は起きない場合もある(各ループで必ず更新されるわけではなく、60fps の平均 16.7 ミリ秒ごとに起こる)

:::message alert
これらについてはこの本を書き始めてからようやく理解できるようになりました。従って、本の内容を精査してかなり修正するはめになりました。間違った内容を出してしまっていたため申し訳無いです。現在は修正したので、また何か間違いがあれば修正いたします。間違いなどを見つけた場合には[スクラップ](https://zenn.dev/estra/scraps/20dc6c4a1b64f8)で教えていただけると非常に助かります。~~また、なるべく早く正確な情報に訂正するため、このページではわかりやすさや細かい説明などは省いて更新しています~~。現時点では新しい知見に基づいた内容を別のページで解説しています。
:::

タスクキューへと供給されるタスク(Task)はタスク源(Task source)と呼ばれる供給源があり、それは複数存在しています。

>Per its source field, each task is defined as coming from a specific task source. For each event loop, every task source must be associated with a specific task queue.
>([task source | HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#task-source)より引用)

例えば、`setTimeout()` API のコールバックとマウスクリックから発火されるイベントからのコールバックはそれぞれべつの Task source から来るタスクであり、それぞれのタスクは別々のタスクキューへと送られます。

このようにタスクはそれぞれ分離した供給源が複数個あります。そして重要なこととして、タスクキューには２つの制約があります。

- 「**同一のソースから運ばれるタスクは同一のキューに属さなければならない**」
- 「タスクはすべてのキューににおいて挿入された順番に処理されなければならない」

通常、`setTimeout()` で非同期処理を考えたりしますが、`setTimeout()` でタスクキューへと送ったタスクはすべて同一のキューへと配置されます。その一方で、ユーザーのクリック操作などから送られてくるタスクは `setTimeout()` でタスクを送ったキューとは別のキューへと配置されます。

そして重要なこととして、イベントループの 1 回のループでは複数あるタスクキューから１つのみを選択してその中で最も古いタスク(キューの頭にあるもの)を実行します。

複数個のタスクキューがあるため、どれを選ぶかということが重要になってきますが、User agent が自由に選択します。従って、開発者側が正確なタイミングで特定の実行することはできません。

ブラウザ環境はユーザー入力などのパフォーマンスに敏感なタスクを優先的に処理しようとしてイベントループ内にある複数のタスクキューから関連するキューを選びだし、その中にあるタスクを処理します。つまり、`setTimeout()` で登録しておいたコールバックの処理よりもマウスクリックなどの操作のほうが優先度の高い場合があり、`setTimeout()` で登録したコールバックが処理されるのが遅延することがあります。

# イベントループのステップ１とステップ２は実質的に同じ

:::message alert
こちらについての情報は完全に理解したので、次の『[タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)』と『[コールスタックと実行コンテキスト](b-epasync-callstack-execution-context)』のチャプターで解説しています。以下の内容は、それを理解するための試行錯誤の内容として考えてください。
:::

JS Visualizer を使い始めた初期段階では次のコードの実行順序が理解できなくなります。実際に実行してみてください。

```js:タスクvsマイクロタスク
console.log("🦖 [1] Mainline");

setTimeout(() => { // コールバック関数はタスクとして処理される
  console.log("⏰ [3] Callback is a task");
}, 0); 

Promise.resolve()
  .then(() => { // コールバック関数はマイクロタスクとして処理される
    console.log("👦 [2] Callback is a microtask");
  });

/* 実行結果
🦖 [1] Mainline
👦 [2] Callback is a microtask
⏰ [3] Callback is a task
*/
```

- [Task vs Microtask - JS Visualizer](https://www.jsv9000.app/?code=Ly8gVGFzayB2cyBNaWNyb3Rhc2sKY29uc29sZS5sb2coIlsxXSIpCgpzZXRUaW1lb3V0KGZ1bmN0aW9uIGEoKSB7CiAgY29uc29sZS5sb2coIlszXSIpOwp9LCAwKTsKClByb21pc2UucmVzb2x2ZSgpLnRoZW4oZnVuY3Rpb24gYigpIHsKICBjb25zb2xlLmxvZygiWzJdIik7Cn0pOwo%3D)

:::message alert
注意: JS Visuzlizer では**実質的に最初のタスクとなるグローバルコンテキストは可視化されない**ので最初のマイクロタスク・タスク実行のタイミングについて誤解しないように注意してください。上のコードの実行順番が理解できない理由はこれです。

説明のところに「**Synchronously execute the script as though it were a function body**. Run until the Call Stack is empty.」という様に言葉では書かれているのですが、最初のタスクとして処理されるグローバルコンテキストそのものは可視化されないので最初このことに気づくのは難しいです。
:::

イベントループを４ステップで考えると、出力は `"⏰ [3] Callback is a task"` → `"👦 [2] Callback is a microtask"` になるはずですが、実際はその逆で `"👦 [2] Callback is a microtask"` → `"⏰ [3] Callback is a task"` になります。そしてこの実行順番はブラウザ環境だけでなく Node でも Deno でも同じになります。

４ステップで考えると、ステップ(2)で本来ならタスクが先に実行されるべきなのに、ステップ(3)のマイクロタスクが先に実行されているため、これによってステップ(２)の「単一のタスクの実行」がスキップされているように思えてしまいます。

最初、自分はこの事実にさえ気付いてさえいなかったのですが、よくよく考えるとおかしいと感じて調査した結果、どうやらステップ(1)の「スクリプトの評価」自体がタスク(Task)として処理されているらしいという結論に至りました。~~現時点で情報が不正確なのが申し訳無いですが、なにせイベントループの学習ソースとなる記事が少ないので「どうやらそうらしい」という結論になってしまいました~~(実際これは正しく、「スクリプトの評価」が最初のタスクとなります)。

JSConf.Asia での Jake Archibald 氏による講演動画『In The Loop』においても、`Script` がコールスタックに載っており、`Script` が Call Stack から pop されて初めてマイクロタスクが実行されています。

↓ 動画の 31:33 ~ のところ。
https://youtu.be/cCOL7MC4Pl0

これまた Jake Archibald 氏の記事内のデモにおいて、最初の「スクリプトの評価」もしくは「スクリプトの実行」自体が実質的にタスクとなっており、Call stack に `script` がプッシュされています。

https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/

実際にそれを証明するためのコードとして、次のようなコードが考えられます。`pause` 関数によって同期的にブロッキングした後に `setTimeout()` 関数と `Promise.resolve().then()` で競争させて、どちらのコールバックが先に実行されるかを検証します。

```js:blockingTimer.js
// メインスレッドを同期的に指定時間だけブロッキングする関数
function pause(milliseconds, order) {
  const dt = new Date();
  while ((new Date()) - dt <= milliseconds) {
    // なにもしない
  }
  console.log(`${order} Timer exit after ${milliseconds}`);
}

console.log("[1] Sync process");

// 環境にタイマー処理を委任する
setTimeout(() => {
  // コールバックはタスクとして処理される
  console.log("[4? - 5] setTimeout[0ms] finished");
}, 0);
setTimeout(() => {
  // コールバックはタスクとして処理される
  console.log("[5? - 6] setTimeout[1000ms] finished");
}, 1000);

Promise.resolve().then(() => {
  // コールバックはマイクロタスクとして処理される
  console.log("[6? - 4] then callback");
});

// ここで 3000[ms] メインスレッドをブロッキングする
pause(3000, "[2]");
// pause() が完了した時点で環境に委任したタイマーの時間は経過している

console.log("[3] Sync process");
```

3000 ミリ秒の間メインスレッドをブロッキングしている最中に環境に委任したタイマー処理自体は完了しているはずなので、コールバックが実行できるようになっているはずです。そしてステップ(1)がステップ(2)と同じでなければ、ステップ(2)の「単一のタスクの実行」を行うはずなので、`console.log("[3] Sync process");` が終わった時点で、次に実行されるのは、`console.log("[4? - 5] setTimeout[0ms] finished");` であり、`"[4? - 5] setTimeout[0ms] finished"` という文字列がログに順番として出力されるはずですが、実際の出力は次のようになります。

```sh
❯ deno run blockingTimer.js
[1] Sync process
[2] Timer exit after 3000
[3] Sync process
[6? - 4] then callback
[4? - 5] setTimeout[0ms] finished
[5? - 6] setTimeout[1000ms] finished
```

やはり、「ステップ(1)とステップ(2)は同質のものであり、実質的にスクリプトの評価自体が最初のタスクである」と考えるとすべて辻褄があうので、この本では「イベントループのステップ１とステップ２は実は同質のものであり、スクリプトの評価自体が最初のタスクとなる」を真として話を進めていきます(実際これは正しかったです)。

:::message alert
実際、スクリプト評価がコードの実行を考える上での「最初のタスク」となります。タスクだけでなく、「グローバルコンテキスト」のポップと「マイクロタスクのチェックポイント」について考えることですべての同期処理の後でタスクではなくマイクロタスクが処理されることが理解できます。

これについては、『[タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)』と『[コールスタックと実行コンテキスト](b-epasync-callstack-execution-context)のチャプターで解説しています。
:::

そして、この事実は JSConf EU 2018 での Erin Zimmer 氏の講演動画である『Further Adventures of the Event Loop』で明確に語られていました。動画の `1:35 ~` のところ。

https://youtu.be/u1kqx6AenYw

https://2018.jsconf.eu/speakers/erin-zimmer-further-adventures-of-the-event-loop.html

ブラウザ環境において、次のようなスクリプトタグがあった場合、ブラウザはこのスクリプトタグをパースして、**同期処理の部分は実際にタスクとして実行されます**。`addEventListener` のコールバック部分については、ブラウザがキーダウンイベントを受け取った時に別のタスクとして実行されます。

```html
<script>
  // <- Task1 (スクリプトの評価: すべての同期処理)
  const foo = bar;
  foo.doSomething();
  
  document.body.addEventLlistener('keydown', (event) => {
    // <- Task2 (コールバック)
    if (event.key === 'PageDown') {
      location.href = "/#/36";
    }
    // Task2 -> 
  });
  // Task1 ->
<script>
```

このように `<script>` タグ内の JavaScript はまとめてタスク(Task1)として処理されます。イベントリスナーに登録されてるコールバックはイベントが発火した時点で別のタスク(Task2)として処理されます。

