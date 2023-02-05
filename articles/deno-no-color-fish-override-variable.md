---
title: "fishのVAR=VALステートメントでDenoの環境変数 NO_COLOR を上書きする"
published: true
cssclass: zenn
emoji: "👽"
type: "tech"
topics: [fish, deno, 環境変数]
date: 2022-03-16
modified: 2023-02-05
url: "https://zenn.dev/estra/articles/deno-no-color-fish-override-variable"
AutoNoteMover: disable
tags: " #shell/fish/syntax  #deno/environment   "
aliases: 記事_fishのVAR=VALステートメントでDenoの環境変数 NO_COLOR を上書きする
---

# はじめに

以前、Deno の出力ログから ANSI escape character を取り除くためのラッパー関数作成について次の記事を書きましたが、解決策が見つかったので新しく記事を書こうと思います。

https://zenn.dev/estra/articles/zenn-fish-shell-argparse-unknown-options

# VAR=VAL ステートメント

`VAR=VAL` のシンタックスは他のシェルで使われているらしいですが、fish では v3.1 から利用できるようになっています。

使い方としては、**コマンドの前で宣言して一時的に変数を上書きできる** というものになります。

https://fishshell.com/docs/current/language.html#overriding-variables-for-a-single-command

例えば、`foo` という変数が `banana` という値でセットしてあったときに、コマンド `echo` の前で `foo=gagaga` と宣言すると引数の `$foo` の値が上書きされます。

```shell
❯ set foo banana
❯ foo=gagaga echo $foo
gagaga
```

この `VAR=VAL` ステートメントで上書きした変数は環境変数としてエクスポートされた状態になります。

```shell
# ローカルスコープでエクスポートされていることがわかる
❯ hoge=fugafuga set -S hoge
$hoge: set in local scope, exported, with 1 elements
$hoge[1]: |fugafuga|
❯ set -S hoge
# 何も表示されない
```

変数を `set -S` で確認してみると分かりますが、`VAR=VAL` ステートメントによって上書きエクスポートされる際にはローカルスコープとなります。

```shell
# グローバルスコープでセット
❯ set -g hoge bar
# 変数をみてみる
❯ set -S hoge
$hoge: set in global scope, unexported, with 1 elements
$hoge[1]: |bar|
# VAR=VAL で上書きした状態でみてみる
❯ hoge=fugafuga set -S hoge
$hoge: set in local scope, exported, with 1 elements
$hoge[1]: |fugafuga|
$hoge: set in global scope, unexported, with 1 elements
$hoge[1]: |bar|
# 上書きは一時的なものなので元に戻っている
❯ set -S hoge
$hoge: set in global scope, unexported, with 1 elements
$hoge[1]: |bar|
```

これによって、**外部コマンドに継承させる環境変数を一時的に上書きできます**。例えば、`VAR=VAL` のステートメントと [ブレース展開](https://fishshell.com/docs/current/language.html#brace-expansion) を組み合わせることによって、子プロセスで bash を立ち上げる際に継承させる `PATH` を `/usr/sbin:/sbin:/usr/bin:/bin` にできます。

```shell
❯ PATH={/usr,}/{s,}bin bash

The default interactive shell is now zsh.
To update your account to use zsh, please run `chsh -s /bin/zsh`.
For more details, please visit https://support.apple.com/kb/HT208050.
bash-3.2$ printenv PATH
/usr/sbin:/sbin:/usr/bin:/bin
bash-3.2$
```

このように環境変数を一時的に上書きすることで、環境を一々汚さずに外部コマンドの実行が可能となります。

# Deno の環境変数 NO_COLOR

それでは Deno の話になりますが、Deno はいくつかの環境変数を用意しており、その中に `NO_COLOR` という環境変数があります。

https://deno.land/manual@v1.19.3/getting_started/setup_your_environment#environment-variables

この `NO_COLOR` 環境変数はセットされていると Deno CLI に対して標準出力と標準エラー出力を書き出す際に ANSI color code を送らないようにする伝えることができます。このフラグの値は、API `Deno.noColor` の値をチェックすることによって、`--allow-env` のパーミッションを指定せずに確認できます。

例えば次のような TypeScript ファイルを `deno run` で実行しています。

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

// NO_COLOR 環境変数が存在しているかどうかの boolean 
console.log(Deno.noColor);
```

```shell
❯ deno run tests/console_test.ts
```

実行すると、次のようにターミナルにカラー出力されます。

![](/images/deno-no-color-fish-override-variable/deno-run-ansi-color.jpg)

以前書いた次の記事では、ラッパー関数を作成して内部で `gesd 's/\x1b\[[0-9;]*m//g'` を噛ませることで ANSI escape code を取り除いていました。

https://zenn.dev/estra/articles/zenn-fish-shell-argparse-unknown-options

このようなラッパー関数の作成は少し大げさでしたので、今回は環境変数の一時的な上書きで ANSI color code を stdout(標準出力) に送信しないように Deno CLI に伝えてあげることでリダイレクトした際にちゃんと表示できるようにしてあげます。

```shell
❯ NO_COLOR= deno run tests/console_test.ts
```

出力が色なしになりました。

![](/images/deno-no-color-fish-override-variable/deno-run-no-ansi-color.jpg)

`NO_COLOR` 環境変数は存在さえしていれば値自体はなんでもよいのですが (`NO_COLOR=true` などでも OK)、`VAR=VAL` のステートメントは値が空でも機能するので上記のコードで色無し出力が可能です。

実際に値なしの状態を調べてみても上書きでエクスポートできていますね。

```shell
# 値無しでも上書きできる
❯ NO_COLOR= set -S NO_COLOR
$NO_COLOR: set in local scope, exported, with 1 elements
$NO_COLOR[1]: ||
```

細かい挙動のカスタマイズはできませんが、ラッパー関数を作成するよりよっぽど楽なのでこちらの方がよいですね。

ちなみに、最近の CLI ツールで出力されるようになっていることが多い ANSI color code については no-color.org というサイトでデファクトスタンダードな情報を知れるらしいです。

https://no-color.org/

Deno CLI のように `NO_COLOR` 環境変数を利用して色無しで出力できるライブラリやツールなどが確認できます。

![](/images/deno-no-color-fish-override-variable/no_color_org.jpg)
