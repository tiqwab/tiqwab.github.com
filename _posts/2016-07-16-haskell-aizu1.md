---
layout: post
title:  "HaskellでAIZU ONLINE JUDGE Part1"
tags: "haskell, aizu"
---

Haskellに慣れるために[AIZU ONLINE JUDGE][1]やってみます。「Introduction to Algorithms and Data Structures」を対象にします。詰まったところや知らなかった関数などをメモします。

### ALDS1_1_A

Insertion Sortを実装してくださいという問題。  

1問目なのですが、Haskellビギナーとしては「処理中の配列を出力する」という本筋ではないところでむちゃくちゃ苦労しました。Writerモナドを使うのかなと考えていましたが、解答では`scanl`により途中の配列をリストとして保持するというアプローチが多いようです。  

以下の2つの関数は今後リストを扱うときに便利そうなのでメモ。

```haskell
*Main Lib> :t splitAt
splitAt :: Int -> [a] -> ([a], [a])
*Main Lib> splitAt 2 [0, 1, 2, 3]
([0,1],[2,3])
*Main Lib> splitAt 0 [0, 1, 2, 3]
([],[0,1,2,3])
*Main Lib> :t span
span :: (a -> Bool) -> [a] -> ([a], [a])
*Main Lib> span (< 2) [0, 1, 2, 3, 4]
([0,1],[2,3,4])
*Main Lib> span (< 2) [4, 2, 1, 0, 3]
([],[4,2,1,0,3])
```

### ALDS1_1_C

与えられた数が素数かどうか判定する問題。基本的な方針は

1. 与えられた数の二乗根(k)をとる
2. k以下の整数でxが割り切れなければ素数

2.で調べる整数の数を減らすことでより効率はよくなる (eg. 奇数のみ調べる、素数のみ調べる)。今回のように与えられる数が複数ある場合、前の計算時に使用して素数だと判明した数をどこかに持っておくとよいのかもしれませんが、いまのHaskellレベルで綺麗に実装できる気がしないため諦め。  

以下が回答。

```haskell
isPrime :: Int -> Bool
isPrime 1 = False
isPrime 2 = True
isPrime x =
  let limit = floor . sqrt $ fromIntegral x
  in  not $ any (\k -> x `mod` k == 0) [2..limit]

main = do
  _ <- getLine
  content <- getContents
  let inputs = map (read :: String -> Int) $ lines content
  putStrLn $ show . length $ filter isPrime inputs
```

### ALDS1_3_B

ラウンドロビン・スケジューリング(round-robin scheduling)を擬似的に実装しましょうという問題。  

はじめは`List`を使って解答を書いたのですが、制限時間overになってしまいました。他の方の解答では`Data.Sequence`を使用しているものが多かったので、`List`部分を`Data.Sequence`に変更して解答としました。  

`List`で遅いのは`++`によるリスト結合時に左の配列の長さ分の再帰が行われるから、なのでしょうか。`++`が右結合である理由もこのためだと述べている情報もあります ([参照][2])。

```haskell
(++) :: [a] -> [a] -> [a]
(++) []     ys = ys
(++) (x:xs) ys = x : xs ++ ys
```

#### Listで解く (Time Limit Exceeded)

```haskell
type TotalTime = Int
type UnitTime = Int

roundRobin :: [(String, Int)] -> [(String, Int)] -> UnitTime -> TotalTime -> [(String, Int)]
roundRobin [] ys _ _ = ys
roundRobin (x:xs) ys u t
  | (snd x) == consume = roundRobin xs ((fst x, total):ys) u total
  | otherwise          = roundRobin (xs ++ [(fst x, snd x - consume)]) ys u total
  where consume = min (snd x) u
        total = t + consume

main = do
  u <- fmap (read . (!! 1) . words) getLine
  input <- fmap ((map words) . lines) getContents
  let queue = map (\[x, y] -> (x, read y))  input
  let result = roundRobin queue [] u 0
  mapM_ putStrLn (reverse $ map (\(x, y) -> x ++ " " ++ show y) result)
```

#### Data.Sequenceで解く

```haskell
-- リストから作成
*Main Lib Data.Sequence> let s = fromList [1, 2, 3]
*Main Lib Data.Sequence> :t s
s :: Num a => Seq a

-- 左から要素を追加する関数
*Main Lib Data.Sequence> 0 <| s
fromList [0,1,2,3]

-- 右から要素を追加する関数
*Main Lib Data.Sequence> s |> 4
fromList [1,2,3,4]

-- SeqのパターンマッチにはViewL, ViewRを使用する。
-- ViewPatternsを利用する方が書きやすいかも。
*Main Lib Data.Sequence> :{
*Main Lib Data.Sequence| case (viewl s) of
*Main Lib Data.Sequence|   EmptyL -> 0
*Main Lib Data.Sequence|   (x :< xs) -> x
*Main Lib Data.Sequence| :}
1
```

```haskell
{-# LANGUAGE ViewPatterns #-}

import Data.Sequence (Seq, (<|), (|>), fromList, viewl, ViewL((:<)), ViewL(EmptyL))

roundRobinS :: Seq (String, Int) -> [(String, Int)] -> UnitTime -> TotalTime -> [(String, Int)]
roundRobinS (viewl -> EmptyL) ys _ _ = ys
roundRobinS (viewl -> x :< xs) ys u t
  | (snd x) == consume = roundRobinS xs ((fst x, total):ys) u total
  | otherwise          = roundRobinS (xs |> (fst x, snd x - consume)) ys u total
  where consume = min (snd x) u
        total = t + consume
```

[1]: http://judge.u-aizu.ac.jp/onlinejudge/index.jsp
[2]: http://d.hatena.ne.jp/sirocco/20110107/1294370177
