---
title: "イテレータとイテラブルとジェネレータ関数"
---

# このチャプターについて

このチャプターでは非同期処理が絡む反復処理に利用するイテレータやイテラブル、ジェネレータ関数などを解説しておきます。

`Promise.all()` などのメソッドも配列ではなくて実はイテラブルなオブジェクトを引数として受け入れているに過ぎません。

一見難しそうですが、「プロトコル」のことが理解できればイテレータとイテラブルは理解できますし、イテレータとイテラブルが理解できればジェネレータ関数も理解できます。非同期ジェネレータ関数もジェネレータ関数と今までの非同期処理が理解できていれば割と簡単に理解できますので、恐れずに進みましょう。

:::message alert
イテレータ(iterator)やイテラブル(iterable)、イテレーション(iteration)などややこしい単語がでてくるのでそこだけは注意してください。
:::

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
// イテレータ(iterator ptorocol を満たすオブジェクト)
const iterator = {
  next() { // メソッドの短縮記法
    const iteratorResult = { 
      value: 42, 
      done: false 
    };
    return iteratorResult;
    // イテレータリザルトを返す
  }
}

console.log(iterator.next());
// => { value: 42, done: false }
```

これだけです。これで何ができるかはもう１つのプロトコルをみないとイマイチ分かりません。

ちなみにメソッドの定義方法は以下のような書き方があります。上の書き方はメソッドの短縮記法と呼ばれるものです。

```js:メソッドの定義方法
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
        const iteratorResult = {
          value: 42,
          done: false
        };
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

# イテレータとイテラブルの利用

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

イテラブルオブジェクトの `[Symbol.iterator]()` メソッドを実行するとイテレータが返ってきました。さらにイテレータは `next()` メソッドを持っており、このメソッドを実行すると `value` と `done` プロパティを持つイテレータリザルトというオブジェクトが返ってきます。

```js
// [Symbol.iteartor]() メソッドを実行するとイテレータが返ってくる
const iterator = iterableObject[Symbol.iterator]();

// next() メソッドを実行するとイテレータリザルトが返ってくる
console.log(iterator.next()); //  => { value: 1, done: false } 
console.log(iterator.next()); //  => { value: 2, done: false }
console.log(iterator.next()); //  => { value: 3, done: false }
console.log(iterator.next()); // => { value: 3, done: true }
console.log(iterator.next()); //  => { value: 3, done: true }
```

このような感じで `next()` メソッドを次々に実行することで反復的に値をインクリメントする処理を行っていくことができます。反復処理において `value` プロパティと `done` プロパティは以下のようなものとして機能することを認識しておくと良いでしょう。

- `value` プロパティには反復処理の結果となる値
- `done` プロパティには反復処理が完了したかどうかの真偽値

特に `done` プロパティは `false` (反復処理が完了していない) から始まり、反復処理が完了すると `true` (反復処理が完了した) となって反復処理が終わったことを表現します。

## for...of

先程定義したイテラブルオブジェクトを実際に `while` ループで反復処理してみます。

```js:sympleIterable.js
const iterableObject = {
  [Symbol.iterator]() { 
    let count = 0;
    const iterator = {
      next() { 
        const iteratorResult = (count < 3)
          ? { value: ++count, done: false }
          : { value: count, done: true };
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

反復処理を行うのに以下のように色々書かないといけないのは面倒です。

```js
const myIterator = iterableObject[Symbol.iterator]();
let myIteratorResult;
while (true) {
  myIteratorResult = myIterator.next();
  if (myIteratorResult.done) break;
  console.log(myIteratorResult.value);
}
```

そこで ES2015 で追加された `for...of` の構文を利用します。この構文は上のような処理の手続きをまとめて行ってくれるのでイテレータから値を反復して取り出すのが簡単にできます。

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
          : { value: count, done: true };
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

配列がイテラブルであることから、オブジェクトのプロパティに対して反復処理を行いたいときも一度 `Object.keys()` や `Object.values()` などのオブジェクトの静的メソッドを使って一旦配列にすることで `for...of` で反復処理ができるようになります。

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

# ジェネレータ関数

ジェネレータ関数(generator function)は関数内の処理の一時停止が可能な関数です。ジェネレータ機能はイテラブルやイテレータをサポートするものです。

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

イテラブルなので、spread 構文や分割代入もできます。

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
yield await Promsie.resolve(2);表現していると捉えられます。

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

反復可能プロトコル(itrable protocol)を満たすオブジェクトは `[Symoble.iterator]()` メソッドによってイテレータリザルトが返ってきましたが、非同期反復可能プロトコル(async iterable protocol)を満たすオブジェクトは `[Symbol.asyncIterator]()` というメソッドによって Promise インスタンスによってラップされたイテレータリザルトが返ってきます。

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

# for await...of
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

# 非同期ジェネレータ関数

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

ジェネレータ関数の内部で Promsie-based API などを利用したいときはこの非同期ジェネレータ関数が役立つことになります。

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

# 参考文献

イテレータとイテラブルとジェネレータ関数についてはこちらの kura07 さんの記事でも非常にわかりやすく解説されているので参考にしてください。

https://qiita.com/kura07/items/cf168a7ea20e8c2554c6

https://qiita.com/kura07/items/d1a57ea64ef5c3de8528

