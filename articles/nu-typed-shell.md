---
title: "Nushell - 型付きシェルの基本とコマンド定義"
published: true
cssclass: zenn
emoji: "🔮"
type: "tech"
topics: [nushell, shell]
date: 2024-09-20
url: "https://zenn.dev/estra/articles/nu-typed-shell"
aliases: 記事『Nushell』
---

## はじめに

これまで Zenn では [fish shell](https://fishshell.com) の記事をいくつか書いてきましたが、現在は Nushell という新しいシェルを使っています。

https://www.nushell.sh

実は Nushell のことは以前から知っていましたが、利用されているプログラミング言語の概念やその恩恵についての知識が無かったため、より初心者にわかりやすい fish shell を利用していました。最近になって型システムや関数型言語などについての概念を取得したため、ようやく Nushell を使い始められました。

使い始めてからまだ1ヶ月ぐらいですが、かなり奥が深く一つの記事で解説しきるのは難しいので、この記事では基本体な設定と型とコマンドについて重点をおいて最後は具体的なカスタムコマンドの定義をいくつか取り上げて解説したいとおもいます。

## Nushellとは

Nushell とは Rust 言語で開発された新しいタイプのシェルであり、以下のような特徴があります。

- クロスプラットフォームシェル(Linux, macOS, Windows, ...)
- 既存データフォーマットとの連携(json, csv, toml, yaml, ...)
- 強力なプラグインシステム
- **型システムと構造化データ**
- 分かりやすいエラーメッセージ
- **パイプラインを前提とした設計**
- スコープされた環境
- デフォルトでイミュータブルな変数
- LSP、IDEサポートの提供

Nushell はシェルでもあると同時にプログラミング言語でもあり、２つの機能的な側面が一つのパッケージとして完全に統合されて提供されています。Nushell のデザイン目標は**シンプルなコマンドをパイプラインで組み合わせて利用する Unix の思想を背景に、以下のような様々な領域からヒントを得てモダンな開発スタイルを構築すること**です。

- Bashのような伝統的なシェル
- PowerShellのようなオブジェクトベースのシェル
- TypeScriptのような漸進的型付け
- 関数型プログラミング
- システムプログラミング

fish 然り、そもそもシェルは「コマンド」という非常に小さな単位でインタラクティブにプログラムを行うことができる環境であり、即座のフィードバックを得られ学習が容易であることから個人的に好きなツールなのですが、そのようなシェル環境においても漸進的な型システムや関数型スタイルなどのモダンプログラミングのスタイルを利用できるのが Nushell の面白いところです。

## 使い方

まず、使い方についてですがクロスプラットフォームシェルを謳っているため、以下のようにそれぞれのプラットフォームで簡単にインストールできます。

### インストール

macOSならHomebrewなどでのインストールが簡単です。

```sh:macOS/Linux
# Homebrew
brew install nushell

# Nix profile
nix profile install nixpkgs#nushell
```

```sh:Windows
# winget
winget install nushell
```

### 環境設定

#### 設定ファイルの場所

ここでは macOS での環境構築について説明します。fish 以上に Nushell はPOSIXに準拠しておらず、意図的にログインシェルにはしないようにしていますので注意してください。(※ログインシェルは zsh を想定します)。

Nushell の環境設定は `XDG_CONFIG_HOME` という環境変数にあるロケーションを見るようになっており、macOS では少し設定がしずらい状況にあります。最新バージョン(0.98.0)においても、`$HOME/.config` 配下を見るようにはなっていないので `.zshrc` などで以下のように環境変数を export します。

```zsh:.zshrc
export XDG_CONFIG_HOME="$HOME/.config"
```

これで他のdotfilesなどの設定と同じ様に `~/.config` などに設定ファイルを配置できいます。

:::message
Nushell が [Alacritty](https://alacritty.org/config-alacritty.html) のように複数のロケーションを見てくれれば楽なのですがね。

```sh:AlacrittyのUNIXシステムでのconfigロケーション
`$XDG_CONFIG_HOME/alacritty/alacritty.toml`

`$XDG_CONFIG_HOME/alacritty.toml`

`$HOME/.config/alacritty/alacritty.toml`

`$HOME/.alacritty.toml`
```

参考: https://github.com/dirs-dev/directories-rs/issues/47
:::

ログインシェルは zsh としておいて、インタラクティブシェルとして起動した時には Nushell とする場合には `.zshrc` に以下のようにインタラクティブシェルとして Nushell を起動させるようにします。

```zsh:.zshrc
# Zshがインタラクティブシェルとして起動しているか確認
if [[ $- == *i* ]]; then
  # インタラクティブシェルの場合のみnushellを起動
  exec nu
fi
```

参考
https://qiita.com/tak-onda/items/a90b63d9618d6b15c18c

これで `~/.config/nushell` 配下に設定ファイルを置けるようになりました。この状態で nushell を起動すればデフォルト設定となるファイルをそのディレクトリにダウンロードするかどうかを尋ねるプロンプトが表示されるので Yes としてデフォルト設定を配置します。

#### 設定システム

Nushell は起動時に `.nu` 拡張子のスクリプトファイルをロードして実行する設定システムとなっており、以下の２つのファイルが必要となります。

- `env.nu`
  `config.nu` が実行される前に環境変数を定義したりファイルへ書き込むに利用されるファイル
- `config.nu`
  グローバル名前空間への定義やエイリアスの追加に使用され、`env.nu` で定義された環境変数や定数を利用できるファイル

したがって、デフォルトの設定ファイルは以下のように配置されるはずです。

```sh
~/.config/nushell/
├── env.nu
└── config.nu
```

そしてこの２つは以下の環境変数からロケーションを参照できます。

```sh
# env.nuの場所
> $nu.env-path

# config.nuの場所
> $nu.config-path
```

環境変数 `$env.EDITOR` にデフォルトのエディタを設定できるようになっており、例えば vscode (`code`) を設定しておくことで、以下のコマンドから設定ファイルを直接エディタを開いて編集できるようになります。

```sh
# env.nuの編集
> config env

# config.nuの編集
> config nu
```

#### 環境変数とPATHの設定

`env.nu` に以下のように環境変数やPATHなどの設定を記述することでパスを通すことができます。それぞれの環境変数は `$env` 配下に設定するようにして、`PATH` も同様に `$env.PATH` 配下に設定します。変数の参照については後で改めて解説しますが他のシェルと同じ様に `$` を変数名にプレフィックスして `$env` のように参照することが可能です。

```sh:.env.nu
# PATHに関連する環境変数の設定
$env.RUSTUP_HOME = ($env.HOME + '/.rustup')
$env.CARGO_HOME = ($env.HOME + '/.cargo')
$env.VOLTA_HOME = ($env.HOME + '/.volta')
$env.GOPATH = ($env.HOME + '/go')
$env.PROTO_ROOT = ($env.HOME + '/proto')

# PATHの設定
$env.PATH = (
  $env.PATH
  | split row (char esep)
  | prepend '/opt/homebrew/bin'
  | prepend ($env.CARGO_HOME + '/bin')
  | prepend ($env.VOLTA_HOME + '/bin')
  | prepend ($env.GOPATH + '/bin')
  | prepend ($env.PROTO_ROOT + '/bin')
  | prepend ($env.HOME + '/.deno/bin')
  | prepend ($env.HOME + '/.ghcup/bin')
  | prepend ($env.HOME + '/.cabal/bin')
  | uniq
)
```

PATH の書き方はまさにパイプライン(`|`)を使ったコマンドの記述となっています。既存の `PATH` の文字列の値を一旦リストに変換(`split row (char essp)`)してからリスト先頭に追加したいパス文字列を追加(`prepend`)していき、最終的に重複を防ぐように `uniq` というフィルターコマンドに通して再代入することで PATH を通しています。

#### 設定ファイルの分割

設定や後で解説するカスタムコマンド(fish でいうところの function)は分けて管理したいので、自分の環境では以下のように環境設定をディレクトリに分割して管理しています。

```sh
~/.config/nushell/
├── completions
│  ├── external.nu
│  ├── git-completions.nu
│  └── index.nu
├── conf.d
│  ├── alias.nu
│  ├── index.nu
│  └── theme.nu
├── functions
│  ├── f-ggl.nu
│  ├── f-lt.nu
│  ├── f-mkdir-cd.nu
│  ├── f-relogin.nu
│  ├── f-to.nu
│  ├── f-vs.nu
│  └── mod.nu
├── config.nu
└── env.nu
```

設定を分割するには、`source`コマンドによる設定の読み込みや、モジュールシステム用の`use`コマンドによるカスタムコマンドのimportを行うことで可能となります。

まず、基本的な設定の分割ですが、aliasやthemeといったものは散らかりがちなので `conf.d` というディレクトリを作成しておいて、それぞれ `alias.nu` と `theme.nu` というファイルに定義しておきます。`conf.d/index.nu` ではそれらの設定ファイルを `source` コマンドで読み込みます。

```nu:conf.d/index.nu
source theme.nu
source alias.nu
source .zoxide.nu
```

:::message alert
環境変数の読み込み順番に注意してください。`$env.config` 内で変数などを参照するには事前に `source` しておく必要がありますので、`config.nu` ファイルでは先頭で `source` するようにします。
:::

そして、`config.nu` ファイルでこの `index.nu` ファイルを `soruce` します。これで設定の分割が完了です。`completions` ディレクトリについても同様です。

```nu
source conf.d/index.nu
source completions/index.nu
```

カスタムコマンドについては `source` でもできるのですが、どうせなら[モジュールシステム](https://www.nushell.sh/book/modules.html#modules-from-directories)を使おうということで、それぞれのカスタムコマンドには `export` コマンドを付与しておきます。後で解説する、`mkdir-cd` というコマンド定義では以下のように `def` コマンドの頭に `export` を付けます。

```nu:f-mkdir-cd.nu
export def --env mkdir-cd [dirname: path] {
  mkdir $dirname
  cd $dirname
}
```

そしで、`functions` ディレクトリ内の `mod.nu` というファイルを作成して、各カスタムコマンドのimportと再exportを以下の形式で行います。

```nu:functions/mod.nu
export use f-ggl.nu *
export use f-vs.nu *
export use f-relogin.nu *
export use f-lt.nu *
export use f-mkdir-cd.nu *
```

そして、`config.nu` で以下の様にimportすることでカスタムコマンドの利用ができるようになります。

```nu:config.nu
# 関数の利用
use functions/
```

#### Starshipの設定

https://starship.rs

Nushell 同様に Rust 言語で開発されたクロスシェルのプロンプトスタイル設定が可能なStartshipの設定を[公式の方法](https://starship.rs/guide/)で行っておきます。まずは、`env.nu` ファイルに以下を書き込みます。

```nu:env.nu
mkdir ~/.cache/starship
starship init nu | save -f ~/.cache/starship/init.nu
```

さらに、`config.nu` で以下のコマンドを記述しておきます。

```nu:config.nu
use ~/.cache/starship/init.nu
```

これで Nushell を起動した時にStarshipが使えるようになります。

なお、Starship 自体の設定は `~/.config/starship.toml` に書き込みます。

## データ型

Nushell の機能で特に強力なのが型システムであり、文字列により標準入出力に頼るトラディショナルなUnixシェルとは異なり、漸進的型付けのシステムや構造的データ型を利用して、あらゆる箇所に型をつけることができるようになっています。

これによってコマンド実行時の型の不一致を検知して分かりやすくエラーメッセージとして表示することができます。

![エラーメッセージの画像](/images/nushell/img_nushell-error.png)

さらに LSP や IDE サポートを提供しているため、シェルスクリプトにも関わらずコマンドの入出力についての型推論や変数への型注釈、注釈省略時の Inlay hints などの機能を使うことができます。

![型推論とインラインヒント](/images/nushell/img_nushell-editor.png)

https://github.com/nushell/vscode-nushell-lang

### 基本的な型と値

型 | 値の例
--|--
整数(`int`) | `-65535`
浮動小数点数(`float`) | `9.9999`, `Infinity`
文字列(`string`)	| `"hole 18"`, `'hole 18'`, `` `hole 18` ``, `hole18`, `r#'hole18'#`
真偽値(`bool`) | `true`
日付(`datetime`) | `2000-01-01`
間隔(`duration`) | `2min + 12sec`
ファイルサイズ(`filesize`) | `64mb`
範囲(`range`) | `0..4, 0..<5`, `0..`, `..4`
バイナリー(`binary`) | `0x[FE FF]`
クロージャ(`closure`) | `{\|e\| $e + 1 \| into string }`, `{ $in.name.0 \| path exists }`
セルパス(`cell-path`) | `$.name.0`
ブロック | `if true { print "hello!" }`, `loop { print "press ctrl-c to exit" }`

この他にも注釈できないような特殊な型などが複数個存在しています。

### desribe コマンド

https://www.nushell.sh/commands/docs/describe.html

`describe` はパイプされた値のデータ型と構造を出力するコマンドです。このコマンドを使うことで値の型をインタラクティブに知ることができます。

```sh
> 'hello' | describe
string
> 42 | describe
int
```

### 特殊な型

漸進的型付けを採用しているため、オプショナルな型付けが可能で、コンパイルタイムとランタイムで型チェックを行います。漸進的型付けのシステムにおける静的に未知な型である [Dynamic type](https://en.wikipedia.org/wiki/Gradual_typing) (TypeScriptで言うところの `any` 型)に相当する `any` 型や、`null` という単一項からなる [Unit type](https://en.wikipedia.org/wiki/Unit_type) である `nothing` 型が利用できるようになっており、柔軟な型システムとなっています。

#### any

https://www.nushell.sh/lang-guide/chapters/types/basic_types/any.html

`any` 型はあらゆる型のスーパーセットとなります。`any` 型として注釈した変数・パラメータ・入力はあらゆる型の値を受け入れるようになります。

```sh
> mux x: any = null
> x = 42
> x = 'st'
```

#### nothing

https://www.nushell.sh/lang-guide/chapters/types/basic_types/nothing.html

`nothing` 型は「値がないこと」を表現する型です。TypeScriptでは`void`型や`null`型が近いです。NushellではTS同様に`null`という値があり、この単一項からなるUnit typeとして`nothing`型が利用されます。

```sh
> null | describe
nothing
> null | to json
null
> "null" | from json
# => 出力なし
```

コマンドの入出力について明示的に何も無いことを示したい場合などはこの型で注釈できます。

```nu
def take-nothing []: nothing -> nothing {
  print "何もしない"
}

# 入力になんらかの値を渡すと型エラー
42 | take-nothing
#    ^^^^^^^^^^^^ Error: Command does not support int input

# これはnothing型の値を渡すのでOK
null | take-nothing
```

例えば、ビルトインコマンドである [`print`](https://www.nushell.sh/commands/docs/print.html#print-for-strings) は入出力として可能な値の肩は以下のパターンとなっています。

input	| output
--|--
any	| nothing
nothing	| nothing

これはつまり、パイプラインの入力から何も値を受け取らないかあらゆる値を受け取る、そしてパイプラインの出力に何も値を渡さないというパターンとなります。

```sh
# 入力としてint型の値を渡す
> 42 | print
42

# 入力に何も渡さない
> print
# => 出力なし

# 上と同じこと
> null | print
# => 出力なし
```

### 構造的データ型

Nushell では以下の構造的データ型が利用できます。

型 | 値の例
--|--
リスト(`list`)	| `[0 1 'two' 3]`
レコード(`record`) |	`{name:"Nushell", lang: "Rust"}`
テーブル(`table`) | `[{x:12, y:15}, {x:8, y:9}]`, `[[x, y]; [12, 15], [8, 9]]`

変数宣言では以下のようになります。なお`table`型の値は内部的には`record`の`list`となっています。

```nu
# list型
let l: list<string> = ['Sam', 'Fred', 'George']

# record型
let r: record<name: string, gender: string> = {
  name: 'taro',
  gender: 'male'
}

# table型
let t: table<x: int, y: int> = [
  {x: 12, y: 5},
  {x: 3, y: 6}
]
```

公式ドキュメントには明示されていませんが、以下のソースコードの部分で２つの型の互換性チェックを行っているようで、構造的部分型付け(**structural subtyping**)のシステムになっているようです。これによって $record \langle a: int, b: int \rangle <: record \langle a: int \rangle$ といった互換性があります。

https://github.com/nushell/nushell/blob/a948ec6c2cd2d2486589e73e701bd2c0a91a7547/crates/nu-parser/src/type_check.rs#L10-L69

実際、型注釈で `record<a: int>` を期待する型でパラメータを注釈したとして、このコマンドに `record<a: int, b: int>` のような型の値を渡しても問題はありません。

```sh
> def type-ch [param: record<a: int>] { print $param }
# OK な例
> type-ch {a : 1, b: 2}
╭───┬───╮
│ a │ 1 │
│ b │ 2 │
╰───┴───╯
# NG な例
> type-ch {b : 2}
Error: nu::parser::type_mismatch

  × Type mismatch.
   ╭─[entry #16:1:9]
 1 │ type-ch {b : 2}
   ·         ───┬───
   ·            ╰── expected record<a: int>, found record<b: int>
   ╰────
```

テーブル型は例えば、`ls` コマンドの出力などに利用されています。

```sh
> ls
╭───┬─────────────┬──────┬──────────┬──────────────╮
│ # │    name     │ type │   size   │   modified   │
├───┼─────────────┼──────┼──────────┼──────────────┤
│ 0 │ completions │ dir  │    160 B │ 2 days ago   │
│ 1 │ conf.d      │ dir  │    192 B │ 2 days ago   │
│ 2 │ config.nu   │ file │ 25.0 KiB │ 2 days ago   │
│ 3 │ env.nu      │ file │  2.0 KiB │ 2 days ago   │
│ 4 │ functions   │ dir  │    320 B │ 2 days ago   │
│ 5 │ history.txt │ file │ 13.8 KiB │ a minute ago │
╰───┴─────────────┴──────┴──────────┴──────────────╯
> ls | describe
table<name: string, type: string, size: filesize, modified: date> (stream)
```

### キャスト

型のキャストについては [`into`](https://www.nushell.sh/commands/docs/into.html) コマンドで行うことができます。例えば、`bool` 型の値への変換は [`into bool`](https://www.nushell.sh/commands/docs/into_bool.html) というコマンドで可能です。

```sh
> true | into bool
true
> 1 | into bool
true
> 0 | into bool
false
> '1' | into bool
true
```

## 変数宣言

Nushell で値は `let`, `const`, `mut` キーワードを使って名前付き変数に割り当て可能で、変数宣言後には `$` を変数名にプレフィックスして参照することが可能です。

```sh
> let val = 42
> print $val
42
```

変数名の命名規則としては以下の文字を変数名に含むことができません。

```txt
.  [  (  {  +  -  *  ^  /  =  !  <  >  &  |
```

他のシェルでは一般的な `$` をプレフィックスして変数を宣言する事が可能となっており、この場合には `$` を付けずに宣言した場合と同じように扱われます。

```sh
let $val = 42
# `let val = 42` と同じ扱いとなる
```

Nushell には３つの変数宣言がありますが、関数型プログラミングのスタイルを受けて変数は**デフォルトでイミュータブル**なものとして扱われます。

宣言の種類 | 作成される変数
--|--
`let` 宣言 | 宣言後に変更不可なイミュータブルな変数(**イミュータブル変数**)
`const` 宣言 | パース時に完全に評価できるイミュータブルな変数(**コンスタント変数**)
`mut` 宣言 | 宣言後に再割り当て可能なミュータブルな変数(**ミュータブル変数**)

```sh
> let v1 = 42
> $v1 = 2 # NG
Error: nu::compile::assignment_requires_mutable_variable

  × Assignment to an immutable variable.
   ╭─[entry #46:1:1]
 1 │ $v1 = 2
   · ─┬─
   ·  ╰── needs to be a mutable variable
   ╰────
  help: declare the variable with `mut`, or shadow it again with `let`

Error:   × Can't evaluate block in IR mode
   ╭─[entry #46:1:1]
 1 │ $v1 = 2
   · ───┬───
   ·    ╰── block is missing compiled representation
   ╰────
  help: the IrBlock is probably missing due to a compilation error
```

```sh
> const v2 = 42
> $v2 = 2 # NG
Error: nu::compile::assignment_requires_mutable_variable

  × Assignment to an immutable variable.
   ╭─[entry #44:1:1]
 1 │ $v2 = 2
   · ─┬─
   ·  ╰── needs to be a mutable variable
   ╰────
  help: declare the variable with `mut`, or shadow it again with `let`

Error:   × Can't evaluate block in IR mode
   ╭─[entry #44:1:1]
 1 │ $v2 = 2
   · ───┬───
   ·    ╰── block is missing compiled representation
   ╰────
  help: the IrBlock is probably missing due to a compilation error
```

```sh
> mut v3 = 42
> $v3 = 2 # OK
```

## パイプライン

トラディショナルなUnixシェルでは文字列による標準入出力を使って複数のコマンドを組み合わせ処理するパイプラインという技術を利用しています。

### 外部コマンドとの組み合わせ

Nushellのビルトインコマンド(内部コマンド)同士のパイプラインではNushellのデータ型を使ったやり取りがおこなわれますが、外部コマンドが絡んだ以下のようなパターンでは、それぞれうまく機能するように調整されています。


- (1) `内部コマンド | 外部コマンド` : 内部コマンドの出力は文字列に変換されて外部コマンドの`stdin`へと送信される
- (2) `外部コマンド | 内部コマンド` : 外部コマンドの出力は自動的にUTF-8テキストへと変換されて内部コマンドへと送信される
- (3) `外部コマンド | 外部コマンド` : Bashなどの他のシェルと同様に扱われる

### 特殊な in 変数

パイプラインの入力として渡ってくる値は一時的に変数として参照できると便利で、各パイプラインの `in` という変数に保持されます。例えば、明日の日付を使ってディレクトリを作成する際には以下のようなパイプラインを実行すればいいですが、`date now` というコマンドの出力結果は `$in` で参照できるので、その日付の値に一日追加することで次の日付が作成でき、その値を更に次のパイプラインへと流してフォーマットするということができています。

```sh
date now            # 1: 今日の日付
| $in + 1day        # 2: 明日の日付
| format date '%F'  # 3: YYYY-MM-DD としてフォーマット
| $'($in) Report'   # 4: ディレクトリ名を作成
| mkdir $in         # 5: ディレクトリの作成
```

この `in` 変数はコマンドのパラメータとして入力値を渡したいときや何らかの条件でフィルターなどを行うときなどに有用です。

## カスタムコマンド

https://www.nushell.sh/book/custom_commands.html

fish shell の [`function`](https://fishshell.com/docs/current/cmds/function.html) のようにカスタムのコマンドを定義するには Nushell では [`def`](https://www.nushell.sh/commands/docs/def.html) コマンドを使って以下のようなシグネチャでコマンドを定義します。

```nu
def greet [name] {
  ['hello' $name]
}
```

`greet` はコマンド名で、`name` はパラメータ名となります。そして Nushell ではカスタムコマンドの最後の行がそのコマンドの返り値として扱われます。つまり、次のパイプラインの入力として渡すことができる値を生成します。

:::message
`return` コマンドで明示的に返り値とすることも可能です。
:::

Nushellでは型注釈はオプショナルなので、上記コマンドのパラメータ `name` は `any` 型として推論されますが、型注釈を施すと以下のようにできます。

```nu
def greet [name: string] -> list<string> {
  ['hello' $name]
}
```

### コマンドの型シグネチャ

コマンドの型シグネチャは少し特殊です。普通のプログラミング言語の関数の入出力では単に引数と返り値という２つしかなく、それらに型注釈を施します。例えば TypeScript で以下のように定義したコマンドを考えます。

```ts:TypeScript
function greetSentence(name: string): string {
  return `hello, ${name}`;
}
```

似た処理を Nushell のカスタムコマンドで定義すると以下のようになるでしょうか？

```nu
def greetSentence [name: string] -> string {
  $"hello, ($name)"
}
```

ここで注意したいのは、シェルにおける入出力はパイプラインについてのものであり、パラメータは入力とは異なるものです。以下のようなパイプラインを考えると分かりやすいですが、コマンドのパラメータとは別にパイプラインの入力という値がコマンドに渡ってくるわけです。

```sh
| 42 | greetSentence 'Alice' | print
#   --> パイプラインの入力として42が渡る
#                  <--- コマンドのパラメータとして 'Alice' が greetSentence に渡る
#                           --> パイプラインの出力として "hello, Alice" が次のコマンドに渡る
```

ということで、`greetSentence` のパイプラインからの入力を主な処理として考える場合には以下のようにコマンドを定義します。

```nu
def greetSentence []: string -> string {
  $"hello, ($in)"
}
```

パイプラインの入力として渡ってくる値は特殊な `in` 変数で参照できたのでこのような形になります。

ちょっと分かりづらいですが、要するにカスタムコマンドの最初の一行目のコマンドがパイプラインの入力を受ける訳です。

わかりやすく別の変数に保持させるようにすれば以下のようになります。

```nu
def greetSentence []: string -> string {
  let name: string = $in;
  $"hello, ($name)"
}
```

まあ、このような処理の場合にはパラメータとして定義して、利用するパイプラインにおいて `$in` で参照してパラメータとして渡すとかの方が自然な感じがしますね。後、パイプラインの入力について明示的に `any` 型を受けるとして型注釈を行う事もできます。

```nu
def greetSentence [name: string] -> string {
  $"hello, ($name)"
}

'Alice` | greetSentence $in
```

少し話がそれましたが、カスタムコマンドの型シグネチャは以下のようになります。

```nu
def command-name [
  param: ParamType
]: InputType -> OutputType {
  # ...
}
```

### 具体例

ここからは簡単なコマンド定義からコマンドの作成方法を見ていきます。

#### mkdir して cd するコマンド

```nu
export def --env mkdir-cd [dirname: path] {
  mkdir $dirname
  cd $dirname
}
```

このコマンドの定義では気をつけるべき点が二点あります。

まず、コマンド引数の変数は `path` 型の注釈が必要となります。`string` 型とは少々扱いが異なるで fish などとのコマンド定義とは違う点に注意が必要です。

https://www.nushell.sh/lang-guide/chapters/types/other_types/path.html

公式ドキュメントの説明を使わせてもらうと、以下のようにカスタムコマンドのパラメータの型注釈を `string` とするか `path` 型とするかで、処理が異なります。

```nu
> def show_difference [
 p: path
 s: string
] {
 print $"The path is expanded: ($p)"
 print $"The string is not: ($s)"
}

# 使ってみると path 型の変数はパスを展開してくれることがわかる
> show_difference ~ ~
The path is expanded: /Users/roshi
The string is not: ~
```

このように `path` 型として注釈することで正しくパスを認識して展開できるようになるので、パラメータの型注釈は `path` とする必要があります。

また、Nushell では環境変数の変換はブロックでスコープされてしまうので、外部へと継続させるために [`def`](https://www.nushell.sh/commands/docs/def.html) コマンドに `--env` オプションを付ける必要があります。`cd` コマンドはそもそも `PWD` という環境変数を変更するため、`cd $dirname` での環境変数の変更をスコープ化されたコマンドブロックの外部へと継続できるようになります。

#### vscode のラッパーコマンド

次は、`code` という vscode のCLIコマンドのラッパーを定義します。

```nu
# vscodeのラッパー
export def vs [
  p: path = '.', # パス
  --insider (-i) # insider版を使うか
] {
  if $insider {
    ^code-insiders $p
  } else {
    ^code $p
  }
}
```

これもパス文字列をパラメータとして受ける場合には `path` 型として型注釈を施す必要があります。この時、`vs` とだけコマンドを実行した場合にはカレントディレクトリの vscode で開きたいのでデフォルト引数として `.` カレントディレクトリを指定します。

また、Insiderバージョンを使いたい場合があるので、その場合に備えてパラメータに `--insider` または省略版の `-i` と取るように定義します。

そして、外部コマンドを明示的に指定する場合には `^commandName` のように頭にキャレットをつけるようにします。これで名前が他のカスタムコマンドや内部コマンドと衝突することを避けることができます。

これで、以下のようなコマンド形式で実行することが可能となります。

```sh
# 一つ上のディレクトリ階層をインサイダー版で開く
vs ../ -i
```

また、コマンド定義を見ると `#` でコマンド名の前と、パラメータの後に説明を加えていることがわかると思いますが、これのコメントは自動的に `-h` または `--help` オプションで出力されるようになるという便利機能がついています。

```sh
> vs -h
vscodeのラッパー

Usage:
  > vs {flags} (p)

Flags:
  -i, --insider - insider版を使うか
  -h, --help - Display the help message for this command

Parameters:
  p <path>: パス (optional, default: '.')

Input/output types:
  ╭───┬───────┬────────╮
  │ # │ input │ output │
  ├───┼───────┼────────┤
  │ 0 │ any   │ any    │
  ╰───┴───────┴────────╯
```

#### eza のオプションラッパー

次はモダンな`ls`コマンドである [`eza`](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&ved=2ahUKEwi18YOuxNiIAxV4lFYBHf_NFFQQFnoECBUQAQ&url=https%3A%2F%2Fgithub.com%2Feza-community%2Feza&usg=AOvVaw37eaziSid2ANrcBeH3wcXU&opi=89978449) (`exa` のメンテナンス版)の `--level` オプションを使いやすくするためのラッパーコマンドを定義します。

```nu
# ezaのツリー表示
export def lt [
  path: path = '.',
  --level (-l): int = 1,
  ...options: string
] {
  ^eza --tree --level $level ...$options $path
}
```

vscode では単に `-i` という形でしたが、このコマンドのフラグパラメータ `-l` は `-l 2` のような形式で引数を取ることを可能にしています。型注釈は他のパラメータと同じ様にして、これもデフォルト引数を `1` で取るようにしてます。

他のオプションはレストパラメータ(`...`)の形式で `eza` にわたすことができるようにしています。

## 終わり

いかがでしたでしょうか。自分もNushellを使い始めて日が浅いので細かいことについてはまだ調査中ですが、型がついていたり、TypeScriptのようなエディタ上での書き味でシェルスクリプトが書けるので非常に気に入っています。

まだまだ開発途中のシェルなので、fish shell の方が優れている部分もいくつかあります。例えば補完周りの機能は圧倒的に fish shell の方が良いですし、`abbr` といった便利なシステムもありません。そしてなにより fish shell の方がユーザーフレンドリーで初心者にも若入りやすいです。ただ、型システムや構造化データ、パイプラインを全面に押し出した設計などは、他のプログラミング言語でそういったものに慣れていると非常に使い勝手の良いシェルに思えてきます。

自分のようにTypeSciptを使っている方であれば気に入ると思うので是非使ってみてください。
