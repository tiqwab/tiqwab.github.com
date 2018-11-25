---
layout: post
title: SLS 8.5 あるいは Pattern Matching Anonymous Function
tags: "scala"
comments: true
---

Scala を書いているとたまに以下のようなコンパイルエラーに遭遇します。

```scala
scala> A.h { case (x, y) => x + y }
```

```
<console>:13: error: missing parameter type for expanded function
The argument types of an anonymous function must be fully known. (SLS 8.5)
Expected type was: ?
       A.h { case (x, y) => x + y }
```

反射的に型を明示することで対応していたのですが、2.13 に入る予定のプルリクを見ていく中でこの関数リテラルのこと実はちゃんとわかってないなと気付いたので一度整理しておこうと思いました。

実行環境は別途記載が無い限り Scala 2.12.7 です。

### Pattern Matching Anonymouns Function

上のエラーメッセージでは SLS 8.5 というものが参照されています。SLS は Scala Language Specification のことであり、[8.5][1] は Pattern Matching Anonymous Function についての項です。[コップ本][5] (邦訳) では「15.7.2 部分関数としてのケースシーケンス」で取り上げられており、[実践 Scala 入門][4] では「パターンマッチ無名関数」として何箇所かで扱われています。ここでは長いですが pattern matching anonymous function という名で呼ぶことにします。

pattern matching anonymous function は以下のように case を 1 つ以上並べた形式で定義されます。

```scala
{ case p_1 => b_1 ... case p_n => b_n }
```

pattern matching anonymous function は関数 (`FunctionN[S1, S2, ... Sn, R]`)が求められる場所で使用することができます。

コップ本引用:

> 中括弧で囲んだケースのシーケンスは、関数リテラルが使えるあらゆる位置で使える。ケースシーケンスは関数リテラルであり、それを一般的にしたものにすぎない。

以下の例では `(Int, Int) => Int` が求められる箇所に pattern matching anonymous function を使用しています。

```scala
scala> val f = Seq(1, 2, 3).foldLeft[Int](0) _
f: ((Int, Int) => Int) => Int = $$Lambda$5129/1987567327@3b59250f

scala> f { case (x, y) => x + y }
res24: Int = 6
```

また部分関数 `PartialFunction[S, R]` を求める箇所に使用することができます。

コップ本引用:

> もう一つの一般化の形も知っておくべきだろう。ケースシーケンスは部分関数になるのである。

例えば `scala.util.Try[T]` が持つ `recover` メソッドは `PartialFunction[Throwable, T]` を受け取りますが、そこに pattern matching anonymous function を使用できます。

```scala
scala> val f = Try(0).recover _
f: PartialFunction[Throwable,Int] => scala.util.Try[Int] = $$Lambda$5149/981747008@84367a

scala> f { case NonFatal(e) => 1 }
res31: scala.util.Try[Int] = Success(0)
```

### コンパイルエラーに出会う例

ここまで見た限りだと pattern matching anonymous function は FunctionN にも PartialFunction にもなってくれる便利なやつという印象なのですが、ときたま冒頭のようなコンパイルエラーに出会います。

```scala
scala> A.h { case (x, y) => x + y }
```

```
<console>:13: error: missing parameter type for expanded function
The argument types of an anonymous function must be fully known. (SLS 8.5)
Expected type was: ?
       A.h { case (x, y) => x + y }
```

大体の場合型を明示すれば上手くいくはずです。

```scala
scala> A.h({ case (x, y) => x + y }: Function2[Int, Int, Int])
```

このようなコンパイルエラーになる例としては overload が挙げられます。
以下のような Function をとる関数と PartialFunction をとる関数がある場合、型を明示しないと overload を解決できずコンパイルエラーとなります (ただし実行環境は 2.12.7 です。下で触れますが少なくとも 2.13.0-M5 では挙動が変わります)。

```scala
scala> object A {
     |   def h(f: Function2[Int, Int, Int]): Int = 1
     |   def h(pf: PartialFunction[(Int, Int), Int]): Int = 2
     | }
scala> A.h { case (x, y) => x + y }
```

```
<console>:13: error: missing parameter type for expanded function
The argument types of an anonymous function must be fully known. (SLS 8.5)
Expected type was: ?
       A.h { case (x, y) => x + y }
```

型を明示する、あるいはここでは (pattern matching anonymous function ではない) 関数リテラルを渡すことで解決できます。

```scala
scala> A.h({ case (x, y) => x + y }: Function2[Int, Int, Int])
res2: Int = 1

scala> A.h({ case (x, y) => x + y }: PartialFunction[(Int, Int), Int])
res3: Int = 2

scala> A.h((x, y) => x + y)
res0: Int = 1
```

ただしこの話はどうやら 2.12 までの話で、2.13 では型を明示しない場合 overload の解決においては PartialFunction の方が優先されるようになっています (恐らく [scala/scala#5698][2] により。プルリクを見る感じ本来の目的では無かったけれど変更の結果そうなったという感じっぽいですが)。

```scala
scala> A.h { case (x, y) => x + y }
res0: Int = 2
```

別のコンパイルエラーになる例としては [Scala で何故か型推論できないパターン][8] で触れられているように上限、下限境界を持つような型パラメータが絡む場合もあります。

```scala
scala> case class MySeq[A](xs: Seq[A]) {
     |   def map1[B](f: A => B): MySeq[B] = MySeq(xs.map(f))
     |   def map2[A1 >: A, B](f: A1 => B): MySeq[B] =  MySeq(xs.map(f))
     |   def map3[A1 <: A, B](f: A1 => B): Unit = ()
     | }
defined class MySeq

scala> val seq = MySeq(Seq(1, 2, 3))
seq: MySeq[Int] = MySeq(List(1, 2, 3))

scala> seq.map1((x: Int) => x + 1)
res66: MySeq[Int] = MySeq(List(2, 3, 4))

scala> seq.map1 { case x => x + 1 }
res68: MySeq[Int] = MySeq(List(2, 3, 4))

scala> seq.map2((x: Int) => x + 1)
res69: MySeq[Int] = MySeq(List(2, 3, 4))

scala> seq.map2 { case x => x + 1 }
                ^
       error: missing parameter type for expanded function
       The argument types of an anonymous function must be fully known. (SLS 8.5)
       Expected type was: ? => ?

scala> seq.map3 { case x => () }
                ^
       error: missing parameter type for expanded function
       The argument types of an anonymous function must be fully known. (SLS 8.5)
       Expected type was: ? => ?
```

型推論の詳細を知らないと理解はできませんが、関数のパラメータの型を決定できないとだめだよ、ということみたいですね。

過去に一度は `seq.map2 { case (x: Int) => x + 1 }` というような型アノテーションで解決しようとして失敗した覚えがあります。これがダメなのは結局 `Function1[Int, Int]` にもなりうるし `Function1[Any, Int]` にもなり得るから、ということなのでしょうか。

### 余談

pattern matching anonymous function のよく使われる用途として

```scala
xs map {
  case (x, y) => x + y
}
```

のようにパターンマッチで tuple のパラメータを取り出すみたいのがあると思うのですが、[Dotty][6] だと

```scala
xs.map {
  (x, y) => x + y
}
```

のように TupleN に応じて FunctionN が渡せるようになるみたいですね ([Automatic Tupling of Function Parameters][7])。

[1]: https://www.scala-lang.org/files/archive/spec/2.12/08-pattern-matching.html#pattern-matching-anonymous-functions
[2]: https://github.com/scala/scala/pull/5698
[3]: https://github.com/ReactiveX/RxScala/issues/160
[4]: https://www.amazon.co.jp/%E5%AE%9F%E8%B7%B5Scala%E5%85%A5%E9%96%80-%E7%80%AC%E8%89%AF-%E5%92%8C%E5%BC%98/dp/4297101416
[5]: https://www.amazon.co.jp/Scala%E3%82%B9%E3%82%B1%E3%83%BC%E3%83%A9%E3%83%96%E3%83%AB%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0%E7%AC%AC3%E7%89%88-Martin-Odersky/dp/4844381490
[6]: http://dotty.epfl.ch/
[7]: https://dotty.epfl.ch/docs/reference/auto-parameter-tupling.html
[8]: http://d.hatena.ne.jp/ponkotuy/20131212/1386867686
