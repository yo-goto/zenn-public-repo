---
title: "環論による模型"
cssclass: zenn
date: 2024-02-10
modified: 2024-02-10
AutoNoteMover: disable
tags: type/zenn/book, TypeTheory/Subtyping, TypeScript/type, math/algebra
aliases: AST本『環論による模型』
---

## 半環としての構造

さて、この章では束論の最後で扱った代数法則を環論の観点で見ていきます。

環論 (Ring theory) において、「半環 (Semi-ring あるいは Rig)」とは、環の公理から加法的逆元 (additive inverse) を除いた代数的構造であり、具体的には半環は台集合 $R$ と以下の公理を満たす加法演算 $+$ と乗法演算 $*$ の二つの二項演算を備えた構造 $(R, +, *)$ です。

- 演算の閉性 (closed)
    - $a + b \in R$
    - $a * b \in R$
- 結合律 (associative)
    - $a + (b + c) = (a + b) + c$
    - $a * (b * c) = (a * b) * c$
- 単位元 (identity element) の存在性
    - $a + 0 = a$ (加法単位元)
    - $a * 1 = a$ (乗法単位元)
- 吸収元 (annihilating element) の存在性
    - $a * 0 = 0 = 0 * a$
- 分配律 (distributive)
    - (乗法が加法上に分配的)
        - $a * (b + c) = (a * b) + (a * c)$
        - $(b + c) * a = (b * a) + (c * a)$

加法演算と乗法演算をそれぞれ別々に考えると、$(R, +)$ と $(R, *)$ はそれぞれ単位元 $0$ と $1$ を持つモノイド (monoid) と呼ばれる構造となります。

:::message
モノイド (monoid) とは、台集合 $S$ 上の演算 $・$ が結合律と単位律を満たすだけのマグマ $(S, ・)$ であり、とてもシンプルな代数的構造です。
:::

なお加法と乗法に相当する演算を持つ型の体系が半環であることは、Bartosz Milewski 氏の以下の記事などで語られています。

> Mathematicians have a name for such two intertwined monoids: it’s called a _semiring_. It’s not a full _ring_, because we can’t define subtraction of types. That’s why a semiring is sometimes called a _rig_, which is a pun on “ring without an _n_” (negative).
> ([Simple Algebraic Data Types |   Bartosz Milewski's Programming Cafe](https://bartoszmilewski.com/2015/01/13/simple-algebraic-data-types/) より引用)

TypeScript の型の集合において、加法演算と乗法演算はユニオン型 (`|`) とインターセクション型 (`&`) の型構築子による**型の合成**が相当します。実際に、それぞれの代数法則を $\text{TYPES'}$ に置き換えて、核を同値関係として考えます。

- 演算の閉性 (closed)
    - $A\ |\ B \in \text{TYPES'}$
    - $A\ \&\ B \in \text{TYPES'}$
- 結合律 (associative)
    - $A\ |\ (B\ |\ C) \equiv (A\ |\ B)\ |\ C$
    - $A\ \&\ (B\ \&\ C) \equiv (A\ \&\ B)\ \&\ C$
- 単位元 (identity element) の存在性
    - $A\ |\ never \equiv A$ (加法単位元)
    - $A\ \&\ unknown \equiv A$ (乗法単位元)
- 吸収元 (annihilating element) の存在性
    - $A\ \&\ never \equiv never \equiv never\ \&\ A$
- 分配律 (distributive)
    - (乗法が加法上に分配的)
        - $A\ \&\ (B\ |\ C) \equiv (A\ \&\ B)\ |\ (B\ \&\ C)$
        - $(A\ |\ B)\ \&\ C \equiv (A\ \&\ C)\ |\ (B\ \&\ C)$

具体的に TypeScript の型の集まりが半環となっているか、つまり上記の代数法則を満たしているかどうかを検証してみると、実際に成り立っていることが分かります。※ もちろん厳密な証明などではないので注意してください。

```ts
type A = { fst: number };
type B = { snd: string };
type C = { trd: boolean };

// 結合律
type A1 = Compat<A | (B | C), (A | B) | C>;
// => Equivalent
type A2 = Compat<A & (B & C), (A & B) & C>;
// => Equivalent
type A3 = Compat<A | B | C, C | A | B>

// 単位元の存在性
// 乗法(&)単位元 1 : unknown
type R1 = Compat<A, A & unknown>;
// => Equivalent
// 加法(|)単位元 0 : never
type R2 = Compat<A, A | never>;
// => Equivalent

// 吸収元の存在性
type H1 = Compat<A & never, never>;
// => Equivalent

// 分配律
type D1 = Compat<A & (B | C), (A & B) | (A & C)>;
// => Equivalent
type D2 = Compat<(A | B) & C, (A & C) | (B & C)>;
// => Equivalent
```

なお、`Identical<Fst, Snd>` を使っても同一性レベルで成り立つことが分かります。

さて、TypeScript の型の集合 $\text{TYPES'}$ は更に、可換律と冪等律を満たします。

- 可換律 (commutative)
    - $a + b = b + a$
    - $a * b = b * a$
- 冪等律 (idempotent)
    - $a + a = a$
    - $a * a = a$

またもや対象を型に、核を同値で置き換え、それが成り立つか確認してみます。

- 可換律 (commutative)
    - $A\ |\ B \equiv B\ |\ A$
    - $A\ \&\ B \equiv B\ \&\ A$
- 冪等律 (idempotent)
    - $A\ |\ A \equiv A$
    - $A\ \&\ A \equiv A$

```ts
// 可換律
type C1 = Compat<A | B, B | A>;
// => Equivalent
type C2 = Compat<A & B, B & A>;
// => Equivalent

// 冪等律
type I1 = Compat<A & A, A>;
// => Equivalent
type I2 = Compat<A | A, A>;
// => Equivalent
```

可換律と冪等律を満たす半環をそのまま「**可換冪等半環**」と呼びます。TypeScript の型の集合は可換冪等半環としてみなせることがわかりました。
