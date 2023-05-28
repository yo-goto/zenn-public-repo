---
title: "fish shellからgoogle検索するコマンドを作ってみた"
published: true
cssclass: zenn
emoji: "🐟"
type: "tech"
topics: [fish, shell, macOS, 初心者]
date: 2022-01-16
modified: 2023-05-28
url: "https://zenn.dev/estra/articles/google-search-from-fish-shell"
tags: [" #shell/fish #google "]
aliases:
  - 記事_fish shellからgoogle検索する関数の作成
  - ggl.fish
---

:::details Changelog
- 2023-05-28
    - フォーマット
:::

## モチベーション

シェルを触っている最中に検索したい事柄がよくでてくることがあります。コマンドラインからそのまま流れるように検索したい&&シェルスクリプトを勉強してみたい、ということで fish shell で CLI から google 検索できるようにする関数 (コマンド) を作ってみました。

環境は macOS/iterm2 で、fish 言語 (fish language) を使って作りました。

fish でのこういったスクリプトについての情報はあまり多くなかったので bash での実装を参考にしてつくりました。

https://s10i.me/whitenote/post/40

ただ検索できるようにするだけだとそのままになってしまうので、google 検索で使える**英語検索**と**画像検索**、**完全一致検索**、**個人最適化検索無効化**などのコマンドオプションを追加しました。この記事を読めば**fish 関数でのオプション処理の基本**が関数の実装を通して分かるように解説しています。

## スクリプト

実際のスクリプト `ggl.fish` は以下の gist で公開しています。

:::details 実際のスクリプト ggl.fish
@[gist](https://gist.github.com/yo-goto/7acfa712006488466d73ff42b9d952cc)
:::

:::message
gist のスニペットからオープンなプロジェクトとして、リポジトリを開きました。プラグインマネージャーの fisher を使ってインストールできます。gist から進化させたので是非使ってみてください。

https://github.com/yo-goto/ggl.fish

![ggl-logo](https://user-images.githubusercontent.com/50942816/152551047-464d7181-7c50-4a72-abc4-aea7e1aaff9d.png)

```shell
$ fisher install yo-goto/ggl.fish
```
:::

## 参考資料

先に使用する fish の文法と機能とコマンドについての参考資料をあげておきます。

fish の文法と機能
- [コマンド置換](https://fishshell.com/docs/current/language.html?highlight=pipe#command-substitution)
- [引数のハンドリング](https://fishshell.com/docs/current/language.html#argument-handling) 
- [セミコロンによるコマンドの分割](https://fishshell.com/docs/current/tutorial.html#separating-commands-semicolon)
- [複数行の編集](https://fishshell.com/docs/current/interactive.html#multiline-editing)

fish コマンド
- [argparseコマンド](https://fishshell.com/docs/current/cmds/argparse.html#flag-value-validation)  
    - argparse の使い方: [fishでのオプション解析の決定版！argparseコマンドの紹介 - Qiita](https://qiita.com/ryotako/items/e36114ec1f86dbdd274a#%E5%BC%95%E6%95%B0%E3%82%92%E5%8F%96%E3%82%8B%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%92%E4%BD%BF%E3%81%86)
- [setコマンド](https://fishshell.com/docs/current/cmds/set.html#cmd-set)  
- [string escapeコマンド](https://fishshell.com/docs/current/cmds/string-escape.html?highlight=encoding)
- [string joinコマンド](https://fishshell.com/docs/current/cmds/string-join.html)
- [andコマンド](https://fishshell.com/docs/current/cmds/and.html#cmd-and)
- [testコマンド](https://fishshell.com/docs/current/cmds/test.html)

bash でのパラメータ展開の代わりに fish での文字列操作  
- [fish shellで文字列操作](https://blog.nijohando.jp/post/fish-shell-manipulating-string/#%E7%B5%90%E5%90%88)
- [string manipulation](https://fishshell.com/docs/current/fish_for_bash_users.html?highlight=expansion#string-manipulation)

google 検索パラメータ  
- [Google検索のパラメータ（URLパラメータ）一覧 - fragment.database.](http://www13.plala.or.jp/bigdata/google.html)

## 解説

fish では、[Autoloading Function](https://fishshell.com/docs/current/language.html#autoloading-functions) という機能があり、コマンドにエンカウントすると、`~/.config/fish/functions/` ディレクトリ内のコマンド名と同一名のファイルを探して、自動ロードします。この機能を使って、`ggl` という名前で「引数をブラウザで google 検索する」コマンドを作成します。

```shell
cd ~/.config/fish/functions/
touch ggl.fish
# ファイル作成したら、vscode等で開きます
code ggl.fish 
```

fish 言語での関数作成は、`function 関数名; 処理内容; end` で作成できます。この場合ファイル名と同じ `ggl` という名前の関数を作成します。ちなみに、fish ではセミコロン `;` を使うことで、複数のコマンドを一行で書くことできます (これは他のシェルも同じだそうです)。

例えば、`function` コマンドで関数を定義する際にも簡単な内容であれば `function ll; ls -l $argv; end` というようにワンラインで書くことができます。

fish ドキュメントの各コマンドのページの Synopsis の項目には、そのコマンドの使い方が記載されていますが、このセミコロンを使ってワンラインで記載されている場合があります。例えば、fish の `function` コマンドの Synopsis は次の通りです。

```fish
function NAME [OPTIONS]; BODY; end
```

セミコロンを使わずに複数行で書く場合には、次のように書きます。

```fish
function ll
    ls -l $argv
end
```

function コマンドの詳細な使い方については公式ドキュメントの次のページに記載されています。
[function - create a function — fish-shell 3.3.1 documentation](https://fishshell.com/docs/current/cmds/function.html)

さて、それでは実際に `ggl` コマンドを作成します。

```fish
function ggl
    # 処理内容
end
```

fish の関数は単一コマンドにように呼ぶことができるコマンドのブロックのことであり、関数を使うことで複数の単純なコマンドの集合をより高度な 1 つのコマンドとしてまとめておくことができます。上記のコードの処理内容の部分に一連のコマンドを記載することでひとつのコマンドとしてまとめます。

Fish がインストールされていれば、macOS に元々入っている BSD 系のコマンド以外にビルトインと呼ばれるコマンドが使えるようになります。`builtin -n` ですべてのビルトインコマンドを確認できます。`function` コマンドのそのビルトインの 1 つです。これらのコマンドを集合させて fish 関数を作成します。

fish では、関数を使った自作コマンドの引数は `$argv` という変数へ格納されます。引数は複数個入力でき、その際に `$argv` にすべて格納されますが、この `$argv` はリスト変数 (list variable) であり、関数へ渡されたすべての引数を含みます。要素へのアクセスには `$argv[1]` などインデックスを指定することでできます (ただし、インデックスは 1 から始まります)。

```shell
$ function myfunc
    echo $argv[1]
    echo $argv[3]
end
$ myfunc 一個目 二個目 三個目
一個目
三個目
```

自作した関数では、[argparseコマンド](https://fishshell.com/docs/current/cmds/argparse.html#cmd-argparse) を使うことで、コマンドに渡されれるオプションの解析を行うことができます。これで自分の関数に好きなオプションを設定できます。

```fish
function mybetterfunc
    argparse 'h/help' 's/second' -- $argv
    or return # 認識できないオプションによってargparseが失敗したらexitする
    
    # -h または --help オプションが与えられた場合、以下のような文書を出力してreturnする
    if set -q _flag_help
        echo "mybetterfunction [-h|--help] [-s|--second] [ARGUMENTS...]"
        return 0
    end

    # -s または --second オプションが与えられた場合、二番目の引数を出力する
    if set -q _flag_second
        echo $argv[2]
    else
        echo $argv[1]
        echo $argv[3]
    end
end
```

オプション引数が `ggl -h` のように渡された場合には、ローカル変数として `_flag_help` という変数が生成されます。これで関数内でこの変数を使ってオプションの処理を行うことができます。オプション処理は、`if` や `set -q` などを使って処理します (この方法は fish のドキュメントに記載されていますが、他に、もっとよい方法があれば教えて下さい)。

`argparse` の使い方は、例えば、ヘルプオプションのみを設定したいとして、`argparse 'h/help' -- $argv` とします。`h/help` は `-h` と `--help` という short バージョンと long バージョンの 2 つのオプションを設定することを意味します。そのあとの `-- $argv` はお約束として書きます。これによって関数内で `$argv` を使うときに渡されたオプションの文字列が取り除かれた状態のリストとして使うことができます。さらに、`or return` は引数のパースに失敗した場合のために `argprse` のつぎの行につけておきます。これが基本的な使い方です。

```shell
# 上記の mybetterfunc を使ってみる
$ mybetterfunc -s 二個目の引数 三個目の引数 四個目の引数
二個目の引数
四個目の引数
# 関数内での$argv は -s という文字列が含まれないリストになる
# $argv[1] と $argv[3] が出力されると、実際の引数である文字列 二個目の引数 と 四個目の引数 が出力される
```

それでは、`ggl` コマンドで設定するオプションを考えます。まず、基本的なオプションオプションとして、次の 3 つを設定します。

- 英語での検索オプション : `e/english`
- 画像での検索オプション : `i/image`
- ヘルプオプション : `h/help`

さらに、どのブラウザで検索するかのブラウザオプションとして 4 つを設定します。

- Vivaldi : `v/vivaldi`
- Chrome : `c/chrome`
- Firefox : `f/firfox`
- Safari : `s/safari`

とりあえず、この 7 つのオプションを設定していきます。関数の冒頭で `argparse` コマンドを使って、残りの部分でオプション引数を取り除いた状態で `$argv` を使えるようにします。

```fish
function ggl 
    argparse \
        'e/english' 'i/image' \
        'h/help' \
        'v/vivaldi' 'c/chrome' 'f/firefox' 's/safari' -- $argv
    or return
    
    # オプション処理
    # 基本処理
end
```

fish では、[一行のコマンドを複数行にできるように改行する手段](https://fishshell.com/docs/current/interactive.html#multiline-editing) があり、`\` で改行できます。これを使ってオプションをわかりやすくグループ化しておきます。

それでは `ggl` 関数の基本的な機能である検索機能を実装していきます。検索機能といっても、引数に渡したキーワードをブラウザ上で検索するといった簡単な処理です。googler のようにターミナル内で表示してブラウジングするような高度なものではないです。

https://github.com/jarun/googler

macOS なので `open -a Safari https://www.google.com/search?q=keywrods` のようにコマンドを使ってアプリケーションを指定して URL を開くようにします。まずは、引数に渡したキーワードを検索できるように URL を作成します。とりあえず Safari で開けるようにします。

```fish
function ggl
    argparse # 省略
    or return
    
    oepn -a Safari "https://www.google.com/search?q=$argv"
end
```

この実装では、次のように英単語を引数にすれば URL を開くことができますが、日本語だとうまく URL を開くことができません。

```shell
$ ggl keyword
```

そこで URL として使えるように％エンコーディングします。％エンコーディングには fish の [string escape](https://fishshell.com/docs/current/cmds/string-escape.html) コマンドを使用します。`--style=url` オプションで URL エンコーディングが可能です。

```shell
$ string escape --style=url あいうえお
%E3%81%82%E3%81%84%E3%81%86%E3%81%88%E3%81%8A
# あいうえお という文字列をURL用の%エンコーディングしたもの
```

このエンコーディングした文字列を `https://www.google.com/search?q=` のあとに結合して日本語でも検索できるようにします。また、複数の引数を渡せるように各引数をホワイトスペース (空文字) で連結します。最終的には連結した状態の文字列を string escape でエンコーディングします。

```fish
set -l baseURL "https://www.google.com/search?q="
set -l keyword (string join " " $argv)
set -l encoding (string escape --style=url $keyword)
set -l searchURL (string join "&" (string join "" $baseURL $encoding))
```

URL 生成の処理はだいたい上のようなコードになります。それでは、ひとつずつ説明していきます。

fish では、ローカル変数、グローバル変数、ユニバーサル変数というように [シェル変数に種類が3つ](https://fishshell.com/docs/current/language.html#shell-variables) ありますが、今回は関数内のみで使うの変数を設定するのでローカル変数を変数設定用の [set](https://fishshell.com/docs/current/cmds/set.html) コマンドにローカルオプション `-l` を付けて変数の設定を行います。

```shell
# set -l 変数名 値
set -l foo hoge # ローカル変数 foo に値 hoge を設定
set foo huga # 再代入
echo $foo # huga という文字列が出力される
```

fish では、変数の値を参照したい場合には `$foo` のように変数名の頭に `$` をつけます。

また、参考記事の「[ターミナルからでもGoogle検索がしたい！](https://s10i.me/whitenote/post/40)」にある bash スクリプトでは、`${prev_dir%/*}` のようなブレースによるパラメーター展開を行っていますが、fish ではそのような bash のブレース展開はサポートしていないので、`string` コマンド系の文字列操作によって代用します (公式ドキュメントでも明示されています)。

さらに、fish の [コマンド置換](https://fishshell.com/docs/current/language.html#command-substitution)(command substitution) という、`()` で囲むことによってコマンドの一部に他のコマンドを埋め込むことができる機能を使うと、文字列の操作によって行をいくつか圧縮して書くことができます。以下のように関数の引数である `$argv` を空文字で連結した文字列をローカル変数として設定する際にコマンド置換を使用しています。

```fish
set -l keyword (string join " " $argv)
# ()内のコマンドで返ってくる文字列をローカル変数 keyword にセットします
```

[string join](https://fishshell.com/docs/current/cmds/string-join.html) コマンドは、第一引数によって第二引数以降の引数の文字列を連結できます。ホワイトスペースを指定するなら `" "` をセパレーターとして第一引数に、そのまま連結したい場合には `""` をセパレーターとして第一引数に渡します。

今回のケースでは、例えば、日本語で「fish shell 使い方」というように検索したいので、`ggl fish shell 使い方` というように `ggl` コマンドに文字列が 3 つ渡されるような使い方になるはずです。この使い方だと、コマンドの引数は `$argv` にリストの形でセットされます。

`$argv` はリストなので `string join " " $argv` としたときには、リストの値を 1 つずつとりだして `string join` コマンドの第一引数に渡した `" "` というホワイトスペースで連結されます。つまり、`ggl fish shell 使い方` なら `fish shell 使い方` という文字列が生成されます。例えば、`string join` の第一引数を `"+"` にしたら、`fish+shell+使い方` という文字列が生成されます。

さて、これで先程のコードの意味がわかりやすくなりました。

```fish
set -l baseURL "https://www.google.com/search?q="
set -l keyword (string join " " $argv)
# コマンドの引数から渡される複数の文字列を "fish shell 使い方" のような一つの文字列に連結

set -l encoding (string escape --style=url $keyword)
# 連結した文字列をURLエンコーディングしてURLとして開けるようにする
# "fish shell 使い方" なら "fish%20shell%20%E4%BD%BF%E3%81%84%E6%96%B9" のようになる

set -l searchURL (string join "&" (string join "" $baseURL $encoding))
# baseURLとして設定した google検索用の文字列と、エンコーディングした文字列を結合してクエリを作成
# "https://www.google.com/search?q=fish%20shell%20%E4%BD%BF%E3%81%84%E6%96%B9" が生成される
```

これで一通り `ggl` を書いてみると

```fish
function ggl 
    argparse #省略
    or return

    set -l keyword (string join " " $argv)
    set -l encoding (string escape --style=url $keyword)
    set -l baseURL "https://www.google.com/search?q="

    if test -n "$encoding"
        set -l searchURL (string join "" $baseURL $encoding)
        open -a Safari "$searchURL"
        return
    else 
        echo "検索したい言葉を引数として実行してください"
    end
end
```

fish では、`if 条件; 処理内容; end` で if 節を書くことができるので、検索する単語の引数があった場合にのみブラウザを開くようにします。[test](https://fishshell.com/docs/current/cmds/test.html) コマンドで `encoding` 変数に値があるかどうか調べて、あった場合にのみブラウザを開かせます。`test -n` の `-n` オプションで引数の文字列の長さが 0 でない場合にのみ true を返します。`"$encoding"` で変数の値を文字列にしてテストします。

これで引数が無い場合には次のようなコメントが出力されるようになります。

```shell
$ ggl 
検索したい言葉を引数として実行してください
```

こんな感じで「ブラウザで検索する」という基本的な機能ができたのであとは `argparse` で指定したオプションの処理を加えていきます。また、生成した URL を確認できるように新しく、`t/test` オプションも追加しておきます。

```fish
function ggl
    argparse \
        't/test' 'h/help' \
        'e/english' 'i/image' \
        'v/vivaldi' 'c/chrome' 's/safari' 'f/firefox' -- $argv
    or return 
    
    # 変数とURL生成の準備
    set -l keyword (string join " " $argv)
    set -l encoding (string escape --style=url $keyword)
    set -l baseURL "https://www.google.com/search?q="
    # 色の設定
    set -l pcolor bryellow

    if set -q _flag_help
        echo "ヘルプの内容(オプション等)"
        return
    end

    if test -n "$encoding"
        # google検索のオプションが指定されているか調べて、あったらクエリパラメータを設定
        set -q _flag_english; and set _flag_english "lr=lang_en"
        set -q _flag_image; and set _flag_image "tbm=isch"

        # 検索用のURLを生成
        set -l searchURL (string join "&" (string join "" $baseURL $encoding) $_flag_english $_flag_image) 

        # テストオプション(生成されるURLを確認する)
        if set -q _flag_test
            echo (set_color $pcolor)"Keyword     :"(set_color normal) "$keyword"
            echo (set_color $pcolor)"URL encoding:"(set_color normal) "$encoding"
            echo (set_color $pcolor)"Search URL  :"(set_color normal) "$searchURL"
            return
        end

        # ブラウザオプションが指定されているか調べる
        if set -q _flag_vivaldi
            open -a Vivaldi "$searchURL"
        else if set -q _flag_chrome
            open -a "Google Chrome" "$searchURL"
        else if set -q _flag_safari
            open -a Safari "$searchURL"
        else if set -q _flag_firefox
            open -a Firefox "$searchURL"
        else
            #### デフォルトブラウザで開く
            open "$searchURL"
        end
            echo "\"$argv\"" "について検索完了しました"
        return
    else
        echo "検索したい言葉を引数として実行してください"
        return
    end
end
```

fish での条件処理として if コマンド以外に `and` コマンドなどが便利に使えます。これらを使って、オプション引数が渡されているかどうかを調べます。`set -q 変数名` で変数が存在するかテストできるので、これと組み合わせて `set -q 変数名; and 処理内容` で変数が存在したら and 以降の処理を実行できます。

google 検索では、英語で検索したい場合や画像検索したい場合があるので、これらをコマンドのオプションとして `e/english` と `i/image` として argparse コマンドに設定しました。これらのオプションが渡されていればローカル変数 `_flag_english` と `_flag_image` が設定されているはずなので、それらをテストし、存在すれば google 検索で使えるクエリパラメータを `_flag_english` と `_flag_image` に設定しなおします。

```fish
# google検索のオプションが指定されているか調べて、あったらクエリパラメータを設定
set -q _flag_english; and set _flag_english "lr=lang_en"
set -q _flag_image; and set _flag_image "tbm=isch"
```

google の検索パラメータについては次のサイトの記事を参考にしました。

[Google検索のパラメータ（URLパラメータ）一覧 - fragment.database.](http://www13.plala.or.jp/bigdata/google.html)

これで、`ggl` コマンドのオプションとして `-e` と `-i` が渡されていれば、検索用の URL にそのパラメータを文字列として結合させます。例えば `ggl -e fish shell 使い方` なら英語検索のオプションが渡されているので、クエリに `lr=lang_en` を `&` で `strin join` コマンドを再び使って結合します。

```fish
# 検索用のURLを生成
set -l searchURL (string join "&" (string join "" $baseURL $encoding) $_flag_english $_flag_image) 
```

これで、`ggl -e fish shell` というようにオプション指定して実行した際に、実際に生成される URL は `https://www.google.com/search?q=fish%20shell&lr=lang_en` となります。
上のコードでは、`baseURL` と検索 Word を URL エンコーディングした `encoding` を結合してから、オプションのクエリパラメータを `&` で結合しています。これで URL は生成できるはずなので、`t/test` オプションで実際に生成される URL を出力して確認できるようにします。

```fish
# テストオプション(生成されるURLを確認する)
if set -q _flag_test
    echo (set_color $pcolor)"Keyword     :"(set_color normal) "$keyword"
    echo (set_color $pcolor)"URL encoding:"(set_color normal) "$encoding"
    echo (set_color $pcolor)"Search URL  :"(set_color normal) "$searchURL"
    return
end	
```

`-t` オプションが `ggl` コマンドに渡されていれば、if の後で true で内部処理が実行でるはずです。また、テストしたい場合はブラウザを開かないように if ブロック終了時に return してコマンドを終了するようにしています。`set_color` コマンドを使うと出力される文字列の色を変更できるのでこれをコマンド置換で埋め込んで echo でそれぞれの変数の値を出力してテスト内容を見やすくします。また、`set_color normal` を使って出力の色を元に戻します。

```shell
$ ggl -t fish shell 使い方
Keyword     : fish shell 使い方
URL encoding: fish%20shell%20%E4%BD%BF%E3%81%84%E6%96%B9
Search URL  : https://www.google.com/search?q=fish%20shell%20%E4%BD%BF%E3%81%84%E6%96%B9
$ ggl -tei how to use fish shell
Keyword     : how to use fish shell
URL encoding: how%20to%20use%20fish%20shell
Search URL  : https://www.google.com/search?q=how%20to%20use%20fish%20shell&lr=lang_en&tbm=isch
```

これで、基本的なオプション処理は完了しました。いくつかのオプションを排他的にして、例えば、ヘルプオプション (`-h`) とテストオプション (`-t`) が両方一緒に使えないようにします。これに `argparse` コマンドの `-x` オプションで実現できます。

具体的には、ブラウザオプションについてそれぞれ排他的になるように `-x 'v,c,s,f'` というように設定します。テストとヘルプのユーティリティオプションが排他的になるように `-x 't,h'` というように設定します。

```fish
function ggl
    argparse \
        -x 'v,c,s,f' \
        -x 't,h' \
        't/test' 'h/help' \
        'e/english' 'i/image' 'p/perfect' \
        'v/vivaldi' 'c/chrome' 's/safari' 'f/firefox' -- $argv
    or return
```

これで、`-t` オプションと `-h` オプションを同時に使った際に次のようにエラーが出力されるようになりました。

```shell
$ ggl -th
ggl: Mutually exclusive flags 'h/help' and `t/test` seen
```

あとは、追加で完全一致検索のオプション (`p/perfect`) と個人最適化検索無効のオプション (`n/nonperson`) もつけておきます。google での完全一致 (exact match) は引用符 (`""`) で単語を囲んで検索することで実現します。Personalized サーチを無効にするにはパラメータ `pws=0` をクエリに連結します。

```fish
set -q _flag_english; and set _flag_english "lr=lang_en"
set -q _flag_image; and set _flag_image "tbm=isch"
set -q _flag_nonperson; ande set _flag_nonperson "pws=0"
# 完全一致のオプションがあれば、｢"｣ をエンコーディングした文字列｢%22｣で文字列を囲む
set -q _flag_perfect; and set _flag_perfect "%22"
and set encoding (string join "" $_flag_perfect $encoding $_flag_perfect)

set -l searchURL (string join "&" (string join "" $baseURL $encoding) $_flag_english $_flag_image $_flag_nonperson)
```

これで、最初の `ggl` コマンドが完成しました。

:::details ggl.fish
@[gist](https://gist.github.com/yo-goto/7acfa712006488466d73ff42b9d952cc)
:::

## おわり

コマンドラインから google 検索したいというモチベーションで、fish 言語について調べて実装してみました。fish 言語の取っ掛かりとしてかなりよかったと思います。今回は、説明用に小さく色々書いてみたのでよい復習となりました。これから、fish を使って色々つくってみようと思います。

## 追記

gist から更に改造したコードを Github のリポジトリでプロジェクトとして公開しました。[fisher](https://github.com/jorgebucaran/fisher) を使ってインストールできますので、使ってみてください。

https://github.com/yo-goto/ggl.fish

```shell
fisher install yo-goto/ggl.fish
```
