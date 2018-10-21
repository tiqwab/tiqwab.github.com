---
layout: post
title: FreeDOS での開発にまつわる etc.
tags: "qemu,cpu,x86,486,freedos"
comments: true
---

『初めて読む 486』 を読みながら MS-DOS (正確には FreeDOS) 環境で開発する上で役に立った知識や気になった点をつらつら書き留めています。

記事の内容は全て FreeDOS 環境における内容なので、考え方として共通する部分はあるにしても現代のプログラミングで直接役に立つものはあまり無いと思います。

1. [wd について](#wd)
2. [QEMU monitor について](#qemu-monitor)
3. [アセンブリ言語で実行可能ファイルを作成する](#asm-exe)
4. [ソフトウェア割り込みについて](#interrupt)
5. [呼出規約について](#calling-convention)
6. [アセンブリ言語から C の関数を呼ぶ](#call-c-func)

環境:

- OS (ホスト): Arch Linux (Linux kernel: 4.18.12)
- QEMU: 3.0.0
- FreeDOS: 1.2

---

<div id="wd" />

### 1. wd について

C の嬉しいところであり同時に辛いところはレジスタやメモリを意識してプログラミングできることだと思います。ただ C (と MS-DOS 環境) 初心者な現状ではその力の代償に様々なバグを埋め込んでしまいデバッガが欠かせません。

[先日用意した FreeDOS][1] では C 開発環境として Open Watcom を使用しており、そこでは wd というデバッガが提供されています。これを使えばソースコードやアセンブリコード、レジスタ、メモリ等を確認しながらプログラムの動作を確認することができます。

wd を使う場合、事前に C やアセンブリファイルはデバッグ情報付きでコンパイルしておきます。これは必須ではありませんがソースコードをもとに breakpoint を貼る、といったことをやるには必要な手順です。

```
# for assembly with d1 option
> wasm -ms -d1 a.asm
# for C source with d3 option
> wcl -ecc -ms -d3 main.c
```

デバッガを起動するには対象の実行ファイルを指定して wd を実行します。

```
> wd main.exe
```

デバッガ画面は以下のようになっています。

<img
  src="/images/freedos-dev/wd1.png"
  title="wd1"
  alt="wd1"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

操作は全てキーボードで行います。今のところ

- F6 で各ウィンドウを順繰りに移動
- Alt + F, R, ... で上部メニューの各項目を選択
- e で指定したメモリ番地を表示
- space で step over
- i で step in

ぐらいの操作がわかれば何とかなっています。

例として以下の関数 `add` の呼び出し `add(1, 2)` の実行途中で breakpoint を貼ってみます。

```c
int add(int x, int y)
{
  return x + y;
}
```

起動した wd の Assembly 画面、あるいは Source 画面の該当箇所まで移動し、Alt + B (Break メニュー) -> At Cursor で breakpoint を設定し、Alt + R (Run メニュー) -> Go で実行できます。

<img
  src="/images/freedos-dev/wd2.png"
  title="wd2"
  alt="wd2"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

画面内 CS, IP レジスタが示すように 14FC:0042 の命令を実行中で、ソースでいうと x + y を計算する直前です。 mov のオペランドである `08[bp]` の指すメモリ番地を見るためには e を押して SS, BP レジスタの指す 16F9:0B50 を入力します。すると画面下部に該当箇所まわりの Memory が表示され、16F9:0B58 から 4 bytes を確認すると、下位アドレスから `01 00 02 00` となっており、確かに x, y として 1, 2 が使用されているな... といったことがわかります。

一応 wd の本気の資料としては [こちら][2] があるので、もしやりたい操作がわからなければこちらを確認することができます。

<div id="qemu-monitor" />

### 2. QEMU monitor について

FreeDOS 環境を QEMU で作成している場合、[QEMU monitor][3] を使うことでもレジスタやメモリの値を確認することができます。wd が使えるならばそちらの方が扱いやすいのですが、例えば 『はじめて読む 486』 のようにプロテクトモードにおける割り込みを実装しているという場合、breakpoint がうまく機能しないことがあります (というのは恐らく breakpoint が int 3 というソフトウェア割り込みで実装されているため)。こうした場合 OS よりも外側に存在する QEMU からエミュレーションの様子を見ることで状況を確認することができます。

QEMU monitor へは QEMU 起動中に Ctrl + Alt + 2 を押下することで移行できます。OS へ戻る場合は Ctrl + Al + 1 です。

レジスタの情報を見る場合は表示されたコマンドプロンプトで `info registers` を入力します。

```
(qemu) info registers
EAX=0000090f EBX=00160592 ECX=00000000 EDX=00000020
ESI=00210400 EDI=002115d0 EBP=002121f6 ESP=002121c4
EIP=000008ca EFL=00000046 [---Z-P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0592 00005920 0000ffff 00009300
CS =09cb 00009cb0 0000ffff 00009b00
SS =0592 00005920 0000ffff 00009300
DS =0592 00005920 0000ffff 00009300
FS =0158 00001580 0000ffff 00009300
GS =0020 00000200 0000ffff 00009300
...
```

一連のレジスタとそこにセットされている値が表示されました。

(これのセグメントレジスタの表記がちょっとわからない。486 はセグメントレジスタのサイズは 16 bit でセグメントディスクリプタキャッシュが 8 bytes だと思っているけど、それだと表示がうまく結びつかない)

メモリを見る場合は xp コマンドを入力します。以下は 「アドレス 33854 から 40 bytes, 16 進数表記で表示」 というコマンドになります。

```
 (qemu) xp /40xb 33854
 000000000000843e: 0x22 0x00 0x5e 0x0b 0x6e 0x03 0x71 0x03
 0000000000008446: 0x7d 0x00 0x01 0x00 0x02 0x00 0x54 0x02
 000000000000844e: 0x00 0x00 0x6e 0x03 0x71 0x03 0x00 0x00
 0000000000008456: 0x00 0x00 0x70 0x0b 0x29 0x04 0x8f 0x07
 000000000000845e: 0xe8 0x02 0x7c 0x20 0x00 0x00 0x00 0x00
```

参考:

- [QEMU/Monitor - Wikibooks][3]
- [qemu monitor を使って自作 OS をデバッグする][4]

<div id="asm-exe" />

### 3. アセンブリ言語で実行可能ファイルを作成する

新しい言語に触れるときにお馴染みの Hello World ですが、Open Watcom のアセンブラである wasm の場合以下のように書けます。 wasm は MASM と不十分ながら互換性があるとのこと (by [Wikipedia][6]) で、軽く触っている感じは MASM の資料が十分参考になりそうです。

```asm
.model small
.stack 100h

.data
msg db 'Hello world!$'

.code
start:
  mov ax,seg msg
  mov ds,ax
  mov ah,09h
  lea dx,msg
  int 21h
  mov ax,4c00h
  int 21h

end start
```

上のコードはほとんど参考の [x86 assembly language - Wikipedia][5] から引っ張ってきているのですが、そのままだと出力がおかしかったので、はじめに `seg` operator を使用して DS レジスタをセットし直しています。

上のアセンブリファイルにアセンブラ、リンカを通せば実行ファイルが作成されます。

```
> wasm -ms sample.asm
> wcl -ecc -ms sample.obj
> sample.exe
Hello world!
```

自分は MASM をちゃんと書くのは 『はじめて読む 486』 本が初めてだったのですが、そこで記述されているアセンブリと上のコードはちょっと様子が違って見えます。どうやら本とは違い、上は MASM の簡略化セグメントというのを使用した形式のようです。上と同じ記述を簡略化セグメントを使用しないで書き換えてみたコードが下記になります。

```asm
.386p

;; can group multiple segments if necessary
;; DGOUP group _DATA, _BSS
;;       assume ds:DGROUP

_DATA segment byte public use16 'DATA'

msg db 'Hello world!$'

_DATA ends

_STACK segment byte public use16 'STACK'
_STACK ends

_TEXT segment byte public use16 'CODE'
      assume cs:_TEXT, ds:_DATA

public _start
_start proc near
       mov ax,seg _DATA
       mov ds,ax
       mov ah,09h
       mov dx,offset msg
       int 21h
       mov ax,4c00h
       int 21h
_start endp

_TEXT ends
      end _start
```

`<segment_name> segment ... <segment_name> ends`  で一つのセグメントを定義します。このように自分で明確にセグメントを定義する形式だと、何となく実行時どのようにプログラムが配置されるかもイメージできますね。

またプログラムのエントリポイントは最後の `end <proc_name>` で指定した procedure になるので、ここでは `end _start` としています。

参考:

- [x86 assembly language - Wikipedia][5]

<div id="interrupt" />

### 4. ソフトウェア割り込みについて

上のアセンブリファイルで何気なく `int` 命令を使用しましたが、これはソフトウェア割り込み (トラップ) を発生させるための命令です。 `int 21h` の場合、割り込み番号 21h 番の割り込み処理ルーチンを実行するということになります。

MS-DOS では割り込み番号 21h は DOS の提供する API をコールするために使用されます。割り込み番号  21h は単一の処理を実装しているわけではなく、 ah レジスタにセットした値に応じて異なる処理を実行します。また処理によっては指定されたレジスタを介して値の受け渡しを行います。

(これは MS-DOS におけるシステムコール、という捉え方ができそうだけど、あまりちゃんとした資料でそういう表現を見ないのは MS-DOS という OS においては権限という概念が無いためかも。システムコールというと通常のアプリケーションには許されていない操作を OS に依頼できるというニュアンスがあるけれど、int 21h の場合は単にハードウェア毎の差異を抽象化して処理が書ける、というのが主目的な気がする)

例えば上のアセンブリファイルの場合、以下の 2 つの用途で 21h の割り込みを使用しました。

#### 画面への文字列表示

- ah レジスタ: 09h
- 入力: ds, dx レジスタ
  - ds:dx で指定されるメモリに存在する文字列を表示する

#### プログラムの終了

- ah レジスタ: 4Ch
- 入力: al レジスタ
  - al で指定される値を終了コードとしてプログラムを終了する

参考:

- [DOS API - Wikipedia][7]
- [MS-DOS 割り込み][8]

<div id="calling-convention" />

### 5. 呼出規約について

呼出規約 (calling convention) はサブルーチンの呼び出し方を定めたもので ABI の一部です。具体的にはパラメータの渡し方や計算結果の受け取り方等を定めます。

普段高級言語だけでプログラムを書いている場合全く気にしないレイヤの話ですが、今回のようにアセンブリ言語でサブルーチンを記述する場合は必要になる知識です。呼び出し規約を統一すれば自作のアセンブリ言語で書いたサブルーチンから C の関数を呼べたりその逆も可能になります。

x86 においてメジャーな呼出規約は cdecl と呼ばれるものです。この解説としてネット上で見つかるものでは [Guide to x86 Assembly][10] の Calling Convention 項がわかりやすくまとめられていて参考になると思います。

簡単な例として 2 つの int 型の和を計算して返す関数をアセンブリ言語で定義し、C から呼んでみます。

add.asm:

```asm
.486

_TEXT segment byte public use16 'CODE'
      assume cs:_TEXT

;; int add(int x, int y)
public _add
_add proc near
     push bp         ; (1)
     mov  bp,sp      ; (1)
     mov  ax,[bp+4]  ; [bp+4] = parameter x (2)
     add  ax,[bp+6]  ; [bp+6] = parameter y (2)
     pop  bp         ; (1)
     ret
_add endp

_TEXT ends
      end
```

main.c:

```c
#include <stdio.h>

int add(int x, int y);

void main(void)
{
  int res;

  res = add(2, 4);
  printf("res = %d\n", res);
}
```

実行結果:

```
> wasm -ms -d1 add.asm
> wcl -ecc -ms -d3 main.c add.obj
> main.exe
res = 6
```

呼出規約ではサブルーチンの呼び出し側 (caller) と呼ばれる側 (callee) それぞれに守るべきルールを定めています。そうしたものの一つとしてサブルーチン呼出前後で保持すべきレジスタというのがあり、保持すべきレジスタについて caller はサブルーチン前後でその値が変更されないと想定できる一方で、保持されないレジスタについては必要であれば自分で別途値を保存しておく必要があります。 `add.asm` の (1) では、はじめに bp レジスタの値を push し、最後に pop することでサブルーチン呼出前後で変えてはならない bp レジスタの値を保持しています。

サブルーチン内部では本処理を始める前に `mov bp,sp` で bp レジスタの値を上書いていますが、これにより higher address には return address, 関数のパラメータが並び、lower address にはローカル変数が並ぶといった感じになるのでパラメータへのアクセスの見通しがよくなります。

実際に wd で `mov bp,sp` 実行前のスタック、レジスタの状態を見ると以下のようになります。

<img
  src="/images/freedos-dev/calling-convention1.png"
  title="wd1"
  alt="wd1"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

命令実行後は `[bp] (0B52h)` で旧 bp の値 (OB5Eh)、 `[bp+2] (0B54h)` で return address (002Dh)、 `[bp+4] (0B56h)` で引数 x の値である 2, `[bp+6] (0B58h)` で引数 y の値である 4 にアクセスできることがわかります。 `add.asm` 内 (2) でも示しましたがサブルーチンへのパラメータは C の関数定義でいう右から順にスタックに積まれています。

最後に cdecl ではサブルーチンの返り値を ax レジスタにセットするので `[bp+4]` , `[bp+6]` の add した結果を ax レジスタにセットしています。

もう一つローカル変数や si, di レジスタを使用する例を見てみます。 si, di は bp と同様 callee が呼び出し前後で保持すべきレジスタです。ここでは `addp` という 2 つの `int *` を受け取り、それらが指す整数と 1 を足した結果を返すという関数を定義します。なおここでは例として使用するために多少冗長な実装をしています。

addp.asm:

```asm
.486

_TEXT segment byte public use16 'CODE'
      assume cs:_TEXT

;; int addp(int *x, int *y)
public _addp
_addp proc near
      push bp
      mov  bp,sp
      sub  sp,2              ;; Allocate space for local variable
      push di                ;; Store di register
      push si                ;; Store si register

      mov  word ptr [bp-2],1 ;; size directive is necessary
      mov  ax,[bp-2]

      mov  si,[bp+4]
      mov  di,[bp+6]
      add  ax,[si]
      add  ax,[di]

      pop  si                ;; Restore si register
      pop  di                ;; Restore di register
      mov  sp, bp            ;; Deallocate local variable
      pop  bp
      ret
_addp endp

_TEXT ends
      end
```

main.c:

```c
#include <stdio.h>

int addp(int *x, int *y);

void main(void)
{
  int res, a, b;

  a = 4;
  b = 9;
  res = addp(&a, &b);
  printf("res = %d\n", res);
}
```

実行:

```
> wasm -ms -d1 addp.asm
> wcl -ecc -ms -d3 main.c addp.obj
> main.exe
res = 14
```

始めに bp レジスタの保存とセットを行うというのは add サブルーチンと同様ですが、addp ではそのあとにローカル変数格納用のスペースや保持すべき di, si レジスタの値をスタックに格納しています。

`mov  word ptr [bp-2],1` 実行後のスタックの状態は以下のようになります。 `[bp-2]` に 0001h, `[bp-4]` に di レジスタの値 036Ch、 `[bp-6]` に si レジスタの値 0372h が保存されています。

<img
  src="/images/freedos-dev/calling-convention2.png"
  title="wd1"
  alt="wd1"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

また関数のパラメータ x, y としては 0B5Ah, 0B5Ch が渡されており、 ds レジスタの 16FAh から 16FA:0B5A, 16FA:0B5C の指すメモリ番地を見に行くと、それぞれ 4, 9 という想定通りの値が渡されていることがわかります。

<img
  src="/images/freedos-dev/calling-convention3.png"
  title="wd1"
  alt="wd1"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

ちなみにポインタのサイズですが 2 bytes なのは small model でコンパイルしているためだと思います。small model ではプログラム中でセグメントレジスタの値が変更されないのでポインタとしては offset 2 bytes 分あれば任意のメモリ番地を特定することができます。逆に言えばセグメントレジスタが固定とは限らない large model ではポインタとして 4 bytes (segment 2bytes, offset 2 bytes) 必要なはずで、実際に試してみると確かにそのようになっています。

```c
#include <stdio.h>

void main(void)
{
  printf("size of ptr: %d\n", sizeof(int *));
  printf("size of __far ptr: %d\n", sizeof(int __far *));
}
```

```
# compile for small model
> wcl -ecc -ms ptr_size.c
> ptr_size.exe
size of ptr: 2
size of __far ptr: 4

# compile for large model
> wcl -ecc -ls ptr_size.c
> ptr_size.exe
size of ptr: 4
size of __far ptr: 4
```

例外として small model でも上のように `__far` キーワードをつけると segment 2 bytes を含めた 4 bytes のサイズになるようです。以下のコードを small model でコンパイルし、check 呼出直後のスタックを見ると、nearX は offset のみの 2bytes (0064h), farY は segment を含めた 4 bytes (1546h:0062h) が積まれていることがわかります。

main.c:

```c
#include <stdio.h>
#include <dos.h>

int x, y;
int __far *z;

void check(int *nearX, int __far *farY);

void main(void)
{
  struct SREGS seg;

  segread(&seg);
  x = 1;
  y = 2;
  z = MK_FP(seg.ds, (unsigned short) (&y));
  check(&x, z);
}
```

check.asm:

```asm
_TEXT segment byte public use16 'CODE'
      assume cs:_TEXT

;; void check(int *nearX, int __far *farY)
public _check
_check proc near
       ret       ;; do nothing
_check encp

_TEXT ends
      end
```

実行:

```
> wasm -ms -d1 check.asm
> wcl -ecc -ms -d3 main.c check.obj
> wd main.exe
```

<img
  src="/images/freedos-dev/funcptr1.png"
  title="wd1"
  alt="wd1"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

参考:

- [呼出規約 - Wikipedia][9]
- [Guide to x86 Assembly][10]

<div id="call-c-func" />

### 6. アセンブリ言語から C の関数を呼ぶ

C の extern 的な感じで extrn をつければ良さそうです。下の例では関数ポインタの場合、 DATA ではなく BSS セグメントに配置しないとだめでした。 DATA は初期値あり、BSS はなしの変数配置場所ということですが、このあたりの挙動はまだ勉強不足ですね...

main.c:

```c
#include <stdio.h>

int doAdd(int x, int y);
int doAdd2(int x, int y);
int (*f)(int, int);

int add(int x, int y)
{
  return x + y;
}

void main(void)
{
  int ret;

  f = add;
  ret = doAdd(5, 9);
  printf("ret = %d\n", ret);
  ret = doAdd2(6, 10);
  printf("ret = %d\n", ret);
}
```

main\_a.asm:

```asm
.486

_BSS segment byte public use16 'BSS'
      assume ds:_DATA

extrn _f:word ;; 2 bytes variable defined in elsewhere

_BSS ends

_TEXT segment byte public use16 'CODE'
      assume cs:_TEXT

extrn _add:near ;; _add exists in the same segment

;; int doAdd(int x, int y)
public _doAdd
_doAdd proc near
       push bp
       mov  bp,sp

       mov  ax,[bp+4]
       mov  cx,[bp+6]
       push cx
       push ax
       call _add

       mov  sp,bp
       pop  bp
       ret
_doAdd endp

public _doAdd2
_doAdd2 proc near
        push bp
        mov  bp,sp

        mov  ax,[bp+4]
        mov  cx,[bp+6]
        push cx
        push ax
        call _f

        mov  sp,bp
        pop  bp
        ret
_doAdd2 endp

_TEXT ends
      end
```

実行:

```
> wasm -ms -d1 main_a.asm
> wcl -ecc -ms -d3 main.c main_a.obj
> main.exe
ret = 14
ret = 16
```

[1]: {% post_url 2018-10-01-freedos %}
[2]: https://github.com/open-watcom/travis-ci-ow-builds/blob/master/docs/wd.pdf
[3]: https://en.wikibooks.org/wiki/QEMU/Monitor
[4]: http://yuyubu.hatenablog.com/entry/2018/06/30/qemu_monitor%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%A6%E8%87%AA%E4%BD%9COS%E3%82%92%E3%83%87%E3%83%90%E3%83%83%E3%82%B0%E3%81%99%E3%82%8B
[5]: https://en.wikipedia.org/wiki/X86_assembly_language
[6]: https://en.wikipedia.org/wiki/Open_Watcom_Assembler
[7]: https://en.wikipedia.org/wiki/DOS_API
[8]: http://software.aufheben.info/oohdyna/ms-dos.html
[9]: https://ja.wikipedia.org/wiki/%E5%91%BC%E5%87%BA%E8%A6%8F%E7%B4%84
[10]: http://www.cs.virginia.edu/~evans/cs216/guides/x86.html
