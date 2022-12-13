---
title: "同期 API とブロッキング"
cssclass: zenn
date: 2022-06-15
modified: 2022-12-03
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『同期 API とブロッキング』
---

# このチャプターについて

このチャプターでは、非同期 API と対になっている同期 API について解説します。同期 API を理解することで非同期 API の有用性と難しさについて理解できます。また、ソースコードの配置と実行タイミングの関係性について考えることができる有用なケースとなります。

# 同期 API は意図的にブロッキングする

非同期 API と環境について前のチャプターで解説しました。非同期 API のおかげで環境がバックグラウンドで API 処理をしてくれている間も、同時に JavaScript コードを実行できるのは、実際のところ非同期 API がメインスレッドを (専有しないように) ブロッキングしないよう **デザインされている** からです。

https://nodejs.org/ja/docs/guides/blocking-vs-non-blocking/

その一方で、Node 環境や Deno 環境では意図的にブロッキングを起こすようにデザインされた「**同期 API(Synchronous API)**」が存在しています。`console.log()` などの Web APIs(Web Platform APIs) は置いておいて、そういった API は名前の最後が `Sync` で終わるケースのものとして提供されています (I/O 関連の処理など)。

:::message
Node や Deno では HTTP やファイルシステムにアクセスする機能を API として提供していますが、これらの API は OS の機能を利用するので OS API(Operation System API) と呼ばれることがあります。
:::

例えば、ファイルへの書き込みを行う API ですが、名前は安直に `writeFile` として Node でも Deno でも大体同じ機能で提供されています。Deno は Node の後発なので、**ユーザー側は Node にすでに存在している API 機能が Deno にもあるだろうと期待して探します**。そして実際にあります (開発側からしたら同じ機能は同じ名前にするのが妥当でしょう)。ただし、`writeFile` と言っても上で説明したような非同期型と同期型が以下のように存在しており、同期型は両方とも `Sync` で名前が終わっていることが分かります。

:::message
「非同期 API / 同期 API」という呼称だと理解が難しい場合には、「Non-blocking API / Blocking API」をメインに考えた方が分かりやすいかもしれません。
:::

- 非同期 API (Non-blocking API)
  - Node:
    - [fs.writeFile](https://nodejs.org/dist/v18.2.0/docs/api/fs.html#fswritefilefile-data-options-callback) (Callback-based API[^callback-based])
    - [fsPromises.writeFile](https://nodejs.org/dist/v18.2.0/docs/api/fs.html#fspromiseswritefilefile-data-options) (Promise-based API)
  - Deno: [Deno.writeFile](https://doc.deno.land/deno/stable/~/Deno.writeFile) (Promise-based API)
- 同期 API (Blocking API)
  - Node: [fs.wirteFileSync](https://nodejs.org/dist/v18.2.0/docs/api/fs.html#fswritefilesyncfile-data-options)
  - Deno: [Deno.wirteFileSync](https://doc.deno.land/deno/stable/~/Deno.writeFileSync)

  [^callback-based]: Node に存在している古いタイプのタスクベースの API です。これを使って逐次処理を行うには Callback hell をつくることになります。

`fs.writeFileSync` や `Deno.writeFileSync` などの同期 API を使うことで **ソースコードの配置と処理順番が完全一致するようにできます**。つまり、コード配置と実行順序がずれてしまう難しい非同期処理を考える必要がなくなります。ただし、同時に複数のことができる非同期 API のメリットを捨て去ることになります。

例えば、Deno 環境においてテキストファイルにデータを書きこんだ後に読みこんでコンソールに出力することを同期 API(`Deno.writeTextFileSync` と `Deno.readTextFileSync`) と非同期 API(`Deno.writeTextFile` と `Deno.readTextFile`) のそれぞれで考えてみます。

```js:apiSync.js(同期APIを利用したコード)
// apiSync.js
const path = "./tests/helloSync.txt";
const inputData = "Hello Synchronously Zenn!";

// 書いた順番に実行される(ソースコードの配置順番どおりにしている)
console.log("[1]");

Deno.writeTextFileSync(path, inputData); // blocking
// この処理が完了してから次の処理に進む
const data = Deno.readTextFileSync(path); // blocking
// この処理が完了してから次の処理に進む
console.log("[2]", data);

console.log("[3]");
```

同期 API は意図的にメインスレッドをブロッキングするようデザインされているので、ソースコードを上から下に書いた順番そのままで実行し、一行ずつ完了するのを待って次の行に移動します。実際に上のコードを実行すると次の出力を得ます。

```sh
# read と write のパーミッションが必要なので --allow-all で代用
❯ deno run --allow-all apiSync.js
[1]
[2] Hello Synchronously Zenn!
[3]
```

そのため、最後の `console.log()` がファイルの書き込み・読み出しの完了が終わってから実行されています。

一方、非同期 API を利用した場合はどうなるでしょうか。Deno の非同期 API である `Deno.writeTextFile()` と `Deno.readTextFile()` はそれぞれ Promise インスタンスを返してきますので、Promise chain が構築できます (API の名前の最後に `Sync` がついていないことに注意してください)。

```js:apiAsync.js(非同期APIを利用したコード)
// apiAsync.js
const path = "./tests/helloAsync.txt";
const inputData = "Hello Asynchronously Zenn!";

// 書いた順番に実行されない(ソースコードの配置順番どおりにしていない)
console.log("[1]");

// この処理の完了を待たずに次の処理に進む
Deno.writeTextFile(path, inputData) // non-blocking
  .then(() => Deno.readTextFile(path)) // non-blocking
  .then((data) => console.log("[3]", data));

// 上の処理の完了を待たずにコンソールに出力
console.log("[2]");
```

非同期 API はメインスレッドをブロッキングしないようにデザインされているので、ソースコードは上から下に書いたとおりには実行されません。その代わりに、環境が時間のかかる非同期 API の処理を裏で行っている間も同時に別のことができます。実際に上のコードを実行すると次の出力を得ます。

```sh
# read と write のパーミッションが必要なので --allow-all で代用
❯ deno run --allow-all apiAsync.js
[1]
[2]
[3] Hello Asynchronously Zenn!
```

そのため、ファイルの書き込み・読み出しの完了を待たずに最後の `console.log()` を実行できています。

アプリケーションではなく書き捨てのスクリプトや簡単なテストではこの「同期 API(Blocking API)」が役立ちます。実際、この程度のスクリプトなら非同期にしても大した意味がない (時間的な効率は問題にならない) ので同期で書くのが多いのではないでしょうか (ただし、非同期 API や非同期処理を理解するためにはこのぐらいの短さで考えた方がいいです)。

というのも、同期 API では **書いた順番通りに実行されるので明らかに処理の流れが分かりやすい** からです。ただし、一度に複数のことができる非同期 API のメリットを捨てることになるので、明らかに非同期 API より時間がかかることになります。逆に、非同期 API は「**処理の流れが分かりづらいが効率が良い**」ということです。

:::message
**まとめ**

- 同期 API(Blocking API) : 書いた順番通りに実行されるので処理の流れがわかりやすいが、意図的に同時に１つのことしかやらないようにしているので効率は悪い。
- 非同期 API(Non-blocking API) : 書いた順番通りに実行されないため処理の流れがわかりづらく制御は難しいが、同時に複数のことができるため効率は良い。後続の関連作業の順序を制御するための書き方が、Callback のネスト形式や Promise chain、async/await となる。
:::

# 「実行と完了」の順番を保証する書き方

結局のところ Promise などの非同期のシンタックスは非同期 API が登場しない限り出番がありません。そして「**同時に複数のことをやりたい**」がためにわざわざ難しい非同期 API を使います。同時に複数のことをやっているので、バックグラウンドでの API 処理が完了したら、**その処理に関連する別作業** をメインスレッドで行うための「ソースコードの書き方」が必要になります。そのシンタックス (書き方) が Callback のネスト形式 (Callback hell) や Promise chain、async/await です。

ソースコード全体で考えた時には配置順番通りに実行がされなくても、非同期 API を起点としたある箇所について「実行と完了」自体が特定の順番に起きるよう適切に書くことが重要です。例えば、「ファイルにデータを書き込む」ことを行ってから「ファイルのデータを読み出す」というのも、「書き込みが完了してから読み出しを実行する」という実行と完了の順番が担保されるようにしていることを意味しています。

```js:Promise chain
// [A] -> [B] -> [C] という順番で実行と完了がなされるのを保証する
Deno.writeTextFile(path, inputData) // [A]
  .then(() => Deno.readTextFile(path)) // [B]
  .then((data) => console.log("[3]", data)); // [C]
```

```js:async/await
// 上のコードを async/await で書き換えた
(async function writeThenRead() {
  // [A] -> [B] -> [C] という順番で実行と完了がなされるのを保証する
  await Deno.writeTextFile(path, inputData); // [A]
  const data = await Deno.readTextFile(path); // [B]
  console.log("[3]", data); // [C]
})(); // 即時実行
```

:::message alert
もちろん、現実的にはエラーハンドリングが付き纏うので完全な保証ではないです。上の例も説明のために例外処理を省いていますので注意してください。
:::

Callback 形式なら適切にネストさせることで、Promise chain なら適切に連鎖させることで、async/await なら適切に await することで、コード全体での順序では **時間的に非連続であったとしても注目している特定の範囲内に存在するコードの実行と完了の順番を保証させます**。

ちなみに、Callback-based API や Promise-based API の処理オーダー(順番) が重要であることが Node API Document の fs の項目に直接的に言及されています。

> Because **they are executed asynchronously by the underlying thread pool**, there is no guaranteed ordering when using either the callback or promise-based methods.
> (中略)
> **It is important to correctly order the operations by awaiting the results of one before invoking the other**:
> (中略)
> Or, when using the callback APIs, **move the `fs.stat()` call into the callback of the `fs.rename()` operation**:
> ([File system | Node.js v18.2.0 Documentation](https://nodejs.org/dist/v18.2.0/docs/api/fs.html#ordering-of-callback-and-promise-based-operations) より引用、太字は筆者強調)

また、以下の様に Deno の非同期 API でドキュメントのサンプルに常に `await` キーワードが付いているのは Promise-based API であることを示すのと同時に、完了を担保してから次の処理を行うケースが一般的だからという理由が考えられます。

```js
await Deno.writeTextFile("hello1.txt", "Hello world\n");  // overwrite "hello1.txt" or create it
```

https://doc.deno.land/deno/stable/~/Deno.writeTextFile

このように、非同期 API を起点にした一連の関連作業が特定順序で実行・完了されることを保証するための正しい書き方とその仕組みを学ぶということが非同期処理というテーマでの学習です。つまり、時間的には非連続になる可能性のある「逐次 (sequential) 処理」をどうやって書いて、どういうメカニズムでその処理が実現されているのかを知ることが重要ということです。

# 非同期 API の利用が目的

前のチャプターで『[非同期処理の目的と仕組み](f-epasync-asynchronous-apis#非同期処理の目的と仕組み)』について解説しましたが、結局の所はあえて非同期のシンタックスそのものを使いたい理由があって使うのではなく、「**同時に複数のことができる性質を持った非同期 API を使用するという目的**」を達成するために難しい非同期処理のシンタックス (Promise chain, async/await) を書かざる負えないというケースが多いだけです。あるいは、特定ライブラリの async 関数を使う際に非同期のシンタックスが必要となる場合などでも、結局のところ **内部的に非同期 API を利用している抽象化されたメソッド** を使いたいがために書かざる負えないということになります (import で利用する async 関数は内部で Promise インスタンスを返すような処理や非同期 API が利用されており、ラップする際には await するために async 関数で書く必要がある)。

Promise や async/awiat などの処理は、結果としてタイミングがずれて非同期的に処理されてしまうコードであるというだけで、非同期のシンタックスそのものは目的 (やりたいこと) ではないケースがほとんどです。ユーザーインタラクションやイベント処理、`setTimeout()` などのタイマー処理によってスケジューリングすることで意図的に非同期にすることを除けば、**効率の良い非同期 API を使用したいがために結果的に付随する処理がタイミングのズレる非同期処理になってしまう** という訳です。

同時に複数のことをやることで効率的な処理をしたいから非同期 API を使うのであって、タイミングのズレる非同期処理がやりたから Promise chain やら async/await を使うのではないです。

実際に MDN のドキュメントでは、async/await の目的が Promise-based API の利用のためであることがで明言されています。

> Note: async/await の目的は、プロミスベースの API を利用するのに必要な構文を簡素化することです。 async/await の動作は、ジェネレータとプロミスの組み合わせに似ています。
> ([非同期関数 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/async_function) より引用)

別のチャプターで解説しますが、イベントループとコールスタックによって、非同期 API の処理結果と次の処理をメインスレッドに集約的に通知させます。同時に複数のことをやるが、その結果を使った処理を再度１つのスレッド (=１つのコールスタック) に集めて、次の処理を実行したり、別の非同期 API を起動させたりするという一連の作業順番を制御するために Promise chain や async/await を書く必要があるということです。そして、その結果としてコード上の配置と実行タイミングがずれて処理されるコードがでてくるということです。

基本的には、非同期のシンタックスを書くことは「結果的に使わざる負えない手段」であって「目的」ではないです。

ユーザーインタラクションのイベント処理やタイマー系で意図的に非同期にしてスケジューリングすることなどを除けば、処理の流れが分かりずらなくなる非同期処理をわざわざ書かなくても良いなら使う必要は一切ありません。

# アンチパターンを知る

非同期 API や非同期処理の書き方は **アンチパターンを知ることも重要** になってきます。

正しい書き方を行わない場合、例えば Promise chain なら Promise を返す処理を `return` せずに「副作用 (Side Effect)」として使用してしまったり、async/await なら適切に `await` しないことで、「実行と完了」の順番を担保できなくなります。場合によっては I/O で競合が起きたり、意図した結果とならないケースがでてきます。

```js:副作用にしてしまったアンチパターン
Deno.writeTextFile(path, inputData) // [A]
  .then(() => {
    Deno.readTextFile(path);
    // return していないので副作用となる
  }) // [B]
  .then((data) => console.log("[3]", data)); // [C] undefined が出力される
```

```js:適切にawaitしないアンチパターン
(async function writeAndRead() {
  // [A] と [B] が競合し、データの書き込み→読み込みという完了すべき作業の順番を担保できない
  Deno.writeTextFile(path, inputData); // [A]
  // [A] の完了に関わらず [B] を開始
  const data = Deno.readTextFile(path); // [B]
  console.log("[3]", data); // [C]
  // そもそも Proimise インスタンスから値が取り出せていないので Promise{ <pending> } が出力される
})();
```

Deno でも考え方は同じですが、Node では、Promise-based な File System 系の API 操作は [スレッドセーフ](https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89%E3%82%BB%E3%83%BC%E3%83%95) ではないので同時に同じファイルを修正してしまうような場合に注意するように書かれています。

> The promise APIs use the underlying Node.js threadpool to perform file system operations off the event loop thread. These operations are **not synchronized or threadsafe**. Care must be taken **when performing multiple concurrent modifications on the same file or data corruption may occur**.
> ([File system | Node.js v18.2.0 Documentation](https://nodejs.org/dist/v18.2.0/docs/api/fs.html#promises-api) より引用、太字は筆者強調)

上のコードの例では同時に書き込みを行ってはいませんが、「書き込んだデータを読み出す」という意図のコードを書くなら、ファイル書き込みが完了してからファイル読み込みを行うのが良いでしょう (つまり副作用にしないことや適切に await することで実行と完了の順序を決める)。

非同期 API を使って「同時に複数のことができる」からといって **競合するような複数の API 操作は同時にしてはいけない** ということです。それらは順番に完了を待って行うようにしましょう。特定の実行順番が守られている操作群に対して関係の無い操作を同時に行うのなら大丈夫です。もちろん同期 API を使うならそもそも同時に複数のことをやらないので、そういった心配は必要ないです。

その一方で、関連のない複数操作を並列化 (非同期 API 処理は同時に複数実行できる) させて、効率化を測ることができます。複数の Promise 処理を１つずつ await するのではなく、処理を起動した後で `Promise.allSetteld()` などの静的メソッドでまとめて await する (`await Promise.allSettled([...promises])`) ことで並列化できます。これについては第３章の『[Promise の静的メソッドと並列化](17-epasync-static-method)』のチャプターであらためて解説します。

# API の補足と標準モジュール

## API としてのグローバルオブジェクト

API という言葉は非常にわかりづらく、とにかく曖昧なので読む人によっては非常に混乱させる用語です。文脈や視点に応じて ECMAScript のビルトインオブジェクトあるいはコンストラクタ関数のことを API と呼んでいる人も見かけます。JavaScript エンジンの外側からそれらを見た時には API と呼べるかもしれませんが、基本的に混乱の元になるので、この本では JavaScript エンジンから呼び出せるエンジンの外側にある環境から提供される機能のことを API として呼ぶことにして、ビルトインオブジェクトのことはビルトインオブジェクトと明言するようにしてます。(初学者はこれに気をつけないと意味不明になります)。

:::message alert
従って、他の解説で「Promise API」と言っている場合には二通りの可能性があることに注意してください。

基本的には `fetch` などの Promise-based API のことを言っている場合が多いですが、ECMAScript のビルトインオブジェクトである Promise コンストラクタや `Promise.all()` などの静的メソッドについてわざわざ「API」として述べている場合があります。
:::

## Deno globals と Deno Std

`Deno` から名前が始まる API は「Deno globals」と呼ばれ、公式の API ドキュメントでは「Deno CLI APIs」とも呼ばれています。つまり CLI(コマンドランインターフェース) のための API であるため、必ずしも Node の API と対応付けることができません。環境 (直接的には Deno の実行ファイル) が提供する API の機能とは別に Deno では「標準モジュール (Standard module: Deno std)」というウェブ上で配布されているモジュールが存在しており、これらは API とは呼ばれていません。

> These modules do not have external dependencies and they are reviewed by the Deno core team. The intention is to have a standard set of high quality code that all Deno projects can use fearlessly.
> ([std@0.145.0 | Deno](https://deno.land/std@0.145.0) より引用)

標準モジュールの実装は TypeScript で行われており、内部を見ると Deno globals の API や標準モジュール自体を組み合わせることでより **実用性の高いユーティリティ機能を実現したもの** を配布しているようです。それぞれの機能ごとに `fs` (File System) や `http`、`io` などのモジュールに分割されています。

例えば、ファイルをコピーするための機能として提供されている `copy` 関数を次のように URL から `import` を行うことでコードをダウンロード・キャッシュして使えるようにします。

```js
import { copy } from "https://deno.land/std@0.145.0/fs/mod.ts";
// URL から標準モジュールである copy 関数をインポートする

const src = "./txtFiles/test1.md";
const dist = "./txtFiles/copied.md";

try {
  // コピーしてから読み出す
  await copy(src, dist); // Promise インスタンスを返すので await
  const text = await Deno.readTextFile(dist); // Promise インスタンスを返すので await
  console.log(text);
} catch (err) {
  console.error(err);
}
```

この fs モジュールの `copy` という関数は公式リポジトリの以下の場所に存在しています。`export async function copy` で非同期の `copy` 関数としてエクスポートされているものです。

https://github.com/denoland/deno_std/blob/0.145.0/fs/copy.ts#L85-L269

ソースコードを見てみると、内部的にはエクスポートされていない `copyFile` 関数などが使われており、さらにそれらは内部で `Deno.lstat` や `Deno.copyFile` などの Deno globals(CLI API) が利用されています。

要するに、Deno globals などの API はより低レイヤーのコア機能として Deno のバイナリに格納されており (コード上ではグローバルネームスペース)、コアではないユーティリティ機能はそこから標準モジュール (std) として分離して提供されていると解釈できます。Deno ではもともとは CLI API として提供されていたものが std に方に機能移行されることなどもあります (`Deno.Buffer` → `io/buffer`)。

標準モジュールでも同期型・非同期型のものがあり、名前も同期型のものは `Sync` で終わります。同期型は内部で Deno globals の同期 API が使われているためブロッキングします。非同期型は async 関数として定義されており、内部的にも Deno globals の非同期 API が利用されていたりします (それらの非同期 API は Promise インスタンスを返すので内部で await 評価した時にそれを利用する関数は async 化する必要がある)。

## Node API

`path` モジュールなどは顕著ですが、Node の `path` モジュールに準ずる機能の殆どは Deno では標準モジュールの `path` として提供されています。そもそも Node のモジュールも API と呼べるのか怪しい部分がありますが、Node API Document として提供されているので API と認めてもよいでしょう。

https://nodejs.org/dist/v18.2.0/docs/api/path.html

https://deno.land/std@0.145.0/path

ちなみに、Deno の std に対して、Node の場合なら File System 系の便利なユーティリティ機能を提供する OSS の `fs-extra` というパッケージなどがあります。Deno の std の fs モジュールとして提供されている関数と同じ名前のもの `ensure` 系統などが提供されていますが、Node の公式ライブラリではなくサードパーティのパッケージとして存在しています。

https://github.com/jprichardson/node-fs-extra

内部で依存関係として持っているのは `node-graceful-fs` というパッケージです。ソースコードを見ると Node の fs モジュールの API が利用されています。

https://github.com/isaacs/node-graceful-fs

Deno の std ではそういったユーティリティ機能を公式の開発チームによってメンテしつつ提供するようにしたものだと考えられます。
