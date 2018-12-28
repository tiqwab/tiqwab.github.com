---
layout: post
title: CanBuildFrom にお別れを
tags: "scala"
comments: true
---

現行の Scala で標準コレクションライブラリをはじめて触ったときに map のメソッドシグネチャを見て困惑した覚えがあります。

```scala
  def map[B, That](f: A => B)(implicit bf: CanBuildFrom[Repr, B, That]): That = ...
```

この CanBuildFrom が何なのかを深く理解しないまま過ごしてきたのですが、どうやら Scala 2.13 では彼が消えてしまうようです。お別れを言う前に彼が陰ながらどのように活躍していたのかその一端でも理解したいと思い、実際に自分で手を動かして CanBuildFrom の役割を感じることにしました。

サンプルに使用したコードの最終形は [gist][5] にあげています。

また特に記載が無い限り Scala 2.12.7 における実装を参考にしています。

---

### 1. オレオレコレクション の定義

まずはサンプルとして使用するためのオレオレコレクションを定義します。

Scala の 標準コレクションライブラリが持つデータ構造は Seq, Set, Map の 3 種類に大別できると思うので、それぞれについて対応するクラスを定義します。ここでは単純に標準の List, Set, Map を wrap したクラスを用意しました。各データ構造にはコレクションに対する代表的な操作として size, filter, map の 3 メソッドを提供するものとします。

(ちなみにメソッドの選び方としては、[The Architecture of Scala 2.13's Collection][1] を参考に reduction 関数, 元と同じ型を返す transformation 関数, 違うコレクション型を返すかもしれない transformation 関数という分類で 1 つずつ選んでいます)

```scala
class MyList[+A] private (elems: List[A]) {
  def size: Int = elems.size
  def filter(p: A => Boolean): MyList[A] = new MyList(elems.filter(p))
  def map[B](f: A => B): MyList[B] = new MyList(elems.map(f))
  override def toString: String = s"MyList(${elems.mkString(",")})"
}

object MyList {
  def apply[A](elems: A*): MyList[A] = new MyList(elems.toList)
}

class MySet[A] private (elems: Set[A]) {
  def size: Int = elems.size
  def filter(p: A => Boolean): MySet[A] = new MySet(elems.filter(p))
  def map[B](f: A => B): MySet[B] = new MySet(elems.map(f))
  override def toString: String = s"MySet(${elems.mkString(",")})"
}

object MySet {
  def apply[A](elems: A*): MySet[A] = new MySet(elems.toSet)
}

class MyMap[A, +B] private (elems: Map[A, B]) {
  def size: Int = elems.size
  def filter(p: ((A, B)) => Boolean): MyMap[A, B] = new MyMap(elems.filter(p))
  def map[C, D](f: ((A, B)) => (C, D)): MyMap[C, D] = new MyMap(elems.map(f))
  override def toString: String = s"MyMap(${elems.mkString(",")})"
}

object MyMap {
  def apply[A, B](elems: (A, B)*): MyMap[A, B] = new MyMap(elems.toMap)
}
```

一応動作を確認しておきます。

```
// MyList

scala> val l = MyList(1, 2, 3)
l: example.MyList[Int] = MyList(1,2,3)

scala> l.size
res0: Int = 3

scala> l.filter(_ > 1).map(_ * 2)
res2: example.MyList[Int] = MyList(4,6)

// MySet

scala> val s = MySet(1, 2, 3)
s: example.MySet[Int] = MySet(1,2,3)

scala> s.size
res3: Int = 3

scala> s.filter(_ > 1).map(_ * 2)
res5: example.MySet[Int] = MySet(4,6)

// MyMap

scala> val m = MyMap("one" -> 1, "two" -> 2)
m: example.MyMap[String,Int] = MyMap(one -> 1,two -> 2)

scala> m.size
res6: Int = 2

scala> m filter { case (k, v) => v > 1 } map { case (k, v) => (k, v * 2) }
res8: example.MyMap[String,Int] = MyMap(two -> 4)
```

ここからこの 3 つのクラスに対し共通の size, filter, map 実装を与えることを目指していきます。上のように 3 つのクラスに 3 つのメソッドぐらいであればその抽象化いる? という感じになるかもしれません。ただ標準ライブラリのように数多くのコレクションクラス、 共通のコレクションメソッドを提供し、しかもそれらが似たような実装になるのならばそのようなモチベーションが出てくるのもわかるように思います。Scala の場合 (自分で確認していないのですが) 2.8 までは各データ構造が別々に実装を持ち、2.8 で現在のコレクション実装になっているようなので、このあたりできっとそういった判断があったのだと思います。

(とはいえ共通に実装を持ったとしても、性能のために実装先のクラスで override するというのは珍しくありません。例えば size なんかはそうであることが多いと思います)

### 2. 共通 trait の作成と size の実装

共通のふるまい、ということでまずは MyTraversable という trait を追加し、各データ構造を表すクラスに継承してもらうというオブジェクト指向的アプローチをとります。 traverse というのはコンピュータサイエンスではあるデータ構造の各要素を辿る、何らかのアクションを適用する、というようなニュアンスの単語で、Scala でいえば foreach のことだと言えると思います。天下り的ですが、標準ライブラリでも Traversable がコレクションライブラリの最上位に位置し、 foreach メソッドのみを要求するという実装になっているのでそれを踏襲したという形です。

```scala
trait MyTraversable[+A] {
  def foreach(f: A => Unit): Unit
  ...
}
```

foreach があれば先程の size は以下のように実装することができます。

```scala
  def size: Int = {
    var count = 0
    foreach(_ => count = count + 1)
    count
  }
```

オレオレコレクション側では MyTraversable を継承して自分なりの foreach を与えれば OK です。

MyList の例:

```scala
class MyList[+A](elems: List[A]) extends MyTraversable[A] {
  override def foreach(f: A => Unit): Unit = elems.foreach(f)
  def filter(p: A => Boolean): MyList[A] = new MyList(elems.filter(p))
  def map[B](f: A => B): MyList[B] = new MyList(elems.map(f))
  override def toString: String = s"MyList(${elems.mkString(",")})"
}
```

これで size メソッドの共通化ができました。

この調子で次に MyTraversable に filter を実装... しようとすると単純にはうまくいかないことに気付きます。

```scala
  def filter(p: A => Boolean): MyTraversable[A] = {
    // Use java.util.ArrayList to append elements, avoiding to use scala collection library
    val jList = new java.util.ArrayList[A]
    foreach(x => if (p(x)) { jList.add(x); () } else ())
    g =>
      jList.forEach(x => g(x)) // SAM conversion
  }
```

(実装で `java.util.ArrayList` を使用していますが、これは Scala 標準ライブラリに依存しないクラスを使いたかったというだけの理由です)

実装の方針としては foreach で各要素を辿り、渡された述語関数 p を満たす要素のみを集める、というのでいいのですが、返り値の型が MyTraversable になってしまいます。MyTraversable からは実装先のクラスのことがわからないのでこうするしかないのですが、本来利用者からすると MyList に filter を適用すれば返ってくる型も MyList であることを期待すると思います。実際 Scala ではこのような原則を [戻り値同型の原則 (same-result-type principle)][2] と呼び、可能な限りコレクションメソッドがこれを満たすようにしています。

### 3. Builder の登場と filter の実装

では filter が戻り値同型の原則を満たすために何が必要か考えると以下の 2 つが挙げられます。

- 返すべきコレクション型の情報
- 返すべきコレクションの構築方法

標準ライブラリではこの役目に `scala.collection.mutable.Builder` を使用しています。ここでは Builder から今回の例に必要な実装を抽出し MyBuilder として以下のように定義しました。

```scala
trait MyBuilder[-Elem, +To] {
  def +=(elem: Elem): this.type
  def result(): To
}
```

`+=` で `Elem` 型の要素を追加していき、 `result` で結果のコレクション型 `To` を返すというのが MyBuilder の使い方です。

この MyBuilder を使用すると filter は以下のように実装できます。

```scala
trait MyTraversable[+A, +Repr] {
  ...

  // access modifier is necessary to use A for the first position of MyBuilder[-Elem, +To]
  protected[this] def newBuilder: MyBuilder[A, Repr]

  def filter(p: A => Boolean): Repr = {
    val b = newBuilder
    foreach(x => if (p(x)) { b += x; () } else ())
    b.result()
  }
}
```

MyTraversable を継承するデータ構造は newBuilder を実装することでどのように自分と同じ型のコレクションを作成すればいいのかという情報を与えます。

```scala
class MyList[+A](elems: List[A]) extends MyTraversable[A, MyList[A]] {
  ...

  override protected[this] def newBuilder: MyBuilder[A, MyList[A]] = new MyBuilder[A, MyList[A]] {
    val l = new java.util.ArrayList[A]
    override def +=(elem: A): this.type = { l.add(elem); this }
    override def result(): MyList[A] = new MyList(l.asScala.toList)
  }
}

class MySet[A](elems: Set[A]) extends MyTraversable[A, MySet[A]] {
  ...

  override protected[this] def newBuilder: MyBuilder[A, MySet[A]] = new MyBuilder[A, MySet[A]] {
    val l = new java.util.ArrayList[A]
    override def +=(elem: A): this.type = { l.add(elem); this }
    override def result(): MySet[A] = new MySet(l.asScala.toSet)
  }
}

class MyMap[A, +B](elems: Map[A, B]) extends MyTraversable[(A, B), MyMap[A, B]] {
  ...

  override protected[this] def newBuilder: MyBuilder[(A, B), MyMap[A, B]] = new MyBuilder[(A, B), MyMap[A, B]] {
    val l = new java.util.ArrayList[(A, B)]
    override def +=(elem: (A, B)): this.type = { l.add(elem); this }
    override def result(): MyMap[A, B] = new MyMap(l.asScala.toMap)
  }
}
```

これで filter に対する共通実装を提供することができました。

### 4. CanBuildFrom の登場と map の実装

残るメソッドは map だけです。試しに MyList の map をもとに抽象化を試みます。

```scala
trait MyTraversable[A, C[_]] {
  def foreach(f: A => Unit): Unit
  def newBuilder[B]: mutable.Builder[B, C[B]]

  def size: Int = {
    var count = 0
    foreach(_ => count = count + 1)
    count
  }
  def map[B](f: A => B): C[B] = {
    val b = newBuilder[B]
    foreach(x => { b += f(x); () })
    b.result()
  }
  def filter(p: A => Boolean): C[A] = {
    val b = newBuilder[A]
    foreach(x => if (p(x)) { b += x; () } else ())
    b.result()
  }
}
```

MyTraversable の型パラメータとして `C[_]` を受け取り、 map メソッドで `C[A]` から `C[B]` を作れるように定義してみました。一見上手くいきそうなのですが、 `C[_]` では型パラメータを 2 つとる MyMap との共通化がうまくいきません。

また別の問題として、 Scala の標準コレクションライブラリの map では渡す関数によって結果のコレクション型が変化するのですが、その挙動を再現できていないというのがあります。一つの例として SortedSet の変換を以下に記載します。この場合変換後のコレクションの要素型に Ordering が定義されているか否かで結果のコレクション型が変化します。

```scala
scala> import scala.collection.SortedSet
import scala.collection.SortedSet

// Define Person without Ordering
scala> case class Person(name: String, age: Int)
defined class Person

scala> val s = SortedSet(3, 2, 1)
s: scala.collection.SortedSet[Int] = TreeSet(1, 2, 3)

// The result of map is TreeSet
scala> s.map(_ * -1)
res0: scala.collection.SortedSet[Int] = TreeSet(-3, -2, -1)

// The result of map is not SortedSet
scala> s.map(x => Person("Alice", x))
res0: scala.collection.Set[Person] = Set(Person(Alice,1), Person(Alice,2), Person(Alice,3))
```

(こんな柔軟性いる? と一瞬思ったのですが、SortedSet が Traversable を継承しているのでこの挙動が無いと Liskov の置換原則に違反することになってしまいます)

上の 2 つの問題を解決する 1 つのアプローチは、map 評価時に必要な結果型を判断し欲しい MyBuilder をどこかから引っ張ってくることです。このために Builder を提供する CBF という型を用意し、map メソッドの implicit parameter として受け取れるようにします。

```scala
trait CBF[-From, -Elem, +To] {
  def apply(): MyBuilder[Elem, To]
}
```

```scala
trait MyTraversable[+A, +Repr] {
  ...
  def map[B, That](f: A => B)(implicit cbf: CBF[Repr, B, That]): That = {
    val b = cbf()
    foreach(x => { b += f(x); () })
    b.result()
  }
  ...
}
```

各コレクションクラスでは自分用の CBF をコンパニオンオブジェクトで定義します。

```scala
class MyList[+A](elems: List[A]) extends MyTraversable[A, MyList[A]] {
  override def foreach(f: A => Unit): Unit = elems.foreach(f)
  override protected[this] def newBuilder: MyBuilder[A, MyList[A]] = MyList.newBuilder[A]
}

object MyList {
  def apply[A](elems: A*): MyList[A] = new MyList(elems.toList)

  def newBuilder[B]: MyBuilder[B, MyList[B]] = new MyBuilder[B, MyList[B]] {
    val l = new java.util.ArrayList[B]
    override def +=(elem: B): this.type = { l.add(elem); this }
    override def result(): MyList[B] = new MyList(l.asScala.toList)
  }

  implicit def cbf[From, Elem]: CBF[From, Elem, MyList[Elem]] = () => newBuilder[Elem]
}

class MySet[A](elems: Set[A]) extends MyTraversable[A, MySet[A]] {
  override def foreach(f: A => Unit): Unit = elems.foreach(f)
  override protected[this] def newBuilder: MyBuilder[A, MySet[A]] = MySet.newBuilder[A]
}

object MySet {
  def apply[A](elems: A*): MySet[A] = new MySet(elems.toSet)

  def newBuilder[B]: MyBuilder[B, MySet[B]] = new MyBuilder[B, MySet[B]] {
    val l = new java.util.ArrayList[B]
    override def +=(elem: B): this.type = { l.add(elem); this }
    override def result(): MySet[B] = new MySet(l.asScala.toSet)
  }

  implicit def cbf[From, Elem]: CBF[From, Elem, MySet[Elem]] = () => newBuilder[Elem]
}

class MyMap[A, +B](elems: Map[A, B]) extends MyTraversable[(A, B), MyMap[A, B]] {
  override def foreach(f: ((A, B)) => Unit): Unit = elems.foreach(f)
  override protected[this] def newBuilder: MyBuilder[(A, B), MyMap[A, B]] = MyMap.newBuilder[A, B]
}

object MyMap {
  def apply[A, B](elems: (A, B)*): MyMap[A, B] = new MyMap(elems.toMap)

  def newBuilder[A, B]: MyBuilder[(A, B), MyMap[A, B]] = new MyBuilder[(A, B), MyMap[A, B]] {
    val l = new java.util.ArrayList[(A, B)]
    override def +=(elem: (A, B)): this.type = { l.add(elem); this }
    override def result(): MyMap[A, B] = new MyMap(l.asScala.toMap)
  }

  implicit def cbf[From, A, B]: CBF[From, (A, B), MyMap[A, B]] = () => newBuilder[A, B]
}
```

これで map も MyTraversable に実装することができました。

ここで使用した CBF がまさに CanBuildFrom です。つまり CanBuildFrom は

- 汎用的なコレクション操作メソッドをできるだけ一箇所で定義したい
- メソッドに渡した関数によって結果のコレクション型を変えたい

といった課題を解決するために導入された仕組みということになります。

CBF の仕組みを利用することで先の SortedSet と同様な挙動も実現することができます。

```scala
class MySortedSet[A: Ordering](elems: immutable.SortedSet[A]) extends MySet(elems) with MyTraversable[A, MySortedSet[A]] {
  override def foreach(f: A => Unit): Unit = elems.foreach(f)
  override def newBuilder: MyBuilder[A, MySortedSet[A]] = MySortedSet.newBuilder[A]
}

object MySortedSet {
  def apply[A: Ordering](elems: A*): MySortedSet[A] = new MySortedSet(immutable.SortedSet(elems: _*))

  def newBuilder[A: Ordering]: MyBuilder[A, MySortedSet[A]] = new MyBuilder[A, MySortedSet[A]] {
    val l = new java.util.ArrayList[A]
    override def +=(elem: A): this.type = { l.add(elem); this }
    override def result(): MySortedSet[A] = MySortedSet(l.asScala: _*)
  }

  implicit def cbf[From, Elem: Ordering]: CBF[From, Elem, MySortedSet[Elem]] = () => MySortedSet.newBuilder[Elem]
}
```

```
scala> val s = MySortedSet(3, 2, 1)
s: example.MySortedSet[Int] = MySortedSet(1,2,3)

scala> s.map(_ + 1)
res0: example.MySortedSet[Int] = MySortedSet(2,3,4)

scala> s.map(x => Person("Alice", x))
res1: example.MySet[Person] = MySet(Person(Alice,1),Person(Alice,2),Person(Alice,3))
```

---

ということで一からコレクションライブラリの実装を共通化していく中でどのように CanBuildFrom が役に立ったのかということを見ていきました。

プログラミングにおいて多くの場面でそうであるように、コレクションライブラリの共通化もやり方は一つではなく、実際に Scala 2.13 では CanBuildFrom を使用しない実装に移行されます。なので CanBuildFrom 自体の知識が今後役に立つことは多くは無いかもしれませんが、課題とそれに対する implicit parameter による解決法の組み合わせとして面白い話だなと思いました。

参考:

- [Scala 2.13 Collection Rework][1]
- [The Architecture of Scala Collections][2]
- [Tribulations of CanBuildFrom][4]
- [Scala の CanBuildFrom について][3]

[1]: https://docs.scala-lang.org/overviews/core/architecture-of-scala-213-collections.html
[2]: https://www.scala-lang.org/docu/files/collections-api/collections-impl.html
[3]: https://togetter.com/li/157234
[4]: https://www.scala-lang.org/blog/2017/05/30/tribulations-canbuildfrom.html
[5]: https://gist.github.com/tiqwab/40b6ddb9c49656a897fbacd9ad67e85b
