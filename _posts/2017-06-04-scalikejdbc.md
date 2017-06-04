---
layout: post
title: ScalikeJDBC 入門
tags: "scalikejdbc, scala"
comments: true
---

最近 Scala を始めたので、DB 処理によく使用されているのを見る [ScalikeJDBC][1] を触ってみます。

1. [GetStarted](#anchor1)
2. [Typeseaf Config を使用した初期化](#anchor2)
3. [SQLSyntaxSupport の使用](#anchor3)
4. [Query DSL の使用](#anchor4)

### 環境

- scala: 2.12
- sbt: 0.13.15
- scalikejdbc: 3.0.0

---

<div id="anchor1" />

### 1. GetStarted

ScalikeJDBC の使い始めとして最もシンプルな使い方を押さえておきます。
この使い方ではコードを読んで何を行っているかがすぐにわかるぐらいシンプルに処理を書くことができます。
一方で Connection を明示的に初期化する必要がある等少し煩雑なところもあります。

ScalikeJDBC の使用には以下のライブラリが必要です (version は適宜変更)。
h2 に関しては使用する DB が異なればそれに応じた JDBC Driver を代わりに用意することになります。

```
"org.scalikejdbc" %% "scalikejdbc" % "3.0.0"
"ch.qos.logback" % "logback-classic" % "1.2.3"
"com.h2database" % "h2" % "1.4.195"
```

ScalikeJDBC を使用したコード例は以下の通りです。

- (1) JDBC ドライバの読込
- (2) コネクションプールのセットアップ
  - 複数のデータソースを使用することも可能 (ref. [コネクション管理][7])
- (3) `scalikejdbc.DB` を使用してセッションの開始 (ref. [トランザクション][6])
  - `autoCommit`, `readOnly`, `localTx`, `withinTx` といったものがある
- (4) `sql` 補完子によるクエリの定義
  - 本来プレイスホルダを置く位置に `$var_name` のようにして直接変数を埋め込める
  - この値はプレイスホルダ使用時と同様エスケープ処理される

```scala
import scalikejdbc._

/**
 * Insert and select records from a created 'members' table.
 */
object BasicSample extends App {

  // (1)
  Class.forName("org.h2.Driver")
  // (2)
  ConnectionPool.singleton("jdbc:h2:mem:hello", "user", "pass")

  // (3)
  DB autoCommit { implicit session =>
    sql"""
      create table members (
        id serial not null primary key,
        name nvarchar(64),
        created_at timestamp not null
      )
    """.execute.apply()
  }

  DB localTx { implicit session =>
    Seq("Alice", "Bob", "Chris") foreach { name =>
      // (4)
      sql"insert into members (name, created_at) values ($name, current_timestamp)".update.apply()
    }
  }

  DB readOnly { implicit session =>
    // (4)
    val entities: List[Map[String, Any]] = sql"select * from members".map(_.toMap).list.apply()
    println(entities)
  }

}
```

<div id="anchor2" />

### 2. Typeseaf Config を使用した初期化

上の 1. で少し煩雑だったところをうまくやるために、[Typeseaf Config][5] を使用した Connection のセットアップを行います。
まずプロジェクトに以下のライブラリを追加します (version は適宜変更)。

```
"org.scalikejdbc" %% "scalikejdbc-config" % "3.0.0"
```

次に `scr/main/resources/application.conf` に Connection の設定等を書きます。
ここでは Connection Pool の設定は全てデフォルトのものを使用するのでコメントアウトしています。

```
# JDBC settings
db.default.driver="org.h2.Driver"
db.default.url="jdbc:h2:mem:hello"
db.default.user="user"
db.default.password="pass"

# Connection Pool settings
# db.default.poolInitialSize=5
# db.default.poolMaxSize=7
# db.default.poolConnectionTimeoutMillis=1000
# db.default.poolValidationQuery="select 1 as one"
# db.default.poolFactoryName="commons-dbcp"
```

使用時には `scalikejdbc.config.DBs` を使用して初期化を行います。
そのあとは 1. と同様です。

```scala
import scalikejdbc._
import scalikejdbc.config._

object ConfigSample extends App {

  // Load `application.conf` and setup.
  DBs.setupAll()

  DB autoCommit { implicit session =>
  ...
  }
}
```

<div id="anchor3" />

### 3. SQLSyntaxSupport の使用

`scalikejdbc.SQLSyntaxSupport` は `ResultSet` とモデル間の変換まわりの記述量を減らすために定義されている trait です (ref. [SQLSyntaxSupport][8])。

ここからは Group と Member という 1 対 多の関係を持つモデルを使用していきます。

- (1) モデルを表すクラスのコンパニオンオブジェクトに `SQLSyntaxSupport` を実装させる
  - schema, table の指定
  - `ResultSet` から 対象オブジェクトを作成するための `apply` メソッドの実装

```scala
import org.joda.time.DateTime
import scalikejdbc._

case class Group(id: Long, name: String, createdAt: DateTime)

// (1)
object Group extends SQLSyntaxSupport[Group] {

  override val schemaName: Option[String] = None
  override val tableName: String = "groups"

  def apply(g: ResultName[Group])(rs: WrappedResultSet): Group = {
    Group(rs.long(g.id), rs.string(g.name), rs.jodaDateTime(g.createdAt))
  }

}

// Cannot remove groupId to use in query...
case class Member(id: Long, name: Option[String], groupId: Long, group: Group, createdAt: DateTime)

// (1)
object Member extends SQLSyntaxSupport[Member] {

  override val schemaName: Option[String] = None
  override val tableName: String = "members"

  def apply(m: ResultName[Member], g: ResultName[Group])(rs: WrappedResultSet): Member = {
    val group = Group(g)(rs)
    Member(rs.long(m.id), rs.stringOpt(m.name), rs.long(m.groupId), group, rs.jodaDateTime(m.createdAt))
  }

}
```

実際にクエリを作成するコードは以下のようになります。

- (2) 上で定義したコンパニオンオブジェクトの `column` フィールドにモデルが持つカラム一覧が入っている
  - 初回テーブルアクセス時に取得してきている?
- (3) (2) で取得した `column` を使用して INSERT クエリを投げる
- (4) SELECT に関しては、はじめにコンパニオンオブジェクトの `syntax` メソッドを実行し、それを利用してクエリを作成する
  - `m.id`, `m.result.id`, `m.resultName` がそれぞれ違う意味を持っていることに注意

```scala
import com.tiqwab.example.scalikejdbc.model.{Group, Member}
import scalikejdbc._
import scalikejdbc.config._

object SyntaxSupportSample extends App {

  DBs.setupAll()

  DB autoCommit { implicit session =>
    sql"""
      create table groups (
        id serial not null primary key,
        name nvarchar(64) not null,
        created_at timestamp not null
      )
    """.execute.apply()

    sql"""
      create table members (
        id serial not null primary key,
        name nvarchar(64),
        group_id bigint,
        created_at timestamp not null,
        foreign key (group_id) references groups (id)
      )
    """.execute.apply()
  }

  DB localTx { implicit session =>
    // (2)
    val g = Group.column
    Seq("group1", "group2") foreach { name =>
      // (3)
      sql"insert into ${Group.table} (${g.name}, ${g.createdAt}) values ($name, current_timestamp)".update.apply()
    }
    // (2)
    val m = Member.column
    Seq(("Alice", 1), ("Bob", 1), ("Chris", 2)) foreach {
      case (name, groupId) =>
        // (3)
        sql"insert into ${Member.table} (${m.name}, ${m.groupId}, ${m.createdAt}) values ($name, $groupId, current_timestamp)".update.apply()
    }
  }

  DB readOnly { implicit session =>
    // (4)
    val (m, g) = (Member.syntax("m"), Group.syntax("g"))
    val member =
      sql"""
        select
          ${m.result.*}, ${g.result.*}
        from
          ${Member as m} inner join ${Group as g}
        where
          ${m.groupId} = ${g.id}
      """.map(Member(m.resultName, g.resultName)(_)).first.apply()
    println(member)
  }

}
```

<div id="anchor4" />

### 4. Query DSL の使用

最後に型安全に SQL を構築できる Query DSL を使用してみます (ref. [Query DSL][9])。
上の 3. で使用した例を Query DSL で書き直すと以下のようになります。

- (1) `withSQL` の中で SQL like なメソッドチェーンによりクエリを構築する

```scala
import com.tiqwab.example.scalikejdbc.model.{Group, Member}
import org.joda.time.DateTime
import scalikejdbc._
import scalikejdbc.config._

object QuerySample extends App {

  DBs.setupAll()

  DB localTx { implicit session =>
    val (m, g) = (Member.column, Group.column)
    Seq("group1", "group2") foreach { name =>
      // (1)
      withSQL {
        insert.into(Group).columns(g.name, g.createdAt).values(name, DateTime.now)
      }.update.apply()
    }
    Seq(("Alice", 1), ("Bob", 1), ("Chris", 2)) foreach {
      case (name, groupId) =>
        // (1)
        withSQL {
          insert.into(Member).columns(m.name, m.groupId, m.createdAt).values(name, groupId, DateTime.now)
        }.update.apply()
    }
  }

  DB.readOnly { implicit session =>
    val id = 1
    val (m, g) = (Member.syntax("m"), Group.syntax("g"))
    // (1)
    val member = withSQL {
      select.from(Member as m).innerJoin(Group as g).on(m.groupId, g.id)
        .where.eq(m.id, id)
    }.map(Member(m.resultName, g.resultName)(_)).single.apply()
    println(member)
  }

}
```

### References

- [ScalikeJDBC][1]
- [Getting Started in Japanese][2]

[1]: http://scalikejdbc.org/
[2]: https://github.com/scalikejdbc/scalikejdbc/wiki/GettingStartedInJapanese
[3]: http://scalikejdbc.org/documentation/configuration.html
[4]: http://scalikejdbc.org/documentation/operations.html
[5]: https://github.com/typesafehub/config
[6]: https://github.com/scalikejdbc/scalikejdbc/wiki/GettingStartedInJapanese#%E3%83%88%E3%83%A9%E3%83%B3%E3%82%B6%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3
[7]: https://github.com/scalikejdbc/scalikejdbc/wiki/GettingStartedInJapanese#%E3%82%B3%E3%83%8D%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E7%AE%A1%E7%90%86
[8]: http://scalikejdbc.org/documentation/sql-interpolation.html
[9]: http://scalikejdbc.org/documentation/query-dsl.html
