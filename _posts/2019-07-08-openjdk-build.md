---
layout: post
title: OpenJDK のビルドやデバッグ
tags: "jvm, openjdk"
comments: true
---

最近 OpenJDK 11 のソースコードを読み始めたので、そのために行ったビルドやデバッグについてまとめます。

1. [ビルド](#openjdk-build)
2. [CLion との連携](#openjdk-clion)
3. [gdb デバッグ ](#openjdk-gdb)
4. [hsdis の用意](#openjdk-hsdis)

<div id="openjdk-build" />

### 1. ビルド

ビルドについての公式資料は [Building the JDK][1] です。分量が多いですが公式もそれはわかっていてはじめに TL;DR があるのでまずはそこだけ確認すればいいと思います。今回はデバッグビルドまで以下のような手順で行いました。

```
# ソースを落とす。
# 今回は jdk ではなく jdk-updates というリポジトリを使用している。
# 過去のこのバージョンでビルドしたいというのがあればこのリポジトリを使うのが良さそう。
#
# jdk < 10 と jdk >= 10 でリポジトリ構成が変わっているので注意。
# http://openjdk.java.net/jeps/296
#
# "stream ended unexpectedly" で失敗しまくるときは少しずつ changeset を持ってくる手もある
# https://stackoverflow.com/questions/21655574/stream-ended-unexpectedly-on-clone
$ hg clone http://hg.openjdk.java.net/jdk-updates/jdk11u

# デバッグ情報付きでビルドできるように configure
# https://programmer.help/blogs/compile-and-debug-openjdk.html
# ビルドには jdk >= 10 を要求される
$ bash ./configure --disable-warnings-as-errors --enable-debug --with-native-debug-symbols=internal

# ビルド
$ make

# ビルドした JDK の確認
$ build/linux-x86_64-normal-server-fastdebug/jdk/bin/java -version
openjdk version "11.0.4-internal" 2019-07-16
OpenJDK Runtime Environment (fastdebug build 11.0.4-internal+0-adhoc.nm.openjdk11u)
OpenJDK 64-Bit Server VM (fastdebug build 11.0.4-internal+0-adhoc.nm.openjdk11u, mixed mode)
```

make ターゲットについて、はじめ TL;DR 通りに default ではなく image を選択していました。しかしそうすると clean な状態からはうまくいくのですが incremental compile に失敗 (cpu 使用率が異常に上がり PC が固まる) していました。[Running Make][2] によると default ターゲットについては、

> This will build a minimal (or roughly minimal) set of compiled output (known as an "exploded image") needed for a developer to actually execute the newly built JDK. The idea is that in an incremental development fashion, when doing a normal make, you should only spend time recompiling what's changed (making it purely incremental) and only do the work that's needed to actually run and test your code.

と言及され、image については、

> Build the JDK image

と書かれています。正直これを読んでも default と image ターゲットの具体的な違いはわからないですがデバッグ用途なら default で十分そうなので途中からは default でビルドしています。

<div id="openjdk-clion" />

### 2. CLion との連携

自分は C, C++ の IDE として CLion を使用しているので、そちらでソースが読めるように CMakeLists.txt を手動で加えました。内容は [OpenJDKのC++コードを、CLionでデバッグする][3] を参考にしました。

CmakeLists.txt:

```cmake
cmake_minimum_required(VERSION 3.7)
project(hotspot)
include_directories(
        src/hotspot/cpu
        src/hotspot/os
        src/hotspot/os_cpu
        src/hotspot/share
        src/hotspot/share/precompiled
        src/hotspot/share/include
        src/java.base/unix/native/include
        src/java.base/share/native/include
)
file(GLOB_RECURSE SOURCE_FILES "*.cpp" "*.hpp" "*.c" "*.h")
add_executable(hotspot ${SOURCE_FILES})
```

CMakeLists.txt をプロジェクトルートに配置し、CLion から File -> Open で選択すると、いくつかエラーは残っていますが何とか補完が効く、という状態に持っていけました。

<div id="openjdk-gdb" />

### 3. gdb デバッグ

デバッグ情報付きでビルドしているので、ソースコードをもとに breakpoint や watchpoint を設定できます (下記の例は一部出力を省略しています)。

```
$ gdb --args build/linux-x86_64-normal-server-fastdebug/jdk/bin/java Hello
(gdb) b main
Breakpoint 1 at 0x10d0: file src/java.base/share/native/launcher/main.c, line 141.

(gdb) r
Starting program: build/linux-x86_64-normal-server-fastdebug/jdk/bin/java Hello
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".

Breakpoint 1, main (argc=2, argv=0x7fffffffe618)
    at src/java.base/share/native/launcher/main.c:141
141	    JLI_InitArgProcessing(jargc > 0, const_disable_argfile);

(gdb) b JavaMain
Breakpoint 2 at 0x7ffff7d6a060: file src/java.base/share/native/libjli/java.c, line 395.

(gdb) c
Continuing.
[New Thread 0x7ffff59ef700 (LWP 3535)]
[Switching to Thread 0x7ffff59ef700 (LWP 3535)]

Thread 2 "java" hit Breakpoint 2, JavaMain (_args=0x7fffffffa320)
    at src/java.base/share/native/libjli/java.c:395
395	    int argc = args->argc;

(gdb) watch StubRoutines::_call_stub_entry 
Hardware watchpoint 3: StubRoutines::_call_stub_entry

(gdb) c
Continuing.

Thread 2 "java" received signal SIGSEGV, Segmentation fault.
0x00007fffe100053b in ?? ()

(gdb) c
Continuing.

Thread 2 "java" hit Hardware watchpoint 3: StubRoutines::_call_stub_entry

Old value = (address) 0x0
New value = (address) 0x7fffe10008e4 "UH\213\354H\203\354`L\211M\370L\211E\360H\211M\350\211U\340H\211u\330H\211}\320H\211]\310L\211e\300L\211m\270L\211u\260L\211}\250\305\370\256]\240\213E\240\201\340\300\377"
StubGenerator::generate_initial (this=0x7ffff59ee8a0)
    at src/hotspot/cpu/x86/stubGenerator_x86_64.cpp:5593
5593	    StubRoutines::_catch_exception_entry = generate_catch_exception();
```

上の例では途中 Segmentation fault が発生していますが、Hotspot VM ではどうやら正常系として (?) 発生させることがあるらしく、デバッグ中は無視して c (continue) すると目的の場所に辿り着けます。

また Hotspot VM では動的なネイティブコード生成を (JIT コンパイル以外でも) かなり駆使しており、先程 watchpoint を設定した `StubRoutines::_call_stub_entry` もそうしたコードの一つです。動的生成されたコードはもちろんソース上には存在しませんが、breakpoint をアドレスに設定すれば関数呼び出し直後のレジスタやスタックぐらいは確認できます。

```
(gdb) b *0x7fffe10008e4
Breakpoint 4 at 0x7fffe10008e4

(gdb) c
...
Continuing.
[New Thread 0x7fffcd479700 (LWP 3804)]

Thread 2 "java" hit Breakpoint 4, 0x00007fffe10008e4 in ?? ()
(gdb) p /x $rbx
$6 = 0x7ffff59ee580
```

別の悩みどころとして (OpenJDK に限った話ではないですが) gdb で遊んでいるとたまに print したい対象が optimized out だと言われることがあります。

```
(gdb) b generate_call_stub(unsigned char*&) 
Breakpoint 3 at 0x7ffff73181a0: file src/hotspot/cpu/x86/stubGenerator_x86_64.cpp, line 213.

(gdb) c
Continuing.
...

Thread 2 "java" hit Breakpoint 3, StubGenerator::generate_call_stub (this=this@entry=0x7ffff59ee8a0, 
    return_address=@0x7ffff7b69908: 0x0)
    at src/hotspot/cpu/x86/stubGenerator_x86_64.cpp:213
213	    StubCodeMark mark(this, "StubRoutines", "call_stub");

(gdb) n
214	    address start = __ pc();

(gdb) p start
$1 = <optimized out>
```

これは [Stack Overflow][4] を見るとどうやらコンパイル時の最適化の結果 gdb で参照できなくなったというような状況のようです。最適化を避けるには

- 該当の変数を volatile にする
- コンパイルオブションで最適化を調整する (e.g. `-O0`)

といった方法があるそうです。今回は後者の方が都合が良いので以下のように configure し直してみたのですが変化が見られませんでした。 `make LOG=TRACE` を見るとコンパイルオプションにはちゃんと `-O0` が追加されているように見えるのですが...

```
$ bash ./configure --disable-warnings-as-errors --enable-debug --with-native-debug-symbols=internal --with-extra-cflags='-O0' --with-extra-cxxflags='-O0'
$ make clean
$ make
```

<div id="openjdk-hsdis" />

### 4. hsdis の用意

Hotspot JVM はネイティブコードの動的生成をする、という話をしましたが、hsdis (hotspot diassembly) というツールを使うことで生成されたコード (のアセンブリ) を標準出力に吐き出すことができます。

hsdis は自分でビルドする必要があり、手順は以下の感じです。

1. `cd hotspot/src/share/tools/hsdis`
2. [binutils][6] から ソースを取得 (バージョンは適当に最新のでも良さそう)
3. 解凍して `hsdis/build/binutils` として配置されるようにする
4. `make all64`。 `make all` だと多分 32 bit 向けにも作ろうとするのでいらなければ all64 が吉
5. `build/linux-amd64/hsdis-amd64.so` あたりにできたビルド産物を `<built_jdk_path>/jdk/lib/` に置く

なお現状の jdk11 の場合、[こちら][5] で言及されている修正が必要でした。

hsdis を用意した状態で例えば java コマンドに `-XX:+PrintInterpreter` や `-XX:+PrintStubCode` といった開発用オプションを渡すと動的に生成されたコードがどばっと出力されます。

```
$ build/linux-x86_64-normal-server-fastdebug/jdk/bin/java -XX:+PrintInterpreter Hello

...
----------------------------------------------------------------------
iadd  96 iadd  [0x00007f86c1021b60, 0x00007f86c1021ba0]  64 bytes

  0x00007f86c1021b60: mov    (%rsp),%eax
  0x00007f86c1021b63: add    $0x8,%rsp
  0x00007f86c1021b67: mov    (%rsp),%edx
  0x00007f86c1021b6a: add    $0x8,%rsp
  0x00007f86c1021b6e: add    %edx,%eax
  0x00007f86c1021b70: movzbl 0x1(%r13),%ebx
  0x00007f86c1021b75: inc    %r13
  0x00007f86c1021b78: movabs $0x7f86d9a15d60,%r10
  0x00007f86c1021b82: jmpq   *(%r10,%rbx,8)
  0x00007f86c1021b86: nop
  0x00007f86c1021b87: nop
  0x00007f86c1021b88: int3   
  0x00007f86c1021b89: int3   
  0x00007f86c1021b8a: int3   
  0x00007f86c1021b8b: int3   
  0x00007f86c1021b8c: int3   
  0x00007f86c1021b8d: int3   
  0x00007f86c1021b8e: int3   
  0x00007f86c1021b8f: int3   
  0x00007f86c1021b90: int3   
  0x00007f86c1021b91: int3   
  0x00007f86c1021b92: int3   
  0x00007f86c1021b93: int3   
  0x00007f86c1021b94: int3   
  0x00007f86c1021b95: int3   
  0x00007f86c1021b96: int3   
  0x00007f86c1021b97: int3   
  0x00007f86c1021b98: int3   
  0x00007f86c1021b99: int3   
  0x00007f86c1021b9a: int3   
  0x00007f86c1021b9b: int3   
  0x00007f86c1021b9c: int3   
  0x00007f86c1021b9d: int3   
  0x00007f86c1021b9e: int3   
  0x00007f86c1021b9f: int3   
...
```

例えばこれは iadd という Java byte code 用に生成されたネイティブコードです。自分は今回始めて知って面白いなと思ったのですが、Hotspot VM はデフォルトの挙動だと起動直後に各 Java byte code 毎にネイティブコードを生成し、Java byte code 実行時は可能な限りこの生成したコードだけで処理を済ますような実装になっているようです。何となくいままで Java byte code を読んで switch 分岐なんかで処理をわけて、Hotspot VM 用に用意したスタック領域を使用して実行する、ただし頻度の高いメソッドは JIT コンパイルする、みたいなイメージがあったのですが、どうやらよりアグレッシブなことをしているみたいです。JVM のパフォーマンスの良さを考えると納得ですが。

このあたりの仕組みは Hotspot VM では Template interpreter と呼ばれています。デバッグの環境も整えたことですし今後はこの Template interpreter を理解していこうと思います。

[1]: https://hg.openjdk.java.net/jdk/jdk11/raw-file/tip/doc/building.html
[2]: https://hg.openjdk.java.net/jdk/jdk11/raw-file/tip/doc/building.html#running-make
[3]: https://www.sakatakoichi.com/entry/openjdkclion
[4]: https://stackoverflow.com/questions/5497855/what-does-value-optimized-out-mean-in-gdb
[5]: https://stackoverflow.com/questions/56101856/unable-to-use-hsdis-with-openjdk-11
[6]: http://ftp.gnu.org/gnu/binutils/ 
