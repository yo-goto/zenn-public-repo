---
title: "Event loop は内部にネストしたループがある[作成中]"
---

`setTimeout` とマクロタスクが分かったところで、Event loop には実はネストがあることを確認したいと思います。

Event loop の詳細については[Event loop の概要と注意点](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/2-epasync-event-loop) のチャプターに記載しましたが、ブラウザ環境における Event loop の疑似コードは以下のようになっていました。

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

正直、レンダリングまでは考慮したくないのでとりあえずレンダリングの部分を省略します(Deno でもレンダリングのステップはないので)。

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
  // レンダリングの部分を省略
}
```

見て分かるように `while` ループ内部にもう１つ `while` ループが存在していますね。

Event loop 自体は最初の大きなループですが、内部のネストされているループは Microtask queue を完全に空にするまで実行するためのループです。

Event loop の1回のループにおいて Task(Macrotask) と Microtask の処理には次のような違いがあります。

- Task(Macrotask) は一個のみが処理
- Microtask はすべて処理される(キューが完全に空になるまで)

これを理解していないと次のコードの実行順番がわかりません。

```js
// realitySettimeout.js
// Event loop においてループがネストしていることを知らないと実行順序が分からない
console.log("[1] Sync process");

setTimeout(() => {
  console.log("[3] setTimeout[0ms] finished");
  Promise.resolve("1st Promise")
    .then(value => {
      console.log("[4] Resolved value:", value);
    })
    .then(() => {
      console.log("[5] Next chain");
    })
}, 0);
setTimeout(() => {
  console.log("[6] setTimeout[1000ms] finished");
  Promise.resolve("2nd Promise")
    .then((value) => {
      console.log("[7] Resolved value:", value);
    })
    .then(() => {
      console.log("[8] Next chain");
    });
}, 0);

console.log("[2] Sync process");
```

実際に実行すると次の出力を得ます。

```sh
❯ deno run realitySettimeout.js
[1] Sync process
[2] Sync process
[3] setTimeout[0ms] finished
[4] Resolved value: 1st Promise
[5] Next chain
[6] setTimeout[1000ms] finished
[7] Resolved value: 2nd Promise
[8] Next chain
```

実際にはこのような書き方はめったに見ないと思います。というのもこのコードのやりたいこととしては、「特定の時間が経過したらあるタスクを実行して、そのタスクが完了したら所定のタスクを実行する」というものなので、それなら前のチャプターで見たように Promise で `setTimeout()` をラップして Promise チェーンにすればよいので(上のコードでは `setTimeout()` の第二引数に遅延時間を指定していないため、`0` が指定された場合と同じように扱われます)。

通常は次のように Promsie でラップして書くと思います。

```js
// realitySettimeout-normal.js
const wrappingPromise = (resolveValue, order, delayTime) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log(`${order} setTimeout[${delayTime}ms] finished`);
      resolve(resolveValue);
    }, delayTime);
  });
};

console.log("[1] Sync process");

wrappingPromise("1st Promise", "[3]", 1000)
  .then((value) => {
    console.log("[4] Resolved value:", value);
  })
  .then(() => {
    console.log("[5] Next chain");
  });
wrappingPromise("2nd Promise", "[6]", 1000)
  .then((value) => {
    console.log("[7] Resolved value:", value);
  })
  .then(() => {
    console.log("[8] Next chain");
  });

console.log("[2] Sync process");
```

これを実行すると以下の出力を得ます。

```sh
❯ deno run realitySettimout-normal.js
[1] Sync process
[2] Sync process
[3] setTimeout[1000ms] finished
[4] Resolved value: 1st Promise
[5] Next chain
[6] setTimeout[1000ms] finished
[7] Resolved value: 2nd Promise
[8] Next chain
```

今度のコードはもっと難しいです。

```js
// realitySettimeout-nested.js
// Event loop においてループがネストしていることを知らないと実行順序が分からない
// ネストさせたので確実にわからないと解けない
console.log("[1] Sync process");

setTimeout(() => {
  console.log("[3] setTimeout[0ms] finished");
  Promise.resolve("1st Promise")
    .then((value) => {
      console.log("[4] Resolved value:", value);
    })
    .then(() => {
      console.log("[5] Next chain");
    });

  setTimeout(() => {
    console.log("[9] setTimeout[0ms] finished");
    Promise.resolve("2nd Promise")
      .then((value) => {
        console.log("[10] Resolved value:", value);
      })
      .then(() => {
        console.log("[11] Next chain");
      });
  });
});
setTimeout(() => {
  console.log("[6] setTimeout[1000ms] finished");
  Promise.resolve("3rd Promise")
    .then((value) => {
      console.log("[7] Resolved value:", value);
    })
    .then(() => {
      console.log("[8] Next chain");
    });
});

console.log("[2] Sync process");
```

実際に実行すると次の出力を得ます。

```sh
❯ deno run realitySettimeout-nested.js
[1] Sync process
[2] Sync process
[3] setTimeout[0ms] finished
[4] Resolved value: 1st Promise
[5] Next chain
[6] setTimeout[1000ms] finished
[7] Resolved value: 3rd Promise
[8] Next chain
[9] setTimeout[0ms] finished
[10] Resolved value: 2nd Promise
[11] Next chain
```

それでは、トップレベルに Promise チェーンを配置してみました。どうなるでしょうか?

```js
// realitySettimeout-nestedPlus.js
console.log("[A-1] Sync process");

setTimeout(() => {
  console.log("[B-4] setTimeout[0ms] finished");
  Promise.resolve("1st Promise")
    .then((value) => {
      console.log("[C-5] Resolved value:", value);
    })
    .then(() => {
      console.log("[D-6] Next chain");
    });

  setTimeout(() => {
    console.log("[E-10] setTimeout[0ms] finished");
    Promise.resolve("2nd Promise")
      .then((value) => {
        console.log("[F-11] Resolved value:", value);
      })
      .then(() => {
        console.log("[G-12] Next chain");
      });
  });
});
setTimeout(() => {
  console.log("[H-7] setTimeout[1000ms] finished");
  Promise.resolve("3rd Promise")
    .then((value) => {
      console.log("[I-8] Resolved value:", value);
    })
    .then(() => {
      console.log("[J-9] Next chain");
    });
});
// トップレベルに Promsei チェーンを追加
Promise.resolve("4th Promise")
  .then((value) => console.log("[K-3]", value));

console.log("[L-2] Sync process");
```

出力は次のようになります。

```sh
❯ deno run realitySettimeout-nestedPlus.js
[A-1] Sync process
[L-2] Sync process
[K-3] 4th Promise
[B-4] setTimeout[0ms] finished
[C-5] Resolved value: 1st Promise
[D-6] Next chain
[H-7] setTimeout[1000ms] finished
[I-8] Resolved value: 3rd Promise
[J-9] Next chain
[E-10] setTimeout[0ms] finished
[F-11] Resolved value: 2nd Promise
[G-12] Next chain
```

Event loop のステップにおいて、最初は「スクリプトの評価」が Task として扱われるので、コードを上から下に順番に評価し同期処理をすべて実行します。そして Event loop は次のステップへと移行して、Microtask queue が完全に空になるまで実行します。






次は更に難しいです。

```js
// realitySettimeout-nestedPlus.js
console.log("[A-1] Sync process");

setTimeout(() => {
  console.log("[B-4] setTimeout[0ms] finished");
  Promise.resolve("1st Promise")
    .then((value) => {
      console.log("[C-6] Resolved value:", value);
    })
    .then(() => {
      console.log("[D-7] Next chain");
    });
  console.log("[E-5]");

  setTimeout(() => {
    console.log("[F-12] setTimeout[0ms] finished");
    Promise.resolve("2nd Promise")
      .then((value) => {
        console.log("[G-13] Resolved value:", value);
      })
      .then(() => {
        console.log("[H-14] Next chain");
      });
  });
});
setTimeout(() => {
  console.log("[I-8] setTimeout[1000ms] finished");
  Promise.resolve("3rd Promise")
    .then((value) => {
      console.log("[J-10] Resolved value:", value);
    })
    .then(() => {
      console.log("[K-11] Next chain");
    });
  console.log("[L-9]");
});
Promise.resolve()
  .then(() => console.log("[M-3]"));


console.log("[N-2] Sync process");
```

これを出力すると次のようになります。

```sh
❯ deno run realitySettimeout-nestedPlus.js
[A-1] Sync process
[N-2] Sync process
[M-3]
[B-4] setTimeout[0ms] finished
[E-5]
[C-6] Resolved value: 1st Promise
[D-7] Next chain
[I-8] setTimeout[1000ms] finished
[L-9]
[J-10] Resolved value: 3rd Promise
[K-11] Next chain
[F-12] setTimeout[0ms] finished
[G-13] Resolved value: 2nd Promise
[H-14] Next chain
```