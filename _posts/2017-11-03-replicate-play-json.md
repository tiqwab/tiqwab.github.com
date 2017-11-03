---
layout: post
title: play-json 模写
tags: "replication, play-json, scala"
comments: true
---

絵と同じでプログラミングもライブラリの模写をすることで勉強になるのでは、という気持ちで試しに play-json もどきを作ってみました。

ライブラリの模写として具体的には 「型とドキュメントから対象のライブラリとインターフェース上同様の振る舞いを示すもの」 を作成することを目指しています。

play-json の最も基本的であろう機能 5 つを取り上げて順番に定義し、最後に実際のライブラリの実装を見ながら答え合わせをしようと思います。

1. [JsValue の定義](#anchor1)
2. [JSON 構造をたどる](#anchor2)
3. [Writes の定義](#anchor3)
4. [Reads の定義](#anchor4)
5. [Reads, Writes コンビネータ](#anchor5)
6. [答え合わせ](#anchor6)
7. [感想](#anchor7)

---

<div id="anchor1" />

### 1. JsValue の定義

- play-json では `JsValue` というデータ構造により任意の JSON を表現する
- `JsValue` を作成する一つの方法は `Json` オブジェクトの提供するメソッドを利用すること
  - `Json.obj` for `JsObject`
  - `Json.arr` for `JsArray`
- 多くの場合 Scala の標準型からの implicit conversion が定義されている
    - e.g. `Int` to `JsNumber`, `String` to `JsString`
- `Json.strinfigy` で JSON 文字列に変換できる
  - 逆の関数として `Json.parse` があるが、今回は省略
- ドキュメント記述は [ここ][2]

これらの情報をもとに以下のように `JsValue`, `Json` を定義します。

```scala
package com.tiqwab.replication.play.json

sealed trait JsValue

case class JsString(value: String) extends JsValue {
  override def toString: String = "\"" + value.toString + "\""
}
case class JsNumber(value: Number) extends JsValue {
  override def toString: String = value.toString
}
case class JsBoolean(value: Boolean) extends JsValue {
  override def toString: String = value.toString
}
case class JsObject(value: (String, JsValue)*) extends JsValue {
  override def toString: String =
    "{" + value.map { case (k, v) => "\"" + k.toString + "\"" + ":" + v.toString }.mkString(", ") + "}"
}
case class JsArray(value: JsValue*) extends JsValue {
  override def toString: String = "[" + value.map(_.toString).mkString(", ") + "]"
}
case object JsNull extends JsValue {
  override def toString: String = "null"
}

object JsValue {

  implicit def stringToJsString(value: String): JsString = JsString(value)
  implicit def intToJsNumber(value: Int): JsNumber = JsNumber(value)
  implicit def doubleToJsNumber(value: Double): JsNumber = JsNumber(value)
  implicit def booleanToJsBoolean(value: Boolean): JsBoolean = JsBoolean(value)
  implicit def arrayToJsArray(value: Seq[JsValue]): JsArray = JsArray(value: _*)

}
```

```scala
package com.tiqwab.replication.play.json

object Json {

  def stringify(json: JsValue): String = json.toString
  def obj(value: (String, JsValue)*): JsObject = JsObject(value: _*)
  def arr(value: JsValue*): JsArray = JsArray(value: _*)

}
```

実際に repl で動作を確認してみます。

```
scala> val json: JsValue = Json.obj(
     |   "name" -> "Watership Down",
     |   "location" -> Json.obj("lat" -> 51.235685, "long" -> -1.309197),
     |   "residents" -> Json.arr(
     |     Json.obj(
     |       "name" -> "Fiver",
     |       "age" -> 4,
     |       "role" -> JsNull
     |     ),
     |     Json.obj(
     |       "name" -> "Bigwig",
     |       "age" -> 6,
     |       "role" -> "Owsla"
     |     )
     |   )
     | )

json: com.tiqwab.replication.play.json.JsValue = {"name":"Watership Down", "location":{"lat":51.235685, "long":-1.309197}, "residents":[{"name":"Fiver", "age":4, "role":null}, {"name":"Bigwig", "age":6, "role":"Owsla"}]}
```

どうやらそれっぽく動いているようです。

<div id="anchor2" />

### 2. JSON 構造をたどる

- `JsValue` の `\` オペレータにより JSON 構造をたどって特定の位置の値を取得できる
- `\` の結果は失敗もありうるので `JsLookupResult` というデータ型で表される
  - 成功時が `JsDefined`, 失敗は `JsUndefined` となる
- `\` で 配列の特定の位置を取得したり `\\` というオペレータもあるがここでは省略
- ドキュメント記述は [ここ][4]

上の内容を踏まえると `JsValue` に 以下のような `\` オペレータを用意し `JsLookupResult` を返させるようにすれば良さそうです。

```scala
sealed trait JsValue { self =>

  def \(name: String): JsLookupResult = self match {
    case json: JsObject =>
      json.value.find(_._1 == name) match {
        case Some((_, v)) =>
          JsDefined(v)
        case None =>
          JsUndefined(s"no such element $name")
      }
    case _ =>
      JsUndefined(s"no such element: $name")
  }

}
```

```scala
package com.tiqwab.replication.play.json

sealed trait JsLookupResult { self =>

  def get: JsValue
  def toOption: Option[JsValue]
  def toEither: Either[String, JsValue]

  def \(name: String): JsLookupResult = self match {
    case JsDefined(json) =>
      json \ name
    case x @ JsUndefined(_) =>
      x
  }

}

case class JsDefined(value: JsValue) extends JsLookupResult {
  override def get: JsValue = value
  override def toOption: Option[JsValue] = Some(value)
  override def toEither: Either[String, JsValue] = Right(value)
}
case class JsUndefined(message: String) extends JsLookupResult {
  override def get: JsValue = throw new NoSuchElementException(message)
  override def toOption: Option[JsValue] = None
  override def toEither: Either[String, JsValue] = Left(message)
}
```

動作を確認します。

```
scala> val json: JsValue = ... (same as the above)
json: com.tiqwab.replication.play.json.JsValue = {"name":"Watership Down", "location":{"lat":51.235685, "long":-1.309197}, "residents":[{"name":"Fiver", "age":4, "role":null}, {"name":"Bigwig", "age":6, "role":"Owsla"}]}

scala> (json \ "name").toOption
res1: Option[com.tiqwab.replication.play.json.JsValue] = Some("Watership Down")

scala> (json \ "location" \ "lat").toOption
res2: Option[com.tiqwab.replication.play.json.JsValue] = Some(51.235685)

scala> (json \ "unknown").toOption
res3: Option[com.tiqwab.replication.play.json.JsValue] = None
```

良さそうです。このまま進みます。

<div id="anchor3" />

### 3. Writes の定義

- play-json ではあるクラスのオブジェクトが JSON に変換できるということを示すために `Writes` を定義する
- `Json.toJson` で実際の変換が行える
- `Writes` を定義する方法にはコンビネータを使用するものもあるが後述
- ドキュメント記述は [ここ][3]

上の内容を踏まえ `Writes` は以下のように定義してみました。

```scala
package com.tiqwab.replication.play.json

trait Writes[T] {

  def writes(value: T): JsValue

}

object Writes {

  def apply[T](f: T => JsValue) = new Writes[T] {
    override def writes(value: T): JsValue = f(value)
  }

  implicit def stringWrites: Writes[String] = Writes { x =>
    JsString(x)
  }
  implicit def intWrites: Writes[Int] = Writes { x =>
    JsNumber(x)
  }
  implicit def doubleWrites: Writes[Double] = Writes { x =>
    JsNumber(x)
  }
  implicit def booleanWrites: Writes[Boolean] = Writes { x =>
    JsBoolean(x)
  }
  implicit def seqWrites[T: Writes]: Writes[Seq[T]] = Writes { xs =>
    JsArray(xs.map(implicitly[Writes[T]].writes(_)): _*)
  }

}
```

`Json` には `Writes` を扱うユーティリティメソッド `toJson` を定義します。

```scala
object Json {
  ...

  def toJson[T: Writes](obj: T): JsValue =
    implicitly[Writes[T]].writes(obj)
}
```

また先程 `JsValue` に各型に対応する implicit conversion を定義したのですが、「Writes が定義されているのならば JSON にできる」 という定義に従い、`Writes` に対する implicit conversion として定義し直します。

```scala
object JsValue {

  implicit def writesToJsValue[T: Writes](v: T): JsValue =
    implicitly[Writes[T]].writes(v)

  /*
  implicit def stringToJsString(value: String): JsString = JsString(value)
  implicit def intToJsNumber(value: Int): JsNumber = JsNumber(value)
  implicit def doubleToJsNumber(value: Double): JsNumber = JsNumber(value)
  implicit def booleanToJsBoolean(value: Boolean): JsBoolean = JsBoolean(value)
  implicit def arrayToJsArray(value: Seq[JsValue]): JsArray = JsArray(value: _*)
  */
}
```

実際に `Writes` を定義したクラスのオブジェクトに対し `Json.toJson` を使用してみます。

```
scala> case class Address(city: String, country: String)

scala> case class Person(name: String, age: Int, address: Address)

scala> implicit val addressWrites: Writes[Address] = Writes { x =>
     |   Json.obj("city" -> x.city, "country" -> x.country)
     | }
addressWrites: com.tiqwab.replication.play.json.Writes[Address] = com.tiqwab.replication.play.json.Writes$$anon$1@6047ebe0

scala> implicit val personWrites: Writes[Person] = Writes { x =>
     |   Json.obj("name" -> x.name, "age" -> x.age, "address" -> x.address)
     | }
personWrites: com.tiqwab.replication.play.json.Writes[Person] = com.tiqwab.replication.play.json.Writes$$anon$1@11be8298

scala> val person = Person("Alice", 20, Address("Tokyo", "Japan"))
person: Person = Person(Alice,20,Address(Tokyo,Japan))

scala> Json.toJson(person)
res0: com.tiqwab.replication.play.json.JsValue = {"name":"Alice", "age":20, "address":{"city":"Tokyo", "country":"Japan"}}
```

<div id="anchor4" />

### 4. Reads の定義

- play-json ではあるクラスのオブジェクトが JSON からデシリアライズできるということを示すために `Reads` を定義する
- `Json.fromJson` で実際の変換が行える
- JSON からのデシリアライズ結果は `JsResult` として返される
  - `JsSuccess` が成功、`JsError` が失敗
- `Writes` と同様コンビネータによる定義は後述

まずは `JsResult` を定義してみます。
`JsResult.sequence` の定義がちょっとごちゃっとしていますが、それ以外は単純に成功ならその値、失敗ならその内容という風になっています。

```scala
package com.tiqwab.replication.play.json

sealed trait JsResult[T] { self =>
  def get: T
  def map[U](f: T => U): JsResult[U] = self match {
    case JsSuccess(v) =>
      JsSuccess(f(v))
    case JsError(msgs) =>
      JsError(msgs)
  }
  def flatMap[U](f: T => JsResult[U]): JsResult[U] = self match {
    case JsSuccess(v) =>
      f(v)
    case JsError(msgs) =>
      JsError(msgs)
  }
}

case class JsSuccess[T](value: T) extends JsResult[T] {
  override def get: T = value
}
case class JsError[T](messages: Seq[String]) extends JsResult[T] {
  override def get: T = throw new NoSuchElementException(s"no such element: $messages")
}

object JsResult {

  def sequence[T](results: Seq[JsResult[T]]): JsResult[Seq[T]] = {
    def loop(res: Seq[JsResult[T]], result: JsResult[Seq[T]]): JsResult[Seq[T]] =
      res.headOption match {
        case None =>
          result
        case Some(x) =>
          val nextResult = x match {
            case JsSuccess(v) =>
              result match {
                case JsSuccess(vs) =>
                  JsSuccess(v +: vs)
                case JsError(msgs) =>
                  JsError[Seq[T]](msgs)
              }
            case JsError(msg) =>
              result match {
                case _: JsSuccess[Seq[T]] =>
                  JsError[Seq[T]](msg)
                case JsError(msgs) =>
                  JsError[Seq[T]](msg ++ msgs)
              }
          }
          loop(res.tail, nextResult)
      }
    loop(results, JsSuccess(Seq.empty[T]))
  }

}
```

上の `JsResult` を使用して `Reads` を定義します。

```scala
trait Reads[T] { self =>
  def reads(json: JsValue): JsResult[T]

  def map[U](f: T => U): Reads[U] = Reads { json =>
    self.reads(json).map(f)
  }

  def flatMap[U](f: T => Reads[U]): Reads[U] = Reads { json =>
    self.reads(json) match {
      case JsSuccess(v) =>
        f(v).reads(json)
      case JsError(msgs) =>
        JsError(msgs)
    }
  }
}

object Reads {
  def apply[T](f: JsValue => JsResult[T]): Reads[T] = new Reads[T] {
    override def reads(json: JsValue): JsResult[T] = f(json)
  }

  implicit def stringReads: Reads[String] = Reads {
    case JsString(v) =>
      JsSuccess(v)
    case x =>
      JsError(Seq(s"$x cannot be read to String"))
  }
  implicit def intReads: Reads[Int] = Reads {
    case JsNumber(v) =>
      JsSuccess(v.intValue())
    case x =>
      JsError(Seq(s"$x cannot be read to Int"))
  }
  implicit def doubleReads: Reads[Double] = Reads {
    case JsNumber(v) =>
      JsSuccess(v.doubleValue())
    case x =>
      JsError(Seq(s"$x cannot be read to Double"))
  }
  implicit def booleanReads: Reads[Boolean] = Reads {
    case JsBoolean(v) =>
      JsSuccess(v)
    case x =>
      JsError(Seq(s"$x cannot be read to Boolean"))
  }
  implicit def seqReads[T: Reads]: Reads[Seq[T]] = Reads {
    case arr: JsArray =>
      JsResult.sequence(arr.value.map(implicitly[Reads[T]].reads(_)))
    case x =>
      JsError[Seq[T]](Seq(s"$x cannot be read to Seq"))
  }
}
```

また `JsValue` に `as`, `asOpt` を定義して `Reads` を使いやすくします。

```scala
sealed trait JsValue { self =>
  ...

  def as[T](implicit reads: Reads[T]): T =
    reads.reads(self) match {
      case JsSuccess(v) =>
        v
      case JsError(msg) =>
        throw new IllegalArgumentException(msg.toString)
    }

  def asOpt[T](implicit reads: Reads[T]): Option[T] =
    reads.reads(self) match {
      case JsSuccess(v) =>
        Some(v)
      case JsError(_) =>
        None
    }
}
```

実際に `Reads` を使用してみます。

```
scala> case class Address(city: String, country: String)

scala> case class Person(name: String, age: Int, address: Address)

scala> implicit val addressReads: Reads[Address] = Reads { json =>
     |   (for {
     |     city <- (json \ "city").get.asOpt[String]
     |     country <- (json \ "country").get.asOpt[String]
     |   } yield Address(city, country)).fold[JsResult[Address]](JsError(Seq("cannot read as Address")))(JsSuccess(_))
     | }
addressReads: com.tiqwab.replication.play.json.Reads[Address] = com.tiqwab.replication.play.json.Reads$$anon$1@2518ff77

scala> implicit val personReads: Reads[Person] = Reads { json =>
     |   (for {
     |     name <- (json \ "name").get.asOpt[String]
     |     age <- (json \ "age").get.asOpt[Int]
     |     address <- (json \ "address").get.asOpt[Address]
     |   } yield Person(name, age, address)).fold[JsResult[Person]](JsError(Seq("cannot read as Person")))(JsSuccess(_))
     | }
personReads: com.tiqwab.replication.play.json.Reads[Person] = com.tiqwab.replication.play.json.Reads$$anon$1@72d390a1

scala> val json = Json.obj(
     |   "name" -> "Alice",
     |   "age" -> 20,
     |   "address" -> Json.obj(
     |     "city" -> "Tokyo",
     |     "country" -> "Japan"
     |   )
     | )
json: com.tiqwab.replication.play.json.JsObject = {"name":"Alice", "age":20, "address":{"city":"Tokyo", "country":"Japan"}}

scala> Json.fromJson[Person](json)
res0: Person = Person(Alice,20,Address(Tokyo,Japan))
```

現時点では冗長な記述に感じるところもありますが、ひとまず JSON からオブジェクトへのデシリアライズが行えるようになりました。

<div id="anchor5" />

### 5. Reads, Writes コンビネータ

- `Reads`, `Writes` を定義するために play-json ではコンビネータを提供している
  - コンビネータを使用した方が複雑な定義でも書きやすい (気がする)
- ドキュメント記述は [ここ][5]

まずは `JsPath` という JSON の任意の位置を表現するためのクラスを定義します。
`JsPath` は `JsValue` と同様に `\` で JSON の構造をたどることができ、また `write`, `read` でそれぞれ対応する `Writes`, `Reads` が定義できるようになっています。

```scala
package com.tiqwab.replication.play.json

// FIXME: どういうデータ構造がいいんだろう
// parnet, child ともに持たせるような双方向の依存にするべき？
case class JsPath(name: String, parent: Option[JsPath]) { self =>
  def \(key: String): JsPath = JsPath(key, Some(self))

  def write[T: Writes]: Writes[T] = Writes { x =>
    def loop(jsPathOpt: Option[JsPath], elem: JsValue): JsValue = jsPathOpt match {
      case None =>
        elem
      case Some(jsPath) =>
        loop(jsPath.parent, Json.obj(jsPath.name -> elem))
    }
    loop(Some(self), implicitly[Writes[T]].writes(x))
  }

  def read[T: Reads]: Reads[T] = Reads { json =>
    def listPath(jsPathOpt: Option[JsPath], paths: Seq[JsPath]): Seq[JsPath] = jsPathOpt match {
      case None =>
        paths
      case Some(jsPath) =>
        listPath(jsPath.parent, jsPath +: paths)
    }
    def loop(paths: Seq[JsPath], elem: JsValue): JsResult[T] = paths.headOption match {
      case None =>
        implicitly[Reads[T]].reads(elem)
      case Some(jsPath) =>
        (elem \ jsPath.name).toEither match {
          case Left(msg) =>
            JsError(Seq(msg))
          case Right(j) =>
            loop(paths.tail, j)
        }
      // TODO: fold だと末尾再帰にならない??
      // (elem \ jsPath.name).toEither.fold(msg => JsError(Seq(msg)), j => loop(paths.tail, j))
    }
    loop(listPath(Some(self), Seq.empty[JsPath]), json)
  }
}

object JsPath {
  def \(key: String): JsPath = JsPath(key, None)
}
```

`JsPath` はこんな感じに動作します。

```
scala> val write: Writes[String] = (JsPath \ "address" \ "city").write[String]
write: com.tiqwab.replication.play.json.Writes[String] = com.tiqwab.replication.play.json.Writes$$anon$1@6647ed70

scala> write.writes("Tokyo")
res0: com.tiqwab.replication.play.json.JsValue = {"address":{"city":"Tokyo"}}

scala> val read: Reads[String] = (JsPath \ "address" \ "city").read[String]
read: com.tiqwab.replication.play.json.Reads[String] = com.tiqwab.replication.play.json.Reads$$anon$1@68cd7341

scala> read.reads(res0)
res1: com.tiqwab.replication.play.json.JsResult[String] = JsSuccess(Tokyo)
```

次にコンビネータの定義はどう書けるかな、というのが難しいのですがドキュメントを見ると play-json ではどうやら 2 つの `Reads` を `and` オペレータでつないで `FunctionalBuilder[Reads]#CanBuild2[T, U]` のようなものを作成しているようなので、それをヒントにしてみます。

```scala
trait Reads[T] { self =>
  ...

  def and[U, C](another: Reads[U]): ReadsBuilder.CanBuild2[T, U] =
    ReadsBuilder.CanBuild2(self, another)
}
```

ここでは一歩抽象化を断念して `FunctionalBuilder[Reads]` のような形ではなく以下で定義する `ReadsBuilder` を使用しました。

`ReadsBuilder.CanBuild2` には渡された 2 つの `Reads` から新しく `Reads` を生成する `apply` を定義します。

また `Reads` は 2 つと言わず 3, 4, ... と本来は組み合わせることができるはずなので、ここでは試しに `CanBuild3` ケースクラスで 3 つの場合に対応してみました。
上手くやる方法がわからなかったので、愚直に引数の数に対して個別に `CanBuildx` を定義しています。

```scala
package com.tiqwab.replication.play.json

// FIXME: うまくやる方法がわからないので、愚直に引数の数に対して個別に CanBuild を定義
object ReadsBuilder {
  case class CanBuild2[A, B](r1: Reads[A], r2: Reads[B]) {
    def apply[T](f: (A, B) => T) = Reads { json =>
      for {
        a <- r1.reads(json)
        b <- r2.reads(json)
      } yield f(a, b)
    }
    def and[C](r3: Reads[C]): CanBuild3[A, B, C] = CanBuild3(r1, r2, r3)
  }

  case class CanBuild3[A, B, C](r1: Reads[A], r2: Reads[B], r3: Reads[C]) {
    def apply[T](f: (A, B, C) => T) = Reads { json =>
      for {
        a <- r1.reads(json)
        b <- r2.reads(json)
        c <- r3.reads(json)
      } yield f(a, b, c)
    }
  }
}
```

`Writes` も同様に定義できます。

```scala
trait Writes[T] { self =>
  ...

  def and[U](another: Writes[U]): WritesBuilder.CanBuild2[T, U] =
    WritesBuilder.CanBuild2(self, another)

}
```

```scala
package com.tiqwab.replication.play.json

object WritesBuilder {
  case class CanBuild2[A, B](w1: Writes[A], w2: Writes[B]) {
    def apply[T](f: T => (A, B)): Writes[T] = Writes { t =>
      val (a, b) = f(t)
      // FIXME: JsObject であることを保証できない　
      // 恐らくそれゆえに OWrites が存在する
      w1.writes(a).asInstanceOf[JsObject] ++ w2.writes(b).asInstanceOf[JsObject]
    }
    def and[C](w3: Writes[C]): CanBuild3[A, B, C] = CanBuild3(w1, w2, w3)
  }

  case class CanBuild3[A, B, C](w1: Writes[A], w2: Writes[B], w3: Writes[C]) {
    def apply[T](f: T => (A, B, C)): Writes[T] = Writes { t =>
      val (a, b, c) = f(t)
      // FIXME: JsObject であることを保証できない　
      // 恐らくそれゆえに OWrites が存在する
      w1.writes(a).asInstanceOf[JsObject] ++ w2.writes(b).asInstanceOf[JsObject] ++ w3.writes(c).asInstanceOf[JsObject]
    }
  }
}
```

またケースクラスに対してコンビネータにより `Writes` を定義する場合の利便性のために以下の以下の `unlift` も定義しておきます。

```scala
package com.tiqwab.replication.play

package object json {
  // FIXME 実際の定義はこんな適当なのか？
  def unlift[T, U](f: T => Option[U]): T => U =
    t => f(t).get
}
```

実際に使用してみます。

Reads combinator

```
scala> implicit val addressReads: Reads[Address] =
     |   ((JsPath \ "city").read[String] and
     |     (JsPath \ "country").read[String])(Address.apply)
addressReads: com.tiqwab.replication.play.json.Reads[Address] = com.tiqwab.replication.play.json.Reads$$anon$1@6145ef81

scala> implicit val personReads: Reads[Person] =
     |   ((JsPath \ "name").read[String] and
     |     (JsPath \ "age").read[Int] and
     |     (JsPath \ "address").read[Address])(Person.apply)
personReads: com.tiqwab.replication.play.json.Reads[Person] = com.tiqwab.replication.play.json.Reads$$anon$1@81e7df9

scala> val json = Json.obj(
     |   "name" -> "Alice",
     |   "age" -> 20,
     |   "address" -> Json.obj(
     |     "city" -> "Tokyo",
     |     "country" -> "Japan"
     |   )
     | )
json: com.tiqwab.replication.play.json.JsObject = {"name":"Alice", "age":20, "address":{"city":"Tokyo", "country":"Japan"}}

scala> Json.fromJson[Person](json)
res2: Person = Person(Alice,20,Address(Tokyo,Japan))

```

Writes combinator

```
scala> implicit val addressWrites: Writes[Address] =
     |   ((JsPath \ "city").write[String] and
     |     (JsPath \ "country").write[String])(unlift(Address.unapply))
addressWrites: com.tiqwab.replication.play.json.Writes[Address] = com.tiqwab.replication.play.json.Writes$$anon$1@63863d31

scala> implicit val personWrites: Writes[Person] =
     |   ((JsPath \ "name").write[String] and
     |     (JsPath \ "age").write[Int] and
     |     (JsPath \"address").write[Address])(unlift(Person.unapply))
personWrites: com.tiqwab.replication.play.json.Writes[Person] = com.tiqwab.replication.play.json.Writes$$anon$1@7b71dead

scala> Json.toJson(res2)
res3: com.tiqwab.replication.play.json.JsValue = {"name":"Alice", "age":20, "address":{"city":"Tokyo", "country":"Japan"}}
```

<div id="anchor6" />

### 6. 答え合わせ

ここからは実際に play-json のライブラリの実装を見て今回自分が作成したものと比較してみようと思います。

細かい違い含め異なる点は多くあるのですが、その中でも特に気になった点をいくつかまとめます。

#### JsLookup

- `JsValue` に定義していた JSON 構造をたどるためのメソッド (e.g. `\`) は実は `JsLookup` で定義されている
  - `JsValue` には `JsLookup` への implicit conversion が定義されている
- `JsLookup` はデータ構造的には `JsLookupResult` をラップするようなクラス
- `JsLookup` は JSON 上のある特定のパスに対応する値を表す
  - 常に正しく読めるとは限らないので、メソッドの返り値は `JsLookupResult` のように失敗の可能性を表せる型になる

```scala
case class JsLookup(result: JsLookupResult) extends AnyVal {
  def \(index: Int): JsLookupResult = ...
  def \(fieldName: String): JsLookupResult = ...
  def \\(fieldName: String): Seq[JsValue] = ...
  ...
}
```

#### JsValue への変換

- 模写の中の定義は `Writes` から `JsValue` への implicit conversion を定義していたが、実際はそんなにざっくりしたものではない
- `Json` オブジェクトの限られたメソッドでしか変換が行われないようにしている
- その制御は自身で定義している `JsValueWrapperImpl` への implicit conversion

```scala
object Json extends JsonFacade {
  sealed trait JsValueWrapper

  private case class JsValueWrapperImpl(field: JsValue) extends JsValueWrapper

  import scala.language.implicitConversions

  implicit def toJsFieldJsValueWrapper[T](field: T)(implicit w: Writes[T]): JsValueWrapper = JsValueWrapperImpl(w.writes(field))

  def obj(fields: (String, JsValueWrapper)*): JsObject = JsObject(fields.map(f => (f._1, f._2.asInstanceOf[JsValueWrapperImpl].field)))

  ...
}
```

#### JsPath のデータ構造

- `JsPath` は実際は `List[PathNode]` をラップしたようなクラス
- `JsPath#read` が呼ばれたときには `PathNode#apply` をもとに候補となる JsValue を見つける
- `JsPath#write` も `PathNode` をもとにするという点は同じ

```scala
sealed trait PathNode { ... }
case class RecursiveSearch(key: String) extends PathNode { ... }
case class KeyPathNode(key: String) extends PathNode { ... }
case class IdxPathNode(idx: Int) extends PathNode { ... }

case class JsPath(path: List[PathNode] = List()) {
  def \(child: String) = JsPath(path :+ KeyPathNode(child))
  def \\(child: Symbol) = JsPath(path :+ RecursiveSearch(child.name))
  def apply(idx: Int): JsPath = JsPath(path :+ IdxPathNode(idx))
  ...
}
```

#### OWrites

- `JsPath#write` など `JsObject` を返すことが明らかな場合 `Writes` ではなく `OWrites` を使うことがある
- より型安全な実装が可能になる

```scala
case class JsPath(path: List[PathNode] = List()) {
  def write[T](implicit w: Writes[T]): OWrites[T] = Writes.at[T](this)(w)
  ...
}

trait Writes[-A] {
  def writes(o: A): JsValue
  ...
}

trait OWrites[-A] extends Writes[A] {
  def writes(o: A): JsObject
  ...
}
```

#### FunctionalBuilder

- `Reads`, `Writes` のコンビネータ定義の中心は `FunctionalBuilder`
- `FunctionalBuilder` の定義は以下のようになっている
  - `CanBuildx` は 2 から 22 まで地道に同様に定義されている
- `FunctionalCanBuild[M]` を与えることができる型 `M[_]` が対象 
- `Reads` と `Writes` でそれぞれ `FunctionalCanBuild` を与えることができれば、定義の重複を避けることができる

```scala
package play.api.libs.functional

class FunctionalBuilderOps[M[_], A](ma: M[A])(implicit fcb: FunctionalCanBuild[M]) {

  def ~[B](mb: M[B]): FunctionalBuilder[M]#CanBuild2[A, B] = {
    val b = new FunctionalBuilder(fcb)
    new b.CanBuild2[A, B](ma, mb)
  }

  def and[B](mb: M[B]): FunctionalBuilder[M]#CanBuild2[A, B] = this.~(mb)
}

class FunctionalBuilder[M[_]](canBuild: FunctionalCanBuild[M]) {

  class CanBuild2[A1, A2](m1: M[A1], m2: M[A2]) {

    def ~[A3](m3: M[A3]) = new CanBuild3[A1, A2, A3](canBuild(m1, m2), m3)

    def apply[B](f: (A1, A2) => B)(implicit fu: Functor[M]): M[B] =
      fu.fmap[A1 ~ A2, B](canBuild(m1, m2), { case a1 ~ a2 => f(a1, a2) })

    def apply[B](f: B => (A1, A2))(implicit fu: ContravariantFunctor[M]): M[B] =
      fu.contramap(canBuild(m1, m2), { (b: B) =>
        val (a1, a2) = f(b)
        new ~(a1, a2)
      })

    def apply[B](f1: (A1, A2) => B, f2: B => (A1, A2))(implicit fu: InvariantFunctor[M]): M[B] =
      fu.inmap[A1 ~ A2, B](
        canBuild(m1, m2), { case a1 ~ a2 => f1(a1, a2) },
        (b: B) => { val (a1, a2) = f2(b); new ~(a1, a2) }
      )

    ...
  }

  class CanBuild3[A1, A2, A3](m1: M[A1 ~ A2], m2: M[A3]) {
    def ~[A4](m3: M[A4]) = new CanBuild4[A1, A2, A3, A4](canBuild(m1, m2), m3)

    def and[A4](m3: M[A4]) = this.~(m3)

    def apply[B](f: (A1, A2, A3) => B)(implicit fu: Functor[M]): M[B] =
      fu.fmap[A1 ~ A2 ~ A3, B](canBuild(m1, m2), { case a1 ~ a2 ~ a3 => f(a1, a2, a3) })
    ...
  }

}
```

- `FunctionalCanBuild[M]` がどこから来るのかというと以下のように 型 `M[_]` に `Applicative[M]` が定義されていれば導出できる

```scala
package play.api.libs.functional

import scala.language.higherKinds

case class ~[A, B](_1: A, _2: B)

trait FunctionalCanBuild[M[_]] {
  def apply[A, B](ma: M[A], mb: M[B]): M[A ~ B]
}

object FunctionalCanBuild {
  implicit def functionalCanBuildApplicative[M[_]](implicit app: Applicative[M]): FunctionalCanBuild[M] = new FunctionalCanBuild[M] {
    def apply[A, B](a: M[A], b: M[B]): M[A ~ B] = app.apply(app.map[A, B => A ~ B](a, a => ((b: B) => new ~(a, b))), b)
  }
}
```

- `Applicative` 自体は一般的な概念で例えば Haskell の標準ライブラリでは [このように][7] 定義されている
- play-json では以下のように定義されている

```scala
trait Applicative[M[_]] {

  def pure[A](a: A): M[A]
  def map[A, B](m: M[A], f: A => B): M[B]
  def apply[A, B](mf: M[A => B], ma: M[A]): M[B]

}

object Applicative {

  // Option に対する Applicative が定義されている
  implicit val applicativeOption: Applicative[Option] = new Applicative[Option] {

    def pure[A](a: A): Option[A] = Some(a)

    def map[A, B](m: Option[A], f: A => B): Option[B] = m.map(f)

    def apply[A, B](mf: Option[A => B], ma: Option[A]): Option[B] = mf.flatMap(f => ma.map(f))

  }

}
```

- `Reads` には `Applicative[Reads]` の定義が与えれている
- `Writes` は直接 `FunctionalCanBuild[Writes]` が定義されている

```scala
object Reads extends ConstraintReads with PathReads with DefaultReads with GeneratedReads {
  import play.api.libs.functional._

  implicit def applicative(implicit applicativeJsResult: Applicative[JsResult]): Applicative[Reads] = new Applicative[Reads] {

    def pure[A](a: A): Reads[A] = Reads.pure(a)

    def map[A, B](m: Reads[A], f: A => B): Reads[B] = m.map(f)

    def apply[A, B](mf: Reads[A => B], ma: Reads[A]): Reads[B] = new Reads[B] { def reads(js: JsValue) = applicativeJsResult(mf.reads(js), ma.reads(js)) }

  }
  ...
}
```

```scala
object OWrites extends PathWrites with ConstraintWrites {
  import play.api.libs.functional._

  implicit val functionalCanBuildOWrites: FunctionalCanBuild[OWrites] = new FunctionalCanBuild[OWrites] {
    def apply[A, B](wa: OWrites[A], wb: OWrites[B]): OWrites[A ~ B] = MergedOWrites[A, B](wa, wb)
  }
  ...
}
```

#### その他

- implicit conversion 定義時には `scala.language.implicitConversions` を import するべき
  - [参考][6]
- sealed trait に値コンストラクタに対応した結果を返させる場合、そのメソッド定義はどこに置くのがいいのか
  - sealed trait に置いてパターンマッチか、オブジェクト指向的に各サブクラスか
  - play-json は見ている感じどちらもある (e.g. `JsLookupResult`, `PathNode`)
  - 複雑な定義になると各サブクラスで定義したくなるかも
- case class の `Writes` 定義でよく見る `unlift` は実際は `scala.Function.unlift`

<div id="anchor7" />

### 7. 感想

- 感覚的にはライブラリの方がより細かくクラスが定義され、機能が凝集されているように感じた
  - 自分が普段もっとざっくりした定義しかしていないということだと
- 単純にドキュメントやソースを眺めていてもぱっとわからなかったところが自分で作って悩むことで解決できた点もあってよかった
- データ構造とインタフェースを定義すれば実際のメソッド定義は型を合わせに行けば (ひとまずは) 実装できるということが多くて気持ちよかった

また別のライブラリでやってみてもいいかなと思うぐらい、勉強にはなった気がします。

[1]: https://www.playframework.com/documentation/2.6.x/ScalaJson
[2]: https://www.playframework.com/documentation/2.6.x/ScalaJson#The-Play-JSON-library
[3]: https://www.playframework.com/documentation/2.6.x/ScalaJson#Using-Writes-converters
[4]: https://www.playframework.com/documentation/2.6.x/ScalaJson#Traversing-a-JsValue-structure
[5]: https://www.playframework.com/documentation/2.6.x/ScalaJsonCombinators
[6]: https://qiita.com/nojima/items/6b85f387de66e39b964b
[7]: https://hackage.haskell.org/package/base-4.10.0.0/docs/Control-Applicative.html
