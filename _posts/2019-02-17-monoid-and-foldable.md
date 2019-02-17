---
layout: post
title: Monoid と Foldable について
tags: "monoid, foldable, functional programming, scala"
comments: true
---

最近 [FP in Scala][11] や [すごい H 本][12] を読み返しています。これらの本では型クラスとしていくつか解説があるのですが、その中でも Monoid, Foldable について自分なりの理解をまとめてみます。

なお途中いくつか Scala で実装例を載せていますがこれらはあくまで理解のためであり、実務で使用する際には [Scalaz][13] や [Cats][14] といったライブラリを使用する方が無難だと思います。

1. [Monoid](#monoid)
2. [Foldable](#foldable)

---

<div id="monoid" />

## Monoid

Scala において [Monoid][15] はある型と単位元、その上の二項演算という形で表されます。

```scala
trait Monoid[A] {
  def zero: A
  def append(a1: A, a2: A): A
}
```

Monoid が満たすべき性質として

- associativity (結合律)
- left identity (左恒等性)
- right identity (右恒等性)

があり、それぞれ以下のように表すことができます。

```scala
  trait MonoidLaws {
    def associativity(a1: A, a2: A, a3: A): Boolean =
      append(append(a1, a2), a3) == append(a1, append(a2, a3))
    def leftIdentity(a: A): Boolean =
      append(zero, a) == a
    def rightIdentity(a: A): Boolean =
      append(a, zero) == a
  }
```

例えば Int 上の加算と乗算は Monoid です。

```scala
  def intAddMonoid: Monoid[Int] = new Monoid[Int] {
    override def zero: Int = 0
    override def append(a1: Int, a2: Int): Int = a1 + a2
  }

  def intMultMonoid: Monoid[Int] = new Monoid[Int] {
    override def zero: Int = 1
    override def append(a1: Int, a2: Int): Int = a1 * a2
  }
```

また Tuple2 や Function1 等必要な Monoid が既に定義されていれば自動的に Monoid を導出することができるものもあります。Monoid 自体は型とその上の二項演算によって定義されるものですが、これらについてはそのデータ構造から自然と Monoid が決められると考えていいんだと思います。

(例えば Haskell では標準でこれらと同様な Monoid が定義されている)

```scala
  implicit def tuple2Monoid[A, B](a: A, b: B)(implicit ma: Monoid[A], mb: Monoid[B]): Monoid[(A, B)] =
    new Monoid[(A, B)] {
      override def zero: (A, B) =
        (ma.zero, mb.zero)
      override def append(x: (A, B), y: (A, B)): (A, B) =
        (ma.append(x._1, y._1), mb.append(x._2, y._2))
    }

  implicit def function1Monoid[A, B](implicit m: Monoid[B]): Monoid[A => B] =
    new Monoid[A => B] {
      override def zero: A => B =
        _ => m.zero
      override def append(f1: A => B, f2: A => B): A => B =
        a => m.append(f1(a), f2(a))
    }
```

[FP in Scala][11] ではこのような性質を 「Monoid は合成できる」 と表現しています (c.f. Monad は一般には合成できない)。

### Semigroup

Scalaz や Cats における Monoid の定義を見ると Semigroup (半群) というものを継承していることがわかります。Semigroup はある型 A とその上の結合律を満たす二項演算として表されます。

```scala
trait Semigroup[A] {
  def append(a1: A, a2: A): A
}
```

Monoid の定義と比べるとわかるように Monoid は 単位元を持つ Semigroup です。

```scala
trait Monoid[A] extends Semigroup[A] {
    def zero: A
}
```

Semigroup の例としては Scalaz や Cats でいう NonEmptyList が挙げられます。普通の List は空の List を単位元、List 同士の連結を二項演算として Monoid が定義できますが、NonEmptyList の場合は空の List が存在しないために Monoid を定義できません。

```scala
// List can be instance of Monoid
sealed trait List[+A]
case object Nil extends List[Nothing]
final case class Cons[+A](head: A, tail: List[A]) extends List[A]

object List {
  implicit def monoid[A]: Monoid[List[A]] = new Monoid[List[A]] {
    override def zero: List[A] = Nil
    override def append(a1: List[A], a2: List[A]): List[A] = {
      @scala.annotation.tailrec
      def loop(xs: List[A], acc: List[A]): List[A] = xs match {
        case Nil => acc
        case Cons(h, t) => loop(t, Cons(h, acc))
      }
      loop(a1, a2)
    }
  }
}
```

```scala
// NonEmptyList can be instance of Semigroup but not Monoid
final class NonEmptyList[A](val head: A, val tail: List[A])

object NonEmptyList {
  implicit def semigroup[A]: Semigroup[NonEmptyList[A]] = new Semigroup[NonEmptyList[A]] {
    override def append(a1: NonEmptyList[A], a2: NonEmptyList[A]): NonEmptyList[A] =
      new NonEmptyList(a1.head, a1.tail ++ List(a2.head) ++ a2.tail)
  }
}
```

(本筋とあまり関係ないですが、Scalaz と Cats それぞれの NonEmptyList の定義を見ると Scalaz は invariant で Cats は covariant にしているんですね。[ValidatedNel and NonEmptyList are invariant - typelevel/cats][8] を見る限りだと covariant な方が (Scala ユーザ的には) 使い勝手が良いかもしれないけど invariant の方が型推論的には利点がある、みたいに見えます)

### Homomorphisms

ある型 A, B に対しそれぞれ Monoid M, N が定義されており、関数 `f: A => B` が Monoid 構造を維持する写像ならば、Monoid Homomorphisms (モノイド準同型写像) と呼び、任意の a1: A, a2: A に対し

`f(M.op(a1, a2)) == N.op(f(a1), f(a2))`


が成立します。

例えば A として String, B として Int, f として `(a: String) => a.length`, M として文字列の連結、 N として Int 上の加算を考えると f は Monoid homomorphism になるようです。

```scala
scala> ("abc" ++ "de").length == ("abc".length + "de".length)
res15: Boolean = true
```

この例をもとに Monoid Homomorphism である嬉しみを考えてみると、仮に length の計算量は文字列の長さの 2 乗に比例だ (もちろん本当はそんなこと無いのですが) とすると右辺の方が効率のいい計算になるよね、といったことの判断に使えるとかがありそうです。

### with Scalaz

冒頭でも述べたように実務で Monoid のような型クラスを使用する場合、ライブラリの定義を利用することが多いと思います。Scala の場合だと Scalaz と Cats というライブラリがあるので、それらを使用した Monoid の利用例というのも見ておきます。

これらのライブラリを使用する場合、地味に難しいのは欲しい定義がどこにあるのかということだと感じます。Scalaz で何をどう import すればいいのかという方針は [独習 Scalaz - 13日目 (import ガイド)][9] がわかりやすかったです。

```scala
// import names such as type classes
scala> import scalaz._
import scalaz._

// import type classes (and helper function?) of list
scala> import scalaz.std.list._
import scalaz.std.list._

scala> val m = implicitly[Monoid[List[Int]]]
m: scalaz.Monoid[List[Int]] = scalaz.std.ListInstances$$anon$4@561edf87

scala> m.append(m.append(List(1, 2), List(3, 4)), m.zero)
res10: List[Int] = List(1, 2, 3, 4)

// import syntax for monoid
scala> import scalaz.syntax.monoid._
import scalaz.syntax.monoid._

scala> List(1, 2) mappend List(3, 4) mappend mzero
res8: List[Int] = List(1, 2, 3, 4)
```

### with Cats

Scalaz と同様に [猫番 - import ガイド][10] が Cats の import 方針の参考になります。

```scala
// import names such as type classes
scala> import cats._
import cats._

// import type classes (and helper function?) of list
scala> import cats.instances.list._
import cats.instances.list._

scala> val m = implicitly[Monoid[List[Int]]]
m: cats.Monoid[List[Int]] = cats.kernel.instances.ListMonoid@22a8a4d6

scala> m.combine(m.combine(List(1, 2), List(3, 4)), m.empty)
res2: List[Int] = List(1, 2, 3, 4)

// import syntax for monoid
scala> import cats.syntax.monoid._
import cats.syntax.monoid._

scala> List(1, 2) |+| List(3, 4) |+| Monoid[List[Int]].empty
res5: List[Int] = List(1, 2, 3, 4)
```

### Monoid 利用応用例

検索でぽろぽろ見つけたものを載せておきます。

- [Monoidってどういうところで使えばいいのさ...簡単な実装例][5]
- [CRDT (Conflict-free Replicated Data Types) を15分で説明してみる][3]
  - 可換な Monoid であれば CRDT で表現できるよう

<div id="foldable" />

## Foldable

Scala のコレクションライブラリには foldLeft や foldRight のようないわゆる 「畳み込み」 を行う関数が定義されています。

```scala
// (((((0 - 1) - 2) - 3) - 4) - 5)
scala> List(1, 2, 3, 4, 5).foldLeft(0)(_ - _)
res8: Int = -15

// (1 - (2 - (3 - (4 - (5 - 0)))))
scala> List(1, 2, 3, 4, 5).foldRight(0)(_ - _)
res7: Int = 3
```

このような畳み込みは 「データ構造の持つ各要素から何らかの計算により単一の値を返す」 という操作であり、コレクションライブラリのクラスに限らず種々のデータ構造でも定義することができます。そのような畳み込みを行えるデータ構造を表現するために Foldable という型クラスが存在します。

```scala
trait Foldable[F[_]] {
  import Foldable._

  def foldMap[A, B](xs: F[A])(f: A => B)(implicit m: Monoid[B]): B =
    foldRight(xs)(m.zero)((a, b) => m.append(f(a), b))
  def foldLeft[A, B](xs: F[A])(z: B)(f: (B, A) => B): B =
    foldMap(xs)((a: A) => (b: B) => f(b, a))(dual(endoMonoid[B]))(z)
  def foldRight[A, B](xs: F[A])(z: B)(f: (A, B) => B): B =
    foldMap(xs)(f.curried)(endoMonoid[B])(z)
}

object Foldable {
  private def dual[A](m: Monoid[A]): Monoid[A] = new Monoid[A] {
    def append(x: A, y: A): A = m.append(y, x)
    def zero = m.zero
  }

  private def endoMonoid[A]: Monoid[A => A] =
    new Monoid[A => A] {
      override def zero: A => A = a => a
      override def append(f1: A => A, f2: A => A): A => A = f1 compose f2
    }
}
```

Haskell における Foldable の定義 (GHC 8.6.3) を見ると Foldable を定義するためには `foldMap` あるいは `foldRight` の実装を与えればよいということだったので、それを踏襲した実装にしました。

例えば以下のような二分木に対する Foldable は以下のように定義できます。

```scala
sealed trait Tree[A]
case class Leaf[A](value: A) extends Tree[A]
case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]

object Tree {
  implicit val treeFoldable: Foldable[Tree] = new Foldable[Tree] {
    override def foldMap[A, B](xs: Tree[A])(f: A => B)(implicit m: Monoid[B]): B = xs match {
      case Leaf(v)      => f(v)
      case Branch(l, r) => m.append(foldMap(l)(f), foldMap(r)(f))
    }
  }
}
```

```scala
scala> val tree: Tree[Int] = Branch(Branch(Leaf(1), Leaf(2)), Branch(Leaf(3), Branch(Leaf(4), Leaf(5))))
tree: com.tiqwab.example.Tree[Int] = Branch(Branch(Leaf(1),Leaf(2)),Branch(Leaf(3),Branch(Leaf(4),Leaf(5))))

scala> def intAddMonoid: Monoid[Int] = new Monoid[Int] {
     |   override def zero: Int = 0
     |   override def append(a1: Int, a2: Int): Int = a1 + a2
     | }

scala> implicitly[Foldable[Tree]].foldMap(tree)(identity)(intAddMonoid)
res2: Int = 15
```

上の Foldable の定義なのですが、最初 foldRight, foldLeft が foldMap から実装できるということが少し不思議に感じました。foldRight, foldLeft は同じ引数 z, f を与えても f が結合律を満たす演算でない (例えば Int 上の減算) ならば計算結果は異なり得ます。それなのに foldMap という Monoid を求める関数から実装できるのか? という感じで。

ただ foldRight の定義をじっくり追うと、これは自分が foldMap で求めている Monoid を勘違いしていたせいだということがわかりました。

上述した Foldable の定義から foldRight を中心に抜粋します。

```scala
  def foldRight[A, B](xs: F[A])(z: B)(f: (A, B) => B): B =
    foldMap(xs)(f.curried)(endoMonoid[B])(z)

  def foldMap[A, B](xs: F[A])(f: A => B)(implicit m: Monoid[B]): B =
    foldRight(xs)(m.zero)((a, b) => m.append(f(a), b))

...

  private def endoMonoid[A]: Monoid[A => A] =
    new Monoid[A => A] {
      override def zero: A => A = a => a
      override def append(f1: A => A, f2: A => A): A => A = f1 compose f2
    }
```

よく見ると foldRight の使用する Monoid は endoMonoid ということで関数合成を行う Monoid なんですね。

foldMap では xs の各要素に f の演算を行いますが、foldRight の場合これは `f.curried` ということで `f: A => (B => B)` のように `a: A` を受け取って `B => B` という関数を返す関数と見ることができます。このとき f という関数を xs の i 番目の要素に適用し得られた関数を \\( g\_i \\) と表現することにすると、 \\( g\_i \\) 同士は endoMonoid という単純な関数合成として合成されるので結果として \\( g\_1 \\circ g\_2 \\circ g\_3 \\circ g\_4 \\circ g\_5 \\) という `B => B` な関数が得られます。これは foldRight の計算順と一致しているので合成された関数に z を渡せば期待通りの計算結果が得られるという感じです。

文章で上手く説明できている自信が無いので簡単なメモ書きも載せてみます。

<figure>
  <img
    src="/images/monoid/foldRight.jpg"
    title="foldRight via foldMap"
    alt="foldRight via foldMap"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
  <figcaption>Implementation of foldRight via foldMap</figcaption>
</figure>

foldLeft の場合も話はほとんど同じなのですが、 \\( g\_i \\) の関数合成の方法が `endoMonoid` ではなく `dual(endoMonoid)` というように変わります。 dual? となるのですがこれは endoMonoid が `f compose g`  あるいは `(a: A) => f(g(a))` のような合成だったとすると `f andThen g` あるいは `(a: A) => g(f(a))` のような形になっただけです。

`dual(endoMonoid` を使用するだけで foldRight ではなく foldLeft になる、というのも実際にメモ書きしてみるとわかりやすいように思います。

<figure>
  <img
    src="/images/monoid/foldLeft.jpg"
    title="foldLeft via foldMap"
    alt="foldLeft via foldMap"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
  <figcaption>Implementation of foldLeft via foldMap</figcaption>
</figure>

### With Scalaz

事前に上でも使用した Tree と Int 上の加算に対する Monoid を定義しておきます。

```scala
scala> :paste
// Entering paste mode (ctrl-D to finish)

sealed trait Tree[A]
case class Leaf[A](value: A) extends Tree[A]
case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]

// Exiting paste mode, now interpreting.

defined trait Tree
defined class Leaf
defined class Branch

scala> val tree: Tree[Int] = Branch(Leaf(1), Branch(Leaf(2), Leaf(3)))
tree: Tree[Int] = Branch(Leaf(1),Branch(Leaf(2),Leaf(3)))

scala> val addMonoid = new Monoid[Int] {
     |   override val zero: Int = 0
     |   override def append(a1: Int, a2: => Int): Int = a1 + a2
     | }
addMonoid: scalaz.Monoid[Int]{val zero: Int} = $anon$1@164b89c3
```

Scalaz における Foldable は以下のように利用できます。

```scala
// import necessary names
scala> import scalaz.{Monoid, Foldable}
import scalaz.{Monoid, Foldable}

scala> implicit val treeFoldable: Foldable[Tree] = new Foldable.FromFoldMap[Tree] {
     |   override def foldMap[A, B](fa: Tree[A])(f: A => B)(implicit F: Monoid[B]): B = fa match {
     |     case Leaf(v) => f(v)
     |     case Branch(l, r) => F.append(foldMap(l)(f), foldMap(r)(f))
     |   }
     | }
treeFoldable: scalaz.Foldable[Tree] = $anon$1@33662f77

scala> treeFoldable.foldMap(tree)(identity)(addMonoid)
res5: Int = 6

// import syntax of Foldable
scala> import scalaz.syntax.foldable._
import scalaz.syntax.foldable._

scala> tree.foldMap(identity)(addMonoid)
res10: Int = 6
```

### With Cats

Cats の場合の Foldable です。

```scala
// import necessary names
scala> import cats.{Monoid, Foldable, Eval}
import cats.{Monoid, Foldable}

scala> implicit val treeFoldable: Foldable[Tree] = new Foldable[Tree] {
     |   override def foldLeft[A, B](fa: Tree[A], b: B)(f: (B, A) => B): B = fa match {
     |     case Leaf(v)      => f(b, v)
     |     case Branch(l, r) => foldLeft(r, foldLeft(l, b)(f))(f)
     |   }
     |   override def foldRight[A, B](fa: Tree[A], lb: Eval[B])(f: (A, Eval[B]) => Eval[B]): Eval[B] = fa match {
     |     case Leaf(v)      => f(v, lb)
     |     case Branch(l, r) => foldRight(l, foldRight(r, lb)(f))(f)
     |   }
     | }
treeFoldable: cats.Foldable[com.tiqwab.example.cats.MyFoldable.Tree] = $anon$1@47bbac44

scala> treeFoldable.foldMap(tree)(identity)(addMonoid)
res2: Int = 6

// import syntax of Foldable
scala> import cats.syntax.foldable._
import cats.syntax.foldable._

scala> tree.foldMap(identity)(addMonoid)
res3: Int = 6
```

---

### 参考

- [Haskell/Monoids][1]
- [Monoid - HaskellWiki][2]
- [Monoidってどういうところで使えばいいのさ...簡単な実装例][5]
- [CRDT (Conflict-free Replicated Data Types) を15分で説明してみる][3]
- [Distributed Data - Akka][4]
- [Cats: Type Classes][7]

[1]: https://en.wikibooks.org/wiki/Haskell/Monoids
[2]: https://wiki.haskell.org/Monoid
[3]: https://qiita.com/everpeace/items/bb73ec64d3e682279d26
[4]: https://doc.akka.io/docs/akka/2.5.4/scala/distributed-data.html
[5]: http://labs.septeni.co.jp/entry/2016/01/21/132531
[6]: https://github.com/fpinscala/fpinscala/wiki/Chapter-12:-Applicative-and-traversable-functors
[7]: https://typelevel.org/cats/typeclasses.html
[8]: https://github.com/typelevel/cats/issues/1419
[9]: http://eed3si9n.com/learning-scalaz/ja/day13.html
[10]: http://eed3si9n.com/herding-cats/ja/import-guide.html
[11]: https://www.amazon.co.jp/exec/obidos/ASIN/4844337769/
[12]: https://www.amazon.co.jp/dp/B009RO80XY
[13]: https://github.com/scalaz/scalaz
[14]: https://github.com/typelevel/cats
[15]: https://ja.wikipedia.org/wiki/%E3%83%A2%E3%83%8E%E3%82%A4%E3%83%89
