---
title: "Event loop の概要と注意点"
---

# Event loop の各ステップ
Promise チェーンの解説に入る前に、まずは Event loop の各ステップについて説明しておきます。

ただし、このループはブラウザ環境の場合であることに注意してください。実装はそれぞれ違いまし、ブラウザ環境である UI Rendering のステップなどは Node や Deno ではありません。Node の Event loop にあるいくつかのフェーズはもちろんブラウザ環境にはありません。

ある程度の抽象度でイベントループを理解したら**各環境でのループの実装について調べる必要があります**(現在調査中)。

:::message
Chrome などのブラウザ環境と Node.js や Deno の環境では JavaScript エンジンとして同じ V8 エンジンを採用してますが、Event Loop の実装はそれぞれ違います。それぞれの Event loop を実装するために利用されているライブラリは以下のものとなります。

- [Livevent](https://libevent.org) (Chrome): an event notification library
- [Livuv](https://libuv.org) (Node.js) : a multi-platform support library with a focus on asynchronous I/O
- [Tokio](https://tokio.rs) (Deno) : an asynchronous Rust runtime 

V8 エンジンが担当しているのは、Heap と Call Stack です。V8 のソースコードには setTimeout や DOM、HTTP request といった Web APIs は含まれていません。

「What the heck is the event loop anyway?」の動画で解説されているのは、ブラウザ環境での Event Loop です。Promise チェーンや async/await といった言語としての非同期処理を理解するレベルでの抽象化なら、Node.js でも Deno でも同じように考えてある程度は大丈夫です。Web APIs の部分はブラウザ特有のものなので、Node.js や Deno ではそれに対応するものとして Runtime APIs で置き換えて考えます。
:::

Event Loop は Macrotask queue と Microtask queue 内にある Macrotask/Microtask を処理するための**ループアルゴリズム**です。Event Loop は次に実行する Macrotask/Microtask を選択して、Call Stack へと配置します。

:::message
Event loop 自体は ECMAScript で仕様定義されていません。従って ECMAScript の言語機能ではなく、環境側で考慮すべきものとなります。
具体的な Event loop の仕様は whatwg の「HTML Living Standard」で言及されており、基本的にはブラウザ環境はこれに基づいていると考えられます。これとは別に Node.js は更に複雑な Event loop を持っています。

この本では、Promise のためのキューである Microtask を主に考えていくため、
:::

Event Loop は基本的に次の４つのステップから構成されます。

1. スクリプトの評価: 関数本体であるかのように(例えば `main()` のように)、スクリプトを同期的に実行し、コールスタックが空になるまで実行する。
2. **単一の** Task(Macrotask) の実行: Task queue(Macrotask queue) から最も古いタスク(マクロタスク)を選択して Call Stack が空になるまで実行する。
3. **すべての** Microtask の実行: Microtask queue から最も古いマイクロタスクを選択して Call Stack が空になるまで実行し、Microtask queue が空になるまで繰り返す。
4. UI のレンダリング更新: UI をレンダリング更新してステップ 2 に戻る(ブラウザ環境のみで、Node や Deno では存在しない)。

参考
https://www.jsv9000.app

４つのステップと言いましたが、最初のステップは実質２番目のステップであり、「スクリプトの評価」自体が最初の Task(Macrotask) になっています。

実は JS Visualizer を使っている内はこれを理解できませんでしたので、Even loop は次の擬似コードで理解したほうがよいです。

# Event loop への誤解

以下はブラウザ環境における疑似コードでの Event loop のループの仕組みです。

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
- `eventLoop.selectTaskQueue()` は複数個存在している Task queue から１つの Task queue を選ぶ処理
- `taskQueue.hasNextTask()` そのタスクキューにタスクが存在しているかどうかの真偽値
- `taskQueue.processNextTask()` はタスク(マクロタスク)の処理
- `microtaskQueue.hasNextMicrotask()` はマイクロタスクキューにマイクロタスクがあるかどうかの真偽値
- `microtaskQueue.processNextMicrotask()` はマイクロタスクの処理
- `shouldRender()` はレンダリングを更新すべきかの真偽値

実は疑似コードもいくつかパターンがあるのですが、次のサイトのページから拝借しています。疑似コードについてはこのページを見てもらうとよく理解できるはずです。Event loop の全貌につてもかなり解像度で理解できると思います。
https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/

そして自分が**最近まで誤解・勘違いしていた重要な点**について挙げておきます。

- 最初のタスクは「**スクリプトの評価**」となる
- **ループはネストしている**(Event loop の各ループにおいて、マクロタスクは一個の処理なのにマイクロタスクの処理は完全に終わるまで続く)
- Task queue(Macrotask queue) は複数あり、そこから１つのキューを選択してひとつの Task(Macrotask) を実行する
- レンダリングの更新は起きない場合もある(各ループで必ず更新されるわけではない)

:::message alert
これらについては本を書き始めてからようやく理解できるようになりました。従って、この本の内容を精査する必要がでてきたのでかなり修正するはめになりました。間違った内容を出してしまっていたため申し訳無いです。現在は修正したので、また何か間違いがあれば修正いたします。

また、なるべく早く正確な情報に訂正するため、このページではわかりやすさや細かい説明などは省いて更新しています。
:::

Task queue (Macrotask queue) へと供給される Task は Task source と呼ばれる供給源があり、それは以下のように複数存在しています。

Generic task sources
- The DOM manipulation
- The user interaction
- The networking
- The history traversal

参考
https://html.spec.whatwg.org/multipage/webappapis.html#generic-task-sources

このようにタスク(マクロタスク)はそれぞれ分離した供給源が複数個あります。そして重要なこととして、Task queue には２つの制約があります。

- 「**同一のソースから運ばれるタスクは同一のキューに属さなければならない**」
- 「タスクはすべてのキューににおいて挿入された順番に処理されなければならない」

通常、`setTimeout()` で非同期処理を考えたりしますが、`setTimeout()` で Task queue へと送ったタスク(マクロタスク)はすべて同一のキューへと配置されます。その一方で、ユーザーのクリック操作などから送られてくるタスクは `setTimeout()` でタスクを送ったキューとは別のキューへと配置されます。

そして重要なこととして、Event loop の 1 回のループでは複数ある Task queue から１つのみを選択してその中で最も古いタスク(キューの頭にあるもの)を実行します。

複数個の Task queue があるため、どれを選ぶかということが重要になってきますが、User agent が自由に選択します。従って、開発者側が正確なタイミングで特定の実行することはできません。

ブラウザ環境はユーザー入力などのパフォーマンスに敏感なタスクを優先的に処理しようとして Event loop 内にある複数の Task queue から関連するキューを選びだし、その中にあるタスクをを処理します。つまり、`setTimeout()` で登録しておいたコールバックの処理よりもマウスクリックなどの操作のほうが優先度の高い場合があり、`setTimeout()` で登録したコールバックが処理されるのが遅延することがあります。

結局のところ、ブラウザ環境では Event loop において考えるべき大きな仕事は次の３つがあることになります。
- Task (Macrotask)
- Microtask
- Rendering の更新

便宜上、最後の Rendering も Event loop において残り２つと同等のものとして考えています。

# Event loop のステップ１とステップ２は実質的に同じ
JS Visualizer を使い始めた初期段階では次のコードが理解できなくなります。実際に実行してみてください。

```js
// Macrotask vs Microtask 
setTimeout(() => console.log('timeout'), 0);
Promise.resolve('promise resloved').then(res => console.log(res))
```

- [Macrotask vs Microtask - JS Visualizer](https://www.jsv9000.app/?code=Ly8gTWFjcm90YXNrIHZzIE1pY3JvdGFzawpzZXRUaW1lb3V0KGZ1bmN0aW9uIGEoKSB7fSwgMCk7CgpQcm9taXNlLnJlc29sdmUoKS50aGVuKGZ1bmN0aW9uIGIoKSB7fSk7)

Event loop を４ステップで考えると、出力は "timeout" → "promise resolved" になるはずですが、実際はその逆で "promise resolved" → "timeout" になります。そしてこの実行順番はブラウザ環境と Node でも Deno でも同じになります。

本来なら Task(Macrotask) が先に実行されるべきなのに、Microtask が先に実行されているため、これによってステップ２の「単一の Task(Macrotask) の実行」がスキップされているように思えてしまいます。

最初、自分はこのこのに気付いてさえいなかったのですが、よくよく考えるとおかしいと感じて調査した結果、どうやらステップ１の「スクリプトの評価」自体が Task(Macrotask) として行われているらしいという結論に至りました。現時点で情報が不正確なのが申し訳無いですが、なにせ Event loop の学習ソースとなる記事が少ないので「どうやらそうらしい」という結論になってしまいました。

:::message
JavaScript の非同期処理の学習はこういう感じで ECMAScript の言語機能だけでなく、環境の実装や「**謎**」を解いていく必要があるので、本当に難しいです。JavaScript 自体の学習だけでは永遠に非同期処理が分かりません。
:::

例えば、次の記事内のデモでは、最初の「スクリプトの評価」もしくは「スクリプトの実行」自体が実質的に Task となっており、Call stack に `script` がプッシュされています。

https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/

JSConf.Asia での Jake Archibald 氏による講演動画『In The Loop』においても、`Script` がコールスタックに載っており、`Script` が Call Stack から pop されて初めて Microtask が実行されています。

↓ 動画の 31:33 ~ のところ。
@[youtube](cCOL7MC4Pl0)

あとは見つけることができたものとして次の Q&A サイトがあります。

https://dev-qa.com/2379514/does-event-loop-skip-first-task-and-straight-the-microtasks

>You won't believe it, but the code that runs when you load the script is a separate task. It goes on the task stack and runs until the stack is empty. Then all accumulated microtasks are executed, and then the next task is moved to the stack from the task queue.
>(上記サイトのページから引用)

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

やはり、「ステップ１とステップ２は同質のものであり、実質的にスクリプトの評価自体が最初の Task(Macrotask)である」と考えるとすべて辻褄があうので、この本では「Event loop のステップ１とステップ２は実は同質のものであり、スクリプトの評価自体が最初の Task(Macrotask) となる」を真として話を進めていきます。

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

