---
title: "TypeScript の基本知識"
cssclass: zenn
date: 2024-08-15
modified: 2024-08-14
AutoNoteMover: disable
tags: type/zenn/book, JavaScript/async
aliases: Promise本『TypeScript の基本知識』
---

## このチャプターについて

次のチャプターでは TypeScript における Promise の型注釈を考えていきますが、その前に TypeScript の基本知識について抑えておきましょう。

TypeScript の基本知識については日本語でオープンソースに公開されており、筆者もお世話になっている『[サバイバル TypeScript](https://typescriptbook.jp)』が非常に参考になります。具体的な Promise の型注釈および async 関数における型注釈については以下のページで解説されています。

https://typescriptbook.jp/reference/promise-async-await

また、以下の mizchi さんの記事でも Promise の型注釈について詳細に説明されています。

https://zenn.dev/mizchi/articles/understanding-promise-by-ts-eventloop

このようにすでに Promise の型注釈については有用な記事などがいくつかあるのですが、ここでは初学者の目線からアウトプットを兼ねて筆者なりの解釈で TypeScript についての基礎と Promise の型注釈に必要な知識を説明してみたいと思います(型注釈については基本的なことしか解説しませんのでより応用的な内容は別のリソースに頼ってください)。

:::message
非同期処理の本質的な部分は JavaScript で見てきたもので十分なので、このチャプターは「非同期処理での型注釈を通して逆に TypeScript を理解してみよう」という試みになります。

また、このチャプターは『[TypeScript の基礎から Promise の型注釈まで駆け登る](https://zenn.dev/estra/articles/ts-with-promise-type-annotation)』の記事と同じ内容になるので、すでに読まれた方はスキップしてもらって構いません。
:::

## TypeScript について

TypeScript は JavaScript に[型システム](https://www.wikiwand.com/ja/%E5%9E%8B%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0)を導入した言語です。

言語の基底となる JavaScript のことををしっかり学べば TypeScript は怖くありません。逆に TypeScript から入ってしまうと JavaScript の機能に加えて型情報の操作や型チェックのエラーといった学ぶべき事柄が膨大になるので圧倒されてしまいます。

これは TypeScript を理解するためには JavaScript の知識が欠かせないということでもあります。『[サバイバルTypeScript](https://typescriptbook.jp/)』でも次のように言われています。

> TypeScriptから見ると、JavaScriptはTypeScriptの一部と言えます。そのため、TypeScriptを十分に理解するには、JavaScriptの理解が欠かせません。まだJavaScriptをよく分かっていない場合は、TypeScriptの学習と平行してJavaScriptも学ぶ必要があります。
> ([JavaScriptはTypeScriptの一部 | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/overview/javascript-is-typescript) より引用)

そして JavaScript での非同期処理が理解できれば TypeScript の非同期処理は恐るるに足りません。『[TypeScript Deep Dive](https://typescript-jp.gitbook.io/deep-dive/recap)』でも次のように言われています。

> TypeScriptは、単に、JavaScriptのコードを良いドキュメントにする方法を標準化したものに過ぎません。
> (...中略)
> 本質的には、TypeScriptはJavaScriptのリンター(コードの静的解析ツール)です。型情報を持たない他のJavaScriptのリンターよりも優れているだけです。
> ([JavaScript - TypeScript Deep Dive 日本語版](https://typescript-jp.gitbook.io/deep-dive/recap) より引用)

TypeScript はより良い JavaScript を書くためのリンターに過ぎません。つまり JavaScript を書くための道具です。

そして、TypeScript の非同期処理は **JavaScript の非同期処理のコードに型情報を上乗せしたもの** であり、本質的には Promise や async/await といった JavaScript(ECMAScript) の非同期シンタックスやその処理を実現するためのイベントループの機構、ランタイム(JS エンジン)を埋め込んでいる環境とそこから提供される非同期 API を[理解すれば良い訳です](https://zenn.dev/estra/articles/js-async-programming-roadmap)。つまり、**「非同期処理」を理解するために必要な知識そのものと TypeScript には殆ど関係性がありません**。

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

(もちろん型の再利用や Narrowing など TypeScript に特化した難しさはありますが)こういう恐れすぎない心持ちのもとで学習を進めていきます。非同期処理についても JavaScript から始めて型無しで学んだ後に「より堅牢なコードを書くために TypeScript による型注釈を加えて扱うデータに対しての具象性を高めていく」という考えのもとで進めていきます。

実際、**JavaScript と TypeScript の境界線がどこにあるのかを意識することでスッキリと理解できる場合が多いです**。また、何か分からないことがでてきた場合も、問題を解決するために調べる必要のあるレイヤーがどれか分かることは非常に重要です。TypeScript についてわからないと思っていたことが実は ECMAScript のシンタックスだったり(その場合は MDN で調べる)、ECMAScript の関数が分からないと思っていたらその関数は JavaScript 実行環境が独自定義する API だったり(その場合はランタイム環境のマニュアルや API ドキュメントで調べる)、あるいは型ガード関数という TypeScript 独自の書き方で型の解析に利用するものだったり(その場合は TypeScript Handbook で調べる)と、境界線が分かっていないと調べる領域を間違ってしまう場合があるのでかなり効率が悪くなってしまいます。

そういったことを踏まえて、JavaScript をすでに知っている学習者は TypeScript 公式ハンドブックの『TypeScript for JavaScript Programmers』の項目を読むことで JavaScript から TypeScript にする方法の概要を短い時間で学ぶことができます。TypeScript に特化した機能がなんなのか分かってしまえば、学ぶべき量がそこまで多くないことが分かります(もちろん少なくはないですが、TypeScript だけで学ぼうとする場合よりも遥かに少ないことが認識できます)。

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

VS Code などを使っていれば Deno 専用の拡張機能を入れることによってエディタ上でリンターを使えます。

https://marketplace.visualstudio.com/items?itemName=denoland.vscode-deno

そして、Deno では V8 エンジンがランタイムになっています。TypeScript は JavaScript へとトランスパイル(コンパイルの一種)を行うことで実際には JavaScript を JavaScript エンジンで動かしているに過ぎません。TypeScript でのエラーはコンパイル時の型チェックエラーと実際に JavaScript として動かした時のランタイムエラーとなります。

型チェックでエラーがでても JavaScript エンジンでランタイムエラーがでないで JavaScript として正しく動く場合もあります。JavaScript として正しく動いたとしても、型チェックで意図的に警告を出させることで、実際に動かす前によりよいコードを書くように書き直す機会を得ることができます。

## 型注釈の基本

TypeScript では既存の JavaScript コードに型の情報を付与していくことから学習が始まります。

### 変数への型注釈

例えば、文字列リテラルの値で初期化した変数に明示的に `string` 型であると型の情報を付与することが型注釈(type annotation)と呼ばれる行為です。

変数に型を注釈するには変数名の後に `:` を付けて型の名前を書きます。JavaScript のプリミティブ型である文字列型なら `string` というように決まった型の名前があるのでそれを変数名の後に追加します。

```ts
const str1 = "文字列"; // JavaScript
const str2: string = "文字列の型注釈を追加"; // TypeScript
//          ^^^^^^ string 型の型注釈
//                 str2 は string 型だよとコンパイラに伝えているだけ
```

上のコードでは文字列リテラルで初期化しているので明らかに文字列型(`string` 型)であることがコンパイラは推論できるので上の場合には必ずしも書く必要がありません。

TypeScript のコンパイラは賢いので型注釈を省略してもある程度は推論してくれます。従って、次のように型注釈を省略しても TypeScript ではコードとして大丈夫です。

```ts
const str3 = "文字列リテラル"; // TypeScript
// 文字列リテラルで初期化しているのは明らかであり、型注釈は省略できる
```

型を省略してもそのコードから型を推論して自動的に型情報が得られるこの機能を型推論(type inference)と言います。上のような変数宣言では初期値から型が推論されます。

Deno ではこのような明らかに型推論が容易な変数宣言ではむしろ型注釈を省略するように促すリンタールール "no-inferrable-types" がありますので、省略しないと怒られてしまいます。

https://lint.deno.land/?q=infer#no-inferrable-types

>Variable initializations to JavaScript primitives (and `null`) are obvious in their type. Specifying their type can add additional verbosity to the code. For example, with `const x: number = 5`, specifying `number` is unnecessary as it is obvious that `5` is a number.
>([deno_lint docs no-inferrable-types](https://lint.deno.land/?q=infer#no-inferrable-types) より引用)

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

JSの主要なデータ型 | 型注釈での名前 | 値
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

同じようなオブジェクトに何回もこのような型注釈をしなくてはならない場合は非常に冗長になってしまうので、同じ型を参照できるように使いまわしたいケースが多いです。`type` キーワードを使って型に名前を付けることできます。この機能を型エイリアス(Type Alias)と呼びます。

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

エイリアス(別名)なので既に存在している型に別名を付けることができます。これは別名を付けているだけで新しい型をつくっている訳ではないことに注意してください。

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
型エイリアスでメソッドを持つオブジェクトの型を１から作成してみましたが、定義したオブジェクトから型を抽出して別の場所で使い回すようなことをしたい場合もあります。そのような場合には `typeof` 型演算子(typeof type operator)を使って変数から型を抽出できます。

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

オブジェクトや配列を引数として取る関数において、引数にとるオブジェクトのプロパティや配列の要素について分割代入して関数内部でプロパティや要素を変数で扱かえるようにしたいときには分割代入の構文を関数の引数で使う「分割代入引数(destructuring assignment parameter)」の書き方が使えます。

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

**ジェネリクス(generics)** は関数のように型が引数(あるいは変数)を扱えるようにすることでより一般的な処理を記述できるようにする TypeScript の機能(あるいはその概念)です。

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

配列の要素の型は `Array<Type>` のように `Type` に `string` といった実際の型名を指定します。この `Type` のような型の変数を **型変数(type variable)** と呼びます(実際に存在している型の名前ではなく変数です)。上の場合は配列の要素の型を指定するためのものとなっていますね。

これがジェネリクスです。型が変数を使えるようになったことで、ジェネリクスを持つ配列の型注釈では、配列要素が持つことのできる値の型を記述できます。

>**Generics provide variables to types**. A common example is an array. An array without generics could contain anything. An array with generics **can describe the values that the array contains**.
>([TypeScript: Documentation - TypeScript for JavaScript Programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html#generics) より引用、太字は筆者強調)

`Array<Type>` という `<Type>` の中に実際に書く型である `string` などが **型引数(type argument)** と呼ばれるものであり、これで関数のように型に引数を指定します(実際に存在している型名を指定します)。

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

>Use of the `any` type disables the type check system around that variable, defeating the purpose of Typescript which is to provide type safe code. Additionally, the use of `any` hinders code readability, since it is not immediately clear what type of value is being referenced. It is better to be explicit about all types. For a more type-safe alternative to `any`, use `unknown` if you are unable to choose a more specific type.
>([no-explicit-any](https://lint.deno.land/?q=any#no-explicit-any) より引用)

`unknown` 型は「どんな型か分からない時に使う型」で `any` よりも安全性が高い型です。

```ts
let uarr: Array<unknown>;
let uar: unknown[];
```

また、このジェネリクスを使って自分で一般的な型を定義することも可能です。

https://www.typescriptlang.org/docs/handbook/2/objects.html#generic-object-types

例えば、`data` プロパティの値の型がなんでもいい型をつくりたい場合に次のように一々色々な具体的な型を指定した型をつくらずに、(一般化する際に `any` や `unknown` も使わないで)型引数を指定してその型に適応した型を作り出せるようにしたいです。

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

ジェネリック関数(generic function)はこのジェネリクスの概念を利用した関数になります。

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

この関数の処理は配列の要素の型に依存せず配列要素をただ返すだけなので、配列要素の型に関わらず「あらゆる配列」を受け入れるように「**一般化**」したいです。

`unknown` 型は「どんな型か分からない時に使う型」で `any` よりも安全性が高い型だという話でしたので、`unknown[]` という配列要素の型が分からない配列という型注釈はどうでしょうか。

```ts
function returnArrEl(
  arr: unknown[] // 配列要素の型が分からないという型注釈
): unknown { // 配列要素の型が分からないという型注釈
  return arr[0];
}
```

これで、どんな要素を持つ配列が来るかはわからないようにしています。これで型の情報が「一般化」されたように思えますが、型注釈をしない時に `any` 型として推論されてしまう場合と対して変わりません。この関数の利用時には返り値の型が `unknown` としてエディタでも表示されるので型の情報がほとんど何もないことになります。

型推論で配列要素の型が実際に表示されるようにしたいわけです。ジェネリクスは「一般化」を意味しますが、ここでジェネリクスを使って関数を記述することで型を一般化できます。このような関数をジェネリック関数(generic function)と呼びます。

ジェネリック関数は `Array<Type>` で見たように型変数として `Type` (実際の名前はなんでもよい)を関数名の後に `<>` をつけて定義します。関数の引数の定義と似ていますね。これによって、一般的にあらゆる型を受け入れるようにできます。

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

このように関数の入力となる値の型と、出力となる値の型に関連性がある場合には型変数を利用して**相互の型をリンクさせることができます**。以下の関数の型注釈では、型変数によって入力の値と出力の値の型が同じになるようにリンクさせています。

```ts:ジェネリック関数
function returnArrEl<Type>( // Type は型変数
  arr: Type[] // 入力と出力の値の型がリンクした
): Type { // 入力と出力の値の型がリンクした
  return arr[0];
}
```

ジェネリック関数においてこのように複数の型を型変数でパラメータ化できるため、この場合の型変数 `Type` を型パラメータ([Type parameter](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#type-parameters))と呼びます。`extends` を併用することで特定の条件を満たす型へと拘束することも可能です。

このようなジェネリック関数として定義することでより一般的な処理となる関数を書くことができます。呼び出す際に型引数(type argument)として実際に存在している型名を指定することで型を明示できます。`Array<string>` のように配列の型注釈をするのと同じように関数を使用する際に具体的な型引数を指定するわけです。

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

:::message
次の章で解説しますが、ジェネリック関数はこのようなユーザー定義の関数だけでなく、ビルトインのオブジェクトのメソッドなどでも利用されています。実際 `Promise.resolve()` などはジェネリック関数になっているので、`Promise.resolve<number>` や `new Promise<void>()` などのように履行値についての方指定ができるようになっています。

```ts
const p1 = Promise.resolve<number>(42);
//    ^: Promise<number> 型として推論される

const p2 = new Promise<string>(resolve => resolve("st"));
//    ^: Promise<string> 型として推論される
```
:::

このようなジェネリック関数では、**入力と出力の値の型がリンクしているため**、実は型引数の部分は省略しても引数の値から型推論してくれます。

```ts
// 両方とも型エラーにならない
const result1 = returnArrEl([4, 0, 3]);
// 返り値の型は number 型であるとエディタ上でしっかり表示される
console.log(result); // => 4

const result2 = returnArrEl(["A", "B", "C"]);
// 返り値の型は string 型であるとエディタ上でしっかり表示される
console.log(result2); // => "A"
```

実際には引数の配列の要素が空の場合もありえるのでより正確に型注釈するとこの関数の返り値の型は `Type | undefined` という **ユニオン型(union type)** になります。`undefined` は実際に存在する型です。配列が空の場合にはこの関数からは `undefined` という値が返ります。

```ts
function returnArrEl<Type>( // Type は型変換
  arr: Type[] // 入力と出力の値の型がリンクした
): Type | undefined { // 戻り値の型は Type または undefined
  return arr[0];
}
```

例えば `string | number` などがユニオン型ですが、これは `string` 型または `number` 型という２つの型を受け入れる合成された型です。このように２つの型を組み合わせることを「**型の合成(Composing Types)**」と呼びます。

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

こういったコードの構造に基づいて値の型をより具体的に推定できるようにすることを(型の範囲をより具体的なものに狭めることから) **Narrowing(型の絞り込み)** と呼びます(あるいはその現象そのものを Narrwing と呼びます)。つまり、型情報の選別やフィルターを行う行為がコードを書く上でも必要となります。

https://www.typescriptlang.org/docs/handbook/2/narrowing.html

上のコードでの `if` 節や `switch` や `while` などのコードの構造によって各場所での変数の型を絞り込みます。このようなコードを書くと TypeScript (コンパイラやエディタの拡張機能)はある変数が特定のブランチなどに到達した時点でその型がなんであるか解析をしています。この解析を「**制御フロー解析(Control flow analysis: CFA)**」と呼びます。

もしも、次のように `else` ブランチを増やしてもそのブランチには決して到達することはありません(`string` 型か `number` 型しか引数に受け取らないため)。

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

:::details リテラル型(literal type)

数値リテラルや文字列リテラルなどのリテラルからも型(type)は作成できます。リテラルで指定したプリミティブ型の特定の値だけを代入可能にするような型がリテラル型の特徴です。

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

:::details インターセクション型(intersection type)

型の合成としてユニオン型(union type)がありましたが、もう１つの合成の仕方としてインターセクション型(intersection type)が存在しています。

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
