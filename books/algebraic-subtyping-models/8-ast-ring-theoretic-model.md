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

環論 ([ring theory](https://en.wikipedia.org/wiki/Ring_theory)) において、「半環 ([semi-ring](https://en.wikipedia.org/wiki/Semiring) あるいは **rig**)」とは、環の公理から加法的逆元 (additive inverse) を除いた代数的構造であり、より具体的には台集合 $R$ と以下の公理を満たす閉じた加法演算 $+$ と乗法演算 $*$ の二つの二項演算を備えた構造 $(R, +, *)$ です。

| 法則名 | 恒等式 |
|---|:---:|
| 演算の閉性(closure) | $a + b \in R$ <br/> $a * b \in R$ |
| 結合律 (associativity) | $a + (b + c) = (a + b) + c$ <br/> $a * (b * c) = (a * b) * c$ |
| 吸収元 (annihilating element) | $a * 0 = 0 = 0 * a$ |
| 単位元 (identity element) | $a + 0 = a$ (加法単位元: additive identity) <br/> $a * 1 = a$ (乗法単位元: multiplicative identity) |
| 分配律 (distributivity) | (乗法が加法上に分配的) <br/> $a * (b + c) = (a * b) + (a * c)$ <br/> $(b + c) * a = (b * a) + (c * a)$ |

加法演算と乗法演算をそれぞれ別々に考えると、$(R, +)$ と $(R, *)$ はそれぞれ単位元 $0$ と $1$ を持つモノイド (monoid) と呼ばれる構造となります。

:::message
モノイド (monoid) とは、台集合 $S$ 上の演算 $・$ が結合律と単位律を満たすだけのマグマ $(S, ・)$ であり、とてもシンプルな代数的構造です。
:::

TypeScript の型の集合 $\text{TYPES'}$ は実は半環となります。半環における加法演算と乗法演算はユニオン型(`|`)とインターセクション型(`&`)の型構築子による**型の合成**が相当します。

- 演算の閉性 (closure)
  - $A\ |\ B \in \text{TYPES'}$
  - $A\ \&\ B \in \text{TYPES'}$
- 結合律 (associativity)
  - $A\ |\ (B\ |\ C) \equiv (A\ |\ B)\ |\ C$
  - $A\ \&\ (B\ \&\ C) \equiv (A\ \&\ B)\ \&\ C$
- 単位元 (identity element) の存在性
  - $A\ |\ \text{never} \equiv A$ (加法単位元)
  - $A\ \&\ \text{unknown} \equiv A$ (乗法単位元)
- 吸収元 (annihilating element) の存在性
  - $A\ \&\ \text{never} \equiv \text{never} \equiv \text{never}\ \&\ A$
- 分配律 (distributivity): (乗法が加法上に分配的)
  - $A\ \&\ (B\ |\ C) \equiv (A\ \&\ B)\ |\ (B\ \&\ C)$
  - $(A\ |\ B)\ \&\ C \equiv (A\ \&\ C)\ |\ (B\ \&\ C)$

前の章で束が満たす代数法則を紹介し、そこでは加法演算と乗法演算に非常によく似た性質をもっていることがわかりましたが、実際に型の演算は加法と乗法を持つ代数的構造となっています。

具体的に TypeScript の型の集まりが半環となっているか、つまり上記の代数法則を満たしているかどうかは前の章で確認しましたが、型の集合 $\text{TYPES'}$ は更に、可換律と冪等律を満たします。

| 法則名 | 恒等式 |
|---|:---:|
| 可換律 (commutativity) | $a + b = b + a$ <br/> $a * b = b * a$ |
| 冪等律 (idempotent) | $a + a = a$ <br/> $a * a = a$ |

これも前の章で実は見ていましたね。

- 可換律 (commutative)
  - $A\ |\ B \equiv B\ |\ A$
  - $A\ \&\ B \equiv B\ \&\ A$
- 冪等律 (idempotent)
  - $A\ |\ A \equiv A$
  - $A\ \&\ A \equiv A$

```ts
// 可換律
type C1 = Relation<A | B, B | A>;
// => Identical
type C2 = Relation<A & B, B & A>;
// => Identical

// 冪等律
type I1 = Relation<A | A, A>;
// => Identical
type I2 = Relation<A & A,, A>;
// => Identical
```

可換律と冪等律を満たす半環をそのまま「**可換冪等半環**」と呼びます。これで TypeScript の型の集合は代数法則の観点からは可換冪等半環としてみなせることがわかりました。

なお加法と乗法に相当する演算を持つ型の体系が半環であることは、Bartosz Milewski 氏の以下の記事などで語られています。

> Mathematicians have a name for such two intertwined monoids: it’s called a _semiring_. It’s not a full _ring_, because we can’t define subtraction of types. That’s why a semiring is sometimes called a _rig_, which is a pun on “ring without an _n_” (negative).
> ([Simple Algebraic Data Types |   Bartosz Milewski's Programming Cafe](https://bartoszmilewski.com/2015/01/13/simple-algebraic-data-types/) より引用)
