---
title: "コールバックで副作用となる非同期処理"
cssclass: zenn
date: 2022-04-17
modified: 2023-02-17
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『コールバックで副作用となる非同期処理』
---

## このチャプターについて

このチャプターは、別のチャプター『[then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)』の続きとしての内容となります。

`then()` メソッドのコールバックにおいて、単なる Promise インスタンスを返すだけでなく、非同期 API や Promise chain などによって最終的に Promise インスタンスが返る場合を考えます。

:::message alert
このチャプターの解説は『[Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)』のチャプターで解説した「内容の間違い」の影響を以前まで受けていましたが、現在は内容を修正・補足しました。
:::

## 副作用とは

このチャプターの本題へ入る前に「**副作用 (Side Effect)**」とは何かを簡単に説明しておきます。

:::message
副作用の概念は React の関数コンポーネントなどで出てくるため覚えておくと良いです。
:::

副作用の概念は関数型プログラミングの文脈などでよくでてくるものです。そして、関数型プログラミングの考え方の１つとして、「副作用の使用を避けて可能な限り Pure に考える」というものがあります。

Pure とは関数を純粋関数 (Pure function) にするということを意味しています。関数の基本は「入力を受け取って出力を返す」というものですが、出力に関与しないような操作を関数内で行わないというのが「副作用の使用を避ける」ことになります。これに加えて、入力以外のグローバル変数などを計算で使用せずに出力の値を計算して返すような関数を純粋関数と呼びます (正確な定義はべつのところへ任せます)。

```js
// Pure ではない
const name = "PADAone";
function greet() {
  // (1) 入力としての引数がなく、グローバルスコープから変数を読み込んでしまっている
  console.log("Hi, I'm " + name);
  // (2) 関数内で何も出力として返していない
}

// Pure な関数
// 出力の計算と関係無いものが何もないので純粋関数
// 入力のみを出力の計算に使っている
function greet(name) {
  return "Hi, I'm " + name;
}
```

というわけで、関数の最後に `return` で出力として返す値の計算に関係のない操作はすべて「副作用」となります。

```js
function noPureOp(input) {
  console.log(input); // 出力に関係ないので副作用
  const output = input + 1;
  console.log(output); // 出力に関係ないので副作用
  return output;
}
```

副作用については以下の JSConf EU 2017 での Anjana Vakil 氏の講演動画『Learning Functional Programming with JavaScript』が非常にわかりやすいので興味があれば視聴してみるとよいです。

https://www.youtube.com/watch?v=e-5obm1G_FY&list=TLGGz_fwguCfqL8yMDA3MjAyMg

## return の代わりに副作用を使用しない

さて、今まで `then()` メソッドのコールバック関数内にて返すものとしては次のパターンでした。

- (1) 文字列や数値などの通常の値を `return` する
  - 直ちに次の `then()` メソッドのコールバックがマイクロタスクキューへと追加されて、コールバック関数の引数には `return` した値が渡される
- (2) Promise インスタンスを `return` する
  - 待機状態ならそれが解決してから次の `then()` メソッドのコールバックがマイクロタスクキューへと追加され、`resolve` した値がコールバック関数の引数に渡される
  - 履行状態なら直ちに次の `then()` メソッドのコールバックがマイクロタスクキューへと追加され、`resolve` した値がコールバック関数の引数に渡される
- (3) 何も `return` しない
  - 直ちに次の `then()` メソッドのコールバックがマイクロタスクキューへと追加されて、コールバック関数の引数は `undefined` となる

(2) と (3) を混同してしまう場合に気をつけてください。`then()` メソッドのコールバック関数で Promise を使った非同期処理を行う場合には必ず Promise インスタンスを `return` するようにしてください。`then()` メソッドのコールバック関数内部で、非同期処理を使用する場合に、`return` をして Promise インスタンスを返していない場合、その非同期処理は「**副作用 (Side Effect)**」となります。この場合、次の `then()` メソッドのコールバック関数へ値を繋ぐことができなくなり、そもそも意図した実行順番にならなくなる場合があります。

次のコードの例では、Promise chain で値が繋がりません。

```js
//  promiseShouldBeReturned-non.js
console.log("🦖 [1] Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("👦 [5] Async");
    console.log("👦 Resolved value: ", value);
    // return しない場合は副作用となり値が渡らない
    returnPromise("2nd Promise", "[6]");
    // 🐝 このコールバックからは Promise が返されていないので追加のマイクロタスクが発生しない
  })
  .then((value) => {
    // この value は undefined となる
    console.log("👦 [9] Async");
    console.log("👦 Resolved value: ", value); // undefined が表示される
  });
returnPromise("3rd Promise", "[3]")
  .then((value) => {
    console.log("👦 [7] Async");
    console.log("👦 Resolved value: ", value);
    // Promise インスタンスについては必ず return するようにする
    return returnPromise("4th Promise", "[8]")
    // 🔥 このコールバックからは Promise が返されるので追加のマイクロタスクが２つ発生する
  })
  .then((value) => {
    console.log("👦 [10] Async");
    console.log("👦 Resolved value: ", value); // 値が繋がるので 4th Promise と表示される
  });

console.log("🦖 [4] Sync");
```

これを実行すると次の出力を得ます。`undefined` となっているところに注目してください。

```sh
❯ deno run promiseShouldBeReturned-non.js
🦖 [1] Sync
👻 [2] (a)sync
👻 [3] (a)sync
🦖 [4] Sync
👦 [5] Async
👦 Resolved value: 1st Promise
👻 [6] (a)sync
👦 [7] Async
👦 Resolved value: 3rd Promise
👻 [8] (a)sync
👦 [9] Async
👦 Resolved value: undefined
👦 [10] Async
👦 Resolved value: 4th Promise
```

従って、値を正しく繋げたい場合には、副作用ではなく `return` をつけるようにしましょう。

今度は、もう少し簡単にしてみます。これまで２つのメインとなる Promise chain で考えていましたが、ここでは１つにします。その代わりに、Promise chain 内部であえてネストを作ります。再びテストとして次のコードで `[A-G]` までの文字がどのような順番で出力されるか考えてみてください。

```js
// promiseShouldBeReturnedAddThen-right.js
console.log("🦖 [A] Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[B]")
  .then((value) => {
    console.log("👦 [C] Async");
    console.log("👦 Resolved value:", value);
    // return で正しいチェーンを作る
    return returnPromise("2nd Promise", "[D]")
      .then((value) => {
        console.log("👦 [E] Async");
        console.log("👦 Resolved value:", value);
        return "Pass next value";
      });
    // 🔥 このコールバックからは Promise が返されるので追加のマイクロタスクが２つ発生する
  })
  .then((value) => {
    console.log("👦 [F] Async");
    console.log("👦 Resolved value:", value);
  });

console.log("🦖 [G] Sync");
```

「then メソッドのコールバックで Promise インスタンスを返す」や「[Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)」のチャプターでネストは経験したので正解できましたか?

:::details 答え
答えは、「A → B → G → C → D → E → F」となります。

```sh:数字付きで出力
❯ deno run promiseShouldBeReturnedAddThen-right.js
🦖 [A-1] Sync
👻 [B-2] (a)sync
🦖 [G-3] Sync
👦 [C-4] Async
👦 Resolved value: 1st Promise
👻 [D-5] (a)sync
👦 [E-6] Async
👦 Resolved value: 2nd Promise
👦 [F-7] Async
👦 Resolved value: Pass next value
```
:::

`return returnPromise("2nd Promise", "[D]").then(callback)` の部分において Promise chain をネストさせていますが、ここで `return` しているのは最終的に `then(callback)` で返ってくる Promise インスタンスでした。

そして、`then()` メソッドのコールバック関数内にて返すものとして Promise インスタンスを選択した場合には、それが解決してから (実行が完了してから) 次の `then()` メソッドのコールバック関数が実行されるという話でした。

## return しないと非同期処理の完了を待てない

もう少し複雑化してみましょう。あとで代わりに Promise-based な非同期 API である `fetch()` 関数を使用した説明も行います。

次のコードでは、今までのコードでメインとなる Promise chain を１つにした上で、`returnPromise()` 関数内で Promise chain を行うように改造しました。つまり Promise chain をネストさせています。

Promise インスタンスを返す処理は常に `return` するべきですが、このコードではあえて `return` させていません。

まずは次のコードを考えてみましょう。
実際の実行順番とアルファベット `[A-H]` の出力順番はどうなるでしょうか？予測してみてください。

```js
// promiseShouldBeReturnedNest.js
console.log("🦖 [A] Sync");

const returnPromise = (resolvedValue, order, nextOrder) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
    // ↓ ここにチェーンを追加してみる
  }).then((value) => {
    console.log(`👹 ${nextOrder} Additional nested chain`);
    return value;
  });
};

returnPromise("1st Promise", "[B]", "[C]")
  .then((value) => {
    console.log("👦 [D] Async");
    console.log("👦 Resolved value: ", value);
    // ここで敢えて return しないとどういう実行順番になるか?
    returnPromise("2nd Promise", "[E]", "[F]");
    // 🔥 このコールバックからは Promise が返されないので追加のマイクロタスクが発生しない
  })
  .then((value) => {
    console.log("👦 [G] Async");
    console.log("👦 Resolved value: ", value);
  });

console.log("🦖 [H] Sync");
```

:::details 答え
答えは、「A → B → H → C → D → E → F → G」となります。

数字付きで実際に出力してみるとこうなります。
```sh
❯ deno run promiseShouldBeReturnedNest.js
🦖 [A] Sync
👻 [B] (a)sync
🦖 [H] Sync
👹 [C] Additional nested chain
👦 [D] Async
👦 Resolved value: 1st Promise
👻 [E] (a)sync
👹 [F] Additional nested chain
👦 [G] Async
👦 Resolved value: undefined
```
:::

最後の出力である `Resolved value` のところが `undefined` になっているので、値 `"2nd Promise"` が繋げていないことがわかります。
実行順番については、基本的に Promise インスタンスを返すような処理は `return` しないと順番を保証できないのですが、今回の場合は `returnPromise("2nd Promise", "[E]", "[F]");` が完了してから、次の `then()` メソッドのコールバックが実行されていますね。その理由としては、マイクロタスクを供給する Promise が少ないからたまたまそうなっているだけです。実際の動きを Visualizer で確認してみてください。

- [promiseShouldBeReturnedNest.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gcHJvbWlzZVNob3VsZEJlUmV0dXJuZWROZXN0LmpzCmNvbnNvbGUubG9nKCJbQS0xXSBTeW5jIHByb2Nlc3MiKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIsIG5leHRPcmRlcikgPT4gewogIHJldHVybiBuZXcgUHJvbWlzZSgocmVzb2x2ZSkgPT4gewogICAgY29uc29sZS5sb2coYCR7b3JkZXJ9IFRoaXMgbGluZSBpcyAoQSlTeW5jaHJvbm91c2x5IGV4ZWN1dGVkYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZyhgJHtuZXh0T3JkZXJ9IEFkZGl0aW9uYWwgbmVzdGVkIGNoYWluYCk7CiAgICByZXR1cm4gdmFsdWU7CiAgfSk7Cn07CgpyZXR1cm5Qcm9taXNlKCIxc3QgUHJvbWlzZSIsICJbQi0yXSIsICJbQy00XSIpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0QtNV0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICAgIHJldHVybiByZXR1cm5Qcm9taXNlKCIybmQgUHJvbWlzZSIsICJbRS02XSIsICJbRi03XSIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0ctOF0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCJbSC0zXSBTeW5jIHByb2Nlc3MiKTs%3D)
- ⚠️ 注意: JS Visualizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク・タスクの実行タイミングについて誤解しないように注意してください
- ⚠️ 注意: JS Visualizer では可視化されていないマイクロタスクが存在しています

Promise インスタンスを返すような処理を `return` しない場合に実行順番が保証できなくなってしまう例を挙げてみます。次のコードでは、`returnPromise()` 関数の内部に `then()` メソッドを更に追加して Promise chain を伸ばしています。実行順番を予想してみてください。

```js
// promiseShouldBeReturnedNest-3rd.js
console.log("🦖 [A] Sync");

const returnPromise = (resolvedValue, order, secondOrder, thirdOrder) => {
  return new Promise((resolve) => {
      console.log(`👻 ${order} (a)sync`);
      resolve(resolvedValue);
    })
    .then((value) => {
      console.log(`👹 ${secondOrder} Additional nested chain`);
      return value;
    })
    .then((value) => {
      console.log(`🦄 ${thirdOrder} Additional nested chain`);
      return value;
    });
};

returnPromise("1st Promise", "[B]", "[C]", "[D]")
  .then((value) => {
    console.log("👦 [E] Async");
    console.log("👦 Resolved value: ", value);
    returnPromise("2nd Promise", "[F]", "[G]", "[H]");
    // 🔥 このコールバックからは Promise が返されないので追加のマイクロタスクが発生しない
  })
  .then((value) => {
    console.log("👦 [I] Async");
    console.log("👦 Resolved value: ", value);
  });

console.log("🦖 [N] Sync");
```

:::details 答え
答えは、「A → B → N → C → D → E → F → G → I → H」となります。

数字付きで実際に出力してみるとこうなります。
```sh
❯ deno run promiseShouldBeReturnedNest-3rd.js
🦖 [A-1] Sync
👻 [B-2] (a)sync
🦖 [N-3] Sync
👹 [C-4] Additional nested chain
🦄 [D-5] Additional nested chain
👦 [E-6] Async
👦 Resolved value: 1st Promise
👻 [F-7] (a)sync
👹 [G-8] Additional nested chain
👦 [I-9] Async
👦 Resolved value: undefined
🦄 [H-10] Additional nested chain
```
:::

注目してほしいのは、`[H]` と `[I]` の順番です。H が終わっていないのに、I が実行されていますね。

上のコードではマイクロタスクキューへと連続でマイクロタスクを送っていますが、その送る順番は `return` をしなかったことで、`returnPromise("2nd Promise", "[F]", "[G]", "[H]");` 内部の Promise chain が 2 番目の `then()` メソッドのコールバックが終わって Call stack が空になった瞬間 `returnPromise("1st Promise", "[B]", "[C]", "[D]").then(cb1).then(cb2)` のコールバック `cb2` がキューへと送られてしまうためです。

言葉で説明するのが難しいので、実際に見てみてください。

- [promiseShouldBeReturnedNest-3rd.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gcHJvbWlzZVNob3VsZEJlUmV0dXJuZWROZXN0LTNyZC5qcwpjb25zb2xlLmxvZygiW0EtMV0gU3luYyBwcm9jZXNzIik7Cgpjb25zdCByZXR1cm5Qcm9taXNlID0gKHJlc29sdmVkVmFsdWUsIG9yZGVyLCBzZWNvbmRPcmRlciwgdGhpcmRPcmRlcikgPT4gewogIHJldHVybiBuZXcgUHJvbWlzZSgocmVzb2x2ZSkgPT4gewogICAgICBjb25zb2xlLmxvZyhgJHtvcmRlcn0gVGhpcyBsaW5lIGlzIChBKVN5bmNocm9ub3VzbHkgZXhlY3V0ZWRgKTsKICAgICAgcmVzb2x2ZShyZXNvbHZlZFZhbHVlKTsKICAgIH0pCiAgICAudGhlbigodmFsdWUpID0%2BIHsKICAgICAgY29uc29sZS5sb2coYCR7c2Vjb25kT3JkZXJ9IEFkZGl0aW9uYWwgbmVzdGVkIGNoYWluYCk7CiAgICAgIHJldHVybiB2YWx1ZTsKICAgIH0pCiAgICAudGhlbigodmFsdWUpID0%2BIHsKICAgICAgY29uc29sZS5sb2coYCR7dGhpcmRPcmRlcn0gQWRkaXRpb25hbCBuZXN0ZWQgY2hhaW5gKTsKICAgICAgcmV0dXJuIHZhbHVlOwogICAgfSk7Cn07CgpyZXR1cm5Qcm9taXNlKCIxc3QgUHJvbWlzZSIsICJbQi0yXSIsICJbQy00XSIsICJbRC01XSIpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0UtNl0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICAgIHJldHVyblByb21pc2UoIjJuZCBQcm9taXNlIiwgIltGLTddIiwgIltHLThdIiwgIltILTEwXSIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0ktOV0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCJbTi0zXSBTeW5jIHByb2Nlc3MiKTsK)
- ⚠️ 注意: JS Visualizer ではグローバルコンテキストは可視化されないので最初のマイクロタスク・タスクの実行タイミングについて誤解しないように注意してください
- ⚠️ 注意: JS Visualizer では可視化されていないマイクロタスクが存在しています

とにかく、Promise インスタンスを返すような処理は Promise chain において、`return` しないと意図した実行の順番を保証できないので、返す `return` するようにしてください。

```js
// promiseShouldBeReturnedNest-3rdReturn.js
console.log("🦖 [A] Sync");

const returnPromise = (resolvedValue, order, secondOrder, thirdOrder) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  })
    .then((value) => {
      console.log(`👹 ${secondOrder} Additional nested chain`);
      return value;
    })
    .then((value) => {
      console.log(`🦄 ${thirdOrder} Additional nested chain`);
      return value;
    });
};

returnPromise("1st Promise", "[B]", "[C]", "[D]")
  .then((value) => {
    console.log("👦 [E] Async");
    console.log("👦 Resolved value:", value);
    // ちゃんと return する
    return returnPromise("2nd Promise", "[F]", "[G]", "[H]");
  })
  .then((value) => {
    console.log("👦 [I] Async");
    console.log("👦 Resolved value:", value);
  });

console.log("🦖 [N] Sync");
```

上のコードでは、しっかりと `return` するように変更しました。このコードの実行結果は以下のようになります。

```sh
❯ deno run promiseShouldBeReturnedNest-3rdReturn.js
🦖 [A-1] Sync
👻 [B-2] (a)sync
🦖 [N-3] Sync
👹 [C-4] Additional nested chain
🦄 [D-5] Additional nested chain
👦 [E-6] Async
👦 Resolved value: 1st Promise
👻 [F-7] (a)sync
👹 [G-8] Additional nested chain
🦄 [H-9] Additional nested chain
👦 [I-10] Async
👦 Resolved value: 2nd Promise
```

しっかりと実行順番が制御できていますね。

今までの例ではそこまで重要なことに思えないかもしれませんが、後のチャプターで説明するタスクを発行する `setTimeout()` 関数を使用した場合や、時間のかかる I/O 処理、またはネットを介したデータフェッチを行う `fetch()` 関数などを副作用として使ってしまった場合には、その処理が終わっていないにも関わらず次の `then()` メソッドのコールバックが実行されたりして意図した処理結果とならない場合がでてきます。

例えば、次のコードでは、17 行目の `returnPromise("2nd Promise", "6", "8");` では内部の `setTimeout()` 関数の処理が完了するのを待たずに、次の `then()` メソッドのコールバック関数が実行されてしまいます。

```js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order, nextOrder) => {
  return new Promise((resolve) => {
    console.log(`${order} (a)sync`);
    setTimeout(() => {
      console.log(`${nextOrder} Always async`);
      resolve(resolvedValue);
    }, 3000);
  });
};

returnPromise("1st Promise", "[2]", "[4]")
  .then((value) => {
    console.log("[5] Async");
    console.log("Resolved value: ", value);
    returnPromise("2nd Promise", "[6]", "[8]"); // 7 ではなく 8 となる
  })
  .then((value) => {
    console.log("[7] Async");
    console.log("Resolved value: ", value); // undefined が表示される
  });

console.log("[3] Sync process");
```

## Promise chain の目的

もうすこし一般化して考えてみます。Promise chain では正しく処理を連鎖させることで逐次的に一連の処理を行うことができます。

今まで `then()` メソッドのコールバックで同期処理をして、また次の `then()` メソッドのコールバックをしていたため、そもそも、今までの Promise chain は本質的にあまり意味の無い行為でした。例えば、『[Promise chain で値を繋ぐ](7-epasync-pass-value-to-the-next-chain)』のチャプターで見た次のコードですが、本来このようにチェーンを無駄に長くする必要などありません。

```js
// chainValueName.js
console.log("🦖 [1] MAINLINE(Start): Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} Sync`);
    resolve(resolvedValue);
  });
};

// 文字列 "🐵 1st Promise" で解決された後にその値を最後まで連鎖させる
returnPromise("🐵 1st Promise", "[2]")
  .then((value) => {
    console.log("👦 [4]", value); // 🐵 1st Promise
    return value;
  })
  .then((value) => {
    console.log("👦 [5]", value); // 🐵 1st Promise
    return value;
  })
  .then((value) => {
    console.log("👦 [6]", value); // 🐵 1st Promise
    return value;
  })
  .then((value) => {
    console.log("👦 [7]", value); // 🐵 1st Promise
  });

console.log("🦖 [3] MAINLINE(End): Sync");
```

↓ いらない Promise chain をなくしてみます。

```js
// chainValueName-kai.js
console.log("🦖 [1] Sync");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`👻 ${order} (a)sync`);
    resolve(resolvedValue);
  });
};
returnPromise("1st Promise", "[2]").then((value) => {
  console.log("👦 Resolved value: ", value); // 1st Promise
  console.log("👦 Resolved value: ", value); // 1st Promise
  console.log("👦 Resolved value: ", value); // 1st Promise
  console.log("👦 Resolved value: ", value); // 1st Promise
});

console.log("🦖 [3] Sync");
```

Promise chain を利用する用途は基本的には、「非同期処理を逐次的に行う」ような場合や「Promise インスタンスから解決値を取り出して処理する」ような場合や「非同期処理のエラーハンドリング」を行うためとなります。

### 非同期処理を逐次的に行う

>Promise chain では正しく処理を連鎖させることで逐次的 (順番に) に一連の処理を行うことができます。

このように言いましたが、`chainValueName-kai.js` で見たように同期処理なら１つの `then()` メソッドのコールバック関数内にすべて書いてしまえばそれですべて順番に行えます。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
doAsyncTask()
  .then(() => {
    // すべて同期処理
    doSthSync1();
    doSthSync2();
    doSthSync3();
    doSthSync4();
    doSthSync5();
  });
```

しかし、ある非同期処理 A が終わってから別の非同期処理 B を行い、それが完了してからまた別の非同期処理 C を行いたい場合はどうでしょうか。もう少し具体的に言うと非同期処理 A の結果としてなにかデータが返ってきて、そのデータを非同期処理 B で加工して、さらにそこから返ってきたデータを非同期処理 C で再び加工して出力するなどの場合です。

それぞれの処理を Promise インスタンスを返す関数として考えてみます。まずは失敗するパターンから。それぞれの非同期処理を次の関数で考えてみます。

- `doSthAsyncA(path)` : `path` にあるデータを 3000ms かけて取得する
- `doSthAsyncB(data)` : 引数に渡した `data` を 3000ms かけて加工する
- `doSthAsyncC(data)` : 引数に渡した `data` を 3000ms かけて加工する

実際にはこれらの関数内部でメインスレッドのブロッキングを行わない何かしらの非同期 API (`fetch()` メソッドなど) が使われていると想定しておきます。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
doAsyncTask()
  .then(() => {
    // これは失敗する
    const data = doSthAsyncA(path);
    const processedData_1st = doSthAsyncB(data);
    // data はまだ無いのに加工してしまっている
    const processedData_2nd = doSthAsyncC(processedData_1st);
    // processedData_1st はまだ無いのに加工してしまっている
    console.log(processedData_2nd); // undefined
  });
```

上のコードはあきらかに正しくないですね。コールバック関数の中は上から下へと順番に実行されますが、`doSthAsyncA()` は非同期の関数であり、時間がかかります。また内部で非同期 API を利用していることから、ブロッキングしないはずなので、そのまま処理完了をまたずに次の処理 `doSthAsyncB()` が実行されます。

データ取得が完了していないので、`data` は `undefined` で渡されてしまいます。そしてまた時間をかけて (undefined なので実際に時間はかかるかどうかはわかりませんが) とにかく、存在しないデータを加工して、次の処理へと移行し、再び存在しないデータを `doSthAsyncC()` で加工して出力してしまっています。

結果として Promise インスタンスを返す非同期処理を順番に行うには、Promise chain を正しく構築しないといけません。Promise インスタンスを返す非同期処理を逐次的に (順番に) 実行させるには次のよう返ってくるはずの Promise インスタンスを `return` をさせます。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
// 正しい Promise chain
doAsyncTask()
  .then(() => {
    // Promise インスタンスを返す関数
    return doSthAsyncA(path);
  })
  .then(data => {
    return doSthAsyncB(data);
  })
  .then(data => {
    return doSthAsyncC(data);
  })
  .then(date => {
    console.log(data); // 加工したデータが表示される
  });
```

これで、非同期処理 A が終わってから次の非同期処理 B を行い、そして B が終わってから次の非同期処理 C を行い、C が終わってからコンソールに出力できています。さらに、Promise chain において値を繋いでいることがわかります。

`return` をつけることで確実にそれぞれの非同期処理が完了してから次にいくことができています。

さらにアロー関数の省略形を使うことで `return` を省略して次のように書くこともできます。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
// 正しい Promise chain
doAsyncTask()
  .then(() => doSthAsyncA(path))
  .then(data => doSthAsyncB(data))
  .then(data => doSthAsyncC(data))
  .then(date => console.log(data)); // 加工したデータが表示される
```

それでは、次の場合はどうでしょうか？

```js
// 非同期処理 doAsyncTask() が完了したら何かする
doAsyncTask()
  .then(() => {
    const data = doSthAsyncA(path);
    return data;
  })
  .then(data => {
    const processedData_1st = doSthAsyncB(data);
    return processedData_1st;
  })
  .then(data => {
    const processedData_2nd = doSthAsyncC(processedData_1st);
    return processedData_2nd;
  })
  .then(date => {
    console.log(data);
  });
```

この場合も OK です。非同期処理から返ってくる Promise インスタンスをコールバック関数の中で `return` しているのですべて順次的に実行されて、ちゃんと値もつながるので最終的に欲しいデータが出力されます。

ただし、`then()` メソッドのコールバック関数内において実行した非同期処理は次のチェーンに行くまでに終わっていないことに注意してください。

次のように途中経過を見ようとして `console.log(data)` を行ってもまだ非同期処理は終わっていませんのでログに出力することはできません。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
doAsyncTask()
  .then(() => {
    const data = doSthAsyncA(path);
    console.log(data);
    return data;
  })
  .then(data => {
    const processedData_1st = doSthAsyncB(data);
    console.log(processedData_1st);
    return processedData_1st;
  })
  .then(data => {
    const processedData_2nd = doSthAsyncC(processedData_1st);
    console.log(processedData_2nd);
    return processedData_2nd;
  })
  .then(date => {
    console.log(data);
  });
```

そして、重要なこととして、`console.log()` で出力した実際のログには `Promise { <pending> }` という値が表されます。非同期処理 A, B, C はそもそも Promise インスタンスを返す非同期処理でした。実際に値を取り出して経過を見たり、追加で何かしらの処理を行うにはどうすればよいでしょうか?

### Promise インスタンスから解決値を取り出して処理する

結論としては、`then()` メソッドのコールバック内で Promise インスタンスを `return` して次の `then()` メソッドのコールバックへ値を繋いでから、処理や出力を行います。上のコードで `console.log()` の位置をずらすことで Promise の解決値をログに出力して確認できます。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
doAsyncTask()
  .then(() => {
    const data = doSthAsyncA(path);
    return data;
  })
  .then(data => {
    console.log(data); // ここでデータを見る
    const processedData_1st = doSthAsyncB(data);
    return processedData_1st;
  })
  .then(data => {
    console.log(data); // ここでデータを見る
    const processedData_2nd = doSthAsyncC(processedData_1st);
    return processedData_2nd;
  })
  .then(date => {
    console.log(data);
  });
```

結論はもう言ってしまったのですが、Promise chain のもう 1 つの用途である「Promise インスタンスから解決値を取り出して処理する」について解説します。この項目については「非同期処理を逐次的に行う」の項目とかぶる部分があります。

非同期処理を逐次的に行う例として非同期 API `fetch()` メソッドを利用して例をあげます。

『[非同期 API と環境](f-epasync-asynchronous-apis)』のチャプターで解説したように、Deno では本来 Web API であるはずの `fetch()` が Web 互換な API (Web Platform APIs) として同じ名前・同じ使い勝手で提供されています。

:::message
ちなみに Node 18 でも `fetch()` メソッドが利用できるようになったそうです。

[Node.js 18 is now available! | Node.js](https://nodejs.org/en/blog/announcements/v18-release-announce/)
:::

Deno では `fetch()` を使ったローカルファイルの取得をサーバーを立てることなくできるようになっています。

次のコードでは、ローカルファイルへの相対パスから絶対ファイル URL を作成して `fetch()` に渡しています。

```js
const relativePath = "./testTextFile/textForFetch.txt";
const localUrl = new URL(relativePath, import.meta.url).toString();

console.log("sync process 1");

fetch(localUrl)
  .then((response) => {
    if (!response.ok) {
      throw new Error("Error");
    }
    console.log(`got data from "${localUrl}"`);
    return response.text();
  })
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error(error.message);
  });

console.log("sync process 2");
```

このファイルが存在するディレクトリからの相対パス `./testTextFile/` のロケーションに適当なテキストファイルを用意しておきます。

```txt:testTextFile/textForFetch.txt

In laboris aliquip pariatur aliqua officia veniam quis aliquip. Dolor eu magna reprehenderit pariatur pariatur labore officia. Sit irure et excepteur dolor. Minim tempor nisi nulla veniam mollit. Esse elit aute reprehenderit id minim non et anim non id. Quis sunt elit labore officia voluptate cillum incididunt labore mollit ea adipisicing dolor eiusmod. Veniam cupidatat mollit occaecat mollit ullamco.

```

実行すると次のような出力を得ます。Deno ではパーミッションフラグをつけないとローカルファイルの読み取りができないので実行の際には `--allow-read` フラグをつけて利用しています。

```js
❯ deno run --allow-read denoFetchLocal.js
sync process 1
sync process 2
got data from "file:///Users/roshi/Development/Testing/understanding-async/deno-async/testTextFile/textForFetch.txt"

In laboris aliquip pariatur aliqua officia veniam quis aliquip. Dolor eu magna reprehenderit pariatur pariatur labore officia. Sit irure et excepteur dolor. Minim tempor nisi nulla veniam mollit. Esse elit aute reprehenderit id minim non et anim non id. Quis sunt elit labore officia voluptate cillum incididunt labore mollit ea adipisicing dolor eiusmod. Veniam cupidatat mollit occaecat mollit ullamco.

```

このコードで注目してほしいのは、最初の `then()` メソッドのコールバックの最後で、`return response.text();` を行っていることです。

```js
fetch(localUrl)
  .then((response) => {
    if (!response.ok) {
      throw new Error("Error");
    }
    console.log(`got data from "${localUrl}"`);
    return response.text();
  })
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error(error.message);
  });
```

`response.text()` は Promise インスタンスを返します。そして実際に解決されれる値、つまりテキストデータの文字列に対して何か処理を行ったり、コンソールに表示させたりするためには、一度 Promise chain で次の `then()` メソッドのコールバックへ渡しすために `return` する必要があります。

これは、`response.json()` なども同じで Promise インスタンスを返すような処理については Promise chain で値を繋いでから何かします。

ところで、『[Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)』のチャプターで Promise chain のネストは基本的にはアンチパターンであると言いましたが、ネストが深くならないなら、別にやっても問題は無いです。ただネストを行う場合にはエラーハンドリングに気をつけましょう。

```js
fetch(localUrl)
  .then((response) => {
    if (!response.ok) {
      throw new Error("Error");
    }
    console.log(`got data from "${localUrl}"`);
    // 完結に書くために敢えてネストさせた
    return response.text().then(data => console.log(data));
  })
  .catch((error) => {
    console.error(error.message);
  });
```

Promise インスタンスから解決値を取り出す方法としては、実はもう１つ await 式がありますが、詳細はここでは解説しません。『[Promise chain から async 関数へ](14-epasync-chain-to-async-await)』のチャプターで詳しく解説します。

```js
// async function 内部で Promise インスタンスから直接値を取り出す
const getDataByAwait = async () => {
  // myValue は直接的な値
  const myValue = await returnPromise().then(result => result.data);
  return myValue;
  // ただし返り値は Promise インスタンスにラップされる
};

// top-level await ならこのスコープで取り出せる
const myLocalValue = await returnPromise().then(result => result.data);
```
