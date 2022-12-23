---
title: "記事や本を作成しよう"
cssclass: zenn
date: 2022-12-23
modified: 2022-12-23
url: "https://zenn.dev/estra/articles/oz-make-article-book"
AutoNoteMover: disable
tags: [" #type/zenn/book #obsidian  "]
aliases: OZ本『記事や本を作成しよう』
---

# このチャプターについて

このチャプターでは、実際にノートから記事に作成を行う方法を解説します。

# 記事の作成

## コマンドパレットの使用

Obsidian では基本的にすべての操作がコマンドとして実行可能です。「コマンドパレット」のコアプラグインを有効化することで、コマンド入力用モーダルを開き、コマンドの呼び出しが可能になります。

「設定」→「コアプラグイン」→「コマンドパレット」からコマンドパレットプラグインを有効化してください。

![img](/images/oz/img_oz-command-palette.jpg)

デフォルトのホットキー `Cmd+P` でコマンドパレットを開くことができます。

![img](/images/oz/img_oz-open-cmd-palette.jpg)

## 新規ノートの作成

コマンドパレットが使用できるようになったら、「新規ノートの作成」コマンドを実行してください。これで新しいノートファイルが作成されます。このコマンドのデフォルトホットキーは `Cmd+N` なのでそのショートカットキーを使ってもよいです。

あるいは新規タブを開くと、以下のようにファイル作成操作がサジェストされるとそれをクリックすることでもファイルが作成できます。

![img](/images/oz/img_oz-make-new-note.jpg)

「新規ノートの作成」操作の実行後、ファイル名 (ノート名) の入力ができるようになるので、記事の URL となるスラグを入力してください。

作成するファイル名は適当に「my-new-article」という名前にしておきます。

## テンプレートの使用

記事用のファイルを作成したら『[プラグインを導入しよう](a-oz-add-plugins.md)』のチャプターで解説した Templater プラグインなどを使って記事テンプレートを展開していきます。

以下のようなテンプレートを作成してテンプレートディレクトリ `Templater/` などに配置しておきましょう。

```md:tp zenn
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
tags: [" #type/zenn  "]
aliases: 
---

# はじめに

```

Templater では以下のようなコマンドが提供されているので、ホットキーを設定しておきましょう。

![img](/images/oz/img_oz-templater-hotkeys.jpg)

よく利用するコマンドは「Open Insert Template modal」というコマンドで、これはテンプレート用ディレクトリに配置しているマークダウンフィアルから挿入したいテンプレートファイルを選択するコマンドです。`Ctlr+T` などのホットキーを設定しておくことで、すぐにスニペットやテンプレートを挿入できるようにします。

ホットキーを実行すると以下のように入力モーダルが開くので Zenn という名前を付けておいたテンプレートファイルを選択し Enter を入力することで現在開いているファイルにテンプレートの挿入ができます。

![img](/images/oz/img_oz-templater-cmd-for-zenn.jpg)

これで開いている `my-new-article.md` ファイルにテンプレートが展開されるので、必要なメタデータを YAML フロントマターに記載しておきましょう。

```md:my-new-article.md
---
title: "新しい記事のタイトル"
published: false
cssclass: zenn
emoji: "👻"
type: "tech" # theh or idea
topics: ["Zenn"] # ５つまで
date: 2022-12-23
AutoNoteMover: disable
url: "https://zenn.dev/estra/articles/my-new-article"
tags: [" #Zenn  "]
aliases: 新しい記事
---

# はじめに

```

これで記事の作成準備ができたので、あとは内容をがんばって書くだけです。学習した技術などについてまとめていきましょう!!

## ファイルの移動

サイドバーにあるファイルエクスプローラ上で `zenn-repo/articles/` ディレクトリから直接ファイルを作成してもよいですが、まだまだ未完成のドラフトならリポジトリ外の保管庫のどこかに保存しておいて、完成してからリポジトリに移動させてもよいでしょう。

Obsidian では新規ノートを作成するデフォルトのフォルダを指定することができます。これによって「新規ノートの作成」を行った時に作成されるノートは自動的に指定したフォルダに作成されます。

フォルダの場所は「設定」→「ファイルとリンク」→「新規ノートを作成するフォルダ」から設定できます。

![img](/images/oz/img_oz-default-note-dir.jpg)

指定していなければ保管庫のルートフォルダにファイルが作成されます。

それではデフォルトで作成されるファイルをリポジトリに移動させましょう。使用するコマンドは「**ファイルを別のフォルダに移動する**」というコマンドです。このコマンドについてもホットキーを `Cmd+M` などを設定しておきましょう。

このコマンドを実行するとフォルダパスの入力モーダルが表示されます。

![img](/images/oz/img_oz-move-file-modal.jpg)

`zenn-repo/articles` などのパスを入力すると移動先としてサジェストされるので Enter でファイルをリポジトリに移動させます。

![img](/images/oz/img_oz-move-file-to-zennrepo.jpg)

# 本のチャプターの作成

## 設定ファイルの作成

本を作成する場合には `config.yaml` ファイルなどが必要となるので、`Cmd+D` で「デフォルトアプリで開く」コマンド実行して VSCode などを開き、そこから `config.yaml` ファイルを作成してください。

![img](/images/oz/img_oz-make-configyaml.jpg)

## テンプレートの使用

本のテンプレートは記事用のテンプレートと対して変わりません。記事と同じ様に以下のようなテンプレートファイルをテンプレートディレクトリに用意しておきます。

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

チャプター用のファイルを「新規ファイルの作成」コマンドを実行し作成します。

ファイルが作成できたら、Templater を起動してファイルにチャプター用のテンプレートを展開し、必要なメタデータを YAML フロントマターに入力してきます。

```md:oz-make-article-book.md
---
title: "記事や本を作成しよう"
cssclass: zenn
date: 2022-12-23
AutoNoteMover: disable
url: "https://zenn.dev/estra/articles/oz-make-article-book"
tags: [" #type/zenn/book  "]
aliases: OZ本『記事や本を作成しよう』
---

# このチャプターについて

```

これでチャプターの内容を記述する準備ができました。あとは本の内容を実際につくっていくだけです。

# メモから記事や本を作ろう

Obsidian を使い始めると大量のメモを作成するクセがつきます。そういったメモは記事や本のネタになります。

『[情報を再利用しよう](6-oz-reuse-memo.md)』のチャプターでも言っていますが、もちろんメモそのものを記事にする k とも可能です。アトミックなノートとは別にまとめ用のメモを作っておき、そういったメモから記事やチャプターに転化することができます。

記事にできると思ったタイミングで、メモ用のディレクトリから「ファイルを別のフォルダに移動する」コマンドを使用して Zenn のリポジトリに転送しましょう。これで YAML フロントマターを Zenn 様に調整して git push すれば記事の出来上がりです。

逆にアーカイブしてきたいファイルやリポジトリの管理から外したいファイルなどはメモ用のディレクトリに転送することで記事やチャプターから除外することもできます。

また、記事になるようなクオリティや内容的にそぐわないけど、メモ自体を公開したいという場合にはアドオンサービスである Obsidian Publish を使いましょう。

https://obsidian.md/publish

筆者も利用していますが、記事にならないようなものでも生のメモとして公開しています。ここからまとまったら本のチャプターなどへ内容を反映させるようにしています。

https://publish.obsidian.md/ankiyorihajimeyo/Publish/Publish+Start+Page

また、Zenn にそぐわないような内容を含むものは Hugo などのマークダウンファイルから静的サイトをビルドできるツールをつかって別のサイトにホスティングすることもできます。Zenn と同じ様にリポジトリを専用のディレクトリに配置してマークダウンファイルを Obsidian で作成・編集するだけです。

# 記事やチャプターの継続編集を行おう

Zenn では GitHub のリポジトリからデプロイできるため、継続的な編集作業がかなり簡単にできます。Obsidian に Zenn のリポジトリをいれておけば、参照するための情報源として頻繁に使い回すことができます。

これによって、記事やチャプターなどにアクセスできる機会が飛躍的に向上するため、気づいたときやちょっとした瞬間に追加の編集ができるようになります。

また、『[プラグインを導入しよう](a-oz-add-plugins.md)』のチャプターで紹介した [Old Note Admonitor](https://github.com/tadashi-aikawa/obsidian-old-note-admonitor) プラグインなどを利用すれば、古い記事やチャプターであることがひと目で分かるので編集すべきタイミングが簡単にわかります。

記事やチャプターを生きたものとするために、ぜひ継続的に編集していきましょう。
