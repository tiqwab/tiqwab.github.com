---
layout: post
title: 帰納的関数について
tags: "recursive function"
comments: true
---

主に『[論理と計算のしくみ][1]』を参考に。

---

### 計算可能関数

- チューリングマシンについては [別 post][2] を参考
- 自然数から自然数への部分関数が、あるチューリングマシンが計算する部分関数に一致するとき、その部分関数を計算可能 (computable) 、あるいはチューリング計算可能という
- チューリングマシンの表す関数が部分関数であるとは
  - 関数の定義域が自然数全体ではない
  - 定義域内の入力に対しては機械が停止する
  - 定義域外の入力に対しては機械が停止しない

### 原始帰納的関数

原始帰納的関数 (primitive recursive function) と呼ばれる自然数上の関数を以下の 3 つを使用して定義する。

#### 初期関数

- ゼロを表す定数関数: \\( zero() = 0 \\)
- 後継者関数: \\( succ(x) = x + 1 \\)
- 射影関数: \\( proj^{n}\_{i}(x\_1, ..., x\_n) = x\_i \ (1 \leq i \leq n) \\)

#### 関数の合成

関数 \\( g: \mathbb{N}^m \rightarrow \mathbb{N} \ \\) と \\( \ g\_1, ..., g\_m: \mathbb{N}^n \rightarrow \mathbb{N} \\) を使用して関数 \\( f: \mathbb{N}^n \rightarrow \mathbb{N} \\) を以下のように作る。

\\( f(x\_1, ..., x\_n) = g(g\_1(x\_1, ..., x\_n), ..., g\_m(x\_1, ..., x\_n)) \\)

#### 原始帰納法

関数 \\( g: \mathbb{N}^n \rightarrow \mathbb{N} \ \\) と \\( \ h: \mathbb{N}^{n+2} \rightarrow \mathbb{N} \\) を使用して関数 \\( f: \mathbb{N}^{n+1} \rightarrow \mathbb{N} \\) を以下のように作る。

\\( f(x\_1, ..., x\_n, 0) = g(x\_1, ..., x\_n) \\)
\\( f(x\_1, ..., x\_n, y+1) = h(x\_1, ..., x\_n, y, f(x\_1, ..., x\_n, y)) \\)

初期関数に関数の合成と原始帰納法を有限回繰り返し適用して得られる関数 (原始帰納的に得られる関数) を原始帰納的関数と呼ぶ。

例えば原始帰納的関数として以下のものが挙げられる。

- \\( pred(x) = x - 1 \\)
  - \\( pred(0) = zero() \\)
  - \\( pred(x+1) = proj^{2}\_{1}(x, pred(x)) = x \\)
- \\( add(x, y) = x + y \\)
  - \\( add(x, 0) = proj^{1}\_{1}(x) = x \\)
  - \\( add(x, y+1) = succ(proj^{3}\_{3}(x, y, add(x, y))) = succ(add(x, y)) \\)
- \\( mult(x, y) = x \times y \\)
  - \\( mult(x, 0) = 0 \\)
  - \\( mult(x, y+1) = add(proj^{3}\_{1}(x, y, mult(x, y)), proj^{3}\_{3}(x, y, mult(x, y))) = add(x, mult(x, y)) \\)

原始帰納的定義を厳密に書くと煩わしいため、今後は最右のような表記を採用する。

### 有界最小化

\\( f(x,y) \\) が原始帰納的関数であるとき、次のような関数 \\( g(x,z) \\) を原始帰納的関数として定義することができる (原始帰納的定義は専門書を参照)。

\\( g(x,z) = \mu y. (y \lt z \land f(x,y) = 0) \\)

これは \\( y \lt z \\) を満たす \\( y \\) で \\( f(x,y) = 0 \\) を満たすものがあればそのような \\( y \\) の最小のものを返し、無ければ \\( z \\) を返す関数を表している (ただし \\( z \\) である必然性はない)。

このように関数 \\( f(x,y) \\) から関数 \\( g(x,z) \\) を定義することを有界最小化 (bounded minimization) という。

有界最小化を使用することで、関数 \\( div \\) を定義することができる ( \\( \lt \\) は真のとき \\( 0 \\) , 偽のとき \\( 1 \\) となるような原始帰納的関数である) 。

- \\( div(x, y) = x \div y \\) (の整数部分)
  - \\( div(x, y) = \mu z. (z \lt x \land x \lt mult(succ(z), y)) \\)

### 有限列の符号化

原始帰納的関数として以下のものを考える。

\\( p(x,y) = div((x+y+1)(x+y), 2) + y \\)

この関数は \\( \mathbb{N} \times \mathbb{N} \rightarrow \mathbb{N} \\) の単射である (というより全単射のはず)。
そのため \\( z = p(x, y) \\) の \\( z \\) から \\( x, y \\) をそれぞれ求める \\( p\_1(z) = x, p\_2(z) = y \\)  なる原始帰納的関数も存在する。

関数 \\( p(x, y) \\) は自然数の組を一つの自然数に対応させるので、組 \\( \langle x,y \rangle \\) に対して自然数 \\( p(x, y) \\) を \\( \langle x,y \rangle \\) の符号と呼ぶ。

- 文字コードを考えるとわかるように一つの文字は自然数で表すことができる
  - 符号化を繰り返し適用することで文字列は符号化できる
- 同様にリストやスタック等のデータ構造を符号化することが可能

原始帰納的関数ならば計算可能な関数であり、計算可能な関数の多くは原始帰納的に表される。しかし原始帰納的ではない計算可能関数も存在する (e.g. Ackermann 関数)。それらは帰納的関数として定義される。

### 帰納的関数

(有界でない) 最小化とは 原始帰納的関数 \\( f(x,y) \\) と 自然数 \\( x \\) に対して、 \\( f(x,y) = 0 \\) を満たす最小の \\( y \\) を求める操作であり、以下のように表される。

\\( \mu y. (f(x,y) = 0) \\)

(ただしある \\( x \\) に対してそのような \\( y \\) が存在しない場合、関数 \\( f \\) は その \\( x, y \\) に関して未定義となる)

原始帰納的関数と最小化を使用して定義される関数を帰納的関数と呼ぶ。

任意の帰納的関数 \\( h(x) \\) は 原始帰納的関数 \\( f(x,y) \\) と \\( g(x) \\) を使用して以下のように表現できる。

\\( h(x) = g( \mu y. (f(x,y) = 0) ) \\)

(これは感覚的には \\( h(x) = g( x, \mu y.(f(x,y) = 0)) \\) じゃないかという気もするのだけれど、結局 \\( f, g \\) の取り方が変わるだけで同じこと...か?)

帰納的関数として定義される関数のクラスとチューリング計算可能な関数のクラスは一致する。

### 停止性問題

帰納的述語 \\( T(e,x,y) \\) を以下のように定義する (このような述語は [クリーネの T 述語][3] と呼ばれる)。

\\( T(e,x,y) = f\_{M}(x,y) \\)

- チューリングマシンのある時点における構成は現在の状態、ヘッダ下の文字、テープ上の文字列の 3 つ組で表現される 
  - チューリングマシンの開始から停止までにおけるこの 3 つ組の並びは符号化できる
- \\( f\_{M}(x,y) \\) はチューリングマシン \\( M \\) に対し入力 \\( x \\) を与えた場合の 3 つ組の並びの符号化が \\( y \\) であるときに \\( 0 \\) 、そうでない場合に \\( 1 \\) を返す関数である
- \\( e \\) はチューリングマシン \\( M \\) のテーブルを符号化したものである (\\( M \\) のインデックス)

すなわち関数 \\( T \\) は、 \\( e \\) をテーブルとするチューリングマシンが入力 \\( x \\) に対して走り始めてから停止するまでに得られる 3 つ組の並びの符号化が、 \\( y \\) に等しいかを調べる。

このときチューリング機械 \\( M \\) のインデックス \\( e \\) と入力 \\( x \\) に対して、

\\(
\begin{eqnarray}
h(p(e,x))
= 
  \begin{cases}
    0 & (T(e,x,y) = 0 \ \small{を満たす自然数} \ y \ \small{が存在する}) \\\\ 
    1 & (T(e,x,y) = 0 \ \small{を満たす自然数} \ y \ \small{が存在しない})
  \end{cases}
\end{eqnarray}
\\)

を満たすような関数 \\( h \\) を考える。もしこの関数が定義できるのであれば、これはチューリング機械 \\( M \\) に入力 \\( x \\) を与えたときに停止するかどうかを判定する関数になる。

( \\( p \\) はこのとき万能チューリングマシンのような働きをしているものと考えられる?  
関数 \\( h \\) は \\( h: \mathbb{N}^2 \rightarrow \mathbb{N} \\) という全域関数として考えている...はず)

しかしこのような帰納的関数は存在しないことが知られている。

もし \\( h \\) が計算可能であった場合、入力 \\( x \\) に対して

- \\( h(p(x,x)) = 0 \\) ならば停止しない
- \\( h(p(x,x)) = 1 \\) ならば停止する

というチューリングマシン \\( M\_0 \\) を作ることができる。 \\( M\_0 \\) のインデックスを \\( e\_0 \\) とする。 \\( h(p(e\_0, e\_0)) = 0 \\) であるとすると、関数 \\( h \\) の定義よりインデックス \\( e\_0 \\) で表されるチューリングマシンが入力 \\(e\_0 \\) に対して停止するということになる。しかしこれは実際のチューリングマシン \\( M\_0 \\) の動作とは矛盾する。 \\(h(p(e\_0, e\_0)) = 1 \\) の場合も同様に矛盾する。

よってチューリングマシンが入力に対して有限時間内に停止するかどうかを判定するアルゴリズムは存在しない。

### 帰納的集合

\\( A \subset \mathbb{N}^{n} \\) とする。述語 \\( \vec{x} \in A \\) が決定可能なとき \\( A \\) を帰納的集合と呼ぶ。

\\( A \\) がある帰納的関数 \\( f: \mathbb{N}^{n} \rightarrow \mathbb{N} \\) の定義域に等しいとき、 \\( A \\) を帰納的可算集合という。

### おまけ

原始帰納的関数の Haskell のデータ型による表現。

原始帰納法に対する `eval` の定義が難しい。
定義通りだと `y+1` のパターンマッチから `y` の使用が必要なんだけれど、それをうまく表現する方法がわからなかった。
諦めて `y-1` を定義に使用してしまうとコメントアウトした通りの実装になり、直感的には理解しやすくなる。

他に `0` からの計算で `eval R (f g) xs` を計算しているのもあったので、ここではそちらを使用している。

```haskell
import           Prelude hiding (pred, succ)

{-
Implementation of primitive recursive function by Haskell.

References:
https://www.nayuki.io/page/primitive-recursive-functions
-}

-- | Assume this is natural number including zero.
type Natural = Int

-- | Primitive recursive function
data Prf = Z -- zero
         | S -- successor
         | P Natural Natural -- projection
         | C Prf [Prf] -- composition
         | R Prf Prf -- recursion
         deriving (Show)

-- | Evaluate a primitive recursive function
eval :: Prf -> [Natural] -> Natural
eval Z _        = 0
eval S [x]       = x+1
eval S [] = error "Wrong number of arguments"
eval S (_:_:_) = error "Wrong number of arguments"
eval (P n i) xs
  | length xs /= n = error ("Wrong number of arguments: " ++ show (length xs))
  | otherwise  = xs !! (i-1)
eval (C f gs) xs = eval f $ map (\g -> eval g xs) gs
eval (R f g) xs = evalR 0 (eval f ys)
    where
        (ys, y) = (init xs, last xs)
        evalR i val
          | i == y    = val
          | otherwise = evalR (i+1) (eval g (i : val : ys))
{-
eval (R f g) xs
  | y > 0     = eval g ((y-1) : val : ys)
  | otherwise = eval f ys
  where (ys, y) = (init xs, last xs)
        val = eval (R f g) (ys ++ [y-1])
-}

add :: Prf
add = R (P 1 1) (C S [P 3 2])

mult :: Prf
mult = R Z (C add [P 3 3, P 3 2])

pred :: Prf
pred = R Z (P 2 1)

sub :: Prf
sub = R (P 1 1) (C pred [P 3 2])

main :: IO ()
main = do
    print $ eval add [3, 2] == 5
    print $ eval add [3, 4] == 7
    print $ eval mult [2, 3] == 6
    print $ eval mult [4, 2] == 8
    print $ eval pred [2] == 1
    print $ eval pred [0] == 0
    print $ eval sub [3, 2] == 1
    print $ eval sub [2, 3] == 0
```

[1]: https://www.amazon.co.jp/%E8%AB%96%E7%90%86%E3%81%A8%E8%A8%88%E7%AE%97%E3%81%AE%E3%81%97%E3%81%8F%E3%81%BF-%E8%90%A9%E8%B0%B7-%E6%98%8C%E5%B7%B1/dp/4000061917
[2]:{% post_url 2017-03-26-turing-machine %} "チューリングマシンについて"
[3]: https://en.wikipedia.org/wiki/Kleene%27s_T_predicate "Kleene's T predicate"
