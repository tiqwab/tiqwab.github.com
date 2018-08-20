---
layout: post
title: Stateful Test with ScalaCheck
tags: "stateful test, scalacheck, scala"
comments: true
---

最近 Property Based Testing 力を鍛えたくてググって見つけた [PropEr Testing][5] という資料を読みふけっています。この本は網羅的に Property Based Testing について書かれており (Erlang を頑張って読み解ければ) 随所に参考になる点が散りばめられている素敵な本です。自分が特に興味を持ったのは Stateful なシステムに対して Property Based Testing を実践という章なのですが、調べてみると [ScalaCheck にもそのための機能][2] があるようなので少し遊んでみようと思いました。

環境:

詳細は [build.sbt][1] を。

- Scala: 2.12.6
- ScalaCheck: 1.14.0

### 1. Stateful Test 概要

Stateful Test がどのような実装になっているのかというのは [Stateful Properties - PropEr Testing][3] を読むか、[ScalaCheck の User Guide][2] の該当する項を読むのがいいかと思います。前者はサンプルコードが Erlang なので慣れていないと難しいのですが、解説はこちらの方が詳しいと思います。はじめの図と文章だけでも読むとイメージがつかみやすいかもしれません。

ScalaCheck では Stateful Test を行うための実装がほとんどライブラリ側で用意されており、ユーザが最低限与えるものは Sut, State, Command という 3 つだけで済みます。

Sut というのは System Under Test のことでテスト対象のシステムです。[ScalaCheck の User Guide][3] 内の例では `Counter` というカウント値を状態として持つシステムを定義して使用しています。

State は Sut の内部状態に相当するものを表現するためのデータ構造です。上の `Counter` に対してはそのまま `Int` を使用していますね。相当するものという回りくどい表現をしているのは、Sut の内部状態を必ずしも完全に模倣しないといけないわけじゃなく、テストとして必要なものを定義できればいいためです。

Command は Sut に行う操作を表現するためのデータ型です。主に以下の関数で構成されます。

- preCondition
  - Command を実行するための事前条件
- run
  - Sut に行う操作
- postCondition
  - Command 実行後に満たすべき事後条件
- nextState
  - State に行う操作

これらを使用した Stateful Test の大まかな流れは

1. テストを行うための Command 列を生成する。このとき Command の preCondition, nextState を使用して、各 Command 実行時に preCondition が満たされるような Command 列にする
2. 得られた Command 列の先頭から Command が持つ run を実行し、sut への操作を行う。各 Command の run 実行後、postCondition が満たされているかを確認し、そうでなければテストを失敗させる。満たされていれば nextState により State を更新し、次の Command 実行に移る。

というようになります。

### 2. Stateful Test 第一歩

ScalaCheck でどのように Stateful Test を実装していくかを見るために、ここでは RDB を使用する repository に対して Stateful Test を実装してみます。

まずは `User` という entity と `UserRepositoryOnJdbc` という repository を用意します。

```scala
case class User(id: Long, name: String, age: Int)
```

```scala
class UserRepositoryOnJdbc {
  def insert(user: User): Unit = ???
}
```

[UserGuide][2] の Stateful Testing 項を参考に以下のようにテストを実装します。

```scala
import org.scalacheck._
import org.scalacheck.Arbitrary._
import org.scalacheck.commands.Commands
import scalikejdbc.config._

// (1)
object UserRepositoryCheck extends Commands {

  // (2)
  override type Sut = UserRepositoryOnJdbc

  // (3)
  override type State = Map[Long, User]

  override def canCreateNewSut(newState: State,
                               initSuts: Traversable[State],
                               runningSuts: Traversable[Sut]): Boolean = {
    true
  }

  override def newSut(state: State): Sut = {
    new UserRepositoryOnJdbc()
  }

  override def destroySut(sut: Sut): Unit = ()

  override def initialPreCondition(state: State): Boolean = state.isEmpty

  override def genInitialState: Gen[State] = Gen.const(Map.empty[Long, User])

  // --- Gen

  def nameGen: Gen[String] =
    for {
      len <- Gen.choose(1, 255)
      chars <- Gen.listOfN(len, Gen.alphaNumChar)
    } yield chars.mkString

  def userGen: Gen[User] =
    for {
      id <- Gen.choose(1, Long.MaxValue)
      name <- nameGen
      age <- Gen.choose(1, 255)
    } yield User(id, name, age)

  def insertCmdGen: Gen[Insert] = userGen.map(Insert.apply)

  // (4)
  override def genCommand(state: State): Gen[UserRepositoryCheck.Command] = {
    insertCmdGen
  }

  // --- Command

  // (5)
  case class Insert(user: User) extends UnitCommand {

    override def preCondition(state: Map[Long, User]): Boolean = true

    // When an exception was thrown, the 'success' argument in postCondition will be false
    override def run(sut: UserRepositoryOnJdbc): Unit = {
      sut.insert(user)
    }

    override def postCondition(state: Map[Long, User], success: Boolean): Prop =
      success

    override def nextState(state: Map[Long, User]): Map[Long, User] = state + (user.id -> user)

  }

  // --- main

  def main(args: Array[String]): Unit = {
    DBs.setupAll()
    UserRepositoryCheck.property().check()
  }

}
```

(1) ScalaCheck では Stateful Test のための trait として `org.scalacheck.commands.Commands` を提供しています。

(2) Sut の型を定義します

(3) Sut が持つ状態を表現できるようなデータ構造を定義します。今回は id をプライマリキーとして RDB に永続化を行う Repository に対するテストなので Map を使用しました。

(4) Command に対する Gen 定義です

(5) Command の定義です。ここでは preCondition は常に満たされているとしています。 postCondition では success に sut への操作が成功したか (例外が投げられてはいないか) が渡されるので、それを評価します。なお、postCondition で何が渡されるかは継承する trait (ここでは `UnitCommand`) により変わります。 nextState では操作に応じて state を更新します。今の場合 Insert なので Map への要素の追加でよいと考えられます。

main を実行すると UserRepository の insert はまだ未実装なので想定通りテストがコケます。

```
! Falsified after 1 passed tests.
> Labels of failing property: 
initialstate = Map()
seqcmds = (Insert(User(7640483227752933834,qLffdpxesAluugyuswnujuuktgxvNge9
  aei3n5ccUkyfgDufv9mbO53ho5irxidq1bXklnbyEggrbybvovryxacs2yszvcend02tspgsn
  nnlpa8tyxga07RCpxRgsdo5cvdrVxbwas8elbowsrw1mhS0W6hbZtdctghpozRkb9vqhvtoev
  w1rgklq475cybKhqn9e,48)) => Failure(scala.NotImplementedError: an impleme
  ntation is missing))
> ARG_0: Actions(Map(),List(Insert(User(7640483227752933834,qLffdpxesAluugy
  uswnujuuktgxvNge9aei3n5ccUkyfgDufv9mbO53ho5irxidq1bXklnbyEggrbybvovryxacs
  2yszvcend02tspgsnnnlpa8tyxga07RCpxRgsdo5cvdrVxbwas8elbowsrw1mhS0W6hbZtdct
  ghpozRkb9vqhvtoevw1rgklq475cybKhqn9e,48))),List())
```

UserRepository の insert を定義します。

```scala
import scalikejdbc._

class UserRepositoryOnJdbc {

  def insert(user: User)(implicit session: DBSession = AutoSession): Unit =
    sql"INSERT INTO USERS (id, name, age) VALUES (${user.id}, ${user.name}, ${user.age});".execute.apply()

}
```

再度テストを実行すると今度は通ります。

```
+ OK, passed 100 tests.
```

同様にして一通り CRUD 操作に対するテストが行えるようにします。

`UserRepositoryOnJdbc` に findById, update, deleteById を定義します。

```scala
import scalikejdbc._

class UserRepositoryOnJdbc {

  def insert(user: User)(implicit session: DBSession = AutoSession): Unit =
    sql"INSERT INTO USERS (id, name, age) VALUES (${user.id}, ${user.name}, ${user.age});".execute.apply()

  def findById(id: Long)(implicit session: DBSession = AutoSession): Option[User] =
    sql"SELECT id, name, age FROM USERS WHERE id = $id"
      .map { rs =>
        User(rs.long("id"), rs.string("name"), rs.int("age"))
      }
      .single
      .apply()

  def update(user: User)(implicit session: DBSession = AutoSession): Unit = {
    sql"UPDATE USERS SET name = ${user.name}, age = ${user.age} WHERE id = ${user.id}".update.apply()
    ()
  }

  def deleteById(id: Long)(implicit session: DBSession = AutoSession): Unit = {
    sql"DELETE FROM USERS WHERE id = $id".update.apply()
    ()
  }

}
```

テスト側では findById, update, deleteById にそれぞれ対応する Command を定義します。

```scala

  override def genCommand(state: State): Gen[UserRepositoryCheck.Command] =
    Gen.oneOf(insertCmdGen, findByIdCmdGen, updateCmdGen, deleteByIdCmdGen)

  // --- Command

  case class Insert(user: User) extends UnitCommand {
      (省略)
  }

  case class FindById(id: Long) extends UnitCommand {

    override def preCondition(state: Map[Long, User]): Boolean = true

    override def run(sut: UserRepositoryOnJdbc): Unit = { sut.findById(id); () }

    override def postCondition(state: Map[Long, User], success: Boolean): Prop = success

    override def nextState(state: Map[Long, User]): Map[Long, User] = state

  }

  case class Update(user: User) extends UnitCommand {

    override def preCondition(state: Map[Long, User]): Boolean = true

    override def run(sut: UserRepositoryOnJdbc): Unit = sut.update(user)

    override def postCondition(state: Map[Long, User], success: Boolean): Prop = success

    override def nextState(state: Map[Long, User]): Map[Long, User] = state.updated(user.id, user)

  }

  case class DeleteById(id: Long) extends UnitCommand {

    override def preCondition(state: Map[Long, User]): Boolean = true

    override def run(sut: UserRepositoryOnJdbc): Unit = sut.deleteById(id)

    override def postCondition(state: Map[Long, User], success: Boolean): Prop = success

    override def nextState(state: Map[Long, User]): Map[Long, User] = state - id

  }
```

print デバッグで実際に生成される Command を確かめてみます。

```
FindById(9149094264222000987)
DeleteById(531240210253435891)
Insert(User(4606761962908959740,4Ifmztr1iwBOOsuufbcdobkazlf3nfnRoypqgzwiafogOgni7k1xDdeagd8reuwkfldwztzjEkhykaRorX2atz3zh0j7ydHw9fjyQmZjbzq6ivczx2u3uTmjRmytgmv6rjawo9njje0e0m6anxhmy5smeXbkm8idnmrvpiLaVfjaqQk,66))
FindById(8075987065827466226)
Update(User(3706352515100550283,ny2pufmpeoffmjev,230))
DeleteById(6590231960033013110)
DeleteById(1742295382157389395)
Update(User(548976644593665019,bttIisdmx1bnbblyuin6mv9wjlcbhohdzkzucktnyOinaeilirabyvse01yoizoUvyabtj5bkka7ltbh9rdEwhp1iIqsnmbx,51))
Insert(User(1663415093799002157,kWh6fvunkjbapwmuna4qeazXrNnanhoRrpBnsvdw9wdr5a3uHukw2fawjrbB4svqalnoCa5dheHcoaO7tmexpoe8gjPkgrctssdwbrdd8rz9duiwsgei,73))
...
```

Command の preCondition はいずれも常に true ですし、frequency も特に定義していないので、CRUD が均等に実行されているはずです。

### 3. もう少し意味を持たせた Command を

これで repository に対する Stateful Test ができた! と言いたいところなのですが、上のテストだと多くの場合不十分だと思います。というのは User を完全にランダムに生成しているので、「新しく User を追加する」 という操作は実行される可能性が高い一方で 「既に存在する User を追加する」 操作が実行されにくいといったことがあるためです。ランダムな User を使用することで fuzzing 的なテストは行なえますが、実行される Command をよりコントロールしたいという要求もあると思います。

これに対するアプローチとして [Case Study: Stateful Properties With a Bookstore Stateful Properties - PropEr Testing][3] では 3 つ挙げられていますが、ここではその中から 「Introduce more determinism in the commands」 という方法を実装してみます。

これを自分なりに言い換えると 「Command を期待する状況に応じて細分化する」 となるのですが、言葉で説明するよりは実装例を見る方がわかりやすそうです。

例えば insert について 「すでにその User が存在するならば UserAlredayExistsException を返す」 というのが仕様に入ったとして、それをテストでカバーできるようにしてみます。

```scala
import scalikejdbc._

import scala.util.Try

class UserRepositoryOnJdbc {

  // Changed the type of return value from Unit to Try[Unit]
  def insert(user: User)(implicit session: DBSession = AutoSession): Try[Unit] =
    findById(user.id) map {
      case None =>
        sql"INSERT INTO USERS (id, name, age) VALUES (${user.id}, ${user.name}, ${user.age});".execute.apply()
      case Some(_) =>
        throw new UserAlreadyExistsException()
    }

  ...

}

class UserAlreadyExistsException extends Exception
```

これに応じて、これまでユーザを追加するということしか表現していなかった Insert Command を 「新しく User を追加する」、「既に存在する User を追加する」 という 2 つの Command に細分化します。そしてそれぞれに適切な preCondition, postCondition, nextState を定義します。

```scala
  case class InsertWithNewUser(user: User) extends SuccessCommand {

    override type Result = Try[Unit]

    override def preCondition(state: Map[Long, User]): Boolean = !state.contains(user.id)

    override def run(sut: UserRepositoryOnJdbc): Result = sut.insert(user)

    override def postCondition(state: Map[Long, User], result: Try[Unit]): Prop =
      // postCondition cannot see the result of nextState
      result.isSuccess // && state.contains(user.id)

    override def nextState(state: Map[Long, User]): Map[Long, User] = state + (user.id -> user)

  }

  case class InsertWithExistingUser(user: User) extends SuccessCommand {

    override type Result = Try[Unit]

    override def preCondition(state: Map[Long, User]): Boolean = state.contains(user.id)

    override def run(sut: UserRepositoryOnJdbc): Result = sut.insert(user)

    override def postCondition(state: Map[Long, User], result: Try[Unit]): Prop = {
      val resultCheck = result.fold(
        {
          case _: UserAlreadyExistsException =>
            Prop.passed
          case e =>
            Prop.falsified :| s"expect UserAlredayExistsException, but $e"
        }, { _ =>
          Prop.falsified :| "expect UserAlreadyExistsException"
        }
      )
      resultCheck && state.contains(user.id)
    }

    override def nextState(state: Map[Long, User]): Map[Long, User] = state

  }
```

既存の Insert Command と大きく違う点として SuccessCommand を実装することで、sut からの結果 (その型は type Result で定義される) を使用して postCondition が書けるようになっています。

postCondition は nextState よりも前に実行されるという部分が少しハマりどころかもしれません。最初 InsertWithNewUser の postCondition で新しく User が追加されたことをチェックしようとしたのですが、実際は state の更新が nextState で行われるのでそれは不可能だということに気付きました。契約プログラミングの事後条件的な感じで考えればいいかなーと思っていたのですが、そう考えるとここは微妙にずれてしまいますね。

これでテストを実行すると 2 種類の Insert が実行されていることが観察できます (`---` 間が一つの Command 列。state はこの間で初期化されているはず)。

```
---
---
---
InsertWithNewUser(User(249730458178956903,mat9drocCpcw26Fe8ccjthZzcmxmfj3akfo3xrgbbpqnwtluiothsovkNcuiyguqlfhzu6wczikzkvvhcuqEQrnypmMznjrzbfs2upfjpciezqsvcixelztkxezR1iY0aFmftwghaiqfviCbgq5V4v59fy5wmxrAypSbkxXgafrcphXhz,198))
InsertWithExistingUser(User(249730458178956903,mat9drocCpcw26Fe8ccjthZzcmxmfj3akfo3xrgbbpqnwtluiothsovkNcuiyguqlfhzu6wczikzkvvhcuqEQrnypmMznjrzbfs2upfjpciezqsvcixelztkxezR1iY0aFmftwghaiqfviCbgq5V4v59fy5wmxrAypSbkxXgafrcphXhz,198))
---
FindById(756168933191045415)
FindById(5847350816276567841)
InsertWithNewUser(User(1641040789162680695,gJb7tZmfokhWwh8Kefvkqey0yxyol9sakofgokE4edpcnuggdgoObkkwHjlmezywyWbqfqqt2xvkiwJiLxwuPuubIFeTBKYiBdznnm9tkaenlVktglttsNsdwmcbhyhktjflmWrr2ljw1zek3z60dxeMcfhqwur2hSbqvlurresmi6Uvphwgyxa,175))
---
InsertWithNewUser(User(5951405727369649509,jejw22ueEdwp1qxCszp9oe73Ip76cyuysmameqjinime96cl9xorwdtpm3kwjQbiono8wothjj,96))
---
InsertWithNewUser(User(7515321593212396447,dja9izVhgqbmvnsfdh3n6clc32zanvEdbJn9mYhrlnNrmfwyj6xa4Gypdnzp3x7EkwabllxLRyubcxd0geggywtdtyp7SomyPdyyflExittoLpQCngpvyedlgzvBxb3mr111vlynywbvaehviz9v,73))
InsertWithNewUser(User(595862198723207,wor7nqxrDe1iZijrfsmpgDxdLyqssxtbgj0bcdoaowh06zkdbhtcbuNu54ipkk01lhxxujjkjh9eepx,217))
Update(User(4864357582705815611,9myhr2fxbmdgwcrqackajkePmmnrrtuewny3o5eBkzxgdoehZmMevzrhftjtmXfUhdbvivyZwtph9lgwcfcboauvvkcgsYymohd3gxwdkqmslto1zhpJjoztbys8l,16))
---
FindById(9127233414655442518)
Update(User(7362942754036969012,umpbewvlghrqmtvpjtsavc5zcuksrqdtun4jrmfI9f47p,130))
Update(User(5118043425662858512,EhbbMbetehb0iyqezr9cmuezapt7aaHLzfhulNkfjmsrnmilOzquftrlYywFYbxxv3,38))
Update(User(8505299442142655235,0ryk6nbkjqqydsbJnsengOyelfsblxbupqwz4ddlnyx2plezNvh8mejahdgwcsokczvgk5lykBlcn36d7xqqqy2t6r2fFgmz6Ai2lFdt4tmIxs4vwtgjasw6htncfg,31))
FindById(3798713816869640929)
...
```

他の find, update, delete についても既にユーザが存在するか否かで Command を分けるようにしたものを [UserRepositoryCheck.scala][5] に置いています。

### 4. 感想

- 自分の中で Property Based Testing でテストできる対象が広がった気がする
- ただ一回の実行でけっこう時間かかる
  - 外部システムを使用するので割とどうしようもない
  - 共通で使用する CRUDRepository のようなものがあってそれに対して実装するとか、複雑なロジックが絡む部分にだけ実装するとかが良さそう
- `org.scalacheck.commands.Commands.property()` のパラメータには実行時のスレッド数を指定できるようで、これはマルチスレッドでの Sut の挙動を見るのに使えるよう
- 生成されるコマンドの割合とかパッと見れる仕組みがあると嬉しそう
  - ScalaCheck でいう collect
- [ScalaCheck の User Guide][2] ではテスト失敗時 Command 列が shrink するぜ! ということを言っているけれど、うまく機能していない?
  - 追ってみると `org.scalacheck.commands.Commands` 内で定義している `Shrink[Actions]` ではない別の `Shrink` が implicit に解決されてしまっているのでは?

[1]: https://github.com/tiqwab/example/blob/master/scalacheck-sample/build.sbt
[2]: https://github.com/rickynils/scalacheck/blob/master/doc/UserGuide.md#stateful-testing
[3]: https://propertesting.com/book_stateful_properties.html
[4]: https://propertesting.com/book_case_study_stateful_properties_with_a_bookstore.html
[5]: https://github.com/tiqwab/example/blob/master/scalacheck-sample/src/test/scala/com/tiqwab/example/scalacheck/UserRepositoryCheck.scala
[6]: https://propertesting.com/index.html
