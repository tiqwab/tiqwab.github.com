---
layout: post
title: scalaz と cats で有限状態機械のシミュレーション
tags: "scalaz, cats, scala"
comments: true
---

Scala で関数型プログラミングといえば現在は [scalaz][2], [cats][3] の 2 つのライブラリが話題に上がることが多いです。ここでは [有限状態機械][7] (Finite State Machine) を題材にそれぞれのライブラリの使用感、具体的には

- パッケージの import 方針
- モナドトランスフォーマを使用したコードの感じ
- 末尾再帰モナドの利用したコードの感じ

なんかをさっと確認できるようにしたいと思っています。

サンプルコードの全体はこちらの [gist][1] に置いています。内容としては gist で完結しているので、以下は流れを詳細に追っていくという感じになります。また [scalaz][2] や [cats][3] の準備はそれぞれの README に従っています。

環境

- Scala: 2.12.4
- scalaz: 7.2.20
- cats: 1.1.0

### 1. with scalaz

まず scalaz の提供するパッケージのうち必要なものを import します。[scalaz の README][2] や [独習 scalaz の 13 日目 (import ガイド)][4] を参考に、ここでは `StateT` や 標準ライブラリの `Option` に scalaz が提供する型クラスを利用するために以下のパッケージを import します。

```scala
import scalaz._
import scalaz.std.option._
```

有限状態機械をデータ型として考えてみます。有限状態は与えられた入力に応じて自身の状態を変化させていくので、状態を `Int` で表すことにすれば `State Int a` のように表現できると思います。型 a はちょっと悩むのですが、とりあえず消費中の文字列である `String` にしました。また有限状態機械が入力文字列を必ず受理できるとは限らないので、その成否を `Option` として表そうと思います。ということでここでの有限状態機械を以下のような型になります。

```scala
type StateMachine = StateT[Option, Int, String]
```

受理できれば `Some((0, ""))` のような結果が、受理できなければ `None` が返ります。

次に有限状態機械に入力文字列を渡し初期化する関数 `initial` と先頭 1 文字を消費する `consume`  を定義します。本来ならここで遷移規則が登場するはずなのですが、それはまだ導入していないので、一旦 'a' のみを消費して状態は維持するというお粗末な実装でいきます。

```scala
  def initial(input: String): StateMachine = StateT.stateT(input)
  def consume(input: String): StateMachine = StateT { s =>
    input.headOption flatMap {
      case 'a' => Some((s, input.tail)) // only consume 'a' now
      case _   => None
    }
  }
```

使用例はこのような感じです。

```scala
    val res1: StateMachine =
      for {
        str1 <- initial("aaa")
        str2 <- consume(str1)
        str3 <- consume(str2)
        str4 <- consume(str3)
      } yield str4
    println(res1.run(0)) // Some((0,))

    val res2: StateMachine =
      for {
        str1 <- initial("aba")
        str2 <- consume(str1)
        str3 <- consume(str2)
        str4 <- consume(str3)
      } yield str4
    println(res2.run(0)) // None
```

正しく動いていますが冗長なので、与えられた入力文字列が受理可能かを判断する関数 `isAcceptable` を定義します。

```scala
  def consumeAll(input: String): StateMachine = input.headOption match {
    case None => StateT.stateT("")
    case _    => consume(input) flatMap consumeAll
  }
  def isAcceptable(input: String): Boolean =
    consumeAll(input).run(0).isDefined
```

使用してみます。

```scala
  println(s"accept 'aaa': ${isAcceptable("aaa")}") // true
  println(s"accept 'aba': ${isAcceptable("aba")}") // false
```

想定通りです、めでたし... としたいところですが、上で定義した `consumeAll` は自身を (末尾再帰ではない形で) 再帰的に呼び出しているため、長い入力文字列に対してはスタックオーバーフローします。

```
scala> isAcceptable("a" * 1000)
java.lang.StackOverflowError
    at scalaz.IndexedStateT$$Lambda$2694/1964895088.<init>(Unknown Source)
...
```

これは単純にモナドをバインドしていくのではなく、[末尾再帰モナド][5] を利用することで回避できます。scalaz では `BindRec[F[_]]` がそれに当たるみたいです。

```scala
package scalaz

////
/**
 * [[scalaz.Bind]] capable of using constant stack space when doing recursive
 * binds.
 *
 * Implementations of `tailrecM` should not make recursive calls without the
 * `@tailrec` annotation.
 *
  * Based on Phil Freeman's work on stack safety in PureScript, described in
  * [[http://functorial.com/stack-safety-for-free/index.pdf Stack Safety for
  * Free]].
 */
////
trait BindRec[F[_]] extends Bind[F] { self =>
  ////

  def tailrecM[A, B](a: A)(f: A => F[A \/ B]): F[B]
  ...
}
```

`BindRec[F[_]]` を利用して先程の `isAcceptable` を書き直します。

```scala
  def consumeAllTailRec(input: String): StateMachine =
    StateT
      .stateTBindRec[Int, Option]
      .tailrecM[String, String] { a =>
        a.headOption match {
          case None => StateT.stateT(\/-(""))
          case _    => consume(a).map(-\/.apply)
        }
      }(input)
  def isAcceptable(input: String): Boolean =
    consumeAllTailRec(input).run(0).isDefined
```

これで先程の文字列も処理できるようになりました。

```
scala> isAcceptable("a" * 1000)
res2: Boolean = true

scala> isAcceptable("a" * 1000 + "b")
res4: Boolean = false
```

長さの問題が解決できたので、次はちゃんと遷移規則に基づいて処理を行うようにします。遷移規則は遷移元状態、入力文字、遷移先状態の 3 つで構成されるクラスで表現します。一連の遷移規則を `RuleSet` として、これを各処理で渡すために `ReaderT` を使用します。

```scala
  case class TransitionRule(start: Int, char: Char, end: Int)
  type RuleSet = Set[TransitionRule]
  type StateMachineReader = ReaderT[({ type l[A] = StateT[Option, Int, A] })#l, RuleSet, String]
```

`({ type l[A] = StateT[Option, Int, A] })#l` みたいな型の表現は初めて知ったのですが、(いまの) 素の Scala だと一階カインド型を表現するにはこうするか別途定義した `type` を使用する感じになるようです。

これまで定義した関数を `StateMachine` ではなく `StateMachineReader` を扱うように変更します。いきなりこれを見ると少し圧倒されそうなのですが、上を一度書いたあとだと、ただ `ReaderT` を機械的に処理していくだけという感じでいけました。ただ型を明示的に指定しないとうまくコンパイルが通らなかったりするのがちょっと負けた気分です。

```scala
  def initialR(input: String): StateMachineReader = initial(input).liftReaderT[RuleSet]
  def consumeR(input: String): StateMachineReader = {
    def consumeWithRuleSet(str: String, rules: RuleSet): StateMachine =
      StateT { start =>
        for {
          c <- str.headOption
          rule <- rules.find(rule => rule.start == start && rule.char == c)
        } yield (rule.end, str.tail)
      }
    for {
      rules <- ReaderT.ask[({ type l[A] = StateT[Option, Int, A] })#l, RuleSet]
      next <- consumeWithRuleSet(input, rules).liftReaderT[RuleSet]
    } yield next
  }
  // this is not stack-free
  def consumeAllR(input: String): StateMachineReader = input.headOption match {
    case None => StateT.stateT[Option, Int, String]("").liftReaderT[RuleSet]
    case _    => consumeR(input) flatMap consumeAllR
  }
  def isAcceptableR(rules: Set[TransitionRule], start: Int, goal: Int)(input: String): Boolean =
    consumeAllR(input).run(rules).run(start).exists { case (s, _) => s == goal }
```

使用例です。

```scala
val rules = Set(
  TransitionRule(0, 'a', 1),
  TransitionRule(1, 'b', 0)
)

println(s"accept 'ab': ${isAcceptableR(rules, 0, 0)("ab")}") // true
println(s"accept 'aba': ${isAcceptableR(rules, 0, 1)("aba")}") // true
println(s"accept 'ab': ${isAcceptableR(rules, 0, 1)("ab")}") // false
println(s"accept 'aa': ${isAcceptableR(rules, 0, 1)("aa")}") // false
```

### 2. with cats

同じ内容を cats を使用して書いていく... ということをやったのですが細かい違いはあれど scalaz の場合と大差はありません。逆に言えば概念さえわかっており複雑なことをやらなければ補完におんぶにだっこで何とかなりそうに思いました。

あと全体的に `ReaderT` や `StateT` の apply を使用するよりは lift 等を使う方が型推論が上手く行きやすいかなという気がしました (気のせい?)。

[1]: https://gist.github.com/tiqwab/aec7018725df7b91d1cb3c7b292158d9
[2]: https://github.com/scalaz/scalaz
[3]: https://github.com/typelevel/cats
[4]: http://eed3si9n.com/learning-scalaz/ja/day13.html
[5]: http://eed3si9n.com/herding-cats/ja/tail-recursive-monads.html
[6]: https://www.scala-lang.org/
[7]: https://ja.wikipedia.org/wiki/%E6%9C%89%E9%99%90%E3%82%AA%E3%83%BC%E3%83%88%E3%83%9E%E3%83%88%E3%83%B3
