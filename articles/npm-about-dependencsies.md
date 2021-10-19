---
title: "npmの依存関係について勘違いしていたこと"
emoji: "🐉"
type: "tech"
topics: [npm, node]
published: true
date: 2021-10-23
url: "https://zenn.dev/estra/articles/npm-about-dependencsies"
aliases: [記事_npmの依存関係について勘違いしていたこと]
tags: " #node/npm #type/zenn "
---


# はじめに

前回の記事でnpmの依存関係とロックファイルについて調べてみましたが、その後色々と調べたり試しているうちに、勘違いしていたことや分かっていなかったことがいくつかあったことが判明したのでそれらについてのまとめを作成したいと思います。

前回の記事↓
https://zenn.dev/estra/articles/npm-lockfile-dependencies


# そもそも
前回の記事では、chalkやcowsayというパッケージをインストールしてみてロックファイルの構造や依存関係のことを考えた。そして｢node_modulesフォルダには`package.json`ファイルのdependenciesに記載されているパッケージだけでなく、それらパッケージの依存もすべてインストールする｣ということが分かった、というのが趣旨であった。実際node_modulesフォルダを開いてみたら知らないパッケージが多く配置されていた。

そしてこれらがインストールされたすべての依存パッケージであると思っていたが、かなり勘違いしていたことが分かった。というのもとあるパッケージのフォルダには更にnode_modulesというフォルダがあったからだ。

これが分かったのは`npm explain`コマンドを実行して違和感を覚えたためであった。

前回と同じく、cowsayというパッケージを単体でインストールした際のnode_modulesフォルダの構造を見てみる。node_modulesの構造を見てみると以下のように依存関係であるパッケージがフォルダとして配置されている。

![](/images/npm-dependencies/img_npmdependencies_4.jpg)

この中のパッケージで実際にどのような依存によってインストールされるのかということを、前回の記事で`npm explain`というコマンドを使って理由追求することができるということを説明した。

この`npm explain`コマンドで指定パッケージの依存元を階層的に表示することができる。実際にこのコマンドでansi-regexというパッケージのインストール原因を追求してみると次のような出力結果を得た。

```log
$ npm explain ansi-regex
ansi-regex@5.0.1
node_modules/ansi-regex
  ansi-regex@"^5.0.1" from strip-ansi@6.0.1
  node_modules/strip-ansi
    strip-ansi@"^6.0.0" from cliui@6.0.0
    node_modules/cliui
      cliui@"6.0.0" from the root project
    strip-ansi@"^6.0.0" from wrap-ansi@6.2.0
    node_modules/cliui/node_modules/wrap-ansi
      wrap-ansi@"^6.2.0" from cliui@6.0.0
      node_modules/cliui
        cliui@"6.0.0" from the root project
    strip-ansi@"^6.0.1" from string-width@4.2.3
    node_modules/string-width
      string-width@"^4.2.0" from cliui@6.0.0
      node_modules/cliui
        cliui@"6.0.0" from the root project
      string-width@"^4.1.0" from wrap-ansi@6.2.0
      node_modules/cliui/node_modules/wrap-ansi
        wrap-ansi@"^6.2.0" from cliui@6.0.0
        node_modules/cliui
          cliui@"6.0.0" from the root project

ansi-regex@3.0.0
node_modules/wrap-ansi/node_modules/ansi-regex
  ansi-regex@"^3.0.0" from strip-ansi@4.0.0
  node_modules/wrap-ansi/node_modules/strip-ansi
    strip-ansi@"^4.0.0" from wrap-ansi@3.0.0
    node_modules/wrap-ansi
      wrap-ansi@"3.0.0" from the root project
    strip-ansi@"^4.0.0" from string-width@2.1.1
    node_modules/wrap-ansi/node_modules/string-width
      string-width@"^2.1.1" from wrap-ansi@3.0.0
      node_modules/wrap-ansi
        wrap-ansi@"3.0.0" from the root project
```

ansi-regexが2つあった...

よく見てみると、`anis-regex@5.0.1`と`ansi-regex@3.0.0`が存在している。そして`npm explain`ではインストール場所も示してくれるらしいが、`ansi-regex@5.0.1`は`node_modules/ansi-regex`にある一方、`ansi-regex@3.0.0`は`node_modules/wrap-ansi/node_modules/ansi-regex`にある。

:::message
ちなみに、前回の記事を書いた時点では`npm explain`がパッケージの依存元を示すということしか分かっていなかった。
:::

ということで`ansi-regex`のバージョン`3.0.0`パッケージが`node_modules/wrap-ansi/node_modules/ansi-regex`というネストされたnode_modulesフォルダに配置されているわけだが、なぜnode_modulesフォルダがネストされるのか、なぜ別のバージョンがインストールされているのかということが分からなかったのでこれについて色々調べてみた。

## 調べてみると

実際に調べていくうちに分かったのは、npmはv2とv3で大きな変更があり、**パッケージの配置方法が大きく変わった**ということだ。パッケージの配置方法はかなり重要なコンセプトであり、現在ではv7で登場したarboristというプログラムがフォルダツリーと呼ばれるモデルを構築し、実際のフォルダ配置の中枢を担っているらしい。

npmは現在バージョン8まででており、かなり歴史があるようなのでコアコンセプトは何年か前の記事で紹介されているものが多かった。しかし、最近登場したArboristについてはセキュリティ上問題があったという記事のみで、日本語でも英語でも具体的にどんなことをやっているのかという記事をみつけることができなかった。

パッケージ配置のコンセントについては後発のyarnやpnpmといったもの、rushというモノリポジトリ管理に特化したツールなどで改良や独自のアプローチがとられているため、それらツールのドキュメントではnpmの問題点やnpmのアプローチに関しての独自定義の専門用語などを使って説明されているものを見つけることができたので紹介する。

ちなみに、npmのドキュメントはv7が最新のものであるが、必ずしもわかりやすいとは言えない。後発ツールのドキュメントやnpmの過去のドキュメントを見つけることができて、ようやくnpmの全貌がおぼろげに見えてきた。それらの用語を使って今までnpmについて勘違いしていたことと知らなかったことを説明していく。

## 前回の時点での勘違い
しらべて自分が知らなかった、もしくは勘違いしていたことは次の事柄である。

- npmの役割そのもの
- `node_modules`フォルダの構造
- `npm install`が実際にやること
- ロックファイルの役割と構造、そしてリストされているパッケージについて
- `npm explain`と`npm ls`のコマンドが表現するものについて

これらについて以下まとめていきたいと思う。


# npmの役割

まず前提を知る必要がある。というのも、npm自体の役割として自分が見落としていた部分があることに気づいた。

『そもそもnpmとは何をするためのものか？』

実際、npmのことを説明しようと思ったらどのように説明すればよいのか。これは知っておく必要があった。ちなみに公式ドキュメントには以下のように書かれている。

>npmはNode JavaScriptプラットフォームのためのパッケージマネジャーである。npmはnodeによってモジュールが発見できるようにモジュールを正しく配置し、依存関係の衝突を賢くマネージする。
>
>npm is the package manager for the Node JavaScript platform. **It puts modules in place so that node can find them, and manages dependency conflicts intelligently**.
>\- [npm | npm Docs](https://docs.npmjs.com/cli/v7/commands/npm)より引用


npmと呼ばれるパッケージマネジャーの役割は大きく分けて次の2つとのことだ。

- **nodeが**発見できるようにモジュールを配置
- 依存関係の衝突のマネージ

ただ、このモジュールの配置というのが、パッケージをnode_modulesフォルダに配置するだけなのに依存関係の解決と並んでいるのには最初違和感があった。しかし、当初自分が想定していたnode_modulesフォルダへの配置と`npm install`によって行われているモジュールの配置はまったく異なるものであることが後々判明した。これについても後で説明していく。

# ロックファイルの構造再び
前回の記事でcowsayのパッケージを単体インストール際の`package-lock.json`ファイルを見たが、今回もそれを使っていく。

まずは再び`package-lock.json`ファイルの構造を見てみる(※相対パスの後は省略してある)。

```json:pakcage-lock.jsonの構造
{
  "name": "cows",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "requires": true,
  "packages": {
    "": {...},
    "node_modules/ansi-regex": {...},
    "node_modules/ansi-styles": {...},
    "node_modules/camelcase": {...},
    "node_modules/cliui": {...},
    "node_modules/cliui/node_modules/ansi-regex": {...},
    "node_modules/cliui/node_modules/is-fullwidth-code-point": {...},
    "node_modules/cliui/node_modules/string-width": {...},
    "node_modules/cliui/node_modules/strip-ansi": {...},
    "node_modules/color-convert" {...}, 
    "node_modules/color-name" {...},
    "node_modules/cowsay" {...},
    "node_modules/decamelize" {...},
    "node_modules/emoji-regex" {...},
    "node_modules/find-up" {...},
    "node_modules/get-caller-file" {...},
    "node_modules/get-stdin" {...},
    "node_modules/is-fullwidth-code-point" {...},
    "node_modules/locate-path" {...},
    "node_modules/p-limit" {...},
    "node_modules/p-locate" {...},
    "node_modules/p-try" {...},
    "node_modules/path-exists" {...},
    "node_modules/require-directory" {...},
    "node_modules/require-main-filename" {...},
    "node_modules/set-blocking" {...},
    "node_modules/string-width" {...},
    "node_modules/strip-ansi" {...},
    "node_modules/strip-final-newline" {...},
    "node_modules/which-module" {...},
    "node_modules/wrap-ansi" {...},
    "node_modules/wrap-ansi/node_modules/ansi-regex" {...},
    "node_modules/wrap-ansi/node_modules/is-fullwidth-code-point" {...},
    "node_modules/wrap-ansi/node_modules/string-width" {...},
    "node_modules/wrap-ansi/node_modules/strip-ansi" {...},
    "node_modules/y18n" {...},
    "node_modules/yargs" {...},
    "node_modules/yargs-parser" {...},
    "node_modules/yargs/node_modules/ansi-regex" {...},
    "node_modules/yargs/node_modules/is-fullwidth-code-point" {...}, 
    "node_modules/yargs/node_modules/string-width" {...},
    "node_modules/yargs/node_modules/strip-ansi" {...}
  },
  "dependencies": {...}
}
```

npm v7でみるべきpackageのフィールドに記載されているのはnode_modulesフォルダにインストールされたすべての依存パッケージである。依存パッケージの名前は**プロジェクトルートからの相対パス**で示されている。

"node_modules/cliui"の後を見ると、つぎのようにnode_moduelsフォルダがネストされた状態でansi-regex、is-fullwidth-code-print、string-width、strip-ansiという4つのパッケージが配置されていることがわかる。

```json
    "node_modules/cliui/node_modules/ansi-regex": {...},
    "node_modules/cliui/node_modules/is-fullwidth-code-point": {...},
    "node_modules/cliui/node_modules/string-width": {...},
    "node_modules/cliui/node_modules/strip-ansi": {...},																
```

さらに`package-lock.json`ファイルを見てみると同様にwrap-ansiとyargsパッケージの下にもネストされたnode_moduelsフォルダがあることがわかる。

改めて`package-lock.json`の役割は何か確認する。

>It describes the **exact tree that was generated, such that subsequent installs are able to generate identical trees**, regardless of intermediate dependency updates.
>(中略)
>As of npm v7, lockfiles include **enough information to gain a complete picture of the package tree**, reducing the need to read `package.json` files, and allowing for significant performance improvements.
>\- [package-lock.json | npm Docs](https://docs.npmjs.com/cli/v7/configuring-npm/package-lock-json)より引用

ロックファイルは後続のインストールにおいて同一のツリーを生成するための正確な**ツリー**を記述する、とある。これが意味していたところは同一のnode_moduelsツリー(またはフォルダツリー)を再現するためにすべてのインストールされたパッケージの正確なバージョンと相対パスでのロケーションを記録して、node_moduleフォルダの正確なレイアウトを表現するということだったのだ。

# ツリーとグラフ
｢ツリー｣という言葉がでてきたが、そもそも依存関係というのはツリー構造になっていない。

前回の記事で紹介した[npmのパッケージの依存関係を表現してくれるツール](https://npm.anvaka.com/#/)でも見たが、これは依存グラフ(dependency graph)と呼ばれている。

![](/images/npm-dependencies/img_npmdependencies_3.jpg)


>パッケージマネジメントについて議論される際にdependency tree(依存ツリー)という言葉に言及したくなるが、実質的にdependenciesの間での関係性は厳密にはツリー(tree)ではなくグラフ(graph)である。サイクル(閉路)や重複する関係性を持つため、単一のノードがシステムの中で複数の役割を果たすことができる。
>\- [npm Blog Archive: npm v7 Series - Arborist Deep Dive](https://blog.npmjs.org/post/618653678433435649/npm-v7-series-arborist-deep-dive.html)より

依存とはツリーではなくグラフ構造であった。更に正確に言うならば[有向非巡回グラフ](https://ja.wikipedia.org/wiki/%E6%9C%89%E5%90%91%E9%9D%9E%E5%B7%A1%E5%9B%9E%E3%82%B0%E3%83%A9%E3%83%95)(DAG: Directed acyclic graph)という名前のグラフになる。DAGとTreeの構造は下図のように異なる。

| DAG | Tree | 
|---|---|
| ![DAG](https://upload.wikimedia.org/wikipedia/commons/thumb/3/39/Directed_acyclic_graph_3.svg/2560px-Directed_acyclic_graph_3.svg.png) | ![Tree](https://upload.wikimedia.org/wikipedia/commons/thumb/6/67/Sorted_binary_tree.svg/1920px-Sorted_binary_tree.svg.png) | 


# npm lsとexplainが表現するもの

となると、`npm ls`コマンドで出力されるツリー構造は一体何なのか？ドキュメントには以下のように書いてある。

>`npm ls -a`で出力されるツリーはパッケージの依存関係に基づいた論理的な依存ツリー(logical dependency tree)であって、物理的な`node_modules`ディレクトリ内部のレイアウトではない。
>\- [npm-ls | npm Docs](https://docs.npmjs.com/cli/v7/commands/npm-ls)より

また新しい単語がでてきたが、論理的な依存ツリーとは一体何なのか？

正確な定義は記載されていないが公式ドキュメントの後にはこう記されている。

>`npm ls`コマンドの出力と動作はnpmがすべての依存関係を単純にネストするnode_modulesフォルダを作成していたときには非常に意味があった(npm v2以前)。そのような場合には、論理的依存ツリーと物理的にディスク上に存在しているパッケージツリー(physical tree of packages on disk)がほぼ同じであった。
>npm v3において登場したインストール時自動重複排除(automatic install-time deduplication of depednencies)の機能によってnpm lsの出力は論理的依存グラフ(logical depdendency graph)をツリー構造として表示するように修正された。
>\- [npm-ls | npm Docs](https://docs.npmjs.com/cli/v7/commands/npm-ls)より

グラフ構造をツリーとして表現されたものとのこと。厳密な言葉の定義がないので自分なりに解釈するとnode_modulesの物理的なレイアウトを表現していないためactualやphysicalではなくlogicalな依存ツリーであるということではないだろうか。

そして逆に`npm explain`はその論理的依存ツリー構造を特定のパッケージから見て逆にたどったものを表現していると考えられる。

このドキュメントででてきた概念として重複排除(deduplication)やnode_moduelsのネストが重要である。このドキュメントだけを見てもよくわからないがnpm v2やv3に関するドキュメントや解説記事をみるとようやく意味がつかめる。


まずは重複排除について

`npm ls -a`で表示されるdedupedは重複したパッケージということではなく重複排除(deduplicaiton)を行われたということを意味していた。

:::message
**重複排除**とはバックアップの際に対称データを解析して、重複データを自動的に検知して排除する技術。英語ではDe-dupulication, deduplicationなどと呼ばれれる。データサイズをなるべく抑えることを目的とする。動詞は｢dedupe｣
\- 参考 [重複排除（デデュプリケーション、デデュープ）](https://www.fujitsu.com/jp/products/computing/storage/lib-f/tech/de-duplication/)
:::

例えば、`npm ls`コマンドは後にパッケージ名を指定するとそれが末端となるツリーを表示してくれる。

```shell
$ npm ls strip-ansi
cows@1.0.0 /Users/userName/Projects/Cows
└─┬ cowsay@1.5.0
  ├─┬ string-width@2.1.1
  │ └── strip-ansi@4.0.0
  └─┬ yargs@15.4.1
    ├─┬ cliui@6.0.0
    │ ├─┬ string-width@4.2.3
    │ │ └── strip-ansi@6.0.1 deduped
    │ ├── strip-ansi@6.0.1
    │ └─┬ wrap-ansi@6.2.0
    │   ├─┬ string-width@4.2.3
    │   │ └── strip-ansi@6.0.1 deduped
    │   └── strip-ansi@6.0.1
    └─┬ string-width@4.2.3
      └── strip-ansi@6.0.1
```

strip-ansiについてls出力してみると上の様に表示される。strip-ansiといパッケージが出現しているのは6箇所あるが、dedupedとついた箇所が2つある。これはその2箇所にインストールされるはずだったstrip-ansiパッケージは実際には取り除かれているということを意味している。

逆にstrip-ansiがどこにインストールされているかを`npm expalin`で調べる。explainコマンドは指定パッケージのインストール原因となる依存関係をボトムアップにプロジェクトルートまで駆け上がって表示してくれる。

```log
$ npm why strip-ansi
strip-ansi@6.0.1
node_modules/cliui/node_modules/strip-ansi
  strip-ansi@"^6.0.0" from cliui@6.0.0
  node_modules/cliui
    cliui@"^6.0.0" from yargs@15.4.1
    node_modules/yargs
      yargs@"15.4.1" from cowsay@1.5.0
      node_modules/cowsay
        cowsay@"^1.5.0" from the root project
  strip-ansi@"^6.0.1" from string-width@4.2.3
  node_modules/cliui/node_modules/string-width
    string-width@"^4.2.0" from cliui@6.0.0
    node_modules/cliui
      cliui@"^6.0.0" from yargs@15.4.1
      node_modules/yargs
        yargs@"15.4.1" from cowsay@1.5.0
        node_modules/cowsay
          cowsay@"^1.5.0" from the root project

strip-ansi@4.0.0
node_modules/strip-ansi
  strip-ansi@"^4.0.0" from string-width@2.1.1
  node_modules/string-width
    string-width@"~2.1.1" from cowsay@1.5.0
    node_modules/cowsay
      cowsay@"^1.5.0" from the root project

strip-ansi@6.0.1
node_modules/wrap-ansi/node_modules/strip-ansi
  strip-ansi@"^6.0.0" from wrap-ansi@6.2.0
  node_modules/wrap-ansi
    wrap-ansi@"^6.2.0" from cliui@6.0.0
    node_modules/cliui
      cliui@"^6.0.0" from yargs@15.4.1
      node_modules/yargs
        yargs@"15.4.1" from cowsay@1.5.0
        node_modules/cowsay
          cowsay@"^1.5.0" from the root project
  strip-ansi@"^6.0.1" from string-width@4.2.3
  node_modules/wrap-ansi/node_modules/string-width
    string-width@"^4.1.0" from wrap-ansi@6.2.0
    node_modules/wrap-ansi
      wrap-ansi@"^6.2.0" from cliui@6.0.0
      node_modules/cliui
        cliui@"^6.0.0" from yargs@15.4.1
        node_modules/yargs
          yargs@"15.4.1" from cowsay@1.5.0
          node_modules/cowsay
            cowsay@"^1.5.0" from the root project

strip-ansi@6.0.1
node_modules/yargs/node_modules/strip-ansi
  strip-ansi@"^6.0.1" from string-width@4.2.3
  node_modules/yargs/node_modules/string-width
    string-width@"^4.2.0" from yargs@15.4.1
    node_modules/yargs
      yargs@"15.4.1" from cowsay@1.5.0
      node_modules/cowsay
        cowsay@"^1.5.0" from the root project
```

strip-ansiの同一名パッケージが4つ入っている。
バージョンは4.0.0が一つと6.0.1が3つ。しかし、よく見てみると最初の`strip-ansi@6.0.1`と二番目の`strip-ansi@6.0.1`にはそのパッケージが必要とされている箇所がそれぞれ2箇所ずつあることに気づく。それぞれを整理してみると以下のようにそれぞのパッケージ場所と必要とされている条件がすこしずつ違う。

- `strip-ansi@6.0.1 node_modules/cliui/node_modules/strip-ansi`
	- `strip-ansi@"^6.0.0" from cliui@6.0.0 node_modules/cliui`
	- `strip-ansi@"^6.0.1" from string-width@4.2.3 node_modules/cliui/node_modules/string-width`
- `strip-ansi@6.0.1 node_modules/wrap-ansi/node_modules/strip-ansi`
	- `strip-ansi@"^6.0.0" from wrap-ansi@6.2.0 node_modules/wrap-ansi`
	- `strip-ansi@"^6.0.1" from string-width@4.2.3 node_modules/wrap-ansi/node_modules/string-width`

書かれているパスとパッケージ名からnode_modulesフォルダ内での物理的なレイアウト構造を再現してみる。

```log
.
└── node_modules
    ├── cliui@6.0.0
    │   └── node_modules
    │       └── strip-ansi@6.0.1
    ├── strip-ansi@4.0.0
    ├── wrap-ansi@6.2.0
    │   └── node_modules
    │       └── strip-ansi@6.0.1
    └── yargs@15.4.1
        └── node_modules
            └── strip-ansi@6.0.1
```

これと`npm ls`で出力したツリー構造をみてみると何かわかるだろうか？

```log
cows@1.0.0 /Users/userName/Projects/Cows
└─┬ cowsay@1.5.0
  ├─┬ string-width@2.1.1
  │ └── strip-ansi@4.0.0
  └─┬ yargs@15.4.1
    ├─┬ cliui@6.0.0
    │ ├─┬ string-width@4.2.3
    │ │ └── strip-ansi@6.0.1 deduped
    │ ├── strip-ansi@6.0.1
    │ └─┬ wrap-ansi@6.2.0
    │   ├─┬ string-width@4.2.3
    │   │ └── strip-ansi@6.0.1 deduped
    │   └── strip-ansi@6.0.1
    └─┬ string-width@4.2.3
      └── strip-ansi@6.0.1
```

なんだか構造が似ているような気がするが正直よくわからなかった。これの違いがわかるようになるためにはnpm v2とv3での機能的な変更についての知識が必要になる。もういちど`npm ls`コマンドについての公式ドキュメントを呼んで見る。

>`npm ls`コマンドの出力と動作はnpmがすべての依存関係を単純にネストするnode_modulesフォルダを作成していたときには非常に意味があった(npm v2以前)。そのような場合には、論理的依存ツリーと物理的にディスク上に存在しているパッケージツリー(physical tree of packages on disk)がほぼ同じであった。
>\- [npm-ls | npm Docs](https://docs.npmjs.com/cli/v7/commands/npm-ls) より

ちなみに(npm v2以前)という言葉は自分で補った部分である。npm v2とv3以降ではnode_modulesフォルダの構造がまったく違うものになっている。そして過去のnpm(v2)では`npm ls`で出力されたツリーとnode_modulesフォルダがほぼ同じであったという旨が読み取れる。

:::message
ちなみに初見でこの旨を読み取るのは難しく、他のドキュメント等を読んでからようやく理解できた。
:::

これについては`npm install`のアルゴリズムがv2とv3で大きく変わったことを知る必要がある。npm v3については過去のドキュメントが以下のURLから見ることができる。

https://npm.github.io/how-npm-works-docs/npm3/how-npm3-works.html


## 依存にまつわる用語
node_moduleフォルダの構造について説明する前にここで一度用語の紹介をしておきたいと思う。というのもnode_modulesフォルダの構造がなぜそうなるのを理解するのに役立つからだ。

- Prod: Production dependencies
	- 動作や実行するのに必要なパッケージ
	- `package.json`の`dependencies`
	- `peerDependencies`や`optionalDependencies`もこれに含まれる
- Dev: Development dependencies
	- 開発中のみに必要とされるパッケージ
	- `package.json`の`devDependencies`
- Primary dependencies: Direct dependencies
	- プロジェクトルートの`package.json`ファイルから要求される依存パッケージ
- Secondary dependencies: Indirect dependenceis
	- Primary(Direct) dependenciesまたは他のSecondary(Indirect)から呼ばれる間接的な依存パッケージ
	- node_moduelsフォルダに含まれる大多数がこちらに属する
	- 別名
		- [Transitive dependencies](https://en.wikipedia.org/wiki/Transitive_dependency)(推移的依存関係)
		- [Phantom dependencies](https://rushjs.io/pages/advanced/phantom_deps/)(rushにおける定義)
- Flat package
	- node_modulesフォルダのルートに配置されたパッケージ
	- この状態を｢Flatにインストール｣とか言う。
- Nested package
	- Flat packgeフォルダ内部のnode_modulesフォルダ内部に配置されたパッケージ
- deep tree
	- node_modulesフォルダが何重にもネストされたフォルダの状態
- [npm doppelganger](https://rushjs.io/pages/advanced/npm_doppelgangers/)
	- npmのインストールによって複数個インストールされてしまった同一バージョンのパッケージのことを指す
- node_modules tree(Folder tree)
	- 実際のnode_modulesフォルダの構造
- logical dependnecy tree(論理的依存ツリー)
	- node_modules treeとは異なる依存の論理的な木構造
	- dependency graphをtree構造にoverlayしたもの
	- `npm ls`コマンドで出力されるtree構造
- dependency graph
	- パッケージ間の本質的な依存関係をグラフ構造に表現したもの
	- DAG(Directed acyclic graph)


参考
https://tech.groww.in/introduction-to-package-dependency-resolution-in-npm-ad1b374fc13a

https://snyk.io/blog/whats-an-npm-dependency/

以上のようにdependecnyやtreeといってもいろいろな種類や見方がある。


# Primary(Direct)の配置
PrimaryとSecondaryで考えるとパッケージの配置について理解しやすくなる。

`npm install`ではまず`package.json`に記載されているPrimary(Direct) dependenciesのパッケージをnode_moduelsのルートにFlatに配置する。

例えば、cowsayを入れるとまずはディレクトリがイメージとしてこのような状態となる。

```shell
.
├── node_modules
│   └── cowsay
└── package.json
```

`npm install`にオプション`--timing`を付けるとインストールログをみることができる。

@[gist](https://gist.github.com/yo-goto/8bcb8047d9fb1103590662ab386e47ee)

npmレジストリからパッケージをfetchする順番がみることができるが、順番としてはまずはPrimaryであるcowsayが一番最初。そのつぎにSecondary(Indirect) dependenciesをfetchするがまずはcowsayのdependenciesであるget-sidin、string-width、strip-final-newline、yargsの4つをfetchしている。

ちなみにこの依存のレベルは`npm ls --depth`でみることができる。

```log
$ npm ls --depth=1
cows@1.0.0 /Users/userName/Projects/Cows
└─┬ cowsay@1.5.0
  ├── get-stdin@8.0.0
  ├── string-width@2.1.1
  ├── strip-final-newline@2.0.0
  └── yargs@15.4.1
```

Primaryの配置が終わった後はこの`--depth=1`のSecondaryの配置が考慮される。

その次には以下の`--depth=2`のSecondaryについて考える...といったように依存の深さのレベルごとにfetchや配置が行われていくと考える(インストールログを見てみても深さごとにfetchが行われているのがわかる)。

```log
$ npm ls --depth=2
cows@1.0.0 /Users/userName/Projects/Cows
└─┬ cowsay@1.5.0
  ├── get-stdin@8.0.0
  ├─┬ string-width@2.1.1
  │ ├── is-fullwidth-code-point@2.0.0
  │ └── strip-ansi@4.0.0
  ├── strip-final-newline@2.0.0
  └─┬ yargs@15.4.1
    ├── cliui@6.0.0
    ├── decamelize@1.2.0
    ├── find-up@4.1.0
    ├── get-caller-file@2.0.5
    ├── require-directory@2.1.1
    ├── require-main-filename@2.0.0
    ├── set-blocking@2.0.0
    ├── string-width@4.2.3
    ├── which-module@2.0.0
    ├── y18n@4.0.3
    └── yargs-parser@18.1.3
```

# Secondary(Indirect)の配置
それでは`--depth=1`のレベルの4つの依存パッケージについてPrimaryがnode_modulesフォルダのルート配置された次に配置されるとしたらどういった構造で配置されるか。

npm v2ではSecondaryはPrimaryのパッケージディレクトリにすべてネストすることで解決を図っていた。

例えば、AとBというパッケージが`package.json`のdependenciesと宣言されており、さらにAはXとY、BはXとZという依存を持つとする。

```shell
.
├── A
│   ├── X
│   └── Y
└── B
    ├── X
    └── Z
```

これを先程と同じように`npm install`で配置するとnpm v2では以下のようにネストさせる。

```shell
.
├── node_modules
│   ├── A
│   │   └── node_modules
│   │       ├── X
│   │       └── Y
│   └── B
│       └── node_modules
│           ├── X
│           └── Z
└── package.json
```

 X,Y,Zが更に依存を持つ場合にはそれぞのパッケージディレクトリに更にネストさせたnode_modulesをつくっていくわけであるが、それらの依存パッケージがさらに依存を持つと、再びネストしたnode_modulesフォルダが作成されていく。最終的にはフォルダは何重にもネストされた状態になってしまう。このような状態をDeep treeと言う。
 
この形式であればPrimary dependenciesのみがFlatに配置され、Secondary dependenciesはNestされた状態でわかりやすい構造であるし、`npm ls`で出力した論理的依存ツリーと構造がほぼ一致する。npm v2ではnode_modulesフォルダをこのような構造にしていた。
 
  しかし、ディレクトリの状態を見ればXというパッケージはAとBのフォルダに両方存在していることがわかる。これではデータ量が無駄におおきくなってしまう。Deep treeの状態であればどれだけ無駄な重複ができてしまうかわからない。
 
 このような状態を避けてなるべく冗長性をへらすようにnpm v3では構造の最適化が行われるようになった。すべてのFlat dependenciesはPrimaryだけではなく一部のSecondaryを含むようになり、node_noduelsフォルダルートにFlatに配置されるようになった。上の構造のようにXが重複している場合にはFlatに配置してあげることによって冗長性をへらすことができる。実際には必ずしもルートに配置するのではなくパッケージが存在する上の階層のnode_moduelsフォルダに同じ名前のパッケージがなければパッケージを配置しようとし、更に上の階層のnode_modulesフォルダに同名パッケージがなければさらに上の階層に配置...というように最終的にルートまで上がって同じ名前のパッケージがなければFlatに配置する。これをパッケージの巻き上げ(hoisting)という。
 
```shell
├── node_modules
│   ├── A
│   │   └── node_modules
│   │       └── Y
│   ├── B
│   │   └── node_modules
│   │       └── Z
│   └── X
└── package.json
```
 
 これが基本コンセントとなるが、npm v3ではあらゆるSecondary dependenciesについて可能な限り巻き上げを行いFlatに配置してフォルダツリーをなるべく浅くしようとする。つまり上の構造はただしくなく実際には以下のように配位される。
 
```shell
.
├── node_modules
│   ├── A
│   ├── B
│   ├── X
│   ├── Y
│   └── Z
└── package.json
```
 
 これによってインストールされるXを一つ減らすことできた。この重複したパッケージを削除することをdedupeといい。巻き上げ(hoisting)とこれに伴う自動的な重複排除(deduplicaition)がnpm v3で実装された。
 
 例えばchalkのような単純なパッケージを単体でインストールした際には同じようにすべてのパッケージがFlatに配置される。
 
```shell
.
├── node_modules
│   ├── ansi-styles
│   ├── chalk
│   ├── color-convert
│   ├── color-name
│   ├── has-flag
│   └── supports-color
├── package-lock.json
└── package.json
```

これに論理的依存ツリーは`npm ls -a`で見ると次のようになっている。

```shell
$ npm ls -a
chalktest@1.0.0 /Users/userName/Projects/ChalkTest
└─┬ chalk@4.1.2
  ├─┬ ansi-styles@4.3.0
  │ └─┬ color-convert@2.0.1
  │   └── color-name@1.1.4
  └─┬ supports-color@7.2.0
    └── has-flag@4.0.0
```

chalkの依存グラフはシンプルなものであったからすべてFlatに配置することができた。重複排除されたパッケージがあればdedupedと表示されるが、chalkの場合には重複がないためなにも削除されなかった。

問題なのはこれでもネストが起きてしまう場合である。再びcowsayのインストールに戻る。

cowsayのnode_moduelsでは例えばstrip-ansiというパッケージが複数個存在し、他パッケージのネストされたnode_modulesフォルダに配置されているものがいくつかあった。node_modulesフォルダのstirp-ansiがインストールされている場所だけを見てみると次のようになっている。

```shell
.
└── node_modules
    ├── cliui
    │   └── node_modules
    │       └── strip-ansi # @6.0.1
    ├── strip-ansi # @4.0.0
    ├── wrap-ansi
    │   └── node_modules
    │       └── strip-ansi # @6.0.1
    └── yargs
        └── node_modules
            └── strip-ansi # @6.0.1
```

strip-ansiのバージョン4.0.0はFlatに配置されているがバージョン6.0.1が3つネストされて配置されている事がわかる。

`npm explain`で見たように。あるパッケージから求められるバージョンの条件と別のパッケージから求められるパッケージの条件がことなっている場合がある。そういった場合には依存の衝突が起きていることになり、別々のバージョンで同じパッケージをインストールしなくてはならない。その場合ルートまでに巻き上げが行われる(=Flatに配置される)パッケージは一つとなる。単純に同じディレクトリに同じ名前のフォルダを配置することができないからだ。つまりstrip-ansiのバージョン6.0.1は巻き上げがこれ以上できないので仕方なく配置されている。ちなみにこういった複数個インストールされている同一バージョンのパッケージのことを[rush](https://rushjs.io)のドキュメントではnpmドッペルゲンガー(doppelganger)と呼ばれている。

一方もう一度`npm ls`で論理的依存ツリーを見てみみるとdeduped(重複排除)されたパッケージがあることが示されている。

```shell
$ npm ls strip-ansi
cows@1.0.0 /Users/userName/Projects/Cows
└─┬ cowsay@1.5.0
  ├─┬ string-width@2.1.1
  │ └── strip-ansi@4.0.0
  └─┬ yargs@15.4.1
    ├─┬ cliui@6.0.0
    │ ├─┬ string-width@4.2.3
    │ │ └── strip-ansi@6.0.1 deduped
    │ ├── strip-ansi@6.0.1
    │ └─┬ wrap-ansi@6.2.0
    │   ├─┬ string-width@4.2.3
    │   │ └── strip-ansi@6.0.1 deduped
    │   └── strip-ansi@6.0.1
    └─┬ string-width@4.2.3
      └── strip-ansi@6.0.1
```

パッケージの巻き上げ(hoisting)によって自動的にstirp-ansiパッケージは親の階層になるべく上げて配置されるようになっているためどこかで必要とされるパッケージから共通化されていることになる。

再び`npm explain`で依存元を見てみると最初のブロックで`strip-ansi@6.0.1`が2つのパッケージ(2つの場所)から必要とされていることが分かる。

```shell
$ npm explain strip-ansi
strip-ansi@6.0.1
node_modules/cliui/node_modules/strip-ansi
  strip-ansi@"^6.0.0" from cliui@6.0.0
  node_modules/cliui
    cliui@"^6.0.0" from yargs@15.4.1
    node_modules/yargs
      yargs@"15.4.1" from cowsay@1.5.0
      node_modules/cowsay
        cowsay@"^1.5.0" from the root project
  strip-ansi@"^6.0.1" from string-width@4.2.3
  node_modules/cliui/node_modules/string-width
    string-width@"^4.2.0" from cliui@6.0.0
    node_modules/cliui
      cliui@"^6.0.0" from yargs@15.4.1
      node_modules/yargs
        yargs@"15.4.1" from cowsay@1.5.0
        node_modules/cowsay
          cowsay@"^1.5.0" from the root project
		  
...省略
```

`cliui@6.0.0`と`string-width@4.2.3`の2つから必要とされている。論理的依存ツリーの以下の部分が対応している。ちょうどバージョン条件を両方満たす6.0.1で共通化し片方をdeduped(重複排除)してサイズをへらすことに成功している。

```shell
    ├─┬ cliui@6.0.0
    │ ├─┬ string-width@4.2.3
    │ │ └── strip-ansi@6.0.1 deduped
    │ ├── strip-ansi@6.0.1
```

しかし`cliui@6.0.0`は`wrap-ansi@6.2.0`というパッケージを依存としてもっており、そのパッケージの依存先に`strip-ansi@6.0.1`が2つあることが論理的依存ツリーからみてとれる。

これはどのように説明できるか。

wrap-ansiは依存元がcliuiだけなので重複が起こらず巻き上げがルートまで起きている。つまりFlatに配置されている。そのFlatに配置されたwrap-ansiパッケージのフォルダ内で2つの`strip-ansi@6.0.1`は共通化され、片方はdedupeされている。`npm explain strip-ansi`の続きを見てみると以下のようにwrap-ansinフォルダのnode_modulesに配置されたstirp-ansiパッケージによって2つの依存が共通化れていることがわかる。

```shell
strip-ansi@6.0.1
node_modules/wrap-ansi/node_modules/strip-ansi
  strip-ansi@"^6.0.0" from wrap-ansi@6.2.0
  node_modules/wrap-ansi
    wrap-ansi@"^6.2.0" from cliui@6.0.0
    node_modules/cliui
      cliui@"^6.0.0" from yargs@15.4.1
      node_modules/yargs
        yargs@"15.4.1" from cowsay@1.5.0
        node_modules/cowsay
          cowsay@"^1.5.0" from the root project
  strip-ansi@"^6.0.1" from string-width@4.2.3
  node_modules/wrap-ansi/node_modules/string-width
    string-width@"^4.1.0" from wrap-ansi@6.2.0
    node_modules/wrap-ansi
      wrap-ansi@"^6.2.0" from cliui@6.0.0
      node_modules/cliui
        cliui@"^6.0.0" from yargs@15.4.1
        node_modules/yargs
          yargs@"15.4.1" from cowsay@1.5.0
          node_modules/cowsay
            cowsay@"^1.5.0" from the root project
```

このように実際のnode_modulesの配置がどのようになるかというのは複雑に配置が行われるので予測するのが困難になることがだいたいであるらしい。依存がさらに多くなったプロジェクトでは何十ものパッケージの依存解決の末に重複排除や巻き上げがいくつもおこるからだ。

さらに、そもそもインストールの順番によってどのパッケージのバージョンがFlatに配置され、逆に重複排除されたりするかというのがパッケージのインストール順番によって変わってきてしまうという事実が問題をややこしくしている。npmに新しくパッケージを追加したときと最初から`package.json`にdepednenciesとして書いた状態で`npm install`するのではインストール順番が異なってしまうためのnode_modulesの構造(フォルダツリー)がまったく異なった物となってしまう可能性がある。

このため`package-lock.json`ファイルはすべてのインストールされるパッケージの正確なバージョンと相対パスによるパッケージのロケーションを記録しており、そこから正確なnode_modulesツリーを再現することができるようになっているとのことだ。これを『決定論的』という。`package.json`しかない状態ではどんなフォルダツリーになるのか予測できないためこの場合には『非決定論的』である状態と言える。

# DAG to Tree

なぜこれほどめんどくさいことが起きているかというと、結局のところ依存関係というのものがgraph(DAG: directed accclic graph)であったのに、それをtree構造に無理やり変換してdisk上のファイルシステム上に実現しようとしているからだ。

Primaryが中心となってできるDAGをPrimayを階層のルートとしたTreeにすると閉路や重複した部分において依存を一度切り離す必要がでてくる。切り離した結果として重複する。もう一度ドキュメントを見てみると

>パッケージマネジメントについて議論される際にdependency tree(依存ツリー)という言葉に言及したくなるが、実質的にdependenciesの間での関係性は厳密にはツリー(tree)ではなくグラフ(graph)である。サイクル(閉路)や重複する関係性を持つため、単一のノードがシステムの中で複数の役割を果たすことができる。
>\- [npm Blog Archive: npm v7 Series - Arborist Deep Dive](https://blog.npmjs.org/post/618653678433435649/npm-v7-series-arborist-deep-dive.html)より

｢単一のノードがシステム内で複数の役割を果たす｣ためこれがパッケージの重複の原因となる。hoistingの結果dedupeすべきパッケージがこの重複したノードであり、また、バージョン条件が衝突した結果として複数バージョンのインストールおよびhoistingの衝突が引き起こされたりしてnpmドッペルゲンガーが出現してくる。

一連のDependency Graph→Logical Dependency Tree→node_modules Tree(Folder tree)への流れを図にまとめてみると以下のようになる。

![](/images/npm-dependencies/dagToTree.jpg)

cowsayのインストールにおけるstrip-ansiパッケージで見てみると

↓これが(Dependency Graph)
![](/images/npm-dependencies/img_npmdependencies_5_stripansi.jpg)

↓こうで(Logical Dependency Tree)
![](/images/npm-dependencies/img_npmdependencies_6_ls.jpg)

↓最終的にこうなる(Folder Tree)
```shell
.
└── node_modules
    ├── cliui@6.0.0
    │   └── node_modules
    │       └── strip-ansi@6.0.1
    ├── strip-ansi@4.0.0
    ├── wrap-ansi@6.2.0
    │   └── node_modules
    │       └── strip-ansi@6.0.1
    └── yargs@15.4.1
        └── node_modules
            └── strip-ansi@6.0.1
```

node_modulesフォルダルートに配置されている`strip-ansi@4.0.0`のせいでこれ以上巻き上げができない3つの`strip-ansi@6.0.1`がnpmドッペルゲンガー。

# モジュール検索アルゴリズム
そして、なぜこれほどまでにパッケージの配置が問題になるのかという前提に立ち返ると

>npm is the package manager for the Node JavaScript platform. **It puts modules in place so that node can find them, and manages dependency conflicts intelligently**.
>\- [npm | npm Docs](https://docs.npmjs.com/cli/v7/commands/npm)より引用

npmがモジュール配置を行うのはnodeつまりNode.jsのプログラムがパッケージを見つけることができるようにするためだ。

Nodeのモジュール検索アルゴリズムはrequireしているファイルの存在するディレクトリのnode_modulesフォルダ配下のパッケージからルートディレクトリまで駆け上がって必要なモジュールを発見するまで探す。

探索は例えば、`/home/ry/projects/foo.js`にあるファイル`foo.js`が`require('bar')`を呼んだときNode.jsは次のロケーションを順番に探す。

```shell
.
├── home
│   ├── node_modules
│   │   └── bar # ←(3)
│   └── ry
│       ├── node_modules
│       │   └── bar # ←(2)
│       └── projects
│           ├── foo.js # → require('bar.js')
│           └── node_modules
│               └── bar # ←(1)探索スタート
└── node_modules
    └── bar # ←(4)探索終了
```

これによって、`npm install`によるパッケージインストールはこの検索アルゴリズムによってパッケージを効率よく、そしてなるべく依存しているモジュールを共有して見つけることができるようにnode_modulesフォルダに正しく配置することが大きな役割である。

例えば、cowsayのインストールにおいて、`strip-ansi@6.0.1`は2つのパッケージからrequireされている。

```shell
$ npm explain strip-ansi
strip-ansi@6.0.1
node_modules/cliui/node_modules/strip-ansi
  strip-ansi@"^6.0.0" from cliui@6.0.0
  node_modules/cliui
    cliui@"^6.0.0" from yargs@15.4.1
    node_modules/yargs
      yargs@"15.4.1" from cowsay@1.5.0
      node_modules/cowsay
        cowsay@"^1.5.0" from the root project
  strip-ansi@"^6.0.1" from string-width@4.2.3
  node_modules/cliui/node_modules/string-width
    string-width@"^4.2.0" from cliui@6.0.0
    node_modules/cliui
      cliui@"^6.0.0" from yargs@15.4.1
      node_modules/yargs
        yargs@"15.4.1" from cowsay@1.5.0
        node_modules/cowsay
          cowsay@"^1.5.0" from the root project
	  
...(省略)
```

`stirp-ansi@6.0.1`パッケージを必要とするそれぞれのパッケージの`index.js`では`require('strip-ansi')`がコールされているためNodeは上で説明したようにモジュールの探索を行う。

```shell:string-width@4.2.3のrequire
.
└── node_modules
    └── cliui@6.0.0
        ├── index.js
        └── node_modules
            ├── string-width@4.2.3
            │   └── index.js # ←require('strip-ansi')
            └── strip-ansi@6.0.1 # ←(2)探索終了
```

```shell:cliui@6.0.0のrequire
.
└── node_modules
    └── cliui@6.0.0
        ├── index.js # ←reqire('strip-ansi')
        └── node_modules
            ├── string-width@4.2.3
            │   └── index.js
            └── strip-ansi@6.0.1 # ←(1)探索開始かつ探索終了
```

本来ならば同一パッケージが2個インストールされてしまうところ片方はdedupe(重複排除)されている。そしてこのようにNodeのモジュール発見アルゴリズムからそれぞれが依存する`strip-ansi@6.0.1`を共通のモジュールとして発見することができるためモジュールの依存を共有することができていることがわかる。パッケージのインストール時自動重複排除の効果と言える。

参考
https://nodejs.org/api/modules.html#modules_loading_from_node_modules_folders

https://blog.tai2.net/node-quiz-about-npm-install.html

# Arborist(樹林管理士)について

実際にフォルダツリーの構築を行っているのはnpm v7の時点では[Arborist](https://github.com/npm/arborist)という大規模にリファクタリングされたプログラムが中核を担っているみたいである。さすがにArboristのソースコードまで立ち入ることはむずかしいが、概要としてはAroboristはactualTree, virtualTree, idealTreeという3つのツリーを管理し、`pakcage-lock.json`などのメタデータから読み込まれたvirtualTreeから実際のnode_modulesツリーであるactualTreeのデータをidealTreeとの差分を取りながら変形していくとのことである。

参考: 
https://blog.npmjs.org/post/618653678433435649/npm-v7-series-arborist-deep-dive.html

cowsayの`npm install`のログを再び見てもらうとどういうプロセスで実行しているかなんとなくわかる。

```log:インストールログ(かなり省略してある)
npm timing config:~~

npm timing npm:load:~~
npm timing arborist:ctor ~~
npm idealTree:init Completed

npm http fetch GET 200 https://registry.npmjs.org/cowsay 959ms (cache revalidated)
npm http fetch GET 200 https://registry.npmjs.org/string-width 84ms (cache revalidated)
npm http fetch GET 200 https://registry.npmjs.org/yargs 228ms (cache revalidated)
npm http fetch GET 200 https://registry.npmjs.org/strip-final-newline 279ms (cache revalidated)
npm http fetch GET 200 https://registry.npmjs.org/get-stdin 281ms (cache revalidated)
npm timing idealTree:#root Completed in 1250ms

npm http fetch GET 200 https://registry.npmjs.org/require-directory 83ms (cache revalidated)
npm http fetch GET 200 https://registry.npmjs.org/strip-ansi 89ms (cache revalidated)
npm http fetch GET 200 https://registry.npmjs.org/cliui 100ms (cache revalidated)
~~

npm timing idealTree:node_modules/cowsay Completed
npm timing idealTree:node_modules/~~
npm timing idealTree:node_modules/~~

npm timing reify:loadTrees Completed
npm timing reify:diffTrees Completed
npm timing reify:retireShallow Completed 

npm timing reifyNode:node_modules/~~
npm timing reifyNode:node_modules/~~
npm timing reifyNode:node_modules/~~
~~
npm timing reifyNode:node_modules/cowsay Completed
npm timing unpack Completed

npm timing build:queue Completed

npm timing build:link:node_modules/cowsay Completed in 2ms
npm timing build:link Completed
npm timing build:deps Completed
npm timing build Completed
npm timing reify:build Completed
npm timing reify:trash Completed
npm timing reify:save Completed

~~
```

idealTreeのところで依存グラフを満たす仮想のフォルダツリー構築を行い、reifyとはreification(具象化)のことでidealTreeの具象化(つまりnode_modulesフォルダへの配置)を行っているものと考えられる。

# 今後の発展

## npm v7のビジョン
ちなみにv7の目指しているビジョンは以下の4つらしい。

-   Reduce noise that is not actionable
-   Manage your packages for you
-   Strict separation of concerns
-   Be as fast as possible while behaving correctly

特に3つ目の項目では、｢npmCLIはユーザーインターフェースのレイヤーになりつつあり、すべての複雑なツリー管理とレジストリのレジストリとのインタラクションはarborist、pacote、そして様々なnpmcli libnpmモジュールに移行した｣との旨が書いてあった。npmCLIの機能はかなり分割されており、これ以上立ち入るにはarboristだけでなくそれぞれの分割されたパッケージについてしらべていく必要がありそうだ。

https://blog.npmjs.org/post/617484925547986944/npm-v7-series-introduction

## npm v8での変更予定

さらに先日リリースしたnpm v8では[Kat Marchán](https://thenewstack.io/open-source-leaders-kat-marchan/)が2019年に発表した概念論証である『Tink』をモデルにしたシンボリックリンクまたは仮想ファイルシステムアプローチを検討する予定とのこと。pnpmが採用しているようなnode_modulesの構造に近づくかもしれない。ということでせっかくnpm v7のnode_modulesについて調べたがフォルダの構造が変わってしまうかもしれない笑

https://blog.npmjs.org/post/621733939456933888/npm-v7-series-why-keep-package-lockjson.html

Tinkについて詳しくは以下の動画で見ることができる。
https://www.youtube.com/watch?v=SHIci8-6_gs

# 参考文献

- [JSエコシステムぶらり探訪(3): npmとyarnとnode_modules - Qiita](https://qiita.com/qnighy/items/677bf64063c666b21034)
- [The 5 dimensions of an npm dependency | Snyk](https://snyk.io/blog/whats-an-npm-dependency/)
- [Node.jsクイズ第58問 ./node_modules直下にはどのパッケージが入る? | blog.tai2.net](https://blog.tai2.net/node-quiz-about-npm-install.html)
- [Understanding dependency management with Node Modules | by Jason Lai | Level Up Coding](https://levelup.gitconnected.com/understanding-dependency-management-with-node-modules-1c47bcdee98b)
- [Understanding npm dependency resolution | by Bhammarker Rahul | Learn With Rahul | Medium](https://medium.com/learnwithrahul/understanding-npm-dependency-resolution-84a24180901b)
- [Understanding npm dependency resolution part 1 | by Rishav Thakur | Groww Engineering](https://tech.groww.in/introduction-to-package-dependency-resolution-in-npm-ad1b374fc13a)
- [npm Blog Archive: npm v7 Series - Why Keep `package-lock.json`?](https://blog.npmjs.org/post/621733939456933888/npm-v7-series-why-keep-package-lockjson.html)

