---
layout: post
title: MS-DOS の exe フォーマット
tags: "exe,freedos,msdos"
comments: true
---

平成も終わるというのに何故か相変わらず MS-DOS (FreeDOS) と遊んでいます。

MS-DOS の exe フォーマットについてまだ完全では無いですがある程度読み方がわかってきたのでその内容をまとめます。

### 資料

現代にも `.exe` 拡張子を持つ実行ファイルは存在しますが、これはフォーマットとしては [PE (Portable Executable)][4] と呼ばれるものになり、MS-DOS における実行ファイルのフォーマットとは異なります。

では MS-DOS の exe ファイルフォーマットの資料を、と探すと以下のようなものが見つかります。

- [MZ - OSDev Wiki][1]
- [Format of .exe files?][2]
- [Notes on the format of DOS .exe files][3]

見つかるのですが、これだけだとあまりピンと来ないのでいくつかサンプルの実行ファイルを作成、解析して内容を理解していきます。

### 実行ファイル解析第一歩

まずは exe フォーマットの雰囲気だけを掴むために、小さなプログラムをアセンブリ言語で用意しました。exitcode 1 で終了するだけのプログラムです。

ちなみに FreeDOS は QEMU で用意しているので、コンパイルやリンクは FreeDOS 上に用意した環境で行っていますが、ファイルを確認する際にはイメージファイルをホストのループバックデバイスを介してマウントしています (詳細は[以前のポスト][5])。

```asm
$ cat /mnt/freedos/works/exe_fmt/ex1.asm 
.model small
.386p
.stack 100h

.code
start:
  ; exit with status 1
  mov ax,4c01h
  int 21h
end start
```

コンパイル、リンクして実行ファイルを作成し、実行できることを確認します。

```
> wasm -ms ex1.asm
> wcl -ecc -ms ex1.obj
> ex1.exe
> echo %ERRORLEVEL%
1
```

その中身を od で見ます。

```
$ od -t x1z -A x /mnt/freedos/works/exe_fmt/ex1.exe
000000 4d 5a 25 00 01 00 00 00 02 00 01 01 ff ff 01 00  >MZ%.............<
000010 00 10 00 00 00 00 00 00 20 00 00 00 00 00 00 00  >........ .......<
000020 b8 01 4c cd 21                                   >..L.!<
000025
```

MS-DOS exe フォーマットでは最初にヘッダがあり、ここでは 0x000000 - 0x00001f までの 32 bytes がそれにあたります (判断の仕方は後述)。

最後の 0x000020 から始まる `0xb8 0x01 0x4c`, `0xcd 0x21` がそれぞれ `mov ax,4c01h`, `int 21` にあたります。参考にしている資料ではこのヘッダ終了後のコード領域 (ここでは 0x000020 から) を load module と呼んでいるようなので、ここでもそれを踏襲しようと思います。

(実行時にメモリにロードされる部分なので load module ということかと)

実行ファイルというと何だか難しそうなイメージがありましたが、このように最小構成で見れば高々 40 のバイトの連なりにしか過ぎず、頑張れば理解できそうな気がしてきますね。

### exe フォーマットヘッダ

ということでここからは MS-DOS exe フォーマットの詳細を見ていきます。実行ファイルを理解する上で何よりも重要なのは先頭のヘッダ部分と言っていいと思います。というかこれが全て理解できれば実行ファイルが読めるようになったと言ってもいいんじゃないかと。

ヘッダが含む情報とその位置、サイズを以下のテーブルにまとめました。

| Offset | Field                     | Size |
|--------|:-------------------------:|:-----|
| 0x00   | Signature                 | 2    |
| 0x02   | Last Page Size            | 2    |
| 0x04   | File Pages                | 2    |
| 0x06   | Relocation items          | 2    |
| 0x08   | Header paragraphs         | 2    |
| 0x0A   | Minimum allocation        | 2    |
| 0x0C   | Maximum allocation        | 2    |
| 0x0E   | Initial SS                | 2    |
| 0x10   | Initial SP                | 2    |
| 0x12   | Checksum                  | 2    |
| 0x14   | Initial IP                | 2    |
| 0x16   | Initial CS                | 2    |
| 0x18   | Relocation table offset   | 2    |
| 0x1A   | Overlay                   | 2    |
| 0x1C   | Overlay information       | 2    |

テーブル中 page, paragraph という単語が出てきますが、ここではそれぞれ 512 bytes, 16 bytes というサイズを表します。

#### わかりやすい項目たち

まずはヘッダに含まれる項目のうちわかりやすいものについて、実際の実行ファイルを見ながら確認してみます。上で使用した exitcode 1 で終了する実行ファイルのバイナリ表示にここで説明したい 6 項目 の位置を付せたものが下記になります。

```
$ od -t x1z -A x /mnt/freedos/works/exe_fmt/ex1.exe
000000 4d 5a 25 00 01 00 00 00 02 00 01 01 ff ff 01 00  >MZ%.............<
       -----                   -----             -----
       (1)                     (2)               (3)

000010 00 10 00 00 00 00 00 00 20 00 00 00 00 00 00 00  >........ .......<
       -----       ----- -----
       (4)         (5)   (6)

000020 b8 01 4c cd 21                                   >..L.!<



000025
```

(1) Signature で示される先頭 2 byte は exe 実行フォーマットであれば常に `0x4d 0x5a` (MZ) 固定となります。

(2) Header paragraphs はヘッダサイズを表すものであり、ファイル先頭から `Header paragraphs x 16 bytes` がヘッダ領域になります。上の実行ファイルの場合 0x0002 なので 32 bytes がヘッダ領域だとわかります。

(3), (4), (5), (6) はそれぞれプログラム開始時の SS, SP, IP, CS です。MS-DOS exe 実行ファイルではプログラム実行に設定が必須といえるこれらのレジスタについて初期値が与えられます。一方で DS や ES レジスタについては指定されておらず、プログラム中で必要に応じて指定する必要があるようです。

(なので例えば C で Hello World を書いた場合、main 関数にたどり着く前に DS の初期化のような処理が走っているはず。まだちゃんと確認できていないけれど)

(実行ファイルに指定は無いけど、MS-DOS は DS, ES レジスタの初期値として PSP 領域を指すようにセットするみたい?)

実は SS, CS については実行時にここで指定されたものとは違う値が入るのが普通で、その理由は relocation によるものなのですがそれに関してはすぐに下で見ます。

#### relocation に関する項目たち

MS-DOS exe ファイルは実行時、プログラムを開始する前に relocation 処理を行う必要があります。

MS-DOS exe ファイルの実行における relocation とは「実行時、プログラム開始前にセグメント値の調整を行う」ことです。exe ファイル実行時 load module をメモリにロードしますが、そのロード先のアドレスがどこになるかが決まらないと埋められないオペランド (と初期 SS, CS レジスタ) というのが存在するので、それをプログラム開始前に解決するという作業になります。

具体例と共に relocation の詳細を見てみます。このプログラムは実行すると Hello World! を出力します。サンプルとしての都合で始めに意味のない cx, es レジスタへの mov 命令を含んでいます。

```asm
$ cat /mnt/freedos/works/exe_fmt/ex2.asm 
.model small
.386p
.stack 100h

.data
msg db 'Hello World!$'

.code
start:
  mov ax,seg msg
  mov ds,ax
  mov cx,seg msg
  mov es,ax
  ;
  mov ah,09h
  lea dx,msg
  int 21h
  ;
  mov ax,4c00h
  int 21h
end start
```

このアセンブリコードから作成した実行ファイルのバイナリは以下のようになります。

```
$ od -t x1z -A x /mnt/freedos/works/exe_fmt/ex2.exe 
000000 4d 5a 55 00 01 00 02 00 03 00 01 01 ff ff 03 00  >MZU.............<
                         -----
                         (7)

000010 00 10 00 00 00 00 00 00 20 00 00 00 00 00 00 00  >........ .......<
                               -----
                               (8)

000020 01 00 00 00 06 00 00 00 00 00 00 00 00 00 00 00  >................<



000030 b8 01 00 8e d8 b9 01 00 8e c0 b4 09 8d 16 08 00  >................<



000040 cd 21 b8 00 4c cd 21 00 48 65 6c 6c 6f 20 57 6f  >.!..L.!.Hello Wo<



000050 72 6c 64 21 24                                   >rld!$<



000055
```

上のバイナリを見ると、Header paragraphs は 0x0003 となっています。ということは前の例で見たバイナリよりもヘッダサイズが 16 bytes 増えているのですが、この 0x000020 - 0x00002f 領域には relocation table というものが配置されています。

このことはヘッダ中

- (8) Relocation table offset により relocation table の開始位置 (ここでは 0x000020)
- (7) Relocation items によりテーブルのアイテム数 (ここでは 2 個)

を知ることで判断できます。

各 relocation table の要素サイズは 4 bytes で relocation が必要な load module 内のアドレスを示します。前半 2 bytes が offset, 後半 2 bytes がセグメント値です。上の例だと 0 番目は 0x0000:0x0001 であり, load module 内で言えば 1 byte 目、ファイルオフセットでいえば 0x000031 の `0xb8 0x01 0x00` の 0x0001 というセグメント値を指すアドレスを示しています。

MS-DOS exe ファイルにおける relocation というのは「実行時、プログラム開始前にセグメント値の調整を行う」ことであり、具体的にはリンカが用意した relocation table に従い OS のような exe ファイルを起動するプログラムが調整していくということになります。

実際にメモリに展開された load module を確認することで relocation の結果を確認してみます。若干無理矢理ですが上のアセンブリの途中に無限ループをはさみ、qemu monitor でその時点のレジスタ、メモリの様子を確認することができます。

```asm
$ cat /mnt/freedos/works/exe_fmt/ex3.asm
.model small
.386p
.stack 100h

.data
msg db 'Hello World!$'

.code
start:
  mov ax,seg msg
  mov ds,ax
  mov cx,seg msg
  mov es,ax
  ;

; infinite loop
loop1:
  nop
  jmp loop1

  mov ah,09h
  lea dx,msg
  int 21h
  ;
  mov ax,4c00h
  int 21h
end start
```

```
# show registers
(qemu) info registers
EAX=00000593 EBX=001e0000 ECX=00000593 EDX=00000582
ESI=001d0000 EDI=001d1000 EBP=001e091e ESP=00001000
EIP=0000000a EFL=00000202 [-------] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0593 00005930 0000ffff 00009300
CS =0592 00005920 0000ffff 00009b00
SS =0595 00005950 0000ffff 00009300
DS =0593 00005930 0000ffff 00009300
FS =0e98 0000e980 0000ffff 00009300
GS =0e98 0000e980 0000ffff 00009300
...

# show memory
(qemu) xp /48xb 0x5920
0000000000005920: 0xb8 0x93 0x05 0x8e 0xd8 0xb9 0x93 0x05
0000000000005928: 0x8e 0xc0 0x90 0xeb 0xfd 0xb4 0x09 0x8d
0000000000005930: 0x16 0x0a 0x00 0xcd 0x21 0xb8 0x00 0x4c
0000000000005938: 0xcd 0x21 0x48 0x65 0x6c 0x6c 0x6f 0x20
0000000000005940: 0x57 0x6f 0x72 0x6c 0x64 0x21 0x24 0x52
0000000000005948: 0x06 0x1e 0x55 0x56 0x57 0x51 0x53 0x50
```

これを見るとどうやら start segment は 0x0592 だということがわかります。

実行前は `0xb8 0x01 0x00` だった先頭 3 byte の mov 命令がロード後は `0xb8 0x93 0x05` となっており、relocation により元々のセグメント値に start segment の値が足されていることがわかります。

また SS, CS セグメントについてもヘッダで示された Initial SS, Initial CS の値に 0x0592 を足して relocation したセグメント値が使われています。

このように実行時に start segment が決定され relocation が行われた様子を確認することができました。start segment がどう決まるのか、というのはまだ整理できていないのですが、他のヘッダ情報を理解すれば自ずとわかるのかなと思います。

[1]: https://wiki.osdev.org/MZ
[2]: http://www.textfiles.com/programming/FORMATS/exefs.pro
[3]: http://www.tavi.co.uk/phobos/exeformat.html#reloctable
[4]: https://ja.wikipedia.org/wiki/Portable_Executable
[5]: {% post_url 2018-10-01-freedos %}
