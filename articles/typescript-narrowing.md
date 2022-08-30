---
title: "TypeScript の Narrowing"
emoji: "💃"
type: "tech"
topics: ["typescript"]
published: true
date: 2022-07-29
url: "https://zenn.dev/estra/articles/typescript-narrowing"
tags: [" #type/zenn "]
aliases: [記事_TypeScript の Narrowing]
---

# はじめに

今回の記事では、その Widening(型の拡大) の対となる Narrowing(型の絞り込み) について解説します。Narrowing の方がよく知られている概念であり、パターンが多く解説が大変ですがやっていこうと思います(一部書ききれていないパターンや理解度の低いものもありますが後から追記していきます)。

基本的には [TypeScript 公式 Handbook](https://www.typescriptlang.org/docs/) の以下の『Narrowing』のページを参照して解説しています。

https://www.typescriptlang.org/docs/handbook/2/narrowing.html

最新の公式ドキュメント(v2)は**非常にわかりやすい構成でシンプルに TypeScript を理解できるような内容**になっていますので英語であっても絶対に読むべきおすすめの学習リソースですが、ここに今までの『[Widening(型の拡大)](https://zenn.dev/estra/articles/typescript-widening)』という対になる概念と『[型の集合性](https://zenn.dev/estra/articles/typescript-type-set-hierarchy)』などを加えることでよりスッキリと理解することが可能になります(特に判定可能なユニオン型などについて)。

# ユニオン型と型集合

[前回の記事](https://zenn.dev/estra/articles/typescript-type-set-hierarchy)では、型は以下の図のように具体的な値の集合であると解説しました。単位型(Unit type)である具体的な値から作られるリテラル型の集合によって集合型(Collective type)たる `string` 型や `number` 型、`boolean` 型などのプリミティブ型が構成されます。

![unit vs collective](/images/typescript-widen-narrow/img_typeSet_1.png)

そして、あらゆる型は、全体集合となる `unknown` 型の部分集合であり、`never` 型は空集合としてみなすことができます。

![全体集合](/images/typescript-widen-narrow/img_typeSet_4.png)

各リテラル型の積集合や異なる集合型(Collection type)の積集合をインターセクション型で作ろうとすると共通要素が全く無いので `never` 型となりました。また、`never` 型は空集合ということで、`never` 型そのものと他の型との和集合をユニオン型で作成すると `never` 型は無かったかのようにユニオン型の構成要素として使用した要素の型そのものとなります。

```ts
type StrOrNever = string | never;
// type StrOrNever = string; と同じ
```

`string` 型や `number` 型などのプリミティブ型を合成してユニオン型を作ると以下の図のようにそれぞれの構成要素の型(集合)を合成した和集合を作ります。

![集合型の和集合](/images/typescript-widen-narrow/img_typeSet_6.png)

この記事のメインテーマである **Narrowing** とは「型の絞り込み」のことですが、「型を絞り込む」というのは特定の変数が上記のようなユニオン型からより扱いやすい具体的なプリミティブ型や特定のオブジェクトリテラルの型へと範囲を絞り込んでいくプロセスや行為のことを指します。

:::message
TypeScript ではこのように「**型を集合として扱う**」ことが非常に重要となります。型を集合論的に扱いやすくする機能を提供するように TypeScript 自体がデザインされていると[公式ドキュメントで明言されて](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-oop.html#types-as-sets)いますが、その中でもユニオン型は非常に重要な機能です。

Narrowing については冒頭で紹介したように公式ドキュメントでわざわざ１ページもさかれて解説されており、特にユニオン型から特定の型へと絞り込む方法が丹念に解説されているのでどれだけ重要かは想像が付きます。
:::

# Narrowing の必要性

具体的に Narrowing がどのようなものかを見てみます。`number | string` という２つのプリミティブ型から構成されるユニオン型を引数として受け入れる関数を考えます。関数内では、例えば次のような `if` 文などで条件判定して特定のブランチ内でユニオン型よりも具体的な特定のプリミティブ型であると絞り込むことができます。

```ts
// number | string 型のみを引数として受け入れる関数
function narrowUnion(
  param: number | string
): void {
  if (typeof param === "string") {
    // param はこのブランチで string 型であると絞り込まれた
    console.log(param.toUpperCase());
    //          ^^^^^: string 型
  }
  else if (typeof param === "number") {
    // param はこのブランチで number 型であると絞り込まれた
    console.log(Math.floor(param));
    //                     ^^^^^: number 型
  }
}
```

`if` や `else if` の各ブランチで変数の型をユニオン型ではなく具体的なプリミティブ型として絞込んでいるのでその型で使えるプロトタイプメソッドや静的メソッドを利用できるようになります。

このような型の絞り込みを行わなかった場合にどうなるか見てみましょう。絞り込みのための if 文を無くしてそれぞれのメソッドを利用しようとすると型エラーとなります。

```ts
function narrowUnion(
  param: number | string
): void {
  console.log(param.toUpperCase())
  //                ^^^^^^^^^^^ [Error]
  // Property 'toUpperCase' does not exist on type 'string | number'.
  // Property 'toUpperCase' does not exist on type 'number'.

  console.log(Math.floor(param));
  //                     ^^^^^ [Error]
  // Argument of type 'string | number' is not assignable to parameter of type 'number'.
  // Type 'string' is not assignable to type 'number'
}
```

`string | number` というユニオン型は `string` 型と `number` 型の和集合であり、この関数はそれぞれの型の変数を受け入れます。そして、この関数の引数として渡す変数が具体的な値であるときには結局は `string` 型の値か `number` 型の値のどちらかです。

渡した変数が `string` 型であったときには `Math.floor()` は使えませんし、逆に `number` 型であったときには `toUpperCaser()` メソッドは使えません。静的メソッドである `Math.floor()` に `string` 型を渡した場合には `NaN` が得られますが、`number` 型に対して `toUpperCase()` メソッドを呼び出そうとすると確実にエラーとなります。

これは JavaScript として記述して実行すれば分かることです。JavaScript だと TypeScript の時にエディタ上で得られた上記のような型エラーがでてこないので実行するまでエラーになるかどうかわかりません。

```js:JavaScript
function cantNarrowUnion(param) {
  console.log(param.toUpparCase());
  console.log(Math.floor(param));
}

cantNarrowUnion(42.3);
cantNarrowUnion("str");
```

TypeScript での型の利便性を知っている状態だとこれは非常に恐ろしいですね。文字列型も数値型も引数として受け入れう場合には型を絞込んでからその型で利用できるメソッドを使うようにしないとエラーになります。

型の絞り込む必要があるのは、上記だと `string | number` というそれぞれで違う操作体系を持つ集合の和集合として型をつくってしまっているためです。`string` 型と `number` 型では扱えるプロトタイプメソッドやその型の変数に対して加えることのできる操作などが変わってくるために場合分けをする必要がでてきます。

`string | number` という和集合をつくった時に共通して利用できるプロトタイプメソッドは `toString()` や `toLocaleString()`、`valueOf()` などの限定されたものしかありません。これらのメソッドは `String.prototype.toString()` と `Number.prototype.toString()` といいようにそれぞれの同じ名前のプロトタイプメソッドとして定義されているので共通して利用できます。そのような同じ名前のメソッドではないものを使いたい場合(大半の場合)は型の絞り込みを行って場合分けする必要があるわけです。

わかりやすいプリミティブ型のユニオン型だけではなく、TypeScript では `undefined` とのユニオン型がよく出現します。オプション引数やオプショナルプロパティなどを使うことで強制的に型注釈した型と `undefined` 型とのユニオン型としてみなされます。

```ts
// オプション引数を使った関数定義
function acceptOptionalStr(
  str?: string // string | undefined となる
) {
  // ...
}

acceptOptionalStr("text"); // string 型を渡せる
acceptOptionalStr(); // 引数は省略可能
```

:::message alert
オプション引数ではなく、引数の型注釈を直接的に `string | undefined` というように注釈すると引数が省略できなくなるという違いがあるので注意してください。
:::

`string | undefined` というユニオン型は `string` 型よりも広い集合(superset)を表現しており、`string` 型や `undefined` 型の supertype となります。広いといっても実際には具体的な `undefined` リテラルから構成される Unit type である `undefined` 型を `stirng` 型に加えただけです。

この場合に `string` 型で利用できるプロトタイプメソッド `toUpperCase()` を関数内で利用しようとすると型エラーになります。もちろん、`undefined` にはプロパティとして `toUpperCase()` というようなメソッドを持たないからです。

```ts
function acceptOptionalStr(
  str?: string // string | undefined となる
) {
  console.log(str.toUpperCase());
  //          ^^^ [Error]
  // Object is possibly 'undefined'.
}

acceptOptionalStr("text"); // string 型を渡せる
acceptOptionalStr(); // 引数省略した際には自動的に undefiend となりエラーとなる
```

このコードは実行時にエラーとなります。これを回避するにはいくつか方法はあると思いますが、例えば ES2020 で ECMAScript の新機能として導入された [Optional chaining 演算子](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Optional_chaining)(`?.`)を使うことで回避できます。これを使うことで `undefined.toUpperCase()` のような存在しないプロパティアクセスによる例外発生を回避して `undefined` を返すことができます。

```ts
function acceptOptionalStr(
  str?: string // string | undefined となる
) {
  console.log(str?.toUpperCase());
  //             ^^ optional chaining 演算子
}

acceptOptionalStr("text"); // string 型を渡せる
//  => "TEXT"
acceptOptionalStr(); // 引数省略可能
//  => undefined
```

Optional chaining 演算子を使わずに、`number | string` ユニオンで利用したように `typeof` 演算子を `if` 文の条件として利用して型の絞り込みするなどももちろん可能です。あるいはオプション引数ではなくデフォルト引数とすることででそもそも `undefined` が入り込まないようにするなどの方法もありえます。

```ts
{
  const acceptOptionalStr = (
    str?: string // string | undefined
  ) => {
    if (typeof str === "string") {
      console.log(str.toUpperCase());
    }
  };
}

{
  const acceptOptionalStr = (
    str?: string // string | undefined
  ) => {
    if (typeof str === "undefined") {
      console.log(undefined);
    } else {
      console.log(str.toUpperCase());
    }
  };
}

{
  const acceptOptionalStr = (
    str = "str" // デフォルト引数
  ) => {
    console.log(str.toUpperCase());
  };
}
```

とにかく、ユニオン型となる場合には構成要素となる複数の型同士で共通して使えるメソッドはかなり限定的になるため関数内部では受け取る引数の型を絞り込んで場合分けする必要がでてきます。

オプション引数のように明示的にユニオン型としなくてもユニオン型となる場合があるので型の絞り込み(Narrowing)が重要であることが理解できたと思います。

# 型範囲の拡大縮小

ユニオン型(Union type)は複数の型を合成した型です。集合論的には複数の集合の和集合(合併: Union)となります。逆にインターセクション型(Intersection type)は集合論的には積集合(共通部分、交差: Intersection)となります。

例えば `string | number` などがユニオン型ですが、これは `string` 型または `number` 型という２つの型を受け入れる合成された型です。このように２つの型を組み合わせることを「**型の合成(Composing Types)**」と呼びました。

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

ここで Widening の復習をしておきますが、次のように三項演算子を使った上で変数の初期化を行った場合には、`const` 宣言なら具体的なリテラル型のユニオン型として型推論され、`let` 宣言なら一般的な `string` や `number` 型のユニオン型として拡大(Widening)されて型推論されます。

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

Literal Wideing ではこのように具体的な文字列リテラル型(Unit type)から一般的なプリミティブ型である `string` 型(Collective type)へと型が拡大されました。

集合論的には各文字列リテラル型はその特定の文字列リテラル値によって成る単集合(singleton)あるいは単位集合(unit set)であり、その文字列リテラル型の集合が `string` 型を構成しています。

従って Widening では、対象となる集合が subset から superset へと範囲が拡大されることになります。supertype-subtype の関係性で考えると、subtype である文字列リテラル型から supertype である `string` 型へと拡大されます。逆に Narrowing では、対象となる集合を superset から subset へと絞り込むことになります。supertype-subtype の関係性で考えると、supertype であるユニオン型から subtype である `string` 型や `number` 型へと絞り込みます。

図にすると以下のような関係性となります。

![narrowing_widening](/images/typescript-widen-narrow/img_typeSet_8.png)

Widening では基本的にリテラル型から一般的なプリミティブ型への拡大を考え、Narrowing ではユニオン型から特定のプロトタイプメソッドなどが使えるようになるプリミティブ型への絞り込みを考えます。

Narrowing では、例えば `toUpperCase()` というメソッドは `string` 型でしか使えないので `string | number` などのユニオン型から型の候補を減らし(reduce)て、型を `string` 型まで絞り込みます。`number` 型でしか使えないメソッドを使いたいなら `number` 型まで絞り込みます。

# 制御フロー解析(CFA)

ということで、ユニオン型が関数の引数となることで、関数内部で引数に対して利用できるメソッドがそのユニオン型に含まれる型によって変わってくるので場合分けをする必要がでてきます。

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

こういったコードの構造に基づいて値の型をより具体的に推定できるようにすることを(型の範囲をより具体的なものに狭めることから) Narrowing(型の絞り込み)と呼びました。

そして実際には、上のコードでの `if` 節や `switch` や `while` などのコードの構造によって各場所での変数の型を絞り込みます。このようなコードを書くと TypeScript (コンパイラやエディタの拡張機能)はある変数が特定のブランチなどに到達した時点でその型がなんであるか解析をしています。この解析を「[制御フロー解析(Control flow analysis: CFA)](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#control-flow-analysis)」と呼びます。

この CFA ですが、TypeScript 公式のチートシートの１つとして以下のページでまとめられています。

https://www.typescriptlang.org/static/TypeScript%20Control%20Flow%20Analysis-8a549253ad8470850b77c4c5c351d457.png

CFA ではユニオン型の変数の型をいくつかの真偽値のロジックパターンに基づいて型を絞り込んでいきます。基本的には、`if` 節で条件判定しますが、`switch` などを使う場合もあります。

```ts
// このブランチ内で変数を string 型として絞り込む
if (typeof input === "string") {
  console.log(input);
  //          ^^^^^^ string 型として CFG で解析される
  console.log(input.toUpperCase());
  // このブランチでは string 型データのプロトタイプメソッドなどが利用できる
} else {
  // このブランチ以降は型の候補から string 型が外される
}
```

# 判別可能なユニオン型

:::message
この記事では、Narrowing の基本パターンから始めるまえに集合に基づいて判定可能なユニオン型から解説します。

基本パターンについて一部未完成の箇所があるのも理由の１つです。すべて補ったら再構成するかもしれません。
:::

判別可能なユニオン型(Discriminated union type)あるいはタグ付きユニオン型(Tagged union type) は型システム一般では [Sum 型](https://www.wikiwand.com/en/Sum_type)と呼ばれる類のものであり、$\sigma+\tau$ として表記されます。

![型システムでの Sum 型の表記](/images/typescript-widen-narrow/img_typeSystem_notation_2.jpg)*[Type system - Wikipedia](https://en.wikipedia.org/w/index.php?title=Type_system#Specialized_type_systems) より引用*

特殊なユニオン型であり、判別可能なユニオン型として定義しておくことで Narrowing がしやすくなるものです。

集合論で考えると、判別可能なユニオン型は [Disjoint union(非交和)](https://en.wikipedia.org/wiki/Disjoint_union?oldformat=true) と呼ばれるものになります。２つの集合の和集合をつくった時に共通部分がない、つまり交差(intersection)を持たない和集合のことを指します。"disjoint" とは互いに素であることを意味します。

![disjoin union](/images/typescript-widen-narrow/img_typeSet_11.png)

型は具体的な値の集合で、特にプリミティブ型は具体的なリテラル型の集合としてみなせした。`string` 型と `number` 型は共通部分がないので和集合をつくった際には共通部分がないので自動的に Disjoint union となります。積集合(交差)を作り出そうとインターセクション型で `string` 型と `number` 型を合成しようとすると空集合で値を持たないことを表現する `never` 型となります。

ということで、実は今までのユニオン型の図は正しくなく、交差を持たないのでより正確に図示すると右のようになります。

![交差を排除して表現](/images/typescript-widen-narrow/img_typeSet_9.png)

`string | number` のようなプリミティブ型のユニオン型を Narrowing する際にはすでに知っている `typeof` 演算子で判別すればよいので特に問題はありません。

```ts
type StrOrNum = string | number;
// disjoint union を作成する

function padLeft(pad: StrOrNum) {
  if (typeof pad === "string") {
    // string 型として CFA で絞り込まれる
  } else {
    // number 型として CFA で絞り込まれる
  }
}
```

実は、判別可能なユニオン型として知られているのはオブジェクトの型についてのユニオン型を考えるときのものです。要するにオブジェクトの型でのユニオン型の作り方のプラクティスの話となりあます。

異なるプリミティブ型同士をユニオン型として合成すると自動的に Disjoint union になりましたが、オブジェクト型をユニオン型として合成すると Disjoint union になるとは限りません。例えば、`{ a: "st" }` と `{ b: 42 }` というオブジェクトの型を合成すると以下の図のように両方のプロパティを持つ型が交差として出現します。

![オブジェクトの型合成](/images/typescript-widen-narrow/img_typeSet_5.png)

ということで、オブジェクト型を Disjoin union として合成してタグ付きユニオン型とするにはある方法を取る必要がでてきます。

例えばよくある例として図形情報を表現するオブジェクトの型を考えます。具体的には四角形(square)、三角形(triangle)、円(circle)の３つの種類の図形について考えます。図形オブジェクトを受け取ってプロパティとして持たせた属性情報から何かしらの計算をして値を返すような関数を作りたいとします。

この場合、引数の型 `Shape` をどのように定義するかというのが問題になります。

まずは悪い例として、次のように１つのオブジェクトの型の中にオプションプロパティを入れてたりする方法がありえます。`kind` プロパティに図形種類を文字列リテラル型のユニオン型で指定できるようにして、それぞれの図形でプロパティが指定できるようにオプションプロパティとしていますが、この方法は型の精度がよくなく、無駄に広い型となってしまっています。"square" タイプの図形が本来持たなくても良いプロパティをもたせてしまうことができますし、関数の引数にとって Narrowing する際に不都合がでてきます。

また、この方法だと図形の種類を追加したいときなどにも実は不便です。

```ts
type Shape = {
  kind: "square" | "triangle" | "circle";
  radius?: number; // "circle" の図形が持つべきプロパティ
  length?: number; // "square" の図形が持つべきプロパティ
  angle?: number;  // "triangle" の図形が持つべきプロパティ
};

const sORt: Shape = {
  kind: "square",
  lenth: 100,
  radius: 50, // "square" では無駄なプロパティ
};
```

一応後で説明しやすいようにこの `Shape` 型を次のように分解して定義しなおしておきます。ユーティリティ型 `Partical<Type>` を使えば型引数に指定したオブジェクト型のプロパティを optional にできますので、それをインターセクション型で合成して型を作成します。また、オプショナルプロパティではなくすべてのプロパティが必須となるような型 `Strict` も定義しておきます。

:::message
図示する関係上 `kind` プロパティの値の型名などを短くしたいので以下のように書き換えます。適宣読み替えてください。

- `"circle"` → `"A"`、`radius` → `r`
- `"square"` → `"B"`、`length` → `l`
- `"triangle"` → `"C"`、`length` → `l`
:::

```ts
type Kind = {
  kind: "A" | "B" | "C",
};

type Props = {
  r: number;
  l: number;
  a: number;
};

type Shape = Kind & Partial<Props>;
/*
type Shape =
  { kind: "A" | "B" | "C"; } &
  { r?: number; l?: number; a?: number; }
----------------------------------------
= {
  kind: "A" | "B" | "C";
  r?: number;
  l?: number;
  a?: number;
}
*/

type Strict = Kind & Props;
/*
type Strict =
  { kind: "A" | "B" | "C"; } &
  { r: number; l: number; a: number; }
--------------------------------------
= {
  kind: "A" | "B" | "C";
  r: number;
  l: number;
  a: number;
}
*/
```

この `Shape` 型と `Strict` 型は制約の強さとしては `Strict` 型の方が強く、集合的に考えても `Shape` 型の方が広い集合であり、`Strict` 型を包含します。したがって、`Shape` 型は `Strict` 型の supertype(superset) です(あとでまとめて図示します)。ちなみにこれは条件型を使っても判別できます。

```ts
type FirstIsSubType<T, U> = T extends U ? true : false;
type StrictIsSubTypeOfShape = FirstIsSubType<Strict, Shape>;
// true
```

`Shape` 型は良くない例ですが、このような方は次のようにオブジェクトの型をそれぞぞれの図形の型として分割定義した上でユニオン型として合成したほうが使いやすくなります。この方法で合成した型は `Sum` 型という名前にしておきます。

```ts
type A = {
  kind: "A"; // "circle"
  r: number;
};
type B = {
  kind: "B"; // "square"
  l: number;
};
type C = {
  kind: "C"; // "triangle"
  a: number;
};

type Sum = A | B | C;
```

`kind` という共通のプロパティを持つオブジェクト型でユニオン型を合成している点が重要です。ここでは `kind` プロパティの値の型がそれぞれ異なる文字列リテラル型として定義しています。これによってユニオン型として `A`、`B`、`C` の３つの型を合成すると、集合的には型は交差を持たない互いに素である和集合(Disjoint union)となります。共通部分が無いことから、この型に代入可能なのはユニオン型の構成要素そのもののいずれかとなります。

`Shape` と `Sum` の２つの型は明確に違いますので注意してください。`Sum` 型はタグ付きユニオン型(判別可能なユニオン型)です。

```ts
type Shape = {
  kind: "A" | "B";
  r?: number; // => number | undefined
  l?: number; // => number | undefined
  a?: number; // => number | undefined
};

type Sum =
  | { kind: "A"; r: number; }
  | { kind: "B"; l: number; }
  | { kind: "C"; a: number; };
```

判別可能なユニオン型(タグ付きユニオン型)は、特定のプロパティの値(リテラル型)から元の型(あるいは集合)を特定することができるので、「判別可能」というわけです。実際、これを使うことによってユニオン型から構成要素の型へと Narrowing して絞り込むことができます。

```ts
function handleShape(shape: Sum) {
  if (shape.kind === "A") {
    // A 型として絞り込まれる
    console.log(shape.r);
    //          ^^^^^^^: それぞれの型に存在するプロパティにアクセスができる
  } else if (shape.kind === "B") {
    // B 型として絞り込まれる
    console.log(shape.l);
    //          ^^^^^^^: それぞれの型に存在するプロパティにアクセスができる
  } else if (shape.kind === "C") {
    // C 型として絞り込まれる
    console.log(shape.a);
    //          ^^^^^^^: それぞれの型に存在するプロパティにアクセスができる
  }
}
```

これが元の `Shape` 型でも Narrowing できますが、その型定義から `kind` の値が `"A"` だっとしても `r` プロパティはオプショナルであり必ず存在しているとは限りませんので、プロパティアクセス時に `undefined` となる可能性があります。従って型の安全性としては `Sum` 型よりも低くなります。

それぞれの型を集合論的に図示すると以下のようになります。

![タグ付きユニオン型](/images/typescript-widen-narrow/img_typeSet_10.png)

集合の包含関係は図を見ればわかりますが、部分集合(subset)となる型がそれを包含している集合(superset)に対して subtype となります。

`Kind` 型が最も条件が緩いので集合としての範囲が大きく、それに更に条件制約を書けていくとより詳細な型となり subtype へと派生していきます。`Shape`、`Strict`、`Sum` の３つ型を比較してみると、最初に定義した `kind` 以外のプロパティがオプショナルな `Shape` 型は集合としてはかなり大きいことがわかります。つまり制約が緩いわけです。逆にすべてのプロパティを必須にした `Strict` 型はかなり制約がつよく小さいことがわかります。そして `Sum` 型はその中間に位置しており、条件としては `Shape` よりも強く、`Strict` よりも緩くなっています。

図から `Strict` も実は判別可能なユニオン型であると言えます。実際 `Strict` 型は `Sum` 型の subtype です。ですが、`r`、`l`、`a` の３つのプロパティがすべて必須となっているので、そもそも Narrowing しなくてもプロパティアクセスが可能です。ということは、図形の種類としてもつべきではないプロパティを必ず持たなくてはならないことになるので冗長で無駄ですし、モデルとしてもこの型は不適当です。

また `Sum` 型のような定義方法を使うことで別の種類の図形の型をユニオン型の要素として追加したいときに簡単に追加できます。

```ts
type D = {
  kind: "D"; // 例えば "star" という図形として想定
  s: number; // 中心店から端点までの距離
};

type Sum =
  | A // { kind: "A"; r: number; }
  | B // { kind: "B"; l: number; }
  | C // { kind: "C"; a: number; }
  | D // { kind: "D": s: number; }
```

関数内で Narrowing する際にも対応するブランチを増やせばよいだけです。

## Exausitve check

タグ付きユニオン型でそのように要素を増やした時に上の例では４つしかユニオン型の構成要素がないので見た目ですべてを網羅していることが理解できますが、構成要素が例えば１０を超えると見た目で判別するのは難しいですし面倒でしょう。

そこで `never` 型(集合的には空集合)を利用することでタグ付きユニオン型の構成要素となる型すべてを関数内で利用しているかどうかを見た目に頼らずチェックすることができます。

というのも、Narrowing はユニオン型という和集合から合成元の型(集合)を取り除いていく行為でもあります。すべての構成要素となる型を取り除くと空集合、つまり何も要素が無い集合となるので最終的には `never` 型となります。構成要素をすべて列挙しないと `never` 型にすることはできないので、この性質を利用してあえてエラーを起こして網羅しているかのチェックをするのが網羅性チェック(Exaustive check)です。

再度、タグ付きユニオン型である `Sum` 型を考えてみます。更に `E` 型を加えて見ましょう。

```ts
type E = {
  kind: "E";
  x: number;
};

// ５つの空から構成されるタグ付きユニオン型
type Sum = A | B | C | D | E;
```

この `Sum` 型を引数に取る関数無いで網羅性チェックをしてみます。`if` だと文字数が多くなるので `switch` でやってみます。

```ts
function handleShapeX(shape: Sum) {
  switch (shape.kind) {
    case "A": {
      // A 型として絞り込まれる
      console.log(shape.r);
      return;
    }
    case "B": {
      // B 型として絞り込まれる
      console.log(shape.l);
      return;
    }
    case "C": {
      // C 型として絞り込まれる
      console.log(shape.a);
      return;
    }
    case "D": {
      // D 型として絞り込まれる
      console.log(shape.s);
      return;
    }
    default: {
      const _exhaustiveCheck: never = shape;
      // Type 'E' is not assignable to type 'never'
      return _exhaustiveCheck;
    }
  }
}
```

`never` 型はすべての型の subtype であり Bottom type ですから自身以外の型から代入することはできません。従って、タグ付きユニオン型の変数に対して Narrowing の過程で型の候補を減らしていった時にすべての型候補を網羅していない場合には型エラーとなります。上の例では `E` 型を候補からへらしていないので `never` 型になっておらず型エラーが起きています。つま、網羅できていないことがわかります。

このように型エラーによってユニオン型の構成要素を網羅できていないことに気づくことでできます。次のように修正した結果、網羅していれば型エラーとなりません。

```ts
function handleShapeY(shape: Sum) {
  switch (shape.kind) {
    case "A": {
      console.log(shape.r);
      return;
    }
    case "B": {
      console.log(shape.l);
      return;
    }
    case "C": {
      console.log(shape.a);
      return;
    }
    case "D": {
      console.log(shape.s);
      return;
    }
    case "E": {
      console.log(shape.x);
      return;
    }
    default: {
      // ユニオン型の構成要素をすべて網羅しているので型エラーとはならない
      const _exhaustiveCheck: never = shape;
      return _exhaustiveCheck;
    }
  }
}
```

# Narrowing のパターン

## 代入による Narrowing

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

## キーワードを使った Narrowing

以下であげるような話題は Narrowing(型の絞り込み) よりも、Type guard(型ガード) という話題で解説されることが多いですが、Narrowing という目的に沿って解説した方が公式ドキュメントにも沿っているのでそうします。

### typeof 演算子を使った Narrowing

`typeof` 演算子によって変数の型を基本的な判定ができます。`typeof` 演算子で判定できるものは以下のような基本的な型となります。

- `"string"`
- `"number"`
- `"bigint"`
- `"boolean"`
- `"symbol"`
- `"undefined"`
- `"object"`
- `"function"`

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

## 🛠 Equality Narrowing

:::details 未完成
単純な(厳密)等価演算子や(厳密)不等価演算子での判定を型ガードに使って Narrowing することも可能です。

今までやってきた `typeof` 型ガードによる方法やタグ付きユニオン型における絞り込みも実はこれを駆使していました。
:::

## Truthiness Narrowing

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

## ユーザー定義型ガード関数による Narrowing

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
Type predicate の [predicate](https://en.wikipedia.org/wiki/First-order_logic)) とは日本語で言うと「述語」となります。数理論理学におけるタームから派生して利用されているようです。ここでは型についての情報を記述するための表記方法のようなものだと考えればよいです。
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