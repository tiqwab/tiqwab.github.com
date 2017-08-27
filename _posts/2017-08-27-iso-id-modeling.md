---
layout: post
title: Iso な ID のモデリング
tags: "iso, modeling, scala"
comments: true
---

現在 Scala で実装されたサービスの開発に携わっているのですが、そこで定義される Entity のほとんどは ID を case class で表現しています (e.g. Application クラスに対し `case class ApplicationId(value: Long)` で ID を表現する)。

ID を 1 つのクラスとして表現するというのはよくあるモデリングだと思うのですが、見ているとそうしたクラスには Iso 型クラスの定義が与えられていることに気が付きました。

Iso という概念に触れるのは初めてだったのですが、しばらく触っていてこれが地味に便利な表現だなと感じたので少し ID と Iso について整理してみたいと思いました。

使用しているサンプルは [こちら][5] で管理しています。

1. [Iso とは](#anchor1)
2. [ID trait](#anchor2)
3. [e.g. Application Entity](#anchor3)
4. [ScalaCheck と Iso な ID](#anchor4)
5. [SkinnyORM と Iso な ID](#anchor5)

---

<div id="anchor1" />

### 1. Iso とは

拾い物ですが [こちらのスライド][1] の 4 枚目あたりの解説がわかりやすいと思います。

平易な日本語で表現すると、2 つの集合間で相互変換が保証されているという感じでしょうか。

Scala では trait を使用して以下のように表現できそうです。
(上の定義を見ると本当は `reverseGet(get(a)) == a` のような保証が必要な気がします -> あとで ScalaCheck でテストする)

```scala
// 相互に変換する関数から Iso を定義する
// trait の不変条件として reverseGet(get(a)) == a, get(reverseGet(b)) == b
trait Iso[A, B] {
  def get(a: A): B
  def reverseGet(b: B): A
}

object Iso {
  // 相互に変換する関数を与えることで Iso を生成する
  def apply[A, B](_get: (A) => B)(_reverseGet: (B) => A): Iso[A, B] =
    new Iso[A, B] {
      override def get(a: A): B = _get(a)
      override def reverseGet(b: B): A = _reverseGet(b)
    }
}
```

<div id="anchor2" />

### 2. ID trait

ドメインモデルにおける各 Entity が ID をクラスとして表現すると想定します。

例えば `Application` entity に対する ID `ApplicationId` は以下のように定義できます。

```scala
case class ApplicationId(value: Long)
```

`ApplicationId` というのは名前で ID っぽいというのはわかるかもしれませんが、それを明示的に示すために `IdLike` という trait を定義します。

```scala
trait IdLike {
  type Value
  def value: Value
}

trait LongIdLike extends IdLike {
  override type Value = Long
}
```

`ApplicationId` は `LongIdLike` です。

```scala
case class ApplicationId(value: Long) extends LongIdLike
```

`IdLike` を実装した `ApplicationId` のような case class は、`IdLike#Value` 型との間で相互変換が行える、つまり Iso であると言えるはずです。
そのため以下のような trait で Iso が導出されるようにできます。

```scala
trait IsoIdCompanion[A <: IdLike] {
  def apply(a: A#Value): A

  implicit lazy val iso: Iso[A#Value, A] =
    Iso[A#Value, A](apply)(_.value)
}
```

最終的な `ApplicationId` の定義は以下の感じになります。

```scala
case class ApplicationId(value: Long) extends LongIdLike

object ApplicationId extends IsoIdCompanion[ApplicationId]
```

<div id="anchor3" />

### 3. e.g. Application Entity

上で定義した `ApplicationId` を使用した簡単な `Application` entity の定義は以下のようになります。

```scala
case class ApplicationId(value: Long) extends LongIdLike
object ApplicationId extends IsoIdCompanion[ApplicationId]

case class Application(
    id: ApplicationId,
    name: String
) extends Entity[ApplicationId]
```

ここで `Entity[T]` は 文字通り entity を表すための trait で、`id` を持つこと、また `id` のみで同値を判断することが定義されています。

```scala
trait Entity[ID] {
  def id: ID

  override def equals(obj: Any): Boolean = obj match {
    case e: Entity[ID] => this.id == e.id
    case _             => false
  }

  override def hashCode(): Int = 31 * id.##
}
```

<div id="anchor4" />

### 4. ScalaCheck と Iso な ID

`IdLike` に `Iso` を定義すると何が嬉しいのかという話なのですが、簡単にいうと様々な型クラスや定義の導出がシンプルかつ一貫したやり方で行えるというところかなと思います。

上で `Iso` trait を定義した際に、trait の不変条件として `reverseGet(get(a)) == a, get(reverseGet(b)) == b` を設定しました。
ここではその性質を保証できるようなテストを [ScalaCheck][2] で書いてみます。

Iso に対する上の不変条件のチェックは以下の `isoLaws` のようにして表現できると思います。

```scala
package com.tiqwab.example.app

import com.tiqwab.example.modeling.Iso
import org.scalacheck.{Gen, Prop}

trait PropUtils {

  def isoLaws[A: Gen, B: Gen](implicit iso: Iso[A, B]): Prop = {
    val x = Prop.forAll(implicitly[Gen[A]]) { (a: A) =>
      iso.reverseGet(iso.get(a)) == a
    }
    val y = Prop.forAll(implicitly[Gen[B]]) { (b: B) =>
      iso.get(iso.reverseGet(b)) == b
    }
    x && y
  }

}

object PropUtils extends PropUtils
```

上の `isoLaws` は Iso が定義される型 A, B それぞれに対し `Gen` 定義されていないといけないため、下記の `Gens` trait でそれを用意します。

```scala
package com.tiqwab.example.app

import com.tiqwab.example.modeling.{Iso, LongIdLike}
import org.scalacheck.Gen
import org.scalacheck.Arbitrary.arbitrary

trait Gens {

  def positiveLongGen: Gen[Long] =
    Gen.chooseNum(0, Long.MaxValue)

  // Long は Long でも ID 値として使用されるものは 正の数であるはず
  def longIdGen[A <: LongIdLike](implicit iso: Iso[Long, A]): Gen[A] =
    positiveLongGen.map(iso.get(_))

  lazy val applicationIdGen: Gen[ApplicationId] =
    longIdGen[ApplicationId]

}

object Gens extends Gens
```

ここでは `Gen[ApplicationId]` を `longIdGen` という `LongIdLike` なものから `Gen[A]` を定義できる関数によって簡単に定義できるというのが嬉しいと思います。

例えば新たに `UserId` のようなものが出てくる場合でも、同様に `longIdGen` から導出できるはずです。
(というかうまくやれば `xxxIdGen` の宣言すらいらなくて `longIdGen` だけでいけるのでしょうか)

`Gen[A]` を定義したので、以下のようにして `isoLaws` による `ApplicationId` の Iso な性質のチェックを行うことができます。

```scala
package com.tiqwab.example.app

import org.scalatest.FunSuite
import org.scalatest.prop.Checkers._

class ApplicationIdTest extends FunSuite {
  import Gens._
  import PropUtils._

  test("iso check") {
    implicit val longGen = Gens.positiveLongGen
    implicit val appIdGen = applicationIdGen
    check(isoLaws[Long, ApplicationId])
  }

}
```

<div id="anchor5" />

### 5. SkinnyORM と Iso な ID

Scala の ORM ライブラリ [SkinnyORM][3] には上記の `ApplicationId` のようにドメインモデルとして表現される ID と実際の RDB 上での id カラムとのマッピングを行うための trait が提供されています (e.g. [SkinnyCRUDMapperWithId][4])。

ID モデルに Iso を定義することでこういった実装も repository 間である程度共通化することができます。

まずドメインモデルとして entity の永続化を行う `Repository` を定義します。

```scala
trait Repository[ID, E <: Entity[ID], Context] {
  def findById(id: ID)(implicit ctx: Context): Option[E]
  def store(entity: E)(implicit ctx: Context): E
  def deleteById(id: ID)(implicit ctx: Context): Int
}
```

次に永続化先を RDB とした `RepositoryOnJdbc` を定義します。
ここでは `Context` の型を `scalikejdbc.DBSession` に具体化しています。

またこのクラスは `CRUDFeatureWithId[ID, E]` を自分型アノテーションとして宣言しているので、SkinnyORM を知っているクラスということになります。

```scala
import scalikejdbc.{AsIsParameterBinder, DBSession}
import scalikejdbc.interpolation.SQLSyntax
import skinny.orm.feature.CRUDFeatureWithId

trait RepositoryOnJdbc[ID, E <: Entity[ID]]
    extends Repository[ID, E, DBSession] {
  this: CRUDFeatureWithId[ID, E] =>

  protected def namedValuesWithoutId(entity: E): Seq[(SQLSyntax, Any)]

  protected def namedValues(entity: E): Seq[(SQLSyntax, Any)] =
    idNamedValue(entity) +: namedValuesWithoutId(entity)

  protected def idNamedValue(entity: E): (SQLSyntax, Any) =
    column.id -> AsIsParameterBinder(idToRawValue(entity.id))

  override def store(entity: E)(implicit ctx: DBSession): E = {
    if (findById(entity.id).isDefined) {
      updateById(entity.id).withNamedValues(namedValuesWithoutId(entity): _*)
    } else {
      createWithNamedValues(namedValues(entity): _*)
    }
    entity
  }
}
```

`CRUDMapperIsoIdRepository` では SkinnyORM が提供する `SkinnyMapperWithId[ID, E]` trait, `CRUDFeatureWithId[ID, E]` trait を継承して、SkinnyORM による ORM Mapper を表現する抽象クラスを定義します。

SkinnyORM の上記 2 trait は `idToRawValue(id: ID): Any`, `rawValueToId(value: Any): ID` の実装を要求しますが、これらは Iso の持つ関数から定義することができます。

```scala
import skinny.orm.SkinnyMapperWithId
import skinny.orm.feature.CRUDFeatureWithId

abstract class CRUDMapperIsoIdRepository[ID, DbID, E <: Entity[ID]](
    implicit val iso: Iso[DbID, ID]
) extends SkinnyMapperWithId[ID, E]
    with CRUDFeatureWithId[ID, E]
    with RepositoryOnJdbc[ID, E] {

  override def useAutoIncrementPrimaryKey: Boolean = false

  override def idToRawValue(id: ID): Any =
    iso.reverseGet(id)

  override def rawValueToId(value: Any): ID =
    iso.get(value.asInstanceOf[DbID])

}
```

あとは Iso な ID を持つ Entity に関しては `CRUDMapperIsoIdRepository` を継承していい感じに Repository を実装することができます。

```scala
import com.tiqwab.example.app.{Application, ApplicationId}
import com.tiqwab.example.modeling.CRUDMapperIsoIdRepository
import scalikejdbc.WrappedResultSet
import scalikejdbc.interpolation.SQLSyntax
import skinny.orm.Alias

class ApplicationRepositoryOnJdbc(override val tableName: String)
    extends CRUDMapperIsoIdRepository[ApplicationId, Long, Application] {

  override def defaultAlias: Alias[Application] = createAlias("AP")

  override protected def namedValuesWithoutId(
      entity: Application): Seq[(SQLSyntax, Any)] = Seq(
    column.name -> entity.name
  )

  override def extract(rs: WrappedResultSet,
                       n: scalikejdbc.ResultName[Application]): Application =
    Application(
      id = ApplicationId(rs.get(n.id)),
      name = rs.get(n.name)
    )

}
```

### 細々と

はじめに述べた通り、ここで行っている modeling のベースは現在仕事で使用しているコードベースを参考にしたものでした。

他にも play-json の `Format` 定義でも同様に導出ができたりするので、Iso という概念は色々なところで便利に使える表現に思います。

ただいくつか実装上でなぜこうなっているのか理解ができないという部分もあり、Scala でモデルを考える経験が足りないなというのを実感します (コードの良い悪いではなく、そうしている判断や必然性に自分の理解が追いついていない)。

- `IdLike` は総称性で定義してもいいのでは
  - クラスに型メンバを定義することと総称性を持たせることは基本的には同じことができると思っている
  - 型メンバでないとできないこととしては共変まわりの話はあるはず
  - ただ今回の場合はどちらでも実装できそうには思う
- `CRUDMapperIsoIdRepository[ID, DbID, E <: Entity[ID]]` が trait ではなく抽象クラスなわけ
  - `implicit に Iso[DbID, ID]` を受け取りたいためでは
  - SkinnyORM から継承するメソッドもあるので、各メソッドに implicit に受け取らせるわけにはいかない
- `RepositoryOnJdbc[ID, E <: Entity[ID]]` って自分型アノテーションを使用する必然性があるのか
  - 継承ではなく自分型アノテーションを使うモチベーションは？

[1]: https://www.slideshare.net/JulienTruffaut/beyond-scala-lens
[2]: https://www.scalacheck.org/
[3]: http://skinny-framework.org/
[4]: https://github.com/skinny-framework/skinny-framework/blob/master/orm/src/main/scala/skinny/orm/SkinnyCRUDMapperWithId.scala
[5]: https://github.com/tiqwab/example/tree/master/modeling-id
