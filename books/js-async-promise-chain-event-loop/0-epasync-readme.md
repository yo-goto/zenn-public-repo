---
title: "EPAsync book 管理ノート"
cssclass: zenn
date: 2022-08-14
modified: 2024-08-14
url: "https://zenn.dev/estra/books/js-async-promise-chain-event-loop"
AutoNoteMover: disable
tags: type/zenn/book, JavaScript/async
aliases:
  - EPAsync
  - Promise本『イベントループとプロミスチェーンで学ぶ非同期処理』
  - Promise本
  - イベントループ本
---

## MOC

セクション管理
- [part-01-epasync](part-01-epasync)
- [part-02-epasync](part-02-epasync)
- [part-03-epasync](part-03-epasync)
- [part-04-epasync](part-04-epasync)
- [part-05-epasync](part-05-epasync)

メイン
- [1-epasync-begin](1-epasync-begin)
- [2-epasync-event-loop](2-epasync-event-loop)
- [3-epasync-promise-constructor-executor-func](3-epasync-promise-constructor-executor-func)
- [4-epasync-callback-is-sync-or-async](4-epasync-callback-is-sync-or-async)
- [5-epasync-multiple-promises](5-epasync-multiple-promises)
- [6-epasync-then-always-return-new-promise](6-epasync-then-always-return-new-promise)
- [7-epasync-pass-value-to-the-next-chain](7-epasync-pass-value-to-the-next-chain)
- [8-epasync-return-promise-in-then-callback](8-epasync-return-promise-in-then-callback)
- [9-epasync-dont-nest-promise-chain](9-epasync-dont-nest-promise-chain)
- [10-epasync-dont-use-side-effect](10-epasync-dont-use-side-effect)
- [11-epasync-omit-return-by-arrow-shortcut](11-epasync-omit-return-by-arrow-shortcut)
- [12-epasync-wrapping-macrotask](12-epasync-wrapping-macrotask)
- [13-epasync-loop-is-nested](13-epasync-loop-is-nested)
- [14-epasync-chain-to-async-await](14-epasync-chain-to-async-await)
- [15-epasync-v8-converting](15-epasync-v8-converting)
- [16-epasync-top-level-async](16-epasync-top-level-async)
- [17-epasync-static-method](17-epasync-static-method)
- [18-epasync-await-position](18-epasync-await-position)
- [19-epasync-async-loop](19-epasync-async-loop)

サブ
- [a-epasync-promise-basic-concept](a-epasync-promise-basic-concept)
- [b-epasync-callstack-execution-context](b-epasync-callstack-execution-context)
- [c-epasync-what-event-loop](c-epasync-what-event-loop)
- [d-epasync-task-microtask-queues](d-epasync-task-microtask-queues)
- [e-epasync-v8-engine](e-epasync-v8-engine)
- [f-epasync-asynchronous-apis](f-epasync-asynchronous-apis)
- [f-epasync-synchronus-apis](f-epasync-synchronus-apis)
- [g-epasync-resolve-reject](g-epasync-resolve-reject)
- [h-epasync-catch-finally](h-epasync-catch-finally)
- [j-epasync-ts-promise-type-annotation](j-epasync-ts-promise-type-annotation)
- [k-epasync-iterator-generator](k-epasync-iterator-generator)
- [m-epasync-promise-prototype-then](m-epasync-promise-prototype-then)
- [n-epasync-promise-spec-compare](n-epasync-promise-spec-compare)
- [o-epasync-async-await-pattern-history](o-epasync-async-await-pattern-history)
- [x-epasync-epilogue](x-epasync-epilogue)
- [y-epasync-conclusion](y-epasync-conclusion)
- [z-epasync-reference](z-epasync-reference)

解析図
- [[非同期処理を理解するために必要な知識.canvas]]

## 管理用ノート

タスク。
- [x] `config.yml` ファイルでチャプター管理した方が良さそう
  - 内容を後で差し替えたり、追加したりする時に管理しづらい。
- [x] yaml データを各データに記載する

追加内容。
- [x] レンダリングパイプライン
- [x] タスク詳細
- [x] 擬似コード node など
- [x] 実行コンテキスト
- [x] Promise の状態の詳細
- [x] async/await
- [x] Top-level await
- [x] 静的メソッド
- [x] ループ
- [x] ジェネレータ
- [x] Promise の型注釈

## メモ

メッセージが多すぎるので注釈に入れ替える部分を検討する。
太文字が多すぎるので減らす。

## チャプター管理

`config.yml` に記載するチャプターデータ。

```yaml
  - 1-epasync-begin # 『はじめに』
  - part-01-epasync # 『第１部 - API を提供する環境と実行メカニズム』
  - f-epasync-asynchronous-apis # 『非同期 API と環境』
  - f-epasync-synchronus-apis # 『同期 API とブロッキング』
  - 2-epasync-event-loop # 『イベントループの概要と注意点』
  - d-epasync-task-microtask-queues # 『タスクキューとマイクロタスクキュー』
  - e-epasync-v8-engine # 『V8エンジン』
  - b-epasync-callstack-execution-context # 『コールスタックと実行コンテキスト』
  - c-epasync-what-event-loop # 『それぞれのイベントループ』
  - part-02-epasync # 『第２部 - Promise インスタンスと連鎖』
  - a-epasync-promise-basic-concept # 『Promise の基本概念』
  - 3-epasync-promise-constructor-executor-func # 『Promise コンストラクタ』と Executor 関数』
  - 4-epasync-callback-is-sync-or-async # 『コールバック関数の同期実行と非同期実行』
  - g-epasync-resolve-reject # 『resolve 関数と reject 関数の使い方』
  - 5-epasync-multiple-promises # 『複数の Promise を走らせる』
  - 6-epasync-then-always-return-new-promise # 『then メソッドは常に新しい 』Promise を返す』
  - 7-epasync-pass-value-to-the-next-chain # 『Promise chain で値を繋ぐ』
  - 8-epasync-return-promise-in-then-callback # 『then メソッドのコールバックで Promise インスタンスを返す』
  - 9-epasync-dont-nest-promise-chain # 『Promise chain はネストさせない』
  - 10-epasync-dont-use-side-effect # 『コールバックで副作用となる非同期処理』
  - 11-epasync-omit-return-by-arrow-shortcut # 『アロー関数で return を省略』する』
  - h-epasync-catch-finally # 『catch メソッドと finally メソッド』
  - 12-epasync-wrapping-macrotask # 『古い非同期APIをPromiseでラップする』
  - 13-epasync-loop-is-nested # 『イベントループは内部にネストしたループがある』
  - part-03-epasync # 『第３部 - async 関数と await 式の挙動』
  - 14-epasync-chain-to-async-await # 『Promise chain から async 関数へ』
  - 15-epasync-v8-converting # 『V8 エンジンによる async/await の内部変換』
  - 16-epasync-top-level-async # 『Top-level await』
  - part-04-epasync # 『第４部 - 制御と型注釈』
  - 17-epasync-static-method # 『Promise の静的メソッドと並列化』
  - 18-epasync-await-position # 『await 式の配置による制御』
  - 19-epasync-async-loop # 『反復処理の制御』
  - k-epasync-iterator-generator # 『イテレータとイテラブルとジェネレータ関数』
  - j-epasync-ts-promise-type-annotation # 『TypeScript における Promise の型注釈』
  - part-05-epasync # 『第５部 - 仕様およびその他の番外編』
  - m-epasync-promise-prototype-then # 『Promise.prototype.then の仕様挙動』
  - n-epasync-promise-spec-compare # 『Promise chain と async/await の仕様比較』
  - o-epasync-async-await-pattern-history # 『async/await Pattern の歴史的発展』
  - y-epasync-conclusion # 『総括 - 非同期処理のまとめ』
  - x-epasync-epilogue # 『あとがき』
  - z-epasync-reference # 『参考文献』
```
