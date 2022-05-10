---
title: "イベントループの概要と注意点"
---

:::message
こちらのチャプターは以前に作成した内容です。次のチャプターからは完全に新しい内容となります。内容的に重複する部分がありますが、ここに記載されている内容も必要なのでこのままにしておきます。内容に訂正はほとんどないですが注意してください。
:::

# イベントループの各ステップ
Promise チェーンの解説に入る前に、まずはイベントループの各ステップについて説明しておきます。

ただし、このループはブラウザ環境の場合であることに注意してください。実装はそれぞれ違いますし、ブラウザ環境である UI Rendering のステップなどは Node や Deno ではありません。Node のイベントループにあるいくつかのフェーズはもちろんブラウザ環境にはありません。

ある程度の抽象度でイベントループを理解したら**各環境でのループの実装について調べる必要があります**(これについては『それぞれのイベントループ』のチャプターで詳しく解説しています)。

:::message
Chrome などのブラウザ環境と Node.js や Deno の環境では JavaScript エンジンとして同じ V8 エンジンを採用してますが、イベントループの実装はそれぞれ違います。それぞれのイベントループを実装するために利用されているライブラリは以下のものとなります。

- [Libevent](https://libevent.org) (Chrome): an event notification library
- [Libuv](https://libuv.org) (Node.js) : a multi-platform support library with a focus on asynchronous I/O
- [Tokio](https://tokio.rs) (Deno) : an asynchronous Rust runtime 

V8 エンジンが担当しているのは、Heap と Call Stack です。V8 のソースコードには setTimeout や DOM、HTTP request といった Web APIs は含まれていません。

『What the heck is the event loop anyway?』の動画で解説されているのは、ブラウザ環境でのイベントループです。Promise チェーンや async/await といった言語としての非同期処理を理解するレベルでの抽象化なら、Node.js でも Deno でも同じように考えてある程度は大丈夫です。Web APIs の部分はブラウザ特有のものなので、Node.js や Deno ではそれに対応するものとして Runtime APIs で置き換えて考えます。
:::

イベントループはタスクキューとマイクロタスクキュー内にあるタスク/マイクロタスクを処理するための**ループアルゴリズム**です。イベントループは次に実行するタスクとマイクロタスクを選択して、コールスタック(Call Stack)へと配置します。

:::message
イベントループ自体は ECMAScript で仕様定義されていません。従って ECMAScript の言語機能ではなく、環境側で考慮すべきものとなります。
具体的なイベントループの仕様は whatwg の「HTML Living Standard」で言及されており、基本的にはブラウザ環境はこれに基づいていると考えられます。これとは別に Node.js は少しばかり複雑なイベントループを持っています。
:::

イベントループは基本的に次の４つのステップから構成されます。

1. スクリプトの評価: 関数本体であるかのように(例えば `main()` のように)、スクリプトを同期的に実行し、コールスタックが空になるまで実行する。
2. **単一の**タスク(マクロタスク)の実行: タスクキュー(マクロタスクキュー) から最も古いタスク(マクロタスク)を選択してコールスタックが空になるまで実行する。
3. **すべての**マイクロタスクの実行: マイクロタスクキューから最も古いマイクロタスクを選択してコールスタックが空になるまで実行し、マイクロタスクキューが空になるまで繰り返す。
4. UI のレンダリング更新: UI をレンダリング更新してステップ２に戻る(ブラウザ環境のみで、Node や Deno では存在しない)。

参考
https://www.jsv9000.app

４つのステップと言いましたが、最初のステップは実質２番目のステップであり、「スクリプトの評価」自体が最初のタスクになっています。

実は JS Visualizer を使っている内はこれを理解できませんでしたので、イベントループは次の擬似コードで理解したほうがよいです。

# イベントループへの誤解

以下はブラウザ環境における疑似コードでのイベントループのループの仕組みです。

:::message alert
『それぞれのイベントループ』でより詳しく様々な環境のイベントループについて解説しました。ブラウザ環境の疑似コードは一部代わりますが、本質は変わらないので大丈夫です。もとの疑似コードも残しておきます。
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
- `taskQueue.processNextTask()` はタスク(マクロタスク)の処理
- `microtaskQueue.hasNextMicrotask()` はマイクロタスクキューにマイクロタスクがあるかどうかの真偽値
- `microtaskQueue.processNextMicrotask()` はマイクロタスクの処理
- `shouldRender()` はレンダリングを更新すべきかの真偽値

実は疑似コードもいくつかパターンがあるのですが、次のサイトのページから拝借しています。疑似コードについてはこのページを見てもらうとよく理解できるはずです。イベントループの全貌についてもかなり解像度で理解できると思います。

https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/

そして自分が**最近まで誤解・勘違いしていた重要な点**について挙げておきます。

- 最初のタスクは「**スクリプトの評価**」となる
- **ループはネストしている**(イベントループの各ループにおいて、マクロタスクは一個の処理なのにマイクロタスクの処理は完全に終わるまで続く)
-タスクキューは複数あり、そこから１つのキューを選択してひとつのタスクを実行する
- レンダリングの更新は起きない場合もある(各ループで必ず更新されるわけではなく、60fps の平均 16.7 ミリ秒ごとに起こる)

:::message alert
これらについては本を書き始めてからようやく理解できるようになりました。従って、この本の内容を精査する必要がでてきたのでかなり修正するはめになりました。間違った内容を出してしまっていたため申し訳無いです。現在は修正したので、また何か間違いがあれば修正いたします。

また、なるべく早く正確な情報に訂正するため、このページではわかりやすさや細かい説明などは省いて更新しています。
:::

タスクキューへと供給されるタスク(Task)は Task source と呼ばれる供給源があり、それは複数存在しています。

>Per its source field, each task is defined as coming from a specific task source. For each event loop, every task source must be associated with a specific task queue.
>([task source | HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#task-source)より引用)

例えば、`setTimeout()` API のコールバックとマウスクリックから発火されるイベントからのコールバックはそれぞれべつの Task source から来るタスクであり、それぞれのタスクは別々のタスクキューへと送られます。

このようにタスク(マクロタスク)はそれぞれ分離した供給源が複数個あります。そして重要なこととして、タスクキューには２つの制約があります。

- 「**同一のソースから運ばれるタスクは同一のキューに属さなければならない**」
- 「タスクはすべてのキューににおいて挿入された順番に処理されなければならない」

通常、`setTimeout()` で非同期処理を考えたりしますが、`setTimeout()` でタスクキューへと送ったタスク(マクロタスク)はすべて同一のキューへと配置されます。その一方で、ユーザーのクリック操作などから送られてくるタスクは `setTimeout()` でタスクを送ったキューとは別のキューへと配置されます。

そして重要なこととして、イベントループの 1 回のループでは複数あるタスクキューから１つのみを選択してその中で最も古いタスク(キューの頭にあるもの)を実行します。

複数個のタスクキューがあるため、どれを選ぶかということが重要になってきますが、User agent が自由に選択します。従って、開発者側が正確なタイミングで特定の実行することはできません。

ブラウザ環境はユーザー入力などのパフォーマンスに敏感なタスクを優先的に処理しようとしてイベントループ内にある複数のタスクキューから関連するキューを選びだし、その中にあるタスクを処理します。つまり、`setTimeout()` で登録しておいたコールバックの処理よりもマウスクリックなどの操作のほうが優先度の高い場合があり、`setTimeout()` で登録したコールバックが処理されるのが遅延することがあります。

# イベントループのステップ１とステップ２は実質的に同じ

:::message alert
こちらについての情報は完全に理解したので、次の『タスクキューとマイクロタスクキュー』と『ールスタックと実行コンテキスト』のチャプターで解説しています。
:::

JS Visualizer を使い始めた初期段階では次のコードが理解できなくなります。実際に実行してみてください。

```js
// Macrotask vs Microtask 
setTimeout(() => console.log('timeout'), 0);
Promise.resolve('promise resloved').then(res => console.log(res))
```

- [Macrotask vs Microtask - JS Visualizer](https://www.jsv9000.app/?code=Ly8gTWFjcm90YXNrIHZzIE1pY3JvdGFzawpzZXRUaW1lb3V0KGZ1bmN0aW9uIGEoKSB7fSwgMCk7CgpQcm9taXNlLnJlc29sdmUoKS50aGVuKGZ1bmN0aW9uIGIoKSB7fSk7)

イベントループを４ステップで考えると、出力は "timeout" → "promise resolved" になるはずですが、実際はその逆で "promise resolved" → "timeout" になります。そしてこの実行順番はブラウザ環境と Node でも Deno でも同じになります。

本来ならタスクが先に実行されるべきなのに、マイクロタスクが先に実行されているため、これによってステップ２の「単一のタスクの実行」がスキップされているように思えてしまいます。

最初、自分はこの事実にさえ気付いてさえいなかったのですが、よくよく考えるとおかしいと感じて調査した結果、どうやらステップ１の「スクリプトの評価」自体がタスク(Task)として行われているらしいという結論に至りました。現時点で情報が不正確なのが申し訳無いですが、なにせイベントループの学習ソースとなる記事が少ないので「どうやらそうらしい」という結論になってしまいました。

:::message
JavaScript の非同期処理の学習はこういう感じで ECMAScript の言語機能だけでなく、環境の実装や「**謎**」を解いていく必要があるので、本当に難しいです。JavaScript 自体の学習だけでは永遠に非同期処理が分かりません。
:::

JSConf.Asia での Jake Archibald 氏による講演動画『In The Loop』においても、`Script` がコールスタックに載っており、`Script` が Call Stack から pop されて初めて Microtask が実行されています。

↓ 動画の 31:33 ~ のところ。
@[youtube](cCOL7MC4Pl0)

これまた Jake Archibald 氏の記事内のデモにおいて、最初の「スクリプトの評価」もしくは「スクリプトの実行」自体が実質的に Task となっており、Call stack に `script` がプッシュされています。

https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/

実際にそれを証明するためのコードとして、次のようなコードが考えられます。`pause` 関数によって同期的にブロッキングした後に `setTimeout()` 関数と `queueMicrotask()` で競争させて、どちらのコールバックが先に実行されるかを検証します。

```js
// blockingTimer.js

// メインスレッドを同期的にブロッキングする関数
function pause(milliseconds, order) {
  const dt = new Date();
  while ((new Date()) - dt <= milliseconds) {
    // Do nothing
  }
  console.log(`${order} Timer exit after ${milliseconds}`);
}

console.log("[1] Sync process");

setTimeout(() => {
  console.log("[4? - 5] setTimeout[0ms] finished");
}, 0);
setTimeout(() => {
  console.log("[5? - 6] setTimeout[1000ms] finished");
}, 1000);
queueMicrotask(() => {
  console.log("[6? - 4] queueMicrotask");
});

pause(3000, "[2]"); // 3000[ms] メインスレッドをブロッキング

console.log("[3] Sync process");
```

3000 ミリ秒の間メインスレッドをブロッキングしている最中に環境に委任したタイマー処理自体は完了しているはずなので、ステップ１がステップ２と同じでなれば、ステップ２の「単一の Task(Macrotask) の実行」を行うはずですので、`console.log("[3] Sync process");` が終わった時点で、次に実行されるのは、`console.log("[4? - 5] setTimeout[0ms] finished");` であり、`"[4? - 5] setTimeout[0ms] finished"` という文字列がログに順番として出力されるはずですが、実際の出力は次のようになります。

```sh
❯ deno run blockingTimer.js
[1] Sync process
[2] Timer exit after 3000
[3] Sync process
[6? - 4] queueMicrotask
[4? - 5] setTimeout[0ms] finished
[5? - 6] setTimeout[1000ms] finished
```

やはり、「ステップ１とステップ２は同質のものであり、実質的にスクリプトの評価自体が最初の Task(Macrotask)である」と考えるとすべて辻褄があうので、この本では「イベントループのステップ１とステップ２は実は同質のものであり、スクリプトの評価自体が最初の Task(Macrotask) となる」を真として話を進めていきます。

JSConf EU 2018 での Erin Zimmer 氏の講演動画である『Further Adventures of the Event Loop』で明確に語られていました。動画の `1:35 ~` のところ。

@[youtube](u1kqx6AenYw)

https://2018.jsconf.eu/speakers/erin-zimmer-further-adventures-of-the-event-loop.html

ブラウザ環境において、次のようなスクリプトがあった場合、ブラウザはスクリプトタグをパースして、Task を作成し、**同期処理の部分は実際にタスクとして実行されます**。`addEventListener` のコールバック部分については、ブラウザがキーダウンイベントを受け取った時に別の Task として実行されます。

```html
<script>
  // <- Task1
  const foo = bar;
  foo.doSomething();
  
  document.body.addEventLlistener('keydown', (event) => {
    // <- Task2
    if (event.key === 'PageDown') {
      location.href = "/#/36";
    }
    // Task2 -> 
  });
  // Task1 ->
<script>
```

