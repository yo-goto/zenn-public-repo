---
title: "Promise の型注釈"
cssclass: zenn
date: 2022-07-12
modified: 2024-08-14
AutoNoteMover: disable
tags: type/zenn/book, JavaScript/async
aliases: Promise本『Promise の型注釈』
---

## このチャプターについて

このチャプターでは、TypeScript での Promise の型注釈について考えてみます。async 関数では本質的には「**Promise オブジェクトを扱っている**」ということを意識することが重要なので、TypeScript を使って型注釈を通じて async 関数などを考えてみるのは理解の上で重要となります。

:::message alert
TypeScript について詳しくない場合には、このチャプターを読む前に前のチャプター『[TypeScript の基本知識](j-epasync-ts-basic)』を読むようにしてください。
:::

## Promise の型注釈

さて、Promise の型注釈をするには今まで見てきたジェネリクスの概念と型引数・型変数が必要です。まずは簡単な変数宣言から型注釈を始めてみます。次のように文字列を履行値として直ちに履行する Promise インスタンスを作成して変数に代入します。

```js:JavaScript
const sp = new Promise(resolve => {
  resolve("文字列で履行する");
});
```

この変数 `sp` には Promise オブジェクトが割り当てられるので Promise の型を注釈したいと思います。Promise オブジェクトの型はジェネリクスを使って `Promise<Type>` というような形式で記述できます。`Type` は上で見たような型変数であり、型注釈する際には実際に存在する型名を型引数として指定します。では型変数にどのような型引数を指定すればよいかというと、履行値の型名を指定してあげるのが基本となります。

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

一般化して考えると、Promise は値を入れ込むことができたのでその値の型を型引数として指定するわけです。

### Response 型

非同期 API である `fetch()` メソッドはその結果である `Response` オブジェクトを Promise の中に入れ込んで結果として返してくれました。そういう訳で `fetch()` から返ってくる Promise インスタンスを代入する変数には `Promise<Response>` というような型注釈ができます。

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

### 履行値が無い場合の Promise の型注釈

Promise コンストラクタの `resolve()` 関数に何も引数を渡さない場合、履行値は `undefined` ちなります。この場合、Promise オブジェクトに対してどのように型付けするかといえば、大抵の場合 `Promise<void>` という型注釈をします。

```ts
const p1: Promise<void> = new Promise(resolve => {
  resolve(); // 引数を何も渡さない場合には undefined が履行値になる
});

// Promise.resolve() でも同じように undefined で履行される
const p2: Promise<void> = Promise.resolve();
```

履行値が `undefined` なので、もちろん `Promise<undefined>` としても良いわけですが、`undefined` を強調したいなどの理由がない限りは `Promise<void>` としておくのが一般的です。

:::message
`void` 型は戻り値が無い関数の型に利用され、以下のように `return` 文が無い関数の戻り値の型注釈として使われます。

```ts:戻り値が無い関数の型注釈はvoidとする
function sayHello(): void {
  console.log("Hello!");
}
const u = sayHello(); // => 実際には undefined が返る
```

JavaScript の関数は戻り値が無い場合には値として `undefined` を返します。

そして、`void` 型は `undefined` 型の上位型(supertype)であり、以下のように `undefined` 型の値を `void` 型の変数に割り当てることはできますが、逆はできません。

```ts:voidはundefinedの上位型
const v: void = undefined; // void型はundefined型の値を割当可能

const u: undefined = v; // 逆はできないので型エラー
//    ^ Error: Type 'void' is not assignable to type 'undefined'.(2322)
```

したがって、意図的に `undefined` を `return` するような関数の戻り値の型を `void` としても型エラーにはならない正しいコードとなります。

```ts:voidによる関数の型注釈
function noR1(): void {
  return undefined;
}

function noR2(): void {
  return;
}

function noR3(): void {
}
```

もちろん、そのように書けたとしても `void` 型はそもそも戻り値が無いということを表現する型なので、`return` 文がそもそも無いような `no3` のような関数の戻り値の型注釈として使うのが適切です。
:::

`undefined` を使うのは例えば、`number | undefined` など他の型との合成型を作る場合などです。以下のようなコードでは数値 `42` が履行値としてあり得るので `Promise<number | undefined>` として型注釈します。

```ts
const p: Promise<number | undefined> = new Promise(resolve => {
  const value = Math.random() < 0.5 ? 42 : undefined;
  //    ^: number | undefined として型推論される
  resolve(value);
});
```

履行値を特に意識しない、最初から無いことがわかっているような場合には `undefined` は特に使う必要なく `Promise<void>` として扱えばいいです。

実際、`Promise.resolve()` の場合には、型注釈を省略した場合でも `Promise<void>` として型推論されます。

```ts
const p1 = Promise.resolve();
//    ^: Promise<void> と型推論される
```

:::message
なお、拒否状態の Promise オブジェクトの場合には `Promise<never>` として型推論されます。

```ts
const up = Promise.reject(42);
//    ^: Promise<never> として型推論される
```
:::

Promise コンストラクタで記述する場合、`resolve()` に引数を渡さない状態で型注釈をしないとエラーになります。

```ts:この場合には型エラーになる
const p2 = new Promise(resolve => {
  resolve();
  // Error: Expected 1 arguments, but got 0. Did you forget to include 'void' in your type argument to 'Promise'?
  // (value: unknown) => void として型推論される
});
```

このエラーは引数が 1 つ必要だが 0 つしか渡されていないというものです。この場合、`Promise<void>` という型注釈をすることで `resolve()` 関数の型推論が修正されてエラーを回避できます。

```ts:型エラーにならない
const p2: Promise<void> = new Promise(resolve => {
  resolve();
  // (value: void | PromiseLike<void>) => void として型推論される
});
```

履行値が無い Promise オブジェクトが返るというのは意外とよくあるケースであり、次に説明する Promise オブジェクトを返す関数や戻り値が無い async 関数などの注釈には `Promise<void>` がよく使われます。例えば、Promise 化された `setTimeout()` によるタイマー関数などを考えると以下の様に `setTimeout()` の第一引数には `resolve` 関数をそのまま渡しており、実行されるときには引数なしで行われます。

```ts
const delay = (ms: number): Promise<void> => {
  //                        ^^^^^^^^^^^^: この型注釈がないと Promise<unknown> として推論されてしまう
  return new Promise(resolve => {
    setTimeout(resolve, ms);
    //         ^: コールバックに渡したresolve関数は引数なしで実行されるので履行値は無い(undefined)
  });
};
```

こういった場合には型注釈を意図的に施さないと、関数の戻り値の型が `Promise<unknown>` として推論されてしまうので、`Promise<void>` として型注釈するか、`new Promise<void>` としてジェネリック関数の呼び出しを行うようにするのがいいでしょう。

## Promise を返す関数の型注釈

さて、上ですでに例を見ましたが、Promise インスタンスを返す関数は次のように返り値の型注釈を書きます。履行値の値の型を `Promise<Type>` の型引数として指定することで、この関数から返ってくる Promise インスタンスがどのような値をもっているのかがわかりやすくなりますね。

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

:::details インターフェース型(interface type)

オブジェクトの型を定義するために型エイリアス(type alias)が使えましたが、インターフェース型(interface type)はオブジェクトの型を定義するためのもう１つの方法です。型エイリアスはあらゆる型に名前を付けるものとして使えますが、インターフェース型はオブジェクトやクラスの型を表現するためのものとして利用できます。

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

また、`.d.ts` という拡張子のファイルは[型定義ファイル](https://typescriptbook.jp/reference/declaration-file)と呼ばれるものです。TypeScript が提供する型や npm のパッケージ配布を行うために作成されます。
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

この JSDoc で関数の説明や引数・返り値の説明などを加えることによってエディタ上で使い方などが表示されるようになります。TypeScript そのものとは関係ないですが、組み合わせて使えます。TypeScript の公式ドキュメントにもリファレンスが記載されています。

- 参考: [TypeScript: Documentation - JSDoc Reference](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html)
:::

ということで、通常は省略しますが `Promise.resolve()` や `then()` メソッドを呼び出す際には型引数に明示的にコールバック関数から返される値の型を指定できます。

```ts
Promise.resolve<string>("string") // 履行値の型を明示的に型引数として指定
  .then<number>(val => val.length) // コールバックの返り値の型を型引数として指定
  .then<void>(val => console.log(val)); // コールバックの返り値の型を型引数として指定
```

実は、`Promise()` というコンストラクタ関数もジェネリック関数なので、明示的に型引数を指定できます(通常は省略できます)。

```ts
// 履行値は number 型なので型引数に number 型を指定
const p = new Promise<number>(resolve => resolve(42));
```

従って、履行値が無い場合の Promise については以下のように書くこともできます。

```ts
const p1 = new Promise<void>(resolve => resolve());
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

履行値を遅延時間そのものとして引数に `number` 型の値を取るようにさせます。履行値の型が `number` 型なので返ってくる Promise インスタンスが持つ値の型が `number` 型となりますね。ということで `pTimer()` の実装は次のようになります(型注釈を加えただけです)。

```ts:TypeScript
function pTimer(
  time: number
): Promise<number> {
  return new Promise(resolve => setTimeout(() => {
    console.log(`${time}[ms]でタイムアウトしました`);
    resolve(time);
  }, time));
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

今までは返り値の型がシンプルなものしか見てきませんでしたが、この場合には例外捕捉の際に分岐するので、それぞれの節で返される値の型を合成する必要がでてきます。

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
const urls: stirng[] = [
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

JavaScript で言う所の [try-catch hell](https://www.youtube.com/watch?v=ITogH7lJTyE) を避けるための `[data, error]` という２つの値からなる配列を返すパターンを使っています。このパターンを使うことで返り値の型注釈が面倒なことになっていますね(現時点ではあまりうまく型付けできていないと思います)。

要素の型がそれぞれ異なる配列は TypeScript では**タプル(Tuple)型**と呼ばれます。

https://typescriptbook.jp/reference/values-types-variables/tuple

関数は基本的に１つの値しか返せないため、戻り値を複数にしたい場合には配列に格納して返すことで実現できます。

```js:JavaScript
function reutrnMultipleValue() {
  return [42, "文字列", true];
  // 色々な型の値が要素となった配列を返す
}
```

この戻り値を型注釈するためにタプル型と呼ばれる型の書き方で注釈します。

```ts:TypeScript(タプルの型注釈)
function reutrnMultipleValue(): [number, string, boolean] {
  return [42, "文字列", true];
}
```

タプル型は変数宣言のときには次のようになります。

```ts
const list1: [number, string, boolean] = reutrnMultipleValue();

// 型エイリアスで型を作成した場合
type MyTuple = [number, string, boolean];
const list2: MyTuple = reutrnMultipleValue();
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

タプル型の値を返すことで、async 関数利用時の try-catch を大量に書くのを防ぎます。基本は async 関数に例外捕捉を閉じ込めて標準化してデータフェッチの成功時には `[data, null]` を返し、失敗時には `[null, error]` を返すようにするというパターンです。

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

JavaScript だと簡単ですが、返り値の型注釈をしっかりしようとすると以外にうまくいきません。一応は型エラーにならず動くのがこの型注釈です。

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

もっとうまい型注釈はあるだろうなと思いますが、とりあえずはこれで型エラーにはなりません(型の表現方法はいくつもあります)。

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

実際、型の表現をどのように行うかという点が TypeScript の難しいところでもあり、醍醐味でもあります。例えば、関数の返り値などは意図に応じて様々な型の表現がありえます。次のような２つの数値の大きさを比較する関数を考えてみましょう。

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

a と b の数値を比較して等しいなら `0` を返して、a の方が大きいなら `1` を返し、b の方が大きいなら `-1` を返すような関数の戻り値は上のよう単純に `number` 型として注釈もできますが、下のようにより具体的な３つの数値リテラルのいづれかというリテラル型のユニオン型で型注釈も可能です。

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

また、今まで学んでてきたものは基本的な型表現であり、それら以外にも Typescript ではジェネリクスなどを組み合わせた汎用的で便利な型をいくつか用意しています。こういった型はユーティリティ型(Utility type)と呼ばれいくつも種類が存在しています。

:::details ユーティリティ型(Utility type)

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

`Required<Type>` とは逆に、型引数に指定したオブジェクトの型のプロパティをすべてオプショナルなものとした型を作成できる `Partial<Type>` というユーティリティ型も存在しています。

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

https://github.com/type-challenges/type-challenges
