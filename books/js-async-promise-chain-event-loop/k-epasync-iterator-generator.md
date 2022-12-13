---
title: "イテレータとイテラブルとジェネレータ関数"
cssclass: zenn
date: 2022-07-18
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『イテレータとイテラブルとジェネレータ関数』
---

# このチャプターについて

このチャプターでは非同期処理が絡む反復処理に利用するイテレータやイテラブル、ジェネレータ関数などを解説しておきます。

`Promise.all()` などのメソッドも配列ではなくて実はイテラブルなオブジェクトを引数として受け入れているに過ぎません。

一見難しそうですが、「プロトコル」のことが理解できればイテレータとイテラブルは理解できますし、イテレータとイテラブルが理解できればジェネレータ関数も理解できます。非同期ジェネレータ関数もジェネレータ関数と今までの非同期処理が理解できていれば割と簡単に理解できますので、恐れずに進みましょう。

:::message alert
イテレータ(iterator)やイテラブル(iterable)、イテレーション(iteration)などややこしい単語がでてくるのでそこだけは注意してください。
:::

## 参考文献

イテレータとイテラブルとジェネレータ関数についてはこちらの kura07 さんの記事でも非常にわかりやすく解説されているので詳細などについては参考にしてください。

https://qiita.com/kura07/items/cf168a7ea20e8c2554c6

https://qiita.com/kura07/items/d1a57ea64ef5c3de8528

# 反復処理プロトコル

まず「プロトコル(protocol)」とは約束事や規約のことです。ES2015 では新しい反復処理の機能のためにとあるプロトコルが導入されました。それが「反復処理プロトコル(iteration protocol)」です。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Iteration_protocols

この「反復処理プロトコル(iteration protocol)」を満たすオブジェクトは同様に ES2015 で追加された `for...of` といった新しい反復処理の構文で使うことができます。

反復処理プロトコル(iteration protocol)はさらに２つのプロトコルから構成されます。

- 反復処理プロトコル(**iteration** protocol)
  - (1) 反復子プロトコル(**iterator** protocol)
  - (2) 反復可能プロトコル(**iterable** protocol)

(1) と (2) の両方のプロトコルを満たすことで反復処理プロトコル(iteration protocol)になります。ということで基本的には両方のプロトコルが実装してある必要があります。

:::message alert
各プロトコルについて概要は説明しますが、厳密な定義の説明はしませんので、注意してください。
:::

## イテレータと反復子プロトコル

反復子プロトコル(iterator protocol)は以下のような規約です。

- `next()` という引数が０個または１個のメソッドを持つ
- `next()` が実行されると `done` と `value` という少なくとも２つのプロパティを持つイテレータリザルト(iterator result)というオブジェクトを返す

この規約を満たすようなオブジェクトをイテレータ(iterator)と呼びます。実際に最小限でイテレータをつくってみると次のようになります。

```js
// イテレータ(iterator protocol を満たすオブジェクト)
const iterator = {
  next() { // メソッドの短縮記法
    const iteratorResult = { 
      value: 42, // イテレータリザルトが持つべきプロパティ
      done: false // イテレータリザルトが持つべきプロパティ
    };
    return iteratorResult;
    // イテレータリザルトを返す
  }
}

console.log(iterator.next());
// => { value: 42, done: false }
```

これだけです。これで何ができるかはもう１つのプロトコルをみないとイマイチ分かりません。

:::message
ちなみにメソッドの定義方法は以下のような書き方があります。上の書き方はメソッドの短縮記法と呼ばれるものです。

```js
// オブジェクトのメソッドの定義方法は３つ
const obj = {
  method1: function() { 
    // function キーワードを使ったメソッド定義
  },
  method2: () => {
    // アロー関数を使ったメソッド定義
  },
  method3() {
    // 短縮記法を使ったメソッド定義
  },
};
```
:::

## イテラブルオブジェクトと反復可能プロトコル

反復可能プロトコル(iterable protocol)は以下のような規約です。

- `[Symbol.iterator]()` という引数なしのメソッドを持つ
- `[Symbol.iterator]()` が実行されるとされるとイテレータ(反復子プロトコルを満たすオブジェクト)を返す

この規約を満たすようなオブジェクトをイテラブルなオブジェクト(iterable object)と呼びます。日本語なら反復可能オブジェクトです。実際に最小限でイテラブルなオブジェクトをつくってみると次のようになります。

```js
const iterableObject = {
  // 短縮記法と計算プロパティ名でメソッド定義
  [Symbol.iterator]() { 
    const iterator = {
      // 短縮記法でメソッド定義
      next() { 
        const iteratorResult = { value: 42, done: false };
        // イテレータリザルトを返す
        return iteratorResult;
      }
    };
    // イテレータを返す
    return iterator;
  }
};

// ブラケット記法でメソッドを実行するとイテレータが返ってくる
const myiterator = iterableObject[Symbol.iterator]();
```

`Symbol.itrator` のメソッドをオブジェクト内で直接的に定義する際にはこのようにブラケットで囲む必要があります。この書き方は計算プロパティ名(Computed property names)と呼ばれるものです。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Object_initializer#%E8%A8%88%E7%AE%97%E3%83%97%E3%83%AD%E3%83%91%E3%83%86%E3%82%A3%E5%90%8D

# イテラブルの活用

反復可能プロトコル(iteable protocol)を満たしていれば `[Symbol.iterator]()` メソッドがイテレータを返すので、ほとんど反復処理プロトコル(iteration protcol)を満たしていると言ってもよいでしょう。

ということで、反復処理プロトコル(iteration protocol)という言葉はあまり使われず、反復可能プロトコル(iterable protocol)を満たしているかどうかに基本的に焦点があてられます。

さて、上のコードのままでは何が嬉しいのかイマイチわからないのでイテラブルなオブジェクトの実装にもう少し条件を加えて、内部のカウンターをインクリメントしていく処理を実装してみます。

```js
const iterableObject = {
  [Symbol.iterator]() { 
    let count = 0;

    const iterator = {
      next() { 
        // 三項演算子で返すものを条件づけ
        const iteratorResult = (count < 3)
          ? { value: ++count, done: false }
          : { value: undefined, done: true };

        // イテレータリザルトを返す
        return iteratorResult;
      }
    };
    // イテレータを返す
    return iterator;
  }
};
```

イテラブルオブジェクトの `[Symbol.iterator]()` メソッドを実行するとイテレータが返ってきました。さらにイテレータは `next()` メソッドを持っており、このメソッドを実行すると `value` と `done` プロパティを持つイテレータリザルトというオブジェクトが返ってきます。

```js
// [Symbol.iteartor]() メソッドを実行するとイテレータが返ってくる
const iterator = iterableObject[Symbol.iterator]();

// next() メソッドを実行するとイテレータリザルトが返ってくる
console.log(iterator.next()); //  => { value: 1, done: false } 
console.log(iterator.next()); //  => { value: 2, done: false }
console.log(iterator.next()); //  => { value: 3, done: false }
console.log(iterator.next()); // => { value: undefined, done: true }
console.log(iterator.next()); //  => { value: undefined, done: true }
```

このような感じで `next()` メソッドを次々に実行することで反復的に値をインクリメントする処理を行っていくことができます。反復処理において `value` プロパティと `done` プロパティは以下のようなものとして機能することを認識しておくと良いでしょう。

- `value` プロパティには反復処理の結果となる値
- `done` プロパティには反復処理が完了したかどうかの真偽値

特に `done` プロパティは `false` (反復処理が完了していない) から始まり、反復処理が完了すると `true` (反復処理が完了した) となって反復処理が終わったことを表現します。

## for...of

先程定義したイテラブルオブジェクトを実際に `while` ループで `done` プロパティが `true` になるまで反復処理を行ってみましょう。

```js:sympleIterable.js
const iterableObject = {
  [Symbol.iterator]() { 
    let count = 0;
    const iterator = {
      next() { 
        const iteratorResult = (count < 3)
          ? { value: ++count, done: false }
          : { value: undefined, done: true };
        return iteratorResult;
      }
    };
    return iterator;
  }
};

const myIterator = iterableObject[Symbol.iterator]();
let myIteratorResult;
while (true) {
  myIteratorResult = myIterator.next();
  if (myIteratorResult.done) break;
  // done: true になったら break する
  console.log(myIteratorResult.value);
}
```

`deno run` で実行すると以下の出力を得ます。
```sh
❯ deno run simpleIterable.js
1
2
3
```

このように反復処理プロトコルを実装しているオブジェクトに反復処理を行うのに以下のように色々書かないといけないのは面倒です。

```js
const myIterator = iterableObject[Symbol.iterator]();
let myIteratorResult;
while (true) {
  myIteratorResult = myIterator.next();
  if (myIteratorResult.done) break;
  console.log(myIteratorResult.value);
}
```

そこで ES2015 で追加された `for...of` の構文を利用します。この構文は上のような処理の手続きをまとめて行ってくれるのでイテレータから値を反復して取り出すのが簡単にできます(実際には、「`for...or` 構文を使えるのがイテラブルオブジェクト」という認識で良いです)。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/for...of

`for...of` は次のような構文です。`of` の後にはイテラブルオブジェクトを期待して、`variable` にイテレータリザルトの `value` プロパティの値を割り当ててブロック内で反復したい処理を行います。

```js
for (variable of IterableObject) {
  // variable を使った反復処理
}
```

これでイテラブルオブジェクトからイテレータをわざわざ取り出す必要がなくなります。

実際に上のイテラブルオブジェクトを使って反復処理を行ってみます。

```js:simpleForOf.js
const iterableObject = {
  [Symbol.iterator]() { 
    let count = 0;
    const iterator = {
      next() { 
        const iteratorResult = (count < 3)
          ? { value: ++count, done: false }
          : { value: undefined, done: true };
        return iteratorResult;
      }
    };
    return iterator;
  }
};

for (const v of iterableObject) {
  console.log(v);
}
```

`deno run` で実行すると以下の出力を得ます。
```sh
❯ deno run simpleForOf.js
1
2
3
```

`v` を `const` 宣言しているのは各イテレーション毎にブロックで変数を定義するからです。内部でインクリメントなどしなければこれで大丈夫です。

## ビルトインのイテラブルオブジェクト

実はビルトインオブジェクトのいくつかがイテラブルなオブジェクトとして規定されています。イテラブルなビルトインオブジェクトは以下のものとなります。

- `String`
- `Array`
- `TypedArray`
- `Map`
- `Set`

:::message alert
オブジェクトリテラルなどで作成した通常のオブジェクトは、自分で `[Symbol.iterator]()` や `next()` などを実装する必要があったのでイテレータやイテラブルではありません。
:::

つまり、これらのオブジェクトは `for...of` の構文で反復処理ができることになります。

```js
const arr = [1, 2, 3];
for (const v of arr) console.log(v);
/* 出力
1
2
3
*/

const str = "ABC";
for (const v of str) console.log(v);
/* 出力
A
B
C
*/
```

文字列や配列などはイテラブルオブジェクトであり、反復可能プロトコルと反復子プロトコルの両方を満たしています。従って、次のように `for...of` で省略した手続きを自分で書くこともできます。

```js
const arr = [1, 2, 3];
// イテレータを取得
const arrIterator = arr[Symbol.iterator]();
let arrIteratorResult;
while (true) {
  arrIteratorResult = arrIterator.next();
  if (arrIteratorResult.done) break;
  console.log(arrIteratorResult.value);
}
/* 出力
1
2
3
*/

const str = "ABC";
// イテレータを取得
const strIterator = str[Symbol.iterator]();
let strIteratorResult;
while (true) {
  strIteratorResult = strIterator.next();
  if (strIteratorResult.done) break;
  console.log(strIteratorResult.value);
}
/* 出力
A
B
C
*/
```

配列がイテラブルであることから、オブジェクトのプロパティに対して反復処理を行いたいときも `Object.keys()` や `Object.values()` などのオブジェクトの静的メソッドを使って一旦配列にすることで `for...of` で反復処理ができるようになります。

```js
const fruits = {
  apple: "🍎",
  cherry: "🍒",
  banana: "🍌",
};

for (const value of Object.values(fruits)) {
  console.log(value);
}
/* 出力
🍎
🍒
🍌
*/

for (const key of Object.keys(fruits)) {
  console.log(key);
}
/* 出力
apple
cherry
banana
*/

for (const entry of Object.entries(fruits)) {
  console.log(entry);
}
/* 出力
[ "apple", "🍎" ]
[ "cherry", "🍒" ]
[ "banana", "🍌" ]
*/
```

## spread 構文と分割代入

イテラブルオブジェクトに使える構文は `for...of` だけでなく、spread 構文や分割代入、`yield*` なども含まれています。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Iteration_protocols#%E5%8F%8D%E5%BE%A9%E5%8F%AF%E8%83%BD%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%82%92%E6%9C%9F%E5%BE%85%E3%81%99%E3%82%8B%E6%A7%8B%E6%96%87

spread 構文などは配列に使える構文というより、本質的にはイテラブルオブジェクトに使える構文という訳です。

```js:iterableSyntax.js
const arr = [1, 2, 3];
const str = "ABC";

// spread 構文
console.log(...arr); // => 1 2 3
console.log(...str); // => A B C

// 分割代入
const [num1, num2, num3] = arr;
console.log(num1);  // => 1
console.log(num2);  // => 2
console.log(num3);  // => 3
const [char1, char2, char3] = str;
console.log(char1); // => A
console.log(char2); // => B
console.log(char3); // => C
```

`Promise.all()` などの Promise の性的メソッドは実は引数にイテラブルオブジェクトを取るようになっています。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all

MDN のシンタックス表示。
```js
Promise.all(iterable);
```

実際、文字列はイテラブルなので `Promise.all()` の引数として渡すことができます。
```js
const str = "ABC";

Promise.all(str).then(val => console.log(val))
/* 出力
[ "A", "B", "C" ]
*/
```

通常 `Promise.all()` には Promise インスタンスの配列を引数として渡しますが、このようにイテラブルオブジェクトであればなんでも渡せます。イテラブルオブジェクト内に Promise インスタンスが無ければ、内部的に１回マイクロタスクを消費して履行状態となるので、非同期的に解決されることになります。

例えば上のコードを少し改造してみます。

```js:promiseAllTest.js
const str = "ABC";

Promise.all(str) // 非同期的に解決される(マイクロタスクを消費)
  .then(val => console.log("[3]", val));

Promise.resolve() // 同期的に直ちに解決される
  .then(() => console.log("[2] Promise.resolve"));

console.log("[1] MAINLINE");
```

このようなコードを実行すると、`Promise.resolve()` で同期的に直ちに履行する Promise インスタンスに chain されている `then()` に登録されているコールバック関数の方が先に処理されます。

```sh
❯ deno run promiseAllTest.js
[1] MAINLINE
[2] Promise.resolve
[3] [ "A", "B", "C" ]
```

## 非同期の反復処理で for...of を利用する

配列などがビルトインのイテラブルオブジェクトであると分かった今、『[反復処理の制御](19-epasync-async-loop)』のチャプターで見たような反復処理は `for...of` 構文を使ってわかりやすくイテレーションできることに気づきます。次のような API エンドポイントの URL が配列などで与えられていれば、これを使った反復処理は `for...of` でできます。

```js
const urls = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];
```

前のチャプターでは、古典的な初期化子 `i` のインクリメントを使った `for` のイテレーションでやっていました。

```js
(async () => {
  for (let i = 0; i < urls.length; i++) {
    await fetchThenConsole(urls[i]); 
    // fetchThenConsole() は async 関数
  }
  console.log("すべての非同期処理が完了しました");
})();
```

これを新しいイテラブルオブジェクトで使える `for...of` によって書き換えると次のようになります。

```js
(async () => {
  // urls は配列でイテラブルなので for...of が使える
  for (const url of urls) {
    await fetchThenConsole(url); 
  }
  console.log("すべての非同期処理が完了しました");
})();
```

また、文字列を１秒ずつ待って出力する次の処理も配列を使っているので `for...of` でイテレーションするように変えることができます。

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

`for...of` で書き換えると次のようになります。

```js
import sleep from "./sleep.js";

const chars = ["A", "B", "C", "D", "E"];

(async () => {
  console.log("１秒ごとにアルファベットの出力を開始します");
  await sleep(1000);
  for (const char of chars) {
    console.log(char);
    await sleep(1000);
  }
  console.log("すべてのアルファベットを出力しました");
})();
```

配列に限らず、ビルトインのイテラブルオブジェクトである `Map` や `Set` などでも使える汎用性のある構文です。

## ジェネレータ関数

ジェネレータ関数(generator function)は関数内の処理の一時停止が可能な関数です。ジェネレータの機能はイテラブルやイテレータをサポートするものです。

ジェネレータ関数は `function*` キーワードで関数宣言ができます。

```js
function* generatorFn() {
  //... なんらかの処理
}
```

ジェネレータ関数からはジェネレータオブジェクト(generator object)というオブジェクトが返されます。

```js
function* generatorFn() {
  //... なんらかの処理
}
// ジェネレータオブジェクト(ジェネレータ関数から取得)
const generator = generatorFn();
```

そして、ジェネレータ関数では async 関数の `await` 式のように `yield` というものがあり、ジェネレータ関数内の `yield` する値を返して関数内の処理を一時停止できます。

ジェネレータ関数から返されるジェネレータオブジェクトはイテレータであるので `next()` メソッドを使ってイテレータリザルトを返すことができます。

```js
function* generatorFn(n) {
  yield n; // このポイントで処理を中断できる
  n++;
  yield n; // このポイントで処理を中断できる
  n++;
  yield n; // ジェネレータ関数の終了
}
// ジェネレータオブジェクト(ジェネレータ関数から取得)
const generatorIsIterator = generatorFn(1);

// ジェネレータオブジェクトはイテレータなので next メソッドが使える
console.log(generatorIsIterator.next()); // { value: 1, done: false }
console.log(generatorIsIterator.next()); // { value: 2, done: false }
console.log(generatorIsIterator.next()); // { value: 3, done: false }
console.log(generatorIsIterator.next()); // { value:  undefined, done: true }
```

このジェネレータオブジェクトはイテレータであると同時にイテラブルであるので `for...of` で反復処理ができます。

```js
function* generatorFn(n) {
  yield n; // このポイントで処理を中断できる
  n++;
  yield n; // このポイントで処理を中断できる
  n++;
  yield n; // ジェネレータ関数の終了
}
// ジェネレータオブジェクト(ジェネレータ関数から取得)
const generatorIsIterable = generatorFn(1);

// ジェネレータオブジェクトはイテラブルでもある
for (const v of generatorIsIterable) console.log(v);
/* 
1
2
3
*/
```

ジェネレータ関数を実行するとイテラブルなジェネレータオブジェクトが返ってくるので、`for...of` で直接実行してもよいです。

```js
function* genFn(start, end) {
  while (start <= end) {
    yield start++;
  }
}

// ジェネレータ関数からはイテラブルなジェネレータオブジェクトが返る
for (const v of genFn(2, 5)) console.log(v);
/* 出力
2
3
4
5
*/
```

ジェネレータ関数(generator function)はこのように値を自由に生成(generatate)するなどに利用できます。

ジェネレータオブジェクトはイテラブルなので、spread 構文や分割代入もできます。

```js
function* genFn(start, end) {
  while (start <= end) {
    yield start++;
  }
}

// spread 構文
console.log(...genFn(1, 3)); // => 1 2 3
// 分割代入
const [num1, num2, num3] = genFn(1, 3);
console.log(num1) // => 1
console.log(num2) // => 2
console.log(num3) // => 3
```

ジェネレータ関数内で利用できるもう１つの式して `yield*` 式があります。この式によってジェネレータ関数内でイテラブルなオブジェクトの反復処理ができるようになります。具体的には `yield*` 式で評価したイテラブルなオブジェクトに対して反復的に `yield` を行うことができます。

```js
function* genFn() {
  yield* [1, 2, 3]; // 配列はイテラブル
  yield* "ABC"; // 文字列はイテラブル
}

// ジェネレータオブジェクト(イテレータかつイテラブル)を取得
const gen = genFn();

// ジェネレータオブジェクトはイテレータなので next() でイテレータリザルトが返ってくる
console.log(gen.next()); // => { value: 1, done: false }
console.log(gen.next()); // => { value: 2, done: false }
console.log(gen.next()); // => { value: 3, done: false }
console.log(gen.next()); // => { value: A, done: false }
console.log(gen.next()); // => { value: B, done: false }
console.log(gen.next()); // => { value: C, done: false }
console.log(gen.next()); // => { value: undefined, done: true }
```

上のジェネレータ関数を `yield` のみで書くと次のようになります。`yield*` はイテラブルなオブジェクトの要素に対する連続的な `yield` を
yield await Promise.resolve(2);表現していると捉えられます。

```js
function* genFnA() {
  for (const v of [1, 2, 3]) yield v;
  for (const k of "ABC") yield k;
}

// 要素ごとに分解して書くとこうなる
function* genFnB() {
  yield 1;  
  yield 2;
  yield 3;  
  yield "A";
  yield "B";
  yield "C";
}
```

ジェネレータオブジェクトはイテレータかつイテラブルだったので、`for...of` で反復処理が可能です。

```js
function* genFn() {
  yield* [1, 2, 3]; // 配列はイテラブル
  yield* "ABC"; // 文字列はイテラブル
}
for (const v of genFn()) console.log(v);
/* 出力
1
2
3
A
B
C
*/
```

# 非同期反復可能プロトコル

ようやく非同期処理に戻ってくることができましたね。

反復可能プロトコル(itrable protocol)ですが、実は非同期版もあります。それが非同期反復可能プロトコル(async iterable protocol)です。

反復可能プロトコル(itrable protocol)を満たすオブジェクトは `[Symoble.iterator]()` メソッドによってイテレータリザルトが返ってきましたが、非同期反復可能プロトコル(async iterable protocol)を満たすオブジェクトは `[Symbol.asyncIterator]()` というメソッドによって Promise インスタンスでラップされたイテレータリザルトが返ってきます。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Symbol/asyncIterator

実際に自分で実装するとこのようになります。

```js
const asyncIterable = {
  [Symbol.asyncIterator]() {
    let count = 0;
    const asyncIterator = {
      next() {
        const promiseIteratorResult = (count < 3)
          ? Promise.resolve({ value: ++count, done: false })
          : Promise.resolve({ done: true });
        return promiseIteratorResult;
      }
    };
    return asyncIterator;
  },
};
```

async イテラブルオブジェクトの `[Symbol.asyncIterator]` からは async イテレータが返ってくるので `next()` メソッドが実行できます。`next()` メソッドからは Promise インスタンスが返ってくるので値を取り出すためには `await` 式で評価してあげる必要があります。

```js
(async () => {
  const asyncIterator = asyncIterable[Symbol.asyncIterator]();
  // next() メソッドからは Promise インスタンスが返ってくるので await 式で評価する
  console.log(await asyncIterator.next()); // => { value: 1, done: false}
  console.log(await asyncIterator.next()); // => { value: 2, done: false}
  console.log(await asyncIterator.next()); // => { value: 3, done: false}
  console.log(await asyncIterator.next()); // => { done: true }
})();
```

## for await...of

通常の反復可能プロトコルを満たしているオブジェクトが反復処理の構文である `for...of` の利用を期待できたのと同じように、非同期の反復可能プロトコルを見対しているオブジェクトでは非同期の反復処理の構文である `for await...of` の利用を期待できます。

```js
const asyncIterable = {
  [Symbol.asyncIterator]() {
    let count = 0;
    const asyncIterator = {
      next() {
        const promiseIteratorResult = (count < 3)
          ? Promise.resolve({ value: ++count, done: false })
          : Promise.resolve({ done: true });
        return promiseIteratorResult;
      }
    };
    return asyncIterator;
  },
};

(async () => {
  const asyncIterator = asyncIterable[Symbol.asyncIterator]();
  let myIteratorResult;
  while (true) {
    myIteratorResult = await asyncIterator.next();
    if (myIteratorResult.done) break;
    // done: true になったら break する
    console.log(myIteratorResult.value);
  }
})();
/* 出力
1
2
3
*/
```

再びイテレータから値を取り出すプロセスを `for...of` と同じように新しい構文である `for await...of` でまとめあげることができます。

```js
const asyncIterable = {
  [Symbol.asyncIterator]() {
    let count = 0;
    const asyncIterator = {
      next() {
        const promiseIteratorResult = (count < 3)
          ? Promise.resolve({ value: ++count, done: false })
          : Promise.resolve({ done: true });
        return promiseIteratorResult;
      }
    };
    return asyncIterator;
  },
};

(async () => {
  for await (const v of asyncIterable) {
    console.log(v);
  }
})();
/* 出力
1
2
3
*/
```

これだけです。

## 非同期ジェネレータ関数

ジェネレータ関数が反復可能プロトコルを実装していたように、非同期ジェネレータ関数(async generator funciton)は非同期反復可能プロトコルを実装しています。

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncGenerator

それゆえ、`for await...of` の構文が使えます。非同期ジェネレータ関数は `async function*` というようにジェネレータ関数の宣言の前に `async` キーワードをつけるだけです。

```js
async function* asyncGenFn(start, end) {
  while (start <= end) {
    yield start++;
  }
}

(async () => {
  // 非同期ジェネレータ関数からは非同期のイテラブルオブジェクトが返る
  for await (const v of asyncGenFn(1, 3)) {
    console.log(v);
  }
})();
/* 出力
1
2
3
*/
```

`await` 式を利用しているのでマイクロタスクが各イテレーションで発生し、更に非同期ジェネレータ関数内の各 `yield` ごとに Promise インスタンスが返されることに注意してください。

非同期ジェネレータ関数の意味は「ジェネレータ関数の内部で `await` 式が利用できるようになった」程度で十分です。実際、ジェネレータ関数で `yield` だけでなく `await` 式も使えるようになっただけです。

ジェネレータ関数の内部で Promise-based API などを利用したいときはこの非同期ジェネレータ関数が役立つことになります。

```js
async function* asyncGenFn(url) {
  console.log("非同期のジェネレータ関数の処理を開始");
  // ジェネレータ関数内部で Promise-based API を利用
  try {
    const response = await fetch(url);
    yield response;
    const text = await response.text();
    yield text;
  } catch (err) {
    console.error(err);
    yield err;
  } finally {
    console.log("非同期ジェネレータ関数の処理を終了");
  }
}

(async () => {
  const endpoint = "https://api.github.com/zen";
  const asyncIterator = asyncGenFn(endpoint);
  const res = (await asyncIterator.next()).value;
  console.log(res);
  const text = (await asyncIterator.next()).value;
  console.log(text);
  await asyncIterator.next();
})();
```

非同期ジェネレータ関数はこのように async/await の知識さえあれば、あとはイテラブルとジェネレータを理解するだけで理解できます。

# 型注釈と型定義

第４章の最終チャプター『[TypeScript における Promise の型注釈](j-epasync-ts-promise-type-annotation#ジェネリクス)』において TypeScript について学んだら、この項目に戻ってきてみてください。理解できるようになっているはずです。

## イテレータとイテラブルの型注釈

イテレータとイテラブルの型注釈にはそれぞれ専用の型が存在しています。『[TypeScript における Promise の型注釈](j-epasync-ts-promise-type-annotation#ジェネリクス)』のチャプターで見たように `Promise<Type>` と同じくジェネリクスで型が定義された `Iterator<Type>` と `Iterable<Type>` が存在しているのでこの型を使って型注釈を行います。

とはいえ、自分でイテレータやイテラブルを作ることはあまりないかもしれませんが、自分で再び作成してみると型について理解できます。

JavaScript で反復子プロトコル(iterator protocol)と反復可能プロトコル(iterable protocol)を両方実装しているオブジェクトを定義すると次のようになりました。

```js:JavaScript
const iterableObject = {
  [Symbol.iterator]() { 
    let count = 0;

    const iterator = {
      next() { 
        const iteratorResult = (count < 3)
          ? { value: ++count, done: false }
          : { value: count, done: true };

        // イテレータリザルトを返す
        return iteratorResult;
      }
    };
    // イテレータを返す
    return iterator;
  }
};
```

これと `Iterator<Type>` と `Iterable<Type>` という２つのジェネリクス型を使って型注釈してみます。この２つの型の型引数は省略できませんので、何らかの型を指定しておく必要があります。型引数には `next()` メソッドから返ってくるイテレータリザルトの `value` プロパティの値の型を指定します。

上のコードでは、`value` の値は数値だったので `number` 型を型引数として両方に指定した上で型注釈します。また、イテレータから返るイテレータリザルトのオブジェクトにも専用のジェネリクス型 `IteratorResult<Type>` が存在しているのでそれにも `number` 型を型引数として指定します。

```ts:TypeScript
const iterable: Iterable<number> = {
  [Symbol.iterator](): Iterator<number> {
    // このメソッドからはイテレータが返るので
    // 返り値の型注釈は Iterator<number>
    let count = 0;

    const iterator: Iterator<number> = {
      next(): IteratorResult<number> {
        // value プロパティの値の方は number
        const iteratorResult: IteratorResult<number> = (count < 3)
          ? { value: ++count, done: false }
          : { value: undefined, done: true };
        
        return iteratorResult;
      }
    };
    return iterator;
  }
};
```

このようにイテラブルについてはしっかり型注釈をしておかないと型エラーになります。ただしこの場合は、メソッドの返り値まで型注釈してしまうのは冗長なので、省略しておきます。

```ts:メソッドの返り値の型注釈は省略
// イテラブルオブジェクトには Iterable 型
const iterable: Iterable<number> = {
  [Symbol.iterator]() {
    let count = 0;

    // イテレータには Iterator 型
    const iterator: Iterator<number> = {
      next() {
        // イテレータリザルトには IteratorResult 型
        const iteratorResult: IteratorResult<number> = (count < 3)
          ? { value: ++count, done: false }
          : { value: undefined, done: true };
        
        return iteratorResult;
      }
    };
    return iterator;
  }
};
```

もう少し具体的な使い方を見てから型定義について考えましょう。例えば、イテラブルオブジェクトは spread 構文の利用ができたので、イテラブルオブジェクトを引数にとって配列として変換して返すようなジェネリック関数(generic function)の型注釈などは次のようになります。

```ts
// ジェネリック関数
function arrayConverter<Type>(
  param: Iterable<Type> // 型変数でリンク
): Type[] { // 型変数でリンク
  return [...param];
  // イテラブルオブジェクトは spread 構文が使える
}

const str = "ABC";
// 文字列はイテラブルオブジェクトなので引数として渡せる
const strArray = arrayConverter<string>(str);
// typeof 型演算子で型を抽出してみる
type StringArray = typeof strArray;
// string[] の型が抽出される
```

:::message alert
ジェネレータ(generator)やジェネレータ関数(generator function)と、ジェネリクス(generics)やジェネリクス関数(generics function)は名前が似ていますが、関係の無い別物なので注意してください。
:::

## イテレータとイテラブルの型定義

さて、こういったイテラブルに関する型定義は `lib.es2015.iterable.d.ts` に記載されています。ちなみに、TypeScript から提供される ECMAScript のビルトインオブジェクトやビルトインメソッドなどの型定義は次のリポジトリから閲覧できます。

https://github.com/microsoft/TypeScript/tree/main/lib

```ts:lib.es2015.iterable.d.ts
interface IteratorYieldResult<TYield> {
    done?: false;
    value: TYield;
}

interface IteratorReturnResult<TReturn> {
    done: true;
    value: TReturn;
}

type IteratorResult<T, TReturn = any> = IteratorYieldResult<T> | IteratorReturnResult<TReturn>;

interface Iterator<T, TReturn = any, TNext = undefined> {
    // NOTE: 'next' is defined using a tuple to ensure we report the correct assignability errors in all places.
    next(...args: [] | [TNext]): IteratorResult<T, TReturn>;
    return?(value?: TReturn): IteratorResult<T, TReturn>;
    throw?(e?: any): IteratorResult<T, TReturn>;
}

interface Iterable<T> {
    [Symbol.iterator](): Iterator<T>;
}
```

こういった型は複数の型が関連して定義されているため、一気に理解するのは難しいです。イテレータリザルト→イテレータ→イテラブルの順番にみていきます。

まずは、イテレータが持つ `next()` メソッド返されるイテレータリザルトのオブジェクトの型を見てみましょう。

```ts
let count = 0;

// イテレータ(iterator protocol を満たすオブジェクト)
const iterator = {
  next() {
    // イテレータリザルトは IteratorResult 型
    // value プロパティの値が数値なので型引数に number を指定
    const iteratorResult: IteratorResult<number> = (count < 3)
      ? { value: ++count, done: false }
      : { value: undefined, done: true };
    return iteratorResult;
    // イテレータリザルトを返す
  },
};

console.log(iterator.next());
// => { value: 1, done: false }
console.log(iterator.next());
// => { value: 2, done: false }
console.log(iterator.next());
// => { value: 3, done: false }
console.log(iterator.next());
// => { value: undfined, done: ture }
```

型エイリアスで定義された `IteratorResult<T, TReturn = any>` 型には実は２つの型変数が `T` と `TReturn` 使われていますが、片方の `TReturn` はデフォルト型引数として `any` が指定されているので、型注釈する際には省略できるようになっています。

```ts
// IteratorYiedlReturn と IteratorReturnResult のユニオン型
type IteratorResult<T, TReturn = any> = IteratorYieldResult<T> | IteratorReturnResult<TReturn>;
```

この型定義を見ると、`IteratorResult<T, TReturn = any>` の型は `IteratorYieldResult<T>` と `IteratorReturnResult<TReturn>` という２つの型のユニオン型であることが分かります。

ジェネリクス関数では型変数には型をリンクする機能がありましたが、型定義の際にも同じことが言えます。`IteratorResult<T, Teturn = any>` のジェネリクスに使われている型変数 `T` と `Treturn` はユニオン型を構成する２つの型である `IteratorYieldResult` と `IteratorReturnResult` の型変数としても使われています。

ユニオン型の構成要素たる２つの型の型定義は以下のようになっています。

```ts
// TYield の型は value プロパティの値の型
interface IteratorYieldResult<TYield> {
    done?: false; // optional property
    value: TYield;
}

// TReturn の型は value プロパティの値の型
interface IteratorReturnResult<TReturn> {
    done: true;
    value: TReturn;
}
```

イテレータリザルトは `IteratorResult` 型で型注釈できますが、この型は実質的に上の２つの型のどちらかなので、そのままユニオン型でも型注釈できます。

```ts
let count = 0;

const iterator = {
  next() {
    const iteratorResult: IteratorYieldResult<number> | IteratorReturnResult<undefined> = (count < 3)
      ? { value: ++count, done: false }
      : { value: undefined, done: true };
    return iteratorResult;
    // イテレータリザルトを返す
  },
};
```

２つの型のユニオン型になっているのは、反復処理が終了したときに `next()` から返るイテレータリザルトの `done` プロパティが `true` となり、`value` プロパティの値自体は `undefined` とかでいいからです。

それなら次のように定義してもよいですが、汎用性のためにオプショナルプロパティや `true` や `false` といった真偽値のリテラル型を使った２つの型の合成として定義している訳です。

```ts
interface IteratorResult<T> = { done?: boolean; value: T | undefined };
```

`T | any` や `T | undefined` よりも２つの型変数 `TYield` と `TReturn` が絡むようにした方が使いやすいです。

ここだけ見てもまだあまり意味がわからないと思うので、`next()` メソッドでイテレータリザルトを返すイテレータの型定義を見てみましょう。

```ts
interface Iterator<T, TReturn = any, TNext = undefined> {
    // NOTE: 'next' is defined using a tuple to ensure we report the correct assignability errors in all places.
    next(...args: [] | [TNext]): IteratorResult<T, TReturn>;
    return?(value?: TReturn): IteratorResult<T, TReturn>;
    throw?(e?: any): IteratorResult<T, TReturn>;
}
```

イテレータは反復子プロトコル(iterator protocol)を満たすので `next()` メソッドを持っていましたが、実は `return()` メソッドと `throw()` メソッドも持つことができます。ただし、`?` でオプショナルプロパティ(メソッド)として型定義されているので必ずしも実装する必要はありません。

また、`Iterator<Type>` と思ってものも型変数が３つあり、最初の型変数意外の２つはデフォルト型引数が指定されています。型注釈として利用する際には基本的にこの２つは省略できます。そして、最初の型変数 `T` は `next()` メソッドの返り値の型 `IteratorResult<T, TRerun>` とリンクしています。`IteratorResult` の最初の型変数 `T` には `next()` メソッドの返り値であるイテレータリザルトの `value` プロパティの値の型を指定する必要があったので、型変数 `T` でリンクしている `Iterator` の型引数にも同じ型を指定することになります。

```ts
let count = 0;

// 同じ number 型を型引数として指定する
const iterator: Iterator<number> = {
  next() {
    // 同じ number 型を型引数として指定する
    const iteratorResult: IteratorResult<number> =
      count < 3
        ? { value: ++count, done: false }
        : { value: undefined, done: true };
    return iteratorResult;
  },
};
```

イテレータの型が分かった所で、`[Symbol.iterator]()` メソッドでイテレータを返すイテラブルオブジェクトの型を `Iterable<Type>` をみていきます。とりあえずは `[Symbol.iterator]()` メソッドで上のイテレータを返すイテラブルオブジェクトの実装を行っておきましょう。

```ts
const iterableObject = {
  [Symbol.iterator]() {
    let count = 0;
    
    // イテレータ(iterator protocol を満たすオブジェクト)
    const iterator: Iterator<number> = {
      next() {
        const iteratorResult: IteratorResult<number> =
          count < 3
            ? { value: ++count, done: false }
            : { value: undefined, done: true };
        return iteratorResult;
        // イテレータリザルトを返す
      },
    };
    return iterator;
    // イテレータを返す
  }
}

// イテラブルオブジェクトは for...of 構文で反復子が可能
for (const v of iterableObject) {
  console.log(v);
}
```

イテラブルオブジェクトの型は次のように型定義されていました。

```ts
interface Iterable<T> {
    [Symbol.iterator](): Iterator<T>;
}
```

`Iterable<T>` 型のオブジェクトは `[Symbol.iterator]()` メソッドで `Itrator<T>` 型のオブジェクトを返すということが定義されています。

イテレータとイテレータリザルトの型同士の関係と同じ様に型変数 `T` でリンクさせていますね。従って、イテレータにイテレータリザルトの `value` プロパティの値の型を指定したように、さらにリンクしているイテラブルの型の型引数にもその値の型を指定することになります。

イテラブルオブジェクトの型注釈は以下のようになります。

```ts
// 同じ number 型を型引数として指定する
const iterableObject: Iterable<number> = {
  [Symbol.iterator]() {
    let count = 0;

    // 同じ number 型を型引数として指定する
    const iterator: Iterator<number> = {
      next() {
        // 同じ number 型を型引数として指定する
        const iteratorResult: IteratorResult<number> =
          count < 3
            ? { value: ++count, done: false }
            : { value: undefined, done: true };
        return iteratorResult;
      },
    };
    return iterator;
  },
};

for (const v of iterableObject) {
  console.log(v);
}
```

型変数がカスケードするようになっているため、`number` という型引数が上から下のすべての場所で使われています。

これでイテラブルオブジェクトの型注釈が完成です。型定義も１つずつ見ていけばおそるるに足りません。

## ジェネレータ関数の型注釈と型定義

ジェネレータ関数の型注釈はまずはジェネレータ関数から返るジェネレータオブジェクトの型定義を知る必要があります。ジェネレータオブジェクトもイテラブルオブジェクトのように `Generator<Type>` というジェネリクス型が存在しています。ジェネリクスとジェネレータは名前が似ていますが別の単語であり、それぞれに概念的な関係は無いので注意してください。

`Generator<Type>` の型定義は `lib.es2015.generator.ts` に存在しています。

```ts:lib.es2015.generator.ts
interface Generator<T = unknown, TReturn = any, TNext = unknown> extends Iterator<T, TReturn, TNext> {
  next(...args: [] | [TNext]): IteratorRestult<T, TReturn>;
  return(value: TReturn): IteratorResult<T, TReturn>;
  throw(e: any): IteratorResult<T, TReturn>;
  [Symbol.iterator](): Generator<T, TReturn, TNext>;  
}
```

結構複雑に見えますが、１つずつ見ていけばそこまで難しいものではありません。まずは、`extends` ですが、これは型の拡張(extention)です。

https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#differences-between-type-aliases-and-interfaces

オブジェクトの型を拡張してみると分かりやすいですが、拡張元のオブジェクトのプロパティを持つような型を新しく定義できます。

```ts
interface Animal {
  name: string;
}
// Animal 型を拡張した Bear 型(この場合はより詳細にした)
interface Bear extends Animal {
  honey: boolean;
}

const abstractAnimal = {
  name: "動物",
};
const bear: Bear = {
  name: "熊",
  honey: true,
}
```

ということで、`Generator` 型は `Iterator` 型を拡張した型となります。比べてみると所有するメソッドの型定義はほとんど同じですが、ジェネレータの方には `[Symbol.iteraotr]()` メソッドが追加されており、そこからさらにジェネレータオブジェクト型の値が返ることを理解できます。

```ts
interface Iterator<T, TReturn = any, TNext = undefined> {
    next(...args: [] | [TNext]): IteratorResult<T, TReturn>;
    // optional method
    return?(value?: TReturn): IteratorResult<T, TReturn>; 
    // optional method
    throw?(e?: any): IteratorResult<T, TReturn>; 
}

// Genrator は Iterator 型の拡張
interface Generator<T = unknown, TReturn = any, TNext = unknown> extends Iterator<T, TReturn, TNext> {
    next(...args: [] | [TNext]): IteratorResult<T, TReturn>;
    return(value: TReturn): IteratorResult<T, TReturn>;
    throw(e: any): IteratorResult<T, TReturn>;
    // Generator 型のオブジェクトが返る
    [Symbol.iterator](): Generator<T, TReturn, TNext>;
}
```

ジェネレータオブジェクトはイテレータであり、イテラブルでもあったので、このようにイテレータリザルトを返す `next()` メソッドを持ち、イテレータを返す `[Symbol.iterator]()` メソッドの両方を実装する型となっています。

型定義についてより具体的に見ていきましょう。

`Generator<T = unknown, TReturn = any, TNext = unknown>` というようにジェネリクス型としてインターフェイスで定義されてるので、この `T`、`TReturn`、`TNext` はそれぞれ型引数です。何を指定するかは以下のようになっています。

- `T` : `yield` する値の型
- `TReturn` : `return` する値の型
- `TNext` : `next` メソッドの引数にタプルを与える場合の型

イテレータとイテラブルの型定義でみたものとまったく同じです。

このようなジェネリクス型による型定義によって、型引数が内部的にリンクしていることが分かります。トップにあるインターフェイスで定義された `Geneartor<T = unknown, TReturn = any, TNext = unknown>` で定義された３つの型変数 `T`、`Teturn`、`TNext` は内部で型定義されている `next()` や `return()` メソッドの型定義に使われている `IteratorResult<Tyep>` などにカスケードして利用されています。

基本的にはイテレータリザルトの `value` プロパティの値の型となる第一型引数のみを気にすればよいです。

例えば、JavaScript で以下のようなジェネレータ関数があったとします。
```js:JavaScript
function* genFn(n) {
  n++;
  yield n;
  n++;
  yield n;
  n++;
  yield n;
}
for (const v of genFn(0)) console.log(v);
```

ジェネレータ関数からはジェネレータオブジェクトが返るので、このジェネレータ関数を型注釈する際には、以下のように返り値の型は `Generator` とできます。この場合はデフォルト型引数どおり `Generator<unknown, any, unknown>` として型注釈を行ったことになります。

```ts:TypeScript
function* genFn(
  n: number
): Generator { // Generator<unknown, any, unknown> となる
  n++;
  yield n;
  n++;
  yield n;
  n++;
  yield n;
}
```

`function*` というようにジェネレータ関数として定義しているので、この型注釈をしなくても型推論されます。というのも、`Geneartor<T = unknown, TReturn = any, TNext = unknown>` というように３つの型変数にはすべてデフォルト型引数が指定されています。通常、`Array<Type>` や `Promise<Type>` といったジェネリクス型は型引数を指定する必要がありましたが、ジェネレータ型の型変数に対してはすべてデフォルト型引数が指定されているので型引数を省略して型注釈をすることが可能です。

というか、この場合は型注釈を省略して型推論させた方がましです。`yield` される値の型から `Generator<T, TReturn, TNext>` の第一型引数として `T` に指定されるべき型が推論されるので、ほとんど型の情報が無いに等しいからです。

```ts
function* genFn(
  n: number
) { // Genrator<number, void, unknown> として推論される
  n++;
  yield n; // number 型
  n++;
  yield n; // number 型
  n++;
  yield n; // number 型
}
```

明示的に `yield` される値の型を指定したいなら、その型を `Genrator` 型の第一型引数に指定します。

```ts
function* genFn(
  n: number
): Generator<number> { // Genrator<number, void, unknown> として推論される
  n++;
  yield n; // number 型
  n++;
  yield n; // number 型
  n++;
  yield n; // number 型
}
```

ジェネレータ関数から `yield` される値の型が同一のものでないなら、もちろんリテラル型を型引数に指定する必要があります。

```ts
function* genMultiType(): Generator<number | string | boolean> { // 第一型引数をリテラル型に
  yield 42; // number 型
  yield "ABC"; // string 型
  yield true; // boolean 型
}
```

このような場合に `Genrator` の型注釈を省略してしまうと、ジェネレータ型の第一型引数の型がそれぞれの値のリテラル型のユニオン型として型推論されてしまいます。

```ts
// Generator<true | 42 | "ABC", void, unknown> として型推論される
function* genMultiType() {
  yield 42;
  yield "ABC";
  yield true;
}
```

## 非同期ジェネレータ関数の型注釈と型定義

ジェネレータ関数と同様に非同期ジェネレータ関数からは非同期ジェネレータオブジェクトというオブジェクトが返ってきます。これにも `AsyncGenerator` という型がジェネリクスで定義されています。非同期ジェネレータ関数や非同期イテレータなどは ES2018 で追加された機能ということで、`lib.es2018.**.d.ts` に型定義されています。

```ts:lib.es2018.asyncgenerator.d.ts
interface AsyncGenerator<T = unknown, TReturn = any, TNext = unknown> extends AsyncIterator<T, TReturn, TNext> {
    // NOTE: 'next' is defined using a tuple to ensure we report the correct assignability errors in all places.
    next(...args: [] | [TNext]): Promise<IteratorResult<T, TReturn>>;
    return(value: TReturn | PromiseLike<TReturn>): Promise<IteratorResult<T, TReturn>>;
    throw(e: any): Promise<IteratorResult<T, TReturn>>;
    [Symbol.asyncIterator](): AsyncGenerator<T, TReturn, TNext>;
}
```

非同期イテラブルや非同期イテレータも同様に専用の型があり、上の `AsyncGenerator` の定義に使われていますね。

```ts:lib.es2018.asyncIterable.d.ts
interface AsyncIterator<T, TReturn = any, TNext = undefined> {
    // NOTE: 'next' is defined using a tuple to ensure we report the correct assignability errors in all places.
    next(...args: [] | [TNext]): Promise<IteratorResult<T, TReturn>>;
    return?(value?: TReturn | PromiseLike<TReturn>): Promise<IteratorResult<T, TReturn>>;
    throw?(e?: any): Promise<IteratorResult<T, TReturn>>;
}

interface AsyncIterable<T> {
    [Symbol.asyncIterator](): AsyncIterator<T>;
}
```

通常のイテレータ `Iteartor` やジェネレータオブジェクト `Generator` の型定義と大差ありません、次のように比較してみれば分かりますが、ただ中身がそれぞれ `Promise<Type>` でラップされているだけです。

```ts
interface Iterator<T, TReturn = any, TNext = undefined> {
    next(...args: [] | [TNext]): IteratorResult<T, TReturn>;
    return?(value?: TReturn): IteratorResult<T, TReturn>; 
    throw?(e?: any): IteratorResult<T, TReturn>; 
}
interface Iterable<T> {
    [Symbol.iterator](): Iterator<T>;
}
interface Generator<T = unknown, TReturn = any, TNext = unknown> extends Iterator<T, TReturn, TNext> {
    next(...args: [] | [TNext]): IteratorResult<T, TReturn>;
    return(value: TReturn): IteratorResult<T, TReturn>;
    throw(e: any): IteratorResult<T, TReturn>;
    [Symbol.iterator](): Generator<T, TReturn, TNext>;
}

interface AsyncIterator<T, TReturn = any, TNext = undefined> {
    next(...args: [] | [TNext]): Promise<IteratorResult<T, TReturn>>;
    return?(value?: TReturn | PromiseLike<TReturn>): Promise<IteratorResult<T, TReturn>>;
    throw?(e?: any): Promise<IteratorResult<T, TReturn>>;
}
interface AsyncIterable<T> {
    [Symbol.asyncIterator](): AsyncIterator<T>;
}
// AsyncGenerator は AsyncIterator 型の拡張
interface AsyncGenerator<T = unknown, TReturn = any, TNext = unknown> extends AsyncIterator<T, TReturn, TNext> {
    next(...args: [] | [TNext]): Promise<IteratorResult<T, TReturn>>;
    return(value: TReturn | PromiseLike<TReturn>): Promise<IteratorResult<T, TReturn>>;
    throw(e: any): Promise<IteratorResult<T, TReturn>>;
    [Symbol.asyncIterator](): AsyncGenerator<T, TReturn, TNext>;
}
```

通常のジェネレータオブジェクトと変わらず、とりあえずは `yield` で返る値の型としての第一型引数だけ気にしておけばよいでしょう。非同期ジェネレータ関数の型注釈はジェネレータ関数とほとんど変わりませんし、考え方も同じです。

```ts
const endpoint = "https://api.github.com/zen";

async function* asyncGen(
  url: string
) { // AsyncGenerator<string, void, unknown> となる
  const res = await fetch(url);
  const text: string = await res.text();
  yield text;
  yield "Github says..." + text;
  yield "I restpect" + text;
}

// 非同期ジェネレータ関数をイテレーションするなら for await...of
for await (const v of asyncGen(endpoint)) {
  console.log(v);
}
```

明示的に型注釈するならこうです。

```ts
async function* asyncGen(
  url: string
): AsyncGenerator<string> { 
  const res = await fetch(url);
  const text: string = await res.text();
  yield text;
  yield "Github says..." + text;
  yield "I restpect" + text;
}
```

ジェネレータ関数など使う機会がそこまで多くないかもしれませんが、ビルトインメソッドやビルトインオブジェクトの型定義などを調べて理解するプロセスが分かったと思うので、こういった見方で他のビルトインの型定義も理解できるはずです(型定義についてはもちろん JavaScript での書き方を正しく知っておくことも重要です)。
