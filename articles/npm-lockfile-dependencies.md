---
title: "npmの依存関係とv7のロックファイルについて調べてみた"
emoji: "🌳"
type: "tech"
topics: [npm, node, json]
published: true
date: 2021-10-19
url: "https://zenn.dev/estra/articles/npm-lockfile&dependencies"
aliases: [npmの依存関係とv7のロックファイルについて調べてみた]
tags: " #node/npm #type/zenn  "
---

 
# はじめに
以下のURLにてnpmやTypeScriptを使ったプラグイン開発に関する記事を書いた際にnpmに関するわからなかったこと、主に｢依存関係とpackage-lock.json｣について調べてみました。

https://zenn.dev/estra/articles/obsidian-dev-plugin_1

:::message
記載している情報は完全に正確、網羅しているわけではなく、自分のアウトプットとしての記事として作成している点をご了承ください。
:::

# Overview

『依存関係(dependencies)』とは
プログラム内で外部ライブラリを使っていて、それが無いと動かないという状態を依存関係という。npmを導入する理由で最も多いのはこの依存関係を管理するためとのこと。

- プロジェクトルートにある`pacakge.json`のdependenciesに記載されているパッケージがインストールされる(場合によってはdevDependenciesも)。
- インストールされるdependenciesに記載されているパッケージが持つ各`package.json`ファイルのdependenciesに記載されているパッケージが更にインストールされる。そして、すべての依存しているパッケージがインストールされる。
- `package-lock.json`にはインストールされたすべてのパッケージの正確なバージョン情報が記載されており、このファイルを使えば異なるマシンで異なる時間にパッケージのインストールをしても正確に同じ環境を作ることができる。
- npmにはバージョンがあり、使用しているバージョンの`package-lock.json`構造について知る必要があり、npm v7では`package`オブジェクトであり、`dependencies`オブジェクトは無視されるため気にかける必要はない。


# npmの依存関係

## そもそも
Obsidianのプラグイン開発においてGithubのリポジトリからcloneしたリポジトリで手動でビルドする必要があり、`npm i`で必要なパッケージが`node_modules`というディレクトリにインストールされるということは知っていたが、`node_moduels`ディレクトリになぜこんなにも多くのパッケージがインストールされているのか疑問だった。`package.json`ファイルのdependenciesの項目に記載されているパッケージが動作に必要でdevDependenciesは開発に必要なパッケージであるということはわかったが、その2つのセクションに書いていないパッケージが`node_modules`に入っており意味がわからないというのが調査の発端。

npmというものを漠然と使ってたため、良い機会なので色々と調べてみた。

## npm iでインストールされるパッケージ
`npm i`でインストールするといっても色々とパターンがあるのでそれぞれ試してみた。

- パターン1: 最初からpackage.jsonがある状態で`npm i`
	- 自分で`npm init`せずにGithubのリポジトリからクローンしたものなどは`package.json`が最初からある状態で`npm i`コマンドでインストールすると、dependenceisに書かれているパッケージ以外のパッケージがnode_modulesにインストールされた。
- パターン2: 自分でpackage.jsonを用意してから`npm i`
	- `npm init`でプロジェクトを初期化して`package.json`を用意、dependenciesに一つだけパッケージを記載して`npm i`を行ってみたところ、まったく知らないパッケージが`node_modules`にインストールされた。
- パターン3: package.jsonを用意してから`npm i パッケージ名`
	- パターン2と同様に`node_modules`ディレクトリに知らないパッケージがインストールされた。

どのパターンでもプロジェクトの`package.json`に記載されていない多くのパッケージがインストールされていた。ググってて記事を漁って見てもあまり良くわからなかったところ、以下のYoutubeの動画をみたら一発で理解することができた。

@[P3aKRdUyr0s]

6:00~の｢Dealing with npm package dependencies｣のところを参照。

つまり、インストールしたいパッケージが特定の一つであってもそれを動かすためには依存しているパッケージもろもろすべてが必要であるためプロジェクトルートの`package.json`に記載されていないそれらを`node_modules`にインストールしていたということだった。

たとえば動画のように`npm install chalk`でchalkというパッケージ一つをインストールしてみると`node_moduels`にはchalk以外の5つのパッケージがインストールされている。

![](/images/npm-dependencies/img_npmdependencies_0.jpg)

各パッケージのディレクトリを覗いてみるとそれぞれのパッケージごとに`package.json`ファイルが存在し、それにはdependenciesの項目に別のパッケージが記載されていた。

```json:chalkのpackage.jsonのdependencies
"dependencies": {
	"ansi-styles": "^4.1.0",
	"supports-color": "^7.1.0"
},
```

chalkというパッケージはこのdependenciesに記載されているansi-styles、supports-colorというパッケージが動かすために必要ということだ。そのため`npm i chalk`ではchalkを実際に使うためにansi-styles、supports-colorというパッケージを追加でインストールされている。

しかし、まだ3つも知らないパッケージが`node_modues`には存在している。理由は簡単でchalkを動かすために必要な2つのパッケージにもそれぞれ動かすために必要なパッケージがあったということだ。2つのパッケージのディレクトリも覗いてみるとchalkと同様にそれぞれ`pacakge.json`ファイルを持ち、dependenciesを持っていることがわかった。

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

よって、これら2つのパッケージを動かすためには更にcolor-convertとhas-flagというパッケージをインストールする必要があるといことがわかる。そしてそれら2つの追加パッケージのpackage.jsonもみてみるとdependenciesの項目がないので追加でインストールする必要のあるパッケージがないことがわかる。

従って、すべてのパッケージのpackage.jsonのdependenciesに記載されているパッケージがインストールされているのでこれでようやくchalkが動かすことができる。


![](/images/npm-dependencies/img_npmdependencies_1.jpg)


プロジェクトフォルダをみてみると上の画像のようにあるパッケージは単独で`package.json`ファイルを各々持ち、dependenciesに必要なパッケージがかかれている場合にはそれぞれの依存を満たすために必要なパッケージをすべてインストールする必要がある。

## 依存関係の把握に使えるコマンド

ここで依存関係を把握するために便利なnpmコマンドを知ることができたので以下のcowsayというパッケージを単体でインストールしてみたログを使って紹介する。

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
# dedupedは重複

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
	- 依存関係をツリー構造で表示してくれるコマンド。オプション`-a`をつけることで完全なツリー構造表示になる。
	- dedupedは重複しているパッケージを示す。
- [npm explain](https://docs.npmjs.com/cli/v7/commands/npm-explain)
	- `npm ls -a`ではルートという上層から見た構造表示であったが、このコマンドではパッケージを指定することでそのパッケージが逆にどのパッケージから必要とされているかを階層表示してくれる。

ちなみにcowsayパッケージの`package.json`ファイルは以下

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

dependenciesに記載されているパッケージ(get-stdin, string-widht, strip-final-newline, yargsの4つ)はcowsayが動作する上で必要なのでこのパッケージもインストールされる。しかし、この4つのパッケージにもそれぞれ`package.json`ファイルがあり、それぞれdependenciesを持つ。つまりそれらのdependenciesも動作する上で必要なのでインストールされる。更にインストールされたパッケージも更にdependenciesを持つ、動作に必要なのでインストールされる....ということを繰り返してすべての依存関係のあるパッケージがインストールされた結果としてnode_modulesディレクトリが大きくなり、`npm ls -a`で表示されるような依存ツリーが出来上がる。

次のようなパッケージの依存関係をグラフ表示てくれるサービスもあるので試してみた。

https://npm.anvaka.com/#/

![](/images/npm-dependencies/img_npmdependencies_2.jpg)

Chalkの依存関係は比較的少ないのでかなりわかりやすい。

# ロックファイル

`package.json`ファイルに記載されたdependenciesや`npm install パッケージ名`でインストールしたパッケージの依存する他パッケージがインストールされるのはわかったが、`node_modules`に入っている全パッケージの情報はどうやって確認するのかという疑問が立ち上がる。例えばGithubのプロジェクトでcloneしてから実際に`npm i`してみるまでインストールされるものがわからないということは無いはずだ。どんなサイズになるかもわからないし、セキュリティ的にもよろしくないのではないか。

もちろんその方法は存在しており、すべてのインストールされるべきパッケージを記載しており、依存関係をすべて把握しているのが`package-lock.json`ファイルということになる。

## ロックファイルの目的

ということで、npm v7で使用される`package-lock.json`ファイルの目的と主要なフィールドについて公式ドキュメントから調べてみた。

https://docs.npmjs.com/cli/v7/configuring-npm/package-lock-json

>`package-locl.json`ファイルは`node_modules`ツリーもしくは`package.json`ファイルを変更するすべての操作対して自動的に生成されるファイルであり、後のインストールによって同一の依存ツリーを生成できるように正確なツリーを記述する。
>
>このファイルはソースリポジトリへコミットするように意図されており、以下の様々な目的に役立つ。
>
> -   同一の依存関係を正確にインストールできることを保証するために依存ツリーを単一の表現として記述する
> -   `node_modules`についてディレクトリそのものをコミットする必要なく、推移的な状態を確認することを可能にする
> -   ソースの読みやすい差分コントロールによってツリー変更を見やすくする
> -   過去にインストールされたパッケージについて繰り返される冗長なメタデータ解決を省略することでインストールプロセスを最適化する
>   - npm v7においてロックファイルはパッケージツリーの完全な全体図を得るための十分な情報を含んでおり、`package.json`ファイルを読む必要を減らした結果、パフォーマンスが大幅に向上する

:::message
ロックファイルの大きな目的は`package-lock.json`ファイルにすべての推移的な依存関係を含ませることで、異なる時間に異なるマシンを使ったとしてもまったく同一の依存関係を再現したインストールを保証してくれることにある。
参考: - [# Lockfile とは?このファイルのコミットって必要？ [Beginner's Series to Node.js 9/26]](https://youtu.be/7XQU0Obs_wk)
:::

## package-lock.jsonの構造

- `name`: パッケージの名前
	`package.json`のnameと一致する
- `version`: `package-lock.json`のバージョン
	`package.json`のvserisionと一致する
- `lockfileVersion`: 整数のバージョン番号
	- バージョン番号がない: npm v5以前の古いshrinkwarpファイルであることを示す
	- `1`: npm v5またはv6で使用されているロックファイルであることを示す
	- `2`: npm v7によって使用されているロックファイルであることを示し、v1のロックファイルへの後方互換性を持っている
	- `3`: npm v7によって使用されているロックファイルであることを示すが、後方互換性はない。`node_modules/.package-lock.json`の隠しロックファイルに使用されており、将来的なnpmで使用される可能性が高い。
- `requires`: 公式ドキュメントに記載なし(ブール値)
- `package`: パッケージの情報を含むオブジェクトへのパッケージロケーションをマッピングするオブジェクト
	- ルートプロジェクトは`""`というキーでリストされており、他のパッケージはルートプロジェクトフォルダからの相対パスで記載される。
	- `version`: `package.json`に記載されているバージョンと一致する。
	- `resolved`: パッケージの実際のロケーション。
		- npmレジストリから取得されている場合にはtarballへのURL
		- git依存の場合にはコミットハッシュ付きgitのフルURL
	- `integrity` : このロケーションに解凍された依存パッケージに対するSRI(サブリソース完全性: [Standard Subresource Integrity](https://w3c.github.io/webappsec/specs/subresourceintegrity/)))として使われる文字列(sha512またはsha1で暗号化されたハッシュ値)
	- `bin`, `license`, `engines`, `dependenceis`, `optionalDependnecies`: `package.json`に記載されているフィールドから転記。
- `dependencies`: `lockfileVersion: 1`を使用しているnpmのバージョンをサポートするレガシーデータ。npm v7では`pacakge`セクションが存在すればこのセクションを完全に無視するが、npm v6とv7のスイッチングをサポートするためにデータを保持している。なので基本的にv7を使っていれば中身は無視してよい。


```json:package-lock.jsonファイルの中身
{
    "name": "ProjectName",
	"version": "1.0.0",
    "lockfileVersion": 2,
    "requires": true,
    "package": {
        "": { // こいつがルートプロジェクト
            "dependencies": {
                "cowsey": "^1.5.0"
            }
        },
        // 以下インストールしたパッケージをアルファベット順ですべて記載
        "node_modules/ansi-regex": {
            "version": "3.0.0",
            "resolved": "https://registry.npmjs.org/ansi-regex/-/ansi-regex-3.0.0.tgz",
            "integrity": "sha1-7QMXwyIGT3lGbAKWa922Bas32Zg=",
            "engines": {
                "node": ">=4"
            }
        },
        // 中略
        "node_modules/yargs/node_modules/strip-ansi": {
            "version": "6.0.1",
            "resolved": "https://registry.npmjs.org/strip-ansi/-/strip-ansi-6.0.1.tgz",
            "integrity": "sha512-Y38VPSHcqkFrCpFnQ9vuSXmquuv5oXOKpGeT6aGrr3o3Gc9AlVa6JBfUSOCnbxGGZF+/0ooI7KrPuUSztUdU5A==",
            "dependencies": {
            "ansi-regex": "^5.0.1"
            },
            "engines": {
            "node": ">=8"
            }
        }
    },
    "dependencies": {
		// npm v7ではこのセクションは基本的に無視するので省略
    }
}
```


# メモ

- npmについて調べるには使用しているバージョンのnpmについての情報が必要でありバージョンが変更されたら公式ドキュメントを再び読む必要がある。
- Youtubeの英語版の動画にはかなり分かりやすくクオリティの高いものがあるのでドキュメントだけではなく動画も調べる価値がある。
