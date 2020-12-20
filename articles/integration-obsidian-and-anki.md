---
title: "Obsidian To Ankiの使い方: ZettelkastenとSRSを組み合わせる"
emoji: "☄️"
type: "idea"
topics: ["Obsidian", "Anki", "PKM", "学習", "勉強"]
published: false
date: 2021-01-01
alias: [Obsidian to Anki]
---


# はじめに
前回の記事OHZフローから更に応用的運用を行うためのフローについてのBookにつながる内容です(現在作成中)。

現在の環境は以下です。
Obsidian: insider v0.10.4
Obsidian_to_Anki: v3.4.1

それぞれ開発の初期段階なので今後様々な変更があると思われます。注意してください。

---

# 二つのプラグイン
Obsidian内部の保管庫からAnki化するためのプラグインは二つある。
- [reuseman/flashcards-obsidian: An Anki plugin for Obsidian.md](https://github.com/reuseman/flashcards-obsidian)
- [Pseudonium/Obsidian_to_Anki: Script to add flashcards from text/markdown files to Anki](https://github.com/Pseudonium/Obsidian_to_Anki)

基本的にはreusemanのFlashcardプラグインの方が簡単で使いやすい。ファイルごとにAnki化するコマンドを打てたり、タグを指定するだけでAnki化できるので手軽にAnki化したい人にはreusemanのFlashcardがおすすめ。

ただ、ちゃんとやりたいタイプの人にはPseudoniumのObsidian_to_Ankiがかなりおすすめ。こちらではかなり個人の運用に最適化することが可能。しかしそれだけ文法が多くなったり、設定が複雑になるのでプラグインの使い方を覚えるのが大変。

また運用戦略なども考える必要がでてきたり、現在は保管庫全体をスキャンしてAnki化するのでむしろデメリットなどが存在する可能性もあるので注意して使う必要がある。

シンプルなAnkiカードやシンプルな運用が好きな人はreusemanのFlashcardを使ってほしい(Flashcardでできることは基本的にObsidian_to_Ankiでも可能だが、Obsidian_to_Ankiはファイル単位での同期がまだできない)。

# Obsidian_to_Anki

## Obsidian_to_Anki チュートリアル説明動画

https://youtu.be/PXyv6pnVGhA

Santi Youngerによる英語でのセットアップ説明動画。ノートタイプなどについては詳しい説明はないがチュートリアルとして分かりやすい。英語がわかる人はざっと見てみると概要が分かる。

今回はこの動画に書いてないような詳しいやり方について運用戦略と絡めて説明していく。

## wikiをcloneする
Obsidian_to_AnkiプラグインのWikiがあるので、それを絶対に参照してほしい。

まずはそれをCloneしてくる。[Home · Pseudonium/Obsidian_to_Anki Wiki](https://github.com/Pseudonium/Obsidian_to_Anki/wiki)にアクセスして
`Clone this wiki locally`のボタンをクリックしてURLをコピーする。

![[wikiclone.png]]

Termianalから適当なディレクトリに`git clone`する(Obsidianのテスト用の保管庫などにcloneすると便利)。

```shell
$ git clone https://github.com/Pseudonium/Obsidian_to_Anki.wiki.git
```

[[Obsidian_to_Anki wiki MOC|wiki用のMOC]]を作成して管理する。 基本的にはwikiの[[Home]]からすべてのコンテンツにアクセスできる。

wikiのリポジトリが入っている保管庫でこのプラグインのテストするとwiki内での説明に利用している文法によって**エラーが起きる**ので別のテスト保管庫を作成して同時起動し、そちらでプラグインのテストを行う必要がある。

cloneしなくてもwebから見れるのでgitを利用しない場合にはwebの内容をそのまま参照してほしい。

ちなみにこの記事をWikiをクローンした保管庫にコピペするとそのままWiki内のmarkdownノートにリンクできるので活用してみてほしい(リンク切れなどあったらごめんなさい)。

## Obsidianとの統合
--> [[Obsidian Integration]]

- URIを利用したソースとなるファイルへのリンクを自動生成することが可能
	- どのフィールドにリンクを作成するか設定から追加する。
- リンクサポート
	- markdown形式のリンク`[]()`とwikilink形式のリンク`[[]]`をフルサポートしている。
	- リンクはURIリンクに変換され、クリックすることでObsidianのノートを開く。
- 埋め込みサポート --> [[Image formatting]]
	- wikilinkの`![[]]`形式とmarkdownの`![]()`形式両方の音声と画像埋め込みをサポートしている。
	- 自動的に画像や音声がAnkiにコピーされる。
- タグのサポート
- 目次によるコンテキストサポート
	- 目次(#)レベルの情報をカードに追記。
- フォルダごとの設定を追加可能
	- フォルダごとにどのタグを挿入するかを設定可能。

# 実際の使い方
Anki化する際にはObsidian_to_Ankiの標準文法`START/END`でAnki化したい内容を囲み、コマンドパレットから`Obsidian_to_Anki: Scan Vault`を実行するかリボンにあるAnkiアイコンをクリックすることで自動的に保管庫全体をスキャンする(現時点ではファイルごとのAnki化ができない)。

後述するが、カスタムシンタックスを利用すれば標準文法の`START/END`を利用せずに`== ==`でハイライトした部分だけをcloze化するなどもできる。

## 基本的な文法
--> [[Note formatting]]

大文字の`START`と`END`で囲まれた部分がAnki化される。最初の行にノートタイプを書き、次の行からノートのフィールド名とフィールドに記載する内容を書く。

```markdown:文法
START
{Note Type}
{Note Fields}
Tags:
END
```

```markdown:具体例
START
基本
Front: これはテスト
Back: テスト成功
Tags: テスト
END
```

一番最初のフィールドのプレフィックス(具体例だとFrontフィールド)を省略することができる。
--> [[Field formatting]]

```markdown:最初のフィールドを省略
START
基本
これはテスト
Back: テスト成功
Tags: テスト
END
```

次の行にも続けることができる。
```markdown:行を続ける
START
基本
これはテスト
これもフロントフィールドに追加される
Back: テスト成功
Tags: テスト
END
```

**すべてのフィールドを書く必要がなく省略することができる**。

Obsidian_to_Ankiプラグインのコマンドを走らせると次のようにブロック内に一意のIDが挿入される。 --> [[Updating existing notes]]

```markdwon:Anki化後のIDが挿入される
START
基本
これはテスト
Back: テスト成功
Tags: テスト
<!--ID: 1566052191670-->
END
```

これによって、フィールドの内容を変更した後もコマンドを実行することでAnkiのカードをアップデートすることが可能になる。

## タグの指定
--> [[Tag formatting]]

タグに関しては、Anki化するブロックごとにタグを指定する(1)**ノートベースのレベル**とファイル内のすべてのブロックに共通のタグを指定する(2)**ファイルベースのレベル**の二つが存在する。

### ノートベース
複数のタグを挿入したい場合には次のようにタグの間にスペースを入れる(`Tags:`の後にもスペースが必要)。

```markdown
Tags: Tag1 Tag2 Tag3
```

### ファイルベース
`FILE TAGS`のキーワードと共にファイルのどこか(主にトップがボトムの箇所に)次のようなコードを置く。

パターン1
```markdown
FILE TAGS
数学 学校 物理
```

パターン2
```markdown
FILE TAGS: 数学 学校 物理
```

この二つのパターンのどちらかのコードをファイルのどこかに入れる。

## デッキの指定
--> [[Deck formatting]]

タグと同じ用に`TARGET DECK`キーワードと共にファイルのどこかに次のようなコードを挿入する。

パターン(1)
```markdown
TARGET DECK
数学デッキ
```

パターン(2)
```markdown
TARGET DECK: 数学デッキ
```


## マークダウンフォーマット
--> [[Markdown formatting]]

マークダウンのフォーマットは基本的にサポートされている。

## Cloze フォーマット
--> [[Cloze formatting]]
様々なスタイルからclozeを作成することが可能。もちろんAnki標準のclozeフォーマット`{{c1::クローズノート}}`をそのまま使うことも可能。

| Clozeスタイル                                                              | 変換後(Ankiカード)                                                     |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| これは{一つ目のcloze}になる。これは{二つ目のcloze}になる。                 | これは{{c1::一つ目のcloze}}になる。これは{{c2::二つ目のcloze}}になる。 |
| これは{2:二つ目のcloze}になる。これは{1:一つ目のcloze}になる。             | これは{{c2::二つ目のcloze}}になる。これは{{c1::一つ目のcloze}}になる。 |
| これは{1&#124;一つ目のcloze}になる。これは{2&#124;二つ目のcloze}になる。   | これは{{c1::一つ目のcloze}}になる。これは{{c2::二つ目のcloze}}になる。 |
| これは{c1:一つ目のcloze}になる。これは{c2:二つ目のcloze}になる。           | これは{{c1::一つ目のcloze}}になる。これは{{c2::二つ目のcloze}}になる。 |
| これは{c1&#124;一つ目のcloze}になる。これは{c2&#124;二つ目のcloze}になる。 | これは{{c1::一つ目のcloze}}になる。これは{{c2::二つ目のcloze}}になる。 | 

上のスタイルを混ぜて使用可能であり、id(番号)を指定しない場合にはid指定されてない最初のclozeが1番目として扱われ、順次カウントアップされるようになる。

具体例

```
idを指定しない場合の{複数}の{クローズ}では、{2:id指定したクローズ}と同じく、{c1|他のスタイル}と合わせて使うことができる。
```

↓Ankiでの変換後

```
idを指定しない場合の{{c1::複数}}の{{c2::クローズ}}では、{{c2::id指定したクローズ}}と同じく、{{c1::他のスタイル}}と合わせて使うことができる。
```

`{}`カーリーブラケットの内側に数式を書いてもOK。

## インラインノート
--> [[Inline notes]]
インラインでAnki化したい場合には次の
`STARTI`と`ENDI`で囲む。

```
STARTI [{Note Type}] {Note Data} ENDI
```

具体例
```
STARTI [Basic] This is a test. Back: Test successful! ENDI  
```

Basicノートタイプで保存される。

## ノートの削除
--> [[Deleting notes]]

ブロック削除にとくに気を付ける。内容を消すときはまずDELETEコマンドをいれておかないとダメ。プラグインのコマンドを走らせてから内容そのものを消す。

Anki化したブロックを削除したい場合には**直接そのブロックを削除してはいけない**。削除対象のブロックから離れた場所に最低一つは行を入れて次のコードを書いた状態でプラグインのコマンドを走らせる必要がある。

```
DELETE
ID: 123456789
```

IDには各ブロックに生成された番号で削除したいブロックの番号を指定する。

## Regex lineのスタイル(カスタムシンタックスのテンプレート)
Wikiの[[Regex]]ページに行くとカスタムシンタックスのテンプレートをリストアップしている。自分用にスタイルを最適なものを選べる。なんなら**自分専用のスタイルを完全にオリジナルで作成できる**。ObsidianやAnkiの運用にあわせたものを選ぶと良い。かならずしもカスタムシンタックスを使う必要はないが、ナレッジベースからスムーズにAnki化したい場合には利用したほうがよいかもしれない。

### テンプレートリスト
* [[RemNote single-line style]] : 
	* レムノートスタイル
	* `パラグラフ::これが答え`
* [[Header paragraph style]]
	* 目次(#)がFrontになり、配下の段落がBackになる
* [[Question answer style]] 
	* Q: 問題
	* A: 解答
	* 上のスタイルでどれだけスペースがあっても使えるので(正規表現)、かなり使えそう
* [[Neuracache flashcard style]]
	* `#flashcard`の前後でFrontとBackが定義される。
	* [[Flashcard Plugin]]がこれに当たる。
* [[Ruled style]]
	* 水平線で区切った部分でFrontとBackが定義される。
* [[Markdown table style]]
	* マークダウンのテーブルスタイル
	* テーブル外のものは見ない。
	* テーブルのヘッダーがFrontにそれ以外がBackになる。
	* 列は2つ以上でも一つとしてみなされる(二列を作っても一つの列として扱われる)。
* [[Cloze Paragraph style]]
	* Obsidianとの相性が良いため個人的にこれが一番使えると思う。
	* カーリーブラケット`{}`でcloze(Ankiの場合だと二重のカーリーブラケットが必要。例: `{{c1::test}}` )を作成できる。
	* 代わりにハイライトをcloze化することができる。

### ノートの削除
上記のスタイルを利用するときは、削除のプロセスでデフォルトとは違う次のコードを書く必要がある。

```markdown
{Delete Regex Note Line}  
<!--ID: 129840142123-->
```

デフォルトの設定だと`DELETE`でよい。
```markdown
DELETE  
<!--ID: 129414201900-->
```

異なるパターンが衝突しないように(かさなるようになると衝突する)気を付ける必要がある。

### Cloze Paragraph Style
Regex lineの設定: `((?:.+\n)*(?:.*==.*)(?:\n(?:^.{1,3}$|^.{4}(?<!<!--).*))*)`

ハイライト`== ==`をカーリーブラケット`{}`の代わりにcloze deletionにできる。設定項目で`CurlyCloze - Highlights to Clozes`も有効化する。

![[Cloze Paragraph style#Highlight-cloze style Obsidian Plugin only]]


## 設定画面

### Note Settings : Ankiのノートごとの設定
`Note Type Table`が存在する。

| Note Type          | Cutsom Regexp                                               | File Link Field                        | Context Field                          |
| ------------------ | ----------------------------------------------------------- | -------------------------------------- | -------------------------------------- |
| ノートタイプを指定 | カスタムシンタックス用のRegexを指定                         | ソースリンクを挿入するフィールドを指定 | コンテキストを挿入するフィールドを指定 |
| TestNoteForOBANKI  | `((?:.+\n)*(?:.*==.*)(?:\n(?:^.{1,3}$|^.{4}(?<!<!--).*))*)` | SouceLink                              | Context                                |

--> [[Regex]]

この設定項目から使いたい正規表現パターンを指定してあげる。例えば上の例で使っている`((?:.+\n)*(?:.*==.*)(?:\n(?:^.{1,3}$|^.{4}(?<!<!--).*))*)`のパターンを指定すると、ハイライトされた箇所があるブロックは強制的にTestNoteForOBANKIのノートタイプとしてAnki化されるようになる。

### Folder Settings : Obsidianのフォルダごとの設定

| Folder     | Foder deck                                               | Folder Tags                      |
| ---------- | -------------------------------------------------------- | -------------------------------- |
| フォルダ名 | フォルダ内のノートから生成されるAnkiカードのデッキを指定 | フォルダごとに挿入するタグを指定 |
| TestFolder | TestAnkiDeck                                             | TestObsidian                     | 


### Actions
- `Regenerate Note Type Table` : ここからノートタイプの再認識を行う
- `Clear Media Cache`
- `Clear File Hash Cache`

## 応用

### Frozen Fields
--> [[Frozen Fields]]
[AnkiのFrozen Fields](https://ankiweb.net/shared/info/516643804)プラグインからインスパイアされた機能。

対象となるファイルのどこかに次のコードを書く。

```
FROZEN - {Note type}:
{Field 1}: {Content 1}
{Field 2}: {Content 2}

```

具体例

```
FROZEN - English Vocabulary:
Context: TOEFL3800 Level 3

```

ファイル内のあらゆるAnki化用のブロックのノートタイプが｢English Vocabulary｣となり、コンテキストフィールドに｢TOEFL3800 Level 3｣が追加される。

エラーが起きないように他のコンテンツから離れるように一行あける。またファイルの一番下かトップに置いておくように推奨されている。


---


# 運用戦略

ObsidianのナレッジベースからAnki化するための戦略は概して2つある。

### パターン(1): Obsidianによるナレッジベース作成を主軸としたAnki化
Obsidianと相性の良いものがある。Obsidianでナレッジベースを作成することを考え、Ankiでカード化することを余り考える必要なくフラッシュカードを高速作成。

**具体的方法**
ハイライトやカーリーブラケットで問題にしたい部分を囲みcloze化する。ハイライトにするとObsidian内での表現性を損なうことなくスムーズにAnki化できる。

### パターン(2): Ankiカードを主軸としたObsidianでのナレッジベース作成
Ankiによる学習を主軸とした効率よく学習できるようなAnkiカードそのものを作成することを主眼にObsidianでノートを作成する。

**具体的方法**
Obsidian_to_Ankiプラグイン標準文法の`START/END`ブロックを作成して各種フィールドについて記述する。

### どちらがよいか?
どちらのパターンがよいかはトレードオフなので自分の運用(対象とするドメインや必要時間など)で決める必要がある。

ただ、ナレッジベースそのものを[ZettelkastenやLYTで作成すること自体に学習効能はあるので](https://zettelkasten.sorenbjornstad.com/#AnkiZettelkastenIsomorphism)テスト対策などでなければパターン(1)でナレッジベースを作成することに注力しその過程でAnki化するのがよいかもしれない。

基本的には内容やプロジェクト、時間によってどちらのパターンも両方利用したりミックスするのがよいと思われる。

タグに関してはファイルベースで指定して水平線と合わせると便利。


---
links: [[096 Obsidian MOC|Obsidian]] | [[Anki MOC|Anki]] | [[Obsidian_to_Ankiの雑感]]
srcs: 
type: #type-post
tags: #anki | #obsidian | #PKM |

---
