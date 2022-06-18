---
date: 2022-06-16
title: "第１章 - API を提供する環境と実行メカニズム"
alieases: [EPAsync 第１章]
---

# この章について

第１章では API を提供する環境と JavaScript の実行メカニズムについて見ていきます。この章の内容は「非同期処理」についてのメタ的な視点での解説となります。

先取りで[第２章](sec-02-epasync)の知識(Promise チェーンなど)を利用している場面があるので、この章が難しい場合には具体的なコードから解説している第２章の内容から読み進めるのも良いかもしれません。ただし、この章では非同期処理の実行順序を予測するために理解すべきイベントループの解説があり、重要度としては非常に高いので注意してください。

# チャプター

- [非同期 API と環境](f-epasync-asyncronous-apis)
- [同期 API とブロッキング](f-epasync-synchronus-apis)
- [イベントループの概要と注意点](2-epasync-event-loop)
- [タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)
- [V8 エンジンについて](e-epasync-v8-engine)
- [コールスタックと実行コンテキスト](b-epasync-callstack-execution-context)
- [それぞれのイベントループ](c-epasync-what-event-loop)

