---
title: "Narrowing Pattern"
emoji: "🖇"
type: "tech"
topics: ["typescript"]
published: true
date: 2022-09-01
url: "https://zenn.dev/estra/articles/typescript-narrowing-patterns"
tags: [" #type/zenn "]
aliases: [記事_TypeScript の Narrowing Pattern]
---

# はじめに

前回の『[TypeScript の Narrowing](https://zenn.dev/estra/articles/typescript-narrowing)』の記事では Narrowing について集合論的なアプローチでどのようなものであるかを解説しました。この記事では、より実用的に Narrowing の基本パターンを解説します。

この記事は基本的には TypeScript 公式 Handbook の『[Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)』のページを参照して解説しています。

最新の公式ドキュメント(v2)は**非常にわかりやすい構成でシンプルに TypeScript を理解できるような内容**になっているので英語であっても必ず読むべきおすすめの学習リソースです。独自の集合論的な話はすでに前の記事で行ってしまったので、この記事は公式ドキュメントのまとめ的な自分用のアウトプットとなります。従って、未完成のところや理解度の低い内容を含みますので注意してください(継続的に更新していくつもりです)。

# 代入による Narrowing

CFA での典型的な Narrowing パターンの解説に入る前に、もっと基本的な Narrowing について見ておきます。

前の記事を見て入れば Widening を知っているわけですが、実はその過程ですでに Narrowing についても知っています。というのも、ユニオン型として `let` 宣言した変数では、具体的な値を代入することでその型が確定することになるので、「[代入(Assignment)](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#assignments)」も Narrowing の１つとしてカウントされます。

TypeScript は代入した際の右辺の値を見て変数の型が絞り込むことによって、その型のプロトタイプメソッドなどを使っても型エラーとならなくなります。ただし、`let` 宣言した変数では再代入が何度でも可能なので、再代入時には変数宣言時に使用した型注釈であるユニオン型の要素の型の値を代入できます。代入以降は再代入した値の型として見なされるので使えるプロトタイプメソッドもその型のものとなります。

```ts
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

- `"undefined"`
- `"string"`
- `"number"`
- `"bigint"`
- `"boolean"`
- `"symbol"`
- `"function"` (関数)
- `"object"` (`null` も `"object"` として評価される)

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
    // CFA で string 型として解析される
    console.log("変数は string 型として Narrowing されている");
    console.log(param.toUpperCase());
    //          ^^^^^: string
  } else if (typeof param === "number") {
    // CFA で number 型として解析される
    console.log("変数は number 型として Narrowing されている");
    console.log(param.toPrecision(4));
    //          ^^^^^: number 型
  } else if (typeof param === "boolean") {
    // CFA で boolean 型として解析される
    console.log("変数は boolean 型として Narrowing されている");
    console.log(param.toString());
    //          ^^^^^: boolean 型
  } else {
    // CFA で never 型として解析される
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

１つの行で複数の条件を組み合わせて Narrowing することもできます。

```ts
const strOrNum = Math.random() < 0.5 ? "text" : 42;
//    ^^^^^^^^ "text" | 42 というリテラル型のユニオン型

const length = (typeof strOrNum === "string" && strOrNum.length) || strOrNum;
```

プリミティブ値などではこのようにうまくいきますが、`typeof` 演算子では、オブジェクト型の判定はうまく機能しません。というのも JavaScript では、`typeof null` が `"object"` として判定されてしまうからです(過去の仕様の負債です)。

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

JavaScript には `instanceof` 演算子というものがありますが、これを型ガードとして利用することもできます。変数がクラスのインスタンスであるかを判別することに利用します。

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

# 🛠 Equality Narrowing

:::details 未完成
単純な(厳密)等価演算子や(厳密)不等価演算子での判定を型ガードに使って Narrowing することも可能です。

今までやってきた `typeof` 型ガードによる方法やタグ付きユニオン型における絞り込みも実はこれを駆使していました。

```ts

```
:::

# Truthiness Narrowing

`typeof null === "object"` のような判定がされてしまうことから、Truthiness check (真実性チェック) が必要になってきます。JavaScript では `if` ステートメントの条件式では強制的かつ暗黙的に真偽値へと型変換が行われて評価が行われます。

```ts
if (obj) { // obj は真偽値へと変換されて評価される
  // obj が true 評価ならこの節の処理が行われる
}
```

強制的に変換された結果として `false` になるものは **falsy**、`true` になるものは **truthy** と呼ばれます。truety なものは無限にありますが、falsy なものは限られていることから、falsy でないなら truthy というように考えます。falsy な値は以下のものです。

- `false`
- `0`
- `0n` (0 の bigint バージョン)
- `""` (空文字列)
- `null`
- `undefined`
- `NaN`

これらのものは `false` として評価されて、逆にこれら以外のすべては `ture` であると評価されます。`if` の条件式で評価せずとも、`!!` という二重否定の演算子を付けることであらゆる値を強制的に真偽値へと変換できます。

```ts
// 以下すべて false という真偽値リテラル型として型推論される
const falsy1 = !!0;
//    ^^^^^^ false
const falsy2 = !!0n;
//    ^^^^^^ false
const falsy3 = !!"";
//    ^^^^^^ false
const falsy4 = !!null;
//    ^^^^^^ false
const falsy5 = !!undefined;
//    ^^^^^^ false
```

`null` や `undefined` などが絡む際には、変数の値が falsy かどうかのチェック、つまり Truthiness narrowing をして型を絞り込みます。特に 0 や空文字列が falsy であるのが厄介です。

例えば、次のような単純な型ガードを行って CFA で型解析させてもうまくいきません。
```ts
function isStrOrArr(
  param: string | string[] | null
) {
  if (param) {
    // param の値が truty ならこのブランチの処理を行う
    if (typeof param === "string") {
      console.log(param, ": truty & string");
    } else {
      console.log(param, ": truty & string[]");
    }
  } else {
    console.log(param, ": falsy");
  }
}
isStrOrArr(["a", "b"]); // => [ "a", "b" ] : truty & string[]
isStrOrArr("test"); // => test : truty & string
isStrOrArr(null); // => null : falsy

// 空文字列が falsy として評価されてしまう
isStrOrArr(""); // =>  : falsy
```

次のように型ガードを構成することでうまく機能するようになります。

```ts
function isStrOrArrOk(
  param: string | string[] | null
) {
  if (param && typeof param === "object") {
    // param の値が truty かつ object 型の範疇ならこのブランチの処理を実行
    console.log(param, ": truty & string[]");
    //          ^^^^^: string[] 型
  } else if (typeof param === "string") {
    // param が string 型ならこのブランチの処理を実行
    console.log(param, ": truty & string");
    //          ^^^^^: string 型
  } else {
    // どの型ガードにも引っかからないならこのブランチの処理を実行
    console.log(param, ": falsy");
    //          ^^^^^: null 型
  }
}

isStrOrArrOk(["a", "b"]); // => [ "a", "b" ] : truty & string[]
isStrOrArrOk("test"); // => test : truty & string
isStrOrArrOk(null); // => null : falsy

// 空文字列もしっかりと文字列として判定できる
isStrOrArrOk(""); // =>  : truty & string
```

配列かどうかの汎用的な判定は `Array.isArray()` という静的メソッドを型ガードとして利用することで可能です。

```ts
const strArr = ["A", "B"];
if (Array.isArray(strArr)) { // 型ガード
  console.log(strArr);
  //          ^^^^^^: string[]
}
```

このような静的メソッドを使っても CFA で解析できるので、if ステートメントのブランチ内部では、`param` は配列型であると解析されて、型エラーとはならずにすみます。

そして、`Array.isArray()` はビルトインメソッドであり、配列の静的メソッドですが、型ガード関数として機能しています。型ガード関数は **Type predicate** という**特殊な返り値の型注釈**を施した上で真偽値を返す関数として定義することで自作することもできます。

# ユーザー定義型ガード関数による Narrowing

上のように特定の静的メソッドは CFA において型ガードとして機能します。そのような関数を型ガード関数(Type guard function)と呼びますが、このようばビルトインのものだけではなく、自分自身で型ガード機能を持つような独自の関数を作成することもできます。

そのような関数を「ユーザー定義型ガード関数(User-defined type guard function)」と呼びます。ユーザー定義型ガード関数は内部的なロジックから真偽値を返す関数ですが、返り値の型注釈を特殊な書き方にすることで、それを型ガードとして使用しているブランチ内で CFA で特定の型であると解析できるように伝える特殊な関数です。

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
Type predicate の [predicate](https://en.wikipedia.org/wiki/First-order_logic) とは日本語で言うと「述語」となります。数理論理学におけるタームから派生して利用されているようですが、ここでは型についての情報を記述するための表記方法のようなものだと考えればよいです。
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

Type predicate を記述することではじめて型ガード関数となります。

## 🛠 Assertion 関数による Narrowing

:::details 未完成

Assertion 関数は [TypeScript v3.7](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions) で導入された機能です。What's new の『Overview』ページの以下のところに記載されています。

- [Assertion Functions | TypeScript: Documentation - Overview](https://www.typescriptlang.org/docs/handbook/release-notes/overview.html#assertion-functions)

Assertion 関数は Node 環境の `assert` 関数をモデルにしていおり、次のような条件(`condition`)が `true` と評価されたときに現在のスコープにおいて型を絞り込みます。関数の戻り値の型注釈として `asserts condition` という特殊な形式の注釈を行います。

```ts
function assertSth(
  condition: any,
  msg?: string
): asserts condition { // condition は引数
// ^^^^^^^ ^^^^^^^^^
  if (!condition) {
    // 条件に一致しなければエラーを throw する
    throw new AssertionError(msg);
  }
}
```

例外がスローされれば型が

```ts
// 専用のアサーション関数
function assertResponse(
  obj: any
): asserts obj is SuccessResponse {
  //       ^^^^^^^^^^^^^^^^^^^^^^ condition
  if (!(obj instanceof SuccessResponse)) {
    // CFA で obj が SuccessResponse のインスタンスでなければ例外を throw するようにする
    throw new Error("Not a success!");
  }
}

const response = getResponse();
//    ^^^^^^^^: SuccessResponse | ErorrResponse 型

// 現在のスコープにおいて型を Narrowing するように CFA に伝える
assertResponse(response);
// 現在のスコープで変数の型が絞り込まれたので以降は SuccessResponse 型として見なされる
response; // SuccessResponse 型
```
:::
