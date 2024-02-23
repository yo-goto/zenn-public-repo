---
title: "はじめに"
cssclass: zenn
date: 2024-02-10
modified: 2024-02-10
AutoNoteMover: disable
tags: type/zenn/book, TypeTheory/Subtyping, TypeScript/type, math/algebra
aliases: TAM本『はじめに』
---

この本では、これまで書いてきた以下の TypeScript の型についての話をさらに発展させた内容についての解説およびその調査記録について語りたいと思います。

- [TypeScript における型の集合性と階層性](https://zenn.dev/estra/articles/typescript-type-set-hierarchy)
- [TypeScript の Widening](https://zenn.dev/estra/articles/typescript-widening)
- [TypeScript の Narrowing](https://zenn.dev/estra/articles/typescript-narrowing)

というのも、実はこれまでリサーチしていた TypeScript の型が構成する代数的構造についての知識がある程度の内容としてまとまってきたので、改めて記事にしておきたいと思いました。ただし、いつものごとく記事を執筆していく中で内容が長くなってしまったのと、継続的に訂正や追記更新の可能性が高い内容となっため本にすることにしました。

なお、部分型を使う型システムを有するプログラミング言語は TypeScript だけではないので、Scala や Kotlin といった部分型を使用している他の言語にもある程度の応用が効く話となっています。

:::message warn
この本ではかなり多くの数学概念や用語を使いますが、ここでは数学的な厳密性よりもそれらの諸概念を型について理解するためのツールとして利用しています。

また、筆者の調査能力と数学能力の限界で、結構危ない議論をしているかと思いますが、これを読んでくれた皆さんによるツッコミやさらなる調査・検証によって TypeScript 型世界の意味論を解き明かしてくれることを期待していますので、ご協力よろしくお願いします。
:::
