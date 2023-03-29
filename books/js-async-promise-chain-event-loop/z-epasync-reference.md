---
title: "参考文献"
cssclass: zenn
date: 2022-06-30
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
aliases: Promise本『参考文献』
---

## [第１章 - API を提供する環境と実行メカニズム](sec-01-epasync)

### [非同期 API と環境](f-epasync-asynchronous-apis)

- [JavaScriptとは · JavaScript Primer #jsprimer](https://jsprimer.net/basic/introduction/#javascript-ecmascript)
- [Async Await JavaScript Tutorial – How to Wait for a Function to Finish in JS](https://www.freecodecamp.org/news/async-await-javascript-tutorial/)
- [WinterCG(Web-interoperable Runtimes Community Group)](https://wintercg.org)
- [Inside look at modern web browser (part 2) - Chrome Developers](https://developer.chrome.com/blog/inside-browser-part2/)
- [What the heck is the event loop anyway? – Philip Roberts](https://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html)
- [Angular Basics: Introduction to Processes, Threads—Web UI](https://www.telerik.com/blogs/angular-basics-introduction-processes-threads-web-ui-developers)
- [A Taste of JavaScript's New Parallel Primitives - Mozilla Hacks - the Web developer blog](https://hacks.mozilla.org/2016/05/a-taste-of-javascripts-new-parallel-primitives/)

MDN
- [How to use promises - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises#conclusion)
- [サードパーティ API(Third-party APIs)](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Client-side_web_APIs/Third_party_APIs#restful_api_%E2%80%94_nytimes)
- [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)
- [Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API)
- [Web Worker の使用 - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers)
- [Atomics - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Atomics)
- [SharedArrayBuffer - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)

### [同期 API とブロッキング](f-epasync-synchronus-apis)

- [スレッドセーフ - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89%E3%82%BB%E3%83%BC%E3%83%95)
- [ブロッキングとノンブロッキングの概要 | Node.js](https://nodejs.org/ja/docs/guides/blocking-vs-non-blocking/)

### [イベントループの概要と注意点](2-epasync-event-loop)

- [Writing a JavaScript framework - Execution timing, beyond setTimeout - RisingStack Engineering](https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/)
- [Tasks, microtasks, queues and schedules - JakeArchibald.com](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

### [タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)

- [JavaScriptの非同期処理をじっくり理解する (1) 実行モデルとタスクキュー](https://zenn.dev/qnighy/articles/345aa9cae02d9d)

MDN
- [setInterval() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/setInterval)
- [Using microtasks in JavaScript with queueMicrotask() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [queueMicrotask() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask)
- [JavaScript で queueMicrotask() によるマイクロタスクの使用 - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/HTML_DOM_API/Microtask_guide#enqueueing_microtasks)
- [MutationObserver() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver/MutationObserver)
- [In depth: Microtasks and the JavaScript runtime environment - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth)

### [V8 エンジンについて](e-epasync-v8-engine)

- [V8 JavaScript engine](https://v8.dev/)
- [JavaScript V8 Engine Explained | HackerNoon](https://hackernoon.com/javascript-v8-engine-explained-3f940148d4ef)
- [JS Visualizer 9000](https://www.jsv9000.app/)
- [ブラウザ JavaScript / Node.js の仕組みを知ろう！ ～トラブルに迅速に立ち向かえる様に - Qiita](https://qiita.com/megmogmog1965/items/e180d02be711cecdc038)
- [JavaScriptがブラウザでどのように動くのか | メルカリエンジニアリング](https://engineering.mercari.com/blog/entry/20220128-3a0922eaa4/)
- [GoogleChromeLabs/jsvu: JavaScript (engine) Version Updater](https://github.com/GoogleChromeLabs/jsvu)
- [fishで「パスを通す」ための最終解答](https://zenn.dev/estra/articles/zenn-fish-add-path-final-answer)
- [Using d8 · V8](https://v8.dev/docs/d8)

MDN
- [並行モデルとイベントループ - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/EventLoop)

### [コールスタックと実行コンテキスト](b-epasync-callstack-execution-context)

- [JavaScript Execution Context – How JS Works Behind The Scenes](https://www.freecodecamp.org/news/execution-context-how-javascript-works-behind-the-scenes/)
- [zero-cost async stack traces // slidr.io](https://slidr.io/bmeurer/zero-cost-async-stack-traces-1#27)

Spec
- [ECMAScript® 2023 Language Specification](https://tc39.es/ecma262/#sec-execution-contexts)

MDN
- [In depth: Microtasks and the JavaScript runtime environment - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth#javascript_execution_contexts)

### [それぞれのイベントループ](c-epasync-what-event-loop)

- [Timers | Node.js v18.4.0 Documentation](https://nodejs.org/api/timers.html#timers-promises-api)
- [timers: run nextTicks after each immediate and timer by apapirovski · Pull Request #22842 · nodejs/node](https://github.com/nodejs/node/pull/22842)
- [Microtask queue is not handled correctly in Deno · Issue #11731 · denoland/deno](https://github.com/denoland/deno/issues/11731)
- [Faster async functions and promises · V8](https://v8.dev/blog/fast-async#tasks-vs.-microtasks)
- [JS Visualizer 9000](https://www.jsv9000.app/)
- [Further Adventures of the Event Loop - Erin Zimmer - JSConf EU 2018 - YouTube](https://www.youtube.com/watch?v=u1kqx6AenYw)
- [libevent](https://libevent.org/)
- [Blink Scheduler - Google ドキュメント](https://docs.google.com/document/d/11N2WTV3M0IkZ-kQlKWlBcwkOkKTCuLXGVNylK5E2zvc/edit)
- [Blink Scheduler](https://chromium.googlesource.com/chromium/src/+/master/third_party/blink/renderer/platform/scheduler/README.md)
- [Scheduling docs](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/platform/scheduler/links.md)
- [Prioritize important incoming tasks in renderers - Google ドキュメント](https://docs.google.com/document/d/1SWpjgtwIaL_hIcbm6uGJKZ8o8R9xYre-yG0VDOjFBxU/edit)
- [JavaScript のスレッド並列実行環境](https://nhiroki.jp/2017/12/10/javascript-parallel-processing#1-%E3%83%AC%E3%83%B3%E3%83%80%E3%83%AA%E3%83%B3%E3%82%B0%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%B3%E3%81%A8-javascript-%E3%81%AE%E5%AE%9F%E8%A1%8C%E3%83%A2%E3%83%87%E3%83%AB)
- [Rendering Performance](https://web.dev/rendering-performance/)
- [How Blink works - Google ドキュメント](https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg/)
- [Life of a Pixel - Google スライド](https://docs.google.com/presentation/d/1boPxbgNrTU0ddsc144rcXayGA_WF53k96imRH8Mp34Y/edit#slide=id.ga884fe665f_64_6)
- [Jake Archibald: In The Loop - setTimeout, micro tasks, requestAnimationFrame, requestIdleCallback, … - YouTube](https://www.youtube.com/watch?v=cCOL7MC4Pl0)
- [深入探究 eventloop 与浏览器渲染的时序问题 - 404Forest](https://404forest.com/2017/07/18/how-javascript-actually-works-eventloop-and-uirendering/)
- [Timing of microtask triggered from requestAnimationFrame · Issue #2637 · whatwg/html](https://github.com/whatwg/html/issues/2637)
- [The Node.js Event Loop, Timers, and process.nextTick() | Node.js](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#event-loop-explained)
- [2016 Node Interactive.pdf - Google ドライブ](https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view?resourcekey=0-lR-GaBV1Bmjy086Fp3J4Uw)
- [timers: run nextTicks after each immediate and timer by apapirovski · Pull Request #22842 · nodejs/node](https://github.com/nodejs/node/pull/22842)
- [Event Loop performance different in different version · Issue #3512 · nodejs/help](https://github.com/nodejs/help/issues/3512)
- [Learn Node.js, Unit 5: The event loop - YouTube](https://www.youtube.com/watch?v=X9zVB9WafdE&list=TLGGmD0fij1sF90wNTA1MjAyMg)
- [IBM Developer](https://developer.ibm.com/tutorials/learn-nodejs-the-event-loop/#why-you-need-to-understand-the-event-loop)

Spec
- [Event loops | HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3)

MDN
- [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)
- [Blob.text() - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Blob/text)
- [The event loop - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop#event_loop)
- [Overview of the RenderingNG architecture - Chrome Developers](https://developer.chrome.com/articles/renderingng-architecture/)
- [Web Worker の使用 - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers)

## [第２章 - Promise インスタンスと連鎖](sec-02-epasync)

### [Promise の基本概念](a-epasync-promise-basic-concept)

- [promises-unwrapping/states-and-fates.md at master · domenic/promises-unwrapping](https://github.com/domenic/promises-unwrapping/blob/master/docs/states-and-fates.md)
- [What is the correct terminology for javascript promises - Stack Overflow](https://stackoverflow.com/questions/29268569/what-is-the-correct-terminology-for-javascript-promises/29269515#29269515)

### [Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)

- [When to use a function declaration vs. a function expression](https://www.freecodecamp.org/news/when-to-use-a-function-declarations-vs-a-function-expression-70f15152a0a0/)
- [アロー関数 (arrow function) | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/functions/arrow-functions)
- [javascript - Using _ (underscore) variable with arrow functions in ES6/Typescript - Stack Overflow](https://stackoverflow.com/questions/41085189/using-underscore-variable-with-arrow-functions-in-es6-typescript)
- [JavaScript: 通常の関数とアロー関数の違いは「書き方だけ」ではない。異なる性質が10個ほどある。 - Qiita](https://qiita.com/suin/items/a44825d253d023e31e4d)
- [従来の関数とアロー関数の違い | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/functions/function-expression-vs-arrow-functions)

MDN
- [Promise() コンストラクター - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise)

### [コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async)

MDN
- [Callback function (コールバック関数) - MDN Web Docs 用語集: ウェブ関連用語の定義 | MDN](https://developer.mozilla.org/ja/docs/Glossary/Callback_function)

### [resolve 関数と reject 関数の使い方](g-epasync-resolve-reject)

- [Understanding Promises in JavaScript: Part V - Resolved Promises and Promise Fates | Saurabh Misra](https://www.saurabhmisra.dev/promises-in-javascript-resolved-promise-fates)

### [複数の Promise を走らせる](5-epasync-multiple-promises)

- [JS Visualizer 9000](https://www.jsv9000.app/)

### [then メソッドは常に新しい Promise を返す](6-epasync-then-always-return-new-promise)

- [JavaScript Promiseの本](https://azu.github.io/promises-book/#then-return-new-promise)

### [Promise chain で値を繋ぐ](7-epasync-pass-value-to-the-next-chain)

- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)

### [then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)

- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)

MDN
- [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises)

### [Promise chain はネストさせない](9-epasync-dont-nest-promise-chain)

- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)

MDN
- [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises)

### [コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)

- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)
- [Node.js 18 is now available! | Node.js](https://nodejs.org/en/blog/announcements/v18-release-announce/)

### [アロー関数で return を省略する](11-epasync-omit-return-by-arrow-shortcut)

- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)

### [catch メソッドと finally メソッド](h-epasync-catch-finally)

MDN
- [Promise.prototype.catch() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch)
- [Promise.prototype.finally() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally)

### [古い非同期 API を Promise でラップする](12-epasync-wrapping-macrotask)

- [Promisification](https://ja.javascript.info/promisify)

MDN
- [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)

### [イベントループは内部にネストしたループがある](13-epasync-loop-is-nested)

- [Writing a JavaScript framework - Execution timing, beyond setTimeout - RisingStack Engineering](https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/)
- [Tasks, microtasks, queues and schedules - JakeArchibald.com](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [JavaScriptの非同期処理をじっくり理解する (1) 実行モデルとタスクキュー](https://zenn.dev/qnighy/articles/345aa9cae02d9d)
- [Further Adventures of the Event Loop - Erin Zimmer - JSConf EU 2018 - YouTube](https://www.youtube.com/watch?v=u1kqx6AenYw)

## [第３章 - async 関数と await 式の挙動](sec-03-epasync)

### [Promise chain から async 関数へ](14-epasync-chain-to-async-await)

- [文と式 · JavaScript Primer #jsprimer](https://jsprimer.net/basic/statement-expression/#expression)

### [V8 エンジンによる async/await の内部変換](15-epasync-v8-converting)

- [Faster async functions and promises · V8](https://v8.dev/blog/fast-async#await-under-the-hood)
- [Normative: Reduce the number of ticks in async/await by MayaLekova · Pull Request #1250 · tc39/ecma262](https://github.com/tc39/ecma262/pull/1250)
- [Zero-cost async stack traces - Google ドキュメント](https://docs.google.com/document/d/13Sy_kBIJGP0XT34V1CV3nkWya4TwYx9L3Yv45LdGB6Q/)
- [徹底解説！ return promiseとreturn await promiseの違い](https://zenn.dev/uhyo/articles/return-await-promise)
- [Resolve Javascript Promise outside the Promise constructor scope - Stack Overflow](https://stackoverflow.com/questions/26150232/resolve-javascript-promise-outside-the-promise-constructor-scope)
- [return と return await の3つの違い](https://zenn.dev/azukiazusa/articles/difference-between-return-and-return-await#%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9%E3%81%AE%E5%87%BA%E5%8A%9B)
- [JavaScriptの非同期処理をじっくり理解する (3) async/await](https://zenn.dev/qnighy/articles/3a999fdecc3e81#%E9%9D%9E%E5%90%8C%E6%9C%9F%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9)
- [node.js - JS Promise's inconsistent execution order between nodejs versions - Stack Overflow](https://stackoverflow.com/questions/62032674/js-promises-inconsistent-execution-order-between-nodejs-versions)

### [Top-level await](16-epasync-top-level-async)

- [JavaScript modules · V8](https://v8.dev/features/modules)
- [Top-level await · V8](https://v8.dev/features/top-level-await)
- [top-level awaitがどのようにES Modulesに影響するのか完全に理解する - Qiita](https://qiita.com/uhyo/items/0e2e9eaa30ec2ff05260)

MDN
- [JavaScript モジュール - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Modules)

## [第４章 - 制御と型注釈](sec-04-epasync)

### [Promise の静的メソッドと並列化](17-epasync-static-method)

- [Promise combinators · V8](https://v8.dev/features/promise-combinators)

MDN
- [Promise.allSettled() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
- [Promise.all() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)

### [await 式の配置による制御](18-epasync-await-position)

- [require-await | deno_lint docs](https://lint.deno.land/?q=require-await#require-await)
- [JavaScriptの非同期処理をじっくり理解する (4) AbortSignal, Event, Async Context](https://zenn.dev/qnighy/articles/772f632af595aa)
- [オブジェクト · JavaScript Primer #jsprimer](https://jsprimer.net/basic/object/#optional-chaining-operator)

MDN
- [AbortController - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/AbortController)
- [AbortSignal - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/AbortSignal)

### [反復処理の制御](19-epasync-async-loop)

MDN
- [Array.prototype.reduce() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce)
- [for await...of - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/for-await...of)

### [イテレータとイテラブルとジェネレータ関数](k-epasync-iterator-generator)

- [JavaScript の イテレータ を極める！ - Qiita](https://qiita.com/kura07/items/cf168a7ea20e8c2554c6)
- [JavaScript の ジェネレータ を極める！ - Qiita](https://qiita.com/kura07/items/d1a57ea64ef5c3de8528)

MDN
- [反復処理プロトコル - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Iteration_protocols)
- [計算プロパティ名 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Object_initializer#%E8%A8%88%E7%AE%97%E3%83%97%E3%83%AD%E3%83%91%E3%83%86%E3%82%A3%E5%90%8D)
- [for...of - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/for...of)
- [反復可能オブジェクトを期待する構文 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Iteration_protocols#%E5%8F%8D%E5%BE%A9%E5%8F%AF%E8%83%BD%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%82%92%E6%9C%9F%E5%BE%85%E3%81%99%E3%82%8B%E6%A7%8B%E6%96%87)
- [Promise.all() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
- [Symbol.asyncIterator - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Symbol/asyncIterator)
- [AsyncGenerator - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncGenerator)

### [TypeScript における Promise の型注釈](j-epasync-ts-promise-type-annotation)

- [Promise / async / await | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/promise-async-await)
- [タプル (tuple) | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/values-types-variables/tuple)
- [イベントループと TypeScript の型から理解する非同期処理](https://zenn.dev/mizchi/articles/understanding-promise-by-ts-eventloop)
- [JavaScript - TypeScript Deep Dive 日本語版](https://typescript-jp.gitbook.io/deep-dive/recap)
- [TypeScript: Documentation - TypeScript for JavaScript Programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html)
- [TypeScript: Documentation - Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [deno lint rule no-inferrable-types](https://lint.deno.land/?q=infer#no-inferrable-types)
- [deno lint rule no-explicit-any](https://lint.deno.land/?q=any#no-explicit-any)
- [Async Await try-catch hell - YouTube](https://www.youtube.com/watch?v=ITogH7lJTyE)

## [第５章 - 仕様およびその他の番外編](sec-05-epasync)

### [Promise.prototype.then の仕様挙動](m-epasync-promise-prototype-then)

- [Creating a JavaScript promise from scratch, Part 1: Constructor - Human Who Codes](https://humanwhocodes.com/blog/2020/09/creating-javascript-promise-from-scratch-constructor/)
- [Creating a JavaScript promise from scratch, Part 2: Resolving to a promise - Human Who Codes](https://humanwhocodes.com/blog/2020/09/creating-javascript-promise-from-scratch-resolving-to-a-promise/)
- [Creating a JavaScript promise from scratch, Part 3: then(), catch(), and finally() - Human Who Codes](https://humanwhocodes.com/blog/2020/10/creating-javascript-promise-from-scratch-then-catch-finally/)
- [Creating a JavaScript promise from scratch, Part 4: Promise.resolve() and Promise.reject() - Human Who Codes](https://humanwhocodes.com/blog/2020/10/creating-javascript-promise-from-scratch-promise-resolve-reject/)
- [humanwhocodes/pledge: A custom promise implementation for JavaScript](https://github.com/humanwhocodes/pledge)
- [Understanding the ECMAScript spec, part 1](https://v8.dev/blog/understanding-ecmascript-part-1)
- [How to Read the ECMAScript Specification](https://timothygu.me/es-howto/)

### [Promise chain と async/await の仕様比較](n-epasync-promise-spec-compare)

- [Normative: Reduce the number of ticks in async/await by MayaLekova · Pull Request #1250 · tc39/ecma262](https://github.com/tc39/ecma262/pull/1250/files)
- [tc39/proposal-faster-promise-adoption: Reduce the number of microtask ticks required to adopt the state of a promise](https://github.com/tc39/proposal-faster-promise-adoption)


## その他

### Deno

[std@0.145.0 | Deno](https://deno.land/std@0.145.0)

### Node

[Node.js v18.2.0 Documentation](https://nodejs.org/dist/v18.2.0/docs/api/documentation.html)

### 仕様書

- [HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html)
- [ECMAScript® 2023 Language Specification](https://tc39.es/ecma262/)
