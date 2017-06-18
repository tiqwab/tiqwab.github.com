---
layout: post
title: 自分用 sbt テンプレートの作成
tags: "scalafmt, scala"
comments: true
---

Scala のビルドツール [sbt][7] では事前に用意したテンプレートからプロジェクトを作成することができます。

```
$ sbt new scala/scala-seed.g8
```

公式に用意されている Scala 用のテンプレートの一つに `scala/scala-seed.g8` というものがあります。
これは Scala プロジェクトとして最低限の設定を持つテンプレートですが、実際使用するときにはもう少しコンパイルやフォーマットの設定を行ったものがあると嬉しいと感じたため、自分用にテンプレートを作成することにしました。

作成したテンプレートは [こちら][8] に置いています。

1. [Project の name, organization 設定](#settings-project)
2. [ScalacOptions の設定](#settings-scalac)
3. [Scalafmt の設定](#settings-scalafmt)

---

<div id="settings-project" />

### 1. Project の name, organization 設定

`build.sbt` には作成する project の name と organization を定義する場所があるのですが、標準のテンプレートではそこがデフォルトのままになるので、生成時に与えた値を使用するように変更します。

[Giter8 テンプレートの作り方][4] を参考に以下の 2 箇所を変更しました。

- (1) `src/main/g8/default.properties` に `organization` を加える

```
name=My Something Project
organization=com.example // (1)
description=Minimum Scala build.
```

- (2) `src/main/g8/build.sbt` の name, organization に project 生成時に与えた値を設定する

```
lazy val root = (project in file(".")).
  settings(
    inThisBuild(List(
      organization := "$organization$", // (2)
      scalaVersion := "2.12.2",
      version      := "0.1.0-SNAPSHOT"
    )),
    name := "$name$", // (2)
    scalacOptions := commonScalacOptions,
    libraryDependencies += scalaTest % Test
  )
```

<div id="settings-scalac" />

### 2. ScalacOptions の設定

- (1) `build.sbt` に `scalacOptions` を設定する
  - 各 option の詳細は `scalac -help`, `scalac -X`, `scalac -Y` で確認できる

```
// (1)
// scalacOptions
// See `scalac -help`, `scalac -X`, or `scalac -Y`
lazy val commonScalacOptions = Seq(
  "-feature" // Emit warning and location for usages of features that should be imported explicitly.
  , "-deprecation" // Emit warning and location for usages of deprecated APIs.
  , "-unchecked" // Enable additional warnings where generated code depends on assumptions.
  , "-encoding" // Specify encoding of source files
  , "UTF-8"
  // , "-Xfatal-warnings"
  , "-language:_"
  , "-Ywarn-adapted-args" // Warn if an argument list is modified to match the receiver
  , "-Ywarn-dead-code" // Warn when dead code is identified.
  , "-Ywarn-inaccessible" // Warn about inaccessible types in method signatures.
  , "-Ywarn-infer-any" // Warn when a type argument is inferred to be `Any`.
  , "-Ywarn-nullary-override" // Warn when non-nullary `def f()' overrides nullary `def f'
  , "-Ywarn-nullary-unit" // Warn when nullary methods return Unit.
  , "-Ywarn-numeric-widen" // Warn when numerics are widened.
  , "-Ywarn-unused" // Warn when local and private vals, vars, defs, and types are unused.
  , "-Ywarn-unused-import" // Warn when imports are unused.
)

lazy val root = (project in file(".")).
  settings(
    inThisBuild(List(
      organization := "$organization$",
      scalaVersion := "2.12.2",
      version      := "0.1.0-SNAPSHOT"
    )),
    name := "$name$",
    scalacOptions := commonScalacOptions,
    libraryDependencies += scalaTest % Test
  )
```

<div id="settings-scalafmt" />

### 3. Scalafmt の設定

[Scalafmt][1] により Scala コードがフォーマットできるようにしておきます。

これにより `sbt scalafmt --test` でフォーマットのチェックが、`sbt scalafmt` で実際にフォーマットをかけられます。

IntelliJ からは別途 [plugin][2] をインストールすることで保存時にフォーマットがかかるようにできます。

- (1) `project/plugins.sbt` に以下のライブラリ依存を追加
  - 詳細は [sbt-scalafmt][5]

```
libraryDependencies += "com.geirsson" %% "scalafmt-bootstrap" % "0.6.6"
```

- (2) `build.sbt` に以下の設定を行う
  - 詳細は [sbt-scalafmt][5]

```
def latestScalafmt = "1.0.0-RC4"
commands += Command.args("scalafmt", "Run scalafmt cli.") {
  case (state, args) =>
    val Right(scalafmt) =
      org.scalafmt.bootstrap.ScalafmtBootstrap.fromVersion(latestScalafmt)
    scalafmt.main("--non-interactive" +: args.toArray)
    state
}
```

- (3) `.scalafmt.conf` にフォーマッティングの設定をする
  - 詳細は [Configuration][6]

```
# For pretty alignment
align = some
# Only format files tracked by git.
project.git = true
# Manually exclude files to format.
project.excludeFilters = ["target/"]
```

[1]: http://scalameta.org/scalafmt/
[2]: http://scalameta.org/scalafmt/#IntelliJ
[3]: https://stackoverflow.com/questions/18601372/how-can-i-find-a-description-of-scala-compiler-flags-options
[4]: http://www.foundweekends.org/giter8/ja/template.html
[5]: http://scalameta.org/scalafmt/#sbt-scalafmt
[6]: http://scalameta.org/scalafmt/#Configuration
[7]: http://www.scala-sbt.org/index.html
[8]: https://github.com/tiqwab/scala-seed.g8/tree/2.12.x
