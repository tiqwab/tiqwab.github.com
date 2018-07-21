---
layout: post
title: Linux カーネルをビルド、インストールしてみた
tags: "linux"
comments: true
---

ここ半年程度 CPU まわりとちまちま遊んでいたのですが、そろそろ一つレイヤを上げて Linux のような OS 部分にも手を出したいなと思っています。そのための第一歩として自分でカーネルのビルドとインストールをしてみました。

対象 Linux カーネルバージョンは 4.17.2 です。

- [Vagrant で Arch Linux 環境を用意](#vagrant)
- [カーネルのビルドに必要なパッケージのインストール](#packages)
- [カーネルのダウンロード](#download-kernel)
- [カーネル設定](#kernel-config)
- [システムコール追加](#system-call)
- [カーネルのコンパイル](#compile-kernel)
- [モジュールのコンパイル](#compile-module)
- [/boot にカーネルをコピー](#copy-kernel-image)
- [初期 RAM ディスクの生成](#initramfs)
- [ブートローダの設定](#boot-loader)
- [動作確認](#run)

<div id="vagrant" />

### Vagrant で Arch Linux 環境を用意

- カーネルの動作確認を実機でやるのは辛い
- 今回は Vagrant (with VirtualBox provider) で環境を用意した
  - ただ Vagrant をはさむとブート周りが扱いづらくて失敗だった気がする
  - VirtualBox 直接触るか [QEMU][21] を使うのが良さそう
  - [QEMUを使って最新のLinuxカーネル(v4.0-rc7)を起動する][22]
- [Vagrant - ArchWiki][1] を見て [こちら][2] の box を使用した
  - 何故かやたら遅かったので box を手動で落として `vagrant box add` した

```
$ curl -O http://cloud.terry.im/vagrant/archlinux-x86_64.box
$ vagrant add box archlinux-x86_64.box --name archlinux
```

- Vagrantfile を用意して `vagrant up` で VM 立ち上げ

```
$ cat Vagrntfile
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "archlinux"
  config.vm.network "private_network", ip: "192.168.33.10"
  # config.vm.network "forwarded_port", guest: 8888, host:8888

  config.vm.provider "virtualbox" do |vm|
    vm.memory = 4096
  end
end

$ vagrant up
```

- VM に入り、まずパッケージの更新
  - `libalpm.so.10` のエラーで躓いたので [こちら][3] に従って解決

```
$ vagrant ssh

# ...after resolving `libalpm.so.10` error
$ yaourt -Syua
```

<div id="packages" />

### カーネルのビルドに必要なパッケージのインストール

- [Minimal requirements to compile the Kernel][5] に従いパッケージのインストール
  - ただ環境によって必要でないものもある
- 現環境では bc コマンドのみ追加で必要そうなのでそれをインストールする
  - bc は数値計算用のコマンド
  - [Why is 'bc' required to build the Linux Kernel?][4]

```
$ yaourt -S bc
```

<div id="download-kernel" />

### カーネルのダウンロード

- [mirros.edge.kernel.org][6] から
  - 今回の対象バージョンは 4.17.2

```
$ curl -O https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.17.2.tar.gz
```

<div id="kernel-config" />

### カーネル設定

- カーネル設定はたくさんあるので一から作成するのは厳しい
- 既存の (稼働中の) OS が使用している設定をコピーし、必要な箇所だけ修正するという方針で
  - `CONFIG_LOCALVERSION` は既存と異なるものにする。 変更後の Linux カーネルバージョンは `4.17.2-tiqwab` のようになる

```
# decompress
$ tar xvf linux-4.17.2.tar.gz
$ cd linux-4.17.2

# copy configs in the running OS
$ zcat /proc.config.gz > .config
$ make oldconfig

# ... edit CONFIG_LOCALVERSION
$ grep "CONFIG_LOCALVERSION" .config
CONFIG_LOCALVERSION="-tiqwab"
# CONFIG_LOCALVERSION_AUTO is not set
```

<div id="system-call" />

### システムコール追加

- ビルドの成功を確認するために少しカーネルに手を加える
- ここでは簡単なシステムコールを追加する
- まだざっとしか理解していないけれどシステムコールを追加するには以下の 2 つをやれば良い
  - システムコール用のテーブルにエントリを追加
  - 関数を定義
- システムコールのような CPU アーキテクチャ依存のコードは `arch` 以下に置かれている

```
# ... edit arch/x86/entry/syscalls/syscall_64.tbl
$ grep "333" arch/x86/entry/syscalls/syscall_64.tbl 
333 common  hello  __x64_sys_hello

# ... add system call
$ sed -n 2644,2652p kernel/sys.c
SYSCALL_DEFINE1(hello, char *, msg)
{
    char buf[256];
    long copied = strncpy_from_user(buf, msg, sizeof(buf));
    if (copied < 0 || copied == sizeof(buf))
        return -EFAULT;
    printk(KERN_INFO "hello \"%s\"\n", buf);
    return 0;
}
```

一旦この時点の box を作成しておく。

```
$ vagrant halt
$ vagrant package
$ vagrant box add customized_linux_build package.box
```

<div id="compile-kernel" />

### カーネルのコンパイル

- 自分の環境だと 2 時間ぐらいかかって終わったぽい
  - `make -j 4` のようにすると並列化できそう?

```
$ make
```

<div id="compile-module" />

### モジュールのコンパイル

- こちらは割と早く終わった

```
$ make modules_install

# Compiled modules are copied to /lib/modules
$ ls -F /lib/modules
4.17.2-1-ARCH/  4.17.2-tiqwab/  extramodules-4.17-ARCH/
```

<div id="copy-kernel-image" />

### /boot ディレクトリにカーネルをコピー

```
$ cp arch/x86_64/boot/bzImage /boot/vmlinuz-linux-tiqwab
```

<div id="initramfs" />

### 初期 RAM ディスクの生成

- 初期 RAM ディスク (initial ramdisk)
  - Linux カーネルのブート時に使用される一時的なルートファイルシステム
  - メモリ上に読み込まれる
  - 真のルートファイルシステムをマウントする前にファイルシステムを必要とする作業を行うために使用される

以下 [initrd - Wikipedia][12] より引用。

> ブートローダはカーネルとinitrdイメージをメモリ上にロードし、カーネルを起動する際に initrd のメモリアドレスを渡す。ブートの最終段階で、カーネルはinitrdイメージの先頭数ブロックを読み込み、そのフォーマットを判断する

特に今回はフォーマットとして cpio を使用するので以下の流れになる。

> そのイメージがgzipで圧縮されたcpioアーカイブの場合、中間段階としてカーネルがそれを展開して initramfs （Linux 2.6.13 以降利用可能なtmpfsの特殊インスタンス）とし、それを初期化用ルートファイルシステムとする。

実際に必要な操作は以下の通り。

```
$ sed s/linux/linux-tiqwab/g < /etc/mkinitcpio.d/linux.preset > /etc/mkinitcpio.d/linux-tiqwab.preset
$ mkinitcpio -p linux-tiqwab
```

- `mkinitcpio` は initramfs CPIO イメージを作成するためのツール
- メインの設定ファイルは `/etc/mkinitcpio.conf`
  - 設定の詳細は [mkinitcpio - ArchWiki][12] を参照
- それに加えてカーネルパッケージ毎にプリセットと呼ばれる設定を追加するらしい
  - `/etc/mkinitcpio.d/`
- 既存のプリセットをコピーし、ファイル名やファイル内でのカーネル名にカスタムカーネルの名前を使うように変更する
- `mkinitcpio -p` でプリセットを指定
- 実行後プリセットの定義に基づいて `/boot` 下にイメージができるはず

プリセット例:

```
$ sudo cat /etc/mkinitcpio.d/linux-tiqwab.preset
# mkinitcpio preset file for the 'linux-tiqwab' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-tiqwab"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
default_image="/boot/initramfs-linux-tiqwab.img"
#default_options=""

#fallback_config="/etc/mkinitcpio.conf"
fallback_image="/boot/initramfs-linux-tiqwab-fallback.img"
fallback_options="-S autodetect"
```

`mkinitcpio` 後 の `/boot`:

```
$ ls -l /boot
total 81364
drwxr-xr-x 6 root root     4096 Jul 12 22:45 grub
-rw-r--r-- 1 root root 28943926 Jun 21 09:04 initramfs-linux-fallback.img
-rw-r--r-- 1 root root  7382653 Jun 21 09:04 initramfs-linux.img
-rw-r----- 1 root root 28936243 Jul 12 22:38 initramfs-linux-tiqwab-fallback.img
-rw-r----- 1 root root  7384434 Jul 12 22:38 initramfs-linux-tiqwab.img
-rw-r--r-- 1 root root  5326800 Jun 16 21:08 vmlinuz-linux
-rw-r----- 1 root root  5326800 Jul 12 22:35 vmlinuz-linux-tiqwab
```

<div id="boot-loader" />

### ブートローダの設定

- GRUB (GRand Unified Bootloader) を使用する。

[GRUB - ArchWiki][7] からの引用。

> ブートローダーはコンピューターが起動した時に最初に走るソフトウェアプログラムです。Linux カーネルのロードとコントロールの移譲を担当しています。そして、カーネルはオペレーションシステムの他全てを初期化します。

ブートまわりは知識が薄いのであとで整理するとして、やること自体は少ない。

```
$ grub-mkconfig -o /boot/grub/grub.cfg
```

これで `/boot` を見てエントリを追加してくれるよう。

<div id="run" />

### 動作確認

- リブートして自身のカーネルが使われるかを確認する
  - Vagrant だとブート時に OS の選択ができない
  - しかしリブートしたら自分でビルドしたのが選ばれている
  - grub のエントリの優先度の問題なんだろうけど、確認方法がいまいち？

カーネルバージョンのチェック:

```
$ uname -r
4.17.2-tiqwab
```

追加したシステムコール確認用のコード:

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscall.h>
#include <stdio.h>

# define SYS_hello 333

int main(int argc, char **argv)
{
    if (argc <= 1) {
        printf("Must provide a string to give to system call.\n");
        return -1;
    }
    char *arg = argv[1];
    printf("Making system call with \"%s\".\n", arg);
    long res = syscall(SYS_hello, arg);
    printf("System call returned %ld.\n", res);
    return res;
}
```

実行:

```
$ cc -o hello hello.c
$ ./hello 'world'
Making system call with "world".
System call returned 0.

$ dmesg | tail -n 1
[  370.007738] hello "world"
```

### 参考

- [チュートリアル - システムコールの書き方][8]
  - Linux カーネルをビルドする流れはほぼこの通りに行っている
- [カーネル/コンパイル/伝統的な方法 - ArchWiki][9]
  - 適宜補足的に確認
- [カーネル/コンパイル/Arch Build System - ArchWiki][10]
  - ArchLinux の場合はこちらの方法を使用することもできる
  - ただカーネルに手を加えたいだけならこちらの方が楽そう

[1]: https://wiki.archlinux.jp/index.php/Vagrant
[2]: https://github.com/terrywang/vagrantboxes/blob/master/archlinux-x86_64.md
[3]: http://norikakip.hatenadiary.jp/entry/2018/05/31/182349
[4]: https://unix.stackexchange.com/questions/439482/why-is-bc-required-to-build-the-linux-kernel
[5]: https://www.kernel.org/doc/html/latest/process/changes.html
[6]: https://mirrors.edge.kernel.org/pub/linux/kernel/
[7]: https://wiki.archlinux.jp/index.php/GRUB
[8]: https://postd.cc/kernel-dev-ep3/
[9]: https://wiki.archlinux.jp/index.php/%E3%82%AB%E3%83%BC%E3%83%8D%E3%83%AB/%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%AB/%E4%BC%9D%E7%B5%B1%E7%9A%84%E3%81%AA%E6%96%B9%E6%B3%95
[10]: https://wiki.archlinux.jp/index.php/%E3%82%AB%E3%83%BC%E3%83%8D%E3%83%AB/%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%AB/Arch_Build_System
[11]: https://ja.wikipedia.org/wiki/Initrd
[12]: https://wiki.archlinux.jp/index.php/Mkinitcpio
[13]: https://wiki.archlinux.jp/index.php/%E3%83%96%E3%83%BC%E3%83%88%E3%83%AD%E3%83%BC%E3%83%80%E3%83%BC
[14]: https://wiki.archlinux.jp/index.php/Unified_Extensible_Firmware_Interface
[15]: https://github.com/osdev-jp/osdev-jp.github.io/wiki/UEFI
[16]: http://majyuteikoku.com/pc/biosindex/bios1/
[17]: https://ja.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface
[18]: https://wiki.archlinux.jp/index.php/Systemd-boot
[19]: https://www.slideshare.net/syuu1228/uefi-boot-loaders
[20]: https://orumin.blogspot.com/2014/12/whats-uefi.html
[21]: https://wiki.archlinux.jp/index.php/QEMU
[22]: https://qiita.com/kairi199088/items/a2cd962f375cf875fe52
