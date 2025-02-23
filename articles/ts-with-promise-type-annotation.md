---
title: TypeScript の基礎から Promise の型注釈まで駆け登る
published: true
emoji: 🧞‍♀️
type: tech
topics:
  - javascript
  - typescript
  - deno
  - promise
date: 2022-07-14
modified: 2023-10-14
url: https://zenn.dev/estra/articles/ts-with-promise-type-annotation
tags:
  - type/zenn
  - TypeScript
aliases:
  - 記事_TypeScript の基礎から Promise の型注釈まで駆け登る
  - contextual typing
---

## はじめに

:::message alert
この記事は Zenn の Book の方で公開している『[イベントループとプロミスチェーンで学ぶ JavaScript の非同期処理](https://zenn.dev/estra/books/js-async-promise-chain-event-loop)』で収録済みのチャプターから記事として切り出したものになります (チャプター単体でも記事になるものとして判断して公開しています)。
:::

この記事は JavaScript の非同期処理を学習した人間が Promise や非同期処理を介して逆に TypeScript の型注釈を理解しようという試みの記事になります。

実は Promise の型注釈や TypeScript の非同期処理の解説については以下のように既にいくつも有用なリソースがあります。

- [Promise / async / await | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/promise-async-await)
- [イベントループと TypeScript の型から理解する非同期処理](https://zenn.dev/mizchi/articles/understanding-promise-by-ts-eventloop)

今回は初学者の目線からアウトプットを兼ねて自分なりの解釈で TypeScript についての基礎から Promise の型注釈に必要な知識まで一気に駆け上がって説明してみたいと思います (型注釈については基本的なことしか解説しませんので個別の詳細については日本語でオープンソースに公開されている『[サバイバル TypeScript](https://typescriptbook.jp/)』などを参考にしてください)。

:::details 参考ドキュメントについて
TypeScript や JavaScript については Web で使われる言語のことはあって、インターネット上でいくつも参考になるドキュメントが公開されています。自分が読んでいるもので個人的な感覚でいくつか紹介します。

- 『[JavaScript Primer](https://jsprimer.net/)』
  最新の ECMAScript 仕様に基づいて JavaScript のシンタックスを網羅的に学べるドキュメントです。JavaScript を学ぶならかなりおすすめです。
- 『[MDN Web Docs](https://developer.mozilla.org/ja/)』
  Web 開発必須のドキュメントです。JavaScript や Web API について分からないことがあったらとりあえず MDN を読むと解決します。
- 『[サバイバルTypeScript](https://typescriptbook.jp/)』
  現実的に使う際に注意する点や実務で使うコーディングなどを前提にした分かりやすい日本語の解説になっており、付随してフロントエンドで利用するツールやフレームワークなどを包括的に解説しています。TypeScript 学習の入り口となるおすすめのドキュメントです。
- 『[The TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)』
  TypeScript 公式のドキュメントです。TypeScript の機能を効率的にシンプルに知ることができます。実はめちゃくちゃわかりやすく解説されているので、英語に抵抗が無いなら絶対におすすめです (これに最近気づきました)。
- 『[TypeScript Deep Dive](https://typescript-jp.gitbook.io/deep-dive/)』
  初学者には難易度高めで抽象的な印象を受けますが、本質的な説明で比較的短く解説されています。「なるほど、そういうことね」みたいな納得感が得られます。

学習のレベルに沿ってわかりやすさが変わってくるので、今分かりづらいものでも、時間がたつとなるほどとなることが多いです。できれば全部目を通すのが良いかなと思います (いきなり頭から全部読むということではなく、どれかを起点にしてこっちのドキュメントではどう説明されているんだろうという感じで状況に応じてつまみ食いするのがいいです)。あと、公式ドキュメントが実はかなり分かりやすい構成なので英語だからと言って食わず嫌いしないで読んでみるといいと思います。

その他 Youtube などにある分かりやすい解説動画で補う形を自分は取っています。『[JSConf](https://www.youtube.com/c/JSConfEU)』などのオーソリティのあるカンファレンス動画での深堀りや『[Fireship](https://www.youtube.com/c/Fireship)』などのショート動画、日本語では『[トラハック](https://www.youtube.com/channel/UC-bOAxx-YOsviSmqh8COR0w)』さんなど分かりやすく視聴しやすいです。
:::

「**JavaScript で非同期処理を学んでから TypeScript を見れば怖くないよ**」ということを趣旨として内容を練り上げましたが、今回は前提となる内容を本で解説してしまっているため、比較がわかりにくいかもしれません。「JavaScript から TypeScript までは大した距離が無い」ということだけでも伝わると思うので、TypeScript 初学者の方や非同期処理に興味ある方は読んでてみてください。

ということで、JavaScript の基礎や非同期処理そのものについての解説は省略させてもらいます。『[イベントループとプロミスチェーンで学ぶ JavaScript の非同期処理](https://zenn.dev/estra/books/js-async-promise-chain-event-loop)』の方でかなり詳細に解説しているので興味がある方はそちらを見てください。

ちなみに環境は [Deno](https://deno.land/manual) を使います (Deno を使う理由については後述します)。

## TypeScript について

TypeScript は JavaScript に [型システム](https://www.wikiwand.com/ja/%E5%9E%8B%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0 ) を導入した言語です。

個人的な (浅い ) 経験から言うと JavaScript をしっかり学べば TypeScript は怖くありません (使いこなせるかは別の話として、怖くないと思うことが重要です)。逆に TypeScript から入ってしまうと JavaScript の機能に加えて型情報の操作や型チェックのエラーといった学ぶべき事柄が膨大になるので圧倒されてしまいます。

これは、TypeScript を理解するためには JavaScript の知識が欠かせないということでもあります。『[サバイバルTypeScript](https://typescriptbook.jp/)』でも次のように言われています。

  > TypeScrip t から見ると、JavaScrip t は TypeScrip t の一部と言えます。そのため、TypeScrip t を十分に理解するには、JavaScrip t の理解が欠かせません。ま だ JavaScrip t をよく分かっていない場合は、TypeScrip t の学習と平行し て JavaScrip t も学ぶ必要があります。
> ([JavaScriptはTypeScriptの一部 | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/overview/javascript-is-typescript) より引用)

そして JavaScript での非同期処理が理解できれば TypeScript の非同期処理はおそるるに足りません。『[TypeScript Deep Dive](https://typescript-jp.gitbook.io/deep-dive/recap)』でも次のように言われています。

  > TypeScrip t は、単に、JavaScrip t のコードを良いドキュメントにする方法を標準化したものに過ぎません。
> (中略)
> 本質的には、TypeScript は Java Script のリンター(コードの静的 解析ツール) です。型情報を 持たない他の Java Script のリンターよりも優れているだけです。
> ([JavaScript - TypeScript Deep Dive 日本語版](https://typescript-jp.gitbook.io/deep-dive/recap) より引用)

TypeScript はより良い JavaScript を書くためのリンターに過ぎません。つまり JavaScript を書くための道具です。

そして、TypeScript の非同期処理は **JavaScript の非同期処理のコードに型情報を上乗せしたもの** であり、本質的には Promise や async/await といった JavaScript(ECMAScript) の非同期シンタックスやその処理を実現するためのイベントループの機構、ランタイ ム (JS エンジン ) を埋め込んでいる環境とそこから提供される非同期 API を [理解すれば良い訳です](https://zenn.dev/estra/articles/js-async-programming-roadmap)。つまり、**「非同期処理」を理解するために必要な知識そのものと TypeScript には殆ど関係性がありません**。

私見では以下のような「型の情報操作機能」が JavaScript に追加されたものが TypeScript であると認識しています。

- 型情報の推論 (Type inference)
- 型情報の付与 (Type annotation)
- 型情報の定義 (Type defining)
- 型情報の合成 (Type composing)
- 型情報の主張 (Type assertion)
- 型情報の再利用 (Type reusing)
- 型情報の絞り込み (Type narrowing)

あとは型情報の操作によって副次的に追加されたコードの書き方やいくらかの演算子とキーワードなどが加わっただけで、それ以外はただの JavaScript です。図で表すと次のような関係になっています。中枢には実行環境に関わらず共通の動作を定める仕様となる ECMAScript があります。

![JSとTSの関係](/images/js-async/img_whatIsJSTS.jpg)*[JavaScript - TypeScript Deep Dive 日本語版](https://typescript-jp.gitbook.io/deep-dive/recap) を参考に図を作成*

(もちろん型の再利用や Narrowing など TypeScript に特化した難しさはありますが ) こういう恐れすぎない心持ちのもとで学習を進めていきます。非同期処理についても JavaScript から始めて型無しで学んだあとで、「より堅牢なコードを書くために TypeScript による型注釈を加えて扱うデータに対しての具象性を高めていく」という考えのもとで進めていきます。

実際、**JavaScript と TypeScript の境界線がどこにあるのかを意識することでスッキリと理解できる場合が多いです**。また、何か分からないことがでてきた場合も、問題を解決するために調べる必要のあるレイヤーがどれか分かることは非常に重要です。TypeScript についてわからないと思っていたことが実は ECMAScript のシンタックスだったり (その場合は MDN で調べる)、ECMAScript の関数が分からないと思っていたらその関数は JavaScript 実行環境が独自定義する API だったり (その場合はランタイム環境のマニュアルや API ドキュメントで調べる)、あるいは型ガード関数という TypeScript 独自の書き方で型の解析に利用するものだったり (その場合は TypeScript Handbook で調べる ) と、境界線が分かっていないと調べる領域を間違ってしまう場合があるのでかなり効率が悪くなってしまいます。

そういったことを踏まえて、JavaScript をすでに知っている学習者は TypeScript 公式ハンドブックの『TypeScript for JavaScript Programmers』の項目を読むことで JavaScript から TypeScript にする方法の概要を短い時間で学ぶことができます。TypeScript に特化した機能がなんなのか分かってしまえば、学ぶべき量がそこまで多くないことが分かります (もちろん少なくはないですが、TypeScript だけで学ぼうとする場合よりも遥かに少ないことが認識できます)。

https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html

そして朗報です。Deno では TypeScript が設定なしですぐに使えます。これが Deno を主な実行環境に選んだ理由の１つです。Node 環境であれば TypeScript そのものをローカルインストールしたり [ts-node](https://typestrong.org/ts-node/) といったコマンドラインから実行するためのツールが必要になったり、コンパイルオプションを定義する必要などがあるので、TypeScript の学習をしたい人にとって高いハードルがあるのですが、Deno がこれを解決してくれます。

JavaScript ファイルと同じようにターミナルからコマンドラインで `deno run` コマンドを実行することで TypeScript のスクリプトファイルを実行できます。

```sh:コマンドライン
# JS ファイルと同じく TS ファイルを引数にして実行できる
❯ deno run hellowold.ts
hello world!
```

これによってコマンドラインから手軽に TypeScript の実行ができるので何度でもテストできます。

さらに、Deno には備え付けのリンターがあり、そのリントルールの注意を見ることで良い TypeScript を書く訓練ができます。リントルールの詳細は次の公式ドキュメントから閲覧できます。

https://lint.deno.land

VS Code などを使っていれば Deno 専用の拡張機能を入れることでエディタ上でリンターを使えます。

https://marketplace.visualstudio.com/items?itemName=denoland.vscode-deno

そして、Deno では V8 エンジンがランタイムになっています。TypeScript は JavaScript へとトランスパイ ル (コンパイルの一種 ) を行うことで実際には JavaScript を JavaScript エンジンで動かしているに過ぎません。TypeScript でのエラーはコンパイル時の型チェックエラーと実際に JavaScript として動かした時のランタイムエラーとなります。

型チェックでエラーがでても JavaScript エンジンでランタイムエラーがでないで JavaScript として正しく動く場合もあります。JavaScript として正しく動いたとしても、型チェックで意図的に警告を出させることで、実際に動かす前によりよいコードを書くように書き直す機会を得ることができます。

## 型注釈の基本

TypeScript では既存の JavaScript コードに型の情報を付与していくことから学習が始まります。

### 変数への型注

釈
例えば、文字列リテラルの値で初期化した変数に明示的に `string` 型であると型の情報を付与することが型注釈 (type annotatio n) と呼ばれる行為です。

変数に型を注釈するには変数名の後に `:` を付けて型の名前を書きます。JavaScript のプリミティブ型である文字列型なら `string` というように決まった型の名前があるのでそれを変数名の後に追加します。

```ts
const str1 = "文字列"; // JavaScript
const str2: string = "文字列の型注釈を追加"; // TypeScript
//          ^^^^^^ string 型の型注釈
//                 str2 は string 型だよとコンパイラに伝えているだけ
```

上のコードでは文字列リテラルで初期化しているので明らかに文字列型 (`string` 型 ) であることがコンパイラは推論できるので上の場合には必ずしも書く必要がありません。

TypeScript のコンパイラは賢いので型注釈を省略してもある程度は推論してくれます。従って、次のように型注釈を省略しても TypeScript ではコードとして大丈夫です。

```ts
const str3 = "文字列リテラル"; // TypeScript
// 文字列リテラルで初期化しているのは明らかであり、型注釈は省略できる
```

型を省略してもそのコードから型を推論して自動的に型情報が得られるこの機能を型推論 (type inference ) と言います。上のような変数宣言では初期値から型が推論されます。

Deno ではこのような明らかに型推論が容易な変数宣言ではむしろ型注釈を省略するように促すリンタールール "no-inferrable-types" がありますので、省略しないと怒られてしまいます。

https://lint.deno.land/?q=infer#no-inferrable-types

  > Variable initializations to JavaScript primitives (and `null`) are obvious in their type. Specifying their type can add additional verbosity to the code. For example, with `const x: number = 5`, specifying `number` is unnecessary as it is obvious that `5` is a number.
> ([deno_lint docs no-inferrable-types](https://lint.deno.land/?q=infer#no-inferrable-types) より引用)

リンタードキュメントには記載されていますが、以下のような型注釈に警告がなされて冗長なので型注釈を省略するようにと言われます。

```ts:無効となるコード(型注釈を省略しないと怒られる)
// 値の初期化
const a: bigint = 10n;
const b: bigint = BigInt(10);
const c: boolean = true;
const d: boolean = !0;
const e: number = 10;
const f: number = Number("1");
const g: number = Infinity;
const h: number = NaN;
const i: null = null;
const j: RegExp = /a/;
const k: RegExp = RegExp("a");
const l: RegExp = new RegExp("a");
const m: string = "str";
const n: string = `str`;
const o: string = String(1);
const p: symbol = Symbol("a");
const q: undefined = undefined;
const r: undefined = void someValue;

class Foo {
  prop: number = 5;
}

// デフォルト引数を使うときも型推論が容易なので省略すべき
function fn(s: number = 5, t: boolean = true) {}
```

なるほど、変数にプリミティブ値などを代入する際にはこのように型注釈をすればよいのかということが逆に分かります。

型注釈に利用する `string` などは JavaScript の各プリミティブ型やオブジェクトの名前そのものです。基本的なものは次のようになっています。

J S の主要なデータ型 | 型注釈での名前 | 値
--|--|--
文字列 | `string`  | `"文字列"`, `'文字列'`
数値 | `number` | `42`
真偽値 | `boolean` | `true`, `false`
シンボル | `symbol` | `Symbol("シンボル")`
正規表現オブジェクト | `Regex` | `RegExp("a")`

:::message
型注釈といった「JS へ追加した TS の機能」が省略できるのは、 JavaScript をそのまま TypeScript として扱えるということも意味しています。つまり、部分的に型を厳密に書いたり、緩く書いたりできるので、かなり柔軟に型を記述していくことができるというわけです。学習プロセスに基づいて JavaScript から緩い型のある TypeScript へ、厳密な型のある型安全な TypeScript へと昇華させていくことができます。
:::

ということで基本的な値の初期化での型注釈は、以下のように省略します。

```ts:有効なコード
const a = 10n;
const b = BigInt(10);
const c = true;
const d = !0;
const e = 10;
const f = Number("1");
const g = Infinity;
const h = NaN;
const i = null;
const j = /a/;
const k = RegExp("a");
const l = new RegExp("a");
const m = "str";
const n = `str`;
const o = String(1);
const p = Symbol("a");
const q = undefined;
const r = void someValue;

class Foo {
  prop = 5;
}

function fn(s = 5, t = true) {}
```

明らかに JavaScript のままですね。TypeScript を始めた際にはこのように型推論によって型が省略できてしまうので、想像していたものよりも型を書かなくても済むことに気づきます。

プリミティブ型ではない、配列やオブジェクトなどの型注釈には気をつける必要があります。

配列は要素の型に配列リテラルと同じ `[]` を付けて型注釈を行います。Deno では配列の型注釈を書いても怒られません。もちろん省略しても型推論が働いてくれます。

```ts
const narr: number[] = [1, 2, 3];
//          ^^^^^^^^ 明示的に number 型の値をもつ配列だと型注釈
const sarr: string[] = ["A", "B", "C"];
//          ^^^^^^^^ 明示的に string 型の値をもつ配列だと型注釈
const barr = [true, false];
//    ^^^^   ^^^^^^^^^^^^^ boolean[] として推論される
```

空配列で初期化するような場合、初期値からの型推論ができないので、特に型注釈をしておく必要があるでしょう。型注釈しない場合は `any[]` として推論されてしまいます。

```ts
const narr: number[] = [];
// 数値型の要素のみを受け入れる
narr.push(42); // OK

narr.push("文字列"); // NG
// [Error]: Argument of type 'string' is not assignable to parameter of type 'number'
```

:::message
`const tuple: [number] = [1];` というような `[number]` の型注釈もありますが、これはタプル型と呼ばれるもので通常の配列の型注釈とは少し違いますので注意してください。
:::

オブジェクトの型注釈も今まで同じように `:` の後に注釈を追加します。オブジェクトリテラルで使う `{}` の中にさらにプロパティの値の型注釈を追加できます。プロパティを区切るときには基本的にセミコロン `;` を利用します。

```ts
// ワンライナーで型注釈
const box: { width: number; height: number; } = {
  width: 100,
  height: 200,
}; // 値の初期化

// 改行して型注釈
const cube: {
  width: number; // プロパティの型注釈はセミコロン区切り
  height: number; // プロパティの型注釈はセミコロン区切り
  depth: number; // プロパティの型注釈はセミコロン区切り
} = {
  width: 300, // カンマ区切り
  height: 200, // カンマ区切り
  depth: 300, // カンマ区切り
}; // 値の初期化
```

複雑なオブジェクトになると型注釈をワンライナーでやるのは可読性が低くなるので改行します。

同じようなオブジェクトに何回もこのような型注釈をしなくてはならない場合は非常に冗長になってしまうので、同じ型を参照できるように使いまわしたいケースが多いです。`type` キーワードを使って型に名前を付けることできます。この機能を型エイリア ス (Type Alias ) と呼びます。

```ts
type Cube = {
  width: number;
  height: number;
  price: number;
};

// Cube 型として型注釈
const mycube: Cube = {
  width: 300,
  height: 200,
  depth: 300,
}
```

エイリア ス (別名 ) なので既に存在している型に別名を付けることができます。これは別名を付けているだけで新しい型をつくっている訳ではないことに注意してください。

```ts
// 既に存在している string 型に別名を付ける
type MyString = string;
// string 型として型注釈をしているのと同じ
const mystr: MyString = "文字列";
```

型エイリアスは「型情報の定義」や「型情報の参照」の機能として認識しておくと良いでしょう。

### 関数への型注釈

変数への型注釈の基本がわかったところで関数への型注釈の基本を解説しておきます。

次のような文字列を受け取りその長さを返す `strLength()` という関数を考えてみます。

```js:JavaScript
function strLength(str) {
  return str.length;
}
```

関数には引数と返り値の２つの型の情報があるとその関数の利用時にどのような値を渡してどのような値が返ってくるかということがエディタで表示されるので、その２つの値に対して型注釈を加えてあげます。上の関数なら引数は文字列なので `string` 型で、戻り値は数値なので `number` 型として注釈します。

以下のように変数での型注釈と同じ容量で `引数名: 型` として引数の型注釈を行い、`()` の後に `(): 型名` として戻り値の型注釈を追加します。

```ts:TypeScript
function strLength(str: string): number {
  return str.length;
}
```

引数がいくつもあったりすると関数宣言の行が長くなって見づらくなるので改行してあげると見やすくなります。これで引数や戻り値の型注釈に対してもコメントしやすくなります。

```ts:TypeScript
function strsLength(
  str1: string, // カンマで区切ることを忘れない
  srt2: string
): number {
  const join = str1 + str2;
  return join.length;
}
```

返り値がない場合の関数は特殊な型 `void` で型注釈します。

```ts
function consoleStr(
  str: string
): void { // 関数の返り値が無いことを表現する void 型
  console.log(str);
}
```

戻り値の型注釈を省略しても `return` 文の値から型推論されるので大丈夫です。`return` 文が無ければ基本的には `void` 型です。

```ts
function consoleStr(str: string) {
  console.log(str);
}
```

ちなみに、コールバック関数に使用する無名関数の定義に型注釈をする必要はありません。例えば、`map()` メソッドのコールバック関数の引数に型注釈をする必要はありません。

```ts
const floats: number[] = [1.1, 2.2, 3.3];

const floors = floats.map(function (item) {
  // floats は number[] 型なのでその要素は number 型であり、コールバックの入力値の型は number 型として通知される
  return Math.floor(item);
  // Math.floor は number 型なら利用できる静的メソッド
});
console.log(floors); // => [ 1, 2, 3 ]
```

型注釈も可能ですが、冗長になります。

```ts
const floors = floats.map(function (item: number): number {
  return Math.floor(item);
});
```

このようなプロセスは関数のコンテキストが自身の型を通知することから **Contextual typing** と呼ばれます。

アロー関数でも同じです。Contextual typing によって型注釈は省略できます。

```ts
const floors = floats.map((item) => {
  return Math.floor(item);
});
```

ということで、`then()` メソッドに登録するコールバック関数も一々型注釈をする必要はありません。

```ts
Promise.resolve(1.1) // 数値なので number 型が通知される
  .then((num) => console.log(Math.floor(num))); // => 1
  // コールバック関数の型注釈は省略できる
```

コールバックで使うときなどは上で見たように Contextual typing の仕組みによって型注釈を省略できる場合がありますが、それ以外の場合でアロー関数を定義する際の型注釈は次のようになります。通常の関数宣言の型注釈と大差ありません。

```ts
const arrowStrsLength1 = (str1: string, str2: string): number => {
  const join = str1 + str2;
  return join.length;
};

// 引数などの型注釈が長くなったら改行して見やすくする
const arrowStrsLength2 = (
  str1: string,
  str2: string
): number => {
  const join = str1 + str2;
  return join.length;
};
```

こういったアロー関数の型も型エイリアスによって使い回せるようにできます。ただし、書き方が `(引数: 引数の型) => 戻り値の型` というようになるので注意してください。

```ts
// 関数の型に StrsLength という名前を付ける
type StrsLength = (str1: string, str2: string) => number;

// 関数の型を代入する変数に対して注釈してあるので、引数や返り値の型注釈は省略できる
const arrowStrsLength: StrsLength = (str1, str2) => {
  const join = str1 + str2;
  return join.length;
}
```

型エイリアスなどによって関数の型の作成する際にはいくつか書き方があるので注意してください。

```ts
// 関数の型の作成(アロー関数構文)
type StrsLength1 = (str1: string, str2: string) => number;

// 関数の型の作成(メソッド構文)
type StrsLength2 = {
  (str1: string, str2: string): number
};
```

アロー関数のように書くアロー関数構文は戻り値の方をアロー記号の後に記述します。メソッド構文は [Call Signature](https://www.typescriptlang.org/docs/handbook/2/functions.html#call-signatures) とも呼ばれています。

:::details メソッドの型注釈
オブジェクトのメソッドの型注釈は上で見たアロー関数の形に似た型注釈をする場合が多いですが、JS ではメソッドの定義方法も次のようにいくつかり、その方法に基づいて型注釈を行えます。

```js:JavaScript
const obj = {
  prop: 42,
  // functionキーワードによるメソッド定義
  method1: function(str) { return str.length; },
  // 短縮記法によるメソッド定義
  method2(str) { return str.length; },
  // アロー関数によるメソッド定義
  method3: (str) => { return str.length; },
};
```

TypeScript でそれぞれの方法に対して型注釈を施すと次のようになります。とは言っても、どのタイプで定義するかは統一しておいたほうが良いでしょう。

```ts:TypeScript
const obj = {
  prop: 42,
  method1: function(str: string): number {
    return str.length;
  },
  method2(str: string): number {
    return str.length;
  },
  method3: (str: string): number => {
    return str.length;
  },
};
```

型エイリアスで上のようなオブジェクトの型を作成したい場合には次のようにします。方法はどれでもいいですが、実際のメソッドや関数の定義ではアロー関数と通常の関数で `this` などの挙動が変わるので、実際に使っているものに統一した方がいいでしょう。

そして通常の関数の場合には `function` キーワードを使わずに短縮記法のみで型を宣言します。以下のように２つの方法しか使えません。

```ts
type MyObj = {
  prop: number;
  // Function field (省略記法の書き方)
  method1(str: string): number;
  method2(str: string): number;
  // Arrow function field (アロー関数の書き方)
  method3: (str: string) => number;
};
```

型エイリアスで作成したオブジェクトの型を実際に変数に注釈として割り当てる場合にはすでに注釈が加わっているため、メソッドの実装で型注釈を省略できます。また、`function` キーワードを使った定義も可能です。

```ts
// 変数に MyObj 型として型注釈して初期化(メソッド定義の際の引数とか返り値の型注釈は省略できる)
const myobj: MyObj = {
  prop: 42,
  method1: function (str) { return str.length; },
  method2(str) { return str.length; },
  method3: (str) => { return str.length; },
};
```
:::

:::details typeof 型演算子
型エイリアスでメソッドを持つオブジェクトの型を１から作成してみましたが、定義したオブジェクトから型を抽出して別の場所で使い回すようなことをしたい場合もあります。 そのような場合には `typeof` 型演算子 (typeof type operator) を使って変数から型を抽出できます。

```ts
const objWithArrowFn = {
  prop: 42,
  method1: (str: string): number => str.length,
  method2: (str: string): number => str.length,
  method3: (str: string): number => str.length,
};

// 変数から型の抽出
type ReusingType1 = typeof objWithArrowFn;
// これと同じ意味
type ReusingType2 = {
  prop: number;
  method1: (str: string) => number;
  method2: (str: string) => number;
  method3: (str: string) => number;
};
```

もちろんオブジェクトの型だけでなく、配列などが代入されているものなどもこの `typeof` 型演算子で型を抽出できます。

```ts
// 空配列で初期化
const numarr: number[] = [];
type NumArr = typeof numarr;
// number[] 型が抽出される
```
:::

:::details 分割代入引数と残余引数の型注釈

オブジェクトや配列を引数として取る関数において、引数にとるオブジェクトのプロパティや配列の要素について分割代入して関数内部でプロパティや要素を変数で扱かえるようにしたいときには分割代入の構文を関数の引数で使う「分割代入引数 (destructuring assignment parameter)」の書き方が使えます。

```js:JavaScript
// オブジェクトの分割代入引数
function destObj({ a, b }) {
  // オブジェクトのプロパティに変数を割り当てる
  console.log(a, b);
}

// 配列の分割代入引数
function destArr([a, b]) {
  // 配列要素に変数を割り当てる
  console.log(a, b);
}

destObj({ a: 1, b: 2 }); // => 1 2
destArr([1, 2]); // => 1 2
```

分割代入引数について型注釈を行う際には次のようにします。オブジェクトの分割代入引数では、オブジェクトリテラルの型注釈と同じようにし、配列の分割代入引数では配列の型注釈となります。

```ts:TypeScript
// オブジェクトの分割代入引数
function destObj(
  { a, b }: { a: number; b: number; }
): void {
  console.log(a, b);
}

// 配列の分割代入引数
function destArr(
  [a, b]: number[]
): void {
  console.log(a, b);
}

destObj({ a: 1, b: 2 }); // => 1 2
destArr([1, 2]); // => 1 2
```

関数の引数を可変長引数にしたい場合には、残余引数を使って以下のように書けました。
```js:JavaScript
// 残余引数による可変長引数
function rest(...params) {
  // 関数内部では params は配列として濃縮されている
  console.log(params);
}

rest(1, 2, 3); // => [1, 2, 3]
```

実際に関数内では可変長引数として渡した複数の引数は配列として濃縮されているので、型注釈をする際には残余引数に対して配列の型注釈を行います。

```ts:TypeScript
function restNum(...params: number[]) {
  // 関数内部では params は配列として濃縮されている
  console.log(params);
}
function restStr(...params: string[]) {
  // 関数内部では params は配列として濃縮されている
  console.log(params);
}

restNum(1, 2, 3); // => [1, 2, 3]
restStr("A", "B", "C"); // => ["A", "B", "C"]
```

型を一般化したい場合には、後で解説するジェネリクス関数にすることで実現できます。
```ts
function restGeneric<Type>(...params: Type[]) {
  console.log(params);
}

restGeneric<number>(1, 2, 3); // => [1, 2, 3]
restGeneric<string>("A", "B", "C"); // => ["A", "B", "C"]
```
:::

## ジェネリクス

**ジェネリク ス (generics)** は関数のように型が引数 (あるいは変数 ) を扱えるようにすることでより一般的な処理を記述できるようにする TypeScript の機能 (あるいはその概念 ) です。

ジェネリクスは TypeScript の型システムを支える重要な概念であり、Promise の型注釈を理解する上でも必要です。いかつい名前が付いていて難しそうですが実はそこまで難しくはありません。そして、「ジェネリクス」が分かると TypeScript の型について一気に理解できることが多くなります。

配列型がジェネリクスを理解するための分かりやすい例です。変数の型注釈で配列は次のように型注釈を行っていましたね。

```ts
const narr: number[] = [1, 2, 3];
//          ^^^^^^^^ number 型の要素を持つ配列の型注釈
const sarr: string[] = ["A", "B", "C"];
//          ^^^^^^^^ string 型の要素を持つ配列の型注釈
```

実は配列の型注釈にはもう１つやり方があります。配列は JavaScript でいうところの Array オブジェクトです。この Array オブジェクトとして型注釈を行うことができます。

もちろん要素の型も指定したいので上の `number[]` や `string[]` と同じように要素の型も指定すると次のような型注釈となります。

```ts
const narr: Array<number> = [1, 2, 3];
//          ^^^^^^^^^^^^^ number 型の要素を持つ配列の型注釈
const sarr: Array<string> = ["A", "B", "C"];
//          ^^^^^^^^^^^^^ string 型の要素を持つ配列の型注釈
```

配列の要素の型は `Array<Type>` のように `Type` に `string` といった実際の型名を指定します。この `Type` のような型の変数を **型変 数 (type variable)** と呼びます (実際に存在している型の名前ではなく変数です)。上の場合は配列の要素の型を指定するためのものとなっていますね。

これがジェネリクスです。型が変数を使えるようになったことで、ジェネリクスを持つ配列の型注釈では、配列要素が持つことのできる値の型を記述できます。

  > **Generics provide variables to types**. A common example is an array. An array without generics could contain anything. An array with generics **can describe the values that the array contains**.
> ([TypeScript: Documentation - TypeScript for JavaScript Programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html#generics) より引用、太字は筆者強調)

`Array<Type>` という `<Type>` の中に実際に書く型である `string` などが **型引 数 (type argument)** と呼ばれるものであり、これで関数のように型に引数を指定します (実際に存在している型名を指定します)。

`Array<Type>` で型注釈をする場合には `<>` の部分は省略できないことに注意してください。

```ts
const arr: Array<number> = [1, 3]; // OK

// Array<Type> の <Type> の部分は省略できない
const arr: Array = [1, 3]; // [Error]: Generic type 'Array<T>' requires 1 type argument(s).
```

したがって、配列にどのような型の要素を入れるか決めていない場合などにはどんな型の代入も受け入れる特殊な `any` 型を利用します。

```ts
let arr: Array<any>;
let ar: any[];
```

特殊な `any` 型はとりあえずコンパイルエラーを起こさないように型注釈をする場合に役立ちます。つまり、TypeScript の型チェックのメリットそのものを放棄します。また、`any` 型は型注釈をしないことで暗黙的に推論されてでてくる型でもあります。

ただし、Deno でこの `any` を使おうとすると "no-explicit-any" というリンタールールに注意され、代わりに `unknown` 型を利用するようにと言われます。

https://lint.deno.land/?q=any#no-explicit-any

  > Use of the `any` type disables the type check system around that variable, defeating the purpose of Typescript which is to provide type safe code. Additionally, the use of `any` hinders code readability, since it is not immediately clear what type of value is being referenced. It is better to be explicit about all types. For a more type-safe alternative to `any`, use `unknown` if you are unable to choose a more specific type.
> ([no-explicit-any](https://lint.deno.land/?q=any#no-explicit-any) より引用)

`unknown` 型は「どんな型か分からない時に使う型」で `any` よりも安全性が高い型です。

```ts
let uarr: Array<unknown>;
let uar: unknown[];
```

また、このジェネリクスを使って自分で一般的な型を定義することも可能です。

https://www.typescriptlang.org/docs/handbook/2/objects.html#generic-object-types

例えば、`data` プロパティの値の型がなんでもいい型をつくりたい場合に次のように一々色々な具体的な型を指定した型をつくらずに、(一般化する際に `any` や `unknown` も使わないで ) 型引数を指定してその型に適応した型を作り出せるようにしたいです。

```ts
type StringProp = {
  data: string;
};
type NumberProp = {
  data: number;
}
type BooleanProp = {
  data: boolean;
}
```

`Array<Type>` のように型変数を使えるようにするには型定義の際に `<>` を型名の後ろにつけて適当な名前の型変数を付けてあげます。こういった型は "**Generic Object Type(ジェネリックオブジェクト型)**" と呼ばれています。

```ts
type GeneralProp<YourType> = { // 型変数の名前はなんでもよい
  data: YourType;
};

// type StringProp = { data: string }; と同じ型で型注釈
const strProp: GeneralProp<string> = {
  data: "文字列",
};
// type NumberProp = { data: number }; と同じ型で型注釈
const numProp: GeneralProp<number> = {
  data: 42,
};
```

型変数の名前はなんでもよいので今回は `YourType` としてみました。慣習的は `T` や `K` などの文字が使われます。

## ジェネリック関数
ジェネリック関数 (generic function n) はこのジェネリクスの概念を利用した関数になります。

https://www.typescriptlang.org/docs/handbook/2/generics.html

例えば、配列を引数に取って、その配列要素を返すという関数を JavaScript で書くと次のようになります。

```js:JavaScript
function returnArrEl(arr) {
  return arr[0];
}
```

この処理を TypeScript で書くとどのようになるでしょうか。型注釈を省略して関数宣言を行うとその引数は `any` 型として推論されてしまいます。

```ts:TypeScript
// 引数の型注釈を行わない
function returnArrEl(
  arr //: any (暗黙的に any 型として推論される)
) {
  return arr[0];
}
```

このままだと、引数が配列ではない場合には `undefined` が出力されたり、間違って文字列を渡しても許容されたりして意図した処理とならない可能性があります。TypeScript で堅牢なコードにするためには「引数は配列である」という型注釈を加えたいです。

配列の型注釈は配列要素の型に `[]` を付けたものでした。例えば次のように型注釈をするのはどうでしょうか。

```ts
function returnArrEl(
  arr: number[]
): number {
  return arr[0];
}

// number 型の配列は引数として受け入れる
const result1 = returnArrEl([4, 0, 3]);
console.log(result); // => 4

// string 型の配列は引数として受け入れない
const result2 = returnArrEl(["A", "B", "C"]);
// 型エラーになる
console.log(result2);
```

この型注釈だと数値を要素とした配列しか引数に受付なくなってしまいますね。実際、VS code ならエディタ上で型チェックに引っかかり警告されますが、`deno check` コマンドでコマンドラインから型チェックを実行してもエラーが吐き出されます。

:::details deno check で型チェック
```sh
❯ deno check generic.ts
Check file:///Users/roshi/Development/Testing/js-syntax/ts-syntax/generic.ts
error: TS2322 [ERROR]: Type 'string' is not assignable to type 'number'.
const result2 = returnArrEl(["A", "B", "C"]);
                             ~~~
    at file:///Users/roshi/Development/Testing/js-syntax/ts-syntax/generic.ts:8:30

TS2322 [ERROR]: Type 'string' is not assignable to type 'number'.
const result2 = returnArrEl(["A", "B", "C"]);
                                  ~~~
    at file:///Users/roshi/Development/Testing/js-syntax/ts-syntax/generic.ts:8:35

TS2322 [ERROR]: Type 'string' is not assignable to type 'number'.
const result2 = returnArrEl(["A", "B", "C"]);
                                       ~~~
    at file:///Users/roshi/Development/Testing/js-syntax/ts-syntax/generic.ts:8:40

Found 3 errors.
```
:::

この関数の処理は配列の要素の型に依存せずにただ配列要素を返すだけなので、配列要素の型に関わらず「あらゆる配列」を受け入れるように「**一般化**」したいです。

`unknown` 型は「どんな型か分からない時に使う型」で `any` よりも安全性が高い型だという話でしたので、`unknown[]` という配列要素の型が分からない配列という型注釈はどうでしょうか。

```ts
function returnArrEl(
  arr: unknown[] // 配列要素の型が分からないという型注釈
): unknown { // 配列要素の型が分からないという型注釈
  return arr[0];
}
```

これで、どんな要素を持つ配列が来るかはわからないようにしています。これで型の情報が「一般化」されたように思えますが、型注釈をしない時に `any` 型として推論されてしまう場合と対して変わりません。この関数の利用時には返り値の型が `unknown` としてエディタでも表示されるので型の情報がほとんど何もないことになります。

型推論で配列要素の型が実際に表示されるようにしたいわけです。ジェネリクスは「一般化」を意味しますが、ここでジェネリクスを使って関数を記述することで型を一般化できます。このような関数をジェネリック関数 (generic function ) と呼びます。

ジェネリック関数は `Array<Type>` で見たように型変数として `Type` (実際の名前はなんでもよい ) を関数名の後に `<>` をつけて定義します。関数の引数の定義と似ていますね。これによって、一般的にあらゆる型を受け入れるようにできます。

```ts:ジェネリック関数
// Type は型変数で実際に存在している string などの型名ではない
function returnArrEl<Type>( 
  arr: Type[] // Type 型の要素を持つ配列の型注釈
) {
  return arr[0];
}
```

`Type` は実際に存在する型の名前ではなく型変数ですから、これで一般化されたことになります。

元々の関数を見てみると、`arr[0]` の型は `arr` という配列要素の型と同じですね。この処理では配列や配列要素の型がなんであろうと別に関係なく、引数として受け取った配列の要素をただ返すという処理です。関数の引数という入力の値の型と関数の返り値という出力の値の型には**関連性が存在**しています。

このように関数の入力となる値の型と出力となる値の型に関連性がある場合には型変数を利用して**相互の型をリンクさせることができます**。以下の関数の型注釈では、型変数によって入力の値と出力の値の型が同じになるようにリンクさせています。

```ts:ジェネリック関数
function returnArrEl<Type>( // Type は型変数
  arr: Type[] // 入力と出力の値の型がリンクした
): Type { // 入力と出力の値の型がリンクした
  return arr[0];
}
```

ジェネリック関数においてこのように複数の型を型変数でパラメータ化できるため、この場合の型変数 `Type` を型パラメー タ ([Type parameter](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#type-parameters) ) と呼びます。`extends` を併用することで特定の条件を満たす型へと拘束することも可能です。

このようなジェネリック関数として定義することでより一般的な処理となる関数を書くことができます。呼び出す際に型引数 (type argument ) として実際に存在している型名を指定することで型を明示できます。`Array<string>` のように配列の型注釈をするのと同じように関数を使用する際に具体的な型引数を指定するわけです。

```ts
function returnArrEl<Type>( // Type は型変数
  arr: Type[] // 入力と出力の値の型がリンクした
): Type { // 入力と出力の値の型がリンクした
  return arr[0];
}

// 型引数として具体的な number 型を指定
const result1 = returnArrEl<number>([4, 0, 3]);
// 返り値の型は number 型であるとエディタ上でしっかり表示される
console.log(result); // => 4

// 型引数として具体的な string 型を指定
const result2 = returnArrEl<string>(["A", "B", "C"]);
// 返り値の型は string 型であるとエディタ上でしっかり表示される
console.log(result2); // => "A"
```

**入力と出力の値の型がリンクしているため**、実は型引数の部分は省略しても引数の値から型推論してくれます。

```ts
// 両方とも型エラーにならない
const result1 = returnArrEl([4, 0, 3]);
// 返り値の型は number 型であるとエディタ上でしっかり表示される
console.log(result); // => 4

const result2 = returnArrEl(["A", "B", "C"]);
// 返り値の型は string 型であるとエディタ上でしっかり表示される
console.log(result2); // => "A"
```

実際には引数の配列の要素が空の場合もありえるのでより正確に型注釈するとこの関数の返り値の型は `Type | undefined` という **ユニオン 型 (union type)** になります。`undefined` は実際に存在する型です。配列が空の場合にはこの関数からは `undefined` という値が返ります。

```ts
function returnArrEl<Type>( // Type は型変換
  arr: Type[] // 入力と出力の値の型がリンクした
): Type | undefined { // 戻り値の型は Type または undefined
  return arr[0];
}
```

例えば `string | number` などがユニオン型ですが、これは `string` 型または `number` 型という２つの型を受け入れる合成された型です。このように２つの型を組み合わせることを「**型の合 成 (Composing Types)**」と呼びます。

次のように変数宣言で型注釈をする際にももちろん使えます。

```ts
// let 宣言して型定義
let strornum1: string | number;
strornum1 = Math.random() < 0.5 ? "文字列" : 42; // 三項演算子

// type で型作成
type StrOrNum = string | number;
let strornum2: StrOrNum;
strornum2 = 42; // number 型の値も代入できるし
strornum2 = "文字列"; // string 型の値も代入できる
```

このユニオン型が関数の引数となることで、関数内部で引数に対して利用できるメソッドがそのユニオン型に含まれる型によって変わってくるので場合分けをする必要がでてきます。

```ts
// string 型または number 型 やそのユニオン型で注釈された変数を受け入れる(それ以外は受け入れない)
function strOrNum(
  param: string | number
): void {
  if (typeof param === "string") {
    // string 型のプロトタイプメソッド
    console.log(param.toUpperCase());
  } else { // string 型でないなら number 型
    // number 型の値に使える静的メソッド
    console.log(Math.floor(param));
  }
}
```

こういったコードの構造に基づいて値の型をより具体的に推定できるようにすることを (型の範囲をより具体的なものに狭めることから) **Narrowing(型の絞り込み)** と呼びます (あるいはその現象そのものを Narrwing と呼びます)。つまり、型情報の選別やフィルターを行う行為がコードを書く上でも必要となります。

https://www.typescriptlang.org/docs/handbook/2/narrowing.html

上のコードでの `if` 節や `switch` や `while` などのコードの構造によって各場所での変数の型を絞り込みます。このようなコードを書くと TypeScript (コンパイラやエディタの拡張機能 ) はある変数が特定のブランチなどに到達した時点でその型がなんであるか解析をしています。この解析を「**制御フロー解 析 (Control flow analysis: CFA)**」と呼びます。

もしも、次のように `else` ブランチを増やしてもそのブランチには決して到達することはありません (`string` 型か `number` 型しか引数に受け取らないため)。

```ts
// string 型または number 型 やそのユニオン型で注釈された変数を受け入れる(それ以外は受け入れない)
function strOrNum(
  param: string | number
): void {
  if (typeof param === "string") {
    // param: string として CFA で解析される
    console.log(param.toUpperCase());
  } else if (typeof param === "number") {
    // string 型でないなら number 型
    // param: number として CFA で解析される
    console.log(Math.floor(param));
  } else {
    // param: never として CFA で解析される
    console.log(param);
    //          ^^^^^ never 型(決して観測されない)
  }
}
```

ということで、その `else` ブランチ内で引数を参照しようとすると制御フロー解析によってその値は `never` 型として見なされます。`never` 型の値は決して観測されることがないことを表現する型です。

他にも、例外をスローするだけの関数では返り値を決して取れないため、返り値の型を `never` 型として注釈します。無限ループを作り出す関数なども返り値が取れないので返り値の型注釈は `never` 型になります。

```ts
function throwError(msg: string): never {
  throw new Error(msg);
}
```

:::details リテラル型 (literal type)

数値リテラルや文字列リテラルなどのリテラルからも型 (type ) は作成できます。リテラルで指定したプリミティブ型の特定の値だけを代入可能にするような型がリテラル型の特徴です。

例えば、次のように `'string-literal'` という文字列リテラルで作成されたリテラル型の変数は `'string-literal'` という値しか代入できません。

```ts
let specific: "string-literal"; // リテラル型
specific = "string-literal"; // 代入OK

specific = "strin-literal"; // 型エラーとなる
```

上のように代入する値のスペルを間違えると型エラーになります。リテラル型として表現できるものは数値リテラル、文字列リテラル、真偽値リテラルの３つです。そしてリテラル型同士を組み合わせてユニオン型を作ることもできます。

```ts
type PositiveOddNumbersUnderTen = 1 | 3 | 5 | 7 | 9;
// 10以下の奇数のみ代入可能な型
```

このような型はマジックナンバーなどに利用できます。
:::

:::details インターセクション型 (intersection type)

型の合成としてユニオン型 (union type ) がありましたが、もう１つの合成の仕方としてインターセクション型 (intersection type ) が存在しています。

これはブール演算の論理和と論理積と同じです。ユニオン型は `A | B` として「A または B」という型の合成でしたが、インターセクション型は `A & B` として「A かつ B」という型の合成を行います。主にオブジェクトの型同士で積を作成して、合成に使ったすべてのメンバーを持つオブジェクトの型を作成します。

```ts
type Colorful = {
  color: string;
};
type Text = {
  text: string;
}
// インターセクション型の作成
// (両方のメンバーを持つオブジェクトの型を作成)
type ColorfulText = Colorful & Text;

const t: ColorfulText = {
  color: "red",
  text: "文字列",
};
```

プリミティブ型同士でインターセクション型を作成すると `never` 型となってしまいます。これは型同士の共通部分を探そうとしても空集合となるためです。

```ts
// string 型と number 型の Union
type StrOrNum = string | number;
const sn = Math.random() < 0.5 ? "string" : 42;

// string 型と number 型の Intersection
type Nev = string & number;
const nv = 42; // 型エラー
```

`A | B` は型が A か B のどれかということで取り得る値の範囲が広くなり、型の制約が緩くなりますが、`A & B` は A と B のどれもということで型がとり得る値の範囲が狭まり、型の制約が厳しくなります。これはオブジェクトの型同士のユニオン型とインターセクション型を比較すると理解できます。先程の例を使ってみます。

```ts
type Colorful = {
  color: string;
};
type Text = {
  text: string;
};
// インターセクション型(Colorful かつ Text)
type ColorAndText = Colorful & Text;

const cAndT: ColorAndText = {
  color: "色",
  text: "文字列",
}; // 必ず２つのプロパティが必要になる

// ユニオン型(Colorful または Text)
type ColorOrText = Colorful | Text;

const cOrT1: ColorOrText = {
  color: "色",
  text: "文字",
};
const cOrT2: ColorOrText = {
  color: "色", // 片方のプロパティがなくても大丈夫
};
const cOrT3: ColorOrText = {
  text: "文字",
};
```

参考: [TypeScriptのUnion / Intersection Typesで遊んだ - Lambdaカクテル](https://blog.3qe.us/entry/2020/01/21/142740)
:::

ジェネリック関数に話を戻すと型変数を複数個定義することでそれぞれをリンクすることも可能です。例えば、配列のプロトタイプメソッドである `map()` の機能を新しく関数として定義する場合にこのような使い方ができます。

```ts
function mymap<Input, Output>( // Input と Output は型変数
  arr: Input[], // 入力同士のリンク
  func: (arg: Input) => Output // 入力同士と出力とのリンク
): Output[] { // 入力と出力のリンク
  return arr.map(func);
}

// 型引数を省略して利用する(引数の値から型推論される)
const parsed = mymap(["1", "2"], (item) => parseInt(item));
const parsedN = mymap<string, number>(["1", "2"], (item) => parseInt(item));
```

引数にコールバックが来る場合には `func: (arg: Input) => Output` というアロー関数のような形の型注釈を行います。`type` で別に作成して参照するのもありです。

## Promise の型注釈

さて、Promise の型注釈をするには今まで見てきたジェネリクスの概念と型引数・型変数が必要です。まずは簡単な変数宣言から型注釈を始めてみます。次のように文字列を履行値として直ちに履行する Promise インスタンスを作成して変数に代入します。

```js:JavaScript
const sp = new Promise(resolve => {
  resolve("文字列で履行する");
});
```

この変数 `sp` には Promise インスタンスが代入されるので Promise オブジェクトの型を注釈したいと思います。Promise オブジェクトの型はジェネリクスを使って `Promise<Type>` というような形式になっています。`Type` は上で見たような型変数であり、型注釈する際には実際に存在する型名を型引数として指定します。では型変数にどのような型引数を指定すればよいかというと、履行値の型名を指定してあげるのが基本となります。

この場合は `"文字列で履行する"` という文字列の値で履行するので `string` という型名を型引数として指定します。`Array<Type>` のように Promise 型で `Promise<Type>` のように型注釈する場合には型引数は省略できません。

```js:TypeScript
// <string>は省略できないので注意
const sp: Promise<string> = new Promise(resolve => {
  resolve("文字列で履行する");
});
```

`Promise.resolve()` でも同じことですね。数値で履行するなら型引数には `number` 型を指定します。

```js:TypeScript
const np: Promise<number> = Promise.resolve(42);
```

一般化して考えると、Promise は値を入れ込むことができたのでその値の型を型引数として指定するわけです。非同期 API である `fetch()` メソッドはその結果である `Response` オブジェクトを Promise の中に入れ込んで結果として返してくれました。そういう訳で `fetch()` から返ってくる Promise インスタンスを代入する変数には `Promise<Response>` というような型注釈ができます。

```ts
const responsePromise: Promise<Response> = fetch("https://api.github.com/zen");

// Promise インスタンスなので chain できる
responsePromise
  .then(response => response.text())
  .then(text => console.log(text));
```

この `Response` という型は Deno 側が元々用意してくれている型で `lib.deno.fetch.d.ts` というファイルに定義されています。VS Code などで `Response` をクリックすると定義もとに飛べます。

型定義は API ドキュメントの以下のページでも確認できます。

https://doc.deno.land/deno/stable/~/Response

```ts:DenoでのResponseの型定義
class Response implements Body {
  constructor(body?: BodyInit | null, init?: ResponseInit);
  readonly body: ReadableStream<Uint8Array> | null;
  readonly bodyUsed: boolean;
  readonly headers: Headers;
  readonly ok: boolean;
  readonly redirected: boolean;
  readonly status: number;
  readonly statusText: string;
  readonly trailer: Promise<Headers>;
  readonly type: ResponseType;
  readonly url: string;
  
  arrayBuffer(): Promise<ArrayBuffer>;
  blob(): Promise<Blob>;
  clone(): Response;
  formData(): Promise<FormData>;
  json(): Promise<any>;
  text(): Promise<string>;
  
  static error(): Response;
  static json(data: unknown, init?: ResponseInit): Response;
  static redirect(url: string, status?: number): Response;
}
```

このように API メソッドやそれに付随するインターフェイスの型定義を Deno 側で既に用意してくれているのでそれらの恩恵を受けてコードを書くことができます。

`fetch()` から返る Promise インスタンスの中身は Response であきらかですから、型注釈は省略して型推論させても良いです。

```ts
const responsePromise = fetch("https://api.github.com/zen");
```

ただし、Promise chain の場合は最後の chain のコールバックから返ってくる値の型を指定する必要がありますね。まあ、省略しても推論してくれます。

```ts
const p: Promise<string> = fetch("https://api.github.com/zen").then(response => response.text()); // 最後のコールバックでは文字列が返るはず

// 省略しても Promise<string> と推論してくれる
const pn = fetch("https://api.github.com/zen").then(response => response.text());
```

結局のところ JavaScript が正しく書かれていれば TypeScript で動きます。ただし、正しい型注釈をしようと思ったらそれなりに TypeScript のことを知らないと難しいです。実際、現実的にはエラーハンドリングをしますから、エラーオブジェクトなどの型注釈も必要となります。

## Promise を返す関数の型注釈

Promise インスタンスを返す関数は次のように返り値の型注釈を書きます。履行値の値の型を `Promise<Type>` の型引数として指定することで、この関数から返ってくる Promise インスタンスがどのような値をもっているのかがわかりやすくなりますね。

```ts
function returnPromise(
  str: string
): Promise<string> {
  return new Promise(resolve => {
    console.log("履行値が文字列の場合は <string> にする");
    resolve(str); // string 型の値で履行する
  });
}
```

JavaScript では async 関数は `async` キーワードが付いているため Promise インスタンスを返すことが一目で分かりましたが、このような Promise インスタンスを返す通常の関数や `setTimeout()` などの API を Promisification したものは分かりづらかったので、型注釈をしたことで返り値の型を見れば一目で理解できるようになりまたね。

引数と返り値の値の型をリンクさせて一般化するには型変数を使ってジェネリック関数にします。

```ts
// ジェネリック関数
function generalPromise<Type>( // Type は型変数
  param: Type // 入力と出力の型がリンク
): Promise<Type> { // 入力と出力の型がリンク
  return Promise.resolve(param);
  // 引数で履行する Promise インスタンスを返却
}
```

実際に呼び出す際には型引数の指定は省略して型推論に任せることができます。

```ts
// オブジェクトの型 { key: string } を型引数として指定
generalProimse<{ key: string }>({ key: "value" })
  .then(val => console.log(val));

// 型引数を省略しても引数から型推論してくれる
generalProimse({ key: "value" })
  .then(val => console.log(val));
```

型引数には関数の引数となるオブジェクトの型 `{ key: string }` を指定しています。

TypeScript のジェネリクスは色々なところで使われています。実際 `then()` メソッドの型定義などにもこのジェネリクスが利用されています。

```ts:lib.es5.d.ts
/**
 * Represents the completion of an asynchronous operation
 */
interface Promise<T> {
    /**
     * Attaches callbacks for the resolution and/or rejection of the Promise.
     * @param onfulfilled The callback to execute when the Promise is resolved.
     * @param onrejected The callback to execute when the Promise is rejected.
     * @returns A Promise for the completion of which ever callback is executed.
     */
    then<TResult1 = T, TResult2 = never>(onfulfilled?: ((value: T) => TResult1 | PromiseLike<TResult1>) | undefined | null, onrejected?: ((reason: any) => TResult2 | PromiseLike<TResult2>) | undefined | null): Promise<TResult1 | TResult2>;

    /**
     * Attaches a callback for only the rejection of the Promise.
     * @param onrejected The callback to execute when the Promise is rejected.
     * @returns A Promise for the completion of the callback.
     */
    catch<TResult = never>(onrejected?: ((reason: any) => TResult | PromiseLike<TResult>) | undefined | null): Promise<T | TResult>;
}
```

ジェネリクスではデフォルトの型引数を指定することもできます。`then<TResult1 = T, TResult2 = never>` の `T` や `never` はデフォルトの型引数であり、明示的に指定しない場合の型引数となります。

:::details インターフェース型 (interface type)

オブジェクトの型を定義するために型エイリア ス (type alias ) が使えましたが、インターフェース 型 (interface type ) はオブジェクトの型を定義するためのもう１つの方法です。型エイリアスはあらゆる型に名前を付けるものとして使えますが、インターフェース型はオブジェクトやクラスの型を表現するためのものとして利用できます。

以下のように型エイリアスとインターフェースでは書き方が異なることに注意してください。

```ts
// インターフェースで型を定義
interface Animal1 {
  name: string;
  habitat: string;
} // 宣言なのでセミコロンは書かない


// 型エイリアスで型を定義
type Animal2 = { // 代入
  name: string;
  habitat: string;
};
```
:::

このようなビルトインメソッドなどの型定義では型変数が大量に使われていて複雑なように見えますが、JavaScript での書き方を知っていて、しっかり見れば結構分かります。実際次のように型を抽出・整理してみることでスッキリと理解できます。

```ts
// 関数の型と unedfined 型と null 型のユニオン型
type OnFulfilled<T, TResult1> = ((value: T) => TResult1 | PromiseLike<TResult1>)
  | undefined
  | null;
// 関数の型と unedfined 型と null 型のユニオン型
type OnRejcted<TReulst2> = ((reason: any) => TResult2 | PromiseLike<TResult2>)
  | undefined
  | null;

// 型変数はリンクしているので注意
interface Promise<T> {
  then<TResult1 = T, TResult2 = never>(
    onfulfilled?: OnFulfilled<T, TReulst1>,  // optional
    onrejected?: OnRejcted<TResult2> // optional
  ): Promise<TResult1 | TResult2>;

  catch<TResult = never>(
    onrejected?: OnRejcted<TResult> // optional
  ): Promise<T | TResult>;
}
```

ちなみに、`Promise.resolve()` メソッドもジェネリクスで型が定義されています。

```ts:lib.es2015.promise.d.ts
    /**
     * Creates a new resolved promise for the provided value.
     * @param value A promise.
     * @returns A promise whose internal state matches the provided promise.
     */
    resolve<T>(value: T | PromiseLike<T>): Promise<T>;
```

:::message
ECMAScript のシンタックスとして具体的な型に依存しないタイプのプロトタイプメソッドはこのように型が抽象化、あるいは一般化されたものとして型定義が用意されています。今日の JavaScript には `Map`、`Set` といったデータのコレクションを表現するためのビルトインオブジェクトが存在しています。予想されるように `Array<T>` や `Promise<T>` と同じくこれらのコレクションのデータ型はジェネリクスで `Map<K, V>` や `Set<T>` として一般化されて型定義されています。

また、`.d.ts` という拡張子のファイルは [型定義ファイル](https://typescriptbook.jp/reference/declaration-file ) と呼ばれるものです。TypeScript が提供する型や npm のパッケージ配布を行うために作成されます。
:::

:::details JSDoc の補足
型定義ファイルに記載されている `@param` などは JSDoc と呼ばれる JavaScript のソースコードにアノテーションを追加するために使われるマークアップです。元々 JavaScript で利用されていましたが、TypeScript でも利用可能です。以下のように、関数定義の直前に様々な情報を記載します。

```js
/**
 * 人物の名前を引数にとって挨拶文を出力する関数
 *
 * @param {(string|string[])} [sombody=John Doe] - 人物の名前または名前の配列
 **/
function sayHello(somebody) {
  if (somebody) {
    sombody = 'John Doe';
  } else if (Array.isArray(somobody)) {
    somebody = sombody.join(', ');
  }
  alert('Hello' + somebody);
}
```

この JSDoc で関数の説明や引数・返り値の説明などを加えることでエディタ上で使い方などが表示されるようになります。TypeScript そのものとは関係ないですが、組み合わせて使えます。TypeScript の公式ドキュメントにもリファレンスが記載されています。

- 参考: [TypeScript: Documentation - JSDoc Reference](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html)
:::

ということで、通常は省略しますが `Promise.resolve()` や `then()` メソッドを呼び出す際には型引数に明示的にコールバック関数から返される値の型を指定できます。

```ts
Promise.resolve<string>("string") // 履行値の型を明示的に型引数として指定
  .then<number>(val => val.length) // コールバックの返り値の型を型引数として指定
  .then<void>(val => console.log(val)); // コールバックの返り値の型を型引数として指定
```

実は、`Promise()` というコンストラクタ関数もジェネリック関数なので、明示的に型引数を指定できます (通常は省略できます)。

```ts
// 履行値は number 型なので型引数に number 型を指定
const p = new Promise<number>(resolve => resolve(42));
```

さて、ジェネリック関数での定義がわかったところで Proimse-based なタイマーである `pTimer()` に対して TypeScript で型注釈を加えてみます。

```js:JavaScript
function pTimer(time) {
  return new Promise(resolve => setTimeout(() => {
    console.log(`${time}[ms]でタイムアウトしました`);
    resolve(time);
  }), time);
}
```

履行値を遅延時間そのものとして引数に `number` 型の値を取るようにさせます。履行値の型が `number` 型なので返ってくる Promise インスタンスが持つ値の型が `number` 型となりますね。ということで `pTimer()` の実装は次のようになります (型注釈を加えただけです)。

```ts:TypeScript
function pTimer(
  time: number
): Promise<number> {
  return new Promise(resolve => setTimeout(() => {
    console.log(`${time}[ms]でタイムアウトしました`);
    resolve(time);
  }), time);
}

pTimer(1000).then(val => console.log("履行値は", val));
// => 履行値は 1000
```

## async 関数の型注釈

それでは、async 関数での型注釈を考えてみますが、ここまでくれば基本は余裕ですね。

次の関数は引数に文字列の値を取って履行する特に意味のない async 関数です。

```js:JavaScript
async function rPromise(msg) {
  console.log(`"${msg}"という文字列で履行します`);
  return await Promise.resolve(msg);
}
```

これに型注釈を加えると次のようになります。

```ts:TypeScript
async function rPromise(
  msg: string
): Promise<string> {
  console.log(`"${msg}"という文字列で履行します`);
  return await Promise.resolve(msg);
}
```

async 関数は常に Promise インスタンスを返すので関数のボディで何も `return` しない空の async 関数の場合も次のように戻り値が Promise インスタンスとなります。その Promise インスタンスは `undefined` で履行されますが、関数ボディ自体からは何も返さないので、「何も返さない」ということを表現する `void` 型を型引数に指定して `Promise<void>` として型注釈できます。

```ts
async function empty(): Promise<void> {}
```

もちろん、型注釈は省略できますので、この場合も一々書く必要は特に無いでしょう。

次に、『[await 式の配置による制御](https://zenn.dev/estra/books/js-async-promise-chain-event-loop/viewer/18-epasync-await-position)』のチャプターの最後で登場した Promise-based なキャンセル可能タイマー `dTimer` に型注釈してみます。

```js:JavaScript
import { delay } from "https://deno.land/std@0.145.0/async/mod.ts";

// キャンセル可能な Promise-based なタイマー
async function dTimer(msg, time, option = {}) {
  try {
    await delay(time, option);
    console.log(`${time}[ms]が経過しました`);
    return msg;
  } catch (err) {
    console.log("タイマーはキャンセルされました" ,err)
  }
}
```

今までは返り値の型がシンプルなものしか見てきませんでしたが、この場合には例外補足の際に分岐するので、それぞれの節で返される値の型を合成する必要がでてきます。

```ts:TypeScript
import {
  delay,
  DelayOptions,
} from "https://deno.land/std@0.145.0/async/mod.ts";

async function dTimer(
  msg: string,
  time: number,
  option: DelayOptions = {}
): Promise<string | void> {
  // ユニオン型を入れ込んだ Promise 型
  try {
    await delay(time, option);
    console.log(`${time}[ms]が経過しました`);
    return msg; // string 型を返却
  } catch (err) {
    console.log("タイマーはキャンセルされました", err);
    // 何も返却しない void 型
  }
}

const controller = new AbortController();
const signal = controller.signal;
const rTimes = [200, 100, 300];

(async () => {
  const promises: Promise<string | void>[] = rTimes.map((time) =>
    dTimer(`${time}[ms]のタイマー`, time, { signal })
  );
  const winner = await Promise.race(promises);
  controller.abort(); // すべてのタイマーを停止させる
  console.log("raceの結果:", winner);
  await Promise.allSettled(promises);
  console.log("タイマーの競争が終了しました");
})();
```

`catch` 節で何も `return` していないので返り値が何もない場合の `void` 型と通常の成功時の返り値である `string` 型を合成したユニオン型 `string | void` を `Promise<Type>` の型引数として指定してあげています。また、`delay()` のオプションとして渡す引数の型として `DelayOptions` も import するようにします。

他には、３つの API のエンドポイントからデータフェッチする関数を考えみると次のような感じになるでしょうか。

```ts:TypeScript
const urls: string[] = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];
// データフェッチで返ってくる JSON データはこんな感じの構造
type Todo = {
  userId: number;
  id: number;
  title: string;
  completed: boolean;
};

async function fetcher(
  url: string
): Promise<[Todo|null, null|unknown]> {
  try {
    const response = await fetch(url);
    const json: Todo = await response.json();
    return [json, null];
  } catch (error) {
    return [null, error];
  }
}

(async () => {
  let todos: (Todo|null)[] = [];

  for (let i = 0; i < urls.length; i++) {
    const [data, error] = await fetcher(urls[i]);
    todos = [...todos, data];
    if (error) { console.error(error); }
  }
  console.log({ todos });
})();
```

JavaScript で言う所の [try-catch hell](https://www.youtube.com/watch?v=ITogH7lJTyE) を避けるための `[data, error]` という２つの値からなる配列を返すパターンを使っています。このパターンを使うことで返り値の型注釈が面倒なことになっていますね (現時点ではあまりうまく型付けできていないと思います)。

要素の型がそれぞれ異なる配列は TypeScript では**タプ ル (Tuple ) 型**と呼ばれます。

https://typescriptbook.jp/reference/values-types-variables/tuple

関数は基本的に１つの値しか返せないため、戻り値を複数にしたい場合には配列に格納して返すことで実現できます。

```js:JavaScript
function returnMultipleValue() {
  return [42, "文字列", true];
  // 色々な型の値が要素となった配列を返す
}
```

この戻り値を型注釈するためにタプル型と呼ばれる型の書き方で注釈します。

```ts:TypeScript(タプルの型注釈)
function returnMultipleValue(): [number, string, boolean] {
  return [42, "文字列", true];
}
```

タプル型は変数宣言のときには次のようになります。

```ts
const list1: [number, string, boolean] = returnMultipleValue();

// 型エイリアスで型を作成した場合
type MyTuple = [number, string, boolean];
const list2: MyTuple = returnMultipleValue();
```

複数の型の要素を受け入れる配列に対して、タプル型として型注釈を加えない場合にはそれぞれの要素の値の型のユニオン型の配列となります。これはタプル型ではないので注意してください。

```ts
const unionArr1 = [42, "文字列", true];
// unionArr1: (number | string | boolean)[] として推論されてしまう
const unionArr2: (number | string | boolean)[] = [42, "文字列", true, 32, "text", false];
// ユニオン型の配列なので要素数に制限はない
// 配列要素の型が number または string または boolean という意味

// これがタプル型の注釈(要素数は型のとおり３つとなる)
const tuple: [number, string, boolean] = [42, "文字列", true];
// 値の並びも number, string, boolean の順番に決まっている
```

タプル型の値を返すことで、async 関数利用時の try-catch を大量に書くのを防ぎます。基本は async 関数に例外補足を閉じ込めて標準化してデータフェッチの成功時には `[data, null]` を返し、失敗時には `[null, error]` を返すようにするというパターンです。

```js:JavaScript
async function fetcher(url) {
  try {
    const response = await fetch(url);
    const json = await response.json();
    // 成功時には [data, null] を返す
    return [json, null];
  } catch (error) {
    // 失敗時には [null, error] を返す
    return [null, error];
  }
}
```

返り値を受ける側は配列の分割代入を使います。

```js:JavaScript
(async () => {
  let todos = [];

  for (let i = 0; i < urls.length; i++) {
    // 配列の分割代入
    const [data, error] = await fetcher(urls[i]);
    todos = [...todos, data];
    if (error) { console.error(error); }
  }
  console.log({ todos });
})();
```

JavaScript だと簡単ですが、返り値の型注釈をしっかりしようとすると以外とうまくいきません。一応型エラーにならずに動くのがこの型注釈です。

```ts:TypeScript
async function fetcher(
  url: string
): Promise<[Todo|null, null|unknown]> {
  try {
    const response = await fetch(url);
    // JSON データに型を付けておく
    const json: Todo = await response.json();
    // 成功時には [data, null] を返す
    return [json, null];
  } catch (error) {
    // 失敗時には [null, error] を返す
    return [null, error];
  }
}
```

値がないことを示す `null` 型という型が存在しているので、その型と
`Todo` 型の合成としてのユニオン型を `Todo|null` として、エラーの型については詳しくないのでとりあえず `unknown` 型と `null` 型のユニオン型 `null|unknown` という２つのユニオン型のタプル型を `Promise<Type>` の型引数として指定してあげています。

もっとうまい型注釈はあるだろうなと思いますが、とりあえずはこれで型エラーにはなりません (型の表現方法はいくつもあります)。

あるいは、タプル型ではなく次のようなオブジェクトで返すというのが便利かもしれません。

```ts
// fetcher 関数から返ってくるオブジェクトの型定義
type DataFromFetcher = {
  data: Todo | null;
  error: any;
};
```

また、配列ならイテラブルで `for...of` を使った反復処理ができるので少しコードを改造して以下のようにします。

```ts
const endPoints: string[] = [
  "https://jsonplaceholder.typicode.com/todos/1",
  "https://jsonplaceholder.typicode.com/todos/2",
  "https://jsonplaceholder.typicode.com/todos/3",
];

type Todo = {
  userId: number;
  id: number;
  title: string;
  completed: boolean;
};
type DataFromFetcher = {
  data: Todo | null;
  error: any;
};

async function fetcher(
  url: string
): Promise<DataFromFetcher> {
  try {
    const response = await fetch(url);
    const json: Todo = await response.json();
    return {
      data: json,
      error: null,
    };
  } catch (error) {
    return {
      data: null,
      error: error,
    };
  }
}

(async () => {
  let todos: Todo[] = [];
  // 配列はイテラブルなので for...of で反復処理できる
  for (const url of endPoints) {
    // 分割代入
    const { data, error } = await fetcher(url);
    // 二重否定で null でないことを判定
    if (!!data) todos = [...todos, data];
    if (error) console.error(error);
  }
  console.log({ todos });
})();
```

こちらのコードの方が型についてはスッキリしていますね。

## 型表現の多様さ

実際、型の表現をいかにするかということが TypeScript の難しいところでもあり、醍醐味だとも思います。関数の返り値などは意図に応じて様々な型の表現がありえます。例えば次のような２つの数値の大きさを比較する関数を考えてみましょう。

```ts
function compare(
  a: number,
  b: number
): number { // ただの数値型
  return a === b
    ? 0
    : (a > b ? 1 : -1);
}
```

a と b の数値を比較して等しいなら `0` を返して、a の方が大きいなら `1` を返し、b の方が大きいなら `-1` を返すような関数の戻り値は上のように単純に `number` 型として注釈もできますが、下のようにより具体的な３つの数値リテラルのいづれかというリテラル型のユニオン型で型注釈も可能です。

```ts
function compare(
  a: number,
  b: number
): -1 | 0 | 1 { // リテラル型のユニオン型
  return a === b
    ? 0
    : (a > b ? 1 : -1);
}
```

このように型の表現というのはいくらでもあります。

また、今まで学んでてきたものは基本的な型表現であり、それら以外にも Typescript ではジェネリクスなどを組み合わせた汎用的で便利な型をいくつか用意しています。こういった型はユーティリティ型 (Utility type ) と呼ばれいくつも種類が存在しています。

:::details ユーティリティ型 (Utility type)

以下の公式ハンドブックのページに網羅されています。いきなり全部覚えるのは難しそうなので少しづつ覚えていきます。

- [TypeScript: Documentation - Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)

例えば、ジェネリクスで型引数として指定したオブジェクトの型のプロパティをすべて必須にする `Required<Type>` というユーティリティ型があります。この型の型引数として指定したオブジェクトなどの型のプロパティはオプショナルプロパティとして定義されていてもすべて必須にした型を作成してくれます。

```ts
type Props = {
  a?: number; // optional property
  b?: string; // optional property
}
const obj1: Props = { a: 1 }; // b は省略可能

const obj2: Required<Props> = {
  a: 42, // 両方とも省略できない
  b: "文字列", // 両方とも省略できない
};

// プロパティを省略すると型エラーになる
const obj3:  Required<Props> = {
  a: 42,
};
// Property 'b' is missing in type '{ a: number; }' but required in type 'Required<Props>'
```

`Required<Type>` とは逆に型引数に指定したオブジェクトの型のプロパティをすべてオプショナルなものとした型を作成できる `Partial<Type>` というユーティリティ型も存在しています。

```ts
type Props = {
  a: number; // 省略できない
  b: string; // 省略できない
};

const obj1: Partial<Props> = {
  a: 42,
}; // b も a も省略可能な型
// { a?: number; b?: string; } という型注釈と同じ
```

ユーティリティ型は一見難しそうに見えますが、ジェネリクスの概念と型の基礎がわかっていれば全然難しくありません。
:::

## 型チャレンジ

TypeScript の非同期処理は JavaScript の非同期処理に過ぎません。ジェネリクスや型変数・型引数を理解できればある程度の型注釈はできるようになります。

ただし、複雑な関数の戻り値などの型注釈をしっかりしようと思うとやはり TypeScript の型の記述方法についてより詳しく知る必要があるでしょう。

以下のように **type challenges** というコミュニティによる型表現のテストがあるので型についての基礎が理解できたらぜひ取り組んでいきましょう。
