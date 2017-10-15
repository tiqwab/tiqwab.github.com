---
layout: post
title: Scala で書いたプログラムをさくっとデーモンで動かす
tags: "daemon, scala"
comments: true
---

以前ちょっとしたバッチプログラムを Scala で書いて EC2 のインスタンス上で長時間回しておきたいということがあったのですが、デーモンプロセスとして動かすのってどうするんだっけと手間取ってしまったのでそのメモを残しておきたいと思います。

あまり Scala に関係する内容では無いです。

### デーモンプロセスとは

デーモンって何なのさというのを一度整理するために以下のページを参考にしました。

- [デーモン][4]
- [Linux でプロセスのデーモン化][3]
- [デーモンの作り方][2]

Wikipedia の引用になりますが、デーモンとは

> 技術的に厳密に言えば、Unix系システムでは親プロセスが終了していて initプロセス（プロセス番号1）を親プロセスとしていて制御端末を持たないプロセス

と考えて良さそうです。

単純に SSH で入ってコマンド実行して exit... では端末切断時に子プロセスが全て終了されてしまうのでプログラムを回し続けておくことができません。

そこでプログラムをデーモンとして動かせば制御端末から切り離されたプロセスになるので、回し続けられるということになります。

### プログラム実行用の jar を作成する

実際にデーモン化して... の前に、今回対象としているのは Scala で書かれたプログラムなので、何らかの実行できる形にビルドしてあげる必要があります。

方法は色々あると思いますが、ここでは [sbt-assembly][1] で実行用の jar を作成します。

```
$ sbt assembly
```

### setsid でデーモンとして実行する

コマンドをデーモンとして実行するためにここでは `setsid` を使用します。

```
$ setsid java -jar target/scala-2.12/scala-daemonize-sample-assembly-0.1.0-SNAPSHOT.jar > /dev/null 2>&1 < /dev/null
```

`pstree` で確認すると確かにデーモンとしてプログラムが実行できていることがわかります。

```
# To check process id
$ ps ax | grep java
9166 ?        Ssl    0:00 java -jar target/scala-2.12/scala-daemonize-sample-assembly-0.1.0-SNAPSHOT.jar

$ pstree -g
systemd(1)-+- ...
           ...
           |-java(9166)-+-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            |-{java}(9166)
           |            `-{java}(9166)
           ...
```

`setsid` が実際に行うことは

- 指定したコマンドを実行するプロセスをリーダーとして制御端末を持たないセッションを開始
- 指定したコマンドを実行するプロセスをリーダーとしてプロセスグループも作成

のようで、この結果 init プロセスを直接の親として持つ形でプロセスが実行できるようです。

なおコマンド後半の `> /dev/null 2>&1 < /dev/null` は標準入出力、エラー出力を全て `/dev/null` に捨てるための記述です。

参考

- [Run bash script as daemon][5]

[1]: https://github.com/sbt/sbt-assembly
[2]: http://draft.scyphus.co.jp/series/daemon.ja/01_daemonize.html
[3]: https://qiita.com/0xfffffff7/items/08d9c268c728da46d20b
[4]: https://ja.wikipedia.org/wiki/%E3%83%87%E3%83%BC%E3%83%A2%E3%83%B3_(%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2)
[5]: https://stackoverflow.com/questions/19233529/run-bash-script-as-daemon
