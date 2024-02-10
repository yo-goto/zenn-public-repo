---
title: "部分型関係の概念"
cssclass: zenn
date: 2024-02-10
modified: 2024-02-10
AutoNoteMover: disable
tags: type/zenn/book, TypeTheory/Subtyping, TypeScript/type, math/algebra
aliases: TAM本『部分型関係の概念』
---

## 部分型関係

代数的構造や代数法則のモデルが頭にあることで型の互換性についての推論が可能になるといいましたが、そもそも型の互換性とはなんでしょうか?

TypeScript は「**構造的部分型** (structural subtyping)」の型システムを持ちますが、構造的であるとは、型の形状 (shape) が同じであれば部分型として扱えるということです。

そもそも部分型関係とは、ある型 $S$ の項が別の型 $T$ の項が期待されている文脈で安全に使用可能であるということを定める関係性のことで、部分型互換性はその部分型関係に基づく**型同士の互換性の概念**です。部分型関係は $S <: T$ と表記して、このときに型 $S$ は型 $T$ の部分型 (subtype) であると言います。

以下のようなオブジェクト型 `A` と `B` について考えてみると、両者の型は `fst` というプロパティの型が同じであり、その箇所についての型の形状が同じです。結論から言えば型 `B` は型 `A` の部分型であり、$A :> B$ と表記できます。

```ts
type A = { fst: number; };

type B = { fst: number; snd: string; };

// A型が期待される文脈でB型の項を割り当てることができる
const b1: B = { fst: 42, snd: "st", };
const a1: A = b;

// その逆はできない(安全に置換できない)
const a2: A = { fst: 42, };
const b2: B = a2;
//    ^^ Error: プロパティ 'snd' は型 'A' にありませんが、型 'B' では必須です。
```

プリミティブ型とリテラル型、そのユニオン型についての互換性について考えてみると以下の３つの型は $Collection :> Some :> Unit$ という部分型関係になります。

```ts
type Collection = number;
type Some = 1 | 2 | 3;
type Unit = 1;

const u: Unit = 1;
const s: Some = u;
const c: Collection = s;
```

`number` や `string` のようなプリミティブ型はその型の具体的な要素となる値を集めた集合的な型 (collective type) といえます。そして `42` や `"st"` のような具体的な値からなるリテラル型は具体的な型 (concrete type) と呼ばれます。部分型関係は集合の包含関係 (inclusion) としてみなすことでうまくイメージがしやすくなり、型 $A$ という集合に型 $B$ が包含されるなら、型 $B$ は型 $A$ の部分型であり、$A \subseteq B$ なら $A :> B$ であると考えることができます。

全体集合と空集合の概念に対応する `unknown` 型 (Top 型) と `never` 型 (Bottom 型) を加えれば更に集合的なイメージに合致します。

```ts
type Universe = unknown;
type Collection = number;
type Some = 1 | 2 | 3;
type Unit = 1;
type Empty = never;

declare const e: Empty;
const u: Unit = e;
const s: Some = u;
const c: Collection = s;
const u: Universe = g;
```

この場合には部分型関係は以下のようになっています。

$$
Universe :> Collection :> Some :> Unit :> Empty
$$

このように型を集合論に扱ったり、考えたりできることは次の記事で書きました。

https://zenn.dev/estra/articles/typescript-type-set-hierarchy

ただし、型が厳密に集合であるかどうかについては話は別です。型理論は素朴集合論についての矛盾として有名な「[ラッセルのパラドックス](https://ja.wikipedia.org/wiki/%E3%83%A9%E3%83%83%E3%82%BB%E3%83%AB%E3%81%AE%E3%83%91%E3%83%A9%E3%83%89%E3%83%83%E3%82%AF%E3%82%B9)」から始まっており、型理論と集合論は明らかに密接な関係がありますが、型と集合がどのように違のかを説明するのは結構難しいです。例えば再帰型などの循環的な定義が含またりしたときに「集合全体の集合」は考えられないという上記のパラドックスに行き着くために集合と型を同一視できなくなる場合があります。
※ このような場合に圏論が役立ち、始代数などの概念で再帰を説明できるようになります。

型と集合の違いについては John L. Bell の 「TYPES, SETS AND CATEGORIES」などの文献を参照してください。

## 割当互換性

さて、TypeScript では上記の部分型互換性を拡張した概念である割当互換性 (assignment compatibility) というものがあることに注意してください。

これは大雑把に言えば TypeScript の型システムが漸新的 (gradual) であることから必要となっている `any` 型という静的型付けと動的型付けの世界の境界となる型ついての法則を追加して拡張した部分型互換性です。

細かい法則は公式ドキュメントの次のページのテーブルに記載されていますが、簡単に理解するなら、`any` 型から別のあらゆる型への割当、別のあらゆる型から `any` 型を部分型互換性に追加したものです。※ 厳密には更に `enum` 型から・への割当の関係も追加したものです。

`any` 型は `unknown` 型が導入されるまで TypeScript 世界の唯一の Top 型として機能していましたが、現在はこの二つの型が Top 型として機能しています。

割当可能性 (assignability) は様々なところで機能しており、例えば `extends` を使った条件型の判断や部分型関係の基本的な判断は実際には割当互換性が利用されていることに注意してください。割当可能性については公式ドキュメントにはあまり記載はないですが、古のリファレンスには細かい記載があります。

https://github.com/microsoft/TypeScript-New-Handbook/blob/master/reference/Assignability.md

さて、実際に型同士が部分型関係 (≒割当可能関係) にあるかどうかを判別する最も簡単な「型構築子 (type constructor)」は以下のようなジェネリクス型で定義できます。

```ts
type Assignable<Sub, Super> =
  [Sub] extends [Super] ? true : false;

type E1 = Assignable<1, number>;
//   ^^: true
type E2 = Assignable<string, "st">;
//   ^^: false
type E3 = Assignable<number, string>;
//   ^^: false
```

:::message
「**型構築子** (type constructor)」とは、型から新しい型を構築するための機能のことで、`Array<T>` や `Promise<T>`、`Partial<T>` などのことを指します。関数型の `->` やユニオン型の `|`、インターセクション型の `&` などの型構築子に相当します。
:::

型構築子の名前は `IsSubtypeOf` などでもよいですが、条件型での判定には割当互換性が利用されているので `Assignable` (あるいは `IsAssignableTo` など)としています。

なお、`[Sub] extends [Super]` のようにタプルに一度変換しているのは、分配条件型 ([Distributive conditional type](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types)) という条件型をジェネリクスで使った場合に発生する機能を利用しないようにしているためです。

変性 (variance) の概念については圏論のところで後述しますが、タプル型の型構築子の変性は共変 (covariant) なので、タプル型に変換したとしても元々の型同士の部分型関係は保存されており、元々の型同士の部分型関係について判断することができる、というロジックで上記の判別式が任意の型について利用できます。

## 順序関係

さて、ここからはまた数学の話になります。部分型関係 $:>$ は、型を集めて作った集合における「二項関係 (binary relation)」の一種であり、ある種の順序関係 (order relation) を構築します。順序関係は後述する順序理論 (order theory) のコアとなる重要な概念であり、圏論においても活用されます。

順序関係とは簡単に言えば、数値の大小関係のような文字通りの対象間の順序の関係のことで、一般的には集合上の二項演算として定義され、いくつかの種類が存在します。そして、そのような要素同士で順序関係を持つような集合を順序集合 (ordered set) などと呼びます。

二項関係においても、代数法則のような法則 (law) があり、例えば以下の二つの法則を満たす関係 $\prec$ を持つ集合 $S$ とその関係の組 $(S,\prec)$ は「**前順序集合** (pre-ordered set)」と呼ばれます。

- 反射律 (reflexive) : $a \prec a$
- 推移律 (transitive) : $a \prec b \land b \prec c \Rightarrow a \prec c$

:::message
それぞれの記号について、$\land$ は「かつ」、$\lor$ は「または」、$\Rightarrow$ は「ならば」の意味で使っています。$\prec$ は prec (precedes) 記号といい、一般的な順序を表現するときなどに使えます。また、任意を表す記号である $\forall$ については冗長となるので省略しています。
:::

記号 $\prec$ を整数の大小関係 $\le$ で置き換えてみるとイメージがしやすいです。まず、反射律は英語 reflexive から想像される再帰的 (自己言及的) な性質であり、$1 \le 1$ が成り立ちます。推移律は関係が要素間で伝播するような性質で、数値の大小関係なら $1 \le 2 \land 2 \le 4 \Rightarrow 1 \le 4$ というような感じです。

上記二つの法則を満たす「前順序関係 (pre-order relation)」においては、まったくその関係が成り立たないような要素が集合内にあってもよいことに注意してください。

前順序関係の二つの法則に加えて以下の法則を満たす関係は「**半順序関係** (partial order relation」とよばれ、そのような二項関係を備えている集合を「**半順序集合** (partial ordered set)」と呼びます。

- 反対称律 (asymmetric) : $a \prec b \land b \prec a \Rightarrow a = b$

これは双方向に関係が成り立つなら、両者に等価関係 (equality) が成り立つ、つまり同一の数学的対象であることを示しています。

さらに、半順序関係に加えて以下の法則を満たす関係は「**全順序関係** (total order relation)」と呼ばれ、そのような二項関係を備えている集合を「**全順序集合** (total ordered set)」と呼びます。

- 完全律 (total) : $a \prec b \lor b \prec a$

これは集合内の任意の要素 $a, b$ について必ず関係があるということを言っています。逆に、この法則を満たしていない前順序集合や半順序集合には関係を持たない要素同士があるということになります。

前順序集合のところで、整数の大小関係について考えましたが、実は整数全体の集合 $\mathbb{Z}$ は全順序集合となります。もちろん、$\mathbb{Z}$ は前順序でも半順序でもあります。

## 同値関係

さて、基本的な順序関係について説明したので、次は同値関係について説明しておきましょう。

「**同値関係** (equivalence relation)」とは、二項関係の一種であり、以下の３つの法則を必要十分で満たす二項関係 $\sim$ です。

- 反射律 (asymmetric) : $a \sim a$
- 対称律 (symmetric) : $a \sim b \Rightarrow b \sim a$
- 推移律 (transitive) : $a \sim b \land b \sim c \Rightarrow a \sim c$

上記３つをまとめて**同値律**とよび、同値関係そのものを $a \equiv b$ などと表記します。

同値関係はオペランド同士が同一の対象であることや同じ値を持つことを示す等価関係 (equality) $=$ とは異なる概念であることに注意してください。

同値関係について説明したのは、それが満たす対称律と半順序関係が満たした反対称律についての違いに注意する必要があるからです。関係の記号を同じ $\prec$ にして比べてみましょう。

- 対称律 (symmetric) : $a \prec b \Rightarrow b \prec a$
- 反対称律 (asymmetric) : $a \prec b \land b \prec a \Rightarrow a = b$

対称律では、要素 $a, b$ についていずれかの方向で関係が定まるなら逆方向についても関係が定まるという関係ですが、反対称律では、両方の方向で関係が成り立つ場合に限り二つの対称は等価関係にある (つまり同一の対象である) と言えるということになります。

等価関係 (`=`) と同値関係 (`≡`) は関連していますが、異なる概念なので注意してください。

## 部分型関係の順序

さて、ここまで順序関係について説明してきたのは、実は部分型関係 $<:$ は実は順序関係と同じ法則をいくつか満たしているからです。部分型関係は以下の法則を満たします (むしろこの法則を満たすように部分型関係は定義されます)。

- 反射律 (reflexive) : $A <: A$
- 推移律 (transitive) : $A <: B \land B <: C \Rightarrow A <: C$

型の集合内の任意の型 $A, B$ についてこれは成り立ちます。実際に確認してみます。

```ts
type Assignable<Sub, Super> =
  [Sub] extends [Super] ? true : false;

type Unit = 1;
type Some = 1 | 2 | 3;
type Collection = number;

// 反射律: A <: A
type R1 = Assignable<Unit, Unit>;
// => true
type R2 = Assignable<Some, Some>;
// => true

// 推移律: A <: B かつ B <: C なら A <: C
type T1 = Assignable<Unit, Some>;
// => true
type T2 = Assignable<Some, Collection>;
// => true
type T3 = Assignable<Unit, Collection>;
// => true
```

ということで、部分型関係は前順序関係の法則を満たしていると言えます。

それでは以下の法則についてはどうでしょうか?

- 反対称律 (asymmetric) : $A <: B \land B <: A \Rightarrow A = B$

実は部分型関係が半順序 (partial order) であると嬉しいことがあるのでこれは成り立っていてほしいところなのですが、本当に成り立つかは正直微妙なところですね。

等価関係 (equality) が成り立つかどうかですが、等価関係とは二つのオペランドが同一の対象であることでした。ここでは $A$ と $B$ の型が同一 (identical) の型であることを言うと考えられますが、そもそも「型の同一性 (type identity)」とは何かを考える必要がありそうです。

## 型の同一性

型の同一性 (identity) の概念は公式ドキュメントには記載されていませんが、古の仕様書には記載されている概念です。

https://github.com/microsoft/TypeScript/blob/3c99d50da5a579d9fa92d02664b1b66d4ff55944/doc/spec-ARCHIVED.md#L2225-L2261

仕様書においては、そもそも TypeScript には型の関係性 (type relationship) として同一性、部分型、上位型、割当互換性という４つの関係があることが語られています。

> Types in TypeScript have **identity**, **subtype**, **supertype**, and **assignment** compatibility relationships as defined in the following sections.

関係性の定義は実装だと `checker.ts` の以下のように定義されているところですね。仕様書の頃よりもいくつか関係が増えていますね。

https://github.com/microsoft/TypeScript/blob/7d9399e353c1b770ab1b5c859c98e014cd3fda03/src/compiler/checker.ts#L2222-L2227

話を戻すと、この古の仕様書の時点においては、二つの型は以下の条件を満たしたときに同一 (identical) であると言えます。

- 二つの型が両方とも `any` 型
- 二つの型が両方とも同じプリミティブ型
- 二つの型が同じ型パラメータ
- 二つの型が同一の構成要素から成るユニオン型
- 二つの型が同一の構成要素からなるインターセクション型
- 二つの型が同一のメンバーセットを持つオブジェクト型

同一性の判定は以下の実装の箇所でしょう。
https://github.com/microsoft/TypeScript/blob/7d9399e353c1b770ab1b5c859c98e014cd3fda03/src/compiler/checker.ts#L19663-L19665

例えば、ユニオン型の `A | B` と `B | A` は同一の構成要素 `A, B` からなるユニオン型なので両者の型は同一 (identical) であると言えます。

型の同一性の概念が (部分的に) 定義されましたが、`checker.ts` などの TypeScript のソースコードを見ることなく型の等しさ (同じさ) についてプログラマーが語りたいときに本当に両者の型が同一の対象、つまり $A = B$ であると言い切れるでしょうか?

一方、ユニオン型やインターセクション型などの記述では、同値関係 (equivalence) の概念かどうはわかりませんが、`A | B` と `B | A` は同等 (equivalent to) であると記載されていますが、構成要素の型の順序も情報として保持されており、ユニオン型の呼び出しや構築シグネチャにおいて問題となる場合があるとも語られています。

> A union type encompasses an ordered set of constituent types. While it is generally true that _A_ | _B_ is equivalent to _B_ | _A_, the order of the constituent types may matter when determining the call and construct signatures of the union type.
> ([spec](https://github.com/microsoft/TypeScript/blob/3c99d50da5a579d9fa92d02664b1b66d4ff55944/doc/spec-ARCHIVED.md#34-union-types) より引用)

つまり、二つの型が対象として完全に同一であるかは明らかではないですし、ソースコードに立ち入らず、既存のシンタックスをつかって同一性を証明することは難しいということです。

現在自分たちの手にあるのは条件型を駆使した割当可能性を検証できる以下のような型構築子のみです。

```ts
type Assignable<Sub, Super> = [Sub] extends [Super] ? true : false;
```

さて、問題にしていた反対称律 ($A <: B \land B <: A \Rightarrow A = B$) についてですが、`Assignable` 型構築子では**二つの型が相互に部分型であるとしか言えません**。つまり、$A <: B \land A :> B$ のみが以下のように言えます。

```ts
type A = { fst: number };
type B = { fst: number };

type E1 = Assignable<A, B>;
// => true
type E2 = Assignable<B, A>;
// => true
```

$A <: B \land A :> B$ が言えたとしても $A = B$ は言えません。

一般に部分型関係は前順序 (pre-order) になると決まっています。例えば型システムにもよってレコード型 (簡易的なオブジェクトといえる) のプロパティ (あるいはメンバー) の順序 (order) などが変わることで同一の型ではなくなる場合があります。※ TypeScript ではプロパティ順序に関わらず同一とみなします。

その他にも TypeScript でいえば、`Object` と `object` と `{}` という３つのオブジェクトを表現する型は相互に部分型 (相互に割当可能) ですが、完全に同一の型ではありません。それぞれの型は振る舞いや役割が異なるからです。

```ts
type E1 = Assignable<object, Object>;
// => true
type E2 = Assignable<Object, {}>;
// => true
type E3 = Assignable<{}, object>;
// => true
type E4 = Assignable<object, {}>;
// => true
type E5 = Assignable<Object, object>;
// => true
type E6 = Assignable<{}, Object>;
// => true
```

$A <: B \land A :> B$ となっても常に $A = B$ が成り立たないなら、反対称律は成り立たないということになります。

ここまで型の同一性を調べるのは困難であるということを述べてきましたが、実際には型の同一性を調べる方法として有名なハックが存在しています。公式ドキュメントにも載っておらず、いくつかのケースでは例外的にうまくいかない場合もあるとのことですが、大抵は以下の型構築子 `Identical` で検証することが可能です。

```ts
type Identical<Fst, Snd> =
  (<T>() => T extends Fst ? 1 : 2) extends
  (<T>() => T extends Snd ? 1 : 2)
    ? true
    : false;

// エイリアスとして Equals とも呼ぶようにする
type Equals<Fst, Snd> = Identical<Fst, Snd>;
```

実際にいくつかの型が同一であることを確認してみます。

```ts
type I0 = Identical<string, string>;
// => true

// 相互に割当可能な Object, object, {} は同一ではない
type I1 = Identical<Object, object>;
// => false
type I2 = Identical<object, {}>;
// => false

// ユニオン上のインターセクションの分配法則が同一性レベルで成り立つ
type A = { fst: number };
type B = { snd: string };
type C = { trd: boolean };
type I3 = Identical<A & (B | C), (A & B) | (A & C)>;
// => true

// オブジェクトのプロパティ順序によらず同一
type O1 = { fst: string; snd: number; };
type O2 = { snd: number; fst: string; };
type I4 = Identical<O1, O2>;
// => true
```

`Identical` の型構築子が型の同一性チェックとして機能するのは、二つの条件型の条件型 (比較) では `Fst` と `Snd` の二つの型パラメータが同一 (identical) であるときに限ってこの型が `true` ブランチの型を生成するからです。そして、条件型は通常は比較することができませんが、型引数として与えられない型パラメータ `T` によって条件型の解決が遅延することで条件型同士の比較が可能となっているというハックです。

実際に、`Object, object, {}` の同一性を検証するといずれの組み合わせも `false` になり、同一の型ではないことが分かります。

:::message
元ネタは、TypeScript リポジトリの [このissueのコメント](https://github.com/microsoft/TypeScript/issues/27024#issuecomment-421529650) で、型の名前は `Equals` として紹介されており、Type Challenge の [Utils](https://github.com/type-challenges/type-challenges/blob/main/utils/index.d.ts#L7-L9) としても実装されています。

この型についての解説は[リポジトリにあったこの解答](https://github.com/type-challenges/type-challenges/discussions/9100#discussioncomment-6896958)が詳しいので参照してください。
:::

## 同値類と商集合

型の同一性について紹介しましたが、型の集合において反対称律が成り立っていると色々構造として綺麗になるので、ここで無理やり反対称律を成り立たせるために同値類と商集合の概念を導入して、反対称律の等価関係を同値関係に置き換えます。

まずは、我々が二つの型における「同じさ」については言えるのは型同士の同一の対象であるという同一性と、相互に部分型関係であるという同値関係の二つです。

ここで思い出してほしいのは、同値関係が満たす法則の一つに対称律というものがありました。対称律は記号を $\prec$ から $\equiv$ に置き換えて書くと以下のようなものです。

$$
A \equiv B \Leftrightarrow B \equiv A
$$

ここで、二つの型 $A$ と $B$ について相互に部分型関係が成り立つ場合に限って二つの型が同値関係にあると定義して、$A \equiv B$ と表記するようにしましょう。

つまり、型同士に相互に部分型関係がなりたつことを同値関係と呼ぶなら以下のように同値関係が満たすべき同値律も成り立ちます。

- 反射律 (asymmetric) : $A \equiv A$
- 対称律 (symmetric) : $A \equiv B \Rightarrow B \equiv A$
- 推移律 (transitive) : $A \equiv B \land B \equiv C \Rightarrow A \equiv C$

:::message
TypeScript ではハックして型の同一性の概念検証を手に入れることができましたが、別の言語ではそのようなことができない場合もあるかもしれません。

二つの型が相互に部分型であるときにそれらの型を同値関係($\equiv$)として扱うというのは、筆者のアイデアなどでなく、例えば Kotlin 言語の仕様書にある[部分型のセクション](https://kotlinlang.org/spec/type-system.html?paragraph=,subtyping,3#subtyping-rules)などに見られます。

> Two types $A$ and $B$ are equivalent ($A \equiv B$), iff $A <: B \land B <: A$.
:::

これまで部分型関係 (割当可能性) についての検証を行うために使ってきた型構築子 `Assignable` の代わりに、型の同値関係を調べるための新しい型構築子 `Equivalent` と型の関係を調べる `Compat` を以下のように定義して導入します。

```ts
type Equivalent<Fst, Snd> =
  Assignable<Fst, Snd> extends true
    ? Assignable<Snd, Fst> extends true
      ? true
      : false
    : false;

type Compat<Fst, Snd> = Equivalent<Fst, Snd> extends true
  ? "Equivalent"       // Fst ≡ Snd
  : Assignable<Fst, Snd> extends true
      ? "Subtype"      // Fst <: Snd
      : Assignable<Snd, Fst> extends true
        ? "Supertype"  // Fst :> Snd
        : "Unrelated"; // Incomparable
```

これらの型構築子を使うことで、第一型パラメータと第二型パラメータについての部分型関係および同値関係を調べることができます。

```ts
type E1 = Equivalent<1, number>;
// => false
type E2 = Equivalent<object, {}>;
// => true
type C1 = Compat<1, number>;
// => Subtype: 1 <: number
type C2 = Compat<number, string>;
// => Unrelated

// 反射律: A ~ A
type R = Compat<string, string>;
// => Equivalent

// 対称律: A ~ B iff B ~ A
type S1 = Compat<object, Object>;
// => Equivalent
type S2 = Compat<Object, {}>;
// => Equivalent

// 推移律: A ~ B and B ~ C then A ~ C
type T = Compat<object, {}>;
// => Equivalent
```

:::message
一応、型の関係性についてすべてを網羅するための型構築子 `Relation` も導入しておきます。

```ts
type Relation<Fst, Snd> =
  Identical<Fst, Snd> extends true
    ? "Identical"
    : Compat<Fst, Snd>;
```

同一でない場合には同値かどうかをフォールバックとして検証できます。
:::

さて、型について同値関係を導入できたので、次に同値類の概念を導入します。

同値関係を備えた集合 $S$ において、同値関係にある要素同士を一つにまとめてその中の一つの代表となる要素として扱うように集合 $S$ を再構成した集合です。

上で定義した同値関係 (つまり相互に部分型関係あるいは相互に割当可能) を使い、例えば型の集合 $Types$ において `Object`、`object`、`{}` は同値関係があるので、`{}` をその３つの代表となる要素として扱いましょう。このような代表となる要素は代表元 (representative) と呼ばれます。そして、この３つの型について `{}` を代表元として構成し直した集合を `{}` を代表元とする「**同値類** (equivalence class)」と言います。

さらに、オブジェクト型の３つの表現以外にも同値関係 $\equiv$ が定義された同値類をすべてかき集めて新しい型の集合を $Types$ からつくりなおします。このような集合は「商集合 (quotient set)」と呼ばれます。集合 $S$ が同値関係 $\sim$ による同値類から作った集合は $S/{\sim}$ と表記されます。

それでは、型の集合 $Types$ から同値関係 $\equiv$ による同値類から作った商集合 $Types/{\equiv}$ として、この集合を新しく $\text{TYPES}$ と呼ぶことにします。

これで面倒な型の同一性 (identity) について考える必要がなくなり、その代わりに緩い同値関係で型の同じさと違いについて考えることができます。実際、以下のようなかなり面倒な型同士の比較は同値で表現でき、同じものとして考えることができます。

```ts
type I1 = object & Object;
type I2 = Object & {};

type U1 = object | Object;

type C1 = Relation<I1, I2>;
// => Equivalent
type C2 = Relation<I1, U1>;
// => Equivalent
```

$\text{TYPES}$ においては、同値関係に基づき反対称律が成り立っています。

- 反対称律 (asymmetric) : $A <: B \land B <: A \Rightarrow A \equiv B$

:::message
ここで $\equiv$ は核 (kernel) と呼ばれるもので、半順序関係における核は等価関係 $=$ となります。
:::

これで型の集合は半順序集合 (partial ordered set) となりました。