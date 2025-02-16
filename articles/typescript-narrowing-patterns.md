---
title: "Narrowing Pattern"
published: true
cssclass: zenn
emoji: "🖇"
type: "tech"
topics: ["typescript"]
date: 2022-09-01
modified: 2022-09-24
url: "https://zenn.dev/estra/articles/typescript-narrowing-patterns"
tags: [" #type/zenn "]
aliases:
  - 記事 TypeScript の Narrowing Pattern
  - Narrowingのパターン
---

# はじめに

前回の『[TypeScript の Narrowing](https://zenn.dev/estra/articles/typescript-narrowing)』の記事では Narrowing について集合論的なアプローチでどのようなものであるかを解説しました。この記事では、より実用的な Narrowing の基本パターンを解説します。

この記事は基本的には TypeScript 公式 Handbook の『[Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)』のページを参照して解説しています。集合論的な話はすでに前の記事で行ってしまったので、この記事は公式ドキュメントのまとめ的な自分用のアウトプットとなります。

:::message alert
[TypeScript v3.7](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions) で導入された Assertion 関数については応用的な話であるため、この記事では省くことにしました。
:::

# 代入による Narrowing

CFA での典型的な Narrowing パターンの解説に入る前に、もっと基本的な Narrowing について見ておきます。

前前回の記事を見て入れば Widening を知っているわけですが、実はその過程ですでに Narrowing についても知っています。というのも、ユニオン型として `let` 宣言した変数では、具体的な値を代入することでその型が確定することになるので、「[代入(Assignment)](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#assignments)」という行為も Narrowing の一種であることになります。

TypeScript は代入した際の右辺の値を見て変数の型が絞り込むことによって、その型のプロトタイプメソッドなどを使っても型エラーとならなくなります。ただし、`let` 宣言した変数では再代入が何度でも可能なので、再代入時には変数宣言時に使用した型注釈であるユニオン型の要素の型の値を代入できます。代入以降は再代入した値の型として見なされるので使えるプロトタイプメソッドもその型のものとなります。

```ts
/* assignment.ts */

let unionVal: string | number;
//            ^^^^^^^^^^^^^^^ ユニオン型として型注釈

unionVal = 1.1; // 代入の行ではエディタ上ではユニオン型

unionVal; // :number (代入以降は number 型として見なされる)
// 宣言された型はユニオン型なので要素になっている型の値を代入可能
console.log(unionVal.toPrecision(4)); // => 1.100
//          ^^^^^^^^: number

// let 宣言の変数は変数宣言時のユニオン型の値を代入可能
unionVal = "str"; // 代入の行ではエディタ上ではユニオン型

unionVal; // :string (代入以降は string 型として見なされる)
// 宣言された型はユニオン型なので要素になっている型の値を代入可能
console.log(unionVal.toUpperCase()); // => STR
//          ^^^^^^^^: string
```

実際にそれぞれの行にエディタ上でカーソルを当てると、`1.1` という数値を代入した後の変数 `unionVal` では `number` 型として型が絞り込まれていることが確認できます。

![エディタ上での表示1](/images/typescript-widen-narrow/img_narrow_assignment1.jpg)

`"str"` という文字列を代入した後の変数 `unionVal` では `string` 型として型が絞り込まれていることも確認できます。

![エディタ上での表示2](/images/typescript-widen-narrow/img_narrow_assignment2.jpg)

また const アサーションによって Widening を抑止するのも Narrowing の一種です。

```ts
const animal = {
  name: "bear",
};
type Animal = typeof animal;
// { name: string; } 型が抽出される

const vehicle = {
  name: "bike" as const, // Narrwoing (Widening を抑制するように文字列リテラル型としてアサーション)
};
type Vehicle = typeof vehicle;
// { name: "bike"; } 型が抽出される
```

# キーワードを使った Narrowing

以下であげるような話題は Narrowing(型の絞り込み) よりも、Type guard(型ガード) という話題で解説されることが多いですが、Narrowing という目的に沿って解説した方が公式ドキュメントにも沿っているのでそうします。

### typeof 演算子を使った Narrowing

`typeof` 演算子によって変数の型を基本的な判定ができます。`typeof` 演算子で判定できるものは以下のような基本的な型となります。

- `"string"`
- `"number"`
- `"bigint"`
- `"boolean"`
- `"symbol"`
- `"undefined"`
- `"object"` (`null` も `"object"` として評価される)
- `"function"` (関数)

`typeof` 演算子は JavaScript の機能です。MDN で `typeof` 演算子によって返される値がリストアップされています。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/typeof

:::message
配列であるかどうかの判定は後述する `Array.isArray()` という静的メソッドを型ガードとして利用することで可能です。
:::

実際に Narrowing する際には `if` 文や `switch` で利用します。

```ts
function testPrimitiveUnion(
  param: string | number | boolean
) {
  if (typeof param === "string") {
    // 変数 param は CFA で string 型として解析される
    console.log(param.toUpperCase());
    //          ^^^^^: string
  } else if (typeof param === "number") {
    // 変数 param は CFA で number 型として解析される
    console.log(param.toPrecision(4));
    //          ^^^^^: number 型
  } else if (typeof param === "boolean") {
    // 変数 param は CFA で boolean 型として解析される
    console.log(param.toString());
    //          ^^^^^: boolean 型
  } else {
    // このブランチではユニオン型の候補がすべてなくなったため never (空集合) となる
    // 変数 param は CFA で never 型として解析される
    console.log(param);
    //          ^^^^^: never 型(決して観測されない)
  }
}
```

このような関数を使って値をテストすると次のようになります。

```ts
testPrimitiveUnion("text");
testPrimitiveUnion(42);
testPrimitiveUnion(false);

/*
変数は string 型として Narrowing されている
TEXT
変数は number 型として Narrowing されている
42.00
変数は boolean 型として Narrowing されている
false
 */
```

このように CFA で型を解析できるように `typeof` 演算子などを使って型を Narrowing する箇所やその行為そのものを型ガード(Type guard)と呼びます。特に `typeof` の場合は typeof 型ガードと呼びます。

```ts
if (typeof param === "string") { // typeof 型ガード
  console.log("変数は string 型として Narrowing されている");
  console.log(param.toUpperCase());
  //          ^^^^^: string
}
```

１つの行で複数の条件を組み合わせることでも Narrowing できます。

```ts
const strOrNum = Math.random() < 0.5 ? "text" : 42;
//    ^^^^^^^^ "text" | 42 というリテラル型のユニオン型

const length = (typeof strOrNum === "string" && strOrNum.length) || strOrNum;
```

プリミティブ値などではこのようにうまくいきますが、`typeof` 演算子では、オブジェクト型の判定はうまく機能しません。というのも JavaScript では、`typeof null` が `"object"` として判定されてしまうからです([過去の仕様の負債](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/typeof#typeof_null)です)。

```ts
const objOrNull = Math.random() < 0.5 ? { a: 42 } : null;

if (typeof objOrNull === "object") {
  // { a: 42 } でも null でも判定が通ってしまう
  console.log(objOrNull);
  //          ^^^^^^^^^: { a: number; } | null (ユニオン型)
}
```

このような場合には後述する Truthiness Narrowing が必要となります。

### instanceof 演算子を使った Narrowing

JavaScript には `instanceof` 演算子というものがありますが、これを型ガードとしても利用できます。変数がクラスのインスタンスであるかを判別することに利用します。

```ts
let today = Math.random() < 0.5
  ? new Date()
  : "2022/07/30";

if (today instanceof Date) {
  // today は Date インスタンス
  console.log(today.toUTCString());
} else {
  // today は string 型
  console.log(today);
}
```

### in 演算子を使った Narrowing

JavaScript の `in` 演算子を使えばオブジェクトが特定のメソッドやプロパティを持っていることを判別するができます。型の Narrowing というようりは、オブジェクトの shape を確定していく作業です。

例えば、`data` オブジェクトに `error` というプロパティがあるかどうかを判別するには以下のように型ガードの条件として利用して絞り込みます。

```ts
if ("error" in data) { // 型ガード
  // このブランチ内では data オブジェクトが error プロパティを持つことが保証される
  data; // { error: ... }
}
```

オブジェクト型のユニオン型などを考えるときには型の絞り込みとして型ガードに利用できます。

```ts
type Fish = { swim: () => void };
type Bird = { fly: () => void };
type Human = {
  walk: () => void;
  swim?: () => void;
  fly?: () => void;
};

function move(
  animal: Fish | Bird | Human
): void {
  if ("walk" in animal) {
    // Human 型に絞り込まれる
    animal.walk();
  } else if ("fly" in animal) {
    // Bird 型に絞り込まれる
    animal.fly();
  } else {
    // Fish 型に絞り込まれる
    animal.swim();
  }
}
```

# Truthiness Narrowing

`typeof null === "object"` のような判定がされてしまうことから、Truthiness check (真実性チェック) が必要になってきます。JavaScript では `if` 分の条件式では強制的かつ暗黙的に真偽値へと型変換が行われて評価が行われます。

```ts
if (obj) { // obj は真偽値へと変換されて評価される
  // obj が true 評価ならこの節の処理が行われる
}
```

強制的に変換された結果として `false` になるものは **[falsy](https://developer.mozilla.org/ja/docs/Glossary/Falsy)**、`true` になるものは **[truthy](https://developer.mozilla.org/ja/docs/Glossary/Truthy)** と呼ばれます。

truthy なものは無限にありますが、falsy なものは限られていることから、falsy でないなら truthy というように考えます。falsy な値は以下のものですべてです。

- `false`
- `0`
- `-0` (マイナスゼロ)
- `0n` (0 の bigint バージョン)
- `""` (空文字列)
- `null`
- `undefined`
- `NaN`

https://developer.mozilla.org/ja/docs/Glossary/Falsy

これらのものは `false` として評価されて、`if` ブランチのコードは実行されません。

```js
/* falsy な値の評価 */
if (false) {/* 実行されない */}
if (0) {/* 実行されない */}
if (-0) {/* 実行されない */}
if (0n) {/* 実行されない */}
if ("") {/* 実行されない */}
if (null) {/* 実行されない */}
if (undefined) {/* 実行されない */}
if (NaN) {/* 実行されない */}
```

逆にこれら以外のすべては `ture` であると評価され、ブロック内のコードが実行されます。以下のようなあやしい値もすべて Truthy なので `true` と評価されます。

```js
/* truty な値の評価 */
if ({}) {/* 実行される */}
if ([]) {/* 実行される */}
if ("0") {/* 実行される */}
if ("false") {/* 実行される */}
if (new Date()) {/* 実行される */}
if (Infinity) {/* 実行される */}
if (-Infinity) {/* 実行される */}
```

`if` の条件式で評価せずとも、`!!` という[二重否定の演算子](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Logical_NOT#%E4%BA%8C%E9%87%8D%E5%90%A6%E5%AE%9A_!!)(Double NOT)を付けることであらゆる値を強制的に真偽値へと変換できます。

```ts
// 以下すべて false という真偽値リテラル型として型推論される
const flasy0 = !!false;
//    ^^^^^^: false リテラル型
const falsy1 = !!0;
//    ^^^^^^: false リテラル型
const flasy2 = !!(-0);
//    ^^^^^^: false リテラル型
const falsy3 = !!0n;
//    ^^^^^^: false リテラル型
const falsy4 = !!"";
//    ^^^^^^: false リテラル型
const falsy5 = !!null;
//    ^^^^^^: false リテラル型
const falsy6 = !!undefined;
//    ^^^^^^: false リテラル型

const flasy7 = !!NaN;
//    ^^^^^^: boolean 型
console.log(flasy7); // => flase


// truty な値の二重否定なら true が返されれる
const truty1 = !!42;
//    ^^^^^^: true リテラル型
```

この変換自体は上記の truhy/falsy による評価によって行われます。`!!tuthy` は `true` を返し、`!!falsy` は `false` を返します。ただし、`NaN` の二重否定は `false` という値自体は返しますが、変数の型は `boolean` 型として型推論されます。

`null` や `undefined` などが絡む際には、変数の値が falsy かどうかのチェック、つまり Truthiness narrowing をして型を絞り込みます。この際に 0 や空文字列が falsy であることが厄介です。

例えば、次のような単純な型ガードを行って CFA で型解析させてもうまくいきません。

```ts
function isStrOrArrNG(
  param: string | string[] | null
) {
  if (param) { // truthiness narrowing
    // param の値が truty ならこのブランチの処理を行う
    if (typeof param === "string") {
      console.log(param, ": truty & string");
    } else {
      console.log(param, ": truty & string[]");
    }
  } else {
    // 空文字 "" と null は flasy なので truthiness narrowing で弾かれる
    console.log(param, ": falsy");
    //          ^^^^^: string | null
  }
}
isStrOrArrNG(["a", "b"]); // => [ "a", "b" ] : truty & string[]
isStrOrArrNG("test"); // => test : truty & string
isStrOrArrNG(null); // => null : falsy

// 空文字列が falsy として評価されてしまう
isStrOrArrNG(""); // =>  : falsy
```

次のように型ガードを構成することでうまく機能するようになります。

```ts
function isStrOrArrOk(
  param: string | string[] | null
) {
  if (param && typeof param === "object") {
    // param の値が truty かつ object 型の範疇ならこのブランチの処理を実行
    console.log(param, ": string[]");
    //          ^^^^^: string[] 型
  } else if (typeof param === "string") {
    // param が string 型ならこのブランチの処理を実行
    console.log(param, ": string");
    //          ^^^^^: string 型
  } else {
    // どの型ガードにも引っかからないならこのブランチの処理を実行
    console.log(param, ": falsy");
    //          ^^^^^: null 型
  }
}

isStrOrArrOk(["a", "b"]); // => [ "a", "b" ] : string[]
isStrOrArrOk("test"); // => test : string
isStrOrArrOk(null); // => null : falsy

// 空文字列もしっかりと文字列として判定できる
isStrOrArrOk(""); // =>  : string
```

上記関数では Truthiness Narrowing を `&&` 演算子で `typeof param === "object"` という typeof 演算子による Narrowing を組み合わせて使っています。

`null` は truthy ではないので、最初の `if (param ...)` のブランチで弾かれます。さらに、次のブランチの `typeof param === "string"` でも弾かれます。空文字列(`""`)は falsy なので最初の `if (param ...)` のブランチで弾かれますが、次のブランチでは `string` 型なので受け入れらています。

上記の関数では、関数の引数の型が `string | string[] | null` というユニオン型だったので、Truty かつ `typeof` 演算子による判定が `"object"` なら配列しかありえませんので、このような絞り込みができましたが、配列かどうかのより汎用的な判定は `Array.isArray()` という静的メソッドを型ガードとして利用することで可能です。

```ts
const strArrOrNumber = Math.random() < 0.5 ? ["A", "B"] : 42;
//    ^^^^^^^^^^^^^: string[] | 42 ユニオン型

if (Array.isArray(strArrOrNumber)) { // 型ガード
  // このブランチ内では配列であると絞り込まれる
  console.log(strArrOrNumber);
  //          ^^^^^^^^^^^^^^: string[]
} else {
  // string[] でないなら 42 数値リテラル型
  console.log(strArrOrNumber);
  //          ^^^^^^^^^^^^^^: 42
}
```

このような静的メソッドを使っても CFA で解析できるので、if ステートメントのブランチ内部では、`param` は配列型であると解析されて、型エラーとはならずにすみます。

そして、`Array.isArray()` はビルトインメソッドであり、配列の静的メソッドですが、**型ガード関数**(Type guard function)として機能しています。型ガード関数は **Type predicate** という**特殊な返り値の型注釈**を施した上で真偽値を返す関数として定義することで自作することもできます。

# Equality Narrowing

以下のような等価演算子や不等価演算子での判定を型ガードに使って Narrowing することも可能です。こういった型の絞り込み方法を Equality Narrowing と呼びます。日本語なら「等価性による型の絞り込み」といったところでしょうか。

- `===`
- `!==`
- `==`
- `!=`

今までやってきた `typeof` 型ガードによる方法やタグ付きユニオン型における絞り込みも実はこれを駆使していました。

```ts
if (typeof param === "string") {
  //             ^^^ 厳密等価演算子を使った型の絞り込み
  // 変数 param は CFA で string 型として解析される
  console.log(param.toUpperCase());
  //          ^^^^^: string
}
```

上の内容だけなら、`typeof` 型ガードについての解説だけで完結するところですが、Equality Narrowing についての特筆すべき例の１つは、２つの変数の比較ができる点にあります。

例えば、以下のように２つの変数を受け取る関数内で２つの変数の値が同じであるときは両者が `string` 型であって、さらに値が同じときに限ります。従って、`x === y` という型ガードが行われる `if` のブランチでは２つの変数 `x` と `y` は `string` 型であることが推論できます。実際にそのブランチの中では両者の型が `string` 型であると絞り込まれるので、`toUpperCase()` といった文字列のプロトタイプメソッドが利用できます。

```ts
function acceptTowParam(
  x: string | number,
  y: string | boolean
): void {
  if (x === y) {
  //  ^^^^^^^ x === y となるのはそれぞれが string 型のときのみ
    console.log(x.toUpperCase(), y.toUpperCase());
    //          ^: string 型     ^: string 型
  } else {
    console.log(x, y);
    //          ^  ^: string | number 型
  }
}
```

「２つの変数の値が等しいなら型そのものの等しいはずだ」という理屈です。単なる等価演算子(`==`)だと暗黙的な型変換が行われて `x = 1` (`number` 型)と `y = "1"` (`string` 型)の場合などに `true` という評価になってしまうため、厳密等価演算子を使っています。

等しいかどうか、あるいは等しくないかどうかの判定を型ガードとして利用することで `if/else` のブランチ内で型の絞り込みが可能となります。

「等しくない」場合で有用なのが、`null` というリテラル型を型の候補から除去するタイプの Equality Narrowing です。`null` という値が発生しうる場合にはこれを使うことで安全に値を利用できるようになります。厳密不等価演算子(`!==`)を `null` に使って `null` リテラル型あるいは `null` という値そのものを除去します。

```ts
if (str !== null) {
  // このブランチ内では null ではないことが保証される
}
```

厳密(`===`、`!==`)ではなく、単なる等価演算子(`==`)や不等価演算子(`!=`)を利用することで、`null` と `undefined` の両者を型の候補から排除することができます。これは `== undefined` と `== null` の両者が `null` または `undefined` であると判定されてしまうことを利用した型の絞り込みです。Truthiness narrowing で行った方法よりも簡単に行なえます。

```ts
function test(param: number | null | undefined) {
  if (param != null) {
    // null 型と undefined 型の両方を取り除く
    console.log(param);
    //          ^^^^^: number 型
  }

  if (param != undefined) {
    // null 型と undefined 型の両方を取り除く
    console.log(param);
    //          ^^^^^: number 型
  }

  if (param == null) {
    // null 型と undefined 型の両方として絞り込まれる
    console.log(param);
    //          ^^^^^: null | undefined 型
  }

  if (param == undefined) {
    // null 型と undefined 型の両方として絞り込まれる
    console.log(param);
    //          ^^^^^: null | undefined 型
  }
}
```

# ユーザー定義型ガード関数による Narrowing

『Truthiness Narrowing』の項目最後で見た `Array.isArray()` のような特定の静的メソッドは CFA において型ガードとして機能します。そのような関数を型ガード関数(Type guard function)と呼びますが、このようばビルトインのものだけではなく、自分自身で型ガード機能を持つような独自の関数を作成することもできます。

そのような関数を「**ユーザー定義型ガード関数(User-defined type guard function)**」と呼びます。ユーザー定義型ガード関数は内部的なロジックから真偽値を返す関数ですが、返り値の型注釈を特殊な書き方にすることで、それを型ガードとして使用しているブランチ内で CFA で特定の型であると解析できるように伝える特殊な関数です。

```ts
function isErrorResponse(
  obj: Response
): obj is APIErrorResponse {
// ^^^^^^^^^^^^^^^^^^^^^^^^ type predicate の型注釈
// obj は APIErrorRespnose 型であると記述する
  return obj instanceof APIErrorResponse;
}
```

返り値の型注釈を Type predicate にすることによって単に真偽値を返すだけではなく、CFA において型を絞り込んで解析できるようにしています。

:::message
Type predicate の [predicate](https://en.wikipedia.org/wiki/First-order_logic) とは日本語で言うと「述語」となります。数理論理学におけるタームから派生して利用されているようですが、ここでは**型についての情報を記述するための表記方法**だと考えればよいです。
:::

ユーザー定義型ガード関数は TypeScript v.1.6 で導入された古い機能です。『Overview』の以下の場所に記載されています。

- [User-defined type guard functions | TypeScript: Documentation - Overview](https://www.typescriptlang.org/docs/handbook/release-notes/overview.html#user-defined-type-guard-functions)

戻り値の型注釈を `x is T` というように記述することで Type Predicate となります(`x` はパラメータで、`T` は何かしらの型)。ユーザー定義型ガード関数は `if` ブロックで変数を渡して呼び出された際にはその変数の型が Narrowing されます。

```ts
type Mammals = {
  species: "mammals";
};
type Cat = Mammals & {
  name: string;
  meow: () => void;
};
type Dog = Mammals & {
  name: string;
  bow: () => void;
};

const cat: Cat = {
  name: "kitty",
  species: "mammals",
  meow: () => {
    console.log("meowmewo");
  },
};
const dog: Dog = {
  name: "snoopy",
  species: "mammals",
  bow: () => {
    console.log("bowwow");
  },
};

function isCat(
  animal: { species: string; }
): animal is Cat {
// ^^^^^^^^^^^^^ type predicate
  return (animal as Cat).meow !== undefined;
  // 実際に返しているのは真偽値だが type predicate として返り値を型注釈することで CFA で Narrowing を起こすように伝える
}

const mypet: Cat | Dog = Math.random() < 0.5
  ? cat
  : dog;

// CFA において変数の型が解析される
if (isCat(mypet)) {
  // Cat 型に Narrowing されるためその型のメソッドが使える
  mypet.meow();
//^^^^^: Cat 型
} else {
  // Dog 型に Narroing されるためその型のメソッドが使える
  mypet.bow();
//^^^^^: Dog 型
}
```

戻り値の型が単なる `boolean` 型だったり、type predicate を省略してしまうと CFA での Narrowing を行なわない単なる真偽値を返すだけの関数になってしまうので注意してください。

```ts
// ただの真偽値を返すだけの関数で型ガード関数として機能しない
function isErrorResponse(
  obj: Response
): boolean {
  return obj instanceof APIErrorResponse;
}
```

Type predicate を記述することではじめて型ガード関数となります。これは type predicate の記述によってコンパイラなどに型ガード関数であるということを認識させているようです。
