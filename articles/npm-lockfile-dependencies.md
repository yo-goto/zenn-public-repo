---
title: "npmの依存関係とv7のロックファイルについて調べてみた"
published: true
cssclass: zenn
emoji: "🌳"
type: "tech"
topics: [npm, node, json]
date: 2021-10-19
modified: 2023-02-05
url: "https://zenn.dev/estra/articles/npm-lockfile-dependencies"
AutoNoteMover: disable
tags: " #node/npm #type/zenn  "
aliases: 記事_npmの依存関係とv7のロックファイル
---

# はじめに

以下の URL にて npm や TypeScript を使ったプラグイン開発に関する記事を書いた際に npm に関するわからなかったこと、主に「npm の依存関係と `package-lock.json`」について調べてみました。

https://zenn.dev/estra/articles/obsidian-plugin-dev_1

:::message
記載している情報は完全に正確、網羅しているわけではなく、自分のアウトプットとして作成している点をご了承ください。
:::

## そもそも

Obsidian のプラグイン開発において Github のリポジトリから clone したリポジトリで手動でビルドする必要があり、`npm i` で必要なパッケージが `node_modules` というディレクトリにインストールされるということは知っていたが、`node_moduels` ディレクトリになぜこんなにも多くのパッケージがインストールされているのか疑問だった。`package.json` ファイルの dependencies の項目に記載されているパッケージが動作に必要で devDependencies は開発に必要なパッケージであるということはわかったが、その 2 つのセクションに書いていないパッケージが `node_modules` に入っており意味がわからないというのが調査の発端。

また npm もかなり漠然と使ってたため、良い機会なので概念や目的も色々と調べてみた。

# 要点

『依存関係 (dependencies)』とは
プログラム内で外部ライブラリを使っていて、それが無いと動かないという状態を依存関係という。npm を導入する理由で最も多いのはこの依存関係を管理するためとのこと。

- プロジェクトルートにある `pacakge.json` の dependencies に記載されているパッケージがインストールされる (場合によっては devDependencies も)。
- インストールされる dependencies に記載されているパッケージが持つ各 `package.json` ファイルの dependencies に記載されているパッケージが更にインストールされる。そして、すべての依存しているパッケージがインストールされる。
- `package-lock.json` にはインストールされたすべてのパッケージの正確なバージョン情報が記載されており、このファイルを使えば異なるマシンで異なる時間にパッケージのインストールをしても正確に同じ環境を作ることができる。
- npm にはバージョンがあり、使用しているバージョンの `package-lock.json` 構造について知る必要があり、npm v7 では `package` オブジェクトが読み取られ、`dependencies` オブジェクトは無視されるため気にかける必要はない。

# npm の依存関係

## npm i でインストールされるパッケージ

`npm i` でインストールするといっても色々とパターンがあるのでそれぞれ試してみた。

- パターン 1: 最初から package.json がある状態で `npm i`
	- 自分で `npm init` せずに Github のリポジトリからクローンしたものなどは `package.json` が最初からある状態で `npm i` コマンドでインストールすると、dependenceis に書かれているパッケージ以外のパッケージが node_modules にインストールされた。
- パターン 2: 自分で package.json を用意してから `npm i`
	- `npm init` でプロジェクトを初期化して `package.json` を用意、dependencies に 1 つだけパッケージを記載して `npm i` を行ってみたところ、まったく知らないパッケージが `node_modules` にインストールされた。
- パターン 3: package.json を用意してから `npm i パッケージ名`
	- パターン 2 と同様に `node_modules` ディレクトリに知らないパッケージがインストールされた。

どのパターンでもプロジェクトの `package.json` に記載されていない多くのパッケージがインストールされていた。ググって記事を漁ってみてもあまりよく分からなかったところ、以下の YouTube の動画をみたら一発で理解できた (npm の全貌についてもかなりわかり易く解説されていた)。

@[youtube](P3aKRdUyr0s)

6:00~の「Dealing with npm package dependencies」のところを参照。

つまり、インストールしたいパッケージが特定の 1 つであってもそれを動かすためには依存しているパッケージもろもろすべてが必要であるためプロジェクトルートの `package.json` に記載されていないそれらを `node_modules` にインストールしていたということだった。

たとえば動画のように `npm install chalk` で chalk というパッケージ 1 つをインストールしてみると `node_moduels` には chalk 以外の 5 つのパッケージがインストールされている。

![](/images/npm-dependencies/img_npmdependencies_0.jpg)

各パッケージのディレクトリを覗いてみるとそれぞれのパッケージごとに `package.json` ファイルが存在し、それには dependencies の項目に別のパッケージが記載されていた。

```json:chalkのpackage.jsonのdependencies
"dependencies": {
	"ansi-styles": "^4.1.0",
	"supports-color": "^7.1.0"
},
```

chalk というパッケージはこの dependencies に記載されている ansi-styles、supports-color というパッケージが動かすために必要ということだ。そのため `npm i chalk` では chalk を実際に使うために ansi-styles、supports-color というパッケージを追加でインストールされている。

しかし、まだ３つも知らないパッケージが `node_modues` には存在している。理由は簡単で chalk を動かすために必要な 2 つのパッケージにもそれぞれ動かすために必要なパッケージがあったということだ。2 つのパッケージのディレクトリも覗いてみると chalk と同様にそれぞれ `pacakge.json` ファイルを持ち、dependencies を持っていることがわかった。

```json:ansi-sytlesのpackage.jsonのdependencies
"dependencies": {
	"color-convert": "^2.0.1"
},
```

```json:supports-colorのpackage.jsonのdependencies
"dependencies": {
	"has-flag": "^4.0.0"
},
```

よって、これら 2 つのパッケージを動かすためには更に color-convert と has-flag というパッケージをインストールする必要があるといことがわかる。そしてそれら 2 つの追加パッケージの package.json もみてみると dependencies の項目がないので追加でインストールする必要のあるパッケージがないことがわかる。

従って、すべてのパッケージの package.json の dependencies に記載されているパッケージがインストールされているのでこれでようやく chalk が動かすことができる。

![](/images/npm-dependencies/img_npmdependencies_1.jpg)

プロジェクトフォルダをみてみると上の画像のようにあるパッケージは単独で `package.json` ファイルを各々持ち、dependencies に必要なパッケージがかかれている場合にはそれぞれの依存を満たすために必要なパッケージをすべてインストールする必要がある。

## 依存関係の把握に使えるコマンド

ここで依存関係を把握するために便利な npm コマンドを知ることができたので以下の `cowsay` というパッケージを単体でインストールしてみたログを使って紹介する。

```shell
# npmのプロジェクト初期化
$ npm init -y
# package.jsonが作成される

# cowsayというパッケージをインストール
$ npm i cowsay
# 作成されたpackage.jsonをみてみる
$ cat package.json
{
  "name": "nodetest2",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "cowsay": "^1.5.0"
  }
}

# dependenciesにかかれている依存のみ表示
$ npm ls
nodetest2@1.0.0
└── cowsay@1.5.0

$ npm ls -a
# すべての依存関係をツリー表示
nodetest2@1.0.0
└─┬ cowsay@1.5.0
  ├── get-stdin@8.0.0
  ├─┬ string-width@2.1.1
  │ ├── is-fullwidth-code-point@2.0.0
  │ └─┬ strip-ansi@4.0.0
  │   └── ansi-regex@3.0.0
  ├── strip-final-newline@2.0.0
  └─┬ yargs@15.4.1
    ├─┬ cliui@6.0.0
    │ ├─┬ string-width@4.2.3
    │ │ ├── emoji-regex@8.0.0 deduped
    │ │ ├── is-fullwidth-code-point@3.0.0
    │ │ └── strip-ansi@6.0.1 deduped
    │ ├─┬ strip-ansi@6.0.1
    │ │ └── ansi-regex@5.0.1
    │ └─┬ wrap-ansi@6.2.0
    │   ├─┬ ansi-styles@4.3.0
    │   │ └─┬ color-convert@2.0.1
    │   │   └── color-name@1.1.4
    │   ├─┬ string-width@4.2.3
    │   │ ├── emoji-regex@8.0.0 deduped
    │   │ ├── is-fullwidth-code-point@3.0.0
    │   │ └── strip-ansi@6.0.1 deduped
    │   └─┬ strip-ansi@6.0.1
    │     └── ansi-regex@5.0.1
    ├── decamelize@1.2.0
    ├─┬ find-up@4.1.0
    │ ├─┬ locate-path@5.0.0
    │ │ └─┬ p-locate@4.1.0
    │ │   └─┬ p-limit@2.3.0
    │ │     └── p-try@2.2.0
    │ └── path-exists@4.0.0
    ├── get-caller-file@2.0.5
    ├── require-directory@2.1.1
    ├── require-main-filename@2.0.0
    ├── set-blocking@2.0.0
    ├─┬ string-width@4.2.3
    │ ├── emoji-regex@8.0.0
    │ ├── is-fullwidth-code-point@3.0.0
    │ └─┬ strip-ansi@6.0.1
    │   └── ansi-regex@5.0.1
    ├── which-module@2.0.0
    ├── y18n@4.0.3
    └─┬ yargs-parser@18.1.3
      ├── camelcase@5.3.1
      └── decamelize@1.2.0 deduped
# deduped は重複排除されたパッケージ

# explainコマンドでbottom up view(このパッケージがなぜツリーに含まれるのかを構造的に表示)
$ npm explain cowsay
cowsay@1.5.0
node_modules/cowsay
  cowsay@"^1.5.0" from the root project
$ npm explain get-stdin
get-stdin@8.0.0
node_modules/get-stdin
  get-stdin@"8.0.0" from cowsay@1.5.0
  node_modules/cowsay
    cowsay@"^1.5.0" from the root project

# strip-anisはdeduped(重複)した依存なので色々なパッケージから必要とされていることがわかる
$ npm expalin strip-ansi
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

使用コマンド
- [npm ls](https://docs.npmjs.com/cli/v7/commands/npm-ls)
	- 依存関係をツリー構造で表示してくれるコマンド。オプション `-a` をつけることで完全なツリー構造表示になる。
	- 出力が多すぎるため途中のバージョン変更において `-a` オプションをつけないとすべてを表示しない仕様に変更された。
	- ~~deduped は重複しているパッケージを示す~~
	- deduped は [重複排除](https://www.fujitsu.com/jp/products/computing/storage/lib-f/tech/de-duplication/) されたパッケージを示す。
- [npm explain](https://docs.npmjs.com/cli/v7/commands/npm-explain)
	- `npm ls -a` ではルートという上層から見た構造表示であったが、このコマンドではパッケージを指定することでそのパッケージが逆になぜツリーに含まれているか、どのパッケージから必要とされているかを階層表示してくれる。
	- 重複でインストールされているバージョンの異なるパッケージについてそれぞれのバージョンがどのパッケージの dependencies に記載されているかをルートまでさかのぼってそれぞれを表示する。

ちなみに cowsay パッケージの `package.json` ファイルは以下。

```json:cowsayのpackage.json(一部)
"dependencies": {
  "get-stdin": "8.0.0",
  "string-width": "~2.1.1",
  "strip-final-newline": "2.0.0",
  "yargs": "15.4.1"
},
"devDependencies": {
  "@rollup/plugin-commonjs": "15.0.0",
  "@rollup/plugin-node-resolve": "9.0.0",
  "execa": "5.0.0",
  "rollup": "2.26.5",
  "rollup-plugin-string": "3.0.0",
  "tap-dot": "2.0.0",
  "tape": "5.0.1"
},
```

dependencies に記載されているパッケージ (get-stdin, string-widht, strip-final-newline, yargs の 4 つ) は cowsay が動作する上で必要なのでこのパッケージもインストールされる。しかし、この 4 つのパッケージにもそれぞれ `package.json` ファイルがあり、それぞれ dependencies を持つ。つまりそれらの dependencies も動作する上で必要なのでインストールされる。更にインストールされたパッケージも更に dependencies を持つ、動作に必要なのでインストールされる....ということを繰り返してすべての依存関係のあるパッケージがインストールされた結果として node_modules ディレクトリが大きくなり、`npm ls -a` で表示されるような依存ツリーが出来上がる。

次のようなパッケージの依存関係をグラフ表示てくれるサービスもあるので試してみた。

https://npm.anvaka.com/#/

![](/images/npm-dependencies/img_npmdependencies_2.jpg)

chalk の依存関係は比較的少ないのでかなりわかりやすい。

![](/images/npm-dependencies/img_npmdependencies_3.jpg)

一方、cowsay は 33 個もの node が存在しており依存がより複雑になっていることがわかる。

# ロックファイル

`package.json` ファイルに記載された dependencies や `npm install パッケージ名` でインストールしたパッケージの依存する他パッケージがインストールされるのはわかったが、`node_modules` に入っている全パッケージの情報はどうやって確認するのかという疑問が立ち上がる。例えば Github のプロジェクトで clone してから実際に `npm i` してみるまでインストールされるものがわからないということは無いはずだ。どんなサイズになるかもわからないし、セキュリティ的にもよろしくないのではないか。

もちろんその方法は存在しており、すべてのインストールされるべきパッケージを記載しており、依存関係をすべて把握しているのが `package-lock.json` ファイルということになる。

## ロックファイルの目的

ということで、npm v7 で使用される `package-lock.json` ファイルの目的について公式ドキュメントから調べてみた。

https://docs.npmjs.com/cli/v7/configuring-npm/package-lock-json

>`package-locl.json` ファイルは `node_modules` ツリーもしくは `package.json` ファイルを変更するすべての操作対して自動的に生成されるファイルであり、後のインストールによって同一の依存ツリーを生成できるように正確なツリーを記述する。
>
>このファイルはソースリポジトリへコミットするように意図されており、以下の様々な目的に役立つ。
>
> - 同一の依存関係を正確にインストールできることを保証するために依存ツリーを単一の表現として記述する
> -   `node_modules` についてディレクトリそのものをコミットする必要なく、推移的な状態を確認することを可能にする
> - ソースの読みやすい差分コントロールによってツリー変更の可読性を向上させる
> - 過去にインストールされたパッケージについて繰り返される冗長なメタデータ解決を省略することでインストールプロセスを最適化する
>   - npm v7 においてロックファイルはパッケージツリーの完全な全体図を得るための十分な情報を含んでおり、`package.json` ファイルを読む必要を減らした結果、パフォーマンスが大幅に向上する

:::message
ロックファイルの大きな目的は `package-lock.json` ファイルにすべての推移的な依存関係を含ませることで、異なる時間に異なるマシンを使ったとしてもまったく同一の依存関係を再現したインストールを保証してくれることにある。
参考: [Lockfile とは?このファイルのコミットって必要？ [Beginner's Series to Node.js 9/26]](https://youtu.be/7XQU0Obs_wk)
:::

## pacakge.json だけではだめな理由

そもそもの package.json ではなぜ正確な依存関係をインストールできないのか。

それはセマンティックバージョニング ([Semantice Versioning](https://semver.org/lang/ja/): semver) と呼ばれるバージョン管理番号の指定方法と関連がある。

:::message
セマンティックバージョニング (Semantic Versinoing) とは、最もポピュラーなプロジェクトの固有バージョン方法であり、この方法を使うことでソフトウェアの変更追跡を容易ににしバージョンをクリーンでシンプルに保つことができる。
参考: [Semantic Versioning | Developer Experience Knowledge Base](https://developerexperience.io/practices/semantic-versioning)
:::

セマンティックバージョニングでは `x.y.z` という方法でバージョン番号を付ける (x,y,z は整数)。それぞれの数字には意味があり、次のように使う。

- x: **メジャーバージョン**  
    API 変更等、非後方互換
- y: **マイナーバージョン**  
    新規機能実装、フレームワークや機能向上、後方互換
- z: **パッチバージョン**  
    バグフィクス、メンテンスリリース、後方互換

このセマンティックバージョニングを使ってソフトウェアのバージョン管理を行うのが一般的であり、`package.json` ではこの番号を dependencies や devDepenendencies にパッケージ名と共に記述する。しかし、このバージョン指定の方法によって正確な依存関係をインストールできなくなっている。例えば、先程の chalk のパッケージ内に含まれる `package.json` の dependenceis では以下のようにセマンティックバージョニングのバージョン番号の頭にキャレット `^` と呼ばれる記号がついている。

```json:chalkのpackage.jsonのdependencies
"dependencies": {
	"ansi-styles": "^4.1.0",
	"supports-color": "^7.1.0"
},
```

このキャレットがついたバージョン番号は「一番左側にある、ゼロではないバージョンは変えず、それ以下のバージョンが有る場合には許容してインストールする」ことを意味している。つまり、ansi-styles というパッケージについては「`4.1.0以上4.2.0未満` の最新バージョンをインストールする」というバージョンの範囲を指定していることになる。したがって `npm i` ではこの範囲での ansi-styles の最新バージョンをインストールする。

ある時間にインストールした ansi-sytles のバージョンが `4.1.1` であったとしても、ソフトウェアの開発は時間と共に進み ansi-styles というパッケージのバージョンは進むので、別の時間インストールすると `4.1.7` というバージョンがインストールされることがある。このバージョンの範囲指定での最新を入れるので他の人が同じ `package.json` ファイルを使ってインストールしても異なるバージョンのパッケージ群が入ってしまう可能性が大きい。より複雑なプロジェクトになれば依存先はかなり多いため各パッケージのバージョン更新が短い間にいくつもあるかもしれない。

といことで、実際に開発の環境でインストールしたパッケージの正確なバージョン番号というものが必要になってくるわけだ。

参考
https://zenn.dev/luvmini511/articles/56bf98f0d398a5

そもそも、`dependencies` は **バージョン範囲** を指定するものであると公式ドキュメントに書いてある。これを知らなかったため、なぜ `npm i` で異なるバージョンを入れてしまうのかが分からなかった。といことで、公式ドキュメントを参考にして `dependencies` と `devDependencies` についてもそれぞれ明確にしておくと以下のような違いがある。

- `dependencies`
	- dependencies(依存関係) はパッケージ名を **バージョン範囲** にマップするシンプルなオブジェクトで指定される。バージョン範囲は 1 つ以上のスペースで区切られる記述子を持つ文字列で記述できる。dependencies は tarball または Git URL でも指定できる。
	- チルダやキャレットによるバージョン範囲の記法があるため紛らわしい。
	- [The semver parser for node](https://github.com/npm/node-semver#versions) に範囲の記述方法のすべてが記載されている。
- `devDependencies`
	- そのパッケージを開発･ビルドするための外部ツールやフレームワークなど、ただプログラムを実行して使いたいユーザーには必要ないパッケージ。
	- この項目にリストされているパッケージはこれが記載されている `pakcage.json` ファイルがルートにある状態で `npm install` まはた `npm link` コマンドを実行したとき。

https://docs.npmjs.com/cli/v7/configuring-npm/package-json#main

## package-lock.json の構造

主要なフィールドについて同様に公式ドキュメントから抜粋して調べてみた。npm v5/v6 からこのロックファイルの構造は変わったようだ。ドキュメントを読んでいると以前のバージョンでは `package.json` と `package-lock.json` の両方を読んでいたようだが、いまや `pakage-lock.json` が完全な依存ツリーの情報を持つようになったのでパフォーマンスが向上したとのこと。

- `name`: パッケージの名前
	`package.json` の name と一致する
- `version`: `package-lock.json` のバージョン
	`package.json` の vserision と一致する
- `lockfileVersion`: 整数のバージョン番号
	- バージョン番号がない: npm v5 以前の古い shrinkwarp ファイルであることを示す
	- `1`: npm v5 または v6 で使用されているロックファイルであることを示す
	- `2`: npm v7 によって使用されているロックファイルであることを示し、v1 のロックファイルへの後方互換性を持っている
	- `3`: npm v7 によって使用されているロックファイルであることを示すが、後方互換性はない。`node_modules/.package-lock.json` の隠しロックファイルに使用され、npm v6 のサポートが終了しだい将来的な npm のバージョンで使用される可能性が高い。
- `requires`: 公式ドキュメントに記載なし (ブール値)
- `package`: パッケージの情報を含むオブジェクトへのパッケージロケーションをマッピングするオブジェクト
	- ルートプロジェクトは `""` というキーでリストされており、他のパッケージはルートプロジェクトフォルダからの相対パスで記載される。
	- `version`: 実際にインストールされたパッケージ内部の `package.json` に記載されているバージョンと一致する。
	- `resolved`: パッケージの実際のロケーション。
		- npm レジストリから取得されている場合には tarball への URL
		- Git 依存の場合にはコミットハッシュ付き git のフル URL
	- `integrity` : このロケーションに解凍された依存パッケージに対する SRI(サブリソース完全性: [Standard Subresource Integrity](https://w3c.github.io/webappsec/specs/subresourceintegrity/)) として使われる文字列 (sha512 または sha1 で暗号化されたハッシュ値)
	- `bin`, `license`, `engines`, `dependenceis`, `optionalDependnecies`: `package.json` に記載されているフィールドから転記。
- `dependencies`: `lockfileVersion: 1` を使用している npm のバージョンをサポートするレガシーデータ。npm v7 では `pacakge` セクションが存在すればこのセクションを完全に無視するが、npm v6 と v7 のスイッチングをサポートするためにデータを保持している。なので基本的に v7 を使っていれば中身は無視してよい。

```json:package-lock.jsonファイルの中身(chalk単体インストール)
{
  "name": "ProjectName",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "requires": true,
  "package": {
      "": { // こいつがルートプロジェクト
        "name": "ProjectName",
        "version": "1.0.0",
        "license": "ISC",
        "dependencies": {
          "chalk": "^4.1.2"
        }
      },
      // 以下インストールしたパッケージをアルファベット順ですべて記載
      "node_modules/ansi-styles": {
        "version": "4.3.0", // これが実際にインストールされたパッケージのバージョン
        "resolved": "https://registry.npmjs.org/ansi-styles/-/ansi-styles-4.3.0.tgz",
        "integrity": "sha512-zbB9rCJAT1rbjiVDb2hqKFHNYLxgtk8NURxZ3IZwD3F6NtxbXZQCnnSi1Lkx+IDohdPlFp222wVALIheZJQSEg==",
        "dependencies": {
          "color-convert": "^2.0.1"
        },
        "engines": {
          "node": ">=8"
        },
        "funding": {
          "url": "https://github.com/chalk/ansi-styles?sponsor=1"
        }
      },
      // 中略
      "node_modules/supports-color": {
        "version": "7.2.0",
        "resolved": "https://registry.npmjs.org/supports-color/-/supports-color-7.2.0.tgz",
        "integrity": "sha512-qpCAvRl9stuOHveKsn7HncJRvv501qIacKzQlO/+Lwxc9+0q2wLyv4Dfvt80/DPn2pqOBsJdDiogXGR9+OvwRw==",
        "dependencies": {
          "has-flag": "^4.0.0"
        },
        "engines": {
          "node": ">=8"
        }
      },
  "dependencies": {
  // npm v7ではこのセクションは基本的に無視するので省略
  // "package"と同じ様に依存ツリーについて記述されている
  }
}
```

## 隠しロックファイルについて

npm v7 においては、`node_modules/.package-lock.json` という隠しロックファイルが使用されており、node_moduldes フォルダに対して繰り返される処理を避けるようにしているとのこと。このファイルには依存ツリーに関する情報が含まれており、次の条件が満たされている場合には `node_modules` フォルダの階層全体を読み取る代わりに使用される。

この隠しロックファイルは古いバージョンの npm からは無視されるようになっているので現在の通常ロックファイルにあるような後方互換性は含まない。よって npm v5/v6 はこのファイルを無視する。現時点では `lockfileVersion` は 2 になっている。

```json:.package-lock.jsonファイルの中身
{
  "name": "ProjectName",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "package": {
    "node_modules/ansi-styles": {
      "version": "4.3.0",
      "resolved": "https://registry.npmjs.org/ansi-styles/-/ansi-styles-4.3.0.tgz",
      "integrity": "sha512-zbB9rCJAT1rbjiVDb2hqKFHNYLxgtk8NURxZ3IZwD3F6NtxbXZQCnnSi1Lkx+IDohdPlFp222wVALIheZJQSEg==",
      "dependencies": {
        "color-convert": "^2.0.1"
      },
      "engines": {
        "node": ">=8"
      },
      "funding": {
        "url": "https://github.com/chalk/ansi-styles?sponsor=1"
      }
    },
    // 中略
    "node_modules/supports-color": {
      "version": "7.2.0",
      "resolved": "https://registry.npmjs.org/supports-color/-/supports-color-7.2.0.tgz",
      "integrity": "sha512-qpCAvRl9stuOHveKsn7HncJRvv501qIacKzQlO/+Lwxc9+0q2wLyv4Dfvt80/DPn2pqOBsJdDiogXGR9+OvwRw==",
      "dependencies": {
        "has-flag": "^4.0.0"
      },
      "engines": {
        "node": ">=8"
      }
    }            
  }
}
```

`pakcage-lock.json` との違いは npm v5/v6 用のフィールドとしてあった "dependencies" がなくなっているのとルートプロジェクトを示す "" がなくなっている以外は同じ。

参考: 
https://nitayneeman.com/posts/catching-up-with-package-lockfile-changes-in-npm-v7/

# メモ

- npm について調べるには使用しているバージョンの npm についての情報が必要でありバージョンが変更されたら公式ドキュメントを再び読む必要がある。
- YouTube の英語版の動画にはかなり分かりやすくクオリティの高いものがあるのでドキュメントだけではなく動画も調べる価値がある。

# 追記

## npm v8 について

というかいつの間に npm CLI v8 がリリースされていました。メジャーバージョンが上がったので破壊的変更があったのではないでしょうか。ドキュメントはまだないみたいですね。追って確認したいと思います。

https://github.blog/changelog/2021-10-07-npm-cli-upgraded-to-version-8/

https://github.com/npm/rfcs/issues/445

## 勘違いしていたこと

今回の記事を書いた後に気づいたことや、勘違いしていたことが多々あったのでそれについて別の記事にまとめました。
https://zenn.dev/estra/articles/npm-about-dependencsies
