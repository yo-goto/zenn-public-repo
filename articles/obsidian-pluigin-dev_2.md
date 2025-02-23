---
title: "ゼロから始めるObsidianプラグイン開発-02"
published: true
cssclass: zenn
emoji: "🕊"
type: "tech"
topics: ["typescript", "obsidian", "npm", "プラグイン"]
date: 2021-10-05
modified: 2023-02-05
url: "https://zenn.dev/estra/articles/obsidian-pluigin-dev_2"
AutoNoteMover: disable
tags: " #type/zenn #obsidian/plugin "
aliases: ゼロから始めるObsidianプラグイン開発-02
---

# はじめに

こちらは [Obsidian October 2021](https://forum.obsidian.md/t/obsidian-october-2021-make-plugins-and-themes-together-and-win-awards/24471) 記念用のプラグイン開発記事 2 回目です。[前回](https://zenn.dev/estra/articles/obsidian-dev-plugin_1) の記事ではサンプルプラグインの clone から npm を使って node_modules をインストールして、ビルドするとこまで解説しました。

## 今回の記事の内容

今回はサンプルプラグインの中身、基本的な構造について解説してきたいと思います。具体的には、ソースコード `main.ts`, `styles.css`, `manifest.json` の 3 つとプラグインのビルドに必要な他ファイルの中身を説明してきたいと思います。

:::message alert
筆者は TypeScript や開発初心者の観点から書いていますので、間違った箇所や勘違いしている箇所があるかもしれません。気づいたらコメントで指摘していただけるとありがたいです。
:::

:::message
実はサンプルプラグインや他のコミュニティプラグインのソースコードをみれば、JavaScript や TypeScript のことをちゃんと知らなくてもつくれてしまうので、あまり恐れずに挑戦してみてください。コードの細かい点については開発しながら学べば十分です。
:::

## 環境

開発を行っているのは M1 チップの macOS Big Sur v11.3 です。windows や linux 版については解説しませんので注意してください。

- obsidian API: v0.12.0
- Obsidian Sample Plugin: commit hash c228a70

## リソース

この連載記事は初心者の観点からプラグイン開発について書いています。より詳しいリソースは次のドキュメントや記事を参考にすることをおすすめします。

https://marcus.se.net/obsidian-plugin-docs/

https://forum.obsidian.md/t/plugins-mini-faq/7737

https://phibr0.medium.com/how-to-create-your-own-obsidian-plugin-53f2d5d44046

実際の開発を行う際に [Marcus Olsson](https://www.buymeacoffee.com/marcusolsson/) の非公式プラグイン開発ドキュメントを参考にするのをおすすめします。彼は Obsidian October の開発メンターです。

公式の開発途中である API 用の `obsidian.d.ts` から API リファレンスをドキュメントとして作成したツワモノです。彼のドキュメントには React や Svelte などのガイドも載っています。リポジトリが開かれているので興味があればコミットしてみてもいいかもしれません。

https://marcus.se.net/obsidian-plugin-docs/api

[Github Repo](https://github.com/marcusolsson/obsidian-plugin-docs)

# サンプルプラグインの構造

この章で理解すべき項目は大きく 2 つです。
- ディレクトリ構造
- ファイル構造

まずディレクトリ内にどんなファイルが存在し、それらがどんな役割をしているかを知る必要があります。前回の記事で紹介しましたが、おさらいします。

## ディレクトリ構造

サンプルプラグインのディレクトリ構造は、前回説明したとおり、以下のファイルからなります。

:::message alert
※ 前回の記事を踏まえて `npm run dev` でビルドした状態のフォルダとなっています。
:::

ファイル名 | 役割 |
---|---
`.gitignore` | git 監視しないアイテムを記述する (ビルドした `main.js` など)
`README.md` | コミュニティリリース時の説明ドキュメントとなる
`main.ts` | プラグインのメインプログラムファイルでソースコード
`main.js` | プラグインのメインプログラムファイルでリリースとして配布される
`manifest.json` | プラグインのメタ情報 (作者やバージョン情報など) が記載されている
`package.json` | node モジュールの依存や npm の script などを記載
`package-lock.json` | `npm install` で実際にインストールされたパッケージ情報が記載されている
`rollup.config.js` | [Rollup](https://rollupjs.org/guide/en/) 用の設定ファイル
`styles.css` | プラグイン用のスタイル (カスタム CSS と考えれば良い)
`tsconfig.json` | `main.ts`(Typescript) を `main.js`(JavaScript) にコンパイルするための設定ファイル
`versions.json` | プラグインのバージョン情報
`node_modules` | `npm i` コマンドでローカルインストールされた node のパッケージ

## リリースとして配布されるファイル

サンプルプラグイン自体はコミュニティプラグインとしてリリースされているわけではないのでコミュニティプラグインの項目からインストールすることはできませんが、実際に自分がプラグインを開発してリリースする際には、リポジトリの上記の他ファイルはインストールしたユーザーには配布されません。

Obsidian の設定画面のコミュニティプラグインの項目からプラグインをインストールすると `.obsidian/plugins/` にインストールしたプラグイン名のプラグイン用フォルダが作成されます。例えば、僕の作った Card View Mode というプラグインなら `obsidian-card-view-mode` というフォルダが作成されます。インストールによってこのフォルダには Github のリリースとして作成したビルドずみの `main.js` と `styles.css`、`manifest.json` という 3 つのファイルが配置されます。配布できるのはこれら 3 つのファイルだけです。

## main.ts の構造概要

`main.ts` ファイルの構造は以下の通りです。class 定義の `{}` 内部は省略してあります。

```ts:main.tsファイルの中身
// (1):obsidian.d.tsからモジュールのインポート
import { App, Modal, Notice, Plugin, PluginSettingTab, Setting } from 'obsidian';
// (2) 設定用のインターフェースを作成
interface MyPluginSettings {
	mySetting: string;
}
// (3) 設定用のデフォルト値を作成
const DEFAULT_SETTINGS: MyPluginSettings = {
	mySetting: 'default'
}
// (4):プラグインの基本機能を作成する
export default class MyPlugin extends Plugin {...}
// (5):モーダルを作成する
class SampleModal extends Modal {...}
// (6):設定画面を構築する
class SampleSettingTab extends PluginSettingTab {...}
```

:::message
MyPlugin- や Sample- がついているのはユーザー定義のクラスなどで、自分が開発を行うときはこれらの名前は自由に決めることができます。
:::

やっていることは以下の 6 つです。

- (1):obsidian.d.ts からモジュールのインポート
- (2): 設定用のインターフェースを作成
- (3): 設定用のデフォルト値を作成
- (4): プラグインの基本機能を作成
- (5):[モーダル](https://ja.wikipedia.org/wiki/%E3%83%A2%E3%83%BC%E3%83%80%E3%83%AB%E3%82%A6%E3%82%A3%E3%83%B3%E3%83%89%E3%82%A6)(子ウィンドウ) を作成
- (6): 設定画面を構築

プラグインで特に重要なのは (4) の基本機能の実装部分です。それ以外の部分についてはだいたい同じようにつくれば簡単なものなら作ることができます。

:::message
キーワード `import` などの詳しい機能や使い方を知りたい場合には「キーワード JavaScript」、「キーワード TypeScript」などで調べてみましょう。mozilla にドキュメントなどがあります。
- `import`: https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/import
:::

### ソースコードの見方

Obsidian に API ドキュメントと呼べるものがいまだありません。なので基本的にはサンプルプラグインから作り方の基本を学び、他のコミュニティプラグインでの実装のやり方を参考にしながら自分の作り上げたいものを開発するという流れになります。

そこで重要なのが他のプラグインのソースコードを見ることですが、初心者の人がいきなりすべてを理解するというのは無理な話で、僕自身も最初まったくわかりませんでした。

ここで `main.ts` を見ながら、プラグイン開発をやってみて工夫した方法を紹介したいと思います。

:::message
素人が独自にやっている方法なので、もっと良い方法を知っている方はコメントで教えてください👍
:::

TypeScript ではクラスを使用することができるので、一番上の階層から見ていきます。VSCode などが使えれば、画像のようにクラスの部分や処理のブロックで **折り畳む** ことができます。実際にサンプルプラグインの `main.ts` の中身を開いてコードブロックを折り畳んでみてみると下の画像のようになります。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_21.jpg)

このようにして上の構造 (コードブロックの上層) から徐々になにをやっているかということを特定していくことで少しずつ理解度を上げていきます。`main.ts` も最初開いた時点では何をやっているのか圧倒されてしまうと思いますが、このように折り畳んであげればたった 6 つのブロックしかないので恐れずに解析していけます。

クラスがわかったら、次はクラスの中の構造で何をしているのかを突き止めます。クラス内のメソッドを折りたたんでメソッドの名前を把握していきます。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_22.jpg)

例えば、`MyPlugin` のクラス内を見てみると定義されている method は `onload()` と `onunload()`、`loadSettings()`、`saveSettings()` のたった 4 つしかありません。メソッドの名前からして

- `onload()`: ロードしたときになにかする
- `onunload()`: アンロードしたときになにかする
- `loadSettings()`: 設定をロードする
- `saveSettings()` : 設定を保存する。

の 4 つの機能だろうと予測します。予測したら、メソッドを展開して中を見てきます。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_23.jpg)

ここでも文を折りたたんで名前をまずは把握します。`onlodoad()` の中身は以下で、名前から機能を予測します。

- `console.log(...);`: コンソールにログ出力する
- `this.loadSettings();` : 設定のロード
- `this.addRibbontIcon(...);`: RibbonIcon を追加する
- `this.addStatusBarItem().setText(...);`: StatusBar アイテムを追加してテキストをセットする
- `this.addCommand(...);`: コマンドを追加する
- `this.addSettingTab(...);` : 設定タブを追加
- `this.registerCodeMirror(...);` : CodeMirror の登録
- `this.registerDomEvent(...);` : Dom イベントを登録
- `this.registerInterval(...);` : インターバルを登録

だいたいの予測ができたら、さらに下の階層を展開して中身を見てきます。このようにして徐々に階層をもぐってコードをみていくとソースコードに圧倒されずに理解していくことができます。

また、`addCommand()` などのわかりやすい名前は Obsidian が公式に提供している API で実装されているためにこのような名前になっています。例えば VSCode を使っていれば、`addCommand` の部分を選択し右クリックするとメニューが出てきますが、そのメニューに「定義へ移動」という項目があります。この項目をクリックするとそれが定義された箇所にとぶことができます。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_14.jpg)

実際に定義にとんでみると、次のように obsidian パッケージの `obsidian.d.ts` ファイルが開き、モジュール内部に API として定義されていることがわかります。

:::message
`npm i` コマンドによってダウンロードされた npm_modules フォルダの中には `obsidian` という名前のパッケージが入っており、その中に API 用のモジュールとして `obsidian.d.ts` が入っています。`.d.ts` 拡張子を持つファイルは **型定義ファイル** と呼ばれています。
:::

これによって `addCommand()` は引数として Command 型、仮値として Command 型を返すということがわかります。また、ドキュメントとして「グローバルにコマンドを登録して、コマンド id とコマンド名にはこのプラウドの id と名前が自動的にプレフィックスされる」というような内容がかかれていることがわかります。

```ts:obsidian.d.tsで定義された箇所
    /**
     * Register a command globally. The command id and name will be automatically prefixed with this plugin's id and name.
     * @public
     */
    addCommand(command: Command): Command;
```

API をすべて読むというのはいきなりできないので、こうやって出会ったものについてはちくいち読んでいき、なれたら園周辺や上の階層を探っていくのがよいと思います。また、API はまだ開発の初期段階なのでドキュメント自体も整備が進んでない部分が多いです。開発者 lishid によるこういったコメントで内容を把握します。

さて、`this.addCommand();` は文なので、これでようやくこの `main.ts` ファイルの一番下の階層にくることができました。では実際に `addCommand()` はどうやってサンプルプラグインで使っているのかを見てみましょう。

```ts
this.addCommand({
	id: 'open-sample-modal',
	name: 'Open Sample Modal',
	// callback: () => {
	// 	console.log('Simple Callback');
	// },
	checkCallback: (checking: boolean) => {
		let leaf = this.app.workspace.activeLeaf;
		if (leaf) {
			if (!checking) {
				new SampleModal(this.app).open();
			}
			return true;
		}
		return false;
	}
});
```

ここで何を見ればいいかというと「どんなコマンドを追加しているか」、「自分がプラグイン開発するときにどこの箇所を入れ替えればいいか」という２点です。

どんなコマンドかということについては、`new SampleModal(this.app).open();` とあるので多分モーダル (子ウィンドウ) を開くというコマンドなのかな、という予測が働くかもしれません。

ここで `SampleModal` の定義を先程と同じように VSCode の「定義へ移動」を使ってみてみると、`obsidian.d.ts` ではなく、このファイル `main.ts` の下の部分で定義されていることがわかります。一番最初にみた上の階層で定義されているクラスであることがわかります。

「ああ、この箇所で使うものを定義していたのか」ということがわかりました。逆に画像のように `SampleModal` を選択、右クリックしてメニューから「参照へ移動」という項目をクリックすると `addCommand` のところへ戻ることができます。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_14.jpg)

![](/images/obsidian-plugin-dev_2/img_obpldev-2_16.jpg)

このようにして定義や参照などをしらべていくことでソースコードへの解像度があがってきます。しかし、こういった機能をもつようなエディタがないとソースコードを読むのは苦しいでしょう。便利ですね。こういった機能を使って

- 定義
- 型定義
- 参照
- 実装

などを把握していきます。また、ドキュメントがあるものについては読み、実際に自分で使用する際には引数や戻値などについても詳細に把握します。

:::message
サンプルプラグインでは TypeScript ファイルが `main.ts` のみとなりますが、他のコミュニティプラグインではモジュールとして分割して作成しているものが多い (例えば、`settings.ts` など) ので、他のプラグインを見る際には定義が他のファイルでされていたり、参照がファイルをまたいでいることがあるので気をつけてください。
:::

さて、SampleModal の実態にせまっていきます。

```ts
class SampleModal extends Modal {
	constructor(app: App) {
		super(app);
	}

	onOpen() {
		let {contentEl} = this;
		contentEl.setText('Woah!');
	}

	onClose() {
		let {contentEl} = this;
		contentEl.empty();
	}
}
```

ここでやっていることは「Modal というクラスの継承クラスとしての SampleModal の定義」です。クラスの継承についての詳細ははぶきますが、API で定義されている Modal というクラスとそのメソッドをフォーマットとして利用して自分でつかえるようにしています。SampleModal の中にある `onOpen()` と `onClose()` も基本的な使い方は API で定義されており、おなじみに「定義へ移動」を使うと `obsidian.d.ts` でどちらも Modal クラスのメソッドであり、戻り値は void 型として定義されていることがわかります。

```ts:obsidian.d.tsでのModalクラス
/**
 * @public
 */
export class Modal implements CloseableComponent {
    /**
     * @public
     */
    app: App;
    /**
     * @public
     */
    scope: Scope;
    /**
     * @public
     */
    containerEl: HTMLElement;
    /**
     * @public
     */
    modalEl: HTMLElement;
    /**
     * @public
     */
    titleEl: HTMLElement;
    /**
     * @public
     */
    contentEl: HTMLElement;
    
    /**
     * @public
     */
    shouldRestoreSelection: boolean;
    
    /**
     * @public
     */
    constructor(app: App);
    /**
     * @public
     */
    open(): void;
    /**
     * @public
     */
    close(): void;
    /**
     * @public
     */
    onOpen(): void;
    /**
     * @public
     */
    onClose(): void;

}
```

メソッドの名前から推測するに、モーダルが開いたときの処理、モーダルが閉じた時の処理であろうということがわかるかもしれません。さて、予測だけだと実際にこのプラグインがどのような機能を提供しているのかはわかりませんので、Sample Plugin を実際に使ってみましょう。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_19.jpg)

コマンドパレットを開いて「Sample Plugin」と入力すると「Sample Plugin: Open Sample Modal」コマンドがサジェストされるのでそれを実行します。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_20.gif)

小さいウィンドウが開き「Woah!」というテキストが表示されました。予想通り単純にモーダルを開くコマンドでした。もういちど `addCommand()` の部分を見てみると

```ts
this.addCommand({
	id: 'open-sample-modal',
	name: 'Open Sample Modal',
	// callback: () => {
	// 	console.log('Simple Callback');
	// },
	checkCallback: (checking: boolean) => {
		let leaf = this.app.workspace.activeLeaf;
		if (leaf) {
			if (!checking) {
				new SampleModal(this.app).open();
			}
			return true;
		}
		return false;
	}
});
```

name のところに 'Open Sample Modal' とあります。これが多分コマンドの名前なのだろうなということが推測できます。実際に名前であることを確認するためにこの部分だけ書き換えます。

その前に失敗してもよいように git でブランチを切っておきます。ターミナルでこのリポジトリを開いて

```shell
git checkout -b test
# testという名前のブランチを作成してチェックアウトします
```

これでなにかやらかしても元の master ブランチに戻れば元の状態に復元できます。

では、この状態から `addCommand()` の中身について id と name だけ変えてみます。

```ts
this.addCommand({
	id: 'open-test',
	name: 'Open Test',
	// callback: () => {
	// 	console.log('Simple Callback');
	// },
	checkCallback: (checking: boolean) => {
		let leaf = this.app.workspace.activeLeaf;
		if (leaf) {
			if (!checking) {
				new SampleModal(this.app).open();
			}
			return true;
		}
		return false;
	}
});
````

おまけに SampleModal クラスの方もいじってみます。

```ts
class SampleModal extends Modal {
	constructor(app: App) {
		super(app);
	}

	onOpen() {
		let {contentEl} = this;
		contentEl.setText('モーダルの表示テストです');
	}

	onClose() {
		let {contentEl} = this;
		contentEl.empty();
	}
}	
```

保存したら、再びターミナルからビルドします

```shell
npm run dev
```

コンパイル成功後、プラグインを再読み込みし Sample Plugin のコマンドを実行します。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_24.gif)

画像のようにコマンド名とモーダルでの表示テキストを変更できました。このように、どういう挙動をするのか知りたい場合には色々なところをいじってみて実際に動かしてみるのがわかりやすいです。

## main.ts の構造詳細

では、main.ts の構造についてより詳細に説明していきます。もう一度全体を見渡します。

```ts:main.tsファイルの中身
// (1):obsidian.d.tsからモジュールのインポート
import { App, Modal, Notice, Plugin, PluginSettingTab, Setting } from 'obsidian';
// (2) 設定用のインターフェースを作成
interface MyPluginSettings {
	mySetting: string;
}
// (3) 設定用のデフォルト値を作成
const DEFAULT_SETTINGS: MyPluginSettings = {
	mySetting: 'default'
}
// (4):プラグインの基本機能を実装する
export default class MyPlugin extends Plugin {...}
// (5):モーダルを作成する
class SampleModal extends Modal {...}
// (6):設定画面を構築する
class SampleSettingTab extends PluginSettingTab {...}
```

### (1) obsidian.d.ts からクラスのインポート

この (1) はお約束みたいなもので、Notice や Modal 以外は基本的に使うクラスを `obsidian` パッケージからインポートしています。App や Plugin などはそれぞれ VSCode でおなじみの「定義へ移動」で API の型情報を見ることができます。

自分でプラグイン開発を行う際には、`obsidian.d.ts` から必要なクラスなどを追加で見つけて自分で `import { ~ }` のところに書く必要があります。

### (2)(3) 設定用の型と定数をつくる

(2) と (3) はセットで、設定用の変数に対応してデフォルト値を与えます。サンプルプラグインでは設定用の変数が一つしかありません。

```ts
// (2) 設定用のインターフェースを作成
interface MyPluginSettings {
	mySetting: string;
}
// (3) 設定用のデフォルト値を作成
const DEFAULT_SETTINGS: MyPluginSettings = {
	mySetting: 'default'
}
```

ここでプラグインの設定に利用するインターフェースとデフォルト値をためしに作成します。ちょっと設定を増やそうと思って書き換えてみると以下のようになります。

```ts
interface HogehogeSettings {
	hogehogeSetting1: string;
	hogehogeSetting2: number;
	hogehogeSetting3: boolean;
}

const DEFAULT_SETTINGS: HogehogeSettings = {
	hogehogeSetting1: 'string setting',
	hogehogeSetting2: 10,
	hogehogeSetting3: false
}
```

ここの書き方もお約束みたいなものなので、設定項目を増やしたい場合にはこのようインターフェースの方には設定値の型を記載し、デフォルト値の方にはその型に合わせた値を入れておきます。この `interface` と `const DEFAULT_SETTINGS` はセットで覚えてください。`const DEFAULT_SETTINGS` の後には `interface` でつけたインターフェースの名前 `HogehogeSettings` を型として指定します。

### (6) 設定タブの構築

順番は前後しますが、(6) を見ていきます。まずコードの概略ですが、`display()` メソッドは中身を省略してこのようになっています。ちなみに以下のコードはほぼテンプレートとなり、自分で開発する際に書き換える部分はプラグイン名と `display()` の中身になります。

```ts
class SampleSettingTab extends PluginSettingTab {
	plugin: MyPlugin;
	constructor(app: App, plugin: MyPlugin) {
		super(app, plugin);
		this.plugin = plugin;
	}
	display(): void {...}
}
```

TypeScript ではクラスをキーワードとして使うことができます。クラスの宣言の仕方は以下のような形になります。

```ts
class クラス名 {
	メンバ1: 型;
	メンバ2: 型;
	constructor(メンバ1: 型, メンバ2: 型) {
		初期化処理
	} 
	メソッド1(引数: 型){
		処理
	}
}
```

参考
https://typescript-jp.gitbook.io/deep-dive/future-javascript/classes

API で定義されたクラスを雛形にして自分が開発するプラグイン用のクラスを作成したいので、クラスの継承を行います。

クラス継承の際には `extends` キーワードを使って継承するクラスを指定します。

```ts
class クラス名 extends 継承するクラス名{
	メンバ3: 型;
	constructor(メンバ1: 型, メンバ2: 型, メンバ3: 型){
		super(メンバ1, メンバ2);
		this.メンバ3 = メンバ3;
	}
	メソッド2(引数: 型){...}
}
```

さて、設定タブの構築では obsidian のパッケージからインポートしたクラス `PluginSettingTab` を継承して自分のプラグイン用の設定タブのためのクラスを作成します。次のように利用します。

```ts
class HogehogeSettingTab extends PluginSettingTab {
	plugin: HogehogePlugin;
	constructor(app: App, plugin: HogehogePlugin) {
		super(app, plugin);
		this.plugin = plugin;
	}
	display(): void {...}
}
```

`display()` メソッドについては次のようになっています。`display()` は設定タブの画面上に表示されるテキストや処理などを担当しています。

```ts
display(): void {
	let {containerEl} = this;

	containerEl.empty();

	containerEl.createEl('h2', {text: 'Settings for my awesome plugin.'});

	new Setting(containerEl)
		.setName('Setting #1')
		.setDesc('It\'s a secret')
		.addText(text => text
			.setPlaceholder('Enter your secret')
			.setValue('')
			.onChange(async (value) => {
				console.log('Secret: ' + value);
				this.plugin.settings.mySetting = value;
				await this.plugin.saveSettings();
			}));
}
```

ちなみに `obsidian.d.ts` での PluginsSettingTab の型定義は以下の通り。

```ts
/**
 * @public
 */
export abstract class PluginSettingTab extends SettingTab {
    
    /**
     * @public
     */
    constructor(app: App, plugin: Plugin_2);
}
```

`PluginSettingTab` クラスは `SettingTab` クラスのサブクラスなのでさらに `obsidian.d.ts` の `SettingTab` クラスを見てみると次のようになっています。

```ts
/**
 * @public
 */
export abstract class SettingTab {

    /**
     * @public
     */
    app: App;
    
    /**
     * @public
     */
    containerEl: HTMLElement;

    /**
     * @public
     */
    abstract display(): any;
    /**
     * @public
     */
    hide(): any;
}
```

`display()` は `abstract` キーワードがついているため、これは抽象メンバであり `SampleSettingTab` クラスでは実装する必要があります。SamplePlugin での実際の実装は以下の通りです。

```ts
	display(): void {
		let {containerEl} = this;

		containerEl.empty();

		containerEl.createEl('h2', {text: 'Settings for my awesome plugin.'});

		new Setting(containerEl)
			.setName('Setting #1')
			.setDesc('It\'s a secret')
			.addText(text => text
				.setPlaceholder('Enter your secret')
				.setValue('')
				.onChange(async (value) => {
					console.log('Secret: ' + value);
					this.plugin.settings.mySetting = value;
					await this.plugin.saveSettings();
				}));
	}
```

上から 4 行目ぐらいまではテンプレートです。HTML 要素を作成して、画面上にテキストによる説明を表示します。ちょっと説明すると

```ts
let { containerEl } = this;
```

この一番最初の部分ですがオブジェクトの分割代入と呼ばれるもので、やっていることとしては下のコードとほぼ同じです。

```ts
let containerEl = this.contentEl;
```

詳しくは以下のドキュメントによくまとまっています。
https://ja.javascript.info/destructuring-assignment

`containerEl` は抽象クラス `SettingTab` を見てみると内側で定義されていたプロパティです。`containerEl: HTMLElement;` と書いてあるので型は HTMLElement です。

実は API のいくつかのコンポーネントで `containerEl` や `contentEl` などのプロパティがあります。これらの型は `HTMLElement` であり、画面構成や UI、後で解説するモーダルの作成にも使用します。`HTMLElement` の親の親のインターフェースである `Node` には `empty()` メソッドがあります。初期化的なメソッドです。画面構築などをする際にはこのメソッドをとりあえず使用します。

```ts
containerEl.empty();
```

さて、`containerEl` の初期化が終わったらここに色々な要素をつけて画面をつくります。

```js
containerEl.createEl('h2', {text: 'Settings for my awesome plugin.'});
```

`createEl()` メソッドは HTML 要素を作成するメソッドで引数についてはいつもと同じように「定義へ移動」で `obsidian.d.ts` を見ます。ですが、見なくても次のようにエディタでサジェストしてくれるのこれを見ればよいですね。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_27.jpg)

使い方はこういったものをちまちま見ていきます。今回は以下のように HTML の `<h2>` 要素を作成して `Settings for my awesome plugin.` というテキストを表示させています。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_25.jpg)

そして重要な一番最後の部分となります。

```ts
new Setting(containerEl)
	.setName('Setting #1')
	.setDesc('It\'s a secret')
	.addText(text => text
		.setPlaceholder('Enter your secret')
		.setValue('')
		.onChange(async (value) => {
			console.log('Secret: ' + value);
			this.plugin.settings.mySetting = value;
			await this.plugin.saveSettings();
		}));
```

`Setting` は `obsidian.d.ts` からインポートしてきたクラスで設定要素を作成するために使うクラスです。設定項目が増えるたびに `new Setting(containerEl)` でインスタンス化して設定画面の要素を増やしてあげます。

サンプルプラグインでは設定画面上に暗号を入力して保存するということをやるためのコードがかかれています。

`Setting` の型定義を `obsidian.d.ts` に見に行くとメソッドの戻り値の型はすべて `this` となっていることがわかります。

```ts
    /**
     * @public
     */
    constructor(containerEl: HTMLElement);
    /**
     * @public
     */
    setName(name: string | DocumentFragment): this;
    /**
     * @public
     */
    setDesc(desc: string | DocumentFragment): this;
    /**
     * @public
     */
    setClass(cls: string): this;
    /**
     * @public
     */
    setTooltip(tooltip: string): this;
```

メソッド内の this なので `new Setting(containerEl)` のあとに連続でドットを使って各メソッドを使うことができます。サンプルプラグインの書き方を少し変えると以下のコードになります。

```ts
new Setting(containerEl).setName('Setting #1').setDesc('It\'s a secret').addText(...);
// addText()の中身は省略												 
```

サンプルプラグインでは見やすいようにわざわざ改行していますが、`this` でかえってくるインスタンスに連続でアクセスしてメソッドを使っているだけです。

`addText()` メソッドの中ではアロー関数の省略形が使われています。詳しくはいまは覚える必要はなく、テンプレートのように部分部分を書き換えることで自分のつくりたい設定を作成します。

```ts
.addText(text => text
	.setPlaceholder('画面の入力項目に表示されるテキスト')
	.setValue('')
	.onChange(async (value) => {
		console.log('コンソールに表示されるテキスト' + value);
	// valueは入力した値
		this.plugin.settings.mySetting = value;
		await this.plugin.saveSettings();
	}))
```

最後の `this.plugin.settings.mySetting = value` の `mySetting` は (2)(3) の項目で作成したプロパティ名を入れてあります。そして、`await this.pluin.saveSetting();` で入力した値を設定値として保存します。

`addText()` メソッドは値を入力できるテキストボックスを作成してくれます。もしドロップダウンリストなどを作成したい場合には `obsidian.d.ts` に型定義してある `Setting` のメソッドを探してみてください。ドロップダウンリストは `addDropdown()` というメソッドが用意されているので、`addText()` の部分をこれに書き換えることで欲しい画面設定要素をつくることができます。

### (5) モーダルの作成

(6) の内容を踏まえると (5) は結構簡単に理解できると思います。デモで見せたようにコマンドで開くモーダルウィンドウを作成したいので、`obsidian.d.ts` で型定義された Modal クラスを継承させた `SampleModal` クラスを作成します。この `Modal` クラスでもともと使えるメソッドはモーダルを開いたときの処理を行う `onOpen()` やモーダルを閉じたときの処理を行う `onClose()` などです。

```ts
class SampleModal extends Modal {
	constructor(app: App) {
		super(app);
	}

	onOpen() {
		let {contentEl} = this;
		contentEl.setText('Woah!');
	}

	onClose() {
		let {contentEl} = this;
		contentEl.empty();
	}
}
```

この 2 つのメソッドも Modal クラスの型定義の部分にかかれているものです。実際にモーダルを開いたとき・閉じたときにどんな処理をするかというのは定義されていませんので、自分で上のように定義します。(6) で説明したように `let {containerEl} = this;` で分割代入してあげて表示項目をプロパティ `contentEl` に作成してあげます。

### (4) プラグインの基本機能

プラグイン構築に欠かせないのが、Plugin クラスを継承したこの `MyPlugin` クラスです。(実際に開発する際には名前は自分のプラグインの名前にしてください)

これもテンプレートとして考えてください。

```ts
export dafult class MyPlugin extends Plugin {
	// プロパティの型は自分で作成したMyPluginSettings
	settging: MyPluginSettings;
	// ロード時処理
	async onload(){...}
	// アンロード時処理
	onunload(){...}
	// 設定のロード
	async loadSettings(){
		this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());
	}
	// 設定の保存
	async saveSettings(){
		await this.saveData(this.settings);
	}
}
```

:::message
この `Plugin` クラスのサブクラスである `MyPlugin` クラスには `export dafult` をつけます。
:::

基本的にはロード時の処理がメインで書く部分となります。詳細については「ソースコードの見方」で解説したので省きます。同じようなことをやりたい場合にはそのままコードを真似ることで可能です。

### main.js のまとめ

それでは、`main.ts` について最低限必要なテンプレートとしてまとめ直してみると以下のようになります。

```ts
// (A):obsidian.d.tsから最低限必要なモジュールのインポート
import { App, Plugin, PluginSettingTab, Setting } from 'obsidian';

// (B) 設定用のインターフェースを作成
interface MyPluginSettings {...}
// (C) 設定用のデフォルト値を作成
const DEFAULT_SETTINGS: MyPluginSettings = {...}
											
// (D):プラグインの基本機能を実装する
export default class MyPlugin extends Plugin {...}

// (E):設定画面を構築する(設定画面がいらないならつけない)
class SampleSettingTab extends PluginSettingTab {...}
```

ちなみに設定画面がいらなければ (E) の部分もいりません。

## styles.css

`styles.css` はプラグインを設定から有効化すると読み込まれるスタイルファイルで、カスタム CSS スニペットと同じように考えてもらえばよいです。サンプルプラグインでやっているのは単純に文字の色を赤に変更するというものです。`styles.css` を覗いてみると次のコードしかかれていません。

```css
body {
    color: red;
}
```

サンプルプラグインではカスタム CSS スニペットで簡単に再現できるようなものですが、プラグイン開発ではコマンドを通じてスタイルを変更したり設定画面でスタイルの設定などを変更できるようにすることができます。またプラグインで作った要素やモーダルなどを構築する際に使えます。他のコミュニティプラグインを見てみるとよいと思います。

## manifest.json

manifeset.json ですが、これもプラグインのコード配布に必要なファイルとなります。内容としては、このプラグインの作成者の名前やプラグインフォルダの名前となる id やこのプラグインを使うことのできる Obsidian の最小バージョンなどプラグインとしての体裁を保つのに必要な情報です。サンプルプラグインの `manifest.json` には以下が書かれています。

```json
{
	"id": "obsidian-sample-plugin",
	"name": "Sample Plugin",
	"version": "1.0.1",
	"minAppVersion": "0.9.12",
	"description": "This is a sample plugin for Obsidian. This plugin demonstrates some of the capabilities of the Obsidian API.",
	"author": "Obsidian",
	"authorUrl": "https://obsidian.md/about",
	"isDesktopOnly": false
}
```

自分で開発する際には `:` で区切られた部分の右側を自分用に書き換えます。具体的な項目の説明は `node_modules/obsidian` のフォルダにある `README.md` に実は書かれています。

- `id` : プラグインの ID です。プラグインフォルダの名前になります。
- `name` : プラグインの名称です。コミュニティプラグインの項目に表示される名前です。
- `author` : プラグイン開発者の名前です。実名でもペンネームでもどちらでも OK。
- `version` : プラグインのバージョン番号です。Github のリリースに合わせてつけます。
- `minAppVersion` : このプラグインが動く Obsidian の最小バージョンの番号です。
- `description` : プラグインが何をするのか説明する文章を書きます。
- `authorUrl` : 開発者のサイトの URL などを書きます。(これはオプションなのでなくても OK)
- `isDesktopOnly` : Desktop 版のみで使えるようにするかどうかを選択します。

`manifest.json` でこれらの項目を埋めてリリースします。

## JavaScript と TypeScript

初心者の視点からソースコードを見ていると、TypeScript 特有の問題なのかどうかわからない場合が多くあります。そういった場合には `main.ts` と `main.js` を見比べてみるとよいです。コンパイルによって書き方が変わっていたら TypeScript の問題として検索などができます。書き方が変わっていなかったら JavaScript の問題として扱いましょう。

たとえば、途中ででてきた「オブジェクトの分割代入」ですが、`main.ts` でも `main.js` でもまったく同じ用に書かれていることがわかります。したがって TypeScript の特殊な書き方ではなく JavaScript で存在する書き方ということがわかります。

![](/images/obsidian-plugin-dev_2/img_obpldev-2_26.jpg)

### コーディングとソースコード解析用リソース

ソースコードを読み解いたりコーディングするのには以下 4 つのドキュメントがおすすめです。

JavaScript 用リソース
https://jsprimer.net

https://ja.javascript.info

TypeScript 用リソース
https://typescript-jp.gitbook.io/deep-dive/

https://book.yyts.org

# 終わり

今回はサンプルプラグインの構造について解説しました。次回は実際の開発のやり方とミュニティリリースを行うところまで解説します。
