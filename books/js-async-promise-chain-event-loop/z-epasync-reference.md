---
title: "参考文献"
aliases: [ch_参考文献]
---

# [はじめに](1-epasync-begin) (🦄)

- [JSの非同期処理を理解するために必要だった知識と学習ロードマップ](https://zenn.dev/estra/articles/js-async-programming-roadmap)
- [非同期処理:コールバック/Promise/Async Function · JavaScript Primer #jsprimer](https://jsprimer.net/basic/async/)
- [MDN Web Docs](https://developer.mozilla.org/ja/)
- [What the heck is the event loop anyway? | Philip Roberts | JSConf EU - YouTube](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
- [Effective Deno](https://zenn.dev/uki00a/books/effective-deno)
- [Node.jsとはなにか？なぜみんな使っているのか？ - Qiita](https://qiita.com/non_cal/items/a8fee0b7ad96e67713eb)

# [第１章 - API を提供する環境と実行メカニズム](sec-01-epasync.md)

## [非同期 API と環境](f-epasync-asyncronous-apis) (🦄)

- [How to use promises - Learn web development | MDN](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises#conclusion)
- [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)
- [JavaScriptとは · JavaScript Primer #jsprimer](https://jsprimer.net/basic/introduction/#javascript-ecmascript)
- [Async Await JavaScript Tutorial – How to Wait for a Function to Finish in JS](https://www.freecodecamp.org/news/async-await-javascript-tutorial/)
- [Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API)
- [サードパーティ API(Third-party APIs)](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Client-side_web_APIs/Third_party_APIs#restful_api_%E2%80%94_nytimes)
- [WinterCG(Web-interoperable Runtimes Community Group)](https://wintercg.org)
- [Inside look at modern web browser (part 2) - Chrome Developers](https://developer.chrome.com/blog/inside-browser-part2/)
- [What the heck is the event loop anyway? – Philip Roberts](https://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html)
- [Web Worker の使用 - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers)

## [同期 API とブロッキング](f-epasync-synchronus-apis.md) (🦄)

- [スレッドセーフ - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89%E3%82%BB%E3%83%BC%E3%83%95)
- [ブロッキングとノンブロッキングの概要 | Node.js](https://nodejs.org/ja/docs/guides/blocking-vs-non-blocking/)

## [イベントループの概要と注意点](2-epasync-event-loop) (🦄)

- [Writing a JavaScript framework - Execution timing, beyond setTimeout - RisingStack Engineering](https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/)
- [Tasks, microtasks, queues and schedules - JakeArchibald.com](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

## [タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues) (🦄)

- [JavaScriptの非同期処理をじっくり理解する (1) 実行モデルとタスクキュー](https://zenn.dev/qnighy/articles/345aa9cae02d9d)
- [setInterval() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/setInterval)
- [Using microtasks in JavaScript with queueMicrotask() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [queueMicrotask() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask)
- [JavaScript で queueMicrotask() によるマイクロタスクの使用 - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/HTML_DOM_API/Microtask_guide#enqueueing_microtasks)
- [MutationObserver() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver/MutationObserver)
- [In depth: Microtasks and the JavaScript runtime environment - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth)

## [V8 エンジンについて](e-epasync-v8-engine) (🦄)

- [V8 JavaScript engine](https://v8.dev/)
- [JavaScript V8 Engine Explained | HackerNoon](https://hackernoon.com/javascript-v8-engine-explained-3f940148d4ef)
- [JS Visualizer 9000](https://www.jsv9000.app/)
- [並行モデルとイベントループ - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/EventLoop)
- [ブラウザ JavaScript / Node.js の仕組みを知ろう！ ～トラブルに迅速に立ち向かえる様に - Qiita](https://qiita.com/megmogmog1965/items/e180d02be711cecdc038)
- [JavaScriptがブラウザでどのように動くのか | メルカリエンジニアリング](https://engineering.mercari.com/blog/entry/20220128-3a0922eaa4/)
- [GoogleChromeLabs/jsvu: JavaScript (engine) Version Updater](https://github.com/GoogleChromeLabs/jsvu)
- [fishで「パスを通す」ための最終解答](https://zenn.dev/estra/articles/zenn-fish-add-path-final-answer)
- [Using d8 · V8](https://v8.dev/docs/d8)

## [コールスタックと実行コンテキスト](b-epasync-callstack-execution-context) (🦄)

- [ECMAScript® 2023 Language Specification](https://tc39.es/ecma262/#sec-execution-contexts)
- [JavaScript Execution Context – How JS Works Behind The Scenes](https://www.freecodecamp.org/news/execution-context-how-javascript-works-behind-the-scenes/)
- [In depth: Microtasks and the JavaScript runtime environment - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth#javascript_execution_contexts)
- [zero-cost async stack traces // slidr.io](https://slidr.io/bmeurer/zero-cost-async-stack-traces-1#27)


## [それぞれのイベントループ](c-epasync-what-event-loop) (🦄)

- [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)
- [Blob.text() - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Blob/text)
- [Timers | Node.js v18.4.0 Documentation](https://nodejs.org/api/timers.html#timers-promises-api)
- [timers: run nextTicks after each immediate and timer by apapirovski · Pull Request #22842 · nodejs/node](https://github.com/nodejs/node/pull/22842)
- [Microtask queue is not handled correctly in Deno · Issue #11731 · denoland/deno](https://github.com/denoland/deno/issues/11731)
- [Faster async functions and promises · V8](https://v8.dev/blog/fast-async#tasks-vs.-microtasks)
- [JS Visualizer 9000](https://www.jsv9000.app/)
- [Further Adventures of the Event Loop - Erin Zimmer - JSConf EU 2018 - YouTube](https://www.youtube.com/watch?v=u1kqx6AenYw)
- [libevent](https://libevent.org/)
- [Blink Scheduler - Google ドキュメント](https://docs.google.com/document/d/11N2WTV3M0IkZ-kQlKWlBcwkOkKTCuLXGVNylK5E2zvc/edit)
- [Prioritize important incoming tasks in renderers - Google ドキュメント](https://docs.google.com/document/d/1SWpjgtwIaL_hIcbm6uGJKZ8o8R9xYre-yG0VDOjFBxU/edit)
- [JavaScript のスレッド並列実行環境](https://nhiroki.jp/2017/12/10/javascript-parallel-processing#1-%E3%83%AC%E3%83%B3%E3%83%80%E3%83%AA%E3%83%B3%E3%82%B0%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%B3%E3%81%A8-javascript-%E3%81%AE%E5%AE%9F%E8%A1%8C%E3%83%A2%E3%83%87%E3%83%AB)
- [The event loop - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop#event_loop)
- [Event loops | HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html#definitions-3)
- [Rendering Performance](https://web.dev/rendering-performance/)
- [Overview of the RenderingNG architecture - Chrome Developers](https://developer.chrome.com/articles/renderingng-architecture/)
- [Jake Archibald: In The Loop - setTimeout, micro tasks, requestAnimationFrame, requestIdleCallback, … - YouTube](https://www.youtube.com/watch?v=cCOL7MC4Pl0)
- [深入探究 eventloop 与浏览器渲染的时序问题 - 404Forest](https://404forest.com/2017/07/18/how-javascript-actually-works-eventloop-and-uirendering/)
- [Timing of microtask triggered from requestAnimationFrame · Issue #2637 · whatwg/html](https://github.com/whatwg/html/issues/2637)
- [Web Worker の使用 - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API/Using_web_workers)
- [The Node.js Event Loop, Timers, and process.nextTick() | Node.js](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#event-loop-explained)
- [2016 Node Interactive.pdf - Google ドライブ](https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view?resourcekey=0-lR-GaBV1Bmjy086Fp3J4Uw)
- [timers: run nextTicks after each immediate and timer by apapirovski · Pull Request #22842 · nodejs/node](https://github.com/nodejs/node/pull/22842)
- [Event Loop performance different in different version · Issue #3512 · nodejs/help](https://github.com/nodejs/help/issues/3512)
- [Learn Node.js, Unit 5: The event loop - YouTube](https://www.youtube.com/watch?v=X9zVB9WafdE&list=TLGGmD0fij1sF90wNTA1MjAyMg)
- [IBM Developer](https://developer.ibm.com/tutorials/learn-nodejs-the-event-loop/#why-you-need-to-understand-the-event-loop)

# [第２章 - Promise インスタンスと連鎖](sec-02-epasync)

## [Promise の基本概念](a-epasync-promise-basic-consept)

- [promises-unwrapping/states-and-fates.md at master · domenic/promises-unwrapping](https://github.com/domenic/promises-unwrapping/blob/master/docs/states-and-fates.md)
- [What is the correct terminology for javascript promises - Stack Overflow](https://stackoverflow.com/questions/29268569/what-is-the-correct-terminology-for-javascript-promises/29269515#29269515)

## [Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func) (🦄)

- [Promise() コンストラクター - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise)
- [When to use a function declaration vs. a function expression](https://www.freecodecamp.org/news/when-to-use-a-function-declarations-vs-a-function-expression-70f15152a0a0/)
- [アロー関数 (arrow function) | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/functions/arrow-functions)
- [javascript - Using _ (underscore) variable with arrow functions in ES6/Typescript - Stack Overflow](https://stackoverflow.com/questions/41085189/using-underscore-variable-with-arrow-functions-in-es6-typescript)
- [JavaScript: 通常の関数とアロー関数の違いは「書き方だけ」ではない。異なる性質が10個ほどある。 - Qiita](https://qiita.com/suin/items/a44825d253d023e31e4d)
- [従来の関数とアロー関数の違い | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/functions/function-expression-vs-arrow-functions)

## [コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async) (🦄)

- [Callback function (コールバック関数) - MDN Web Docs 用語集: ウェブ関連用語の定義 | MDN](https://developer.mozilla.org/ja/docs/Glossary/Callback_function)

## [resolve 関数と reject 関数の使い方](g-epasync-resolve-reject.md) (🦄)

- [Understanding Promises in JavaScript: Part V - Resolved Promises and Promise Fates | Saurabh Misra](https://www.saurabhmisra.dev/promises-in-javascript-resolved-promise-fates)

## [複数の Promise を走らせる](5-epasync-multiple-promises)

なし。

## [then メソッドは常に新しい Promise を返す](6-epasync-then-always-return-new-promise)

なし。

## [Promise チェーンで値を繋ぐ](7-epasync-pass-value-to-the-next-chain)

なし。

## [コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect) (🦄)

- [Node.js 18 is now available! | Node.js](https://nodejs.org/en/blog/announcements/v18-release-announce/)

## [アロー関数で return を省略する](11-epasync-omit-return-by-arrow-shortcut) (🦄)

なし。

## [catch メソッドと finally メソッド](h-epasync-catch-finally)

なし。

## [古い非同期 API を Promise でラップする](12-epasync-wrapping-macrotask) (🦄)

- [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)
- [Promisification](https://ja.javascript.info/promisify)

## [イベントループは内部にネストしたループがある](13-epasync-loop-is-nested)

なし。

# [第３章 - async 関数と await 式の挙動](sec-03-epasync.md)

## [Promise チェーンから async 関数へ](14-epasync-chain-to-async-await) (🦄)

- [文と式 · JavaScript Primer #jsprimer](https://jsprimer.net/basic/statement-expression/#expression)

## [V8 エンジンによる async/await の内部変換](15-epasync-v8-converting) (🦄)

- [Faster async functions and promises · V8](https://v8.dev/blog/fast-async#await-under-the-hood)
- [Normative: Reduce the number of ticks in async/await by MayaLekova · Pull Request #1250 · tc39/ecma262](https://github.com/tc39/ecma262/pull/1250)
- [Zero-cost async stack traces - Google ドキュメント](https://docs.google.com/document/d/13Sy_kBIJGP0XT34V1CV3nkWya4TwYx9L3Yv45LdGB6Q/)
- [徹底解説！ return promiseとreturn await promiseの違い](https://zenn.dev/uhyo/articles/return-await-promise)
- [Resolve Javascript Promise outside the Promise constructor scope - Stack Overflow](https://stackoverflow.com/questions/26150232/resolve-javascript-promise-outside-the-promise-constructor-scope)
- [return と return await の3つの違い](https://zenn.dev/azukiazusa/articles/difference-between-return-and-return-await#%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9%E3%81%AE%E5%87%BA%E5%8A%9B)
- [JavaScriptの非同期処理をじっくり理解する (3) async/await](https://zenn.dev/qnighy/articles/3a999fdecc3e81#%E9%9D%9E%E5%90%8C%E6%9C%9F%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9)
- [node.js - JS Promise's inconsistent execution order between nodejs versions - Stack Overflow](https://stackoverflow.com/questions/62032674/js-promises-inconsistent-execution-order-between-nodejs-versions)

## [Top-level await](16-epasync-top-level-async) (🦄)

- [JavaScript modules · V8](https://v8.dev/features/modules)
- [JavaScript モジュール - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Modules)
- [Top-level await · V8](https://v8.dev/features/top-level-await)
- [top-level awaitがどのようにES Modulesに影響するのか完全に理解する - Qiita](https://qiita.com/uhyo/items/0e2e9eaa30ec2ff05260)

## [Promise の静的メソッドと並列化](17-epasync-static-method) (🦄)

- [Promise combinators · V8](https://v8.dev/features/promise-combinators)
- [Promise.allSettled() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
- [Promise.all() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)

## [await 式の配置による制御](18-epasync-await-position) (🦄)

- [require-await | deno_lint docs](https://lint.deno.land/?q=require-await#require-await)
- [AbortController - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/AbortController)
- [AbortSignal - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/AbortSignal)
- [JavaScriptの非同期処理をじっくり理解する (4) AbortSignal, Event, Async Context](https://zenn.dev/qnighy/articles/772f632af595aa)
- [オブジェクト · JavaScript Primer #jsprimer](https://jsprimer.net/basic/object/#optional-chaining-operator)

## [反復処理の制御](19-epasync-async-loop.md) (🦄)

- [JSONPlaceholder - Free Fake REST API](https://jsonplaceholder.typicode.com/)
- [Array.prototype.reduce() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce)
- [for await...of - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/for-await...of)

## Promise の型注釈 (🦄)

- [Promise / async / await | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/promise-async-await)

# Deno (🦄)
[std@0.145.0 | Deno](https://deno.land/std@0.145.0)

# Node (🦄)
[About this documentation | Node.js v18.2.0 Documentation](https://nodejs.org/dist/v18.2.0/docs/api/documentation.html)

# 仕様書 (🦄)
[HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html)
[ECMAScript® 2023 Language Specification](https://tc39.es/ecma262/)


