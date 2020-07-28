---
layout: post
title: アセンブリで 32 bit に収まらないシンボルのアドレスを取得したい
tags: "x86-64"
comments: true
---

慣れない x86-64 向けのアセンブリを書いていてアドレス値が 32 bit に収まらないシンボルのアドレスを取得する際に困ったことがあったので、その解決方法と調べていく中でわかったことをまとめました。

### 困った内容の説明

自分が何に苦戦したのかを説明するために簡単な例を用意しました。msg シンボルの差すアドレスを rax レジスタに入れるだけのプログラムです。

```assembly
.text
.global _start
_start:
  movq $msg,%rax

loop:
  hlt
  jmp loop

.data
msg: .ascii "hello world"
```

このアセンブリコードを `foo.S` とし、アセンブル、リンクします。

```
# ここでは PIE について考えたくないので -no-pie を渡しています
$ gcc -o foo.o -c -no-pie foo.S

# ここでは linker.ld として `ld --verbose` で確認できるデフォルトのリンカスクリプトを使用しています
$ ld -o main -Tlinker.ld foo.o
```

作成された実行可能ファイルの内容は以下のようになります。msg が 0x402000 に配置されており、mov 命令でそれを問題なくロードできています。

```
$ objdump -S main
main:     file format elf64-x86-64

Disassembly of section .text:

0000000000401000 <_start>:
  401000:       48 c7 c0 00 20 40 00    mov    $0x402000,%rax

0000000000401007 <loop>:
  401007:       f4                      hlt    
  401008:       eb fd                   jmp    401007 <loop>

$ nm main
000000000040200b D __bss_start
000000000040200b D _edata
0000000000402010 D _end
0000000000401007 t loop
0000000000402000 d msg
0000000000401000 T _start
```

次にこの msg を > 4 GiB の仮想アドレスに配置されるようにします。そのためにリンカスクリプトの以下の部分を変更し、通常は 0x400000 から始まる text 領域の位置を 0x400000000 に変更しました。

```
$ diff linker.ld.orig linker.ld
14c14
<   PROVIDE (__executable_start = SEGMENT_START("text-segment", 0x400000)); . = SEGMENT_START("text-segment", 0x400000) + SIZEOF_HEADERS;
---
>   PROVIDE (__executable_start = SEGMENT_START("text-segment", 0x400000000)); . = SEGMENT_START("text-segment", 0x400000000) + SIZEOF_HEADERS;
```

このようにすると以下のように リンク時にエラーが発生します。

```
$ ld -o main -Tlinker.ld foo.o
foo.o: in function `_start':
(.text+0x3): relocation truncated to fit: R_X86_64_32S against `.data'
```

このエラーを解決したい、というのが今回の内容です。

なお本記事は以下の環境で動作確認しています。

```
$ cat /proc/version
Linux version 5.7.8-arch1-1 (linux@archlinux) (gcc version 10.1.0 (GCC), GNU ld (GNU Binutils) 2.34.0) #1 SMP PREEMPT Thu, 09 

$ uname -a
Linux <hostname> 5.7.8-arch1-1 #1 SMP PREEMPT Thu, 09 Jul 2020 16:34:01 +0000 x86_64 GNU/Linux

$ as --version
GNU assembler (GNU Binutils) 2.34.0
Copyright (C) 2020 Free Software Foundation, Inc.
This program is free software; you may redistribute it under the terms of
the GNU General Public License version 3 or later.
This program has absolutely no warranty.
This assembler was configured for a target of `x86_64-pc-linux-gnu'.

$ ld --version
GNU ld (GNU Binutils) 2.34.0
Copyright (C) 2020 Free Software Foundation, Inc.
This program is free software; you may redistribute it under the terms of
the GNU General Public License version 3 or (at your option) a later version.
This program has absolutely no warranty.
```


### エラーの意味

エラー内容を再掲します。

```
foo.o: in function `_start':
(.text+0x3): relocation truncated to fit: R_X86_64_32S against `.data'
```

出力を見る限りどうやらリロケーションまわりでエラーが発生しているようです。アセンブラは各オブジェクトファイル生成時にリロケーションに必要な情報もまとめてくれているので、それを `objdump -r` で確認します。

```
$ objdump -S -r foo.o

foo.o:     file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <_start>:
   0:   48 c7 c0 00 00 00 00    mov    $0x0,%rax
                        3: R_X86_64_32S .data

0000000000000007 <loop>:
   7:   f4                      hlt    
   8:   eb fd                   jmp    7 <loop>
```

アドレス 0x0 の mov 命令のオペランド値にリロケーションが必要なことがわかります。その情報として `R_X86_64_32S` というのがありますが、[stackoverflow][3] を見ると sign-extend な 32 bit 値を表すということになっています。

しかしリンカはこのデータを 0x400000000 以降のアドレスに配置しようとするので、32 bit の範囲に収まらず truncate されてしまう、というのが今回のエラーの内容のようです。

なおリロケーションについては以下の資料も参考にしました。

- [Linkers - Linux Inside][1]
- [System V Application Binary Interface][2]

### 解決策 1

このエラーを解決するために色々調べた結果、2 通りの解決方法がわかりました。
その一つは PC 相対アドレッシングを使用するものです。

通常 AT&T シンタックスでは `offset(base,index,scale)` という表記 (ただし base, index はレジスタ、scale は 1, 2, 4, or 8) でメモリアドレス `(base + index * scale + offset)` を参照できますが、x86-64 では base に %rip を指定する `offset(%rip)` という表記で PC 相対アドレッシングによるメモリ参照を指定できます。その使用例を [GNU assembler のマニュアル][4] から引用します。

```
# from https://sourceware.org/binutils/docs/as/i386_002dMemory.html
AT&T: `1234(%rip)', Intel: `[rip + 1234]'
  Points to the address 1234 bytes past the end of the current instruction.
AT&T: `symbol(%rip)', Intel: `[rip + symbol]'
  Points to the symbol in RIP relative way, this is shorter than the default absolute addressing. 
```

前者の即値を渡す例は従来の `offset(base,index,scale)` と同様の解釈で理解できますが、後者は初見だと理解し辛く実例を見たほうがわかりやすいと思います。ということでこれまでの例に PC 相対アドレッシングを使用すると以下のようになります。

```assembly
.text
.global _start
_start:
  # PC 相対アドレッシングの使用。計算後のアドレス自体が欲しいので mov ではなく lea を使用していることに注意
  leaq msg(%rip),%rax

loop:
  hlt
  jmp loop

.data
msg: .ascii "hello world"
```

コンパイル、リンクしてその内容を確認します。main 実行ファイルの lea 命令で確かに msg のアドレスが計算できていることがわかります。

```
$ gcc -o foo.o -c -no-pie foo.S
$ ld -o main -Tlinker.ld foo.o


# オブジェクトファイルのリロケーション情報が R_X86_64_PC32 に変わっている
$ objdump -S -r foo.o
foo.o:     file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <_start>:
   0:   48 8d 05 00 00 00 00    lea    0x0(%rip),%rax        # 7 <loop>
                        3: R_X86_64_PC32        .data-0x4

0000000000000007 <loop>:
   7:   f4                      hlt
   8:   eb fd                   jmp    7 <loop>


# msg のアドレスを %rip + 0x00000ff9 として計算している
# このときの %rip は次の命令を差すので 0x400001007 であり確かに計算すると 0x400002000 となることがわかる
$ objdump -S main
main:     file format elf64-x86-64

Disassembly of section .text:

0000000400001000 <_start>:
   400001000:   48 8d 05 f9 0f 00 00    lea    0xff9(%rip),%rax        # 400002000 <msg>

0000000400001007 <loop>:
   400001007:   f4                      hlt
   400001008:   eb fd                   jmp    400001007 <loop>


$ nm main
000000040000200b D __bss_start
000000040000200b D _edata
0000000400002010 D _end
0000000400001007 t loop
0000000400002000 d msg
0000000400001000 T _start
```

この解決策では従来と同じようにリロケーションアドレスのサイズは 32 bit のままですが対象シンボルの %rip からの相対値がその範囲に収まるのでうまく動作するようになっています。逆にいうと msg がその範囲に収まらない場合はやはり同様にリロケーション時にエラーになるはずです。

([System V Application Binary Interface][2] によれば通常オブジェクトファイル生成時は small model というメモリモデルを想定しており、すべてのシンボルのアドレスが 0x0 から 2 GiB の範囲に収まることを想定しているようです。なのでこれを medium model や large model に変えればいいのではと思い gcc に `-mcmodel=large` を渡したりしたのですがうまく動かせずに諦めました)

### 解決策 2

2 つ目の解決策はあとで gas のマニュアルを見て気付いたのですが単に mov の代わりに movabs 命令を使用するというものです。

そもそもの問題はアセンブル時になぜか 64 bit アドレスを扱いたいのに 32 bit のリロケーションを想定されてしまっているというものだったので、何かしらアセンブラへの指示方法が間違っているのだろうと思っていました。そのうえで [GNU assembler のマニュアル][5] を見ると、

> In 64-bit code, movabs can be used to encode the mov instruction with the 64-bit displacement or immediate operand

のように movabs 命令を使用するべきということが明記されていました。

これに従いサンプルコードを movabs 命令で書き直します。

```assembly
.text
.global _start
_start:
  movabs $msg,%rax

loop:
  hlt
  jmp loop

.data
msg: .ascii "hello world"
```

コンパイル、リンク後の内容を確認します。

```bash
$ gcc -o foo.o -c -no-pie foo.S
$ ld -o main -Tlinker.ld foo.o


# リロケーションアドレスのサイズが R_X86_64_64 のように 64 bit になっている
$ objdump -S -r foo.o

foo.o:     file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <_start>:
   0:   48 b8 00 00 00 00 00    movabs $0x0,%rax
   7:   00 00 00
                        2: R_X86_64_64  .data

000000000000000a <loop>:
   a:   f4                      hlt
   b:   eb fd                   jmp    a <loop>


# movabs 命令で msg の絶対アドレスをロードすることができている
$ objdump -S main
main:     file format elf64-x86-64

Disassembly of section .text:

0000000400001000 <_start>:
   400001000:   48 b8 00 20 00 00 04    movabs $0x400002000,%rax
   400001007:   00 00 00

000000040000100a <loop>:
   40000100a:   f4                      hlt
   40000100b:   eb fd                   jmp    40000100a <loop>


$ nm main
000000040000200b D __bss_start
000000040000200b D _edata
0000000400002010 D _end
000000040000100a t loop
0000000400002000 d msg
0000000400001000 T _start
```

x86-64 のアセンブリでは PC 相対アドレッシングに慣れることも大事だと思いますが、今回の場合は movabs を使うほうが素直な解決策なように思います。

[1]: https://0xax.gitbooks.io/linux-insides/content/Misc/linux-misc-3.html
[2]: https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf
[3]: https://stackoverflow.com/questions/6093547/what-do-r-x86-64-32s-and-r-x86-64-64-relocation-mean
[4]: https://sourceware.org/binutils/docs/as/i386_002dMemory.html
[5]: https://sourceware.org/binutils/docs/as/i386_002dVariations.html
