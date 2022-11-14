---
title: "タスクキューとマイクロタスクキュー"
cssclass: zenn
date: 2022-05-06
modified: 2022-11-14
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: ch_タスクキューとマイクロタスクキュー
---

# このチャプターについて

このチャプターでは、タスク(Task)とマイクロタスク(Microtask)、そしてそれらをイベントループで処理するために必要なタスクキュー(Task queue)とマイクロタスクキュー(Microtask queue)について解説してきます。

:::message
各環境での具体的なイベントループについては別のチャプター『[それぞれのイベントループ](c-epasync-what-event-loop)』で詳しく解説していますので、そちらを参照してください。
:::

Masaki Hara さんの記事で解説されている図も参考にすると頭が整理されると思いますので、参考にしてください。

https://zenn.dev/qnighy/articles/345aa9cae02d9d

このチャプターではなるべく仕様に沿った情報で考えていきます。実際の仕様については次の URL から確認してください。

https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3

# イベントループとはそもそも何？

『What the heck is the event loop anyway?』の動画で、イベントループの概略自体はつかめていると思いますが、その定義から考えていきます。

> To coordinate events, user interaction, scripts, rendering, networking, and so forth, user agents must use event loops as described in this section. Each [agent](https://tc39.es/ecma262/#sec-agents) has an associated event loop, which is unique to that agent.
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3) より引用)

イベントループとは、**イベントやユーザーインタラクション、スクリプト、レンダリング、ネットワーキングなどをまとめ上げて調整するために**、ユーザーエージェントが使用しなくてはならないものであると述べられています。

イベントループは以下で解説するタスクキューとマイクロタスクキューを所有しています。

# タスクキュー

:::message alert
この項目には筆者の勘違いがあったので、修正致しました。

コミュニティ用のスクラップで指摘していただいた点について修正しています。詳しい修正点については[スクラップ](https://zenn.dev/link/comments/1b145433c5b5df)の方を確認してください。
:::

WHATWG の仕様において、イベントループは１つ以上のタスクキュー(Task queue)を所有していると述べられています。つまり、**タスクキューは１つではなく、複数個存在してもよい**ということが分かります。

> An [event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop) has one or more task queues.
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue) より引用)

タスクキュー(Task queue)とは、文字通りタスクのキューであるとここでは考えてください。ただし、後述しますが、仕様上のタスクキューは厳密にはキュー(Queue)ではなくセット(Set)というデータ型です。

ここで重要な話として、イベントループは複数のタスクキューから実行可能なタスクが少なくとも一つ以上あるようなタスクキューを一つ選択した上で、そのキュー内に存在する最初の実行可能なタスクを処理しなくてはいけませんが、複数あるタスクキューからどのタスクキューを選ぶかというのは **実装側(つまり環境側)が定義する方法** で考慮します。

これについては、WHATAG 仕様の [Processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) の項目にて述べられています。

> 2. If the [event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop) has a [task queue](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue) with at least one [runnable](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task-runnable) [task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task), then:
>     1. Let taskQueue be one such [task queue](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue), chosen in an [implementation-defined](https://infra.spec.whatwg.org/#implementation-defined) manner.
>
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) より引用)

この Processing model がイベントループが実際に行うことです。この本ではすべての詳細を触れずに擬似コードで解説するので、ブラウザ環境のイベントループの詳細を詳しく知りたい場合にはこの Processing model を参照してください。

:::message
Node 環境では、フェーズ(Phase)という概念があり、６つあるフェーズのそれぞれに結びついたタスクキュー(あるいは準ずるもの)からタスクを処理するようになっています。Node 環境では、この６つあるフェーズを１つずつ選択して順番にフェーズを経過して一周することでイベントループの一周となります。つまり、タスクキューが６つあるとして考えられます(実際にタスクキューとして機能するのは４つ程度)。６つあるタスクキューは順番に選択されて順番に処理されます。

ブラウザ環境では、レンダリングエンジンである Blink がタスクキューの選択を行っており、それぞれのタスクキューの優先度を振り分けています。
:::

## タスクキューとは Set である

タスクキューは名前上はタスクのキュー(Queue)となっていますが、実際にはタスクの Set である、ということが仕様では述べられています。

> A [task queue](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue) is a [set](https://infra.spec.whatwg.org/#ordered-set) of [tasks](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task).
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue) より引用)

ここで言う Queue や Set とは特定のデータ構造(data strature)のことです(以前の解説では、この点についてただの集合であると誤解していました)。データ構造については WHATAG 仕様に定義されています。具体的には HTML Standard ではなく、それらの仕様が基づく用語や概念を定義している [Infra Standard](https://infra.spec.whatwg.org/) というページに記載されています。

Queue や Set は Infra Standard の [5. Data structures](https://infra.spec.whatwg.org/#data-structures) の項目に記載されています。このページを見ると、Queue や Set とは List という仕様におけるデータ型の一つであることがわかります。List には他にも Stack というデータ構造が定義されています。

- List
  - Stack:
  - Queue: ← マイクロタスクキュー
  - Set: ← タスクキュー

ここで、List とは以下のようなものであると定義されていて、有限個の要素からなる順序付きの列からなる仕様の型であることが述べられています。日本語訳版だと「[有限個の アイテム （ item ）からなる有順序連列](https://triple-underscore.github.io/infra-ja.html#lists)」となっています。

> A list is a specification type consisting of a finite ordered sequence of items.
> ([Infra Standard](https://infra.spec.whatwg.org/#lists) より引用)

List は上で述べたように Stack や Queue や Set である可能性があります。言い換えれば、これらのデータ構造は List の派生となります。

問題となるタスクキューは Queue ではなく、Set です。この Set は List でもあるので、順序が付いた列です。定義的には、順序集合([ordered set](https://ja.wikipedia.org/wiki/%E9%A0%86%E5%BA%8F%E9%9B%86%E5%90%88#%E5%89%8D%E9%A0%86%E5%BA%8F%E3%83%BB%E5%8D%8A%E9%A0%86%E5%BA%8F%E3%83%BB%E5%85%A8%E9%A0%86%E5%BA%8F))と呼ばれるものであり、同一のアイテムを重複してもたない List です。

> Some [lists](https://infra.spec.whatwg.org/#list) are designated as ordered sets. An ordered set is a [list](https://infra.spec.whatwg.org/#list) with the additional semantic that it must not contain the same [item](https://infra.spec.whatwg.org/#list-item) twice.
> ([Infra Standard](https://infra.spec.whatwg.org/#sets) より引用)

仕様ではタスクキューがその名前とは裏腹に Queue ではなく Set である理由が述べられています。

> [Task queues](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue) are [sets](https://infra.spec.whatwg.org/#ordered-set), not [queues](https://infra.spec.whatwg.org/#queue), because the [event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) grabs the first [_runnable_](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task-runnable) [task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task) from the chosen queue, instead of [dequeuing](https://infra.spec.whatwg.org/#queue-dequeue) the first task.
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3) より引用)

イベントループの Processing model では、上で述べたようにまずは複数個ありえるタスクキューの中から環境定義の方法で一つのタスクキューを選択しますが、そのタスクキューの中から最初のタスクを選択(dequeuing)するのではなく、実行可能(runnable)という状態である最初のタスクを選択して処理します。

Set は List でもあるので順序がありますが、その中で実行可能(runnable)かつ順序的に最初のタスクが処理されるということです。ということで、ほとんど Queue と同じような処理となりますが、実行可能ではないものはとばされてしまうということになります。

ちなみに後で解説するマイクロタスクキュー(Microtask queue)は厳密に Queue なので注意してください。

## タスク

タスク(Task)は、タスクキュー(Task queue)にプッシュされるものですが、タスクそれ自体がなにかを仕様から見ていきます。

まず、タスクの実態は形式的に以下のものを所有する構造体([struct](https://infra.spec.whatwg.org/#structs): Infra Standard で定義された仕様型の１つ)であるとも仕様で定義されています。

- ステップ(Steps)
  - 当該のタスクによって処理される仕事を指定する一連のステップ
- ソース(A source)
  - 当該のタスクに関連するタスクをグループ化してシリアライズするために利用されるタスクソースの１つ
- ドキュメント(A document)
  - 当該のタスクに関連付けられた [Document](https://html.spec.whatwg.org/multipage/dom.html#document)(DOM が定義している Document インターフェイス)、あるいは Window event loop にないタスクの場合の null
- スクリプト評価の環境設定オブジェクトの Set (A script evaluation environment settings object set)
  - タスク実行中のスクリプト評価を追跡するために使用される [Environment settings object](https://html.spec.whatwg.org/multipage/webappapis.html#environment-settings-object) の [Set](https://infra.spec.whatwg.org/#sets)

> Tasks encapsulate algorithms that are responsible for such work as:
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3) より引用)

実態は上記のような構造体(struct)ですが、タスク(Task)は以下のような作業の責務を持つ**アルゴリズムをカプセル化**します。

- イベント(Events)
- パース(Parsing)
- コールバック(Callbacks)
- リソースの使用(Using a resource)
- DOM 操作への反応(Reacting to DOM manipulation)

ここでは、上記のような操作(あるいはアルゴリズム)そのものがタスクであると考ればよいです。わかりやすく例をあげると、後述する `setTimeout()` API に登録されたコールバック関数はタスクの１つであり、クリックイベントなどもタスクの１つです。注意点として、スクリプトのパースなどもタスクの一種です。これは『[最初のタスク](#最初のタスク)』の項目で説明します。

:::message alert
WHATWG の仕様的には後述するマイクロタスク(Microtask)はタスク(Task)の一種なので注意してください。
:::

### タスクソース

タスクの説明にあったように、タスクには供給されるタスクソース(Task source)というものがあり、似た種類のタスクは同一のタスクキューへとプッシュされます。

> Essentially, [task sources](https://html.spec.whatwg.org/multipage/webappapis.html#task-source) are used within standards to separate logically-different types of tasks, which a user agent might wish to distinguish between. [Task queues](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue) are used by user agents to coalesce task sources within a given [event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop).
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-for-spec-authors) より引用)

タスクキューで見てきた通り、タスクキューは１つ以上存在できるので、もちろんタスクキューにタスクを供給してくる供給源であるタスクソースも複数あります。上記に上げたタスクはそれぞれ異なるタスクソースで、異なるタスクキューにタスクを供給すると考えてよいでしょう。

タスクキューで重要なルールは以下のものとなります。

- (1) 複数あるタスクキューはどの順番に処理されるか決められていない。
- (2) 同一のタスクキュー内に存在しているタスクは到着した順番に処理される
- (3) 同一の供給源から来たタスクは同じタスクキューへと送られる

(1) については、環境が独自のルールで決めます。Chrome ブラウザ環境の場合は、レンダリングエンジン Blink がマウスクリックなどから発火するイベントを優先的に処理したり、それぞれのタスクキューのスケジューリングをしています。Node の場合はフェーズで順繰りに処理されます。  
(2) については、キューなので FIFO(First In First Out)であるということです。  
(3) については、タイマー系の API からくるコールバックはすべて同一のタスクキューへ、クリックイベントなどからくるコールバックは別のキューへ行くようになっています。ただし、Node の API である `setImmediate()` のコールバックは Check phase の専用のキューへと行きます。  

タスク(Task)を発行する非同期 API はいくつか存在しますが、代表的なものとしてはタイマー処理を行う次の API が挙げられます。

- `setTimeout()` API
- `setInterval()` API

### setTimeout API

タスクベースの非同期 API である、`setTimeout()` は、`setTimeout(cb, delay)` というように指定した遅延時間が経過した後に、引数のコールバック関数をタスクとしてタスクキューに発行します。

```js
// タスクベースの非同期 API
const timerId = setTimeout(() => {
  console.log("⏰ TIMRES: task [Functional Execution Context]");
}, 1000);
// 1000 ミリ秒後にタイマー用タスクキューにタスクを発行する
```

戻り値は、`setTimeout()` で作成したタイマーを識別するための ID で 0 でない正の整数値です。`clearTimeout(timerId)` を使ってタイマーを解除できます。

https://developer.mozilla.org/ja/docs/Web/API/setTimeout

`setTimeout()` API の遅延時間を 0 秒にすることでタスクキューへタスクを簡単に発行できますが、あくまでタイマー処理であり、０ミリ秒遅延は実際０ミリ秒にはならず１ミリ秒以上の時間がかかることに注意してください。また、タスクキューに他のタスクがある場合には先に存在していたタスクが処理されるので、その場合もタスク処理が指定時間よりも遅く処理されることになります。

```ts:Denoでの型定義
function setTimeout(
  cb: (...args: any[]) => void,
  delay?: number,
  ...args: any[],
): number;
```

https://doc.deno.land/deno/stable/~/setTimeout

### setInvertal API

タスクベースの非同期 API である、`setInterval()` は、`setInverval(cb, interval)` というように指定したインターバル時間が経過するたびに、引数のコールバック関数をタスクとしてタスクキューに発行します。

```js
// タスクベースの非同期 API
const inervalId = setInterval(() => {
  console.log("⏰ TIMRES: task [Functional Execution Context]");
}, 1000);
// 1000 ミリ秒経過するごとにタイマー用タスクキューにタスクを発行する
```

戻り値は `setInterval()` で作成したタイマーを識別するユニークな ID で、0 でない正の整数値です。`clearInterval(intervalId)` を使って以後のインターバルをキャンセルできます。

https://developer.mozilla.org/en-US/docs/Web/API/setInterval

```ts:Denoでの型定義
function setInterval(
  cb: (...args: any[]) => void,
  delay?: number,
  ...args: any[],
): number;
```

https://doc.deno.land/deno/stable/~/setInterval

# イベントループの所有物

続いて仕様にはイベントループが所有するものについてもいくつか定義されています。理解する上で重要な部分について触れていきます。

> Each [event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop) has a currently running task, which is either a [task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task) or null. Initially, this is null. It is used to handle reentrancy.
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3) より引用)

> Each [event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop) has a microtask queue, which is a [queue](https://infra.spec.whatwg.org/#queue) of [microtasks](https://html.spec.whatwg.org/multipage/webappapis.html#microtask), initially empty. A microtask is a colloquial way of referring to a [task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task) that was created via the [queue a microtask](https://html.spec.whatwg.org/multipage/webappapis.html#queue-a-microtask) algorithm.
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3) より引用)

イベントループは Currnetly running task (現在実行中のタスク) を持ち、さらに**単一の**マイクロタスクキュー(Microtask queue)を持つというように定義されていますね。

仕様の『[spin the event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model:currently-running-task-5)』の項目では、次のように記載されています。

> 1. Let task be the [event loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop)'s [currently running task](https://html.spec.whatwg.org/multipage/webappapis.html#currently-running-task).
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model:currently-running-task-5)より引用)

> task could be a [microtask](https://html.spec.whatwg.org/multipage/webappapis.html#microtask).
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model:currently-running-task-5)より引用)

イベントループの開始時にタスクをイベントループの Currently running task として扱いますが、この場合マイクロタスクであってもよいと書かれているので、現在実行中のタスクとマイクロタスクも Currnetly running task として考慮できます。従って、本質的にはタスクとマイクロタスクも同じ扱いで、コールスタック上にプッシュされた実行コンテキストであると理解できます。

# マイクロタスクキュー

次に、マイクロタスクキューについて深堀りしましょう。まずマイクロタスクキュー(Microtask queue)はタスクキューではありません。

> The [microtask queue](https://html.spec.whatwg.org/multipage/webappapis.html#microtask-queue) is not a [task queue](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue).
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3)より引用)

上で見たとおり、イベントループはマイクロタスクキューを１つ持つといっていますね。

:::message
本来、１つのイベントループには１つのマイクロタスクキューしか無いはずですが、Node 環境は歴史的経緯から、Promise の機能をしっかりと取り入れる前に、もう１つのマイクロタスクキューとして `nextTickQueue` を導入しています。

そのキューにマイクロタスクを発行する `process.nextTick()` という API は現在では非推奨とはいかないまでも、代わりにデファクトスタンダードな API として `queueMicrotask()` を使用するように推奨しています。つまり、通常のマイクロタスクキューのみを利用したほうが良いということです。

Deno 環境では、基本的にはマイクロタスクキューが１つですが、Node のコードに対して互換性のある動きができるように、ポリフィルを使って `nextTickQueue` が使えるようにできます。基本的にはマイクロタスクキューは１つとして考えてよいでしょう。
:::

マイクロタスクキューはタスクキューよりも優先的に処理されます。単一タスクが終わったら、すべてのマイクロタスクを処理するというのはそういうことです。node の `nextTickQueue` にあるのもマイクロタスクですが、単一タスクが終わったら必ずマイクロタスクキューにあるすべてのマイクロタスクが処理されます。マイクロタスクキューで重要なことはこれだけです。

## マイクロタスク

マイクロタスクキューに送られるマイクロタスクについて考えてみます。WHATWG 仕様では以下のように述べられています。

> A microtask is a colloquial way of referring to a [task](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task) that was created via the [queue a microtask](https://html.spec.whatwg.org/multipage/webappapis.html#queue-a-microtask) algorithm.
> ([HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#microtask) より引用、太字は筆者強調)

「[queue a microtask](https://momdo.github.io/html/webappapis.html#queue-a-microtask)」と呼ばれるアルゴリズムによって作成されるタスクのことを指す俗称ということです。仕様的には**マイクロソフトタスクはタスクの一種**ということですね。

仕様だとかなり分かりづらいので、MDN のドキュメントを見ると、こう書かれています。

> A microtask is a short function which is executed after the function or program which created it exits and only if the JavaScript execution stack is empty, but before returning control to the event loop being used by the user agent to drive the script's execution environment.
> ([Using microtasks in JavaScript with queueMicrotask() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)より引用)

マイクロタスク自体は、それを呼び出し関数やプログラムが実行された後に**コールスタックが空になった後にのみ実行される短い関数**です。API や Promise の `then()` メソッドなどの引数に渡すコールバック関数がマイクロタスクとして扱われます。

`Promise.resolve().then(callback1).catch(callback2).finally(callback3)` のように、Promise のプロトタイプメソッドである、`then()`, `catch()`, `finally()` の引数に渡すコールバック関数はすべてマイクロタスクとしてマイクロタスクキューへと送られます。

基本的にはマイクロタスクを作成するのは、Promise 関連の処理(`then(cb)` や async/await など)ですが、他にもマイクロタスクをマイクロタスクキューに追加する API が存在しています。

- `queueMicrotask()` API
- `MutationObserver()` API

### queueMicrotask API

`queueMicrotask(cb)` は引数のコールバック関数をマイクロタスクとしてマイクロタスクキューに発行しますが、**戻り値は何もない**ことに注意してください。つまり、`Promise.then()` のように、Promise インスタンスを返しません。イベントループにおけるマイクロタスクの挙動などをテストしたり、その他のコールバックが処理される前のクリーンアップなどに役に立ちます。

```js
// Web API
queueMicrotask(() => {
  console.log("👦 MICRO: microtask [Functional Execution Context]");
}); // 戻り値なしなので Promise chain はできない
// ただちにマイクロタスクキューにマイクロタスクを発行する
```

https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask

単にマイクロタスクを作成したいだけなら、`Promise.resolve().then()` よりも `queueMicortask()` を使用することが推奨されます。`queueMIcrotask()` を使用することで、マイクロタスクを作成するためにプロミスを使うことで発生するオーバーヘッドなどを回避できます。

https://developer.mozilla.org/ja/docs/Web/API/HTML_DOM_API/Microtask_guide#enqueueing_microtasks

Promise インスタンスが返ってこないので chain できない点以外については、基本的に `Promise.reoslve().then()` と同じように考えることができます。

```js
// qmt.js
Promise.resolve().then(() => console.log("[1] 🍎"));
queueMicrotask(() => console.log("[2] 🫐"));
Promise.resolve().then(() => console.log("[3] 🍎"));
queueMicrotask(() => console.log("[4] 🫐"));

/* 実行結果
❯ deno run qmt.js
[1] 🍎
[2] 🫐
[3] 🍎
[4] 🫐
*/
```

`queueMicrotask()` はブラウザ環境で提供される Web API ですが、Node でも Deno でも同じ名前で使用できます。

https://nodejs.org/dist/v18.2.0/docs/api/globals.html#queuemicrotaskcallback

上で説明したとおり、Node 環境では `process.nextTick()` API よりも `queueMicrotask()` API の使用が推奨されます。

```ts:Denoでの型定義
function queueMicrotask(func: VoidFunction): void;
```

https://doc.deno.land/deno/stable/~/queueMicrotask

### MutationObserver API

`MutationOberver()` コンストラクタ関数は MutationObserver API という大きなインタフェースの一部として提供されています。DOM 内の要素を監視して、何かの変更があった際にコールバックをマイクロタスクとして発火する Web API です。この API が発行するマイクロタスクは Promise とは関係なく、`MutationObserver()` 自体からも Promise ではなくオブザーバインスタンスが返ってくるので注意してください。

マイクロタスクキューは基本的に Promise 処理のための機構ですが、この Web API はその機構を利用します。

```js
// 監視対象とするノードを取得
const targetNode = document.getElementById('target');

// オブザーバーインスタンスを作成
const observer = new MutationObserver((mutationRecords) => {
  console.log("Detect mutation", mutationRecords);
});

// オブザーバーインスタンスに対象ノードとオブザーバーの設定をアタッチする
observer.observe(targetNode, ({
  childList: true,
  subtree: true,
  characterDataOldValue: true
}));
```

https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver/MutationObserver

https://ja.javascript.info/mutation-observer

# 最初のタスク

タスクについて説明しましたが、仕様だけでは理解しづらい部分がやはり多々あるので MDN のドキュメントも見てみましょう。特に最初のタスクが何になるかは誤解しやすいので非同期処理の予測で重要です。

https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth

> A task is any JavaScript scheduled to be run by the standard mechanisms such as **initially starting to execute a program**, **an event triggering a callback**, and so forth. Other than by using events, you can enqueue a task by using `setTimeout()` or `setInterval()`.
> ([上記ページ](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth) より引用、太字は筆者強調)

タスク(Task)とは、**プログラムの実行開始**や**イベントがコールバックをトリガーする**などの**標準的なメカニズムにより実行されるようにスケジューリングされた JavaScript のこと**である、と述べられています。

:::message
MDN のドキュメントである『[In depth: Microtasks and the JavaScript runtime environment](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth)』の冒頭では、以下のような旨が述べられていました。

「JavaScript とは、もともとがシングルスレッドの言語であり、それが開発された時代ではその選択がベターなものだったが、時代と共にコンピュータの性能そのものが向上したことや、JavaScript が様々なアプリケーションにて使われるようになったことから、シングルスレッド言語の制約を超えるための機能が必要とされるようになった。**`setTimeout()` や `setInterval()` といった Web API スケジューリング機能をブラウザ環境の機能として提供することから始まり**、徐々にタスクスケジューリングとマルチスレッドアプリケーション開発を可能とさせる強力な機能を提供するようになった」

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
</script>
```

チャプター『[コールスタックと実行コンテキスト](b-epasync-callstack-execution-context)』で説明したようにコードが実行開始されると、グローバルコンテキスト(Global Execution Context)が作成されて、コールスタック(Call stack)上にそのグローバルコンテキストが積まれます。そして、同期処理の関数呼び出しなどはすべてこのグローバルコンテキストに積まれることで実行されていきます。同期処理部分がすべて実行されて、このグローバルコンテキストが破棄された後で、ある時間にキーダウンイベントなどをブラウザが受け取ると、Web API がそのイベントを受け取って `addEventListener()` で登録しておいたコールバックを別のタスクとしてイベント用のタスクキューへと送ります。

イベントループはそのタスクが存在していることを確認し、元々スクリプトタグが評価された時点からイベントが発火されるまで時間がたっていたとしても、そのコールバック関数により作成される関数実行コンテキストをコールスタック上に配置して、Ruuning Execution Context として実行します(その際、タスクは Currently running task として扱われています)。

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

![output_doubleScriptTag](/images/js-async/img_doubleScriptTag.jpg)

**結局のところ、JavaScript コードはすべてタスクかマイクロタスクになります**(仕様上のマイクロタスクはタスクの一種なので注意)。タスクとして実行される JavaScript コードは、このように script であるか、非同期のコールバック関数のどちらかになります。

ランタイム環境でも同じです。Node と Deno も Chrome と同じ V8 エンジンを積んでおり、コールスタックは V8 エンジンが管理しているのでまったく同じ様に考えることができます。プログラムの開始時のスクリプト評価で同期処理はすべてまとめてタスクとしてカウントして、グローバルコンテキストを作成し、コールスタック上に配置されます。同期処理がすべて終わるとグローバルコンテキストはコールスタックからポップします。そして単一タスクの後はすべてのマイクロタスクを処理するので、同期処理が終わり次第マイクロタスクは処理されます。別の言い方で言えば、コールスタックが空になったらマイクロタスクを実行します。

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

プログラム実行開始のスクリプトの評価が最初のタスク(Task)となり、同期処理がすべて終われば、単一タスクが実行されたことになるので、次はマイクロタスクをすべて処理するのがイベントループです。従って、Deno でも Node でも実行結果は同じになります。

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
筆者はこれが理解できるまでにかなり時間がかかりました。JS Visualizer ではグローバルコンテキストが積まれることが認識できませんし、最初のタスクがプログラムの開始実行(すべての同期処理)になることも理解しづらいので、情報を補う必要があったわけです。
:::

# 非同期処理の本質

この本の結論をもう言ってしまいますが、非同期処理の本質的な仕組みは「**イベントループにおけるタスクとマイクロタスクの処理**」です。

非同期処理の起点は「非同期 API」による並列的作業です。環境がバックグラウンドで代行している非同期 API の処理(`fetch()` メソッドによるリソース取得など)の間、メインスレッドでは別の処理を行うことができます。そして、その並列的作業の処理が終わり次第、メインスレッドへと通知させて、起点となった非同期 API の処理結果を使って別のことをやります(取得したリソースのデータを加工するなど)。非同期 API を起点とした一連の作業は「**特定の順番に行うこと**」、つまり「A したら B する、B したら C する」というような「**逐次処理**」が肝になります。

その順番をうまく設定するのが開発者であり、コード内で特定の順番となるように処理のスケジューリングを行います。そして、API を起点とした一連の逐次処理のための書き方がコールバック関数や Promise chain、async/await となります。その書き方から実際に実行するためにあれやこれやをやる仕組みがイベントループであり、タスクキュー・マイクロタスクキューです。その処理の単位がタスクやマイクロタスクです。

ということで「A したら B する、B したら C する」というような逐次処理について、他の同期処理とは別のタイミングで(つまり非同期的に)それぞれ順番に実行したいなら、タスクやマイクロタスクを連鎖(chain)させます。**それぞれの処理が連鎖的に起きることで、逐次処理となります**。

:::message
非同期処理の学習においてやっかいな点は、非同期処理を理解するためには「非同期(asynchronous)」だけではなく、「同期(synchronous)」「並列(parallel)」「並行(concurrent)」「逐次(sequential)」などの概念を組み合わせて考える必要があることです。(並列だけは厳密なものではない)
:::

コールバックベースの API の利用に伴うネストされたコールバック(Callback hell)についてはいわば「**タスク連鎖(Task chain)**」といえるでしょう。イベントループ上でタスクの連鎖的処理を行うことで非同期的に逐次処理を行うことができます。

Promise chain や async/await、Promise-based API などについては Promise インスタンスを連鎖させることで非同期的に逐次処理を行えます。いわば「**マイクロタスク連鎖(Microtask chain)**」といえるでしょう。イベントループ上でマイクロタスクの連鎖的処理を行うことで非同期的に逐次処理を行うことができます。これを分かりやすくイメージできるのがまさに Promise chain であり、`promiseBasedApi().then(cb1).then(cb2).then(cb3)` というように「Promise-based API の並列的処理を起点として、それが完了したら次に `cb1` を行い、それが完了したら次に `cb2` を行い、それが完了したら次に `cb3` を行う」といった感じで、コールバック関数の処理を `then()` メソッドで連鎖させて繋ぐことで逐次処理ができます。

"タスク連鎖" や "マイクロタスク連鎖" は筆者が作成したただの造語ですが(本質的な部分を捉えるために考えた言葉です)、このようにイメージができるとイベントループのメンタルモデルが盤石になり、頭の中で処理順番がどうなるか想起できるようになると思います。

ここで語った内容も含めて「非同期処理」の全体については『[総括 - 非同期処理のまとめ](y-epasync-conclusion)』のチャプターで再度まとめておきますので安心してください。
