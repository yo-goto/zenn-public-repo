---
title: "TypeScript における型の集合性と階層性"
emoji: "🕴"
type: "tech"
topics: ["typescript"]
published: true
date: 2022-08-04
url: "https://zenn.dev/estra/articles/typescript-type-set-hierarchy"
tags: [" #type/zenn #TypeScript/inference "]
aliases: [
  記事_TypeScript における型の集合性と階層性,
  型の集合性,
  型の階層性,
  型の階層図,
  Type hierarchy
]
---

# はじめに

前回の記事では TypeScript における型の拡大(Widening)について解説しました。

https://zenn.dev/estra/articles/typescript-widening

Narrowing の解説に入る前に(Widening を深く理解するためにも)必要な知識や考え方がいくつかあると思ったので、今回は型の集合性と階層性について見ていきたいと思います(短いですが内容的に切り出しても興味深いので)。ただ、集合論や型理論については詳しくないので間違っているところがあるかもしれません(調べられた範囲で解説しています)。

# Collective type と Unit type

前回記事で解説した "Literal Widening" は `"text"` や `42`、`true` などのリテラルから作られるリテラル型は、変数が mutable になる場所では `string` 型といった一般的な型に拡大されて型推論されるという話でした。

```ts
const strConst = "text";
//    ^^^^^^^^: "text" 文字列リテラル型として型推論される
let strLet = "text";
//  ^^^^^: string 型として拡大されて型推論される

let mutable = strConst;
//  ^^^^^^^: string 型として拡大される
```

実は `"text"` や `42`、`true` といった具体的なリテラルの値から作られるリテラル型に対して、通常はプリミティブ型(Primitive type)と呼ばれる `string` や `number`、`boolean` といった一般的な型は **集合型(Collective type)** と呼ばれることがあります。

https://www.freecodecamp.org/news/typescript-literal-and-collective-types/

現時点最新の公式ドキュメントには記載されていませんが、古いバージョンでは Collective type について言及されています。あるいは Playground の [Literals のサンプル](https://www.typescriptlang.org/play/?q=69#example/literals)にも記載されています。

>**A literal is a more concrete sub-type of a collective type**. What this means is that "Hello World" is a string, but a string is not "Hello World" inside the type system.
>([TypeScript: Handbook - Literal Types](https://www.typescriptlang.org/docs/handbook/literal-types.html) より引用)

リテラル型は集合型の具体的な **subtype(部分型、派生型、下位型)** である旨が記載されていますね。

ちなみに subtype とは型の親子関係のようなもので、ある型の下位の型、つまり子の型として定義されたものを subtype と呼びます。suptype の派生元、つまり親の型となるものは **supertype(基本型、上位型)** と呼ばれます。オブジェクト指向における継承のようなものです。TypeScript で採用されているこういった型の仕組みを Subtyping と呼ばれるそうです。

https://ja.wikipedia.org/wiki/%E3%82%B5%E3%83%96%E3%82%BF%E3%82%A4%E3%83%94%E3%83%B3%E3%82%B0_(%E8%A8%88%E7%AE%97%E6%A9%9F%E7%A7%91%E5%AD%A6)

実際、型は値(各リテラル)の集合であり、具体的な文字列の値はすべての文字列を集めた `string` 型の要素として考えることができます。つまり、具体的な文字列リテラルによってつくられる１つの文字列リテラル型は `string` 型という集合の要素としてみなせます。

>Type 型とは：型とは、値の集合であり、その集合に対して実行できることの集合である。
>少しわかりにくいと思うのでいくつか例を示しましょう。
>
>- boolean type は、全ての boolean 値（といっても二つしかないが。true と false の二つである）の集合であり、この集合に対して実行できる操作の集合である。
>- number type は全ての数値の集合であり、この集合に対して実行できる操作の集合である(例えば `+, -, *, /, %, ||, &&, ?`)である。これらの集合に対して実行できる操作には、.toFixed, .toPrecision, .toString といったものも含まれる。
>- string type は全ての文字列の集合であり、それに対して事項できる操作の集合である。(例えば `+ , || , や &&` ) .concat や .toUpperCase などが含まれる。
>
>([合法 TypeScript 第3章 Type の全て](https://uncle-javascript.com/valid-typescript-chapter3) より引用)

そして、集合型(Collection type)に対して、単位型(Unit type)という概念もあることが数値リテラル型などのプルリクエストで言及されています。

>All literal types as well as the `null` and `undefined` types are considered **unit types**. **A unit type is a type that has only a single value**.
>([Number, enum, and boolean literal types by ahejlsberg · Pull Request #9407 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/pull/9407) より引用、太字は筆者強調)

単位型(Unit type)は、**単一の値のみを持つ型**であり、すべてのリテラル型は `null` 型や `undefined` 型と同じく単位型であると見なされるとのことです。

https://en.wikipedia.org/wiki/Unit_type

`string` 型は単位型である文字列リテラル型の集合型であり、各文字列リテラル型は `string` 型の subtype ということです。これは他のリテラル型とその型を Widening した集合型にも言えます。実際、`boolean` 型は `true` と `false` という真偽値リテラル型のユニオン型、つまり `true | false` という型と等しいことも明言されています。

>The predefined `boolean` type is now equivalent to the union type `true | false`.
>([Number, enum, and boolean literal types by ahejlsberg · Pull Request #9407 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/pull/9407) より引用)

あるいは Handbook の『[TypeScript for Functional Programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#unit-types)』の項目にも記載されていました。

>**Unit types are subtypes of primitive types that contain exactly one primitive value**. For example, the string "foo" has the type "foo".
>(中略)
>When needed, the compiler widens — **converts to a supertype** — the unit type to the primitive type, such as "foo" to string. This happens when using mutability, which can hamper some uses of mutable variables:
>([TypeScript: Documentation - TypeScript for Functional Programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#unit-types) より引用)

Unit type は単一のプリミティブ値を持つプリミティブ型の subtype であり、文字列リテラル型 `"foo"` は `string` 型の subtype である旨が記載されていますね。そして、mutable な場所で Widening が起きるときには subtype から派生元の supertype へと変換されるとも書いてあります。

# 型の集合性

Unit type の集合が Collective type(具体的には `string` などのプリミティブ型) であったわけですが、型が具体的な値の集合であるということは、Microsoft Developers の以下の Youtube の動画にて TypeScript の開発者である Anders Hejlsberg 氏(この記事や前の記事で参照しているプルリクエストは大体この方の作成)による公演でも説明されていました(26:32~あたりから)。

https://www.youtube.com/watch?v=hDACN-BGvI8&t=1592s

公演の解説を参考に図を作成してみました。

まず、型は各リテラル値によって作られるリテラル型(Unit type)によって構成されています。これが集合の元となります。`null` 型や `undefined` 型も単一の値によって作られるので Unit type です。そして同じ種類のリテラル型の集合が Collective type たる `string` 型や `number` 型を構成します。

![集合の元](/images/typescript-widen-narrow/img_typeSet_1.png)

ユニオン型によって型の和集合を作成できます。例えば `0 | 1 | true | "a"` のような数値リテラル型と真偽値リテラル型と文字列リテラル型によるユニオン型は以下のような集合を作成します。

![和集合](/images/typescript-widen-narrow/img_typeSet_2.png)

`number | string | undefined` のようにいくつかの Collective type や Unit type の和集合を作ることもできます。

![集合型の和集合](/images/typescript-widen-narrow/img_typeSet_3.png)

空集合は `never` 型であり、各リテラル型の積集合や異なる集合型の積集合をインターセクション型で作ろうとすると `never` 型となります。そして型の全体集合は `unknown` 型となります(`unknown` 型がなぜ全体集合になるかは後述する階層性で見れば分かります)。

![全体集合](/images/typescript-widen-narrow/img_typeSet_4.png)

和集合(Union type)と積集合(Intersection type)はオブジェクトの型で考えると分かりやすいです。分かりやすいベン図になります。

![オブジェクトの型](/images/typescript-widen-narrow/img_typeSet_5.png)

`{ a: "st" }` という型と `{ b: 42 }` という型の和集合(Union type)と積集合(Intersection type)が上の図になるのは以下のコードで分かります。ユニオン型が和集合なのでそれぞれの集合を受け入れますが、インターセクション型は共通要素のみしか受け入れません。

```ts
type A = { a: "st" };
type B = { b: 42 };

type Union = A | B;
type Intersection = A & B;

const vAandB: Intersection = { a: "st", b: 42 };
const vOrB1: Union = { a: "st" };
const vOrB2: Union = { b: 42 };
const vOrB3: Union = { a: "st", b: 42 };
```

ちなみに `{ a: "st" }` という型が `{ a: "st"; c: number }` など任意の他のプロパティを持つ型の集合であることは集合で考えれば分かりやすいのですが、実際にコードで書こうとするとエラーがでてそうではないのではないかと疑問に思ってしまうことがあります。

```ts
type A = { a: "st" };

const vA: A = { a: "st", c: "anything" };
//                       ^^^^^^^^^^^^^: [Error]
// Object literal may only specify known properties, and 'c' does not exist in type 'A'.
```

実はこれは型システムにおける違反検知の型チェックエラーではなくて、**余剰プロパティチェック(execc property cehecks)** という別のエラーです。TypeScript の型システム的には問題なくても、エラーとして検知してくれるものです。型システム上は問題ないので、別の変数を経由させることでエラーとならなくなります。

```ts
type A = { a: "st" };

const obj = { a: "st" as const, c: "anything" };
const vA: A = obj; // エラーとならないで受け入れられる
```

ということで、`{ a: "st" }` という型は `{ a: "st"; c: number; d: string }` など `a: "st"` 以外の任意のプロパティを持つオブジェクトの型の集合であることが分かります。

剰余プロパティチェックについては以下の uhyo さんの記事が非常に分かりやす解説されていたので参考にしてください。

https://qiita.com/uhyo/items/b1f806531895cb2e7d9a


# 型の階層性

subtype や supertype という関係から分かる通り、型には親と子の関係があり、階層性があります。すべての型の最上位となる親の型は TypeScript では `unknown` 型であり、[型理論(Type theory)](https://en.wikipedia.org/wiki/Type_theory)ではこのような型を **Top type** と呼ぶそうです。

https://en.wikipedia.org/wiki/Top_type

逆に最下位となる型は TypeScript では `never` 型であり、型理論ではこのような型を **Bottom type(ボトム型、ゼロ型、空型)** と呼ぶそうです。

https://en.wikipedia.org/wiki/Bottom_type

公式 Handbook の『[TypeScript for Functional Programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#other-important-typescript-types)』の項目でも `unknown` 型が top type で `never` 型が bottom type であると明示されています。

![unknown & never type](/images/typescript-widen-narrow/img_ts_handbook_toptype_bottomtype.jpg)*[TypeScript for Functional Programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#other-important-typescript-types) より引用*

型の集合性で見たとおり、型は値の集合なので、TypeScirpt のリテラル型は単一の値からなる単集合で、`never` 型は値を持たないので空集合です。全体集合を `unknown` 型としてみなすと以下のような図が書けました。

![全体集合](/images/typescript-widen-narrow/img_typeSet_4.png)

それぞれの型は部分集合であり、ユニオン型(`A | B`)は `A ∪ B` という和集合で、インターセクション型(`A & B`)は `A ∩ B` という積集合としてみなせます。

![積集合と和集合](/images/typescript-widen-narrow/img_typeSet_3.png)

参考文献
https://blog.logrocket.com/when-to-use-never-and-unknown-in-typescript-5e4d6c5799ad/

そして、subtype と supertype の関係を辿ると以下のような型の階層図(Type hierarchy)もできあがります。ただし、以下の図は mermaid で記述したものですが、以下の図は完全に正確ではないかもしれないので注意してください(複数の文献を参考にして作成してますが、TypeScript のバージョン更新によって古い階層図と変わっているところなどもあるので)。また、`enum` などの型は JS に存在しない TS の独自機能なので意図的に排除しています。

```mermaid
graph LR
  U[unknown]
  N[never]
  A[any]
  O["Object, { }"]
  obj[object]
  U --> A
  A --> void --> undefined --> N
  A --> null --> N
  A --> O --> Number & String & Boolean & BigInt & Symbol & obj
  subgraph Primitive
    subgraph Unit[Unit type]
      undefined
      null
      nl
      sl
      bl
      bil
      us
    end
    subgraph Col[Collective type]
      number
      string
      boolean
      bigint
      symbol
    end
    subgraph Wrap[Wrapper]
      Number
      String
      Boolean
      BigInt
      Symbol
    end
    Number --> number --> nl[number literal]
    String --> string --> sl[string literal]
    Boolean --> boolean --> bl[boolean literal]
    BigInt --> bigint --> bil[bigint literal]
    Symbol --> symbol --> us[unique symbol]
  end
  nl[number literal] --> N
  sl[string literal] --> N
  bl[boolean literal] --> N
  bil[bigint literal] --> N
  us[unique symbol] --> N
  obj --> Function --> N
  obj --> ReadonlyArray --> Array & RT[readonly Tuple] --> Tuple --> N
```

左が supertype で、右が suptype の方向となります。subtype は supertype の型の変数へ代入可能です。

>スーパータイプは、そのサブタイプの数々によって代替/代入可能とされており、これは代入可能性（substitutability）と呼ばれる。そのスーパータイプとサブタイプの関係は、[is-a](https://ja.wikipedia.org/wiki/Is-a)とも言われる。記号 `<:` を用いて `subtype <: supertype` と表記される。
>([サブタイピング (計算機科学) - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%B5%E3%83%96%E3%82%BF%E3%82%A4%E3%83%94%E3%83%B3%E3%82%B0_(%E8%A8%88%E7%AE%97%E6%A9%9F%E7%A7%91%E5%AD%A6)?oldformat=true) より引用)

そして、Widening(型の拡大) が起きる方向は子から親、つまり subtype → supertype の方向であり、代入可能となるのも subtype → supertype の方向で、その逆は型エラーとなります。

```ts
const literal = "text" as const;
let str: string;
let strWrapper: String;
let myObject: Object;
let myAny: any;
let myUnknown: unknown;

// subtype → supertype で代入していくと型エラーにならない
str = literal;
strWrapper = str;
myObject = strWrapper;
myAny = myObject;
myUnknown = myAny;
```

この図と Handbook の『[Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html#any-unknown-object-void-undefined-null-and-never-assignability)』の図を見比べると subtype → supertype で代入可能である一方で、supertype → subtype で代入できないとういうのが上の階層図と一致しているので納得できます(`any` 型は例外)。

![型の互換性](/images/typescript-widen-narrow/img_ts_type_compatibility.png)*[Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html#any-unknown-object-void-undefined-null-and-never-assignability) より引用*

型の互換性(compatibility)と呼ばれるルールによって代入可能であることや subtype の関係性が定められていますが、代入可能であること subtype であることは近似していますが、微妙に異なるということが Handbook の以下の箇所に記載されています。

>So far, we’ve used “compatible”, which is not a term defined in the language spec. In TypeScript, **there are two kinds of compatibility: subtype and assignment**. These differ only in that **assignment extends subtype compatibility with rules to allow assignment to and from any**, and to and from enum with corresponding numeric values.
>([TypeScript: Documentation - Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html#subtype-vs-assignment) より引用、太字は筆者強調)

subytype 互換性を拡張したものが代入(assignment)可能性であり、具体的には、`any` 型から様々な型に代入できることと `any` 型に様々な型を代入できるというルールが追加されていることが記載されていますね(`enum` についての言及は無視しています)。

実際、`any` 型は型チェックを放棄するので、例外的にすべての型に代入可能であり、自身の subtype である型にも代入できます(上の代入可能であるかの図でもそうなっていますね)。ただし Bottom type である `never` 型には `never` 型以外は何も代入できなので `any` 型でも代入することはできません。

```ts
// any 型は型チェックしなくなるので assignable の概念もなくなってすべての型の変数に代入できてしまう
const test = 42 as any;
let myany: undefined = test;
let str: string = test;

// ただし Bottom type である never 型には代入できない
let mynever: never = test; // [Error]
// Type 'any' is not assignable to type 'never'
```

`never` 型は本来は「値を持たない」ということを表現する型なので、`never` 型には `never` 型しか代入できないということを検証するには型アサーションで `never` 型としてあげることで可能です。また、`never` 型は最下層の subtype つまり Bottom type なので `never` 型からみればあらゆる型が supertype となり代入可能です(各リテラル型も supertype です)。

```ts
// 通常 never 型は値を持たないはずだが、型アサーションで never 型にできて、never 型に代入できる
let nev: never = 1 as never;

// never 型は bottom type なのでどんな型にも代入できる
let str: string = nev;
let strArray: string[] = nev;
let strLiteral: "text" = nev; // リテラル型にも代入可能
let myAny: any = nev;
let mynever: never = nev; // never 型自身に代入できる
```

参考文献
https://knmts.com/as-a-engineer-52/
https://qiita.com/dico_leque/items/06ac5837b7a333c5c8da
https://gist.github.com/laughinghan/31e02b3f3b79a4b1d58138beff1a2a89
