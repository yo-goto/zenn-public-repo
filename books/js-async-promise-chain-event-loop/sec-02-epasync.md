---
date: 2022-06-16
title: "第２章 - Promise インスタンスと連鎖"
alieases: [EPAsync 第３章]
---

# この章について

第２章では Promise インスタンスと Promise チェーンについて見ていきます。非同期処理の処理予測を行うためのメンタルモデルを構築する上で基礎となる Promise チェーンについて具体的なコードを実行して非同期処理の実行順序を予測できるように訓練していきます。

第１章の内容が難しい場合にはこちらの章から始めるのも良いかもしれません。この本の執筆経緯的にもこの章の内容をはじめに書いています。この章の内容を学ぶことで、第３章の async/await を理解できるようになります。

# チャプター

- [Promise の基本概念](a-epasync-promise-basic-concept)
- [Promise コンストラクタと Executor 関数](3-epasync-promise-constructor-executor-func)
- [コールバック関数の同期実行と非同期実行](4-epasync-callback-is-sync-or-async)
- [resolve 関数と reject 関数の使い方](g-epasync-resolve-reject)
- [複数の Promise を走らせる](5-epasync-multiple-promises)
- [then メソッドは常に新しい Promise を返す](6-epasync-then-always-return-new-promise)
- [Promise チェーンで値を繋ぐ](7-epasync-pass-value-to-the-next-chain)
- [then メソッドのコールバックで Promise インスタンスを返す](8-epasync-return-promise-in-then-callback)
- [Promise チェーンはネストさせない](9-epasync-dont-next-promise-chain)
- [コールバックで副作用となる非同期処理](10-epasync-dont-use-side-effect)
- [アロー関数で return を省略する](11-epasync-omit-return-by-arrow-shortcut)
- [catch メソッドと finally メソッド](h-epasync-catch-finally)
- [古い非同期 API を Promise でラップする](12-epasync-wrapping-macrotask)
- [イベントループは内部にネストしたループがある](13-epasync-loop-is-nested)

