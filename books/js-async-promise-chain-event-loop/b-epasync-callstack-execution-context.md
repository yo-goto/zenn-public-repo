---
title: "コールスタックと実行コンテキスト"
cssclass: zenn
date: 2022-05-06
modified: 2023-02-13
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『コールスタックと実行コンテキスト』
---

## このチャプターについて

このチャプターでは、コールスタックに積まれる「**実行コンテキスト**」について知識を補います。

実行コンテキストは `this` キーワードの挙動を理解する上で実は重要だったりしますが、このチャプターではイベントループについて理解を深めるための知識として解説しています。

:::message alert
このチャプターの内容は、Promise や `setTimeout()` などを絡めて解説を行うため、後のチャプターの内容を先取りしています (出し惜しみせずにすべての知識を使っています)。理解できない場合はざっと目を通してから進めてもらって、後から戻ってきてください。
:::

## 実行コンテキストとコールスタックの関係

『What the heck is the event loop anyway?』の動画でみてもらったように、コールスタックには関数などが積まれているように見えますが、正確にはコールスタックに積まれるのは実行コンテキスト (Execution context) と呼ばれるものです。

実行コンテキストとはなんでしょうか。ECMAScript 仕様の [Execution Context](https://tc39.es/ecma262/#sec-execution-contexts) の項目をみてみると次のように記述されています。

> **An execution context is a specification device that is used to track the runtime evaluation of code** by an ECMAScript implementation. **At any point in time, there is at most one execution context per agent that is actually executing code**. This is known as **the agent's running execution context**. All references to the running execution context in this specification denote the running execution context of the surrounding agent.
>
> **The execution context stack is used to track execution contexts**. **The running execution context is always the top element of this stack**. A new execution context is created whenever control is transferred from the executable code associated with the currently running execution context to executable code that is not associated with that execution context. **The newly created execution context is pushed onto the stack and becomes the running execution context**.
> ([https://tc39.es/ecma262/#sec-execution-contexts](https://tc39.es/ecma262/#sec-execution-contexts) より引用、太字は筆者強調)

引用で示されているように、実行コンテキスト (Execution context) はコードの実行時評価を追跡するために使用される機構であり、どの時点でも実際にコードを実行している実行コンテキストはエージェント (Agent) あたり最大で１つとなります。これはエージェントの実行中実行コンテキスト (**Agent's running execution context**) として知られています。

実行コンテキストスタック (Execution context stack) は、実行コンテキスト (Execution context) を追跡するために使用され、実行中の実行コンテキスト (**Running execution context**) は常にこのスタックの最上位の要素です。新しく作成された実行コンテキスト (Execution context) はスタックにプッシュされて、実行中の実行コンテキスト (Running execution context) になります。

:::message
ちなみに ECMA の仕様で言われている実行コンテキストスタック (Execution context stack) はお馴染みのコールスタック (**Call stack**) のことを指しています。
:::

実行コンテキストがコールスタックに積まれていくものであるというのはなんとなく分かったと思いますが、理解するにはもう少し情報が必要ですね。MDN と freeCodeCamp の記事を見てみましょう。

https://www.freecodecamp.org/news/execution-context-how-javascript-works-behind-the-scenes/

https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth#javascript_execution_contexts

ブラウザでは、JavaScript コードを直接的に理解できないので、機械語に変換される必要があります。例えば、HTML のパース中に `<script>` タグを介してブラウザが JavaScript コードにエンカウントしたりすると、ブラウザはそのコードを JavaScript エンジン (Chrome なら V8) へと送信します。

V8 エンジン (JavaScript エンジン) がコードを受信すると、その JavaScript コードの変換と実行を処理するための特別な環境、つまり実行コンテキスト (Execution context) を作成します。実行コンテキストには、現在実行中のコードとその実行を支援するためのすべての情報が含まれており、実行コンテキストがコールスタックに積まれ、スタックの最上位要素である Running execution context となることで実際に実行が行われます。具体的には、コードはパーサーによって解析されて、変数と関数がメモリに格納されて、実行可能なバイトコードが生成されてからコードが実行されます。

JavaScript のコードが実行されるとき、そのコードは実行コンテキスト (Execution context) 内部で実行されます。コードが作成する実行コンテキストには以下の３つの種類があります。

- (A) **Global Execution Context** (GEC):
  「**グローバルコンテキスト (Global context)**」とも呼ばれる。これはユーザーコードの本体 (main body) を実行する際に作成される実行コンテキストです。つまり、JavaScript の関数の外側に存在するあらゆるコードは、それが実行される際にグローバルコンテキストが作成されます。
- (B) **Functional Execution Context** (FEC):
  「関数実行コンテキスト」または「関数コンテキスト」と呼ばれるコンテキストです。各関数は、それ自身の実行コンテキスト内で実行されます。この実行コンテキストは「**ローカルコンテキスト (Local context)**」とも呼ばれます。
- (C) Eval Function Execution Context:
  現在は非推奨な関数である `eval()` 関数によって作成される実行コンテキスト。これについては基本的には考えなくて良いです。

基本的に実行コンテキストを考える際には (A) と (B) だけを考えればよいです。

:::message alert
「グローバルコンテキスト」は JS Visualizer で何故か可視化されてないコンテキストですが非常に重要です。その重要性から、可視化されないのは単に JS Visualizer の実装ミスだと個人的に考えています。Chrome ブラウザの開発者ツールなどでコールスタックの項目を見るとしっかりと匿名 (anonymous) のコンテキストとして積まれていることが確認できるので注意してください (後述)。
:::

## ブラウザ環境のコールスタック

例えば、ブラウザ環境において、次のような `index.html` と `main.js` ファイルがあったとします。この場合にどのようなことが起こるかを考えてみます (ブラウザ環境でやっているのは実際にコールスタックがどうなるかを確認するためです)。

```html:index.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <script src="main.js"></script>
  </body>
</html>
```

```js:main.js
function first() {
  count++;
  second();
}
function second() {
  count++;
}

let count = 0;
first();
console.log(count); // => 2
```

HTML のパースによって `<script>` タグが読み込まれます。`src` として指定されている `main.js` ファイルの JavaScript コードは JavaScript エンジン (V8) に送信されて、グローバルコンテキスト (Global Execution Context) がデフォルトの実行コンテキストとして作成されます。JS エンジンは実行を開始して、コールスタックの一番下に配置します。

コールスタックはスタックですから、LIFO(Last In, First Out) です。

`main.js` ファイルには３つの実行コンテキストがあり、その内のいくつかはプログラムの実行中に何度も作成され、破棄されます。１つの実行コンテキストの実行が終わるとその実行コンテキストは削除されます。このファイルが実行される際にどのように実行コンテキストが作成されていくかを見てみます。

コールスタックには次の図のような過程で実行コンテキストが積まれていきます。

![execution context stack](/images/js-async/img_executionContextStack_1.jpg)

詳しく見てみると以下のようになります。

- (0 → 1) プログラムが開始されると、まずグローバルコンテキスト (GEC: Global Execution Context) が作成されて、コールスタック (Execution context stack) へとプッシュされます。プッシュされた実行コンテキスト (Execution context) はコールスタック上でトップになるので、グローバルコンテキストは Running Execution Context へとなります。
- (1 → 2) コードは上から読まれ、処理されていきます。関数呼び出し `first()` に到達すると、`first()` 関数用の関数実行コンテキスト (FEC: Functional Execution Context) が作成され、この実行コンテキストはコールスタックへとプッシュされます。プッシュされた実行コンテキストはコールスタック上でトップなので Running Execution Context になります。
- (2 → 3) `first()` 内の処理が行われていき、内部で `second()` が呼び出されます。これによって `second()` 用の関数実行コンテキストが作成され、この実行コンテキストはコールスタックへとプッシュされます。プッシュされた実行コンテキストはコールスタック上でトップになるので Running Execution Context となります。
- (3 → 4) `second()` 内の処理が行われていき、すべての処理が終わると、その実行コンテキストである関数実行コンテキストがコールスタック上からポップして破棄されます。
- (4 → 5) コールスタック上でトップの実行コンテキストは `first()` のコンテキストで、Running Execution Context となり、`second()` の呼び出し元であった `first()` に制御が戻り、再び `first()` 内の処理が再開されます。関数内の処理がすべて終わると、`first()` の実行コンテキストはコールスタック上からポップして破棄されます。
- (5 → 6) コールスタック上のトップはグローバルコンテキストであるため、Running Execution Context はグローバルコンテキストとなります。関数 `first()` の呼び出し元であったグローバルコンテキスト内の処理が再開されます。
- (6 → 7 → 8) `console.log()` のコンテキストが積まれ、コンソールにインクリメントの結果である `2` が出力されます。
- (8 → 9 → 10) グローバルスコープの処理がすべて終わったのでグローバルコンテキストがポップし破棄されます。これによりコールスタックは完全に空の状態になります。

Chrome などのブラウザ環境ではソースにブレークポイントを設けて、コールスタックの状態を確認できます。以下のようにグローバルコンテキスト (この場合は「匿名」) の上に `first()` が積まれ、さらにその上に `second()` の実行コンテキストが積まれていく状態が確認できます。

![Callstack in browser](/images/js-async/img_callstackInBrowser2.png)

非同期処理では、このようにコールスタックに積まれていく実行コンテキスト (グローバルコンテキストと関数実行コンテキスト) を考えていくことが必要になります。というのも、コールバック関数の実行のタイミングを考える時に非常に重要になるからです。

## 非同期コールバック関数の実行コンテキスト

非同期処理としてタスクキューなどに送られていく、コールバック関数も関数なので実行される際には関数実実行コンテキストが作成されてコールスタックへと積まれることで実行されます。その際に重要なことは、**非同期のコールバック** が作成する関数実行コンテキストは **グローバルコンテキストがすでにポップされた状態の後でコールスタックに積まれる** ということです。

:::message
コールバック関数自体の処理が同期的に行われるか非同期的に行われるかは、そのコールバック関数を引数として受け取る側の問題ですので、注意してください。

この話については、『[コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async)』のチャプターで詳細に解説しているのでそちらを参考にしてください。
:::

### タスクとなるコールバックの場合

:::message alert
スクラップの [コメント](https://zenn.dev/link/comments/6f4fa06b9f805d) にて指摘していただいたコード最後の `console.log()` の実行コンテキストが抜け落ちてしまっている箇所を修正いたしました。
:::

`setTimeout(callback, delay)` の場合のコールバック関数 `callback` は非同期的に処理されてタスクキュー(Task queue) へと追加されます。タスクキューへ追加されるタイミングは環境 (Environment) がタイマーによって別スレッドで計測しています。指定した遅延時間 `delay` が経過したことが分かると、環境はそのコールバックをタスクキューへと送信します。`setTimeout()` や `setInterval()` で指定した時間を環境が並列的にバックグラウンドで計測してくれているおかげで、メインスレッドで別の作業ができます。

例えば、次のように `setTimeout()` でタスクを発行するシンプルなスクリプトで考えてみます。ブラウザ環境とランタイム環境で考え方は変わりません。

```js:simpleTask.js
// simpleTask.js
console.log("[1] 🦖 MAINLINE: Start [GEC]");

setTimeout(function taskFunc() {
  console.log("[3] ⏰ TIMERS: timeout 5000ms [EFC]");
}, 5000); // 5000 ミリ秒後にタスクキューへタスクを発行

console.log("[2] 🦖 MAINLINE: End [GEC]");
```

:::message
説明しやすいようにコールバック関数にあえて名前をつけていまが、通常はアロー関数などで無名関数 (匿名関数) になっているのが普通ですので注意してください。
:::

この時の出力は次のようになります。

```sh
[1] 🦖 MAINLINE: Start [GEC]
[2] 🦖 MAINLINE: End [GEC]
[3] ⏰ TIMERS: timeout 5000ms [FEC (taskFunc)]
```

この時に何が起こるかを考えてみます。コールスタックには次の図のような過程で実行コンテキストが積まれていきます。

![Execution Context Stack2](/images/js-async/img_executionContextStack_2_task.jpg)

- (0 → 1) プログラムが開始されると、まずグローバルコンテキスト (GEC: Global Execution Context) が作成されて、コールスタック (Execution context stack) へとプッシュされます。プッシュされた実行コンテキスト (Execution context) はコールスタック上でトップになるので、グローバルコンテキストは Running Execution Context へとなります。
- (1 → 2) コードは上から読まれ、処理されていきます。`console.log()` 用の関数実行コンテキストが作成され、この実行コンテキストはコールスタックへとプッシュされます。プッシュされた実行コンテキストはコールスタック上でトップなので Running Execution Context になります。
- (2 → 3 → 4) コンソールに出力がなされて、すぐにこの実行コンテキストはコールスタックからポップして破棄されます。再びグローバルコンテキストがトップになり、Running Execution Context となります。
- (4 → 5) そして、次の処理である `setTimeout()` が実行され、その実行コンテキストである関数実行コンテキストがコールスタック上からプッシュされて Running Execution Context となります。これは非同期 API であるため環境にタイマーで指定時間 5000 ミリ秒を測るように指示します。
- (5 → 6) コールスタック上でトップの実行コンテキストはすぐにポップし破棄されます。呼び出し元であったグローバルコンテキストが再びトップで Running Execution Context となります。
- (6 → 7 → 8) コード配置上最後にある `console.log()` 用の関数実行コンテキストが最初のステップと同じ要領で作成され、この実行コンテキストはコールスタックへとプッシュさて、Running Execution Context となり、コンソールに出力が実行されます。
- (8 → 9 → 10) コールスタック上のトップはグローバルコンテキストであるため、Running Execution Context はグローバルコンテキストとなりますが、他の何も処理すべきものが残っていないのでグローバルコンテキストはすぐにポップします。
- (10 → 11) しばらく、コールスタックは空の状態となりますが (ブラウザ環境ならレンダリングの作業が一定時間の間隔でなされています)、`setTimeout()` で指定していた遅延時間 5000 ミリ秒が経過した時点で登録しておいたコールバック関数が Web API からタスクキューにタスクとして発行されます。
- (12 → 13) タスクキューにあるタスクはコールスタック上へと関数実行コンテキストを積むことで実行されます。今回グローバルコンテキストは存在していませんので、一番下のコンテキストが `taskFunc()` による関数実行コンテキストとなります。
- (13 → 14) コールバック関数 `taskFunc()` の中身がすべて実行されます。`console.log()` の実行コンテキストが上に積まれてコンソールに出力されますが、すぐにポップして破棄されます。
- (14 → 15) コールバック関数 `taskFunc()` 内の処理がすべて終わったので実行コンテキストはポップして破棄されます。再びコールスタックは空の状態となります。

このように非同期のコールバックは同期処理がすべて処理され、コールスタックが空になった後で、タスクとしてコールスタックに送られて実行されるようになっています。最初にコールスタックが空になるのはグローバルコンテキストがポップした後です。

## マイクロタスクになるコールバックの場合

:::message alert
スクラップの [コメント](https://zenn.dev/link/comments/6f4fa06b9f805d) にて指摘していただいたコード最後の `console.log()` の実行コンテキストが抜け落ちてしまっている箇所を修正いたしました。
:::

マイクロタスクのチェックポイント、つまり「いつマイクロタスクを実行するか」が非同期処理の制御予測には重要です。マイクロタスクのチェックポイントは「**コールスタックが空になった時点**」です。コールスタックが空になったら必ずマイクロタスクを処理し、マイクロタスクキュー内にあるすべてのマイクロタスクが完全になくなるまですべて処理します。

タスクとマイクロタスクが処理されていくモデルである、イベントループを複雑なものに感じてしまう場合には、「**コールスタックが空になったらすべてのマイクロタスクが処理される**」と覚えると良いです。このルールだけを覚えておけば基本的にすべてに対応できます。

それでは今度はタスクではなく、マイクロタスクを発行するシンプルなスクリプトで何が起きるか考えてみます。

```js:simpleMicroTask.js
// simpleMicroTask.js
console.log("[1] 🦖 MAINLINE: Start [GEC]");

Promise.resolve()
  .then(function microTaskFunc() {
    console.log("[3] 👦 MICRO: [FEC (microTaskFunc)]");
  }); // 直ちにマイクロタスクキューへマイクロタスクを発行する

console.log("[2] 🦖 MAINLINE: End [GEC]");
```

この時の出力は次のようになります。

```sh
[1] 🦖 MAINLINE: Start [GEC]
[2] 🦖 MAINLINE: End [GEC]
[3] 👦 MICRO: [FEC (microTaskFunc)]
```

流れは完全にタスクの場合と同じです。

タスクの場合は 5000 ミリ秒が経過した後でタスクが発行されていましたが、今回 `Promise.resolve().then()` の場合は直ちにコールバック関数をマイクロタスクキューへマイクロタスクとして発行します。従って、グローバルコンテキストがコールスタックからポップされた直後にコールスタックへと積まれて登録していたコールバック関数が実行されます。

![3](/images/js-async/img_executionContextStack_3_microtask.jpg)

:::message alert
冗長になるので図では、 `Promise.resolve()` の呼び出しと `then()` メソッドの呼び出しを一つの実行コンテキストとして表現しています。
:::

## マイクロタスクとタスクの実行

それでは、タスクとマイクロタスクがコールスタック上でどのように積まれていくかを見ていきます。次のようにタスクとマイクロタスクをそれぞれ発行するようなコードを考えます。

```js:simpleTaskMicrotask.js
// simpleTaskMicrotask.js
// <- Task1
setTimeout(function taskFunc2() {
  console.log("[3] ⏰ TIMERS: [FEC]/[Task2]");
  Promise.resolve().then(function microTaskFunc2() {
    console.log("[4] 👦 THEN: [FEC]/[Microtask2]");
  }).then(function microTaskFunc3() {
    console.log("[5] 👦 THEN: [FEC]/[Microtask3]");
  });
});

setTimeout(function taskFunc3() {
  console.log("[6] ⏰ TIMERS: [FEC]/[Task3]");
});

Promise.resolve().then(function mircoTask1() {
  console.log("[2] 👦 THEN: [FEC]/[Microtask1]");
});

console.log("[1] 🦖 MAINLINE: End [GEC]/[Task1]");
// Task1 ->
```

この時の出力は次のようになります。

```sh
[1] 🦖 MAINLINE: End [GEC]/[Task1]
[2] ⏰ TIMERS: [FEC]/[Task2]
[3] 👦 THEN: [FEC]/[Microtask1]
[4] 👦 THEN: [FEC]/[Microtask2]
[5] ⏰ TIMERS: [FEC]/[Task3]
```

コールスタックに積まれる実行コンテキストの様子は以下のようになります。`console.log()` のコンテキストまで追加していくと非常に冗長になるので省略していることに注意してください。また、わかりやすくするため細かいコンテキストは気にしていません。

![4](/images/js-async/img_executionContextStack_4.jpg)

まず、プログラム開始時にいつもどおりグローバルコンテキストが作成されます。最初のタスクとして同期処理がすべて処理されていきます。`setTimeout()` の実行コンテキストが作成されてそれぞれ０秒遅延でタスクとしてコールバック関数 `taskFunc2()` と `taskFunc3()` を発行するように環境へと伝えます。そして、`Promise.resolve().then()` の関数実行コンテキストが作成されて、プロミスインスタンス自体は直ちに履行状態になったため、マイクロタスクが発行されます。

最後に `console.log()` の処理を行い、同期処理がすべて処理されて何もすることが無くなった時点でグローバルコンテキストがコールスタックからポップして破棄されます。この時点でコールスタックは空の状態となりました。コールスタックが空になったら **マイクロタスクのチェックポイント** です。マイクロタスクキューにマイクロタスクがあればすべてのマイクロタスクが処理されます。別の考え方だと、プログラムの実行開始という **単一タスクが完了したのですべてのマイクロタスクを処理** します。

:::message
マイクロタスクのチェックポイントは、コールスタックが空になったら、performing a microtask checkpoint boolean という真偽値 (マイクロタスクを実行中かどうかの真偽値) をチェックすることでその値が false であるならば true に変更して、マイクロタスクを実行します。マイクロタスクキューにあるすべてのマイクロタスクを処理したら、その真偽値を false へと戻します。

- [perform a microtask checkpoint | HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint)

理解の上では、とにかく「**コールスタックが空になったらマイクロタスクチェックポイントでマイクロタスクが処理される**」ものとして考えておけばよいと思います。
:::

というわけで、マイクロタスクキューに発行されていたマイクロタスクであるコールバック関数 `microTask1()` の関数実行コンテキストがコールスタックに積まれて実行されます。やることは、`console.log()` だけなのですぐに実行コンテキストはポップして破棄されます (`console.log()` のコンテキストは省略しています)。コールスタックが空になりました。マイクロタスクキューにあるすべてのマイクロタスクを処理しきったので、今度はタスクが実行されます。タスクキューの先頭にある `taskFunc2()` がコールスタックへと送られて実行コンテキストが積まれます。コールバック関数内の処理が順番に行われていきます。再び `Promise.resolve().then()` でマイクロタスクが同期的に発火されます。登録したコールバック関数 `microTaskFunc2()` は直ちにマイクロタスクキューへと送られます。とりあえずこの時点でコールバック関数内の同期的な処理はすべて終わったので `taskFunc2()` の実行コンテキストがポップして破棄されます。

再びコールスタックが空の状態になりました。マイクロタスクのチェックポイントです。別の言い方だと単一タスクが実行されたので、すべてのマイクロタスクを処理します。マイクロタスクキューの先頭にあるマイクロタスク `microTaskFunc2()` の実行コンテキストがコールスタックに配置されてコールバック関数内の同期処理が実行されます。処理が完了したことで、`Promise.resolve().then()` から返ってくる Promise インスタンスの状態が Pending から Fulfilled になったため、チェーンにおける次の `then()` のコールバック関数 `microTaskFun3()` が直ちにマイクロタスクキューへマイクロタスクとして送られます。そして、`microTaskFunc2()` の処理がすべて終わったのでコールスタックからポップして破棄されます。コールスタックは空で、マイクロタスクキューには先程送られたマイクロタスク `microTaskFun3()` が存在していますので、これもキューが完全に空になるまで処理されます。ということで、同じ様にコールスタックに配置されて実行され、ポップし破棄が行われます。

マイクロタスクキューにはマイクロタスクがもう残っていませんので、タスクキューの先頭にある単一タスクを今度は処理します。タスクキューの先頭には `setTimeout()` で発火しておいたタスク `taskFunc3()` がありますので、これをタスクキューからコールスタックへと実行コンテキストとして配置し実行します。コールバック関数内の処理がすべて終われば、その実行コンテキストをコールスタックからポップして破棄します。

これでコールスタックが再び空の状態になりました。タスクキューにはなにもありませんし、マイクロタスクキューにも何もありません。

従って、ランタイム環境でコンソールなどからファイルを実行していた場合にはイベントループから脱出してプログラム終了となります。ブラウザ環境の場合はタブなどを閉じない限り、レンダリング等の作業があるのでイベントループは終わらずに無限につづいていきます。

これが、非同期処理の一連の流れとなります。

気をつけてほしいのは、今回の場合はタスクキューが `setTimeout()` が発火するタスクを集めるためのキューであり、それ１つしか使っていません。実際には複数のタスクキューが存在しており、単一タスクを実行する際にどのタスクキューを選ぶかというのが明瞭ではない場合があります。

:::message
複数のタスクキューに複数のタスクが存在している場合は、環境側の定めるルールでどのタスクキューを選ぶかを決定します。Node 環境の場合はフェーズがあるためある程度予測できますが、ブラウザ環境の場合はユーザーインタラクションやクリックイベントなどのタスクソースから発行されるタスクを保持しておくタスクキューをレンダリングエンジンが `setTimeout()` などのタスクよりも優先的に選択します。

これについては、『[それぞれのイベントループ](c-epasync-what-event-loop)』のチャプターで解説しているのでそちらを参照してください。
:::

## 注意

マイクロタスクキューに複数のマイクロタスクがあり、それらを１つずつ実行する際にコールスタック上の実行コンテキストが一々空になっているかは微妙で、`Run microtasks` というコンテキスト (あるいはスタックフレーム) が配置されている可能性があります。

というのも V8 エンジンのサイトでの記事でそのように積まれているからです。実際に `console.trace()` を使って見ようとしても確認できません。これについては、Chrome でも Deno でも確認できません。可能性としては、視認できないように省略されているか、解説記事でのわかりやすさのために図示しているなどが考えられます。

https://v8.dev/blog/fast-async#await-under-the-hood

https://slidr.io/bmeurer/zero-cost-async-stack-traces-1#27

V8 で `console.trace()` によってスタックトレースすると、`EntryFrame` や `InternalFrame`、`StubFrame` というスタックフレームが積まれることを確認できたりします。

```js:asyncTrace.js
async function foo() {
  console.trace("👾 Stack Trace [async function context]");
  await bar();
  return 42;
}

async function bar(x) {
  await Promise.resolve();
  throw new Error("Let's have a look...");
}

Promise.resolve()
  .then(() => console.log("🦄 then"))
  .then(function thenCall() {
    console.trace("🦄 Trace");
  });

foo(1).catch((e) => console.log(e.stack));
```

実行してみると次のようになる。

```sh
❯ v8 asyncTrace.js

==== JS stack trace =========================================

Security context: 0x2a55001d1009 <JSGlobalObject>#0#
    0: builtin exit frame: trace(this=0x2a55001c6161 <console map = 0x2a5500202941>#1#,0x2a55001c6161 <console map = 0x2a5500202941>#1#,0x2a55001d3b51 <String[39]: u#\xd83d\xdc7e Stack Trace [async function context]>)

    1: foo [0x2a55001d3ab9] [asyncTrace.js:5] [bytecode=0x2a55001d3c21 offset=30](this=0x2a55001c37dd <JSGlobalProxy>#2#)
    2: /* anonymous */ [0x2a55001d3a35] [asyncTrace.js:21] [bytecode=0x2a55001d39a1 offset=63](this=0x2a55001c37dd <JSGlobalProxy>#2#)
    3: InternalFrame [pc: 0x10fe8ab4c]
    4: EntryFrame [pc: 0x10fe8a7e8]
=====================

🦄 then

==== JS stack trace =========================================

Security context: 0x2a55001d1009 <JSGlobalObject>#0#
    0: builtin exit frame: trace(this=0x2a55001c6161 <console map = 0x2a5500202941>#1#,0x2a55001c6161 <console map = 0x2a5500202941>#1#,0x2a55001d3eb5 <String[8]: u#\xd83e\xdd84 Trace>)

    1: thenCall [0x2a550004a3c5] [asyncTrace.js:18] [bytecode=0x2a55001d3f09 offset=12](this=0x2a55001c37dd <JSGlobalProxy>#2#)
    2: StubFrame [pc: 0x10ff5a998]
    3: StubFrame [pc: 0x10feb2744]
    4: EntryFrame [pc: 0x10fe8aa28]
=====================

Error: Let's have a look...
    at bar (asyncTrace.js:12:9)
    at async foo (asyncTrace.js:6:3)
```

`console.trace()` でコンテキストがどのように積まれるかを見る際には、環境によって表示されるものが違うということがよくあります (特にマイクロタスクやタスクなどの実行時に)。そういう訳で、ここでの話はある程度の抽象的なものとして扱ってください。
