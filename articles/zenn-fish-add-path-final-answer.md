---
title: "fishで「パスを通す」ための最終解答"
emoji: "🗽"
type: "tech"
topics: [fish, shell]
published: true
date: 2022-03-06
url: "https://zenn.dev/estra/articles/zenn-fish-add-path-final-answer"
aliases: [記事_fishでパスを通す最終解答]
tags: " #type/zenn #shell/fish/env  "
---

# はじめに

今回は、fish shell の大詰めとして、環境変数である `PATH` に新しいディレクトリを登録する、つまり「パスを通す」方法への最終解答を紹介します。

パス関連、環境変数関連の話は鬼門で、以外とちゃんと設定するのが難しいですよね。自分もなかなか上手く行かずに調査を繰り返しました。

「パスを通す」については、調べると色々な方法がでてきますが、現時点(2022/03/06)の最新バージョン `v3.3.1` では最も簡単でシンプルな方法が１つ提示されています。

その答えは、


...
...
...
...
...


`fish_add_path` 関数を使用する。


以上です。

パスを通すにはこれだけを使用し、むしろこれ以外を使おうとすると、うまく行かないケースがほとんどです。その理由については後で説明します。
この `fish_add_path` 関数は fish `v3.2.0` (2021/3/1 リリース)から提供されている比較的新しい関数なので、これ以前の古い記事に記載されていないので注意してください。

https://fishshell.com/docs/current/relnotes.html#fish-3-2-0-released-march-1-2021

# fish_add_path とは

`fish_add_path` 関数は fish の `$PATH` にコンポーネント(パス)追加するシンプルな方法であり、使用することでユニバーサル変数 `$fish_user_paths` または直接的に `$PATH` へと指定したコンポーネントを追加できます(`$PATH` に直接追加する場合には `--path` スイッチを使用する必要があります)。

https://fishshell.com/docs/current/cmds/fish_add_path.html


# fish_add_path の使い方

使い方は次の二通りあります。ドキュメントを見ると fish 側としてはインタラクティブな使用を推している感じですが、`config.fish` に記載する使用方法も用意されていました。

- (A) コマンドライン(インタラクティブ)で 1 回だけ実行する
- (B) `~/.config/fish/config.fish` に記載する

## コマンドラインで 1 回だけ実行
コマンドラインで追加したいパスにつき 1 回だけ次のように `fish_add_path` を実行します。

```shell
$ fish_add_path $HOME/.deno/bin
```

これによって、ユニバーサル変数である `fish_user_paths` にコンポーネントを追加します。追加方法はデフォルトで prepend つまり先頭に追加します。上のコードのように新しくパスを追加した場合には `$HOME/.deno/bin` が先頭になりコマンドサーチの最優先として登録されます。

```shell:fish_user_pathsの中身
# 上が先頭 $fish_user_paths[1]
❯ printf '%s\n' $fish_user_paths
/Users/roshi/.deno/bin
/Users/roshi/.volta/bin
/Users/roshi/.cargo/bin
/opt/homebrew/bin
```

`fish_user_paths` はユニバーサル変数であるため、すべての fish セッション間においてこの変数の値は共有され、コンピュータを再起動したりしても永続的に保持されます。実際には、ユニバーサル変数の値は、`~/.config/fish/fish_variables` に保存されており、このファイルは直接編集してはいけません。

さて、`fish_user_paths` 変数に新しいパスを追加されたことで環境変数 `$PATH` にも変更が反映されます。

```shell:PATHの中身
# 上が先頭 $PATH[1]
❯ printf '%s\n' $PATH
/Users/roshi/.deno/bin
/Users/roshi/.volta/bin
/Users/roshi/.cargo/bin
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

これで、パスを通すことができました。逆にパスを消去するには `fish_user_paths` からその要素を削除します。fish の変数はリストになっているのでインデックスを指定して削除できます。削除には `set` ビルトインコマンドの `-e, --erase` オプションを使用します。

```shell
❯ echo $fish_user_paths[1]
/Users/roshi/.deno/bin
# 要素 1 を削除する
❯ set -e fish_user_paths[1]
# 削除された
❯ printf '%s\n' $fish_user_paths
/Users/roshi/.volta/bin
/Users/roshi/.cargo/bin
/opt/homebrew/bin
```

PC を再起動すること `$PATH` に反映されます。

## config.fish に記載する

環境変数だけでなく `PATH` も一緒にファイルで管理したい場合はこちらの方法を行います。`~/.config/fish/config.fish` に `fish_add_path TARGET` を記載するだけです。 

```shell:$HOME/.config/fish/config.fish
# homebrew 用
fish_add_path /opt/homebrew/bin
# Rustup & Cargo 用
set -gx RUSTUP_HOME $HOME/.rustup
set -gx CARGO_HOME $HOME/.cargo
fish_add_path $CARGO_HOME/bin
# Volta 用
set -gx VOLTA_HOME $HOME/.volta
fish_add_path $VOLTA_HOME/bin
# Deno 用
fish_add_path $HOME/.deno/bin
```

fish は起動すると、各種設定ファイルを読みこんで実行しますが、`~/.config/fish/config.fish` はユーザーの指定した初期化処理として最後の方に読み込まれます。

https://fishshell.com/docs/current/index.html#configuration-files

これによって、再ログインや新規シェルセッションの立ち上げ時 `fish_user_paths` に指定パスが保持されていることを保証できます。

`fish_add_path` 関数が賢いのは、指定したコンポーネント(パス)が `$PATH` に存在している場合は再度追加せずに重複を防ぎ、`$PATH` 内のコンポーネント位置を保つことが可能である点です。

上記のコードの例では、パスに何も入れていない状態から fish を立ち上げて場合に、4 つのコンポーネントが上から順番に追加された状態でセッション内の `$PATH` に保持されていることを保証できます。コンポーネントはファイルの上から次のような順番になっています。

- `/opt/homebrew/bin`
- `$CARGO_HOME/bin`
- `$VOLTA_HOME/bin`
- `$HOME/.deno/bin`

これが上からリストへ順番に追加されるので次のように変数内に格納されます(`$HOME` などは実行時に展開される変数です)。

```shell
❯ printf '%s\n' $fish_user_paths
/Users/roshi/.deno/bin # index: 1
/Users/roshi/.volta/bin # index: 2
/Users/roshi/.cargo/bin # index: 3
/opt/homebrew/bin # index: 4
```

パスを削除する場合、fish の立ち上げ時に再びパスが登録されないよう、まず `config.fish` にある `fish_add_paths` の行を削除するかコメントアウトしてください。

```diff shell:config.fish
# homebrew 用
fish_add_path /opt/homebrew/bin
# Rustup & Cargo 用
set -gx RUSTUP_HOME $HOME/.rustup
set -gx CARGO_HOME $HOME/.cargo
fish_add_path $CARGO_HOME/bin
# Volta 用
set -gx VOLTA_HOME $HOME/.volta
fish_add_path $VOLTA_HOME/bin
# Deno 用: コメントアウトしておく
- fish_add_path $HOME/.deno/bin
+ # fish_add_path $HOME/.deno/bin
```

次に、`fish_user_paths` に格納されているパスの値を削除します。`fish_user_paths` はユニバーサル変数なので明示的に要素を削除するか上書きする必要があります。

```shell:コマンドライン
# インデックス 1 の要素を削除する
❯ set -e fish_user_paths[1]
# 削除されたことを確認
❯ printf '%s\n' $fish_user_paths
/Users/roshi/.volta/bin
/Users/roshi/.cargo/bin
/opt/homebrew/bin
```

## コンポーネントの位置を変更
`fish_user_paths` の順番を変更したい場合には、`fish_add_paths` 関数の `-m, --move` オプションを使用します。これによって指定したコンポーネントの位置が新規追加される場所(つまり先頭の位置)に変更されます。残りのコンポーネントはそのままシフトされます。

```shell
❯ printf '%s\n' $fish_user_paths
/Users/roshi/.deno/bin
/Users/roshi/.volta/bin
/Users/roshi/.cargo/bin
/opt/homebrew/bin
❯ fish_add_path -m $HOME/.volta/bin
❯ printf '%s\n' $fish_user_paths
/Users/roshi/.volta/bin
/Users/roshi/.deno/bin
/Users/roshi/.cargo/bin
/opt/homebrew/bin
```

これは実行するたびに位置を変えてしまう可能性があるので、`config.fish` では実行せずに、コマンドラインから直接実行するようにしましょう。

# PATHのアンチパターン

`config.fish` のファイルにおいて次のようにエクスポートやユニバーサルで設定するのはやってはいけません。すべてダメです。細かい理由については後述します。

```shell:config.fish
set -x PATH $HOME/.deno/bin $PATH
set -g PATH $HOME/.deno/bin $PATH
set -gx PATH $HOME/.deno/bin $PATH
set -U PATH $HOME/.deno/bin $PATH
set -Ux PATH $HOME/.deno/bin $PATH
```

まず、前提として `PATH` という環境変数はグローバル変数です。
fish shell の開発者の一人が次のように述べています。

>[ridiculousfish commented on 20 Feb 2013](https://github.com/fish-shell/fish-shell/issues/527#issuecomment-13811988)  
>It strikes me that PATH ought never to be a universal variable, because it must be inherited from the environment. That is, if someone sets PATH in bash and then invokes fish, fish ought to respect that, and not overwrite it a universal value.
>`PATH` はユニバーサル変数ではなくグローバル変数です。`PATH` は環境そのものから継承されるので、ユニバーサル変数であってはならず、bash で `PATH` を設定して fish を呼び出す場合 fish はそのパスを尊重してユニバーサルな値を上書きしないようにします。

そもそも、fish shell は起動時にすべての環境変数を読み込みそれらをグローバル変数に変換しているとのことです。

>[krobelus commented on 11 Dec 2020](https://github.com/fish-shell/fish-shell/issues/7539#issuecomment-742647069)  
>The global PATH is here because when fish starts, it reads all environment variables and turns them into global variables.
(historical reasons.. maybe one day we can merge global and universal scopes).

実際、外部コマンドの `printenv` で表示できる環境変数についてそれぞれ調べてみるとグローバル変数でした。シェル変数は `set -S` を使って、その変数のスコープ、値、エクスポートされているか(環境変数になっているか)を調べることができます。

```shell
❯ set -S SHELL
$SHELL: set in global scope, exported, with 1 elements
$SHELL[1]: |/opt/homebrew/bin/fish|
❯ set -S HOME
$HOME: set in global scope, exported, with 1 elements
$HOME[1]: |/Users/roshi|
❯ set -S LANG
$LANG: set in global scope, exported, with 1 elements
$LANG[1]: |ja_JP.UTF-8|
```

環境変数は慣習的に大文字ですが、上のコードの通り、すべてグローバルスコープで exported されている(つまり、環境変数になっている)ことが分かります。

このように、fish 自体が起動する際に、親プロセス(通常は terminal)から環境変数を継承します。それらには `PATH` やロケール変数などのシステム構成が含まれおり、fish はそれらをグローバル変数として変換してセッションにて保持します。

:::message
シェル変数は "export" することで、環境変数として利用できるようになります。環境変数であることは、ローカル・グローバル・ユニバーサルといったスコープ自体とは関係なく、シェル変数の特定の状態を指します。なので、シェル変数自体はグローバル・ローカル・ユニバーサルのスコープに関わらず、あらゆるスコープでエクスポートできます。スコープによって変わるのは、その環境変数がどのように利用できかという点です。
参考: [fish Documents: How do I set or clear an environment variable?](https://fishshell.com/docs/current/faq.html#how-do-i-set-or-clear-an-environment-variable)

さらに、変数のスコープルールとして "inside out" つまりローカルからグローバル、そして最後にユニバーサルという優先順位でサーチが行われるため、グローバルな環境変数と同一名のユニバーサルな環境変数があった場合、グローバル環境変数が優先されて参照されます。 これにってユニバーサルな環境変数の値を変更してもグローバルな環境変数が存在するため、変更が反映されないという scope shadowing と呼ばれる問題が引き起こされます。
参考: [fish Documents: Why doesn't set -Ux (exported universal variables) seem to work?](https://fishshell.com/docs/current/faq.html#why-doesn-t-set-ux-exported-universal-variables-seem-to-work)

```shell
~> echo $NEWVAR

~> set -gx NEWVAR global ; echo $NEWVAR
global
~> set -Ux NEWVAR universal ; echo $NEWVAR
global
~> set -e NEWVAR; echo $NEWVAR
universal
~> set -e NEWVAR; echo $NEWVAR
```
参考: [zanchey commented on 22 May 2013](https://github.com/fish-shell/fish-shell/issues/806#issuecomment-18267479)

スコープによる挙動の違いは次のようになります。
- シェル変数がグローバルまたはユニバーサルでエクスポートされている場合は、すべての関数がその変数にアクセスでき、それぞれの変数の保持はスコープ通りになる。
- シェル変数がグローバルでもユニバーサルでもないがエクスポートされている場合は、呼び出された関数は関数にローカルなその変数のコピーを受け取る(関数によってそれらの値が変更されたとしてもコピー元の値は変わらない)
参考: [faho commented on 25 Aug 2016](https://github.com/fish-shell/fish-shell/issues/1091#issuecomment-242213527)

シェル変数を "export" するには、`set` ビルトインコマンドの `-x, --export` スイッチを使用します。
:::

環境変数としてユニバーサルでエクスポートするのは、ユニバーサル変数のすべてのセッションにおいて保持されるという性質から、理想的のように思えますが、以下に述べる理由から避けるべきです。

## set -Ux の問題

結論から述べますが、`PATH` という環境変数について、`config.fish` などのファイルに次のように記載すると fish の起動や再ログインのたびに読み込まれるのため、ユニバーサル変数に毎回コンポーネントが追加されることでパスがどんどん長くなっていきます。

```shell:$fish_config_dir/config.fish
set -Ux PATH /opt/homebrew/bin $PATH
# (再)起動のたびにユニバーサル変数が長くなっていく
```

ここでやってること意味は、`PATH` という名前の変数をユニバーサルスコープで定義し、export された状態(つまり環境変数)にし、値 `/opt/homebrew/bin` と `$PATH` をセットということです。`$PATH` では元々の `PATH` の値を展開しているので、つまり、元の `PATH` の値と新しく `/opt/homebrew/bin` の値を追加した値を同時に `PATH` へとセットしています。

例えば、fish を起動した直後に次のような `PATH` だったとします。

```shell
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

この状態で、 `source ~/.config/fish/config.fish` などをやると、もう一度 `set -Ux PATH /opt/homebrew/bin $PATH` が実行されので、次のようになります。

```shell
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

この状態から `exec $SHELL -l` などでシェルを新しくしても再度 `source ~/.config/fish/config.fish` が読み込まれるので `set -Ux PATH /opt/homebrew/bin $PATH` が実行されて次のようになります。

```shell
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/opt/homebrew/bin
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

ここから、ターミナル自体を終了して、もう一度立ち上げたとしても、やはり再度 `source ~/.config/fish/config.fish` が読み込まれるので `set -Ux PATH /opt/homebrew/bin $PATH` が実行されて次のようになります。

```shell
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/opt/homebrew/bin
/opt/homebrew/bin
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

ユニバーサル変数は全シェルセッションで共有され、永続化しているので、あるシェルセッションを開始した際に、`config.fish` が読み込まれると `PATH` の値に `/opt/homebrew/bin` が追加されます。さらに別新しいシェルセッションを開始すると、再び `config.fish` を読み込み、`PATH` に `/opt/homebrew/bin` を追加します。これによって、`PATH` には重複した `/opt/homebrew/bin` がいくつも追加された状態となります。

公式ドキュメントにも以下のようにそのそもユニバーサル変数を `config.fish` に記載しないように注意書きがなされています。
>Do not append to universal variables in config.fish, because these variables will then get longer with each new shell instance. Instead, simply set them once at the command line.  
> [fish documents: More on universal variables](https://fishshell.com/docs/current/language.html#more-on-universal-variables)

ユニバーサル変数のセットは 1 回だけ行えば永続化・伝播するので、ユニバーサル変数を使用する場合には、コマンドラインから行うようにとも書かれていますね。

```shell:コマンドライン
❯ set -Ux EDITOR vim
```

特段 `PATH` については、コマンドラインからでも `set -U` や `set -Ux` を行わない方がいいです。グローバル変数の方が優先される scope shadowing という問題が頻繁に issue でも言及されています。同一名の環境変数がグローバルスコープにあるとユニバーサルスコープの環境変数よりも優先されるため、ユニバーサルスコープで環境変数の値を変更しても設定に反映されないように見える状況に陥ります。

参考: [fish Documents: Why doesn't set -Ux (exported universal variables) seem to work?](https://fishshell.com/docs/current/faq.html#why-doesn-t-set-ux-exported-universal-variables-seem-to-work)

この scope shadowing の問題から、`config.fish` 内において環境変数の定義をする場合には、グローバルスコープでの export `set -gx` の使用を推奨しています。`PATH` 以外の環境変数においては次のように `config.fish` に記載するのが良いです。

```shell:config.fish
set -gX EDITOR vim
```

このように、コマンドラインからの実行と `config.fish` 内での作業が全く異なる行為であることから多くのユーザーにとっての混乱の元凶になっています。さらに、環境変数の中でも `PATH` はかなり特別扱いで、他の環境変数とは違う方法をとらなてはならないので、余計混乱させています。

個人的には、scope shadowing の問題があるので、コマンドラインからも迂闊に `set -Ux` すべきではなく、`set -Ux` の使用はいかなる場合でもやるべきではない思います。これが役に立つケースは同時に起動しているすべてのシェルセッションにおいて、再起動をする必要なく一気に環境変数を変更できるというようなケースに絞られます。

## set -gx の問題

では、`PATH` について `config.fish` でグローバル変数としてエクスポート(`set -gx`)ならいいのかというと、そういう訳でも無いです。

```shell:config.fish
set -gx PATH /opt/homebrew/bin $PATH
```

fish を起動した際に、`config.fish` は読み込まれ上記コードが実行されます。

```shell
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

グローバル変数は現在の fish のセッションに特有なので、そのセッションを終了するまで生存します。そのセッションにおいては、`set -e` で明示的に消去しない限り消えません。`exit` などで終了すれば消滅します。なので、一度ターミナルなどを終了してもう一度立ち上げればグローバル変数は消えています。

一度、終了してから再び `PATH` を見てみると、次のように意図通りの `PATH` になっているはずです。

```shell
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

一見良さそうですが、この状態で、 `source ~/.config/fish/config.fish` などをやると、次のようになります。

```shell
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

つまり、`config.fish` 内の設定を何かしらいじって、そのセッションにおいて読み込ませたい場合 `source` などをすると再び `set -gx PATH /opt/homebrew/bin $PATH` が実行されので、パスが重複登録されるということです。

子プロセスで fish をネストしても、同じ結果になります。上記の `PATH` の状態から `fish` を起動します。

```shell
❯ fish
Welcom to fish shell 3.3.1
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/opt/homebrew/bin
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

環境変数は子プロセスに継承され、さらに fish 立ち上げの際に `config.fish` が読み込まれるはずなので、上記のように `PATH` には `/opt/homebrew/bin` が更に追加された状態になります。

このようなわけで、`config.fish` において `set -gx` で `PATH` の設定は行うべきでありません。

ただし、`set -gx` を使用すべきではないというのは、`PATH` という特殊な環境変数についてのみです。それ以外の環境変数については `set -gx` が推奨されます。

```shell:config.fish
set -gx VOLTA_HOME $HOME/.volta
# ログインのたびに評価されていく
```

`PATH` は以前の `PATH` の値を使用して追加していくというやり方なので、`set` してはならないということです。リストになっていて、前のリストの値を追加するという操作をするのでパスが長くなっていきますが、上記のような処理 `set -gx VOLTA_HOME $HOME/.volta` は `PATH` のように以前のリストの値に追加していくというやり方ではないので OK です。なので例えば `soruce ~/.config/fish/config.fish` のように `source` コマンドで再度読み込んでも大丈夫です。

`PATH` のように元の値に追加していく形式の環境変数は特殊なため `set -gx` を `config.fish` に記載してはいけません。もしそのような環境変数が `PATH` 以外にもあれば `set -gx` を `config.fish` で使用する場合に前の値を使わずにすべての値を書く、またはコマンドラインから `set -Ux` を一度実行して、さらにグローバルスコープに同一名の変数が無いことを保証する必要があります。

## 環境変数や PATH に set -U や set -g はダメなのか

環境変数は、コマンドラインの文脈で言えば、ユーザーにとっては主に外部コマンドが参照するための仕組みです。環境変数は、そのシェルのコンテキスト内で実行されるすべてのコマンドによって継承されます。

一部の外部コマンドはその継承される環境変数に依存しています。例えば、`git` や `man` などの外部コマンドは `$PAGER` 変数からどの pager(テキストスクロールするプログラム)を使用するか判別していますし、`$BROWSER` (使用するブラウザ) や `$LANG` (使用言語)、なども他の外部コマンドの一部で使用されています。外部コマンドでの環境変数の参照は、関数内でシェル変数を参照するとは異なるため環境変数は普通のシェル変数とは異なり export される必要があります。そもそも `set -x` された変数が環境変数という状態になります。

外部コマンド(プログラム)から利用されない、関数内のローカル変数やシェル自体の設定用のグローバル変数やユニバーサル変数は環境変数という状態にする必要はありません。

環境変数という状態にする必要のある変数のみ `set -x` でエクスポートします。

すでにエクスポートされている変数の値について変更する場合には、`-x` を明示する必要はありません。エクスポートにはスコープのルールと同様にエクスポートルールというものがあります。

- 変数がエクスポートされるかエクスポートされないよう明示的に設定されている場合、その設定が優先される
- 変数が明示的にエクスポートまたはエクスポートされないよう設定されていないが、以前に定義されている場合、変数の以前のエクスポートルールが保持される
- そうでなければ、デフォルトで変数はエクスポートされない
- 変数がローカルスコープで、エクスポートされる場合、呼び出される関数はそのコピーを受け取るため、関数が戻ると、変数に加えられた変更はすべて消える
- グローバル変数は、エクスポートされているかどうかに関係なく、**関数から**アクセスできる。

https://fishshell.com/docs/current/cmds/set.html

`set -S` で変数のスコープとエクスポートされているか分かりますが、次のように `-x` なしで値を変更してもエクスポートされたままであることがわかります。

```shell
❯ set -S LANG
$LANG: set in global scope, exported, with 1 elements
$LANG[1]: |ja_JP.UTF-8|
❯ set LANG en_US
❯ set -S LANG
$LANG: set in global scope, exported, with 1 elements
$LANG[1]: |en_US|
```

# ユニバーサル変数の廃止?
少し話が脱線しますが、scope shadowing やら、環境変数との混同からそもそも、ユニバーサル変数を廃止または別のメカニズムで置き換えようという議論もあります。

https://github.com/fish-shell/fish-shell/issues/7317
https://github.com/fish-shell/fish-shell/issues/7379

開発者の一人である ridiculousfish 氏がユニバーサル変数のスコープを除去し、ユニバーサル変数をグローバル変数にするというプルリクエストを投下していました(結局は棄却というか、自分で取り下げていましたが)。
https://github.com/fish-shell/fish-shell/pull/8455


ユニバーサル変数の問題点と環境変数についての会話抜粋

>[ridiculousfish commented on 19 Nov 2021](https://github.com/fish-shell/fish-shell/pull/8455#issue-1057599268)  
>This PR addresses the "scope confusion" aspect of universal variables, as discussed in [#7317](https://github.com/fish-shell/fish-shell/issues/7317) and [#7379](https://github.com/fish-shell/fish-shell/issues/7379). It is a major change, so would go in after 3.4.0 release.
>Today, there is a universal variable scope above the global scope, which causes confusion:
>
>1. A global variable may shadow a universal variable, with a different value.
>2. Calling set at top level will modify a universal variable if it exists, otherwise create a global. It's not always obvious which one you'll get.
>3. If you export a universal variable, then nested fish shell instances will inherit an environment variable, which becomes a global. This is so serious that we have a hack to prevent the common case.
>
>このプルリクエストは [#7317](https://github.com/fish-shell/fish-shell/issues/7317) と [#7379](https://github.com/fish-shell/fish-shell/issues/7379) で議論されているユニバーサル変数のおける"スコープの混乱"という問題について取り組んだものです。
>現在、ユニバーサル変数のスコープはグローバルスコープの上にあり、これによって以下の混乱が引き起こされています。
>
>1. 値がそれぞれ異なる場合に、グローバル変数がユニバーサル変数をシャドーする可能性がある。
>2. `set` をトップレベルで呼び出す際にユニバーサル変数が存在しているとそれを修正し、そうでなければ、グローバル変数として作成する。どちらになるかというのは、必ずしも明確ではありません。
>3. ユニバーサル変数をエクスポートした場合、ネストされた fish shell のインスタンスは環境変数を継承しますが、それらはグローバル変数になります。これは非常に深刻なので、一般的なケースを防ぐためのハック方法があるほどです。  


>[andmis commented on 19 Nov 2021](https://github.com/fish-shell/fish-shell/pull/8455#issuecomment-973170023)  
>Have you considered making it illegal to set universal variables as exportable, and warning when launching a fish instance if there is an environment variable whose name collides with that of a universal variable?
>
>ユニバーサル変数を export 可能であるものとしてセット出来るの禁止して、変数名がユニバーサル変数名と衝突している環境変数がある場合に fish インスタンスの起動時に警告を出すようにしてみるのはどうですか?

>[ridiculousfish commented on 19 Nov 2021](https://github.com/fish-shell/fish-shell/pull/8455#issuecomment-973299305)  
>Exported universal variables are very useful, e.g. `$EDITOR`, so we certainly want to preserve that.
>
>例えば `$EDITOR` など、エクスポートされたユニバーサル変数をはとても役に立つ。なのでこの機能は維持したいと思っています。

>[andmis commented on 19 Nov 2021](https://github.com/fish-shell/fish-shell/pull/8455#issuecomment-973359992)  
>Interesting -- I do set `-gx EDITOR vim` in my `config.fish`. But I made that decision a long time ago before I really knew my way around fish, and particularly before I even understood what universal variables are. Is there a reason to make things like `EDITOR`, `PYTHONPATH`, etc., universal instead of setting them as `-gx` in `config.fish`?
>
>興味深いですね。私の場合は `config.fish` 内にて `set -gx EDITOR` を行っています。しかし、この決定をしたのは私が fish の仕組みについてよく知るずっと以前のことで、なんならユニバーサル変数がどんなものなか理解する前でしたよ。`EDITOR` や `PYTHONPATY` といったものについて `config.fish` で `-gx` として設定するのではなく、ユニバーサルにしてしまうことに何らかの理由があるのですか?

>[kopischke commented on 19 Nov 2021](https://github.com/fish-shell/fish-shell/pull/8455#issuecomment-973868803)  
>Changing global state. Changes to global variables set in `config.fish` need that file to be edited and only take effect when it is read, i.e. effectively when (re-)launching your shell. Changes to universal variables only need a `set` command to be applied to all running instances of fish. I think the utility of that is obvious, but in practice there are scope shadowing issue between universal and global variables, which is why uvars being replaced by another mechanism is being discussed.
>
>グローバル変数の状態の変更が問題です。`config.fish` 内でセットされたグローバル変数を変更するためには、ファイルを編集する必要がありますし、そのファイル読み込まれて初めて変更が反映されます。つまり、シェルを(再)起動したときにのみ変更が有効になります。ユニバーサル変数への変更では、すべての実行中の fish インスタンスに変更が適用されるのに必要なのは `set` コマンドを実行するだけです。この利便性については明らかだと思いますが、現実的にはユニバーサル変数とグローバル変数の間においてスコープシャドーイングの問題があり、そういうわけでユニバーサル変数(uver)は別のメカニズムに置き換えられるべきかの議論が行われています。


他の issue も含めて全体的に見ると、結局のところユニバーサル変数が取り除かれることは無さそうな印象を受けました。結論としては、ユニバーサル変数は維持されそうです。

ユニバーサル変数のエクスポートにもそれなりに利点があるそうですが、個人としても、他の人の記事とかを見ても `config.fish` で管理したい人が多いって印象でした。

# 結局どうすればよいか

結論として、`config.fish` で管理する場合には、冒頭で述べたように `fish_add_path` 関数を使用し、`PATH` 以外の環境変数には `set -gx` を使用しましょう。

コマンドラインからでも、`config.fish` においても `PATH` については fish 側から `fish_add_path` というすばらしい関数と、`fish_user_paths` というユニバーサル変数を組み合わせたメカニズムが提供されており、これを利用します。

```shell:config.fish
# 通常の環境変数については set -gx
set -gx VOLTA_HOME $HOME/.volta
# PAHT 環境変数については fish_add_path 関数
fish_add_path $VOLTA_HOME/bin
```

`fish_add_path` は `PATH` 内のコンポーネントが重複しないように一度だけ `PATH` に対象コンポーネントを追加するので `set -Ux` や `set -gx` のように何回もパスを重複追加することを避けられます。

コマンドラインから使用する場合には、一度だけ `fish_add_path` を実行すればよいです。`fish_add_path` 関数を実行することで、ユニバーサル変数 `fish_user_paths` にパスが追加され永続化されます。

注意点として、`fish_user_paths` はユニバーサル変数で元々定義されているのでグローバルで宣言してはいけません。

```shell
❯ set -S fish_user_paths
$fish_user_paths: set in universal scope, unexported, with 4 elements
$fish_user_paths[1]: |/Users/roshi/.cargo/bin|
$fish_user_paths[2]: |/Users/roshi/.volta/bin|
$fish_user_paths[3]: |/Users/roshi/.deno/bin|
$fish_user_paths[4]: |/opt/homebrew/bin|
```

例えば、遊びで `set -g fish_user_paths` みたいなことをやれば scope shadowing でそちらが優先されので、`fish_add_path` をやっても意味がなくなってしまうので、グローバルスコープから削除してください。`set -e` で削除する際にはスコープを明示すればよいです。

```shell
❯ set -e -g fish_user_paths
```

スコープを明示しない場合には、local → global → universal というスコープの順番で先にヒットしたものを削除します。

参考: [Feature Request: 'set --erase' for multiple/all scopes · Issue #7711 · fish-shell/fish-shell](https://github.com/fish-shell/fish-shell/issues/7711)


