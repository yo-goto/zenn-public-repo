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

これまで Zenn では fish shell の記事をいくつか書いてきましたが、現在は Nushell という新しいタイプのシェルを使っています。

https://www.nushell.sh

実は Nushell のことは以前から知っていましたが、プログラミング言語の概念をあまり知らなかったのでより初心者にわかりやすい fish shell を利用していました。最近になって型システムや関数型言語などについての概念を取得したため、ようやく Nushell を使い始められました。この記事では Nushell とはどのようなシェルか、また fish で作成してきたようないくつかのコマンドを Nushell でどのように定義するかを見てきいきます。

## Nushellとは

Nushell とは Rust で開発された新しいタイプのシェルであり、以下のような特徴があります。

- クロスプラットフォームシェル(Linux, macOS, Windows)
- 型システムと構造化データ
- 強力なプラグインシステム
- 既存データフォーマットとの連携(json, csv, toml, yaml, ...)
- 分かりやすいエラーメッセージ
- パイプライン
- スコープされた環境
- デフォルトでイミュータブルな変数
- LSPの提供

Nushell はシェルでもあると同時にプログラミング言語でもあり、２つの機能的な側面が一つのパッケージとして統合されて提供されています。Nushell のデザイン方針はシンプルなコマンドをパイプラインで組み合わせて利用する Unix 哲学を背景に、様々な領域からヒントを得てモダンな開発スタイルを構築することです。具体的には以下のような複数の領域からエッセンスを取り込んでいます。

- Bashのような伝統的なシェル
- PowerShellのようなオブジェクトベースのシェル
- TypeScriptのような漸進的型付け
- 関数型プログラミング
- システムプログラミング

## 使い方

### インストール

```sh
brew install nushell
```

### 環境設定

## データ型

Nushell の機能で特に強力なのが型システムであり、伝統的な標準入出力に頼る Bash などとは異なり、漸進的型付けのシステムや構造的データ型を利用して、あらゆる箇所に型をつけることができるようになっています。

これによってコマンド実行時の型の不一致を検知して分かりやすくエラーメッセージとして表示することができます。

さらに LSP や IDE サポートを提供しているため、シェルスクリプトにも関わらずコマンドの入出力についての型推論や変数への型注釈、注釈省略時のインラインヒント表示などの機能を使うことができます。

https://github.com/nushell/vscode-nushell-lang

### any と nothing

漸進的型付けを採用しているため、オプショナルな型付けが可能で、コンパイルタイムとランタイムで型チェックを行います。漸進的型付けのシステムで利用される Dynamic type (TypeScriptで言うところの `any` 型)に相当する `any` 型や `null` という単一項からなる Unit type である `nothing` 型が利用できるようになっており、柔軟な型システムとなっています。

### 構造的データ型

Nushell では以下の構造的データ型が利用できます。

- list
- record
- table

構造的型付けのシステムのため、$record<a: int, b: int> <: record<a: int>$ のような互換性があります。

## 変数宣言

Nushellには３つの変数宣言があります。

- let 宣言 (イミュータブル変数)
- mut 宣言 (ミュータブル変数)
- const 宣言 (コンスタント変数)

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
