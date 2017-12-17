---
layout: post
title: retry ライブラリ模写
tags: "replication, scala"
comments: true
---

### retry とは

- [softprops/retry][1]
- 任意の `Future` を返す関数にリトライ処理を実装するためのライブラリ
- 外部システムにアクセスできなかったときに 5 秒間隔で 3 回までリトライをかける... といった処理が簡単に書ける

### 模写とは

- 既存のライブラリのインタフェースやドキュメントを見て、自分で同様の機能を実装する
- 絵を書く人が練習、上達のために模写をするならプログラミングでも効果が期待できるのでは？

### retry 使用例

- 模写開始前に retry の機能を整理
- retry の使用例は以下のようになる
  - 模写では以下と同様のことが行えるものを作成することを目標に

```scala
import retry.Success

import scala.concurrent.{Await, ExecutionContext, Future}
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits._

object RetrySample {

  def main(args: Array[String]): Unit = {
    // (1) Define what is assumed to be the success
    implicit val success = retry.Success.always
    println("pause:")
    pauseSample()
    println("backoff:")
    backoffSample()
  }

  // Sample of retry.Pause
  def pauseSample()(implicit success: Success[String], ec: ExecutionContext): Unit = {
    // (2) Define `Pause` retry policy
    // This policy retries three times with two seconds interval (2, 4, 6 seconds later from the first execution)
    val policy: retry.Policy = retry.Pause(max = 3, delay = 2.seconds)
    // Create Future which succeeds at the third attempt
    var count = 0
    val f: Future[String] = policy(() => {
      if (count % 3 == 2) {
        Future.successful("success")
      } else {
        count = count + 1
        Future.failed(new RuntimeException("fail"))
      }
    })
    Await.result(f, Duration.Inf) // "success"
  }

  // Sample of retry.Backoff
  def backoffSample()(implicit success: Success[String], ec: ExecutionContext): Unit = {
    // (3) Define `Backoff` retry policy
    // This policy retries three times with backoff (1, 2, 4 seconds later from the first execution)
    val policy: retry.Policy = retry.Backoff(max = 3, delay = 1.second, base = 2)
    // Create Future which succeeds at the third attempt
    var count = 0
    val f: Future[String] = policy(() => {
      // success at the third call
      if (count % 3 == 2) {
        Future.successful("success")
      } else {
        count = count + 1
        Future.failed(new RuntimeException("fail"))
      }
    })
    Await.result(f, Duration.Inf) // "success"
  }

}
```

- 順番が前後するけれどメインは (2), (3)
  - `retry.Policy` でリトライの方針を定義する (最大リトライ回数、リトライ間隔等)
  - リトライを行いたい処理は作成した `retry.Policy` の apply に渡してあげる
- (1) では何を持って処理が成功とみなすかを定義している
  - `retry.Success.always` は Future が failed していなければ成功とみなす
  - 他に例えば `retry.Success.option` なら `Some(x)` が返る場合のみ成功とみなす

### 模写ステップ

1. Define Policy
2. Define Success

### Define Policy

- `Policy` は上に見たようにリトライのやり方を定義する
- リトライのかけかた自体は共通に実装できると思う (`apply`)
- `Policy` 継承先ではリトライ間隔を決定する `calcDuration` を実装してもらう
- 次のリトライまでの待ちには `Timer` を使用する
  - retry ライブラリと同じ `odelay.Timer` を流用している
  - デフォルトでは `java.util.concurrent.ScheduledThreadPoolExecutor` を使用

```scala
package com.tiqwab.replicate.retry

import scala.concurrent.duration.FiniteDuration
import scala.concurrent.{ExecutionContext, Future}

trait Policy {
  // Use timer to implement delay
  def timer: Timer
  def maxRetryCount: Int
  def initialDelay: FiniteDuration
  // Calculate duration of the delay until next trial
  def calcDuration(currentDuration: FiniteDuration): FiniteDuration
  // Execute f with retry
  def apply[T](f: () => Future[T])(implicit success: Success[T], ec: ExecutionContext): Future[T] = {
    def loop(g: () => Future[T], remains: Int, delayDuration: FiniteDuration): Future[T] =
      g()
        .flatMap {
          case x if success.predicate(x) =>
            Future.successful(x)
          case x if remains <= 0 =>
            Future.successful(x)
          case _ =>
            timer.delay(delayDuration).flatMap { _ =>
              val nextDuration = calcDuration(delayDuration)
              loop(g, remains - 1, nextDuration)
            }
        }
        .recoverWith {
          case NonFatal(_) if remains > 0 =>
            timer.delay(delayDuration).flatMap { _ =>
              val nextDuration = calcDuration(delayDuration)
              loop(g, remains - 1, nextDuration)
            }
        }
    loop(f, maxRetryCount - 1, initialDelay)
  }

  require(maxRetryCount > 0)

}
```

```scala
case class Pause(maxRetryCount: Int, initialDelay: FiniteDuration) extends Policy {
  override def timer: Timer = DefaultTimer
  override def calcDuration(currentDuration: FiniteDuration): FiniteDuration = currentDuration
}

case class Backoff(maxRetryCount: Int, initialDelay: FiniteDuration, base: Long = 2L) extends Policy {
  override def timer: Timer = DefaultTimer
  override def calcDuration(currentDuration: FiniteDuration): FiniteDuration = currentDuration * base
}
```

### Define Success

- 何をもってリトライ完了かを判断するために `Success` を定義する
- 本家ライブラリと同様に `Success` 同士を組み合わせる `and`, `or` も定義した
- はじめ `Success` を invariant で定義していたけど、それだと `Success[Any]` 型を持ち何にでも適用できるはずの `always` が使用時に型チェックに引っかかった
  - `Success[String]` が欲しいところに `Success[Any]` を渡してあげたいということになるので `Success` の型パラメータを contravariant にした
- `Option[_]` とか書いてみたけど型パラメータのアンダースコアって何を意味するんだっけと
  - [Any vs underscore in generics][2]
  - 存在型とは
    - [存在型を学ぼう][3]
  - ここでは `Option` としての振る舞いのみに興味があり、その型パラメータは特に気にしていないのでアンダースコアでそれを表現する
  - ただ covariant な `Option` に対しては `Option[Any]` と同じことになるみたい

```scala
package com.tiqwab.replicate.retry

trait Success[-T] {
  def predicate(v: T): Boolean
  def and[TT <: T](success: Success[TT]): Success[TT] =
    (x: TT) => predicate(x) && success.predicate(x)
  def or[TT <: T](success: Success[TT]): Success[TT] =
    (x: TT) => predicate(x) || success.predicate(x)
}

object Success {
  val always: Success[Any] = (_: Any) => true
  val option: Success[Option[_]] = (x: Option[_]) => x.isDefined
}
```

### 使用例

- 上で定義した `Pause` と `Success.always` を使用すると本家ライブラリと同様にしてリトライ処理がかけるようになりました

```scala
package com.tiqwab.replicate.retry

import org.scalatest.FunSuite

import scala.concurrent.{Await, Future}
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits._

class PauseTest extends FunSuite {
  test("retry with Pause") {
    var count = 3
    val pause = Pause(3, 2.seconds)
    implicit val success = Success.always
    val f = () =>
      Future {
        count = count - 1
        if (count <= 0) {
          "OK"
        } else {
          throw new RuntimeException("fail")
        }
    }

    val startMillis = System.currentTimeMillis
    val result = Await.result(pause(f), Duration.Inf)
    val endMillis = System.currentTimeMillis
    assert(count === 0)
    assert(result === "OK")
    assert(endMillis - startMillis > 4000)
  }
}
```

### 本家 retry との比較とか

- 本家では `Policy` をより抽象的に定義している
  - 自分の場合 `Policy` にすでにリトライ回数や間隔の概念を入れていた
  - 本家の場合リトライ処理の流れのみを定義し、リトライの実装については一切触れていない
  - これにより状況によってリトライ方針を換えるような `When` の実装が見やすくなっている

本家 `Policy` コード:

```scala
trait Policy {
  def apply[T](promise: () => Future[T])
    (implicit success: Success[T],
     executor: ExecutionContext): Future[T]

  protected def retry[T](
    promise: () => Future[T],
    orElse: () => Future[T],
    recovery: Future[T] => Future[T] = identity(_: Future[T]))
    (implicit success: Success[T],
     executor: ExecutionContext): Future[T] = {
      val fut = promise()
      fut.flatMap { res =>
        if (success.predicate(res)) fut
        else orElse()
      }.recoverWith {
        case NonFatal(_) => recovery(fut)
      }
    }
}
```

本家 `When` ポリシーの使用例:

```scala
val policy = retry.When {
  case RetryAfter(retryAt) => retry.Pause(delay = retryAt)
}
val future = policy(issueRequest)
```

- リトライに対する回数の概念は `Policy` を継承した `CountingPolicy` で加えられる

本家 `CountingPolicy` コード:

```scala
trait CountingPolicy extends Policy {
  protected def countdown[T](
    max: Int,
    promise: () => Future[T],
    orElse: Int => Future[T])
    (implicit success: Success[T],
     executor: ExecutionContext): Future[T] = {
      // consider this successful if our predicate says so _or_
      // we've reached the end out our countdown
      val countedSuccess = success.or(max < 1)
      retry(promise, () => orElse(max - 1), { f: Future[T] =>
        if (max < 1) f else orElse(max - 1)
      })(countedSuccess, executor)
    }
}
```

- `Pause` は上記 `CountingPolicy` を生成するファクトリになっている

本家 `Pause` コード抜粋:

```scala
object Pause {
  ...
  /** Retry with a pause between attempts for a max number of times */
  def apply(max: Int = 4, delay: FiniteDuration = Defaults.delay)
   (implicit timer: Timer): Policy =
    new CountingPolicy {
      def apply[T]
        (promise: () => Future[T])
        (implicit success: Success[T],
         executor: ExecutionContext): Future[T] = {
          def run(max: Int): Future[T] = countdown(
            max, promise,
            c => Delay(delay)(run(c)).future.flatMap(identity))
          run(max)
        }
    }
}
```

### 作成したコード

- [replicate-retry][4]

[1]: https://github.com/softprops/retry
[2]: https://stackoverflow.com/questions/15186520/scala-any-vs-underscore-in-generics
[3]: http://slides.com/lyrical_logical/rpscala164/#/
[4]: https://github.com/tiqwab/example/tree/master/replicate-retry
