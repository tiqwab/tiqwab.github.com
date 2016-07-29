---
layout: post
title:  "HaskellでAIZU ONLINE JUDGE Part2"
tags: "haskell, aizu"
---

引き続きHaskellに慣れるために[AIZU ONLINE JUDGE][1]やってみます。「Introduction to Algorithms and Data Structures」を対象にします。詰まったところや知らなかった関数などをメモします。

### ALDS1_4_C

DNAの塩基配列がぽいぽい渡されたりそれを探したり、という問題。  

とりあえず動くものは実装できましたが、TLEで失敗。またメモリも使いすぎている。以下の2点を改善したいのですが、Haskellでのうまいやり方がわからないので一旦保留に。

- `insert`, `find`関数の実装を二分探索木に。
- `find`関数の結果を随時出力し、メモリ上に保存しないように。

```haskell
type Result = ([String], [Bool])

dictionary :: Result -> String -> Result
dictionary (items, found) input
  | cmd == "insert" = (insert target items, found)
  | cmd == "find"   = (items, (find target items) : found)
  where inputs = words input
        cmd    = inputs !! 0
        target = inputs !! 1

insert :: String -> [String] -> [String]
insert x [] = [x]
insert x (y:ys)
  | x `compare` y == GT = y : (insert x ys)
  | otherwise           = x:y:ys

find :: String -> [String] -> Bool
find x [] = False
find x (y:ys)
  | x == y    = True
  | otherwise = find x ys

main = do
  _ <- getLine
  inputs <- fmap lines getContents
  let result = foldl dictionary ([], []) inputs
  mapM_ putStrLn (map yesno $ reverse $ snd result)

yesno :: Bool -> String
yesno True = "yes"
yesno False = "no"
```

### ALDS2_4_B

binary searchしてくださいという問題。  

はじめは何も考えずListを使い実装 -> TLEでした。なのでデータ構造として配列を使用するように変更して解決。Haskellでとりあえず配列を扱いたいという場合、`Array` or `UArray`を使えば良さそうです (他にも`IOArray`やら色々あるようですが、今回の用途には`UArray`で問題なし)。

```haskell
import Data.Array.Unboxed

{-
Array
  -> Imported from Data.Array
  -> 参照の配列
  -> 実際に添字を使ってアクセスされるまで値が評価されない
UArray
  -> Imported from Data.Array.Unboxed
  -> 値の配列
  -> 正格評価
-}
binarySearch :: Int -> (Int, Int) -> UArray Int Int -> Bool
binarySearch target (start, end) arr
  | start == end    = target == arr!i
  | target == arr!i = True
  | target < arr!i  = binarySearch target (start, i) arr
  | otherwise       = binarySearch target (i+1, end) arr
  where i = floor $ (fromIntegral (end + start)) / 2

main = do
  n <- fmap read getLine
  seqNum <- fmap (map read . words) getLine
  let arr = listArray (1, n) seqNum
  _ <- getLine
  targets <- fmap (map read . words) getLine
  putStrLn $ show . length . filter (\x -> binarySearch x (1, n) arr) $ targets
```

### ALDS1_7_C

Binary tree walkしましょう、各nodeの番号を指定されたorderで読み上げましょう、という問題。  

Writerモナドの練習と便利さを感じるのにちょうど良い問題な気がします。`preorder`, `inorder`, `postorder`の挙動がコードにシンプルに現れているのではないでしょうか。

```haskell
import Data.List
import Control.Monad.Writer

data Tree a = EmptyTree | Node a (Tree a) (Tree a) deriving (Show)

-- ルートノードを見つける。
findRoot :: Int -> [(Int, (Int, Int))] -> Int
findRoot x xs = let
                  findLR (idx, (l, r)) = l == x || r == x
                  node = find findLR xs
                in
                  case node of
                    Just (idx, _) -> findRoot idx xs
                    Nothing       -> x

-- 二分木を構築する。
makeTree :: Int -> [(Int, (Int, Int))] -> Tree Int
makeTree x xs = let
                  node = lookup x xs
                in
                  case node of
                    Just (l, r) -> Node x (makeTree l xs) (makeTree r xs)
                    Nothing            -> EmptyTree

-- PreOrderでノード番号を取得する。
preorder :: Tree Int -> Writer [Int] (Tree Int)
preorder EmptyTree = return (EmptyTree)
preorder (Node idx l r) = do
  tell [idx]
  preorder l
  preorder r

-- InOrderでノード番号を取得する。
inorder :: Tree Int -> Writer [Int] (Tree Int)
inorder EmptyTree = return (EmptyTree)
inorder (Node idx l r) = do
  inorder l
  tell [idx]
  inorder r

-- PostOrderでノード番号を取得する。
postorder :: Tree Int -> Writer [Int] (Tree Int)
postorder EmptyTree = return (EmptyTree)
postorder (Node idx l r) = do
  postorder l
  postorder r
  tell [idx]
  return (EmptyTree)

-- 出力用文字列のフォーマット
format :: [Int] -> String
format xs = foldl (\acc x -> acc ++ " " ++ show x) "" xs

main = do
  n <- fmap (read) getLine
  lns <- sequence $ replicate n getLine
  let nodes = map ((\[x,y,z] -> (x, (y,z))) . map read . words) lns
      r = findRoot (fst (nodes !! 0)) nodes
      tree = makeTree r nodes
  putStrLn "Preorder"
  putStrLn $ format . snd . runWriter $ preorder tree
  putStrLn "Inorder"
  putStrLn $ format . snd . runWriter $ inorder tree
  putStrLn "Postorder"
  putStrLn $ format . snd . runWriter $ postorder tree
```

### ALDS1_10_C

いわゆる最長共通部分列(LCS)問題と呼ばれるものです。  

よく動的計画法の例として用いられる問題です。Haskellさんで部分問題に分解するのはお手の物なのですが、どうメモ化部分を実装したらいいかがわからず、ひとまずその部分を無視して実装しました。

```haskell
{- First trial -}
lcs :: (String,String) -> Int -> Int
lcs ([],_) n = n
lcs (_,[]) n = n
lcs (ax@(x:xs),ay@(y:ys)) n
  | x == y    = lcs (xs,ys) (n+1)
  | otherwise = max (lcs (ax,ys) n) (lcs (xs,ay) n)

{- Parse input -}
main = do
  getLine
  inputs <- fmap lines getContents
  let targets = zip (takeOnlyOdd inputs) (takeOnlyEven inputs)
  mapM_ putStrLn $ map (\(x,y) -> show . length $ lcs' x y) targets

takeOnlyEven :: [a] -> [a]
takeOnlyEven xs = map snd . filter (\(i, n) -> i `mod` 2 == 0) $ zip [1..] xs
takeOnlyOdd :: [a] -> [a]
takeOnlyOdd xs = map snd . filter (\(i, n) -> i `mod` 2 == 1) $ zip [1..] xs
```

これで正解にはたどり着けますが、パフォーマンスは悪く、submitしても1問目から時間切れになります。  

ということで実際のHaskellのLCS実装を調べたところ、Haskellのメモ化パッケージとして`data-memocombinators`というのがあるようです。

```haskell
import qualified Data.MemoCombinators as M

-- M.integral :: Integral a => (a -> r) -> a -> r
-- Integralを受け取る1引数関数をメモ化された関数にする。
fib = M.integral fib'
  where
    fib' 0 = 0
    fib' 1 = 1
    fib' x = fib (x-1) + fib (x-2)

-- M.memo2 :: Memo a -> Memo b -> (a -> b -> r) -> a -> b -> r
-- 2引数関数をメモ化された関数にする。
memoPlus = M.memo2 mString mString (++)

-- Memo型のStringは直接は用意されていないのでCharのListとして表現する。
mString = M.list M.char
```

これを使用してLCSを解くと以下の通りです。

```haskell
import qualified Data.MemoCombinators as M

{- Second trial -}
{- Most of codes come from RosettaCode... -}
lcs' = memoize lcsm
       where
         lcsm [] _ = []
         lcsm _ [] = []
         lcsm ax@(x:xs) ay@(y:ys)
           | x == y    = x:(lcs' xs ys)
           | otherwise = maxl (lcs' ax ys) (lcs' xs ay)

maxl x y = if length x > length y then x else y
memoize = M.memo2 mString mString
mString = M.list M.char

{- Parse input -}
main = do
  getLine
  inputs <- fmap lines getContents
  let targets = zip (takeOnlyOdd inputs) (takeOnlyEven inputs)
  mapM_ putStrLn $ map (\(x,y) -> show . length $ lcs' x y) targets

takeOnlyEven :: [a] -> [a]
takeOnlyEven xs = map snd . filter (\(i, n) -> i `mod` 2 == 0) $ zip [1..] xs
takeOnlyOdd :: [a] -> [a]
takeOnlyOdd xs = map snd . filter (\(i, n) -> i `mod` 2 == 1) $ zip [1..] xs
```

ただ残念ながらAIZU ONLINEではパッケージが対応していない？ためruntime errorとなります。またコード自体も依然パフォーマンス不十分なようでローカルで実行した`in4.txt`でかなり時間がかかります。ListをArrayにする等できることはまだありますが、最初のものよりは改善されたということでとりあえずはよしとします。

[1]: http://judge.u-aizu.ac.jp/onlinejudge/index.jsp
