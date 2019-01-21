---
layout: post
title: 64 bit 環境で 32 bit C プログラムのコンパイル、実行
tags: "c"
comments: true
---

1 月になってから読み始めた [30 日 OS 自作入門][5] も 6 日目に入り、どうやらこの本の世界は現世よりも時間の進みが 2 倍遅いようだ、と気付いた頃なのですが、本を片手に試行錯誤している中で、手元の環境だと 32 bit 向けに書いた C プログラムがコンパイル、リンクできる環境が整っていないことに気が付きました。

```
$ cat hello.c 
#include <stdio.h>
#include <stdlib.h>

void main(void)
{
    printf("Hello World\n");
    exit(0);
}

$ gcc -c -Wall -march=i486 -m32 -o hello.obj hello.c
In file included from /usr/include/features.h:452,
                 from /usr/include/bits/libc-header-start.h:33,
                 from /usr/include/stdio.h:27,
                 from hello.c:1:
/usr/include/gnu/stubs.h:7:11: fatal error: gnu/stubs-32.h: No such file or directory
 # include <gnu/stubs-32.h>
           ^~~~~~~~~~~~~~~~
compilation terminated.
```

ということでこの Hello World を 32 bit 向けに実行できるように目指します。
やりたいことは

- gcc でコンパイルしオブジェクトファイルを得て
- それをリンクして実行ファイルを作る

というのを gcc で一気通貫せずに行うという感じです。

環境:

- OS: Arch Linux x86\_64 (Linux version: 4.20.2-arch1-1-ARC)
- gcc: version 8.2.1 20181127 (GCC)

### コンパイルを通す

上のエラーメッセージ自体は開発環境に 32 bit ライブラリが存在しないことによるもののようなので、まずはそれを用意します。

各 OS で具体的な手順は異なると思いますが、Arch Linux の場合は [Multlib][1] リポジトリを有効にして必要なパッケージをインストールすれば OK です。

```
$ yay -S lib32-gcc-libs
resolving dependencies...
looking for conflicting packages...

Packages (2) lib32-glibc-2.28-5  lib32-gcc-libs-8.2.1+20181127-1

Total Installed Size:  89.05 MiB
...
```

コンパイル再トライ。

```
$ gcc -c -Wall -march=i486 -m32 -o hello.obj hello.c

$ file hello.obj
hello.obj: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
```

オブジェクトファイルはできました。

### エントリポイントを変えて実行可能にする

こうしてできた `hello.obj` ですが、単純にリンクすると失敗します。

```
$ ld -m elf_i386 -o hello hello.obj
ld: warning: cannot find entry symbol _start; defaulting to 0000000008049000
ld: hello.obj: in function `main':
hello.c:(.text+0x26): undefined reference to `printf'
```

エラーとしては

- `_start` シンボルが見つからない
- `printf` シンボルが見つからない

の 2 つを指摘されているようです。

シンボルとはリンカが関数や変数を識別するときに使用する名前で、 `nm` コマンドにより確認することができます。

```
# Show value, size, class, and name of symbol
$ nm -S hello.obj
         U _GLOBAL_OFFSET_TABLE_
00000000 0000003c T main
         U printf
00000000 T __x86.get_pc_thunk.ax
```

この内 U や T で表されるシンボルクラスが結構重要で、例えば B は BSS 領域、D が初期化済データセクション、T がコードセクション、U が未定義 (動的にリンクされるはずのもの。つまり他のオブジェクトファイルや共有ライブラリに実体がいることが期待される) となります。また大文字だとグローバル、小文字だとファイルローカルなシンボルだという表現になります。

#### printf シンボルの解決

`printf` が無いよというエラーは `hello.obj` 中の `printf` シンボルが動的にリンクされるはずなのに見つからなかったということを言っていたようです。

これを解決するには `printf` を提供する共有ライブラリをリンカに教えてあげます。そのオプションを追加した ld コマンドが以下のようになります。

```
$ ld -m elf_i386 -o hello1 -L /usr/lib32 hello.obj -lc
```

`-L` でリンカに新しいライブラリ探索先のディレクトリを追加しています。ここでは上で用意した 32 bit ライブラリの配置先 `/usr/lib32` を指定しています。

`-l` でリンク対象のオブジェクトファイルやライブラリを追加しています (`-lc` と指定するとリンク時に `libc.so` を使用するようになる。また今回は関係ないし探索の優先度も下がるがアーカイブファイル `libc.a` も探索対象になるっぽい)。

(libc.a もあるしもしかして static link もやろうと思えばできるのか?)

#### \_start シンボルの解決

`_start` が見つからないというエラーについては ld がプログラムのエントリポイントをデフォルトでは `_start` だと認識しているためなので、明示的に `-e` オプションで `main` を指定して逃げます。

ということで以下のような ld コマンドによりリンクが通るようになりました。

```
$ ld -m elf_i386 -e main -o hello1 -L /usr/lib32 hello.obj -lc
```

#### 動的リンカの明示的な設定

しかしこれでめでたく実行可能... というようなことにはならず、実行時エラーになります。

```
$ ./hello1 
-bash: ./hello1: No such file or director
```

[ステップバイステップデバッグガイド - ArchWiki][2] によるとインタプリタが存在しない可能性があるということで、調べてみると確かに存在しないものが指定されています。

```
$ readelf -a ./hello1 | grep interp
  [ 1] .interp           PROGBITS        08048154 000154 000013 00   A  0   0  1
      [Requesting program interpreter: /usr/lib/libc.so.1]
   01     .interp 
   02     .interp .hash .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.plt

$ file /usr/lib/libc.so.1
/usr/lib/libc.so.1: cannot open `/usr/lib/libc.so.1' (No such file or directory)
```

ここで一旦試しに gcc コマンドで一気通貫に実行ファイルを作成してみると、作成された実行ファイルではインタプリタとして `/lib/ld-linux.so.2` が指定されているようなので、 32 bit 実行ファイルであれば本来欲しいのはこの子のようです。

```
$ gcc -Wall -march=i386 -m32 -o hello-gcc hello.c
$ readelf -a ./hello-gcc | grep interp
  [ 1] .interp           PROGBITS        00000194 000194 000013 00   A  0   0  1
      [Requesting program interpreter: /lib/ld-linux.so.2]
   01     .interp 
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt 
```

ということで ld にこの情報を加えることで無事動作する実行ファイルを作成することができました。

```
$ gcc -c -Wall -march=i486 -m32 -o hello.obj hello.c
$ ld -m elf_i386 -e main -dynamic-linker /lib/ld-linux.so.2 \
    -o hello1 -L /usr/lib32 hello.obj -lc
$ ./hello1
Hello World
```

### gcc のやり方に従ってリンクし実行可能にする

上では色々試行錯誤したのですが、実際のところ gcc はどうやって実行ファイルを作っているの? という方向からリンクの設定を構築するというアプローチもあります。本来 C プログラムを gcc でコンパイル、リンクした場合 `main` の前後でプログラムの動作に必要な処理が行われています。上のやり方ではこうした処理を全て無視して単に `main` だけを実行するようにしていました。

gcc がどのようなコマンドを実行しているかというのは `gcc -v` によりわかります。下記に参考までと出力を載せていますが長いのでけっこう省略しています。

```
$ gcc -v -Wall -march=i386 -m32 -o hello-gcc hello.c
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/lto-wrapper
Target: x86_64-pc-linux-gnu
Configured with: /build/gcc/src/gcc/configure ...
...

/usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/cc1 -quiet -v -imultilib 32 hello.c -quiet -dumpbase hello.c -march=i386 -m32 -auxbase hello -Wall -version -o /tmp/cconWpfz.s
GNU C17 (GCC) version 8.2.1 20181127 (x86_64-pc-linux-gnu)
...

as -v --32 -o /tmp/ccWMNefV.o /tmp/cconWpfz.s
...

/usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/collect2 -plugin /usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/liblto_plugin.so -plugin-opt=/usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/lto-wrapper -plugin-opt=-fresolution=/tmp/ccmx5Rfh.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --build-id --eh-frame-hdr --hash-style=gnu -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -pie -o hello-gcc /usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/../../../../lib32/Scrt1.o /usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/../../../../lib32/crti.o /usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/32/crtbeginS.o -L/usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/32 -L/usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/../../../../lib32 -L/lib/../lib32 -L/usr/lib/../lib32 -L/usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1 -L/usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/../../.. /tmp/ccWMNefV.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/32/crtendS.o /usr/lib/gcc/x86_64-pc-linux-gnu/8.2.1/../../../../lib32/crtn.o
...
```

今回大事なのは最後の collect2 コマンドのオプションです。collect2 とは? という感じなのですが どうやらこの内部で ld が呼ばれているようで、collect2 で指定したオプションが ld にそのまま渡されていそうに見えるので、これが参考になりそうです。

ここから `_start` と `printf` のシンボルを解決するためには以下のような 32 bit ライブラリを渡せば良さそうだということがわかりました。

```
$ld -m elf_i386 -o hello2 -L /usr/lib32 -dynamic-linker /lib/ld-linux.so.2 \
   hello.obj /usr/lib32/Scrt1.o /usr/lib32/crti.o /usr/lib32/crtn.o -lc
```

Scrt1.o, crti.o, crtn.o というファイルが出てきたのですが、 [What's the usage of Mcrt1.o and Scrt1.o? - Stack Overflow][3] を見るとこれがまさに main の前後処理にあたる関数を提供するものみたいです。手元にあった [Binary Hacks 本][4] の 58 番を見るとどうやら `_start` がエントリポイントになり、 `__libc_start_main` が呼ばれ、`main` に辿り着く、というような感じになるようです。

```
$ nm -S /usr/lib32/Scrt1.o
00000000 D __data_start
00000000 W data_start
00000000 00000004 R _fp_hw
         U _GLOBAL_OFFSET_TABLE_
00000000 00000004 R _IO_stdin_used
         U __libc_csu_fini
         U __libc_csu_init
         U __libc_start_main
         U main
00000000 00000036 T _start

$ nm -S /usr/lib32/crti.o
00000000 T _fini
         U _GLOBAL_OFFSET_TABLE_
         w __gmon_start__
00000000 T _init
00000000 00000004 T __x86.get_pc_thunk.bx

$ nm -S /usr/lib32/crtn.o
```

`__libc_start_main` がどこから来るの? という疑問がありますが、これは ldd を見るとどうやら `/usr/lib32/libc.so.6` から動的リンクされているようです。

```
# Show shared object dependencies
$ ldd ./hello2
	linux-gate.so.1 (0xf7f7b000)
	libc.so.6 => /usr/lib32/libc.so.6 (0xf7d74000)
	/lib/ld-linux.so.2 => /usr/lib/ld-linux.so.2 (0xf7f7c000)

# Trace call functions in shared libraries
$ ltrace ./hello2
__libc_start_main(0x8049060, 1, 0xffa11cc4, 0x80490e0 <unfinished ...>
puts("Hello World"Hello World
)                                                = 12
exit(0)                                                            = <void>
+++ exited (status 0) +++
```

[1]: https://wiki.archlinux.jp/index.php/Multilib
[2]: https://wiki.archlinux.jp/index.php/%E3%82%B9%E3%83%86%E3%83%83%E3%83%97%E3%83%90%E3%82%A4%E3%82%B9%E3%83%86%E3%83%83%E3%83%97%E3%83%87%E3%83%90%E3%83%83%E3%82%B0%E3%82%AC%E3%82%A4%E3%83%89
[3]: https://stackoverflow.com/questions/16436035/whats-the-usage-of-mcrt1-o-and-scrt1-o
[4]: https://www.amazon.co.jp/Binary-Hacks-%E2%80%95%E3%83%8F%E3%83%83%E3%82%AB%E3%83%BC%E7%A7%98%E4%BC%9D%E3%81%AE%E3%83%86%E3%82%AF%E3%83%8B%E3%83%83%E3%82%AF100%E9%81%B8-%E9%AB%98%E6%9E%97-%E5%93%B2/dp/4873112885
[5]: https://www.amazon.co.jp/30%E6%97%A5%E3%81%A7%E3%81%A7%E3%81%8D%E3%82%8B-OS%E8%87%AA%E4%BD%9C%E5%85%A5%E9%96%80-%E5%B7%9D%E5%90%88-%E7%A7%80%E5%AE%9F/dp/4839919844/ref=sr_1_1?s=books&ie=UTF8&qid=1547973975&sr=1-1&keywords=OS+%E8%87%AA%E4%BD%9C
