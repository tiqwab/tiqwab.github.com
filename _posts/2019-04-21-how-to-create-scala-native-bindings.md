---
layout: post
title: scala-native の libyaml binding を作成した
tags: "libyaml, scala-native, c, scala"
comments: true
---

[scala-native][1] に慣れる一環として [libyaml][2] の binding, [tiqwab/scala-native-libyaml][3] を作成しました。binding とは [Native code interoperability][4] から引用すると

> an interop layer that makes it easy to interact with foreign native code. This includes C and other languages that can expose APIs via C ABI (e.g. C++, D, Rust etc.)

というもので、例えば libc で提供される構造体や関数等を scala-native から扱えるようにするためのものです。

libyaml というのは C で書かれた YAML を扱うためのライブラリです。YAML という親しみのある対象を扱い、かつ手頃の大きさであるライブラリなので binding を書く題材として丁度良さそうと思い選びました。

---

### 1. binding の書き方

あるライブラリの binding を作成するとは、端的にはそのライブラリが提供するヘッダファイルを scala-native で再定義することです。[Finding the right signature][7] でまとめられているように、scala-native では C の型に対応した型が用意されています。それを使用してヘッダファイルで定義される構造体や関数を書き直していく、また関数についてはその実装を `extern` にする (C の関数の宣言にあたる)、という感じです。

例えば以下のようなヘッダファイルがあった場合、

```c
struct foo {
    int x;
    int y;
};

typedef enum foo_type_e {
    A,
    B,
    C
} foo_type_t;

void call_foo(struct foo *foo, foo_type_t fe);
```

binding の一例は以下のようになります。

```scala
package example

import scala.scalanative._
import scala.scalanative.native._

@native.extern
object simple {
  // Define foo struct
  type foo = CStruct2[CInt, CInt]

  // Define enum_foo_type enum
  type enum_foo_type_t = FooType
  type FooType = CUnsignedInt
  object FooType {
    final val A: FooType = 0.toUInt
    final val B: FooType = 1.toUInt
    final val C: FooType = 2.toUInt
  }

  def call_foo(foo: native.Ptr[foo], ft: enum_foo_type_t): Unit = extern
}
```

やることと上の例を見ると、binding 作成というのは機械的な手順に落とし込めそうに感じます。実際 [scala-native-bindgen][6] というツールが存在し、ヘッダファイルを入力として binding コードを出力することができます。scala-native-bindgen で上と同じヘッダファイルから生成した binding は以下のようになります。

```scala
package example

import scala.scalanative._
import scala.scalanative.native._

@native.extern
object simple {
  type enum_foo_type_e = native.CUnsignedInt
  object enum_foo_type_e {
    final val A: enum_foo_type_e = 0.toUInt
    final val B: enum_foo_type_e = 1.toUInt
    final val C: enum_foo_type_e = 2.toUInt
  }

  type struct_foo = native.CStruct2[native.CInt, native.CInt]
  type foo_type_t = enum_foo_type_e
  def call_foo(foo: native.Ptr[struct_foo], fe: enum_foo_type_e): Unit = native.extern

  object implicits {
    implicit class struct_foo_ops(val p: native.Ptr[struct_foo]) extends AnyVal {
      def x: native.CInt = !p._1
      def x_=(value: native.CInt): Unit = !p._1 = value
      def y: native.CInt = !p._2
      def y_=(value: native.CInt): Unit = !p._2 = value
    }
  }

  object struct_foo {
    import implicits._
    def apply()(implicit z: native.Zone): native.Ptr[struct_foo] = native.alloc[struct_foo]
    def apply(x: native.CInt, y: native.CInt)(implicit z: native.Zone): native.Ptr[struct_foo] = {
      val ptr = native.alloc[struct_foo]
      ptr.x = x
      ptr.y = y
      ptr
    }
  }
}
```

はじめに挙げた例と比較すると、一番の違いは implicit class の定義が存在することです。これの利点は構造体を使用する scala-native コードが書きやすく読みやすくなることです。例えば scala-native で構造体のメンバへの代入を行う場合 `!foo._1 = 1` のような不慣れな見た目になるのですが、implicit class の定義により同じコードを `foo.x = 1` と書くことができます。

ということで binding を作成する場合、scala-native-bindgen で生成するか、あるいはそれを参考にして書いていくというのがいいかと思います。後者の選択肢必要か？ という感じですが、現状の実装だとそのまま生成されたコードが使えない (コンパイルできない) 場合もあるためです。自分が遭遇した悩ましい点として implicit 定義が重複してしまうというのがあります。

### 2. 異なる構造体で implicit 定義が重複することがある

例えば以下のようなメンバの数と型は同じ 2 つの構造体を考えます。

```c
struct foo {
    int x;
    int y;
};

struct bar {
    int x;
    int y;
};
```

これを scala-native-bindgen で処理すると以下のようなコードが生成されます。

```scala
package example

import scala.scalanative._
import scala.scalanative.native._

object same_structs {
  type struct_foo = native.CStruct2[native.CInt, native.CInt]
  type struct_bar = native.CStruct2[native.CInt, native.CInt]

  object implicits {
    implicit class struct_foo_ops(val p: native.Ptr[struct_foo]) extends AnyVal {
      def x: native.CInt = !p._1
      def x_=(value: native.CInt): Unit = !p._1 = value
      def y: native.CInt = !p._2
      def y_=(value: native.CInt): Unit = !p._2 = value
    }

    implicit class struct_bar_ops(val p: native.Ptr[struct_bar]) extends AnyVal {
      def x: native.CInt = !p._1
      def x_=(value: native.CInt): Unit = !p._1 = value
      def y: native.CInt = !p._2
      def y_=(value: native.CInt): Unit = !p._2 = value
    }
  }

  object struct_foo {
    import implicits._
    def apply()(implicit z: native.Zone): native.Ptr[struct_foo] = native.alloc[struct_foo]
    def apply(x: native.CInt, y: native.CInt)(implicit z: native.Zone): native.Ptr[struct_foo] = {
      val ptr = native.alloc[struct_foo]
      ptr.x = x
      ptr.y = y
      ptr
    }
  }

  object struct_bar {
    import implicits._
    def apply()(implicit z: native.Zone): native.Ptr[struct_bar] = native.alloc[struct_bar]
    def apply(x: native.CInt, y: native.CInt)(implicit z: native.Zone): native.Ptr[struct_bar] = {
      val ptr = native.alloc[struct_bar]
      ptr.x = x
      ptr.y = y
      ptr
    }
  }
}
```

このコードで定義される implicit class を使用してみると implicit が一意に解決できずコンパイルエラーとなります。

```
[error] Note that implicit conversions are not applicable because they are ambiguous:
[error]  both method struct_foo_ops in object implicits of type (p: scala.scalanative.native.Ptr[example.same_structs.struct_foo])example.same_structs.implicits.struct_foo_ops
[error]  and method struct_bar_ops in object implicits of type (p: scala.scalanative.native.Ptr[example.same_structs.struct_bar])example.same_structs.implicits.struct_bar_ops
[error]  are possible conversion functions from ptr.type to ?{def x: ?}
[error]       ptr.x = x
[error]       ^
```

libyaml の binding 作成ではワークアラウンドとして scalaz 等が持つ [Tagged Type][8] を利用してみました。

```scala
trait Tag {
  type Tagged[T] = { type Tag = T }
  type @@[A, T] = A with Tagged[T]

  def tagged[A, T](a: A): A @@ T = a.asInstanceOf[A @@ T]
}
```

Tagged Type を利用して binding を作成すると以下のようになります。

```scala
package example

import scala.scalanative.native._

object same_structs2 extends Tag {

  // tag 用
  private[example] object tag {
    sealed trait Foo
    sealed trait Bar
  }

  // 元の型 CStruct2[CInt, CInt] に tag を付与
  type struct_foo = CStruct2[CInt, CInt] @@ tag.Foo
  type struct_bar = CStruct2[CInt, CInt] @@ tag.Bar

  object implicits {
    implicit class struct_foo_ops(val p: Ptr[struct_foo]) extends AnyVal {
      // Original の型に cast しないといけない
      type Original = CStruct2[CInt, CInt]
      def x: CInt = !p.cast[Ptr[Original]]._1
      def x_=(value: CInt): Unit =
        !p.cast[Ptr[Original]]._1 = value
      def y: CInt = !p.cast[Ptr[Original]]._2
      def y_=(value: CInt): Unit =
        !p.cast[Ptr[Original]]._2 = value
    }

    implicit class struct_bar_ops(val p: Ptr[struct_bar]) extends AnyVal {
      // Original の型に cast しないといけない
      type Original = CStruct2[CInt, CInt]
      def x: CInt = !p.cast[Ptr[Original]]._1
      def x_=(value: CInt): Unit =
        !p.cast[Ptr[Original]]._1 = value
      def y: CInt = !p.cast[Ptr[Original]]._2
      def y_=(value: CInt): Unit =
        !p.cast[Ptr[Original]]._2 = value
    }
  }

  object struct_foo {
    import implicits._
    def apply()(implicit z: Zone): Ptr[struct_foo] =
      alloc[struct_foo]
    def apply(x: CInt, y: CInt)(implicit z: Zone): Ptr[struct_foo] = {
      val ptr = alloc[struct_foo]
      ptr.x = x
      ptr.y = y
      ptr
    }
  }

  object struct_bar {
    import implicits._
    def apply()(implicit z: Zone): Ptr[struct_bar] =
      alloc[struct_bar]
    def apply(x: CInt, y: CInt)(implicit z: Zone): Ptr[struct_bar] = {
      val ptr = alloc[struct_bar]
      ptr.x = x
      ptr.y = y
      ptr
    }
  }
}
```

無事以下のようなコードもコンパイルが通り実行できます。

```scala
package example

import same_structs2._
import same_structs2.implicits._

import scala.scalanative.native._

object SameStructMain {
  def main(args: Array[String]): Unit = {
    Zone { implicit z =>
      val foo = struct_foo()
      val bar = struct_bar()

      foo.x = 1
      foo.y = 2
      bar.x = 1
      bar.y = 2

      stdio.printf(toCString("foo{x=%d, y=%d}\n"), foo.x, foo.y)
      stdio.printf(toCString("bar{x=%d, y=%d}\n"), bar.x, bar.y)
    }
  }
}
```

```
sbt> run
...
foo{x=1, y=2}
bar{x=1, y=2}
```

この方法はコードを書く分には元と変わらない書き心地で良いかと思うのですが、構造体のメンバへのアクセスで毎回 cast をしないといけないのが気になります。

ただ確認した限りは tagged type や cast を駆使しても生成される LLVM IR には差が出なかったので、どうやら型安全でなくなりはしても実行時の性能には影響が無さそうです。確かに cpu や memory の世界からすれば型や cast なんて存在しないので納得な気がします。

### 3. 既存の binding 実装

参考になりそうな既存の binding たち (がまとめられた場所) です。

- scala-native 自信がもつ binding
  - [for 標準 C ライブラリ][9]
  - [for POSIX][10]
- [awesome-scala-native][5]

[1]: https://github.com/scala-native/scala-native
[2]: https://github.com/yaml/libyaml
[3]: https://github.com/tiqwab/scala-native-libyaml
[4]: https://scala-native.readthedocs.io/en/v0.3.8/user/interop.html
[5]: https://github.com/tindzk/awesome-scala-native
[6]: https://github.com/scala-native/scala-native-bindgen
[7]: https://scala-native.readthedocs.io/en/v0.3.8/user/interop.html#finding-the-right-signature
[8]: https://timperrett.com/2012/06/15/unboxed-new-types-within-scalaz7/
[9]: https://github.com/scala-native/scala-native/tree/master/clib/src/main/scala/scala/scalanative/libc
[10]: https://github.com/scala-native/scala-native/tree/master/posixlib/src/main/scala/scala/scalanative/posix
