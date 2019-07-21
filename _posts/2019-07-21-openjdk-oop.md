---
layout: post
title: Hotspot JVM の oop を理解する
tags: "oop, compressed oops, jvm, hotspot jvm, openjdk"
comments: true
---

OpenJDK のソースコードでよく出現する oop や圧縮 oop についてまとめました。

対象 OpenJDK の commit は [jdk11u][3] の changeset `51892:e86c5c20e188` です。

1. [oop とは](#oop)
2. [CompressedOops (圧縮 oop) とは](#compressed-oops)
3. [Hotspot JVM の oop 関連クラス](#oop-class)
4. [oop をデバッガで見てみる](#oop-debug)

<div id="oop" />

### oop とは

oop は「Hotspot VM 内でのインスタンスに対する管理ポインタ」です。Hotspot VM というのは OpenJDK の JVM 実装名称ですね。

ランタイム時にインスタンスが生成されると、そのインスタンス用にヒープ領域からメモリが割り当てられます。それを参照するための情報が oop です。この領域には例えばインスタンスフィールドであったり配列なら長さや要素が格納されます。

oop は後述の圧縮という処理を考えなければその領域へのポインタと言ってしまっていいはずです。

<div id="compressed-oops" />

### CompressedOops (圧縮 oop) とは

CompressedOops については [Hotspot JVMの圧縮OOP - Oracle][1] で解説されています。
こちらから要点を抜粋します。

- CompressedOops を行う背景
  - LP64 システムにおいて以下の要望を両立するためのアプローチ
    - ILP32 と同じ程度のヒープ使用量を維持したい
      - oop のサイズを 32 -> 64 bit にすると全体で 1.5 倍程度使用量が増える？
        - 具体的な数字の根拠は置いといて、必要量が増えるのはわかる
    - 最大ヒープサイズは大きくしたい
      - LP32 で単純に考えると 4 GB までのアドレスしか表現できない
- UseCompressedOops フラグ
  - `-XX:+UseCompressedOops`
    - oop 圧縮する
  - `-XX:-UseCompressedOops`
    - oop 圧縮しない
  - 64 bit 環境ならばデフォルトは有効
    - ただし JVM に割り当てるヒープ領域が大きいと無効になる？
    - 後述の計算方法を踏まえると多分 32 GB 以上は圧縮 oop では表現できないので無効になるのでは？
- 圧縮対象の oop
  - すべての oop 使用箇所で圧縮した値を使用するわけではない
  - ヒープに保存する oop が対象
    - すべてのオブジェクトの klass フィールド
    - すべての oop のインスタンスフィールド
    - oop 配列 (objArray) の全要素
  - 対象外のもの
    - 関数への受け渡しとか
- 圧縮アルゴリズム
  - パラメータとして base と shift (通常 3) がある
  - compressed oop から oop への計算
    - `oop = (compressed oop << shift) + base`
- ゼロベースの圧縮 oop
  - 可能であれば base, shift パラメータは 0 に設定される
    - ヒープサイズが 4 GB 未満
      - base, shift を 0
    - ヒープサイズが ~30 GB 未満
      - base を 0

<div id="oop-class" />

### Hotspot JVM 上の oop 関連クラス

次に OpenJDK ソースコード上の oop やそれに関連するクラスを見ていきます。

#### oopDesc 

in src/hotspot/share/oops/oop.hpp

- C++ overlay for any Java object
- つまり Hostspot JVM 内での Java object を表現するクラス
  - (ただ名前のせいでその理解でいいのか不安になる)

oopDesc のはじめには mark と klass という情報が入ります。それぞれのサイズは 64 bit 環境ならば mark が 8 bytes、klass は UseCompressedOops ならば 4 bytes, そうでなければ 8 bytes になります ([hospot-under-the-hood][2] の P.3)。

```c++
class oopDesc {
 ...
 private:
  markOop _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
  ...
```

markOop は markOopDesc\* のエイリアスで、細かいところは追えていないですが例えばロックに関するフラグなんかの管理に使われていそうです。

klass は自身のクラスに関する情報を指すポインタで Klass クラスについては後ほど出てきます。

これらのメンバのあとに (定義上では出現しませんが) インスタンスフィールドだったり配列であればその長さや要素が続きます。

#### oop

in src/hotspot/share/oops/oopsHierarchy.hpp

- C++ accessor for Java reference
- そのままだけど Hotspot JVM 内での oop を表現するクラス

oop は

```c++
typedef class oopDesc* oop;
```

あるいは、デバッグビルドならば

```c++
class oop {
    ...
    oopDesc *o;
    ...
    operator oopDesc*();
    ...
}
```

となっています。

#### narrowOop

in src/hotspot/share/oops/oopsHierarchy.hpp

- Offset instead of address for an oop within a java object
- 圧縮された oop を表現するクラス

narrowOop は単に juint への typedef です。
juint は環境に依りますが例えば uint32\_t へのエイリアスです。

```c++
typedef juint narrowOop;
```

#### CompressedOops

in src/hostspot/share/oops/compressedOops.inline.hpp

- Functions for encoding and decoding compressed oops
- oop を encode して narrowOop にしたりその逆 (decode) を行う関数を定義

#### Klass

in `src/hotspot/share/oops/klass.hpp`

- 以下の 2 つを提供するとしている
  - language level class object (method dictionary etc.)
  - provide vm dispatch behavior for the object
- oopDesc は自分の Klass へのポインタを持つ
- Java 仕様の static field にあたるものを管理するのも多分 Klass

#### InstanceKlass

in src/hotspot/share/oops/instanceKlass.hpp

- コメントから抜粋
  - VM level representation of a Java class
  - It contains all information needed for an class at execution runtime
- つまり Hostspot JVM 内での Java クラス表現
- 例えば実行時に constant pool の情報が必要なときはここに見に来る
- Klass を継承している

InstanceKlass と Klass の指すデータ構造はどうやら共通で、内部では Klass から cast して InstanceKlass を使用することがよくあります。

```c++
  # InstanceKlass::cast
  static InstanceKlass* cast(Klass* k) {
    return const_cast<InstanceKlass*>(cast(const_cast<const Klass*>(k)));
  }
```

<div id="oop-debug" />

### oop をデバッガで見てみる

前回せっかく [OpenJDK をデバッグビルドできる環境を作った][4] ので gdb から oop の値を確認してみます。oop は様々な箇所で利用されていますが、デバッガで見やすいのは JNI で定義した関数の jarray や jobject 型の引数かなと思います。JNI はネイティブコードを Java コードから呼び出すためのインタフェースであり、例えばここに以下のような配列を渡す関数を定義すると、 C のコードとしては jarray を受け取る関数が対応することになります。

```java
/**
 * Check oop for an array
 */
class Sample2 {
    static {
        System.loadLibrary("sample");
    }

    public static native void callArrayLength(int[] args);

    public static void main(String[] args) {
        int arr[] = {1, 2, 3};
        callArrayLength(arr);
    }
}
```

```c
#include <jni.h>

JNIEXPORT
void JNICALL Java_Sample2_callArrayLength(
  JNIEnv *env, jobject obj, jarray arr
) {
    (*env)->GetArrayLength(env, arr);
}
```

jarray は jobject のエイリアスであり、jobject は単に oop を参照するポインタのようなので、この値を見ることで渡した配列の oop が確認できます。

(jobject が oop を参照するポインタ、という根拠となるソースコードはちゃんと追えていないのですが。ただ JavaCallArguments というクラスを見ると jobject を oop\* として扱っているように見える。)

[サンプルコード][5] では `jarray arr` の oop, oopDesc を確認するような gdb コマンドを定義しており、その出力は以下のようになります。

```
$ make debug_sample2
...
gdb -x 'sample2.gdb' --args ~/workspace/openjdk11u/build/linux-x86_64-normal-server-fastdebug/jdk/bin/java  -Djava.library.path=. Sample2
...

(gdb) r
(gdb) c
...

Thread 2 "java" hit Breakpoint 1, Java_Sample2_callArrayLength (env=0x7ffff001ab90, 
    obj=0x7ffff59ef880, arr=0x7ffff59ef890) at sample2.c:4
4	    (*env)->GetArrayLength(env, arr);

# (1) print arr value
$1 = 0x7ffff59ef890

# (2) print memory where arr references
0x7ffff59ef890:	0x30	0x69	0x6f	0x19	0x07	0x00	0x00	0x00
0x7ffff59ef898:	0x98	0xf8	0x9e	0xf5	0xff	0x7f	0x00	0x00
0x7ffff59ef8a0:	0x91	0x53	0xa2	0xcd	0xff	0x7f	0x00	0x00
0x7ffff59ef8a8:	0xf8	0xf8	0x9e	0xf5	0xff	0x7f	0x00	0x00

# (3) print memory where oop references
0x7196f6930:	0x01	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7196f6938:	0x40	0x0c	0x00	0x00	0x03	0x00	0x00	0x00
0x7196f6940:	0x01	0x00	0x00	0x00	0x02	0x00	0x00	0x00
0x7196f6948:	0x03	0x00	0x00	0x00	0x00	0x00	0x00	0x00
```

(1) で渡した配列に対応する jarray の値が 0x7ffff59ef890 であることがわかります。これが oop を指すポインタなのでその先を見に行ったのが (2) でここから oop の値が `0x0000007196f6930` であることがわかります。(3) は oop の指し示すアドレスを見に行ったものです。内容としては

- 0x7196f6930-0x7196f6937 
  - mark (= 0x01)
- 0x7196f6938-0x7196f693b
  - klass (= 0x0c40)
- 0x7196f693c-0x7196f693f
  - length (=3)
- 0x7196f6940-0x7196f694b
  - 配列の要素 (1, 2, 3)

となります。

この klass の値からは自身のクラスに対応する Klass を辿るためのポインタなのですが、oop と同様圧縮されています。計算方法自体は oop と同じなのですが base, shift のパラメータ値が異なり、その値は Universe というクラスで管理されています。

```
(gdb) p Universe::_narrow_klass._base
$2 = (address) 0x800000000 ""
(gdb) p Universe::_narrow_klass._shift
$3 = 0
(gdb) p Universe::_narrow_oop._base
$4 = (address) 0x0
(gdb) p Universe::_narrow_oop._shift
$5 = 3
```

この情報から Klass にアクセスすることができます。

```
(gdb) p ((Klass *) (((long) (0x00000c40)) + Universe::_narrow_klass._base))->_name->as_C_string()
$7 = 0x7ffff0019390 "[I"
```

今回 oop を確認するために JNI を利用しましたが、JNI 用の関数を定義する `jni.h`, `jni.cpp` は 他にも HotspotVM のデータ構造を理解するのに参考になりそうです。

[1]: https://www.oracle.com/technetwork/jp/articles/java/compressedoops-427542-ja.html
[2]: https://www.infoq.com/presentations/hotspot-memory-data-structures/
[3]: http://hg.openjdk.java.net/jdk-updates/jdk11u
[4]: {% post_url 2019-07-08-openjdk-build %}
[5]: https://github.com/tiqwab/example/tree/master/openjdk-oop/jni1
