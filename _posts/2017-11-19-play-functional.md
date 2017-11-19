---
layout: post
title: play-json の inmap とか
tags: "functor, play-json, scala"
comments: true
---

[play-json][1] を触り始めたときに play-json の tips を紹介されている [この記事][2] を読んで 「inmap, contramap... 便利だけど何だんだこいつら」 と感じた記憶を最近思いだしたので、調べた内容整理しておきます。

### play-json おさらい

play-json で `inmap` がどのようなときに有用かというと、単純な例としては以下のような ID を表現するモデル (`UserId`) を扱うときというのがあります。

```scala
// User モデルの定義
> case class UserId(value: Long)
> case class User(id: UserId, name: String)

// 上の User を以下のような json で表現したい
> val json = Json.obj("id" -> "1", "name" -> "Alice")

// Format の定義
// inmap を使うことで JsString と UserId の相互変換を定義している
// String => Long が失敗し得るので丁寧にやるならば例えば Reads を単独に定義して JsResult で失敗を表現する
> implicit val userIdFormat: Format[UserId] =
>   implicitly[Format[String]].inmap(x => UserId(x.toLong), _.value.toString)
> implicit val userFomrat: Format[User] = Json.format[User]

// 上記の Format 定義により json から User インスタンスが作成できる
> Json.fromJson[User](json)
res7: play.api.libs.json.JsResult[User] = JsSuccess(User(UserId(1),Alice),)
// もちろん逆も
> Json.toJson(res7.get)
res8: play.api.libs.json.JsValue = {"id":"1","name":"Alice"}
```

記事にある通り `Format` に対する `inmap` のようなことをやりたい場合、 `Reads` には `map` (あるいは `fmap`), `Writes` には `contramap` が存在します。

### play-json の functional パッケージ

play-json における `fmap`, `inmap`, `contramap` は `play.api.libs.functional` パッケージで以下のように定義されています。

```scala
class FunctorOps[M[_], A](ma: M[A])(implicit fu: Functor[M]) {
  def fmap[B](f: A => B): M[B] = fu.fmap(ma, f)
}

class ContravariantFunctorOps[M[_], A](ma: M[A])(implicit fu: ContravariantFunctor[M]) {
  def contramap[B](f: B => A): M[B] = fu.contramap(ma, f)
}

class InvariantFunctorOps[M[_], A](ma: M[A])(implicit fu: InvariantFunctor[M]) {
  def inmap[B](f: A => B, g: B => A): M[B] = fu.inmap(ma, f, g)
}
```

`Reads` には `Functor` が定義されています (`Reads` が `Applicative` であることは別途定義されている)。

```scala
  implicit def functorReads(implicit a: Applicative[Reads]) = new Functor[Reads] {
    def fmap[A, B](reads: Reads[A], f: A => B): Reads[B] = a.map(reads, f)
  }
```

`OWrites` には `ContravariantFunctor` が定義されています。

```scala
  implicit val contravariantfunctorOWrites: ContravariantFunctor[OWrites] =
    new ContravariantFunctor[OWrites] {
      def contramap[A, B](wa: OWrites[A], f: B => A): OWrites[B] =
        OWrites[B](b => wa.writes(f(b)))
    }
```

`OFormat` には `InvariantFunctor` が定義されています。

```scala
  implicit val invariantFunctorOFormat: InvariantFunctor[OFormat] =
    new InvariantFunctor[OFormat] {
      def inmap[A, B](fa: OFormat[A], f1: A => B, f2: B => A): OFormat[B] =
        OFormat[B]((js: JsValue) => fa.reads(js).map(f1), (b: B) => fa.writes(f2(b)))
    }
```

### Functor たち

- `inmap`, `contramap` といったものが Functor と関係ありそうなことがわかってきた
- ここらへんについて検索してみると [Of variance and functors][3] がまさにその内容の解説
- ちゃんと理解するなら圏論の話になるみたい

---

[1]: https://github.com/playframework/play-json
[2]: https://dev.classmethod.jp/server-side/scala/play-json-5-frequent-patterns/
[3]: https://typelevel.org/blog/2016/02/04/variance-and-functors.html
