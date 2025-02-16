---
title: "カスタム CSS スニペットでデザインしよう"
cssclass: zenn
date: 2022-12-22
modified: 2022-12-24
url: "https://zenn.dev/estra/articles/mbo-css-snippets"
AutoNoteMover: disable
tags: [" #type/zenn/book #obsidian "]
aliases: OZ本『カスタム CSS スニペットでデザインしよう』
---

# このチャプターについて

このチャプターでは、Obsidian の CSS カスタムスニペットを利用する例として公開ノートや記事に対して視覚的な警告表示を行う方法を紹介します。

# コミュニティテーマについて

Obsidian では Electron で開発されており、外観等は CSS を使って自分でデザインすることが可能です。

コミュニティによって開発されたコミュニティテーマは以下のようにいくつも公開されています。これは「設定」→「外観」→「テーマ」→「管理」から確認でき、クールなテーマから好きなものを選んでインストールできます。

![img](/images/oz/img_oz-obsidian-community-themes.jpg)

テーマは現在 (2022-12-22 時点) において 129 個提供されていますが、以下の Minimal などが人気で、筆者も以前まで使っていたおすすめのテーマです。

https://github.com/kepano/obsidian-minimal

もちろん、テーマは自分で開発することができ、コミュニティに申請することで多くの人につかってもらうことができるため、挑戦したい方はぜひやってみてください。

ただし、いきなりテーマを作るのも大変なので次に紹介するカスタム CSS スニペットから初めて見るのをおすすめします。

# カスタム CSS スニペットについて

上記のコミュニティテーマ以外の外観のカスタマイズ方法として、カスタム CSS スニペットというものがあります。

文字通り、CSS のスニペットファイルを使って Obsidian の外観を調整することが可能です。

実は大分昔のバージョンでは、`obsidian.css` という１つのファイルで CSS のカスタマイズを行っていた時期がありますが、現在では複数の CSS スニペットを有効・無効にすることで外観を調整することが可能となっています。

使用用途としては、コミュニティテーマの調整や、機能的に常に変更しておきたい部分などについてのデザインを CSS スニペットに分離して管理しておくことが可能です。

筆者の使用環境では、デフォルトの Obsidian テーマに機能的な CSS スニペットで調整を行うという方法を好んで使っています。

## スニペットの管理

カスタム CSS スニペットの管理は「設定」→「外観」→「CSS スニペット」からそれぞれのスニペットファイルを有効・無効として個別で設定できます。利用したいスニペットだけ有効化してください。

![img](/images/oz/img_oz-snippets-setting.jpg)

設定画面右上のフォルダアイコンをクリックするとスニペットが管理されている `VaultName/.obsidian/snippets` フォルダが開きます。このフォルダに CSS ファイルを用意して、設定画面右上のリフレッシュボタンをクリックするとスニペットファイルが読み込まれ、有効化できるようになります。

## CSS 変数

CSS スニペットの作り方の基本は CSS カスタムプロパティを利用してそれぞれの設定項目に色などを定義しておきます。

https://developer.mozilla.org/ja/docs/Web/CSS/Using_CSS_custom_properties

筆者の場合はデフォルトのスタイルが用意されている `app.css` の CSS 変数をオーバーライドしたくないので、`--sdf` というプリフィクスをつけた変数名で定義して、`var()` 関数で参照するようにしています。

https://developer.mozilla.org/ja/docs/Web/CSS/var

Obsidian にはダークとライトという２つのベーステーマがあるので、それぞれで利用する CSS 変数は対応する `theme-dark` クラスと `theme-light` クラスで定義します。

```css
/* ダークとライトの両方のテーマで利用する */
.theme-dark,
.theme-light {
  --sdf-header-color-border-left-h1: gray;
  --sdf-header-color-border-bottom-h2: gray;
}
/* ダークテーマでのみ利用する */
.theme-dark {
  --sdf-header-color-background-h1-for-label: #000000;

  --sdf-header-color-text-h1: #f2f2f2;
  --sdf-header-color-text-h2: hsl(25, 80%, 52%);
  --sdf-header-color-text-h3: hsl(25, 60%, 52%);
  --sdf-header-color-text-h4: hsl(25, 40%, 52%);
  --sdf-header-color-text-h5: hsl(25, 20%, 52%);
  --sdf-header-color-text-h6: hsl(25, 00%, 52%);
}
/* ライトテーマでのみ利用する */
.theme-light {
  --sdf-header-color-background-h1-for-label: #f2f2f2;

  --sdf-header-color-text-h1: #000000;
  --sdf-header-color-text-h2: hsl(25, 80%, 58%);
  --sdf-header-color-text-h3: hsl(25, 60%, 58%);
  --sdf-header-color-text-h4: hsl(25, 40%, 58%);
  --sdf-header-color-text-h5: hsl(25, 20%, 58%);
  --sdf-header-color-text-h6: hsl(25, 00%, 58%);
}
```

# 機能的なスタイルスニペット

ここでは、Zenn や他のブログ用記事の作成に使えるようないくつかのカスタム CSS スニペットを紹介します。

:::message
ここで紹介する CSS スニペットはあくまで筆者が利用しているものなので、合わなければ自分に合わせて改造したり、組み合わせたりして活用してください。
:::

## 見出し

マークダウンの見出しは非常に重要な機能であり、筆者はこれがあるからマークダウン記法を使っているといっても過言ではないです。

見出しのレベリングによってその情報がノートや記事においてどのような粒度を持つかを視認できるようにスタイリング調整を施しておくと非常に便利です。

ここではその設定を行うためのセレクターと筆者が行っているスタイリングを紹介します。

Obsidian の最新エディタではビューモードに「リーディングビュー」と「編集ビュー」という２つのビューがあります。CSS を使ってカスタムする場合にはこの２つのビューについて調整する必要があります。

ビューモード | 説明 | 基本セレクター
--|--|--
リーディングビュー | 装飾用の記号が完全に非表示となり編集できないビューモード | `markdown-preview-view`
編集ビュー | 編集が可能で入力行では装飾記号がレンダリングされる | `cm-` 系列

リーディングビューは分かりやすく、`markdown-preview-view` というクラスがあり、この配下のクラスにスタイル調整を行えばよいです。

一方、編集ビューでは、`cm-` というプリフィクスがついたクラスを自分で開発者ツールを起動して探すことになります。

:::message
基本セレクターとして挙げているクラスは例であり、その他のセレクターでもうまくいく場合があります。
:::

さて、両方のビューモードで見出しのカラーリングを変更するスタイリングを行うには上記で説明した `theme-dark` と `theme-light` でそれぞれカスタム CSS プロパティを定義し、リーディングと編集ビューの両方でスタイリング定義したいです。

編集ビューにおける見出し用のクラスは `cm-header` です。それぞれのレベルごとに `cm-header-` というクラスが更にあるので、両方のセレクタを持つ span 要素があるので、例えばレベル１見出しなら `.cm-hader.cm-header-1` としてスタイリングを定義します。リーディングビューの方は単純に `.markdown-preview-view h1` というよう `markdown-preview-view` クラス配下の見出し要素にすればうまくいきます。

```css
.cm-header.cm-header-1,
.markdown-preview-view h1 {
  color: var(--sdf-header-color-text-h1);
}
```

それでは今まで解説してきた CSS を記述する適当な名前のスニペットファイル `__header-style.css` を用意します。

見出しのレベルによって色を徐々に変化させることで視覚的にどのレベルなのかが把握できるようなスタイリングにするには CSS の [hsl 関数](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/hsl) を利用するとよいです。

```css
.theme-dark {
  --sdf-header-color-text-h1: #f2f2f2;
  --sdf-header-color-text-h2: hsl(25, 80%, 52%);
  --sdf-header-color-text-h3: hsl(25, 60%, 52%);
  --sdf-header-color-text-h4: hsl(25, 40%, 52%);
  --sdf-header-color-text-h5: hsl(25, 20%, 52%);
  --sdf-header-color-text-h6: hsl(25, 00%, 52%);
}
.theme-light {
  --sdf-header-color-text-h1: #000000;
  --sdf-header-color-text-h2: hsl(25, 80%, 58%);
  --sdf-header-color-text-h3: hsl(25, 60%, 58%);
  --sdf-header-color-text-h4: hsl(25, 40%, 58%);
  --sdf-header-color-text-h5: hsl(25, 20%, 58%);
  --sdf-header-color-text-h6: hsl(25, 00%, 58%);
}

.cm-header.cm-header-1,
.markdown-preview-view h1 {
  color: var(--sdf-header-color-text-h1);
}
.cm-header.cm-header-2,
.markdown-preview-view h2 {
  color: var(--sdf-header-color-text-h2);
}
.cm-header.cm-header-3,
.markdown-preview-view h3 {
  color: var(--sdf-header-color-text-h3);
}
.cm-header.cm-header-4,
.markdown-preview-view h4 {
  color: var(--sdf-header-color-text-h4);
}
.cm-header.cm-header-5,
.markdown-preview-view h5 {
  color: var(--sdf-header-color-text-h5);
}
.cm-header.cm-header-6,
.markdown-preview-view h6 {
  color: var(--sdf-header-color-text-h6);
}
```

これでレベルが深くなるにつれて、色が暗くなるような見出しのスタイリングとなりました。

ただし、色を６段階まで変化させて視認するのは大変なので、特定の見出しレベルに対してさらに特徴をつけて重み付けしたいと思います。

見出し h1 のレベルはラベルとなるようなデザインに変更します。これでトップレベルの見出しが一目でわかり、記事の区切れが把握できるようになります。また、見出し h2 のレベルはメモでも多用するので、デザインはそのままで下線を引いて区切りが分かりやすいように変更します。以下の CSS と対応するカスタム CSS プロパティをスニペットファイルに追加しておきます。

```css
/* =======================
見出しh1の装飾 (ラベル装飾)
======================= */
.HyperMD-header-1,
.markdown-preview-view h1 {
  /* -----------------------
  labeling
  ----------------------- */
  background: var(--sdf-header-color-background-h1-for-label);
  border-radius: 3px;
  border: solid 1px white;

  /* -----------------------
  spacing (この項目は important が必要)
  ----------------------- */
  padding-top: 2px !important;
  padding-bottom: 2px !important;
  padding-left: 5px !important;
}

/* =======================
見出しh2の装飾 (下線装飾)
======================= */
.HyperMD-header-2,
.markdown-preview-view h2 {
  border-bottom: solid 2px var(--sdf-header-color-border-bottom-h2);
}
```

また、深いレベルでは色の段階を見分けるのが困難な場合があるので、見出し h5 のレベルだけは別の色に変更しておきましょう。

これによって以下の画像のようなスタイリング調整が完了しました。

![img](/images/oz/img_oz-header-styling.jpg)

最終的なカスタム CSS スニペットのファイルは以下のようになります。

:::details 最終的なスニペットファイル

```css:__header-style.css
.theme-dark,
.theme-light {
  --sdf-header-color-border-left-h1: gray;
  --sdf-header-color-border-bottom-h2: gray;
}
.theme-dark {
  --sdf-header-color-background-h1-for-label: #000000;

  --sdf-header-color-text-h1: #f2f2f2;
  --sdf-header-color-text-h2: hsl(30, 100%, 50%);
  --sdf-header-color-text-h3: hsl(30, 80%, 50%);
  --sdf-header-color-text-h4: hsl(30, 40%, 50%);
  /* h5, h6 は基本的に利用しない */
  --sdf-header-color-text-h5: hsl(200, 40%, 50%);
  --sdf-header-color-text-h6: hsl(200, 10%, 50%);
}
.theme-light {
  --sdf-header-color-background-h1-for-label: #f2f2f2;

  --sdf-header-color-text-h1: #000000;
  --sdf-header-color-text-h2: hsl(30, 100%, 50%);
  --sdf-header-color-text-h3: hsl(30, 80%, 50%);
  --sdf-header-color-text-h4: hsl(30, 40%, 50%);
  /* h5, h6 は基本的に利用しない */
  --sdf-header-color-text-h5: hsl(200, 40%, 50%);
  --sdf-header-color-text-h6: hsl(200, 10%, 50%);
}

/* =======================
編集画面とプレビューの目次の色
======================= */
.cm-header.cm-header-1,
.markdown-preview-view h1 {
  color: var(--sdf-header-color-text-h1);
}
.cm-header.cm-header-2,
.markdown-preview-view h2 {
  color: var(--sdf-header-color-text-h2);
}
.cm-header.cm-header-3,
.markdown-preview-view h3 {
  color: var(--sdf-header-color-text-h3);
}
.cm-header.cm-header-4,
.markdown-preview-view h4 {
  color: var(--sdf-header-color-text-h4);
}
.cm-header.cm-header-5,
.markdown-preview-view h5 {
  color: var(--sdf-header-color-text-h5);
}
.cm-header.cm-header-6,
.markdown-preview-view h6 {
  color: var(--sdf-header-color-text-h6);
}

/* =======================
見出しh1の装飾 (ラベル装飾)
======================= */
.HyperMD-header-1,
.markdown-preview-view h1 {
  /* -----------------------
  labeling
  ----------------------- */
  background: var(--sdf-header-color-background-h1-for-label);
  border-radius: 3px;
  border: solid 1px white;

  /* -----------------------
  spacing (この項目は important が必要)
  ----------------------- */
  padding-top: 2px !important;
  padding-bottom: 2px !important;
  padding-left: 5px !important;
}

/* =======================
見出しh2の装飾 (下線装飾)
======================= */
.HyperMD-header-2,
.markdown-preview-view h2 {
  border-bottom: solid 2px var(--sdf-header-color-border-bottom-h2);
}
```
:::

## 公開情報に対する視覚的な警告

この本で紹介している方法では、Obsidian 内部のナレッジベースにメモや様々なレイヤーのための記事が作成されます。

そのような混合的な環境では、どのメモが記事になっているかを把握できるようにしたい場合があります。例えば、個人情報などを誤って記事に書かないように、公開されているメモや記事は通常のメモとは別に特別扱いしたいというケースです。

このようなケースではいくつかやり方があると思いますが、カスタム CSS スニペットを利用することで簡単に公開ノートに対して視覚的な警告表示を出すことができます。

Obsidian の YAML フロントマターでは `casclass` キーという特別なサポートキーが存在しています。例えば、以下のようにフロントマターで `publish` という値を設定したとします。

```yaml
cssclass: publish
```

これによって実はノートの以下の場所に `publish` クラスが追加されます。

```html
<div class="view-content">
  <div class="markdown-source-view cm-s-obsidian mod-cm6 is-live-preview node-insert-event publish">
    ...
```

この `cssclass` キーに設定できるクラス名を利用することでそのノートに対して色などで警告表示を出すのが分かりやすい方法です。上記の要素関係から以下のようなセレクタでスタイリング設定を行えます。

```css
.workspace-split.mod-root .view-content .publish {
  background: red;
}
```

このように単色で背景色を変更してもよいですが、ダークとライトでそれぞれ設定したりするのも面倒で、見づらくなる場合が多いので、[linear-gradient](https://developer.mozilla.org/ja/docs/Web/CSS/gradient/linear-gradient) 関数を利用して現在のベースとなるカラーとのグラデーションで表示されるように以下のように CSS スニペットを作成します。

```css
.theme-dark,
.theme-light {
  --sdf-published-note-bg-color-from: var(--background-primary);
}
.theme-dark {
  --sdf-published-note-bg-color-to: hsl(350, 50%, 30%);
}
.theme-light {
  --sdf-published-note-bg-color-to: hsl(350, 80%, 80%);
}

.workspace-split.mod-root .view-content .publish {
  background: linear-gradient(var(--sdf-published-note-bg-color-from), var(--sdf-published-note-bg-color-to));
}
```

`app.css` の `--background-primary` 変数には基本となる背景色が定義されているので、これを組み合わせて CSS スニペットとして実装すれば良いです。

さて、Zenn 用のクラスとして `zenn` を用意しておき、さらに Hugo や Obsidian Publish、deno blog など自分が持つブログサイトごとに特殊クラスを用意することで、以下のようにひと目でどのレイヤーに公開されているのを把握できるようになります。

![img](/images/oz/img_oz-color-blog-admonition.jpg)

これで誤った情報の公開を可能な限り減らすことができるでしょう。

:::details 最終的なスニペットファイル

```css:__specific-class-note.css
/* yaml にて cssclass: publish を指定したノートの背景を変更 */
.theme-dark,
.theme-light {
  --sdf-specific-note-bgcolor-basic: var(--background-primary);
}
.theme-dark {
  --sdf-specific-note-bgcolor-publish: hsl(350, 50%, 30%);
  --sdf-specific-note-bgcolor-zenn: hsl(188, 50%, 30%);
  --sdf-specific-note-bgcolor-hugo: hsl(19, 50%, 34%);
  --sdf-specific-note-bgcolor-blog-deno: hsl(32, 55%, 47%);
}
.theme-light {
  --sdf-specific-note-bgcolor-publish: hsl(350, 80%, 80%);
  --sdf-specific-note-bgcolor-zenn: hsl(188, 80%, 80%);
  --sdf-specific-note-bgcolor-hugo: hsl(19, 80%, 80%);
  --sdf-specific-note-bgcolor-blog-deno: hsl(32, 55%, 80%);
}

.workspace-split.mod-root .view-content .publish {
  background: linear-gradient(
    var(--sdf-specific-note-bgcolor-basic),
    var(--sdf-specific-note-bgcolor-publish)
  );
}
.workspace-split.mod-root .view-content .zenn {
  background: linear-gradient(
    var(--sdf-specific-note-bgcolor-basic),
    var(--sdf-specific-note-bgcolor-zenn)
  );
}
.workspace-split.mod-root .view-content .hugo {
  background: linear-gradient(
    var(--sdf-specific-note-bgcolor-basic),
    var(--sdf-specific-note-bgcolor-hugo)
  );
}
.workspace-split.mod-root .view-content .blog-deno {
  background: linear-gradient(
    var(--sdf-specific-note-bgcolor-basic),
    var(--sdf-specific-note-bgcolor-blog-deno)
  );
}
```
:::

## 引用と埋め込み

これは記事の執筆とはあまり関係ないですが、重要なスタイリングではあると思うので紹介しておきます。

現在の最新のデフォルトスタイルでは、引用ブロックと埋め込みブロックが同じスタイルで表現されてしまっています。これを見分けやすいようにカラーを調整しておきましょう。

これは単純に埋め込みノートのボーダーカラー側を変更しておけば済みます。

```css:__embed-md.css
/* 埋め込みノートのボーダーカラー */
.markdown-embed {
  border-left-color: darkviolet;
}
```

画像のように引用と埋め込みでボーダーカラーが異なるようになりました。

![img](/images/oz/img_oz-quote-embed-diff.jpg)
