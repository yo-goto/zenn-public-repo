---
title: "fishのargparseで未定義オプションを透過させるラッパー関数を作成する"
emoji: "🦖"
type: "tech"
topics: [fish, shell, deno, homebrew]
published: true
date: 2022-03-04
url: "https://zenn.dev/estra/articles/zenn-fish-shell-argparse-unknown-options"
aliases: [記事_fishのargparseで未定義オプションを透過させるラッパー関数を作成する]
tags: " #shell/fish #type/zenn  "
---

# モチベーション
最近は Deno がお気に入りで、特に `deno run` コマンドを使えば TypeScript をすぐさま実行できて、リンターもデフォルトで色々やってくれるので、TypeScript の学習環境としてとても良いと感じています。

https://deno.land

ただ、Deno CLI の `deno run` コマンドで実行した TypeScript のデバッグに `console.log` の出力をリダイレクションを使って適当なファイルに出力すると、[ANSI escape code](https://www.wikiwand.com/en/ANSI_escape_code) という制御コードが一緒に出力されてしまい見づらかったので、これを取り除く処理をラッパー関数内に噛ませることで、きれいに出力できるようにしたい思います。

というわけで、Deno CLI のコマンドである `deno run` のラッパーを作成しつつ、fish shell の `argparse` の `--igonre-unknown` というオプションを使ったラッパーを作成します。ある程度、一般化できるかなと思い、今回はコマンドのラッパー関数を作成し、ラップ元にそのコマンド自体のオプションを渡すということをや解説していきます。

# やりたいこと

コマンドのラッパーにしたいので、やりたいこととしてはシンプルに次の三点です。

- ラップ元の `deno run` コマンドの補完(completion)の引き継ぎ
- ラップ元の `deno run` のオプションを使用できるようにする
- `deno run` で出力される結果から ANSI escape code を取り除くような処理をかませる

割と簡単に作成できます。`alias` を使えば早いのですが、ちょっとしたオプション分岐も付けたいので通常の `function` と `argparse` ビルトインコマンドを使用してラッパーを作成します(`fisher` や `git` を使ってプラグイン的に開発した方が管理もしやすいです)。

# 基本構造
通常のプラグイン作成と同じようにつくります(`fish-plugin-template` を使用してプロジェクト内に次の３つのディレクトリとテンプレートを展開しても OK)。

`fish-plugin-template` については以前の記事で紹介したので参照してください。
https://zenn.dev/estra/articles/zenn-fish-plugin-template

関数の名前は `deno-run-out` ということにしておきます。あとは、`deno run` で走らせるテスト用の TypeScript ファイルを `tests` ディレクトリに作成しておきます。

```shell:ディレクトリ構造
├── completions
│  └── deno-run-out.fish
├── conf.d
│  └── deno-run-out.fish
├── functions
│  └── deno-run-out.fish
└── tests
   ├── console_test.ts
   └── read_write_test.ts
```

`deno-run-out.fish` は次のような感じで、このテンプレートを改造していきます。

```shell:functions/deno-run-out.fish
function deno-run-out -d 'DISCRIPTION'
    argparse \ 
        -x 'v,h' \ 
        'v/version' 'h/help' -- $argv
    or return 1

    set --local version_deno_run_out 'v0.0.1'

    if set -q _flag_version
        echo "deno-run-out: " $version_deno_run_out
    else if set -q _flag_help
        __deno-run-out_help
    else
        # main body
    end
end

function __deno-run-out_help
    echo 'USAGE:'
    echo '      deno-run-out [OPTION]'
    echo 'OPTIONS:'
    echo '      -v, --version       Show version info'
    echo '      -h, --help          Show help'
end
```

名前については `alias` を使って短縮したものを適当に `conf.d` に定義しておきます。今回は `derun` という短縮名で使用できるようにします。

```shell:conf.d/deno-run-out.fish
alias derun="deno-run-out"
```

`read_write_test.ts` は次のように `--allow-read` と `--allow-write` パーミッションが必要なように適当なコードを書いておきます。

```ts:tests/read_write_test.ts
import * as fs from "https://deno.land/std@0.126.0/fs/mod.ts";
// docs: https://deno.land/std@0.126.0/fs

const dir_name = "created_dir";
const file_name = "created_file.md"

await fs.ensureDir(dir_name);
// ディレクトリが存在することを保証する、存在しなければ作成する(mkdir -p と同等)
console.log(`directory ${dir_name} is created by ensureDir()!`);
await fs.ensureFile(`${dir_name}/${file_name}`);
// ファイルが存在することを保証する、存在しなければ作成する
console.log(`file ${file_name} is created by ensureFile()!`)
```

`console_test.ts` にはリダイレクションによって ANSI escape code が入る出力用のテストコードを書いておきます。

```ts:tests/console_test.ts
const array= [1, 2, 3, 4, 5];

const new_item = array.push(6, 7, 8);
console.log({ array });
console.log({ new_item });

const removed_item = array.pop();
console.log({ removed_item });
console.log({ array });

const unshifted_length = array.unshift(-1, 0);
console.log({ unshifted_length });
console.log({ array });

const shifted_item = array.shift();
console.log({ shifted_item });
console.log({ array });
```

これを実際にリダイレクションしてみます。

```shell
❯ deno run tests/console_test.ts > tests/console_test.log
```

`[33m` のような制御コードが紛れ込んでしまっています。

```log:tests/console_test.log
{ array: [
    [33m1[39m, [33m2[39m, [33m3[39m, [33m4[39m,
    [33m5[39m, [33m6[39m, [33m7[39m, [33m8[39m
  ] }
{ new_item: [33m8[39m }
{ removed_item: [33m8[39m }
{ array: [
    [33m1[39m, [33m2[39m, [33m3[39m, [33m4[39m,
    [33m5[39m, [33m6[39m, [33m7[39m
  ] }
{ unshifted_length: [33m9[39m }
{ array: [
    [33m-1[39m, [33m0[39m, [33m1[39m, [33m2[39m, [33m3[39m,
     [33m4[39m, [33m5[39m, [33m6[39m, [33m7[39m
  ] }
{ shifted_item: [33m-1[39m }
{ array: [
    [33m0[39m, [33m1[39m, [33m2[39m, [33m3[39m,
    [33m4[39m, [33m5[39m, [33m6[39m, [33m7[39m
  ] }
```

# ラップ元の補完の引き継ぎ

補完の引き継ぎは、`completions` ディレクトリにある関数名と同一名のファイル `deno-run-out.fish` というファイルに以下を書けば終了です。これだけで、`deno run` の補完が引き継がれます。

```shell:completions/deno-run-out.fish
# deno run の補完の設定を deno-run-out に引き継ぐ
complete -c deno-run-out -w "deno run"
```

これによって、例えば `derun --` とコマンドラインに入力すると `deno run` のオプション補完が表示されるようになります。 
![image_derun_completion](/images/fish-derun/img_denorun_completion_.jpg)

あとは、ラッパーコマンド自体のオプションの補完を追加しておきます。

```shell:completions/deno-run-out.fish
complete -c deno-run-out -s v -l version -f -d "Show version info"
complete -c deno-run-out -s h -l help -f -d "Show help"
complete -c deno-run-out -s s -l stdout -f -d "Strip ANSI escape code for stdout"
```

補完スクリプトのつくり方については以下の記事が網羅的かつ分かりやすくまとまっています。
https://qiita.com/nil2/items/128363097ac031653ea1#commandline

:::message
ちなみにですが、Deno CLI における `deno run` コマンドの補完がどこに入っているかというと、自分の環境では M1 mac に Homebrew を使って deno をインストールしたため、次の場所に格納されています。

```shell
❯ greadlink -f (which deno)
/opt/homebrew/Cellar/deno/1.19.1/bin/deno
# homebrew の rack 内の全ファイルをツリー表示
❯ exa /opt/homebrew/Cellar/deno --tree -a
/opt/homebrew/Cellar/deno # <= homebrew の rack
└── 1.19.1 # <= homebrew の keg
   ├── .brew
   │  └── deno.rb # <= homebrew の formula (ruby ファイル)
   ├── .crates.toml
   ├── .crates2.json
   ├── bin
   │  └── deno # <= バイナリファイル(deno コマンドの実体)
   ├── etc
   │  └── bash_completion.d
   │     └── deno
   ├── INSTALL_RECEIPT.json
   ├── LICENSE.md
   ├── README.md
   └── share
      ├── fish
      │  └── vendor_completions.d
      │     └── deno.fish # <= fish shell 用補完スクリプト
      └── zsh
         └── site-functions
            └── _deno
```

- `greadlink -f (which deno)` : `deno` コマンドの配置されているディレクトリからシンボリックリンク先の絶対パスを取得 (`greadlink` コマンドは `brew install coreutils` でインストールできます)
- `exa TARGET --tree -a` : `TARGET` のディレクトリ内のすべてのファイルについてツリー状に表示 (`exa` コマンドは `brew install exa` でインストールできます)

`deno.fish` という completion 用の fish ファイルに `deno` コマンドのすべての補完が配置されていますね。これは、deno の開発チームから提供された fish completion のようです(内部的には[generate in clap_complete::generator](https://docs.rs/clap_complete/latest/clap_complete/generator/fn.generate.html)を使用したコマンド `deno completions fish` で補完スクリプトを自動生成してるみたいです)。ちなみにこのファイルは `/opt/homebrew/share/fish/vendor_completions.d/deno.fish` からシンボリックリンクされています。

```shell
# homebrew を使ってインストールした外部コマンド(vender)から提供されている fish 用補完スクリプトの配置場所(実際にはシンボリックリンク)
# ファイル末尾に @ がついていのはすべてシンボリックリンク
❯ exa -F /opt/homebrew/share/fish/vendor_completions.d/
bat.fish@   deno.fish@  fd.fish@  pipenv.fish@
brew.fish@  exa.fish@   gh.fish@  starship.fish@
# シンボリックリンク先を表示
❯ greadlink -f /opt/homebrew/share/fish/vendor_completions.d/*
/opt/homebrew/Cellar/bat/0.19.0/share/fish/vendor_completions.d/bat.fish
/opt/homebrew/completions/fish/brew.fish
/opt/homebrew/Cellar/deno/1.19.1/share/fish/vendor_completions.d/deno.fish
/opt/homebrew/Cellar/exa/0.10.1/share/fish/vendor_completions.d/exa.fish
/opt/homebrew/Cellar/fd/8.3.2/share/fish/vendor_completions.d/fd.fish
/opt/homebrew/Cellar/gh/2.5.1/share/fish/vendor_completions.d/gh.fish
/opt/homebrew/Cellar/pipenv/2022.1.8/share/fish/vendor_completions.d/pipenv.fish
/opt/homebrew/Cellar/starship/1.3.0/share/fish/vendor_completions.d/starship.fish
```

fish shell の completion については、[`$fish_complete_path` という特殊なリスト変数内に登録されていディレクトリ](https://fishshell.com/docs/current/completions.html?highlight=complete_path#where-to-put-completions)の中を fish は自動的に検索しロードしています。実際に見てみると、`/opt/homebrew/share/fish/vendor_completions.d/` が登録されていることがわかります。

```shell
❯ printf '%s\n' $fish_complete_path
/Users/roshi/.config/fish/completions
/opt/homebrew/Cellar/fish/3.3.1/etc/fish/completions
/opt/homebrew/Cellar/fish/3.3.1/share/fish/vendor_completions.d
/opt/homebrew/share/fish/vendor_completions.d
/opt/homebrew/Cellar/fish/3.3.1/share/fish/completions
/Users/roshi/.local/share/fish/generated_completions
```

なので、`/opt/homebrew/share/fish/vendor_completions.d/` に Homebrew でインストールした補完スクリプトへのシンボリックリンクが配置されることによって、補完が読み込めるようになっています。
:::

# ラップ元のオプションを使用できるようにする
次のように `alias` を定義するだけなら、当たり前ですがラップ元のオプションをそのまま使用できます。

```shell:aliasを使った場合
$ alias deno-run="deno run"
$ deno-run --allow-read --allow-write tests/read_write_test.ts
directory created_dir is created by ensureDir()!
file created_file.md is created by ensureFile()!
```

また、`argparse` を使わなければエラーも無いので簡単にラップ元にオプションを渡せますが、今回はラッパー自体に独自オプションを定義したいので `argparse` ビルトインコマンドとそのオプションである `--ignore-unknown` を使用して関数作成します(`argparse` を使えば色々楽ですしね)。

`--ignore-unknown` は未定義オプションを無視して `$argv` にそのままフラグを残すというオプションです。これによって、ラップ元のオプションについて `argparse` がエラーを吐かずに関数内部へ通過させることができます。

逆に `--ignore-unknown` を使わない場合、例えば以下のように `argparse` を使って引数のパースをするような関数 `bad-pattern` を定義したとします。

```shell
function bad-pattern
    argparse \
        -x 'v,h' \
        'v/version' 'h/help' -- $argv
    or return 1

    set --local version_bad_pattern "v0.0.1"
    echo "argv:" $argv 

    if set -q _flag_version
        echo "bad-pattern: " $version_bad_pattern
    else if set -q _flag_help
        echo "help message"
    else
        command deno run $argv
    end
end
```

この場合、`argparse` のオプション処理対象として定義している `-v, --version` と `-h, --help` についてはうまく処理しますが、未定義のオプション(例えば `deno run` の ` --allow-read` オプションなど)を入力したときにエラーを吐き出します。

```shell
❯ bad-pattern test.ts -v
argv: test.ts
bad-pattern:  v0.0.1
❯ bad-pattern --allow-read --allow-write read_write_test.ts
bad-pattern: Unknown option '--allow-read'
```

`--allow-read` のように `argparse` で処理を定義していないオプションをそのまま `deno run` コマンドにわたすために `--ignore-unknown` オプションを使用してあげます。

```shell
function good-pattern
    argparse --ignore-unknown \
        -x 'v,h' \
        'v/version' 'h/help' -- $argv
    or return 1

    set --local version_good_pattern "v0.0.1"
    echo "argv:" $argv 

    if set -q _flag_version
        echo "good-pattern: " $version_good_pattern
    else if set -q _flag_help
        echo "help message"
    else
        command deno run $argv
    end
end
```

これで、未定義のオプションを `argparse` に無視(通過)させて `deno run` にわたすことができるようになりました。内部的には定義されてある `-v, --version` と `-h, --help` フラグについてはそれが引数として渡されるとフラグ変数 `_flag_version` と `_flag_hel` などをローカルスコープで生成しますが、未定義オプション(`--allow-read` など)は定義済みオプション以外のすべての引数を含む `$argv` に保存されます。

```shell
# 定義済みオプション -v は $argv に格納されず、内部変数 _flag_version に保存される
❯ good-pattern -v
argv:
good-pattern:  v0.0.1
# 未定義オプション --allow-read は $argv にそのまま格納される
❯ good-pattern -v --allow-read
argv: --allow-read
good-pattern:  v0.0.1
❯ good-pattern --allow-read --allow-write read_write_test.ts
argv: --allow-read --allow-write read_write_test.ts
directory created_dir is created by ensureDir()!
file created_file.md is created by ensureFile()!
```

# ANSI escape codeを取り除く

自分としては、これが本来的にやりたかったことなんですが、TypeScript のデバッグに `console.log` の出力をリダイレクションを使って適当なファイルに出力すると、ANSI escape code という制御コードが一緒に出力されてしまって見づらかったので、これを取り除く処理をラッパー内に噛ませます。関数内部で利用する外部コマンドは GNU 実装の `sed` で macOS の場合は `brew install gnu-sed` でインストールすると `gsed` としてインストールされます。

`gsed` と正規表現 `s/\x1b\[[0-9;]*m//g` を組みわせると ANSI escape code が取り除けます。次のようにパイプでフィルターします。

```shell
command deno run $argv | command gsed 's/\x1b\[[0-9;]*m//g'
```

正規表現の参考(&解説): 
https://superuser.com/questions/380772/removing-ansi-color-codes-from-text-stream

これによってリダイレクションしたときに制御コードが紛れずに済みます。これをラッパーのオプションとして `-s, --stdout` というフラグを併用して実行することで使用できるようにします。

あとは `uname -s` と `test` で OS 判定させて Linux と macOS において sed と gsed を切り替えられるようにしておきます。

```shell
# -s, --stdout オプションフラグが渡されたときの処理
if set -q _flag_stdout
    set --local sed_version
    # OS 判定して変数 sed_version に gsed or sed を格納
    if test (uname -s) = "Darwin"
        set sed_version "gsed"
        # gsed がインストールされているかを確認
        if not type --query gsed
            echo "Plase install gnu-sed"
            return 1
        end
    else
        set sed_version "sed"
    end
    # 実行時に $sed_version は sed or gsed に展開される
    command deno run $argv | command $sed_version 's/\x1b\[[0-9;]*m//g'
else
    command deno run $argv
end
```

全体は次のようになります。

:::details functions/deno-run-out.fish 
```shell:functions/deno-run-out.fish
function deno-run-out -d "deno run wrapper"
    # ignore unknown option flags to pass them to deno run command (use -i option in argparse)
    argparse --ignore-unknown \
        -x 'v,h,s' \
        'v/version' 'h/help' 's/stdout' -- $argv
    or return 1

    set --local version_deno_run_out 'v0.1.1'

    if set -q _flag_version
        echo "deno-run-out: " $version_deno_run_out
    else if set -q _flag_help
        __deno-run-out_help
    else if not test (count $argv) -eq 0
        if set -q _flag_stdout
            set --local sed_version
            if test (uname -s) = "Darwin"
                set sed_version "gsed"
                if not type --query gsed
                    echo "Plase install gnu-sed"
                    return 1
                end
            else
                set sed_version "sed"
            end
            command deno run $argv | command $sed_version 's/\x1b\[[0-9;]*m//g'
        else
            command deno run $argv
        end
    else
        echo "Pass a file"
        return 1
    end
end

# helper function
function __deno-run-out_help
    printf '%s\n' \
        'ALIAS:' \
        '      derun' \
        'USAGE:' \
        '      deno-run-out [-v|-h]' \
        '      deno-run-out [-s] [deno-run-OPTIONS...] TARGETFILE' \
        'OPTIONS:' \
        '      -v, --version       Show version info' \
        '      -h, --help          Show help' \
        '      -s, --stdout        Strip ANSI escape code for stdout'
end
```
:::

これで、リダイレクションしてもうまくログを残せるようになりました。(たぶん、もっといい方法があるというか `console` のメソッドそのものの使い方で制御コードが入らないようにできる気がします)

`fihser` を使ってローカルインストールしたらコマンドとして使用できるようになります。

```shell
$ fishe install $PWD
```

実際にリダイレクションで使用してみます。

```shell
❯ derun --allow-read --allow-write read_write_test.ts > tests/new_console_test.log
```

中身を見ると、ANSI escape code が取り除かれています。

```log:tests/new_console_test.log
{ array: [
    1, 2, 3, 4,
    5, 6, 7, 8
  ] }
{ new_item: 8 }
{ removed_item: 8 }
{ array: [
    1, 2, 3, 4,
    5, 6, 7
  ] }
{ unshifted_length: 9 }
{ array: [
    -1, 0, 1, 2, 3,
     4, 5, 6, 7
  ] }
{ shifted_item: -1 }
{ array: [
    0, 1, 2, 3,
    4, 5, 6, 7
  ] }
```

あとは、同じ方法で色々微調整したりします。

余談ですが、`deno doc --builtin` で API のドキュメントを出力して `grep` などをするときに正規表現でうまく検索できず、この `gsed` のパターンを組みわせるとちゃんと検索できました。そのようなケースや、結果をリダイレクションをしたい場合などには `gsed` を活用してみてください。

```shell
❯ deno doc --builtin | gsed 's/\x1b\[[0-9;]*m//g' | grep -e "^\s*interface"
# interface が行頭にある行だけ出力
```

`gsed 's/\x1b\[[0-9;]*m//g'` 自体になにかエイリアスを定義して使うとかが楽そうですね。

