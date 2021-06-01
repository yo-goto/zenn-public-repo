---
title: "Obsidian_to_Ankiの使い方 : ZettelkastenとSRSを組み合わせる"
emoji: "💫"
type: "idea"
topics: ["Obsidian", "Anki", "PKM", "学習", "勉強"]
published: true
aliases: [OTA, Obsidian_to_Ankiの使い方]
tags: " #type-zenn #obsidian #anki #PKM "
---

# はじめに
前回の[OHZフロー](https://zenn.dev/estra/articles/ohzflow-zenn-hugo-obsidian)についての記事から更に応用的運用を行うためのフローについてのBookにつながる内容です(現在作成中)。

今回はObsidiain内部の**ナレッジベースからAnkiへとフラッシュカード化する方法**とそのプラグインを紹介します。前回はOHZフロー(Zenn & Hugo in Obsidian)だったので、今回も暫定的に**OTA(Obsidian To Anki)フロー**と呼んで名称の簡略化と概念化を行います。

Obsidianとこの記事で紹介するプラグインの両方とも開発の中途段階なので今後様々な変更があると思われます。注意してください。

現在の環境は以下です。
Obsidian: insider v0.10.4
Obsidian_to_Anki: v3.4.1

また著者も研究途中の場所があるのでその部分には触れていません。今後この記事を更新する際に順次追加する予定です(OHZフローでObsidian内部の保管庫から記事をpublishしているので更新頻度は高いです)。

｢そもそもObsidianて何よ?｣っていう人は[前回の記事](https://zenn.dev/estra/articles/ohzflow-zenn-hugo-obsidian)を見てみてください。

## 対象読者
- [Obsidian](https://obsidian.md)と[Anki](https://apps.ankiweb.net/)に興味がある人
- SRS([Spaced Repetition System](https://docs.ankiweb.net/#/background))に興味がある人
- ナレッジベースから効率的にフラッシュカードを作成したい人
- 長期的に記憶したい事がある人
- 情報を分散させたくない人
- **問題解決と学習のシームレスな統合**を行いたい人

## 注意点
Obsidian_to_Anki v3.4.1で起きうる可能性のある問題を｢現時点での課題｣として追記しました。**メインの保管庫で本格運用を行う前にテストを十分に行ってから**実践するようにしてください。

:::message alert
前回の記事で紹介した方法と今回のプラグインを組み合わると一部問題が発生します。詳しくは｢現時点での課題｣を参照してください。
:::

また、wikiそのものをまとめているこの記事自体の量がそこそこ多いです。Ankiを触れたことがないとちょっと厳しいので、まずはAnkiで遊んでみることからはじめて欲しいと思います。Ankiユーザーに有名な次のサイトがおすすめです。

https://rs.luminousspice.com/how-to-anki/

Anki自体もそうですが、紹介するプラグイン自体がかなりクセがあるので、たぶんこの記事を最初見た人たちの何人かは確実に躓くと思います。なので記事自体は読んでほしいですが、急ぎの方は｢ZettelkastenとSRS｣の項目だけ読んでもらって、あとは直接触らずに時間があるときに色々実験してみてください。

プラグインについての質問などあればコメント欄にしてください。

---

# ZettelkastenとSRS
今回はObsidian(方法論: Zettelkasten)とAnki(方法論かつアルゴリズム: SRS)を組み合わせるためのプラグインの使用方法とその運用について説明していくが、そもそもZettelkastenとSRS自体があまり知られてないので簡単に説明しておく。

ちなみに前回の記事で紹介したLYT(**Linking Your Thinking**)はZettelkastenをデジタル環境であつかいやすくするような応用的方法論なので分かりづらくなるかもしれないが注意してほしい。コア概念はZettelkastenとSRSだ。

## SRSとは?
**Spaced Repetition System**の略。**情報を復習するための理想的な時間を追跡し､ユーザーの記憶パフォーマンスに基づいて､その時間を最適化する**というシステム。AnkiはSRSを実装したソフトウェアであり、世界中の人々から愛されている。

https://apps.ankiweb.net

簡単にイメージしてもらうと[エビングハウスの忘却曲線](https://atsueigo.com/forgettingcurve/)を情報ごとに(例えば、英単語5000個それぞれの復習時間を)管理し、テスト(**Active Recall Testing**)によってその曲線を情報ごとに最適化していくようなシステムのこと。色々なアルゴリズムがあるがAnkiはSM-2アルゴリズムを採用している。ちなみにAnkiは、1972年ごろに提唱された概念｢Spaced Repetition｣をソフトウェアで実用化したSuperMemoにインスパイアされて開発されたユーザーフレンドリーなソフトウェアの名称である(名称は日本語の暗記から来ている)。

>SRSによって､情報の復習間隔をばらけさせることで3000枚ものフラッシュカードでもコントロール可能になる｡それぞれ**いつ復習すべきは､SRSが情報ごとに追跡して復習間隔を最適化してくれる**からだ｡人間はそれをいちいち考える必要性はない､むしろ大量の情報を分散的に追跡するのは人力では困難である｡使用しなければ忘れるという人間の記憶に対して､SRSというソフトウェアのシステムによって**情報をそれぞれ追跡管理することで､大量の情報に対して Active Recall Testing を可能とする方法**を実現した｡
> \- 自分のブログ[まずAnkiより始めよ](https://www.ankiyorihajimeyo.com/anki/mazuankiyorihajimeyo/)より引用

こまかいことはHugoのブログに載せてるのでそちらを参照してほしい。

余談になるがSRSの最先端になるとメディアそのものとSRSを組み合わせた｢[Mnemonic medium](https://notes.andymatuschak.org/z4rRX3qwSSJRsEkdXKwH2shamgHNeRthrMLiF?stackedNotes=z2fBHADWa93EZTuNzuww7V3Vi587ZyZ4FHTHm)(ニーモニックメディア)｣なるものが開発されているとのこと。

## Zettelkastenとは?
Zettelkasten(独: **ツェッテルカステン**、英: ゼトルカステン)とは第3世代システム理論｢オートポイエーシス｣を社会システム理論に応用したドイツの社会学者ニクラス・ルーマンが作り出したノートテイキング方法。具体的には、断片的なノート同士をリンクすることで、読んだり考えたことをすべて思い出せるようにし、更にはリンク(参照)によって浮かび上がってくる知識間の新たな関連性や思考のつながりを発見することを可能とする方法論である。[^1]

[^1]: [Zettelkastenメソッド - Zettlr Docs](https://docs.zettlr.com/ja/academic/zkn-method/)

https://gigazine.net/news/20200604-zettelkasten-note/

https://zettelkasten.de/introduction/

ルーマンが考案した時代においてはキャビネットで大量のカード(ノート断片)を管理していたが、今やデジタルデバイスを一人一台もつようになり、様々な人々がZettelkastenを実践するようになった。デバイス内部やインターネットなどのデジタル環境においてもハイパーリンクに始まり、双方向リンク、バックリンク、トランスクルージョン(埋め込み参照)技術などによってリンク自体も進化してきている。

ObsidianやRoam Research、Scrapbox、Notionなどの新しいメモツールやネット技術そのものの進歩によりZettelkastenのさらなる進化が起きている。その一端が[Evergreen Note](https://publish.obsidian.md/andymatuschak/Evergreen+notes)や[LYTフレームワーク](https://publish.obsidian.md/lyt-kit/MOCs+Overview)などである。

# 二つのプラグイン
Obsidian内部の保管庫からAnki化(OTAフロー)するためのプラグインは現時点で二つある。
- [reuseman/flashcards-obsidian: An Anki plugin for Obsidian.md](https://github.com/reuseman/flashcards-obsidian)
- [Pseudonium/Obsidian_to_Anki: Script to add flashcards from text/markdown files to Anki](https://github.com/Pseudonium/Obsidian_to_Anki)

基本的にはreusemanが開発したFlashcards-obsidianのプラグインの方が簡単で使いやすい。ファイルごとにAnki化するコマンドを打てたり、タグを指定するだけでAnki化できるので手軽にAnki化したい人にはreusemanのFlashcardがおすすめ。

ただ、ちゃんとやりたいタイプの人にはPseudoniumのObsidian_to_Ankiがかなりおすすめ。こちらでは**個人の運用に最適化することが可能**。しかしそれだけ**文法が多くなったり、設定が複雑になる**のでプラグインの使い方を覚えるのが大変。また運用戦略なども考える必要がでてきたり、現在は保管庫全体をスキャンしてAnki化するので人によってはデメリットなどが存在する可能性もあるので注意して使う必要がある。

シンプルなAnkiカードやシンプルな運用が好きな人はreusemanのFlashcards-obsidianを使ってほしい(Flashcardでできることは基本的にObsidian_to_Ankiでも可能だが、Obsidian_to_Ankiはファイル単位での同期がまだできない)。

今回はObsidian_to_Ankiの利用方法についてメインで解説していく。

## Obsidian_to_Ankiの特徴
--> [[Obsidian Integration]]

OTA(Obsidian_to_Anki)のプラグインでは**Obsidian特有の機能を活かしつつAnki化する事ができる**。以下にどのような統合が図られているか紹介する。

- **URIを利用したソースとなるファイルへのリンク**を自動生成することが可能
	- どのフィールドにリンクを作成するか設定から追加する。
- リンクのサポート
	- markdown形式のリンク`[]()`とwikilink形式のリンク`[[]]`をフルサポートしている。
	- リンクはURIリンクに変換され、**クリックすることでObsidianのノートを開くことができる**。
- 埋め込みのサポート --> [[Image formatting]]
	- wikilinkの`![[]]`形式とmarkdownの`![]()`形式両方の音声と画像埋め込みをサポートしている。
	- **自動的に画像や音声がAnkiにコピーされる**。
- タグのサポート
	- Obsidianの `#tag` をAnkiのタグに変換可能。
- 目次によるコンテキストサポート
	- 目次(#)レベルの情報をカードに追記。
- フォルダごとの設定を追加可能
	- フォルダごとにどのタグを挿入するかを設定可能。
- マークダウンフォーマットのサポート
	- マークダウンのフォーマットは基本的にサポートされている。--> [[Markdown formatting]]
	

# OTA導入の準備

## チュートリアル動画

@[youtube](PXyv6pnVGhA)

Santi Youngerによる英語でのセットアップ説明動画。ノートタイプなどについては詳しい説明はないがチュートリアルとして分かりやすい。英語がわかる人はざっと見てみると概要が分かる。

今回はこの動画に書いてないような詳しいやり方について運用戦略と絡めて説明していく。

## プラグインのインストール
AnkiとObsidianがインストールされている状態を前提として話を進める。

必要なアドオンAnkiConnectとObsidian_to_AnkiをそれぞれAnkiとObsidianにインストールする。

注意) AnkiとObsidian共にテスト用のプロファイルとテスト用の保管庫でまずは実験してみることをおすすめする。

- [AnkiConnect 1. AnkiWeb](https://ankiweb.net/shared/info/2055492159)
- [Pseudonium/Obsidian_to_Anki: Script to add flashcards from text/markdown files to Anki](https://github.com/Pseudonium/Obsidian_to_Anki)

**Anki側**
--> [[Setup]]
次の様に｢ツール｣ → ｢アドオン｣ → ｢新たにアドオンを取得｣ → ｢アドオンをインストールする｣まで行く。

![zennimage](https://storage.googleapis.com/zenn-user-upload/gwovmvagbzar7iczyiha4fji5olu)

![zennimage](https://storage.googleapis.com/zenn-user-upload/3azwik0efauypg3o3hevcrh8ugnz)

![zenn image](https://storage.googleapis.com/zenn-user-upload/8feaezdfy2udd1d1bs0mc3nqbi3k)

コードに`2055492159`をペーストしてOKをおすと自動インストールされる。AnkiConnetctを選択して右下の｢設定｣をクリックして、最後のところに画像のようにコードをいれる。

``` 
"webCorsOriginList": [
	"http://localhost",
	"app://obsidian.md"
]
```

![zennimage](https://storage.googleapis.com/zenn-user-upload/vlpnf39blldwp4fk0xqgdbzmodqw)

アドオンと設定を有効化するためにAnkiを再起動し、そのままの状態にしておく。

**Obsidian側**
｢設定｣ → ｢サードパーティプラグイン｣ → ｢閲覧｣ → Ankiで検索するとObsidian_to_Ankiのプラグインが出てくるので｢インストール｣する。｢インストールされたプラグイン｣の項目から有効化する。

![zennimage](https://storage.googleapis.com/zenn-user-upload/viqtqc9ol13loqd13vxjhbnh99zx)

これで二つの準備が完了した。

## wikiをcloneする
OTA(Obsidian_to_Anki)プラグインのWikiがあるので、必ず参照してほしい。
 gitを利用できる場合にはOTAプラグインのwikiを自分のローカル環境にcloneする。

[Home · Pseudonium/Obsidian_to_Anki Wiki](https://github.com/Pseudonium/Obsidian_to_Anki/wiki)にアクセスして
`Clone this wiki locally`のボタンをクリックしてURLをコピーする。

![zennimage](https://storage.googleapis.com/zenn-user-upload/y1vntt5b8rj5sdpxoquafcr0sdh6)

Termianalから適当なディレクトリに`git clone`する(Obsidianのテスト用の保管庫などにcloneすると便利)。

```shell
$ git clone https://github.com/Pseudonium/Obsidian_to_Anki.wiki.git
```

![zennimage](https://storage.googleapis.com/zenn-user-upload/3w78ar4app32bmhoevfoc7m1930u)

[[Obsidian_to_Anki wiki MOC|wiki用のMOC]]を作成して管理する。 基本的にはwikiの[[Home]]からすべてのコンテンツにアクセスできる。

wikiのリポジトリが入っている保管庫でこのプラグインのテストするとwiki内での説明に利用している文法によって**エラーが起きる**ので別のテスト保管庫を作成して同時起動し、そちらでプラグインのテストを行う必要がある。

cloneしなくてもwebから見れるのでgitを利用しない場合にはwebの内容をそのまま参照してほしい。ちなみにこの記事をWikiをクローンした保管庫にコピペするとそのままWiki内のmarkdownノートにリンクできるので活用してみてほしい(リンク切れなどあったらごめんなさい)。

![zennimage](https://storage.googleapis.com/zenn-user-upload/8z2pmd1vrjvvkzh0ts59zrqs0ad5)

---

(**ここからはAnkiの前提知識が必要となるので急ぎの方はブラウザバックしてください**)

# 実際の使い方
ここからはAnkiについての基礎知識(デッキ、ノートタイプ、タグなどの用語が分かる)があることを前提としてすすめる。

## 2レベルの条件指定
まず知っていてほしいことは、タグやデッキに関しては、Anki化するブロックごとに条件を指定する(1)**ノートベースのレベル**とファイル内のすべてのブロックに共通の条件を指定する(2)**ファイルベースのレベル**の二つが存在するということだ。

後で説明するが、ファイルベースの条件指定を利用すれば、いちいちすべてのブロックにデッキなどを指定する必要がなくなる。

## 基本的な文法
--> [[Note formatting]]

では準備がととのったことで実際の使い方を紹介していく。

Anki化するための一番易しい方法は、Obsidian_to_Ankiの標準文法`START/END`でAnki化したい内容を囲み、コマンドパレットから`Obsidian_to_Anki: Scan Vault`を実行するかリボンにあるAnkiアイコンをクリックすることで自動的に保管庫全体をスキャンする(現時点ではファイルごとのAnki化ができない)。

後述するが、カスタムシンタックスを利用すれば標準文法の`START/END`を利用せずに`== ==`でハイライトした部分だけをcloze化するなどもできる。

### 標準文法 START/END
大文字の`START`と`END`で囲まれた部分がAnki化される。最初の行にノートタイプを書き、次の行からノートのフィールド名とフィールドに記載する内容を書く(この`START`と`END`のキーワードは設定から変更することができる)。

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

`Tags:` はタグを書かない限り、省略したほうが良い。

### ノートレベルの複数タグ指定
--> [[Tag formatting]]

複数のタグを挿入したい場合には次のようにタグの間にスペースを入れる(`Tags:`の後にもスペースが必要)。

```markdown
START
基本
これはテスト
Back: テスト成功
Tags: Tag1 Tag2 Tag3
<!--ID: 1566052191670-->
END
```

### デフォルトノートタイプ
設定項目においてデフォルトとなるノートタイプは存在しない。つまり、`START/END`の標準文法で囲っても、**ノートタイプを正しく指定しない限りAnki化されない**。

『Anki側に｢基本｣というデッキが存在するとして』

```markdown:具体例1
START
基本
Front: これはテスト
Back: テスト成功
Tags: テスト
END
```

↑これはAnki化される。

```markdown:具体例2
START
Front: これはテスト
Back: テスト失敗
Tags: テスト
END
```

↑これはAnki化されない。

```markdown:具体例3
START
きほん
Front: これはテスト
Back: テスト失敗
Tags: テスト
END
```

↑これもAnki化されない(｢きほん｣というデッキがAnkiに存在しない場合)。

Anki化するには正しくデッキ名を最初に宣言する必要がある。

### インラインノート
--> [[Inline notes]]
インラインでAnki化したい場合には次の
`STARTI`と`ENDI`で囲む。

```markdown:インラインの基本文法
STARTI [{Note Type}] {Note Data} ENDI
```

```markdown:具体例
STARTI [Basic] これはテスト Back: テスト成功 ENDI  
```

この場合にはBasicノートタイプで保存される。しかし、次のように失敗するパターンが何個かあるので気をつけてもらいたい。

```markdown:失敗例1(ノートタイプを書かない)
STARTI これはテスト Back: テスト失敗 ENDI  
```

```markdown:失敗例2(ノートタイプが存在しない)
STARTI [BAsiC] これはテスト Back: テスト失敗 ENDI  
```

この二つのパターンは`START/END`ブロックでノートタイプを正しく書かないとAnki化されないのと同じように正常にAnki化されない。

### Cloze フォーマット
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


## ノートの削除
--> [[Deleting notes]]

Ankiへ送ったノートブロックの削除についてはとくに気を付ける必要がある。

Anki化したブロックを削除したい場合には**直接そのブロックを削除してはいけない**。削除対象のブロックから離れた場所に最低一つは行を入れて次のコードを書いた状態でプラグインのコマンドを走らせる必要がある。

```
DELETE
ID: 123456789
```

IDには各ブロックに生成された番号で削除したいブロックの番号を指定する。

複数のブロックを削除したい場合には次のようの`DELETE`ブロックを複数書く。

```
DELETE
ID: 123456789

DELETE
ID: 294014480
```

削除したいブロックを再度Anki化したい場合には次のようにブロック下部に挿入されているIDの行をすべて削除する必要がある。`DELETE`を実行したIDのブロックについては内容そのものを削除しない限りプラグインを走らせてもAnki化されることはない。

```
The idea of {cloze paragraph style} is to be able to recognise any paragraphs that contain {cloze deletions}.
<!--ID: 1609492666924-->
```


## ファイルの名称変更
Obsidian側のでソースとなるファイル(ノート)の名称を変更した場合には再びOTAのプラグインを走らせて更新する必要がある。自動でソースとなるノートのURIリンクをAnki側に送る設定にしているとそのURIリンクが切れてしまうからだ。

ファイル名を変更したら単純にプラグインを再度走らせるだけでAnkiカードを更新してくれる(コンテキストとURIが更新されるのは確認済み)。

## シンタックスハイライト

Backなどにコードスニペットを書けば、自動的にコードハイライトしたカードが作成される。

````md
START
OTA-Basic-Rev-Code
Front: TypeScriptのクラス
Back: 
```ts
class Point{
	x: number;
	y: number;
	
	constructor(x: number, y: number){
		this.x = x;
		this.y = y;
	}
	add(point: Point) {
		return new Point(this.x + point.y + point.y);
	}
}
// プロパティ、コンストラクタ、メソッドの定義

var p1 = new Point(0, 10);
var p2 = new Point(10,20);
var p3 = p1.add(p2); //メソッドの使用
```
Tags: Typescript
END
````

こんな感じ↓

![OTA-code](https://storage.googleapis.com/zenn-user-upload/t35qf82a1tk0a0aqwzusvkaoxpey)

これとは別にAnkiにPrism.jsを入れる方法などもある。
参考: [Ankiにprism.jsを入れる](https://www.ankiyorihajimeyo.com/anki/prism_codehighlight_into_anki/)

# ファイルベースの条件指定
ファイルベースの条件指定を利用すれば、いちいちすべてのブロックにタグやデッキなどを指定する必要がなくなる。これを組み合わせて使うのが標準的な使用方法になると思われる。

ちなみにFolder Settingsでも同様のことができる(こちらはフォルダ単位)。

## タグ指定 FILE TAGS
--> [[Tag formatting]]

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

## デッキ指定 TARGET DECK
--> [[Deck formatting]]

タグと同じ用に`TARGET DECK`キーワードと共にファイルのどこかに次のようなコードを挿入する。Anki化されるブロック内部やNote type Settingsでデッキが指定されてない場合にこのブロックで指定したデッキにAnki化されることになる。

パターン(1)
```markdown
TARGET DECK
数学デッキ
```

パターン(2)
```markdown
TARGET DECK: 数学デッキ
```

## フィールド凍結 FROZEN
--> [[Frozen Fields]]

[AnkiのFrozen Fields](https://ankiweb.net/shared/info/516643804)プラグインからインスパイアされた機能。

対象となるファイルのどこかに次のようなコードを書く。指定したノートタイプのフィールドを凍結させる(つまり、そのノートタイプのすべてのAnkiカードのフィールドを同一の内容を書くようにする)。

```markdown:文法
FROZEN - {Note type}:
{Field 1}: {Content 1}
{Field 2}: {Content 2}

```

```markdown:具体例
FROZEN - English Vocabulary:
CreatedDay: 2021-01-03
Source: Toelf 3800

```

具体例のように書くとそのファイル内のあらゆるノートタイプ｢English Vocabulary｣となるノートタイプになるブロックにおいて、CreatedDay(作成日)のフィールドに｢2021-01-03｣が追加され、Sourceフィールドに｢Toelf 3800｣が追加される。

エラーが起きないように他のコンテンツから離れるように一行あける。またファイルの一番下かトップに置いておくように推奨されている。

## 上記三つの組み合わせ

```markdown:具体例
TARGET DECK
数学デッキ

FILE TAGS
数学 学校

FROZEN - Math:
Context: Linear algebra

```

上記三つのファイルベースの条件指定を組み合わせるとこのようになる。これはファイルのすべてのノートブロックを｢数学デッキ｣にし、｢数学, 学校｣タグを挿入したAnkiカードを作成する。

ただし｢Math｣ノートタイプがある場合にはコンテキストフィールドに｢Linear algebra｣を挿入する。

タグ指定とデッキに指定は同様にファイルベースでの条件指定だが`FROZEN`だけ特殊なので気をつけてほしい。`FRONZEN`は作成日などの同一情報を複数のAnkiカードで同じフィールドに何回も書かなくてすむようにする機能だ。

# 設定画面

## Note Settings : Ankiのノートごとの設定
`Note Type Table`が存在する。

| Note Type          | Cutsom Regexp                                               | File Link Field                        | Context Field                          |
| ------------------ | ----------------------------------------------------------- | -------------------------------------- | -------------------------------------- |
| ノートタイプを指定 | カスタムシンタックス用のRegexを指定                         | ソースリンクを挿入するフィールドを指定 | コンテキストを挿入するフィールドを指定 |
| TestNoteForOBANKI  | ((?:.+\n)*(?:.*==.*)(?:\n(?:^.{1,3}$&#124;^.{4}(?<!<!--).*))*) | SouceLink                              | Context                                |

--> [[Regex]]

この設定項目から使いたい正規表現パターンを指定してあげる。例えば上の例で使っている`((?:.+\n)*(?:.*==.*)(?:\n(?:^.{1,3}$|^.{4}(?<!<!--).*))*)`のパターンを指定すると、ハイライトされた箇所があるブロックは強制的にTestNoteForOBANKIのノートタイプとしてAnki化されるようになる。

## Folder Settings : Obsidianのフォルダごとの設定

| Folder     | Foder deck                                               | Folder Tags                      |
| ---------- | -------------------------------------------------------- | -------------------------------- |
| フォルダ名 | フォルダ内のノートから生成されるAnkiカードのデッキを指定 | フォルダごとに挿入するタグを指定 |
| TestFolder | TestAnkiDeck                                             | TestObsidian                     | 


## 文法の設定
![zennimage](https://storage.googleapis.com/zenn-user-upload/c3mrwfpa93mcqb8ja36op55yqwwq)

上で説明したきた`START/END`などのキーワードによる文法そのものを変更することがで可能。

個人最適化することが可能だが、トラブルシューティングで困る可能性もあるので何も問題がなければこのままデフォルトの文法でいこう。

## 基本設定

![zennimage](https://storage.googleapis.com/zenn-user-upload/sgfuqfz15b0pt0b506iiwc1gcov2)

- Defaults Tag
	- 自動的にAnki化する際に挿入するタグを決める。
- Deck
	- 何もしていされてない場合に送られるAnkiのデッキを決める。
- Scheduling Interval
	- 保管庫を自動スキャンする際のインターバル。自動スキャンしない場合には0に設定して毎回手動でスキャンする。
- Add File Link
	- ソースとなるObsidianのノートへのURIリンクを追加するかどうか。
	- オンにする場合にはURIリンクを含めるフィールドをNote Type SettingのFile Link Fieldで指定する。
- Add Context
	- ファイル名と目次のレベルをコンテキストとして追加するかどうか。
	- オンにする場合にはコンテキストを含めるフィールドをNote Type SettingのContext Fieldで指定する。
- CurlyCloze
	- カーリーブラケット `{}` をCloze化するかどうか。
	- 利用する場合にはオンにした上でノートタイプを決めてあげてNote Type SettingsのCustom Regexpに正規表現パターンをコピペしてあげる。
- CurlyCloze - Highlights to Clozes
	- ハイライト `== ==` をCloze化するかどうか。
	- 利用する場合にはオンにした上でノートタイプを決めてあげてNote Type SettingsのCustom Regexpに正規表現パターンをコピペしてあげる。
- ID Comments
	- Anki化される各ブロックへ自動挿入されるIDをHTMLのコメントタグで囲むかどうか。
- Add Obsidian Tags
	- Obsidianの `#tag` をAnkiのタグとして扱うかどうか。


特にAdd File LinkとAdd Contextの設定では、File Link FieldとContext Fieldでどのフィールドにそれらを追加するか指定する。指定しないとエラーになって正しくAnki化できない。

![zennimage](https://storage.googleapis.com/zenn-user-upload/q4jknso982m1sizt0d00avtcpoh7)

CurlyClozeやHighlights to Clozesにしたい場合には設定を有効化した上で後で述べるRegex lineのカスタムシンタックス用正規表現のパターンを次のようにコピペしてあげる。

![zennimage](https://storage.googleapis.com/zenn-user-upload/vox3mqxtdd46xpgydbgrhvryrw7u)

## Actions
--> [[Data file]]

- `Regenerate Note Type Table`
	- ここからノートタイプの再認識を行う。
	- 新しくノートタイプを作成した場合やノートタイプを削除した場合などにはこのアクションを実行する。
- `Clear Media Cache`
	- メディアのキャッシュをクリアする。
	- メディアファイル(画像など)の名前を変更せずに内容を変えた場合などに利用する。
- `Clear File Hash Cache` 
	- キャッシュされたFile Hashをクリアする。
 

# Regex lineのスタイル(カスタムシンタックスのテンプレート)
Wikiの[[Regex]]ページに行くとカスタムシンタックスのテンプレートをリストアップしている。自分用にスタイルを最適なものを選べる。なんなら**自分専用のスタイルを完全にオリジナルで作成できる**。ObsidianやAnkiの運用にあわせたものを選ぶと良い。かならずしもカスタムシンタックスを使う必要はないが、ナレッジベースからスムーズにAnki化したい場合には利用したほうがよいかもしれない。

## テンプレートリスト
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
	* [Flashcards-obsidian](https://github.com/reuseman/flashcards-obsidian)の文法がこれに当たる。
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
	* Ankiのcloze文法`{{c1::これはAnki化される}}`もこの正規表現でcloze化される。
	* 代わりにハイライトをcloze化することができる(別の正規表現パターンが必要)。

これらの正規表現パターンを利用する場合には上の設定の項目で説明したNote Type SettingsのCustom Regexpにコピペする必要がある。

これを行うことでその正規表現パターンにマッチするブロックは自動的に指定されたノートタイプとしてAnki化される。｢標準文法`START/END`ではデフォルトのノートタイプが存在しない｣ということを述べたが、このカスタムシンタックスの場合には**指定した正規表現にマッチするものは前もって設定しておいたノートタイプとしてすべて強制的にAnki化される**。この点に注意してほしい。

## ノートの削除
上記のスタイルを利用するときは、削除のプロセスで~~デフォルトとは違う次のコードを書く必要がある~~。デフォルトでは、基本文法と同様に`DELETE`を書いて削除すればよい(キーワードは変更可能)。

```markdown
DELETE  
<!--ID: 129414201900-->
```

異なるパターンが衝突しないように(かさなるようになると衝突する)気を付ける必要がある。

## Highlight-cloze style
--> [[Cloze Paragraph style#Highlight-cloze style Obsidian Plugin only]]

ハイライト`== ==`をカーリーブラケット`{}`の代わりにcloze deletionにできる。

Regex lineの設定: `((?:.+\n)*(?:.*==.*)(?:\n(?:^.{1,3}$|^.{4}(?<!<!--).*))*)`

**使い方**
Regex lineに上の正規表現パターンをコピペした上で設定項目で`CurlyCloze`の項目と`CurlyCloze - Highlights to Clozes`の項目の両方を有効化する必要がある。

### ボールドやイタリックからのClozeパターンについて
ボールドやイタリックでもCloze化はどうかと考えてみたが、強調表現は問題作成とは別に必要であるので、やはりハイライト→Clozeのフォーマットが良い。(ボールドやイタリックにした部分もAnki上で表現される)

イタリックとボールドは正規表現的にも問題がありそう。**ボールド**(`** **`)に対して*イタリック*(`* *`)となる。

# 成功パターンと失敗パターン
紹介したOTAのプラグインはAnki化する際に失敗するパターンが多く存在するのでAnki化されるかテストした上でさらに正しい文法で書く必要がある。可能な限り失敗するパターンと成功するパターンを紹介しておく。

## ノートタイプの宣言
『デフォルトノートタイプ』の項目で書いたように標準文法の`START/END`でAnki化する場合にはノートタイプを正しく宣言しない限りAnki化されない。

## File Link FieldとContext Field

File Link FieldとContext Fieldを正しく認識させた上で、**フィールド指定を正しく行わないと**Anki化されない場合がある。ノートタイプをAnki側で変更したり、

## Ankiカードのフィールド追加と削除
また、フィールドの追加や削除をするたびにOTAのプラグイン側で`Regenreate Note Type`を行う必要がある。バグで再認識できないパターンがあるので再認識されるまでAnki側で新しいノートタイプを追加するなど色々いじる必要がある。

## cloze化でのパターン
cloze化ではいくつかAnki化されるパターンとされないパターンが存在する。相当ややこしいので自分でテストして理解しておく必要がある。まずハイライトをcloze化する場合には`Curly Cloze`もオンにする必要がある。ちなみにCloze化では一番最初のフィールドがcloze用のフィールドとして扱われる。

- `[{これはAnki化される}]`
- `{{< これはAnki化されない >}}` ← Hugoのショートコード
- `{これはAnki化される}` ← 単一のカーリーブラケット(インライン)
- `{{c1::これはAnki化される}}` ← Ankiの正しいCloze文法(二重カーリーブラケットにid指定)
- `{{これはAnki化されない}}` ← Cloze文法だが文法ミス
- `{{c1:これはAnki化されない}}` ←Cloze文法だが文法ミス
- `{{c1|これはAnki化されない}}` ← Cloze文法だが文法ミス
- `{ これは       Anki化される}` ← 単一のカーリーブラケット(インライン)
- `{ <br> これはAnki化される}` ← 単一のカーリーブラケット(インラインだがタグが入っている)
- `==これはAnki化される==` ← ハイライト

↓ 単一のカーリーブラケット(ブロック)
```markdown
{
これはAnki化されない
インラインじゃないとAnki化されない
}
```

↓ コードブロック内部のカーリブラケットとハイライト
````markdown
```
コードブロック内部では{カーリーブラケット}や
==ハイライト==はAnki化されない。
```
````

↓ ハイライトオンでCloze化される(Zennのnode module内のmdファイルでひっかかってしまう)
```md
これはAnki化される 
=========
```


# 現時点での課題

## 双方向同期はできない(Anki側での内容追加について)
Obsidain_to_Ankiであって、Anki_to_Obsidianではない点に注意。Anki側でフィールドの内容を変更したり、**メモをとってもObsidian側には反映されることはないし**、OTAのプラグインを走らせれば**それらの変更は更新されて上書きされるのでなかったことにされる**。

ただしObsidian側で、Anki化しているブロックの内容そのものを変更しない限りはAnki側で追加したフィールドの内容は消されない。変更がない場合にはプラグインを実行してもAnkiで追加した内容が消えることはない。

自動的にソースとなるノートへのURIリンクを挿入できるので、**変更したい内容があればそのリンクからObsidianの元ノートそのものを変更する必要がある**。

## フォルダ除外ができない
Obsidian_to_Ankiに現時点で無い機能として、(1)単一ファイルのスキャン、(2)特定フォルダのスキャン除外があげられる。これによって意図しない箇所でAnki化され、マークダウンファイルの該当箇所にIDが挿入される可能性がある。

Hugoのリポジトリを保管庫に入れている場合にはカスタムシンタックスでカーリーブラケットなどを利用してしまうと例えば、~~ショートコードの下にIDが挿入されてしまう可能性がある~~(ショートコードは正規表現で引っかるがAnki化されない)。また、テンプレート用のプラグインとしてTemplaterなどを利用している場合にもIDが挿入される可能性がある(挿入された)。

ハイライトをcloze化したい場合にも、｢必ずしもすべてのハイライトをAnki化したいという訳ではない｣ということがある。また、ハイライト用の正規表現で検索すれば分かるが、**Zennのリポジトリのnode_modules内のmdファイルが相当に引っかかってしまう**。

そういったこともあるので、現時点では使用方法に気をつける必要がある。利用する際には十分にテストして、バックアップをとった上で行ってほしい(git管理していれば大丈夫だとは思うが)。

### Folder Settingsが長くなる
これもフォルダ除外の話になるが、ZennやHugoのリポジトリを入れているとnode modulesなどのフォルダがあるため、Folder SettingsのFolder Tableがやたら長くなってしまう。

##  いくつかの打開策
フォルダ除外機能が実装されるまでの間でいくつかの打開策が考えられる。
1. reusemanのFlashcards-obsidianのプラグインを使う
2. 標準文法の`START/END`のみとりあえず使う
	- カスタムシンタックスを使う場合には正規表現検索をしたりテストして使えそうなものを選んで利用する
3. 別の保管庫で一時的に使用する
	フォルダ除外機能が出た時点でその保管庫からノートを移動させて再びプラグインを走らせる
4. 保管庫の切り分け方法を組み替える
	Nesetd Vaultを利用してAnki化するときのみそちらの保管庫からプラグインを走らせる


## 個人的解決策
4番目の打開策**Nested Vault**(ネストされた保管庫)を利用した方法で解決する。この方法であれば、特定のフォルダにあるノートのみをAnki化できる。上で挙げたようなフォルダ除外について考える必要がなくなるのでカーリブラケットやハイライトのCloze化での意図しないAnki化を防ぐことが可能。

ただし、Nested Vaultは**公式非推奨の方法**なので利用する場合には自己責任の元、テストした上で利用してほしい。Nested Vaultに関しては次のポストで使用法とリスクを参照。

- [Nested vaults - usage and risks](https://forum.obsidian.md/t/nested-vaults-usage-and-risks/6360/1)
- [Nested Vaults (vault within a vault)](https://forum.obsidian.md/t/nested-vaults-vault-within-a-vault/7366)

方法としては、親となる保管庫の直下にAnki用ディレクトリを作成後、｢別の保管庫を開く｣コマンドで｢保管庫としてフォルダを開く｣を選択しAnki用ディレクトリを選択して子となる保管庫を作成。OTAのプラグインをその保管庫内部にインストールして、その保管庫からプラグインを走らせる(親の保管庫ではOTAのプラグインをアンインストールしておく)。

運用としては、編集は親の方の保管庫で行う。Anki化したいときだけNested Vault(Anki用のsub directory)を開いてプラグインを走らせる。

Anki化したノートを移動させることは可能で、Nested VaultにインストールしたOTAを削除した後に親の保管庫でOTAを再インストールして再びプラグインを走らせればAnkiカードを更新してくれる。つまり、**フォルダ除外機能が実装されるまではNested Vaultで運用して、実装されたらNested Vaultを解体して親の保管庫で通常通りに運用**することができる。単純にNested Vault内部の`.obsidian/`など保管庫の設定用ファイルが入ったディレクトリ等を削除すればそのままもとの子ディレクトリとして活用することも可能。

:::message
**Nested Vaultでの制限**
Nested Vault(ディレクトリ)外に存在するノートへのリンクなどが認識することができない。Anki化する際にリンクを書いてもNested Vault外部にあるノートへのリンクとなるURIではVaultの指定をするため存在しないリンクをクリックしても開かない。
:::


---

# 運用戦略
ObsidianのナレッジベースからAnki化するための戦略は今のところいくつか考えられる(なにか他の戦略があればコメントで教えてください)。

(--> [[ObsidianとAnkiとGoodNotes併用]] "2021-01-16" 分岐)

## パターン(1) Obsidianによるナレッジベース作成を主軸としたAnki化
Obsidian内ので表現と相性の良いものがある。Obsidianでナレッジベースを作成することを主軸として考え、その過程でフラッシュカードを高速作成する。

**具体的方法**
ハイライトやカーリーブラケットで問題にしたい部分を囲みcloze化する。ハイライトにするとObsidian内での表現性を損なうことなくスムーズにAnki化できる。

:::message alert
上で述べたように前回記事で紹介したHugoやZennのリポジトリを入れた状態でハイライトやカーリーブラケットでcloze化すると意図しない場所が書き換えられられる可能性があります。
:::

## パターン(2) Ankiカードを主軸としたObsidianでのナレッジベース作成
Ankiによる学習を主軸とした効率よく学習できるようなAnkiカードそのものを作成することを主眼にObsidianでノートを作成する。

**具体的方法**
Obsidian_to_Ankiプラグイン標準文法の`START/END`ブロックを作成して各種フィールドについて記述する。

## パターン(3) そもそもObsidianからAnki化しない
Obsidianでナレッジベースを作成してからAnki化するのに向いているものや向いていないものがあると思われるのでそもそもObsidianからAnki化しないという戦略もありえる。

例えば、英単語などのAnkiカードを高速作成したり管理するには、Anki標準の機能やAnkiのプラグインを活用したほうがよく、Obsidianのナレッジベースと分離したほうがよい(と個人的には思う)。辞書からのソースが多い英単語などをナレッジベースにいれてしまうとノイズ増加につながる可能性がある。管理の観点からもデータベースとして扱うことの方がよいものはAnkiメインでの管理がよいと思われる。

逆に数学や歴史、プログラミング等はObsidianからAnki化するのに向いていると思われる。英語の学習でも、長文などの管理やWritingの勉強に適しているかもしれない(そこから表現をAnkiに飛ばしたりするなど)。

## パターン(4) 既存のAnkiカードにObsidianのURIリンクを挿入する
AnkiのヘビーユーザーはAnki自体にすでに大量の知識資産があると思われる。それらのよさを残したままObsidianとの統合を図りたい場合には、ObsidianのURIリンクを既存のAnkiカードのフィールドに挿入するなどの方法がよいかもしれない。

例えば、医学などのAnkiカードが大量にある場合にはObsidianでのまとめノートなどのURIをフィールドのどこかに挿入するなどするのはどうだろうか?

## どのパターンが良いのか?
どのパターンがよいかはそれぞれメリットデメリットのトレードオフがあるので自分の運用(対象とするドメインや必要時間など)によって決める必要がある。

ただ、ナレッジベースそのものを[ZettelkastenやLYTで作成すること自体に学習効能はあるので](https://zettelkasten.sorenbjornstad.com/#AnkiZettelkastenIsomorphism)テスト対策などでなければパターン(1)でナレッジベースを作成することに注力しその過程でAnki化するのがよいかもしれない。

基本的には内容やプロジェクト、時間によってどちらのパターンも両方利用したりミックスするのがよいと思われる。

# Ankiテンプレートを作成してテスト
ここまできてようやく実際にAnki化のためのAnkiのノートタイプテンプレートを作成する段階にはいる(長い)。

1. Ankiのテストプロファイルを作成
2. Obsidainのテスト保管庫を作成
3. Ankiでテスト用のノートタイプを作成
4. 標準文法`START/END`で実験してみる
5. カスタムシンタックスで実験してみる
	1. 検索欄で正規表現を使ってどのくらい引っかかるか見てみる
6. 自分の保管庫で実験してみる(気をつけて)
7. 運用戦略を考慮して本格導入

とりあえず標準文法`STRAT/END`用とカスタムシンタックスのcloze用の二つテンプレートを作成しておけばよい。それぞれのテンプレートで用意するフィールドはだいたい4つ程度。ClozeとなるFrontフィールドはAnki側で一番最初のフィールドにする必要がある。

1. Front : 問題文 or clozeの問題文
1. Back : 解答 or clozeの場合には追加情報など
1. SourceLink : ObsidianのソースとなるファイルへのURIリンクを挿入する
1. Context : 目次のレベルを挿入する

この4つを最低限用意して実験してみよう。実験なのでいい加減なテンプレートでよい。

```html:BasicのFrontsideテンプレート
{{Front}}
```

```html:BasicのBacksideテンプレート 
{{FrontSide}}
<br>
<hr>

{{Back}}
<br>
{{SourceLink}}
<br>
{{Context}}

```

```html:clozeのFrontsideテンプレート
{{cloze:Front}}
```

```html:clozeのBacksideテンプレート
{{cloze:Front}}
<br>  
<hr>
{{Back}}
<br>
{{SourceLink}}
<br>
{{Context}}
```

用意できたら、Basicの方で`START/END`の実験とclozeの方でカスタムシンタックスの実験を行ってみよう。

カスタムシンタックスについては、正規表現パターンがどのくらい引っかかるか検索で知ることができる。例えば次のハイライトをAnki化するパターンを検索してみてほしい。
`/((?:.+\\n)\*(?:.\*==.\*)(?:\\n(?:^.{1,3}$|^.{4}(?<!<!--).\*))\*)/`

検索にはかなり引っかかると思うが、マッチしたものすべてがAnki化されるわけではないというのがやっかいなところ。実験した結果わかったのは**インラインのものしかAnki化されない**。

カーリーブラケット`{}`とハイライト`== ==`はインラインで使われているもののみがAnki化される。

# Obsidianテンプレートの設定

## START/END用テンプレート
Templateプラグインで標準文法`START/END`用のテンプレートを用意しておくと便利になる。次の5つのフィールドを設定したノートタイプでAnki化する際に活用できる。

**フィールド構成**
1. Front : 問題文
1. Back : 解答
1. Title : タイトル(あってもなくてもよい)
1. SourceLink ← 設定で自動化するため書かない
1. Context ← 設定で自動化するため書かない(場合によってはこれをタイトルとして扱う)

```markdown:最も簡単なテンプレート
START
// ← ここにノートタイプを正しく記述しないとAnki化されない(利用の際はこのコメントを削除する)
Front:
Back:
Title:
END
```

# 他メモ
## Ankiカード作成日に関して
作成日が変わることはないが、変更日はちゃんとプラグインで更新があったものだけ変更される。

---


# Self
## 記事変更ログ
- 2021-01-01
	- 記事公開
- 2021-01-02
	- [[#運用戦略]]の項目に追記
	- [[#ZettelkastenとSRS]]の項目を追加
- 2021-01-03
	- [[#注意点]]と[[#現時点での課題]]の項目を追加
	- 大幅に目次構造を書き換え
- 2021-01-08
	- [[#成功パターンと失敗パターン]]の項目を追加
	- [[#Obsidianテンプレートの設定]]の項目を追加
- 2021-01-12
	- [[#個人的解決策]]の項目を追加
- 2021-04-22
	- [[#シンタックスハイライト]]の項目追加

## Meta

links: [[096 Obsidian MOC|Obsidian]] | [[Anki MOC|Anki]] | [[Obsidian_to_Ankiの雑感]] | --> [[OTA追記メモ]]
url: [Obsidian_to_Ankiの使い方 : ZettelkastenとSRSを組み合わせる](https://zenn.dev/estra/articles/integration-obsidian-and-anki)

