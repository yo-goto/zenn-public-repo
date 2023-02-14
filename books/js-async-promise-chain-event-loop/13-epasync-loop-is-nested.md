---
title: "イベントループは内部にネストしたループがある"
cssclass: zenn
date: 2022-04-17
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『イベントループは内部にネストしたループがある』
---

# このチャプターについて

このチャプターでは、イベントループには実はネストされたループがあることを再確認しておきます。

# いつものルール

『[それぞれのイベントループ](c-epasync-what-event-loop)』のチャプターですでに色々なイベントループを見ましたので、**もう環境に囚われることはありません**。V8 のデフォルトイベントループで十分です。

大切なことは、「**単一タスクが処理されたら、すべてのマイクロタスクを処理する**」です。V8 エンジンのイベントループの疑似コードを確認します。

```js:V8エンジンのデフォルトイベントループ
while (tasksAreWaiting()) {
  queue = getNextQueue();
  task = queue.pop();
  execute(task);

  while (micortaskQueue.hasTasks()) {
    doMicrotask();
  }
}
```

# イベントループ内のネスト

上の擬似コードを見て分かるように `while` ループ内部にもう１つ `while` ループが存在していますね。

イベントループ全体の１サイクルは単一タスクを処理するための大きなループですが、内部のネストされているループはマイクロタスクを完全に空にするまで実行するためのループです。これを理解していないと次のコードの実行順番がわかりません。

上の疑似コードを参考に実行予測してみてください。

```js
// realitySettimeout.js
// イベントループにおいてループがネストしていることを知らないと実行順序が分からない
console.log("[A] 🦖 MAINLINE: Start");

setTimeout(() => {
  console.log("[B] ⏰ TIMERS: setTimeout callback");
  Promise.resolve("1st Promise")
    .then(value => {
      console.log("[C] 👦 MICRO: Resolved value:", value);
    })
    .then(() => {
      console.log("[D] 👦 MICRO: Next chain");
    })
});
setTimeout(() => {
  console.log("[E] ⏰ TIMERS: setTimeout callback");
  Promise.resolve("2nd Promise")
    .then((value) => {
      console.log("[F] 👦 MICRO: Resolved value:", value);
    })
    .then(() => {
      console.log("[G] 👦 MICRO: Next chain");
    });
});

console.log("[H] 🦖 MAINLINE: End");
```

イベントループの疑似コードが理解できていると出力結果が予測できます。

:::details 答え
答えは、「A → H → B → C → D → E → F → G」となります。

```sh:数字付きで出力
# V8, Node, Deno ですべて同じ結果
❯ deno run realitySettimeout.js
[A-1] 🦖 MAINLINE: Start
[H-2] 🦖 MAINLINE: End
[B-3] ⏰ TIMERS: setTimeout callback
[C-4] 👦 MICRO: Resolved value: 1st Promise
[D-5] 👦 MICRO: Next chain
[E-6] ⏰ TIMERS: setTimeout callback
[F-7] 👦 MICRO: Resolved value: 2nd Promise
[G-8] 👦 MICRO: Next chain
```
:::

どうなるか考えてみましょう。

スクリプト評価による最初のタスクが実行されます。コールスタックにグローバルコンテキストが作成されてすべての同期処理が処理されます。`console.log()` → `setTimeout()` → `setTimeout()` → `console.log()` というように API 呼び出しが行われていきます。

これらの同期処理が終わり、グローバルコンテキストがポップして破棄されると、マイクロタスクキューにあるすべてのマイクロタスクを処理します。ですが、同期処理が終わった時点でマイクロタスクキューにマイクロタスクが無いので、タスクを処理します。遅延時間０でタスクを発行するように `setTimeout()` を介して環境に伝えていたので、ほぼノータイムでタスクキューにタスクが２つ順番にキューされています。というわけでタイマー用タスクキューの先頭にあるタスクを１つ処理します。登録していたコールバック関数内の処理が開始されます。`console.log()` が実行されたら、すぐ `Promise.resolve().then()` に出会うので、直ちにマイクロタスクが発行されます。コールバック関数内の同期処理がすべて完了し、コールバック関数が作成していた関数実行コンテキストがコールスタックからポップして破棄されます。

コールスタックが空になったのでマイクロタスクチェックポイントとなります。マイクロタスクキューにあるすべてのマイクロタスクが実行されます。マイクロタスクがコールスタックに積まれて実行されます。これによって、`Promise.resolve().then()` で返ってくる Promise インスタンスは直ちに履行状態になるので、再びマイクロタスクキューへ直ちにマイクロタスクが発行されます。

そしてマイクロタスクがマイクロタスクキューにある限り処理されるので、これまたマイクロタスクがコールスタックに積まれて処理されます。すべてのマイクロタスクがなくなったので、再びタイマー用タスクキューの先頭にあるタスクを処理します。コールバックがコールスタックに置かれてコールバック関数内の同期処理がすべて行われます。また同じく、`Promise.resolve().then()` で直ちにマイクロタスクが発行されます。同期処理がすべて終わり、実行コンテキストがポップして破棄されコールスタックが空になります。再びマイクロタスクのチェックポイントでマイクロタスクキューにあるマイクロタスクをすべて処理します。これもすぐに `Promise.resolve().then()` で返ってくる Promise インスタンスが履行状態になるので、直ちにマイクロタスクが発行されます。そして生成されたマイクロタスクが処理しつくされます。この時点で、タスクキューとマイクロタスクキューにはなにもなくなり、待機状態のタスクどもがなくなったのでプログラムが終了します。

かなり説明が長くなりましたね😅

実際にはこのような書き方はめったに見ないと思います。というのもこのコードのやりたいこととしては、「特定の時間が経過したらあるタスクを実行して、そのタスクが完了したら所定のタスクを実行する」というものなので、それなら前のチャプターで見たように Promise で `setTimeout()` をラップして Promise chain にすればよいので(上のコードでは `setTimeout()` の第二引数に遅延時間を指定していないため、`0` が指定された場合と同じように扱われます)。

通常は次のように Promise でラップして書くと思います。

```js
// realitySettimeout-normal.js
const wrappingPromise = (resolveValue, order, delayTime) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log(`⏰ ${order} setTimeout[${delayTime}ms] finished`);
      resolve(resolveValue);
    }, delayTime);
  });
};

console.log("🦖 [1] MAINLINE: Sync process");

wrappingPromise("1st Promise", "[3]", 1000)
  .then((value) => {
    console.log("👦 [4] Resolved value:", value);
  })
  .then(() => {
    console.log("👦 [5] Next chain");
  });
wrappingPromise("2nd Promise", "[6]", 1000)
  .then((value) => {
    console.log("👦 [7] Resolved value:", value);
  })
  .then(() => {
    console.log("👦 [8] Next chain");
  });

console.log("🦖 [2] MAINLINE: Sync process");
```

これを実行すると以下の出力を得ます。

```sh
❯ deno run realitySettimout-normal.js
🦖 [1] MAINLINE: Sync process
🦖 [2] MAINLINE: Sync process
⏰ [3] setTimeout[1000ms] finished
👦 [4] Resolved value: 1st Promise
👦 [5] Next chain
⏰ [6] setTimeout[1000ms] finished
👦 [7] Resolved value: 2nd Promise
👦 [8] Next chain
```

イベントループにマイクロタスクを処理するためのループがあることを理解できたと思います。今度のコードはもっと難しいです。実行予測してみてください。

```js
// realitySettimeout-nested.js
console.log("[A] 🦖 MAINLINE: Start");

setTimeout(() => {
  console.log("[B] ⏰ TIMERS: setTimeout[0ms]");

  Promise.resolve("1st Promise")
    .then((value) => console.log("[C] 👦 MICRO: Resolved value:", value))
    .then(() => console.log("[D] 👦 MICRO: Next chain"));

  setTimeout(() => {
    console.log("[E] ⏰ TIMERS: setTimeout[0ms]");

    Promise.resolve("2nd Promise")
      .then((value) => console.log("[F] 👦 MICRO: Resolved value:", value))
      .then(() => console.log("[H] 👦 MICRO: Next chain"));
  });
});

setTimeout(() => {
  console.log("[I] ⏰ TIMERS: setTimeout[0ms]");

  Promise.resolve("3rd Promise")
    .then((value) => console.log("[J] 👦 MICRO: Resolved value:", value))
    .then(() => console.log("[K] 👦 MICRO: Next chain"));
});

Promise.resolve().then(() => console.log("[L] 👦 MICRO: then"));

console.log("[M] 🦖 MAINLINE: End");
```

ここまでくればほぼイベントループのモデルが頭に完成しつつあると思いますので、きっと分かるはずです。

:::details 答え
答えは、「A → M → L → B → C → D → I → J → K → E → F → H」となります。
V8, Node, Deno ですべて同じ結果となります。
```sh
❯ deno run realitySettimeout-nested.js
[A-1] 🦖 MAINLINE: Start
[M-2] 🦖 MAINLINE: End
[L-3] 👦 MICRO: then
[B-4] ⏰ TIMERS: setTimeout[0ms]
[C-5] 👦 MICRO: Resolved value: 1st Promise
[D-6] 👦 MICRO: Next chain
[I-7] ⏰ TIMERS: setTimeout[0ms]
[J-8] 👦 MICRO: Resolved value: 3rd Promise
[K-9] 👦 MICRO: Next chain
[E-10] ⏰ TIMERS: setTimeout[0ms]
[F-11] 👦 MICRO: Resolved value: 2nd Promise
[H-12] 👦 MICRO: Next chain
```
:::

説明は上でやったものとほぼ同じになるので省略します。

:::message
**ヒント**

`"[L] 👦 MICRO: then"` の部分は引っ掛けです。グローバルコンテキストがわかっていればマイクロタスクの実行タイミングが分かるはずです。

分からない場合には、『[コールスタックと実行コンテキスト](b-epasync-callstack-execution-context)』のチャプターを確認してください。
:::
