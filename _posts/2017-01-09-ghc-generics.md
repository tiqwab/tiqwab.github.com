---
layout: post
title:  "GHC.Genericsを利用したgeneric programming"
tags: "generic programming, generics, haskell"
comments: true
---

Haskellとgeneric programmingに関してここしばらく調べており、[GHC.Generics][5] パッケージの提供する機能が便利だなと思ったのでまとめてみました。

実際のコードは[githubのリポジトリ][6]に配置しています。

1. [Generic programming](#anchor1)
2. [型変数](#anchor2)
3. [GHC.Generics](#anchor3)
4. [GenericなSerializableの実装](#anchor4)
5. [その他](#anchor5)

---

<a id="anchor1"></a>

### 1. Generic programming

[Generic programming - Wikipedia][3]では以下のように述べられています。

> Generic programming is a style of computer programming in which algorithms are written in terms of types
> to-be-specified-later that are then instantiated when needed for specific types provided as patameters.

この定義を少し噛み砕いて日本語にすれば、「データ型を抽象化して関数、メソッド、データ構造等を実装するプログラミング手法」と言えるかなと思います。

例えばJavaではgeneric programmingを扱う機能はGenericsと呼ばれており、一例として以下のよう型パラメータ`T`を持つクラス(`Container`)を書くことができます。

```java
import java.util.Arrays;

public class GenericsSample {
    public static void main(String[] args) {
        Container<String> containerStr = new Container<>("Alice");
        System.out.println(
                containerStr.add(new Container<>("Bob"))
        );

        Container<Integer> containerInt = new Container<>(1);
        System.out.println(
                containerInt.add(new Container<>(2).add(new Container<>(3)))
        );
    }
}

// `T` in `Container<T>` is the type parameter.
class Container<T> {
    private Object[] elems;

    public Container(T elem) {
        this.elems = new Object[]{elem};
    }

    private Container(T[] elems) {
        this.elems = elems;
    }

    public Container<T> add(Container<T> other) {
        final int newLen = this.length() + other.length();
        Object[] newElems = new Object[newLen];
        for (int i = 0; i < this.length(); i++) {
            newElems[i] = this.elems[i];
        }
        for (int i = 0; i < other.length(); i++) {
            newElems[i+this.length()] = other.elems[i];
        }
        return new Container<T>((T[])newElems);
    }

    public int length() {
        return this.elems.length;
    }

    @Override
    public String toString() {
        return String.format("Container {elems = [%s]}",
                             Arrays.stream(elems).filter(s -> s != null)
                             .map(s -> s.toString()).reduce((acc,s) -> acc + "," + s)
                             .orElse(""));
    }
}
```

`Container`クラスはいわゆるListを単純化したものを想定して定義しました。
`Container`はある型に対する任意長の配列を持ち、それに対するインタフェースを提供しますが、特定のデータ型に依存するような(特定のデータ型に対してしか定義できないような)メソッドは持ちません。
そのため`Container`クラスが扱う型を抽象化し、インスタンス生成時にパラメータとして具体的な型を受け取ることで、同一のクラス定義から`String`を扱う`Container`である`Container<String>`型、`Integer`を扱う`Container<Integer>`型のオブジェクトを生成しています。

ちなみに`main`関数の実行結果は以下のようになります。

```
Container {elems = [Alice,Bob]}
Container {elems = [1,2,3]}
```

<a id="anchor2"></a>

### 2. 型変数

上と同じことをHaskellでは型変数を使用して行うことができます。

```haskell
> add x xs = x:xs
> :t add
add :: a -> [a] -> [a]
> add 1 . add 2 $ [3]
[1,2,3]
> add 'a' . add 'b' $  ['c']
"abc"
```

型変数はHaskellが提供する素晴らしい機能ですが、ときにはより抽象化した形で関数の実装を行いたいと思う場合があります。

こういった場合によく挙げられている例として`Serializable`という型クラスがあるので、ここでもそれを使用しようと思います。`Serializable`型クラスのインスタンスとなったデータ型ではbit列との相互変換を行えることが保証されます。

```haskell
data Bit = I | O
    deriving (Show, Eq)

class Serializable a where
    serialize :: a -> [Bit]
    deserialize :: [Bit] -> Maybe (a, [Bit]) -- (deserialized object, remain bits)
```

`Serializable`の簡単な例として`Fruit`データ型とその`Serializable`インスタンスを定義します。`Fruit`型は4種の単純な値コンストラクタから定義されるデータ型であるため、シリアル化された形式としては`1`, `01`, `001`, `0001`という形としました。

```haskell
data Fruit = Apple | Banana | Grape | Orange
    deriving (Show, Eq)

instance Serializable Fruit where
    serialize Apple  = [I]
    serialize Banana = [O,I]
    serialize Grape  = [O,O,I]
    serialize Orange = [O,O,O,I]
    deserialize (I:xs)       = Just (Apple, xs)
    deserialize (O:I:xs)     = Just (Banana, xs)
    deserialize (O:O:I:xs)   = Just (Grape, xs)
    deserialize (O:O:O:I:xs) = Just (Orange, xs)
    deserialize _            = Nothing
```

```
> serialize Apple
[I]
> serialize Banana
[O,I]
> deserialize [I] :: Maybe (Fruit, [Bit])
Just (Apple,[])
> deserialize [O,I] :: Maybe (Fruit, [Bit])
Just (Banana,[])
```

もう少し複雑な例としてTree構造のデータに対しても`Serializable`を定義してみます。

```haskell
{-# LANGUAGE ScopedTypeVariables #-}

data Tree a = Node (Tree a) (Tree a) | Leaf a
    deriving (Show, Eq)

instance (Serializable a) => Serializable (Tree a) where
    serialize (Node left right) = [I] ++ serialize left ++ serialize right
    serialize (Leaf a1)         = [O] ++ serialize a1
    deserialize (I:xs) = do (l, remain1) <- deserialize xs
                            (r, remain2) <- deserialize remain1
                            return (Node l r, remain2)
    deserialize (O:xs) = do (x, remain) <- deserialize xs :: Maybe (a, [Bit])
                            return (Leaf x, remain)
```

```
> let tree1 = Node (Node (Leaf Apple) (Leaf Banana)) (Node (Node (Node (Leaf Grape) (Leaf Orange)) (Leaf Apple)) (Leaf Banana))
> serialize tree1
[I,I,O,I,O,O,I,I,I,I,O,O,O,I,O,O,O,O,I,O,I,O,O,I]
> deserialize . serialize $ tree1 :: Maybe (Tree Fruit, [Bit])
Just (Node (Node (Leaf Apple) (Leaf Banana)) (Node (Node (Node (Leaf Grape) (Leaf Orange)) (Leaf Apple)) (Leaf Banana)),[])
```

以上のように与えられたデータ型に対して`Serializable`のインスタンスを定義することが可能ですが、よく見ると`Serializable`インスタンスは一定のルールに従えば機械的に定義できそうだということがわかります。

`data Foo = A | B` という直和型に対してはそれぞれの値コンストラクタで違うbit列を対応させればよく、一例として上記の`Fruit`型のように左の値コンストラクタから順番に`I`, `OI`, `OOI`, ... のように割り振ることができます。

また`Node (Tree a) (Tree a)`のような直積型`data Bar = Bar B C`に関しては、B, Cをserializeしたものを左から順番に連結すればよく、deserializeもその逆だと思えばいけそうです。

このように具体的なデータ型を指定しなくても代数的に構成されたデータ型であれば一般化して関数を定義することができる、という場合に役に立つのが`GHC.Generics`パッケージとなります。

<a id="anchor3"></a>

### 3. GHC.Generics

`GHC.Generics`パッケージの主な役割は任意の代数的データ型を表現するための型クラス、データ型を提供することだと言えると思います。

`GHC.Generics`は主要なデータ型として以下の6つを定義しています。

```
data    V1        p                       -- lifted version of Empty
data    U1        p = U1                  -- lifted version of ()
data    (:+:) f g p = L1 (f p) | R1 (g p) -- lifted version of Either
data    (:*:) f g p = (f p) :*: (g p)     -- lifted version of (,)
newtype K1    i c p = K1 { unK1 :: c }    -- a container for a c
newtype M1  i t f p = M1 { unM1 :: f p }  -- a wrapper
```

また型クラス`Generic`を定義しています。ここでの`Rep a`が抽象化されたデータ型で、これは上の6つの型を使用して表現されます。

```
class Generic a where
  type family Rep a :: * -> *    -- representation of the data a
  from :: a -> Rep a x
  to :: Rep a x -> a
  {-# MINIMAL from, to #-}
    -- Defined in ‘HC.Generics
```

これらを使用したgeneric programmingの手順は、

前提:

- 代数的データ型一般に対して定義できる型クラス`X`がある (e.g. `Serializable`)
- 実際に型クラス`X`のinstanceとしたいユーザ定義のデータ型`a`がある (e.g. `Tree`)

1. ユーザ定義のデータ型`a`を`Generic`のinstanceとする
  - `Rep a`が型`a`を代数的な構造に基づいて抽象化した型となる
  - `Generic`のinstanceとなることで、`a`と`Rep a`との相互変換が可能なことが保証される
2. `Rep`を構成する上記6つのデータ型を`X`のinstanceとする

となります。`a`が`Rep a`に抽象化でき、その`Rep a`が`X`のinstanceとなるので、型`a`にserializeが定義可能になるという感じです。説明はイマイチかもしれませんが、実際に見てみればすぐに納得できると思います。

<a id="anchor4"></a>

### 4. GenericなSerializableの実装

この先では引き続き`Serializable`を例として、`GHC.Generics`によるgeneric programmingの流れをおさえてみたいと思います。

#### 4-1. 型クラス Genericのinstance実装

まずはserializeしたいユーザ定義型を`Generic`のinstanceとするのですが、実はこれはとても簡単でデータ型定義に`deriving (Generic)`を付け足し、`DeriveGeneric`GHC拡張を有効化するだけになります。

例えば`Tree`の場合ですと以下のようになります。

```haskell
{-# LANGUAGE DeriveGeneric #-}

data Tree a = Node (Tree a) (Tree a) | Leaf a
    deriving (Show, Eq, Generic)
```

この状態でGHCi起動時に`-ddump-deriv`オプションを渡すと、対象ソースのロード時に自動導出された`type Rep a`の定義を確認することができます。 (見づらいのでパッケージ名は省略しています)

```
> stack ghci --ghci-options -ddump-deriv
> :l src/Tiqwab/Generic/Tree.hs

type Rep (Tree a_a3oz) = D1 ('MetaData "Tree" "Serializable" "main" 'False)
                            (C1 ('MetaCons "Node" 'PrefixI 'False)
                                (S1 ('MetaSel 'Nothing
                                              'NoSourceUnpackedness
                                              'NoSourceStrictness
                                              'DecidedLazy)
                                    (Rec0 (Tree a_a3oz))
                                :*:
                                 S1 ('MetaSel 'Nothing
                                              'NoSourceUnpackedness
                                              'NoSourceStrictness
                                              'DecidedLazy)
                                    (Rec0 (Tree a_a3oz)))
                            :+:
                             C1 ('MetaCons "Leaf" 'PrefixI 'False)
                                (S1 ('MetaSel
                                     'Nothing
                                     'NoSourceUnpackedness
                                     'NoSourceStrictness
                                     'DecidedLazy)
                                    (Rec0 a_a3oz)))
```

ここでD1, C1, S1, Rec0といったデータ型は以下のように定義されています。

```
type D1 = M1 D
type C1 = M1 C
type S1 = M1 S

type Rec0 = K1 R
```

ということで表示された`Rep (Tree a)`が上記の6つのデータ型を使用して定義されていることがわかります (`M1 D`や`K1 R`のDやRに関してはそれぞれ`M1`, `K1`の種分けをしたいというだけで文字自体には意味があるわけではないようです)。

データ型の細かい内容も[hackage][5]がわかりやすいのでそちらを参照なのですが、最初の認識としては

- `D1`, `C1`, `S1`はデータ型、コンストラクタ、フィールドに対応し、それぞれメタデータを持つ
- `Rec0`は実際のフィールドのwrapperになっている
- 直和を表すために`:+:`
- 直積を表すために`:*:`

という程度を掴んでいれば、簡単な実装には問題無いのではと思います。

#### 4-2. 抽象化されたデータ型に対するinstance実装

一番はじめに`Serializable`型クラスの定義として以下を与えました。

```haskell
class Serializable a where
    serialize :: a -> [Bit]
    deserialize :: [Bit] -> Maybe (a, [Bit]) -- (deserialized object, remain bits)
```

今度は抽象化のためのデータ型(`Rep`を構成するデータ型)に対してシリアル化処理を定義するために`Serializable'`型クラスを作成します。説明では単純化のため`serialize`に対応する関数のみの実装とし、`deserialize`側は省略しています (コード上では両者実装しています)。

```haskell
class Serializable' f where
    serialize' :: f a -> [Bit]
```

もとの`Serializable`型クラスとは違い、パラメータ`f`のkindが`* -> *`であることに注意です。これは`V1`, `M1`といった抽象化のためのデータ型が共通して持つ最後のパラメータ`p`によるもののようですが、実際に処理の中で使用されるものではないのであまり気にしない方が良さそうです。

そして`Serializable'`のinstanceの実装は以下のようになります。

```haskell
{-# LANGUAGE TypeOperators       #-}

instance Serializable' V1 where
    serialize' x = undefined

instance Serializable' U1 where
    serialize' x = []

instance (Serializable' f, Serializable' g) => Serializable' (f :+: g) where
    serialize' (L1 x) = I : serialize' x
    serialize' (R1 x) = O : serialize' x

instance (Serializable' f, Serializable' g) => Serializable' (f :*: g) where
    serialize' (f :*: g) = serialize' f ++ serialize' g

instance (Serializable c) => Serializable' (K1 i c) where
    serialize' (K1 x) = serialize x

instance (Serializable' f) => Serializable' (M1 i t f) where
    serialize' (M1 x) = serialize' x
```

この実装の解説も[hackage][5]がとても親切なのですが、まとめると

- `V1`が渡されたら`undefined`とする
  - `data Foo`のような場合、`V1`が使用される
- `U1`に対しては空のlistを返す
  - `data Foo = Foo | Bar`のようなフィールドを持たない値コンストラクタに対し使用される
- `:+:`は`Serializable`の`Tree`実装で見たような直和の形式
- `:*:`は`Serializable`の`Tree`実装で見たような直積の形式
- `K1`はwrappingしているデータ型に対して`serialize`する
  - `serialize'`ではないことに注意
  - e.g. `Leaf a`の場合の`a`を`serialize`する
- `M1`はwrappingしているデータ型に対して`serialize'`する

のようになります。

次にもとの`Serializable`型クラスの定義を`Serializable'`を使用したものに変更します。

```haskell
{-# LANGUAGE DefaultSignatures   #-}

class Serializable a where
    serialize :: a -> [Bit]
    default serialize :: (Generic a, Serializable' (Rep a)) => a -> [Bit]
    serialize x = serialize' (from x)
```

`DefaultSignatures`GHC拡張により、型クラス定義内部にdefault実装を書くことができ、型が合えばユーザが`serialize`を実装しなくてもdefault実装(ここでは`serialize x = serialize' (from x)`)が使われるようになります。

これで`Generic`のinstanceとした任意のデータ型に対して`serialize`処理を行うことができるようになりました。最後に`DeriveAnyClass`GHC拡張を有効化し、ユーザ定義の型に対して`Serializable`を導出できるようにしておきます。

```haskell
{-# LANGUAGE DeriveAnyClass      #-}

data Fruit' = Apple' | Banana' | Grape' | Orange'
    deriving (Show, Eq, Generic, Serializable)

data Tree' a = Node' (Tree' a) (Tree' a) | Leaf' a
    deriving (Show, Eq, Generic, Serializable)
```

新規に定義した`Fruit'`、`Tree'`は明示的な`Serializable`のinstance定義は持ちませんが、GHC.Genericsのパワーにより以下のようにserialize, deserializeを行うことができます (下調べが足りなかったのですが、型を表現するときの直和の扱い方が違うために最初に手で定義した実装とderivingさせたものでシリアル化したbit列が異なってしまっています。本質的なことではありませんが...)。

```
> serialize Apple'
[I,I]
> serialize Banana'
[I,O]
> deserialize [I,I] :: Maybe (Fruit', [Bit])
Just (Apple',[])
> deserialize [I,O] :: Maybe (Fruit', [Bit])
Just (Banana',[])

> let tree = Node' (Node' (Leaf' Apple') (Leaf' Banana')) (Node' (Node' (Node' (Leaf' Grape') (Leaf' Orange')) (Leaf' Apple')) (Leaf' Banana'))
> serialize tree
[I,I,O,I,I,O,I,O,I,I,I,O,O,I,O,O,O,O,I,I,O,I,O]
> deserialize . serialize $ tree :: Maybe (Tree' Fruit', [Bit])
Just (Node' (Node' (Leaf' Apple') (Leaf' Banana')) (Node' (Node' (Node' (Leaf' Grape') (Leaf' Orange')) (Leaf' Apple')) (Leaf' Banana')),[])
```

<a id="anchor5"></a>

### 5. その他

以上で`GHC.Generics`の提供する機能の7割程度はカバーできるのではないかと思うのですが、[hackage][5]を見る限り他には以下のような内容が含まれるようです。

- `Generic1` class
- Representation of unlifted types

これらに関しては理解が足りていないのでここでは触れられませんが、`Generic1`に関しては[generic-deriving][2]パッケージが理解に役立ちそうです。

### Summary

- `GHC.Generics`はHaskellでgeneric programmingを行う上で強力な機能
- 自分で定義した型クラスでderivingできるのは気持ち良い

---

### 参考

- [GHC.Generics -Hackage][5]
- [Generics - HaskellWiki][1]
- [Generic programming in Haskell][4]

[1]: https://wiki.haskell.org/Generics
[2]: https://hackage.haskell.org/package/generic-deriving
[3]: https://en.wikipedia.org/wiki/Generic_programming
[4]: https://jeltsch.wordpress.com/2016/02/22/generic-programming-in-haskell/
[5]: https://hackage.haskell.org/package/base-4.9.0.0/docs/GHC-Generics.html
[6]: https://github.com/tiqwab/example/tree/master/ghc-generics
