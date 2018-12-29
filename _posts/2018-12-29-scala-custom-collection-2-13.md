---
layout: post
title: Scala 2.13 でのカスタムコレクション実装
tags: "scala"
comments: true
---

Scala 2.13 では標準コレクションライブラリの内部実装が大きく変わります。標準コレクションライブラリを使用してカスタムコレクションを実装する方法は [Custom Collection Types][1] に書いてあるようなのですが、現状微妙に内容が古い気がする (対応する github 上のドキュメントは [これ][5]) ので、ライブラリの実装を見ながらこうやれば良さそうというのを整理してみました。

例として以下のオレオレ List を使用して各手順を見ていきます。

```scala
// just wrap scala.collection.immutable.List
class MyList[+A] private (elems: List[A])

object MyList {
  def apply[A](elems: A*): MyList[A] = new MyList(elems.toList)
}
```

環境は

Scala: 2.13.0-M5

です。

また実装の最終形は [gist][4] に上げています。

### 1. データ構造を表す trait の継承

まずやることは以下の 2 つです。

- 実装したデータ構造に応じた trait を継承する
  - Seq or Set or Map
  - immutable or mutable
- 自身のデータ構造用の iterator を提供する

今回は List を wrap しているだけの実装なので List と同様に immutable.LinearSeq を継承するのが良いと思います。

LinearSeq の場合コメントで述べられているように isEmpty, head, tail も新しく実装しないといけません (コンパイルは通るが実行時コレクション生成で失敗する)。

```scala
import scala.collection.immutable

class MyList[+A] private (elems: List[A]) extends immutable.LinearSeq[A] {
  // To be overridden in implementations, as said in scala.collection.LinearSeq
  override def isEmpty: Boolean = elems.isEmpty
  override def head: A = elems.head
  override def tail: MyList[A] = new MyList(elems.tail)

  override def iterator: Iterator[A] = elems.iterator

  override def toString: String = s"MyList(${elems.mkString(",")})"
}

object MyList {
  def apply[A](elems: A*): MyList[A] = new MyList(elems.toList)
}
```

これだけで主要なコレクションメソッドは使用可能です。しかしいくつかメソッドを試してみるとわかるように、 [戻り値同型の原則][2] には従わず LinearSeq を返すような挙動になります。

```
scala> val l = MyList(1, 2, 3)
l: com.tiqwab.example.step2.MyList[Int] = MyList(1,2,3)

// `filter` returns LinearSeq not MyList
scala> l.filter(_ > 1)
res5: scala.collection.LinearSeq[Int] = List(2, 3)

// `map` returns LinearSeq not MyList
scala> l.map(_ + 1)
res6: scala.collection.LinearSeq[Int] = List(2, 3, 4)
```

### 2. XXXOps の mix-in と 3. XXXFactory の提供

戻り値同型の原則に従わせるにはデータ構造に対応する XXXOps を mix-in します。

```scala
import scala.collection.immutable

// mix-in immutable.LinearSeqOps
class MyList[+A] private (elems: List[A])
  extends immutable.LinearSeq[A] with immutable.LinearSeqOps[A, MyList, MyList[A]] {
  override def isEmpty: Boolean = elems.isEmpty
  override def head: A = elems.head
  override def tail: MyList[A] = new MyList(elems.tail)

  override def iterator: Iterator[A] = elems.iterator

  override def toString: String = s"MyList(${elems.mkString(",")})"
}

object MyList {
  def apply[A](elems: A*): MyList[A] = new MyList(elems.toList)
}
```

ただこれだけだとコンパイルは通りますがコレクションメソッド使用時にエラーになります。

```
scala> l.filter(_ > 1)
java.lang.ClassCastException: scala.collection.immutable.$colon$colon cannot be cast to com.tiqwab.example.step3.MyList
  ... 38 elided
```

これはカスタムコレクションの作り方がわからないためなので、その情報を提供する必要があります。標準コレクションライブラリではこの目的に XXXFactory というのが用意されています。ライブラリの実装を見るとあるデータ構造のコンパニオンオブジェクトに XXXFactory を実装させ、それを `iterableFactory` のようなメソッドで参照するという形が多いので、ここでもそれに従い実装します。

```scala
import scala.collection.mutable.ListBuffer
import scala.collection.{immutable, mutable, SeqFactory}

class MyList[+A] private (elems: List[A])
  extends immutable.LinearSeq[A] with immutable.LinearSeqOps[A, MyList, MyList[A]] {
  override def isEmpty: Boolean = elems.isEmpty
  override def head: A = elems.head
  override def tail: MyList[A] = new MyList(elems.tail)

  override def iterator: Iterator[A] = elems.iterator

  // To be overridden for LinearSeqOps
  override def iterableFactory: SeqFactory[MyList] = MyList

  override def toString: String = s"MyList(${elems.mkString(",")})"
}

object MyList extends SeqFactory[MyList] {
  override def from[A](source: IterableOnce[A]): MyList[A] =
    newBuilder[A].addAll(source).result()
  override def empty[A]: MyList[A] =
    new MyList(List.empty)
  override def newBuilder[A]: mutable.Builder[A, MyList[A]] =
    new ListBuffer[A].mapResult(elems => new MyList(elems))
}
```

これで filter や map もうまく動くようになります。

```
scala> val l = MyList(1, 2, 3)
l: com.tiqwab.example.step3.MyList[Int] = MyList(1,2,3)

scala> l.filter(_ > 1)
res0: com.tiqwab.example.step3.MyList[Int] = MyList(2,3)

scala> l.map(_ + 1)
res1: com.tiqwab.example.step3.MyList[Int] = MyList(2,3,4)
```

### 4. Optimized trait を mix-in

ここまでで実装を終えてもいいのですが、性能面を少し気にするならば標準ライブラリの optimized 用の trait の使用を考えます。

わかりやすいのは StrictOptimized... のような trait です。2.13 以降コレクションライブラリの共通化された (つまり IterableOps で提供される) 実装は lazy なものになっています (詳細は [こちら][3])。もし実装しているデータ構造が lazy である必要が無い (strict な) ものならば StrictOptimized... trait を実装した方がコレクションメソッドの性能はよくなるはずです。

```scala
import scala.collection.mutable.ListBuffer
import scala.collection.{immutable, mutable, SeqFactory, StrictOptimizedLinearSeqOps}

class MyList[+A] private (elems: List[A])
  extends immutable.LinearSeq[A] with immutable.LinearSeqOps[A, MyList, MyList[A]]
  with StrictOptimizedLinearSeqOps[A, MyList, MyList[A]] {
...
}
```

### 参考

- [Custom Collection Types - Scala Documentation][1]

[1]: https://docs.scala-lang.org/overviews/core/custom-collections.html 
[2]: https://www.scala-lang.org/docu/files/collections-api/collections-impl.html
[3]: https://www.scala-lang.org/blog/2017/11/28/view-based-collections.html
[4]: https://gist.github.com/tiqwab/bc8d372ca489a74b72dd2357e7d6b010
[5]: https://github.com/scala/docs.scala-lang/blob/fb3881b3332bd5338d7e371f6a77545e741c7107/_overviews/core/custom-collections.md
