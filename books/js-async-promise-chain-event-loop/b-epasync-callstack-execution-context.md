---
title: "コールスタックと実行コンテキスト[作成中]"
---

# 実行コンテキストとコールスタックの関係

『What the heck is the event loop anyway?』の動画でみてもらったように、コールスタックには関数などが積まれているように見えますが、正確にはコールスタックに積まれるのは実行コンテキスト(Execution context)と呼ばれるものです。

実行コンテキストとはなんでしょうか。ECMA 2022 の仕様 9.4 の項目をみてみます。

https://tc39.es/ecma262/#sec-execution-contexts

>**An execution context is a specification device that is used to track the runtime evaluation of code** by an ECMAScript implementation. **At any point in time, there is at most one execution context per agent that is actually executing code**. This is known as **the agent's running execution context**. All references to the running execution context in this specification denote the running execution context of the surrounding agent.
>
>**The execution context stack is used to track execution contexts**. **The running execution context is always the top element of this stack**. A new execution context is created whenever control is transferred from the executable code associated with the currently running execution context to executable code that is not associated with that execution context. **The newly created execution context is pushed onto the stack and becomes the running execution context**.
>(上記ページより引用)

引用で示されているように、実行コンテキスト(Execution context)はコードの実行時評価を追跡するために使用される機構であり、どの時点でも実際にコードを実行している実行コンテキストはエージェント(Agent)あたり最大で１つとなります。これはエージェントの実行中実行コンテキスト(**Agent's runninng execution context**)として知られています。

実行コンテキストスタック(Execution context stack)は、実行コンテキスト(Execution context)を追跡するために使用され、実行中の実行コンテキスト(**Running execution context**)は常にこのスタックの最上位の要素です。新しく作成された実行コンテキスト(Execution context)はスタックにプッシュされて、実行中の実行コンテキスト(**Runninng execution context**)になります。

:::message
ちなみに ECMA の仕様で言われている実行コンテキストスタック(Execution context stack)はお馴染みのコールスタック(**Call stack**)のことを指しています。
:::

実行コンテキストがコールスタックに積まれていくものであるというのはなんとなく分かったと思いますが、理解するにはもう少し情報が必要ですね。MDN と freeCodeCamp の記事を見てみましょう。

https://www.freecodecamp.org/news/execution-context-how-javascript-works-behind-the-scenes/

https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth#javascript_execution_contexts


ブラウザでは、JavaScript コードを直接的に理解できないので、機械語に変換される必要があります。例えば、HTML のパース中に `<script>` タグを介してブラウザが JavaScript コードにエンカウントしたりすると、ブラウザはそのコードを JavaScript エンジン(Chrome なら V8)へと送信します。

:::message
この本で解説するブラウザ環境と JavaScript エンジンはそれぞれ Chrome と V8 エンジンを想定しています。
:::

V8 エンジン(JavaScript エンジン)がコードを受信すると、その JavaScript コードの変換と実行を処理するための特別な環境、つまり実行コンテキスト(Execution context)を作成します。実行コンテキストには、現在実行中のコードと、その実行を支援するためのすべての情報が含まれており、実行コンテキストがコールスタックに積まれ、スタックの最上位要素である Running execution context となることで実際に実行が行われます。具体的には、コードはパーサーによって解析されて、変数と関数をメモリに格納し、実行可能なバイトコードが生成されてから、コードが実行されます。

JavaScript のコードが実行されるとき、そのコードは実行コンテキスト(Execution context)内部で実行されます。コードが作成する実行コンテキストには３つの種類があります。

:::message
名称に揺れがあるのは、仕様で記載されているものと簡易的に呼ばれる通称名があるためで、どちらも記載しておきます。
:::

- (A) Global Execution Context (GEC): **グローバルコンテキスト(Global context)**。これはユーザーコードの本体(main body)を実行する際に作成される実行コンテキストです。つまり、JavaScript の関数の外側に存在するあらゆるコードは、それが実行される際にグローバルコンテキストが作成されます。
- (B) Functional Execution Context (FEC): 関数実行コンテキストまたは関数コンテキストと呼ばれるコンテキストです。各関数は、それ自身の実行コンテキスト内で実行されます。この実行コンテキストは**ローカルコンテキスト(Local context)**とも呼ばれます。
- (C) Eval Function Execution Context: 現在は非推奨な関数である `eval()` 関数によって作成される実行コンテキスト。これについては基本的には考えなくて良いです。

基本的に実行コンテキストを考える際には (A) と (B) だけを考えればよいです。

# ブラウザ環境のコールスタック

例えば、ブラウザ環境において、次のような `index.html` と `main.js` ファイルがあったとします。この場合にどのようなことが起こるかを考えてみます(ブラウザ環境でやっているのは実際にコールスタックがどうなるかを確認するためです)。

```html:index.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <script src="main.js"></script>
  </body>
</html>
```

```js:main.js
function first() {
  console.log("👻 [2] Functional Execution Context: [first]");
  second();
  console.log("👻 [4] Functional Execution Context: [first]");
}
function second() {
  console.log("🦄 [3] Functional Execution Context: [second]");
}

console.log("🦖 [1] Global execution context: [Script File]");
first();
console.log("🦖 [5] Gobal execution context: [Script File]");
```

HTML のパースによって `<script>` タグが読み込まれます。`src` として指定されている `main.js` ファイルの JavaScript コードは JavaScript エンジン(V8)に送信されて、グローバルコンテキスト(Global Execution Context)がデフォルトの実行コンテキストとして作成されます。JS エンジンは実行を開始して、コールスタックの一番下に配置します。

コールスタックはスタックですから、LIFO(Last In, First Out)です。

`main.js` ファイルを見てもらえば分かるように、最初に実行されるのは、`console.log()` です。

![Execution Context Stack](/images/js-async/img_executionContextStack_1.jpg)



![Callstack in browser](/images/js-async/img_callstackInBrowser.png)