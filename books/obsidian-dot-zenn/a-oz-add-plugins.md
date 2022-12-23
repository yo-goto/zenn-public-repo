---
title: "プラグインを導入しよう"
cssclass: zenn
date: 2022-12-22
modified: 2022-12-23
url: "https://zenn.dev/estra/articles/3-mbo-templater-plugin"
AutoNoteMover: disable
tags: [" #type/zenn/book  "]
aliases: OZ本『プラグインを導入しよう』
---

# このチャプターについて

このチャプターでは、ノート作成や Zenn での執筆の観点から役立つようないくつかのプラグインを紹介し、その使い方を解説します。

# プラグインについて

Obsidian では VSCode のようにプラグインシステムを採用しており、２種類のタイプのプラグインが存在しています。

タイプ | 説明
--|--
コアプラグイン | 公式開発チームから提供されている標準搭載のプラグイン
コミュニティプラグイン | コミュニティによって開発されたサードパーティのプラグイン

コアプラグインは標準搭載されているため、すぐに使い始めることができます。現在 26 個のコアプラグインが提供されており、この内の２つは有料のプラグインサービスとなっていますが、その他はすべて無料で利用することができます。

また、コミュニティプラグインも基本的に無料で使用することができ、多くの開発者は開発サポートなどを受け付けている場合がよくあります。

コミュニティプラグインはプラグインリストに公開するまでに安全性のためにコミュニティからコードレビューを受けますが、現在 (2022/12/22 時点) では 750 個ほどのプラグインが公開されています。

# Templater

この項目では Obsidian のコミュニティプラグインの１つである Templater を使ってテンプレートを作成する方法を紹介します。

## Templater プラグインとは

コアプラグインの１つに「テンプレートプラグイン」というものがありますが、コアプラグインよりもリッチなテンプレート表現を行えるプラグインとして Templater というコミュニティプラグインが提供されています。

公式リポジトリ

https://github.com/SilentVoid13/Templater

公式ドキュメント

https://silentvoid13.github.io/Templater

このプラグインを使って Zenn での記事や本作成に役立つテンプレートを作成する方法を解説します。

## テンプレートの基本設定

Templater の設定画面は以下のようになっており、Templater の利用を開始するには、 Templater Folder Location に記載するテンプレート用マークダウンファイルを保管しておくディレクトリを作成する必要があります。

![img](/images/oz/img_oz-templater-setting.jpg)

筆者の環境では、コアプラグインの方のテンプレートも保管しているので、以下のようなディレクトリ構成としています。

```sh
Templates/
├── Template MOC.md # 管理ノート
├── Template/ # コアプラグイン用テンプレート
└── Templater Plugin/ # コミュニティプラグイン用テンプレート
```

テンプレートファイル関連は Git 保管しており、この `Templates` ディレクトリで保管庫全体とは分離してバージョン管理できるようにしています。

`Template MOC.md` ではテンプレートファイルについての説明や運用方法などについて記載されています。`Templater Plugin` ディレクトリにテンプレートファイルを移動したら、Template Plugin の設定画面の Templater Folder Location の項目にそのパスを記載しておきます。

![img](/images/oz/img_oz-templater-path.jpg)

## 通常のメモ用テンプレートを作成しよう

それでは実際のテンプレートファイルの作成を行ってきます。

Obsidian では YAML フロントマターを利用してノートのメタデータを管理できますが、Zenn においても YAML フロントマターが利用できますね。

Zenn 用のテンプレートを作成するまえに、まずは Obsidian 全体で日常的に利用する基本テンプレートを作成しておきましょう。

このテンプレートは非常に簡素なものとしておきます。

```md:Templater Base.md
---
date: <% tp.date.now("YYYY-MM-DD") %>
tags: [" "]
aliases: <% tp.file.title %>
---

```

Templater では `<% %>` という記法を使用してテンプレート挿入箇所を記述します。現在の日付なら `<% tp.date.now("YYYY-MM-DD") %>` のようにすることで "2022-12-22" のような YYYY-MM-DD 日付フォーマットで挿入することができます。フォーマットは Momen.js を内部利用しているので以下のドキュメントに記載されているフォーマットが利用できます。

https://momentjs.com/docs/#/displaying/format/

また、ファイル名を参照するには `<% tp.file.title %>` とします。これによってファイルタイトルから自動的にノートのエイリアスを生成することができます。

このような通常のメモ用のテンプレートはよく使うものとして個別にホットキーを設定しておくとすぐに呼び出すことができます。

![img](/images/oz/img_oz-templater-cmd-hotkey.jpg)

あるいはコマンドパレットから「Templater: Open Insert Template modal」コマンドを実行することで、テンプレートファイルからどのテンプレートを挿入するか選択するモダールが開き、テンプレートを選択して Enter することで現在のノートへと選択したテンプレートを挿入することができます。

## 記事用のテンプレートを作成しよう

それでは、次に Zenn 用にテンプレートを作成して、記事執筆の際にすぐに呼び出せるようにしておきましょう。

実際に筆者が記事用に使っているテンプレートは以下のようなものです。

```md:tp zenn.md
---
title: "タイトル"
published: false
cssclass: zenn
emoji: "🔥"
type: "tech" # theh or idea
topics: [] # ５つまで
date: <% tp.date.now("YYYY-MM-DD") %>
AutoNoteMover: disable
url: "https://zenn.dev/estra/articles/<% tp.file.title %>"
tags: [" #type/zenn "]
aliases: 
---

# はじめに

```

本文そのものは冒頭の記事内容を紹介するための項目として『はじめに』を設けているだけで、殆どは YAML フロントマターによるメタデータのテンプレートです。

まずは、これを `tp zenn` という名前のマークダウンファイルで作成しておき、Templater 用のディレクトリに移動しておきます。

より自動化したい場合には、例えば以下のようなフォーマットを利用することでテンプレート挿入時にプロンプトを開き、入力するようにしておくことも可能です。

```md
---
published: false
cssclass: zenn
type: "<% await tp.system.suggester(['tech', 'idea'], ['tech', 'idea'], false, '記事のタイプを選択') %>"
emoji: "<% await tp.system.prompt('絵文字を１つ入力', '🔥') %>"
<%*
const title = await tp.system.prompt('記事タイトルを入力')
%>
title: "<%* tR += title  %>"
topics: []
date: <% tp.date.now("YYYY-MM-DD") %>
AutoNoteMover: disable
url: "https://zenn.dev/estra/articles/<% tp.file.title %>"
tags: [" #type/zenn "]
aliases: 記事『<% await tp.frontmatter.title %>』
---

```

## 本用のテンプレートを作成しよう

本用のテンプレートは記事と対して変わりません。筆者が使用しているテンプレートは以下のようなものです。

```md:tp zenn-book.md
---
title: "タイトル"
cssclass: zenn
date: <% tp.date.now("YYYY-MM-DD") %>
AutoNoteMover: disable
url: "https://zenn.dev/estra/articles/<% tp.file.title %>"
tags: [" #type/zenn/book  "]
aliases: 本『』
---

# このチャプターについて
```

## スニペットを登録しておこう

Zenn のマークダウン記法では以下のような独自記法を利用してアコーディオンやメッセージを作成することができます。

```md:tp zenn-snippet-details.md
:::details タイトル
表示したい内容
:::
```

```md:tp zenn-snippet-message.md
:::message
メッセージをここに
:::
```

こういった特殊なブロックタイプのテキストはテンプレートとしてスニペットに登録してすぐに呼び出せるようにしておくと便利です。

テンプレートそれぞれにはホットキーを登録しておくことができるので、より便利にしたい場合にはホットキーを設定しておくと良いでしょう。

# Obsidian Linter

この項目では、どんな時でも一定の執筆体験を担保するためのリンター用プラグインである Obsidian Linter を紹介します。

## Linter プラグインとは

Obsidian Linter は名前の通りのリンタープラグインです。公式リポジトリは以下です。

https://github.com/platers/obsidian-linter

Pretteir や日本語なら Textlint に相当する機能を持っており、メモ内の修正可能な箇所について自動的にフォーマットしてくれます。

このプラグインさえ入れておけば、Obsidian 内のメモや記事などのフォーマットを簡単に統一することができ、一定のクォリティを保つことができます。

あるいは、Textlint などで厳密に校正する必要がない個人的なメモなどのフォーマットはこれで十分でしょう。日常的なメモなどは Linter プラグインによってフォーマットしておき、記事などおの厳密な校正や執筆を行う際には VSCode などに移動して行うようにすると便利です。

## リンタールールを設定しよう

Linter の設定画面は以下のようになっています。

![linter設定](img_oz-linter-setting.jpg)

リンタールールについては公式リポジトリの以下のドキュメントにまとめられています。

https://github.com/platers/obsidian-linter/blob/master/docs/rules.md

リンタールールには以下の項目があります。

項目 | 説明
--|--
General | 一般設定
YAML | YAML のソートなどの設定
Heading | 見出しのフォーマット設定
Footnote | 脚注のフォーマット設定
Content | 内容のフォーマット設定 (主に英語)
Spacing | スペースのフォーマット設定
Paste | ペースト時のフォーマット設定
Custom | Regex を使ったカスタムフォーマット設定

設定項目が非常に多いため、ここでは主要な設定のみを解説しておきます。

## General

以下のような一般的な設定を定めます。

設定 | 説明 | 推奨設定
--|--|--
Lint on save | `Ctrl+S` などの保存コマンド実行時にリントするかどうか | 有効化
Display message on lint | リントによるフォーマット後に何文字変更されたかをメッセージ表示するかどうか | 有効化
Folders to ignore | リントしないように無視するディレクトリ | テンプレートノートのディレクトリ

Obsidian ではデフォルトでノートの自動保存が行われますが、手動での保存コマンド実行時にリンターが走るように設定しておくと便利です。

## YAML

この設定項目では、YAML フロントマターのフォーマットに関しての設定を行うルールがあります。

### キーのソート

この設定は『[YAML Key Sort](https://github.com/platers/obsidian-linter/blob/master/docs/rules.md#yaml-key-sort)』と呼ばれるルールで、有効化するとリンター起動時に YAML のキーをソートすることができます。

筆者は以下のようなキーのソート順番を「YAML Key Priority Sort Order」に設定しており、保存時に自動的にキーがこの順番でソートされるようにしています。

```
title
publish
published
cssclass
cssClasses
emoji
type
topics
date
modified
publish_date
url
AutoNoteMover
tags
aliases
```

### タイムスタンプ

この設定は『[YAML Timestamp](https://github.com/platers/obsidian-linter/blob/master/docs/rules.md#yaml-timestamp)』と呼ばれるルールで、有効化するとリンター起動時に YAML の日時データにファイル変更日を自動的に記載・変更してくれます。

設定 | 説明
--|--
Date Created | ファイル作成日時を挿入するかどうか (デフォルトで `true`)
Date Created Key | 作成日時に使用する YAML キー(デフォルトで `date created`)
Date Modified | ファイル最終変更日時を挿入するかどうか (デフォルトで `true`)
Date Modified Key | 最終変更日時に使用する YAML キー (デフォルトで `date modified`)
Format | 日時の使用フォーマット (デフォルトで `dddd, MMMM Do YYYY, h:mm:ss a`)

## Spacing

この設定項目では、空白や空行に関しての設定を行うルールがあります。

### 空行設定

ブロックの前後に空行を入れるかどうかの設定は以下のルールを有効化することで可能です。

ルール | 説明
--|--
[Empty Line Around Blockquotes](https://github.com/platers/obsidian-linter/blob/master/docs/rules.md#empty-line-around-blockquotes) | 引用ブロックの前後に空行を追加するかどうか
[Empty Line Around Code Fences](https://github.com/platers/obsidian-linter/blob/master/docs/rules.md#empty-line-around-code-fences) | コードブロックの前後に空行を追加するかどうか
[Empty Line Around Math Blocks](https://github.com/platers/obsidian-linter/blob/master/docs/rules.md#empty-line-around-math-blocks) | 数学ブロックの前後に空行を追加するかどうか
[Empty Line Around Tables](https://github.com/platers/obsidian-linter/blob/master/docs/rules.md#empty-line-around-tables) | テーブルの前後に空行を追加するかどうか

### 日本語と英数字間のスペース

この設定は『[Space between Chinese Japanese or Korean and English or numbers](https://github.com/platers/obsidian-linter/blob/master/docs/rules.md#space-between-chinese-japanese-or-korean-and-english-or-numbers)』と呼ばれるルールで、中国語・日本語・韓国語の文字と英数字の間にスペースを入れるかどうかを決めます。

このルールを有効化すると以下の差分のようにフォーマットを行います。

```diff md
- 日本語englishひらがな
- カタカナenglishカタカナ
- ﾊﾝｶｸｶﾀｶﾅenglish１２３全角数字
- 한글english한글
+ 日本語 english ひらがな
+ カタカナ english カタカナ
+ ﾊﾝｶｸｶﾀｶﾅ english１２３全角数字
+ 한글 english 한글
```

リンクやインラインコードについてもスペースを入れるようにフォーマットします。これによって、記事内の記法のフォーマットを統一することができます。

```diff md
- 中文字符串[english](http://example.com)中文字符串。
+ 中文字符串 [english](http://example.com) 中文字符串。
- 中文字符串`code`中文字符串。
+ 中文字符串 `code` 中文字符串。
```

# Old Note Admonitor

この項目では、 古いノートに対して警告を表示する Old Note Admonitor を紹介します。

Old Note Admonitor プラグインはファイルの編集日時などから古い内容であれば視覚的な警告を表示してくれるプラグインです。

https://github.com/tadashi-aikawa/obsidian-old-note-admonitor

このプラグインを使うことで例えば、古いノートや Zenn で１年前に公開した記事などを再度ひらた時に以下のような警告を出してくれます。

![img](/images/oz/img_oz-old-note-adomonitor-ex.jpg)

どのくらいの日時までを許容するかやメッセージなどは設定から変更できます。

![img](/images/oz/img_oz-old-note-admonitor-setting.jpg)

