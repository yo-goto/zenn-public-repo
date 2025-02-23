---
title: "反復処理の制御"
cssclass: zenn
date: 2022-06-30
modified: 2024-08-14
AutoNoteMover: disable
tags: type/zenn/book, JavaScript/async
aliases: Promise本『反復処理の制御』
---

## このチャプターについて

このチャプターでは、非同期処理の反復処理、あるいは Promise が絡む反復処理について、await 式の配置と絡めてどのように反復処理をしていくかを考えて行きます。

主に JSON Placeholder という Fake API サービスを利用して複数のリソースからデータフェッチなどをすることを考えます。

https://jsonplaceholder.typicode.com

以下のように API のエンドポイントの URL を配列内に格納しておき、これを反復処理します。

```js
const urls = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];
```

反復処理では、やりたいことの意図に応じてデータフェッチを並列化したり、順序付けして逐次実行する２つのケースが考えられます。

## (1) 順番に興味がないので並列化して効率化

まずは順番に興味がない場合ですが、各 URL からリソースを取得する順番に意味が無ければ、`Promise.all()` などでまとめて上げて内部的に非同期 API を並列化することで効率化を図ることができます。データフェッチ以外のケースでも、非同期の複数タスクの間に依存関係や順序関係が無いならこのようなやり方で行います。

この場合は配列の `map()` メソッドなどを使って Promise の配列を作り、`await Promise.all()` ですべての完了をまってから次に何かを行うようにします。各 async 関数は Promise の配列を作る過程で並列的に起動させます。実際にはもちろん１つずつ起動させますが、内部的に利用される非同期 API が起動後に環境によってバックグラウンドで時間的に継続処理されるので実質的に「並列化」されることになります。

```js:並列化を分ける
(async () => {
  const responses = await Promise.all(urls.map(url => fetch(url)));
  const jsons = await Promise.all(responses.map(response => response.json()));
  jsons.forEach(json => console.log(json));
  console.log("すべての非同期処理が完了しました");
})();
```

```js:並列化を分けない
(async () => {
  const promises = urls.map(url =>
    fetch(url)
      .then(response => response.json())
      .then(json => console.log(json))
  );
  await Promise.all(proimses);
  console.log("すべての非同期処理が完了しました");
})();
```

どの程度の粒度の操作を単位にして await するかを考える必要がありますが、通常は共通の操作を async 関数にまとめて try-catch に閉じ込めるなどを行うのが一般的であると思われます。async 関数内の処理は独立させてそのレイヤーでの実行と完了の順番が担保されるようにします。

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

配列の `map()` ではなく `forEach()` を使おうとするのは一般的にアンチパターンとなります。`map()` メソッド内部で `return` した async 関数や非同期 API の完了を把握するための返ってくる Promise インスタンスが `forEach()` の場合だと返り値自体無いので取得しずらくなります。さらに `forEach()` のコールバックで await 式を使おうとするのは Syntax Error になりますし、それを回避してコールバックそのものを async 関数にしようとすると意図しない挙動になります。

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

一般的には `forEach()` のコールバックで非同期が絡む作業の実行をやるとミスが起きる上に、`map()` の方がわかりやすいので `map()` の使用が推奨されます。`forEach()` を使用することによって起こるミスについては後で解説します。

## (2) 順番に興味があるので順序付けて実行

もしも、１つ目のリソースを取得が完了してから２つ目のリソースを取得したいという意図があるなら、await 式などで順序付ける必要がでてきます。データフェッチ以外のケースで、非同期の複数タスクの間に依存関係や順序関係がある場合にもこのようなやり方で行います。

いわゆる「直列」でのやり方です。

### 基本は for ループ

古典的な `for` ループを使えるのが async/await のメリットの１つです。古典的な表現ですが、順番に非同期が絡む作業を行うための最も分かりやすい処理の形となります。

```js
(async () => {
  for (let i = 0; i < urls.length; i++) {
    await fetchThenConsole(urls[i]);
    // fetchThenConsole() は async 関数
    console.log(`${i + 1}個目のフェッチが完了しました`);
  }
  console.log("すべての非同期処理が完了しました");
})();
```

各 await 式でこの即時実行の async 関数は処理を一時的に中断して関数の外へと制御が移行します。Promise インスタンスが解決して処理再開を告げるマイクロタスクがコールスタックのトップになることで中断したところから処理再開します。

この場合には順番にリソースの取得が完了してから次のリソース取得を行っています。

### reduce で chain を繋げる

Promise chain しか使えずに async/await が登場するまではこの `for` ループによる古典的な書き方はできませんでした。代わりに配列の `reduce()` メソッドを使って Promise インスタンスにどんどん chain をくっつけていくというやり方で順番付けを行う「スマートなやり方」があります。

今度はデータフェッチではなく、Promise-based なタイマーで一定時間 `sleep()` する処理を考えてみます。このタイマーは別のファイルで `import` して使いたいので `export default` しておきます。

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

`reduce()` メソッドはコールバック関数と累積処理に利用する初期値を引数に取ります。コールバックが配列要素の個数分だけ累積処理を行った最終結果の値が返り値として返ってきます。各コールバック関数内で `return` した値が次の反復処理で使われます。その `return` した値が次のコールバックの入力の `previousValue` となります。

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
}, initialValue); // 初期値(最初の previousValue)
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

初期値を渡さない場合には配列の最初の要素が初期値となり、最初のコールバック処理では、`previousValue` が `array[0]`、`currentItem` が `value[1]` となります。

```js
const array = [1, 2, 3];
const result = array.reduce((previousValue, currentItem) => {
  return previousValue + currentItem;
  // 最初 1 + 2 = 3 が返却される
  // 次 3 + 3 = 6 が返却されて終わり
});
console.log(result); //=>  6
```

次に Promise chain で累積的に計算することを考えてみます。まずは簡単な Proimse chain を `reduce()` で実現するところから始めましょう。

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

これの肝は初期値として最初から履行している Promise インスタンス `Promise.resolve()` を渡すことです。Promise chain を構築する上では chain の頭となる Promise インスタンスが必要となります。`reduce()` の処理によってこの初期値となる Promise インスタンスの後ろには順番に行いたい処理を `then()` で chain しています。

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

作り上げるべき Promise chain を理解するために、あえて上のコードを chain にしてみましょう(分かりやすくするために無駄に一行ずつ `then()` のコールバックにしています)。

```js
(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");

  await Promise.resolve()
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

それではこの chain を目指して先程作った `pChain` を改造していきます。

```js
const chars = ["A", "B", "C", "D", "E"];

const pChain = chars.reduce((previous, item) => {
  return previous.then(() => console.log(item));
}, Promise.resolve());
```

返却する Promise chain に `sleep()` の chain を噛ませるだけです。累積値が Promise chain であると分かりやすいようにコールバックの引数も `promise` にしておきましょう。

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

これを実行することで意図通りの結果が得られます。

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

改良すべきところがあるとするなら、`then()` が多く、マイクロタスクが無駄に発生してしまうところを直しておきます。Promise chain が分かりやすいように初期値を `Promise.resolve()` にしましたが、元のコードと同じく、Promise インスタンスが返ってくる `sleep(1000)` を初期値にして始めてよいでしょう。`console.log()` の実行と `sleep(1000)` の返却も１つの `then()` コールバックにまとめて良いでしょう。

```js:reduceChainKai.js
import sleep from './sleep.js';

const chars = ["A", "B", "C", "D", "E"];
(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");

  const pChain = chars.reduce((promise, item) => {
    return promise.then(() => {
        console.log(item);
        return sleep(100); // 副作用にならないように return する
      });
    }, sleep(1000));
  await pChain;

  console.log("すべてのアルファベットを出力しました");
})();
```

前のコードよりも分かりづらいので改良といえるかは微妙ですが。

とにかく、`reduce()` メソッドで順番に非同期が絡む作業を実行する反復処理を書くのは分かりづらいです。次のように `for` ループで書いたほうが明らかに分かりやすくなります。

```js
import sleep from "./sleep.js";

const chars = ["A", "B", "C", "D", "E"];

(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");
  await sleep(1000);
  for (let i = 0; i < chars.length; i++) {
    console.log(chars[i]);
    await sleep(1000);
  }
  console.log("すべてのアルファベットを出力しました");
})();
```

## async callback

### async callback と forEach

配列の `forEach()` メソッドによる反復処理は使い方をミスすると意図しない結果になるので、おすすめではない方法です。

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
{ userId: 1, id: 2, title: "quis ut name facilis et officia qui", completed: false }
{ userId: 1, id: 1, title: "delectus aut autem", completed: false }
{ userId: 1, id: 3, title: "fugiat veniam minus", completed: false }
```

まずこのコードは VS code で書いていると Deno のリンターの "require-await" ルールによって再び怒られてしまいます。

>Async arrow function has no 'await' expression.
Remove 'async' keyword from the function or use 'await' expression inside.deno-lint(require-await)

await 式は async 関数の直下でのみ有効です。`for` ループとは異なり、`forEach()` メソッドで行っているのはコールバック関数内で await 式を利用しています。コールバック内の await と async の即時実行関数は関連がありません。そのため、Deno のリンターから怒られないようにするにはコールバック関数そのものを async 化して即時実行の async キーワードを取り除きます。

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

async キーワードを取り除くと即時実行する必要がそもそも無くなってしまったので即時実行ではなく普通に実行するようにします(この時点で雲行きがかなり怪しくなってきました)。

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

もちろんこれも意図通りにはなりません。実行すると前と同じ結果になります。

```sh
❯ deno run -A forEachAsync.js
これが先に出力されてしまう
{ userId: 1, id: 2, title: "quis ut name facilis et officia qui", completed: false }
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
  // このコールバックの実行から返される Promise インスタンスが捕捉できない
});
```

コールバックである async 関数から返される Promise インスタンスを取得できて await 式で評価できれば意図通りのコードがかけるはずです。

というか、そもそも特定の範囲での実行と完了の保証をしたかったのに、async の即時実行を辞めてしまったのも問題です。

`forEach()` での順序付けは方法が思いつきませんが(コールバック関数を async 化した時点で既に意図通りになりません)、`forEach()` をなんとか使って並列化した上で制御することはできます。とにかく **async 関数の実行によって返される Promise インスタンスを取得できることが重要**です。

:::message
`forEach()` で `reduce()` のように Promise chain をどんどん繋げていくパターンもあるらしいので、順序付けた逐次処理はそれでできそうです。
:::

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
  // Promise.all() は Promise を返すので、await 式で評価してこの関数内での完了を保証できる
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
  { userId: 1, id: 2, title: "quis ut name facilis et officia qui", completed: false },
  { userId: 1, id: 3, title: "fugiat veniam minus", completed: false }
]
データフェッチがすべて終わってから出力できる
```

`forEach()` を解説したのは「`forEach()` で反復処理をしましょう」ということではなく、「非同期が絡む作業の返り値となる Promise インスタンスを await 式で評価して実行と完了の順序を制御することが重要である」ということを確認するためです。Promise インスタンスの評価ができないものは副作用的な振る舞いとなり特定範囲内での完了を担保できなくなってしまいます。

実際、上のようなコードを書くなら `map()` の方が素直に書けるので良いです。順序付けを行いたいなら `for` ループなどで分かりやすいコードを書いたほうがきっと良いでしょう。

### async callback と map

async callback は `forEach()` ではなく、`map()` メソッドでならちゃんと使うことができます。

`map()` メソッドは渡したコールバック関数の返り値で新しい配列を作成しますが、async callback の返り値は Promise インスタンスですから、Promise インスタンスの配列をしっかりと作成できます。『[V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)』のチャプターで見たとおり、async 関数のボディで何も `return` していなくても `undefined` で履行する Promise インスタンスが返されるので、`map()` メソッドはコールバック関数の返り値としてその Promise インスタンスを捕捉します。

```js
// Promise インスタンスの配列を作成する
const promises = urls.map(async (url) => {
  try {
    const response = await fetch(url);
    const text = await response.text();
    console.log(text);
  } catch(err) {
    console.log(err);
  }
  // return される値が無いので undefined で履行した Promise インスタンスがコールバックから返る
});
```

これは次のように async 関数として外部に切り出して `map()` のコールバックで起動させると変わりません。

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

// コールバック内で async 関数を起動して返ってくる Promise インスタンスを集めた配列を作成
const promises = urls.map(url => fetchThenConsole(url));
```

## for ループの派生

順番に興味があるので順序付けて実行する場合には基本的に `for` ループを使用しますが、`for` ループには以下のようないくつかの派生形が存在しています。

- `for...in`
- `for...of`
- `for await...of`

### for...in

`for...in` はオブジェクトのプロパティを反復するために作られたものですが、この方法にはいくつかの問題があるため、デバッグ目的以外には基本的に使いません。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/for...in

オブジェクトの反復処理をしたい場合にはオブジェクト列挙のための静的メソッドである `Object.keys()` や `Object.values()`、`Object.entries()` などを利用して配列を作り出すことによって反復処理を行います。

### イテラブルオブジェクトの反復処理

残り２つの `for...of` と `for await...of` はイテラブル(iterable)や非同期イテラブル(async iterable)が絡みます。『[イテレータとイテラブルとジェネレータ関数](k-epasync-iterator-generator)』のチャプターで詳しく解説しますが「イテラブル(iterable=反復可能)なオブジェクト」に対して各ループで反復対象となる要素に変数を割り当てて反復処理ができます。

ビルトインオブジェクトでイテラブルなものとして代表的なのは配列です。次のように API のエンドポイントの URL 配列があるときには、`for...of` の構文で反復処理が可能です。

```js:forOfIteration.js
const urls = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];

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
  // 配列はイテラブルオブジェクトなので for...of 構文が使える
  for (const url of urls) {
    await fetchThenConsole(url); 
  }
  console.log("すべての非同期処理が完了しました");
})();
```

実際に実行すると次のような出力を得ます。
```sh
❯ deno run -A forOfIteration.js
{
  "userId": 1,
  "id": 1,
  "title": "delectus aut autem",
  "completed": false
}
{
  "userId": 1,
  "id": 2,
  "title": "quis ut name facilis et officia qui",
  "completed": false
}
{
  "userId": 1,
  "id": 3,
  "title": "fugiat veniam minus",
  "completed": false
}
すべての非同期処理が完了しました
```

`for await...of` は非同期ジェネレータ関数などを使うので詳しくは『[イテレータとイテラブルとジェネレータ関数](k-epasync-iterator-generator)』のチャプターで解説します。
