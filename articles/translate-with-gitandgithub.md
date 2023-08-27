---
title: "プルリクエストによる翻訳作業のやり方 - Fork and pull model"
published: true
cssClasses: zenn
emoji: "👾"
type: "tech"
topics: [git, github, 英語, 翻訳, Obsidian]
date: 2021-03-28
modified: 2023-08-26
url: "https://zenn.dev/estra/articles/translate-with-gitandgithub"
tags: type/zenn, git/GitHub, Zenn, 翻訳, obsidian
aliases:
  - git翻訳のまとめ記事
  - 記事_プルリクエストによる翻訳作業のやり方
---

## はじめに

:::details ChangeLog
- 2023-08-16
  - "Fork and pull model" についての記述を追加
  - mermaid ダイアグラムの修正と追加
  - 内容のリライト
  - GitHub で完結するモデルについての記述を追加
  - GitHub flow についての記述を追加
:::

[Obsidian](https://obsidian.md) というソフトウェアの UI とヘルプドキュメントの翻訳と保守を有志で行っています。翻訳作業を通してプルリクエストのやり方を学び、良い機会だと思ったので git と GitHub の使い方からプルリクエストによって共同で翻訳作業をしていく方法を紹介していこうと思います。

:::message
今回の記事で紹介するワークフローでは GitHub での共同開発モデルの一つである "**Fork and pull model**" を使いますが、このワークフローについては詳細に解説された記事が実はあまり存在しません。この記事では Obsidian の翻訳だけでなく、Fork and pull model を使った一般的な開発や翻訳のワークフローについての方法を解説します。
:::

### 翻訳から始めるOSS

初学者の観点から git の概念やつまずいたところやコマンドの使い方、プルリクエストまでの流れなどをまとめています。次のような記事もあるので、翻訳作業を通して git の使い方をマスターするのは結構おすすめだと思います。

https://qiita.com/yhay81/items/3773ab2e001a9e39ccc8

次のような OSS(オープンソースコミュニティ) へのプルリクエストに関する記事なども参考になりますが、翻訳での継続的なプルリクエスト (2 回目、3 回目~) についての全体の流れがわかる記事はあまり見たことがありませんでしたので主にその点について書いてみました。

https://qiita.com/y-vectorfield/items/b955617712f3b66359f2

ドキュメントの翻訳作業をしてみたいが、やり方がよく分からないという方、Git 初心者に役立てばよいなと思います。

また、UI ではなくヘルプドキュメントの翻訳作業を念頭において書いています。というのもヘルプドキュメントは大量のファイルを扱うので、GitHub 上ではなくローカル環境でしっかりと作業する必要があります。ローカルでの翻訳作業からプルリクエスト作成を通して Git や GitHub の基本的な使い方がマスターできます。翻訳作業そのものに関しては素人なので注意してください。

実際に翻訳されたヘルプドキュメントが Obsidian の公式サイトで公開されています。

https://publish.obsidian.md/help-ja/

上のドキュメントで表現の向上 (こうしたほうがよい) や誤訳、タイポなどを見つけた場合には公式リポジトリや Fork したリポジトリでもいいので issue かプルリクエストを作成してください。

https://github.com/obsidianmd/obsidian-docs/

### 最低限必要な知識や事前に必要なもの

作業を始めるのに次の最低限必要な知識やアプリケーションなどは準備するようにしてください。

- CLI の使い方の基本
- git のインストール
- GitHub のアカウント作成
- VScode などのテキストエディタ

### 主なソース

- [エンジニアのためのGitの教科書](https://amzn.to/3dcc9h5)
  - 薄くてわかりやすいのでおすすめです。わからなくなったら読むといいです。
- [GitHub で Fork してから Pull Request をするまでの流れ](http://kik.xii.jp/archives/179)
  - 最初のプルリクエストの流れが分かりやい記事ですが、2 回目以降については言及がなかったので補足します。また `rebase` や `push -f` など初心者が使わない方がよいです。
- [初めてのPull Request(プルリクエスト)](https://kaworu.jpn.org/kaworu/2017-10-19-1.php#2017-10-19-1-7a386224b42e28c840ef6ce67c51ca62)
  - OSS でのプルリクエストについての記事で、図はないですが細かく、分かりやすいです。

## 基本的概念

### git と GitHub

まずは git と GitHub の違いを把握しておきましょう。

そもそも git とは分散バージョン管理のソフトウェアであり、これを使うことでソフトウェア開発のバージョン管理を行えます。git を使うことで一人でもバージョン管理を行いながらソフトウェア開発ができますが、インターネット上で複数人で共同開発を行うためには git で管理しているデータをホスティングして、そこからデータを引っ張ってきたり(fetch)、逆にアップロード(push)する必要があります。

GitHub は git を使って共同で開発を行うことができるコラボレーションプラットフォームのサービスです。git でバージョン管理されたソフトウェアのソースコードが GitHub というプラットフォーム上にホスティングされ、このプラットフォームからソースコードをダウンロードしたりアップロードすることで共同で開発を行います。

GitHub はただのホスティングサービスなので、実際には他のサービスでも同じようなことをが行なえます。代表的な別サービスとしては [GitLab](https://about.gitlab.com/ja-jp/) などがあげられます。

`git push` や `git fetch` などの git のコマンド操作はホスティングサービス上にあるリモートのリポジトリ(ソースコード)に対してネットワーク越しにデータのアップロードやダウンロードをするための操作です。このコマンド操作を通じてインターネットで共同開発ができる、というわけです。

:::message alert
git と GitHub は別物なわけですから、各種の操作が git のコマンド操作なのか、GitHub での操作なのかを区別することが重要です。例えばリポジトリの複製をおこなう fork や clone といった操作がありますが、fork は GitHub での操作であるのに対して、clone は git のコマンド操作である点に注意してください。

また、git ではホスティングサービスを GitHub に限定しているわけではないことから、公式ドキュメントでは GitHub を使った操作ではなくなんらかのサーバーとして抽象化された説明などになっていることにも注意が必要です。
:::

### リポジトリとプルリクエスト

git ではバージョン管理のために「リポジトリ」というものを作成します。リポジトリとは大雑把に捉えるなら git で管理しているプロジェクトのフォルダです。git がインストール済みなら `git init` というコマンドをコマンドラインから実行することで現在のフォルダをリポジトリとして初期化することができます。

```sh
# 現在のフォルダをリポジトリとして初期化する
git init
```

git で初期化されたフォルダのことを一般にはリポジトリとして呼びます。

git ではデータを小さなファイルシステムとソースコードのスナップショット(ある時点での完全な状態)の集合として考えます。git を使うとそのリポジトリ内部 (フォルダ) でさまざまな変更を行い特定の時点の状態そのものスナップショットとして保存管理できます。このスナップショットをコミットと呼びます。このコミットをどんどん繋げていくことでブランチが形成されます。

`git init` で git 初期化したフォルダをリポジトリとして呼びましたが、インターネットごしに共同開発するには開発者たちはソースコードを共有している必要があります。GitHub ではこのリポジトリをホスティングしており、開発者たちはホスティングされたリポジトリを自分のローカル環境に複製(clone)することで同じリポジトリで作業を行うことができます。つまり、リポジトリとは自分のパソコン上だけでなく、サーバーにもあったり、他の開発者のパソコン上にもあるという状態が一般的になります。

分散バージョン管理システムである git では、このようにリポジトリがいたるところに存在しており、リポジトリへの変更をホスティングサービスを通じて相互に反映しあうことでソースコードの更新を行っていきます。

このとき、GitHub のプラットフォーム上でソースコードの更新のやり取りを分かりやすく行うための仕組みがプルリクエスト(Pull request)です。
プルリクエストは git の機能ではなく、GitHub のサービスの機能であることに注意してください。プルリクエストは自身のローカルにあるリポジトリ内にて行った変更を、GitHub 上 (ウェブ上) にあるリポジトリに対してプルリクエストを作成することでその変更を統合 (`merge`) するように申請する仕組みです。

このようにプルリクエストを通じて開発を行うのが GitHub での一般的な共同開発の方法です。OSS でのソフトウェアの翻訳作業は開発作業と同じく、プルリクエストを通じて行われるケースがよくあり、今回の記事はその方法を通じて GitHub での共同開発を学ぶというのが目的です。

### GitHub flow

GitHub で開発や翻訳を行う際のワークフローではいわゆる **GitHub flow** と呼ばれるフローが利用されます。

https://docs.github.com/en/get-started/quickstart/github-flow

GitHub flow は簡単に言えば GitHub 上でプルリクエストを使った共同開発のワークフローです。オープンソースでの開発や翻訳はこのワークフローを基本としますが、現実的にはこのワークフローをどのようなリポジトリのモデルを行うかが問題になります。

より詳細なワークフローは次に紹介する共同開発モデルを使っていきます。

### Fork and pull model

GitHub を利用した共同開発モデル(collaborative development model)には以下の２種類があります。

- **Fork and pull model** (フォーク&プルモデル)
- **Shared repository model** (共有リポジトリモデル)

これらのモデルは共同開発においてリポジトリをどのように扱うかを問題にした分類です。パブリックリポジトリでのオープンソースのプロジェクトでは「**Fork and pull model**」という複数のリポジトリとプルリクエストを使用して開発や翻訳を行うモデルがよく使用されます。逆にプライベートリポジトリでのクローズドソースのプロジェクトでは「**Shared repository model**」がよく使用されます。

これらのモデルについては GitHub 公式ドキュメントの以下のページで概要が解説されていますが、図なども存在せず、解説自体もかなり分かりづらく分散的になっています。

https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/getting-started/about-collaborative-development-models

俯瞰的に理解するためには、サードパーティで書かれた記事で情報を補うことになります。このモデルについては以下の記事などで比較的に分かりやすい図や説明が記載されているので、この記事の説明を補う形で参考にしてみるとよいと思います。

- [Github で Fork してから Pull Request をするまでの流れ](http://kik.xii.jp/archives/179)
- [The Git Fork-Branch-Pull Workflow](https://www.tomasbeuzen.com/post/git-fork-branch-pull/)
- [Fork-Based Contribution Model](https://jenkinsci.github.io/templating-engine-plugin/2.5/contributing/fork-based/)
- [Using Gitflow with the GitHub Fork & Pull Model](https://www.dalescott.net/using-gitflow-with-githubs-fork-pull-model/)

Fork and pull model を簡単に説明すると、fork (リポジトリの複製) と pull request (変更のマージリクエスト) を使って安全に他者のリポジトリにコントリビュートできるモデルです。なお、git のドキュメントではこのモデル(ワーカーフロー)は Integration-Manager Workflow と呼ばれています。

https://git-scm.com/book/en/v2/Distributed-Git-Distributed-Workflows#wfdiag_b

より具体的には、公式のリポジトリをまず fork して、自分のアカウントに複製されたリポジトリを作成します。その複製した fork リポジトリに自身の変更を反映させていき、一定の変更の塊を元々の公式のリポジトリへと反映させるプルリクエスト(PR)を作成します。このプルリクエストが元々のリポジトリの管理者によってレビューされ、承認、つまりマージされることで自身の行った変更が公式のリポジトリへと実際に反映されます。

簡単なパッチ修正などは実は GitHub や github.dev だけで済ませることが可能です。この際のワークフローではローカルリポジトリを介さない以下のようなモデルとなります。筆者は MDN の翻訳なども最近行っていますが、日本語などの間違いなどを見つけた場合には実際に以下のワークフローで修正して PR を作成します。

```mermaid
---
title: GitHubで完結するパッチ修正のモデル
---
graph TD
  subgraph GitHub
    direction LR
    subgraph Upstream
      A[master]
    end
    subgraph Origin
      B[master]
      E[topic]
    end
  end
  Upstream -->|fork| Origin
  E -->|Pull Request| A
  A -->|sync| B
  B -->|checkout| E
  style GitHub fill:#eee, stroke:#000
```

継続的な翻訳や開発などでじっくり作業したかったり、npm script などで linter や formatter なども走らせたい場合にはローカル環境にリポジトリを clone して作業を行います。

:::message alert
clone と fork は両者ともに「リポジトリの複製」を行いますが、以下のように異なる概念なので使い分けに注意してください。

用語 | 意味
--|--
fork | GitHub 上の他者のリモートリポジトリを自分のアカウントで別のリモートリポジトリとして複製する
clone | GitHub 上のリモートリポジトリをローカル環境に複製する
:::

この際のワークフローでは、上記のモデルに対して更にローカル作業のフローを追加します。ブランチまで含めて図にすると以下のような感じでかなり複雑です。簡単なパッチ修正を除いて、オープンソースの開発や翻訳についてローカルでガッツリ作業を行う際にはこのモデルを使った以下のようなフローをまず習得する必要があります。

```mermaid
---
title: Fork and pull model (通常バージョン)
---
graph RL
  Origin -->|clone| Local
  Upstream -->|fork| Origin
  subgraph GitHub
	subgraph Upstream
	A[master]
	end
	subgraph Origin
	B[master]
	E[topic]
	end
  end
	subgraph Local
	C[master]
	D[topic]
  C -->|checkout| D
	end
  A -->|pull| C
  C -->|push| B
  E -->|Pull Request| A
  D -->|push| E
  style GitHub fill:#eee, stroke:#000
```

図に出てくる `fork` や `clone` などの操作はリポジトリレベルでの操作ですが、`push` や `pull` などの操作はブランチレベルの操作であることに気をつけてください。

なおこのワークフローについて、2021 年頃に追加された GitHub 上での Fork リポジトリの同期機能(**Sync fork**)を使うともう少し分かりやすいフローになります。

https://github.blog/changelog/2021-05-06-sync-an-out-of-date-branch-of-a-fork-from-the-web/

```mermaid
---
title: Fork and pull model (sync forkバージョン)
---
graph RL
  Origin -->|clone| Local
  Upstream -->|fork| Origin
  subgraph GitHub
	subgraph Upstream
	A[master]
	end
	subgraph Origin
	B[master]
	E[topic]
	end
  end
	subgraph Local
	C[master]
	D[topic]
  C -->|checkout| D
	end
  A -->|sync| B
  E -->|Pull Request| A
  D -->|push| E
  B -->|pull| C
  style GitHub fill:#eee, stroke:#000
```

:::message alert
２つのバージョンの違いは単に origin リポジトリの master ブランチの更新をどのようにするかという方法の違いにすぎないのでワークフローに本質的な違いはありません。この記事では伝統的な通常バージョンのフローを解説します。

また、図の "pull" の部分はやっていることは "fetch & merge" であり、"topic" ブランチは任意の作業ブランチのことを示しているので、適宜読み替えてください。
:::

このモデルが利用されているプロジェクトで代表的なのは MDN のドキュメントやその翻訳プロジェクトなどです。

https://github.com/mdn/content

https://github.com/mdn/translated-content

このモデルでのやり方を覚えれば上記のような翻訳やオープンソースの開発プロジェクトなどに参加できるようになりますのでぜひとも覚えておくことを推奨します。

### 必要な３つのリポジトリ

Fork and pull model において、プルリクエストを作成してマージ (変更の統合) を行うまでの流れには、**合計３つのリポジトリを使用します**。この３つというのがなぜ必要なのかが最初分かりづらかったので注意してください。

必要なリポジトリとして、1 つ目は公式のリポジトリ (オリジナルのリポジトリ) で、これを **upstream** (上流) という名前で便宜的に識別します。2 つ目は GitHub の自分のアカウント上に upstream の複製リポジトリを作成します。この行為を fork すると呼びます。この fork によって作成されたリポジトリを **origin** として識別します。origin や upstream はリポジトリの名前として考えてください。この origin と upstream はユーザーにとってはオンライン上に存在するので「**リモートリポジトリ**」と呼びます。

3 つ目は、自分の PC 上での作業のために origin リポジトリから更にローカル環境に複製リポジトリを作成 (clone) します。clone による複製と fork による複製の違いに注意してください。基本的には、この複製されたリポジトリで編集作業を行っていきます。ローカル環境上にあるリポジトリなので先程のリモートリポジトリに対して「**ローカルリポジトリ**」と呼びます。このリポジトリを origin や upstream と同様に **local** という名前で識別します。

これらの３つのリポジトリを使って以下の図のワークフローに従い、プルリクエストを通じて非同期的な共同編集を行います。

```mermaid
---
title: Fork and pull model (通常バージョン)
---
graph RL
  Origin -->|clone| Local
  Upstream -->|fork| Origin
  subgraph GitHub
	subgraph Upstream
	A[master]
	end
	subgraph Origin
	B[master]
	E[topic]
	end
  end
	subgraph Local
	C[master]
	D[topic]
  C -->|checkout| D
	end
  A -->|pull| C
  C -->|push| B
  E -->|Pull Request| A
  D -->|push| E
  style GitHub fill:#eee, stroke:#000
```

なお、upstream は自分が管理するリポジトリではなく、他者が管理するオリジナルのリポジトリです。origin は upstream から複製された自分が管理するリポジトリなので自由に編集できます。しかし、実際に触るのは origin ではなく「**ローカル上のリポジトリである local**」です。このあたりが紛らわしいのですが、オンライン上で作業するよりもローカルで作業するほうが軽く、常にネット接続している必要もありません。ゆえにリポジトリ local で編集作業を行い、その変更をもとのリポジトリ upstream に統合することによって共同で翻訳作業を行っていくわけです。ですが local からいきなり upstream に対して変更を申請することはできません。local から自分の管理するリモートリポジトリ origin に一度変更を反映させる必要があります。その後、origin から upstream に対して変更の統合を申請するというのがプルリクエストの流れになります。

このように、リポジトリの複製の流れや内容変更の反映の流れが複雑になるのでつまづきポイントが結構あります。

表にしてまとめると以下のような３つのリポジトリを使用します。

リポジトリの識別名 | リポジトリの内容 | 備考
--|--|--
**upstream** | オリジナルのリモートリポジトリ | 他人が管理
**origin** | fork して複製した自分のアカウントにあるリモートリポジトリ | 自分が管理
**local** | 自分のローカルマシンに clone したローカルリポジトリ | 自分が管理、ここで編集を行う

### ブランチについて

利用するリポジトリが３つあるわけですが、更にブランチ(branch)という概念が追加されます。

https://docs.github.com/ja/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-branches#working-with-branches

リポジトリでは特定の時点の情報の状態をコミットという形で保存しています。このコミットからさらに変更を加えて意味ある形で再びコミットを作成していくと、コミットが時系列によって連結されたブランチ (枝) ができあがります。枝のある点から全く別の点を派生させることで枝を分岐させることができます。

イメージでまとめると、箱 (リポジトリ) の中に点 (コミット) が時系列に連結した枝 (ブランチ) が存在しているような感じです。分岐して存在する複数の枝の先端の点は枝そのものの名前として扱われます。master ブランチやそこから派生させてつくった作業ブランチなどです。イメージで言うと、プルリクエストというのはリポジトリ upstream の主要な枝 (master ブランチ) の先頭に自分が編集した変更情報を点として (コミットとして) 連結する操作です。

ブランチは各リポジトリごとに存在します。つまり、**upstream の master ブランチ、origin の master ブランチ、local の master ブランチというような感じで重複した名前のブランチが存在**しますが、それぞれは別のものなので注意してください。

結局のところ、翻訳作業で実際に触るのはリポジトリ local ですが、local の master ブランチは基本的には触りません、**master ブランチから派生させた作業ブランチ**(自分で名前をつける) 上で編集します。

結局、言葉で説明しても限界があるので検索して図などで理解するのがよいと思います。また実際に Git を使って**ブランチ操作**をやってみるとわかってくると思います。

参考: [マンガでわかるGit 12話「本家リポジトリに追従する方法」](https://next.rikunabi.com/journal/20180322_t12_iq/)

## (1) 最初のプルリクエストまでの流れ

では、実際にプルリクエスト作成までの流れを説明していきます。最初に流れを時系列順に並べます。その後に具体的な操作について説明していきます。※ブラウザ上での操作とターミナル上での操作があることに気をつけてください。

0. GitHub のアカウントプライバシーを設定し、Git で日本語ファイルが文字化けをしないように設定する
1. 公式リポジトリから自分のアカウントのリポジトリに fork する
2. ローカル環境に fork したリポジトリを clone する
3. コーディングスタイルを確認
4. 作業ブランチを作成してチェックアウト
5. 作業ブランチにて翻訳開始
6. ある程度の作業まとまりで add/commit する
7. origin に push する
8. GitHub にてプルリクエストを作成し、承認を待つ
9. プルリクエストに修正があれば修正後 origin に再び push する

```mermaid
---
title: 1~9までの流れ
---
graph TD
	Upstream -->|1. GitHubにて<br>fork| Origin
	subgraph Upstream
	A[master]
	end
	subgraph Origin
	B[master]
	E[topic]
	end
	subgraph Local
	C[master]
	D[topic]
  D -->|5~6.翻訳作業後<br>add/commit| D
	C -->|4. checkout|D
	end
	B -->|2. clone|C
	D -->|7. push|E
```

### 実際のやり方

上の 0〜9 までの流れを説明していきます。

共同で編集するということで、まず GitHub のアカウントプライバシーの設定を行います。デフォルトの状態だと PC のユーザー名と GitHub で登録してアカウントが commit log に表示されてしまいます。このままプルリクエストを出して承認されてしまうと面倒です (表示されてもよいという方はやらなくて結構です)。

参考: [GitHubにホストされているGitからの個人情報流出を防ぐために](https://tkhs0604.hatenablog.com/entry/github-email-exposure)

次に Git で日本語ファイルの文字化けを回避するためにコマンドを打ちます。

```shell
git config --global core.quotepath false
```

それでは最低限準備したので、本格的に作業を開始します。まず upstream を fork して origin を作成します。GitHub のリポジトリページから右上の fork ボタンをクリックすると自分のアカウントにリポジトリが複製されます。

![2-GitHubでフォーク](https://storage.googleapis.com/zenn-user-upload/hisqw3tezev16w87t3mver3ax1kk)

![3-frokで作成された自分のリポジトリ](https://storage.googleapis.com/zenn-user-upload/5ldouqa8nd0ex0y1jed65uqpy6qp)

![3-クリップボードにコピー](https://storage.googleapis.com/zenn-user-upload/jwq8xcxsra470ka8nh8r8kcv3y85)

ここまで GitHub でできたら、ターミナルを開き、作業ディレクトリを作成しフォークした自分のアカウントにあるリポジトリをクローンします。

```shell
git clone リポジトリのURL
git clone https://github.com/yo-goto/obsidian-docs.git
# ローカルにリポジトリをclone
git branch translation
# 作業ブランチtranslationの作成
git checkout translation
# 作業ブランチにチェックアウトする
git branch -a
# ブランチの確認
```

作業ブランチにチェックアウトしたら Obsidian や VSCode でリポジトリを開き実際に翻訳作業を開始します。編集するフォルダは日本語用のフォルダのみとして他の言語フォルダを編集しないよう注意してください。

翻訳作業が一通り終わったら `git add` でステージングして、`git commit` して保存します。日本語のフォルダのみ変更を加えるため add するのは日本語フォルダ (Obsidian の場合だと `ja`) 以下となります。つまりフォルダごとに add する場合には `git add ja/FolderName` と打つようになります。`git add -A` などのコマンドを利用してしまうと他の言語フォルダを間違って編集してしまった場合にそれらの間違った変更も追加してしまうことになるので `git add -A` などは使わずにフォルダ単位、ファイル単位で add を行うように気をつけてください。

```shell
git add ja/folderName_1
git add ja/folderName_2
# ファイルごと、フォルダごとにaddする
git committ -m "ja: update"
# コミットメッセージを添えてコミット
git push origin translation
# originの作業ブランチtranslationにpush
```

origin(リモートリポジトリ) の作業ブランチに push したら GitHub にてプルリクエストを作成し送信します。master ブランチではなく作業ブランチ translation からプルリクエストを作成する点に注意してください。「Compare & pull request」ボタンを押して、内容に関するコメントを適度に書き承認を待ちます。

![プルリク説明0-1](https://storage.googleapis.com/zenn-user-upload/zgbrrqjax0ys4xx31d60e9w59ggp)

master ではなく push した作業ブランチになっていることを確認してください。作業ブランチになっていれば「Compare & pull request」ボタンをクリックしてください。更新内容について簡単に説明コメントを記載して「Create pull request」ボタンを作成すればプルリクエストの作成完了となります。

![プルリク説明7 コメント](https://storage.googleapis.com/zenn-user-upload/9zbqm2srwaz6ryogpdqrym7smlwn)

プルリクエストが承認されるまでは次のようなページが表示され「open」状態になります。リポジトリの管理者や他のコントリビューターからプルリクエストについてのコメントがあれば下に表示されます。

![プルリク説明6](https://storage.googleapis.com/zenn-user-upload/mhb49o175ooz4p6aeymv44pgnv69)

あとは承認される (マージされる) のを待ちつつ作業するか、作業を中断します。

`git push` した後、時間がたってしまうと「Compare & pull request」が非表示になる場合があります。これであわててしまうことがあるので注意してください。非表示になっている場合には「Pull requests」のセクションに進み、右側にある「New pull request」ボタンをクリックしてください。

![プルリク説明1 非表示](https://storage.googleapis.com/zenn-user-upload/xctt21ypb90rv6ga0kja2022a4p1)

フォークを利用したプルリクエストを作成する場合にはデフォルトの状態ではできないので、「compaer across forks」のリンクをクリックして、比較するブランチを upstream ではなく origin の作業ブランチにします。

![プルリク説明2 非表示](https://storage.googleapis.com/zenn-user-upload/r95hpiq5oo78qlgnmcijy7g01i9t)
![プルリク説明3 非表示](https://storage.googleapis.com/zenn-user-upload/od1ug234k6na04r0q4otevv3xy0g)
![プルリク説明4 非表示](https://storage.googleapis.com/zenn-user-upload/fxaybjrca4mtr8qvv6sdt6r1tubm)

ここまでやって「Create pull request」ボタンをクリックすることでプルリクエストが作成できるようになります。

![プルリク説明5 非表示](https://storage.googleapis.com/zenn-user-upload/w0z2ybbc45apm1q4fghmz0ecsu77)

プルリクエスト作成後に内容を修正する場合には、そのままの状態でローカルで修正を加えた後で同様に add/commit して origin の作業ブランチ translation に再び push します

```shell
git add ja/修正を加えたフォルダやファイル名
git commit -m "コミットメッセージ"
git push origin translation
# originの作業ブランチに再度pushする
```

これによってプルリクエストの内容が自動修正されます。

![プルリク説明9 修正後に再push](https://storage.googleapis.com/zenn-user-upload/ad2d2dk4rww86hezqlhy9kmuw0lm)

プルリクエストが承認されたら (マージされたら)、公式のオリジナルのリポジトリ (upstream) の master ブランチが自分の作成したプルリクエストの内容がマージされた状態となります。

![プルリク説明10 プルリク承認](https://storage.googleapis.com/zenn-user-upload/3iwcvfo1o1obz0vbhxrqhaqkwtuz)

基本的にはこの流れで作業終了となりますが、ヘルプドキュメントでは更新は定期的にあるため、時間が経過したら再び翻訳内容が追加され、再び翻訳作業をしプルリクエストを作成することになります。2 回目以降はいままで流れと若干異なり、ローカルのブランチを更新する作業がでてくるので注意してください。

## (2) 2 回目以降のプルリクエストの流れ

9. `upstream` の追加と確認
10. ローカルの `master` ブランチを `upstream` の最新に追いつかせる
11. `origin` の `master` に `push`
12. 以下のいずれかを行う
    a. `master` の最新内容を前の作業ブランチに `merge` して反映してからチェックアウト
    b. 要らない作業ブランチを削除し作業ブランチを作り直しチェックアウト
13. 更新された内容との差分を確認しながら翻訳再開する
14. add/commit した後に `origin` に `push` する
15. GitHub にてプルリクエストを作成し、承認を待つ
16. プルリクエストに修正があれば修正後 `origin` に再び `push` する

```mermaid
graph TD
	E -->|"15(8). Pull request作成"|A
	subgraph Upstream
	A[master]
	end
	subgraph Origin
	B[master]
	E[topic]
	end
	subgraph Local
	C[master]
	D[topic]
  D -->|13.翻訳作業後<br>add/commit| D
	C -->|"12. (a)merge<br>(b)作業ブランチ作成<br>のどちらかを行い<br>チェックアウト"|D
	end
	C -->|11. push|B
	D -->|14. push|E
	A -->|10. fetch & merge|C
```

### 実際のやり方

上の 9〜16 までの流れを説明していきます。

#### 翻訳作業再開するまで

作業を再開するために、ローカルリポジトリ (local) の `master` ブランチの内容を fork 元のオリジナルのリモートリポジトリ (upstream) の `master` ブランチの状態に追いつかせる必要があります。

まず remote の設定を確認し、`upstream` として追加します。次のコマンドで `upstream` という名前でオリジナルのリモートリポジトリを追加します。なお、_この作業は 3 回目以降のプルリクエストからは必要ありません_。

```shell:今回だけ必要な作業
git remote -v
# fork元の公式オリジナルのリモートリポジトリがremoteのupstreamとして設定しているか確認する。
git remote add upstream fork元のURL
# upstreamとして設定する
git remote add upstream https://github.com/obsidianmd/obsidian-docs.git
```

`upstream` から最新を `fetch` して local の `master` ブランチにマージしてあげます。local の `master` ブランチが最新になったら自分のリモートブランチ (origin) の `master` に `push` して反映させます。上で提示した 9〜16 までの流れの 12(a) になります。

```shell
git checkout master
# masterブランチにチェックアウトする
git fetch upstream master
# upstreamから更新情報をとりよせる
# 更新情報は upstream/master に
git merge --ff-only upstream/master
# ファストフォワードマージを行い、リポジトリlocalのmasterブランチを最新にする
git push origin master
# 自分のリモートリポジトリ(origin)のmasterブランチにその最新内容を反映させる
```

`git merge` コマンドでは、オプション `--ff-only` をつけることでファストフォワードマージ(fast-forward merge)でマージを実行します。ファストフードマージできない場合にはエラーとなります。

#### git fetch と git merge が行うこと

作業の流れからは離れますが、`git fetch` と `git merge` で行っていることを理解しておくことは非常に重要なので、ここで何を行っているかを解説しておきます。

ブランチには種類があり、一般的に作業を行うローカルブランチを `main` としてみると、`main` に対して上流であるリモート追跡ブランチ(remote-tracking branch) の `origin/main` とリモートリポジトリ上にあるリモートブランチ `origin main` という３つのブランチがあることになります。※ なお `origin main` というのはコマンド上での引数としての指定の仕方で通常はリモートの `main` などと識別するのが普通でしょう。

ブランチ名 | 説明
--|--
`main` | ローカルブランチ。
`origin/main` | ローカルブランチ。`origin main` を追跡する、**リモート追跡ブランチ**(remote-tracking branch) である。
`origin main` | リモートブランチ。リモートリポジトリ `origin` の実際の `main` ブランチ。

```mermaid
---
title: ブランチの上流関係
---
graph LR
  subgraph "ローカルリポジトリ(local)"
    subgraph リモート追跡ブランチ
      A1[origin/main]
    end
    A0[main]
  end
  subgraph "リモートリポジトリ(origin)"
    A2[main]
  end
  A0 -.->|上流| A1
  A1 -.->|上流| A2
```

`git fetch` を行う際には実際にはリモートリポジトリからの変更をまずリモート追跡ブランチに反映しており、`git merge` を行う際には、リモート追跡ブランチから変更を作業を行うためのローカルブランチ `main` に反映していることになるわけです。

```mermaid
---
title: fetchとmergeの流れ
---
graph RL
  subgraph "ローカルリポジトリ(local)"
    subgraph リモート追跡ブランチ
      A1[origin/main]
    end
    A0[main]
  end
  subgraph "リモートリポジトリ(origin)"
    A2[main]
  end
  A1 -->|merge| A0
  A2 -->|fetch| A1
```

```sh
git checkout main
git fetch origin main # origin main の変更を origin/main に反映
git merge origin/main # origin/main の変更を main に反映
```

なお、`git pull` というコマンドはこの作業を一気に行うコマンドです。

```sh
git pull origin main
# origin main の変更を origin/main に反映してからその変更を main に反映
```

fetch, merge, pull の関係は以下の GitHub 公式ドキュメントが詳しいです。

https://github.com/git-guides/git-pull

さて、OSS で使われる Fork and pull model では origin だけではなく upstream という２つのリモートリポジトリがあったので、リモート追跡ブランチも複数あることに注意が必要です。図にすると以下のような複雑な関係になっています。

```mermaid
---
title: ３つのリポジトリでの上流関係
---
graph LR
  subgraph local
    subgraph リモート追跡ブランチ
      A1[origin/main]
      AA1[upstream/main]
      B1[origin/feature]
    end
    A0[main] -.-> A1 & AA1
    B0[feature] -.-> B1
  end
  subgraph GitHub
    subgraph origin
      A2[main]
      B2[feature]
    end
    A1 -.-> A2
    B1 -.-> B2
    subgraph upstream
      AA2[main]
    end
  end
  AA1 -.-> AA2
  style GitHub fill:#eee, stroke:#000
```

fork 元のリポジトリである upstream からの変更を main に反映させるには以下のように行います。

```sh
git checkout main
git fetch upstream main # リモート追跡ブランチを更新
git merge upstream/main # リモート追跡ブランチの内容を取り込む
```

`git pull` でやるなら以下のように行います。

```sh
git checkout main
git pull upstream main
```

#### 翻訳作業再開

さて元の作業解説の流れに戻ります。

```shell
git checkout translation
# 作業ブランチにチェックアウトする
git status
# 作業ブランチに変更がないか調べる(もしなにかあればコミットしたり、stashしたりして退避させる)
git log --all decorate --graph --oneline
# グラフを出してコミットのハッシュ値を確認できる。作業ブランチtranslationのハッシュ値をしらべる。冒頭の数桁のみでよい
git show HEAD
# これでも最新のコミットのハッシュがわかる(あとでハッシュ値を使う)
git merge --no-ff master
# 最新内容であるlocalのmasterを作業ブランチにリカーシブマージさせる(これを行うとマージメッセージの入力画面がvimモードででるので適当なメッセージを書いて終了)
```

マージを `--no-ff` オプションで行うとリカーシブマージになります。これを行うと Vim が立ち上がりマージコミットのメッセージを書くことになります。Vim の使い方 (挿入モードや保存終了の方法) を簡単に調べておくとよいです。

前回から更新された内容を確認するため差分を表示させます。上で調べた前回コミットのハッシュ値と master ブランチとの差分を確認します。

```shell
git diff ハッシュ値 master
# これでも差分を確認できる。ハッシュ値は冒頭の数桁の数字をいれればよい
git diff 78d63a7 master en/
# masterブランチとコミット78d63a7間でenフォルダについての差分を確認できる
git diff 78d63a7 master en/Pulgins/
# Pluginフォルダの内容だけ差分を見る
git diff 78d63a7 master en/Pulgins/test.md
# test.mdのファイルの差分を見る
git diff 作業ブランチ master en/
# ブランチで比較
```

更新内容が複数のファイルや追加内容が多い場合にはターミナルで diff を使うよりも [GitHubのcompare機能](https://docs.github.com/ja/github/committing-changes-to-your-project/comparing-commits) や VSCode などの差分表示機能 (特に [GItLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens) などの拡張機能) を使った方がよいですが、なれるまでは `git diff` を使ってひとつずつやる方が後学のためになると思います。

![プルリク説明11 差分表示](https://storage.googleapis.com/zenn-user-upload/mn6hy4y4ghg95u1ll11grgm1vh6f)

差分を確認しながら作業ブランチにて追加内容分の翻訳作業をすすめていきます。ターミナル上で `git diff` を使って差分表示させると修正前と修正後が赤と緑で表示されるので、その内容をみながら日本語版での内容を追加修正していきます。Obsidian の場合、英文 (原文) のドキュメントフォルダ (en/) と日本語などの他言語用のフォルダ (ja/など) は別れています。なので diff を使うときには英語のドキュメントフォルダを対象に差分を見ます。その差分から追加された部分を見て、翻訳を開始します。

##### なるべくマージしないで作業する方法

作業ブランチに master をマージするのではなく、作業ブランチを毎回新しく作りなおす方法を行ったほうがわかりやすいことがあります。個人的に慣れるまではこの方法を行っていたので、マージの意味がよくわかっていない場合には、こちらの方法をとるのがよいです。上で提示した 9〜16 までの流れの 12(b) です。(a) を行わない代わりに (b) をやるという感じですね。

プルリクエスト承認後、`git merge --no-ff master` をやらずに次の操作を行います。

```shell
git checkout master
# masterにチェックアウト
git branch -d translation
# プルリクエストが承認されたので作業ブランチを削除してしまう
git branch -a
# ブランチが削除されたことを確認
git branch translation
# 作業ブランチを最新のmasterから作り直す(分岐させる)
git checkout translation
# これで最新内容になった作業ブランチで作業再開できる
```

マージの意味が理解できたら、いちいち作業ブランチを破棄して作り直す方法をとらずに、マージによって作業ブランチの内容を最新にすればよいです。

#### 作業終了後にプルリクエスト作成

初回のプルリクエストまでの流れと同じく add/commit 後に自分のリモートリポジトリ (origin) の作業ブランチに push します。

```shell
git add ja/folderName_1
git add ja/folderName_2
# ファイルごと、フォルダごとにaddする
git committ -m "ja: update"
# コミットメッセージを添えてコミット
git push origin translation
# originの作業ブランチtranslationにpush
```

push を終えたら同じように GitHub で作業ブランチ translation からプルリクエストを作成し承認を待ちます。

### 流れのまとめ

シーケンス図でのまとめです。リポジトリの種類は３つ(upstream, origin, local)あり、それぞれに `master` や作業ブランチがあることに注意してください。

```mermaid
%%{init: { 'sequence': { 'showSequenceNumbers': true } } }%%
sequenceDiagram
  box GitHub
	participant A as upstream(master)
	participant B as origin(master)
	participant C as origin(feature)
  end
  box Local
	participant D as local(master)
	participant E as local(feature)
  end
	rect rgba(0, 255, 0, .1)
		Note over A,B: 1回目のプルリク
		A->>B: fork
		B->>D: clone
		D->>E: checkout
    loop
      E->>E: 翻訳作業(add/commit)
    end
		E->>C: push
		C->>A: pull request
	end
	rect rgba(250, 150, 10, .1)
		Note over A,B: 2回目以降のプルリク
		A->>D: fetch & merge
		D-->>B: push
		D->>E: checkout & merge
    loop
      E->>E: 翻訳作業(add/commit)
    end
		E->>C: push
		C->>A: pull request
	end
```

:::message alert
`feature` は任意の作業ブランチのことを示しています。
:::

## (3) 慣れたらやってみること

### プルリクエストの分割

別々の意図でプルリクエストを分割できる。ローカルブランチを複数きって別々に `git push` を行い、それぞれでプルリクエストを作成することでプルリクエストの分割を行えます。内容追加と修正でプルリクエストを分割するなどができるようになります。

### Git client の利用

Working Copy などの Git client を利用してモバイルで作業すると暇なときに翻訳作業を行うことができます。こちらでもメールアドレスを GitHub で設定した同一のもにする必要があります。

### vscode の拡張機能で作業効率アップ

GitLens や GitGraph などの拡張機能を入れることで作業効率を上げることができます。

![差分を見ながら追記](https://storage.googleapis.com/zenn-user-upload/xz9q1mkpehaeehd8iqhvvg9ogdsi)

GitLens を利用すればブランチ間 (コミット間) の差分を簡単に確認しながら翻訳作業を行えます。更新された部分のみを見ながら翻訳を追加できます。

## 他によく使うコマンドの概説

```shell
git checkout -b branchName
# ブランチを新規作成した上でチェックアウトする

git merge --no-ff branchName
# non fast-forward(recursiveマージ)
git merge --ff-only branchName
# fast-forwardマージできるならマージする、できない場合にはマージしない

git rm fileName
# git監視下でファイル削除
git mv 旧ファイル名 新ファイル名
# ファイルの名称変更
git mv fileName folderpath
# ファイルの移動

git config --global list
# globalのconfig確認(ユーザー情報などの確認)
git log --graph --oneline
# ログをグラフとして簡易表示
```

リモートブランチ関連

```shell
git push --delete origin branchName
# リモートブランチの削除
git checkout -b branchName origin/branchName
# リモートブランチをローカルにcheckout
```

### fork syncバージョンでやってみる

2021 年頃に追加された GitHub 上での Fork リポジトリの同期機能([Sync fork](https://github.blog/changelog/2021-05-06-sync-an-out-of-date-branch-of-a-fork-from-the-web/))を使うことで origin の master に `push` したり、upstream の master から `pull` しなくてもよくなります。

![sync-fork](https://storage.googleapis.com/zenn-user-upload/84cddb81275f-20230815.jpg)

GitHub 上で fork リポジトリ(`origin`)の "Sync fork" ボタンから "Update branch" ボタンをクリックすることでそのリポジトリの master (あるいは main) ブランチを upstream に追従できます。

あるいは、GitHub CLI を使って以下のようにコマンドラインから GitHub 上の origin リポジトリを upstream リポジトリに同期できます。

```sh
# 自分のアカウントのforkリポジトリ名を指定
gh repo sync yo-goto/obsidian-docs
```

いずれかの方法で origin を upstream に追従させたら、ローカルで以下のコマンドを実行することで local の master を更新できます。

```sh
git pull origin master
```

## 翻訳そのものについて

### 自由にやると結構楽しい

翻訳の厳密性はプロジェクトごとに色々あると思いますが有志の翻訳であれば、ある程度のゆるさがあっても大丈夫だと思います。あまり気にしすぎずに挑戦してみるとよいと思います。分からない表現や専門用語はどんどんネットで検索しましょう。自分の場合には、不自然な日本語になるくらないなら解釈から相当する自然な言い回しを優先しています。文も切ったりつなげたり、結構自由にやっています。参加者の少ない日本語翻訳のプロジェクトの場合、このように自分の好きなようにできるのも利点だと思います。一方、人数が少ないと校正や、表現性の向上、誤訳やタイポなどの修正があまりできないのが難点です (textlint を使うのがいいかもしれません)。

Obsidian の翻訳プロジェクトでは、翻訳に問題や修正、追加内容があれば誰でもプルリクエストや issue を GitHub に作成できるスタイルでやっているので翻訳作業自体は結構はやいスピードで進みます。もちろんニッチなソフトウェアのベータ版という開発途中のもの故にこのような自由な事ができるわけで、他のメジャーなソフトウェアやサービスではなかなか難しいのではないでしょうか。

他の翻訳プロジェクトってどうやってやっているのかと思って他のサービスなどちょっと調べてみました (スケールによって結構違いますねやはり)。

- [Notion Community Translation Guide](https://www.notion.so/Community-Translation-Guide-d1a37aa2e22f4c9daf2a1efe9ae13bc2)
- [Hugo Docs Japanese | Hugo Documentation の和訳・日本語翻訳プロジェクト](https://hugojapan.github.io/)
- [yohamta/typescript-book-jp: TypeScript Bookの日本語訳です。](https://github.com/yohamta/typescript-book-jp)

### 日本語表現が難しい

英文の解釈よりも日本語の表現が難しいです。英語で理解できても日本語自体の表現や語彙のストックがないので (笑)、どうしても直訳っぽくなってしまい、他のアプリケーションやサービスのドキュメントの日本語訳版などを参考にすることもありました。また、他の言語翻訳者がどのように翻訳しているのか Google 翻訳などを使って見ていました (例えばイタリア語やロシア語など)。

実際のところ、まだ稚拙な表現が多いかもしれませんが、現在は「とりあえず翻訳を完成すること」を念頭にプルリクエストで修正点や校正などをはさみつつクオリティを上げていくようにしています。

### UI とヘルプドキュメントの一貫性

また、自分の場合はヘルプドキュメントから翻訳をはじめましたが途中から UI との整合性を図るためにヘルプドキュメントでは UI をベースとした説明文になるように注力しました。UI の翻訳では json ファイルの文字列を翻訳していく形でしたが、英文の文字列を見てもどういった操作なのかわからないことが多いので、実際に Obsidian の画面を出してどの部分の文字列なのか確認しながらやらないといけませんでした。

Obsidian は Electron ベースで開発されているので developer tool が使えます。翻訳であやしい部分があれば UI にあっているか、developer tool で実際に変えてみて合うかどうかを確認しています。

他に、特段注意すべきことは、句読点やスペースや半角全角などの表記ルールや語彙対応表などを早いうちから作成しておくと良いです。あとから整合性を整えるのは面倒なので最初から表現や単語のルールを統一しておきましょう。UI との整合性もを考える上で重要です。といってもいきなりルールを想定することはできません。その場合は翻訳しているうちに必要なルールがどんどんでてきますので、その都度はやめに追加していくのがよいです。表記のスタイルについては [Wrodpressの翻訳スタイルガイド](https://wordpress.com/ja/support/translation-style-guide/) や [JTF(日本翻訳連盟)](https://www.jtf.jp/tips/styleguide) の翻訳スタイルガイドなどが参考になります。ルールを新たに追加する際には既存のドキュメントに対してルールを追加するためテキストエディタの検索と置換機能を駆使して全体に適応します (これも textlint でできそうです。プリセットルールで JTF-style なるものが存在します)。

## 他参考サイト

[Gitをインストールしたら真っ先にやっておくべき初期設定 - Qiita](https://qiita.com/wnoguchi/items/f7358a227dfe2640cce3)
