---
layout: post
title: ScalikeJDBC と 少し ORM
tags: "scalikejdbc, scala"
comments: true
---

ScalikeJDBC は Scala の DB クライアントライブラリの一種ですが、skinnyORM という ORM のベースにもなっています。

skinnyORM の動きを理解するためには ScalikeJDBC 自身の理解を深めるのが一番だと感じたため、ORM としての使用という観点から ScalikeJDBC についての利用例をまとめてみました。

1. [sql 補完子と SQL オブジェクト](#sql-interpolation)
2. [SQLSyntaxSupport の利用](#sqlsyntaxsupport)
3. [TypeBinder の利用](#typebinder)
4. [ManyToOne, OneToMany な関係の表現](#one-to-many)

### 環境

- scala: 2.12
- sbt: 0.13.15
- scalikejdbc: 3.0.0

---

<div id="sql-interpolation" />

### 1. sql 補完子と SQL オブジェクト

ScalikeJDBC を使用したクエリとして [公式マニュアル][1] を参考にして以下のような例を挙げます。

```scala
import scalikejdbc._

DB readOnly { implicit session =>
  val id: Long = 123L
  val ordering: scalikejdbc.SQLSyntax = sqls"order by name"
  val names: List[String] =
    sql"select name from emp where dept_id = ${id} ${ordering}"
      .map(rs => rs.string("name")).list.apply()
}
```

`sql` は ScalikeJDBC が定義する string interpolation であり、クエリの定義を行っています。
`sql` 内で使用した変数は、型によって以下のようにクエリへの組み込まれ方が異なります。

- `SQLSyntax` 型の場合
  - その変数の表す文字列がそのまま組み込まれる
  - 自分で定義する場合は `sqls` が便利
- その他の ScalikeJDBC が native に扱える型 (Int, String, ZonedDateTime, etc.)
  - place holder を持つクエリが作成され、値をバインドして使用する

`sql` は実際には `SQL` 型のオブジェクトを作成しており、以下のコードは上のクエリと同様な動きをします。

```scala
DB readOnly { implicit session =>
  // Create SQL object directly (not recommended)
  val id: Long = 123L
  val names: List[String] =
    SQL("SELECT name from emp where dept_id = ? order by name")
      .bind(id)
      .map(rs => rs.string("name")).list.apply()
}
```

<div id="sqlsyntaxsupport" />

### 2. SQLSyntaxSupport の利用

`sql` 補完子を使用すれば任意の SQL を実行できますが、ORM のような複雑な機能を実装しようと思うと記述量が多くなりがちです。

ScalikeJDBC では簡単な ORM 実装のために `SQLSyntaxSupport[A]` trait が用意されています。

```scala
import java.time.ZonedDateTime
import scalikejdbc._

case class Member(
    id: Option[Long],
    name: String,
    email: String,
    createdAt: ZonedDateTime,
    updatedAt: ZonedDateTime
)

// Companion object extends `SQLSyntaxSupport[A]`
object Member extends SQLSyntaxSupport[Member] {

  // `m` provides type-safe table and column references
  // val m: scalikejdbc.QuerySQLSyntaxProvider[scalikejdbc.SQLSyntaxSupport[Member], Member] = this.syntax("m")
  val m = this.syntax("m")

  // Define table name corresponding to the Member class
  override val tableName = "members"

  // Maybe necessary when using multiple datasources
  // The `autoColumns` macro comes from `scalikejdbc-syntax-support-macro` library
  // override val columns = Seq("id", "name", "email", "created_at", "updated_at")
  override val columns = autoColumns[Member]()

  // Construct Member from WrappedResultSet with macro
  // The `autoConstruct` macro comes from `scalikejdbc-syntax-support-macro` library
  // Ref. http://scalikejdbc.org/documentation/auto-macros.html
  def apply(rs: WrappedResultSet): Member = autoConstruct(rs, m.resultName)

}
```

`SQLSyntaxSupport` を使用してクエリは以下のように実行できます。

```scala
DB autoCommit { implicit session =>
  // Create table
  sql"""
       CREATE TABLE IF NOT EXISTS members (
         id BIGINT AUTO_INCREMENT,
         name VARCHAR(255) NOT NULL,
         email VARCHAR(255) NOT NULL,
         created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
         updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
         PRIMARY KEY (id),
       UNIQUE (email ASC)
       ) ENGINE = InnoDB;
     """.execute.apply()

  // (insert member data)

  // Query using SQLSyntaxSupport
  val memberId = 1
  val alice =
    sql"SELECT ${m.result.*} FROM ${Member as m} WHERE ${m.id} = ${memberId}"
      .map(rs =>
        Member(
          id = rs.longOpt(m.resultName.id),
          name = rs.string(m.resultName.name),
          email = rs.string(m.resultName.email),
          createdAt = rs.zonedDateTime(m.resultName.createdAt),
          updatedAt = rs.zonedDateTime(m.resultName.updatedAt)
      ))
      .single
      .apply()

  // Query with `autoConstruct`
  val alice2 = 
    sql"SELECT ${m.result.*} FROM ${Member as m} WHERE ${m.id} = ${memberId}"
      .map(Member(_))
      .single
      .apply())
}
```

`Member` のコンパニオンオブジェクトでは `m` という名前で `SQLSyntaxProvider` を定義しています。
`SQLSyntaxSupport` の例でよく見るように `sql` 補完子の中で `${m.result.*}` や `${m.id}` という形で、また `ResultSetWrapper` からのクエリ結果取り出しの際に `m.resultName.id` のようにして使用されています。

これらの型はいずれも `SQLSyntax` の実装の一つであり、`SQLSyntaxProvider` はその名の通り `SQLSyntax` を `sql` 補完子に渡すことで、ORM におけるボイラープレートな記述を省くのに貢献しているのだと言えそうです。

参考:

- [SQLInterpolation][2]

<div id="typebinder" />

### 3. TypeBinder の利用

例えば ORM の対象となる `Item` オブジェクトの識別子が Long 等の汎用的な型ではなく `ItemId` ような専用の型であるとします。

```scala
import java.time.ZonedDateTime

case class Item(
    id: ItemId, // identifier
    name: String,
    createdAt: ZonedDateTime,
    updatedAt: ZonedDateTime
)

case class ItemId(value: Long) extends AnyVal
```

この場合そのままでは ScalikeJDBC は `ItemId` 型の扱い方がわからないので、こちらでその情報を与える必要があります (あるいはクエリ実行時に逐一変換処理を記述してもよいが場所が多いと煩雑になる)。

変換の情報として `Item` のコンパニオンオブジェクトに `TypeBinder` を定義します。

```scala
import scalikejdbc._

object Item extends SQLSyntaxSupport[Item] {

  // Define TypeBinder of ItemId implicitly
  // xmap: (f: (A) => B, g: (B) => A) => Binders[B]
  implicit val itemIdBinder =
    Binders.long.xmap[ItemId](ItemId.apply, _.value)

  val i = syntax("i")
  override def tableName: String = "items"
  override def columns: Seq[String] = autoColumns[Item]()
  def apply(rs: WrappedResultSet): Item = autoConstruct(rs, i.resultName)

}
```

こうすることでこれまでと同様の形式でクエリの実行が行えます。

```scala
NamedDB('mysql) autoCommit { implicit session =>
  // Create table
  sql"""
    CREATE TABLE IF NOT EXISTS items (
      id BIGINT AUTO_INCREMENT,
      name VARCHAR(255) NOT NULL,
      CREATED_AT TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
      UPDATED_AT TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
      PRIMARY KEY (id),
      UNIQUE (name ASC)
    )
    ENGINE = InnoDB;
    """.execute.apply()

  // (insert item data)

  val i = Item.i

  // Query without TypeBinder
  val item1 =
    sql"SELECT ${i.result.*} FROM ${Item as i} WHERE ${i.id} = 1"
      .map(rs => {
        val id = ItemId(rs.long(i.resultName.id))
        Item(
          id,
          rs.string(i.resultName.name),
          rs.zonedDateTime(i.resultName.createdAt),
          rs.zonedDateTime(i.resultName.updatedAt))
      })
      .single
      .apply()
  }

  // Query with TypeBinder
  val item2 =
    sql"SELECT ${i.result.*} FROM ${Item as i} WHERE ${i.id} =  1"
      .map(Item(_))
      .single
      .apply()
}
```

<div id="one-to-many" />

### 4. ManyToOne, OneToMany な関係の表現

### ManyToOne

ManyToOne として Storer (荷主) に対し Sku (商品) が n:1 な関係を考えます。
つまり各商品は必ずある Storer 1 つに結びつきます。

モデルとしては Sku 側に Storer への参照を持たせます。

```scala
import scalikejdbc._

case class Storer(
    id: Option[Long],
    name: String
)

object Storer extends SQLSyntaxSupport[Storer] {
  val st = syntax("st")
  override val tableName: String = "storers"
  override val columns: Seq[String] = autoColumns[Storer]()
  def apply(rs: WrappedResultSet): Storer = autoConstruct(rs, st)
}
```

```scala
import scalikejdbc._

case class Sku(
    id: Option[Long],
    name: String,
    storerId: Long,
    storer: Option[Storer] = None
)

object Sku extends SQLSyntaxSupport[Sku] {
  val sk = syntax("sk")
  override val tableName = "skus"
  override def columns: Seq[String] = autoColumns[Sku]("storer")

  // Use `Storer.apply` to construct Sku
  def apply(rs: WrappedResultSet): Sku = {
    val sku = autoConstruct(rs, sk, "storer")
    sku.copy(storer = Some(Storer(rs)))
  }
}
```

クエリの実行は以下のようになります。
`Sku.apply(rs: WrappedResultSet)` 内で `Storer` の構築も行っているのでクエリ上はシンプルになっています。

```scala
NamedDB('mysql) autoCommit { implicit session =>
  sql"""
    CREATE TABLE IF NOT EXISTS storers (
      id BIGINT AUTO_INCREMENT,
      name VARCHAR(255) NOT NULL,
      PRIMARY KEY (id),
      UNIQUE (name asc)
    )
    ENGINE = InnoDB;
    """.execute.apply()

  sql"""
    CREATE TABLE IF NOT EXISTS skus (
      id BIGINT AUTO_INCREMENT,
      name VARCHAR(255) NOT NULL,
      storer_id BIGINT NOT NULL,
      PRIMARY KEY (id),
      UNIQUE (name asc),
      FOREIGN KEY (storer_id) REFERENCES storers (id)
    ) ENGINE = InnoDB;
    """.execute.apply()

  // (insert Storer and Sku data)

  val st = Storer.st
  val sk = Sku.sk

  // Query for list of Sku
  val skus = withSQL {
    selectFrom(Sku as sk).innerJoin(Storer as st).on(sk.storerId, st.id)
  }.map(Sku(_)).list.apply()
}
```

### OneToMany

同様のモデルで OneToMany な関係を表します。
今度は `Storer` に対応する `Sku` のリストをもたせるようにします。

```scala
import scalikejdbc._

case class Storer(
    id: Option[Long],
    name: String,
    skus: Seq[Sku] = Nil
)

object Storer extends SQLSyntaxSupport[Storer] {
  val st = syntax("st")
  override val tableName: String = "storers"
  // Ignore 'skus' which does not exist in table
  override val columns: Seq[String] = autoColumns[Storer]("skus")
  // Ignore 'skus' which does not exist in table
  def apply(rs: WrappedResultSet): Storer = autoConstruct(rs, st, "skus")
}
```

```scala
import scalikejdbc._

case class Sku(
    id: Option[Long],
    name: String,
    storerId: Long,
)

object Sku extends SQLSyntaxSupport[Sku] {
  val sk = syntax("sk")
  override val tableName = "skus"
  override def columns: Seq[String] = autoColumns[Sku]()
  def apply(rs: WrappedResultSet): Sku = autoConstruct(rs, sk)
  def opt(rs: WrappedResultSet): Option[Sku] =
    rs.longOpt(sk.resultName.id).map(_ => apply(rs))
}
```

クエリは以下のようになります。

```scala
NamedDB('mysql) autoCommit { implicit session =>
  // (create table, same as the above)
  // (insert storer and sku data)

  // Query for list of storers
  val st = Storer.st
  val sk = Sku.sk

  val sql: SQL[Storer, NoExtractor] = withSQL {
    selectFrom(Storer as st).leftJoin(Sku as sk).on(st.id, sk.storerId)
  }
  val storers =
    sql
      .one(Storer(_))
      .toMany(Sku.opt(_))
      .map((storer: Storer, skus: Seq[Sku]) => storer.copy(skus = skus))
      .list
      .apply()
}
```

[1]: http://scalikejdbc.org/documentation/operations.html
[2]: http://scalikejdbc.org/documentation/sql-interpolation.html
