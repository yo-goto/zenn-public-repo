---
title: "Nushell - 型付きシェルの基本とコマンド定義"
published: false
cssclass: zenn
emoji: "🔮"
type: "tech"
topics: [nushell, shell]
date: 2024-09-20
url: "https://zenn.dev/estra/articles/nu-typed-shell"
aliases: 記事『Nushell』
---

## はじめに

これまで Zenn では fish shell の記事をいくつか書いてきましたが、現在は Nushell という新しいシェルを使っています。

https://www.nushell.sh

実は Nushell のことは以前から知っていましたが、利用されてているプログラミング言語の概念やその恩恵についての知識が無かったため、より初心者にわかりやすい fish shell を利用していました。最近になって型システムや関数型言語などについての概念を取得したため、ようやく Nushell を使い始められました。この記事では Nushell とはどのようなシェルか、また fish で作成してきたようないくつかのコマンドを Nushell でどのように定義するかを見てきいきます。

## Nushellとは

Nushell とは Rust 言語で開発された新しいタイプのシェルであり、以下のような特徴があります。

- クロスプラットフォームシェル(Linux, macOS, Windows)
- 型システムと構造化データ
- 強力なプラグインシステム
- 既存データフォーマットとの連携(json, csv, toml, yaml, ...)
- 分かりやすいエラーメッセージ
- パイプラインを前提とした設計
- スコープされた環境
- デフォルトでイミュータブルな変数
- LSPやIDEのサポート

Nushell はシェルでもあると同時にプログラミング言語でもあり、２つの機能的な側面が一つのパッケージとして完全に統合されて提供されています。Nushell のデザイン目標はシンプルなコマンドをパイプラインで組み合わせて利用する Unix の思想を背景に、以下のような様々な領域からヒントを得てモダンな開発スタイルを構築することです。

- Bashのような伝統的なシェル
- PowerShellのようなオブジェクトベースのシェル
- TypeScriptのような漸進的型付け
- 関数型プログラミング
- システムプログラミング

fish 然り、そもそもシェルは「コマンド」という非常に小さな単位でインタラクティブにプログラムを行うことができる環境であり、即座のフィードバックを得られ学習が容易であることから個人的に好きなツールなのですが、そのようなシェル環境においても漸進的な型システムや関数型スタイルなどのモダンプログラミングのスタイルを利用できるのが Nushell の面白いところです。

## 使い方

### インストール

Nushell はバイナリをダウンロードしてインストールが可能です。macOSならHomebrewなどでのインストールが簡単です。

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

macOS での環境構築について説明します。意図的にログインシェルにはしないため注意してください(※ログインシェルは zsh を想定します)。

Nushell の環境設定は `XDG_CONFIG_HOME` という環境変数にあるロケーションを見るようになっており、macOS では少し設定がしずらい状況にあります。最新バージョン(0.98.0)においても、`$HOME/.config` 配下を見るようにはなっていないので `.zshrc` などで以下のように環境変数を export します。

```zsh:.zshrc
export XDG_CONFIG_HOME="$HOME/.config"
```

これで他のdotfilesなどの設定と同じ様に `~/.config` などに設定ファイルを配置できいます。

ログインシェルは zsh で、インタラクティブシェルとして起動した時には Nushell にしたい場合には `.zshrc` に以下のようにインタラクティブシェルとして Nushell を起動させるようにします。

```zsh:.zshrc
# Zshがインタラクティブシェルとして起動しているか確認
if [[ $- == *i* ]]; then
  # インタラクティブシェルの場合のみnushellを起動
  exec nu
fi
```

参考
https://qiita.com/tak-onda/items/a90b63d9618d6b15c18c

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

```nu
# env.nuの場所
> $nu.env-path

# config.nuの場所
> $nu.config-path
```

環境変数 `$env.EDITOR` にデフォルトのエディタを設定できるようになっており、例えば vscode (`code`) を設定しておくことで、以下のコマンドから設定ファイルを直接アクセスできるようになります。

```nu
# env.nuの編集
> config env

# config.nuの編集
> config nu
```

#### 環境変数とPATHの設定

`env.nu` に以下のように環境変数やPATHなどの設定を記述することでパスを通すことができます。それぞれの環境変数は `$env` 配下に設定するようにして、`PATH` も同様に `$env.PATH` 配下に設定します。変数の参照については後で改めて解説しますが他のシェルと同じ様に `$` を変数名にプレフィックスして `$env` のように参照することが可能です。

```nu:.env.nu
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

設定や後で解説するカスタムコマンド(fish でいうところの function)は分割して管理したいので、自分の環境では

```sh
~/.config/nushell/
├── env.nu
├── config.nu
├── completions/
│  └── git-completions.nu
├── conf.d/
│  ├── alias.nu
│  ├── commands.nu
│  ├── completions.nu
│  ├── main.nu
│  └── theme.nu
├── functions/
│  ├── ggl-f.nu
│  ├── mod.nu
│  └── to.nu
└── misc/
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

Nushell の機能で特に強力なのが型システムであり、伝統的な標準入出力に頼る Bash などとは異なり、漸進的型付けのシステムや構造的データ型を利用して、あらゆる箇所に型をつけることができるようになっています。

これによってコマンド実行時の型の不一致を検知して分かりやすくエラーメッセージとして表示することができます。

<!-- エラーメッセージの画像 -->

さらに LSP や IDE サポートを提供しているため、シェルスクリプトにも関わらずコマンドの入出力についての型推論や変数への型注釈、注釈省略時の Inlay hints などの機能を使うことができます。

<!-- 型推論とインラインヒントの画像 -->

https://github.com/nushell/vscode-nushell-lang

### any と nothing

漸進的型付けを採用しているため、オプショナルな型付けが可能で、コンパイルタイムとランタイムで型チェックを行います。漸進的型付けのシステムで利用される Dynamic type (TypeScriptで言うところの `any` 型)に相当する `any` 型や `null` という単一項からなる Unit type である `nothing` 型が利用できるようになっており、柔軟な型システムとなっています。

### 構造的データ型

Nushell では以下の構造的データ型が利用できます。

- list
- record
- table

構造的型付けのシステムのため、$record<a: int, b: int> <: record<a: int>$ のような互換性があります。

### キャスト



### desribe コマンド

https://www.nushell.sh/commands/docs/describe.html

`describe` はパイプされた値のデータ型と構造を出力するコマンドです。このコマンドを使うことで値の型をインタラクティブに知ることができます。

```nu
> 'hello' | describe
string
> 42 | describe
int
```

## 変数宣言

Nushell で値は `let`, `const`, `mut` キーワードを使って名前付き変数に割り当て可能で、変数宣言後には `$` を変数名にプレフィックスして参照することが可能です。

```nu
> let val = 42
> print $val
42
```

変数名の命名規則としては以下の文字を変数名に含むことができません。

```txt
.  [  (  {  +  -  *  ^  /  =  !  <  >  &  |
```

他のシェルでは一般的な `$` をプレフィックスして変数を宣言する事が可能となっており、この場合には `$` を付けずに宣言した場合と同じように扱われます。

```nu
let $val = 42
# `let val = 42` と同じ扱いとなる
```

Nushell には３つの変数宣言がありますが、関数型プログラミングのスタイルを受けて変数はデフォルトでイミュータブルなものとして扱われます。

宣言の種類 | 作成される変数
--|--
let 宣言 | 宣言後に変更不可なミュータブルな変数(イミュータブル変数)
const 宣言 | パース時に完全に評価できるイミュータブルな変数(コンスタント変数)
mut 宣言 | 宣言後に再割り当て可能なイミュータブルな変数(ミュータブル変数)

```nu
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

```nu
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

```nu
> mut v3 = 42
> $v3 = 2 # OK
```

## パイプライン

## カスタムコマンド

### シグネチャ

### 具体例

ここからは簡単なコマンド定義からコマンドの作成方法を見ていきます。

#### mkdir して cd するコマンド

```nu
def --env mkdir-cd [dirname: path] {
  mkdir $dirname
  cd $dirname
}
```

このコマンドの定義では気をつけるべき点が二点あります。

まず、コマンド引数の変数は `path` 型の注釈が必要となります。`string` 型とは少々扱いが異なるで fish などとのコマンド定義とは違う点に注意が必要です。

また、Nushell では環境変数の変換はブロックでスコープされてしまうので、外部へと継続させるために `def` コマンドに `--env` オプションを付ける必要があります。`cd` コマンドはそもそも `pwd` という環境変数を変更するため、`cd $dirname` での環境変数の変更をコマンド外部へと継続できるようになります。

#### vscode のラッパーコマンド

```nu
# vscodeのラッパー
def vs [
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

#### ezaのオプションラッパー

```nu
# ezaのツリー表示
def lt [
  path: path = '.',
  --level (-l): int = 1,
  ...options: string
] {
  ^eza --tree --level $level ...$options $path
}
```

#### Web検索コマンド
