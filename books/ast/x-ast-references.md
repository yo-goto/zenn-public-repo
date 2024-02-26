---
title: "参考文献"
cssclass: zenn
date: 2024-02-10
modified: 2024-02-10
AutoNoteMover: disable
tags: type/zenn/book, TypeTheory/Subtyping, TypeScript/type, math/algebra
aliases: AST本『参考文献』
---

## Algebraic Subtyping

部分型を型システムに持つ言語の代数的構造については Stephen Dolan 氏の "Algebraic Subtyping" という PhD 論文が非常に参考になります。この論文は英国コンピュータ協会(BCS: British Computer Society)が[2017年に開いたコンペ](https://www.bcs.org/events/awards-and-competitions/distinguished-dissertations/previous-winners/2017-competition/)の優勝論文であり、部分型の型システムに ML-like な型推論を組み込むという内容です。

https://www.bcs.org/media/2128/algebraic-subtyping.pdf

メインの内容はかなり難しめで長く筆者も読めていないですが、書き方自体わかりやすいです。最初の方には部分型関係が構築する代数的構造についての様々な数学的な概念が語られており、それらの代数法則を公理として言語を構築するというような手法を行っています。

部分型が持つ代数的構造の背景として、ここで語ったような順序理論や Kleene 代数を始めとする環論や圏論などの概念に比較的簡単にアクセスできるのでおすすめです。

## 文献一覧

書籍
- [型システム入門 プログラミング言語と型の理論](https://www.ohmsha.co.jp/book/9784274069116/)
- [復刊 束論](https://www.kyoritsu-pub.co.jp/book/b10007810.html)
- [圏論の歩き方](https://www.nippyo.co.jp/shop/book/6936.html)
- [圏論の道案内 ～矢印でえがく数学の世界～](https://gihyo.jp/book/2019/978-4-297-10723-9)
- [Steve Awodey, "Category Theory"](https://global.oup.com/ukhe/product/category-theory-9780199237180?cc=gb&lang=en&)
- [ベーシック圏論 普遍性からの速習コース](https://www.maruzen-publishing.co.jp/item/b295027.html)

記事
- [Akihiko Koga, "順序集合や束などに関する基本的な概念の説明"](https://www.cs-study.com/koga/lattice/explanations_on_concepts_of_posets.html)
- [Scala で始める圏論入門](https://criceta.com/category-theory-with-scala/)
- [Haskell/圏論 - Wikibooks](https://ja.wikibooks.org/wiki/Haskell/%E5%9C%8F%E8%AB%96)

論文
- [Stephen Dolan, "Algebraic Subtyping"](https://www.bcs.org/media/2128/algebraic-subtyping.pdf)
