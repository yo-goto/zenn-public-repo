---
title: "Zenn & Hugo in Obsidian : OHZフローによるナレッジベースとアウトプットコンテンツの完全統括"
emoji: "💎"
type: "idea"
topics: ["zenn", "obsidian", "hugo", "PKM", "マークダウン"]
published: true
aliases: [OHZフロー, OHZflow]
tags: " #obsidian #Hugo #Zenn #PKM "
---


# はじめに
ZennやHugoをObsidianで利用して運用するためのアイデアとその方法をまとめました。[スクラップ上でまとめたもの](https://zenn.dev/estra/scraps/644e7c229c3dcb)を記事化してます(追加の内容あり)。(現在、より広域な概念を包括したフローに関する内容を追記したbook作成をすすめています。)

日本ではまだ広く知られていないPKM周りの概念や新たなツールの紹介も含むの分かりづらいところや説明が長くなってしまう部分もありますが興味を持っていただければ幸いです。

また、ここで紹介するやり方はObsidianの公式で紹介されてないような応用的な使い方になるので潜在的な問題があるかもしれません。自己責任の元で適宜各ディレクトリのバックアップを行いながら環境構築をしてください。(~~自分の環境では現在のところ特に問題は発生していません~~)

:::message alert
次の記事で出てくるObsidian_to_Ankiとの併用で一部問題が発生しました。詳しくは｢現時点での課題｣を参照してください。Obsidian_to_Ankiのプラグインを利用しない場合には問題ありません。
:::

https://obsidian.md

![graphview](https://obsidian.md/images/screenshot.png)

## 対象読者
- メモ魔の人
- EvernoteやNotion等のメモアプリを使ったことがある人
- ナレッジベースに興味がある人
- 新しいツールが好きな人
- プレーンテキストが好きな人
- マークダウンが好きな人
- Zennを使い始めた人
- Zennだけでなくブログなども書きたい人
- 記事をエディタからコピペしたくない人
- 効率的なアウトプットをしたい人
- 学習を効率化したい人
- テキストを書くすべての人

感想や疑問、アイデア等があればDiscussionの方に気軽にコメントしてください。

---

# Obsidian/Hugo/Zenn

[Obsidian](https://obsidian.md/)内にて[Hugo](https://gohugo.io/)と[Zenn](https://zenn.dev/about)を利用して、ナレッジベースとアウトプットコンテンツのローカル環境での**完全統括**を目指し、あらゆるノートとそこからアウトプットとして作成した記事に散らばる**アイデアと知識の再利用と循環**を行うための環境(知識循環メモ環境)構築のための方法を考える。

- Obsidian : Markdown knowledge base app
	- **ローカル環境**と**マークダウン**ファイルを使った新しいPKMツール
	- graphビューや[ウィキリンク](https://en.wikipedia.org/wiki/Help:Link)というwikipediaで使われてる二重ブラケット記法`[[wikilink]]`やマークダウンのリンク記法`[]()`を使って知識やアイデアをつなげることができる。
- Hugo : Static site generator
	- **ローカル環境**で書いた**マークダウン**ファイルを使って静的サイトを爆速でビルドできるツール
	- gitとNetlifyなどのホスティングサービスを利用すれば無料で静的サイトを運用できる
- Zenn : Knowledge base Community
	- [note](https://note.com/info/n/nea1b96233fbf)のような課金機能を搭載している
	- デベロッパー用のナレッジベース&コミュニティ
	- 本や記事を**マークダウン**で書ける
	- zenn-cliとgitを使えば、**ローカル環境**で執筆できる

すべて**ローカル環境**と**マークダウン**を利用してコンテンツ作成が可能(ただしマークダウンのフレーバーがそれぞれ若干異なる部分があるのでそこは注意)

ZennやHugoについては知ってるけどObsidianについてはもっと基本的なことから知りたいという人がいると思うので、その場合には[jMatsuzakiさん](https://jmatsuzaki.com/archives/author/jmatsuzaki)と[滝林さん](https://twitter.com/ntakibay)がかなり詳しく紹介されているので参照してみてほしい。

https://jmatsuzaki.com/archives/26813

https://note.com/takibayashi/n/n4e5e7c505e16

(--> [[Obsidianとは]])

# PKMとは
そもそもPKMとは何か?

**Personal Knowledge Management** (個人知識管理)の略。PKMのツールとしてPKB(Perosnal Knowledge Base)がある。

PKBとして[Evernote](https://evernote.com/intl/jp)や[Notion](https://www.notion.so/)などの電子メモツールなどが相当するが、[Scrapbox](https://scrapbox.io/product)や[Roam Research](https://roamresearch.com/)、Obsidianといった[networked thought](https://nesslabs.com/networked-thinking)(ネットワーク化された思考)を補助し｢[第二の脳をつくる](https://www.buildingasecondbrain.com/)｣ためのアプリケーションツールが最近注目されている。wikipediaのようなリンクをつかったハイパーテキストによるwikiを個人で作り上げることができる。

>Networked thinking is an explorative approach to problem-solving, whose aim is to consider the complex interactions between nodes and connections in a given problem space. Instead of considering a particular problem in isolation to discover a pre-existing solution, **networked thinking encourages non-linear, second-order reflection in order to let a new idea emerge**.
\- [Networked thinking: a quiet cognitive revolution](https://nesslabs.com/networked-thinking)より引用

より具体的に言えば、拡張知能(Augmented Intelligence)や認知拡張(cognitive augmentation)として、人間の非線形的な思考形態を補助するような思考のためのツール(tool for thought)として着目されている。

![networked thought](https://storage.googleapis.com/zenn-user-upload/ro902hpbothfql568ztvjf1a98pj)

Obsidianはアウトライナーである[Dynalist](https://dynalist.io/)の開発チームが現在開発しているマークダウン&ナレッジベースツール。

この分野は特に2019年頃からアメリカ等で盛り上がってきている。[ObsidianやPKMに関するリンク](https://www.ankiyorihajimeyo.com/obsidian/links_obsidians/)をHugoのサイトの方でまとめているので気になったら見てみてみてね。

# PKBの種類や構成について
PKBには幾つかの特性があり、それぞれの得意分野が存在する。それがNick Miloの次の動画で詳しくわかる。

@[youtube](_x54XJrECvk)

簡単に要約すると、PKBのアプリケーションは
Collector/Databaser/Writer/Connecterの4つの機能の構成割合から次のように分けられる。

- Collector : Evernote
- Writer : [Bear](https://bear.app/), [Ulysses](https://ulysses.app/)
- Connector Databaser : Roam Research
- Connector Writer : Obsidian

もちろんRoamやNotionといったものにはそれぞれの得意分野が存在する。例えば、Notionはチームでのwiki作成、運用などが得意であったり、Roamはアウトライナーとしての能力があり、それぞれ素晴らしいツールとなっているので、自分にあったツールを見つけて使うのが良い。

ただ、Nick曰く、ObsidianはWritingに的したPKMツール(ナレッジベースかつアウトプットできる)としての能力を兼ね備えているので、ナレッジベースからそのままアウトプットをしたい場合にはObsidianはかなり役立つと思われる。

# 得られるメリット

ObsidianのVault(保管庫)内部にてHugoやZenn用のローカルリポジトリを作成すれば、自分のナレッジベースとサイトやブログ、知見記事、本など**すべてをマークダウンファイルで一元化させて管理することができる**。

1. ナレッジベースとアウトプットコンテンツの間の**情報のズレを無くす**ことができる。
	- 記事を公開するのにコピペする必要がなくなる。インターネット上ではレイヤーの異なるアウトプットコンテンツが全て自分のナレッジベースに統括&同期されているので**情報の重複やズレを無くすことができる**。
2. あらゆる情報や知識を**リンク**によって接続することで新たなアイデアを発見したり、知識を創造できる。
	- `[[wikilink]]`によって様々なノートとアウトプットされた記事をリンクすることでローカル環境においてナレッジネットワークを作成することができる。
	- グラフビューによってネットワークの可視化と効率的なネットワークのナビゲーションを行うことが可能になる。
3. **埋め込み参照**によって知識の様々なレベルでの**再利用性**が高まり、デジタルライブラリ内で**効率的に循環**させることが可能になる。
	- Obsidianの埋め込み機能`![[]]`や`![]()`を使えば学習効率や問題解決のスピードがあがる。
	- **ナレッジベースから高速で自分のブログの投稿や記事といったアウトプットコンテンツにアクセスし参照することができる**。
	- 逆にすべての記事をナレッジベースの一部として扱うことができるので、再びアイデアを発見できるかもしれない。
4. 一元化された環境内部で**アウトプットの高速化と頻度向上が可能**になる。(可能性としてのメリットだが、少なくとも記事を書くことの壁が低くなる)
	- アイデアにすらなっていないような思考状態のものから、アイデア、そこから洗練され完成した知識や知見、さらには本まで様々なレベルのものを一元化させてナレッジベース内で完結することで、スムーズにアウトプットできるようになる。
5. 仮に別のツールやプラットフォームに移動する時が今後きたとしても、データはローカルのマークダウンファイルとして存在しつづけるため、ツールやプラットフォームの移動が楽になる。
	- このこと自体が精神的な安心感を与えツールそのものをつかいつづけることができる。

上に列挙したメリットはそれぞれ相互に関連している。

## 1. 情報の重複やズレを少なくできる

『No more Copy & Paste』

Wordpressやnoteなどで記事をつくったことがある人はわかるかもしれないが、自分の好きなエディタで記事を書いても、記事をそのまま公開することはできない。**エディタからいちいち専用のwebページにコピペし直すという工程**が存在する。

何が問題かというと、記事に変更や更新があったときにそのwebページから変更や更新を行う必要があるということだ。それによって、もともとのエディタで書いていた記事と実際に公開されている記事の間で情報のズレが発生することになる。もちろん、エディタで直してから再びコピペし直せばよいのではないかという話はあるが、けっこう面倒くさい。というのも見た目をリアルタイムで確認できない場合があるからだ。変更や更新をしてプレビューしたいことも多いので結局エディタでやらずに直接編集するということになってしまう。これによって公開されている記事の方がローカル環境より最新という状態になってしまう。これではナレッジベース内で再利用することに壁ができてしまう。**自分のナレッジベース内の情報こそが最新状態であるという状態**が望ましい。

ところが、HugoやZennなどであればcliを使ってコマンドを叩けばブラウザからローカルホストでリアルタイムのプレビューを見ることができる。これにより、プレビューを見るためにwebページで編集し直して、エディタ元のオリジナルの記事とズレてしまうということをなくすことができる。**常にローカルとPublishされた記事の情報が同期されている、またはローカル環境の方が最新状態ということにできる**。

## 2. リンクによるナレッジネットワークの形成

### レイヤーを超えたリンキング
各アウトプットコンテンツにリンクリストを作成したり、`[[wikilink]]`を埋め込めばそのままObsidianのネットワークの構成要素として利用することが可能。インターネット上で異なるレイヤー(個人ブログとプラットフォーム)に属しているコンテンツがそのレイヤーを超えてローカル環境でネットワーク化される。

![local graph](https://storage.googleapis.com/zenn-user-upload/d9z4xmfktrq1wxrgzn2ql78r6iz4)
*↑この記事のローカルグラフ(深度 : 1)*

![local graph depth-3](https://storage.googleapis.com/zenn-user-upload/db648lg195qa0q1l2lvmth7zwvho)
*↑この記事のローカルグラフ(深度 : 3)*

この記事の下にリンクリストを設けた。このリンクによるローカルネットワークが右のグラフビューに現れている。**グラフの深度を調整することでこのノートを起点としたリンクの接続を様々なレベルで視覚化できるようになる**。

[ヘルプドキュメントの翻訳プロジェクト](https://zenn.dev/estra/articles/translate-with-gitandgithub)で作成したドキュメントのノードグラフとネットワーク化された一連のメモ郡↓
@[tweet](https://twitter.com/ankiyorihajimey/status/1382644621956714500?s=20)

### ナレッジベースからアウトプットコンテンツへ高速アクセス

![sidebarMOC](https://storage.googleapis.com/zenn-user-upload/jbhh36qo4vt4fyj3j5491z8q51ip)

後で説明するMOCを作成すると(要するにリンクのリスト)高速で自分のアウトプットコンテンツにローカルでアクセスできる。

サイドバーにそのリンクリストを置いておけば、一瞬でアクセスできる。その上、プレビューのプラグインでカーソルをあわせると内容をポップアップで確認できる。

@[tweet](https://twitter.com/ankiyorihajimey/status/1324387918484402177)

また｢検索結果をコピー｣する機能を利用すれば、リンクリストを高速で作成できる。

### エイリアスによる超高速アクセス
ObsidianのYAMLフロントマターにはエイリアスという項目が存在する。

例えば、この記事のYAMLフロントマターには`alias: [OHZフロー, OHZflow]`という項目を記述している。これによって、Obsidianのどこからでも[クイックスイッチャー](https://publish.obsidian.md/help/Plugins/Quick+switcher)を使ってそのエイリアス名`OHZフロー`または`OHZflow`でこのノートを呼び出すことができる。

```yaml:Zennの記事内のyamlに記述したエイリアス
---
title: "Zenn & Hugo in Obsidian : OHZフローによるナレッジベースとアウトプットコンテンツの完全統括"
emoji: "💎"
type: "idea"
topics: ["zenn", "obsidian", "hugo", "PKM", "マークダウン"]
published: true
alias: [OHZフロー, OHZflow]
---
```

もちろんHugoの記事にもエイリアスを設けている。これで記事のファイル名に関係なく、すぐに記事を呼び出すことができる。

```yaml:Hugoの記事内のyamlに記述したエイリアス
---
title: Obsidian PKM Map
type: docs
alias: [pkm map, Obsidian PKM Map]
---
```
 
![エイリアスの呼び出し](https://storage.googleapis.com/zenn-user-upload/6nvj8f16ynfbrzbw90cnwzjqo4ug =900x)
*クイックスイッチャーを利用したエイリアスによるノートの呼び出し*

## 3. 情報や知識の再利用による循環
Obsidianの参照機能には三段階ある(ページレベル･段落レベル･ブロックレベル)。あらゆる自分のアウトプットとしての記事やブログのポストから参照することができる。

![トランスクルージョンによる埋め込み](https://storage.googleapis.com/zenn-user-upload/i9jec242qecsevy1rzd1dycgm495)
*↑段落レベル埋め込み*

↑左のノート(Hugoのサイト記事)の段落を[トランスクルージョン](https://ja.wikipedia.org/wiki/%E3%83%88%E3%83%A9%E3%83%B3%E3%82%B9%E3%82%AF%E3%83%AB%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3)で右のノートに埋め込んでいる。

## 4. アウトプットの高速化と頻度向上
今まで述べてきた1~3のメリットからあらゆる知識や情報が格納されたナレッジベースかつアウトプットコンテンツの生成環境の両方を兼ねることができるが、これのおかげで[Evergreen Note](https://publish.obsidian.md/andymatuschak/Evergreen+notes)を**高いクオリティで実現**できる。すべての公開されている記事がローカル環境において最新であり、インターネット上で切り分けられているあらゆるコンテンツがローカルではネットワークの構成要素として機能する。

何か記事を更新しようと思えば、ローカル環境でObsidianを開けば、そのまますぐに書ける。

記事ではなく、未発表のアイデアにしたいならナレッジベース内のノートにかけばよい。記事と関連するならリンクすればよい。なにせローカル環境が一番最新であり、あらゆるレイヤーにあるすべての情報が存在している。

**『思考の断片』から『アイデアの粒』、記事の『ドラフト』から『完成品の本』まで、あらゆるアウトプット記事とナレッジベース内のすべてのノートが並列に並んでおり、同時にネットワーク化されている**。あくまで可能性としての話だが、これによりアイデアや知識の状態をスムーズにあげていくことが可能になるはずだ。

![すべての記事がナレッジベース内部で並列](https://storage.googleapis.com/zenn-user-upload/yix5o25jppslah3pynq2nefebvme)

左がHugoの記事、中央がナレッジベース内ノート、右がZennのこの記事。

### Evergreen Notes
これによって、すべてのノートと記事そのものを高いクオリティで[Evergreen Note](https://publish.obsidian.md/andymatuschak/Evergreen+notes)化することが可能になる。

@[tweet](https://twitter.com/Mappletons/status/1277262380775415809)

Evergreen Notesとは、Andy Matuschak氏が考案したノートテーキングの概念であり方法論。

一過性のノートによる知識やアイデアの損失を防ぐために、あらゆるノートを蓄積し、さらにそのノートに再訪して情報を追加･修正し、時間経過と共にそのアイデアや知識を成長させ、さらに有機的に別のアイデアや知識に接続させようというような考え。[いくつかの原則](https://publish.obsidian.md/andymatuschak/Evergreen+notes)がある。

上で述べてきたようにナレッジベースからアウトプットへのパスをObsidian内の環境において整備することによってアイデアとノートの成長を促しEvergreenにすることができる。実際にやっている感覚としては『記事』を更新するのではなく、自分の『ナレッジベース内の一つのノート』を常に更新している感じだ。

(--> [[Evergreen Notesの状態]] "2021-01-04" 分岐)

### メモのクオリティについて
ライブラリに存在するノートはナレッジベースとしてのクオリティで作成すると記事に転換する際に書き直す必要性がでてきてしまうので、**効率的にアウトプットできるように日頃からクオリティやわかりやすさに気をつけてメモを行う**ようにする。常にアウトプットを意識してメモをとるようにしよう。

参考記事 : 
https://jmatsuzaki.com/archives/27167

## 5. ツールやプラットフォームの移行のしやすさ
フレーバーが若干異なるところを除けば、すべて基本的な文法は一緒であるローカル環境のマークダウンファイルを利用している。

これは、大きなメリットとなる。というのもこれはWordPressからHugo等に移動する際に感じたことだが、移行はけっこう面倒だったからだ。移行ツールなどが存在するものの結局データの手直しが必要となった。幸い、自分のWordPressで作成したサイトの記事はすくなかったため長年運用しているようなサイトに比べれば比較的に楽に移行完了ができたので良かった。

可能性としては低いかもしれないが、プラットフォームなどのサーバーが突然落ちたり、記事そのものの移行ができないということはなくなる。これもローカル環境のマークダウンファイルの恩恵だ。例えば、画像の扱いを除けば、Hugoに書いた記事をZennにそのまま移行するなど、かなり楽だ。むしろサーバーに上げた画像のリンクから画像をレンダリングしている場合ならそのまま移行することさえできる。逆にインターネットに上げた記事をアーカイブしたい場合やObsidianのナレッジベースの一部に戻したい場合などもファイルを移動させるだけでよい。

今後、HugoやZenn、ひいてはObsidianが廃れる時が来たとしても自分が時間や労力を投じたコンテンツやナレッジベースの互換性や安全性を保証することができる。(もちろんバックアップなどは適宜必要)逆説的に、安心してそれらのツールやコミュニティを使い続けることが可能になる。

# OHZフロー
**ナレッジベースとしてのデジタルライブラリとメディアとしてのポストを融合させ、Obsidianの保管庫内部から様々なレイヤーにアウトプットとしての記事をパブリッシュできるようにする**。

｢Obsidian内のナレッジベース(個人の知識のメモ)→Hugoを利用したブログやサイトのアウトプット→Zennでの知見や本としてのアウトプット｣というような具体的なツールを利用したコンテンツのフローを暫定的に｢**OHZフロー(オーズフロー)**｣と名付ける。

## フローのイメージ

↓ 全体のフローのイメージ図

![全体のフロー](https://storage.googleapis.com/zenn-user-upload/uwh44tt8pyj5lsbef4gi31afkjv5)

↓ OHZフロー(ContentsFlow)の暫定的なイメージ図。(mermaid使ってstate-diagramで作成したイメージ図)

![OHZフロー](https://storage.googleapis.com/zenn-user-upload/pepap9majamzfzleeudiquyj8suz)

すべての流れを書くと汚くなるので省略している(イメージでつかんでほしい)。

ContentsFlowの中のPublish/Hugo/Zennはどれかなくともよいし、別のサービスでもよい。(人それぞれ)例えば、[Gitbook](https://www.gitbook.com)などでもCLIを入れればローカル環境で執筆できるようなので、このフローの構成要素は色々と置換することができる。

ちなみに、このフローに[Anki](https://apps.ankiweb.net/)も追加して学習フローも作成する予定。

## コンテンツレイヤー

### OHZフローでのコンテンツレイヤー
どのようにコンテンツを分けるか。多分個人によると思うので一例というかアイデア。コンテンツに色々なレベルを設けている(といってもZennはまだはじめたばかりなのでこれから...という感じになるが)。**洗練度やスピードなどのレベルが異なるように**コンテンツに対して各ツールと共にレイヤー分けを行う(現在はHugo/Zenn/Publish用にディレクトリを作成して運用している)。

- Obsidian内のまったく公開されないノート(PrivateNoes)
	- 記事のクリッピングや現在進行中のプロジェクト
	- プライベートな内容を含むノート
	- 勉強内容など、ここからAnkiにぶっ飛ばす
	- パーソナルナレッジベース内のノートをすべて管理するための構造用ノートなど
- [Obsidian Publish](https://obsidian.md/publish) : 現在実験中の[Digital Garden](https://www.technologyreview.com/2020/09/03/1007716/digital-gardens-let-you-cultivate-your-own-little-bit-of-the-internet/)(有料)
	- Obsidian内のアイデアノートやまとめなど(個人的なものであるが公開されても良いような中途半端なもの)
	- あまり人が見てもそこまで分からない
- Hugo : プラットフォームではない超個人的なサイト(無料)
	 - 個人的なブログ(なんでも書いてOK、好きなこと書き放題)
	 - ある程度まとまったことについて書いている(ある程度完成している)
- Zenn : デベロッパー用のナレッジベースプラットフォーム(無料)
	- コードや技術的な解決策などのコンテンツ(コンテンツのドメインが限定的: noteにあるようなどんなテーマでも書けるというわけではない)
	- 同様に技術的なことについてかなり詳しくなったことについて本としてまとめてアウトプット
	- プラットフォーム用に内容をある程度調整する必要がある

ここまで切り分けると面倒なのではというツッコミはあるかもしれない笑

最終的なコンテンツが｢Zennの本｣というような扱いになるかもしれない。

@[tweet](https://twitter.com/Mappletons/status/1279798483218767877)

Maggie氏の図のようのインターネットのコンテンツレイヤーのレベルを意識している。

- スピード
- コミュニケーション
- ドメイン
- プライバシー
- 洗練度

...etc


### Zenn内部のコンテンツレイヤー

Zenn内部でもレイヤーがある

- Scrap : 現在進行中の問題解決やアイデア等(obsidian内ではなくzennのwebページで編集している)
	- 編集可能なツイートのイメージ、かなり気軽に書くことができる(ショート・タームのフローかつストック)
	- 他の人とのコラボレーションや会話が可能
	- 最終的にはマークダウン出力をしてObsidianを介してZenn用のリポジトリから記事として再編成する
- Article : まとまった記事(一つのトピックで完結)
- Book : 記事の集合による特定のテーマについての最終完成品

ここらへんについてはまだしばらく試行錯誤していくことになると思う。

## ノートの転送
コマンドパレットからコマンド`Move to other folder`(ファイルを別のフォルダへ移動)を使えば、別のディレクトリへと移動(転送)できる。

![データ転送](https://storage.googleapis.com/zenn-user-upload/mrz3vne86075m1ikxrw3s54k0ox1)
*右上のoptionからも移動することが可能*

これと`[[wikilinks]]`と合わせれば、そのノートのネットワーク構造を維持したままの状態で自由にフォルダ移動させることができるようになる。(フォルダを移動してデータの階層構造が変化してもネットワークの構造は変わらない)

つまり、好きなときにあらゆるノートを簡単にHugoやZennの(内容は別として)記事に昇華させることができる。ナレッジベース内のネットワークはそのままにノートをZennやHugoのディレクトリに移動させるだけだ。

`[[wikilinks]]`ネットワークを利用することで、デジタルライブラリ内で、あらゆるノートを流動的に移動することが可能になる。

## PDFデータとして完成品をライブラリから切り離す

ObsidianではPDFとして出力機能が搭載されており、埋め込み機能やURLによるリンクで表示している画像、コードハイライトなどを含めてまとめて出力することが可能。

![PDFとして切り離す](https://storage.googleapis.com/zenn-user-upload/tiq7ri18ip5npno60ngfjs5gqg3d)
*↑この記事をPDFとして出力したもの*

これにより更新可能な記事としてではなく、ライブラリから特定の時点で切り離した完成品として出力することも可能になる。

(--> [[コンテンツの継承システム]] "2021-01-04" に分岐)

# ノート管理とネットワーク形成

## LYT : Linking Your Thinking 
具体的にObsidian内にてどう管理するのか？個人的に実践している方法を説明すると、Nick Miloの[LYTフレームワークにおけるMOC](https://publish.obsidian.md/lyt-kit/MOCs+Overview)を利用して各ノートへのリンクを一つのノートにリストとしてまとめることによってそれぞれのノートへのアクセスポイントを確保するようにしている(要するに各ノートへのルートを整備している)。

`[[wikilink]]`によるネットワーク形成を行うと、フォルダシステムによる階層的な現実のデータ構造だけでなく、ネットワーク構造が出現するのでどのようにネットワークを形成していくかが重要になってくる。

LYTはIdea Emergence(**アイデアの創発**)のための方法論でもある。(→[[創発とは]])

>創発（そうはつ、英語：emergence）とは、部分の性質の単純な総和にとどまらない性質が、全体として現れることである。局所的な複数の相互作用が複雑に組織化することで、個別の要素の振る舞いからは予測できないようなシステムが構成される。
\- [創発 - Wikipedia](https://ja.wikipedia.org/wiki/%E5%89%B5%E7%99%BA) より引用

LYTの具体的な話は以下のNickのtwitter上のスレッドを追ってほしい。

@[tweet](https://twitter.com/NickMilo/status/1317190821104373761)

(ちなみにLYTはObsidian公式もおすすめしている方法論、というかLYTに最適化するように開発しているところもある)
↓Nickが開いているLYTのワークショップのサイト
https://www.linkingyourthinking.com

ワークショップを受けなくても次のNickのPublishからかなり詳しく分かるので一読をおすすめする。
https://publish.obsidian.md/lyt-kit/MOCs+Overview

LYTは上で紹介したフローイメージのNetworkのセクションが相当する。

![](https://storage.googleapis.com/zenn-user-upload/lttbt3a5l7ggdy4hw90swv3dzv7q)
\- 出典 : [Paul Baran「On Distributed Communications: I Introduction to Distributed Communications Networks」Memorandum RM-3240-PR、August 1964 P2](https://www.rand.org/content/dam/rand/pubs/research_memoranda/2006/RM3420.pdf)

LYTにおけるネットワーク形成方法のイメージはネットワークの大きな構造として上の図のDecentralized(非中央集権型)を中心とするが、Dsitributed(分散型)のようにあらゆるノード同士で結ぶことも可能、というような構造をつくる。実際に作成する際にはどのようにでもネットワークを作ることが可能(どのノートからどのノートにリンクしてもよい)なので、かなり流動的な構造になる。(追加や削除はいつでもしてよい)時間経過で構造がすこしずつ変動してくる。

LYTはノートのスケーリングを考えた方法論なので、最初は何に役立つか分かりづらいが、ノート数が時間経過とともに膨大になった際に有効性が顕著になってくる。(例えばノート数が5000枚ぐらいの数になったときにどうやってコントロールすればよいかという方法論)

OHZフローについてのまとめなので、ここでは深くは説明しないが、簡単にLYTにでてくる概念を紹介する。

- Home Note : DomainごとのMOCの中心となるハブ。ナビゲーション用のハブとなる構造ノート。
	- ObsidianのVaultのアクセスポイントであり、ここを起点としてすべてのノートにアクセスすることができる。
	- 主に大きなMOCにアクセスするための`[[wikilink]]`によるリンクのインデックスリストとなる。
- MOC : Map of Contents
	- `[[wikilink]]`によるリンクのインデックスリスト
	- ここを起点に様々なノートにフォルダによる階層構造をとびこえてアクセスできる。
- Domain MOC : 大きな概念を包括するようなMOC。(これは自分用の概念でMOCを扱いやすくした)
- Topic MOC : 特定のテーマやトピックについてのMOC。様々なAtomicNoteへのリンクをリストアップする。(これは自分用の概念でMOCを扱いやすくした)
- Atomic Note : それぞれのノートであり、各マークダウンファイル。EvergreenNoteやZettelkasteにでてくる概念。

ネットワークの形成の仕方は人それぞれなので、上のLYTなどの方法は一つの参考例として考えてもらいたい。また、それぞれのノートから自由に別のノートにリンクを作成するので、実際に出来上がるネットワークはかなり自由で流動的なものとなる。

↓ Obsidian公式twitterで紹介しているグラフビューの例
@[tweet](https://twitter.com/obsdmd/status/1311079839726817282)

# やり方

まずはGitを導入して、Githubの自分のアカウントにHugoとZenn用の空リポジトリを作成し、ObsidianのVault内部に専用ディレクトリを作成しそこにリポジトリを両方ともクローンする。
Hugoの導入方法は[公式ドキュメント](https://gohugo.io/documentation/)を参照。
Zenn CLIの導入方法についても[公式ドキュメント](https://zenn.dev/zenn/articles/zenn-cli-guide)を参照。

## 実際のディレクトリ構造

```shell
  ObsidianVaultName
  └── Subvalut
      ├── Zenn-repo
      └── Hugo-repo
```
  
適当なディレクトリにリポジトリを用意するだけです。もしくは既存のHugo用のディレクトリ等をObsidianのVaultに移動させてくるなど。

双方ともターミナルにてそれぞれのコマンドを叩いてドキュメント等を用意する、もしくはObsidianでドキュメントを作成する。記事等が完成したら`git push`してリモートリポジトリに飛ばす。HugoならNetlifyなどのホスティングサービスを利用すれば自動的にビルドしサイト上にデプロイされる。


## 複数リポジトリでの管理
ObsidianのVautl(ライブラリ自体)をバックアップやバージョン管理するためにgitを使ってリポジトリ管理する場合には少し複雑になる。

HugoやZennのディレクトリを入れてあるフォルダ(自分の場合だとSubVaultという名前のフォルダ)をgitの追跡から外すように、ライブラリ直下にある`.gitignore`ファイルに以下を追記して、別のプライベートリポジトリで管理できるようにする。

```gitignore:gitの追跡を外す
SubVault/Hugo-repo/
SubVault/Zenn-repo/
```

- ObsidiaのVault全体 → Vautl専用のプライベートリポジトリで管理
- Vautl内のSubVautlに存在するHugoのディレクトリ → Hugo用プライベートリポジトリで管理
- Vautl内のSubVautlに存在するZennのディレクトリ → Zenn用プライベートリポジトリで管理

ちなみに[Obsidian Sync](https://obsidian.md/pricing)という有料の同期サービスが公式から出されているので、mobile版登場までgitで管理する予定だ(Syncでは公式のバージョン管理機能や特定フォルダやファイル拡張子の除外も可能)。

 ## obsidian-gitでpush自動化
 
 gitのセットアップが終わったら、obsidian-gitのプラグインを利用すると自動的に設定した時間ごとにremote repositoryに`git push`してバックアップをとることができるようになる。
 
https://github.com/denolehov/obsidian-git
 
 ![obsidian-git](https://storage.googleapis.com/zenn-user-upload/19hqif0wbhjvtly6abk5wi0db4zz)

もちろんコマンドを使うことで手動でもpushしてバックアップをとることができる。

![gitcommand](https://storage.googleapis.com/zenn-user-upload/w03uhcw603p8bmin1u7i39lqjq9e)

obsidianにおけるgitのセットアップについてはBryan Jenksの記事が詳しい。
https://medium.com/analytics-vidhya/how-i-put-my-mind-under-version-control-24caea37b8a5

## 利用上の注意点

### パスの更新
Obsidian内部にてディレクトリを移動させてしまうと設定によっては相対パスなどが更新されてしまうので、Obsidian内部ではHugoやZennのディレクトリの移動はしないようにする。

### 埋め込みの扱い

`![[]]`や`![]()`などによる記事の埋め込みなどは、ライブラリ内部でしか役にたたず、ZennやHugoなどでの記事では利用できないので、注意する必要がある。

### gitによるバージョン管理での複雑化
gitによるバージョン管理を行う際に、特定のコミットなどに戻した際に、ZennやHugoの記事に貼ってあるライブラリ内部の他のノート名の変更などがあればリンクや埋め込みなどが更新される前の状態にもどることになるので、再びリンクし直すことになる。

ZennやHugoでリンクを貼る場合には気をつけて運用する必要がある。

# 現時点での課題

## 課題1: 画像の扱い

Hugoでは相対パスによる画像表示が可能だが、ページバンドルにする必要がある。

特にHugoでは相対パスによる画像表示を利用することで、マークダウンアプリで記事を触る際に画像をプレビューできる。むしろ、相対パスを利用しないと画像を確認できない。参考 → [Relative path for markdown editor preview - support - HUGO](https://discourse.gohugo.io/t/relative-path-for-markdown-editor-preview/29403)

```shell
  content
  └── section1
      └── mypost
          ├── img1.png
          └── index.md
```

Hugoのディレクトリはこんな感じにする↑

Zennの場合にはGithubに画像を上げたり、Zennの管理ページから画像をアップロードする必要がある。現在、Zennでは相対パスによる画像表示をサポートしてないのでそこは改善してほしい。

## 課題2: フレーバーの違い

Obsidian/Hugo/Zennではマークダウンにそれぞれ独自のフレーバーがある。Obsidianでの再利用を考えるとなるべく独自フレーバーを使わないようにするなど気をつけなくてはならなくなる。(ここは運用によって裁量が決まってくるのでバランス良く考えたほうがよい)

例えば、HugoのショートコードやZennのコンテンツ埋め込み用のコードなど。逆にObsidianの`[[wikilink]]`はHugoやZennでのデプロイされたコンテンツでは認識できないのでリンクを切ることができる。

## 課題3: フォルダ除外
ZennやHugoを保管庫にいれるとnode modulesなどいらないものがグラフビューなどに表示されたりする。

![zennimage](https://storage.googleapis.com/zenn-user-upload/2hfx7cr6ki221sv129f3ejrme5he)

これを解決する一番簡単な方法は画像のようにグラフビューで除外用のコードを入れてあげることだ。また、フォルダが多くなるため｢ノートの転送｣で紹介した`Move file to another folder`(ファイルを別のフォルダに移動)コマンドでもフォルダ名を打つと若干遅くなる

公式のフォルダ除外機能の開発が待たれる。

## 課題4: OTAプラグイン併用
次の記事[Obsidian_to_Ankiの使い方 : ZettelkastenとSRSを組み合わせる](https://zenn.dev/estra/articles/integration-obsidian-and-anki)で紹介するObsidian_to_Ankiのプラグインとの併用でカスタムシンタックスを利用すると意図しないIDの挿入がされる。


# 思考収束の盲点
ここまでObsidianの凄さやOHZフローなどアウトプットの効率化の方法を紹介してきた。水をさすようで悪いが、気をつけてほしいことはObsidianやRoamなどのツールを使っても｢**アウトプットの難しさそのものが変わるわけではない**｣ということだ。｢思考収束の難しさ｣に関しては滝林さんの記事で語られている。

https://note.com/takibayashi/n/n3d1b8948aeea?magazine_key=m38cd2861c756

ツールはツール。思考を収束したりアウトプットを作成するのは結局個人の問題に帰属する。しかし、ナレッジベースから直接記事をアウトプットするという姿勢や方法(ナレッジベースからアウトプットへのルートが確実に整備された状態)は(アウトプットそのものの質を向上させるわけではないが)、アウトプットするための壁を低くしたり、より高い頻度で更新することでアウトプットの質が**結果として向上**する。こういうカラクリだと個人的には思う。

偉そうに言っているが、私も実際にこのフローをやってみてアウトプットには苦労している。結局noteだろうがzennだろうが、hugoだろうがどのプラットフォームでアウトプットしようが｢アウトプットは容易ではない｣ということが理解できた。ただ一言加えるなら、Wordpressでブログを書いていたときよりも断然｢**楽しい**｣。Obsidianは使っていて楽しい(Zennも楽しい)。これにつきる。そういった楽しさもアウトプットの頻度や質を結果としてたかめてくれるはずだ。

# その他の運用

## データとアプリケーション

vscodeとの受け渡し(データはそのままでmarkdownファイルを触るアプリケーションを変える)方法はvscodeの方に次のようなPKM用のプラグインをどれか入れてやる。

- [vscode-memo](https://github.com/svsool/vscode-memo)
- [dendron](https://github.com/dendronhq/dendron)
- [Foam](https://foambubble.github.io/foam/)

これらのプラグインを入れると`[[wikilink]]`形式のリンクを認識できるようになるのでObsidianのvault内のデータをvscodeで開いても同じように認識でき、vscodeでもリンクを作成、またプレビューなどを行うことができるようになる。つまりデータに対してvscodeとobsidianの能力を両方そのまま使うことができる。
ちなみに現在はvscode-memoを利用している。

![vscode-memo popup](https://storage.googleapis.com/zenn-user-upload/s0iyvm4l1af59xb51agsu2gbxvup)


[vim](https://vim-jp.org/)やvscodeやObsidian等、マークダウンファイルは必要に応じて触るアプリケーションを変えることができるのがよい。

![obsidianとvimとvscode](https://storage.googleapis.com/zenn-user-upload/wy767cr6ean33aqhfce61yrwn0wz)

↑ Obsidianとvimとvscodeで同時に開いた状態。

## Webページのクリッピング

Evernoteのようにクリッピングしたい場合には、Obsidianのモデレーターチームの一人であるdeath.auが開発した次のChromeアドオンを利用するとよい。

https://chrome.google.com/webstore/detail/markdownload-markdown-web/pcmpcfapbekmbjjkdalcgopdkipoggdi

ただし、クリッピングしすぎると[ライブラリの汚染(ノイズ増加)になる](https://forum.obsidian.md/t/my-pkm-story-youtube-6-week-workshop-long-read/5750)ので次の様のある程度ルールを設けて注意して使っている(とは言っても大抵は柔軟に活用している)。

- 本当に重要なものだけクリップする
- クリップする必要のないものはブラウザのブックマークで十分
- 一部分だけクリップし、ソースを必ず記述する

## YAMLフロントマター
Obsidian,Hugo,ZennのいずれもYAMLフロントマターを利用することができる。ObsidianのTemplateプラグインでそれぞれYAMLフロントマターのテンプレートを作成しておくと簡単に作成することができる。

- [Hugoのフロントマター](https://gohugo.io/content-management/front-matter/)
- [Zennのフロントマター](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Obsidianのフロントマター](https://publish.obsidian.md/help/Advanced+topics/YAML+front+matter)

```yaml:Hugoのyaml例
categories:
- Development
- VIM
date: "2012-04-06"
description: spf13-vim is a cross platform distribution of vim plugins and resources
  for Vim.
slug: spf13-vim-3-0-release-and-new-website
tags:
- .vimrc
- plugins
- spf13-vim
- vim
title: spf13-vim 3.0 release and new website
darft: ture
alias: [hugoのyaml]
```

```yaml:Zennのyaml例
title: "Zenn & Hugo in Obsidian : OHZフローによるナレッジベースとアウトプットコンテンツの完全統括"
emoji: "💎"
type: "idea"
topics: ["zenn", "obsidian", "hugo", "PKM", "マークダウン"]
published: true
alias: [OHZフロー, OHZflow, zennのyaml]
```

```yaml:Obsidianのyaml例
date: 2020-12-07
tags: [obsidian, zenn, hugo]
alias: [ohzフロー, obsidianのyaml]
```

aliasはZennやHugoでも記述しても特に問題がなかったのでどのフロントマターでも利用できる。


# 記事変更ログ
- [[2020-12-11]] : 
	- 目次｢はじめに｣追加
	- ｢得られるメリット｣の内容をrevise
- [[2020-12-20]] :
	- ｢注意点｣の項目を追加
	- ｢利用上の注意点｣の内容を追加
	- ｢PDF出力｣の項目を追加
	- ｢gitによる複数リポジトリでの管理｣の項目を追加
- [[2020-12-23]] : 
	- 冒頭文追加
	- wikilinkの修正
	- タイポの修正
	- フロントマターとエイリアスについて追加
- [[2020-12-26]] :
	- ｢ノートの転送｣の項目を追加
	- 画像の差し替え
- [[2021-01-03]]
	- ｢思考収束の盲点｣の項目を追加
	- ｢課題3: フォルダ除外｣の項目を追加

---
# Meta
links: [[090 Blog MOC ⌨️|Blog MOC]]
URL: [Zenn & Hugo in Obsidian : OHZフローによるナレッジベースとアウトプットコンテンツの完全統括](https://zenn.dev/estra/articles/ohzflow-zenn-hugo-obsidian)

