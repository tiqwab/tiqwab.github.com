---
layout: post
title:  "Splay Tree"
tags: "python, algorithm, splay tree"
comments: true
---

splay treeに触れる機会があったので、自身の理解のためにまとめをば。

- [Splay Treeの概要](#anchor1)
- [Splay Treeの実装](#anchor2)
- [Zig-Zig, Zig-Zagの視覚化](#anchor3)
- [通常の二分木との比較](#anchor4)

---

<a id="anchor1"></a>

### Splay Treeの概要

splay treeは二分木の実装の一つであり、木へのinsert, delete, findを行うときにsplayと呼ばれる操作を行うことが特徴。

- splay操作
  - zig, zig-zig, zig-zagという操作を繰り返し、指定したキーを持つノードをrootに持ってくる。
  - splayにより左右の木のバランスを自動で(自己的に)改善することができる。
  - splay後は指定したキーを持つノードがrootに来るため、splayを実装すればsearch, insert, deleteといった操作は簡単に実装できる。
- 平均実行時間はO(logN)とのこと

<a id="anchor2"></a>

### Splay Treeの実装

[こちら][1]がとてもわかりやすい。splay操作の実装として'bottom-up'と'top-down'方式があるよう。splayを理解するには'bottom-up'の方がわかりやすく、実際の実装では'top-down'の方が書きやすいかなという印象。

二分木としての操作はsplayを使用して以下のように行う。

- search
  - splay操作を行い、指定したキーを持つノードをrootに持ってくる。
  - 指定したキーを持つノードが無い場合も最後にアクセスしたノードをrootに持ってくる。
- insert
  - insert対象のキーを持つノードが二分木に存在するかをsearchで確認する(つまりsplayする)。
  - 存在しなければrootとして追加する。
- remove
  - remove対象のキーを持つノードが二分木に存在するかをsearchで確認する(つまりsplayする)。
  - 存在すればrootのノードを削除する(search後には目的のノードがrootに来ている)。
    - 左右の子ノードがどちらも存在しない -> rootの削除
    - 左右のいずれかの子ノードのみ存在する -> 存在する子ノードをrootにする。
    - 左右両方の子ノードが存在する -> 左の子に対してremove対象のキーでsplayし、そのrootを木全体のrootにする。remove対象のキーが左の子をrootとする部分木のどのノードよりも大きいことを利用している。

<a id="anchor3"></a>

### Zig-Zig, Zig-Zagの視覚化

splay操作中のzig-zig, zig-zag周りの動きがコードからだけではうまくイメージできなかったので実際に動かして確認をしてみることにする。splayの実装として[こちら][1]のbottom-upを想定しており、途中の変数名等もここから引用している。

#### Zig-Zig

以下のような二分木を用意し、node 0を対象にsplayした場合を考える。node 3, node 1の順にzig-zigを行い、最後にzigでnode 0をrootに持ってくる。zig-zigを行うノードの順番に注意。

```
            5
          /   \
        3      6
      /  \      \
    1     4      7
  /  \
0     2

| path: [(5, LEFT), (3, LEFT), (1, LEFT)]
| node, dir = 1, LEFT
| pnode, pdir = 3, LEFT
|
↓ zig at node 3

      5
        \
   1     6
  / \     \
0    3     7
    / \
   2   4

(left child of node 5 is node 3 actually)

↓ zig at node 1

    5
      \
0      6
 \      \
  1      7
   \
    3
   / \  
  2   4

(left child of node 5 is node 3 actually)

| path: [(5, LEFT)]
↓ assign node 0 as left child of node 5

    5
  /   \
 0     6
  \     \
   1     7
    \
     3
    / \  
   2   4

↓ zig at node 5

 0
  \
   5
  / \
 1   6
  \   \
   3   7
  / \
 2   4
```

実際に以下のPythonコードを用意して上の動作を確認する。

```python
import splay_h as sp

def print_splay_tree(node, x):
    if node is not None:
        print_splay_tree(node.left, x+1)
        print("  " * x, node.data)
        print_splay_tree(node.right, x+1)

# print tree simply with appropriate indent
# the left-most node is the root of tree
def print_simple(node, title):
    print(title)
    print_splay_tree(node, 0)
    print('')

# retrieve node with the specified key, but do not splay.
def find_no_splay(node, x):
    if node is None:
        return None
    if node.data == x:
        return node
    elif x > node.data:
        return find_no_splay(node.right, x)
    else:
        return find_no_splay(node.left, x)

# prepare sample tree
def sample():
    a = None
    a = sp.insert(a, 0)
    a = sp.insert(a, 1)
    a = sp.insert(a, 2)
    a = sp.insert(a, 3)
    a = sp.insert(a, 4)
    a = sp.insert(a, 5)
    a = sp.insert(a, 6)
    a = sp.insert(a, 7)

    a, _ = sp.search(a, 2)
    a, _ = sp.search(a, 3)
    a, _ = sp.search(a, 1)
    a, _ = sp.search(a, 6)
    a, _ = sp.search(a, 5)
    return a



print('### Visualization of zig-zig ###\n')

tree1 = sample()
print_simple(tree1, 'Original splay tree: ')

partial1 = find_no_splay(tree1, 3)
print_simple(partial1, 'Partial tree of node 3: ')

partial1 = sp.rotate_right(partial1)
print_simple(partial1, 'After zig at node 3: ')

partial1 = sp.rotate_right(partial1)
print_simple(partial1, 'After zig at node 1: ')

actual1, _ = sp.search(sample(), 0)
print_simple(actual1, "Actual result after searching '0': ")
```

出力結果は以下の通りで想定した動きになっている。(左をrootとみる出力なので、少し見づらいかも)

```
### Visualization of zig-zig ###

Original splay tree:
       0
     1
       2
   3
     4
 5
   6
     7

Partial tree of node 3:
     0
   1
     2
 3
   4

After zig at node 3:
   0
 1
     2
   3
     4

After zig at node 1:
 0
   1
       2
     3
       4

Actual result after searching '0':
 0
     1
         2
       3
         4
   5
     6
       7
```

#### Zig-Zag

同様にnode 2を対象にsplayした場合を考える。node 1、node3の順にzig-zagの動作を行う。

```
            5
          /   \
        3      6
      /  \      \
    1     4      7
  /  \
0     2

| path: [(5, LEFT), (3, LEFT), (1, RIGHT)]
| node, dir = 1, RIGHT
| pnode, pdir = 3, LEFT
|
↓ zig at node 1

         5
        / \
       3   6
     /  \   \
    2    4   7
   /
  1
 /
0

↓ zag at node 3

      5
       \
    2   6
   / \   \
  1   3   7
 /     \
0       4

(left child of node 5 is node 3 actually)

| path: [(5, LEFT)]
↓ assign node 2 as left child of node 5

      5
     / \
    2   6
   / \   \
  1   3   7
 /     \
0       4

↓ zig at node 5

    2
   / \
  1   5
 /   / \
0   3   6
     \   \
      4   7
```

zig-zag分の出力コードは以下の通り。

```python
print('### Visualization of zig-zag ###\n')

tree2 = sample()
print_simple(tree2, 'Original splay tree: ')

partial2_1 = find_no_splay(tree2, 3)
partial2_2 = find_no_splay(tree2, 1)
print_simple(partial2_1, 'Partial tree of node 3: ')

partial2_1.left = sp.rotate_left(partial2_2)
print_simple(partial2_1, 'After zig at node 1: ')

partial2_1 = sp.rotate_right(partial2_1)
print_simple(partial2_1, 'After zag at node 3: ')

actual2, _ = sp.search(sample(), 2)
print_simple(actual2, "Actual result after searching '2': ")
```

実際の出力は以下の通りで動きが確認できている。

```
### Visualization of zig-zag ###

Original splay tree:
       0
     1
       2
   3
     4
 5
   6
     7

Partial tree of node 3:
     0
   1
     2
 3
   4

After zig at node 1:
       0
     1
   2
 3
   4

After zag at node 3:
     0
   1
 2
   3
     4

Actual result after searching '2':
     0
   1
 2
     3
       4
   5
     6
       7
```

<a id="anchor4"></a>

### 通常の二分木との比較

[通常の二分木実装][2]、[splay tree実装][3]を用意し、splayの効果を確認してみる。splay treeの実行時間の評価に関しては既に何度も参考にしている[こちら][1]で評価されているので、ここでは木の左右のバランス具合を見てみることにする。二分木のバランスを見るのにベストな指標はわからないけれど、木の最大深度を確認するのが少なくとも一番手っ取り早いと考え、上記二分木にノード個数を数える`count`メソッド、最大深度を測定する`max_level`メソッドを実装し比較してみる。

```python
from splay import SplayTree
# from splay_h import Splaytree as SplayTreeH
from standard import BinaryTree
from normal_dist import randint_nd
from random import randrange

'''
check balance of normal tree and splay tree
'''

MAX_KEY = 100000
MIN_KEY = 0

def show_result(st, bt):
    '''
    show count of nodes and maxLevel of tree.
    '''
    print('count of splay tree:', st.count())
    print('count of binary tree:', bt.count())
    print('max_level of splay tree:', st.max_level())
    print('max_level of binary tree:', bt.max_level())

def measure_random_insert():
    '''
    insert random keys
    '''
    st = SplayTree()
    bt = BinaryTree()
    for i in range(0, 100000):
        random_key = randrange(1, 100000)
        st.insert(random_key)
        bt.insert(random_key)
    show_result(st, bt)

def measure_random():
    '''
    combination of insert, find and delete
    '''
    st = SplayTree()
    bt = BinaryTree()
    for i in range(0, 10000):
        command = randrange(1, 10)
        random_key = randrange(1, MAX_KEY)
        if command < 5:
            st.insert(random_key)
            bt.insert(random_key)
        elif command < 9:
            st.find(random_key)
            bt.find(random_key)
        else:
            st.delete(random_key)
            bt.delete(random_key)
    show_result(st, bt)

def measure_ordered():
    '''
    process items in ascending order
    '''
    st = SplayTree()
    bt = BinaryTree()
    for i in range(1, 1000):
        command = randrange(1, 10)
        key = i
        if command < 5:
            st.insert(key)
            bt.insert(key)
        elif command < 9:
            key = randrange(1, i+1)
            st.find(key)
            bt.find(key)
        else:
            key = randrange(1, i+1)
            st.delete(key)
            bt.delete(key)
    show_result(st, bt)

def measure_nd():
    '''
    process items with key generatedy by peudo normal distribution
    '''
    st = SplayTree()
    bt = BinaryTree()
    keys = randint_nd(1, 10000, 100000)
    for random_key in keys:
        command = randrange(1, 10)
        if command < 5:
            st.insert(random_key)
            bt.insert(random_key)
        elif command < 9:
            st.find(random_key)
            bt.find(random_key)
        else:
            st.delete(random_key)
            bt.delete(random_key)
    show_result(st, bt)

if __name__ == '__main__':
    print('# measure_random_insert')
    measure_random_insert()
    print('# measure_random')
    measure_random()
    print('# measure_ordered')
    measure_ordered()
    print('# measure_nd')
    measure_nd()
```

上で使用している`randint_nd`関数の実装は以下の通り。

```python
import scipy.stats as ss
import numpy as np
import matplotlib.pyplot as plt

'''
# original code from http://stackoverflow.com/questions/37411633/how-to-generate-a-random-normal-distribution-of-integers
x = np.arange(-10, 11)
xU, xL = x + 0.5, x - 0.5

# norm means normal distribution
# cdf means 'cumulative density function'(累積分布関数)
# cdf(x: position, loc: expected value, scale: standard deviation)
prob = ss.norm.cdf(xU, scale = 3) - ss.norm.cdf(xL, scale = 3)
prob = prob / prob.sum()
print(prob)

nums = np.random.choice(x, size = 10000, p = prob)
plt.hist(nums, bins = len(x))
plt.show()
'''

def randint_nd(start, end, num=1, show_hist=False):
    # set random range, expected value, and pseudo standard deviation
    r_range = int(end - start)
    r_av = int((end + start) / 2)
    r_sd = int(r_range / 6)

    # get random value
    x = np.arange(-1 * (r_range / 2), (r_range / 2) + 1)
    xU, xL = x + 0.5, x - 0.5
    prob = ss.norm.cdf(xU, scale = r_sd) - ss.norm.cdf(xL, scale = r_sd)
    prob = prob / prob.sum()
    nums = np.random.choice(x, size = num, p = prob) + r_av

    # show histgram
    if show_hist:
        plt.hist(nums, bins = len(x))
        plt.show()
    return nums.tolist()
```

出力結果は以下の通り。

```
# measure_random_insert
count of splay tree: 63120
count of binary tree: 63120
max_level of splay tree: 47
max_level of binary tree: 37
# measure_random
count of splay tree: 4343
count of binary tree: 4343
max_level of splay tree: 32
max_level of binary tree: 28
# measure_ordered
count of splay tree: 422
count of binary tree: 422
max_level of splay tree: 28
max_level of binary tree: 422
# measure_nd
count of splay tree: 6535
count of binary tree: 6535
max_level of splay tree: 33
max_level of binary tree: 30
```

結果としてあまり思ったような結果が出ていない。原因としてありそうなのは

1. splay treeの実装に誤りがある
2. 使用しているデータが不適切

かなと。

1.に関しては既存のsplay treeの実装をそのまま試しても同様な結果だったのでとりあえず可能性として除外する。

2.に関してはそもそも使用するデータがランダムであれば二分木は何もしなくてもバランスが取れるよね、ということ。順序の整ったデータを渡せばそりゃ差はできる(`measure_ordered`関数の例)けど、それも比較としてはね...という感じ。苦し紛れに擬似的に正規分布状の分布をとるような整数値を与えるランダム関数(`randint_nd`)を定義し、使用してみたけれど結果は変わらず。Wikipediaの「[スプレー木][4]」項目には

> 一様なアクセスが発生するような場合、スプレー木は単純な平衡2分探索木よりもずっと性能が悪くなる。

という記述があるので、スプレー木は「キャッシュ化して効果を発揮するようなデータ」に対して構築するのがベターということだけど、そういったデータが手元にないので試してみることはできない...

splayの最大のメリットは頻繁にアクセスするノードが木の上部に集まりやすいということであり、木の左右のバランスが調整されるのは副次的な効果という感じに捉えるのがいいのかもしれない。

### 参考

- [Algorithm with Python/スプレー木][1]

[1]: http://www.geocities.jp/m_hiroi/light/pyalgo21.html
[2]: https://gist.github.com/tiqwab/02cebe3619bfdc5a53547bb22d7762cd
[3]: https://gist.github.com/dbad2ba205ff74f750d037194918d442
[4]: https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%97%E3%83%AC%E3%83%BC%E6%9C%A8
