---
title: "タスクキューとマイクロタスクキュー"
---

# このチャプターについて

このチャプターでは、タスク(Task)とマイクロタスク(Microtask)、そしてそれらをイベントループで処理するために必要なタスクキュー(Task queue)とマイクロタスクキュー(Microtask queue)について解説してきます。

:::message
各環境での具体的なイベントループについては別のチャプター『それぞれのイベントループ』で詳しく解説していますので、そちらを参照してください。
:::

Masaki Hara 氏の記事で解説されている図を参考にすると非常に頭が整理されると思いますので、参考にしてください。

https://zenn.dev/qnighy/articles/345aa9cae02d9d

:::message
この記事はイベントループの仕様に基づいているのでかなり正確な情報だと思います。私の本よりよほど情報信頼性があるので、この本で情報の足りない部分があったり困ったことがあったらこちらの記事を参考にするのがよいと思います。ただし、仕様に近いことも有り(ほぼ仕様)、ある程度のメンタルモデルができていないと最初は難しいかもしれません。
:::

このチャプターではなるべく仕様に沿った情報で考えていきます。

https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3

# イベントループとはそもそも何？

まずイベントループの定義から考えていきます。

>To coordinate events, user interaction, scripts, rendering, networking, and so forth, user agents must use event loops as described in this section. Each agent has an associated event loop, which is unique to that agent.
>(上記ページより引用)

イベントループとは、**イベントやユーザーインタラクション、スクリプト、レンダリング、ネットワーキングなどをまとめ上げて調整するために**、ユーザーエージェントが使用しなくてはならないものであると述べられています。

# タスクキュー

>An event loop has one or more task queues. A task queue is a set of tasks.
>(上記ページより引用)

イベントループは１つ以上のタスクキュー(Task queue)を所有し、タスクキューはタスク(Task)の集合である、ということが述べられています。重要なこととして、タスクキューは１つではなく、複数存在してもよいといことが分かります。

>Task queues are sets, not queues, because the event loop processing model grabs the first runnable task from the chosen queue, instead of dequeuing the first task.
>(上記ページより引用)

複数のタスクキューはキューではなく集合となっていることがわかります。イベントループは複数のタスクキューからから１つのタスクキューを選び、そのキューの先頭にある最も古いタスクを処理しなくてはいけませんが、複数あるタスクキューからどのタスクキューを選ぶかというのは実装側、つまり環境が考慮します。

:::message
Node 環境では、フェーズ(Phase)という概念があり、６つあるフェーズのそれぞれに結びついたタスクキュー(キューに準ずるもの)からタスクを処理するようになっています。Node 環境では、この６つあるフェーズを１つずつ選択して順番にフェーズを経過して一周することでイベントループの一周となります。つまり、タスクキューが６つあるとして考えられます(実際にタスクキューとして機能するのは４つ程度)。６つあるタスクキューは順番に選択されて順番に処理されるようになっています。

ブラウザ環境では、レンダリングエンジンである Blink がタスクキューの選択を行っており、それぞれのタスクキューの優先度を振り分けています。
:::

# タスク

>Tasks encapsulate algorithms that are responsible for such work as:

タスクキューにプッシュされるタスク(Task)は、以下のような作業の責務を持つアルゴリズムをカプセル化するものです。

:::message
タスク(Task)はマクロタスク(Macrotask)とも呼ばれることがあります。
:::

- イベント(Event)
- パース(Parsing)
- コールバック(Callbacks)
- リソースの使用(Using a resource)
- DOM 操作への反応(Reacting to DOM manipulation)

そして、タスクには供給されるタスクソース(Task source)というものがあり、似た種類のタスクは同一のタスクキューへとプッシュされます。

>Essentially, task sources are used within standards to separate logically-different types of tasks, which a user agent might wish to distinguish between. Task queues are used by user agents to coalesce task sources within a given event loop.

タスクキューで見てきた通り、タスクキューは１つ以上存在できるので、もちろんタスクキューにタスクを供給してくる供給源であるタスクソースも複数あります。上記に上げたタスクはそれぞれ異なるタスクソースで、異なるタスクキューにタスクを供給すると考えてよいでしょう。

タスクキューで重要なルールは以下のものとなります。

- (1) 複数あるタスクキューはどの順番に処理されるか決められていない。
- (2) 同一のタスクキュー内に存在しているタスクは到着した順番に処理される
- (3) 同一の供給源から来たタスクは同じタスクキューへと送られる

(1) については、環境が独自のルールで決めます。Chrome ブラウザ環境の場合は、レンダリングエンジン Blink がマウスクリックなどから発火するイベントを優先的に処理したり、それぞれのタスクキューのスケジューリングをしています。Node の場合はフェーズで順繰りに処理するようになっています。
(2) については、キューなので FIFO(First In First Out)であるということです。
(3) については、タイマー系の API からくるコールバックはすべて同一のタスクキューへ、クリックイベントなどからくるコールバックは別のキューへ行くようになっています。ただし、Node の API である `setImmediate()` のコールバックは Check phase の専用のキューへと行きます。

タスク(Task)を発行する非同期 API はいくつか存在しますが、代表的なものとしてはタイマー処理を行う次の API が挙げられます。

- `setTimeout()`
- `setInterval()` 

## setTimeout API

`setTimeout(cb, delay)` は指定時間が経過した後に、引数のコールバック関数をタスクとしてタスクキューに発行します。

```js
// Web API
const timerID = setTimeout(() => {
  console.log("⏰ TIMRES: task [Functional Execution Context]");
}, 1000);
// 1000 ミリ秒後にタイマー用タスクキューにタスクを発行する
```

戻り値は、タイマーの ID となります。`clearTimeout(timerID)` を使ってタイマーを解除します。

`setTimeout()` API の遅延時間を 0 秒にすることでタスクキューへタスクを簡単に発行できますが、あくまでタイマー処理であり、０ミリ秒遅延は実際０ミリ秒にはならず１ミリ秒以上の時間がかかることに注意してください。

# イベントループの所有物

続いて仕様にはイベントループが所有するものについてもいくつか定義されています。理解する上で重要な部分について触れていきます。

>Each event loop has **a currently running task**, which is either a task or null. Initially, this is null. It is used to handle reentrancy.

>Each event loop has **a microtask queue**, which is a queue of microtasks, initially empty. A microtask is a colloquial way of referring to a task that was created via the queue a microtask algorithm.

イベントループは Currnetly running task (現在実行中のタスク) を持ち、さらに**単一の**マイクロタスクキュー(Microtask queue)を持つというように定義されていますね。

仕様の『spin the event loop』のところでは、次のように記載されています。

https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model:currently-running-task-5

>1. Let task be the event loop's currently running task.

>task could be a microtask.

イベントループの開始時にタスクをイベントループの Currently running task として扱いますが、この場合マイクロタスクであってもよいと書かれているので、現在実行中のタスクもマイクロタスクも Currnetly running task として考慮できます。従って、本質的にはタスクもマイクロタスクも同じ扱いで、コールスタック上にプッシュされた実行コンテキストであると理解できます。

# マイクロタスクキュー

次に、マイクロタスクキューについて深堀りしましょう。まずマイクロタスクキュー(Microtask queue)はタスクキューではありません。

>The microtask queue is not a task queue.

上で見たとおり、イベントループはマイクロタスクキューを１つ持つといっていますね。

:::message
本来、１つのイベントループには１つのマイクロタスクキューしか無いはずですが、Node 環境は歴史的経緯から、Promise の機能をしっかりと取り入れる前に、もう１つのマイクロタスクキューとして `nextTickQueue` を導入しています。

そのキューにマイクロタスクを発行する `process.nextTick()` という API は現在では非推奨とはいかないまでも、代わりにデファクトスタンダードな API として `queueMicrotask()` を使用するように推奨しています。つまり、通常のマイクロタスクキューのみを利用したほうが良いということです。

Deno 環境では、基本的にはマイクロタスクキューが１つですが、Node のコードに対して互換性のある動きができるように、ポリフィルを使って、`nextTickQueue` が使えるようにできます。基本的にはマイクロタスクキューは１つとして考えてよいでしょう。
:::

マイクロタスクキューはタスクキューよりも優先的に処理されます。単一タスクが終わったら、すべてのマイクロタスクを処理するというのはそういうことです。node の `nextTickQueue` にあるのもマイクロタスクですが、単一タスクが終わったら必ずマイクロタスクキューにあるすべてのマイクロタスクが処理されます。マイクロタスクキューで重要なことはこれだけです。

# マイクロタスク

マイクロタスクキューに送られるマイクロタスクについて考えてみます。MDN のドキュメントにはこう書かれています。

>A microtask is a short function which is executed after the function or program which created it exits and only if the JavaScript execution stack is empty, but before returning control to the event loop being used by the user agent to drive the script's execution environment.
>([Using microtasks in JavaScript with queueMicrotask() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)より引用)

マイクロタスク自体は、それを呼び出し関数やプログラムが実行された後に**コールスタックが空になった後にのみ実行される短い関数**です。API や Promise の `then()` メソッドなどの引数に渡すコールバック関数がマイクロタスクとして扱われます。

`Promsie.resolve().then(callback1).catch(callback2).finally(callback3)` のように、Promise のプロトタイプメソッドである、`then()`, `catch()`, `finally()` の引数に渡すコールバック関数はすべてマイクロタスクとしてマイクロタスクキューへと送られます。

基本的にはマイクロタスクを作成するのは、 Promise 関連の処理ですが、他にもマイクロタスクをマイクロタスクキューに追加する API が存在しています。

- queueMicrotask
- Mutation Observer

## queueMicrotask API

`queueMicrotask(cb)` は引数のコールバック関数をマイクロタスクとしてマイクロタスクキューに発行しますが、**戻り値は何もない**ことに注意してください。つまり、`promsie.then()` のように、Promise インスタンスを返しません。イベントループにおけるマイクロタスクの挙動などをテストしたり、その他のコールバックが処理される前のクリーンアップなどに役に立ちます。

```js
// Web API
queueMicrotask(() => {
  console.log("👦 MICRO: microtask [Functional Execution Context]");
});
// ただちにマイクロタスクキューにマイクロタスクを発行する
```

https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask

単にマイクロタスクを作成したいだけなら、`Promise.resolve().then()` よりも `queueMicortask()` を使用することが推奨されます。`queueMIcrotask()` を使用することで、マイクロタスクを作成するためにプロミスを使うことで発生するオーバーヘッドなどを回避できます。

https://developer.mozilla.org/ja/docs/Web/API/HTML_DOM_API/Microtask_guide#enqueueing_microtasks

`queueMicrotask()` はブラウザ環境で提供される Web API ですが、Node でも Deno でも使用できます。

https://nodejs.org/api/globals.html#queuemicrotaskcallback

https://doc.deno.land/deno/stable/~/queueMicrotask

上で説明したとおり、Node 環境では `process.nextTick()` API よりも `queueMicrotask()` API の使用が推奨されます。

## Mutation Observer API

Mutation Oberver は DOM 内の要素を監視して、何かの変更があった際にコールバックをマイクロタスクとして発火する Web API です。

```js
const observer = new MutationObserver(callback);
const myElement = document.getElementById('segosaurus');
observer.observe(myElement, ({ subtree: true }));
```

https://developer.mozilla.org/ja/docs/Web/API/MutationObserver/MutationObserver

# 最初のタスク

タスクについては、仕様だけでは理解しづらい部分があるので MDN のドキュメントを見てみましょう。特に最初のタスクが何になるかは誤解しやすいので非同期処理の予測で重要です。

https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth

>A task is any JavaScript scheduled to be run by the standard mechanisms such as initially starting to execute a program, an event triggering a callback, and so forth. Other than by using events, you can enqueue a task by using `setTimeout()` or `setInterval()`.

タスク(Task)とは、**プログラムの実行開始**や**イベントがコールバックをトリガーする**などの**標準的なメカニズムにより実行されるようにスケジューリングされた JavaScript のこと**である、と述べられています。

:::message
MDN のドキュメントでは、以下のような旨が述べられています。

>JavaScript とは、もともとがシングルスレッドの言語であり、それが開発された時代ではその選択がベターなものでしたが、時代と共にコンピュータの性能そのものが向上したことや、JavaScript が様々なアプリケーションにて使われるようになったことから、シングルスレッド言語の限界を超えるための機能が必要とされるようになりました。**`setTimeout()` や `setInterval()` といった Web API スケジューリング機能をブラウザ環境の機能として提供することから始まり**、徐々にタスクスケジューリングとマルチスレッドアプリケーション開発を可能とさせる強力な機能を提供するようになりました。

つまり、Promise によるマイクロタスクなどの機能よりも前に導入された標準的なメカニズム(古いタイプのメカニズム)であることが示唆されています。
:::

https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide#tasks_vs_microtasks

具体的にタスクがタスクキューに追加されるのは以下の時です。

- 新しい JavaScript のプログラムやサブプログラムが**直接的に実行される時**(**コンソールから**や、`<script>` 要素内のコードを実行するなどの形式で)
- イベントが発火し、イベントのコールバック関数がタスクキューへと追加する時
- `setTimeout()` や `setInterva()` で作成されたタイムアウトやインターバルの時間が経過し、登録しておいたコールバックがタスクキューへと追加される時

重要なこととして、イベントループにおいて最初のタスクはプログラムの実行開始そのものであり、コンソールから実行するときや、スクリプトタグのコードを実行する際に同期処理の部分はまとめてタスクとして実行されます。

例えば、ブラウザ環境で HTML ファイルに次のようなスクリプトタグがあった場合、ブラウザはスクリプトタグをパースして、タスクを作成し、同期処理の部分は実際にタスクとして処理されます。`addEventListener()` のコールバック部分については、ブラウザがキーダウンイベントを受け取った時に別のタスクとして実行されます。

```html
<script>
  // <-- task 1
  console.log("Sync process start: [Global Execution Context]");

  const foo = bar;
  foo.doSomething();
  
  document.body.addEventLlistener('keydown', (event) => {
    // <-- task2
    if (event.key === 'PageDown') {
      location.href = "/#/36";
    }
    console.log("Event fired (Async process): [Functional Execution Context]");
    // task2 -->
  });

  console.log("Sync process End: [Global Execution Context]");
  // task1 -->
<script>
```

チャプター『コールスタックと実行コンテキスト』で説明したようにコードが実行開始されると、グローバルコンテキスト(Global Execution Context)が作成されて、コールスタック(Call stack)上にそのグローバルコンテキストが積まれます。そして、同期処理の関数呼び出しなどはすべてこのグローバルコンテキストに積まれることで実行されていきます。同期処理部分がすべて実行されて、このグローバルコンテキストが破棄された後で、ある時間にキーダウンイベントなどをブラウザが受け取ると、Web API がそのイベントを受け取って `addEventListener()` で登録しておいたコールバックを別のタスクとしてイベント用のタスクキューへと送ります。

イベントループはそのタスクが存在していることを確認し、元々スクリプトタグが評価された時点からイベントが発火されるまで時間がたっていたとしても、そのコールバック関数により作成される関数実行コンテキストをコールスタック上に配置して、Ruuning Execution Context として実行します(その際には、タスクを Currently running task として扱わています)。

このように、`<script>` タグの読み込みから、スクリプト内の同期処理をすべて処理するまで最初のタスクとして考えることができます(もちろん内部で色々なプロセスを行っているのでしょうが)。

```html
<script>
  // <- Task1
  // ...
  // Task 1 ->
</script>
```

従って、次のように複数の `<script>` タグがあった場合も、それぞれのスクリプトの評価がタスクとして扱われます。これが理解できていると、次のように２つスクリプトタグがあったときの実行順番が予測できます。

```html
<script>
  // <- Task 1
  console.log("🦖 [1] MAINLINE: script1 start [Global Execution Context]");
  setTimeout(()=>{
    // <- Task 3
    console.log("⏰ [9] TIMERS: settimeout [Functional Execution Context]");
    // Task 3 ->
  },1000)
  new Promise((resolve)=>{
    console.log("😅 [2] Sync callback script1:promise1 [Functional Execution Context]");
    resolve()
  }).then(()=>{
    // <- Microtask 1
    console.log("👦 [4] MICRO: async callback script1:then1 [Functional Execution Context]");
    // Microtask 1 ->
  })
  console.log("🦖 [3] MAINLINE: script1 end [Global Execution Context]");
  // Task 1 ->
</script>
<script>
  // <- Task 2
  console.log("🦖 [5] MAINLINE: script2:start [Global Execution Context]");
  new Promise((resolve)=>{
    console.log("😅 [6] MAINLINE: (Sync callback) script2:promise2 [Functional Execution Context]");
    resolve()
  }).then(()=>{
    // <- Microtask 2
    console.log("👦 [8] MICRO: (async callback) script2:then1 [Functional Execution Context]");
    // Microtask 2 ->
  })
  console.log("🦖 [7] MAINLINE: script2:end [Global Execution Context]");
  // Task 2 ->
</script>
```

コンソールでこのように出力されます。
![](/images/js-async/img_doubleScriptTag.jpg)

結局のところ、JavaScript コードはすべてタスクかマイクロタスクになります。タスクとして実行される JavaScript コードは、このように script であるか、非同期のコールバック関数のどちらかになります。

ランタイム環境でも同じです。Node も Deno も Chrome と同じ V8 エンジンを積んでおり、コールスタックは V8 エンジンが搭載しているのでまったく同じ用に考えることができます。プログラムの開始時のスクリプト評価で同期処理はすべてまとめてタスクとしてカウントして、グローバルコンテキストを作成し、コールスタック上に配置されます。同期処理がすべて終わるとグローバルコンテキストはコールスタックからポップして破棄されます。そして単一タスクの後はすべてのマイクロタスクを処理するので、同期処理が終わり次第マイクロタスクが処理されます。別の言い方で言えば、コールスタックが空になったらマイクロタスクを実行します。

これが理解できることで、次のスクリプトの実行順番も理解できます。

```js
// blockingTimer.js
// <-- Task 1

// メインスレッドを同期的にブロッキングする関数
function pause(milliseconds, order) {
  const dt = new Date();
  while ((new Date()) - dt <= milliseconds) {
    // 何もしない
  }
  console.log(`🦖 ${order} Sync process: timer exit after ${milliseconds}`);
}

console.log("🦖 [1] MAINLINE: Sync process start");

setTimeout(() => {
  // Task 2
  console.log("⏰ [5] TIMERS: setTimeout[0ms] finished");
}, 0);
setTimeout(() => {
  // Task 3
  console.log("⏰ [6] TIMERS: setTimeout[1000ms] finished");
}, 1000);
queueMicrotask(() => {
  // Microtask
  console.log("👦 [4] MICRO: queueMicrotask");
});

pause(3000, "[2]"); // 3000[ms] メインスレッドをブロッキング

console.log("🦖 [3] Sync process end");
// Task 1 -->
```

プログラム実行開始のスクリプトの評価が最初のタスク(Task)となり、同期処理がすべて終われば、単一タスクが実行されたこととになるので、次はマイクロタスクをすべて処理するのがイベントループです。従って、Deno でも Node でも実行結果は同じになります。

```sh
# Deno 環境
❯ deno run blockingTimer.js
🦖 [1] MAINLINE: Sync process start
🦖 [2] Sync process: timer exit after 3000
🦖 [3] Sync process end
👦 [4] MICRO: queueMicrotask
⏰ [5] TIMERS: setTimeout[0ms] finished
⏰ [6] TIMERS: setTimeout[1000ms] finished
# Node 環境
❯ node blockingTimer.js
🦖 [1] MAINLINE: Sync process start
🦖 [2] Sync process: timer exit after 3000
🦖 [3] Sync process end
👦 [4] MICRO: queueMicrotask
⏰ [5] TIMERS: setTimeout[0ms] finished
⏰ [6] TIMERS: setTimeout[1000ms] finished
```

:::message alert
これが理解できるまで時間がかかりました。JavaScript Visualizer 9000 ではグローバルコンテキストが積まれることは認識できませんし、最初のタスクがプログラムの開始実行(すべての同期処理)になることも理解しづらいので、情報を補う必要があったわけです。
:::

