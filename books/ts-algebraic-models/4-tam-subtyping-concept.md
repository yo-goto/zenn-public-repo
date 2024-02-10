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
