---
layout: post
title: Akka でライフゲーム
tags: "akka, game of life, scala"
comments: true
---

[Akka][1] の勉強として[ライフゲーム][2] を作りました。

[akka-life-game][3]

- `Screen` が画面描写
- `Board` が `Cell` の管理者
- `Cell` が各マス上の細胞

といった感じの機能分けをし、それぞれを Actor として実装しています。

Actor はこう書くのが良さそうといったことはソース中のコメントとして書いています。
それらの情報は以下の資料を参考にしました。

- [Akka Documentation][4]
- [並行処理初心者のためのAkka入門][5]
- [Effective actors japanesesub][6]

[1]: http://akka.io/
[2]: https://ja.wikipedia.org/wiki/%E3%83%A9%E3%82%A4%E3%83%95%E3%82%B2%E3%83%BC%E3%83%A0
[3]: https://github.com/tiqwab/example/tree/master/akka-life-game
[4]: http://doc.akka.io/docs/akka/current/scala/index.html
[5]: https://www.slideshare.net/sifue/akka-39611889
[6]: https://www.slideshare.net/shinolajla/effective-actors-japanesesub
