---
title: "then メソッドのコールバックで非同期処理"
---

# 副作用とは
このチャプターの本題に入る前に「副作用(Side Effect)」とは何かを簡単に説明しておきます。

副作用の概念は関数型プログラミングの文脈などでよくでてくるもので、関数型プログラミングの考え方の１つとして、「副作用の使用を避けて可能な限り Pure に考える」というものがあります。

Pure とは関数を純粋関数(Pure function)にするということを意味しています。関数の基本は「入力を受け取って出力を返す」というものですが、出力に関与しないような操作を関数内で行わないというのが「副作用の使用を避ける」ことになります。これに加えて、入力以外のグローバル変数などを計算に使用せずに出力の値を計算して返すような関数を純粋関数と呼びます(正確な定義はべつのところへ任せます)。

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

@[youtube](e-5obm1G_FY)

# returnの代わりに副作用を使用しない
さて、今まで `then()` メソッドのコールバック関数内にて返すものとしては次のパターンでした。

- (1) 文字列や数値などの通常の値を `return` する
  - 直ちに次の `then()` メソッドのコールバックが Microtask queue へと追加されて、コールバック関数の引数には `return` した値が渡される
- (2) Promsie インスタンスを `return` する
  - 待機状態ならそれが解決してから次の `then()` メソッドのコールバックが Microtask queue へと追加され、`resolve` した値がコールバック関数の引数に渡される
  - 履行状態なら直ちに次の `then()` メソッドのコールバックが Microtask queue へと追加され、`resolve` した値がコールバック関数の引数に渡される
- (3) 何も `return` しない
  - 直ちに次の `then()` メソッドのコールバックが Microtask queue へと追加されて、コールバック関数の引数は `undefined` となる

(2) と (3) を混同してしまう場合に気をつけてください。`then()` メソッドのコールバック関数で Promise を使った非同期処理を行う場合には必ず Promise インスタンスを `return` するようにしてください。`then()` メソッドのコールバック関数内部で、非同期処理を使用する場合に、`return` をして Promise インスタンスを返していない場合、その非同期処理は「**副作用(Side Effect)**」となります。この場合、次の `then()` メソッドのコールバック関数へ値を繋ぐことができなくなり、そもそも意図した実行順番にならなくなる場合があります。

次のコードの例では、Promise チェーンで値が繋がりません。

```js
// promiseShouldBeReturned.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // return しない場合は副作用となり値が渡らない
    returnPromise("2nd Promise", "[6]")
  })
  .then((value) => {
    // この value は undefined となる
    console.log("[9] This line is Asynchronously executed");
    console.log("Resolved value: ", value); // undefined が表示される
  });
returnPromise("3rd Promise", "[3]")
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // Promise インスタンスについては必ず return するようにする
    return returnPromise("4th Promise", "[8]")
  })
  .then((value) => {
    console.log("[10] This line is Asynchronously executed");
    console.log("Resolved value: ", value); // 値が繋がるので 4th Promise と表示される
  });

console.log("[4] Sync process");
```

これを実行すると次の出力を得ます。`undefined` となっているところに注目してください。

```sh
❯ deno run promiseShouldBeReturned.js
[1] Sync process
[2] This line is (A)Synchronously executed
[3] This line is (A)Synchronously executed
[4] Sync process
[5] This line is Asynchronously executed
Resolved value:  1st Promise
[6] This line is (A)Synchronously executed
[7] This line is Asynchronously executed
Resolved value:  3rd Promise
[8] This line is (A)Synchronously executed
[9] This line is Asynchronously executed
Resolved value:  undefined
[10] This line is Asynchronously executed
Resolved value:  4th Promise
```

従って、値を正しく繋げたい場合には、副作用ではなく `return` をつけるようにしましょう。

今度は、もう少し簡単にしてみます。これまで２つのメインとなる Promise チェーンで考えていましたが、ここでは１つにします。その代わりに、Promise チェーン内部であえてネストを作ります。再びテストとして次のコードで `[A-G]` までの文字がどのような順番で出力されるか考えてみてください。

```js
// promiseShouldBeReturnedAddThen-right.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  });
};

returnPromise("1st Promise", "[B]")
  .then((value) => {
    console.log("[C] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // return で正しいチェーンを作る
    return returnPromise("2nd Promise", "[D]")
      .then((value) => {
        console.log("[E] This line is Asynchronously executed");
        console.log("Resolved value: ", value);
        return "Pass next value";
      });
  })
  .then((value) => {
    console.log("[F] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[G] Sync process");
```

「then コールバックで Promise インスタンスを返す」や「Promise チェーンはネストさせない」のチャプターでネストは経験したので正解できましたか?

:::details 答え
答えは、「A → B → G → C → D → E → F」となります。

```sh:数字付きで出力
❯ deno run promiseShouldBeReturnedAddThen-right.js
[A-1] Sync process
[B-2] This line is (A)Synchronously executed
[G-3] Sync process
[C-4] This line is Asynchronously executed
Resolved value:  1st Promise
[D-5] This line is (A)Synchronously executed
[E-6] This line is Asynchronously executed
Resolved value:  2nd Promise
[F-7] This line is Asynchronously executed
Resolved value:  Pass next value
```
:::

`return returnPromise("2nd Promise", "[D]").then(callback)` の部分において Promise チェーンをネストさせていますが、ここで `return` しているのは最終的に `then(callback)` で返ってくる Promise インスタンスでした。

そして、`then()` メソッドのコールバック関数内にて返すものとして Promise インスタンスを選択した場合には、それが解決してから(実行が完了してから)次の `then()` メソッドのコールバック関数が実行されるという話でした。

# returnしないと非同期処理の完了を待てない
もう少し複雑化してみましょう。あとで代わりに Promise-based な非同期 API である `fetch()` 関数を使用した説明も行います。

次のコードでは、今までのコードでメインとなる Promise チェーンを１つにした上で、`returnPromise()` 関数内で Promise チェーンを行うように改造しました。つまり Promise チェーンをネストさせています。

Promise インスタンスを返す処理は常に `return` するべきですが、このコードではあえて `return` させていません。

まずは次のコードを感がてみましょう。
実際の実行順番とアルファベット `[A-H]` の出力順番はどうなるでしょうか？予測してみてください。

```js
// promiseShouldBeReturnedNest.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order, nextOrder) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
    // ↓ ここにチェーンを追加してみる
  }).then((value) => {
    console.log(`${nextOrder} Additional nested chain`);
    return value;
  });
};

returnPromise("1st Promise", "[B]", "[C]")
  .then((value) => {
    console.log("[D] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // ここで敢えて return しないとどういう実行順番になるか?
    returnPromise("2nd Promise", "[E]", "[F]");
  })
  .then((value) => {
    console.log("[G] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[H] Sync process");
```

:::details 答え
答えは、「A → B → H → C → D → E → F → G」となります。

数字付きで実際に出力してみるとこうなります。
```sh
❯ deno run promiseShouldBeReturnedNest-wrong.js
[A-1] Sync process
[B-2] This line is (A)Synchronously executed
[H-3] Sync process
[C-4] Additional nested chain
[D-5] This line is Asynchronously executed
Resolved value:  1st Promise
[E-6] This line is (A)Synchronously executed
[F-8] Additional nested chain
[G-7] This line is Asynchronously executed
Resolved value:  undefined
```
:::

最後の出力である `Resovled value` のところが `undefined` になっているので、値 `"2nd Promsie"` が繋げていないことがわかります。
実行順番については、基本的に Promise インスタンスを返すような処理は `return` しないと順番を保証できないのですが、今回の場合は `returnPromise("2nd Promise", "[E]", "[F]");` が完了してから、次の `then()` メソッドのコールバックが実行されていますね。その理由としては、Microtask を供給する Promsie が少ないからたまたまそうなっているだけです。実際の動きを Visualizer で確認してみてください。

- [promiseShouldBeReturnedNest.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gcHJvbWlzZVNob3VsZEJlUmV0dXJuZWROZXN0LmpzCmNvbnNvbGUubG9nKCJbQS0xXSBTeW5jIHByb2Nlc3MiKTsKCmNvbnN0IHJldHVyblByb21pc2UgPSAocmVzb2x2ZWRWYWx1ZSwgb3JkZXIsIG5leHRPcmRlcikgPT4gewogIHJldHVybiBuZXcgUHJvbWlzZSgocmVzb2x2ZSkgPT4gewogICAgY29uc29sZS5sb2coYCR7b3JkZXJ9IFRoaXMgbGluZSBpcyAoQSlTeW5jaHJvbm91c2x5IGV4ZWN1dGVkYCk7CiAgICByZXNvbHZlKHJlc29sdmVkVmFsdWUpOwogIH0pLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZyhgJHtuZXh0T3JkZXJ9IEFkZGl0aW9uYWwgbmVzdGVkIGNoYWluYCk7CiAgICByZXR1cm4gdmFsdWU7CiAgfSk7Cn07CgpyZXR1cm5Qcm9taXNlKCIxc3QgUHJvbWlzZSIsICJbQi0yXSIsICJbQy00XSIpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0QtNV0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICAgIHJldHVybiByZXR1cm5Qcm9taXNlKCIybmQgUHJvbWlzZSIsICJbRS02XSIsICJbRi03XSIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0ctOF0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCJbSC0zXSBTeW5jIHByb2Nlc3MiKTs%3D)

Promise インスタンスを返すような処理を `return` しない場合に事項順番が保証できなくなってしまう例を挙げてみます。次のコードでは、`returnPromise()` 関数の内部に `then()` メソッドを更に追加して Promise チェーンを伸ばしています。実行順番を予想してみてください。

```js
// promiseShouldBeReturnedNest-3rd.js
console.log("[A] Sync process");

const returnPromise = (resolvedValue, order, secondOrder, thirdOrder) => {
  return new Promise((resolve) => {
      console.log(`${order} This line is (A)Synchronously executed`);
      resolve(resolvedValue);
    })
    .then((value) => {
      console.log(`${secondOrder} Additional nested chain`);
      return value;
    })
    .then((value) => {
      console.log(`${thirdOrder} Additional nested chain`);
      return value;
    });
};

returnPromise("1st Promise", "[B]", "[C]", "[D]")
  .then((value) => {
    console.log("[E] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    returnPromise("2nd Promise", "[F]", "[G]", "[H]");
  })
  .then((value) => {
    console.log("[I] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[N] Sync process");
```

:::details 答え
答えは、「A → B → N → C → D → E → F → G → I → H」となります。

数字付きで実際に出力してみるとこうなります。
```sh
❯ deno run promiseShouldBeReturnedNest-3rd.js
[A-1] Sync process
[B-2] This line is (A)Synchronously executed
[N-3] Sync process
[C-4] Additional nested chain
[D-5] Additional nested chain
[E-6] This line is Asynchronously executed
Resolved value:  1st Promise
[F-7] This line is (A)Synchronously executed
[G-8] Additional nested chain
[I-9] This line is Asynchronously executed
Resolved value:  undefined
[H-10] Additional nested chain
```
:::

注目してほしいのは、`[H]` と `[I]` の順番です。H が終わっていないのに、I が実行されていますね。上のコードでは Microtask queue へと連続で Microtask をどんどん送っていますが、その送る順番が `return` をしなかったことで、`returnPromise("2nd Promise", "[F]", "[G]", "[H]");` 内部の Promise チェーンが 2 番目の `then()` メソッドのコールバックが終わって Call stack が空になった習慣に、`returnPromise("1st Promise", "[B]", "[C]", "[D]").then(cb1).then(cb2)` のコールバック `cb2` がキューへと送られてしまうためです。

言葉で説明するのが難しいので、実際に見てみてください。

- [promiseShouldBeReturnedNest-3rd.js - JS Visualizer](https://www.jsv9000.app/?code=Ly8gcHJvbWlzZVNob3VsZEJlUmV0dXJuZWROZXN0LTNyZC5qcwpjb25zb2xlLmxvZygiW0EtMV0gU3luYyBwcm9jZXNzIik7Cgpjb25zdCByZXR1cm5Qcm9taXNlID0gKHJlc29sdmVkVmFsdWUsIG9yZGVyLCBzZWNvbmRPcmRlciwgdGhpcmRPcmRlcikgPT4gewogIHJldHVybiBuZXcgUHJvbWlzZSgocmVzb2x2ZSkgPT4gewogICAgICBjb25zb2xlLmxvZyhgJHtvcmRlcn0gVGhpcyBsaW5lIGlzIChBKVN5bmNocm9ub3VzbHkgZXhlY3V0ZWRgKTsKICAgICAgcmVzb2x2ZShyZXNvbHZlZFZhbHVlKTsKICAgIH0pCiAgICAudGhlbigodmFsdWUpID0%2BIHsKICAgICAgY29uc29sZS5sb2coYCR7c2Vjb25kT3JkZXJ9IEFkZGl0aW9uYWwgbmVzdGVkIGNoYWluYCk7CiAgICAgIHJldHVybiB2YWx1ZTsKICAgIH0pCiAgICAudGhlbigodmFsdWUpID0%2BIHsKICAgICAgY29uc29sZS5sb2coYCR7dGhpcmRPcmRlcn0gQWRkaXRpb25hbCBuZXN0ZWQgY2hhaW5gKTsKICAgICAgcmV0dXJuIHZhbHVlOwogICAgfSk7Cn07CgpyZXR1cm5Qcm9taXNlKCIxc3QgUHJvbWlzZSIsICJbQi0yXSIsICJbQy00XSIsICJbRC01XSIpCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0UtNl0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICAgIHJldHVyblByb21pc2UoIjJuZCBQcm9taXNlIiwgIltGLTddIiwgIltHLThdIiwgIltILTEwXSIpOwogIH0pCiAgLnRoZW4oKHZhbHVlKSA9PiB7CiAgICBjb25zb2xlLmxvZygiW0ktOV0gVGhpcyBsaW5lIGlzIEFzeW5jaHJvbm91c2x5IGV4ZWN1dGVkIik7CiAgICBjb25zb2xlLmxvZygiUmVzb2x2ZWQgdmFsdWU6ICIsIHZhbHVlKTsKICB9KTsKCmNvbnNvbGUubG9nKCJbTi0zXSBTeW5jIHByb2Nlc3MiKTsK)

とにかく、Promsie インスタンスを返すような処理は Promise チェーンにおいて、`return` しないと意図した実行の順番を保証できないので、返す `return` するようにしてください。

```js
// promiseShouldBeReturnedNest-3rdReturn.js
console.log("[A-1] Sync process");

const returnPromise = (resolvedValue, order, secondOrder, thirdOrder) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is (A)Synchronously executed`);
    resolve(resolvedValue);
  })
    .then((value) => {
      console.log(`${secondOrder} Additional nested chain`);
      return value;
    })
    .then((value) => {
      console.log(`${thirdOrder} Additional nested chain`);
      return value;
    });
};

returnPromise("1st Promise", "[B-2]", "[C-4]", "[D-5]")
  .then((value) => {
    console.log("[E-6] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    // ちゃんと return する
    return returnPromise("2nd Promise", "[F-7]", "[G-8]", "[H-9]");
  })
  .then((value) => {
    console.log("[I-10] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
  });

console.log("[N-3] Sync process");
```

上のコードでは、しっかりと `return` するように変更しました。このコードの実行結果は以下のようになります。

```sh
❯ deno run promiseShouldBeReturnedNest-3rdReturn.js
[A-1] Sync process
[B-2] This line is (A)Synchronously executed
[N-3] Sync process
[C-4] Additional nested chain
[D-5] Additional nested chain
[E-6] This line is Asynchronously executed
Resolved value:  1st Promise
[F-7] This line is (A)Synchronously executed
[G-8] Additional nested chain
[H-9] Additional nested chain
[I-10] This line is Asynchronously executed
Resolved value:  2nd Promise
```

しっかりと実行順番が制御できていますね。

今までの例ではそこまで重要なことに思えないかもしれませんげ、後のチャプターで説明する Macrotask を発行する `setTimeout()` 関数を使用した場合や、時間のかかる I/O 処理、またはインターネットを介したデータ取得を行う `fetch()` 関数などを副作用として使ってしまった場合には、その処理が終わっていないにも関わらず次の `then()` メソッドのコールバックが実行されてしまいます。

例えば、次のコードでは、17 行目の `returnPromise("2nd Promise", "6", "8");` では内部の `setTimeout()` 関数の処理が完了するのを待たずに、次の `then()` メソッドのコールバック関数が実行されてしまいます。

```js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order, nextOrder) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    setTimeout(() => {
      console.log(`${nextOrder} This line is always Asynchronously executed`);
      resolve(resolvedValue);
    }, 3000);
  });
};

returnPromise("1st Promise", "[2]", "[4]")
  .then((value) => {
    console.log("[5] This line is Asynchronously executed");
    console.log("Resolved value: ", value);
    returnPromise("2nd Promise", "[6]", "[8]"); // 7 ではなく 8 となる
  })
  .then((value) => {
    console.log("[7] This line is Asynchronously executed");
    console.log("Resolved value: ", value); // undefined が表示される
  });

console.log("[3] Sync process");
```

# Promise チェーンの目的
もうすこし一般化して考えてみます。Promsie チェーンでは正しく処理を連鎖させることで逐次的(順番に)に一連の処理を行うことができます。

今まで `then()` メソッドのコールバックで同期処理をして、また次の `then()` メソッドのコールバックをしていたため、そもそも、今までの Promise チェーンは本質的にあまり意味の無い行為でした。例えば、チャプター７「Promise チェーンで値を繋ぐ」で見た次のコードですが、本来このように無駄にチェーンを長くする必用など基本的にはありません。

```js
// chainValueName.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};
returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
    return value;
  })
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
    return value;
  })
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
    return value;
  })
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
  });

console.log("[3] Sync process");
```

↓ いらない Promise チェーンをなくしてみます。

```js
// chainValueName-kai.js
console.log("[1] Sync process");

const returnPromise = (resolvedValue, order) => {
  return new Promise((resolve) => {
    console.log(`${order} This line is Synchronously executed`);
    resolve(resolvedValue);
  });
};
returnPromise("1st Promise", "[2]")
  .then((value) => {
    console.log("Resolved value: ", value); // 1st Promise
    console.log("Resolved value: ", value); // 1st Promise
    console.log("Resolved value: ", value); // 1st Promise
    console.log("Resolved value: ", value); // 1st Promise
  });

console.log("[3] Sync process");
```

Promsire チェーンを利用する用途は基本的には、「非同期処理を逐次的に行う」ような場合や「Proimse インスタンスから解決値を取り出して処理する」ような場合や「非同期処理のエラーハンドリング」を行うためとなります。

:::message
エラーハンドリングは非常に重要な事柄ではあるのですが、この本では非同期処理の制御の流れを掴むことを目的としているのでここまでほとんど扱いませんでした。

制御の流れが分かったら、エラーの伝播やエラーをどこで補足するかという点について理解する必要もあると思うので将来的に記載することにしています。
:::

## 非同期処理を逐次的に行う

>Promsie チェーンでは正しく処理を連鎖させることで逐次的(順番に)に一連の処理を行うことができます。

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

実際にはこれらの関数内部でメインスレッドのブロッキングを行わない何かしらの非同期 API (`fetch()` メソッドなど)が使われていると想定しておきます。

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

上のコードはあきらかに正しくないですよね。コールバック関数の中は上から下に順番に実行されますが、`doSthAsyncA()` は非同期の関数であり、時間がかかりますので、また内部で非同期 API を利用しているためブロッキングしないはずなので、そのまま処理完了をまたずに次の処理 `doSthAsyncB()` が実行されます。

データ取得が完了していないので、`data` は `undefiend` で渡されてしまいます。そしてまた時間をかけて(undefined なので実際に時間はかかるかどうかはわかりませんが)とにかく、存在しないデータを加工して、次の処理へと移行し、再び存在しないデータを `doSthAsyncC()` で加工して、出力してしまっています。

結果として Promise インスタンスを返す非同期処理を順番に行うには、Promise チェーンを正しく構築しないといけません。Promise インスタンスを返す非同期処理を逐次的に(順番に)実行させるには次のよう返ってくるはずの Promise インスタンスを `return` をさせます。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
// 正しい Promise チェーン
doAsyncTask()
  .then(() => {
    // Prosise インスタンスを返す関数
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

これで、非同期処理 A が終わってから次の非同期処理 B を行い、そして B が終わってから次の非同期処理 C を行い、C が終わってからコンソールに出力できています。さらに、Promsie チェーンにおいて値を繋いでいることがわかります。

`return` をつけることで確実にそれぞれの非同期処理が完了してから次にいくことができています。

さらにアロー関数の省略形を使うことで `return` を省略して次のように書くこともできます。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
// 正しい Promise チェーン
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

## Proimse インスタンスから解決値を取り出して処理する
結論としては、`then()` メソッドのコールバック内で Promise インスタンスを `reutrn` して次の `then()` メソッドのコールバックへ値を繋いでから、処理や出力を行います。上のコードで `console.log()` の位置をずらすことで Promise の解決値をログに出力して確認できます。

```js
// 非同期処理 doAsyncTask() が完了したら何かする
doAsyncTask()
  .then(() => {
    const data = doSthAsyncA(path);
    return data;
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

結論はもう言ってしまったのですが、Promsie チェーンのもう 1 つの用途である「Proimse インスタンスから解決値を取り出して処理する」について解説します。この項目については「非同期処理を逐次的に行う」の項目とかぶる部分があります。

あまり非同期 API の話はこの本の都合上まだやりたくないのですが(未来のチャプターでしっかり話したいので)非同期処理を逐次的に行う例として `fetch()` メソッドを利用して例をあげます。

次の記事で書いたようにに、Deno では本来 Web API であるはずの `fetch()` が Web 互換な API として提供されているので実際に使ってみます。
https://zenn.dev/estra/articles/deno-fetch-local-file-async-practice

:::message
ちなみに Node 18 でも `fetch()` メソッドが利用できるようになったそうです。

[Node.js 18 is now available! | Node.js](https://nodejs.org/en/blog/announcements/v18-release-announce/)
:::

Deno では `fetch()` を使ったローカルファイルの取得をサーバーを立てることなくできるようになっています。詳しくは上の記事を参照してください。

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

`response.text()` は Promise インスタンスを返します。そして実際に解決されれる値、つまりテキストデータの文字列に対して何か処理を行ったり、コンソールに表示させたりするためには、一度 Promise チェーンで次の `then()` メソッドのコールバックへ渡しすために `return` する必要があります。

これは、`response.json()` なども同じで Promise インスタンスを返すような処理については Promsie チェーンで値を繋いでから何かします。

ところで、チャプター９の「Promise チェーンはネストさせない」で Promise チェーンのネストは基本的にはアンチパターンであると言いましたが、ネストが深くならないなら、別にやっても問題は無いです。ただネストを行う場合にはエラーハンドリングに気をつけましょう。

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

Promise インスタンスから解決値を取り出す方法としては、実はもう１つ `await` 式というのがありますが、これについては詳細はここでは解説しません。

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

