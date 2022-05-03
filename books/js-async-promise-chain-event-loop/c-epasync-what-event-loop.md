---
title: "それぞれの Event loop[作成中]"
---

Event loop は前のチャプターで説明したとおり、ECMAScript の範疇ではなく、HTML Living standard や実装するランタイム環境で独自に定められています。

従って、最終的には各環境ごとに Event loop がどのようになっているかということを認識する必要があります。とはいっても、Task と Microtask の考え方は同じです。

それぞれの、Event loop がどのようになっており、どのように違うのかということ、ブラウザ環境 → Node 環境 → Deno 環境という順番に考えてみます。

# ブラウザ環境の Event loop

## Mdn の Event loop

まずは、Mdn で Event loop がどのように語られているか見てみます。Event loop の疑似コードは次のようになっています。

```js
// mdn の Event loop
while (queue.waitForMessage()) {
  queue.processNextMessage()
}
```

>`queue.waitForMessage()` waits synchronously for a message to arrive (if one is not already available and waiting to be handled).

https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop#event_loop

待ち状態のタスクがある限りそのタスクを処理しつづけるというループになっていますが、これは簡略化しすぎているので、非同期処理については何も分かりません。ですが、Event loop というのは本質的には、このようにタスクを処理するための半無限ループであることを理解しておくとよいです。

## レンダリングエンジンとメッセージループ
Chrome ブラウザ環境の Event loop の実装についてはほとんど情報がありません。JavaScript エンジンである V8 は Event loop の実装をせずに外部からプラグインするような作りになっているそうです。

そのため Chrome ブラウザ環境では Livevent というライブラリが実際の Event loop の実装に使われているようです。

https://libevent.org

Event loop はブラウザ環境(Chrome) の文脈では Message loop と呼ばれることがあり、Message loop はレンダリングエンジンである Blink (Chrome の場合)が実装して、それを Event loop としている、という情報もあるのですが、Livevent との情報をあわせると、Task queues から単一の Task queue を選択して Task のスケジューリングをするのが Blink で、実際の Event loop は Libevent であるというように考えられます。

https://docs.google.com/document/d/11N2WTV3M0IkZ-kQlKWlBcwkOkKTCuLXGVNylK5E2zvc/edit

>The main purpose of the scheduler is to decide which task gets to execute on the main thread at any given time. To enable this, the scheduler provides higher level replacements for the APIs that are used to post tasks on the main thread.

https://docs.google.com/a/google.com/document/d/1SWpjgtwIaL_hIcbm6uGJKZ8o8R9xYre-yG0VDOjFBxU/edit

https://nhiroki.jp/2017/12/10/javascript-parallel-processing#1-%E3%83%AC%E3%83%B3%E3%83%80%E3%83%AA%E3%83%B3%E3%82%B0%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%B3%E3%81%A8-javascript-%E3%81%AE%E5%AE%9F%E8%A1%8C%E3%83%A2%E3%83%87%E3%83%AB

このように、ブラウザ環境での Event loop では、Node や Deno といったランタイム環境にはない、レンダリングエンジンの存在があるため、レンダリングの作業そのものを考慮する必要があります。逆にランタイム環境では、レンダリングのタスクが存在しないためシンプルになりますが、タスクの優先度をより細かくするなどの違いがあります。

細かいことは、とりあえず置いておいて、イベントループについて考えていきます。

## ブラウザ環境の Event loop を理解する
ここで、レンダリングを含めたブラウザ環境での Event loop を知るため、JSConfEU 2018 での Erin Zimmer 氏の講演動画である『Further Adventures of the Event Loop』を元に考えてみます。

@[youtube](u1kqx6AenYw)

https://2018.jsconf.eu/speakers/erin-zimmer-further-adventures-of-the-event-loop.html

ブラウザ環境において、HTML ファイルに次のようなスクリプトがあった場合、ブラウザはスクリプトをパースして、Task を作成し、同期処理の部分は最初の Task として実行されます。

```html
<script>
  const foo = bar;
  foo.doSomething();
  
  document.body.addEventLlistener('keydown', (event) => {
    if (event.key === 'PageDown') {
      location.href = "/#/36";
    }
  });
<script>
```

`addEventListern()` は Web API のメソッドであり、そのコールバック部分はブラウザがキーダウンイベントを受け取った際に、別の Task として実行されます。つまり、Task queue へと送られた後に、Call stack へと運ばれて JavaScript として実行されるわけです。

```js
// event loop v1
while (true) {
  task = taskQueue.pop();
  execute(task);
}
```


## 最終的な Event loop
最終的な Event loop は次のようになります。

```js
// event loop last in browser
while (true) {
  queue = getNextQueue();
  task = queue.pop();
  execute(task);
  
  while (micortaskQueue.hasTasks()) {
    doMicrotask();
  }

  if(isReapintTime()) {
    animationTasks = animationQueue.copyTasks();
    for(task in animationTasks) {
      doAnimationTask(task);
    }
    repaint();
  }
}
```

# Node 環境の Event loop

Node 環境では、直感に反してブラウザ環境よりもシンプルなものとなります。以下のように、ブラウザ環境では存在していたものが Node 環境には存在しません。

- スクリプトのパースイベントが存在しない
- やっかいなユーザーインタラクションが存在しない(マウスクリックなど)
- Animation frame callback が存在しない
- レンダリングパイプラインが存在しない

Node 環境の Event loop の実装は、Libuv (**Unicorn Velociraptor**) という非同期ランタイムのライブラリが担当しています。

実は、ブラウザ環境と考え方はほとんど同じです。もう1つの Microtask queue である `nextTickQueue` にのみ注意してください。

```js
// Node の Event loop
// 一周 1 tick 
while (tasksAreWaiting()) {
  queue = getNextQueue();
  // phase (タスクキュー) を１つ選択

  // phase (タスクキュー) でタスクが存在しているかどうか
  // 最大制限がある
  while (queue.hasTasks()) {
    task = queue.pop();
    execute(task);

    // １つの Task を処理したら、すべての Micotasks を処理する
    // NextTick queue の方が promise のキューよりも先に処理される
    while (nextTickQueue.hasTasks()) {
      doNextTickTask();
    }

    while (promiseQueue.hasTasks()) {
      doPromiseTask();
    }
  }
}
```

Chrome ブラウザ環境では、レンダリングエンジンである Blink が Task queue の優先度について考慮しているようですが、Node 環境では Event loop そのものの仕組みに Task queue の優先度を考慮する仕組みが組み込まれています。

マクロタスク、マイクロタスクという分類がまずはありますが、これはブラウザも同じです。

Node 環境ではマイクロタスクのためのキューが二種類用意されています。

- nextTickQueue
- microTaskQueue

マクロタスクについても優先度が割り振られています。対応するキューは Phase とよばれ Event loop のステップになっています。

１つのフェーズでは特定数の Task が実行されて、次のフェーズに行きます。すべてではなく特定数(最大制限)があるのは、１つのフェーズのマクロタスクが多すぎると次のフェーズにいつまでも移行できなくなるからです。では実行されずに残されたタスクはどうなるかというと一旦保留にして、次の Event loop において実行されます。ただし、マクロタスクだけは常に完全にキューが空になるまで実行されます。

https://blog.bitsrc.io/why-is-the-eventloop-for-browsers-and-node-js-designed-this-way-f7f794696c

# Deno 環境の Event loop

Deno 環境の Event loop の実装は Tokio という Rust 言語のための非同期ランタイムが担当しており、JavaScript の Promise などの仕組みは Rust における Future という別の非同期処理の仕組みによって実現されています。

Deno の event loop についてはほとんど情報を見かけないので調べるのが大変ですが、今まで見てきたとおり、１つの Task が完了したら、すべての Microtask を処理するというループであることは変わりません。

Node.js はサーバーサイドであるため、より高いパフォーマンス要件があり、タスクとマイクロタスクについてのよりきめ細かい優先順位付けを行う必要があるのでこのような仕組みになっています。


# 非同期処理を考える上での Event loop

ブラウザ環境でも、ランタイム環境でも Event loop の共通性質として言えることは、次の２つです。

- 開始時のスクリプトの評価は最初の Task になる
- Task を１つ実行したら、すべての Microtask をキューをが空になるまで実行する


