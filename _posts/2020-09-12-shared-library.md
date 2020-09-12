---
layout: post
title: 共有ライブラリ関数が呼ばれるまで
tags: "elf,x86-64"
comments: true
---

C 言語を書いていると stdio.h の printf だったり当たり前に共有ライブラリの関数を使っていますが、その実行がどのように行われているのかということをちゃんと調べたことがなかったので整理しようという試みです。

事前に自分が持っていた共有ライブラリの知識として、

- 共有ライブラリの関数 (の実装) は実行可能ファイル内に含まれるわけではない
- 物理メモリとしては共有ライブラリの関数の所在は一箇所で、各プロセスはそれをマップして共有している
- 関数実行時に何らかの処理をして関数を呼べるようにする
- GOT や PLT といったものが関わるらしい

といった程度のものはありましたが、それを実際にオブジェクトファイルや実行可能ファイルを見ながら確認していきます。

### サンプルプログラム

今回は動作確認用のサンプルとして簡単な共有ライブラリ (libsample.so) とそれが提供する add\_one 関数を使用するプログラム (main.c) を用意しました。

```c
// main.c

int add_one(int x);

int main(void) {
    int x = add_one(1);
    return x;
}
```

```c
// libsample.c

int add_one(int x) {
    return x + 1;
}
```

コンパイルやリンクは以下のように行なっています。

```makefile
# Makefile

CFLAGS := -Wall -Wextra

default: all

all: main

jain: main.o libsample.so
	gcc -o $@ -Wl,-rpath=. -L. $(CFLAGS) $< $(patsubst lib%.so,-l%,$(filter %.so,$^))

main.o: main.c
	gcc -o $@ -c $(CFLAGS) $<

libsample.so: libsample.o
	gcc -o $@.1 -shared -Wl,-soname=$@.1 $<
	ln -s $@.1 $@

libsample.o: libsample.c
	gcc -o $@ -c -fpic $(CFLAGS) $<

clean:
	rm -f *.o *.so *.so.* main
```

### main.o について

まずは add\_one を使用している main.o オブジェクトファイルを見てみます。

```
$ file main.o
main.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

コンパイルされた機械語を確認します。

```
# show machine codes of main.o
# call add_one at 0xd and 4 bytes from 0xe will be relocated.
$ objdump -S main.o

main.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   48 83 ec 10             sub    $0x10,%rsp
   8:   bf 01 00 00 00          mov    $0x1,%edi
   d:   e8 00 00 00 00          callq  12 <main+0x12>
  12:   89 45 fc                mov    %eax,-0x4(%rbp)
  15:   8b 45 fc                mov    -0x4(%rbp),%eax
  18:   c9                      leaveq
  19:   c3                      retq
```

アドレス 0xd の call add\_one 命令の call 先が 0x00000000 になっています。これはリンク時に値が設定される、つまりリロケーションが発生するためです。その情報は `readelf -r` で確認できます。

```
# show relocation sections of main.o
$ readelf -r main.o

Relocation section '.rela.text' at offset 0x1e0 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000000000e  000a00000004 R_X86_64_PLT32    0000000000000000 add_one - 4

Relocation section '.rela.eh_frame' at offset 0x1f8 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0
```

add\_one についてのリロケーション情報を見ると以下のことがわかります。

- relocation section 名が rela.text である
  - text 領域に対するリロケーションだということ
  - rel と rela の違い ([参考資料][3] の P.57 より)
    - ここでいうと -4 という情報が text 領域の 0xe に事前に入っている (rel) かリロケーションテーブル側にあるか (rela) ということ
    - -4 なのはリンク時にリロケーションされるとき 0xe から call 先のアドレスの相対アドレスが入るけど、0xd の call 命令としてはその次の命令である 0x12 からの相対アドレスが欲しいのでその調整
- Offset はリロケーション先
- Info はリロケーションのタイプ (0x4) と対象のシンボル (0xa)
  - タイプによってリロケーションの計算方法を規定している
    -  詳しくは [System V ABI][1] の 4.4 Relocation に載っているけど難しい
  - 下のシンボルテーブルを見るとわかるように 0xa が add\_one を指す

```
# show symbol table of main.o
# 0xa is add_one, which is corresponding to info of the entry: '000a00000004'
$ readelf -s main.o

Symbol table '.symtab' contains 11 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     8: 0000000000000000    26 FUNC    GLOBAL DEFAULT    1 main
     9: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND add_one
```

また main.o のセクションには GOT や PLT に関わるものは含まれません。

```
# show section table
$ readelf -S main.o
There are 12 section headers, starting at offset 0x270:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       000000000000001a  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  000001e0
       0000000000000018  0000000000000018   I       9     1     8
  [ 3] .data             PROGBITS         0000000000000000  0000005a
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .bss              NOBITS           0000000000000000  0000005a
       0000000000000000  0000000000000000  WA       0     0     1
  [ 5] .comment          PROGBITS         0000000000000000  0000005a
       0000000000000013  0000000000000001  MS       0     0     1
  [ 6] .note.GNU-stack   PROGBITS         0000000000000000  0000006d
       0000000000000000  0000000000000000           0     0     1
  [ 7] .eh_frame         PROGBITS         0000000000000000  00000070
       0000000000000038  0000000000000000   A       0     0     8
  [ 8] .rela.eh_frame    RELA             0000000000000000  000001f8
       0000000000000018  0000000000000018   I       9     7     8
  [ 9] .symtab           SYMTAB           0000000000000000  000000a8
       0000000000000108  0000000000000018          10     8     8
  [10] .strtab           STRTAB           0000000000000000  000001b0
       000000000000002b  0000000000000000           0     0     1
  [11] .shstrtab         STRTAB           0000000000000000  00000210
       0000000000000059  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

### libsample.o について

次に libsample.c をコンパイルして作成されたオブジェクトファイルを見ていきますが、こちらに関しては共有ライブラリの呼び出しを行なっておらず特に言及する部分が無いので各コマンドの結果を載せるだけにします。

```
$ file libsample.o
libsample.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped

$ readelf -a libsample.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          512 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         11
  Section header string table index: 10

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       000000000000000f  0000000000000000  AX       0     0     1
  [ 2] .data             PROGBITS         0000000000000000  0000004f
       0000000000000000  0000000000000000  WA       0     0     1
  [ 3] .bss              NOBITS           0000000000000000  0000004f
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .comment          PROGBITS         0000000000000000  0000004f
       0000000000000013  0000000000000001  MS       0     0     1
  [ 5] .note.GNU-stack   PROGBITS         0000000000000000  00000062
       0000000000000000  0000000000000000           0     0     1
  [ 6] .eh_frame         PROGBITS         0000000000000000  00000068
       0000000000000038  0000000000000000   A       0     0     8
  [ 7] .rela.eh_frame    RELA             0000000000000000  00000190
       0000000000000018  0000000000000018   I       8     6     8
  [ 8] .symtab           SYMTAB           0000000000000000  000000a0
       00000000000000d8  0000000000000018           9     8     8
  [ 9] .strtab           STRTAB           0000000000000000  00000178
       0000000000000015  0000000000000000           0     0     1
  [10] .shstrtab         STRTAB           0000000000000000  000001a8
       0000000000000054  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)

There are no section groups in this file.

There are no program headers in this file.

There is no dynamic section in this file.

Relocation section '.rela.eh_frame' at offset 0x190 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0

The decoding of unwind sections for machine type Advanced Micro Devices X86-64 is not currently supported.

Symbol table '.symtab' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS libsample.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    2
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     8: 0000000000000000    15 FUNC    GLOBAL DEFAULT    1 add_one

No version information found in this file.
```

### libsample.so について

libsample.o から作成された共有ライブラリを見ていきます。

```
$ file libsample.so.1
libsample.so.1: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=6662da613329b73c7c965178c45afe16b052dcf0, not stripped
```

libsample.so では libsample.o では見られなかった .dynamic, .got, .got.plt というセクションが作られています。

```
# show section table
$ readelf -S libsample.so
There are 24 section headers, starting at offset 0x36c0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.gnu.bu[...] NOTE             0000000000000238  00000238
       0000000000000024  0000000000000000   A       0     0     4
  [ 2] .gnu.hash         GNU_HASH         0000000000000260  00000260
       0000000000000024  0000000000000000   A       3     0     8
  [ 3] .dynsym           DYNSYM           0000000000000288  00000288
       0000000000000090  0000000000000018   A       4     1     8
  [ 4] .dynstr           STRTAB           0000000000000318  00000318
       0000000000000082  0000000000000000   A       0     0     1
  [ 5] .gnu.version      VERSYM           000000000000039a  0000039a
       000000000000000c  0000000000000002   A       3     0     2
  [ 6] .gnu.version_r    VERNEED          00000000000003a8  000003a8
       0000000000000020  0000000000000000   A       4     1     8
  [ 7] .rela.dyn         RELA             00000000000003c8  000003c8
       00000000000000a8  0000000000000018   A       3     0     8
  [ 8] .init             PROGBITS         0000000000001000  00001000
       000000000000001b  0000000000000000  AX       0     0     4
  [ 9] .text             PROGBITS         0000000000001020  00001020
       00000000000000d8  0000000000000000  AX       0     0     16
  [10] .fini             PROGBITS         00000000000010f8  000010f8
       000000000000000d  0000000000000000  AX       0     0     4
  [11] .eh_frame_hdr     PROGBITS         0000000000002000  00002000
       0000000000000014  0000000000000000   A       0     0     4
  [12] .eh_frame         PROGBITS         0000000000002018  00002018
       000000000000003c  0000000000000000   A       0     0     8
  [13] .init_array       INIT_ARRAY       0000000000003e40  00002e40
       0000000000000008  0000000000000008  WA       0     0     8
  [14] .fini_array       FINI_ARRAY       0000000000003e48  00002e48
       0000000000000008  0000000000000008  WA       0     0     8
  [15] .dynamic          DYNAMIC          0000000000003e50  00002e50
       0000000000000190  0000000000000010  WA       4     0     8
  [16] .got              PROGBITS         0000000000003fe0  00002fe0
       0000000000000020  0000000000000008  WA       0     0     8
  [17] .got.plt          PROGBITS         0000000000004000  00003000
       0000000000000018  0000000000000008  WA       0     0     8
  [18] .data             PROGBITS         0000000000004018  00003018
       0000000000000008  0000000000000000  WA       0     0     8
  [19] .bss              NOBITS           0000000000004020  00003020
       0000000000000008  0000000000000000  WA       0     0     1
  [20] .comment          PROGBITS         0000000000000000  00003020
       0000000000000012  0000000000000001  MS       0     0     1
  [21] .symtab           SYMTAB           0000000000000000  00003038
       0000000000000438  0000000000000018          22    40     8
  [22] .strtab           STRTAB           0000000000000000  00003470
       000000000000016f  0000000000000000           0     0     1
  [23] .shstrtab         STRTAB           0000000000000000  000035df
       00000000000000db  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

.dynamic セクションは実行時に動的リンカが必要とするための情報をまとめたセクションだという認識です。
`readelf -d` により内容を確認できます。

```
# show the detail of .dynamic section
$ readelf -d libsample.so.1
Dynamic section at offset 0x2e50 contains 21 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000e (SONAME)             Library soname: [libsample.so.1]
 0x000000000000000c (INIT)               0x1000
 0x000000000000000d (FINI)               0x10f8
 0x0000000000000019 (INIT_ARRAY)         0x3e40
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x3e48
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x260
 0x0000000000000005 (STRTAB)             0x318
 0x0000000000000006 (SYMTAB)             0x288
 0x000000000000000a (STRSZ)              130 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000007 (RELA)               0x3c8
 0x0000000000000008 (RELASZ)             168 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffe (VERNEED)            0x3a8
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x39a
 0x000000006ffffff9 (RELACOUNT)          3
 0x0000000000000000 (NULL)               0x0
```

主なタイプには以下のようなものがあります。

- NEEDED は依存している共有ライブラリ
  - libsample.c には現れていないが libc.so に依存することになっている
- SONAME で自分の soname を記述
  - soname やその使い方については [Linux 共有ライブラリの簡単なまとめ][4] がわかりやすいです
- STRTAB は .dynstr セクションのアドレスを指す 
  - .strtab の .dynamic セクション版
- SYMTAB は .dynsym セクションのアドレスを指す 
  - .symtab の .dynamic セクション版

また .dynamic セクションのアドレスはいずれかの情報からわかります (今回は 0x3e50)。

- program header から (DYNAMIC セグメント)
- `_DYNAMIC` シンボルから

```
# show program header
$ readelf -l libsample.so

Elf file type is DYN (Shared object file)
Entry point 0x1020
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000470 0x0000000000000470  R      0x1000
  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x0000000000000105 0x0000000000000105  R E    0x1000
  LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 0x0000000000000054 0x0000000000000054  R      0x1000
  LOAD           0x0000000000002e40 0x0000000000003e40 0x0000000000003e40
                 0x00000000000001e0 0x00000000000001e8  RW     0x1000
  DYNAMIC        0x0000000000002e50 0x0000000000003e50 0x0000000000003e50
                 0x0000000000000190 0x0000000000000190  RW     0x8
  NOTE           0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x0000000000000024 0x0000000000000024  R      0x4
  GNU_EH_FRAME   0x0000000000002000 0x0000000000002000 0x0000000000002000
                 0x0000000000000014 0x0000000000000014  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000002e40 0x0000000000003e40 0x0000000000003e40
                 0x00000000000001c0 0x00000000000001c0  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn
   01     .init .text .fini
   02     .eh_frame_hdr .eh_frame
   03     .init_array .fini_array .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.gnu.build-id
   06     .eh_frame_hdr
   07
   08     .init_array .fini_array .dynamic .got

# find _DYNAMIC from symbol table
$ readelf -s libsample.so
   Num:    Value          Size Type    Bind   Vis      Ndx Name
    ...
    35: 0000000000003e50     0 OBJECT  LOCAL  DEFAULT   15 _DYNAMIC
```

上で書いたように libsample.so は libc.so で定義される関数を使うことになっており、そのためにリロケーションが必要になっています。はじめ 3 つについてはわかりませんがその他のものについては .got セクションに対するリロケーションとなっています (名前は .rela.dyn ですが)。これらは [参考資料][5] を見る感じ main のロード時に解決されるようです (add\_one 関数が呼び出し時に解決されるのとは違う。 .rel.plt についてだけ遅延されるということなのか?)。

```
# show relocation table
$ readelf -r libsample.so.1

Relocation section '.rela.dyn' at offset 0x3c8 contains 7 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003e40  000000000008 R_X86_64_RELATIVE                    10e0
000000003e48  000000000008 R_X86_64_RELATIVE                    1090
000000004018  000000000008 R_X86_64_RELATIVE                    4018
000000003fe0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTM[...] + 0
000000003fe8  000200000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000300000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCl[...] + 0
000000003ff8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

# dump .got section
$ readelf -x 16 libsample.so

Hex dump of section '.got':
  0x00003fe0 00000000 00000000 00000000 00000000 ................
  0x00003ff0 00000000 00000000 00000000 00000000 ................
```

GOT (Global Offset Table) というのはプログラムを PIC (位置独立コード) にするために使用されるという認識です。共有ライブラリは複数プロセスで使用されますが、そのときに常に固定のアドレスを使えるとは限らない (各プロセスでどのアドレスにマップするのかわからない) のでどこに配置されてもいいように PIC にする必要があります。その際に共有ライブラリ関数のような実行時までアドレスがわからないシンボルを GOT にまとめ text 領域についてはリロケーションを不要にしています。

PLT (Procedure Linkage Table) は共有ライブラリの関数呼び出しの解決を遅延するために使用されるものだという認識です。実行時にシンボルを解決するといったときにプログラムロード時に解決することもできますが、別の選択肢としてはじめてそれが使用されるときに解決するということも考えられます。共有ライブラリ関数の場合は通常後者の方法が取られ、そのときに使用されるのが PLT です。これについてはこのあと main 実行可能ファイルに触れる中でより詳しく見ていきます。

### main について

最後に最終生成物である main 実行可能ファイルを見ていきます。

```
$ file main
main: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=136be790497f602fbc87d7277bf9040ea1188281, for GNU/Linux 3.2.0, not stripped
```

機械語を確認します。

```
$ objdump -S main

main:     file format elf64-x86-64

...

Disassembly of section .plt:

0000000000001020 <.plt>:
    1020:       ff 35 e2 2f 00 00       pushq  0x2fe2(%rip)        # 4008 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:       ff 25 e4 2f 00 00       jmpq   *0x2fe4(%rip)        # 4010 <_GLOBAL_OFFSET_TABLE_+0x10>
    102c:       0f 1f 40 00             nopl   0x0(%rax)

0000000000001030 <add_one@plt>:
    1030:       ff 25 e2 2f 00 00       jmpq   *0x2fe2(%rip)        # 4018 <add_one>
    1036:       68 00 00 00 00          pushq  $0x0
    103b:       e9 e0 ff ff ff          jmpq   1020 <.plt>

...

Disassembly of section .text:

...

0000000000001139 <main>:
    1139:       55                      push   %rbp
    113a:       48 89 e5                mov    %rsp,%rbp
    113d:       48 83 ec 10             sub    $0x10,%rsp
    1141:       bf 01 00 00 00          mov    $0x1,%edi
    1146:       e8 e5 fe ff ff          callq  1030 <add_one@plt>
    114b:       89 45 fc                mov    %eax,-0x4(%rbp)
    114e:       8b 45 fc                mov    -0x4(%rbp),%eax
    1151:       c9                      leaveq
    1152:       c3                      retq
    1153:       66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
    115a:       00 00 00
    115d:       0f 1f 00                nopl   (%rax)
```

- main
  - 0x1146: add\_one@plt への call 命令
- add\_one@plt
  - 0x1030: 0x4018 に格納されているアドレスへ飛ぶ 

0x4018 は .plt.got セクション内を指します。

```
$ readelf -x 22 main

Hex dump of section '.got.plt':
 NOTE: This section has relocations against it, but these have NOT been applied to this dump.
  0x00004000 d83d0000 00000000 00000000 00000000 .=..............
  0x00004010 00000000 00000000 36100000 00000000 ........6.......
```

ここに格納されているアドレスが add\_one のアドレスということになるのですが、プログラム開始時にはこのアドレスはまだ解決されていません。初回呼び出しではまず 0x1036 というアドレスに飛んでそこの処理でアドレスを解決してこのアドレスを直接 add\_one を指すように書き換える、というのが共有ライブラリ関数の遅延解決になります。

.plt セクションのディスアセンブリ結果を再掲します。

```
Disassembly of section .plt:

0000000000001020 <.plt>:
    1020:       ff 35 e2 2f 00 00       pushq  0x2fe2(%rip)        # 4008 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:       ff 25 e4 2f 00 00       jmpq   *0x2fe4(%rip)        # 4010 <_GLOBAL_OFFSET_TABLE_+0x10>
    102c:       0f 1f 40 00             nopl   0x0(%rax)

0000000000001030 <add_one@plt>:
    1030:       ff 25 e2 2f 00 00       jmpq   *0x2fe2(%rip)        # 4018 <add_one>
    1036:       68 00 00 00 00          pushq  $0x0
    103b:       e9 e0 ff ff ff          jmpq   1020 <.plt>
```

- add\_one@plt
  - 0x1036: push $0x0
    - これからリロケーションする対象を表している
    - .rela.plt セクションでのオフセット?
  - 0x103b: .plt への jmp
- .plt
  - 0x1020: \_GLOBAL\_OFFSET\_TABLE\_ + 0x8 に格納されている値を push
  - 0x1026: \_GLOBAL\_OFFSET\_TABLE\_ + 0x10 に格納されているアドレスへ jmp

\_GLOBAL\_OFFSET\_TABLE\_ はシンボルテーブルから 0x4000、つまり上で見た .got.plt の開始アドレスであることがわかります。

```
# show symbol table
$ readelf -s main
   Num:    Value          Size Type    Bind   Vis      Ndx Name
    ...
    45: 0000000000004000     0 OBJECT  LOCAL  DEFAULT   22 _GLOBAL_OFFSET_TABLE_
```

`_GLOBAL_OFFSET_TABLE` エントリのうちはじめの 3 エントリはどうやら使い方が決まっているようです。

- `_GLOBAL_OFFSET_TABLE_ + 0x0` が (リンク時の) dynamic セグメントの開始アドレス
- `_GLOBAL_OFFSET_TABLE_ + 0x8` が共有ライブラリの情報を持つ構造体のアドレス
  - 実行時に動的リンカによって埋められる (実行可能ファイル内では 0)
- `_GLOBAL_OFFSET_TABLE_ + 0x10` が動的リンクするための関数へのアドレス
  - 実行時に動的リンカによって埋められる (実行可能ファイル内では 0)

それ以降に遅延解決する共有ライブラリ関数へのアドレスが入るみたいです。そうした関数はリロケーションテーブルにも載っています (.rela.plt セクション)。

```
# show relocation table
$ readelf -r main

Relocation section '.rela.dyn' at offset 0x498 contains 8 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003dc8  000000000008 R_X86_64_RELATIVE                    1130
000000003dd0  000000000008 R_X86_64_RELATIVE                    10e0
000000004028  000000000008 R_X86_64_RELATIVE                    4028
000000003fd8  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTM[...] + 0
000000003fe0  000300000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.2.5 + 0
000000003fe8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCl[...] + 0
000000003ff8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x558 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000004018  000100000007 R_X86_64_JUMP_SLO 0000000000000000 add_one + 0
```

### 動的リンカの (軽く) ソースリーディング

ここまでの理解を補完するために動的リンカの実装を少しだけ読んだのでそのメモを残します。main 実行可能ファイルでは動的リンカとして /lib64/ld-linux-x86-64.so.2 を指定しており、これは glibc から提供されているようです。

```
# check it by 'file' command
$ file main
main: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=136be790497f602fbc87d7277bf9040ea1188281, for GNU/Linux 3.2.0, not stripped

# or INTERP in program header
$ readelf -l main

Elf file type is DYN (Shared object file)
Entry point 0x1040
There are 11 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x0000000000000268 0x0000000000000268  R      0x8
  INTERP         0x00000000000002a8 0x00000000000002a8 0x00000000000002a8
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000570 0x0000000000000570  R      0x1000
  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x00000000000001e5 0x00000000000001e5  R E    0x1000
  LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 0x0000000000000110 0x0000000000000110  R      0x1000
  LOAD           0x0000000000002dc8 0x0000000000003dc8 0x0000000000003dc8
                 0x0000000000000268 0x0000000000000270  RW     0x1000
  DYNAMIC        0x0000000000002dd8 0x0000000000003dd8 0x0000000000003dd8
                 0x0000000000000200 0x0000000000000200  RW     0x8
  NOTE           0x00000000000002c4 0x00000000000002c4 0x00000000000002c4
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_EH_FRAME   0x0000000000002004 0x0000000000002004 0x0000000000002004
                 0x0000000000000034 0x0000000000000034  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000002dc8 0x0000000000003dc8 0x0000000000003dc8
                 0x0000000000000238 0x0000000000000238  R      0x1
```

以下 glibc v2.29 に基づいた内容です。

- dl\_main (in `elf/rtld.c`)
  - [Executable and Linkable Format (ELF)][5] によれば多くのロード作業はこの中で行われている
  - 実行可能ファイルの DYNAMIC セグメントから NEEDED な共有ライブラリをロード
    - 最終的に \_dl\_map\_segments in `elf/dl-map-segments.h` で共有ライブラリを mmap する
  - GOT の 2 つの情報を埋める (by elf\_machine\_runtime\_setup in `sysdeps/x86_64/dl-machine.h`)
    - \_GLOBAL\_OFFSET\_TABLE\_ + 0x8
    - \_GLOBAL\_OFFSET\_TABLE\_ + 0x10
- 共有ライブラリの関数がはじめて呼ばれたときに上で設定した関数へ処理が飛んでくるので .rela.plt を書き換える
  - dl\_runtime\_resolve 関数で行っている

### 参考

- [リンカ・ローダ実戦開発テクニック][3]
- [Runtime Dynamic Linking][2]
- [Executable and Linkable Format (ELF)  - stevens.netmeister.org][5]

[1]: https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf
[2]: http://users.eecs.northwestern.edu/~kch479/docs/notes/linking.html
[3]: https://www.amazon.co.jp/%E3%83%AA%E3%83%B3%E3%82%AB%E3%83%BB%E3%83%AD%E3%83%BC%E3%83%80%E5%AE%9F%E8%B7%B5%E9%96%8B%E7%99%BA%E3%83%86%E3%82%AF%E3%83%8B%E3%83%83%E3%82%AF%E2%80%95%E5%AE%9F%E8%A1%8C%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AB%E5%BF%85%E9%A0%88%E3%81%AE%E6%8A%80%E8%A1%93-COMPUTER-TECHNOLOGY-%E5%9D%82%E4%BA%95-%E5%BC%98%E4%BA%AE/dp/4789838072
[4]: https://www.wagavulin.jp/entry/20091026/1256577635
[5]: https://stevens.netmeister.org/631/elf.html
