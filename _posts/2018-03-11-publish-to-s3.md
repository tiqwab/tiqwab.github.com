---
layout: post
title: sbt と s3 と cross build と release
tags: "sbt, s3, scala"
comments: true
---

会社や組織で内部的に管理したい Scala ライブラリがある場合、その実現方法の一つとして AWS S3 に用意したプライベートバケットを利用するというのがあると思います。

ただできるのは知っていてもそのようなライブラリ管理の経験に乏しかったので、この機会に

- S3 をライブラリの保存先として使用する
- sbt での cross build の設定
- [sbt-release][5] プラグインを使用したリリースの管理

あたりの手順を確認したのでメモとしてまとめておこうと思います。

サンプルに使用したプロジェクトは [s3-resolver-sample][6] に置いています。

1. [sbt-s3-resolver を使用する](#sbt-s3-resolver)
2. [cross build する](#cross-build)
3. [sbt-release を使用する](#sbt-release)

---

<div id="sbt-s3-resolver" />

### 1. sbt-s3-resolver を使用する

まず前提として S3 にライブラリ管理用のバケットが作成済であるとします。ここではスナップショット用に `snapshots.tiqwab.com`, リリース用に `releases.tiqwab.com` を用意しました。

次に [sbt-s3-resolver][1] プラグインを対象のプロジェクトに追加します。作業している環境に上記バケットへの権限が設定されていれば、以下のファイルをちょこちょこ編集するだけで OK です。

`project/plugins.sbt`:

```scala
// Add sbt-3-resolver plugin
resolvers += Resolver.jcenterRepo
addSbtPlugin("ohnosequences" % "sbt-s3-resolver" % "0.19.0")
```

`build.sbt`:

```scala
    ...
    // Publish in ivy style
    publishMavenStyle := false,
    publishTo := {
      val prefix = if (isSnapshot.value) "snapshots" else "releases"
      Some(s3resolver.value(s"My ${prefix} S3 bucket", s3(s"${prefix}.tiqwab.com")) withIvyPatterns)
    },
    ...
```

README に記載の通り、開発環境に適切な権限が設定されていればバケットは public, private ともに大丈夫です。デフォルトでは AWS SDK の [com.amazonaws.auth.DefaultAWSCredentialsProviderChain][7] が使用されるので、環境変数や `~/.aws/credentials` 等々でアクセスキーの情報を与えてあげるようにします。

`publishTo` の設定にでてくる `isSnapshot` ですが、これは `version` の末尾が `SNAPSHOT` か否かで判断しているとのことです (ref. [Publishing - sbt Reference Manual][3])。

これで設定は完了なので適当にコードを書いた後おもむろに publish してみます。

```
sbt:s3-resolver-sample> publish
...
sbt:sbt-s3-resolver-plugin-sample> releaseSample/publish
...
[info]     published s3-release-sample_2.12 to s3://snapshots.tiqwab.com/com.tiqwab/s3-release-sample_2.12/0.1.0-SNAPSHOT/jars/s3-release-sample_2.12.jar
[info]     published s3-release-sample_2.12 to s3://snapshots.tiqwab.com/com.tiqwab/s3-release-sample_2.12/0.1.0-SNAPSHOT/srcs/s3-release-sample_2.12-sources.jar
[info]     published s3-release-sample_2.12 to s3://snapshots.tiqwab.com/com.tiqwab/s3-release-sample_2.12/0.1.0-SNAPSHOT/docs/s3-release-sample_2.12-javadoc.jar
[info]     published ivy to s3://snapshots.tiqwab.com/com.tiqwab/s3-release-sample_2.12/0.1.0-SNAPSHOT/ivys/ivy.xml
[success] Total time: 12 s, completed Mar 9, 2018 11:14:25 PM
```

ライブラリ使用側はいつものように `libraryDependencies` にライブラリ情報を追加すればいいのですが、上で使用した S3 バケットを依存関係解決先として登録してあげる必要があります。

`build.sbt`:

```scala
    ...
    resolvers ++= Seq[Resolver](
      s3resolver.value("Release resolver", s3("releases.tiqwab.com")) withIvyPatterns,
      s3resolver.value("Snapshot resolver", s3("snapshots.tiqwab.com")) withIvyPatterns
    ),
    libraryDependencies += "com.tiqwab" %% "s3-release-sample" % "0.1.0-SNAPSHOT"
    ...
```

上の `withIvyPatterns` を忘れて 1 時間近く唸っていたことは秘密です。

なおここで記載した例は [Apache Ivy][2] 向けな設定になっているので、Maven だと少し記述が変わってくるようです。

<div id="cross-build" />

### 2. cross build する

Scala はメジャーバージョン間 (e.g. 2.11, 2,12) でバイナリ互換性を持たないので、ソースコードが同じでもビルドはそれぞれに対して行う必要があります。

sbt ではこのような [Cross-building][4] もサポートしています。

`build.sbt`:

```scala
    // Add Scala versions to be used for build
    crossScalaVersions := Seq("2.12.4", "2.11.12"),
```

Scala ではマイナーバージョン間 (e.g. 2.12.1, 2.12.2 のような 2.12.x) ではバイナリ互換性を持つようなので上で指定するバージョンは (そのメジャーバージョンで) 最新のものやプロジェクトで使用しているものを選べばいいと思います。

sbt のコマンドで頭に `+` を付けると上で指定した全バージョンを対象にできます。

```
sbt:sbt-s3-resolver-plugin-sample> + releaseSample/publish
```

<div id="sbt-release" />

### 3. sbt-release を使用する

ライブラリを作成しているとリリースのタイミングで git タグを打ってリポジトリに公開して開発用にバージョンを上げてといった形式的作業がありますが、こうしたところを代わりにやってくれるのが [sbt-relase][5] プラグインです。

これもどう使うかは README を見るのが一番ですが、標準の流れは

1. プロジェクトにプラグインを足す
2. 開発
3. リリースしたいタイミングで `sbt release`
4. リリースするバージョンと次の (開発中) バージョンを聞いてくるので必要ならば入力
5. 2 へ

という感じです。

```
sbt:sbt-s3-resolver-plugin-sample> project releaseSample
sbt:s3-release-sample> release
Release version [0.2.1] : 
Next version [0.2.2-SNAPSHOT] : 
... (tag, publish, commit, push .etc)
```

上のようにサブプロジェクト単位でもいい感じにやってくれます。

cross build したい場合は `build.sbt` に `releseCrossBuild` の設定が必要です。

`build.sbt`:

```scala
    releaseCrossBuild := true
```

[1]: https://github.com/ohnosequences/sbt-s3-resolver
[2]: http://ant.apache.org/ivy/
[3]: https://www.scala-sbt.org/1.x/docs/Publishing.html
[4]: https://www.scala-sbt.org/1.0/docs/Cross-Build.html
[5]: https://github.com/sbt/sbt-release
[6]: https://github.com/tiqwab/s3-resolver-sample
[7]: https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/DefaultAWSCredentialsProviderChain.html
