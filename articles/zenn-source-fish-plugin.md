---
title: "fish shellで一気にファイルをsourceするプラグインを作ってみた"
emoji: "🍣"
type: "tech"
topics: [fish, shell, 初心者, ターミナル, CLI]
published: true
date: 2022-02-12
url: "https://zenn.dev/estra/articles/zenn-source-fish-plugin"
aliases: [記事_fish shellで一気にファイルをsourceするプラグインを作ってみた]
tags: " #type/zenn #shell/fish  "
---

## モチベーション

[前回の記事](https://zenn.dev/estra/articles/google-search-from-fish-shell)で、コマンドラインからgoogle検索をできるようにするfish関数の開発について書きましたが、それをplugin managerである**fisher**を使ってpluginとしてインストールできるようにGithubのリポジトリで開発をすすめました。

https://github.com/yo-goto/ggl.fish

プラグイン開発の最中、ターミナルから`source`コマンドを使ってコマンドのテストを行うことが多かったのですが、プラグインに使用するfishファイルが増えるごとに`source`するのが面倒になり、一気にfishファイルをソースさせるプラグインを作成しました。

当初はgit管理している`ggl`(コマンドラインからgoogle検索させる自作プラグイン)のディレクトリ内部のfish fileを一気に読み込ませるだけのものでしたが、githubで公開用に一般化するにあたり、

- config directoryにあるスニペットもsoruceさせる
- カレントディレクトリ内の指定したディレクトリをsourceさせる
- テストディレクトリ(`test/`や`tests/`)をsourceさせる
- 最近変更したファイルのみをsourceさせる

といった機能を複数のオプションで搭載させて開発しました。fish functionのオプション処理の基本については前の記事で紹介したので興味があればそちらを見ていただければと思います。

作成した`source-fish`は基本的にはプラグイン開発用のユーティリティとして使えますが、configディレクトリ内の複数のfish fileについてもインタラクティブにsoruceさせることができます。以下が実際に作成したfishプラグインのリポジトリになります。`fisher install yo-goto/source-fish`でインストールできます。

https://github.com/yo-goto/source-fish

この記事では、その開発にあたって得た知見とfish shellについての基礎知識、プラグイン開発の方法などを紹介します。

## プラグイン開発方法

基本的には、プラグインマネージャーであるfisherを使ったインストール･管理を想定してプラグインを開発します。

https://github.com/jorgebucaran/fisher

fisherのリポジトリにプラグインの開発方法が記載されています。基本的には以下のようにディレクトリ構造を作成して定義する関数名と同一の名前の`関数名.fish`のようにファイルを作成します。

- functions : 関数定義
- completions : オートサジェストされるコンプリーションなどの設定
- conf.d : グローバル変数などの設定用変数などの設定

```console
ponyo
├── completions
│   └── ponyo.fish
├── conf.d
│   └── ponyo.fish
└── functions
    └── ponyo.fish
```

この3つのディレクトリすべてが必要ではなく、基本的には、`functions`、`completions`を用意しておけばよいと思います。

この3つのディレクトリを適当な場所に作成して適宜`source`しながら、テストします。または、fisherを使って、`fisher install $PWD`でディレクトリ内のfish filesを一旦プラグインとしてインストールさせて`fisher update $PWD`でアップデートさえてあげることもできます。自分の場合は安定版を他のシェルで使いながら開発したいので、テスト用のシェルセッション内において繰り返し`soruce`させていました。

`ggl`のプロジェクトのディレクトリ構造は以下のような感じになります。`ggl`のプラグインにはそのラッパーである`fin`というコマンドを更に開発で入れたのでこのような複数のファイル構成になりました。

```console
.
├── CHANGELOG.md
├── completions
│  ├── fin.fish
│  └── ggl.fish
├── conf.d
│  └── ggl.fish
├── functions
│  ├── fin.fish
│  └── ggl.fish
├── LICENSE
├── README.md
└── test
   ├── myt.fish
   ├── newtest.fish
   └── xdg-test.fish
```

このように複数のファイルがある場合には、面倒ですが、プラグイン無しだとファイルごとにソースすることになります。

```console
$ source ./functions/ggl.fish
$ soruce ./functions/fin.fish
$ source ./completions/fin.fish
$ source ./completions/gll.fish
```

## おすすめのプラグイン

プラグイン作成では次の3つのプラグインが参考になりました(すべてMITライセンスです)。細かいところまでは見ていませんが、自分のような初心者にとって関数の定義方法やオプション分岐などが参考になる部分が結構ありました。興味があれば確認してみください。

- [fisher](https://github.com/jorgebucaran/fisher)
- [fish_logo](https://github.com/laughedelic/fish_logo)
- [tide](https://github.com/IlanCosman/tide)

## Function, Builtin, External command

他のシェルでも大体同じらしいですが、自分はほぼ最初のシェルがfishなので呼び出すことのできるコマンドに種類が大きく分けて3つあることを知りませんでした。まずは、基礎知識としてそちらについて解説します。

コマンドラインから特定のコマンド名を入力して実行すると呼び出される可能性のあるものは次の3つとなります。

:::message alert
"command"ではなく、"external command"でしたので修正しました。
:::

- **Function** : 他のコマンドをグルーピングし、名前を付けて実行できるようにして定義したコマンド。自作したものや、fish shellから提供される特定のexternal commandのラッパーやユーティリティ(`ls`、`man`や`help`等)など。
- **Builtin** (内部コマンド: Internal command) : fish shellに組み込まれて提供されているコマンド類(`and`, `argparse`, `for`など)
- **External command** (外部コマンド) : binファイル(fish自体とは関係のないプログラム)で、`/bin`や`/usr/bin`、`/usr/local/bin`、`/opt/homebrew/bin/`といったディレクトリに種類ごとに配置されている。マシンにもともと入っている**Unix command**(OSに同梱されていることから**included command**とも呼ばれることもあるらしい[^1])や、Homebrewなどを使って自分で入れることもできるパッケージのコマンド。

[^1]: [Command-line interface - Wikiwand](https://www.wikiwand.com/en/Command-line_interface)の"Anatomy of a shell CLI"の項目では、コマンドは通常、internal command, inclueded command, external commandの3つのうちのどれかであると記載されている

:::message
External command(外部コマンド)については、それぞれのディレクトリには以下のような種類のコマンドが格納されています。
- `/bin` : `cat`, `kill`, `mkdir`など
- `/usr/bin` : `sudo`,`groff`,`find`など
- `/usr/local/bin` : vscodeの`code`など
- `/opt/homebrew/bin/` : `fish`, `deno`, `gh`, `exa`などHomebrewを使用して自分でインストールしたパッケージのコマンド

それぞれのディレクトリにどんなコマンドが格納されているかは`ls`コマンドで閲覧できます。

```shell
❯ ls /bin
[         csh       echo      ksh       mkdir     rm        sync      zsh
bash      dash      ed        launchctl mv        rmdir     tcsh
cat       date      expr      link      pax       sh        test
chmod     dd        hostname  ln        ps        sleep     unlink
cp        df        kill      ls        pwd       stty      wait4path
```

例えば、上記の`/bin`ディレクトリには、OSが壊れた際など非常時でも利用できる、ごく基本的かつ非常時に利用するコマンドが置かれているそうです。

参考: https://linuc.org/study/knowledge/544/
:::

実行するコマンドがどのカテゴリーに属するかは`type`というfish shellが提供するビルトインコマンドで調べることができます。

typeは｢**コマンドラインに入力したコマンドがfish shellにおいてどのように解釈されるかを示す**｣コマンドです。例えばこの`type`自体を調べてみます。

```console
❯ type type
type is a builtin
```

これは、コマンドライン上で`type`を入力して実行した際にbuiltinの`type`として解釈され実行されることを意味します。

今度はUnix commandである`cat`コマンドやHomebrewの`brew`コマンドを`type`してみると次の結果になります。これは、それぞれがexternal commandとして呼び出されることを示しています。

```console
❯ type cat
cat is /bin/cat
❯ type brew
brew is /opt/homebrew/bin/brew
```

さらに今度は指定したコマンドのマニュアルを閲覧するコマンドである`man`コマンドを`type`してます。

```console
❯ type man
man is a function with definition
# Defined in /opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/man.fish @ line 6
function man --description 'Format and display the on-line manual pages'
    # Work around the "builtin" manpage that everything symlinks to,
    # by prepending our fish datadir to man. This also ensures that man gives fish's
    # man pages priority, without having to put fish's bin directories first in $PATH.

    # Preserve the existing MANPATH, and default to the system path (the empty string).
    set -l manpath
    if set -q MANPATH
        set manpath $MANPATH
    else if set -l p (command man -p 2>/dev/null)
        # NetBSD's man uses "-p" to print the path.
        # FreeBSD's man also has a "-p" option, but that requires an argument.
        # Other mans (men?) don't seem to have it.
        #
        # Unfortunately NetBSD prints things like "/usr/share/man/man1",
        # while not allowing them as $MANPATH components.
        # What it needs is just "/usr/share/man".
        #
        # So we strip the last component.
        # This leaves a few wrong directories, but that should be harmless.
        set manpath (string replace -r '[^/]+$' '' $p)
    else
        set manpath ''
    end
    # Notice the shadowing local exported copy of the variable.
    set -lx MANPATH $manpath

    # Prepend fish's man directory if available.
    set -l fish_manpath $__fish_data_dir/man
    if test -d $fish_manpath
        set MANPATH $fish_manpath $MANPATH
    end

    command man $argv
end
```

最初に、`man is a function with definition` とあるので`man`は定義された関数(function)であることがわかります。また、`# Defined in /opt/homebrew/Cellar/fish/3.3.1/share/fish/functions/man.fish @ line 6`とあり、どこで定義されているのかが分かります。

また、最後から2行目で`command man $argv`とあり、結局この`man`というfunctionは同じ名前のexternal commandである`man`のラッパーであることがわかります。ちなみに`command`というコマンド自体は指定した名前のプログラムを実行するというビルトインコマンドです。`command man`はmanというfunctionではなく、manという名前のプログラムを実行させるという意味です。

`type`したものがfunctionの場合、長い定義が表示されてしまうので、`-t`オプションで種類だけ表示させることもできます。

```console
❯ type man -t
function

# external commandの場合にはfileと表示される
❯ type cat -t
file
❯ type brew -t
file
```

さて、この`man`プログラム自体はどこにあるかというと`which`コマンド(これはプログラム)を使って調べることができます。`which`はユーザーパス内のプログラムファイルを探すプログラムです(ちなみにBSDの`which`コマンドはC言語で書かれているそうです)。

```console
❯ which man
/usr/bin/man
```

これで、`man`プログラムが`/usr/bin/man`というところにあることがわかります。

ちなみに`which type`をすると次のような結果になります。

```console
❯ which type
/usr/bin/type
```

これは`type`という名前のプログラムが存在していることを示しますが、コマンドラインに入力して実際に実行されるのはfish shellのビルトインである`type`です。

その理由は、**実際に呼び出されるものに優先度がある**ためで、以下の優先順位で呼び出されます。

```
function > builtin > external command 
```

つまり、ビルトインである`type`はexternal commandである`type`よりも優先度が高いので、ビルトインの`type`が呼び出されるということです。このように、[コマンドとして呼び出されるものには優先度があり](https://github.com/fish-shell/fish-shell/issues/1732)、自分で定義するfuncitonなどが最優先になるため、仮に同じ名前のbuiltinやexteranl commandがあってもそれらは呼び出されなくなります。

:::message
Bashでのコマンドサーチも同様です。

参考: [Command Search and Execution (Bash Reference Manual)](https://www.gnu.org/software/bash/manual/html_node/Command-Search-and-Execution.html#Command-Search-and-Execution)

function → builtinの順番でタイプされたコマンド名を探し、該当するものがなければ、`$PATH`変数に格納されている各ディレクトリに含まれているその名前の実行可能ファイルを検索します。

fishでも同様に[$PATHという特殊変数](https://fishshell.com/docs/current/language.html#special-variables)にコマンドの検索対象となるディレクトリが収められています。

```shell
❯ echo $PATH
/usr/local/bin /usr/bin /bin /usr/sbin /sbin /Users/roshi/.nodebrew/current/bin /opt/homebrew/bin/
```
:::

意図的に"builtin"または"external command"を指定して呼び出したい場合にはそれぞれの名前がついたビルトインコマンドを付けて使用します。例えば、自分で`type`という名前のfunctionを作成すると、最優先はfunctionなので自分で定義した関数が呼び出されます。

```fish
function type -d "自作したtype function"
    echo $argv "はtypeされません"
    echo "これは自作したtypeで最優先に呼び出されます"
end
```

このようなfunctionを作成しconfig dirに配置したとしましょう。コマンドラインに`type`を入力し呼び出してみると、以下のようになります。

```console
❯ type man
man はtypeされません
これは自作したtypeで最優先に呼び出されます
```

builtinを呼び出す場合にはそのまま`builtin`というコマンドを頭につけることで呼び出せます。

```console
❯ bultin type man
# 省略。先述のビルトインを使った type man の結果が表示される
```

external commandを呼び出す場合には`command`というコマンドを頭につけることで呼び出せます。この場合に呼び出されるコマンド種類は環境によって異なります。macOSなら**BSD実装**のtype、Linuxのディストリビューションなら**GNU実装**のtypeが呼び出されます。それぞれで挙動が異なることがあるので注意してください。ググるとでてくるのは基本的にGNU実装のコマンドの解説です。

```cosole
❯ command type man
man is /usr/bin/man
```

:::message
macなどのマシンに元々入っている基本的なコマンドのいくつかは**Unix command**と呼ばれ、[Unix-likeなOS](https://www.wikiwand.com/en/Unix-like)にはそれぞれの実装で同梱されています。

参考: https://www.wikiwand.com/en/List_of_Unix_commands

Linuxの多くのディストリビューションに付随しているUnixコマンドは、[GNU](https://www.wikiwand.com/ja/GNU/Linux)というOSのユーティリティとして提供されているものであり、後述する`find`コマンドなどはGNUでは[POSIX](https://www.wikiwand.com/en/POSIX)と呼ばれるUNIX系OSの標準規格に指定されていない大量の追加機能が実装されているそうです。

参考: https://www.wikiwand.com/en/List_of_GNU_Core_Utilities_commands
:::

ちなみに、すべてのbuiltinコマンドは`builtin -n`というコマンドで閲覧することができます。同じようにすべてのfunctionは`functions -n`で閲覧できます。また、external commandについては`command --all COMMANDNAME`で指定した名前のコマンドがあるか調べることができます。

:::message
fishでは紛らわしいものとしてalias(エイリアス)というものもありますが。これは簡易的な記述でfunctionを作成するためのものです。`man alias`すると｢carete a function｣という概要説明とともに、｢alias is a simple warpper for the function builtin, which creates a function warpping a command｣と出てきます。

```shell
alias type="echo 'not builtin type'"
```

上のようにエイリアス定義するのは関数作成するのと同じです。`type`を実行して呼び出されるのはfunctionとなります。

```console
❯ type
not builtin type
```
:::

このように、function、builtin、external commandの違いを意識し、それぞれを駆使してshell scriptでプラグインを書いていきます。

## findとsourceをラップする

今回作ったプラグインは言ってしまえば、BSD系のexternal commandである`find`とfish builtinである`soruce`コマンドをラップした関数です。付加的に色々な処理をつけたりオプション分岐させて使いやすくしています。前のfish shellの記事で紹介した`ggl`もBSDの`open`コマンドやLinux用ユーティリティである`xdg-open`コマンドのラッパーといえます。

`find`コマンドはかなり複雑なオプション処理ができて便利なコマンドです。`man find`して最初に出てくる説明は｢walk a file hierarchy｣と出てきますが、要するにディレクトリ階層を再帰的に探索して指定したパターンなどにマッチするファイルやディレクトリを探すコマンドです。

https://www.wikiwand.com/en/Find_(Unix)

:::message
ちなみに`find`の代わりに`fd`というRust製の代替コマンドなどもあるらしいです。

https://github.com/sharkdp/fd
:::

`source`ビルトインコマンドは、ファイルの中身を評価してカレントシェルで新規のコードブロックとして使えるようにするコマンドです。

この2つを組み合わせて、`find`でfish fileを探して、`source`で呼び込ませます。

## プラグイン構造

おすすめのプラグインで紹介したものでも使われているのですが、処理が長くなったりするものや何度も使うようなものはメインの関数から分岐させてヘルパー関数として別に定義させると便利です。一つのfish fileで複数の関数を定義できます。

```shell
function source-fish
    # body
end

# helper functions
function __source-fish_times
    # body
end

function __source-fish_help
    # body
end
```

最新バージョンのfish shellでもローカル関数の定義はできず、すべての関数はグローバルスコープになりますが、関数名の頭にアンダースコアをつけつことでヘルパー関数として定義するパターンがよく使われます。またアンダースコアをつけることで`functions`という定義されているコマンド一覧を表示するコマンドから結果を除外して非表示にすることもできます。すべての定義関数を表示したい場合には`functions -a`を使います。

前回の記事で紹介した`argparse`を使って今回もオプション処理を行います。基本はフラグ変数をif分岐でテストします。ヘルプオプションは基本的にヘルパー関数を使用するように分岐させておくと管理が楽なのでヘルパー関数として別に定義しておきます。

```shell
function soruce-fish
    argparse \
            -x 'v,h,r,a,t,c' \
            'v/version' 'h/help' 'r/recent' 'a/all' 't/test' 'c/config' -- $argv
    or return
    
    set --local version_soruce_fish "v0.1.5"
    set --local directory $argv
    # コマンド引数を一旦別の変数にセットする
    
    if set -q _flag_version
        echo "soruce-fish:" $version_source_fish
        return
    else if set -q _flag_help
        ## ヘルパー関数を呼び出す
        __source-fish_help
        return
    else if #他のフラグ処理
        # body
    else if test -n "$directory"
        # コマンド引数があった場合の処理
    else 
        # main 処理
    end
end


function __source-fish_help
    set_color yellow
    echo "Usage:"
    echo "      source-fish [OPTION]"
    echo "      source-fish DIRECOTRIES..."
    echo "Options:"
    echo "      -v, --version   Show version info"
    echo "      -h, --help      Show help"
    echo "      -r, --recent    Find recently modified files & source them"
    echo "      -a, --all       Source all fish files under the current directory"
    echo "      -t, --test      Source all fish files in the \"test\" folder"
    echo "      -c, --config    Source fish files in the config directory"
    set_color normal
end
```

プラグインの基本構造はこのような感じになります。オプション分岐についてすべてを解説すると長くなるので(というかほぼ同じ処理なので)引数無しで実行した場合の処理と中枢で使用するヘルパー関数について解説していきます。

## question loop

今回、コマンドを実行してすぐにメイン処理を行うのではなく、ユーザーに質問を行って実行確認してからメイン処理を行わせるようにします。そういった処理を行うものとして、`while ture; BODY; end`のループを作成し、中で入力を受け取らせるコマンドを使うというパターンがよく使われるそうです。

https://stackoverflow.com/questions/226703/how-do-i-prompt-for-yes-no-cancel-input-in-a-linux-shell-script

fishでは、`read`という入力を受け取るbuiltinコマンドが存在します。この`read`コマンドのオプションをいくつか使い、更に`switch`と`case`コマンドを使って分岐作成し、インタラクティブに質問に対する答えを受け取らせます。

先ほど説明したプラグイン構造内の最後のif分岐であるelseにおいて次のような処理を行います。

```shell
else
    echo "Current:"$cc $PWD $cn
    while true
        read -l -P "Source fish files in this project? [Y/n]: " question
        switch "$question"
            case Y y yes
                # メインのfindとsourceの処理
                return
            case N q n no
                return 1
        end
    end
end
```

`read`コマンドでは、`-l`オプションで指定した文字列(この場合は`question`)をローカル変数として、必要の答えとして受け取らせた文字を格納します。`-P`オプションで、指定した文字(この場合は`Source fish files in this project? [Y/n]: `)をプロンプトとして使えるようにします。これによって、｢このプロジェクト内のfish filesをsourceしますか?｣という質問がコマンドライン上に表示され、必要に対する答えとして`Y`、`y`、`yes`を入力したら、メインの処理を、`N`、`q`、`n`、`no`などの文字列を入力したらコマンドを終了させる、それら以外の文字列を入力したらループ内で再度質問させるという、処理を実現しています。

この一連のパターンでインタラクティブな処理のループを作成しています。

## findでパターンにマッチするファイルを探す

それでは、質問の答えとしてyesを選択した場合の処理を解説していきます。やることとしては、カレントディレクトリ内のfishファイルを探しsourceさせるということですが、まずは`find`をつかって目的のfish fileを探します。

`find`はかなり細かく条件を絞り込んでファイルをさがすことができます。

今回の場合、fish shellのプラグイン開発用のディレクトリである、`functions`、`completions`、`conf.d`というディレクトリがカレントディレクトリの直下に存在すると想定して`find`コマンドにわたすパターンを考えます。

```shell
switch "$question"
    case Y y yes
        set --local list_functions (command find . -type f -depth "-3" -path "./functions/*.fish")
        set --local list_completions (command find . -type f -depth "-3" -path "./completions/*.fish")
        set --local list_conf (command find . -type f -depth "-3" -path "./conf.d/*.fish")

        # source処理
        return
    case N q n no
        return 1
end
```

後でまとめて`source`させるために、まずはコマンド置換内の`find`で見つかったfish fileまでのファイルパスを一旦ローカル変数内に格納させます。

上で説明した通り、他のユーザーの環境に`find`という名前のfunctionがあった場合に備えて明示的に`find`という名前のexternal commandを使うように`command`を頭に付けておきます。

```shell
set --local list_functions (command find . -type f -depth "-3" -path "./functions/*.fish")
```

- (1) `find`コマンドには、探索開始地点を指定してあげることが必要なので、カレントディレクトリを示す`.`をまずは指定してあげます(`find .`)。
- (2) 探索する対象はディレクトリではなくファイルなので対象を｢ファイル｣という種類に限定する`-type f`というプライマリ(pirimary: オプションのようなもの)を指定してあげます(`find . -type f`)。
- (3) さらに探索するディレクトリの階層を最大で3階層という制限を`-depth "-3"`というプライマリを使って、念のために与えておきます(`find . -type f -depth "-3"`)。
- (4) 最後に`-path PATTERN`というプライマリを使って、探すパターンを指定します。例えば、カレントディレクトリの直下にある`functions`というディレクトリ内に存在する拡張子が`.fish`であるファイルの場合には指定するPATTERNとして`./functions/*.fish`という風になります。`*`はワイルドカードで任意の文字列を表します。

これで、最終的な`find . -type f -depth "-3" -path "./functions/*.fish"`が完成しました。同じ用に`completions`と`conf.d`ディレクトリについても`find`します。

見つけることができたfish fileが各ディレクトリにおいて0個でない場合にはそれらのファイルを`soruce`させます。`source`の処理には見つけた個数分処理させる必要があり、さらにこの処理は他のオプション分岐でも活用できるので`__source-fish_times`という名前のヘルパー関数として独立させます。

```shell
switch "$question"
    case Y y yes
        set --local test_flag
        set --local list_functions (command find . -type f -depth $max_find_depth -path "./functions/*.fish")
        set --local list_completions (command find . -type f -depth $max_find_depth -path "./completions/*.fish")
        set --local list_conf (command find . -type f -depth $max_find_depth -path "./conf.d/*.fish")

        test -n "$list_functions"
            and __source-fish_times $list_functions
            and set test_flag "OK"
        test -n "$list_completions"
            and __source-fish_times $list_completions
            and set test_flag "OK"
        test -n "$list_conf"
            and __source-fish_times $list_conf
            and set test_flag "OK"
        not test "$test_flag" = "OK"
            and echo "No files found"
            and return 1
        return
    case N q n no
        return 1
end
```

`test`ビルトインコマンドの`-n`オプションをつかって指定した文字列が0でない場合に限ってヘルパー関数に引数として`find`したパスを渡すようにします。つまり、`find`できたものだけsoruceさせます。どのディレクトリについてもfindできない場合、つまり`test_flag`が"OK"ではない場合に｢No files found｣というメッセージが表示されます。

## bulk sourceさせる

さて、`find`して取得したfish fileまでの一連のファイルパスをリストとして次のヘルパー関数`__source-fish_times`に渡します。

```shell
# helper function
function __source-fish_times

    set --local cc (set_color yellow)
    set --local cn (set_color normal)
    set --local ca (set_color cyan)

    set --local argcount (count $argv)
    if test $argcount -ge 1
        for i in (seq 1 $argcount)
            builtin source $argv[$i]
            and echo $ca"-->completed:"$cc $argv[$i] $cn
        end
    else
        echo $ca"No files found" $cn
    end
end
```

渡されたリスト変数はこの関数内の`$argv`に渡されます。`$argv`はリストであり、その中に渡されているはずの`find`で発見したファイルパスの個数を数えるために`count`コマンドを使用し、別のローカル変数`argcount`にその個数を格納します。この個数が1以上であることを確かめるために`test $argcount -ge 1`でテストし、1以上(-ge: **g**reater than or **e**qual)なら内部の処理(source)を行わせます。

`seq 1 $argcount`で1からリスト要素の個数分の数列を作成し、`for`ループ内でその回数分`source`を実行させます。ここでも念のために`source`をbuiltinであると明示して`builtin source`を使います。

そして、`source`がうまく行った場合には、`and`コマンドで条件判定してcompletedというメッセージとともに対象となったファイルパスをコマンドライン上にechoさせます。見やすくさせるために、カラーもつけています。次のような感じの結果となります↓

![image](/images/source-fish/img_terminalIMG_source-fish.png)

このような感じでプロジェクト内のfishファイルを発見しsourceさせるプラグインが完成しました。

実際のプラグインでは、以下の機能も実装しています。

- config directoryにあるスニペットもsoruceさせる
- カレントディレクトリ内の指定したディレクトリをsourceさせる
- テストディレクトリ(`test/`や`tests/`)をsourceさせる
- 最近変更したファイルのみをsourceさせる

興味のある方はGithubでソースコードを公開していますので確認してみてください。

https://github.com/yo-goto/source-fish


