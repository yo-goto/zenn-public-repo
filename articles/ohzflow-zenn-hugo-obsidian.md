---
title: "Zenn & Hugo in Obsidian : OHZフローによるナレッジベースとアウトプットコンテンツの完全統括"
emoji: "💎"
type: "idea"
topics: ["zenn", "obsidian", "hugo", "PKM", "マークダウン"]
published: true
alias: [OHZフロー, OHZflow]
---

# はじめに

ZennやHugoをObsidianで利用して運用するためのアイデアとその方法をまとめました。[スクラップ上でまとめたもの](https://zenn.dev/estra/scraps/644e7c229c3dcb)を記事化してます(追加の内容あり)。まだ実験段階のアイデアなので変更等かなりあると思います。

日本ではまだ広く知られていないPKM周りの概念や新たなツールの紹介も含むの分かりづらいところや長ったらしい部分もありますが興味を持っていただければ幸いです。

https://obsidian.md

![graphview](https://obsidian.md/images/screenshot.png)

## 対象読者
- メモ魔の人
- EvernoteやNotion等のメモアプリを使ったことがある人
- ナレッジベースに興味がある人
- 新しいツールが好きな人
- プレーンテキストが好きな人
- マークダウンが好きな人
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

すべて**ローカル環境**と**マークダウン**を利用してコンテンツ作成が可能!!
ただしマークダウンのフレーバーがそれぞれ若干異なる部分があるのでそこは注意。

# PKMとは

そもそもPKMとは何か?

**Personal Knowledge Management** (個人知識管理)の略。PKMのツールとしてPKB(Perosnal Knowledge Base)がある。

[Evernote](https://evernote.com/intl/jp)や[Notion](https://www.notion.so/)などの電子メモツールなどが相当するが、[Scrapbox](https://scrapbox.io/product)や[Roam Research](https://roamresearch.com/)、Obsidianといった[networked thought](https://nesslabs.com/networked-thinking)(ネットワーク化された思考)を補助し｢[第二の脳をつくる](https://www.buildingasecondbrain.com/)｣ためのアプリケーションツールが最近注目されている。wikipediaのようなリンクをつかったハイパーテキストによるwikiを個人で作り上げることができる。

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
	- [[wikilink]]によって様々なノートとアウトプットされた記事をリンクすることでローカル環境においてナレッジネットワークを作成することができる。
	- グラフビューによってネットワークの可視化と効率的なネットワークのナビゲーションを行うことが可能になる。
3. **埋め込み参照**によって知識の様々なレベルでの**再利用性**が高まり、デジタルライブラリ内で**効率的に循環**させることが可能になる。
	- Obsidianの埋め込み機能`![[]]`や`![]()`を使えば学習効率や問題解決のスピードがあがる。
	- **ナレッジベースから高速で自分のブログの投稿や記事といったアウトプットコンテンツにアクセスし参照することができる**。
	- 逆にすべての記事をナレッジベースの一部として扱うことができるので、再びアイデアを発見できるかもしれない。
4. 一元化された環境内部で**アウトプットの高速化と頻度向上が可能**となる。
	- アイデアにすらなっていないような思考状態のものから、アイデア、そこから洗練され完成した知識や知見、さらには本まで様々なレベルのものを一元化させてナレッジベース内で完結することで、スムーズにアウトプットできるようになる。

上に列挙した4つのメリットはそれぞれ相互に関連している。

## 1. 情報の重複やズレを少なくできる

『No more Copy & Paste』

Wordpressやnoteなどで記事をつくったことがある人はわかるかもしれないが、自分の好きなエディタで記事を書いても、記事をそのまま公開することはできない。**エディタからいちいち専用のwebページにコピペし直すという工程**が存在する。

何が問題かというと、記事に変更や更新があったときにそのwebページから変更や更新を行う必要があるということだ。それによって、もともとのエディタで書いていた記事と実際に公開されている記事の間で情報のズレが発生することになる。もちろん、エディタで直してから再びコピペし直せばよいのではないかという話はあるが、けっこう面倒くさい。というのも見た目をリアルタイムで確認できない場合があるからだ。変更や更新をしてプレビューしたいことも多いので結局エディタでやらずに直接編集するということになってしまう。これによって公開されている記事の方がローカル環境より最新という状態になってしまう。これではナレッジベース内で再利用することに壁ができてしまう。**自分のナレッジベース内の情報こそが最新状態であるという状態**が望ましい。

ところが、HugoやZennなどであればcliを使ってコマンドを叩けばブラウザからローカルホストでリアルタイムのプレビューを見ることができる。これにより、プレビューを見るためにwebページで編集し直して、エディタ元のオリジナルの記事とズレてしまうということをなくすことができる。**常にローカルとPublishされた記事の情報が同期されている、またはローカル環境の方が最新状態ということにできる**。

## 2. リンクによるナレッジネットワークの形成


### レイヤーを超えたリンキング

各アウトプットコンテンツにリンクリストを作成したり、`[[wikilink]]`を埋め込めばそのままObsidianのネットワークの構成要素として利用することが可能。インターネット上で異なるレイヤー(個人ブログとプラットフォーム)に属しているコンテンツがそのレイヤーを超えてローカル環境でネットワーク化される。

![local graph](https://storage.googleapis.com/zenn-user-upload/iabsguhmm64rdzej5pj4x4tpt79z)

この記事の下にリンクリストを設けた。このリンクによるローカルネットワークが右のグラフビューに現れている。

### ナレッジベースからアウトプットコンテンツへ高速アクセス

![sidebarMOC](https://storage.googleapis.com/zenn-user-upload/cai3rax8z8hhyjnnbhuuzulwowmo)

後で説明するMOCを作成すると(要するにリンクのリスト)高速で自分のアウトプットコンテンツにローカルでアクセスできる。

サイドバーにそのリンクリストを置いておけば、一瞬でアクセスできる。その上、プレビューのプラグインでカーソルをあわせると内容をポップアップで確認できる。

@[tweet](https://twitter.com/ankiyorihajimey/status/1324387918484402177)

また｢検索結果をコピー｣する機能を利用すれば、リンクリストを高速で作成できる。

## 3. 情報や知識の再利用による循環

Obsidianの参照機能には三段階ある(ページレベル･段落レベル･ブロックレベル)。あらゆる自分のアウトプットとしての記事やブログのポストから参照することができる。

![](https://storage.googleapis.com/zenn-user-upload/r7khrd8w7mgdd1tw88x1ls8i1vzj)

↑左のノート(Hugoのサイト記事)の段落を[トランスクルージョン](https://ja.wikipedia.org/wiki/%E3%83%88%E3%83%A9%E3%83%B3%E3%82%B9%E3%82%AF%E3%83%AB%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3)で右のノートに埋め込んでいる。

## 4. アウトプットの高速化と頻度向上

今まで述べてきた1~3のメリットからあらゆる知識や情報が格納されたナレッジベースかつアウトプットコンテンツの生成環境の両方を兼ねることができるが、これのおかげで[Evergreen Note](https://publish.obsidian.md/andymatuschak/Evergreen+notes)を高いクオリティで実現できる。すべての公開されている記事がローカル環境において最新であり、インターネット上で切り分けられているあらゆるコンテンツがローカルではネットワークの構成要素として機能する。

何か記事を更新しようと思えば、ローカル環境でObsidianを開けば、そのまますぐに書ける。

記事ではなく、未発表のアイデアにしたいならナレッジベース内のノートにかけばよい。記事と関連するならリンクすればよい。なにせローカル環境が一番最新であり、すべての情報が存在している。

**『思考の断片』から『アイデアの粒』、記事の『ドラフト』から『完成品の本』まで、あらゆるアウトプット記事とナレッジベース内のすべてのノートが並列に並んでおり、同時にネットワーク化されている**。これによりアイデアや知識の状態をスムーズにあげていくことが可能になるはずだ。

![すべての記事とナレッジベースが並列](https://storage.googleapis.com/zenn-user-upload/fs1yok4l3nu39u9vpdhtayrkr0vt)

左がHugoの記事、中央がナレッジベース内ノート、右がZennのこの記事。

### Evergreen Notes
これによって、すべてのノートと記事そのものを[Evergreen Note](https://publish.obsidian.md/andymatuschak/Evergreen+notes)化することが可能になる。

@[tweet](https://twitter.com/Mappletons/status/1277262380775415809)

Evergreen Notesとは、Andy Matuschak氏が考案したノートテーキングの概念であり方法論。

一過性のノートによる知識やアイデアの損失を防ぐために、あらゆるノートを蓄積し、さらにそのノートに再訪して情報を追加･修正し、時間経過と共にそのアイデアや知識を成長させ、さらに有機的に別のアイデアや知識に接続させようというような考え。[いくつかの原則](https://publish.obsidian.md/andymatuschak/Evergreen+notes)がある。

上で述べてきたようにナレッジベースからアウトプットへのパスをObsidian内の環境において整備することによってアイデアとノートの成長を促しEvergreenにすることができる。

# OHZフロー

｢Obsidian内のナレッジベース(個人の知識のメモ)→Hugoを利用したブログやサイトのアウトプット→Zennでの知見や本としてのアウトプット｣というようなコンテンツのフローを暫定的に｢**OHZフロー(オーズフロー)**｣と名付ける。

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

--> [[when-outputs-finish|アウトプットはどこで完成するのか]]


# ノート管理とネットワーク形成

## LYT : Linking Your Thinking 

具体的にObsidian内にてどう管理するのか？個人的に実践している方法を説明すると、Nick Miloの[LYTフレームワークにおけるMOC](https://publish.obsidian.md/lyt-kit/MOCs+Overview)を利用して各ノートへのリンクを一つのノートにリストとしてまとめることによってそれぞれのノートへのアクセスポイントを確保するようにしている。(要するに各ノートへのルートを整備してあげてる)

[[wikilink]]によるネットワーク形成を行うと、フォルダシステムによる階層的な現実のデータ構造だけでなく、ネットワーク構造が出現するのでどのようにネットワークを形成していくかが重要になってくる。

LYTはIdea Emergence(アイデアの出現)のための方法論でもある。具体的には下のNickのtwitter上のスレッドを追ってほしい。

@[tweet](https://twitter.com/NickMilo/status/1317190821104373761)

LYTは上で紹介したフローイメージのNetworkのセクションが相当する。

![](https://storage.googleapis.com/zenn-user-upload/lttbt3a5l7ggdy4hw90swv3dzv7q)
\- 出典 : [Paul Baran「On Distributed Communications: I Introduction to Distributed Communications Networks」Memorandum RM-3240-PR、August 1964 P2](https://www.rand.org/content/dam/rand/pubs/research_memoranda/2006/RM3420.pdf)

LYTにおけるネットワーク形成方法のイメージはネットワークの大きな構造として上の図のDecentralized(非中央集権型)を中心とするが、Dsitributed(分散型)のようにあらゆるノード同士で結ぶことも可能、というような構造をつくる。実際に作成する際にはどのようにでもネットワークを作ることが可能(どのノートからどのノートにリンクしてもよい)なので、かなり流動的な構造になる。(追加や削除はいつでもしてよい)時間経過で構造がすこしずつ変動してくる。

LYTはノートのスケーリングを考えた方法論なので、最初は何に役立つか分かりづらいが、ノート数が時間経過とともに膨大になった際に有効性が顕著になってくる。

OHZフローについてのまとめなので、ここでは深くは説明しないが、簡単にLYTにでてくる概念を紹介する。

- Home Note : DomainごとのMOCの中心となるハブ。ナビゲーション用のハブとなる構造ノート。
	- ObsidianのVaultのアクセスポイントであり、ここを起点としてすべてのノートにアクセスすることができる。
	- 主に大きなMOCにアクセスするための[[wikilink]]によるリンクのインデックスリストとなる。
- MOC : Map of Contents
	- [[wikilink]]によるリンクのインデックスリスト
	- ここを起点に様々なノートにフォルダによる階層構造をとびこえてアクセスできる。
- DomainMOC : 大きな概念を包括するようなMOC。(これは自分用の概念でMOCを扱いやすくした)
- TopicMOC : 特定のテーマやトピックについてのMOC。様々なAtomicNoteへのリンクをリストアップする。(これは自分用の概念でMOCを扱いやすくした)
- AtomicNote : それぞれのノートであり、各マークダウンファイル。EvergreenNoteやZettelkasteにでてくる概念。

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

## 利用上の注意点
Obsidian内部にてディレクトリを移動させてしまうと設定によっては相対パスなどが更新されてしまうので、Obsidian内部ではHugoやZennのディレクトリの移動はしないようにする。

# 現時点での課題

## 課題 1: 画像の扱い

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

## 課題2 : マークダウンのフレーバーの違い

Obsidian/Hugo/Zennではマークダウンにそれぞれ独自のフレーバーがある。Obsidianでの再利用を考えるとなるべく独自フレーバーを使わないようにするなど気をつけなくてはならなくなる。(ここは運用によって裁量が決まってくるのでバランス良く考えたほうがよい)

例えば、HugoのショートコードやZennのコンテンツ埋め込み用のコードなど。逆にObsidianの[[wikilink]]はHugoやZennでのデプロイされたコンテンツでは認識できないのでリンクを切ることができる。


# その他の運用

## データとアプリケーション

vscodeとの受け渡し(データはそのままでmarkdownファイルを触るアプリケーションを変える)方法はvscodeの方に次のようなPKM用のプラグインをどれか入れてやる。

- [vscode-memo](https://github.com/svsool/vscode-memo)
- [dendron](https://github.com/dendronhq/dendron)
- [Foam](https://foambubble.github.io/foam/)

これらのプラグインを入れると[[wikilink]]形式のリンクを認識できるようになるのでObsidianのvault内のデータをvscodeで開いても同じように認識でき、vscodeでもリンクを作成、またプレビューなどを行うことができるようになる。つまりデータに対してvscodeとobsidianの能力を両方そのまま使うことができる。
ちなみに現在はvscode-memoを利用している。

![vscode-memo popup](https://storage.googleapis.com/zenn-user-upload/s0iyvm4l1af59xb51agsu2gbxvup)


[vim](https://vim-jp.org/)やvscodeやObsidian等、マークダウンファイルは必要に応じて触るアプリケーションを変えることができるのがよい。

![obsidianとvimとvscode](https://storage.googleapis.com/zenn-user-upload/wy767cr6ean33aqhfce61yrwn0wz)

↑ Obsidianとvimとvscodeで同時に開いた状態。

## Webページのクリッピング

Evernoteのようにクリッピングしたい場合には、次のChromeアドオンを利用するとよい。

https://chrome.google.com/webstore/detail/markdownload-markdown-web/pcmpcfapbekmbjjkdalcgopdkipoggdi

ただし、クリッピングしすぎるとライブラリの汚染(ノイズ増加)になるので注意して使っている。

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
```


```yaml:Zennのyaml例
title: "" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
```


```yaml:Obsidianのyaml例
date: 2020-12-07
alias: []
tags: []
```


# 記事変更ログ
[[2020-12-11]] : 
- 目次｢はじめに｣追加
- ｢得られるメリット｣の内容をrevise

---

Linklist 
- [[networked thought]]
- [[Obsidianの名前の由来]]
- [[Hugo]]
- [[Zenn CLIで記事･本を管理する方法|zenn cli]]
- [[090 Blog MOC ⌨️]]
- [[OHZフローのアイデア]]


ここにあるリンクはObsidian上では認識されます。