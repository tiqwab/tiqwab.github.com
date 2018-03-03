---
layout: post
title: scala.concurrent.Future をもう一歩理解したい
tags: "future, promise, scala"
comments: true
---

Scala 界隈ではお天気の次に共通の話題にできる (嘘) scala.concurrent.Future ですが、一年弱使用してきていまだにその挙動について上手く整理できておらずどことなく苦手意識がありました。例えば

```scala
Future.successful(1) flatMap { x =>
  Future.successful(2) flatMap { y =>
    Future.successful(3) map { z =>
      x + y + z
    }
  }
}
```

というコードを見たときに、全体としては `Future[Int]` 型が得られ、計算結果が 6 になるということは理解していますが、その中でどんな Future が生成されどのように最終的な結果が得られるのかはあまりイメージできていませんでした。

そこで Future とより向き合うために本実装から必要最低限のみを抽出した実装を自分で作ってみようと思います。できるだけ本実装の処理を外れないように、でも性能的な面やエラーハンドリングを無視することで単純な例にすることを目指します。

1. [Future と Promise トレイト](#anchor1)
2. [ざっくり Future 実装](#anchor2)
3. [ExecutionContext 導入](#anchor3)
4. [MyFuturePromise 実行時の挙動理解](#anchor4)

環境:

- Scala 2.12.4

<div id="anchor1" />

### 1. Future と Promise トレイト

Scala の標準ライブラリでは並行、並列プログラミングのために `scala.concurrent` パッケージが用意されており、Future や Promise トレイトはここで定義されています。 これらが端的に何かということは [FutureとPromise - Scala研修テキスト][1] の説明がわかりやすいと思うので引用させて頂きます。

> FutureとPromiseは非同期プログラミングにおいて、終了しているかどうかわからない処理結果を抽象化した型です。Futureは未来の結果を表す型です。Promiseは一度だけ、成功あるいは失敗を表す、処理または値を設定することでFutureに変換できる型です。

Future には多くのメソッドが定義されていますが、そのモデルに本質的 (と思われて) かつ実装が与えられていないものは `onComplete`, `transform`, `transformWith` の 3 つしかありません。よく使用する `map` や `flatMap` もそれぞれ `transform`, `transformWith` から定義することができます。ここではこの 3 メソッドを抜き出し新たに MyFuture というトレイトを作成します。なお本来各メソッドの第二パラメータリストには `ExecutionContext` が渡されますが、そこはあとからの対応として一旦無視します。

```scala
import scala.util.Try

trait MyFuture[+T] {
  def onComplete[U](f: Try[T] => U): Unit
  def transform[S](f: Try[T] => Try[S]): MyFuture[S]
  def transformWith[S](f: Try[T] => MyFuture[S]): MyFuture[S]
}
```

型から判断すると、これらはいずれも MyFuture の計算が完了したときに何をしたいのかを定義するためのメソッドかなと推測できます。

次に Promise ですが、同様に未実装なメソッドを抜き出して MyPromise として独自に定義します。

```scala
import scala.util.Try

trait MyPromise[T] {
  def future: MyFuture[T]
  def tryComplete(result: Try[T]): Boolean
}
```

Promise は一度だけ値が設定でき、Future に変換できるという性質を持つ型でした。 `future` はそのままですし、`tryComplete` も渡した値を設定できたかを返すと捉えれば理解できると思います。

これらのインタフェースに沿えば、ただ値をラップするだけの MyFuture, MyPromise 実装はすぐに与えることができます。

```scala
import scala.util.{Success, Try}

private[future] class MySimpleFuture[+T](value: Try[T]) extends MyFuture[T] {
  // 生成時に既に結果を持つので、ただ渡された関数による計算を行う
  override def onComplete[U](f: Try[T] => U): Unit = f(value)
  override def transform[S](f: Try[T] => Try[S]): MyFuture[S] = new MySimpleFuture(f(value))
  override def transformWith[S](f: Try[T] => MyFuture[S]): MyFuture[S] = f(value)
}

object MySimpleFuture {
  def apply[T](value: Try[T]): MyFuture[T] =
    new MySimplePromise(value).future
  def successful[T](v: T): MyFuture[T] =
    new MySimplePromise(Success(v)).future
}

class MySimplePromise[T](value: Try[T]) extends MyPromise[T] {
  override def future: MyFuture[T] = new MySimpleFuture(value)
  // 生成時に値が設定されるので、常に false となる
  override def tryComplete(result: Try[T]): Boolean = false
}
```

```
scala> MySimpleFuture.successful(1) transform {
     |   case Success(x) => Success(x+1)
     |   case Failure(_) => Success(0)
     | } transformWith {
     |   case Success(x) => MySimpleFuture.successful(x * 2)
     |   case Failure(_) => MySimpleFuture.successful(0)
     | } onComplete(println)
Success(4)
```

もちろんこれでは非同期処理にならないので、ここから標準ライブラリを追いかけてより実際の Future に近づけて行きます。

<div id="anchor2" />

### 2. ざっくり Future 実装

上では Future, Promise それぞれに独自のインタフェース、実装を与えてみたのですが、標準ライブラリを見ると実際はこの 2 つを実装した `scala.concurrent.impl.Promise` (上とは違い impl パッケージ下) が定義されていることがわかります。

```scala
package scala.concurrent.impl

private[concurrent] trait Promise[T]
  extends scala.concurrent.Promise[T] with scala.concurrent.Future[T] {
```

雰囲気としては 「非同期計算の結果が Promise として提供される。計算結果は Future のインタフェースを介して更なる計算に利用できる」 という感じでしょうか。確かに普段 Future を扱う際の感覚もこんな感じですね。

これを実装したクラスとして同ファイルには `DefaultPromise` があります。

```scala
class DefaultPromise[T] extends AtomicReference[AnyRef](Nil) with Promise[T] {
```

目を引くのは `AtomicReference[AnyRef](Nil)` を継承しているところです。[AtomicReference][2] は Java の標準ライブラリが持つクラスで、アトミックな更新が可能なオブジェクト参照です。Promise の意味合いを考えると、この AtomicReference には計算完了した値を保持するのかなと思い当たります。しかしそれにしては保持する値の型が T ではなく AnyRef なのがかなりざっくりしてるなという印象です。

その理由は DefaultPromise に付随する説明を読むと理解できます。この長文ではまず冒頭で DefaultPromise には取り得る状態が 3 つあると述べられています。

> A DefaultPromise has three possible states. It can be:
>   1. Incomplete, with an associated list of callbacks waiting on completion.
>   2. Complete, with a result.
>   3. Linked to another DefaultPromise.

そしてその状態がそれぞれ AtomicReference[AnyRef] の中身で表現されるようです。

> A DefaultPromise stores its state entirely in the AnyRef cell exposed by
> AtomicReference. The type of object stored in the cell fully describes the
> current state of the promise.
>
>   1. List[CallbackRunnable] - The promise is incomplete and has zero or more callbacks
>      to call when it is eventually completed.
>   2. Try[T] - The promise is complete and now contains its value.
>   3. DefaultPromise[T] - The promise is linked to another promise.

つまり DefaultPromise は与えられた計算が完了していなければその結果を待つコールバック一覧 `List[CallbackRunnable]` を保持し、完了すればその結果 `Try[T]` を保持するという挙動になるようです (実際は引用のように第 3 の状態があるのですが、これはメモリリーク対策として導入されているということなので雰囲気理解を目指すここではガン無視します)。

ここまでの内容を踏まえると、標準ライブラリの Future 実装のポイントは

- Future, Promise を継承し
- コールバック一覧または計算結果を保持する

の 2 点だと思われます。ということで実際にそのように `MyFuturePromise` を定義してみます。

```scala
package com.tiqwab.example.future

import scala.util.{Success, Try}

// 上で定義した MyFuture, MyPromise を実装するクラス
private[future] class MyFuturePromise[T] extends MyFuture[T] with MyPromise[T] {
  import MyFuturePromise._

  // (1) DefaultPromise では状態を表すために AtomicReference[Any] を継承して使用していた
  // ここでは属性として持つこと、型を別途定義したことが異なる点
  private val value: java.util.concurrent.atomic.AtomicReference[PromiseContent[T]] =
    new java.util.concurrent.atomic.AtomicReference(Listeners(Nil))

  // MyFuture を継承しているので自分を返せばよい
  override def future: MyFuture[T] = this

  // あとで実装する
  override def tryComplete(result: Try[T]): Boolean = ???
  override def transform[S](f: Try[T] => Try[S]): MyFuture[S] = ???
  override def transformWith[S](f: Try[T] => MyFuture[S]): MyFuture[S] = ???
  override def onComplete[U](f: Try[T] => U): Unit = ???
}

object MyFuturePromise {

  // (2) MyFuturePromise の状態を表すためのデータ型
  private sealed trait PromiseContent[T]
  private case class Result[T](value: Try[T]) extends PromiseContent[T]
  private case class Listeners[T, Any](list: Seq[RunnableWithValue[T, Any]]) extends PromiseContent[T]

  // (3) コールバックを表現するクラス
  // Try[T] => U な関数を与えて生成し、事前に引数を与えた上で実行する
  class RunnableWithValue[T, U](f: Try[T] => U) extends Runnable {
    val value: java.util.concurrent.atomic.AtomicReference[Option[Try[T]]] =
      new java.util.concurrent.atomic.AtomicReference(None)
    override def run(): Unit =
      value.get.fold(sys.error("value must be set"))(v => f(v))
  }

  // Future.successful に準ずるもの
  def successful[T](value: T): MyFuture[T] = {
    val f = new MyFuturePromise[T]()
    f.value.set(Result(Success(value)))
    f
  }

}
```

(1) MyFuturePromise (いま見ると名前がひどい) では標準ライブラリの DefaultPromise と同様に AtomicReference により状態の管理を行っています。DefaultPromise は AtomicReference を継承して利用していたのですが、継承する必然性も無い気がしたのでここでは単に属性にしています。

(2) DefaultPromise では状態を単に Any で扱っていたのですが、ここでは PromiseContent という型を定義し、取り得る状態を明示しました。きっとどこかで Any でないと扱えない何かが出てくるのだろうと思っていたのですが、少なくとも今回の範囲では特に問題は出てきませんでした。

(3) RunnableWithValue は計算完了後のコールバック処理を抽象化したようなクラスです。 `f: Try[T] => U` が計算結果を利用したコールバック処理になります。Runnable を継承しているので、これを `java.util.concurrent.ExecutorService` に渡すタスクとして利用することができます。RunnableWithValue を生成したタイミングで対象の計算が完了しているとは限らないので、計算結果はあとから設定できるようになっています。

MyFuturePromise の土台は整ったので、あとは残ったメソッドに実装を与えていきます。

#### tryComplete

```scala
  override def tryComplete(result: Try[T]): Boolean = value.get() match {
    case Result(_) =>
      false
    case l @ Listeners(list) =>
      if (!value.compareAndSet(l, Result(result))) tryComplete(result)
      list.foreach { l =>
        l.value.compareAndSet(None, Some(result))
        new java.lang.Thread(l).start()
      }
      true
  }
```

tryComplete は 「Promise に値を設定できれば true, 既に設定済みであれば false を返す」 ように実装します。MyFuturePromise の value が `Result(_)` ならば既に計算が完了しているということなので何もせず false を返せば OK です。 `Listeners(list)` ならばもらった計算結果で value を更新し、それを使用して保持していたコールバック一覧 `list` を実行します。コールバックの実行は単に新規スレッドを作成し実行させていますが、この部分はあとで標準ライブラリに近い形に整えます。

Future 実装の本筋ではありませんが、計算結果を設定する以下のコード

```scala
if (!value.compareAndSet(l, Result(result))) tryComplete(result)
```

での if 式は絶妙なタイミングでコールバックが登録された場合にそれが実行されないという事態を防ぐためのものです。 `AtomicReference#compareAndSet` は格納されている値が第一引数と等しい場合には第二引数の値を設定し true を返しますが、異なる場合には false を返し渡した値は設定しません。ここではパターンマッチから計算結果の設定までの間に他スレッドでコールバックを新規に追加する処理が走ると、そのコールバックが実行対象に入らなくなってしまう恐れがあると思います。それを防ぐために value が更新されていたら tryComplete をリトライするという処理にしています。このような処理は `java.util.concurrent` パッケージの Atomic ほにゃららクラスを利用する場合によく利用するパターンです。また (実装にバグが無ければ起こり得ないことだとは思うのですが) 二度結果が設定されることを防止するという点でもこのような処理にしておくほうが懸命かと思います。

#### transform

```scala
  override def transform[S](f: Try[T] => Try[S]): MyFuture[S] = {
    val promise = new MyFuturePromise[S]
    onComplete { result =>
      promise.tryComplete(f(result))
    }
    promise.future
  }
```

transform は 「計算結果 `Try[T]` を `Try[S]` に変換する関数を受け取り、`MyFuture[S]` を返す」 ように実装を与えます。概要としては現在の計算が完了したらその結果を関数 f に適用し、その結果を新規生成した MyFuturePromise を介して次の計算に回すという感じです。

#### transformWith

```scala
  override def transformWith[S](f: Try[T] => MyFuture[S]): MyFuture[S] = {
    val promise = new MyFuturePromise[S]
    onComplete { r1 =>
      f(r1).onComplete { r2 =>
        promise.tryComplete(r2)
      }
    }
    promise.future
  }
```

transformWith は型から考えると 「計算結果 `Try[T]` を `MyFuture[S]` に変換する関数を受け取り、`MyFuture[S]` を返す」 ように実装すれば OK です。

#### onComplete

```scala
  override def onComplete[U](f: Try[T] => U): Unit = {
    val newRunnable = new RunnableWithValue[T, Any](f)
    value.get() match {
      case Result(r) =>
        newRunnable.value.compareAndSet(None, Some(r))
        new java.lang.Thread(newRunnable).start()
      case l @ Listeners(list) =>
        if (!value.compareAndSet(l, Listeners(newRunnable +: list))) onComplete(f)
    }
  }
```

onComplete は計算完了時のコールバックを登録するためのメソッドになります。既に計算結果が得られているのならば渡された関数を別スレッドで実行するだけです。まだ計算中ならば自身が持つコールバック一覧 (`Listeners(list)`) に追加し、あとで tryComplete が呼ばれたときに実行できるようにしておきます。

これで MyFuturePromise の定義がほぼ終わりました。MyFuturePromise は性能やリソースの制約を考えると実用的なものではありませんが、標準ライブラリの Future を理解するのには役に立つのではないでしょうか。

<div id="anchor3" />

### 3. ExecutionContext 導入

`scala.concurrent` パッケージが提供する非同期処理フレームワークでは、「どんな処理を行うか」 ということと 「どのように処理を行うか」 ということがそれぞれ別クラスで定義されます。前者はこれまで見てきたように Future が表現するものであり、後者は ExecutionContext が担当するものです。このように責務を分離することでタスクの実行方法を柔軟にコントロールしやすくなります。MyFuturePromise もそれに倣い MyExecutionContext を使用するように修正します。

```scala
trait MyExecutionContext {
  def execute(runnable: Runnable): Unit
}

class MyDelegateGlobalExecutionContext() extends MyExecutionContext {
  override def execute(runnable: Runnable): Unit =
    scala.concurrent.ExecutionContext.Implicits.global.execute(runnable)
}
```

`MyExecutionContext#execute` で渡されたタスクを実行します。その実行方法は継承先のクラスの実装に任せられるので、例えば固定数のスレッドを持つプールを使用したり ForkJoinPool を使用したりといった実装が定義できます。ここでは `scala.concurrent.ExecutionContext.global` に実行を移譲しているので、ForkJoinPool を使用していることになります。

これに合わせて MyFuture のインタフェースや実装先のクラスも MyExecutionContext を利用するように修正します。

```scala
trait MyFuture[+T] {
  // 第二パラメータリストに MyExecutionContext を追加
  def onComplete[U](f: Try[T] => U)(implicit executor: MyExecutionContext): Unit
  def transform[S](f: Try[T] => Try[S])(implicit executor: MyExecutionContext): MyFuture[S]
  def transformWith[S](f: Try[T] => MyFuture[S])(implicit executor: MyExecutionContext): MyFuture[S]
}
```

```scala
private[future] class MyFuturePromise[T] extends MyFuture[T] with MyPromise[T] {
  ...
  // 新規スレッドを作るのではなく、RunnableWithValue を実行するという形式に修正
  override def tryComplete(result: Try[T]): Boolean = value.get() match {
    case Result(_) =>
      false
    case l @ Listeners(list) =>
      if (!value.compareAndSet(l, Result(result))) tryComplete(result)
      list.foreach { l =>
        l.value.compareAndSet(None, Some(result))
        l.execute()
      }
      true
  }

  // RunnableWithValue に第二パラメータリストの MyExecutionContrext を渡すように修正
  override def onComplete[U](f: Try[T] => U)(implicit executor: MyExecutionContext): Unit = {
    val newRunnable = new RunnableWithValue[T, Any](executor, f)
    value.get() match {
      case Result(r) =>
        newRunnable.value.compareAndSet(None, Some(r))
        newRunnable.execute()
      case l @ Listeners(list) =>
        if (!value.compareAndSet(l, Listeners(newRunnable +: list))) onComplete(f)
    }
  }
  ...
}

object MyFuturePromise {
  ...
  // コンストラクタに MyExecutionContext を追加
  // コンストラクタで渡された MyExecutionContext を使用して処理を行う execute メソッドを追加
  class RunnableWithValue[T, U](executor: MyExecutionContext, f: Try[T] => U) extends Runnable {
    val value: java.util.concurrent.atomic.AtomicReference[Option[Try[T]]] =
      new java.util.concurrent.atomic.AtomicReference(None)
    override def run(): Unit =
      value.get.fold(sys.error("value must be set"))(v => f(v))
    def execute(): Unit = executor.execute(this)
  }
  ...
}
```

最後に利便性から MyFuture に `map`, `flatMap` を定義します。

```scala
  def map[U](f: T => U)(implicit executor: MyExecutionContext): MyFuture[U] = transform {
    case Success(x) =>
      Success(f(x))
    case Failure(e) =>
      Failure(e)
  }

  def flatMap[U](f: T => MyFuture[U])(implicit executor: MyExecutionContext): MyFuture[U] = transformWith {
    case Success(x) =>
      f(x)
    case Failure(_) =>
      this.asInstanceOf[MyFuture[U]]
  }
```

これで一通り必要なものは揃いました。実際の使用例はこんな感じです。Future 処理内で `Thread.sleep` するのは本来良くない例ですが、ここでは動作の確認のために入れています。

```
scala> import com.tiqwab.example.future._
scala> implicit val ec: MyExecutionContext = new MySimpleExecutionContext()
scala> MyFuturePromise.successful(1) flatMap { x =>
     |   Thread.sleep(1000L); MyFuturePromise.successful(2) flatMap { y =>
     |     Thread.sleep(1000L); MyFuturePromise.successful(3) map { z =>
     |       x + y + z
     |     }
     |   }
     | } onComplete(println)

(About 2 seconds later...)
Success(6)
```

<div id="anchor4" />

### 4. MyFuturePromise 実行時の挙動理解

せっかく独自に Future っぽいものを定義してみたので、これを使用してその実行の挙動を整理してみたいと思います。MyFuturePromise は標準ライブラリの Future を模倣したものなので、ここでの理解は Future にも適用できるはず (はず...) です。

ここでの目標は上で試した

```scala
MyFuturePromise.successful(1) flatMap { x =>
  Thread.sleep(1000L); MyFuturePromise.successful(2) flatMap { y =>
    Thread.sleep(1000L); MyFuturePromise.successful(3) map { z =>
      x + y + z
    }
  }
}
```

という処理の流れを詳細に追うこととします。

まず細かい処理を追いやすくするために各 MyFuturePromise や RunnableWithValue に名前を付けられるようにします。

```scala
// コンストラクタに name を追加
private[future] class MyFuturePromise[T](val name: String)
  extends MyFuture[T] with MyPromise[T] with LazyLogging {
```

```scala
// コンストラクタに name を追加
class RunnableWithValue[T, U](executor: MyExecutionContext,
                              f: Try[T] => U,
                              name: String) {
```

MyFuture が持つメソッドにも name を渡せるようにします。

```scala
trait MyFuture[+T] {
  def onComplete[U](f: Try[T] => U, name: String)(implicit executor: MyExecutionContext): Unit
  def transform[S](f: Try[T] => Try[S], name: String)(implicit executor: MyExecutionContext): MyFuture[S]
  def transformWith[S](f: Try[T] => MyFuture[S], name: String)(implicit executor: MyExecutionContext): MyFuture[S]
  def map[U](f: T => U, name: String)(implicit executor: MyExecutionContext): MyFuture[U] = ...
  def flatMap[U](f: T => MyFuture[U], name: String)(implicit executor: MyExecutionContext): MyFuture[U] = ...
}
```

各メソッド内部では生成される MyFuturePromise にこの name を与えています。また RunnableWithValue には `from-<コールバック登録先の MyFuturePromise の name>` という名前が振られるようにしました (後述の変更後コードを参照)。

こうした変更を受け実行コードは以下のように変更されます。

```scala
MyFuturePromise.successful(1, "one") flatMap({ x =>
   Thread.sleep(1000L); MyFuturePromise.successful(2, "two") flatMap({ y =>
     Thread.sleep(1000L); MyFuturePromise.successful(3, "three") map({ z =>
       x + y + z
     }, "four")
   }, "five")
 }, "six")
```

はじめに one という MyFuturePromise が生成され、全体としては six という MyFuturePromise を受け取るコードになった、ということです。

また処理を追いやすくするために MyFuturePromise 生成時、tryComplete 呼出時、RunnableWithValue の処理開始時、処理終了時にそれぞれログを吐くようにしました。

変更後の MyFuturePromise, RunnableWithValue の全体は以下のようになりました。

```scala
import com.typesafe.scalalogging.LazyLogging
import scala.util.{Success, Try}

// コンストラクタに name 追加
private[future] class MyFuturePromise[T](val name: String) extends MyFuture[T] with MyPromise[T] with LazyLogging {
  import MyFuturePromise._

  // MyFuturePromise 生成時のログ
  logger.debug(s"created: $this")

  private val value: java.util.concurrent.atomic.AtomicReference[PromiseContent[T]] =
    new java.util.concurrent.atomic.AtomicReference(Listeners(Nil))

  override def future: MyFuture[T] = this

  override def tryComplete(result: Try[T]): Boolean = {
    // 計算結果セット時のログ
    logger.debug(s"completed with $result: $this")
    value.get() match {
      case Result(_) =>
        false
      case l @ Listeners(list) =>
        if (!value.compareAndSet(l, Result(result))) tryComplete(result)
        list.foreach { l =>
          l.value.compareAndSet(None, Some(result))
          l.execute()
        }
        true
    }
  }

  // コンストラクタに name 追加
  override def transform[S](f: Try[T] => Try[S], name: String)(implicit executor: MyExecutionContext): MyFuture[S] = {
    // 渡された name を MyFuturePromise に与える
    val promise = new MyFuturePromise[S](name)
    onComplete({ result =>
      promise.tryComplete(f(result))
    }, s"from-${this.name}")
    promise.future
  }

  // コンストラクタに name 追加
  override def transformWith[S](f: Try[T] => MyFuture[S], name: String)(
    implicit executor: MyExecutionContext): MyFuture[S] = {
    // 渡された name を MyFuturePromise に与える
    val promise = new MyFuturePromise[S](name)
    onComplete({ r1 =>
      val fut = f(r1).asInstanceOf[MyFuturePromise[S]]
      fut.onComplete({ r2 =>
        promise.tryComplete(r2)
      }, s"from-${fut.name}")
    }, s"from-${this.name}")
    promise.future
  }

  // コンストラクタに name 追加
  override def onComplete[U](f: Try[T] => U, name: String)(implicit executor: MyExecutionContext): Unit = {
    // 渡された name を RunnableWithValue に与える
    val newRunnable = new RunnableWithValue[T, Any](executor, f, name)
    value.get() match {
      case Result(r) =>
        newRunnable.value.compareAndSet(None, Some(r))
        newRunnable.execute()
      case l @ Listeners(list) =>
        if (!value.compareAndSet(l, Listeners(newRunnable +: list))) onComplete(f, name)
    }
  }

  override def toString: String = s"MyFuturePromise(name=$name, value=$value)"

}

object MyFuturePromise {

  private sealed trait PromiseContent[T]
  private case class Result[T](value: Try[T]) extends PromiseContent[T]
  private case class Listeners[T, Any](list: Seq[RunnableWithValue[T, Any]]) extends PromiseContent[T]

  // コンストラクタに name を追加
  class RunnableWithValue[T, U](executor: MyExecutionContext, f: Try[T] => U, name: String)
    extends Runnable with LazyLogging {
    val value: java.util.concurrent.atomic.AtomicReference[Option[Try[T]]] =
      new java.util.concurrent.atomic.AtomicReference(None)
    override def run(): Unit = {
      // 処理開始時にログ
      logger.debug(s"start $this")
      value.get.fold(sys.error("value must be set"))(v => f(v))
      // 処理終了時にログ
      logger.debug(s"finished $this")
    }
    def execute(): Unit = executor.execute(this)

    override def toString: String = s"RunnableWithValue(name=$name)"
  }

  def successful[T](value: T, name: String): MyFuture[T] = {
    val f = new MyFuturePromise[T](name)
    f.value.set(Result(Success(value)))
    f
  }

}
```

これを使用して先程のコードを実行すると以下のようなログが出力されます。

```
2018-02-26 23:12:39,475 DEBUG [run-main-0] c.t.e.f.MyFuturePromise created: MyFuturePromise(name=one, value=null)
2018-02-26 23:12:39,482 DEBUG [run-main-0] c.t.e.f.MyFuturePromise created: MyFuturePromise(name=six, value=null)
2018-02-26 23:12:39,501 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise$RunnableWithValue start RunnableWithValue(name=from-one)
2018-02-26 23:12:40,503 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise created: MyFuturePromise(name=two, value=null)
2018-02-26 23:12:40,506 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise created: MyFuturePromise(name=five, value=null)
2018-02-26 23:12:40,507 DEBUG [scala-execution-context-global-51] c.t.e.f.MyFuturePromise$RunnableWithValue start RunnableWithValue(name=from-two)
2018-02-26 23:12:40,507 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise$RunnableWithValue finished RunnableWithValue(name=from-one)
2018-02-26 23:12:41,507 DEBUG [scala-execution-context-global-51] c.t.e.f.MyFuturePromise created: MyFuturePromise(name=three, value=null)
2018-02-26 23:12:41,514 DEBUG [scala-execution-context-global-51] c.t.e.f.MyFuturePromise created: MyFuturePromise(name=four, value=null)
2018-02-26 23:12:41,515 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise$RunnableWithValue start RunnableWithValue(name=from-three)
2018-02-26 23:12:41,515 DEBUG [scala-execution-context-global-51] c.t.e.f.MyFuturePromise$RunnableWithValue finished RunnableWithValue(name=from-two)
2018-02-26 23:12:41,515 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise completed with Success(6): MyFuturePromise(name=four, value=Listeners(List(RunnableWithValue(name=from-four))))
2018-02-26 23:12:41,516 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise$RunnableWithValue finished RunnableWithValue(name=from-three)
2018-02-26 23:12:41,516 DEBUG [scala-execution-context-global-51] c.t.e.f.MyFuturePromise$RunnableWithValue start RunnableWithValue(name=from-four)
2018-02-26 23:12:41,516 DEBUG [scala-execution-context-global-51] c.t.e.f.MyFuturePromise completed with Success(6): MyFuturePromise(name=five, value=Listeners(List(RunnableWithValue(name=from-five))))
2018-02-26 23:12:41,516 DEBUG [scala-execution-context-global-51] c.t.e.f.MyFuturePromise$RunnableWithValue finished RunnableWithValue(name=from-four)
2018-02-26 23:12:41,516 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise$RunnableWithValue start RunnableWithValue(name=from-five)
2018-02-26 23:12:41,516 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise completed with Success(6): MyFuturePromise(name=six, value=Listeners(List()))
2018-02-26 23:12:41,517 DEBUG [scala-execution-context-global-50] c.t.e.f.MyFuturePromise$RunnableWithValue finished RunnableWithValue(name=from-five)
```

このログが MyFuturePromise の実行時の挙動を理解する手助けになるはずですが、とはいえこれをちまちま追うのも辛いので、ログを元に処理の流れを表す図を作成してみました。

<img
  src="/images/understanding-future/future1.png"
  title="process_future"
  alt="abstract of processing Future"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

図では灰色のオブジェクトが Future を表しています。Future の中の丸が計算結果を表し、実践で表されているものは計算が完了し結果が得られている状態、破線がまだ結果が得られていない状態を表します。

callback と書かれた長方形は onComplete で登録されたコールバックであり、計算結果が得られた際に呼ばれる関数を表現しています。callback から伸びている吹き出しや矢印が実際に callback で行う処理を表現しているつもりです。色分けしているのはそれぞれが異なる callback であることを示すためです。

今回のコードではまず one という MyFuturePromise が作成されるところからスタートします。one は `MyFuturePromise.successful` により生成されるので既に計算結果 `1` が得られた状態です。one に対し flatMap を呼び出すとまず以下の 2 つの処理が行われます。

- six という MyFuturePromise の生成
- one へのコールバック登録 (図では黄色)

この時点で手元には処理全体の (未来の) 計算結果を表す six が手に入ります。これ以降見ていく処理は MyExecutionContext が管理するスレッド上で非同期的に実行される処理なので、メインスレッドは six を手に入れた時点でこれらの計算結果を待たずに処理を続けることができます。

one は既に結果を持っているので、渡された callback(黄) はすぐに実行されます。この callback(黄) がどういった処理を行うかは flatMap 内部で呼ばれる transformWith 関数の以下の部分から理解できます。

```scala
    onComplete({ r1 =>
      val fut = f(r1).asInstanceOf[MyFuturePromise[S]]
      fut.onComplete({ r2 =>
        promise.tryComplete(r2)
      }, s"from-${fut.name}")
    }, s"from-${this.name}")
```

この callback 登録時、関数 f としては

```scala
{ x =>
   Thread.sleep(1000L); MyFuturePromise.successful(2, "two") flatMap({ y =>
     Thread.sleep(1000L); MyFuturePromise.successful(3, "three") map({ z =>
       x + y + z
     }, "four")
   }, "five")
}
```

が渡されています。いま実行時には r1 として one の計算結果である 1 が渡されます。callback(黄) はまず 1000 ミリ秒 sleep し、その後 two という MyFuturePromise の生成、それに対する flatMap の呼出が続きます。flatMap では one のときと同様に

- five という MyFuturePromise の生成
- two へのコールバック登録 (図では 緑色)

が行われます。

このあと two はすでに計算結果が得られているので callback(緑) の処理が開始されるのですが、同時にこの時点ではじめの callback(黄) での `fut` が得られます。コードでいうと transformWith 内

```scala
  override def transformWith[S](f: Try[T] => MyFuture[S], name: String)(
    implicit executor: MyExecutionContext): MyFuture[S] = {
    val promise = new MyFuturePromise[S](name)
    onComplete({ r1 =>
      val fut = f(r1).asInstanceOf[MyFuturePromise[S]] // ココ
      fut.onComplete({ r2 =>
        promise.tryComplete(r2)
      }, s"from-${fut.name}")
    }, s"from-${this.name}")
    promise.future
  }
```

の部分ですね。いま `fut` には flatMap の返り値 five が入っています。次に five には `promise` に自身の計算結果を tryComplete するという callback(赤) が登録されます。これが callback(黄) の最後の処理になり、callback(黄) の処理はこの時点で完了します。

ちょっと長くなりそうなのと似たような処理が続くので省略しますが、このあと callback(緑) の処理が進み、three への callback(青) の登録、MyFuturePromise four の生成、four への callback(紫) の登録、callback(青) の実行というように処理が進みます。

callback(青) では計算 `1 + 2 + 3` が実行されます。これが処理全体としての計算結果であり、six から得られることが期待される値のはずです。ここからはそれがどのように six まで伝えられるのか、という部分になります。

callback(青) では計算結果 6 を持って four の tryComplete を呼び出します。それにより four の計算が完了したことになり、登録された callback(紫) が実行されます。この callback(紫) は four の計算結果で five の tryComplete を呼び出す処理です。これにより five にも計算結果 6 が設定され、callback(赤) が実行されます。この callback(赤) により six の tryComplete が実行され、晴れて six から計算結果 6 が得られるようになります。このように six への計算結果の伝搬は tryComplete の連鎖という形になるようです。

[1]: https://dwango.github.io/scala_text/future-and-promise.html
[2]: https://docs.oracle.com/javase/jp/8/docs/api/java/util/concurrent/atomic/AtomicReference.html
