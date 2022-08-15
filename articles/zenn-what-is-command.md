---
title: "fish shellにおける関数・ビルトイン・外部コマンドの実体"
emoji: "🐚"
type: "tech"
topics: [fish, shell, UNIX, macOS, Homebrew]
published: true
date: 2022-02-23
url: "https://zenn.dev/estra/articles/zenn-what-is-command"
aliases: [記事_fish shellビルトインと外部コマンドの実体]
tags: " #type/zenn #shell/fish  "
---

# はじめに

https://zenn.dev/estra/articles/zenn-source-fish-plugin

上記の記事で、コマンドラインに入力したコマンドが、実際に実行される際に呼び出されるものは以下の３つのカテゴリーのいづれかであるということを説明しました。

- **Function** (関数): 他のコマンドをグルーピングし、名前を付けて実行できるようにして定義したもの
- **Builtin** (内部コマンド: **Internal command**): シェルから見て内部にあるコマンドで、fish shell のプログラム自体に組み込まれて提供されている
- **External command** (外部コマンド): シェルから見て外部にあるコマンドで、fish shell 自体とは関係のないプログラム

また、コマンドラインからコマンドが呼び出される際には次のような優先順位がありました。

```
Function > Builtin > External command
```

しかし、これらの「実体は何なのか」ということがまだ不明だったのでそれぞれについて追加で調べてみました。この記事は、自分用のまとめと**実際にコマンドを使用した調査**などを含むので長くなります。お急ぎの方は最後の[まとめ](#まとめ)の項目をご覧ください。

※ この記事では、次のような自分の環境を前提に話を進めますので注意してください。
- OS: macOS
- hardware: arm64 (Apple Silicon)
- fish: version 3.3.1 (Homebrew でインストール)

:::details changelog
- 2022/02/28
  - シンボリックリンクと Homebrew についての記述を追加、それに応じて外部コマンドの記載を修正
  - UNIX command についての記述を追加
- 2022/03/01
  - Homebrew そのものについての記述を追記
  - 環境について明言化
  - function と completion のサーチ対象となる特殊変数についての記述を追加
  - External command の区分けについて記述を追加
:::

# Function

Function(関数)は、コマンドラインから最優先で検索されるものになります。
自作したものは、基本的には `$__fish_config_dir` に登録されているディレクトリ(`~/.config/fish/`)で管理されていますが、fish shell 側から提供されている関数がいくつかあります。

`type` ビルトインの `-p, --path` オプションを使用すると function や extetrnal command のパスを表示します。 `man` function の定義場所のパスを調べてみます。

```shell
❯ type -p man
/opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/man.fish
```

自分の環境では、`/opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/man.fish` です。したがって、`man` のような fish shell 側から提供されている function は `/opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/` に定義されていることが分かります。`ls` コマンドなどで何が入っているか確認できます。実体は、このディレクトリ内の fish shell script です。

```shell
❯ ls /opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/
N_.fish
__fish_abbr_old.fish
__fish_any_arg_in.fish
__fish_anypython.fish
__fish_append.fish
__fish_apropos.fish
__fish_cancel_commandline.fish
__fish_commandline_is_singlequoted.fish
__fish_complete_atool_archive_contents.fish
__fish_complete_bittorrent.fish
__fish_complete_blockdevice.fish
__fish_complete_cd.fish
# 長いので省略
```

これらの関数には以下のようなものが含まれています。

- builtin のラッパー: `cd.fish` など
- external command のラッパー : `man.fish` など
- ユーティリティ : `help.fish`(内部的には `open` や `xdg-open` などの external command を使用している)
- fish shell の補助関数 : `__fish` から始まるやつ
- fihs shell の設定を変更するための関数 : `fish_` から始まるやつ

数えてみると、216 個ありました。

```shell
❯ ls /opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/ | count
216
```

Github で公開されているソースコードからも見られるので興味があれば見てみてください。
https://github.com/fish-shell/fish-shell/tree/master/share/functions

これらの function が定義されている `/opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/` というディレクトリは `fish_function_path` というグローバル変数に登録された検索対象ディレクトリとなっているため、ここに定義されている function が使用できるようになっています。`~/.config/fish/` 以外に登録されている `/opt/homebrew/` のディレクトリについては後で説明する Homebrew というパッケージマネージャーによって作成されたディレクトリです。

```shell
❯ printf '%s\n' $fish_function_path
/Users/roshi/.config/fish/functions
/opt/homebrew/Cellar/fish/3.3.1/etc/fish/functions
/opt/homebrew/Cellar/fish/3.3.1/share/fish/vendor_functions.d
/opt/homebrew/share/fish/vendor_functions.d
/opt/homebrew/Cellar/fish/3.3.1/share/fish/functions
```

completion についても同じようにグローバル変数 `$fish_complete_path` に登録された検索対象ディレクトリがあり、ここに配置されている fish ファイルが completion として読み込まれています。

```shell
❯ printf '%s\n' $fish_complete_path
/Users/roshi/.config/fish/completions
/opt/homebrew/Cellar/fish/3.3.1/etc/fish/completions
/opt/homebrew/Cellar/fish/3.3.1/share/fish/vendor_completions.d
/opt/homebrew/share/fish/vendor_completions.d
/opt/homebrew/Cellar/fish/3.3.1/share/fish/completions
/Users/roshi/.local/share/fish/generated_completions
```

# Builtin

Builtin(ビルトインコマンド)は、コマンドラインから Function の次に検索されるものです。数えてみると 59 個しかありません。

```shell
❯ builtin -n | count
59
```

ビルトインコマンドは文字通りビルトインなので、`and`, `argparse`, `for` などのビルトインコマンドはこの `fish` プログラム(`/opt/homebrew/bin/fish`)自体に入っているはずです。こららのビルトインコマンドのソースコードは Github 上の fish shell のリポジトリで確認できます。

https://github.com/fish-shell/fish-shell/tree/master/src

例えば、`argparse` ビルトインはソースコードとして `builtin_argparse.cpp` という C++ のファイルがあります。これがビルトインの実体です。

https://github.com/fish-shell/fish-shell/blob/master/src/builtin.cpp

:::message
ちなみに、bash において、 Builtin とはその機能が個別のユーティリティを介して得るのに不可能であったり不便である、という理由で shell 自体から提供されているコマンドです。例えば、`cd`, `break` などのコマンドは shell そのものを直接操作すため shell の外側から実装が不可能であり、その結果 Builtin として提供されているとのこと。また `kill` や `pwd` などのコマンドが Builtin として提供されているのは、 shell から分離されたユーティリティとして実装できるが Builtin として利用したほうが便利だからという理由だそうです。

> Shells also provide a small set of built-in commands (_builtins_) implementing functionality impossible or inconvenient to obtain via separate utilities. For example, `cd`, `break`, `continue`, and `exec` cannot be implemented outside of the shell because they directly manipulate the shell itself. The `history`, `getopts`, `kill`, or `pwd` builtins, among others, could be implemented in separate utilities, but they are more convenient to use as builtin commands.
- [Bash Reference Manual](https://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html#What-is-Bash_003f) より引用

これは fish shell に置き換えても同じことが言えるのではないでしょうか。実際、fish shell においても `cd`、`break` は Builtin として提供されていますし、`kill` は External command のラッパーfunction、`pwd` は Builtin として提供されています。
:::

# External command

External command(外部コマンド)は、コマンドラインから Builtin の次に検索されるものです。「コマンド」と呼ばれるものの大多数がこの Exteranl command になります。`/usr/bin` に配置されているコマンドだけでも 1102 個あります。

```shell
❯ ls /usr/bin | count
1102
```

:::message
個人的体験談ですが、外部コマンドについては OS に元々同梱されているコマンドと新しくインストールできるコマンド(パッケージ)があるので理解しづらい部分が多々ありました。そもそも External command の明確な定義や区分けがしづらいのですが、次のように分解してみると理解しやすいと思います(※ 正確な定義などではないので、注意してください)。

```md
External command: コマンドラインシェルから名前で呼び出せるプログラムすべて(`PATH` 環境変数に登録されているディレクトリに配置されたコマンドサーチの対象となるプログラム)
├── Included command: OS 同梱のコマンド
│   ├── UNIX command: UNIX の仕様に記載されているコマンド(macOS や Linux などの UNIX-like OS で見られる)
│   │   ├── BSD実装: macOS などの BSD 系の実装
│   │   └── GNU実装: 多数の Linux ディストリビューションで見られる GNU の実装(いわゆる Linux コマンド)
│   └── その他(大多数)
└── NOT included command: OS に同梱されていない自分でインストールしたコマンド
    ├── Newly installed command: Homebrew や Cargo などのパッケージマネージャーを使用してインストール
    └── Self made command: 自作したもの
```

Windows については UNIX 系 OS ではないのでまた違ったようになると思います。
:::

External command の実体については `file`, `which`, `cat` などといった UNIX コマンド(これらも External command)を使用して調べることができました。
`file` コマンドは指定したファイルについてのテストを行って、ファイルの種類を特定します。

https://www.wikiwand.com/ja/File_(UNIX)

:::message
[UNIX command](https://www.wikiwand.com/en/List_of_Unix_commands) は Single UNIX Specification (SUS) と呼ばれる UNIX の仕様の一部である IEEE Std 1003.1-2008 に定義されている外部コマンドです。Linux や macOS といった [UNIX-like OS](https://www.wikiwand.com/en/Unix-like) のマシンに元々同梱されています。macOS は BSD と呼ばれる OS から派生しているので `man COMMANDNAME` したときにデフォルトで表示されるマニュアルは "BSD General Commands Manual" です。いくつかの Linux ディストリビューションで配布されている UNIX command は GNU 実装なので macOS のコマンドとは挙動やオプションが異なることあります。
:::

いくつかのコマンドについて、`which` でそれぞれのファイルパスを取得して、`file` で調べてみます(`which` はビルトインコマンドの `command -s` で代用できます)。

```shell
❯ file (which cd)
/usr/bin/cd: POSIX shell script text executable, ASCII text
❯ file (which file)
/usr/bin/file: Mach-O universal binary with 3 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64:Mach-O 64-bit executable arm64] [arm64e:Mach-O 64-bit executable arm64e]
/usr/bin/file (for architecture x86_64): Mach-O 64-bit executable x86_64
/usr/bin/file (for architecture arm64): Mach-O 64-bit executable arm64
/usr/bin/file (for architecture arm64e): Mach-O 64-bit executable arm64e
❯ file (which fish)
/opt/homebrew/bin/fish: Mach-O 64-bit executable arm64
```

ということで、上記外部コマンドのプログラムファイルの種類は次のようになってることが分かります。

- `cd` : POSIX shell script text executable
- `file` : Mach-O universal binary with 3 architectures
- `fish` : Mach-O 64-bit executable arm64

:::message
`fish` プログラム(`/opt/homebrew/bin/fish`)は fish shell そのものなので fish 使用時にはシェルの外部にあるコマンド(External command)とは言えませんが、bash など他のシェルを起動しているときに呼び出す際には bash から見れば外側にあるコマンドなので Exteranl command といえるでしょう。
:::

`cd` コマンド(`/usr/bin/cd`)は "POSIX shell script text executable" 、つまり shell script らしいので `cat` でファイルの中身をみてみると次のようになっていることが分かります。

```shell
❯ cat (which cd)
#!/bin/sh
# $FreeBSD: src/usr.bin/alias/generic.sh,v 1.2 2005/10/24 22:32:19 cperciva Exp $
# This file is in the public domain.
builtin `echo ${0##*/} | tr \[:upper:] \[:lower:]` ${1+"$@"}
```

シバンが `#!/bin/sh` とあるので、sh をインタプリタにして実行されるはずです。

https://www.wikiwand.com/ja/%E3%82%B7%E3%83%90%E3%83%B3_(Unix)

とは言っても、fish を使っていれば実際にこの `/usr/bin/cd` が使用されることはありません。fish ではビルトインの `cd` が提供されており、`builtin -n` コマンドを使って全ビルトインを確認すれば `cd` がビルトインとして提供されていることが分かります。結局、コマンドラインに `cd` と入力して実行すると、`/usr/bin/cd` は使用されず、同一名で存在している function の `cd` が呼び出されます(`type cd` を実行すれば分かりますが、直接呼び出されるのはビルトインの `cd` をラップした function なので注意してください)

fish を使っていない場合でも、例えば macOS にデフォルトで入っている bash シェルを使ったとしてもこの `/usr/bin/cd` というスクリプトが呼び出されることはありません。もちろん、bash シェルにも builtin の cd が入っているからです。

一方、このシェルスクリプト自体は sh のビルトインの `cd` を呼び出すというものですが、ある理由から実際にはディレクトリを変更しません。

```shell
# 引数にディレクトリを指定しても何も起こらない
❯ /usr/bin/cd articles/
# 引数に存在しないディレクトリを指定するとエラーが出力される
❯ /usr/bin/cd wat/
/usr/bin/cd: line 4: cd: wat/: No such file or directory
```

この、`/usr/bin/cd` についての解説は stackoverflow にあったので興味があれば見てみてください。

https://stackoverflow.com/questions/38776286/can-someone-explain-the-source-of-the-cd-shell-command

このようなバイナリでない外部コマンドがいくつかあるようなので調べてみたところ、例えば `/bin` ディレクトリに格納されている外部コマンドはすべてバイナリでした。

```shell
❯ count (file (which (ls /bin)) | grep "POSIX shell script text executable")
0
```

`/usr/bin` については 1102 個のプログラムが格納されていましたが、そのうち"POSIX shell script text executable"であるものは 75 個でした。

```shell
❯ count (file (which (ls /usr/bin)) | grep "POSIX shell script text executable")
75
```

例えば、`/usr/bin/type` などもこの種類のものでした。

```shell
❯ cat (which type)
#!/bin/sh
# $FreeBSD: src/usr.bin/alias/generic.sh,v 1.2 2005/10/24 22:32:19 cperciva Exp $
# This file is in the public domain.
builtin `echo ${0##*/} | tr \[:upper:] \[:lower:]` ${1+"$@"}
```

通常のコマンド呼び出しで利用されない、このようなファイルの存在理由については、日本語の記事で解説しているものがあってのでそちらを参照してください。

https://atmarkit.itmedia.co.jp/ait/articles/1112/26/news118_2.html

"Mach-O universal binary" というのは調べてみたところ、自分の環境である M1-mac 上で動くバイナリフォーマットのプログラムであり、「単一ファイルに複数のバイナリを収録できる」という構造を持っているとのことです。また、"with 3 architectures" とは `file (which file)` の結果を見てわかるとおり、`x86_64`, `arm64`, `arm64e` という３つのアーキテクチャに対応したプログラムであることを示しています。

>Macユーザとして気になるのは、ARMアーキテクチャへのスムーズな移行だろうが、結論からいえば「ほぼ問題なからん」となる。MacOS 9より前、いわゆるClassic MacOSのときに行われた68KからPowerPCへの移行はさておき(連載開始前でありフォローしていない)、Mac OS X登場以降2度にわたり行われたアーキテクチャ移行では特段の問題は生じなかった。
>
>その理由は「Mach-O(マーク・オー)」にある。Mach-Oとは、現macOSの源流であるNEXTSTEP/OPENSTEPに採用されたバイナリフォーマットで、いまなおmacOSはもちろんiOSやiPad OSで利用されている。かんたんにいうと、このMach-Oは「単一ファイルに複数のバイナリ(プログラムの実行部分)を収録できる」構造を持ち、外観上は1つのアプリ/コマンドであっても異種アーキテクチャで動作する。
- [この道はいつか来た道、Macのプラットフォーム移行を助ける「Mach-O」と「Rosetta」](https://news.mynavi.jp/article/osxhack-266/) より引用

次に `file` プログラム(ファイルパスは `which file`) や `fish` プログラム(ファイルパスは `which fish`) を `cat` すると謎の文字列が出来きますが、これはバイナリファイルを `cat` したからです。バイナリファイルの中身を見るには `od` という UNIX コマンドを使用します。

```shell
❯ cat (which fish)
# 謎の文字列が出てきます(実際はfileプログラムはC言語で書かれているのでコンパイルされたバイナリ)

# od -x で 16進数表示でみてみる
❯ od -x (which fish)
0000000      facf    feed    000c    0100    0000    0000    0002    0000
0000020      0014    0000    0860    0000    8085    00a1    0000    0000
0000040      0019    0000    0048    0000    5f5f    4150    4547    455a
0000060      4f52    0000    0000    0000    0000    0000    0000    0000
0000100      0000    0000    0001    0000    0000    0000    0000    0000
0000120      0000    0000    0000    0000    0000    0000    0000    0000
0000140      0000    0000    0000    0000    0019    0000    0228    0000
# 以下省略
```

ちなみに `file` プログラムのソースコードは Github で CSV リポジトリの Read-only mirror として公開されています。C 言語製のようです。
https://github.com/file/file

## PATH環境変数

[外部コマンド](https://www.wikiwand.com/ja/%E5%A4%96%E9%83%A8%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89)の実体は実行可能([executable](https://www.wikiwand.com/ja/%E5%AE%9F%E8%A1%8C%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB))なバイナリファイルや shell script ファイルであることが分かりました。

もちろんそういったファイルのすべてが Exteranl command として認識されているわけではありません。あるプログラムをインストールして、ファイルパスを指定せずに  shell から External command として呼び出せるようにするにはコマンド検索用の[環境変数](https://www.wikiwand.com/en/Environment_variable)に登録する必要があります。

> Every program on your computer can be used as a command in fish. **If the program file is located in one of the PATH directories, you can just type the name of the program to use it**. Otherwise the whole filename, including the directory (like /home/me/code/checkers/checkers or ../checkers) is required.
- [The fish language — fish-shell 3.3.1 documentation](https://fishshell.com/docs/current/language.html) より引用

シェルでは、`PATH` という環境変数に External command の配置されている各ディレクトリのパスが格納されています。シェルはこの環境変数を見て、コマンドラインから入力されたコマンドを検索し、呼び出しています。

> $PATH is an environment variable **containing the directories that fish searches for commands**. Unlike other shells, $PATH is a list, not a colon-delimited string.
- [Tutorial — fish-shell 3.3.1 documentation](https://fishshell.com/docs/current/tutorial.html#path) より引用

OS に元々同梱されていない新しい外部コマンドをターミナルなどから使えるようにするにはこの `PATH` にそのコマンドがある場所(ディレクトリパス)を追加する、いわゆる「**パスを通す**」ことが必要となります。他のシェルでは、この `PATH` はコロンで区切られたものとなっていますが、fish の `PATH` はリストになっています。

```shell
❯ echo $PATH
/Users/roshi/.deno/bin /Users/roshi/.nodebrew/current/bin /opt/homebrew/bin /usr/local/bin /usr/bin /bin /usr/sbin /sbin
```

自分の環境では、上記のパスに登録されているディレクトリ内から外部コマンドを検索し、呼び出します。

:::message
ちなみに、`printenv` という外部コマンドを引数無しで実行する `PATH` を含めた全ての環境変数(`HOME`、`PWD`、`USER` など)を閲覧できます。`printenv` で出力した `PATH` はコロン区切りになっています。

```shell
❯ printenv PATH
/Users/roshi/.deno/bin:/Users/roshi/.nodebrew/current/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```
:::

fish shell ではパスを通すには `fish_add_path` という function を使用することで簡単に対象ディレクトリを追加できます。

```shell
❯ fish_add_path $HOME/.deno/bin
❯ fish_add_path $HOME/.nodebrew/current/bin
❯ fish_add_path /opt/homebrew/bin/
```

パスの通し方と `PATH` 環境変数については以下の記事でかなり詳細に調べたので興味のある方は確認してください。

https://zenn.dev/estra/articles/zenn-fish-add-path-final-answer

このようにしてユーザーが追加したパスは `fish_user_paths` という[fishの特殊変数](https://fishshell.com/docs/current/language.html#special-variables)から確認できます。

```shell
❯ echo $fish_user_paths
/Users/roshi/.deno/bin /Users/roshi/.nodebrew/current/bin /opt/homebrew/bin
```

上記以外のパスはマシン購入時に最初から登録されているものがほとんどです。

`PATH` は環境変数ですが、環境変通とはそもそも環境(親プロセス)から継承されることによって fish shell のセッションにてグローバル変数として保持されます。継承元は、たいていは terminal だったりします。

親プロセスのどこかで、`/etc/paths` ファイルが読み込まれており、ここに fish 起動時の最初から登録されているパスが記載されています。

```shell
❯ cat /etc/paths
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

これらのディレクトリには以下のように外部コマンドが分類されて配置されています。

- `/usr/local/bin` : 自分で追加インストールした外部コマンドを配置する(後述しますが、Intel 製 mac なら homebrew でインストールしたパッケージのシンボリックリンクが配置されるはずです)
- `/usr/bin` : UNIX command の多くや、ユーザーが使用するほとんどの OS 同梱の外部コマンドが配置されている(自分の環境では、1102 個のプログラムが配置されている)
- `/bin` : ごく基本的コマンドが配置されている(自分の環境では、36 個のプログラムが配置されている)
- `/usr/sbin` : 管理コマンドが配置されている(自分の環境では、230 個のプログラムが配置されている)
- `/sbin` : 管理システムコマンドが配置されている(自分の環境では、63 個のプログラムが配置されている)

例えば、`/bin` に配置されているコマンドは以下のようなものです。

```shell
❯ ls /bin
[         csh       echo      ksh       mkdir     rm        sync      zsh
bash      dash      ed        launchctl mv        rmdir     tcsh
cat       date      expr      link      pax       sh        test
chmod     dd        hostname  ln        ps        sleep     unlink
cp        df        kill      ls        pwd       stty      wait4path
```

呼び出される外部コマンドは、`PATH` 環境変数に新しく登録されているディレクトリパスの順番に探索されます。`echo` した際に左側が新しく追加したものなので、自分の環境では、`$HOME/.deno/bin` から先に検索され、最後に `/sbin` という順番になります。

```shell
❯ echo $PATH
/Users/roshi/.deno/bin /Users/roshi/.nodebrew/current/bin /opt/homebrew/bin /usr/local/bin /usr/bin /bin /usr/sbin /sbin
```

なので、例えば `foo` という存在しないコマンドを入力して実行した際には、`foo` という名前の Fuction を探し、次に `foo` という Builtin を探します。見つからないので、外部コマンドから探しますが、PATH に登録されている上記の順番にコマンドサーチが実行された結果、どのディレクトリにも見つけられなかったため、`fish: Unknown command: foo` というエラーメッセージが出力されます。

```shel
❯ foo
fish: Unknown command: foo
```

もし、同一名の外部コマンドが複数あった場合には(例えば、新しいバージョンの同一コマンドを `/usr/local/bin` や `/opt/homebrew/bin` に配置した場合)、新しく追加されたディレクトリパスへ配置されている方が先に検索されるので、そちらが使用されます。

例えば、自分の環境では `/usr/bin/` に入っている `python3` という外部コマンドの最新バージョンを Homebrew を使ってインストールしました。この際に `python3` はマシンに２つはいっていることになります。複数の同一名の外部コマンドが入っている場合、`command` ビルトインコマンドの `-a` オプションですべてを閲覧できます(`which -a python3` でも同じ結果になります)。

```shell
❯ command -a python3
/opt/homebrew/bin/python3
/usr/bin/python3
❯ which -a python3
/opt/homebrew/bin/python3
/usr/bin/python3
```

`/opt/homebrew/bin/python3` と `/usr/bin/python3` にこのコマンドがあることが分かります。ですが、`type` を使ってコマンドラインで `python3` がどのように解釈されるのかをみてみると次のようになります(`which python3` でも同じような結果となります)。

```shell
❯ type python3
python3 is /opt/homebrew/bin/python3
❯ which python3
/opt/homebrew/bin/python3
```

つまり、`python3` を実行しようとすると、最新バージョンである `/opt/homebrew/bin/python3` が実行されます。これは Homebrew 用に通したパス `/opt/homebrew/bin` のほうが `/usr/bin/ ` よりも後から追加されたものであるため、そちらが先に検索対象となるからです。

このように `/usr/bin` に配置されているコマンドと同一名の homebrew でインストールされているコマンドを探してみると結構あることがわかりました。これはインストールしたパッケージの依存関係に入っていたと思われます。

```shell
❯ file (which (ls /usr/bin)) | grep "/opt/homebrew/"
/opt/homebrew/bin/addftinfo:                 Mach-O 64-bit executable arm64
/opt/homebrew/bin/afmtodit:                  Perl script text executable
/opt/homebrew/bin/eqn:                       Mach-O 64-bit executable arm64
/opt/homebrew/bin/gdiffmk:                   Bourne-Again shell script text executable, ASCII text
/opt/homebrew/bin/grn:                       Mach-O 64-bit executable arm64
/opt/homebrew/bin/grodvi:                    Mach-O 64-bit executable arm64
/opt/homebrew/bin/groff:                     Mach-O 64-bit executable arm64
/opt/homebrew/bin/groffer:                   Perl script text executable
/opt/homebrew/bin/grog:                      Perl script text executable
/opt/homebrew/bin/grolbp:                    Mach-O 64-bit executable arm64
/opt/homebrew/bin/grolj4:                    Mach-O 64-bit executable arm64
/opt/homebrew/bin/grops:                     Mach-O 64-bit executable arm64
/opt/homebrew/bin/grotty:                    Mach-O 64-bit executable arm64
/opt/homebrew/bin/hpftodit:                  Mach-O 64-bit executable arm64
/opt/homebrew/bin/indxbib:                   Mach-O 64-bit executable arm64
/opt/homebrew/bin/lkbib:                     Mach-O 64-bit executable arm64
/opt/homebrew/bin/lookbib:                   Mach-O 64-bit executable arm64
/opt/homebrew/bin/mmroff:                    Perl script text executable
/opt/homebrew/bin/neqn:                      POSIX shell script text executable, ASCII text
/opt/homebrew/bin/nroff:                     POSIX shell script text executable, ASCII text
/opt/homebrew/bin/pfbtops:                   Mach-O 64-bit executable arm64
/opt/homebrew/bin/pic:                       Mach-O 64-bit executable arm64
/opt/homebrew/bin/pip3:                      Python script text executable, ASCII text
/opt/homebrew/bin/post-grohtml:              Mach-O 64-bit executable arm64
/opt/homebrew/bin/pre-grohtml:               Mach-O 64-bit executable arm64
/opt/homebrew/bin/python3:                   Mach-O 64-bit executable arm64
/opt/homebrew/bin/refer:                     Mach-O 64-bit executable arm64
/opt/homebrew/bin/soelim:                    Mach-O 64-bit executable arm64
/opt/homebrew/bin/tbl:                       Mach-O 64-bit executable arm64
/opt/homebrew/bin/tfmtodit:                  Mach-O 64-bit executable arm64
/opt/homebrew/bin/troff:                     Mach-O 64-bit executable arm64
```

参考:
https://qiita.com/tk3fftk/items/8b389c0e4b1f9c64ebe3
https://kinacom.hatenablog.jp/entry/2016/06/29/180854

## シンボリックリンクとHomebrew
:::message alert
`fish` プログラムは `opt/homebrew/bin/fish` であるかと思いきや、実際にはそうではなかったことが判明したので追記いたしました。
:::

シンボリックリンク(ソフトリンク)とは絶対パスまたは相対パスの形式で別のファイルやディレクトリへの参照を含み、パス名の解決に影響を与えるファイルの用語です。

https://www.wikiwand.com/en/Symbolic_link

実はいくつかの外部コマンドはこのシンボリックリンクによって別の場所にあるファイルを参照しており、コマンドラインから呼び出す際にはそのファイルが指し示すリンク先のバイナリファイルなどを実行していたようです。

問題である `which fish` にて得られたファイル `/opt/homebrew/bin/fish` は、このシンボリックリンクになっていました。

ファイルがシンボリックリンクかどうかは `file -h` で調べることができます。実は `file` コマンドは対象のシンボリックリンクについて辿った結果を表示するのがデフォルトの挙動になっています(オプションとしては、`-L, --deference`)。`-h, --no-dereference` オプションをつけることでシンボリックリンクを辿った結果にならないよう(シンボリックリンク先を)表示します。

```shell
# デフォルトではシンボリックリンクを辿った結果を表示
❯ file (which fish)
/opt/homebrew/bin/fish: Mach-O 64-bit executable arm64
# -h オプションでシンボリックリンク先が存在するなら参照先を表示
❯ file -h (which fish)
/opt/homebrew/bin/fish: symbolic link to ../Cellar/fish/3.3.1/bin/fish
```

更に `readlink` という外部コマンドで対象ファイルのシンボリックリンク先のパスを表示できます。ただ、 `readlink` の BSD 実装版だとシンボリックリンク先の絶対パスが得られないので、Homebrew で GNU 実装のコマンドを入れてみます。

https://formulae.brew.sh/formula/coreutils

```shell
❯ brew install coreutils
```
macOS で提供されているコマンドと同一名のコマンドについては頭に"g"とついた名前でインストールされるそうです。これで `greadlink` コマンドが使えるようになり、オプション `-f` を使って絶対パスを表示できます。もう一度 `fish` について調べます。

```shell
❯ greadlink -f (which fish)
/opt/homebrew/Cellar/fish/3.3.1/bin/fish
```

ということで、実は fish の実体だと思っていた `/opt/homebrew/bin/fish` はシンボリックリンクであり、バイナリファイルは `/opt/homebrew/Cellar/fish/3.3.1/bin/fish` に存在していることが分かりました。つまり、`od -x (which fish)` で見たバイナリファイルの中身は、`/opt/homebrew/bin/fish` のシンボリックリンク先である `/opt/homebrew/Cellar/fish/3.3.1/bin/fish` の中身でした。

調査してみるとパッケージマネージャーである Homebrew はインストールしたパッケージへのシンボリックリンクを特定のディレクトリ(`(brew --prefix)/bin`)に集めて一元管理するようになっています(macOS Intel なら `/usr/local/bin`, ARM なら `/opt/homebrew/bin`)。

>Homebrew installs packages to their own directory and then symlinks their files into /usr/local (on macOS Intel).
- [Homebrew 公式ページ](https://brew.sh/) より引用

※ Homebrew については次の記事が非常に分かりやすく用語等を解説していたので、参照してください。
https://blog.ottijp.com/2020/05/23/homebrew/

:::message
そもそも Homebrew とは、Apple 側が macOS に同梱しない UNIX ツールをインストールするためのツールです。

>Homebrew is the easiest and most flexible way to install the UNIX tools Apple didn’t include with macOS. It can also install software not packaged for your Linux distribution to your home directory without requiring sudo.
- [Homebrew Documentation](https://docs.brew.sh/Manpage#description) より引用

シンボリックリンクを配置するディレクトリである `(brew --prefix)/bin` は環境によって異なり、自分の環境では `/opt/homebrew/bin` です。 この Homebrew の prefix とはマシンによって異なるパッケージのインストールパスです。マシンによって次のように prefix がデフォルトで決まっています。`brew --preifix` コマンドで自分の環境の prefix を確認できます。

- macOS Intel: `/usr/local`
- macOS ARM: `/opt/homebrew`
- Linux: `/homelinuxbrew/.linuxbrew`

この prefix を基準に bin ディレクトリや Cellar の場所が prefix/bin や prefix/Cellar のように決まります。Intel 製の mac なら `/usr/local/bin` にシンボリックリンクが配置されます。そして、Homebrew は、この prefix の外側にはインストールしたものを配置しないようになっているそうです。

>Homebrew won’t install files outside its prefix and you can place a Homebrew installation wherever you like.
- [Homebrew 公式ページ](https://brew.sh/) より引用

ちなみに、Apple Silicon 製の mac にてデフォルトのインストール用の prefix が `/opt/homebrew` になるのは Rosetta 2 用の prefix `/usr/local` と共存して使えるようにするためらしいです。
https://docs.brew.sh/FAQ#why-is-the-default-installation-prefix-opthomebrew-on-apple-silicon
:::

インストールしたパッケージのバイナリファイルが配置されているのは、Cellar と呼ばれるディレクトリです。`brew --cellar` で Cellar ディレクトリの場所を表示できます。

```shell
❯ brew --cellar
/opt/homebrew/Cellar
❯ ls (brew --cellar)
bat              gettext          libtermkey       openjpeg         sqlite
ca-certificates  gh               libtiff          openssl@1.1      starship
coreutils        ghostscript      libuv            pcre2            tmux
dart             gmp              little-cms2      peco             trash
deno             groff            luajit-openresty pipenv           tree
exa              hugo             luv              pstree           tree-sitter
fd               jasper           mpdecimal        psutils          uchardet
fish             jbig2dec         msgpack          python@3.10      unibilium
fontconfig       jpeg             ncurses          python@3.9       utf8proc
freetype         libevent         neovim           readline         xz
fzf              libidn           netpbm           sass
gdbm             libpng           nodebrew         six
```

これらのパッケージ内にあるバイナリファイルは、Homebrew を使ってインストールした際に自動的に行われる [brew link という操作](https://docs.brew.sh/Manpage#link-ln-options-installed_formula-)によって、自動的にシンボリックリンクされます。[exa という外部コマンド](https://the.exa.website)で実際にシンボリックリンクになっているか確認してみます。

```shell
❯ brew install exa
```

exa コマンドの `-F` オプションでファイルの種類を簡単に判別でき、最後に `@` がついているものがシンボリックリンクで、`*` がついているものが executable です。`(brew --prefix)/bin`(`/opt/homebrew/bin`) の中を実際に覗いてみると、`brew` 以外のファイルはすべてシンボリックリンクになっていました。

```shell
❯ exa -F /opt/homebrew/bin
2to3@             gshred@           pamtopnm@            ppmdmkfont@
2to3-3.10@        gshuf@            pamtosrf@            ppmdraw@
411toppm@         gsleep@           pamtosvg@            ppmfade@
addftinfo@        gslj@             pamtotga@            ppmflash@
afmtodit@         gslp@             pamtotiff@           ppmforge@
anytopnm@         gsnd@             pamtouil@            ppmglobe@
asciitopgm@       gsort@            pamtowinicon@        ppmhist@
atktopbm@         gsplit@           pamtoxvmini@         ppmlabel@
autopoint@        gstat@            pamtris@             ppmmake@
avstopam@         gstdbuf@          pamundice@           ppmmix@
b2sum@            gstty@            pamunlookup@         ppmnorm@
base32@           gsum@             pamvalidate@         ppmntsc@
basenc@           gsx@              pamwipeout@          ppmpat@
bat@              gsync@            pbmclean@            ppmquant@
bioradtopgm@      gtac@             pbmlife@             ppmquantall@
bmptopnm@         gtail@            pbmmake@             ppmrainbow@
bmptoppm@         gtee@             pbmmask@             ppmrelief@
brew*             gtest@            pbmminkowski@        ppmrough@
# 長いので省略
```

例えば、Cellar 内の `fish` についてのディレクトリの中身を見てみます。Tree 形式でみたいので `exa` の `--tree` オプションを使用します。

```shell
❯ exa /opt/homebrew/Cellar/fish --tree -a
/opt/homebrew/Cellar/fish
└── 3.3.1
   ├── .brew
   │  └── fish.rb
   ├── bin
   │  ├── fish # <= バイナリファイル(コマンドの本体)
   │  ├── fish_indent
   │  └── fish_key_reader
   ├── CHANGELOG.rst
   ├── COPYING
   ├── etc
   │  └── fish
   │     └── config.fish
   ├── INSTALL_RECEIPT.json
   ├── README.rst
   └── share
      ├── applications
      │  └── fish.desktop
      ├── doc
      ## 長いの省略
```

Hmoebrew では以上のように PATH 環境変数に登録されている `/opt/homebrew/bin` (`(brew --prefix)/bin`)には実際の外部コマンドのバイナリファイルが配置されていません。Cellar と呼ばれるディレクトリにある keg と呼ばれるバージョン番号が付与されたディレクトリ(`/opt/homebrew/Cellar/exa/0.10.1`)に配置されたバイナリファイルへのシンボリックリンク(`/opt/homebrew/bin/exa`)が配置されています。

なぜ、このようにするかについては「**パッケージのバージョンが変わってもで同一のパスを提供する**」ことができる、というのが大きな理由のようです。

https://stackoverflow.com/questions/35337601/why-is-there-a-usr-local-opt-directory-created-by-homebrew-and-should-i-use-it

後は、`exa /opt/homebrew/Cellar/fish --tree -a` で見たように Cellar 内のパッケージには色々なファイルが含まれているので、`(brew --prefix)/bin` にバイナリファイル単体へのシンボリックリンクを集約しているのは合理的に思えます。さらに、`/opt/homebrew/bin` に各バイナリファイルへのシンボリックリンクが集約されていることによって Homebrew 用に PATH 環境変数へ追記する必要のあるディレクトリ(コマンドサーチの対象ディレクトリ)が `/opt/homebrew/bin` だけで済みます。

このように外部コマンドは実体がバイナリファイルであっても検索対象のディレクトリに実際に配置されているのはシンボリックリンクの場合があるとわかりました。それでは、Homebrew でインストールしたものではなく OS に同梱されている Included command の場合でもこのようなケースがあるのでしょうか。

実際に `/usr/bin` 内を調べるとそのようなコマンドは 55 個存在していることがわかりました。結果を一部抜粋すると Python2 系のプログラムが該当しました。

```shell
# command -a を使って /usr/bin 内の外部コマンドとして呼び出せるコマンド名についてすべての場所を表示し、grepでフィルターして fileでテストした後に更にgrepでフィルター
❯ file -h (command -a (ls /usr/bin) | grep "/usr") | grep "symbolic" | count
55
❯ file -h (command -a (ls /usr/bin) | grep "/usr") | grep "symbolic"
# 結果を一部抜粋
/usr/bin/pydoc:                              symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/pydoc2.7
/usr/bin/pydoc2.7:                           symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/pydoc2.7
/usr/bin/python:                             symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7
/usr/bin/python-config:                      symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7-config
/usr/bin/python2:                            symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7
/usr/bin/python2.7:                          symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7
/usr/bin/python2.7-config:                   symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7-config
/usr/bin/pythonw:                            symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/pythonw2.7
/usr/bin/pythonw2.7:                         symbolic link to ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/pythonw2.7
```

exa を使って簡単にファイルタイプを確認(@ がついているものがシンボリックリンク)できます。

```shell
❯ exa -F /usr/bin --sort=type -r -x
yaa@                                 wish@
vimdiff@                             view@
vi@                                  tkcon@
tclsh8.5@                            tclsh@
tar@                                 swcutil@
snmpinform@                          snfsdefrag@
smtpd2.7.py@                         slogin@
sdx@                                 safaridriver@
rvim@                                rview@
reset@                               qlmanage@
pythonw2.7@                          pythonw@
python2.7-config@                    python2.7@
python2@                             python-config@
python@                              pydoc2.7@
pydoc@                               pico@
phar@                                newaliases@
nclist@                              ncinit@
ncdestroy@                           manpath@
mailq@                               ldapadd@
kswitch@                             klist@
infotocap@                           idle2.7@
idle@                                fontrestore@
ex@                                  cvmkfile@
cvmkdir@                             cvcp@
cvaffinity@                          captoinfo@
bzless@                              bzcmp@
auval@                               2to3-2.7@
2to3-@                               zprint*
znew*                                zmore*
zless*                               zipsplit*
# 以降すべて executable なので省略
```

`/usr/bin/python` 等が `/System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7` にシンボリックリンクしているのは macOS のバージョニングシステムの一部のようです。
https://stackoverflow.com/questions/48740260/osx-whats-the-difference-between-usr-bin-python-and-system-library-framewor

これも macOS が利用する Python のバージョンが変更されてもシンボリックリンクの指し示す先が変更されれば `python` コマンドは同一のファイルパス(`/usr/bin/phthon`)で利用できるようになっていますね。
# まとめ

まとめると、このようになります。

- **Function** : 他のコマンドをグルーピングし、名前を付けて実行できるようにして定義したもの。ファイル拡張子は `.fish` で、自作したものや、いくつかの external command のラッパーやユーティリティ(`ls`、`man` や `help` 等)が fish shell から提供されている。fish shell から提供されるものは、Homebrew などを使っている場合には `/opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/` などの場所に fish shell script として定義されている。
- **Builtin** (内部コマンド: **Internal command**) : シェルから見て内部にあるコマンドで、fish shell のプログラム自体に組み込まれて提供されている(`and`, `argparse`, `for` など)。各コマンドに対応するソースコードは C++ で書かれている。
- **External command** (外部コマンド) : シェルから見て外部にあるコマンドで、fish shell 自体とは関係のないプログラム。bash などの別のシェルを使っていても呼び出すことができる。`/bin` や `/usr/bin`、`/usr/local/bin`、`/opt/homebrew/bin/` といったディレクトリに種類ごとで配置されている。macOS などの [UNIX-like OS](https://www.wikiwand.com/ja/Unix%E7%B3%BB) にもともと入っている**UNIX command**(OS に同梱されていることから **Included command** とも呼ばれることもあるらしい)や、Homebrew などのパッケージマネージャーを使って自分で入れることができるパッケージのコマンドなど。実体は executable なバイナリファイルや、テキストファイル(shell script など)。外部コマンドの呼び出し対象となるのは、`PATH` 環境変数に登録されたディレクトリで、新しく追加されたものから検索される。登録されているディレクトリに実際のバイナリファイルが存在するとは限らずシンボリックリンクの場合も多々ある。

