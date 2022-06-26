---
title: "参考文献"
aliases: [ch_参考文献]
---

# [はじめに](1-epasync-begin) (完了)

[JSの非同期処理を理解するために必要だった知識と学習ロードマップ](https://zenn.dev/estra/articles/js-async-programming-roadmap)


# 非同期 API と環境

# コールバック関数の同期実行と非同期実行 (完了)
[Callback function (コールバック関数) - MDN Web Docs 用語集: ウェブ関連用語の定義 | MDN](https://developer.mozilla.org/ja/docs/Glossary/Callback_function)
# 同期 API とブロッキング (完了)
[スレッドセーフ - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89%E3%82%BB%E3%83%BC%E3%83%95)
[ブロッキングとノンブロッキングの概要 | Node.js](https://nodejs.org/ja/docs/guides/blocking-vs-non-blocking/)

# [イベントループの概要と注意点](2-epasync-event-loop) (完了)

[Writing a JavaScript framework - Execution timing, beyond setTimeout - RisingStack Engineering](https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/)
[Tasks, microtasks, queues and schedules - JakeArchibald.com](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

# [Promise の基本概念](a-epasync-promise-basic-consept) (完了)

[promises-unwrapping/states-and-fates.md at master · domenic/promises-unwrapping](https://github.com/domenic/promises-unwrapping/blob/master/docs/states-and-fates.md)
[What is the correct terminology for javascript promises - Stack Overflow](https://stackoverflow.com/questions/29268569/what-is-the-correct-terminology-for-javascript-promises/29269515#29269515)

# [コールスタックと実行コンテキスト](b-epasync-callstack-execution-context) (完了)

[ECMAScript® 2023 Language Specification](https://tc39.es/ecma262/#sec-execution-contexts)
[JavaScript Execution Context – How JS Works Behind The Scenes](https://www.freecodecamp.org/news/execution-context-how-javascript-works-behind-the-scenes/)
[In depth: Microtasks and the JavaScript runtime environment - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth#javascript_execution_contexts)
[zero-cost async stack traces // slidr.io](https://slidr.io/bmeurer/zero-cost-async-stack-traces-1#27)

# [それぞれのイベントループ](c-epasync-what-event-loop)



# [Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func) (完了)

[Promise() コンストラクター - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise)
[When to use a function declaration vs. a function expression](https://www.freecodecamp.org/news/when-to-use-a-function-declarations-vs-a-function-expression-70f15152a0a0/)
[アロー関数 (arrow function) | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/functions/arrow-functions)
[javascript - Using _ (underscore) variable with arrow functions in ES6/Typescript - Stack Overflow](https://stackoverflow.com/questions/41085189/using-underscore-variable-with-arrow-functions-in-es6-typescript)
[JavaScript: 通常の関数とアロー関数の違いは「書き方だけ」ではない。異なる性質が10個ほどある。 - Qiita](https://qiita.com/suin/items/a44825d253d023e31e4d)
[従来の関数とアロー関数の違い | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/functions/function-expression-vs-arrow-functions)



# [then メソッドのコールバックで非同期処理](10-epasync-dont-use-side-effect) (完了)

[Node.js 18 is now available! | Node.js](https://nodejs.org/en/blog/announcements/v18-release-announce/)

# [古い非同期 API を Promise でラップする](12-epasync-wrapping-macrotask) (完了)

[プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E5%8F%A4%E3%81%84%E3%82%B3%E3%83%BC%E3%83%AB%E3%83%90%E3%83%83%E3%82%AF_api_%E3%82%92%E3%83%A9%E3%83%83%E3%83%97%E3%81%99%E3%82%8B_promise_%E3%81%AE%E4%BD%9C%E6%88%90)
[Promisification](https://ja.javascript.info/promisify)

# resolve 関数と reject 関数の使い方 (完了)
[Understanding Promises in JavaScript: Part V - Resolved Promises and Promise Fates | Saurabh Misra](https://www.saurabhmisra.dev/promises-in-javascript-resolved-promise-fates)


# [Promise チェーンから async 関数へ](14-epasync-chain-to-async-await) (完了)

[文と式 · JavaScript Primer #jsprimer](https://jsprimer.net/basic/statement-expression/#expression)

# [V8 エンジンによる async/await の内部変換](15-epasync-v8-converting) (完了)

[Faster async functions and promises · V8](https://v8.dev/blog/fast-async#await-under-the-hood)
[Normative: Reduce the number of ticks in async/await by MayaLekova · Pull Request #1250 · tc39/ecma262](https://github.com/tc39/ecma262/pull/1250)
[Zero-cost async stack traces - Google ドキュメント](https://docs.google.com/document/d/13Sy_kBIJGP0XT34V1CV3nkWya4TwYx9L3Yv45LdGB6Q/)
[徹底解説！ return promiseとreturn await promiseの違い](https://zenn.dev/uhyo/articles/return-await-promise)
[Resolve Javascript Promise outside the Promise constructor scope - Stack Overflow](https://stackoverflow.com/questions/26150232/resolve-javascript-promise-outside-the-promise-constructor-scope)
[return と return await の3つの違い](https://zenn.dev/azukiazusa/articles/difference-between-return-and-return-await#%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9%E3%81%AE%E5%87%BA%E5%8A%9B)
[JavaScriptの非同期処理をじっくり理解する (3) async/await](https://zenn.dev/qnighy/articles/3a999fdecc3e81#%E9%9D%9E%E5%90%8C%E6%9C%9F%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B9)
[node.js - JS Promise's inconsistent execution order between nodejs versions - Stack Overflow](https://stackoverflow.com/questions/62032674/js-promises-inconsistent-execution-order-between-nodejs-versions)

# [Top-level await](16-epasync-top-level-async) (完了)

[JavaScript modules · V8](https://v8.dev/features/modules)
[JavaScript モジュール - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Modules)
[Top-level await · V8](https://v8.dev/features/top-level-await)
[top-level awaitがどのようにES Modulesに影響するのか完全に理解する - Qiita](https://qiita.com/uhyo/items/0e2e9eaa30ec2ff05260)

# [Promise の静的メソッドと並列化](17-epasync-static-method) (完了)

[Promise combinators · V8](https://v8.dev/features/promise-combinators)

# Promise の型注釈 (完了)
[Promise / async / await | TypeScript入門『サバイバルTypeScript』](https://typescriptbook.jp/reference/promise-async-await)

# Deno
[std@0.145.0 | Deno](https://deno.land/std@0.145.0)

# Node
[About this documentation | Node.js v18.2.0 Documentation](https://nodejs.org/dist/v18.2.0/docs/api/documentation.html)

# 仕様書
[HTML Standard](https://html.spec.whatwg.org/multipage/webappapis.html)
[ECMAScript® 2023 Language Specification](https://tc39.es/ecma262/)


