---
title: "はじめに"
cssclass: zenn
date: 2024-02-10
modified: 2024-02-10
AutoNoteMover: disable
tags: type/zenn/book, TypeTheory/Subtyping, TypeScript/type, math/algebra
aliases: AST本『はじめに』
---

## まずはここから

これまでに Zenn では以下のような TypeScript の型についての記事をいくつか執筆してきました。

- [TypeScript における型の集合性と階層性](https://zenn.dev/estra/articles/typescript-type-set-hierarchy)
- [TypeScript の Widening](https://zenn.dev/estra/articles/typescript-widening)
- [TypeScript の Narrowing](https://zenn.dev/estra/articles/typescript-narrowing)

この本では、上記の型についての話をさらに発展させ、TypeScriptの型と型同士の関係性である部分型関係がなす代数的な構造についての解説と、その領域についての調査記録について執筆していきます。

なるべく分かりやすい解説を心がけていますが、調査記録を兼ねているため、内容としての完成度はまだまだであることに注意してください。

なお、部分型を使う型システムを有するプログラミング言語は TypeScript だけではないので、Scala や Kotlin といった部分型を使用している他の言語にもある程度の応用が効く話となっています。

## 注意事項

:::message alert
この本ではかなり多くの数学概念や用語を使いますが、ここでは数学的な厳密性よりもそれらの諸概念を型について理解するためのツールとして利用しています。

また、筆者の調査能力と数学能力の限界で、結構危ない議論をしているかと思いますが、これを読んでくれた皆さんによるツッコミやさらなる調査・検証によって TypeScript 型世界の意味論を解き明かしてくれることを期待していますので、ご協力よろしくお願いします。
:::

コメントや質問などあればこの本に紐づけたスクラップまでお願いします。また GitHub でも PR や issue を受け付けています。

https://zenn.dev/estra/scraps/9a557cab6e6bd0
