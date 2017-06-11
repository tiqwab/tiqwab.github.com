---
layout: post
title: 継続について
tags: "continuation, cps, haskell, scala"
comments: true
---

継続や継続モナドについてのメモ。
まだ理解している内容が薄いので箇条書き形式にしています。

---

### 継続とは

- 継続 (Continuation)
  - プログラムの実行においてある時点において評価されていない残りのプログラムを意味するもの (引用: [Wikipedia][1])
  - 次に行われる計算 (引用: [継続渡しスタイル][2])
  - 現在の計算を続行するための情報 (引用: [なんでも継続][4])
- 感覚的には「これからおこなわれるであろう計算」や「今の時点での環境」といったものをまとめたものというイメージ
- Scheme では継続が第一級オブジェクトであり、変数への代入や関数への適用ができる

### 継続渡しスタイルとは

- 継続渡しスタイル (Continuation Passing Style, CPS)
  - 一般のプログラミング言語では Scheme のように継続を取り出すということはできない
  - 継続をクロージャで表して似たようなことができる
    - これから行う計算や環境を持つという点で、たしかに似たようなことができるのは納得できる

以下は階乗を計算する関数 (fact) とそれを CPS に書き換えたもの (factCont)。

- `fact` は自然数 n を受け取り、その階乗の計算結果を返す関数
- `factCont` は自然数 n と「その階乗の結果を受け取り計算を行う関数」を受け取り、その結果を返す関数と見れる

```
> fact :: Integer -> Integer
> fact 0 = 1
> fact n = n * (fact (n-1))

> fact 5
120

> factCont :: Integer -> (Integer -> a) -> a
> factCont 0 f = f 1
> factCont n f = factCont (n-1) (\x -> f (n * x))

> factCont 5 id
120
```

`factCont` がパット見分かりづらいけれど、一つずつ追っていけば確かに階乗を計算していることがわかる。

```
factCont 3 id
= factCont 3 (\x -> x) -- rewrite 'id' to '\x -> x'
= factCont 2 (\x2 -> (\x -> x) (3 * x2))
= factCont 1 (\x1 -> (\x2 -> (\x -> x) (3 * x2)) (2 * x1))
= factCont 0 (\x0 -> (\x1 -> (\x2 -> (\x -> x) (3 * x2)) (2 * x1)) (1 * x0))
= (\x0 -> (\x1 -> (\x2 -> (\x -> x) (3 * x2)) (2 * x1)) (1 * x0)) 1
= (\x1 -> (\x2 -> (\x -> x) (3 * x2)) (2 * x1)) (1 * 1)
= (\x1 -> (\x2 -> (\x -> x) (3 * x2)) (2 * x1)) 1
= (\x2 -> (\x -> x) (3 * x2)) (2 * 1)
= (\x2 -> (\x -> x) (3 * x2)) 2
= (\x -> x) (3 * 2)
= (\x -> x) 6
= 6
```

### 継続モナドとは (Cont)

- 継続モナド (Cont)
  - 一般に CPS の関数をモナド化したもの

Haskell での定義は以下のようになる。

- `return x` の定義は渡された関数にただ x を適用するというもの
- `m >>= k` の定義は継続 m の結果を関数 k に渡して評価するというもの

```haskell
newtype Cont r a = ContT { runCont :: (a -> r) -> r }

instance Monad (Cont r) where
    return x = Cont $ \ f -> f x
    m >>= k = Cont $ \ c -> runCont m (\ x -> runCont (k x) c)
```

Cont の使用例は以下の通り。

```haskell
-- 2 つの整数の和を計算する関数を CPS で書いたもの
> let addCont x y = return (x + y) :: Cont r Int
> runCont (addCont 1 2) id
3

-- 2 つの整数の積を計算する関数を CPS で書いたもの
> let mulCont x y = return (x * y) :: Cont r Int
> runCont (mulCont 2 3) id
6

-- bind の例
> let addMulCont x y = (addCont x y) >>= (\z -> (mulCont 2 z)) :: Cont r Int
> runCont (addMulCont 1 2) print
6

-- 上と同じものを do 表記で
> addMulCont :: Int -> Int -> Cont r Int
> addMulCont x y = do
>   z <- addCont x y
>   mulCont 2 z

> runCont (addMulCont 1 2) id
6
```

`callCC` を使うと現在の継続 (current configuration) が取り出せる。
これは計算途中での脱出のような使い方ができる。

```haskell
> addMulContCC :: Int -> Int -> Cont r Int
> addMulContCC x y = callCC $ \exit -> do
>   z <- addCont x y
>   when (z < 0) (exit 0)
>   mulCont 2 z

> runCont (addMulContCC 1 2) id
6
> runCont (addMulContCC (-1) (-2)) id
0
```

継続モナドを使用するモチベーションとしては「ある計算とその前後に行いたい処理」を複数扱うようなプログラムをきれいに実装できるということがある。

- [継続モナドによるリソース管理][3]
- [継続モナドを使ってWebアプリケーションのコントローラーを自由自在に組み立てる][5]

### 参考

- [継続渡しスタイル][2]
- [継続モナドによるリソース管理][3]

[1]: https://ja.wikipedia.org/wiki/%E7%B6%99%E7%B6%9A "継続"
[2]: http://www.geocities.jp/m_hiroi/func/haskell38.html
[3]: http://qiita.com/tanakh/items/81fc1a0d9ae0af3865cb#%E7%B6%99%E7%B6%9A%E3%83%A2%E3%83%8A%E3%83%89
[4]: http://practical-scheme.net/docs/cont-j.html
[5]: http://qiita.com/pab_tech/items/fc3d160a96cecdead622
