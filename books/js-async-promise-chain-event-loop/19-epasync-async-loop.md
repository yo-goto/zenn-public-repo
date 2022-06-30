---
title: "反復処理の制御"
aliases: [ch_反復処理の制御]
---

# このチャプターについて

このチャプターでは、非同期処理の反復処理、あるいは Promise が絡む反復処理について、await 式の配置と絡めてどのように反復処理をしていくかを考えて行きましょう。

このチャプターでは主に JSON Placeholder という Fake API サービスを利用して複数のリソースからデータフェッチなどをすることを考えます。

https://jsonplaceholder.typicode.com

以下のように API のエンドポイントの URL を配列内に格納しておき、これを反復処理します。

```js
const urls = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];
```

# (1) 順番に興味がないので並列化して効率化
この場合、各 URL からリソースを取得する順番に意味は特になければ、`Promise.all()` などでまとめて上げて内部的に非同期 API を並列化させて効率化させてもよいでしょう。データフェッチ以外のケースでも、非同期の複数タスクの間に依存関係や順序関係が無いならこのようなやり方で行います。

この場合は配列の `map()` メソッドなどを使って Promise の配列を作り、`await Promise.all()` ですべての完了をまってから次に何かを行うようにします。各 async 関数は Promise の配列を作る過程で並列的に起動させます。実際にはもちろん１つずつ起動させますが、内部的に利用される非同期 API が起動後に環境によってバックグラウンドで時間的に継続処理されるので実質的に「並列化」されます。

```js
(async () => {
  const responses = await Promise.all(urls.map((url) => fetch(url)));
  const jsons = await Promise.all(responses.map((response) => response.json()));
  jsons.forEach((json) => console.log(json));
  console.log("すべての非同期処理が完了しました");
})();
```

```js
(async () => {
  const promises = urls.map(url => fetch(url).then(response => response.json().then(json => console.log(json))));
  await Promise.all(proimses);
  console.log("すべての非同期処理が完了しました");
})();
```

どの程度の粒度の操作で await するかを考える必要がありまが、通常は共通の操作を asycn 関数にまとめて try-catch に閉じ込めるなどを行うのが一般的ではないでしょうか。async 関数内の処理は独立させてそのレイヤーでの実行と完了の順番が担保されるようにします。

```js
async function fetchThenConsole(url) {
  try {
    const response = await fetch(url);
    const text = await response.text();
    console.log(text);
  } catch(err) {
    console.log(err);
  }
}
(async () => {
  const promises = urls.map(url => fetchThenConsole(url));
  await Promise.all(promises);
  console.log("すべての非同期処理が完了しました");
})();
```

配列の `map()` ではなく `forEach()` を使おうとするのは一般的にアンチパターンとなります。`map()` メソッド内部で `return` した async 関数や非同期 API の完了を把握するための返ってくる Promise インスタンスが `forEach()` の場合だと返り値自体無いので取得しずらくなります。さらに `forEach()` のコールバックで await 式を使おうとするのは Syntax Error になりますし、それを回避してコールバックそのものを async 関数下しようとすると意図しない挙動になります。

とはいえ、先に Promise を入れる配列を宣言しておいて、`push()` メソッドで async 関数や非同期 API を chain して起動した時に返ってくる Promise インスタンスを格納しておくことで await 式で評価できるようになります。

```js
(async () => {
  const promises = [];
  urls.forEach(url => {
    // 起動して返ってくる Promise インスタンスを外にある配列に格納
    promises.push(fetch(url).then(response => response.json()).catch(e => console.error(e)));
  });
  const jsons = await Promise.all(promises);
  console.log(jsons);
})();
```

一般的には `forEach()` のコールバックで非同期タスクの実行をやるとミスが起きるのであまり使われないのではないでしょうか。`map()` の方が分かりやすいですし。`forEach()` のミスについては後で解説します。

# (2) 順番に興味があるので順序付けて実行
もしも、１つ目のリソースを取得が完了してから２つ目のリソースを取得したいという意図があるなら、await 式などで順序付ける必要がでてきます。データフェッチ以外のケースで、非同期の複数タスクの間に依存関係や順序関係がある場合にもこのようなやり方で行います。

いわゆる「直列」でのやり方です。

## 基本は for ループ

古典的な `for` ループを使えるのが async/await のメリットの１つです。古典的な表現ですが、順番に非同期タスクを行うための最も分かりやすい処理の形となります。

```js
(async () => {
  for (let i = 0; i < urls.length; i++) {
    await fetchThenConsole(urls[i]); // async 関数
    console.log(`${i + 1}個目のフェッチが完了しました`);
  }
  console.log("すべての非同期処理が完了しました");
})();
```

各 await 式でこの即時実行の async 関数は処理を一時的に中断して関数の外へと制御が以降します。Promise インスタンスが解決して処理再開を告げるマイクロタスクがコールスタックのトップになることで中断したところから処理再開します。

この場合は順番にリソースの取得が完了してから次のリソース取得を行っています。

## reduce で chain を繋げる

Promise chain しか使えずに async/await が登場するまではこの `for` ループによる古典的な書き方はできませんでした。代わりに配列の `reduce()` メソッドを使って Promise インスタンスにどんどん chain をくっつけていくというやり方で順番付けを行う「スマートなやり方」があります。

今度はデータフェッチではなく、Promise-based なタイマーで一定時間 `sleep()` する処理を考えてみます。

```js:sleep.js
export default function sleep(time) {
  return new Promise(resolve => setTimeout(() => {
    console.log(`${time}[ms]でタイムアウトしました`);
    resolve(time);
  }, time));
}
```

配列内のアルファベット文字列を順番に出力して、それぞれの出力の間に `slee(1000` で 1000 ミリ秒置いて実行させることを考えてみます。つまり、やりたいことは次のようなことです。

```js:normal.js
import sleep from './sleep.js';

const chars = ["A", "B", "C", "D", "E"];
(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");

  await sleep(1000);
  console.log(chars[0]);
  await sleep(1000);
  console.log(chars[1]);
  await sleep(1000);
  console.log(chars[2]);
  await sleep(1000);
  console.log(chars[3]);
  await sleep(1000);
  console.log(chars[4]);
  await sleep(1000);

  console.log("すべてのアルファベットを出力しました");
})();
```

これを実行すると次の出力を得ます。

```sh
❯ deno run normal.js
１秒ごとにアルファベットの出力を開始します
1000[ms]でタイムアウトしました
A
1000[ms]でタイムアウトしました
B
1000[ms]でタイムアウトしました
C
1000[ms]でタイムアウトしました
D
1000[ms]でタイムアウトしました
E
1000[ms]でタイムアウトしました
すべてのアルファベットを出力しました
```

これを反復処理で効率よく書いて実現させます。

その前にまずは `reduce()` メソッドの使い方を確認しておきます。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce

`reduce()` メソッドはコールバック関数と累積処理に利用する初期値を引数に取ります。コールバックが配列要素の個数分だけ累積処理を行った最終結果の値が返り値として返ってきます。各コールバック関数内で `return` した値が次の反復処理で使われます。その `return` した値が次のコールバックの入力の `priviousValue` となります。

```js
const result = array.reduce((
  previousValue, // 前回のコールバックから返却された値
  currentItem, // 現在の配列要素
  index, // 現在の配列のインデックス
  array, // 走査対象の配列そのもの 
  ) => {
  // なんらかの処理
  // 次のイテレーションの previousValue として使われる値を返却する
  return { /*...*/ };
}, initialValue);
```

例えば、1000 という数値に対して配列内の各数値を足した値を算出します。`reduce()` メソッドの第二引数に初期値を渡すと、その初期値が最初のコールバック処理の `prviousValue` として与えられます。

```js
const startVal = 1000;
const array = [1, 2, 3];
const result = array.reduce((previousValue, currentItem) => { 
  return previousValue + currentItem;
  // 最初 1000 + 1 = 1001 が返却される
  // 次 1001 + 2 = 1003 が返却される
  // 最後 1003 + 4 = 1006 が返却される
}, startVal);
console.log(result); //=>  1006
```

初期値を渡さない場合には配列の最初の要素が初期値となり、最初のコールバック処理では、`previosuValue` が `array[0]`、`currentItem` が `value[1]` となります。

```js
const array = [1, 2, 3];
const result = array.reduce((previousValue, currentItem) => { 
  return previousValue + currentItem;
  // 最初 1 + 2 = 3 が返却される
  // 次 3 + 3 = 6 が返却されて終わり
});
console.log(result); //=>  6
```

次に Promise chain で累積的に計算を行うことを考えてみます。まずは簡単な chian を `reduce()` で実現するところからです。

```js
const chars = ["A", "B", "C", "D", "E"];
const pChain = Promise.resolve()
  .then(() => console.log(chars[0]))
  .then(() => console.log(chars[1]))
  .then(() => console.log(chars[2]))
  .then(() => console.log(chars[3]))
  .then(() => console.log(chars[4]))
```

上の Promise chain を `reduce()` で実現するには各コールバックで Promise インスタンスを返します。そして返却する Promise インスタンスに `then()` で chain します。

```js
const chars = ["A", "B", "C", "D", "E"];

const pChain = chars.reduce((previous, item) => {
  return previous.then(() => console.log(item));
  // １回目: Promise.resolve().then(() => console.log("A"))
  // ２回目: Promise.resolve().then(() => console.log("A")).then(() => console.log("B"))
  // ３回目: Promise.resolve().then(() => console.log("A")).then(() => console.log("B")).then(() => console.log("C"))
  // ４回目: Promise.resolve().then(() => console.log("A")).then(() => console.log("B")).then(() => console.log("C")).then(() => console.log("D"))
}, Promise.resolve());
```

やっていることは、累積値に chain したものを `return` することで後ろに chain をくっつけていくということです。最初は考えるのが面倒ですが慣れれば結構シンプルだと分かります。

これの肝は初期値として最初から履行している Promise インスタンス `Promise.resolve()` を渡すことです。Promise chain を構築する上では chain の頭となる Promise インスタンスが必要となります。`reduce()` の処理によってこの初期値となる Promise インスタンスの後ろに順番に行いたい処理を `then()` で chian しています。

累積処理の結果として最終的にできあがった Promise chain が返ってくるのでこの一連の処理が終わった後に何かしたい場合にはさらに chain するか await 式で評価します。

```js
const chars = ["A", "B", "C", "D", "E"];

(async () => {
  const pChain = chars.reduce((previous, item) => {
    return previous.then(() => console.log(item));
  }, Promise.resolve());
  await pChain;
  console.log("一連の処理が終了しました");
})();
```

さて、話を戻して次の処理をこの方法で実現してみましょう。

```js:normal.js
import sleep from './sleep.js';

const chars = ["A", "B", "C", "D", "E"];
(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");

  await sleep(1000);
  console.log(chars[0]);
  await sleep(1000);
  console.log(chars[1]);
  await sleep(1000);
  console.log(chars[2]);
  await sleep(1000);
  console.log(chars[3]);
  await sleep(1000);
  console.log(chars[4]);
  await sleep(1000);

  console.log("すべてのアルファベットを出力しました");
})();
```

作り上げるべき Promise chain を理解するために、あえて上のコードを chain にしてみましょう。

```js
(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");

  await Promise.reoslve()
    .then(() => sleep(100))
    .then(() => console.log(chars[0]))
    .then(() => sleep(1000))
    .then(() => console.log(chars[1]))
    .then(() => sleep(1000))
    .then(() => console.log(chars[2]))
    .then(() => sleep(1000))
    .then(() => console.log(chars[3]))
    .then(() => sleep(1000))
    .then(() => console.log(chars[4]))

  console.log("すべてのアルファベットを出力しました");
})();
```

`sleep()` は Promise インスタンスを返す関数ですから、「副作用」とならないように `then()` のコールバックで `return` してあげる必要がありました。`sleep(1000)` から chain をはじめてもよいのですが、`reduce()` の初期値を考えやすくするために最初から履行している Promise インスタンスである `Promise.resolve()` を chain の先頭にします。

それではこの chian を目指して先程作った `pChain` を改造していきます。

```js
const chars = ["A", "B", "C", "D", "E"];

const pChain = chars.reduce((previous, item) => {
  return previous.then(() => console.log(item));
}, Promise.resolve());
```

返却する Promise chain に `sleep()` の chain を噛ませるだけです。コールバックの引数も累積値が Promise chain であることが分かりやすいように `promise` にしておきましょう。

```js
const chars = ["A", "B", "C", "D", "E"];

const pChain = chars.reduce((promise, item) => {
  return promise.then(() => sleep(1000)).then(() => console.log(item));
}, Promise.resolve());
```

元のコードのフォーマットに合わせておきます。

```js:reduceChain.js
import sleep from './sleep.js';

const chars = ["A", "B", "C", "D", "E"];
(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");

  const pChain = chars.reduce((promise, item) => {
    return promise // 前の chain にくっつけていく
      .then(() => sleep(1000))
      .then(() => console.log(item));
    }, Promise.resolve());
  await pChain;

  console.log("すべてのアルファベットを出力しました");
})();
```

これを実行することで意図踊りの結果が得られます。

```sh
❯ deno run reduceChain.js
１秒ごとにアルファベットの出力を開始します
1000[ms]でタイムアウトしました
A
1000[ms]でタイムアウトしました
B
1000[ms]でタイムアウトしました
C
1000[ms]でタイムアウトしました
D
1000[ms]でタイムアウトしました
E
すべてのアルファベットを出力しました
```

改良すべきところがあるとするなら、`then()` が多く、マイクロタスクが無駄に発生してしまうところを直しておきます。Promise chain が分かりやすいように初期値を `Promise.reoslve()` にしましたが、元のコードと同じく、`sleep(1000)` を初期値にして始めてよいでしょう。`console.log()` の実行と `sleep(1000)` の返却も１つの `then()` コールバックにまとめて良いでしょう。

```js:reduceChainKai.js
import sleep from './sleep.js';

const chars = ["A", "B", "C", "D", "E"];
(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");

  const pChain = chars.reduce((promise, item) => {
    return promise.then(() => {
        console.log(item);
        return sleep(100);
      });
    }, sleep(1000));
  await pChain;

  console.log("すべてのアルファベットを出力しました");
})();
```

前のコードよりも分かりづらいので改良といえるかは微妙ですが。

とにかく、`reduce()` メソッドで順番に非同期タスクを実行する反復処理を書くのは分かりづらいです。`for` ループで書いたほうが明らかに分かりやすくはなります。

```js
import sleep from "./sleep.js";

const chars = ["A", "B", "C", "D", "E"];

(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");
  for (let i = 0; i < chars.length; i++) {
    await sleep(1000);
    console.log(chars[i]);
  }
  await sleep(1000);
  console.log("すべてのアルファベットを出力しました");
})();
```

# async callback と forEach

配列の `forEach()` メソッドによる反復処理は使い方をミスすると意図しない結果になるのであまりおすすめではない方法です。

再びデータフェッチで考えてみましょう。`fetch()` から始まる一連の処理を単位にした反復処理を行いたいという次のコードは意図した結果にならないコードです。

```js:forEachAsync.js
const urls = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];

(async () => {
  urls.forEach((url) => {
    await fetch(url)
      .then((res) => res.json())
      .then((json) => console.log(json))
      .catch((err) => console.error(err));
  });
  // すべてのデータフェッチが終わってからコンソールに出力したいが...
  console.log("これが先に出力されてしまう");
})();
```

`forEach()` は `map()` と違って返り値はないので Promise 配列を作って await するようなことはまずできません。では `forEach()` のコールバックでそれぞれ await すればよいのではないかと考えてもダメです。

このコードを実行すると次の出力結果を得ます。

```sh
❯ deno run -A forEachAsync.js
これが先に出力されてしまう
{ userId: 1, id: 2, title: "quis ut nam facilis et officia qui", completed: false }
{ userId: 1, id: 1, title: "delectus aut autem", completed: false }
{ userId: 1, id: 3, title: "fugiat veniam minus", completed: false }
```

まずこのコードは VS code で書いていると Deno のリンターに次のように怒られます。

>Async arrow function has no 'await' expression.
Remove 'async' keyword from the function or use 'await' expression inside.deno-lint(require-await)

await 式は async 関数の直下でのみ有効です。`for` ループとは異なり、`forEach()` メソッドで行っているのはコールバック関数内で await 式を利用しています。コールバック内の await と asycn の即時実行関数は関連がありません。そのため、Deno のリンターに怒られないようにするにはコールバック関数そのものを async 化して即時実行の asycn キーワードを取り除きます。

```js
(() => {
  urls.forEach(async (url) => {
    await fetch(url)
      .then((res) => res.json())
      .then((json) => console.log(json))
      .catch((err) => console.error(err));
  });
  // すべてのデータフェッチが終わってからコンソールに出力したいが...
  console.log("これが先に出力されてしまう");
})();
```

async キーワードを取り除くと即時実行する必要がそのそもなくなってしまったので即時実行ではなく普通に実行するようにします(この時点で雲行きがかなり怪しくなってきました)。

```js
const urls = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];

urls.forEach(async (url) => {
  await fetch(url)
    .then((res) => res.json())
    .then((json) => console.log(json))
    .catch((err) => console.error(err));
});
console.log("これが先に出力されてしまう");
```

もちろんこれも意図踊りにはなりません。実行すると前と同じ結果になります。

```sh
❯ deno run -A forEachAsync.js
これが先に出力されてしまう
{ userId: 1, id: 2, title: "quis ut nam facilis et officia qui", completed: false }
{ userId: 1, id: 1, title: "delectus aut autem", completed: false }
{ userId: 1, id: 3, title: "fugiat veniam minus", completed: false }
```

結論を先に言ってしまうと、`forEach()` で async コールバックを使うこと自体がよくないです。

意図としては「データフェッチがすべて終わってからコンソールに完了のメッセージを出力したい」というものですが、async 化したコールバックの実行時に返ってくる Promise インスタンスを await 式で制御していませんので、どのようなことをしてもコンソール出力が先に行われてしまいます。

```js
// async 関数から返却される Promise インスタンスの制御をしていない
urls.forEach(async (url) => {
  await fetch(url)
    .then((res) => res.json())
    .then((json) => console.log(json))
    .catch((err) => console.error(err));
  // このコールバックの実行から返される Promise インスタンスは補足されていない
});
```

コールバックである async 関数から返される Promise インスタンスを取得できて await 式で評価できれば意図通りのコードがかけるはずです。

というか、そもそも async の即時実行関数でその範囲での実行と完了の保証をしたかったのに、async の即時実行を辞めたのも問題です。

`forEach()` での順序付けは方法が思いつきませんが(コールバックを async 化した時点で既に意図踊りになりません)、`forEach()` をなんとか使って並列化した上で制御することはできます。
とにかく async 関数の実行によって返される Promise インスタンスを取得できることが重要です。

このチャプターの冒頭の方で書いた方法ですが、配列を用意しておいて、その配列に async 関数や非同期 API の chain から返ってくる Promise インスタンスをいれておくことで別のところから制御できるようになります。

```js:forEachOk.js
const urls = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];

(async () => {
  const promises = [];
  // map メソッドでできることをわざわざ forEach を使ってやっている
  urls.forEach((url) => {
    // 起動して返ってくる Promise インスタンスを外にある配列に格納
    promises.push(
      fetch(url)
        .then((response) => response.json())
        .catch((e) => console.error(e))
    );
  });
  // Promise インスタンスを await 式で評価してこの関数内での完了を保証できる
  const jsons = await Promise.all(promises);
  console.log(jsons);

  console.log("データフェッチがすべて終わってから出力できる");
})();
```

実際にこのコードを実行すると意図通りの結果が得られます。

```sh
❯ deno run -A forEachOk.js
[
  { userId: 1, id: 1, title: "delectus aut autem", completed: false },
  { userId: 1, id: 2, title: "quis ut nam facilis et officia qui", completed: false },
  { userId: 1, id: 3, title: "fugiat veniam minus", completed: false }
]
データフェッチがすべて終わってから出力できる
```

`forEach()` の解説したのは `forEach()` で反復処理をしようということではなく、非同期タスクの返り値となる Promise インスタンスを await 式で評価して実行と完了の順序を制御することの重要性を確認するためです。

実際、上のようなコードを書くなら `map()` の方が素直にかけて良いです。順序付けを行いたいなら `for` ループなどで分かりやすいコードを書いたほうがきっと良いでしょう。

# for-await-of について

`for await...of` はジェネレーターやイテレーターが絡む非同期ループの構文です。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/for-await...of

今はまだ解説できるレベルの理解ではないので、これについては解説しません。ジェネレーターのチャプターが追加できればこちらも解説するかもしれません。

とりあえず `for` ループの一種であると考えておいてください。

