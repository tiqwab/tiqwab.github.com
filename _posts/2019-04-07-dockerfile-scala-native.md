---
layout: post
title: scala-native 用 Dockerfile を作成した
tags: "scala-native, docker, scala"
comments: true
---

[scala-native][1] というのは Scala のコードを (LLVM IR を経由して) ネイティブコードにコンパイルするツールです。最近少し遊んでいる中で過去バージョンの clang (LLGM ツールチェインの C や C++ コンパイラ。scala-native で使用) を使用したい場面があり、そのために Dockerfile を作成したのでメモとして残しておきます。

Dockerfile は下のリンク先に置いています。

- [Dockerfile][2]

```shell
$ docker build -t scala-native:latest .
$ docker container run -it scala-native:latest /bin/bash
...
```

のようにして使用できます。

イメージの概要としては

- ベースイメージとして ubuntu:18.04 を使用
- sbt (提供されているパッケージで最新) をインストール
- clang-7 をインストール
- nativeLink, test に必要なヘッダファイル等をインストール

したものになっています。

Dockefile は読みやすさを重視しているので無駄や不適切な箇所がありますが今回の用途には十分です。

一点気をつけるといいますか clang, clang++ へのパスの設定まわりでワークアラウンドをしています。
`apt-get install clang-7` すると `clang-7`, `clang++-7` として clang へパスが通るのですが scala-native 側はデフォルト `clang`, `clang++` を見ることになっています。なのでこのギャップを埋めるために Dockerfile 内でシンボリックリンクを設定するための RUN ステップを追加しています。

[1]: https://github.com/scala-native/scala-native
[2]: https://github.com/tiqwab/example/blob/master/sbt-with-scala-native/Dockerfile
