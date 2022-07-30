---
title: "fishで「パスを通す」ための最終解答"
emoji: "🗽"
type: "tech"
topics: [fish, shell, 環境変数, dotfiles, terminal]
published: true
date: 2022-03-06
url: "https://zenn.dev/estra/articles/zenn-fish-add-path-final-answer"
aliases: [記事_fishでパスを通すための最終解答]
tags: " #type/zenn #shell/fish/env  "
---

# はじめに

今回は、fish shell の学習の大詰めとして、環境変数である `PATH` に新しくコマンドサーチ対象ディレクトリを登録する、つまり「パスを通す」方法への最終解答を紹介します。

:::details Changelog
- 2022/03/09 追記
  - 「set -Ux の問題」について書いていた項目に間違いがあったので修正しました。
- 2022/03/12 追記
  - 「変数のスコープルール」の項目を追加し、エクスポートルールの部分を修正しました。
  - 付加的な項目を「研究」にまとめました。
- 2022/03/13 追記
  - v3.4.0 がリリースされ、調整された `fish_user_paths` の項目を修正しました。
  - スコープ関連も追記しました。
:::

パスや環境変数についての話は鬼門で、意外とちゃんと設定するのが難しいです。自分もなかなか上手く行かずに調査を繰り返しました。

fish 関連の記事をいくつか書きましたが、「パスを通すための方法がなぜそうするのか、なぜこれをやってはいけないのか」ということについてはどこを調べても断片的な情報や不正確な情報しかえ得られなかっため、この話題が一番難しいと感じました。そもそもスコープと環境変数を理解する必要があったり、書いていて、自分も勘違いしていた点がいくつかあったことに気付かされました。

:::message
アンチパターンやスコープ・環境変数についてのかなり長い説明があります。急ぎの方は、[結論](#結論)の方に解決策のまとめを記載しているのでそちらを確認してください。
:::

「パスを通す」については、実際に調べてみると色々な方法がでてきますが、現時点(2022/03/06)の最新バージョン `v3.3.1` では最も簡単でシンプルな方法が１つ提示されています。

:::message alert
2022/03/13 `v3.4.0` がリリースされたので、それに合わせて内容を修正しましたが、解決方法に変更はありません。
:::

その答えは、


...
...
...
...
...
...
...
...


`fish_add_path` 関数を使用する。


以上です。

パスを通すにはこれが一番シンプルで良い方法です。その理由については後で説明します。
この `fish_add_path` 関数は fish `v3.2.0` (2021/3/1 リリース)から提供されている比較的新しい関数であり、これ以前の古い記事に記載されていないので注意してください。

https://fishshell.com/docs/current/relnotes.html#fish-3-2-0-released-march-1-2021

# fish_add_path とは

`fish_add_path` 関数は fish の `$PATH` にコンポーネント(パス)追加するシンプルな方法であり、使用することでユニバーサル変数 `$fish_user_paths` または直接的に `$PATH` へと指定したコンポーネントを追加できます(`$PATH` に直接追加する場合には `--path` スイッチを使用する必要があります)。

https://fishshell.com/docs/current/cmds/fish_add_path.html

# fish_add_path の使い方

使い方は次の二通りあります。

- (A) コマンドラインで 1 回だけ実行する
- (B) config.fish に記載する

ドキュメントを見ると fish 側としてはインタラクティブな使用を推している感じですが、`config.fish` に記載する使用方法も用意されていました。

>It is (by default) safe to use fish_add_path in config.fish, or it can be used once, interactively, and the paths will stay in future because of universal variables. 
>`fish_add_path` は `config.fish` 内において(デフォルトで)安全に使用することができ、インタラクティブに一度だけ使用することも可能です。ユニバーサル変数のおかげで、パスは将来も維持されます。  
>[fish documents: fish_add_path](https://fishshell.com/docs/current/cmds/fish_add_path.html#cmd-fish-add-path) より引用

## (A) コマンドラインで 1 回だけ実行する
コマンドラインで追加したいパスにつき 1 回だけ次のように `fish_add_path` を実行します。(後述しますが、実はこの関数については何回実行しても大丈夫です)

```shell:コマンドライン
❯ fish_add_path $HOME/.deno/bin
```

これによって、ユニバーサル変数である `fish_user_paths` にコンポーネントを追加します。追加方法はデフォルトで "prepend" つまり先頭に追加します。上のコードのように新しくパスを追加した場合には `$HOME/.deno/bin` が先頭になりコマンドサーチの最優先として登録されます。

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

## (B) config.fish に記載する

環境変数だけでなく `PATH` も一緒にファイルで管理したい場合はこちらの方法を行います。`~/.config/fish/config.fish` に `fish_add_path TARGET` を記載するだけです。 

```shell:config.fish
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

これによって、再ログインや新規シェルセッションの立ち上げ時 `fish_user_paths` に指定したパスが保持されていることを保証できます。

`fish_add_path` 関数が賢いのは、指定したコンポーネント(パス)が `$PATH` に存在している場合は再度追加せずに重複を防ぎ、`$PATH` 内のコンポーネント位置を保つことが可能である点です。

>If a component already exists, it is not added again and stays in the same place unless the --move switch is given.  
>[fish documents: fish_add_path](https://fishshell.com/docs/current/cmds/fish_add_path.html) より引用

上記のコードの例では、パスに何も入れていない状態から fish を立ち上げた場合に、4 つのコンポーネントが上から順番に追加された状態でセッション内の `$PATH` に保持されていることを保証できます。コンポーネントはファイルの上から次のような順番になっています。

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

>Components are added in the order they are given, and they are prepended to the path unless `--append` is given  
>コンポーネントは指定された順番に追加され、`--append` スイッチが使用されない限りパスの先頭に追加されます。  
>[fish documents: fish_add_path](https://fishshell.com/docs/current/cmds/fish_add_path.html) より引用

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
# Deno 用
- fish_add_path $HOME/.deno/bin
+ # fish_add_path $HOME/.deno/bin # コメントアウトしておく
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

:::message alert
`fish_add_path` 関数は指定したコンポーネント(つまりディレクトリ)が存在していない場合は、追加せずに無視するため、ディレクトリを作成しておく必要があります。存在しないディレクトリをパスに追加するようにしたいなら、次のようにコマンドラインから一度だけ `set -U` を実行します。

```shell:コマンドライン
# $HOME/.fake/bin は存在しないディレクトリ
❯ set -U fish_user_paths $HOME/.fake/bin $fish_user_paths
```
:::

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

これは実行するたびに位置を変えてしまう可能性があるので、`config.fish` には記載せずに、コマンドラインから直接実行するようにしましょう。

この後は、スコープルールや `PATH` のアンチパターンの話に移りますが、急ぎの方は[結論](#結論)の方に解決策のまとめを記載しているのでそちらを確認してください。

# 変数のスコープルール

`PATH` のアンチパターンについて解説する前に、変数のスコープの種類について解説しておきます。

fish のシェル変数には主に３つのスコープがあります。

- ユニバーサル
	- ユニバーサル変数は 1 つのコンピューター上でユーザーが実行しているすべての fish セッション間でシェアされます。
	- `set -U` または `set --universal` で明示的にセットできます。
- グローバル
	- グローバル変数は現在の fish セッションに固有であり、`set -e` コマンドで明示的に要求されない限り決して消えません。
	- `set -g` または `set --global` で明示的にセットできます。
- ローカル
	- ローカル変数は現在の fish セッションに固有であり、特定のコマンドブロックに関連付けられています。その特定のブロックがスコープがスコープ外になると変数は自動的に削除されます。コマンドのブロックは、for, while, if, function, begin, switch などから始まり、end コマンドで終了する一連のコマンドです。
	- `set -l` または `set --local` で明示的にセットできます。

変数の定義・修正・利用時には以下の基本的なスコープルールが適用されます。

- スコープが明示的に指定された場合、そのスコープが使用されます。異なるスコープに同一名の変数が存在してもその変数については変更されません。
- スコープが指定されず、同一の変数名が存在する場合、それらのうち最も小さなスコープの変数が修正される。スコープ自体は変更されません。
- 特殊なケースとして、初めて定義される変数のスコープが指定されない場合、現在実行している関数のスコープに所属します。これは `-l` や `--local` フラグとは異なり、変数を現在のコードブロックにローカルなスコープとなります。関数が実行されていない場合には、変数はグローバルになります。

https://fishshell.com/docs/current/language.html#variable-scope

:::message
`local` フラグによって指定できるローカルスコープは実際にはブロックスコープなのですが、関数のトップレベルに対してローカルにするためには、以前は関数トップで定義するか、またはスコープを指定せずに定義する必要がありました。

しかし、`v3.4.0` では `set` と `read` ビルトインコマンドに `--function` というオプションが追加され、これによって「関数内のトップレベルにローカルである」と明示的に変数スコープを指定できるようになりました。

>```shell
>function demonstration
>    if true
>        set --function foo bar
>        set --local baz banana
>    end
>    echo $foo # prints "bar" because $foo is still valid
>    echo $baz # prints nothing because $baz went out of scope
>end
>```
>`set` and `read` learned a new option, `--function`, to set a variable in the function’s top scope. This should be a more familiar way of scoping variables and avoids issues with `--local`, which is actually block-scoped ([#565](https://github.com/fish-shell/fish-shell/issues/565), [#8145](https://github.com/fish-shell/fish-shell/issues/8145)):  
> [Release fish 3.4.0 (released March 12, 2022)](https://github.com/fish-shell/fish-shell/releases/tag/3.4.0) より引用
:::


従って、スコープを明示せずに、コマンドラインや `config.fish` ファイルにて変数を定義した場合、グローバルスコープになります。

```shell
❯ set globalvar グローバル
# set -S で対象の変数について調査できる
❯ set -S globalvar
$globalvar: set in global scope, unexported, with 1 elements
$globalvar[1]: |グローバル|
```

次のようにスコープを明示して、異なるスコープにおいて同一名の変数を作成できます。

```shell
# 異なるスコープで同じ名前の変数を作成してみる
❯ set -l testvar 123
❯ set -g testvar 456
❯ set -U testvar 789
```

このように異なるスコープで同じ名前の変数が存在できるため、変数を使用する際には、同じ名前の変数の内で最も小さいスコープのものが使用されます。ローカル変数が存在しているなら、グローバルやユニバーサルの同一名の変数の代わりにローカル変数が使用されます。

```shell
❯ set -S testvar
$testvar: set in local scope, unexported, with 1 elements
$testvar[1]: |123|
$testvar: set in global scope, unexported, with 1 elements
$testvar[1]: |456|
$testvar: set in universal scope, unexported, with 1 elements
$testvar[1]: |789|
❯ echo $testvar
123
# 最も小さいスコープ(ローカルスコープ)の変数が存在するのでローカル変数が参照される
```

このように、fish のスコープルールはより小さいスコープ(内側)から大きいスコープ(外側)へと検索が向かうため "inside out" と呼ばれています。

# PATHのアンチパターン

`config.fish` における `PATH` の書き方の間違いは次の３つのパターンがあります。
※ あくまで `config.fish` においての話ですので注意してください。コマンドラインから行う場合についてもそれぞれで説明します。

- (1)パスがそもそも通らない
- (2)パスは通るが、細かい点で不備がある
- (3)パスは通るが、時間が経つにつれて遅くなる

細かい理由については、あとで説明しますが、まずはそれぞれのパターンを見てみます。

(1)パスがそもそも通らない。
`config.fish` で次のように書いてもパスは通りません。

```shell:config.fish
set -U PATH $HOME/.deno/bin $PATH
set -Ux PATH $HOME/.deno/bin $PATH
```

(2)パスは通るが細かい点で微妙。
`config.fish` で次のようにグローバルスコープで `PATH` を設定するのはやってはいけません。パスは通りますが、実は細かいところでパスが重複するケースがあります。

```shell:config.fish
set PATH $HOME/.deno/bin $PATH
set -x PATH $HOME/.deno/bin $PATH
set -g PATH $HOME/.deno/bin $PATH
set -gx PATH $HOME/.deno/bin $PATH
```

(3)パスは通るが、時間が経つにつれて遅くなる
これがアンチパターンの中で後々問題になるものです。次のように `config.fish` にユニバーサル変数 `fish_user_paths` に対して `set -U` をしてはいけません。上記２パターンよりも更にやってはいけません。時間が経つにつれて起動や処理が遅くなる可能性があります。

```shell:config.fish
set -U fish_user_paths $HOME/.deno/bin $fish_user_paths
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
❯ set -S LANG
$LANG: set in global scope, exported, with 1 elements
$LANG[1]: |ja_JP.UTF-8|
```

環境変数は慣習的に大文字ですが、上のコードの通り、すべてグローバルスコープで exported されている(つまり、環境変数になっている)ことが分かります。

このように、fish 自体が起動する際に、親プロセス(通常は terminal)から環境変数を継承します。それらには `PATH` やロケール変数などのシステム構成が含まれおり、fish は**それらをグローバル変数として変換してセッションにて保持します**。

:::message
シェル変数は "export" することで、環境変数として利用できるようになります。環境変数であることは、ローカル・グローバル・ユニバーサルといったスコープ自体とは関係なく、シェル変数の特定の状態を指します。なので、シェル変数自体はグローバル・ローカル・ユニバーサルのスコープに関わらず、あらゆるスコープでエクスポートできます。スコープによって変わるのは、その環境変数がどのように利用できかという点です。
参考: [fish Documents: How do I set or clear an environment variable?](https://fishshell.com/docs/current/faq.html#how-do-i-set-or-clear-an-environment-variable)  

スコープによる挙動の違いは次のようになります。
- シェル変数がグローバルまたはユニバーサルでエクスポートされている場合は、すべての関数がその変数にアクセスでき、それぞれの変数の保持はスコープ通りになる。
- シェル変数がグローバルでもユニバーサルでもないがエクスポートされている場合は、呼び出された関数は関数にローカルなその変数のコピーを受け取る(関数によってそれらの値が変更されたとしてもコピー元の値は変わらない)  

参考: [faho commented on 25 Aug 2016](https://github.com/fish-shell/fish-shell/issues/1091#issuecomment-242213527)

シェル変数を "export" するには、`set` ビルトインコマンドの `-x, --export` スイッチを使用します。
:::

`PATH` 環境変数についてユニバーサルでエクスポートするのは、ユニバーサル変数のすべてのセッションにおいて保持されるという性質から、理想的のように思えますが、以下に述べる理由から避けるべきです。

## (1)パスがそもそも通らない

:::message alert
`set -Ux` の問題という見出しで書いていましたが修正しました。
また、この項目については、自分も書いていて勘違いしていた部分がいくつかありましたのでそちらについても修正しました。
:::

次の(1)のパターンでは、そもそもパスが通りません。

```shell:config.fish
# そもそも、これらのパターンではパスが通らない
set -U PATH $HOME/.deno/bin $PATH
set -Ux PATH $HOME/.deno/bin $PATH
```

ここでやってることの意味としては、「`PATH` という名前の変数をユニバーサルスコープで定義し、(export された状態にし)、値 `$PATH` と `$HOME/.deno/bin` を順番に `PATH` へセットする」ということです。

~~結論から述べますが、`PATH` という環境変数について、`config.fish` などのファイルに次のように記載すると fish の起動や再ログインのたびに読み込まれるのため、ユニバーサル変数に毎回コンポーネントが追加されることでパスがどんどん長くなっていきます。~~
**修正**: この場合は、パスが重複追加されるのではなくそもそも「パスが通らない」ということが問題でした。なぜなら環境変数 `PATH` 自体がグローバルスコープで環境から継承されるため、そちらの値が実際には優先され、ユニバーサル変数の `PATH` にいくら値を追加しようと無視されるからです。

[PATHのアンチパターン](#PATHのアンチパターン)の冒頭で説明しましたが、`PATH` はグローバル変数です。[変数のスコープルール](#変数のスコープルール) で説明したように fish では "inside out" のスコープルールで、参照される変数は local → global → universal という順番です。つまり、`set -U PATH $HOME/.deno/bin $PATH` で `$PATH` で参照されるのはグローバルスコープに定義された `PATH` の値となります。`set -U` で宣言した `PATH` がユニバーサル変数であっても、その行の最後に参照されている値はグローバル変数の値となります。したがって、ユニバーサル変数 `PATH` に追加したいパスとグローバル変数 `PATH` の値をセットしていることになります。

ユニバーサル変数は全シェルセッションで共有され、永続化しているので、あるシェルセッションを開始した際に、`config.fish` が読み込まれると `PATH` の値に `$HOME/.deno/bin` が追加されます。~~さらに別新しいシェルセッションを開始すると、再び `config.fish` を読み込み、`PATH` に `$HOME/.deno/bin` を追加します。これによって、`PATH` には重複した `$HOME/.deno/bin` がいくつも追加された状態となります。~~  
**修正**: 実際にはグローバル変数 `PATH` の値は環境から継承されるので、fish 起動時に生成されます。その生成されたグローバル変数 `PATH` の値が、起動時に実行される `set -U PATH $HOME/.deno/bin $PATH` によってユニバーサル変数 `PATH` へと追加されるため、パス自体がどんどん長くなるということはありませんでした。ただ、scope shadowing の問題からパスが通らないというだけです。

:::message
Scope Shadowing の問題。  

変数のスコープルールとして "inside out" つまりローカルからグローバル、そして最後にユニバーサルという優先順位でサーチが行われるため、グローバルな環境変数と同一名のユニバーサルな環境変数があった場合、グローバル環境変数が優先されて参照されます。 これにってユニバーサルな環境変数の値を変更してもグローバルな環境変数が存在するため、変更が反映されないという "scope shadowing" と呼ばれる問題が引き起こされます。  

>Environment variables such as EDITOR or TZ can be set universally using set -Ux. However, if there is an environment variable already set before fish starts (such as by login scripts or system administrators), it is imported into fish as a global variable. **The variable scopes are searched from the "inside out", which means that local variables are checked first, followed by global variables, and finally universal variables**.  
>[fish Documents: Why doesn't set -Ux (exported universal variables) seem to work?](https://fishshell.com/docs/current/faq.html#why-doesn-t-set-ux-exported-universal-variables-seem-to-work) より引用

```shell
❯ set -gx NEWVAR global ; echo $NEWVAR
global
# グローバルスコープが参照される
❯ set -Ux NEWVAR universal ; echo $NEWVAR
global
# グローバルスコープが先に削除される
❯ set -e NEWVAR; echo $NEWVAR
universal
❯ set -e NEWVAR; echo $NEWVAR
```  

参考: [zanchey commented on 22 May 2013](https://github.com/fish-shell/fish-shell/issues/806#issuecomment-18267479)  
:::

`set -U` と `set -Ux` の違いは、`-x` によって環境変数という状態にするか否かですが、どちらもユニバーサルスコープで定義するため、結局はグローバルスコープに定義された `PATH` からシャドーされてどちらもパスが通りません。

また、公式ドキュメントでは以下のように `config.fish` ではユニバーサル変数へ値を追加しないようにと注意書きがなされています。

>Do not append to universal variables in config.fish, because these variables will then get longer with each new shell instance. Instead, simply set them once at the command line.  
> [fish documents: More on universal variables](https://fishshell.com/docs/current/language.html#more-on-universal-variables)

ユニバーサル変数のセットは 1 回だけ行えば永続化・伝播します。従って、そのような操作をする場合には、代わりにコマンドラインから行うようにとも書かれていますね。

```shell:コマンドライン
❯ set -Ux EDITOR vim
```

一部の環境変数については、コマンドラインからでも `set -U` や `set -Ux` を行わない方がいいです。元々環境から継承される値については、グローバル変数の方が優先される scope shadowing という問題が頻繁に issue でも言及されています。同一名の環境変数がグローバルスコープにあるとユニバーサルスコープの環境変数よりも優先されるため、ユニバーサルスコープで環境変数の値を変更しても読み取れません。

参考: [fish Documents: Why doesn't set -Ux (exported universal variables) seem to work?](https://fishshell.com/docs/current/faq.html#why-doesn-t-set-ux-exported-universal-variables-seem-to-work)

`TERM` などの環境変数はすでに環境から継承されグローバル変数として存在しているため、このケースにあたります(継承元は親プロセス)。

https://github.com/fish-shell/fish-shell/issues/806

この scope shadowing の問題から、`config.fish` 内において環境変数の定義をする場合には、グローバルスコープでの export `set -gx` の使用を推奨しています。`PATH` 以外の環境変数においては次のように `config.fish` に記載するのが良いです。

```shell:config.fish
set -gX EDITOR vim
```

:::message alert
このように、**コマンドラインからの実行と `config.fish` 内での作業が全く異なる行為であることから、多くのユーザーにとっての混乱の元凶になっています**。さらに、**環境変数の中でも `PATH` は特別扱いで、他の環境変数とは違う方法を取る必要がある**ので、余計混乱させています。
:::

個人的には、scope shadowing の問題があるため、コマンドラインからも迂闊に `set -Ux` すべきではなく、`set -Ux` の使用は基本的にやるべきではない考えます。これが役に立つケースは同時に起動しているすべてのシェルセッションにおいて、再起動をする必要なく一気に環境変数を変更できるというようなケースに絞られます。

## (2)パスは通るが、細かい点で不備がある

:::message alert
「`set -gx` の問題」について取り上げていましたが、見出し名を「(2) パスは通るが、細かい点で微妙」に変更しました。
:::

ユニバーサル変数の `PATH` はダメなので、`config.fish` でグローバル変数にしたり(`set -g`)、グローバルとしてエクスポート(`set -gx`)ならいいのかというと、そういう訳でも無いです。パス自体は通りますが、細かい点で不備があります(とは言っても別に気にしなければ良い、というレベルの問題ですが)。

```shell:config.fish
set PATH /opt/homebrew/bin $PATH
set -g PATH /opt/homebrew/bin $PATH
set -x PATH /opt/homebrew/bin $PATH
set -gx PATH /opt/homebrew/bin $PATH
```

まず、スコープを明示せずに変数の宣言をした場合、関数内などに定義していれば関数にローカルになりますが、`config.fish` 内にべた書きしているのでグローバルスコープとしてみなされるはずです。なので、`set PATH` と `set -g PATH` は同じですし、`set -x` と `set -gx` は同じ意味になります。

細かいスコープルールについては[変数のスコープルール](#変数のスコープルール)で解説したので参照してください。

そして、`PATH` 環境変数はそもそもグローバル変数として環境そのものから継承され生成されるため、すでにグローバルスコープであり「環境変数である」という状態になっています。

すでにエクスポートされている変数の値について変更する場合には、`-x` を明示する必要はありません。エクスポートにはスコープのルールと同様にエクスポートルールというものがあり、次のようになっています。

- 変数はエクスポートされるかエクスポートされないようにするかを明示的に設定できます。エクスポートされた変数がスコープ外になると、エクスポートされません。
- 変数が明示的にエクスポートまたはエクスポートされないよう設定されおらず、以前に定義されている場合、以前のエクスポートルールが保持されます。
- 変数が明示的にエクスポートまたはエクスポートされないよう設定されおらず、初めて定義される場合、変数はエクスポートされません。

https://fishshell.com/docs/current/cmds/set.html

:::message alert
`config.fish` にてスコープを明示せずにエクスポート、つまり `set -x` をしてしまうと、`source` のラッパー関数などで読み込みを行った際にローカルスコープとして評価される場合があるので、スコープは明示するようにしましょう。

https://github.com/fish-shell/fish-shell/issues/1908

手動で、`source ~/.config/fish/config.fish` を行った場合にはグローバルスコープとして読み込まれます。
:::

`set -S` で変数のスコープとエクスポートされているか分かりますが、次のように環境変数 `LANT` を `-x` なしで値を変更してもエクスポートされたままであることがわかります。

```shell
❯ set -S LANG
$LANG: set in global scope, exported, with 1 elements
$LANG[1]: |ja_JP.UTF-8|
❯ set LANG en_US
❯ set -S LANG
$LANG: set in global scope, exported, with 1 elements
$LANG[1]: |en_US|
```

`PATH` もそもそも環境変数なので、`set -g PATH` も `set -gx PATH` も実質的には同じ意味になります。つまり、以下のコードの意味はすべて同じです。

```shell:config.fish
# 以下すべて同じ結果になる
set PATH /opt/homebrew/bin $PATH
set -g PATH /opt/homebrew/bin $PATH
set -x PATH /opt/homebrew/bin $PATH
set -gx PATH /opt/homebrew/bin $PATH
```

ということで、すべて同じなので、分かりやすくスコープとエクスポートが明示されている `set -gx PATH /opt/homebrew/bin $PATH` を例にとって微妙な点について説明します。

先述したようにパス自体は通ります。次のように `config.fish` へ記載したとします。

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

一度、終了してから再び `PATH` を見てみると、再び環境変数が継承されるので `PATH` がグローバルに生成されます。`config.fish` に `set -gx PATH /opt/homebrew/bin $PATH` が記載されている限り、 fish を起動すれば次のように意図通りの `PATH` になっているはずです。

```shell
❯ printf '%s\n' $PATH
/opt/homebrew/bin
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

ここまでは一見良さそうですが、この状態で、 `source ~/.config/fish/config.fish` などをやると、次のようになります。

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

環境変数は子プロセスに継承され、さらに fish 立ち上げの際に `config.fish` が読み込まれるはずなので、上記のように `PATH` には `/opt/homebrew/bin` が更に追加された状態になります。パスは通るが、細かい点で不備があるというのはこういう話です。

このようなわけで、`config.fish` において `set -gx` で `PATH` の設定は行うべきでありません(ただし、これを行ったとしても上述のようにグローバル変数はセッション終了で消滅するので、大きな問題にはなりません)。

また、`set -gx` を使用すべきではないというのは、`PATH` という特殊な環境変数についてのみです。それ以外の環境変数については `set -gx` が推奨されます。こういうわけで同じように `PATH` 環境変数が `set -gx` でセットされてしまっていることが多いようです。

```shell:config.fish
set -gx VOLTA_HOME $HOME/.volta
# ログインのたびに評価されていくが、値がのびていくようなことはない
```

また、先述の通り、スコープの明示をせずに `config.fish` にベタ書きするとスコープはグローバルになるので、スコープを明示しないパターンも見かけます(混乱するのでスコープ明示したほうがいいです)。

```shell:config.fish
# グローバルスコープなので、上と同じ
set -x VOLTA_HOME $HOME/.volta
```

`PATH` は以前の `PATH` の値を使用して追加していくというやり方なので、`set` してはならないということです。リストになっていて、前のリストの値を追加するという操作をするのでパスが長くなっていきますが、上記のような処理 `set -gx VOLTA_HOME $HOME/.volta` は `PATH` のように以前のリストの値に追加していくというやり方ではないので OK です。なので例えば `source ~/.config/fish/config.fish` のように `source` コマンドで再度読み込んでも大丈夫です。

`PATH` のように元の値に追加していく形式の環境変数は特殊なため `set -gx` を `config.fish` に記載してはいけません。もしそのような環境変数が `PATH` 以外にもあれば `set -gx` を `config.fish` で使用する場合に前の値を使わずにすべての値を書く、またはコマンドラインから `set -Ux` を一度実行して、さらにグローバルスコープに同一名の変数が無いことを保証する必要があります。

`PATH` について、`config.fish` ファイルに記載する際には、`set -Ux` もダメ、`set -gx` もダメ、`set -x` はグローバルスコープになりますが、ダメです。

コマンドラインから**一時的にパスを通す**なら次のように 1 回だけ実行してください。

```shell:コマンドライン
❯ set -gx PATH $HOME/.deno/bin $PATH
```

コマンドラインでこれを実行すればそのシェルセッションにおいては、パスが通ります。この場合、`exit` すれば消滅し永続しません。

コマンドラインからやる場合でも、スコープを明示しなかったり、エクスポートを明示しなかったりするパターンなどが紹介されていることもあるので混乱しないように注意してください。

```shell:コマンドライン
# PATH は fish 起動時に環境から継承されており(すでに環境変数という状態になっている)、
# グローバルスコープで保持されているため
# 以下すべて同じ結果となる
❯ set PATH $HOME/.deno/bin $PATH
❯ set -g PATH $HOME/.deno/bin $PATH
❯ set -x PATH $HOME/.deno/bin $PATH
❯ set -gx PATH $HOME/.deno/bin $PATH
```

一時的にパスを通すようなケースを行わない場合には、`fish_add_path` を一貫して使うようにした方が混乱せずに済みます。

## 環境変数についての余談

環境変数は、コマンドラインの文脈で言えば、ユーザーにとっては主に外部コマンドが参照するための仕組みです。環境変数は、そのシェルのコンテキスト内で実行されるすべてのコマンドによって継承されます。

一部の外部コマンドはその継承される環境変数に依存しています。例えば、`git` や `man` などの外部コマンドは `$PAGER` 変数からどの pager(テキストスクロールするプログラム)を使用するか判別していますし、`$BROWSER` (使用するブラウザ) や `$LANG` (使用言語)、なども他の外部コマンドの一部で使用されています。外部コマンドでの環境変数の参照は、関数内でシェル変数を参照するとは異なるため環境変数は普通のシェル変数とは異なり export される必要があります。そもそも `set -x` された変数が環境変数という状態になります。

外部コマンド(プログラム)から利用されない、関数内のローカル変数やシェル自体の設定用のグローバル変数やユニバーサル変数は環境変数という状態にする必要はありません。

環境変数という状態にする必要のある変数のみ `set -x` でエクスポートします。

:::details 環境変数が実際に継承されているか見てみる
環境変数について理解するのが意外と難しいので、少し脱線して、環境変数が実際に継承されるのかを見てみましょう。
`pstree` 外部コマンドを使用してプロセスの関係性を把握し、実際に環境変数が継承されるか見てみます。macOS なら、Homebrew を次のように使用してインストールできます。

```sh
$ brew install pstree
```

この `pstree` は、「現在動作しているプロセスをツリー形式で表示するコマンドのプロセスからどのプロセスが起動しているのか」という親子関係を表示できます。

`fish_pid` という fihs の提供する特殊変数に現在実行中のシェルプロセスの PID(process ID) が格納されているので、その値を使って、プロセス(プログラムのインスタンス)間の関係を表示してみます。`pstree -p PID` で指定した PID の親と子孫のみを表示できます。 
自分は `tmux` を使用しているので、現在実行中のプロセスの親子関係を表示すると次のようになります。

```shell
❯ set -S fish_pid
$fish_pid: set in global scope, unexported, with 1 elements
$fish_pid[1]: |03962|
❯ pstree -p $fish_pid
-+= 00001 root /sbin/launchd
 \-+= 03441 roshi tmux # tmux の PID
   \-+= 03962 roshi /opt/homebrew/bin/fish -l # これがログインシェル fish の PID
     \-+= 79269 roshi pstree -p 3962 # 今実行した pstree の PID
       \--- 79271 root ps -axwwo user,pid,ppid,pgid,command # 内部的に呼び出した ps コマンドの PID
```

`pstree` という外部コマンドをシェルから呼び出すことで、fish shell のプロセスから `pstree` 子プロセス(`79269`)を作成し、さらにそのプロセスは自身も子プロセス(`79271`)を作成して `ps` コマンド(process の status を表示する外部コマンド)を実行していることがわかります。

この状態から、環境変数 `NEWTEST` を作成し、値を `newvalue` をセットしてみます。

```shell
# スコープを明示せずにエクスポートするとグローバルスコープ
❯ set -x NEWTEST newvalue
❯ set -S NEWTEST
$NEWTEST: set in global scope, exported, with 1 elements
$NEWTEST[1]: |newvalue|
# 子プロセスでfishを立ち上げる
❯ fish
Welcom to fish shell 3.3.1
❯ pstree -p $fish_pid
-+= 00001 root /sbin/launchd
 \-+= 03441 roshi tmux
   \-+= 03962 roshi /opt/homebrew/bin/fish -l # 親プロセス(元々のログインシェル)の PID
     \-+= 82945 roshi fish # 現在のシェルの PID
       \-+= 82982 roshi pstree -p 82945
         \--- 82984 root ps -axwwo user,pid,ppid,pgid,command
❯ echo $fish_pid
82945
```

この子プロセスの fish で環境変数 `NEWTEST` について `set -S` で状態を見てみると、スコープも同じくグローバルで、親プロセスで設定した値 `newvalue` が引き継がれ、実際に環境変数として継承されていることがわかります。

```shell
❯ echo $fish_pid
82945
❯ set -S NEWTEST
$NEWTEST: set in global scope, exported, with 1 elements
$NEWTEST[1]: |newvalue|
```

さらに、このプロセスにおいて環境変数を作成し、`exit` して親プロセス(3962)に戻ってみると子プロセス(82945)で定義した環境変数は参照できず未定義の状態であり、つまり継承されていないことが分かります。

```shell
# このプロセス(82945)にてエクスポートするとグローバル
❯ set -x NEWTESTCHILD newvalue_child
❯ set -S NEWTESTCHILD
$NEWTESTCHILD: set in global scope, exported, with 1 elements
$NEWTESTCHILD[1]: |newvalue_child|
# 子プロセスからexitして親プロセスに戻る
❯ exit
# pid を確認すると親プロセスに戻ったことがわかる
❯ echo $fish_pid
3962
# 子プロセスで定義したグローバルな環境変数 NEWTESTCHILD を見てみると未定義になっており、何も表示されない
❯ set -S NEWTESTCHILD
# 元々このプロセスで定義したグローバルな環境変数 NEWTEST は継続している
❯ set -S NEWTEST
$NEWTEST: set in global scope, exported, with 1 elements
$NEWTEST[1]: |newvalue|
❯ pstree -p $fish_pid
-+= 00001 root /sbin/launchd
 \-+= 03441 roshi tmux
   \-+= 03962 roshi /opt/homebrew/bin/fish -l
     \-+= 83435 roshi pstree -p 3962
       \--- 83437 root ps -axwwo user,pid,ppid,pgid,command
# exec $SHELL -l でシェルを新しくしてみる
❯ exec $SHELL -l
Welcom to fish shell 3.3.1
# 環境変数は継続
❯ set -S NEWTEST
$NEWTEST: set in global scope, exported, with 1 elements
$NEWTEST[1]: |newvalue|
```

調査の結果、環境変数は、親プロセスから子プロセスにしか継承されず、子プロセスで作成した環境変数は親に伝播しないということが分かります。外部コマンドを呼び出す際には、シェル変数が環境変数の状態となっていれば、呼び出し元の親プロセス(fish shell)から継承され、外部コマンドはその値を参照できます。

例えば、Volta(JavaScript の toolchain 管理ツール)について言えば、Volta は環境変数 `VOLTA_HOME` というものを必要とします。ただし、こういった外部コマンドはデフォルト値が設定されている場合があるので環境変数として明示的に定義しなくても使えるケースが多く、環境変数を設定することで調整ができるというような感じでしょう。`volta install` などのコマンド実行時に、環境変数として定義された `VOLTA_HOME` にセットされているディレクトリを参照してインストールできます。

```shell:config.fish
set -gx VOLTA_HOME $HOME/.volta
```

上のように `config.fish` で `VOLTA_HOME` 環境変数を明示的に設定することで任意の場所にインストールできます。`pstree` を使った見たプロセス関係のように環境変数が親プロセスであるログインシェルから継承されることによって実現できています。
:::

## (3)パスは通るが、時間が経つにつれて遅くなる

:::message alert
「set -U fish_user_paths の問題」という見出しで書いていましたが、修正しました。
:::

ユニバーサル変数に対して `config.fish` 内で値を追加するような操作を記載してはいけません。従って `fish_add_path` が利用しているユーザーが追加したパスを管理しているユニバーサル変数である `fish_user_pashs` に対しても次のようなことを `config.fish` でやってはいけません。このパターンでは、時間が経つにつれて fish 自体が遅くなる可能性があります。

```shell:config.fish
set -U fish_user_paths $HOME/.deno/bin $fish_user_paths
```

具体的に何が悪いかのを説明します。
まず、上記のコードを `config.fish` に書いたとして、最後の `$fish_user_paths` はユニバーサル変数の値を展開するので、次のようにログインするたびに、パスが重複追加されていきます。

```shell
❯ printf '%s\n' $fish_user_paths
/Users/roshi/.deno/bin
/Users/roshi/.deno/bin
/Users/roshi/.cargo/bin
/Users/roshi/.volta/bin
/Users/roshi/.deno/bin
/opt/homebrew/bin
```

この状態で、 `source ~/.config/fish/config.fish` などをやると、もう一度 `set -U fish_user_paths $HOME/.deno/bin $fish_user_paths` が実行されので、次のようになります。

```shell
❯ printf '%s\n' $PATH
/Users/roshi/.deno/bin
/Users/roshi/.deno/bin
/Users/roshi/.deno/bin
/Users/roshi/.cargo/bin
/Users/roshi/.volta/bin
/Users/roshi/.deno/bin
/opt/homebrew/bin
```

この理由は、`fish_user_paths` がユニバーサル変数だからです。ユニバーサル変数は全シェルセッションで共有され、永続化しているので、あるシェルセッションを開始した際に、`config.fish` が読み込まれますが、そのたびに `set -U fish_user_paths $HOME/.deno/bin $fish_user_paths` が実行され、永続化されている `fish_user_paths` の値に毎回追加されてます。

先述した引用では「パスがどんどん長くなる」と言及されていますが、完全にこのパターンのことです。

>Or you can modify $fish_user_paths yourself, but you should be careful not to append to it unconditionally in config.fish, or it will grow longer and longer.
>`$fish_user_paths` を自身で修正することも可能ですが、`config.fish` では無条件で追加しないように注意する必要があります。そうしないと、`$fish_user_paths` はどんどん長くなっていきます。  
>[fish documents: $PATH](https://fishshell.com/docs/current/tutorial.html#path) より引用

もし `config.fish` で `set -U fish_user_paths` をやるなら `PATH` のときに説明したようにすべての要素を書くようにします。

```shell:config.fish
set -U fish_user_paths $HOME/.deno/bin $CARGO_HOME/bin /opt/homebrew/bin
```

:::message
2022/03/13 : `v3.4.0` のリリースに合わせて追記しました。

`v3.3.1` 以前はこの項目で説明した通り、パスがどんどん長くなるということがありましたが、多くのユーザーがこの間違いをしてしまうという理由から `v3.4.0` でパスが重複追加されないように開側が調整したそうです。従って、次のように `config.fish` に記載してもパスが長くなりません。

```shell:config.fish
set -U fish_user_paths $HOME/.deno/bin $fish_user_paths
```

>$fish_user_paths is now automatically deduplicated to fix a common user error of appending to it in config.fish when it is universal ([#8117](https://github.com/fish-shell/fish-shell/issues/8117)). fish_add_path remains the recommended way to add to $PATH.  
>[Release fish 3.4.0 (released March 12, 2022)](https://github.com/fish-shell/fish-shell/releases/tag/3.4.0) より引用

ただし、`fish_user_path` 関数が依然として推奨方法であるとのことです。

faho 氏の実際のプルリクエスト↓
[Deduplicate $fish_user_paths automatically #8117](https://github.com/fish-shell/fish-shell/pull/8117)

コメントが中々に辛辣です笑。faho 氏は、これについてはドキュメントに記載もしないし、修正したいとも言っているので、この方法は使わない方がいいです。
:::

あと、`fish_user_paths` は環境変数ではないので、`-x` を付けないようにしてください。

```shell
❯ set -S fish_user_paths
$fish_user_paths: set in universal scope, unexported, with 4 elements
$fish_user_paths[1]: |/Users/roshi/.cargo/bin|
$fish_user_paths[2]: |/Users/roshi/.volta/bin|
$fish_user_paths[3]: |/Users/roshi/.deno/bin|
$fish_user_paths[4]: |/opt/homebrew/bin|
```

また、同じ様にコマンドラインからやる場合には 1 回のみ使用するのに限って OK です。永続的なパスが通ります。

```shell:コマンドライン
❯ set -U fish_user_paths $HOME/.deno/bin $fish_user_paths
```

しかし、やはり混乱するので、`fish_add_path` 関数を使用するのが推奨です。

:::message alert
注意点として、`fish_user_paths` はユニバーサル変数で元々定義されているのでグローバルで宣言してはいけません(そもそも宣言する必要はありませんが)。

```shell
❯ set -S fish_user_paths
$fish_user_paths: set in universal scope, unexported, with 4 elements
$fish_user_paths[1]: |/Users/roshi/.cargo/bin|
$fish_user_paths[2]: |/Users/roshi/.volta/bin|
$fish_user_paths[3]: |/Users/roshi/.deno/bin|
$fish_user_paths[4]: |/opt/homebrew/bin|
```

例えば `set -g fish_user_paths` みたいなことをやれば scope shadowing でそちらが優先され、`fish_add_path` をやってもパスを通せなくなります。グローバル変数はセッション終了まで生存するので、そのセッションを終了するか、グローバルスコープから削除してください。`set -e` で削除する際にはスコープを明示すればよいです。

```shell
❯ set -e -g fish_user_paths
```

スコープを明示しない場合には、local → global → universal というスコープの順番で先にヒットしたものを削除します。

参考: [Feature Request: 'set --erase' for multiple/all scopes · Issue #7711 · fish-shell/fish-shell](https://github.com/fish-shell/fish-shell/issues/7711)
:::

# 他のパターン

## status --is-login
これはドキュメントの Introduction の項目に記載されているパスの通し方のパターンですが、`config.fish` に次のコードを記載することでパスを通せます。

```shell:config.fish
if status --is-login
    set -gx PATH ~/linux/bin $PATH 
end
```

https://fishshell.com/docs/current/index.html#configuration-files

これは fish シェルがログインシェルの状態である際にパスが追加されるというものですが、`set -gx` 単独で使用した場合と同じく `config.fish` を `source` することでパスの重複追加が行われます。

また、子プロセスにネストさせたり、`exec $SHELL -l` でシェルの新規化すると重複したパス内のコンポーネントの位置が末尾になったり、移動したりする謎の挙動があります。これは macOS での問題であり、通常 fish は `PATH` 自体を修正したりしないのですが、サブシェルを起動すると `PATH` の構築が行われるらしく、パスが一貫した順序にならなくなります。

https://github.com/fish-shell/fish-shell/issues/5456

## export, setenv

推奨ではないですが、実は fish は他のシェルとの互換のために `export`(bash 用) と `setenv`(csh 用) という関数を提供しており、これらを使って環境変数を設定できるそうです。しかし、ビルトインの `set` コマンドの使用を推奨しているためか、ドキュメントに記載されていません。開発者の一人である、faho 氏が issue で言及しているのを発見しました。

>[faho commented on 3 Feb 2016](https://github.com/fish-shell/fish-shell/issues/2704#issuecomment-178749261)  
>There's an additional point:
>```shell
>> type set
>set is a builtin
>> type setenv
>setenv is a function with definition 
>[...]
>> type export
>export is a function with definition
>[...]
>```
>We ship functions called export and setenv for compatibility. These are simple shims that can be used to help you ease in to fish. If you wish to write idiomatic fish code, use set -x. (There's also a bit of overhead associated with these functions, but it should be minimal)

`type` で関数定義が見られるので興味有る方は確認してみてください。
↓ Github にあるソースコード。
https://github.com/fish-shell/fish-shell/blob/master/share/functions/export.fish
https://github.com/fish-shell/fish-shell/blob/master/share/functions/setenv.fish

これらの関数を使うことによるオーバーヘッドがいくらか有るみたいですね。関数定義を見ると両方とも内部的に `set -gx` を使用しているので互換用のラッパーということでしょう。

# 研究

以下の項目は余談になりますので、興味の無い方は[結論](#結論)まで飛ばしてもらってかまいません。

## スコープと環境変数の選択

「どのスコープで定義されているか」と「環境変数という状態であること」には自体は直接関係がありませんが、これまで見てきたように**どのような場合においてグローバル変数を使うか、ユニバーサル変数を使うか、または環境変数にすべきか**というのは非常に分かりづらいです。

ユニバーサル変数・グローバル変数・環境変数の違いを理解するための例として、Tide プラグインの仕組みが参考になります。

https://github.com/IlanCosman/tide

Tide は fish shell のプロンプトを簡単に設定できるプラグインです。次のコマンドで fisher を使用してインストールできます。

```shell
fisher install IlanCosman/tide@v5
```

Tide が利用している変数の多くはニバーサル変数で、プロンプトの外観についての設定が格納されています(`tide configure` によって設定変更際した際に生成される fake つまりプレビュー用の一時利用されるグローバル変数などもあります)。

```shell
# 名前に tide が含まれるユニバーサル変数の個数は 276 個
❯ set -U | grep tide | count
276
# 実際に見てみる
❯ set -U | grep tide
_fisher_IlanCosman_2F_tide_40_v5_files '/Users/roshi/.config/fish/functions/_tide_detect_os.fish'  …
_tide_prompt_10175 ''  \e\(B\e\[m\e\(B\e\[m\e\[34m\e\[44m\ \e\(B\e\[m\e\[44m\e…
_tide_prompt_13890 ''  \e\[m\co\e\[m\co\e\[34m\e\[44m\ \e\[m\co\e\[44m\e\[37m\…
_tide_prompt_1474 ''  \e\[m\co\e\[m\co\e\[34m\e\[44m\ \e\[m\co\e\[44m\e\[37m\…
_tide_prompt_1493 ''  \e\[m\co\e\[m\co\e\[34m\e\[44m\ \e\[m\co\e\[44m\e\[37m\…
_tide_prompt_1501 ''  \e\[m\co\e\[m\co\e\[34m\e\[44m\ \e\[m\co\e\[44m\e\[37m\…
_tide_prompt_1511 ''  \e\[m\co\e\[m\co\e\[34m\e\[44m\ \e\[m\co\e\[44m\e\[37m\…
_tide_prompt_1520 ''  \e\[m\co\e\[m\co\e\[34m\e\[44m\ \e\[m\co\e\[44m\e\[37m\…
# 長いので省略
tide_vi_mode_bg_color_default green
tide_vi_mode_bg_color_replace yellow
tide_vi_mode_bg_color_visual blue
tide_vi_mode_color_default black
tide_vi_mode_color_replace black
tide_vi_mode_color_visual black
tide_vi_mode_icon_default DEFAULT
tide_vi_mode_icon_replace REPLACE
tide_vi_mode_icon_visual VISUAL
tide_virtual_env_bg_color brblack
tide_virtual_env_color cyan
tide_virtual_env_icon 
```

ただ、これらは"環境変数"ではありません。`set -x` ですべての環境変数が見られますが、tide の文字はどこにもありません。

```shell
❯ set -x
# スコープに関係なくすべての環境変数が表示される
```

ユニバーサル変数はすべてのシェルセッションで共有され、変更されても永続化・伝播するため、この性質を使えば、`tide cofigure` でプロンプトの外観を変更すると他のシェルセッションでも一気にプロンプトの表示が変更されます。これはユニバーサル変数の恩恵です。例えば、tmux でいくつものシェルを開いていれば、その便利さは一目瞭然です。

https://github.com/IlanCosman/tide/issues/242

もし、tide の利用する設定用の変数がグローバル変数だったら、設定変更を直接したシェルセッションでしか設定が反映されず、そのセッションを終了した時点でグローバル変数の効能は消滅するため、毎回設定しなおす必要ができくるでしょう。

また、tide における設定用の変数は他の外部コマンドが参照される必要はとくにありません。環境変数はその外部コマンド特有の変数で内部的に利用し設定される必要のあるものか、またはよく利用されうる情報 `$PAGER` や `$EDITOR` などの場合を覗いて必要ない、というわけです。

その違いについて説明すると、例えば、`pstree` のところで説明しましたが、JavaScript のツールチェインを管理する volta というツールのコマンドとして `volta install` があり、これは node のバージョンを指定してインストールするコマンドです。バージョンを指定しなければ最新の LTS release をインストールします。

以下のように事前に使用する環境変数と `PATH` を `config.fish` に定義したとします。

```shell:config.fish
set -gx VOLTA_HOME $HOME/.volta
fish_add_path $VOLTA_HOME/bin
```

ここから実際に `volta install` を行う際に、環境変数としてエクスポートした `VOLTA_HOME` はコマンドの引数として渡していないことが分かります。

```shell
❯ volta install node
```

コマンド引数に渡さずとも、`volta` コマンドの実行では親プロセスのシェルから環境変数が継承され内部的に参照されているはずです。そして `VOLTA_HOME` に登録されているディレクトリにインストールを実行します。

もしそうでなければ、コマンド引数に渡さなくてはなりません。環境変数は普通のシェル変数と異なり、引数として渡さずともプロセスのインスタンスを生成する際に継承されるわけですから。

```shell
set -g testvalue 1234
# コマンド引数にシェル変数を渡す
❯ echo $testvalue
1234
# 環境変数はコマンド引数に渡さずとも子プロセス(外部コマンド)に継承される
❯ volta install node
```

これが、普通のシェル変数と環境変数の状態になったシェル変数との違いです。環境変数として参照される際には継承によってその変数の複製が渡されます。

>Variables are a way to save data and pass it around. They can be used just by the shell, or they can be "exported", **so that a copy of the variable is available to any external command the shell starts**. An exported variable is referred to as an "environment variable".
>
>変数は、データを保存し周囲へと渡す方法です。変数は単にシェルから利用されることもできますし、変数はエクスポートされることによって、その変数の複製をシェルが開始するあらゆる外部コマンドから参照できるようにすることも可能です。エクスポートされた変数は環境変数として参照されます。
[fish documents: Shell variables](https://fishshell.com/docs/current/language.html#shell-variables) より引用

:::details 参考: Bash における環境変数
>When a program is invoked it is given an array of strings called the environment. This is a list of name-value pairs, of the form name=value.
>
>Bash provides several ways to manipulate the environment. On invocation, the shell scans its own environment and creates a parameter for each name found, automatically marking it for export to child processes. Executed commands inherit the environment. The export and ‘declare -x’ commands allow parameters and functions to be added to and deleted from the environment. If the value of a parameter in the environment is modified, the new value becomes part of the environment, replacing the old. The environment inherited by any executed command consists of the shell’s initial environment, whose values may be modified in the shell, less any pairs removed by the unset and ‘export -n’ commands, plus any additions via the export and ‘declare -x’ commands.  
>
>プログラムが呼び出された際に "環境"(environment) と呼ばれる文字列の配列がプログラムに与えられます。これは `name=value` 形式の name-value のペアのリストです。
>
>Bash はこの環境を操作するための方法をいくつか提供しています。シェルが呼び出される際には、シェルは自身の環境をスキャンし、見つかった name ごとにパラメータを作成し、自動的にそれらを子プロセスにエクスポートできるようにマークを付けます。`export` と `declare -x` のコマンドによって、環境からパラメータと関数が追加や削除されることが許可されます。パラメータの値が環境において修正された場合、新しい値は古い値を置き換えて環境の一部となります。あらゆる実行されたコマンド(executed command)によって継承された環境は、シェルの初期環境から構成されます。その環境の値はシェル内で修正可能ですが、`unset` と `export -n` コマンドによって削除されたペアと `export` と `declare -x` コマンドによる追加は含まれません。  
>[Bash refarence](https://www.gnu.org/savannah-checkouts/gnu/bash/manual/bash.html#Environment) より引用
:::

## ユニバーサル変数の廃止?
scope shadowing やら、今まで述べたようなアンチパターンにおけるグローバル変数・ユニバーサル変数・環境変数との混同などがあるため、いっそのことユニバーサル変数を廃止し別のメカニズムで置き換えようという議論もあります。  

https://github.com/fish-shell/fish-shell/issues/7317  
https://github.com/fish-shell/fish-shell/issues/7379  

開発者の一人である ridiculousfish 氏がユニバーサル変数のスコープを除去し、ユニバーサル変数をグローバル変数にするというプルリクエストを投下していました(そもそもフィードバックを得るためで、結局は棄却というか、自分で取り下げていましたが)。  
https://github.com/fish-shell/fish-shell/pull/8455  


以下、ユニバーサル変数の問題点と環境変数についての会話を抜粋。

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

ユニバーサル変数の利点は分かりましたが、異なるマシン間で同様の設定にしたいユーザーが多いので、`config.fish` に記載するというパターンが求められている一方で、開発者側は `config.fish` ではなくインタラクティブな使用で 1 回実行するというのが好きな人の割合が多いのかなという印象をなんとなく受けました。

他の issue も含めて全体的に見ると、結局のところユニバーサル変数が取り除かれることは無さそうです。

:::message
ユニバーサルスコープの環境変数  

ユニバーサル変数のエクスポートにもそれなりに利点があるそうです、例えば次の引用を見てください。

>シェルの子プロセスはそのシェルの環境を継承しますが、シェルは独立した実行コンテキストであり、環境情報を互いに共有するわけではありません。1つの「ターミナル」ウインドウで設定した変数は、ほかの「ターミナル」ウインドウでは設定されません。
>
>「ターミナル」ウインドウを閉じると、そのウインドウで設定した変数は使用できなくなります。変数の値をセッション間やすべての「ターミナル」ウインドウで保持したい場合は、シェル起動スクリプトで設定する必要があります。zshシェル起動スクリプトを変更して複数のセッション間で変数などの設定を保持する方法については、zshのマニュアルページ（manで表示）の「Invocation（起動）」セクションを参照してください。
>[Macの「ターミナル」で環境変数を使用する - Apple サポート (日本)](https://support.apple.com/ja-jp/guide/terminal/apd382cc5fa-4f58-4449-b20a-41c53c006f8f/mac) より引用

環境変数は親プロセスから継承されているため、その値はシェルセッション間でリアルタイムに共有されているわけではありません。これをある意味克服した仕組みが fish のユニバーサル変数です。独立したコンテキストに閉じ込められていた環境変数をユニバーサルスコープで宣言することで、すべてのシェルにおいてリアルタイムに環境変数を同一の値にでき、一斉に変更できます。zsh など他のシェルではユニバーサル変数の仕組みはないため、fish ではこのユニバーサル変数が killer feature であると開発者達は考えています。
:::

# 結論

以下、２つのパターンではやることがが少しずつ異なるので違いを意識する必要があります。

- `config.fish` でパスを通す場合
- コマンドラインからパスを通す場合

`config.fish` でパスを通す場合には `fish_add_path` 関数を使用し、`PATH` 以外の環境変数を定義するには `set -gx` を使用して定義します。

```shell:config.fish
# 通常の環境変数については set -gx を使用する (set -Ux で永続化せずとも fish 起動時に読み込まるので実質的には永続化される)
set -gx VOLTA_HOME $HOME/.volta
# PATH 環境変数については fish_add_path 関数を使用する
fish_add_path $VOLTA_HOME/bin
```

コマンドラインからパスを通す場合には、一度だけ `fish_add_path` を実行し、 `PATH` 以外の環境変数を定義する場合、`set -Ux` を使ってユニバーサルスコープに定義します。コマンドラインから `set -gx` をやってしまうと永続化できません。

```shell:コマンドライン
❯ set -Ux VOLTA_HOME $HOME/.volta
❯ fish_add_path $VOLTA_HOME/bin
```

コマンドラインからパスを追加して、`PATH` 以外の環境変数を `config.fish` で定義したり、その逆ももちろん可能です。

:::message
環境変数を削除するには、`config.fish` の場合はコードを削除したりコメントアウトしてください。`PATH` だけはユニバーサル変数によって永続化されているため、`config.fish` から `fish_add_path` 関数の部分を削除などした後にコマンドラインからユニバーサル変数 `fish_user_paths` に登録されている要素を明示的に削除します。  

削除の方法もいくつかありますが、簡単なのがインデックスを確認して削除する方法です。

```shell:コマンドライン(インデックスを確認してその要素を削除)
❯ set -S fish_user_paths
$fish_user_paths: set in universal scope, unexported, with 4 elements
$fish_user_paths[1]: |/Users/roshi/.deno/bin|
$fish_user_paths[2]: |/Users/roshi/.volta/bin|
$fish_user_paths[3]: |/Users/roshi/.cargo/bin|
$fish_user_paths[4]: |/opt/homebrew/bin|
❯ set -e fish_user_paths[1]
```

チュートリアルなどに記載されている `string match` などを利用したパターンマッチによる再定義などでも要素を削除できます。

```shell:コマンドライン(string match を使用して対象のパスのみを除去する)
❯ set fish_user_paths (string match -v $VOLTA_HOME/bin $fish_user_paths)
```

参考: [fish documents: path](https://fishshell.com/docs/current/tutorial.html#path)

環境変数をコマンドラインから `set -Ux` を使って定義した場合には、そちらも明示的に削除します。

```shell
❯ set -e -U VOLTA_HOME
```
:::

# v3.4.0

~~余談ですが、そろそろ fish v3.4.0 が出そうです。楽しみですね。~~
2022/03/13 に `v3.4.0` がリリースされましたね。

https://github.com/fish-shell/fish-shell/releases/tag/3.4.0

`v3.4.0` がリリースされたため、修正された点・追加された点について追記しました。

