---
layout: post
title: akka-http サンプル
tags: "akka-http, akka, scala"
comments: true
---

たまに [akka-http][2] を触るときに毎回サーバの起動の仕方や Route の書き方をググり直すのが手間なので、自分なりにサンプルを作成しておくことにしました。

下記は簡単な解説で、サンプルコード全体は [こちら][1] に置いています。

### シンプルに HTTP サーバを動かす

まずは akka-http を最小限の設定とともに動かしてみます。
サーバを起動する main 関数は以下のように書くことができます。

```scala
package com.tiqwab.example.akka

import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.stream.ActorMaterializer
import com.typesafe.config.{Config, ConfigFactory}
import com.typesafe.scalalogging.LazyLogging

import scala.concurrent.{Await, ExecutionContext}
import scala.concurrent.duration._
import scala.util.{Failure, Success}

object SimpleServer extends LazyLogging {

  def main(args: Array[String]): Unit = {
    // Typesafe Config のロード。デフォルトではクラスパス下の `application.conf` を読む
    val config: Config = ConfigFactory.load()
    val host = config.getString("http.host")
    val port = config.getInt("http.port")

    // 上の Config を使用して ActorSystem の作成
    implicit val system: ActorSystem = ActorSystem("simple-system", config)
    // Materializer は akka-http が依存する akka-stream に必要となる
    implicit val materializer: ActorMaterializer = ActorMaterializer()
    implicit val ec: ExecutionContext = system.dispatcher

    // サーバの起動
    val route = new TopicRoute().routes
    Http().bindAndHandle(route, host, port).onComplete {
      case Success(binding) =>
        logger.info(s"Server start at ${binding.localAddress}")
      case Failure(e) =>
        logger.error(s"Error occurred while starting server", e)
        Await.result(system.terminate(), Duration.Inf)
        sys.exit(1)
    }
  }

}
```

akka-http でサーバを起動する際には `Route` と呼ばれるものを渡す必要があります。
上で `bindAndHandle` の第一引数に渡している `route` がそれで、サーバの提供する API を独自の DSL で定義したものです。

```scala
package com.tiqwab.example.akka

import akka.http.scaladsl.server._
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model.StatusCodes._
import akka.http.scaladsl.server.Route

class TopicRoute {

  def routes: Route =
    pathPrefix("topic" / Segment) { topicName =>
      // GET /topic/:topicName
      complete((OK, s"Accept request for $topicName"))
    }

}
```

`application.conf` はお試しで以下のようにしました。

```
akka {
  // 全体のログレベル調整。 'OFF' で抑制可能
  loglevel = INFO
  // システムの startup, shutdown 時に出力されることがある。'OFF' で抑制可能
  stdout-loglevel = INFO

  http {
    server {
      // HTTP レスポンスの Server ヘッダの値
      server-header = "Simple API Server"
    }
  }
}

http {
  host = "127.0.0.1"
  host = ${?SIMPLE_SERVER_HOST}
  port = "8080"
  port = ${?SIMPLE_SERVER_PORT}
}
```

`sbt run` でサーバの動作が確認できます。

```
# Start server
$ sbt run
2017-11-23 14:55:38,953 INFO [simple-system-akka.actor.default-dispatcher-2] c.t.e.a.SimpleServer$ Server start at /127.0.0.1:8080

...

# Execute command in another process
$ curl -v http://localhost:8080/topic/test
> GET /topic/test HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.56.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: Simple API Server
< Date: Thu, 23 Nov 2017 05:47:47 GMT
< Content-Type: text/plain; charset=UTF-8
< Content-Length: 23
< 
Accept request for test
```

### 簡単な API の実装

次は実際に簡単な API を実装してみます。
ここでは以下のような `POST /topic/:name` に body, timestampMillis を持つ json を投げることでメッセージの保存を行い、保存したメッセージの ID を返却する、という API を作成します。

```
$ curl -v -H "Content-Type: application/json" http://localhost:8080/topic/test -d '{"body": "hoge", "timestampMillis": 1}'
> POST /topic/test HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.56.1
> Accept: */*
> Content-Type: application/json
> Content-Length: 38
> 
< HTTP/1.1 200 OK
< Server: Simple API Server
< Date: Thu, 23 Nov 2017 06:50:51 GMT
< Content-Type: application/json
< Content-Length: 11
< 
{"id":"71"}
```

このエンドポイントに相当する Route 定義を抜粋すると以下のようになります。

```scala
package com.tiqwab.example.akka

import akka.actor.ActorSystem
import akka.http.scaladsl.server._
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model.StatusCodes._
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport._
import akka.util.Timeout
import com.typesafe.scalalogging.LazyLogging

import scala.util.{Failure, Success}

class TopicRoute(val system: ActorSystem, val timeout: Timeout)
  extends TopicApi with LazyLogging {

  def routes: Route =
    pathPrefix("topic" / Segment) { topicName =>
      // POST /topic/:name
      post {
        // リクエストから SaveMessageRequest を作成する
        // その際適当な Unmarshaller が必要
        entity(as[SaveMessageRequest]) { request =>
          // onComplete に Future を渡し、完了時の処理を記述する (Try として渡される)
          // saveMessage 処理の内容は後述
          onComplete(saveMessage(topicName, request)) {
            case Success(message) =>
              // complete((StatusCode, T)) で HTTP ステータスコードを指定してレスポンス
              // ここでは 型 T をレスポンスにする適当な Marshaller が必要
              complete((OK, SaveMessageResponse(message.id)))
            case Failure(e) =>
              logger.error("Error occurred while saving message", e)
              complete(InternalServerError)
          }
        }
      } ~
      ...
}
```

この例では `akka-http` における

- リクエストボディの json を指定のクラスにデシリアライズする
- レスポンスの HTTP ステータスコードの指定
- 指定のクラスをレスポンスボディの json としてシリアライズする

といった方法を示しています。

リクエストから、またはレスポンスへとあるオブジェクトを変換したい場合、akka-http ではそれぞれ `Unmarshaller`, `Marshaller` を定義する必要があります。

json を扱う API の場合、主要な json ライブラリ用に `Unmarshaller`, `Marshaller` が既に何らかの形で提供されていることが多いと思うので、それらを使用することができます (上の場合は spray-json を使用し、依存ライブラリ akka-http-spray-json からの `akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport` を import している)。

### Actor の実装

上でリクエストを受け付けられるようになったので、それを実際に処理する部分を実装します。

上の `TopicRoute` に継承してもらっている `TopicApi` trait には Route と処理を行う Actor との橋渡しを行ってもらっており、主に `TopicList` Actor に適当なメッセージを ask するという実装になっています。

このように Route を定義するクラスと具体的な処理を行うクラスを分けておくことでクラス間の責任がはっきりしていいかなと思います。

```scala
package com.tiqwab.example.akka

import akka.actor._
import akka.pattern.ask
import akka.util.Timeout

import scala.concurrent.Future

trait TopicApi {

  implicit val system: ActorSystem
  // 下記の akka.pattern.ask のために必要
  // Timeout で指定した duration を超えてもレスポンスが返ってこない場合、Future の結果が AskTimeoutException になる
  implicit val timeout: Timeout

  val topicList: ActorRef = system.actorOf(TopicList.props, "topicList")

  def saveMessage(topicName: String, request: SaveMessageRequest): Future[Message] =
    // akka.pattern.ask を import することで Actor からのレスポンスを Future で受け取れる
    (topicList ? TopicList.SaveMessage(topicName, request.body, request.timestampMillis)).mapTo[Message]

  def getMessage(topic: String, id: String): Future[Option[Message]] =
    (topicList ? TopicList.GetMessage(topic, id)).mapTo[Option[Message]]

  def listMessage(topic: String): Future[Option[Seq[Message]]] =
    (topicList ? TopicList.ListMessage(topic)).mapTo[Option[Seq[Message]]]

}
```

`TopicList` はよくある感じに実装した Actor であり、`Topic` Actor のまとめ役のような役割をしています。

```scala
package com.tiqwab.example.akka

import akka.actor._

class TopicList extends Actor with ActorLogging {

  var topicMap: Map[String, ActorRef] = Map.empty[String, ActorRef]

  override def receive: Receive = {
    case TopicList.SaveMessage(topicName, body, timestampMillis) =>
      val topic = topicMap.getOrElse(topicName, {
        val newTopic = context.actorOf(Topic.props(topicName))
        topicMap = topicMap + (topicName -> newTopic)
        newTopic
      })
      topic forward Topic.SaveMessage(body, timestampMillis)
    case TopicList.GetMessage(topicName, id) =>
      topicMap.get(topicName) match {
        case None =>
          sender() ! None
        case Some(topic) =>
          topic forward Topic.GetMessage(id)
      }
    case TopicList.ListMessage(topicName) =>
      topicMap.get(topicName) match {
        case None =>
          sender() ! None
        case Some(topic) =>
          topic forward Topic.ListMessage
      }
  }

}

object TopicList {
  def props: Props = Props(new TopicList())
  case class SaveMessage(topic: String, body: String, timestampMillis: Long)
  case class MessageSaved(id: String)
  case class GetMessage(topic: String, id: String)
  case class ListMessage(topic: String)
}
```

`Topic` Actor が実際にメッセージの保存や取得の機能を果たしています。

```scala
package com.tiqwab.example.akka

import akka.actor._

class Topic(val topicName: String) extends Actor with ActorLogging {
  var nextId: Long = 1
  var messages: Seq[Message] = Seq.empty[Message]
  override def receive: Receive = {
    case Topic.SaveMessage(body, timestampMillis) =>
      val id = nextId.toString
      nextId += 1
      val message = Message(id, body, timestampMillis)
      messages = messages :+ message
      sender() ! message
    case Topic.GetMessage(id) =>
      sender() ! messages.find(_.id == id)
    case Topic.ListMessage =>
      // 単体で見ると Some で包む必要は無いが、TopicList からの使われ方の関係で
      sender() ! Some(messages)
  }
}

object Topic {
  def props(topicName: String): Props = Props(new Topic(topicName))
  case class SaveMessage(body: String, timestampMillis: Long)
  case class GetMessage(id: String)
  case object ListMessage
}
```

実装した API の動作を確認します。

```
# Save message
$ curl -v -H "Content-Type: application/json" http://localhost:8080/topic/test -d '{"body": "hoge", "timestampMillis": 1}'
{"id":"1"}

# Get message
$ curl -v -H "Content-Type: application/json" http://localhost:8080/topic/test/1
{"id":"1","body":"hoge","timestampMillis":1}

# List messages
$ curl -v -H "Content-Type: application/json" http://localhost:8080/topic/test
{"messages":[{"id":"1","body":"hoge","timestampMillis":1}]}
```

### 感想

- Actor モデルはあまり馴染みのない概念なのでけっこう面白い
- akka-http は Scala で HTTP インタフェースを持つ簡単な API サーバを作るのに便利そう
- Receive の型が `Any => Unit` の部分関数なので型の保証が無いのが複雑な実装になったときに不安な気がする

[1]: https://github.com/tiqwab/example/tree/master/akka-http-sample
[2]: https://doc.akka.io/docs/akka-http/current/scala/http/
