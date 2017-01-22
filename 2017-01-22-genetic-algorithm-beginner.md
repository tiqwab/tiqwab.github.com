---
layout: post
title: 遺伝的アルゴリズムおさわり
tags: "genetic algorithm, python, java"
comments: true
---

最近『[思考する機械コンピュータ][6]』や『[盲目の時計職人][7]』を読み、影響されやすい人間なので進化計算って面白そうだなーやってみたいなーと思いました。
Wikipediaを見ると進化的計算と呼ばれるアルゴリズムにはいくつかあるようですが、一番始めやすそうなものが遺伝的アルゴリズムかなと感じたので、ここではそれを使ってOneMax問題やぷよぷよの19連鎖構築をやってみました (後者はうまくいっていませんが)。

1. 遺伝的アルゴリズム(GA)概要
2. PythonでOneMax
3. jeneticsでOneMax
4. ぷよぷよ19連鎖をGAで構築する(したかった)

### 環境

- OS: Arch Linux (linux kernel: 4.8.13)
- Java: openjdk version "1.8.0\_112"
- Python: Python 3.6.0

### 1. 遺伝的アルゴリズム(GA)概要

遺伝的アルゴリズムとは何か、どのように機能するか、については参考になる資料が数多く見つかるのでここでは省略します。
今回参考にしたものをいくつか下記に挙げておきます。

- [The Evolution of Code (Chapter 9, Nature of Code)][3]
  - すごい丁寧な説明
    - ある程度内容がわかっている場合、むしろ冗長に感じるかも
- [遺伝的アルゴリズムでナップザック問題を攻略][4]
  - 実際に遺伝的アルゴリズムを問題に適用したコードの全体が載せられている
- [遺伝的アルゴリズム][5]
  - 推奨値の話など実際にどのように遺伝的アルゴリズムを適用しようか考えるときに参考になる
  - 割と淡々としているので、一から調べようとしているときには少し違うかも

『盲目の時計職人』の中でリチャード・ドーキンスは「累積淘汰の基本要素は複製、誤り、そして力」 という記述をしています。
遺伝的アルゴリズムでは複製はそのまま遺伝型として扱っている配列等をコピーすること、誤りは故意に加えた突然変異の操作、力は評価関数を通して計測される評価値ということになるのかなと思っています。

### 2. PythonでOneMax

OneMaxというのは遺伝的アルゴリズムの簡単な例としてよく使われるもので、0 or 1から構成された配列をすべてが1になるように進化させることが目標になります。
ここではOneMaxを遺伝的アルゴリズムで解くプログラムをPythonで実装してみます。

遺伝的アルゴリズムの中心となる概念は以下のようにある世代の集合から評価、選択、変異により次世代を作ることであり、これを指定された回数、または目的の表現型を持つ遺伝型が得られるまで繰り返すことがアルゴリズムの基本的な流れとなります。

```python
def cycle(self, chroms):
    # evaluate
    generation = self.evaluate(chroms)
    # select
    selected, elites = self.select(generation)
    # mate
    mated = self.mate(selected)
    # mutate
    mutated = self.mutate(mated)
    return mutated + [x.chrom for x in elites]
```

ここでは評価、選択、変異といった操作は問題によって変更できるように別メソッドで実装していますが、このように問題毎に考えなければならない要素としては以下のようなものが挙げられます。

- 遺伝子の表現方法 (どういったデータ型をとらせるか)
- 評価関数 (その個体がどのくらい目標に近いのか)
- 交叉法
- 突然変異
- 集団の大きさ
- 世代数

#### 遺伝子の表現方法

遺伝子、クロモソーム(分子生物学由来の用語ですが、ここでは単純に遺伝子の配列程度に考える)のデータ構造を考えます。

- 表現型は遺伝型に依存する (遺伝型から表現型が決定される)
- 遺伝型は表現型に依存しない (表現型が直接遺伝型に影響を与えてはならない)
- 変異は遺伝型に対して適用される

例えば上の参考ページの一つが扱っているナップザック問題では遺伝型は0 or 1で構成される配列、表現型はナップザックの中身ということになるかと思います。

OneMaxでは表現型も遺伝型も0 or 1で構成される整数値の配列と考えられるので、遺伝子は整数値0 or 1を持つGeneとして定義し、そのリストをChromosomeとして定義します。

```python
class Gene:
    def __init__(self, allele):
        assert isinstance(allele, int)
        assert allele == 0 or allele == 1
        self.allele = allele

class Chromosome:
    def __init__(self, genes):
        assert isinstance(genes, list)
        self.genes = genes
```

#### 評価関数

遺伝型をもとに表現型を計算し、その表現型に対し適用値を評価関数により算出します。

- 評価関数が評価するのは表現型で、遺伝型を見ることがあってはならない

OneMaxでは全てが1の配列を目指しているので、単純に1の個数を評価値として返すことにします。

```python
def onemax_evaluator(chrom):
    '''
    Evaluate a chromosome
    '''
    return len(list(filter(lambda x: x == 1, map(lambda x: x.allele, chrom.genes))))
```

#### 交叉

選択された個体同士から新たな個体を生み出す操作で、変異を加える操作の一種です。
最も単純な交叉法としては2つの個体が持つ遺伝子をそれぞれ同じポイントで2分割し、交換する一点交叉があり、今回はこれを採用しています。

```python
class Chromosome:
    def __init__(self, genes):
        assert isinstance(genes, list)
        self.genes = genes

    def crossover(self, other):
        '''
        Perform single-point crossover at the randomized site.
        '''
        genes1 = [x.copy() for x in self.genes]
        genes2 = [x.copy() for x in other.genes]
        point = randint(1, len(genes1)-2)
        chrom1 = Chromosome(genes1[:point] + genes2[point:])
        chrom2 = Chromosome(genes2[:point] + genes1[point:])
        return (chrom1, chrom2)
```

#### 突然変異

選択された個体が持つ遺伝子を一定の確立でrandom化して新たな遺伝子を得る操作で、変異を加えるための操作の一種です。
この操作がない場合、探索空間が初期状態由来のものに限定されてしまいますが、逆に高確率で突然変異を起こしてしまうと、評価値の高い個体が増えにくくなり進化が進まなくなります。

一般には0.1%-2%程度がよいとされているようですが、問題や遺伝子のコード化手法による部分も大きそうなので、適宜調整が必要な項目だと思います。今回のOneMaxでは1%としました。

#### 集団の大きさ

いくつの個体で進化を行うか、という要素も進化の効率に影響を与えるようです。集団の中で競い合わせることでより効率の良い進化を行わせることができますが、これも多すぎると逆に効率が下がるようです。

一般には10-100程度とされているようですが、これも問題や遺伝子のコード化手法の影響が大きそうです。ただ極端な値ではない限りこれによって進化が進まなくなるということはなさそうなのであまり深く考えなくてもよいのかなと思います。

#### 世代数

何世代進化を行わせるかで、これはもうやれるだけやればいいと思います。

最終的にこれらの考察を踏まえ、Python実装を作成しました。コードは[こちら][1]で実行結果例は以下の通りです。
Min, Max, Avgはそれぞれの世代の評価値の最小、最大、平均を表しており、100世代進化の一部始終を載せていますが、順調に40代目あたりで目的のものが生成されていることがわかります。

```
1 -> Min: 7, Max: 13, Avg: 10.200000
2 -> Min: 6, Max: 13, Avg: 9.909091
3 -> Min: 8, Max: 13, Avg: 11.250000
4 -> Min: 9, Max: 14, Avg: 11.153846
5 -> Min: 9, Max: 14, Avg: 10.928571
6 -> Min: 9, Max: 14, Avg: 11.466667
7 -> Min: 9, Max: 15, Avg: 11.812500
8 -> Min: 8, Max: 15, Avg: 12.411765
9 -> Min: 10, Max: 15, Avg: 12.555556
10 -> Min: 10, Max: 15, Avg: 12.736842
11 -> Min: 10, Max: 15, Avg: 13.550000
12 -> Min: 10, Max: 16, Avg: 13.380952
13 -> Min: 10, Max: 16, Avg: 13.909091
14 -> Min: 10, Max: 16, Avg: 14.217391
15 -> Min: 10, Max: 16, Avg: 14.583333
16 -> Min: 12, Max: 16, Avg: 14.720000
17 -> Min: 14, Max: 17, Avg: 15.076923
18 -> Min: 13, Max: 17, Avg: 15.000000
19 -> Min: 13, Max: 17, Avg: 15.285714
20 -> Min: 12, Max: 17, Avg: 15.034483
...
30 -> Min: 14, Max: 18, Avg: 16.307692
31 -> Min: 15, Max: 18, Avg: 16.575000
32 -> Min: 14, Max: 18, Avg: 16.585366
33 -> Min: 14, Max: 18, Avg: 16.500000
34 -> Min: 14, Max: 19, Avg: 16.767442
...
41 -> Min: 15, Max: 19, Avg: 17.020000
42 -> Min: 15, Max: 20, Avg: 17.294118
...
99 -> Min: 16, Max: 20, Avg: 18.138889
100 -> Min: 15, Max: 20, Avg: 18.422018
Best result:  Phenotype {chrom=Chromosome {genes=[1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]}, fitness=20}
```

### 3. jeneticsでOneMax

遺伝的アルゴリズムの一つの利点は自分でちゃちゃっと書けるぐらい実装がシンプルであることだと思っていたのですが、実際に自分で書いてみるとアルゴリズムがrandomnessに依存しているために動作が確認し辛くテストも書き辛いなという印象が強かったです。実は最初に実装したときには「集団の半分に対してしか交叉を実行しない」というしょうもないミスをしていたのですが、生成される遺伝子からはそれが判断できず気づくのにだいぶ時間がかかってしまいました。

ですのでとりあえず自分でOneMaxぐらいなら書けるということに満足しておいて、ここからは[jenetics][2]というJavaの遺伝的アルゴリズム用のライブラリを使用していくことにします。
いくつか他にもJavaやPythonで遺伝的アルゴリズムのライブラリは探していたのですが、jeneticsは遺伝的アルゴリズムに特化しているためにわかりやすいこと、APIが直感的で扱いやすいと思ったこと、またdocumentが(pdf化されているほどに)整備されているという点で良さそうだなと感じ選択しました。

jeneticsで上のOneMax解法を実装すると以下のようになります。

```java
package org.tiqwab.onemax;

import org.jenetics.*;
import org.jenetics.engine.Engine;
import org.jenetics.engine.EvolutionResult;
import org.jenetics.util.Factory;

import java.util.stream.Collectors;

public class OneMaxGA {

    // Fitness function.
    private static Integer calcFitness(Genotype<BitGene> gt) {
        return (int) gt.getChromosome().stream().filter(BitGene::getAllele).count();
    }

    public static void main(String[] args) {
        // Specify what kind of genes and chromosomes are used.
        // Use BitGene and BitChromosome here since 0 or 1 is enough as value of gene in this case.
        Factory<Genotype<BitGene>> gtf = Genotype.of(BitChromosome.of(/*length:*/ 20, /*prob. of 1:*/ 0.2));

        Engine<BitGene, Integer> engine = Engine
                .builder(OneMaxGA::calcFitness, gtf)
                .populationSize(50) // The size of population.
                .selector(new TournamentSelector<>(3)) // The methodology for selection
                .alterers( // The methodology to perform crossover and mutation.
                        new SinglePointCrossover<>(0.2),
                        new Mutator<>(0.15))
                .build();

        Genotype<BitGene> result = engine.stream()
                .limit(100) // The number of generation. In this case, the selection will be performed 100 times.
                .peek(er -> {
                    double averageFitness = er.getPopulation().stream()
                            .collect(Collectors.averagingDouble(Phenotype::getFitness));
                    System.out.println(String.format("%d -> Min: %d, Max: %d, Avg: %f",
                            er.getGeneration(),
                            er.getWorstFitness(),
                            er.getBestFitness(),
                            averageFitness));
                })
                .collect(EvolutionResult.toBestGenotype());

        System.out.println("Best gene: " + result.toString());
    }

}
```

遺伝子(Gene)、遺伝型(Genotype)、表現型(Phenotype)、集団(Population)といった概念が直接クラスとして表されていて、個人的にはとても扱いやすいです。また進化を行う実装としてJavaSE 1.8から導入されているstreamを採用していることもjeneticsの特徴だと思います。

### 4. ぷよぷよ19連鎖をGAで構築する(したかった)

OneMaxは遺伝的アルゴリズムの感覚を掴むには手軽でよいのですが、さすがに単純すぎるのでもう少し面白そうな問題にも適用できないかと考えました。

ここでは少し前に見た「[ぷよぷよ19連鎖の様子を出力する][8]」という記事から発想を受け、それではこの逆でぷよぷよ19連鎖を遺伝的アルゴリズムで作成できないかと思いやってみることにしました。

- 遺伝子のデータ型
  - ぷよぷよの色を整数値として持つ二重配列
- 評価関数
  - 上の二重配列を受け取り、連鎖回数を評価値として返す関数

のように考えればOneMaxの少し発展ver.として実装できるのではと考えられます。

#### ぷよぷよ評価関数の構築

ぷよぷよ19連鎖を生成するための評価関数としては、上の通り直接連鎖数を評価値とする方向で考えます。

遺伝型をぷよぷよの色を表す整数値からなる二重配列として持つので、評価関数内ではこれをぷよぷよの盤面とみなし、連鎖の様子を把握できればよいということになります。

連鎖の評価の仕方は色々考えられると思うのですが、ここではぷよぷよのパターンも盤面も比較的限られているということで、一番考えやすい(しかし一番効率は悪いかもしれませんが)、「ぷよぷよ4つからなる全てのブロックパターンで盤面をさらい、4つとも同じぷよぷよならば消去する」という方法でいきました。

ちなみにこれは出題元のページでは2時間以内に解くことが期待されている問題のようですが、余裕で時間をoverしました。まあ使用したエディタに慣れていなかったとか、(参考にした19連連鎖の都合で)画面外のぷよぷよの概念を入れる必要があったとか苦し紛れに言い訳はできますが、それがなくても2時間はキツそうだったのでまだまだ実力が足りないなという感じです。

使用した19連鎖の盤面と、実行の様子は以下のようになります (なおcolors[0]は盤面の最上段を表しますが、画面外という扱いになっており、連鎖の処理では無視されています)。

```java
@Test
public void cyclesTest() {
    // setup
    // 19連鎖を行う盤面を表す二重配列。0は空、1,2,3はそれぞれ違う色のぷよぷよ
    final int[][] colors = {
            { 1, 2, 0, 0, 2, 2 },
            { 1, 1, 1, 3, 2, 1 },
            { 3, 1, 2, 2, 3, 2 },
            { 1, 2, 3, 3, 2, 1 },
            { 2, 3, 2, 1, 3, 1 },
            { 3, 2, 1, 3, 2, 1 },
            { 2, 3, 2, 1, 3, 2 },
            { 2, 3, 2, 1, 3, 2 },
            { 2, 1, 1, 3, 2, 3 },
            { 1, 2, 3, 1, 3, 3 },
            { 3, 1, 2, 3, 1, 2 },
            { 3, 1, 2, 3, 1, 2 },
            { 3, 1, 2, 3, 1, 2 },
    };
    // Stageは盤面を表す。引数は上の二重配列と、画面に実際に表示される有効な段数
    Stage stage = new Stage(colors, colors.length - 1);
    // BoardScannerで盤面をスキャンし、ぷよぷよが消せるか判断する。
    List<BoardScanner> scanners = Arrays.asList(
            OneByFourScanner.newDefaultInstance(), FourByOneScanner.newDefaultInstance(),
            TwoByThreeScanner.newDefaultInstance(), ThreeByTwoScanner.newDefaultInstance(),
            TwoByTwoScanner.newDefaultInstance()
    );
    // Solverで連鎖評価を実行する
    Solver solver = new Solver(stage, scanners);

    // exercise
    CycleResult cr = solver.cycles();

    // verify
    assertTrue(cr.board.isAllEmpty()); // ぷよぷよが盤面に残っていないことの確認
    assertEquals(19, cr.chainInfo.chainCount); // 19連鎖が行われたことの確認
}
```

実行結果です。

```
15:40:33.372 [main] DEBUG org.tiqwab.puyopuyo.Stage - 0: 
1 2 0 0 2 2
1 1 1 3 2 1
3 1 2 2 3 2
1 2 3 3 2 1
2 3 2 1 3 1
3 2 1 3 2 1
2 3 2 1 3 2
2 3 2 1 3 2
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.431 [main] DEBUG org.tiqwab.puyopuyo.Stage - 1: 
0 0 0 0 2 2
1 0 0 3 2 1
3 2 2 2 3 2
1 2 3 3 2 1
2 3 2 1 3 1
3 2 1 3 2 1
2 3 2 1 3 2
2 3 2 1 3 2
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.439 [main] DEBUG org.tiqwab.puyopuyo.Stage - 2: 
0 0 0 0 2 2
1 0 0 0 2 1
3 0 0 3 3 2
1 0 3 3 2 1
2 3 2 1 3 1
3 2 1 3 2 1
2 3 2 1 3 2
2 3 2 1 3 2
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.447 [main] DEBUG org.tiqwab.puyopuyo.Stage - 3: 
0 0 0 0 0 2
1 0 0 0 2 1
3 0 0 0 2 2
1 0 0 0 2 1
2 3 2 1 3 1
3 2 1 3 2 1
2 3 2 1 3 2
2 3 2 1 3 2
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.471 [main] DEBUG org.tiqwab.puyopuyo.Stage - 4: 
0 0 0 0 0 0
1 0 0 0 0 2
3 0 0 0 0 1
1 0 0 0 0 1
2 3 2 1 3 1
3 2 1 3 2 1
2 3 2 1 3 2
2 3 2 1 3 2
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.475 [main] DEBUG org.tiqwab.puyopuyo.Stage - 5: 
0 0 0 0 0 0
1 0 0 0 0 0
3 0 0 0 0 0
1 0 0 0 0 0
2 3 2 1 3 0
3 2 1 3 2 2
2 3 2 1 3 2
2 3 2 1 3 2
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.479 [main] DEBUG org.tiqwab.puyopuyo.Stage - 6: 
0 0 0 0 0 0
1 0 0 0 0 0
3 0 0 0 0 0
1 0 0 0 0 0
2 3 2 1 0 0
3 2 1 3 3 0
2 3 2 1 3 0
2 3 2 1 3 0
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.492 [main] DEBUG org.tiqwab.puyopuyo.Stage - 7: 
0 0 0 0 0 0
1 0 0 0 0 0
3 0 0 0 0 0
1 0 0 0 0 0
2 3 2 0 0 0
3 2 1 1 0 0
2 3 2 1 0 0
2 3 2 1 0 0
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.503 [main] DEBUG org.tiqwab.puyopuyo.Stage - 8: 
0 0 0 0 0 0
1 0 0 0 0 0
3 0 0 0 0 0
1 0 0 0 0 0
2 3 0 0 0 0
3 2 2 0 0 0
2 3 2 0 0 0
2 3 2 0 0 0
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.507 [main] DEBUG org.tiqwab.puyopuyo.Stage - 9: 
0 0 0 0 0 0
1 0 0 0 0 0
3 0 0 0 0 0
1 0 0 0 0 0
2 0 0 0 0 0
3 3 0 0 0 0
2 3 0 0 0 0
2 3 0 0 0 0
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.510 [main] DEBUG org.tiqwab.puyopuyo.Stage - 10: 
0 0 0 0 0 0
0 0 0 0 0 0
1 0 0 0 0 0
3 0 0 0 0 0
1 0 0 0 0 0
2 0 0 0 0 0
2 0 0 0 0 0
2 0 0 0 0 0
2 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.513 [main] DEBUG org.tiqwab.puyopuyo.Stage - 11: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
1 0 0 0 0 0
3 0 0 0 0 0
1 1 1 3 2 3
1 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.516 [main] DEBUG org.tiqwab.puyopuyo.Stage - 12: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
1 0 0 3 2 3
3 2 3 1 3 3
3 1 2 3 1 2
3 1 2 3 1 2
3 1 2 3 1 2

15:40:33.520 [main] DEBUG org.tiqwab.puyopuyo.Stage - 13: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 3 2 3
0 2 3 1 3 3
0 1 2 3 1 2
0 1 2 3 1 2
1 1 2 3 1 2

15:40:33.534 [main] DEBUG org.tiqwab.puyopuyo.Stage - 14: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 3 2 3
0 0 3 1 3 3
0 0 2 3 1 2
0 0 2 3 1 2
0 2 2 3 1 2

15:40:33.551 [main] DEBUG org.tiqwab.puyopuyo.Stage - 15: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 3 2 3
0 0 0 1 3 3
0 0 0 3 1 2
0 0 0 3 1 2
0 0 3 3 1 2

15:40:33.553 [main] DEBUG org.tiqwab.puyopuyo.Stage - 16: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 2 3
0 0 0 0 3 3
0 0 0 0 1 2
0 0 0 3 1 2
0 0 0 1 1 2

15:40:33.563 [main] DEBUG org.tiqwab.puyopuyo.Stage - 17: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 3
0 0 0 0 0 3
0 0 0 0 0 2
0 0 0 0 2 2
0 0 0 3 3 2

15:40:33.565 [main] DEBUG org.tiqwab.puyopuyo.Stage - 18: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 3
0 0 0 3 3 3

15:40:33.571 [main] DEBUG org.tiqwab.puyopuyo.Stage - 19: 
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
0 0 0 0 0 0
```

#### 遺伝的アルゴリズムの適用

ぷよぷよ評価関数が作成できたので、これを使用して遺伝的アルゴリズムを回してみます。ぷよぷよ用に調整したアルゴリズムの概要は以下のような感じです。

- 遺伝子の表現方法 (どういったデータ型をとらせるか)
  - 整数値からなる二重配列。0はぷよぷよが存在しないこと、1,2,3はそれぞれ別色のぷよぷよを表す
- 評価関数 (その個体がどのくらい目標に近いのか)
  - 上で作成した評価関数を使用する
- 交叉
  - 二点交叉...だがこれは検討の余地がありそう
  - 二重配列に対して、そしてぷよぷよという概念に対して単純に交叉をかけるのはイマイチな気がしている
- 突然変異
  - jeneticsの標準値 = 0.2を使用
    - ざっとjeneticsのソースを見た限りだと、ぷよぷよの各行に変異を入れる確立が(0.2)^(1/3)で、その行の各ぷよぷよに変異を入れる確立が0.2ということになっている？
- 集団の大きさ
  - 50
- 世代数
  - 2000

実際のコードは以下のようになりました (全体は[こちら][9])。

```java
public class PuyopuyoGA {

    private static Integer puyopuyoEvaluator(Genotype<IntegerGene> gt) {
        int[][] colors = convertGenotypeToColors(gt);
        Stage stage = new Stage(colors, colors.length - 1);
        Solver solver = new Solver(stage, scanners);
        CycleResult cr = solver.cycles();
        return cr.chainInfo.chainCount;
    }

    public static void main(String[] args) throws Exception {
        ch.qos.logback.classic.Logger rootLogger = (ch.qos.logback.classic.Logger) LoggerFactory.getLogger(Logger.ROOT_LOGGER_NAME);
        rootLogger.setLevel(Level.INFO/*DEBUG*/);
        Factory<Genotype<IntegerGene>> gtf = Genotype.of(
                IntStream.range(0, 13).mapToObj(x -> IntegerChromosome.of(1, 3, 6)).collect(Collectors.toList())
        );

        Engine<IntegerGene, Integer> engine = Engine
                .builder(PuyopuyoGA::puyopuyoEvaluator, gtf)
                .alterers(new MultiPointCrossover<>(MultiPointCrossover.DEFAULT_ALTER_PROBABILITY), new Mutator<>(Mutator.DEFAULT_ALTER_PROBABILITY))
                .build();

        Genotype<IntegerGene> result = engine.stream()
                .limit(2000)
                .peek(gt -> {
                    double avg = gt.getPopulation().stream().map(x -> x.getFitness()).collect(Collectors.averagingDouble(x -> x));
                    logger.info("{} -> Min: {}, Max: {}, Avg: {}", gt.getGeneration(), gt.getWorstFitness(), gt.getBestFitness(), avg);
                })
                .collect(EvolutionResult.toBestGenotype());

        System.out.println("Best gene: fitness -> " + puyopuyoEvaluator(result));
        System.out.println(formatEvolutionResult(result));
    }
}
```

これを実際に実行すると以下のようになりました。

```
1 -> Min: 1, Max: 7, Avg: 4.02
2 -> Min: 1, Max: 7, Avg: 4.08
3 -> Min: 1, Max: 7, Avg: 4.5
4 -> Min: 2, Max: 7, Avg: 4.64
5 -> Min: 1, Max: 7, Avg: 5.02
6 -> Min: 2, Max: 7, Avg: 5.18
7 -> Min: 2, Max: 7, Avg: 5.52
...
77 -> Min: 2, Max: 7, Avg: 5.38
78 -> Min: 1, Max: 7, Avg: 5.02
79 -> Min: 2, Max: 8, Avg: 5.9
80 -> Min: 2, Max: 8, Avg: 5.88
81 -> Min: 2, Max: 8, Avg: 5.76
...
264 -> Min: 2, Max: 8, Avg: 6.14
265 -> Min: 2, Max: 8, Avg: 6.04
266 -> Min: 2, Max: 8, Avg: 6.36
267 -> Min: 2, Max: 9, Avg: 6.38
268 -> Min: 2, Max: 9, Avg: 6.06
269 -> Min: 2, Max: 9, Avg: 6.06
...
511 -> Min: 3, Max: 9, Avg: 6.92
512 -> Min: 2, Max: 9, Avg: 6.46
513 -> Min: 2, Max: 9, Avg: 6.66
514 -> Min: 2, Max: 10, Avg: 7.46
515 -> Min: 2, Max: 10, Avg: 7.16
...
837 -> Min: 0, Max: 10, Avg: 7.7
838 -> Min: 1, Max: 10, Avg: 7.74
839 -> Min: 1, Max: 11, Avg: 7.36
840 -> Min: 1, Max: 11, Avg: 7.34
841 -> Min: 2, Max: 11, Avg: 7.76
...
1998 -> Min: 1, Max: 11, Avg: 8.92
1999 -> Min: 2, Max: 11, Avg: 8.66
2000 -> Min: 2, Max: 11, Avg: 7.6

Best gene: fitness -> 11
1 2 1 2 3 3
3 2 1 1 3 1
2 1 3 2 3 2
1 2 3 3 2 2
2 1 2 1 1 1
3 2 1 3 3 2
3 2 2 3 3 1
2 3 1 1 1 3
2 3 1 1 2 1
2 2 2 3 1 1
1 2 1 2 2 3
3 2 3 3 1 3
2 3 1 1 2 3
```

このように連鎖数12程度までは何とか進むのですが、何度か実行し、また世代数を増やしてもそれ以上の進化は見られませんでした。
遺伝子の表現方法や評価方法は特に問題があるわけではないと思うので消去法的に交叉法がイマイチなのではないかと思うのですが、単純に交叉法を変えた(SinglePointCrossover, MultiPointCrossover, UniformCrossover)だけでは変化が見られないため、何かぷよぷよに適した変異の形を考えないといけないのかなという気がしています。力が及ばず中途半端な結果ですがここで一旦終了としました。

### Summary

- 遺伝的アルゴリズムは自然が持つ進化の仕組みを応用したもので、直感的に理解しやすく単純な適用範囲はかなり広い
- 一方で問題に合わせて最適な評価関数や変異の入れ方を見つける必要があり、使いこなすのは中々大変そう

ぷよぷよに関してはもう少し実際の問題に遺伝的アルゴリズムを適用した例をあさってみるのが良さそうなので、まずは手元にある『[応用事例でわかる遺伝的アルゴリズムプログラミング][10]』を読んでみることにします。

### 参考

- [The Evolution of Code (Chapter 9, Nature of Code)][3]
- [遺伝的アルゴリズムでナップザック問題を攻略][4]
- [遺伝的アルゴリズム][5]

[1]: https://github.com/tiqwab/example/blob/master/ga-onemax/onemax.py
[2]: http://jenetics.io/
[3]: http://natureofcode.com/book/chapter-9-the-evolution-of-code/
[4]: http://qiita.com/simanezumi1989/items/10cfe1e8a23cd9d4c7b1
[5]: http://www.obitko.com/tutorials/genetic-algorithms/japanese/index.php
[6]: https://www.google.co.jp/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwi5wIaKsdXRAhWEzbwKHeg1BesQFggcMAA&url=https%3A%2F%2Fwww.amazon.co.jp%2F%25E6%2596%2587%25E5%25BA%25AB-%25E6%2580%259D%25E8%2580%2583%25E3%2581%2599%25E3%2582%258B%25E6%25A9%259F%25E6%25A2%25B0%25E3%2582%25B3%25E3%2583%25B3%25E3%2583%2594%25E3%2583%25A5%25E3%2583%25BC%25E3%2582%25BF-%25E8%258D%2589%25E6%2580%259D%25E7%25A4%25BE%25E6%2596%2587%25E5%25BA%25AB-%25E3%2583%2580%25E3%2583%258B%25E3%2582%25A8%25E3%2583%25AB-%25E3%2583%2592%25E3%2583%25AA%25E3%2582%25B9%2Fdp%2F4794220588&usg=AFQjCNFBTo13RUuYycWO_w7i4zAH5W7HyQ
[7]: https://www.google.co.jp/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwiBlfujsdXRAhWDXrwKHWzyDYUQFggcMAA&url=https%3A%2F%2Fwww.amazon.co.jp%2F%25E7%259B%25B2%25E7%259B%25AE%25E3%2581%25AE%25E6%2599%2582%25E8%25A8%2588%25E8%2581%25B7%25E4%25BA%25BA-%25E3%2583%25AA%25E3%2583%2581%25E3%2583%25A3%25E3%2583%25BC%25E3%2583%2589%25E3%2583%25BB%25E3%2583%2589%25E3%2583%25BC%25E3%2582%25AD%25E3%2583%25B3%25E3%2582%25B9%2Fdp%2F4152085576&usg=AFQjCNH_WLdMFna7Dwa_3HxeHKSFv34Qag&bvm=bv.144224172,d.dGc
[8]: http://okajima.air-nifty.com/b/2011/01/2011-ffac.html
[9]: https://github.com/tiqwab/example/tree/master/ga-puyopuyo
[10]: https://www.amazon.co.jp/%E5%BF%9C%E7%94%A8%E4%BA%8B%E4%BE%8B%E3%81%A7%E3%82%8F%E3%81%8B%E3%82%8B%E9%81%BA%E4%BC%9D%E7%9A%84%E3%82%A2%E3%83%AB%E3%82%B4%E3%83%AA%E3%82%BA%E3%83%A0%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0-%E5%B9%B3%E9%87%8E-%E5%BB%A3%E7%BE%8E/dp/489362136X
