---
layout: post
title: メモリコンシステンシについて
tags: "memory consistency, memory order"
comments: true
---

現代のプログラミングではよりよい性能のためにマルチコア、マルチスレッドで動作するコードを書くというのは珍しくないと思います。多くのプログラミング言語ではユーザに利用しやすい形でそのための言語機能や標準ライブラリを提供しています。ですが抽象化のレイヤを降りて、例えば CPU のレベルで考えると同期化のために難しい概念というのが多数登場します。

ここではその中で「メモリコンシステンシ」について自分の理解をまとめました。これは言語によっては完全に隠された概念だと思うのですが C++ や Rust ではアトミックな操作に [std::memory\_order][1] や [std::sync::atomic::Ordering][2] というパラメータを渡すことで、操作が従うべきメモリコンシステンシモデルを指定できます。最初のうちはそう言われても? という感じだったのですが、いくつか良質な資料に出会い徐々に理解ができてきたので、それらを紹介しつつ memory\_order の背景にあるモデルを確認したいと思います。

### メモリコンシステンシモデルとは

> メモリアクセス命令の実行完了の順序変更にどのような制限を前提とするか（どのような順序変更が行われうると覚悟しなければいけないか）を定める

ここでのメモリアクセスというのは具体的には read と write に分けられ、

- read 完了: 値を読み終える
- write 完了: 書き込んだ値を他のPE (命令列) が観察できる

のように定義されます。

(引用元は [メモリコンシステンシモデル][3] の P.2,3、一部自分で捕捉を追加)

### 直感に合う、けれど遅い sequential consistency

上の説明だけではあまりピンと来ないので、まずはじめにメモリコンシステンシモデルとして最もわかりやすいであろう sequential consistency (以下 SC) について見ていきます。以下のような (疑似) コードをシングルスレッドで実行する場合を考えてみます。

```
// example1
// initially a = b = 0
a = 10;
b = 20;
print b;
print a;
```

example1 を見たプログラマは上から逐次的に処理が行われ「20, 10」という出力が得られると期待するでしょうし実際そうなります。コンパイラや CPU の最適化により命令の順番が変化したり、メモリアクセスがレジスタの操作のみで処理されたりといったことはありますが、そのような最適化はあくまで最終的な出力に影響を与えない形で行われます。

SC はこのような期待を並行プログラミングの世界に拡張したものだと理解できると思います。[Shared Memory Consistency Models: Tutorial][5] では SC の必要要件として以下の 2 つを挙げています。

- program order requirement
  - > maintaining program order among operations from a single processor
- write atomicity requirement
  - > maintaining a signle sequential order among all operations

program order requirement とは上で見たように「プログラムが記述された順序で逐次的に実行される」ということだと認識しています。例えば以下の example2 について、(1) -> (2) や (3) -> (4) という順序で命令が実行される必要があり、(2) -> (1) や (4) -> (3) は許されないということです。

```
// example2
// initially a = b = 0

// program2-1
a = 10; // (1)
b = 20; // (2)

// program2-2
print b; // (3)
print a; // (4)
```

write atomicity requirement はプログラムの実行順序が全てのプロセッサで共有される、ということだと認識しています。例えば example3 のプログラムは SC においては必ず C には 1 が入ります。`C = A` が実行されるためには先に `B = 1` が実行される必要があり、そのためには `A = 1` の実行が必要、という順序関係が成立し、全プロセッサがその順序でメモリ操作を見ることが保証されるためです。

```
// example3
// initially A = B = C = 0

// program3-1
A = 1;

// program3-2
while (A != 1) {}
B = 1;

// program3-3
while (B != 1) {}
C = A;
```

この 2 つの必要要件は別の言い方をすれば example2 において実行順序が (1) -> (2) -> (3) -> (4) や (3) -> (1) -> (2) -> (4) のようにそれぞれのプログラム内の命令の順序を保ったままマージされたもの (ここでは 6 通りある) の一つになり、それが全プロセッサに共有されるともいえます。

このように SC はプログラマの直感に合うものでわかりやすいのですが、一方でそれを保証するための制限がきつく、コンパイラや CPU の最適化の余地があまりないという問題があります。[Memory Consistency][4] の P.24 では SC の性能評価の一例が記載されていますが、これによれば CPU は 60-80 % の時間を I/O 待ちや同期待ちで費やさなければならないようです。

そのために SC よりも制限を緩めたメモリコンシステンシモデルというのがいくつも提案、実装されています。

### 制限を緩めたメモリコンシステンシモデル例: TSO

そのようなモデルの一つとして TSO (Total Store Ordering) をここでは紹介します。

SC から制限を緩めるということは program order requirement か write atomicity requirement、あるいはその両方の要件を弱めるということですが、TSO では write -> read (W -> R) の reorder のみを許しています。これは先行する write よりも後続の read を完了させてしまって良いということです。他の reorder のパターン として W -> W, R -> R, R -> W といったものもありますが、TSO はこれらについては SC と同様許しません。例えば以下の example4 では SC であれば print される値として (A, B) は (1, 1), (0, 1), (1, 0) のいずれかしかありえませんが、TSO では (0, 0) という可能性もあります。

```
// example4
// initially A = B = 0

// example4-2
A = 1;
print B;

// example4-2
B = 1;
print A;
```

W -> R の reorder を許すことで CPU レベルで write のレイテンシをうまく隠蔽して性能を上げることができます (ref. [Memory Consistency][4] P.26)。

ちなみに [Memory Consistency][4] P.25 によれば x86 は TSO に近いモデルを採用しているとのことです。内容を詳しくは見ていないのですが、該当する x86 仕様の詳細は [Intel SDM][6] Vol.3 Chapter 8 Multiple-Processor Management の 8.1, 8.2 あたりだと思います。

### (余談) キャッシュコヒーレンスについて

メモリコンシステンシと (個人的に) 似た用語としてキャッシュコヒーレンスというものがあります。メモリコンシステンシが「異なる変数への保証」についての話だとすれば、キャッシュコヒーレンスは「同一変数への保証」についてだと捉えられます。[Computer Architecture: A Quantitative Approach (six edition)][8] の 5.2 Centralized Shared-Memory Architectures ではシステムがキャッシュコヒーレンシを持つことを以下のように定義しています (拙訳)。

- あるプロセッサ P が変数 X に書き込んだ値は、そのあとのプロセッサ P の X からの読み取りで確認できる
- あるプロセッサ P が変数 X に書き込んだ値は、十分な時間が経てば別のプロセッサ Q の X からの読み取りで確認できる
- 1 つ以上のプロセッサが行う同一変数への複数 write の結果が全プロセッサで同一順序に観察できる

例えば example5 のように異なるスレッドで A に異なる値を書き込んでも、全コアで見える値は 1 か 2 のどちらかに統一され、1 が見えるコアと 2 が見えるコアというのが共存することはありません。

```
// example5

// example5-1
A = 1;

// example5-2
A = 2;
```

大抵の CPU ではこの性質を保証するように CPU を設計、実装するらしく、それぐらい基本的な保証だということだと思います。そのためこれまでの例でも、この先の例でも、キャッシュコヒーレンスが保証された CPU だということを前提にして記述しています。

### release consistency モデル

C++ の [memory order][1] に指定できる値として acquire, release というものがありますが、この背景にあるメモリコンシステンシモデルが release consistency (RC) モデルと呼ばれるものです。

このモデルでは TSO で緩めた W -> R だけでなく W -> W, R -> W, R -> R 全ての reorder を認めています。そのためコンパイラや CPU の最適化を行う余地は多くあるのですが、それだけだとユーザの意図通りにプログラムが実行される保証が無いので、RC では acquire 付き read、release 付き write と呼ばれるような命令を定義しています。

acquire はいわば critical section に入るための命令 (lock 操作) であり、

- acquire 後続の read, write は acquire 完了後でなければ実行できない
- acquire は先行する read や write の完了を待たず実行していい

という性質を持ちます。

release はその対になる操作であり、
 
- release は先行する read, write 完了後でなければ実行できない
- release 後続の read, write は release の完了を待たず実行してよい

という unlock にあたるような命令です。

疑似コードよりも実際の C++ のコードの方がわかりやすそうなので、memory order の公式ドキュメントから引っ張ってきた以下の例を見てみます。

```c++
#include <thread>
#include <atomic>
#include <cassert>
#include <string>
 
std::atomic<std::string*> ptr;
int data;
 
void producer()
{
    std::string* p  = new std::string("Hello");
    data = 42;
    ptr.store(p, std::memory_order_release); // release
}
 
void consumer()
{
    std::string* p2;
    while (!(p2 = ptr.load(std::memory_order_acquire))) // acquire
        ;
    assert(*p2 == "Hello"); // never fires
    assert(data == 42); // never fires
}
 
int main()
{
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join(); t2.join();
}
```

この例では

1. producer の data への write
2. producer の ptr への write (release)
3. consumer の ptr の read (acquire)
4. consumer の data の read

という順序関係が成立し、consumer で 42 が読めることが保証されます。このように memory\_order を適切に使用することでコンパイラや CPU の最適化の余地を残しつつ必要な箇所で同期を行わせるということができます。

memory\_order には似たようなものとして memory\_order\_acq\_rel というものもありますがこれは acquire, release 両方の性質を持つもので、主に read-modify-write を実行する際に指定します。

ちなみに RC といってもいくつか種類があるらしく、それによって write atomicity requirement の扱いが異なっているようです (ref. [Shared Memory Consistency Models: Tutorial][5])。

### memory\_order\_seq\_cst について

C++ の memory\_order を使用したアトミック操作ではデフォルト値として seq\_cst つまり SC が使用されます。これは acquire, release あるいは acq\_rel と同様の order requirement を保証しつつ、かつ seq\_cst を指定した操作同士の write order requirement を保証するというものです。

memory\_order ドキュメントでは seq\_cst では動作するがその他の memory\_order では正しく動作しない例を挙げています。これが acquire, release で動作しないことがあるのは、thread c と thread d で x と y への変更が見える順序が逆転する可能性があるためです。

```c++
#include <thread>
#include <atomic>
#include <cassert>
 
std::atomic<bool> x = {false};
std::atomic<bool> y = {false};
std::atomic<int> z = {0};
 
void write_x()
{
    x.store(true, std::memory_order_seq_cst);
}
 
void write_y()
{
    y.store(true, std::memory_order_seq_cst);
}
 
void read_x_then_y()
{
    while (!x.load(std::memory_order_seq_cst))
        ;
    if (y.load(std::memory_order_seq_cst)) {
        ++z;
    }
}
 
void read_y_then_x()
{
    while (!y.load(std::memory_order_seq_cst))
        ;
    if (x.load(std::memory_order_seq_cst)) {
        ++z;
    }
}
 
int main()
{
    std::thread a(write_x);
    std::thread b(write_y);
    std::thread c(read_x_then_y);
    std::thread d(read_y_then_x);
    a.join(); b.join(); c.join(); d.join();
    assert(z.load() != 0);  // will never happen
}
```

他の memory\_order の指定よりも性能上劣るとはいえ、実用上はよほどのっぴきならない事情がない限り seq\_cst を使うべきという感じがします (同期化まわりのバグは怖すぎるので...)。

### x86 における memory\_order の実装

[メモリバリアを理解するために必要な3つのこと][7] では x86 において memory\_order がどのように実装されているかについて記述されています。上述したように x86 は TSO に近いメモリコンシステンシモデルを採用しているので、release/acquire あたりが自動的に保証されると言えるのだと思います。他に [x86/x64におけるメモリオーダーの話][8] も参考になりそうです。

[1]: https://en.cppreference.com/w/cpp/atomic/memory_order
[2]: https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html
[3]: http://www.hpc.is.uec.ac.jp/honda_lab/para1-19-5.pdf
[4]: http://www.cs.cmu.edu/afs/cs/academic/class/15418-s18/www/lectures/13_consistency.pdf
[5]: http://www.ai.mit.edu/projects/aries/papers/consistency/computer_29_12_dec1996_p66.pdf
[6]: https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html
[7]: https://yamasa.hatenablog.jp/entry/20110219/1298133192
[8]: https://www.amazon.co.jp/dp/B078MFDTX4/
[9]: https://github.com/herumi/misc/blob/master/cpp/fence.md
