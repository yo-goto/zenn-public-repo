---
title: "TypeScript の Widening"
emoji: "🕺"
type: "tech"
topics: ["javascript", "typescript", "deno"]
published: true
date: 2022-07-29
url: "https://zenn.dev/estra/articles/typescript-widening"
tags: [" #type/zenn "]
aliases: [記事_TypeScript の Widening]
---

# はじめに

前回の記事では Promise などの非同期処理から逆に TypeScript の型について理解してみるという試みをしてみました。

https://zenn.dev/estra/articles/ts-with-promise-type-annotation

冒頭『[変数の型注釈](https://zenn.dev/estra/articles/ts-with-promise-type-annotation#%E5%A4%89%E6%95%B0%E3%81%B8%E3%81%AE%E5%9E%8B%E6%B3%A8%E9%87%88)』の項目では、TypeScript では変数の初期化において型注釈を省略できることについて説明しました。

```ts
// JavaScript
const str1 = "文字列";

// TypeScript
const str2: string = "文字列の型注釈を追加";
//          ^^^^^^ string 型の型注釈
//                 str2 は string 型だよとコンパイラに伝えているだけ
const str3 = "文字列リテラル";
// 文字列リテラルで初期化しているのは明らかであり、型注釈は省略できる
```

そして、[Deno 環境](https://deno.land/manual) には備え付けのリンターがあり、変数の初期化で型が明らかなものではむしろ型注釈を省略しないと注意される [no-inferrable-types](https://lint.deno.land/?q=infer#no-inferrable-types) というリンタールールがありました。

>Variable initializations to JavaScript primitives (and `null`) are obvious in their type. Specifying their type can add additional verbosity to the code. For example, with `const x: number = 5`, specifying `number` is unnecessary as it is obvious that `5` is a number.
>([deno_lint docs no-inferrable-types](https://lint.deno.land/?q=infer#no-inferrable-types) より引用)

実際、以下のような TypeScript を書くとリンターに型注釈を省略するように注意されます。

```ts:無効となるコード(型注釈を省略しないと怒られる)
// 値の初期化
const a: bigint = 10n;
const b: bigint = BigInt(10);
const c: boolean = true;
const d: boolean = !0;
const e: number = 10;
const f: number = Number("1");
const g: number = Infinity;
const h: number = NaN;
const i: null = null;
const j: RegExp = /a/;
const k: RegExp = RegExp("a");
const l: RegExp = new RegExp("a");
const m: string = "str";
const n: string = `str`;
const o: string = String(1);
const p: symbol = Symbol("a");
const q: undefined = undefined;
const r: undefined = void someValue;

class Foo {
  prop: number = 5;
}

// デフォルト引数を使うときも型推論が容易なので省略すべき
function fn(s: number = 5, t: boolean = true) {}
```

ということで初期化で明らかに型推論が容易なものは以下のように型注釈を省略します。

```ts:有効なコード
const a = 10n;
const b = BigInt(10);
const c = true;
const d = !0;
const e = 10;
const f = Number("1");
const g = Infinity;
const h = NaN;
const i = null;
const j = /a/;
const k = RegExp("a");
const l = new RegExp("a");
const m = "str";
const n = `str`;
const o = String(1);
const p = Symbol("a");
const q = undefined;
const r = void someValue;

class Foo {
  prop = 5;
}

function fn(s = 5, t = true) {}
```

というのが、前回の記事での冒頭内容でした。

実はこのあたりの話題についてもう少し掘り下げてみると **Widening(型の拡大)** という [Narrowing(型の絞り込み)](https://www.typescriptlang.org/docs/handbook/2/narrowing.html) に対応する概念が出てきます。この Widening について知ることで TypeScript の型への理解が深まるのでアウトプットを兼ねて解説していきたいと思います。

# let 宣言とリテラル型

Wideing の前にリテラル型(Literal type)を知っておく必要
があるので、解説しておきます。

JavaScript にはプリミティブ型の値や、配列やオブジェクトなどを表現するための様々なリテラルがあります。

https://developer.mozilla.org/ja/docs/Glossary/Literal

変数宣言時にはそれらのリテラルを使っての初期化が可能でした。

```js:JavaScript
let str = "text";
//        ^^^^^ 文字列リテラル
let num = 42;
//        ^^ 数値リテラル
let bool = true;
//         ^^^^ 真偽値リテラル
let arr = [1, 2, 3];
//        ^^^^^^^^^ 配列リテラル
let obj = { a: "text", b: 42 };
//        ^^^^^^^^^^^^^^^^^^^^ オブジェクトリテラル
```

`const` でなく `let` の宣言で変数を初期化しているのは意味がありますが、一旦それは置いておいて、これらを TypeScript で型注釈をすると次のようになりました(省略できる型注釈をわざわざ書いています)。

```ts:TypeScript
let str: string = "text";
//       ^^^^^^ string 型の型注釈
let num: number = 42;
//       ^^^^^^ number 型の型注釈
let bool: boolean = true;
//        ^^^^^^^ boolean 型の型注釈
let arr: number[] = [1, 2, 3];
//       ^^^^^^^^ number 型要素を持つ配列型の型注釈
let obj: { a: string; b: number; } = { a: "text", b: 42 };
//       ^^^^^^^^^^^^^^^^^^^^^^^^ オブジェクト型の型注釈
```

let 宣言した変数は再代入が可能なので、型注釈した型の値を再代入できますね。

```ts:TypeScript
let str: string = "text";
let num: number = 42;
let bool: boolean = true;
let arr: number[] = [1, 2, 3];
let obj: { a: string; b: number; } = { a: "text", b: 42 };

// let 宣言した変数は同じ型の値で再代入可能
str = "string val";
num = 55;
bool = false;
arr = [100, 99, 98, 97];
obj = { a: "string val", b: 0 };

// 別の型の値を代入しようとすると型エラーとなる
str = 1000; // [Error]: Type 'number' is not assignable to type 'string'
```

これは、初期化の時点での型注釈を省略してもまったく同じことになります。むしろ型注釈を省略することで "no-inferrable-types" のリンタールールに怒られなくなります。

```ts:型注釈の省略
let str = "text";
//  ^^^: string 型として型推論される
let num = 42;
//  ^^^: number 型として型推論される
let bool = true;
//  ^^^^: boolean 型として型推論される
let arr = [1, 2, 3];
//  ^^^: number[] 型として型推論される
let obj = { a: "text", b: 42 };
//  ^^^: オブジェクト { a: string; b: number; } 型として型推論される

// let 宣言した変数は同じ型(型推論された型)の値で再代入可能
str = "string val";
num = 55;
bool = false;
arr = [100, 99, 98, 97];
obj = { a: "string val", b: 0 };

// 別の型の値を代入しようとすると型エラーとなる
str = 1000; // [Error]: Type 'number' is not assignable to type 'string'
```

具体的な値を使って初期化せずに宣言だけする場合には、型注釈のみを書いておくことで型安全になります。

```ts
// 初期化せずに型注釈だけしておく
let str: string;
let num: number
let bool: boolean;
let arr: number[];
let obj: { a: string; b: number; };

// 型注釈した型の値で再代入可能
str = "string val";
num = 55;
bool = false;
arr = [100, 99, 98, 97];
obj = { a: "string val", b: 0 };
```

`string` 型や `number` 型などの一般的なプリミティブ値の型がある一方で、それらよりも更に具体的な型としてリテラル型(Literal type)という型が存在しています。各リテラル値によってそのまま型注釈することでリテラル型として型注釈したことになります。

```ts
let str: "text" = "text";
//       ^^^^^ "text" という文字列のリテラル型として型注釈
let num: 42 = 42;
//       ^^ 42 という数値リテラルのリテラル型として型注釈
let bool: true = true;
//        ^^^^ true という真偽値リテラルのリテラル型として型注釈
```

リテラル型によって、`string` や `number`、`boolean` といった一般的な値の型よりもより具体的な型として変数に型注釈を施したことになります。これにより、値の再代入をしようとしてもリテラル型で指定したモノ以外の値は受け付けなくなります。

```ts
// リテラル型で型注釈
let str: "text" = "text";
let num: 42 = 42;
let bool: true = true;

// 同じ値の再代入はできる
str = "text";
num = 42;
bool = true;

// リテラル型で指定したもの以外の値を代入しようとすると型エラーになる
str = "error"; // [Error]
// Type '"error"' is not assignable to type '"text"'.
num = 100; // [Error]
// Type '100' is not assignable to type '42'.
bool = false; // [Error]
// Type 'false' is not assignable to type 'true'.
```

オブジェクトや配列の場合は話が少し複雑になりますが、内部的に使われている型がリテラル型なので考え方は同じになります。

```ts
let arr: [1, 2, 3] = [1, 2, 3];
//       ^^^^^^^^^ 1, 2, 3 という数値リテラル型の要素を持つタプル型として型注釈
let obj: { a: "text"; b: 42 } = { a: "text", b: 42 };
//       ^^^^^^^^^^^^^^^^^^^ { a: "text"; b: 42 } という
// 文字列リテラル型と数値リテラル型のプロパティの値の型を持つオブジェクトの型として型注釈

// 同じ値の再代入はできる
arr = [1, 2, 3];
obj = { a: "text", b: 42 };

// リテラル型で指定したもの以外の値を代入しようとすると型エラーになる
arr = [100, 99, 98]; // [Error]
//     ^^^ Type '100' is not assignable to type '1'.
//          ^^ Type '99' is not assignable to type '2'.
//              ^^ Type '98' is not assignable to type '3'.
obj = { a: "error", b: 100 }; // [Error]
//      ^ Type '"error"' is not assignable to type '"text"'.
//                  ^ Type '100' is not assignable to type '42'
```

具体的なリテラル値を指定しているわけですから、`string` や `number` などの一般的な型よりも具体的な型を指定していることが分かったと思います。

# const 宣言とリテラル型

値の宣言は再代入しないなら `const` で宣言するのが通例だと思います。Deno 環境でも、[prefer-const](https://lint.deno.land/?q=prefer-const#prefer-const) というリンタールールが存在しており、`let` で宣言した変数で再代入していないものがあれば `const` で宣言するように注意されます。

>Since ES2015, JavaScript supports `let` and `const` for declaring variables. If variables are declared with `let`, then **they become mutable**; we can set other values to them afterwards. Meanwhile, if declared with `const`, **they are immutable; we cannot perform re-assignment to them**.
>
>In general, to make the codebase more robust, maintainable, and readable, it is highly recommended to use `const` instead of `let` wherever possible. **The fewer mutable variables are, the easier it should be to keep track of the variable states** while reading through the code, and thus it is less likely to write buggy code. So this lint rule **checks if there are `let` variables that could potentially be declared with `const` instead**.
>([deno_lint docs prefer-const](https://lint.deno.land/?q=prefer-const#prefer-const) より引用、太字は筆者強調)

より堅牢で、保守可能かつ可読性の高いコードにするため、再代入可能な `let` から再代入のできない `const` に変更できる可能性のあるものには警告を出してなるべく immutable なコードを書かせるようにしているとのことです。

従って、なるべく `const` を使って変数宣言することが Deno 環境では推奨されています(一般的にもその場合が多いです)。

ということで、冒頭で `const` 宣言していたように `let` で紹介したコードも再代入しない限りは `const` で書きます。

```ts:const宣言
const str: string = "text";
const num: number = 42;
const bool: boolean = true;
const arr: number[] = [1, 2, 3];
const obj: { a: string; b: number; } = { a: "text", b: 42 };
```

そして、"no-inferrable-types" のリンタールールから、このような型推論が明らかな変数の初期化における型注釈は省略するように警告されますので、省略して書くと次のようになりました。

```ts:型注釈を省略
const str = "text";
const num = 42;
const bool = true;
const arr = [1, 2, 3];
const obj = { a: "text", b: 42 };
```

もちろん型推論が働きますが、`const` 宣言によってプリミティブ値で初期化した際には、その変数は `string` や `number` といった一般的な型で推論はされません。**実は初期化に使ったリテラルのリテラル型として型推論がされています**。

```ts:リテラル型で型推論されている
const str = "text";
//    ^^^ "text" という文字列のリテラル型として型推論される
const num = 42;
//    ^^^ 42 という数値リテラルのリテラル型として型推論される
const bool = true;
//    ^^^^ true という真偽値リテラルのリテラル型として型推論される
```

実は型注釈を省略していると次のようにリテラル型で型注釈しているのと近い状態で型推論がされています(厳密にはそれらは同じでないことが重要になりますが、いまは置いておきます)。

```ts
const str: "text" = "text";
//         ^^^^^ "text" という文字列のリテラル型として型注釈
const num: 42 = 42;
//         ^^ 42 という数値リテラルのリテラル型として型注釈
const bool: true = true;
//          ^^^^ true という真偽値リテラルのリテラル型として型注釈
```

一方で、配列リテラルやオブジェクトリテラルの場合には `let` 宣言時と同じ用に型推論がされるので注意してください。

```ts
const arr = [1, 2, 3];
//    ^^^: number[] 型として型推論される
const obj = { a: "text", b: 42 };
//    ^^^: オブジェクト { a: string; b: number; } 型として型推論される
```

`const` 宣言によって変数に対しての値の再代入ができなくなるわけですから、一般的な `string` や `number` といった型で型推論するよりも、より具体的なリテラル型として型推論を働かせた方が合理的になるわけです。ただし、**オブジェクトや配列など場合には値の再代入はできなくても、要素の値やプロパティの値は変更が可能であり mutable なわけですから、このような違いがでてくるわけです**。`let` 宣言で見たようにここでリテラル型を使ってしまったら、値の変更ができなくってしまいます。

```ts
const arr = [1, 2, 3];
//    ^^^: number[] 型として型推論される
const obj = { a: "text", b: 42 };
//    ^^^: オブジェクト { a: string; b: number; } 型として型推論される

arr[0] = 100; // 配列要素の値の変更が可能 [mutable]
obj.a = "this is string"; // プロパティの値の変更が可能 [mutable]
```

実際、`let` のときのようにリテラル型をタプル型やオブジェクト型で内部的に使って型注釈をすると配列要素の値やオブジェクトのプロパティの値の変更をすると型エラーになります。

```ts
const arr: [1, 2, 3] = [1, 2, 3];
//         ^^^^^^^^^ 1, 2, 3 という数値リテラル型の要素を持つタプル型として型注釈
const obj: { a: "text"; b: 42 } = { a: "text", b: 42 };
//         ^^^^^^^^^^^^^^^^^^^ { a: "text"; b: 42 } という
// 文字列リテラル型と数値リテラル型のプロパティの値の型を持つオブジェクトの型として型注釈

arr[0] = 100; // [Error]
// Type '100' is not assignable to type '1'.
obj.a = "error"; // [Error]
// Type '"error"' is not assignable to type '"text"'.
```

`const` 宣言でプリミティブ値を初期化した際に型注釈を省略していると、それぞれの初期化に使った値のリテラル型として型推論がされるという話でしたが、このままだと `string` 型や `number` 型といった一般的な型の変数のプロトタイプメソッドやそれらの型を受け入れる関数やメソッドなどが使えないのではないかと疑問に持つはずです。

```ts:TypeScript
const str = "text";
//    ^^^ "text" という文字列のリテラル型として型推論される
const num = 42.22;
//    ^^^ 42.22 という数値リテラルのリテラル型として型推論される

// string 型のプロトタイプメソッドは使えるのか?
console.log(str.toUpperCase());
// number 型の値を受け入れる Math の静的メソッドは使えるのか?
console.log(Math.floor(num));
```

上記のコードは TypeScript の範疇ですが、型注釈などが無いので明らかに JavaScript のままとなっています。そして JavaScript では上のコードは何も間違っていませんし、正しく実行できます。

ということで、基本的に問題はまったくありませんが、TypeScript でもなぜ大丈夫なのかを理解するには Widening という概念が必要になります。

# Playground

Widining の用語は実は Microsoft 公式のドキュメントである『[The TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)』の現時点(2022-07-29)最新のバージョンには記載されていません(どうやら昔のバージョンには記載されていたようです)。

その代わり、TypeScript 公式サイトの [Playground](https://www.typescriptlang.org/play) にはサンプルコードともに説明がされています。Playground のサンプルコードは新しい TypeScript の機能やそういった概念的な話の説明もされています。

JavaScript 機能と TypeScript 独自のサンプルコードと説明があります。

![js-example](/images/typescript-widen-narrow/img_playground-js-ex.jpg)

![ts-example](/images/typescript-widen-narrow/img_playground-ts-ex.jpg)

そして Widening と Narrowing のサンプルと説明は次の場所にあります。

https://www.typescriptlang.org/play/?q=111#example/type-widening-and-narrowing

# Widening

`let` 宣言での文字列などのプリミティブ値で初期化した際には一般的な `string` 型として変数の型が型推論された一方で、`const` 宣言の場合にはそのまま具体的なプリミティブ値のリテラル型として型推論されていました。

```ts
let strLet = "mutable";
//  ^^^^^: string 型として型推論される
const strConst = "immutable";
//    ^^^^^^^^: "immutable" という文字列リテラル型として型推論される

strLet = "再代入できる"; // OK (string 型の値を受け入れる)

strConst = "再代入できない"; // NG 
// [Error]: Cannot assign to 'strConst' because it is a constant
```

`let` 宣言で型注釈を省略した場合に一般的な `string` 型として型推論されるのは再代入可能な変数であり、他の `string` 型の値を再代入できるようにするためであり、具体的すぎるリテラル型として型推論されるのは合理的ではありません。その一方で、`const` した変数はそもそも再代入できないので、一般的な `string` 型として型推論するよりも初期化に使ったリテラル値のリテラル型として型推論した方が合理的ですね。

このように、`let` は `const` よりも広い(wide)型を受け入れるように型推論が働くというわけです。

Widening(型の拡大) や Narrowing(型の絞り込み) は変数の型を広げて受け入れる値を拡大したり、逆に変数の型を具体的に絞り込んで使えるメソッドなどを特定の型のものとして限定するように推定する機能や行為、概念のことを指します。

現在の『The TypeScript Handbook』の方には言及されていませんが、Playground の最後に記載されている以下のリソースには Wideing について詳しく解説されています。

https://sandersn.github.io/manual/Widening-and-Narrowing-in-Typescript.html

https://mariusschulz.com/blog/literal-type-widening-in-typescript

今まで見てきたようなリテラル型に関しての Widening はより具体的には **Literal Widening** と呼ばれており、`const` 宣言では初期値から具体的なリテラル型として型推論されたのに、`let` 宣言ではより一般的な型として広げられて型推論されるのもこの Literal Widening の一部です。

TypeScirpt におけるこの "Literal Widening" の機能はそこそこ大きなもので、次の Pull request でマージされた機能(あるいはコンセプト)であり、PR ではそのコンセプトの説明もなされています。

https://github.com/Microsoft/TypeScript/pull/10676

Literal Widening の具体的な機能やルールは以下のものであると言及されています。いくつかを抜粋しています。

- The type of a literal in an expression is _always_ a literal type (e.g. `true`, `1`, `"abc"`).
- The type of a string or numeric literal occurring in an _expression_ is a widening literal type.
- The type of a string or numeric literal occurring in a _type_ is a non-widening literal type.
- The type inferred for a `const` variable or `readonly` property without a type annotation is the type of the initializer _as-is_.
- The type inferred for a `let` variable, `var` variable, parameter, or non-readonly property with an initializer and no type annotation is the widened literal type of the initializer.
- The type inferred for a property in an object literal is the widened literal type of the expression unless the property has a contextual type that includes literal types.
- The type inferred for an element in an array literal is the widened literal type of the expression unless the element has a contextual type that includes literal types.
>([Always use literal types by ahejlsberg · Pull Request #10676 · microsoft/TypeScript](https://github.com/Microsoft/TypeScript/pull/10676) より抜粋引用)

長いので、全部いきなり理解するのは難しいですがすこしずつ見ていきます。まずはこれですが、式内でのリテラル値の型は常にリテラル型になるといっていますね。

- The type of a literal in an expression is _always_ a literal type (e.g. `true`, `1`, `"abc"`).

次の２つは今まで見てきた `const` と `let` の宣言での違いについてです。

- The type inferred for a `const` variable or `readonly` property without a type annotation is the type of the initializer _as-is_.
- The type inferred for a `let` variable, `var` variable, parameter, or non-readonly property with an initializer and no type annotation is the widened literal type of the initializer.

変数が mutable になる場所では、推論される方はより一般的なものとして拡大(Widening)されます。具体的に言えば、`const` で宣言した変数の値を `let` で宣言した変数の初期化に使う時に Widening が起こり、変数の型は `string` や `number` などの一般的な型に拡大されて型推論されるというわけです。

```ts
const c1 = 1;  // 1 型(数値リテラル型)として型推論される
const c2 = c1;  // 1 型(数値リテラル型)として型推論される
const c3 = "abc";  //"abc" 型(文字列リテラル型)として型推論される
const c4 = true;  // true 型(真偽値リテラル型)として型推論される
const c5 = cond ? 1 : "abc";  // 1 | "abc" 型(数値リテラル型と文字列リテラル型ののユニオン型)として型推論される

let v1 = 1;  // number 型として型が広げられて型推論される
let v2 = c2;  // number 型として型が広げられて型推論される
let v3 = c3;  // string 型として型が広げられて型推論される
let v4 = c4;  // boolean 型として型が広げられて型推論される
let v5 = c5;  // number | string 型として型が広げられて型推論される
```

三項演算子を使って初期化する場合にも、`const` なら具体的なリテラル型のユニオン型として、`let` ならより一般的な型のユニオン型、あるいは三項演算子で対応付けている値の一般的な型が同じならその型として拡大されます。

```ts
const a = cond ? "foo" : "bar";
// "foo" | "bar" という文字列リテラル型同士のユニオン型として型推論される
let b = cond ? "foo" : "bar";
// string 型として型推論される(どちらも文字列リテラルだから)

let c: "foo" | "bar" = cond ? "foo" : "bar";
// 具体的に "foo" | "bar" 型として型注釈したらその型として見なされる
```

`const` 宣言時に配列やオブジェクトの場合はリテラル型が関与した型としてみなされなかったことも言及されていますね。

- The type inferred for a property in an object literal is the widened literal type of the expression unless the property has a contextual type that includes literal types.
- The type inferred for an element in an array literal is the widened literal type of the expression unless the element has a contextual type that includes literal types.

プロパティの値や配列要素の値など、具体的なリテラル型となるところを型が拡大されて型推論されます。

```ts
const a1 = [1, 2, 3];
//    ^^: number[] として型推論される(要素自体は mutable だから)

const a2: [1, 2, 3] = [1, 2, 3];
//    ^^: [1, 2, 3] というリテラル型のタプル型として明示的に型注釈する場合

const o1 = { kind: 0 };
//    ^^: { kind: number } として型推論される(プロパティの値自体は mutable だから)

const o2: { kind: 0 } = { kind: 0 };
//    ^^: { kind: 0 } というリテラル型のプロパティの値を持つオブジェクトの型として明示的に型注釈する場合
```

このように型注釈を省略せずに明示的に施すことによって Widening をしないように制御できます。型注釈を施すことで Widening をしないようにできます。

```ts:Widening を抑制
let str: "text" = "text";
let num: 42 = 42;
let bool: true = true;
let arr: [1, 2, 3] = [1, 2, 3];
let obj: { a: "text"; b: 42 } = { a: "text", b: 42 };
```

ただ、Deno 環境でこのようなリテラル型の型注釈をする [prefer-as-const](https://lint.deno.land/?q=prefer-as-const#prefer-as-const) というリンタールールで以下のように注意されてしまいます。

>Expected a `const` assertion instead of a literal type annotation
>Remove a literal type annotation and add `as const`

リテラル型として型注釈するのではなく、型アサーション(Type assertion)を使って、`as const` を付けろという注意です。

型アサーションについては公式ドキュメントの以下の項目に記載されています。

https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions

型アサーションはコンパイラが型推論するよりもプログラマの方がより具体的な型を知っており、そのように具体的な型であるとコンパイラに伝えたい場合に利用します。Widening の場合も、プログラマがより具体的なリテラル型であると伝えたい場合には型アサーションをして Widening を抑制します。

そのようなリテラル型として変数を宣言する場合には以下の３つの方法があると "prefer-as-const" のルールには記載されています。

- (1) 明示的に型注釈
- (2) 通常の型アサーション(`as "foo"` または `<"foo">`)
- (3) const アサーション(`as const`)

```ts
// (1) 明示的に型注釈
let str1: "text" = "text";

// (2) 通常の型アサーション(`as "foo"` または `<"foo">`)
let str2A = "text" as "text";
let str2B = <"text"> "text";

// (3) const アサーション(`as const`)
let str3 = "text" as const;
```

そして、この "prefer-as-const" のリンタールールでは以下のようなコードが無効なものとして警告されると lint docs では記載されています。

```ts:無効となるコード
let a: 2 = 2; // 型注釈
let b = 2 as 2; // 型注釈
let c = <2> 2; // 型アサーション
let d = { foo: 1 as 1 }; // 型アサーション
```

`as const` という const アサーションでは書き方を統一できて分かりやすいので推奨されるというわけです。というこで、"prefer-as-const" のリンタールールでは以下のようなコードが有効となります。

```ts:有効となるコード
let a = 2 as const;
let b = 2 as const;
let c = 2 as const;
let d = { foo: 1 as const };

let x = 2;
let y: string = "hello";
let z: number = someVariable;
```

すこし話がそれましたが、Widening の話に戻しますと、型注釈なしで `const` 宣言した変数にはその初期化に使った文字列リテラルや数値リテラルなどから具体的なリテラル型として型が推論されますが、その型の Widening を抑制することで `let` 宣言した変数にその値を代入した際にも継続的にその変数の型の Widening が抑制されることになります。

```ts
const strConst1 = "text";
//    ^^^^^^^^: "text" 文字列リテラル型として具体的に型推論される
let strLet1 = strConst1;
//  ^^^^^^^: string 型として拡大されて型推論される

const strConst2 = strConst1;
//    ^^^^^^^^: "text" 文字列リテラル型として具体的に型推論される
let strLet2 = strConst2;
//  ^^^^^^^: string 型として拡大されて型推論される

const strConst3 = strLet1;
//    ^^^^^^^^: string 型として型推論される(代入元の値が string 型だから)

const strConst4 = "text" as const;
//    ^^^^^^^^: "text" 文字列リテラル型として具体的に型を指定(Widening を抑制している)
let strLet4 = strConst4;
//  ^^^^^^^: "text" 文字列リテラル型として具体的に型推論される(Widening の抑制が継続する)
```

Widening のメリットについて、次のような古典的な初期化子 `i` を使った `for` ループを考えてみます。`const` で宣言した変数を `i` の初期値として与える際には、今まで見てきたように実は Widening が起きています。

```ts
const start = 1;
//    ^^^^^: 1 という具体的な数値リテラル型として型推論される
const end = 5;
//    ^^^: 5 という具体的な数値リテラル型として型推論される

for (let i = start; i <= end; i = i + 1) {
  //     ^: number 型として型が拡大されて型推論される
  console.log(i);
}
```

`i` に `start` の値を代入する際に Widening が起きてしまうと、`1` という初期値のリテラル型しか再代入できないことになってインクリメント自体ができなくなってしまいます。もしも、Widening を型アサーションで抑制すると型エラーとなります(JavaScript としてはもちろん問題ありません)。

```ts
const start = 1 as const;
//    ^^^^^: 1 という具体的な数値リテラル型として型アサーション(widening を抑制)
const end = 5;

for (let i = start; i <= end; i = i + 1) {
  //                          ^: 1 という数値リテラル型なので再代入時にインクリメントすると型エラーとなる
  console.log(i);
}
```

参考文献
https://sandersn.github.io/manual/Widening-and-Narrowing-in-Typescript.html

さて、Widening が分かってきたと思いますが、`const` 宣言で型注釈を省略して Widening を抑制しないと具体的なリテラル型として型推論されてしまい、一般的な `string` 型や `number` 型などに使えるプロトタイプメソッドや静的メソッドなどが使えるのという疑問がありました。

```ts:TypeScript
const str = "text";
//    ^^^ "text" という文字列のリテラル型として型推論される
const num = 42.22;
//    ^^^ 42.22 という数値リテラルのリテラル型として型推論される

// string 型のプロトタイプメソッドは使えるのか?
console.log(str.toUpperCase());
// number 型の値を受け入れる Math の静的メソッドは使えるのか?
console.log(Math.floor(num));
```

この話は自分の推測では Widening に直接的に位関係あると思っていたのですが、調べてみたらそこまで直接的に関係なく、これは単純にそれぞれのリテラル型は Widening で一般化されて広げられるような `string` や `number` といった型の部分集合の型(subtype)にあたるとのことでした。subtype であるゆえに、文字列リテラル型の変数の値は `string` 型の変数に代入できます(その逆はできません)。

```ts
const strLiteral = "text" as const;
//    ^^^^^^^^^^: "text" 文字列リテラル型として型アサーション
const str: string = strLiteral;
// 文字列リテラル型は string 型の subtype なので string 型の変数に代入可能
```

実際に TypeScript のリポジトリの次の Pull Request で明言されています。

>A string literal type can be considered **a subtype of the `string` type**. This means that a string literal type is assignable to a plain `string`, but not vice-versa.
>([String literal types by DanielRosenwasser · Pull Request #5185 · microsoft/TypeScript](https://github.com/Microsoft/TypeScript/pull/5185) より引用、太字は筆者強調)

それ故に、subtype であるリテラル型を一般化した `string` や `number` 型などで使えるプロトタイプメソッドや、静的メソッド、それらの一般的な型の変数を引数にとる関数などでもリテラル型を受け入れることができるというわけです。ただし、`toUpperCase()` などのプロトタイプメソッドで返ってくる値を `const` 宣言した変数に代入しようとすると、結局は拡張された `string` 型の変数となります。

```ts
const strLiteral = "text";
//    ^^^^^^^^^: "text" 文字列リテラル型として型推論される

const strUpper = strLiteral.toUpperCase();
//                          ^^^^^^^^^^^ stirng 型で使えるプロトタイプメソッド
//    ^^^^^^^^: string 型として型推論される
```

逆に一般的な型である `string` の型として注釈されている変数の値は、より具体的な型である文字列リテラル型やそれを使ったユニオン型などには代入できないわけです。

```ts
// string 型であると明示的に型注釈
const str: string = "text";

// より一般的な string 型の値を具体的なリテラル型のユニオン型には代入できない
const strLiteralUnion: "text" | "mytext" = str; // [Error]
// Type 'string' is not assignable to type '"text" | "mytext"'.
```

参考文献
https://mariusschulz.com/blog/string-literal-types-in-typescript#string-literal-types-vs-strings

# 終わり
Widening についてはまだいくつかルールがあるので、今回は基本的な解説にとどめておきます。それらのルールについては自分も理解しきっていないところがあるので(理解したら追記するかもしれません)。

Widening の対となる Narrowing については次回の記事で書こうと思います。

