---
date: 2023-07-29
modified: 2023-07-29
title: "Denoでtextlintを使ってZennリポジトリを運用する"
published: true
cssclasses: zenn
emoji: "✍️"
type: tech
topics: [deno, textlint, 執筆]
url: https://zenn.dev/estra/articles/deno-textlint-zenn
tags:
  - type/zenn
  - deno/way
  - linter/textlint
aliases:
  - 記事『Denoでtextlintを使ってみた』
---

## はじめに

お久しぶりです。今回の記事では二番煎じですが Deno 環境で textlint を動かす方法、および Zenn のリポジトリを Deno 運用する方法について語ろうと思います。

自分の環境では極力 `node_modules` ディレクトリを作りたくないので、その方法として Zenn のリポジトリでは Deno を使うことにしました。Deno で textlint を動かす方法についての基本は以下の記事を参考にしていますが、もっと簡単に利用できるための方法を見つけたので紹介したいと思います。

https://zenn.dev/kn1cht/articles/deno-textlint

:::message
利用する環境は以下のバージョンです。

- Deno: v1.35.3
- textlint: v13.3.3
  :::

## 利用上の問題点

Deno で textlint を動かすこと自体はそれほど難しくないのですが、textlint のプレセットルールなどを一緒に動かすのが厄介です。素直にやるだけだとプレセットルールが動かないので事前にキャッシュなどをしておく必要がでてきます。今回はそういった回避方法を使ってより簡単に textlint をコマンドラインから使えるようにする方法を紹介します。

## 環境構築

実際に例として分かりやすいように自分の Zenn の公開リポジトリを Deno で動かすようにしたので、参考にしたい方は以下のリンクから見ることができます。

https://github.com/yo-goto/zenn-public-repo

それでは、基本的な環境構築から始めていきます。まずはこれまで Zenn のリポジトリで利用してきた邪魔な `node_modules` や `package.json`、`pnpm-lock.yaml` などを削除しておきます。ディレクトリは以下のような感じになります。大分すっきりしましたね。

```sh
❯ exa --classify --tree --level=1 -a
./
├── .git/
├── .gitignore
├── .prototools
├── .textlintrc
├── .vscode/
├── articles/
├── books/
├── images/
└── README.md
```

自分の環境ではツールチェインのバージョン管理に proto というツールを利用しており、このツールを使うことで Deno や Node、Rust のツールチェインなどを一元管理できます。

https://moonrepo.dev/proto

さらに、Deno 環境のバージョンを `.prototools` というファイルに記載することでコマンドラインからの実行時に利用するバージョンを固定できます。

```:.prototools
deno = "1.35.1"
```

## 依存パッケージの定義

実際に利用する依存モジュールの管理を行うファイルを作成します。Deno では基本的に `package.json` を使わず、慣習的に `deps.ts` ファイルにリモート依存として URL を配置しておきます。※ Node との互換性のために [`package.json` も利用可能](https://deno.land/manual@v1.35.3/node/package_json)です。

使い方は、以下のように npm specifier を使って以下のように副作用 import を記載しておきます。[import_map](https://deno.land/manual@v1.35.3/basics/import_maps) を使ってもよいですが import_map では直接 cache することができず `dep.ts` ファイルを介して利用することになるので、最初から `deps.ts` を使って依存を import するようにした方がはやいです。

```ts:deps.ts
import "npm:zenn-cli@^0.1.144";
import "npm:textlint@^13.3.3";
import "npm:textlint-rule-preset-ja-spacing@^2.3.0";
import "npm:textlint-rule-preset-jtf-style@2.3.13";
import "npm:textlint-rule-preset-ja-technical-writing@^8.0.0";
```

zenn-cli と textlint のプリセットとしてとりあえず以下の 3 つを使うことにします。追加したいプリセットルールやモジュールがあれば同様に定義してください。

- https://github.com/textlint-ja/textlint-rule-preset-ja-spacing
- https://github.com/textlint-ja/textlint-rule-preset-JTF-style
- https://github.com/textlint-ja/textlint-rule-preset-ja-technical-writing

## taskの定義

さて、Deno では基本的に設定などは必要がないですが、[npm script](https://docs.npmjs.com/cli/v9/using-npm/scripts?v=true) のように定期的に実行したいタスクを `deno.json` や `deno.jsonc` などの設定ファイルを作成しておき、これに task として定義してあげることで簡単に実行できるようになります。

今回は `deno.jsonc` ファイルを作成し、そこに textlint を使って実行したいチェックや修正などのタスクを定義しておきます。textlint のプリセットルールなどを利用する上で重要なこととして、プリセットのモジュールがストアにキャッシュされていることが必要です。したがって、textlint を実行する前にまずはモジュールのキャッシュを行いましょう。モジュールのキャッシュは `deps.ts` ファイルに対して `deno cache` コマンドを実行することで可能となります。これを task として定義しておきます。

```jsonc:deno.jsonc
{
  "tasks": {
    "cache": "deno cache deps.ts",
  }
}
```

これでコマンドラインからモジュールのキャッシュが簡単に行えるようになりました。以下のコマンドで `deps.ts` に配置された依存をキャッシュします。textlint を使う前にこのコマンドを一回だけ実行してください。これによって textlint がプリセットルールを認識できるようになります。

```sh
deno task cache
```

これで、textlint や zenn-cli をすぐに実行できます。

Deno は v1.28 から [npm specifier](https://deno.land/manual@v1.35.3/node/npm_specifiers) という機能が使えるようになりました。これによって npm のモジュールを Deno から利用できます。`deno run -A npm:zenn-cli --init` のように `deno run` コマンドを使うことで `npx` のようにコマンドを実行できます。このようなコマンド実行をいちいち CLI からやるのは面倒なので zenn-cli でのプレビューなどを実行するための task も定義しておきましょう。

```diff jsonc:deno.jsonc
{
  "tasks": {
    "cache": "deno cache deps.ts",
+   "zenn": "deno run -A npm:zenn-cli",
+   "zenn:preview": "deno task zenn preview",
+   "zenn:create:article": "deno task zenn new:article",
+   "zenn:create:book": "deno task zenn new:book",
  }
}
```

zenn-cli の各種コマンドの使い方は以下の公式ドキュメントを参照してください。

https://zenn.dev/zenn/articles/zenn-cli-guide

これで以下のコマンドでプレビューを実行できるようになりました。

```sh
deno task zenn:preview
```

さて、当該の textlint も同様に実行したい CLI からのコマンドを task として定義してあげます。

```diff jsonc:deno.jsonc
  "tasks": {
    "cache": "deno cache deps.ts",
    "zenn": "deno run -A npm:zenn-cli",
    "zenn:preview": "deno task zenn preview",
    "zenn:create:article": "deno task zenn new:article",
    "zenn:create:book": "deno task zenn new:book",
+   "lint": "deno run -A npm:textlint",
+   "lint:fix": "deno task lint --fix",
+   "lint:dry": "deno task lint --dry-run"
  },
```

ついでの Deno のビルトインフォーマッターの設定も追加しておきます。

```jsonc:deno.jsonc
{
  "tasks": {
    "cache": "deno cache deps.ts",
    "zenn": "deno run -A npm:zenn-cli",
    "zenn:preview": "deno task zenn preview",
    "zenn:create:article": "deno task zenn new:article",
    "zenn:create:book": "deno task zenn new:book",
    "lint": "deno run -A npm:textlint",
    "lint:fix": "deno task lint --fix",
    "lint:dry": "deno task lint --dry-run"
  },
  "fmt": {
    "useTabs": false,
    "indentWidth": 2,
    "semiColons": true,
    "singleQuote": false,
    "proseWrap": "preserve",
    "include": ["/articles", "/books"]
  }
}
```

これで textlint を実行するための準備ができました。

## textlintの設定

後は、textlint の設定ファイル `.textlintrc` でリントルールを定義しておきましょう。設定ファイル作成のために以下の初期化コマンドを実行します。

```sh
deno task lint --init
```

`.textlintrc` ファイルが作成されたら、以下のようにお好きなルールを定義してください。

```json:textlintrc
{
  "rules": {
    "preset-ja-spacing": true,
    "preset-jtf-style": {
      "1.1.3.箇条書き": false,
      "2.2.2.算用数字と漢数字の使い分け": false,
      "3.2.カタカナ語間のスペースの有無": false,
      "4.2.7.コロン(：)": false,
      "4.3.1.丸かっこ（）": false,
      "4.3.2.大かっこ［］": false,
      "4.3.3.かぎかっこ「」": false,
      "4.3.4.二重かぎかっこ『』": false
    },
    "preset-ja-technical-writing": {
      "sentence-length": false,
      "no-exclamation-question-mark": false,
      "no-hankaku-kana": true
    }
  }
}
```

後はコマンドを実行するだけです。ためしにこの記事を textlint してみましょう。全角文字と半角文字の間にスペースを入れてくれるルールによって修正がききましたね。

```sh
❯ deno task lint:fix ./articles/how-to-use-textlint-with-deno.md
Task lint:fix deno task lint --fix "./articles/how-to-use-textlint-with-deno.md"
Task lint deno run -A npm:textlint "--fix" "./articles/how-to-use-textlint-with-deno.md"

/zenn-repo/articles/how-to-use-textlint-with-deno.md
   78:37   ✔   原則として、全角文字と半角文字の間にスペースを入れます。  ja-spacing/ja-space-between-half-and-full-width
   78:38   ✔   原則として、全角文字と半角文字の間にスペースを入れます。  ja-spacing/ja-space-between-half-and-full-width
  106:279  ✔   原則として、全角文字と半角文字の間にスペースを入れます。  ja-spacing/ja-space-between-half-and-full-width
  106:283  ✔   原則として、全角文字と半角文字の間にスペースを入れます。  ja-spacing/ja-space-between-half-and-full-width
  130:38   ✔   原則として、全角文字と半角文字の間にスペースを入れます。  ja-spacing/ja-space-between-half-and-full-width
  130:42   ✔   原則として、全角文字と半角文字の間にスペースを入れます。  ja-spacing/ja-space-between-half-and-full-width

✔ Fixed 6 problems
```

リポジトリ内のすべての markdown ファイルについて実行したい場合にはリポジトリのルートで以下のコマンドを実行してください。

```sh
deno task lint:fix .
```

## まとめ

Deno を使って Zenn のリポジトリを運用し、textlint を実行する方法を紹介しました。僕の執筆環境では node_modules を絶対はやしたくないのでこのような構成としました。Deno を使うことでディレクトリが非常にスッキリしました。実際に Zenn の公開リポジトリは上記の方法を使っているので、興味があればリポジトリを見てみるといいかもしれません。
