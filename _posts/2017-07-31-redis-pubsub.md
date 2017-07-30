---
layout: post
title: Redis pub/sub と Akka Stream
tags: "redis, scala"
comments: true
---

アプリケーション作成時 Redis をキャッシュに使用することは多いと思いますが、Redis には Publish/Subscribe メッセージングシステムを構築するのに便利な機能も存在します。

ここでは Redis の Java クライアントライブラリ [Jedis][4] と ストリーミング処理に用いられる [Akka Stream][5] を使用して、 Redis の pub/sub のサンプルを作成してみます。

サンプルの全体は [こちら][6] のリポジトリにあげています。 

1. [Jedis 基本](#anchor1)
2. [Jedis pub/sub](#anchor2)
3. [akka stream 基本](#anchor3)
4. [Redis pub/sub と Akka stream 連携](#anchor4)

<div id="anchor1" />

### 1. Jedis 基本

今回は Scala でサンプルを作成するので、`build.sbt` に以下の依存ライブラリの情報を加えます。

```scala
lazy val jedisClient = "redis.clients" % "jedis" % "2.9.0"
```

Jedis を使用した簡単な処理は以下のようになります。

```scala
import redis.clients.jedis.Jedis

object SampleJedis {

  def main(args: Array[String]): Unit = {
    withRedis { jedis =>
      println(jedis.set("test1", "001"))
      println(jedis.get("test1"))
    }
  }

  def withRedis[T](f: Jedis => T): T = {
    val jedis = new Jedis("localhost")
    try {
      f(jedis)
    } finally {
      jedis.close()
    }
  }

}
```

標準出力

```
OK
001
```

<div id="anchor2" />

### 2. Jedis pub/sub

Jedis を使用して Redis の pub/sub の簡単なサンプルを作成します。

Publisher はランダムな整数値を生成し、 Redis の 'channel1' チャネルに publish します。
Subscriber は 'channel1' チャネルを subscribe し、メッセージを受信するとログに吐き出しています。
Subscriber は 3 つメッセージを受信した時点で処理を終了します。

Jedis では Subscriber の実装のために `JedisPubSub` という抽象クラスが用意されています。
`JedisPubSub` には subscribe の開始と終了、メッセージ受信時等のコールバックを実装します。

```scala
object SampleJedisPubSub extends App {
  val host = "localhost"
  val channel = "channel1"

  // Create subscriber which accepts only 3 messages
  val latch = new CountDownLatch(3)
  val subscriber = new Subscriber(host, channel, latch)

  new Thread(() => subscriber.start()).start()

  // Publish 3 messages with randomized wait time
  new Thread(() => {
    val jedis = new Jedis("localhost")
    List.fill(3)((scala.math.random() * 10000).toLong) foreach {
      waitTimeMillis =>
        Thread.sleep(waitTimeMillis)
        jedis.publish(channel, waitTimeMillis.toString)
    }
  }).start()

  latch.await()
  subscriber.unsubscribe()
}

class Subscriber(val host: String,
                 val channel: String,
                 val latch: CountDownLatch)
    extends JedisPubSub {

  // Assign a new connection since client should not issue commands other than 'subscribe' and 'unsubscribe'
  val jedis = new Jedis(host)
  val logger = LoggerFactory.getLogger(this.getClass)

  def start(): Unit = {
    jedis.subscribe(this, channel)
  }

  override def onUnsubscribe(channel: String, subscribedChannels: Int): Unit = {
    logger.debug(s"unsubscribe $channel")
    jedis.quit()
  }

  override def onSubscribe(channel: String, subscribedChannels: Int): Unit =
    logger.debug(s"subscribe $channel")

  override def onPUnsubscribe(pattern: String, subscribedChannels: Int): Unit =
    logger.debug("onPUnsubscribe")

  override def onPSubscribe(pattern: String, subscribedChannels: Int): Unit =
    logger.debug("onPSubscribe")

  override def onPMessage(pattern: String,
                          channel: String,
                          message: String): Unit =
    logger.debug("onPMessage")

  override def onMessage(channel: String, message: String): Unit = {
    logger.debug(s"get message: $message")
    // Countdown when processes a message
    latch.countDown()
  }

}
```

標準出力

```
14:52:49.126 [Thread-0] DEBUG com.tiqwab.example.messaging.Subscriber - subscribe channel1
14:52:49.483 [Thread-0] DEBUG com.tiqwab.example.messaging.Subscriber - get message: 534
14:52:53.775 [Thread-0] DEBUG com.tiqwab.example.messaging.Subscriber - get message: 4291
14:52:55.830 [Thread-0] DEBUG com.tiqwab.example.messaging.Subscriber - get message: 2053
14:52:55.831 [Thread-0] DEBUG com.tiqwab.example.messaging.Subscriber - unsubscribe channel1
```

参考:

- [redis Pub/Sub/][1]
- [The Redis Cookbook][2]
- [Example usage of Jedis][3]

<div id="anchor3" />

### 3. Akka Stream 基本

Akka Stream はストリーミング処理を実装するために用いられる Java, Scala のライブラリです。

ここでは詳細を省きますが、Stream の始まりとしての `Source`、終わりとしての `Sink` その間の `Flow` といった概念が存在します。

```scala
import scala.concurrent.ExecutionContext.Implicits._

import akka.stream._
import akka.stream.scaladsl._

import akka.actor.ActorSystem

object SampleStream {

  def main(args: Array[String]): Unit = {
    implicit val system = ActorSystem("sample-stream")
    implicit val materializer = ActorMaterializer()

    Source(List(1, 2, 3, 4, 5))
      .map(_ * 2)
      .runWith(Sink.foreach(println(_)))
      .onComplete(_ => system.terminate())
  }
}
```

標準出力

```
2
4
6
8
10
```

参考

- [Akka Stream][5]

<div id="anchor4" />

### 4. Redis pub/sub と Akka Stream 連携

Redis で実装する pub/sub と Akka Sream によるストリーミング処理を連携させるサンプルを作成します。

処理としては

```
定期的にメッセージを生成
↓
PublisherSink が Redis の指定のチャネルへ publish
↓
SubscriberSource が Redis の指定のチャネルを subscribe しメッセージ受信
↓
メッセージ出力
```

というようにしています。

この全体的な流れを実装しているのが 以下の `RedisPubSubStream` です。

```scala
import java.time.ZonedDateTime
import org.slf4j.LoggerFactory

import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits._

import akka.actor.ActorSystem
import akka.stream._
import akka.stream.scaladsl._

object RedisPubSubStream {

  val logger = LoggerFactory.getLogger(this.getClass())

  def main(args: Array[String]): Unit = {
    implicit val system = ActorSystem("pubsub")
    implicit val materializer = ActorMaterializer()

    val channelName = "channel1"

    val publisherSink = new PublisherSink(channelName)
    Source
      .tick(1.seconds, 5.seconds, Message(s"Hello, "))
      .map(m => {
        m.copy(data = m.data + s"this is ${ZonedDateTime.now()}")
      })
      .runWith(Sink.fromGraph(publisherSink))

    val subscriberSource = new SubscriberSource(channelName)
    Source
      .fromGraph(subscriberSource)
      .runWith(Sink.foreach(x => logger.debug(x.toString)))
      .onComplete(_ => system.terminate())
  }

}
```

ストリームを流れてきたメッセージを Redis に publish する `PublisherSink` は以下のようになります。

Akka Stream の [ドキュメント][7] にあるように `GraphStage[SinkShape[T]]` を実装することで独自の Sink を作成することができます。

`PublisherSink` では Redis に直接やり取りことはなく後に登場する `RedisClient` にその役目を任せています。

```scala
import akka.stream._
import akka.stream.stage.{GraphStage, GraphStageLogic, InHandler}
import org.slf4j.LoggerFactory

class PublisherSink(
    val channelName: String
) extends GraphStage[SinkShape[Message]] {

  val logger = LoggerFactory.getLogger(this.getClass)
  val in: Inlet[Message] = Inlet("SubscribeSink.In")
  val channel = Channel(channelName)

  override def shape: SinkShape[Message] = SinkShape(in)

  override def createLogic(inheritedAttributes: Attributes): GraphStageLogic =
    new GraphStageLogic(shape) {
      override def preStart(): Unit = {
        pull(in)
      }

      setHandler(
        in,
        new InHandler {
          override def onPush(): Unit = {
            val message = grab(in)
            logger.debug(s"will publish message: $message")
            RedisClient.publish(message, channel)
            pull(in)
          }
        }
      )
    }

}
```

`SubscriberSource` は `GraphStage[SourceShape[T]]` を実装することで Redis からの subscribe を行っています。

こちらも Redis とのやり取りは `RedisClient` を介して行っています。

少し実装として苦しいのは Redis からの subscribe を待つ部分で、 `OutHandler#onPull` との兼ね合いをもう少し上手く処理したいところです。

```scala
import java.util.concurrent.CyclicBarrier

import akka.stream._
import akka.stream.scaladsl._
import akka.stream.stage.{GraphStage, GraphStageLogic, OutHandler}
import org.slf4j.LoggerFactory

class SubscriberSource(
    val channelName: String
) extends GraphStage[SourceShape[Message]] {

  val logger = LoggerFactory.getLogger(this.getClass())
  val out: Outlet[Message] = Outlet("SubscriberSource.Out")
  val channel = Channel(channelName)

  override def shape: SourceShape[Message] = new SourceShape(out)

  override def createLogic(inheritedAttributes: Attributes): GraphStageLogic =
    new GraphStageLogic(shape) {

      private val barrier = new CyclicBarrier(2)
      private var message: Option[Message] = None
      private var threadName: Option[String] = None

      private val messageReceiver = new MessageReceiver {
        override def handleMessage(m: Message): Unit = {
          message = Some(m.copy())
          barrier.await()
        }
      }

      override def preStart(): Unit = {
        super.preStart()
        threadName = Some(RedisClient.subscribe(channel, messageReceiver))
      }

      override def postStop(): Unit = {
        super.postStop()
        RedisClient.unsubscribe(threadName.get)
      }

      setHandler(
        out,
        new OutHandler() {
          override def onPull(): Unit = {
            barrier.await()
            logger.debug(s"subscribed message: ${message.get}")
            push(out, message.get)
            message = None
            barrier.reset()
          }
        }
      )
    }

}
```

最後に `RedisClient` の実装です。
このオブジェクトが Redis と直接やり取りする機能を負っています。

Subscribe は Jedis コネクションを専有しスレッドをブロックする関係上、要求を受けるたびに専用のコネクション、スレッドを用意する必要があると思います。
そのためこの実装は subscriber が限定された人数だという条件の下でないと機能しません。

```scala
import org.slf4j.LoggerFactory
import redis.clients.jedis.{Jedis, JedisPool, JedisPoolConfig, JedisPubSub}

object RedisClient {

  private val logger = LoggerFactory.getLogger(this.getClass())
  private val host = "localhost"
  private val pool = new JedisPool(new JedisPoolConfig(), host)

  private val subscribers =
    scala.collection.mutable.Map[String, (Thread, Jedis, JedisPubSub)]()

  def publish(message: Message, channel: Channel): Unit = {
    withClient { jedis =>
      jedis.publish(channel.name, message.data)
    }
  }

  def subscribe(channel: Channel, messageReceiver: MessageReceiver): String = {
    val jedis = new Jedis(host)
    val pubsub = createPubSub(messageReceiver)
    val thread = new Thread(() => {
      logger.debug(s"start subscribe to ${channel.name}")
      jedis.subscribe(pubsub, channel.name)
    })
    subscribers.put(thread.getName(), (thread, jedis, pubsub))
    thread.start()
    thread.getName()
  }

  def unsubscribe(name: String): Unit = {
    val (thread, jedis, pubsub) = subscribers(name)
    pubsub.unsubscribe()
    jedis.quit()
  }

  private def withClient[T](f: Jedis => T): T = {
    val jedis = new Jedis(host)
    try {
      f(jedis)
    } finally {
      jedis.quit()
    }
  }

  def createPubSub(messageReceiver: MessageReceiver): JedisPubSub = {
    new JedisPubSub() {
      override def onSubscribe(channel: String,
                               subscribedChannels: Int): Unit =
        logger.debug("onSubscribe")
      override def onUnsubscribe(channel: String,
                                 subscribedChannels: Int): Unit =
        logger.debug("onUnsubscribe")
      override def onPSubscribe(pattern: String,
                                subscribedChannels: Int): Unit =
        logger.debug("onPSubscribe")
      override def onPUnsubscribe(pattern: String,
                                  subscribedChannels: Int): Unit =
        logger.debug("onPUnsubscribe")
      override def onPMessage(pattern: String,
                              channel: String,
                              message: String): Unit =
        logger.debug("onPMessage")
      override def onMessage(channel: String, message: String): Unit =
        messageReceiver.handleMessage(Message(message))
    }
  }

}

trait MessageReceiver {
  def handleMessage(m: Message): Unit
}
```

標準出力

```
14:32:16.672 [Thread-3] DEBUG com.tiqwab.example.messaging.redispubsub.RedisClient$ - start subscribe to channel1
14:32:16.680 [Thread-3] DEBUG com.tiqwab.example.messaging.redispubsub.RedisClient$ - onSubscribe
14:32:17.625 [pubsub-akka.actor.default-dispatcher-2] DEBUG com.tiqwab.example.messaging.redispubsub.PublisherSink - will publish message: Message(Hello, this is 2017-07-30T14:32:17.618+09:00[Asia/Tokyo])
14:32:17.627 [pubsub-akka.actor.default-dispatcher-3] DEBUG com.tiqwab.example.messaging.redispubsub.SubscriberSource - subscribed message: Message(Hello, this is 2017-07-30T14:32:17.618+09:00[Asia/Tokyo])
14:32:17.628 [pubsub-akka.actor.default-dispatcher-3] DEBUG com.tiqwab.example.messaging.redispubsub.RedisPubSubStream$ - Message(Hello, this is 2017-07-30T14:32:17.618+09:00[Asia/Tokyo])
14:32:22.605 [pubsub-akka.actor.default-dispatcher-4] DEBUG com.tiqwab.example.messaging.redispubsub.PublisherSink - will publish message: Message(Hello, this is 2017-07-30T14:32:22.605+09:00[Asia/Tokyo])
14:32:22.610 [pubsub-akka.actor.default-dispatcher-3] DEBUG com.tiqwab.example.messaging.redispubsub.SubscriberSource - subscribed message: Message(Hello, this is 2017-07-30T14:32:22.605+09:00[Asia/Tokyo])
14:32:22.610 [pubsub-akka.actor.default-dispatcher-3] DEBUG com.tiqwab.example.messaging.redispubsub.RedisPubSubStream$ - Message(Hello, this is 2017-07-30T14:32:22.605+09:00[Asia/Tokyo])
...
```

とりあえず出力を見る限り行いたいことはできている感じです。

[1]: https://redis.io/topics/pubsub
[2]: http://www.rediscookbook.org/pubsub_for_asynchronous_communication.html
[3]: https://gist.github.com/FredrikWendt/3343861
[4]: https://github.com/xetorthio/jedis
[5]: http://doc.akka.io/docs/akka/2.5.3/scala/stream/index.html
[6]: https://github.com/tiqwab/example/tree/master/redis-pubsub
[7]: http://doc.akka.io/docs/akka/2.5.3/scala/stream/stream-customize.html
