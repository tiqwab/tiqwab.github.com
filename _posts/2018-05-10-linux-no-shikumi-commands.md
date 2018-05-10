---
layout: post
title: 『Linuxのしくみ』 に登場するコマンドの整理
tags: "linux"
comments: true
---

最近 [Linuxのしくみ ~実験と図解で学ぶOSとハードウェアの基礎知識][1] を読みました。この本では豊富な図と実験で Linux の仕組みの基本が解説されているのですが、その中で自分には馴染みの薄い、でも有用そうなコマンドが豊富に出現していたので備忘録を兼ねて整理したいなと思いました。

コマンド一覧:

- [strace](#strace)
- [sar -P](#sar-P)
- [readelf](#readelf)
- [taskset](#taskset)
- [sar -q](#sar-q)
- [time](#time)
- [free](#free)
- [sar -r](#sar-r)
- [ps -eo](#ps-eo)
- [sar -B](#sar-B)
- [sar -W](#sar-W)
- [sar -d](#sar-d)
- [iostat](#iostat)

### 第2章 ユーザモードで実現する機能

<div id="strace" />

#### strace

- 指定したコマンドを実行し、その実行中に呼ばれたシステムコールを出力する
- 各システムコールの詳細は `man 2 <system_call>` でわかる

実行例:

```bash
$ strace echo foo
execve("/usr/bin/echo", ["echo", "foo"], 0x7fffe8846de8 /* 43 vars */) = 0
brk(NULL)                               = 0x55f64ccb1000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=136757, ...}) = 0
...
write(1, "foo\n", 4foo
)                    = 4
close(1)                                = 0
close(2)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

デフォルトの出力だと `<systemcall>(<args>) = <return value>` のような形式

主要なオプション:

- `-p`
  - コマンドではなく指定したプロセスに対して `strace` を実行する
- `-t`, `-tt`
  - 各システムコールが発行された時刻を表示する
- `-T`
  - 各システムコールにかかった時間 (microseconds) を表示する
- `c`
  - 各システムコールのサマリ (e.g. 呼出回数) を表示する

他参考:

- [strace コマンドの使い方をまとめてみた - sonots:blog](http://blog.livedoor.jp/sonots/archives/18193659.html)

<div id="sar-P" />

#### sar -P

- CPU の使用率を表示する
- ユーザモードでの使用率は `user + nice` (`nice` は優先度を変更したプロセス実行中)
- カーネルモードでの使用率は `system`

実行例:

1 秒間隔で全プロセッサの使用率を表示する

```bash
$ sar -P ALL 1
Linux 4.16.5-1-ARCH (host) 	05/06/2018 	_x86_64_	(4 CPU)

02:06:50 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
02:06:51 PM     all      1.27      0.00      0.25      0.00      0.00     98.48
02:06:51 PM       0      1.00      0.00      1.00      0.00      0.00     98.00
02:06:51 PM       1      1.02      0.00      0.00      0.00      0.00     98.98
02:06:51 PM       2      3.03      0.00      0.00      0.00      0.00     96.97
02:06:51 PM       3      0.00      0.00      0.00      0.00      0.00    100.00

02:06:51 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
02:06:52 PM     all      1.00      0.00      0.75      0.00      0.00     98.25
02:06:52 PM       0      1.98      0.00      0.99      0.00      0.00     97.03
02:06:52 PM       1      0.99      0.00      0.99      0.00      0.00     98.02
02:06:52 PM       2      1.01      0.00      0.00      0.00      0.00     98.99
02:06:52 PM       3      0.00      0.00      1.01      0.00      0.00     98.99
```

### 第3章 プロセス管理

<div id="readelf" />

#### readelf

- ELF (Executable and Linkable Format) についての情報を表示する
  - Linux ディストリビューションで広く採用されている実行ファイルのフォーマット

使用例:

ELF ヘッダを表示する

```bash
$ readelf -h hello
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x540
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6520 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28
```

主要オプション:

- `-S`
  - セクションヘッダの表示
  - 各セクションのファイル内オフセット、サイズ、開始アドレスがわかるらしい
- `-l`
  - プログラムヘッダの表示

ELF については `man 5 elf`

### 第4章 プロセススケジューラ

<div id="taskset" />

#### taskset

- CPU affinity のセットや表示を行う
  - セット: 指定した CPU で処理を実行させることができる
  - 表示: 指定したプロセスがどの CPU で処理され得るかがわかる

使用例:

指定した CPU affinity でコマンド実行

```bash
$ taskset -c 0 yes > /dev/null &

$ sar -P ALL 5
Linux 4.16.5-1-ARCH (host) 	05/07/2018 	_x86_64_	(4 CPU)

09:42:43 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
09:42:48 PM     all      9.56      0.00     15.87      0.00      0.00     74.56
09:42:48 PM       0     36.87      0.00     63.13      0.00      0.00      0.00
09:42:48 PM       1      1.20      0.00      0.00      0.00      0.00     98.80
09:42:48 PM       2      0.20      0.00      0.40      0.00      0.00     99.40
09:42:48 PM       3      0.00      0.00      0.00      0.00      0.00    100.00
```

指定したプロセスの CPU affinity の表示

```bash
$ taskset -p 2508
pid 2508's current affinity mask: 1
```

主要オプション:

- `-c`
  - 対象の CPU を指定して処理を実行する
- `-p`
  - 対象プロセスの affinity mask を表示
    - 例えば CPU を 4 つ持つとした場合、 cpu0 でのみ実行されるなら `1`, cpu0, 1, 2, 3 なら `f` のような 16 進数表記で表される

<div id="sar-q" />

#### sar -q

- ランキューとロードアベレージに関する情報を表示する
  - どちらもサーバへの負荷を監視するのに有用な項目
  - これらが高い場合に、それが CPU 負荷なのか IO 負荷なのか別のメトリクスを見て... という風に進めていくのが良さそう

使用例:

```bash
$ taskset -c 0 yes > /dev/null &
$ taskset -c 1 yes > /dev/null &
$ sar -q 1
Linux 4.16.5-1-ARCH (host) 	05/07/2018 	_x86_64_	(4 CPU)

10:44:34 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
10:44:35 PM         2       380      0.52      0.50      0.61         0
10:44:36 PM         2       380      0.52      0.50      0.61         0
10:44:37 PM         2       380      0.52      0.50      0.61         0
10:44:38 PM         2       380      0.63      0.52      0.62         0
Average:            2       380      0.55      0.51      0.61         0
```

他参考:

- [ロードアベレージを確認できる4つのコマンドと4つの原因の対処方法 - フリエン](https://furien.jp/columns/146/)
- [コマンドによる「負荷」の原因切り分け - kitak/doc.md](https://gist.github.com/kitak/6349463)
- [マルチコア時代のロードアベレージの見方 - naoyaのはてなダイアリー](http://d.hatena.ne.jp/naoya/20070518/1179492085)

<div id="time" />

#### time

- 指定したコマンドの実行時間を計測する

使用例:

```bash
$ time for i in $(seq 1000000); do echo $i > /dev/null; done

real	0m9.151s
user	0m5.785s
sys	0m3.299s
```

- real はコマンド実行にかかった時間
- user はユーザモードで CPU を使用した時間
- system はカーネルがシステムコール実行のために CPU を使用した時間

### 第5章 メモリ管理

<div id="free" />

#### free

- システムが持つ現在の空きメモリ、使用中メモリの量を表示する

使用例:

```bash
$ free
              total        used        free      shared  buff/cache   available
Mem:       16222612     1337712    13906804      183824      978096    14870940
Swap:             0           0           0
```

- total: Total installed memory
- used: 使用中のメモリ (total - free - buffers - cache)
- free: 空きメモリ (MemFree and SwapFree in /proc/meminfo)
- shared: tmpfs によって使用されているメモリ (Shmem in /proc/meminfo)
- buff: カーネルがバッファキャッシュのために使用しているメモリ (Buffers in /proc/meminfo)
- cache: カーネルがページキャッシュのために使用しているメモリ (Cache and SReclaimable in /proc/meminfo)
- available: 実質的な空きメモリ。 free + カーネル内メモリ領域のうち開放できるもの (MemAvailable in /proc/meminfo)

デフォルトでは KiB 表示

<div id="sar-r" />

#### sar -r

- 指定した間隔でシステムが持つ現在の空きメモリ、使用中メモリの量を表示する

使用例:

```bash
$ sar -r 1
Linux 4.16.5-1-ARCH (host) 	05/08/2018 	_x86_64_	(4 CPU)

09:38:58 PM kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
09:38:59 PM  13946764  14915372   2275848     14.03     82524    788808   3425400     21.11   1044640    979128       180
```

kbmemfree, kbbuffers + kbcached あたりは free で見たそれに対応するらしい

<div id="ps-eo" />

#### ps -eo

- 各プロセスの指定した情報を表示する

使用例:

```bash
$ ps -eo pid,comm,vsz,rss,maj_flt,min_flt | grep -e [f]irefox
  685 firefox         2424600 381576  688 2564544
```

オプション:

- `-e`
  - 全プロセスを対象とする
- `-o`
  - 出力フォーマットを指定する

`free` や `sar -r` がシステム全体の情報を表示するのに対し、 `ps` なら特定のプロセスに割り当てられている仮想メモリや実メモリがわかる

<div id="sar-B" />

#### sar -B

- 指定した間隔でページングに関する情報を表示する

使用例:

```bash
$ sar -B 1
Linux 4.16.5-1-ARCH (host) 	05/08/2018 	_x86_64_	(4 CPU)

10:11:53 PM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
10:11:54 PM      0.00      0.00     11.00      0.00      7.00      0.00      0.00      0.00      0.00
```

- pgpgin/s: 秒あたりのディスクからページインしたキロバイト数
- pgpgout/s: 秒あたりのディスクへページアウトしたキロバイト数
  - ただページキャッシュを伴わないディスクへの書き込みもここに含まれるらしい
- falut/s: 秒あたりのページフォールト (メジャー + マイナー) 数
- majflt/s: 秒あたりのメジャーフォールト数

<div id="sar-W" />

#### sar -W

- 指定した間隔でスワッピングに関する情報を表示する

使用例:

```bash
$ sar -W 1
Linux 4.16.5-1-ARCH (host) 	05/08/2018 	_x86_64_	(4 CPU)

10:29:21 PM  pswpin/s pswpout/s
10:29:22 PM      0.00      0.00
```

- pswpin/s: 秒あたりのスワップインされたページ数
- pswpout/s: 秒あたりのスワップアウトされたページ数

<div id="sar-d" />

#### sar -d

- 指定した間隔でストレージデバイスに関する情報を表示する

使用例:

```
$ sar -d -p 1
Linux 4.16.5-1-ARCH (host) 	05/08/2018 	_x86_64_	(4 CPU)

10:53:41 PM       DEV       tps     rkB/s     wkB/s   areq-sz    aqu-sz     await     svctm     %util
10:53:42 PM       sda      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

オプション:

- `-p`
  - デバイス名を整形?表示する
  - オプションなしだと `dev8-0` のような表記になる

### 第8章 ストレージデバイス

<div id="iostat" />

#### iostat

- CPU や デバイス、パーティションの I/O 統計値を表示する

使用例:

CPU について

```bash
$ iostat -c 1
Linux 4.16.5-1-ARCH (host) 	05/09/2018 	_x86_64_	(4 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          10.17    0.01    4.84    0.02    0.00   84.96

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.24    0.00    0.25    0.00    0.00   98.51

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.50    0.00    0.75    0.00    0.00   98.75
```

ディスク I/O について

```
$ iostat -d -x 1
Linux 4.16.5-1-ARCH (host) 	05/09/2018 	_x86_64_	(4 CPU)

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
sda              0.09    0.17      2.13      4.73     0.00     0.21   1.61  55.52    0.22    2.42   0.00    23.99    27.67   0.54   0.01

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
sda              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
sda              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00
```

いずれも最後に渡しているパラメータは統計の間隔 (s)

[1]: https://www.amazon.co.jp/%E8%A9%A6%E3%81%97%E3%81%A6%E7%90%86%E8%A7%A3-Linux%E3%81%AE%E3%81%97%E3%81%8F%E3%81%BF-%E5%AE%9F%E9%A8%93%E3%81%A8%E5%9B%B3%E8%A7%A3%E3%81%A7%E5%AD%A6%E3%81%B6OS%E3%81%A8%E3%83%8F%E3%83%BC%E3%83%89%E3%82%A6%E3%82%A7%E3%82%A2%E3%81%AE%E5%9F%BA%E7%A4%8E%E7%9F%A5%E8%AD%98-%E6%AD%A6%E5%86%85-%E8%A6%9A/dp/477419607X
