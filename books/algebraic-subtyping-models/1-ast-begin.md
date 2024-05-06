---
title: "はじめに"
cssclass: zenn
date: 2024-02-10
modified: 2024-03-03
AutoNoteMover: disable
tags: type/zenn/book, TypeTheory/Subtyping, TypeScript/type, math/algebra
aliases: AST本『はじめに』
---

## プログラミング言語の世情

現代のプログラミング言語の世界では [Rust](https://www.rust-lang.org/ja) や [Haskell](https://www.haskell.org) といった強力な静的型付け言語がますます重要性を増しており、Web の領域でもこれらの言語が注目されています。特に「代数的データ型」や「パターンマッチング」などの機能が強力な表現力と効率性から注目されています。TypeScript もこの影響を受けて代数的データ型、ひいては**代数的構造**を意識したプログラミングが期待されています。

https://www.geekly.co.jp/column/cat-geeklycolumn/specialtalk5_ikyu_shikano

その一方で TypeScript の型がなす代数的構造についての詳細な解説や記事はあまり見かけたことがありません。Haskell などの純粋関数型言語や、関数型との親和性が高い Scala などのコミュニティでは代数的構造についての関心や興味が高く、Web 上にはかなり多くの文献が存在しています。

https://criceta.com/category-theory-with-scala/

そのような文献でよく話題となるのは圏論 (category theory) であり、特に「**型と関数がなす圏**」あるいはより一般に拡張した「**カルテシアン閉圏** (CCC: [Cartesian closed category](https://en.wikipedia.org/wiki/Cartesian_closed_category))」の話が中心となっています。

https://qiita.com/inamiy/items/9af1da1faec22cd968f0

それらの言語習得者が TypeScript の型システムについて言及することがあっても、TypeScript において重要な「部分型」というシステム特有の代数的構造についての理論を展開しているところを見かけることは稀です。

## TypeScriptにおける部分型の意味論

筆者はこれまでに以下のような TypeScript の型がなす構造やその性質についての記事をいくつか執筆してきました。

- [TypeScript における型の集合性と階層性](https://zenn.dev/estra/articles/typescript-type-set-hierarchy)
- [TypeScript の Widening](https://zenn.dev/estra/articles/typescript-widening)
- [TypeScript の Narrowing](https://zenn.dev/estra/articles/typescript-narrowing)

TypeScript においても CCC は考えることができますが、これらの記事の反応を見る限り、TypeScript が持つ型システムでもっと重要かつ基礎的な概念となる部分型(subtyping)についての数理的な意味論はあまり話題に上がっておらず、技術コミュニティにおいてもそのような知識や話題が共有されていないという事実に気づき、より調査を深めていきました。

この本では、上記の記事での話をさらに発展させ、TypeScript における型の関係性である部分型関係がなす代数的な構造について解説し、**強固かつ柔軟な型に関するメンタルモデル**の構築を行います。

より具体的に言えば、部分型はその特性上、どのように部分型関係を順序付けるかという視点で**順序理論** (order theory)が非常に重要なアイデアとなります。そのような順序理論を始めとして、背後で関連している集合論、束論、環論、圏論といった複数の数学理論の観点から、型の代数的構造を眺めることで、型の振る舞いをより理論的に(あるいは自然に)推論できるようになるのではないかと考えています。

## この本の目的

この本は、TypeScript の技術コミュニティにおいて型の意味論を探求が活発にする起爆剤となることを目的としています。上で述べた通り、現状の TypeScript の世界では部分型についての数理的な意味論に関する文献が少ないため、調査に非常に困りました。型システムについての著名な入文書である『型システム入門』を読んではいますが、部分型の数理的な意味論についてはそこまで詳細に記載されていないのが実情です。

https://www.ohmsha.co.jp/book/9784274069116/

このような訳で筆者がこの本を執筆する際にはそもそも TypeScript の文献ではなく、いくつかの数学書と TypeScript ではない別のプログラミング言語の文献が必要となりました。

そのような状況では TypeScript の型の意味論や代数的構造についての知識が共用されづらいわけです。数学が別に得意というわけではないのにこの本を執筆したのは、自分自身がより正確に理解したいからです。

アウトプットとして本を書くのだから詳細に調べあげるのはもちろんのこと、分かりやすく伝えるために自分自身がより理解を求められることとなります。そのような学習上の目的はもちろんのこと、副次的にそういった知識を持つ有識者および探求者がこの話題について記事や考察を出し、自分の知識や理解を超えるものを出してくれれば自分としては非常に助かるわけです。

また、この本は筆者の数学能力に合わせて詳細かつ段階的に必要となる数学知識を導入していますが、決して教科書になるようなものではなく、リアルタイムの調査記録となるように構成しています。参考になる TypeScript の文献はほとんど無いと言って良い状況なので、中には間違った記述もあるかもしれません。

そのような訳で、筆者の調査能力と数学能力の限界からこの本では結構危ない議論をしているかと思いますが、これを読んでくれた皆さんによるツッコミやさらなる調査・検証によって TypeScript における型世界の意味論を解き明かしてくれることを期待していますので、ご協力よろしくお願いします。

コメントや質問などあれば[この本に紐づけたスクラップ](https://zenn.dev/estra/scraps/9a557cab6e6bd0)までお願いします。また GitHub でも PR や issue を受け付けています。

## CHANGELOG

大きな変更のみトラッキングしています。

:::details ChangeLog
- 2024-03-24
  - 『[集合論による模型](6-ast-set-theoretic-model)』および『[束論による模型](7-1-ast-lattice-theoretic-model)』の章で、オブジェクト型の構造について修正
  - 『[圏論による模型](9-ast-category-theoretic-model)』の章で同型性、終対象と始対象の話題を追加
:::

