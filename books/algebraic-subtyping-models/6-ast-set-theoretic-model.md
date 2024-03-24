---
title: "集合論による模型"
cssclass: zenn
date: 2024-02-10
modified: 2024-02-10
AutoNoteMover: disable
tags: type/zenn/book, TypeTheory/Subtyping, TypeScript/type, math/algebra
aliases: AST本『集合論による模型』
---

## 集合論による型の取り扱い

前の章において半順序関係にこだわっていたのは、半順序を使うことで型の集合がいくつかの付加的な構造を持つものとしてみなすことができ、様々な名前を持つ概念が利用できるようになるからです。半順序が役に立つのは主に次の束論のところですが、まずは集合論から行きましょう。

さて、商集合 $\text{TYPES}$ について考えますが、関数型について除いた部分集合 $\text{TYPES'}$ を考えることにしましょう (もちろん再帰型についても除きます)。なぜかと言えば、関数型はこれから話す集合論的なモデルの都合上うまく解釈ができないからです。

:::message
関数型 (関数値) を対象として含めた部分型について、よりうまく集合論的に考えることができる「意味論的部分型 (semantic subtyping)」という型システムの分野があります。
:::

型が厳密には集合ではないことを考慮しても、そもそも型そのものや型の集まりが集合論的に扱えるかどうかは一般的に言えることではありません。

一方で、TypeScript は公式ドキュメントの「[Types as Sets](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-oop.html#types-as-sets)」の項目にも記載してある通り、かなり集合論的に扱えるようにデザインされています。Microsoft Developers の以下の動画にて TypeScript の開発者である [Anders Hejlsberg](https://en.wikipedia.org/wiki/Anders_Hejlsberg) 氏も型が集合として扱えることを語っています。

https://www.youtube.com/watch?v=hDACN-BGvI8&t=1592s

> This works -- this is really more like reasoning about sets of possible values. That is really what the type system is doing.

ということで、直感的な集合論で型についてのメンタルモデルを構築することに大きな問題はありません。集合論で扱えきれない部分については後述する圏論などを使って補います。

集合論的に扱えるとは、例えば、ユニオン型やインターセクション型が集合の演算としての和集合や交差などに相当する概念として使えることや、リテラル型とプリミティブ型の関係や、空集合や全体集合に相当する `never` 型や `unknown` 型が存在しているということです。

ユニオンやインターセクションなどの既存の演算が集合演算として扱えるだけでなく、現在提案されている型の演算として否定(negation)が導入されれば補集合やド・モルガンの法則などを考えることができるので、さらに数学的な集合に近づきますね。
https://github.com/microsoft/TypeScript/issues/4196

なお否定型については束論の章で再度扱うので頭の片隅においておいてください。

## 集合の濃度とユニット型

集合論を扱う上では「**濃度** ([cardinality](https://en.wikipedia.org/wiki/Cardinality))」という概念が重要となります。濃度とは**集合の要素数を一般化した概念**であり、特に無限集合同士の比較などで利用されます。

要素数が有限の有限集合において濃度は自然数で表現されます。例えば、$A = \lbrace 1, 2 \rbrace$ という集合の濃度は $|A| = 2$ などと表されます。

さて、型の理論において非常に重要な型として「**ユニット型** (unit type)」と呼ばれる型の種類が存在します。ユニット型とは要素が一つだけの型であり、集合として解釈する場合には濃度が $1$ の集合に相当します。なお、このような要素数が一つの集合は一点集合や、単集合(singleton)、単位集合(unit set)などと呼ばれます。

TypeScript におけるユニット型は `null` 型と `undefined` 型、そして文字列や数値などのプリミティブ型のリテラル型です。

```ts
const nu: null = null;
const un: undefined = undefined;
const nl: 1 = 1;
const sl: "st" = "st";
const tr: true = true;
```

これらの型はまさにそのリテラルの値だけを割当可能とする型であり、それ以外のあらゆる値の割当を拒絶します。

![濃度1の型](/images/ast/img_cardinality-1-set.png)

濃度が $1$ であるユニット型を用いることで濃度が $n$ (自然数)の型を生成することが可能となります。濃度 $2$ の型を生成するには濃度 $1$ の型同士を合成するればいいわけですが、例えば `null` と `undefined` という型を合成してみると、合成した型 `UN` は `null` と `undefined` という二つの値のみ持つ型となります。

```ts
// 濃度2の型
type UN = null | undefined;

let un: UN;
un = null;
un = undefined;
```

同様に `1` と `2` という濃度 $1$ の数値リテラル型を合成した場合は、`1` と `2` の値を持つ濃度 $2$ の型が生成されます。

```ts
// 濃度2の型
type OneTwo = 1 | 2;

let ot: OneTwo;
ot = 1;
ot = 2;
```

少し特殊な例として `true` と `false` という二つの真偽値リテラル型を合成してみましょう。

```ts
// 濃度2の型
type Bl = true | false;
```

このように合成した型はもちろん濃度 $2$ の型となりますが、合成結果の型は `boolean` という濃度 $2$ の型と同一となります。

```ts
type R = Relation<Bl, boolean>;
// => Identical
```

元々 `boolean` という型は `true` と `false` の二つの値しかない型なのでそのリテラル型を合成した結果の型も `boolean` となるわけです。この型を集合論的に図示すると以下のようになります。

![濃度2の型](/images/ast/img_cardinality-2-set.png)

濃度が $1$ の場合と $2$ の場合を見てきましたが、濃度が $0$ の場合はどのようになるでしょうか。濃度が $0$ の集合はまさに要素数が $0$ なので空集合(empty set) $\phi$ に相当します。型の理論において要素を全く持たない型は空型(empty type)と呼ばれ、TypeScript では `never` 型が空型に相当します。

```ts
let n1: never;
n1 = 1; // => Error: 型 'number' を型 'never' に割り当てることはできません。
```

`never` 型は要素をまったく持たないのであらゆる値の割当を拒絶します。`never` への割当可能なのは自分自身、つまり `never` 型の値ということになります。これは順序理論でいえば反射律です。自分自身の型の値のみが割当可能といっても `never` 型には値そのものがないため(そもそも値が無いことを表現する型)、反射律を検証するには下記のように型アサーションを利用してやるしかありません。

```ts
// 濃度0の型
let n2: never;
// 型アサーションでコンパイラを騙す
n2 = 1 as never;
```

さて、`boolean` 型がその構成要素たるすべての真偽値リテラル型から構成されたように、`string` や `number` といった他のプリミティブ型もその構成要素となるすべてのリテラル型(あるいは値そのもの)をかき集めた集合としてみなすことができます。

![リテラル型の集合](/images/typescript-widen-narrow/img_typeSet_1.png)

ちなみに `boolean` 型は `true` と `false` という二つの値のみからなる型だったので濃度が２になりましたが、`number` や `string` の濃度はどうなるでしょうか。代表的なプリミティブ型の濃度は以下のようになっています。

プリミティブ型 | 値要素の数としての濃度
--|--
`null` | $1$ (ユニット型)
`undefined` | $1$ (ユニット型)
`boolean` | $2$ (`true`, `false`)
`number` | 限りなく大きいが有限
`string` | $\infty$

`number` 型の濃度が無限ではないのは、数値型は実際には他の言語でいうところの `double` 型であり[表現可能な値の範囲が定まっている](https://qiita.com/uhyo/items/f9abb94bcc0374d7ed23)からです。

話を戻して、このようにプリミティブ型はそのリテラル型を集合として包含していると考えることができます。そして、`boolean` という型は `true` と `false` という二つの値を割当できたので、`boolean` 型はこの二つのリテラル型の上位型(supertype)とみなせます。

```ts
type R1 = Relation<boolean, true>;
// => Supertype
type R2 = Relation<boolean, false>;
// => Supertype
```

逆に包含される方の型 `true` と `false` は包含する型 `boolean` の部分型(subtype)となります。

## 部分型関係と包含関係

上記のように考えていく、そもそもリテラル型がプリミティブ型の部分型であり、プリミティブ型がトップ型 (`unknown`) の部分型であったように、それぞれの型を集合として解釈すれば部分型関係は集合の「包含関係 (inclusion)」として解釈できるようになります。

![型全体の包含関係](/images/typescript-widen-narrow/img_typeSet_8_sub.png)

:::message alert
部分型関係は厳密には包含関係ではなく、「型 $S$ が型 $T$ の部分型であるとき ($S <: T$ と表記する)、$T$ 型の値が期待される場所で安全に型 $S$ の値を使用できる」というような型の(順序)関係です。関数型になるとそのような関係についての本質的な理解が必要となり、変性の概念と相まってシンプルな包含関係で捉えきれなくなります。

関数型の部分型関係については後ほど圏論の章で取り扱います。
:::

つまり、型 $S$ と $T$ の間に部分型関係 $S <: T$ があったときにはそれぞれが対応する集合として $S \subseteq T$ ($S$ が $T$ の部分集合である) が成り立ちます。ここでは更に二つの型が同値 (equivalent) であるならば、同じ集合であるとしましょう。具体的には以下のオイラー図のように集合の包含関係が部分型関係に対応するようにして $S \subseteq T\ \Leftrightarrow S <: T$ と考えます。

![型の包含関係と部分型関係](/images/typescript-widen-narrow/img_typeSet_6.png)

## 和集合と共通部分

ユニオン型(`|`)とインターセクション型(`&`)は論理学的に言えば型についての論理和(disjunction)と論理積(conjunction)を表現する演算です。

一方で TypeScript の型を集合として解釈するとき、ユニオン型は集合の和集合(union)の演算に相当し、インターセクション型は集合の共通部分(intersection)の演算に相当します。

![和集合](/images/typescript-widen-narrow/img_typeSet_2.png)

話が少し変わりますが、Haskell や OCaml で「または」という論理和を表現するにはバリアント型(variant type)を利用します。バリアント型は例えば以下のように書きますが、バリアント型は集合的には共通部分がありません。

```hs
type tree =
  Leaf of { value: int } |
  Node of { left: tree; right: tree }
```

バリアント型は和を表す型の一種ですが、共通部分がないというのは以下の図の右のように表現する領域に共通部分がありません。このような集合を「互いに素な合併(disjoint union)」あるいは「非交和」と呼びます。

![Disjoint union](/images/typescript-widen-narrow/img_typeSet_11.png)

バリアント型は常にこのような非交和を表現しますが、TypeScript におけるユニオン型は非交和になる場合もあれば共通部分を持つ両方の場合があります。

例えば、プリミティブ型同士の和集合は共通部分を持たないので非交和を表現します。共通部分を持たないということは型の共通部分の演算であるインターセクション型の結果が空集合に相当する `never` 型になるということです。

```ts
type U = string | number;

const u1: U = "st";
const u2: U = 42;

type N = string & number;
// => never (共通部分がないため)
```

逆にオブジェクト型同士などであれば共通部分を持つので、上図の左のような集合表現と一致します。

```ts
type U = { x: string; } | { y: number; };
type I = { x: string; } & { y: number; };

const u1: U = { x: "st" };
const u2: U = { y: 42 };

// 共通部分となる型の領域の値
const i1: I = { x: "st", y: 42 };
const u3: U = i1;
```

## リテラル型の冪集合

ここまで語ってきたように型を集合であると解釈すると、集合である型の集合 $\text{TYPES'}$ はボトムアップに見ると「**冪集合** (power set)」と呼ばれる構造とみなせそうです。冪集合は集合 $S$ の部分集合をすべてかき集めて作った集合 $P(S)$ であり、その要素は集合となります。冪集合のような集合が要素であるような集まりを一般には「**集合族** (family of sets)」とよびます。例えば集合 $A = \lbrace 1, 2, 3 \rbrace$ (濃度が $3$ の集合)の冪集合の要素は以下の８個です。

集合の濃度 | 要素
--|--
0 | $\phi$
1 | $\lbrace 1 \rbrace$, $\lbrace 2 \rbrace$, $\lbrace 3 \rbrace$
2 | $\lbrace 1, 2 \rbrace$, $\lbrace 2, 3 \rbrace$, $\lbrace 3, 4 \rbrace$
3 | $\lbrace 1, 2, 3 \rbrace$

冪集合は集合の包含関係を半順序関係とする半順序集合を作ることが知られています。つまり上記の冪集合は矢印の方向に対して $a \subseteq b$ (例えば $\lbrace 1 \rbrace \subseteq \lbrace 1, 2 \rbrace$)のような関係があり、これは半順序関係となります。

$$
\begin{aligned}
&\text{reflexive}: && a \subseteq a \\
&\text{antisymmetric}: && a \subseteq b \land b \subseteq a \Rightarrow a = b \\
&\text{transitive}: && a \subseteq b \land b \subseteq c \Rightarrow a \subseteq c
\end{aligned}
$$

従って、上記要素から構成される台集合 $P(A)$ と包含関係 $\subseteq$ による冪集合の半順序集合 $(P(A), \subseteq)$ は以下のようにハッセ図で表現できます。

```mermaid
graph BT
  B["Φ"]
  X["{1}"]
  Y["{2}"]
  Z["{3}"]
  XY["{1, 2}"]
  ZX["{3, 1}"]
  YZ["{2 ,3}"]
  T["{1, 2, 3}"]
  B --> X & Y & Z
  X --> XY
  Y --> XY
  Z --> ZX
  Y --> YZ
  X --> ZX
  Z --> YZ
  XY & YZ & ZX --> T
```

集合 $A$ の冪集合 $P(A)$ (P は Power の P)は $2^A$ とも表現されます。これは集合の要素を考える上で要素ごとにそれを集合に含めるか含めないの 2 通りがあるため、冪集合の要素の個数(部分集合の個数)は 2 を元の集合の個数でべき乗した値となるためです。つまり、上記の集合 $A$ なら要素の個数は 3 個なので、$2^3 = 8$ 個の要素を持つことになります。なおすべての要素を含めない場合(空集合)も要素として数えていることに注意してください。ベン図的な図式で包含関係を表現すると以下のようになります。

![数値リテラル型のべき集合](/images/ast/img_number-literal-venn.png)

実際、`number` 型が数値リテラル型から作成できる型を要素とした冪集合として考えることもできます。例えば３つのリテラル型の集合 $A = \lbrace 1, 2, 3 \rbrace$ についての冪集合を考えるとき、上記の冪集合のハッセ図以下のように表現できます。空集合が `never` 型に相当していることに注意してください。

```mermaid
graph BT
  B_123["never"]
  1["1"]
  2["2"]
  3["3"]
  12["1 | 2"]
  31["3 | 1"]
  23["2 | 3"]
  T_123["1 | 2 | 3"]
  B_123 --> 1 & 2 & 3
  1 --> 12
  2 --> 12
  3 --> 31
  2 --> 23
  1 --> 31
  3 --> 23
  12 & 23 & 31 --> T_123
```

この型の冪集合においては、包含関係が部分型関係と一致しています。実際に、部分型関係を調べれば包含関係と一致していることがわかります。

```ts
type T0 = never;
type T1 = 1;
type T2 = 2;
type T3 = 3;
type T12 = 1 | 2;
type T31 = 3 | 1;
type T23 = 2 | 3;
type T123 = 1 | 2 | 3;

type R_0_1 = Relation<T0, T1>;
// => Subtype (T0 <: T1 => T0 ⊆ T1)
type R_31_2 = Relation<T31,T2>;
// => Unrelated
type R_3_23 = Relation<T3, T23>;
// => Subtype (T3 <: T23 => T3 ⊆ T23)
type R_12_123 = Relation<T12, T123>;
// => Subtype (T12 <: T123 => T12 ⊆ T123)
```

ちなみに `1 | 2` 型と `3 | 1` 型の共通部分をインターセクション型で生成すると両方の型の共通部分である `1` 型となります。分かりやすく `1 | 2` 型を `A`、`3 | 1` 型を `B` とおいてインターセクション型を使って関係を表現すると以下のようになります。この場合 `A | B` 型などは `1 | 2 | 3` となり `A | B | C` と一致しています。

```mermaid
graph BT
  B["A & B & C"]
  X["A & B"]
  Y["C & A"]
  Z["B & C"]
  XY["A"]
  ZX["B"]
  YZ["C"]
  T["A | B | C"]
  B --> X & Y & Z
  X --> XY
  Y --> XY
  Z --> ZX
  Y --> YZ
  X --> ZX
  Z --> YZ
  XY & YZ & ZX --> T
```

ここからさらにもう一つ数値リテラル型を加えた冪集合はかなり複雑になります。このようにボトムアップに要素を追加してくことでほぼ無限に冪集合を構築できます。

```mermaid
graph BT
  B["never"]
  1["1"]
  2["2"]
  3["3"]
  4["4"]
  12["1 | 2"]
  14["1 | 4"]
  23["2 | 3"]
  24["2 | 4"]
  13["1 | 3"]
  34["3 | 4"]
  124["1 | 2 | 4"]
  123["1 | 2 | 3"]
  234["2 | 3 | 4"]
  134["1 | 3 | 4"]
  T["1 | 2 | 3 | 4"]
  B --> 1 & 2 & 3 & 4
  1 --> 12
  2 --> 12
  3 --> 13
  1 --> 13
  2 --> 23
  3 --> 23
  2 --> 24
  1 --> 14
  4 --> 14
  4 --> 24
  4 --> 34
  3 --> 34
  12 --> 123
  13 --> 123
  13 --> 134
  23 --> 123
  12 --> 124
  14 --> 124
  14 --> 134
  24 --> 124
  23 --> 234
  24 --> 234
  34 --> 134
  34 --> 234
  123 & 234 & 134 & 124 --> T
```

今度はもう少し簡単な例を考えます。濃度が $2$ の型 `boolean` は `true` と `false` という二つの要素から構成されました。`boolean` 型の包含関係について考えるには、$\text{boolean} = \lbrace \text{true}, \text{false} \rbrace$ という集合の冪集合を考え、その包含関係をハッセ図として図示します。

```mermaid
graph BT
Bot["never"]
T["true"]
F["false"]
Top["boolean \n (= true | false)"]
Bot --> T & F --> Top
```

このように要素数が二個の集合の冪集合のハッセ図は非常にシンプルになります。

## オブジェクト型の和と積

数値や真偽値といったプリミティブ型のリテラル型だけでなく、オブジェクト型についても考えてみましょう。まずは、`1, 2, 3` という数値リテラル型の冪集合ついてそのまま単一のプロパティ `x` を持つオブジェクトの型に写像した型 `{ x: T }` について考えます(型の写像については圏論での型構築子に箇所で出てきます)。つまり、プロパティとその型の組からなるオブジェクト型を要素とした以下のような集合族をつくります。

```mermaid
graph LR
subgraph A
  direction BT
  B_123["never"]
  1["1"]
  2["2"]
  3["3"]
  12["1 | 2"]
  31["3 | 1"]
  23["2 | 3"]
  T_123["1 | 2 | 3"]
  B_123 --> 1 & 2 & 3
  1 --> 12
  2 --> 12
  3 --> 31
  2 --> 23
  1 --> 31
  3 --> 23
  12 & 23 & 31 --> T_123
end

subgraph A'
  direction BT
  B["{ x: never }"]
  X["{ x: 1 }"]
  Y["{ x: 2 }"]
  Z["{ x: 3 }"]
  XY["{ x: 1|2 }"]
  ZX["{ x: 3|1 }"]
  YZ["{ x: 2|3 }"]
  T["{ x: 1|2|3 }"]
  B --> X & Y & Z
  X --> XY
  Y --> XY
  Z --> ZX
  Y --> YZ
  X --> ZX
  Z --> YZ
  XY & YZ & ZX --> T
end

A -->|"型の写像"| A'
```

この図式は実際の部分型関係を表現できています。ただし、ユニオンとインターセクションについて考えるときには注意が必要です。上記のようにプロパティの型単体について考えるときは数値リテラル型と同じ対応になりますが、オブジェクト型そのもののユニオンとインターセクションはまた別物となります。

```mermaid
graph BT
  B["never"]
  X["{x:1}"]
  Y["{x:2}"]
  Z["{x:3}"]
  XY["{x:1} | {x:2}"]
  ZX["{x:3} | {x:1}"]
  YZ["{x:2} | {x:3}"]
  T["{x: 1} | {x: 2} | {x: 3}"]
  B --> X & Y & Z
  X --> XY
  Y --> XY
  Z --> ZX
  Y --> YZ
  X --> ZX
  Z --> YZ
  XY & YZ & ZX --> T
```

なお、`{ x: 1 } & { x: 2 }` などの合成はその型の集合の共通部分が一切なく、`never` 型として解決されるので注意してください。これは同じプロパティ `x` について考えているので、積での合成では `x` が `1` 型であると同時に、`x` が `2` 型である必要があるということになりますが、そのような型は存在せず `never` に解決されます。

![共通部分がないオブジェクト型](/images/ast/img_no-intersection-object-types.png)

上の図式も積で表現しなおすことができますね。それぞれの和を新しい型 `A, B, C` で置くと以下のようになります。この場合には `A | B = B | C = C | A = A | B | C` です。また、`A & B & C = never` となります。

```mermaid
graph BT
  B["never (A & B & C)"]
  X["A & B"]
  Y["B & C"]
  Z["C & A"]
  XY["A"]
  ZX["B"]
  YZ["C"]
  T["A | B | C"]
  B --> X & Y & Z
  X --> XY
  Y --> XY
  Z --> ZX
  Y --> YZ
  X --> ZX
  Z --> YZ
  XY & YZ & ZX --> T
```

なお、以下ののようにオブジェクト型のユニオンやインターセクションは同一ではなく同値となる型がいくつかあるので注意してください。

```ts
type A = { x: 1 };
type B = { x: 2 };
type A_B = { x: 1 | 2 };
type AB = A | B;
type R = Relation<A_B, AB>;
// => Equivalent (同一ではないが同値となる)

type C = { x: string; };
type D = { y: number; };
type C_D = { x: string; y: number; };
type CD = C & D;
type R = Relation<C_D, CD>;
// => Equivalent (同一ではないが同値となる)
```

さて、ここまでは空集合に相当する `never` 型から要素を直接列挙して型を上方向(上位型の方向)に構築することで冪集合を考えていきました。`never` 型はボトム型であり、つまりこれは部分型関係の一番低い位置である底から文字通りボトムアップにみてきました。

逆に部分型関係の高い位置からトップダウンに型の集合を見てみましょう。例えば、オブジェクト型全体の集合内において最も広い集合、あるいは部分型関係の最も高い位置にある型は `{}` という型になります。空集合となる `never` 型からではなく、高い位置にある `{}` から型の包含関係について考えていきます。

なお、トップダウンに考えるといっても、とても大きな集合族についてその全体から次に小さな部分集合を考えるなどようなことは少々イメージが難しく、図で表現することも困難なので次に部分型関係で低い位置にある型として `{ x: string }` のようなある程度の抽象的な型を列挙して包含関係を考えてみましょう。

また、オブジェクト型の場合にはより情報量の多い(制約が多い)型が部分型になるため、数理リテラル型とは方向が配置が逆転していることに注意してください。制約が無い集合がより詳細な制約を持つ集合を包含すると考えれば実は数理リテラル型のときと同じ話になり、包含関係の図がまったく同じように表現されます。ただし、これは冪集合を表現しているわけではないことに注意してください。

```mermaid
graph BT
  T["{}"]
  X["{x: string; }"]
  Y["{y: number; }"]
  Z["{z: boolean; }"]
  XY["{x: string; y: number; }"]
  ZX["{z: boolean; x: string; }"]
  YZ["{y: number; z: boolean; }"]
  B["{x: string; y: number; z: boolean}"]
  B --> XY & YZ & ZX
  XY --> X
  XY --> Y
  ZX --> Z
  YZ --> Y
  ZX --> X
  YZ --> Z
  X & Y & Z --> T
```

というのも、ボトムアップに作成した場合とは異なり、この図は集合族の要素をすべて表現しきれていません。なお、底である `never` から上方向に無限に型を構築できたように、`{}` から下方向に無限に型を構築することが可能です。

上記の３つの型を `X, Y, Z` と置き換えて、更に共通部分の集合演算に相当するインターセクション型の型構築子(`&`)を使って表現するようにします。

```ts
type X = { x: string };
type Y = { y: number };
type Z = { z: boolean };
```

```mermaid
graph BT
  T["{}"]
  X
  Y
  Z
  XY["X & Y"]
  ZX["Z & X"]
  YZ["Y & Z"]
  B["X & Y & Z"]
  B --> XY & YZ & ZX
  XY --> X
  XY --> Y
  ZX --> Z
  YZ --> Y
  ZX --> X
  YZ --> Z
  X & Y & Z --> T
```

実はこの図ではインターセクションとユニオンが表現しきれていません。ということで、型 `X, Y, Z` についてのすべての和集合と共通部分を書ききれる分だけ書いてみましょう。とりあえず、`X, Y, Z` についてはそれぞれの組み合わせで和集合 `X | Y`、`Y | Z`、`Z | X` が考えられ、さらにそれらの和集合 `X | Y | Z` についても図に反映させます。

```mermaid
graph BT
  T["{}"]
  xyz["X | Y | Z"]
  xy["X | Y"]
  yz["Y | Z"]
  zx["Z | X"]
  X["X"]
  Y["Y"]
  Z["Z"]
  XY["X & Y"]
  ZX["Z & X"]
  YZ["Y & Z"]
  B["X & Y & Z"]
  B --> XY & YZ & ZX
  YZ --> Y
  XY --> Y
  XY --> X
  ZX --> X
  ZX --> Z
  YZ --> Z
  X --> xy
  Y --> xy
  Y --> yz
  Z --> yz
  X --> zx
  Z --> zx
  xy & yz & zx --> xyz --> T
```

これまで 2 つの型の和と 3 つ型の和が同一になったり、2 つの型の積と３つの型の積が同一になる場合がありましたが、このような場合にはすべての型のインターセクションとユニオンが相異なるため、今の所はきれいにすべての型の合成が表現されていることがわかります。ただし、この図ではまだまだ要素がたりません。

この図だけだとすべての要素を考えづらいですが、この集合の包含関係をベン図で表現すると以下のようになり、イメージしやすい関係となります。

![オブジェクト型のベン図表現](/images/ast/img_three-types-venn.png)

さて、現状の集合の包含関係の図において、以下のような和集合ともとの集合の和集合が考慮できていませんでした。`&` を `∩` に、`|` を `∪` に置き換えていますが、型表現で表すと `X | (Y & Z)` のような型が例として挙げられます。

![考慮しずらい和集合](/images/ast/img_difficult-union-venn.png)

このような「共通部分と元の型との和集合」までもを考慮してすべての和集合と共通部分の包含関係を考えます。このとき`{ }` は一旦邪魔なので排除しておき、集合らしく空集合までも含めて $X$ を中心とした和集合と共通部分の組み合わせの可能なパターンを考慮すると以下の９つのパターンが考えられます。これらのパターンについて $X, Y, Z$ の順番を入れ替えれば表現可能なすべてのパターンを考えることができます。

![ベン図の解体](/images/ast/img_venn-distruction.png)

これらの９つのパターンの包含関係を立体的に図示すると以下のようになります。上にある方がより広い集合で、下にある集合を包含するように図示していますが、この図は最小元である $\phi$ から最大元である $X \cup Y \cup Z$ までの二つの鎖の表現にもなっています。$X$ と $(X \cap Y) \cup (Y \cap Z) \cup (Z \cap X)$ は包含関係が無いため、比較不能な要素として鎖がここで分岐していることが分かります。

![ベン図の解体2](/images/ast/img_venn-distruction-2.png)

上記の図は $X$ を中心とした組み合わせなので、このようなパターンについて $Y, Z$ をそれぞれ中心とした構造まで考えると、以下のように同じような構造が三つあると分かります。

![ベン図の解体3](/images/ast/img_venn-distruction-3.png)

三つの構造はどれにおいても $(X \cap Y) \cup (Y \cap Z) \cup (Z \cap X)$ の要素で鎖が分岐しています。各構造は下から上に向けて包含関係 $\subseteq$ がありますが、異なる構造の要素同士でも包含関係がいくつかあることに注意する必要があります。上図の $X$ を主軸とした１つ目の構造での $X \cap Y$ は $Y$ を主軸とした２つ目の構造の $Y \cap (Z \cup X)$ にも包含され、１つ目の構造での $X \cup (Y \cap Z)$ は $Z$ を主軸とした３つ目の構造の $Z \cup X$ にも包含されます。このような要素が $Y, Z$ を主軸とした構造についても考えられます。

したがって、そのような関係についてそれぞれの構造がつながるようにうまく包含関係をすべて図示してやると以下のような立体構造になります

![ベン図の解体4](/images/ast/img_venn-distruction-4.png)

上記の図はすべての包含関係のパターンを表現しており、被覆関係(隣接の順序関係)のみを表示しているのでハッセ図となります。なお、灰色になっている要素は同じ階層にあることを占めています。また、点線にしている箇所は主軸となる要素の構造とは別の構造にある要素との包含関係を示しており、被覆関係となるように経由する要素がある場合には線を引いていないことに注意してください。

このような立体図をよりハッセ図らしく図示しなすと以下のようになります。上図の $X, Y, Z$ をそれぞれ主軸にした鎖については分かりやすいように包含関係に色付けしています。

```mermaid
graph BT
U["{ }"]
Top["X ∪ Y ∪ Z"]

x+y["X ∪ Y"]
y+z["Y ∪ Z"]
z+x["Z ∪ X"]

x+yz["X ∪ (Y ∩ Z)"]
y+zx["Y ∪ (Z ∩ X)"]
z+xy["Z ∪ (X ∩ Y)"]

x["X"]
y["Y"]
z["Z"]

xy+yz+zx["(X ∩ Y) ∪ (Y ∩ Z) ∪ (Z ∩ X)"]

xy+zx["(X ∩ Y) ∪ (Z ∩ X) \n = X ∩ (Y ∪ Z)"]
yz+xy["(X ∩ Y) ∪ (Y ∩ Z) \n = Y ∩ (Z ∪ X)"]
zx+yz["(Z ∩ X) ∪ (Y ∩ Z) \n = Z ∩ (X ∪ Y)"]

xy["X ∩ Y"]
zx["Z ∩ X"]
yz["Y ∩ Z"]

xyz["X ∩ Y ∩ Z"]
Bot["never"]

Bot --> xyz

xyz --> xy
xyz --> yz
xyz --> zx

xy ---> yz+xy
xy ---> xy+zx
zx ---> xy+zx
zx ---> zx+yz
yz ---> yz+xy
yz ---> zx+yz

xy+zx --> x
yz+xy --> y
zx+yz --> z
xy+zx -.-> xy+yz+zx
yz+xy -.-> xy+yz+zx
zx+yz -.-> xy+yz+zx

x --> x+yz
y --> y+zx
xy+yz+zx -.-> y+zx
xy+yz+zx -.-> z+xy
z --> z+xy
xy+yz+zx -.-> x+yz

x+yz --> x+y
x+yz --> z+x
y+zx --> x+y
y+zx --> y+z
z+xy --> z+x
z+xy --> y+z
x+y --> Top
z+x --> Top
y+z --> Top
Top --> U

style x stroke:red
style z stroke:green
style y stroke:blue

style U stroke:orange
style Top stroke:orange
style xyz stroke:orange
style Bot stroke:orange
style xy+yz+zx stroke:fuchsia

linkStyle 0 stroke: orange

linkStyle 1 stroke: red
linkStyle 2 stroke: blue
linkStyle 3 stroke: green

linkStyle 5 stroke: red
linkStyle 7 stroke: green
linkStyle 8 stroke: blue

linkStyle 10 stroke: red
linkStyle 11 stroke: blue
linkStyle 12 stroke: green

linkStyle 16 stroke: red
linkStyle 17 stroke: blue
linkStyle 20 stroke: green

linkStyle 22 stroke: red
linkStyle 25 stroke: blue
linkStyle 26 stroke: green

linkStyle 28 stroke: red
linkStyle 29 stroke: green
linkStyle 30 stroke: blue

linkStyle 31 stroke: orange
```

点線になっている箇所は $(X \cap Y) \cup (Y \cap Z) \cup (Z \cap X)$ を使って構造を繋いだ部分です。また、オレンジ色の箇所は元々構造を共有している要素です。

:::details 2024-03-03更新: 以前の間違った図について
以前までは以下の図を正しいハッセ図として扱っていましたが、$(X \cap Y) \cup (Y \cap Z) \cup (Z \cap X)$ などの要素が考慮しきれていなかっため、この図は間違いであり、訂正いたします🙇‍♂️

```mermaid
graph BT
Bot["X ∩ Y ∩ Z"]
xy["X ∩ Y"]
zx["Z ∩ X"]
yz["Y ∩ Z"]
xy+zx["(X ∩ Y) ∪ (Z ∩ X)"]
yz+xy["(X ∩ Y) ∪ (Y ∩ Z)"]
zx+yz["(Z ∩ X) ∪ (Y ∩ Z)"]
x["X"]
y["Y"]
z["Z"]
x+yz["X ∪ (Y ∩ Z)"]
y+zx["Y ∪ (Z ∩ X)"]
z+xy["Z ∪ (X ∩ Y)"]
x+y["X ∪ Y"]
y+z["Y ∪ Z"]
z+x["Z ∪ X"]
Top["X ∪ Y ∪ Z"]
U["{ }"]

Bot --> xy
Bot --> yz
Bot --> zx
xy ---> yz+xy
xy ---> xy+zx

zx ---> xy+zx
zx ---> zx+yz
yz ---> yz+xy
yz ---> zx+yz
xy+zx --> x
yz+xy --> y
zx+yz --> z

x --> x+yz
y --> y+zx
z --> z+xy

xy --> z+xy
yz --> x+yz
zx --> y+zx

x+yz --> x+y
x+yz --> z+x
y+zx --> x+y
y+zx --> y+z
z+xy --> z+x
z+xy --> y+z
x+y --> Top
z+x --> Top
y+z --> Top
Top --> U

style xy stroke:red
style zx stroke:green
style yz stroke:orange

linkStyle 3 stroke: red
linkStyle 4 stroke: red
linkStyle 5 stroke: green
linkStyle 6 stroke: green
linkStyle 7 stroke: orange
linkStyle 8 stroke: orange

linkStyle 15 stroke: red
linkStyle 16 stroke: orange
linkStyle 17 stroke: green
```
:::

なお、３つのオブジェクト型ではなく、よりシンプルな２つのオブジェクト型についての関係は以下のようになります。

```mermaid
graph BT
subgraph A
  direction BT
  T["{}"]
  xy["{ x: string; } | { y: number; }"]
  X["{ x: string; }"]
  Y["{ y: number; }"]
  XY["{ x: string; } & { y: number; }"]
  XY --> Y
  XY --> X
  X --> xy
  Y --> xy
  xy --> T
end

subgraph B
  direction BT
  T2["{}"]
  xy2["X | Y"]
  X2["X"]
  Y2["Y"]
  XY2["X & Y"]
  XY2 --> Y2
  XY2 --> X2
  X2 --> xy2
  Y2 --> xy2
  xy2 --> T2
end
```

## 濃度差による型挿入

注意点として現在の図式として表現されているオブジェクト型の集合において、要素同士で線が引かれている部分型関係 $<:$ にある任意の二つの型の間には無限に型を挿入できることに注意してください。

例えば、現在考えているオブジェクト型の全体集合として `{}` を使っていますが、`{}` とその配下の型の間の関係は「すべての型の集合」において実際には被覆関係 $\lessdot$ になっていないことに注意してください。例えば３つの型のベン図において新しいオブジェクト型 `V` を導入すれば新しい型の合成(和集合)となる `X | Z | V` や `X | Y | Z | V` などがつくれるため、`{}` と `X | Y | Z` の間には無限に型を挿入できます。

![ベン図に新しい型を追加](/images/ast/img_additional-set-to-venn.png)

なお、図のような `V` の位置となり交差を持たない型は例えば `string` などがあります。プリミティブ型ではなく、通常のオブジェクト型なら基本的に交差が生まれるので４つの集合のベン図を考える必要がありますが、まったく交差がないオブジェクト型も存在し、`{ x: number; y: boolean; z: string; }` などがその一つです。

```ts
type A = { x: string };
type B = { y: number };
type C = { z: boolean };
type W = { x: number; y: boolean; z: string; };

type R1 = Relation<A, W>;
// => Unrelated
type R2 = Relation<B, W>;
// => Unrelated
type R3 = Relation<C, W>;
// => Unrelated
```

空集合についても同様であり、$X \cap Y \cap Z$ に相当する `X & Y & Z` の型と `never` 型は被覆関係にあらず、無限に型を挿入できることに注意してください。

```ts
type X = { x: string; };
type Y = { y: number; };
type Z = { z: boolean; };

type XY = X & Y;
type YZ = Y & Z;
// { y: number; z: boolean; }
type X_YZ = X | YZ;

type yz = { y: 1, z: true };
```

したがって、今後オブジェクト型の包含関係を考えるにあたっては、オブジェクト型の全体集合 `{}` は省略して、考えている型のすべての和集合を一番大きい要素として以下の図のように考えることにしましょう。

```mermaid
graph BT
  xy["X | Y"]
  X["X"]
  Y["X"]
  XY["X & Y"]
  XY --> Y
  XY --> X
  X --> xy
  Y --> xy
```

なお、無限に型が挿入できるのは Top と Bottom だけの話ではなく、`X` と `X | (Y & Z)` の型などの間にも無限に型を挿入することが可能です。

これとは逆に数理リテラル型の冪集合のハッセ図は本当にハッセ図となっており、被覆関係を表示していました。それは例えば `1` と `1 | 2` 型の間には他のいかなる型も存在しないからです。

```mermaid
graph BT
n[never]
12["1 | 2"]
n -->|被覆| 1 -->|被覆| 12
n -->|被覆| 2 -->|被覆| 12
```

これはリテラル型が濃度が１であるユニット型で、二つのユニット型を合成した型の濃度は２で、それぞれの型に存在する要素が `1` と `1, 2` というように一つしか違わないからです。ユニット型との比較が `1 | 2` という濃度２ユニオン型との比較ではなく `1 | 2 | 3` という濃度３のユニオン型との比較であれば、濃度の差が $3 - 1 = 2$ となり、`1 | 2` という型が挿入できます。

```mermaid
graph LR
subgraph A
direction BT
n1[never]
1_1[1]
123_1["1 | 2 | 3"]
n1 -->|"濃度の差が１\n(型挿入できない)"| 1_1 -->|"濃度の差が２\n(型挿入が可能)"| 123_1
end

subgraph A'
direction BT
n2[never]
1_2[1]
12["1 | 2"]
123_2["1 | 2 | 3"]
n2 -->|"濃度の差が１\n(型挿入できない)"| 1_2 -->|"濃度の差が１\n(型挿入できない)"| 12 -->|"濃度の差が１\n(型挿入できない)"| 123_2
end

A --> A'
linkStyle 1 stroke: red
```

数値リテラル型の集合的な表現を再度見てみるとそれが分かります。`1 | 2 | 3` の型に包含され、`1` を包含する型は `1 | 2` のみです。

![数値リテラル型のベン図](/images/ast/img_number-literal-venn.png)

その一方で、プリミティブ型とそのリテラル型、あるいはリテラル型のユニオン型との間には無限に型を挿入できます。例えば、`1 | 2` と具体的な数理リテラルをすべて集めた集合型である `number` 型の間にはほぼ無限に型が挿入できるようになっています。

```mermaid
graph BT
n[never]
12["1 | 2"]
n -->|被覆| 1 -->|被覆| 12
n -->|被覆| 2 -->|被覆| 12
12 -->|型挿入が可能| number

linkStyle 4 stroke: red
```

縦に伸びて少し見づらいので横に回転させますが、例えば `1 | 2 | 3` などのユニット型をさらに追加した型を以下のように挿入していくことが可能です。

```mermaid
graph LR
n[never]
12["1 | 2"]
n -->|被覆| 1 -->|被覆| 12
n -->|被覆| 2 -->|被覆| 12
12 -->|被覆| 123["1 | 2 | 3"] -->|被覆| 1234["1 | 2 | 3 | 4"] -->|型挿入が可能| number

linkStyle 6 stroke: red
```

より分かりやすい例としては、二つのユニット型からなる濃度2の集合型 `boolean` 型であり、以下のように型を挿入できるわけです。

```mermaid
graph LR


subgraph A
  direction BT
  n1[never]
  b1[boolean]

  n1 -->|型が挿入可能| b1
  linkStyle 0 stroke: red
end

subgraph A'
  direction BT
  n2[never]
  t[true]
  f[false]
  b2["boolean \n (= true | false)"]

  n2 --> t & f --> b2
  linkStyle 0 stroke: red
end

A --> A'
```

文字列リテラル型も同様に、具体的な値であるリテラル型のユニオン型とすべての文字列の集合である `string` 型の間には無限に型を挿入できます。

```mermaid
graph BT
n[never]
s[string]
s1["'s'"]
s2["'st'"]
s3["'str'"]
s12["'s' | 'st'"]
s23["'st' | 'str'"]
s31["'str' | 's'"]
n -->|被覆| s1 & s2 & s3
s1 & s2 -->|被覆| s12 -->|型挿入が可能| s
s1 & s3 -->|被覆| s31 -->|型挿入が可能| s
s2 & s3 -->|被覆| s23 -->|型挿入が可能| s

linkStyle 5 stroke: red
linkStyle 8 stroke: red
linkStyle 11 stroke: red
```

プリミティブ型同士ならどうでしょうか。空集合である `never` 型とすべてのリテラルを集めて作った集合 `number` と `boolean` の間にはリテラル型の冪集合があるので型を挿入することができます。

```mermaid
graph BT
bot[never]
n[number]
b[boolean]
nb["number | boolean"]
bot -->|型挿入が可能| n
bot -->|型挿入が可能| b
n -->|挿入可能か?| nb
b -->|挿入可能か?| nb

linkStyle 0 stroke: red
linkStyle 1 stroke: red
```

プリミティブ型とその型と別のプリミティブ型の和集合たるユニオン型の間には実は型が挿入可能です。実際に `never` とプリミティブ型の間、プリミティブ型とプリミティブ型のユニオン型の間に `true`, `false`, `1`, `2` という４つのリテラル型とそのユニオンを挿入してみると以下のようになります。

```mermaid
graph BT
bot[never]
n[number]
b[boolean]
nb["number | boolean"]

t[true]
f[false]
1
2
12["1 | 2"]
1b["1 | boolean"]
2b["2 | boolean"]

bot --> t --> b
bot --> f --> b

bot --> 1 --> 12
bot --> 2 --> 12
12 -->|"型挿入が可能\n(濃度差大)"| n

nt["number | true"]
nf["number | false"]

n --> nt
t --> nt
nt --> nb
n --> nf
f --> nf
nf --> nb

b --> 1b
1 --> 1b
1b -->|"型挿入が可能\n(濃度差大)"| nb
b --> 2b
2 --> 2b
2b -->|"型挿入が可能\n(濃度差大)"| nb

linkStyle 17 stroke: red
linkStyle 20 stroke: red
linkStyle 8 stroke: red
```

赤色の線以外の線は被覆関係となる部分型関係を示しています。実はそれらの被覆関係にある型は濃度の差が１となっています。例えば、空集合たる `never` 型の濃度は $|\text{never}| = 0$ ですが、`true` などのリテラル型はユニット型であったので濃度は $|\text{true}| = 1$ となります。このときの濃度差は１であり、集合間の差分となる要素はそのリテラル型そのものである `true` しかありません。したがって `never` と `true` の型の間にはどのような型も挿入できません。

同様に `number` 型に対して濃度１であるユニット型 `false` との和集合は元の `number` と濃度差１の集合 `number | false` を作ります。したがって `number | false` と `number` 型の間には何も型を挿入できません。

逆に、濃度２である `1 | 2` と濃度が限りなく大きい `number` の濃度差は非常に大きいため、その分だけほぼ無限(実際には有限)に型を挿入できるわけです。

もう一度オブジェクト型について考えてみると、以下のようなプリミティブ型を内部で利用するシンプルなオブジェクト型の和と積において型挿入ができたのは実は型同士の濃度差が大きいためでした。

```mermaid
graph BT
T["{}"]
xy["{ x: string } | { y: number }"]
X["{ x: string }"]
Y["{ y: number }"]
XY["{ x: string } & { y: number }"]
n[never]
n -->|"型挿入可能\n (濃度差大)"| XY
XY -->|"型挿入可能\n (濃度差大)"| Y
XY -->|"型挿入可能\n (濃度差大)"| X
X -->|"型挿入可能\n (濃度差大)"| xy
Y -->|"型挿入可能\n (濃度差大)"| xy
xy -->|"型挿入可能\n (濃度差大)"| T

linkStyle 0 stroke: red
linkStyle 1 stroke: red
linkStyle 2 stroke: red
linkStyle 3 stroke: red
linkStyle 4 stroke: red
linkStyle 5 stroke: red
```

上図では型挿入ができることが自明な型同士の関係の方が多いですが、`{ x: string }` と `{ x: string } & { y: number }` などの関係においてはすこしイメージしずらいかもしれません。この型の間の型挿入は以下のベン図で考えれば結構簡単に挿入可能な型が見つかります。

![ベン図の解体](/images/ast/img_venn-distruction.png)

`Z = { z: boolean }` などの別のオブジェクト型を用意して上記パターンで $X \cap (Y \cup Z)$ となる合成 `X & (Y | Z)` 型を考えれば、上記パターンの $X \cap Y$ となる型 `X & Y` が包含されることが直感的に理解できます。つまり、`X & Y` と `X` の間には `X & (Y | Z)` が挿入可能です。`Z` 以外にも他のオブジェクト型 `V, W` などを新しく作れば無限に挿入可能です。

```ts
type X = { x: string };
type Y = { y: number };
type Z = { z: boolean };

type R1 = Relation<X & Y, X & (Y | Z)>;
// => Subtype
type R2 = Relation<X & (Y | Z), X>;
// => Subtype
```

このように濃度の差が１よりも大きければその分だけ追加の型が挿入することができることがわかりましたが、逆に言えば、被覆関係となるのは濃度の差が１となる型同士ということになり、ユニット型との合成以外では元の型に対して真に被覆関係となる型は作ることができないという話になります。

:::message alert
今後もオブジェクト型やプリミティブ型についての順序関係を表現する図式をハッセ図的に書いていきますが(面倒なのでこのような図式もハッセ図と呼びようにしています)、型の集合全体においては真に被覆関係になっていない場合があるので注意してください。
:::
