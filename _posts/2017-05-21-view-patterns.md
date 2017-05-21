---
layout: post
title: ViewPatterns について
tags: "view patterns, haskell"
comments: true
---

Haskell の GHC 拡張 ViewPaterns についての簡単なまとめです。
View とは何かということと、その使用例について書いています。

1. [ViewPatterns](#view-patterns)
2. [Data.Sequence.Seq と View](#seq-and-view)

---

<div id="view-patterns" />

### 1. ViewPatterns

Haskell における ViewPatterns とは 「関数を適用した計算結果の値に対しパターンマッチができるようになる GHC 拡張」 です。

例としてここでは、与えられた `List[Int]` の末尾が 0 であるかを判断する関数を定義したいとしましょう。

ViewPatterns なしで定義すると以下のようになると思います。

```haskell
{- Without ViewPatterns -}

endWithZero :: [Int] -> Bool
endWithZero [] = False
endWithZero xs
  | last xs == 0 = True
  | otherwise    = False
```

これが ViewPatterns ありでは以下のように書けます。

```haskell
{- With ViewPatterns -}
{-# LANGUAGE ViewPatterns #-}

endWithZero' :: [Int] -> Bool
endWithZero' []          = False
endWithZero' (last -> 0) = True
endWithZero' _           = False
```

2 行目の定義で `(last -> 0)` というものが出てきますが、これが view を使用したパターンマッチです。
ここでは `last` に `endWithZero` の `[Int]` 型引数が渡され、その結果が 0 であればパターンマッチ成功ということになります。

このように ViewPatterns 拡張では関数を使用したパターンマッチを行うことができます。

<div id="seq-and-view" />

### 2. Data.Sequence.Seq と View

上の例だけですと ViewPatterns の意義があまりわかりませんが、現実に使用される例としては、あるデータ構造に対しその実装を抽象化しつつパターンマッチを提供するというものがあるようです。

ここでは例として [Data.Sequence.Seq][1] を挙げます。

Seq は List と同様に要素の並びを表すデータ型です。
List とは違い無限長を扱うことはできませんが、効率的に処理を行えるように構造が工夫されています。
例えば末尾要素へのアクセスは List が O(n) の時間が必要なのに対し、Seq は O(1) で済みます。

Seq のデータ構造は [finger tree][2] と呼ばれるもののようですが、このデータ型を使用する側は実装の詳細を知ることなく扱えるようになっています (現に Seq の値コンストラクタは公開されていない)。

Seq の生成には `fromList`, `empty`, `singleton` といった関数が用意されており、パターンマッチとしてはここでの本題である ViewPatterns 用のデータや関数を用意することで対応しています。

上と同様に末尾の要素が 0 であるかを判断する関数を定義してみましょう。
末尾の要素へアクセスするためにここでは `ViewR` という view を使用します。

```haskell
data ViewR a = EmptyR | (Seq a) :> a

viewr :: Seq a -> ViewR a
```

`viewr` 関数は Seq を与えるとそれに対する view である `ViewR` を返します。
`ViewR` データ型は 2 つの値コンストラクタを持ち、末尾の要素と Seq で構成されます。
これは List が `data [] a = [] | a : [a]` と表されることと似ています。

view を使用すると目的の関数 `endSeqWithZero` は以下のように定義できます。

```haskell
{-# LANGUAGE ViewPatterns #-}

import           Data.Sequence (Seq, ViewR (..), fromList,
                                singleton, viewr, (<|), (|>))
import qualified Data.Sequence as S

endSeqWithZero :: Seq Int -> Bool
endSeqWithZero (viewr -> EmptyR)  = False
endSeqWithZero (viewr -> xs :> x) = x == 0
```

いくつか Seq を与えるとちゃんと機能していることがわかります。

```
ViewSample> import Data.Sequence as S
ViewSample S> endSeqWithZero (S.fromList[1, 2, 3])
False
ViewSample S> endSeqWithZero (S.fromList[1, 2, 0])
True
ViewSample S> endSeqWithZero (S.empty)
False
```

このように ViewPatterns というのは実際のデータ構造を公開せずに、使用者側に見てもらいたい形で公開するための関数とパターンマッチ、と言えるのかなと思います。

[1]: https://hackage.haskell.org/package/containers-0.5.10.2/docs/Data-Sequence.html
[2]: https://ja.wikipedia.org/wiki/2-3_%E3%83%95%E3%82%A3%E3%83%B3%E3%82%AC%E3%83%BC%E3%83%84%E3%83%AA%E3%83%BC
