---
title: "TypeScript の Narrowing"
emoji: "💃"
type: "tech"
topics: ["typescript", "deno"]
published: false
date: 2022-07-29
url: "https://zenn.dev/estra/articles/typescript-narrowing"
tags: [" #type/zenn "]
aliases: [記事_TypeScript の Narrowing]
---

# はじめに

前前回の記事では、Widening(型の拡大) について解説しました。

https://zenn.dev/estra/articles/typescript-widening

前回の記事では、型の集合性と階層性について解説しました。

https://zenn.dev/estra/articles/typescript-type-set-hierarchy

今回は、その Widening(型の拡大) の対となる Narrowing(型の絞り込み) について解説します。Narrowing の方がよく知られている概念なので解説が大変ですが、がんばってみます。

# ユニオン型と型集合

前回の記事では、型は以下のような具体的な値の集合であると解説しました。あらゆる型は、全体集合となる `unknown` 型の部分集合であり、`never` 型は空集合としてみなすことができます。

![全体集合](/images/typescript-widen-narrow/img_typeSet_4.png)

単位型(Unit type)たるリテラル型の集合が `string` 型や `number` 型などの一般的なプリミティブ型であり、これらは集合型(Collective type)と呼ばれました。

各リテラル型の積集合や異なる集合型(Collection type)の積集合をインターセクション型で作ろうとすると共通要素が全く無いのでこの `never` 型となりました。また、`never` 型は空集合ということで他のなんらかの型との和集合をユニオン型で作成すると `never` 型は無かったかのようにユニオン型で和の作成に使用した要素の型となります。

```ts
type StrOrNever = string | never;
// type StrOrNever = string; と同じ
```

ユニオン型を作ると以下のような和集合となります。

![集合型の和集合](/images/typescript-widen-narrow/img_typeSet_3.png)

Narrowing とは型を絞り込んでいくことですが、そもそもなぜ絞り込む必要があるかというと、上記だと `string | number | undefined` というそれぞれで違う操作体系を持つ集合の和集合として型をつくってしまっているためです。

具体的には、`string` 型と `number` 型では扱えるプロトタイプメソッドやその型の変数に対して加えることのできる操作などが変わってくるために場合分けをする必要がでてくるためです。

例えば、`String.prototype.toUpperCase()` などの文字列で使えるプロトタイプメソッドは `number` 型ではつかえません。そういったユニオン型の要素の片方にしかないようなメソッドを使ったりしようとすると、コードのある時点でユニオン型が具体的にはどのような型なのか `string` 型や `number` 型といったレベルで絞り込んでコンパイラに型がなんであるかを伝える必要がでてきます。

また、TypeScript では `undefined` とのユニオン型がよく出現します。オプション引数やオプショナルプロパティなどを使うことで強制的に型注釈した型と `undefined` 型とのユニオン型としてみなされます。

```ts
// オプション引数を使った関数定義
function greeting(
  person?: string // string | undefined となる
) {
  // ...
}
```

`string | undefined` というユニオン型は `string` 型よりも広い集合を表現しており、`string` 型の supertype となります。`string` や `String` などの基本的な型の supertype と subtype の関係性は継承(inheritance)に近いものですが、このように集合性の観点から対象とする要素を広げることでもその型を supertype としてみなすことができます。ちなみに `S` 型が `T` 型の subtype であるときには $S <: T$ というように書きます。

$ string<:(string | undefined)$ なので



# ユニオン型と制御フロー解析

ユニオン型(Union type)は複数の型を合成した型です。例えば `string | number` などがユニオン型ですが、これは `string` 型または `number` 型という２つの型を受け入れる合成された型です。このように２つの型を組み合わせることを「**型の合成(Composing Types)**」と呼びました。

```ts
// let 宣言して型定義
let strornum1: string | number;
strornum1 = Math.random() < 0.5 ? "文字列" : 42; // 三項演算子

// type で型作成
type StrOrNum = string | number;
let strornum2: StrOrNum;
strornum2 = 42; // number 型の値も代入できるし
strornum2 = "文字列"; // string 型の値も代入できる
```

ここで Widening の復習をしておきますが、次のように三項演算子を使った上で変数の初期化を行った場合には、`const` 宣言なら具体的なリテラル型のユニオン型として型推論され、`let` 宣言なら一般的な `string` や `number` 型のユニオン型として Widening されて型推論されます。

```ts
const unionValConst = Math.random() < 0.5 ? "text" : 42;
//   ^^^^^^^^^: "text" | 42 という具体的なリテラル型のユニオン型として型推論される

let unionValLet = Math.random() < 0.5 ? "text" : 42;
//  ^^^^^^^^^^^ string | number というユニオン型に Widening されて型推論される

unionValLet = unionValConst;
// リテラル型は Widening した結果の型の subtype なので代入可能
```

型アサーションのために、`as const` で const アサーションすることによって Widening を抑制できました。

```ts
//    _______________: "text" | 42 というリテラル型のユニオン型として型推論される
const unionValConstAs = Math.random() < 0.5
  ? "text" as const
  : 42 as const;
// (const アサーションでそれぞれ Widening を抑制)

let unionValLetAs = unionValConstAs;
//  ^^^^^^^^^^^^^ "text" | 42 という具体的なリテラル型のユニオン型として型推論される(Widening の抑制が継続)
```

このようなユニオン型が関数の引数となることで、関数内部で引数に対して利用できるメソッドがそのユニオン型に含まれる型によって変わってくるので場合分けをする必要がでてきます。

```ts
// string 型または number 型 やそのユニオン型で注釈された変数を受け入れる(それ以外は受け入れない)
function strOrNum(
  param: string | number
): void {
  if (typeof param === "string") {
    // string 型のプロトタイプメソッド
    console.log(param.toUpperCase());
  } else { // string 型でないなら number 型
    // number 型の値に使える静的メソッド
    console.log(Math.floor(param));
  }
}
```

こういったコードの構造に基づいて値の型をより具体的に推定できるようにすることを(型の範囲をより具体的なものに狭めることから) **Narrowing(型の絞り込み)** と呼びました。上のコードでの `if` 節や `switch` や `while` などのコードの構造によって各場所での変数の型を絞り込みます。このようなコードを書くと TypeScript (コンパイラやエディタの拡張機能)はある変数が特定のブランチなどに到達した時点でその型がなんであるか解析をしています。この解析を「[制御フロー解析(Control flow analysis: CFA)](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#control-flow-analysis)」と呼びました。

この CFA ですが、TypeScript 公式のチートシートの１つとして以下のページでまとめられています。

https://www.typescriptlang.org/static/TypeScript%20Control%20Flow%20Analysis-8a549253ad8470850b77c4c5c351d457.png

そして Narrowing については TypeScript の公式ドキュメントである『The TypeScript Handbook』にまるまる１チャプターとして解説されています(それだけ重要なコンセプトであるということですね)。

https://www.typescriptlang.org/docs/handbook/2/narrowing.html

JavaScript には無い TypeScript の型システムにおいて、引数が特定の型と `undefined` などとの型のユニオン型になることが多いため、ユニオン型は非常に重要です。ということで、上記の CFA において具体的に型を絞り込んでいく Narrowing の方法を学ぶことが必要になってくるわけですね。

CFA ではユニオン型の変数の型をいくつかの真偽値のロジックパターンに基づいて型を絞り込んでいきます。基本的には、`if` 節で条件判定しますが、`switch` などを使う場合もあります。

```ts
if (typeof input === "string") {
  console.log(input);
  //          ^^^^^^ string 型として CFG で解析される
  console.log(input.toUpperCase());
  // このブランチでは string 型データのプロトタイプメソッドなどが利用できる
}
```

# 代入による Narrowing

上のような CFA での典型的な Narrowing の解説に入る前に、もっと基本的な Narrowing について見ておきます。

前の記事を見て入れば Widening を知っているわけですが、実はその過程ですでに Narrowing についても知っています。

ユニオン型として `let` 宣言した変数では、具体的な値を代入することでその型が確定することになるので、「[代入(Assignment)](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#assignments)」も Narrowing の１つとしてカウントされます。

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

## typeof 演算子を使った Narrowing

`typeof` 演算子によって変数の型を基本的な判定ができます。`typeof` 演算子で判定できるものは以下のような基本的な型となります。

- `"string"`
- `"number"`
- `"bigint"`
- `"boolean"`
- `"symbol"`
- `"undefined"`
- `"object"`
- `"function"`

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

## instanceof 演算子を使った Narrowing

JavaScript には `instanceof` 演算子というものがありますが、これを型ガードとして利用することもできます。特にクラスのインスタンスであるかを判別することに利用します。

```ts
const today = Math.random() < 0.5
  ? new Date()
  : "2022/07/30";

if (today instanceof Date) {
  console.log(today.toUTCString());
} else {
  console.log(today);
}
```

```ts
const arrp = Math.random() < 0.5
  ? [22]
  : 42;

if (arrp instanceof Array) {
  console.log("配列", arrp);
} else {
  console.log("数値", arrp);
}
```

## in 演算子を使った Narrowing

```ts
if ("error" in data) {
  data; // { error: ... }
}
```

# Equality Narrowing



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

配列かどうかの判定は `Array.isArray()` という静的メソッドを型ガードとして利用しての判定も可能です。

```ts
const strArr = ["A", "B"];
if (Array.isArray(strArr)) { // 型ガード
  console.log(strArr);
  //          ^^^^^^: string[]
}
```

このような静的メソッドを使っても CFA で解析できるので、if ステートメントのブランチ内部では、`param` は配列型であると解析されて、型エラーとはならずにすみます。

そして、`Array.isArray()` はビルトインメソッドであり、配列の静的メソッドですが、型ガード関数として機能しています。型ガード関数は **Type predicate** という特殊な返り値の型注釈を施した上で真偽値を返す関数として定義することで自作することもできます。

# ユーザー定義型ガード関数による Narrowing

上のように特定の静的メソッドは CFA において型ガードとして機能します。そのような関数を型ガード関数(Type guard function)と呼びますが、このようばビルトインのものだけではなく、自分自身で型ガード機能を持つような独自の関数を作成することもできます。

そのような関数を「ユーザー定義型ガード関数(User-defined type guard function)」と呼びます。ユーザー定義型ガード関数は内部的なロジックから真偽値を返す関数ですが、返り値の型注釈を特殊な書き方にすることで、それを型ガードとして使用しているブランチ内で CFA で特定の型であると解析できるように伝える特殊な関数です。

```ts
function isErrorResponse(
  obj: Response
): obj is APIErrorResponse {
// ^^^^^^^^^^^^^^^^^^^^^^^^ type predicate の型注釈
  return obj instanceof APIErrorResponse;
}
```

返り値の型注釈を Type predicate にすることによって単に真偽値を返すだけではなく、CFA において型を絞り込んで解析できるようにしています。

ユーザー定義型ガード関数は TypeScript v.1.6 で導入された古い機能です。『Overview』の以下の場所に記載されています。

- [User-defined type guard functions | TypeScript: Documentation - Overview](https://www.typescriptlang.org/docs/handbook/release-notes/overview.html#user-defined-type-guard-functions)

戻り値の型注釈を `x is T` というようにします(`x` はパラメータで、`T` は何かしらの型)。ユーザー定義型ガード関数は `if` ブロックで変数を渡して呼び出された際にはその変数の型が Narrowing(絞り込み) されます。

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




# Assertion 関数による Narrowing

Assertion 関数は TypeScript v3.7 で導入された機能です。What's new の『Overview』ページの以下のところに記載されています。

- [Assertion Functions | TypeScript: Documentation - Overview](https://www.typescriptlang.org/docs/handbook/release-notes/overview.html#assertion-functions)

Assertion 関数は Node 環境の `assert` 関数をモデルにしていおり、次のような条件(`condition`)が `true` と評価されたときに現在のスコープにおいて型を絞り込みます。

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

# 判別可能なユニオン型

