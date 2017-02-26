---
layout: post
title: Pythonにおける同値性比較の実装
tags: "python"
comments: true
---

同値関係というのはオブジェクトの性質として基本的なものですが、同時に意外と厳密な定義の難しいものでもあると思います。
最近 Python を扱っていて 同値性を定義する `__eq__` であったり オブジェクトのhash値を計算する `__hash__` であったりの実装で悩むところがあったので、少しそのうまい定義の仕方について考えてみることにしました。

1. [equality の実装](#anchor1)
2. [hashable class の実装](#anchor2)

### 環境

- OS: Arch Linux (linux kernel: 4.9.3)
- Python: 3.6.0

本記事の内容は Python の version に依存した内容になっているかもしれません。
主に 3.6.0 を使用していますが、 他に 2.7.5 でも ( version の差異による部分を書き直して) 記載スクリプトが実行できることを確認しています。

---

<a id="anchor1"></a>

### 1. equality の実装

Python では `__eq__` メソッドを実装することで対象クラスにおける同値性を定義します。
ユーザ定義クラスの `__eq__` デフォルト実装では `id` 関数で返されるオブジェクト間で一意な識別値で同値を判断しますが、大抵の場合これでは意味のある比較にならないため、クラスに合わせて `__eq__` を定義し直す必要があります。

とはいえクラス毎に地道に `__eq__` を実装するのも辛いので、現実的な妥協をしつつ汎用的な `__eq__` の定義の仕方をここで整理しておきたいなと思います。
現実的な、というのは厳密にあらゆる場合に対応できる形を考えるのが結構難しく、それならば自身の用途に合う形で妥協すればいいやということなのですが、具体的には同値の判定において以下の仮定を置いています。

- 2つのオブジェクト x,y の持つ各fieldが同値ならば、 `x == y` とする
  - 一部のfieldを同値の判定から除きたいならば、手作業で書くのはやむを得ない
- x,y の型は同じか、少なくとも継承関係になくてはならない
  - ただし継承関係を許す場合、対称律、推移律を崩さないようにする (後述)

結果からいえば、目的とするクラスを immutable としてよいか否かで以下で記述する2通りかなとなりました。

#### 1-1. Immutable class に対する equality 実装

Immutable class に `__eq__` を手軽に実装するためにここでは namedtuple を使用します。
アイデアは[公式のドキュメント][4]から持ってきているので詳細に関してはそちらにお任せします。

```python
from collections import namedtuple
import math

class Point(namedtuple('Point', ['x', 'y'])):
    '''
    Sample to implement an immutable class
    '''
    # avoid using dict to store fiedls as namedtuple does
    __slots__ = ()

    def distance(self, other):
        return math.sqrt((self.x - other.x) ** 2 + (self.y - other.y) ** 2)

if __name__ == '__main__':
    # Immutable Point class has two fields, x and y
    p1 = Point(1, 2)
    p2 = Point(4, 6)
    print(p1.x)             # 1
    print(p1.y)             # 2
    print(p1)               # Point(x=1, y=2)
    print(p1._replace(y=1)) # Point(x=1, y=1)
    print(p1.distance(p2))  # 5.0

    # check immutability
    try:
        p1.x = 3
    except AttributeError as e:
        print(e) # can't set attribute
    try:
        p1.z = 3
    except AttributeError as e:
        print(e) # 'Point' object has no attribute 'z'

    # check equality
    print(p1 == p1) # True
    print(p1 == p2) # False
    print(p2 == p1) # False
    print(p1 == Point(1, 2)) # True
    print(Point(1, 2) == p1) # True
```

上で示すように namedtuple で作成したクラスを継承することで、通常のユーザ定義クラスと同じ感覚で immutable class が作成できるのが嬉しいところです。
ただし新しいfieldを追加して継承することは推奨されておらず、新規にクラスを作り直した方がよいとされています。
それじゃちょっと...という場合、上記ドキュメントが参照している[こちら][5]が役に立つかもしれません。

#### 1-2. Mutable class に対する equality 実装

上では namedtuple を使用して immutable class を用意しましたが、 Python では普通にやればユーザ定義クラスはmutableなものになります。
mutable classの equality を考えること自体うまくない考えな気もしますが、とはいえその場限りの何かを作っているときにこういうやり方もある、というスタンスで何か押さえておいても良いかなと思っています。

ということで、これも[公式ドキュメント][2]の `__eq__` の記述を確認しながら、最終的には[stack overflow][1]での同様の議論を参考にして整理しました。内容としてはオブジェクトの 全field の比較として `__dict__` の比較を行うということになります。

```python
class Foo:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __eq__(self, other):
        if isinstance(other, Foo):
            return self.__dict__ == other.__dict__
        return False

if __name__ == '__main__':
    # check equality
    assert Foo('Alice', 10) == Foo('Alice', 10)
    assert not Foo('Alice', 10) == Foo('Bob', 10)
    assert not Foo('Bob', 10) == Foo('Alice', 10)
    assert not Foo('Alice', 10) == Foo('Alice', 15)
    assert not Foo('Alice', 15) == Foo('Alice', 10)

    # `__ne__` seems to be defined as the inverse of `__eq__`
    assert not Foo('Alice', 10) != Foo('Alice', 10)
    assert Foo('Bob', 10) != Foo('Alice', 10)
    assert Foo('Alice', 10) != Foo('Alice', 15)
    assert Foo('Alice', 15) != Foo('Alice', 10)
```

この場合、継承に関しても想定通りの動きで同値性の判断をしてくれるはずです。

- 親子で同値関係が定義でき、かつ各fieldの比較でよいのならば、 親の `__eq__` 実装をそのまま使用できる
- 親子で同値とみなしたくなければ、subclass側で `__eq__` を同様に再実装すれば必ず False となる

```python
class BarParentEq(Foo):
    def __init__(self, name, age):
        super().__init__(name, age)

class BarSelfEq(Foo):
    def __init__(self, name, age):
        super().__init__(name, age)

    def __eq__(self, other):
        if isinstance(other, BarSelfEq):
            return self.__dict__ == other.__dict__
        return False

if __name__ == '__main__':
    # check equality of extended classes using the parent's eq
    assert BarParentEq('Alice', 10) == BarParentEq('Alice', 10)
    assert Foo('Alice', 10) == BarParentEq('Alice', 10)
    assert BarParentEq('Alice', 10) == Foo('Alice', 10)

    # check equality of extended classes using their own eq.
    assert BarSelfEq('Alice', 10) == BarSelfEq('Alice', 10)
    assert Foo('Alice', 10) != BarSelfEq('Alice', 10)
    assert BarSelfEq('Alice', 10) != Foo('Alice', 10)
```

実は最初この結果を見たとき少し驚いたのですが、それは Python での同値性の比較順序が初めて見るようなものだからでした。一部は[公式ドキュメント][2]に書いてある内容ですが、Python では `x == y` を評価するために `__eq__` を以下の順序で実行するようです。

```
x == y の比較

1. `x.__eq__(y)` を実行する
   `NotImplemented` が返された場合 2 へ、そうでなければその値が結果
2. `y.__eq__(x)` を実行する
   `NotImplemented` が返された場合 3 へ、そうでなければその値が結果
3. x と y の同一性を比較し、それを結果とする

ただし y が x の subclass である場合、 2 と 1 の順序が逆転する
```

最後の一文がポイントで、つまり「 lhs だろうが rhs だろうが継承関係にあれば subclass の `__eq__` 実装を優先する」ということになります。この点に関してはドキュメントに明記されています。

この狙いまではドキュメントには記述されていませんが、恐らくオブジェクト指向言語において継承時に出てくる同値性比較の問題にうまく対応するためなんじゃないかなと思います。
例えば Java におけるオブジェクトの同値性比較は `equals` メソッドの override で行いますが、継承を考慮して厳密に定義しようとすると結構複雑になることが知られています ([参考][3])。
subclass の同値性比較メソッドを優先して使用することで、継承によって同値関係の対象律や推移律をうっかり崩すことが無くなるので、そうしたことを狙っているのかなーと思っています。こうしたところが Python らしい、ということなのかもしれません。

ちなみに Java の場合、`equals` や `hashCode` を半自動的に生成するIDEなりライブラリなりが揃っており、Python でもそれぐらい楽をしたいなというのが元々こうした内容を調べ始めた動機でした。

あ、あと忘れていましたが 少なくとも最新の Python 処理系では `x != y` は `__eq__` の否定と定義されているようなので、 `__eq__` 実装時に 必ずしも `__ne__` を実装し直す必要はないようです。

<a id="anchor2"></a>

### 2. hashable class の実装

あるクラスにおける同値性を定義し直した場合、そのクラスを hashable にしたいのであれば、必ず `__hash__` も定義し直す必要がでてきます。これは Python に限った話ではなく hash の性質「2つのオブジェクトが同値ならばそのhash値は同じである」を満たすようにするためです (ちなみにこの逆は常に成り立つとは限らず、いわゆる衝突)。

オブジェクトに対するhash値の計算自体はオブジェクトが mutable でも immutable でも定義できると思うのですが、mutableなクラスに対してhash値を計算するメソッドを定義し直すことは混乱の元になる場合が多いです (よくある場合として、[Pythonの公式ドキュメント][6]でも触れられていますが dict のキーに対する使用時が挙げられます) 。

そのため Python では組み込み型の中でもmutableな set や list に関してはhash値を計算することができません。もしこうしたデータ型を dict のキーにしたい場合、代わりに frozenset や tuple を使用することを考えます。

```
>>> hash([1,2,3])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
>>> hash({1,2,3})
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'set'

>>> hash((1,2,3))
2528502973977326415
>>> hash(frozenset({1,2,3}))
-7699079583225461316
```

ユーザ定義クラスに関しても同じ方針で immutable class に対してのhash値計算を考えればよいと思っているのですが、ただ `__eq__` で考えたとき同様、なんちゃって immutable として扱うクラスを定義したいといった場合も考えて、それぞれである程度汎用的に使える `__hash__` の実装を整理してみます。

#### 2-1. Immutable class に対する hash 実装

上で immutable class の実装を考えた際に `namedtuple` を使用したのですが、これは元々が tuple でhashableなために、何もしなくても `__hash__` が正しく定義されています。

```
>>> p1 = Point(1,2)
>>> p2 = Point(4,6)
>>> hash(p1)
3713081631934410656
>>> hash(p2)
3713084879519153381
>>> hash(Point(1,2))
3713081631934410656
>>> hash(Point(4,6))
3713084879519153381
```

ただし元の tuple 同様、fieldの中にhashableではないものが含まれれば、hash値を計算することはできません。
個人的にはよくうっかりして見るエラーです...

```
>>> hash(Point([],[]))
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
```

#### 2-2. Mutable class に対する hash 実装

通常のユーザ定義クラスで `__hash__` を実装する場合、 `__eq__` のときと同様「各fieldを元にした計算」でよければ、下記のように実装するのがよいかなと思います (参考: [公式ドキュメント][6]および[stack overflow][1]) 。

```python
class Foo:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __eq__(self, other):
        if isinstance(other, Foo):
            return self.__dict__ == other.__dict__
        return False

    def __hash__(self):
        return hash(tuple(sorted(self.__dict__.items())))

if __name__ == '__main__':
    # check hash
    assert hash(Foo('Alice', 10)) == hash(Foo('Alice', 10))
    assert hash(Foo('Alice', 10)) != hash(Foo('Bob', 10))
    assert hash(Foo('Bob', 10)) != hash(Foo('Alice', 10))
    assert hash(Foo('Alice', 10)) != hash(Foo('Alice', 15))
    assert hash(Foo('Alice', 15)) != hash(Foo('Alice', 10))
```

ただこの場合は `__eq__` とは違って継承時に必ずしも期待した動きにならない可能性があります。 field が同値ならば親子でのhash値は同じでよい、ということであれば問題無いのですが、そうではない場合はこの実装ではカバーできません。

そうした場合も考えた実装が必要であれば、Python の例ではありませんが Java ライブラリである[lombokにおけるhashCode自動生成][7]なんかが参考になるかなと思います。

### Summary

- Python では namedtuple を使用して immutable class を作成できる
  - `__eq__` や `__hash__` が正しく定義されている
- 通常のユーザ定義クラスで簡単に `__eq__` や `__hash__` を定義する場合は、 `__dict__` を活用するのが一つの手
  - ただし mutable class として扱うなら `__hash__` は定義するべきではない
  - 継承時に想定した動きをするかどうかは注意が必要

### 参考

- [How to Write Equality Method in Java][3]
  - Java における `equals` メソッドの実装について丁寧に解説されています
- [namedtuple - Python 3.6.0 doc][4]
  - namedtuple に関する公式ドキュメント
- [eq - Python 3.6.0 doc][2]
  - `__eq__` 実装に関する公式ドキュメント
- [hash - Python 3.6.0 doc][6]
  - `__hash__` 実装に関する公式ドキュメント

[1]: http://stackoverflow.com/questions/390250/elegant-ways-to-support-equivalence-equality-in-python-classes
[2]: https://docs.python.org/3.6/reference/datamodel.html#object.__eq__
[3]: http://www.artima.com/lejava/articles/equality.html
[4]: https://docs.python.org/3/library/collections.html#collections.namedtuple
[5]: https://code.activestate.com/recipes/577629-namedtupleabc-abstract-base-class-mix-in-for-named/
[6]: https://docs.python.org/3.6/reference/datamodel.html#object.__hash__
[7]: https://projectlombok.org/features/EqualsAndHashCode.html
