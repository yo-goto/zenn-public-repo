---
title: "await 式の配置による制御"
aliases: [ch_await 式の配置による制御]
---

# このチャプターについて

非同期処理の学習の後半戦では「**await 式の配置**」がキーになります。await 式の配置次第で実行や完了の順番が変わってくるため、**await 式の配置によって効率的に順序付けする、あるいは意図的に順序付けしないことが重要です**。

このチャプターに到達するまでに Promise やら非同期 API の挙動については十分に学んだので、今度はこちら側から意図的に「制御の流れ」を作りだしていく訓練を行います。

# 不定性の制御

前の『[Promise の静的メソッドと並列化](17-epasync-static-method)』のチャプターにおいて、`fetch()` から始まる chain の並列化(本質的には非同期 API の並列化)を見ましたが、次のように変形してみるとどこかで視たことがある気がしてきます。

```js
fetch(urls[0])
  .then(response => response.text())
  .then(text => console.log(text)); 
fetch(urls[1])
  .then(response => response.text())
  .then(text => console.log(text));
fetch(urls[2])
  .then(response => response.text())
  .then(text => console.log(text));
```

実は『[複数の Promise を走らせる](5-epasync-multiple-promises)』のチャプターで次のように Promise を返す関数を複数起動させましたが、やっていることはかなり近いことに気づきます。

```js
function returnPromise(value, order) {
  return new Promise((resolve) => {
    console.log(`👻 ${order} 同期的に直ちに履行`);
    resolve(value);
  });
}
returnPromise("1st", "[1]")
  .then(val => console.log("👦 [4]", val))
  .then(() => console.log("👦 [7]"));
returnPromise("2nd", "[2]")
  .then(val => console.log("👦 [5]", val))
  .then(() => console.log("👦 [8]"));
returnPromise("3rd", "[3]")
  .then(val => console.log("👦 [6]", val))
  .then(() => console.log("👦 [9]"));  
```

直ちに履行する Promise インスタンスを返す関数を複数起動させた時の場合に対して、`fetch()` は完了に時間がかかりバックグラウンドで並列化しますが、類型的にはほぼ同じです。ただしこの場合、３つの `fetch()` の内でどれが最初に完了するかは分かりません。並列的に(ほぼ同時に)リクエストを投げているのでどれが最初に完了するかはその時々変わります。このような**不定性**が気になり、意図的に順序付けしたいなら await 式で制御する必要があります。

このような不定性は Promisification を行った `setTimeout()` の遅延時間を `Math.random()` などでランダムにしてあげることで擬似的に再現できます。次のコードでの実行順序は不定となり、その時々によって結果が変わります。これは非同期 API の `setTimeout()` が複数起動できる性質(環境がバックグラウンドで並列的にタイマー処理するので時間継続によって並列化する性質)を持ち、更に遅延時間をランダムにしているからです。このコードで担保されているのは chain においてアルファベットが同じ部分については数字の順番に実行・完了していくということだけです。

```js
function randomTimer(value, order) {
  return new Promise((resolve) => {
    // Promise() コンストラクタのコールバックである Executor 関数自体は同期的に実行される
    // つまり setTimeout() の起動自体は同期的に行われる
    setTimeout(() => { // setTimeout() のコールバックは非同期的に実行される
      console.log(`👻 ${order} いつ履行するか分からない`);
      resolve(value);
    }, 1000 * Math.random()); // 遅延時間をランダムに
  });
}
// 起動自体は 1st → 2nd → 3rd だが、出力結果は実行するたびに変わるので不定である
randomTimer("1st", "[A-1]")
  .then(val => console.log("👦 [A-2]", val))
  .then(() => console.log("👦 [A-3]")); // A-1 → A-2 → A-3 の保証
randomTimer("2nd", "[B-1]")
  .then(val => console.log("👦 [B-2]", val))
  .then(() => console.log("👦 [B-3]")); // B-1 → B-2 → B-3 の保証
randomTimer("3rd", "[C-1]")
  .then(val => console.log("👦 [C-2]", val))
  .then(() => console.log("👦 [C-3]"));  // C-1 → C-2 → C-3 の保証
```

特定範囲内での不定性の制御したいなら async 関数内で await 式を配置して順序づけを行うことで、**この範囲内での実行と完了の順番を担保できます**。いつもどおり、この async 関数の範囲外にあるコードが await 式で中断している最中にもメインスレッドで実行されているので、すべてのコードの実行順序ではないことに注意してください。

```js
(async () => {
  // この範囲内での実行と完了の順番を担保する
  await randomTimer("1st", "[A-1]")
    .then(val => console.log("👦 [A-2]", val))
    .then(() => console.log("👦 [A-3]"));
  console.log("randomTimer chainの１つ目が完了しました");

  await randomTimer("2nd", "[A-4]")
    .then(val => console.log("👦 [A-5]", val))
    .then(() => console.log("👦 [A-6]"));
  console.log("randomTimer chainの２つ目が完了しました");

  await randomTimer("3rd", "[A-7]")
    .then(val => console.log("👦 [A-8]", val))
    .then(() => console.log("👦 [A-9]"));
  console.log("すべてのrandomTimer chainが完了しました");
})();
```

非同期 API は環境がバックグラウンドで並列的に処理してくれていますが、裏でどのように処理されているかは分かりません。`fetch()` に限ったことではなく非同期 API を並列化した際にはある程度不定性がでてくるため、行いたい順番(実行と完了の順番)が重要なら await 式で制御する必要があります。

```js
(async () => {
  await fetch(urls[0])
    .then(response => response.text())
    .then(text => console.log(text)); 
  console.log("fetch chainの１つ目が完了しました");

  await fetch(urls[1])
    .then(response => response.text())
    .then(text => console.log(text));
  console.log("fetch chainの2つ目が完了しました");

  await fetch(urls[2])
    .then(response => response.text())
    .then(text => console.log(text));
  console.log("すべてのfetch chainが完了しました");  
})();
```

少し復習として Promise chain と async/await の変形を考えてみましょう。上記コードの例では await Promise chain の形になっていますが、もちろん Promise chain だけででもそのような順序付けができます。実際に `randomTimer()` のコードを完全な Promise chain にすることで順序付けてみます。

```js:Promise chain
randomTimer("1st", "[A-1]")
  .then(val => console.log("👦 [A-2]", val))
  .then(() => console.log("👦 [A-3]"))
  .then(() => {
    console.log("randomTimer chainの１つ目が完了しました");
    return randomTimer("2nd", "[A-4]")
      .then(val => console.log("👦 [A-5]", val))
      .then(() => console.log("👦 [A-6]"))
      .then(() => {
        console.log("randomTimer chainの２つ目が完了しました")
        return randomTimer("3rd", "[A-7]")
          .then(val => console.log("👦 [A-8]", val))
          .then(() => console.log("👦 [A-9]"))
          .then(() => {
            console.log("すべてのrandomTimer chainが完了しました");
          });
      });
  });
```

ネストが入っていて見づらいので、『[Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)』のチャプターで見た通り、なるべくネストさせないようにフラットに変形すると次のようになります。

```js:ネストをフラットにしたchain
randomTimer("1st", "[A-1]")
  .then(val => console.log("👦 [A-2]", val))
  .then(() => console.log("👦 [A-3]"))
  .then(() => {
    console.log("randomTimer chainの１つ目が完了しました");
    return randomTimer("2nd", "[A-4]")
  });
  .then(val => console.log("👦 [A-5]", val))
  .then(() => console.log("👦 [A-6]"))
  .then(() => {
    console.log("randomTimer chainの２つ目が完了しました");
    return randomTimer("3rd", "[A-7]")
  })
  .then(val => console.log("👦 [A-8]", val))
  .then(() => console.log("👦 [A-9]"))
  .then(() => {
    console.log("すべてのrandomTimer chainが完了しました");
  })
```

基本的に async/await と Promise chain の間で自由に変形できるようになれば async/await で書いた方が見やすいですし書きやすいです。

```js:async/await
(async () => {
  // この範囲内での実行と完了の順番を担保する
  const val1 = await randomTimer("1st", "[A-1]");
  console.log("👦 [A-2]", val1);
  console.log("👦 [A-3]");
  console.log("randomTimer chainの１つ目が完了しました");

  const val2 = await randomTimer("2nd", "[A-4]")
  console.log("👦 [A-5]", val2);
  console.log("👦 [A-6]");
  console.log("randomTimer chainの２つ目が完了しました");

  const val3 = await randomTimer("3rd", "[A-7]")
  console.log("👦 [A-8]", val3);
  console.log("👦 [A-9]");
  console.log("すべてのrandomTimer chainが完了しました");
})();
```

もしも順序に意味がないなら `Promise.all()` などでまとめ上げて並列化させます。これで時間効率がよくなり、すばやく完了します。

```js
(async () => {
  const p1 = randomTimer("1st", "[A-1]")
    .then(val => console.log("👦 [A-2]", val))
    .then(() => console.log("👦 [A-3]"));
  const p2 = randomTimer("2nd", "[B-1]")
    .then(val => console.log("👦 [B-2]", val))
    .then(() => console.log("👦 [B-3]"));
  const p3 = randomTimer("3rd", "[C-1]")
    .then(val => console.log("👦 [C-2]", val))
    .then(() => console.log("👦 [C-3]"));

  await Promise.all([p1, p2, p3]);
  console.log("すべてのrandomTimer chainが完了しました");
})();
```

`randomTimer()` の部分だけ並列化したいなら分離して await します。

```js
(async () => {
  const [v1, v2, v3] = await Promise.all([
    randomTimer("1st", "[A]"),
    randomTimer("2nd", "[B]"),
    randomTimer("3rd", "[C]"),
  ]);
  console.log("すべてのrandomTimerが完了しました");

  console.log("👦 [4]", v1);
  console.log("👦 [5]");
  console.log("👦 [6]", v2);
  console.log("👦 [7]");
  console.log("👦 [8]", v3);
  console.log("👦 [9]");
})();
```

このように意図に応じて色々な変形や書き方がありえます。

ただし、次のように async 関数である `randomTimer()` を await せずに放っておくようなコードを書くと良くないことが起きます。これについては後で解説します。

```js
(async () => {
  // await 式による評価をしないと...
  randomTimer("1st", "[A]");
  randomTimer("2nd", "[B]");
  randomTimer("3rd", "[C]");
  console.log("すべてのrandomTimerが完了しました????");
})();
```

# レイヤーでの制御

変形をもどして await Promise chain の形をもう一度みてみましょう。この形から学べることがいくつかあります。

```js
(async () => {
  await fetch(urls[0])
    .then(response => response.text())
    .then(text => console.log(text)); 
  console.log("fetch chainの１つ目が完了しました");

  await fetch(urls[1])
    .then(response => response.text())
    .then(text => console.log(text));
  console.log("fetch chainの2つ目が完了しました");

  await fetch(urls[2])
    .then(response => response.text())
    .then(text => console.log(text));
  console.log("すべてのfetch chainが完了しました");  
})();
```

やっかないこととして、上記の await で制御しているのは Promise chain という単位で順序付けていることです。Promise chain そのものは内部で「`fetch()` してから `response.text()` でテキスト抽出して、最終的にコンソール出力する」という順序付けがすでになされています。async 関数内ではその chain を単位にしてさらに await で順序付けを行っています。

つまり、小さいスケールでの順序付け(あるいは並列化)と一段回大きなスケールでの順序付け(あるいは並列化)がされていることに気づく必要があります。関心のある領域(レイヤー)内での実行と完了の順番を保証するために await や chain での制御が次のように多重のレイヤーで行われていることがあり得ます。

```js
// 以下のコードそのものは特に意味がないものなので注意してください
async function bigStep() {
  // このレイヤーでは順序付けて順番に行う
  await middleStep();
  await middleStep();
  console.log("すべてのmiddleStepが完了しました");
}

async function middleStep() {
  // このレイヤーでは並列化する (chain 内部での順序付けが更にある)
  const p1 = smallStep().then().then();
  const p2 = smallStep().then().then();
  const p3 = smallStep().then().then();
  await Promise.all([p1, p2, p3]);
  console.log("すべてのsmallStepのchainが完了しました");
}

async function smallStep() {
  // 非同期 API を起点にした chain の並列化(本質的には非同期 API の並列化)
  const p1 = fetch(urls[0]).then(response => response.text()).then(text => console.log(text)); 
  const p2 = Deno.writeTextFile(paths[0], inputData).then(() => console.log("書き込み完了しました"));
  const p3 = fetch(urls[1]).then(response => response.text()).then(text => console.log(text)); 
  const p4 = Deno.writeTextFile(paths[1], inputData).then(() => console.log("書き込み完了しました"));
  
  return await Promise.allSettled([p1, p2, p3, p4]);
}

export default bigStep;
```

ライブラリなどで提供される async 関数は内側でネットワーキングを行う Web API や Node や Deno でのパッケージならそれぞれの Runtime API を使用していますが、ユーザーが直接的に触る async 関数は export されて別の場所で使えるようなったもので、つまりレイヤーのトップとなったものです。内部的に環境の機能である非同期 API が関与した様々な処理がまとめられて抽象化されており、利用するための１つの窓口となった async 関数を使用して、それを使った後に何か関連する作業をしたい場合に await で制御します。

```js
import bigStep from "bigStep";
// レイヤーのトップにあるものを利用する

(async () => {
  await bigStep(); // 内包するすべての処理が完了してから次になにかしたいから await する
  console.log("bigStepが完了しました"); // これがしたいから await した
})();
```

上のコードはかなり極端に書いてレイヤー化したものですが、await や chain で順序付けたり並列化して制御しているのはこのようなレイヤーを持ったスケール感のあるものだということに注意してください。内部的にはレイヤーのスケールごとに並列化されていたり、順序付けされていたりします。

この時点において、モジュールが関与してくると内部的に隠蔽された実行順番がでてくるため、すべての細かい実行順番を考えるのはかなり困難ですし、そこまで意味のある行為ではなくなってきます。できるのは特定の関心範囲での実行と完了の順番を保証させてまとまりのある単位で制御の流れを作ることだけです。

# 投げっぱなしの処理

上でみた `bigStep()` のような async 関数で別の async 関数である `middleStep()` を await しな、つまり await を内部で行わない async 関数だと何が起きるでしょうか。

```diff js
async function bigStep() {
-  await middleStep();
+  middleStep();
-  await middleStep();
+  middleStep();
}
```

上のようなレイヤー化された async 関数を具体的なコードで考えてみます。次のように `noAwait.js` という名前のファイルから `noAwait()` という async 関数がトップレベルとなるものとして `export default` でエクスポートします。

```js:noAwait.js
export default async function noAwait() {
  asyncFn();
  asyncFn();
  console.log("🤪 ２つのasyncFnの処理は完了しました?");
}

async function asyncFn() {
  await Promise.all([
    rTimer().then((t) => console.log(`${t}[ms]経過`)),
    rTimer().then((t) => console.log(`${t}[ms]経過`)),
  ]);
  console.log("🦄 asyncFnの処理を終了します");
}

function rTimer() {
  const rTime = Math.floor(1000 * Math.random());
  return new Promise((resolve) =>
    setTimeout(() => {
      console.log("⏰ rTimerの処理を終了します");
      resolve(rTime);
    }, rTime)
  );
}
```

`noAwait.js` からレイヤーのトップである `noAawit()` 関数を import して利用してみたいと思います。

```js:useNoAwait.js
import noAwait from "./noAwait.js";

(async () => {
  await noAwait();
  console.log("😅 noAwaitの処理は完了しました?");
})();
```

このコードで意図したいのは次のようなことだと想定してください。

>「ランダムな時間で完了するタイマーの処理２つを並列に起動する `asyncFn()` という処理を `noAwait()` 関数内で２つ順番に実行して(注: １つ目が終わってから２つ目を起動させる)それらがすべて完了してからコンソールに完了メッセージを出力して、最終的に `noAwait()` 自体が完了したことを通知するメッセージをコンソールに出力する。そして、それを `useNoAwait.js` を実行することで実現する。」

さて、`useNoAwait.js` ファイルを `deno run` で実行してみるとどのような出力が得られるでしょうか。ちょっと考えてみてください。

:::details 答え
実行結果は次のようになります。

```sh
❯ deno run useNoAwait.js
🤪 ２つのasyncFnの処理は完了しました?
😅 noAwaitの処理は完了しました?
⏰ rTimerの処理を終了します
184[ms]経過
⏰ rTimerの処理を終了します
310[ms]経過
⏰ rTimerの処理を終了します
561[ms]経過
🦄 asyncFnの処理を終了します
⏰ rTimerの処理を終了します
717[ms]経過
🦄 asyncFnの処理を終了します
```
:::

このコードは今までのように意図した通りにはなりません。意図通りのコードであれば次のような出力が得られるはずですがそうはなりませんでした。

```sh
# 意図通りであればこうなるはずだがならない😅...
❯ deno run useNoAwait.js
⏰ rTimerの処理を終了します
184[ms]経過
⏰ rTimerの処理を終了します
310[ms]経過
⏰ rTimerの処理を終了します
561[ms]経過
🦄 asyncFnの処理を終了します
⏰ rTimerの処理を終了します
717[ms]経過
🦄 asyncFnの処理を終了します
🤪 ２つのasyncFnの処理は完了しました?
😅 noAwaitの処理は完了しました?
```

意図通りにするにはどこを修正すべきでしょうか。順序付けがされていないところがあります。

とは言っても、話の流れ的に `noAwait()` 関数に await 式を配置することで意図通りになることが簡単に分かると思います。

```diff js
export default async function noAwait() {
- asyncFn();
+ await asyncFn();
- asyncFn();
+ await asyncFn();
  console.log("🤪 ２つのasyncFnの処理は完了しました?");
}
```

１つ目の `asyncFn()` が完了してから次の `asyncFn()` を行うようにして、更にそれが完了してから `console.log()` でメッセージを出力するようにします。これで意図通りの出力が得られます。

ここで考えるべきは await 式の適切な配置で意図通りになったことではなく、「**await しなかったことで起きた実行順番への影響について**」です。まず、肝心の `noAwait()` 関数内部での実行順番が意図通りのものとなっていません。

```js
export default async function noAwait() {
  // awiat していないからこのレイヤーで並列化している
  asyncFn();
  asyncFn();
  // await していないので asyncFn が終わっていなのにコンソール出力してしまっている
  console.log("🤪 ２つのasyncFnの処理は完了しました?");
}
```

そもそもレイヤーのトップである `noAwait()` はエクスポートされて他のファイル `useNoAwait.js` で利用されていました。`deno run` で実行するのもこのファイルでした。

```js:useNoAwait.js
import noAwait from "./noAwait.js";

(async () => {
  await noAwait();
  console.log("😅 noAwaitの処理は完了しました?");
})();
```

意図としては「`noAwait()` が完了した後でコンソールに完了メッセージを通知する」ということでしたが実際の結果は次のようになっていました。

```sh
❯ deno run useNoAwait.js
🤪 ２つのasyncFnの処理は完了しました?  # <-- 意図していない?
😅 noAwaitの処理は完了しました? # <--- 意図していない?
⏰ rTimerの処理を終了します
184[ms]経過
⏰ rTimerの処理を終了します
310[ms]経過
⏰ rTimerの処理を終了します
561[ms]経過
🦄 asyncFnの処理を終了します
⏰ rTimerの処理を終了します
717[ms]経過
🦄 asyncFnの処理を終了します
```

`😅 noAwaitの処理は完了しました?` の出力は意図したものでしょうか?

混乱しやすいですが、この部分は実は意図通りになっています。意図していた通りに `noAwait()` が完了してからコンソールにメッセージが出力されています。つまりこの時点で **`noAwait()` 自体は完了しています**。つまり、`noAwait()` を使っているレイヤーのコードには問題はありません。

```js:useNoAwait.js(このコードは問題ない)
import noAwait from "./noAwait.js";

(async () => {
  await noAwait(); 
  // noAwait の完了してから次の処理が実行される
  console.log("😅 noAwaitの処理は完了しました?");
})();
```

問題なのは `noAwait()` 関数そのものです。

```js:問題のある箇所
export default async function noAwait() {
  asyncFn();
  asyncFn();
  // await していないので asyncFn が終わっていなのにコンソール出力してしまっている
  console.log("🤪 ２つのasyncFnの処理は完了しました?");
  // コンソール出力が終わった時点でこの関数は処理完了となる
}
```

これを理解するために、『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)』のチャプターで見た何もしない async 関数をもう一度考えてみましょう。

```js:何もしない非同期関数
async function empty() {}
```

この async/await は V8 エンジンは次のように内部的に変換すると想定されました。

```js:V8_Converting
resumable function empty() {
  implicit_promise = createPromise();
  // await 式が無いのでこの関数自体の処理は中断しない
  resolvePromise(implicit_promise, undefined); 
}
```

await 式が無いため async 関数は中断せず、さらに `return` をしていないため、この関数から返される Promise インスタンスは直ちに `undefined` で履行状態となり、マイクロタスクは発生しませんでした。

当該の `noAwait()` はどのように変換されるか考えてみると次のようになるはずです。await 式が無いためこの async 関数自体は中断しません。

```js:V8_Converting
resumable function noAwait() {
  implicit_promise = createPromise();

  asyncFn();
  asyncFn();
  console.log("🤪 ２つのasyncFnの処理は完了しました?");
  // await 式が無いのでこの関数自体の処理は中断しない

  resolvePromise(implicit_promise, undefined); 
}
```

async 関数内の処理は上から下に１つずつ実行されていきます。そして、１つ目の `asyncFn()` は await 式を含む async 関数なので、途中でマイクロタスクが発行されて一時中断するはずです。await 式で async 関数が一時中断されたときにはその async 関数の外側に制御を戻してメインスレッドで別の処理を継続するという話でしたが、この場合は呼び出し元である `noAwait()` の続きへと制御が戻ってきます。ということで次に２つ目の `asyncFn()` に出会うので実行しますが、１つ目と同じく内部の await 式で中断して呼び出し元の `noAwait()` の続きへと制御を戻します。そして、次の行の `console.log()` を実行します。

そして、`return` をしていないので V8 エンジンで内部変換されたコードにおいて `undefined` で直ちに履行する Promise インスタンスを `noAwait` の呼び出し元に返します。つまり、この時点で `noAwait()` 関数の処理は終了します。

つまり、`noAwait()` では２つの `asyncFn()` を起動しただけで後は知りません、という状態になっています。この２つの async 関数の完了はどこになるか感知しません。そしてユーザー側も分かりません。イベントループでマイクロタスクが連鎖的に処理されるのでいつかは処理が完了するはずですが、予測して次の処理を作り出すのはかなり困難でしょう。

`noAwait()` を提供しているのがライブラリだったとすると、ライブラリのコードが壊れているのでユーザーはそのソースを改変しない限り意図通りのコードを書くことができない状態になっています。使う側がちゃんと await していようが内部レイヤーのどこかで壊れていれば適切な順番や制御の流れを作ることができないので、`sleep()` で待機時間を挿入して待つようなことになります。

問題なのは**範囲内での async 関数や非同期 API が投げっぱなしになっている**ことです。これらの処理が起動している呼び出し元の範囲で完了が担保されていないと、さらに上のレイヤーで利用する際に適切に await しても実行順番が意図通りにならないというバグを引き起こします。

```js
export default async function noAwait() {
  // 投げっぱなしの async 関数や非同期 API があると
  // 上のレイヤーで await しても実行順番が保証されない
  asyncFn(); // この範囲内で完了が担保されていない
  asyncFn(); // この範囲内で完了が担保されていない
  console.log("🤪 ２つのasyncFnの処理は完了しました?");
}
```

「投げっぱなしにしておきたい」という意図が無い限り、async 関数や非同期 API はそれらを利用する async 関数内で完了を担保しておかないと、上のレイヤーで使う時に困ることになります。つまり、「起動する」ということのみが行われて、完了した後に何かしたい場合に上のレイヤーでそれを行うのは非常に難しくなってしまいます(これを回避するために `sleep()` を噛ませて一時的に時間を置くなどする必要がでてくる)。

これは『[コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)』のチャプターで見たように `return` しない非同期処理のケースと同じです。`await` 式で Promise 処理を評価しないのは `then()` のコールバックで Promise 処理を `return` しないことによって「副作用」になってしまうのと同じです。

:::message
そもそも `console.log()` 自体が副作用なのですが、何が起きるのか分かりやすくするために利用しています。
:::

順序付けて行う場合でも並列化する場合でも、利用する async 関数や非同期 API に対して await を配置することは、その async 関数内での完了そのものを担保することになります。

```js
export default async function noAwait() {
  // 順序付ける場合
  await asyncFn(); // この範囲内で完了が担保されている
  await asyncFn(); // この範囲内で完了が担保されている
  console.log("😎 この範囲内で利用する非同期関数の完了が担保");
}
```

```js
export default async function noAwait() {
  // 並列化する場合
  await Promise.all([
    asyncFn(),
    asyncFn(),
  ]);
  // この範囲内で完了が担保されている
  console.log("😎 この範囲内で利用する非同期関数の完了が担保");
}
```

そういう訳で適切に await 式を配置することは効率化に関わることだけでなく、制御の流れを作り出すために「**特定の範囲内で完了を担保させる**」という側面があります。そして「async 関数や非同期 API の投げっぱなしは良くないよ」という話でした。

Top-level await が使えるのに async の即時実行などを解説に多用しているのは async 関数という特定範囲について意識できるように限定して考えたいからです。

ちなみに Deno ではリントルールが存在しており、vs code などで拡張機能を有効化しておくとリンターに怒ってもらえます。実際、上で見た `noAwait()` 関数では何らかの処理があるのに await 式を１つも配置していないため "**require-await**" というリントルールによって注意されます。

```js
// async 関数内に処理があるのに await 式が１つも無いと怒られる
async function noAwait() {
  asyncFunc1(); 
  asyncFunc2();
  asyncFunc3();
}
```

>Disallows async functions that have no await expression
>In general, the primary reason to use async functions is to use await expressions inside. If an async function has no await expression, it is most likely an unintentional mistake.
>([require-await | deno_lint docs](https://lint.deno.land/?q=require-await#require-await) より引用)

このリントルールに怒られないようにするには１つでも await 式があればよいのですが、今までの話を踏まえると確実にこの範囲内で起動するすべての非同期処理の完了を担保させておくのが良いでしょう。

```js
async function noAwait() {
  // ２つを並列化
  await Promise.allSettled([ // この範囲内で完了が担保される
    asyncFunc1(),
    asyncFunc2(),
  ]);
  // 上が終わってから実行
  await asyncFunc3(); // この範囲内で完了が担保される
  // すべての処理の完了がこの範囲内で担保される 
}
```

# 競争時の副作用と AbortController

ただし、複数の Promise 処理を `Promise.race()` によって競争(race)させる場合には上で解説した async 関数内の領域(レイヤー)に処理順番を収めることができません。

`Promise.race()` は `Promise.all()` などと同じ様に Promise インスタンスの配列を受け取り、**その中で最も早く Settled になった Promise の状態に連鎖して同一の状態になります**。非同期 API や async 関数を並列起動させて(履行か拒否か完了なく)一番最初に完了した Promise インスタンスに対して次の処理を呼び出すというような処理に使えます。

一番最初に完了した Promise インスタンスが履行状態なら `Promise.race()` から返る Promise インスタンスも履行状態となり、最初に完了した Promise インスタンスが拒否状態なら `Promise.race()` から返る Promise インスタンスも拒否状態になります。

具体的にどうなるか、再び Promisification したタイマーに登場してもらって考えてみましょう。次の実行結果はどうなるでしょうか。

```js
// raceTime.js
function pTimer(time, order) {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log(`👻 ${order}: ${time}[ms]でタイムアウト`);
      resolve(`${time}[ms]のタイマー`);
    }, time);
  });
}

(async () => {
  // await 式で評価して履行値を取り出す
  const raceVal = await Promise.race([
    pTimer(2000, "[A]"),
    pTimer(1000, "[B]"),
    pTimer(3000, "[C]"),
  ]);
  console.log("最初に完了したのは", raceVal);
})();
```

これを `deno run` で実行すると次の出力が得られます。

```sh
❯ deno run raceTime.js
👻 [B]: 1000[ms]でタイムアウト
最初に完了したのは 1000[ms]のタイマー
👻 [A]: 2000[ms]でタイムアウト
👻 [C]: 3000[ms]でタイムアウト
```

`Promise.all()` でやるとどうなるかもう一度みてみます。

```js
// raceNoTime.js
(async () => {
  const vals = await Promise.all([
    pTimer(2000, "[A]"),
    pTimer(1000, "[B]"),
    pTimer(3000, "[C]"),
  ]);
  console.log("すべての処理が完了しました", vals);
})();
```

次のようにすべての処理が完了してからコンソールに完了のメッセージを出力できています。

```sh
❯ deno run raceNoTime.js
👻 [B]: 1000[ms]でタイムアウト
👻 [A]: 2000[ms]でタイムアウト
👻 [C]: 3000[ms]でタイムアウト
すべての処理が完了しました [ "2000[ms]のタイマー", "1000[ms]のタイマー", "3000[ms]のタイマー" ]
```

`Promise.all()` でまとめた Promise 処理はそれを実行する async 関数の範囲に収めてその範囲での完了が担保されていましたが、ここで問題になるのは、`Promise.race()` だと await しても async 関数の範囲での完了あるいは処理そのものの停止が担保されていないことです。

`Promise.race()` を使う目的は複数の Promise 処理を競争させて最初に完了したものの後に何かするということです。より分かりやすく考えると、例えば `fetch()` メソッドなら複数のリソースから同時にデータフェッチさせて一番最初に取得できたデータだけを表示させるなどがやりたい時に利用します。しかし、一番最初に完了したもの以外の `fetch()` はどうなるでしょうか。上の `raceTime.js` のコードで見たように「投げっぱなし」となり、利用する async 関数自体が完了した後になってから投げっぱなしになった処理が完了します。

投げっぱなしになっている処理は気持ち悪いので `Promise.reace()` が完了したら全部キャンセルすることを考えます。例えば、複数のリソースからデータフェッチをして一番最初に完了したものしか使わない場合は環境がバックグラウンドで行っている他のすべての `fetch()` を無駄なので止めたいとか、async 関数の外側で投げっぱなしになった処理によってあとから副作用のようになって制御できないような場合を防ぐために一番最初に完了したもの以外を停止させます。

一旦 Promise 処理から離れると `setTimeout()` API の停止には対応する `clearTimeout()` API というものを利用することで実現できます。`setTimeout()` からは戻りとしてタイマーの ID である数値が返ってくるのでそれを `clearTimeout()` に入力として渡すことでタイマーをキャンセルできます。

```js
const timerId = setTimeout(() => console.log("🥺 これは出力されないよ"), 1000);
// すぐにタイマーをキャンセルするのでコールバックが実行されない
clearTimeout(timerId); 
```

`clearTimeout()` も環境の提供する API 機能なので、これを起動することで環境が管理しているタイマーに対して「止めなさい」とキャンセルさせるように司令できます。Web API(Web Platform API)である `setTimeout()` によって並列的にタイマーが図られているわけですが、タスクキューへコールバックを送る前にそのタイマーを破棄させることでタスクキューへと送るのをこれでやめさせます。

というのが基本的なタイマーのキャンセルですが、素の状態で使うのは色々と面倒なので別のキャンセル用の機構と組み合わされたものを利用します。

『[古い非同期 API を Promise でラップする](12-epasync-wrapping-macrotask)』のチャプターで見ましたが Deno では Promisifiaction されたタイマーである `delay()` 関数が標準モジュールとして提供されています。

https://doc.deno.land/https://deno.land/std@0.145.0/async/mod.ts/~/delay

```ts:型定義
function delay(ms: number, options?: DelayOptions): Promise<void>;
```

このタイマーを使う際には AbortController と AbortSignal という API が利用できます。これらは中止系の処理を統一化した API であり、これらの API を介して中止イベントを発火させて、その通知を受け取った処理を停止させます。

https://developer.mozilla.org/ja/docs/Web/API/AbortController

https://developer.mozilla.org/ja/docs/Web/API/AbortSignal

ちなみに AbortController と AbortSignal はあらゆる環境でサポートされた API です。

![abortControllerのサポート](/images/js-async/img_abortController_support.jpg)*[AbortController - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/AbortController)より引用*

AbortController と AbortSignal の使い方については Masaki Hara さんの次の記事で分かりやすく解説されています。

https://zenn.dev/qnighy/articles/772f632af595aa

基本的な使い方としては、`AbortController()` コンストラクタを使用して `new` で AbortController インスタンスを作成し、中止(abort)のイベントを受信する signal オブジェクトを作ります。

```js
const controller = new AbortController();
const signal = controller.signal;
// const { signal } = controller; // 分割代入の場合
```

`fetch()` なら第二引数に `{ signal: signal }` というオブジェクトを渡すことで `fetch()` を停止させる準備ができます。`fetch()` はこの `signal` を介して `abort` のイベントを受信します。

```js
fetch(url, { signal })
// { signal } はオブジェクトの略記プロパティ名という記法
// { signal: signal } を表す
```

第二引数は `fetch()` のリクエストに適用したいカスタム設定を含むオブジェクトを渡すことができます。`AbortSignal` オブジェクトのインスタンスである `signal` もこのオブジェクトに含めることができる可能オプションです。

`abort` イベントを発火して、`signal` を介して受信することで `fetch()` 操作を停止させます。

```js
controller.abort(); // abort イベントの発火
```

それでは実際に `setTimeout()` で 500 ミリ秒後に `conroller.abort()` を起動することで `fetch()` 処理の中止を通知させるコードを考えてみると次のようになります。

```js
const url = "https://api.github.com/zen";

const controller = new AbortController();
const signal = controller.signal;

console.log("fetchを開始します");
fetch(url, { signal })
  .then((res) => res.text())
  .then((text) => console.log(text))
  .catch((e) => {
    console.error(e);
    if (signal.aborted) { // abortされたか確認できるフラグ
      console.log("時間切れなのでフェッチをキャンセルしました");
    }
  });

setTimeout(() => {
  controller.abort();
}, 500);
```

500 ミリ秒経過後に `fetch()` によるデータフェッチが完了していなければ、`fetch()` の第二引数に渡した `signal` を介して停止命令が受信されて処理がキャンセルされるます。その際には非同期例外として DOMException が throw されるので、chain の `catch()` メソッドで補足できます。

これを実行すると自分の通信環境ではある程度の確率で停止できます。

```sh
❯ deno run -A fetchAbort.js
fetchを開始します
時間切れなのでフェッチをキャンセルしました
DOMException: The signal has been aborted
```

`signal.aborted` ですでに `controller.abort()` によって `abort` イベントが発火されたかどうかを確認できます。また、`signal.addEventListener("abort", eventhandler)` で `abort` イベントをリスンしてハンドルできます。

基本的な使い方がわかったところで、Deno の std である `delay()` について考えます。`delay()` 関数の TypeScript による実装を見てもらうと分かりますが、この AbortController API と AbortSignal API の機構を `setTimeout()` API と `clearTimeout()` API を組み合わせることで**キャンセル可能な Promise-based なタイマー**を実現しています。

https://github.com/denoland/deno_std/blob/0.145.0/async/delay.ts

すでに `abort` イベントが発火されていれば拒否状態の Promise を返し、そうでなければ基本的なタイマー処理を開始します。`singal?.abortEventListener("abort", arbot, { once: true })` によって `abort` イベントをリスンして、通知を受ければアロー関数で定義された `abort` 関数を実行し `clearTimeout()` でタイマーのキャンセルを行います。`?.` 記号は JavaScript の[オプショナルチェーンチェーンという演算子](https://jsprimer.net/basic/object/#optional-chaining-operator)で、プロパティのアクセス元が `null` や `undefined` のときもエラーを発生させずにすみます。

これを自分で実装すると中々手間がかかりますが、すでに std として提供されているのでこのタイマーを使わせてもらいます。型定義を見ると第二引数に `fetch()` と同じ様に `{ signal }` を渡すことができるようになっています。

```ts:delayの型定義
function delay(ms: number, options?: DelayOptions): Promise<void>;
```

```ts:DelayOptoinsの型定義
interface DelayOptions {
  signal?: AbortSignal;
}
```

std を使うには URL から名前付きで import できます。

```js
import { delay } from "https://deno.land/std@0.145.0/async/mod.ts";

(async () => {
  await delay(1000);
  console.log("1000ms経過しました");
})();
```

それでは話を戻して、実際に `await Promise.race(promises)` で複数の遅延時間のタイマーを競争させて最初に完了したもの以外を直ちに停止させてみます。`delay()` を更に async 関数でラップして `dTimer()` 関数というレイヤーを作っておきます。

```js:dtime.js
import { delay } from "https://deno.land/std@0.145.0/async/mod.ts";

const controller = new AbortController();
const signal = controller.signal;
const rTimes = [200, 100, 300];
// タイマーの遅延時間[ms]を収めた配列(どれが早く終わるか競争させる)

(async () => {
  // map メソッドでタイマーの並列化(停止できるようにすべてのタイマーに signal を渡しておく)
  const promises = rTimes.map((time) =>
    dTimer(`${time}[ms]のタイマー`, time, { signal })
  );
  // 競争させて最初に完了する Promise インスタンスの値を取り出す
  const winner = await Promise.race(promises);
  controller.abort(); // すべてのタイマーを停止させる
  console.log("raceの結果:", winner);
  console.log("タイマーの競争が終了しました");
})();

// delay をラップして新しいレイヤーを作る
async function dTimer(msg, time, option = {}) {
  await delay(time, option);
  console.log(`${time}[ms]が経過しました`);
  return msg;
}
```

`rTimes` 配列に格納した遅延時間のどれが早く終るか競争させてみると、次の実行結果が得られます。

```sh
❯ deno run dtime.js
100[ms]が経過しました
raceの結果: 100[ms]のタイマー
タイマーの競争が終了しました
```

当たり前ですが一番短い 100 ミリ秒のタイマーが最初に完了するため、このタイマーが完了次第コンソールに完了の結果が出力されます。そして、`const winner = await Promise.race(promises);` の直後に `controller.abort();` ですべてのタイマーをキャンセルするように指示しているので他のタイマーも含めてすべてのタイマーが `abort` のイベントを受け取り内部的に `clearTimeout()` が呼ばれてタイマー処理がキャンセルされます。

`reject(new DOMException("Delay was aborted.", "AbortError"))` によって `delay()` から返る Promise インスタンスは拒否状態となります。`dTimer` 内部で例外の補足などを行っていないため次のように try-catch で async 関数内で例外補足できます。

```js
async function dTimer(msg, time, option = {}) {
  try {
    await delay(time, option);
    console.log(`${time}[ms]が経過しました`);
    return msg;
  } catch (err) {
    console.log("タイマーはキャンセルされました" ,err)
  }
}
```

この状態で実行すると次の出力を得ます。

```sh
❯ deno run dtime.js
100[ms]が経過しました
raceの結果: 100[ms]のタイマー
タイマーの競争が終了しました
タイマーはキャンセルされました DOMException: Delay was aborted.
タイマーはキャンセルされました DOMException: Delay was aborted.
```

競争終了前にキャンセルのメッセージを通知させたければ、`console.log("タイマーの競争が終了しました");` の前に `dTimer()` から返ってくる Promise インスタンスが Settled となるのを await 式で評価して待ちます。

```js
(async () => {
  const promises = rTimes.map((time) =>
    dTimer(`${time}[ms]のタイマー`, time, { signal })
  );
  const winner = await Promise.race(promises);
  controller.abort(); // すべてのタイマーを停止させる
  console.log("raceの結果:", winner);
  await Promise.allSettled(promises);
  // ここで再度 await してすべての dTimer の完了を待ってから出力
  console.log("タイマーの競争が終了しました");
})();

async function dTimer(msg, time, option = {}) {
  try {
    await delay(time, option);
    console.log(`${time}[ms]が経過しました`);
    return msg;
  } catch (err) {
    console.log("タイマーはキャンセルされました" ,err)
  }
}
```

再度 await することでキャンセルされた `dTimer()` の処理の完了を待ってから終了通知のメッセージを出力できます。

```sh
❯ deno run dtime.js
100[ms]が経過しました
raceの結果: 100[ms]のタイマー
タイマーはキャンセルされました DOMException: Delay was aborted.
タイマーはキャンセルされました DOMException: Delay was aborted.
タイマーの競争が終了しました
```

`dTimer()` の第三引数はデフォルト引数によって省略可能にしてあるので `{ signal }` を渡さなくても機能します。これを渡さないとどうなるでしょうか。

```diff js
(async () => {
  const promises = rTimes.map((time) =>
-   dTimer(`${time}[ms]のタイマー`, time, { signal })
+   dTimer(`${time}[ms]のタイマー`, time)
  );
```

race させていた他のタイマーの「投げっぱなしの処理」が async 関数の処理が終わってから完了することになるので、次のような実行結果になります。

```sh
❯ deno run dTime.js
100[ms]が経過しました
raceの結果: 100[ms]のタイマー
タイマーの競争が終了しました
200[ms]が経過しました
300[ms]が経過しました
```

「投げっぱなしの処理」があると、このように「副作用」的に意図した結果とならなくなりますが、AbortController API と AbortSignal API によって race させている他の処理をキャンセルすることで統制できます。

ちなみに、これは `Promise.any()` による競争でも同じ話が言えます。

