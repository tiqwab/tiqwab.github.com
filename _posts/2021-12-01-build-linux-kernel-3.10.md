---
layout: post
title: Linux カーネル v3.10 のビルド
tags: "linux"
comments: true
---

ちまちま読み始めた『詳解Linuxカーネル 第3版』のおともに Linux カーネル v3.10 のビルド、実行環境を用意したのでその手順についてまとめます。本で扱う Linux カーネルは v2.6 なので一致していないのですが、v2.x はビルド環境を用意するのがかなり大変そうだったので、妥協して Cent OS 7 が使用する v3.10 を対象にしました。

Linux カーネルのビルド方法自体はブロク記事だけでも探せば多数見つかるのですが (自分でも [こちら][12] で以前書いたことがある)、今回ビルド対象の v3.10 と用意した環境の兼ね合いのせいか一部ソースを修正する必要があったのでメモにしました。

### Linux カーネルのビルド

ホスト側に Linux カーネルのソースを用意し、ビルドは CentOS 7 VM 上で行います。

#### ソースの用意

```
$ git clone --depth=1 -b v3.10 https://github.com/torvalds/linux.git linux-3.10
$ cd linux-3.10
```

#### VM の用意

VM は Vagrant (with VirtualBox provider) で用意します。
ここでは上で用意した linux-3.10 ディレクトリ直下に Vagrantfile を配置します。
カーネルのソースコードを VM と共有するようにしています。

```
$ vagrant init
$ vim Vagrantfile
$ cat Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"

  # Share source codes with VM
  config.vm.synced_folder ".", "/works"

  config.vbguest.installer_options = {
    allow_kernel_upgrade: true
  }

  config.vm.provider "virtualbox" do |vb|
    # Trun on when booting VM with custom kernel (to show grub menu on boot)
    # vb.gui = true

    vb.cpus = 4
  
    vb.memory = "4096"
  end
end

$ vagrant up
$ vagrant ssh
```

#### ビルド環境の用意

カーネルビルドに必要なパッケージ群はカーネルソースコード内の `Documentation/Changes` か `Documentation/process/changes.rst` に記載されています。

CentOS 7 の場合は以下のコマンドで一通り必要なものを用意することができます。

```
$ sudo yum groupinstall "Development Tools"
$ sudo yum install bc ncurses-devel

# rootfs 作成時に使用
$ sudo yum install glibc-static
```

#### .config の用意

カーネルビルドでは様々なオプションを設定することができ、その設定はルート直下の `.config` で管理されます。

オプションは大量にあるので一から自作するよりは、既に動作している Linux 環境から持ってくるか、 `make defconfig` 等によりデフォルト設定を利用することが多いかと思います。動作しているシステムから持ってくる場合は、`/boot` 下に参照用に配置されているものか、`/proc/config.gz` を利用します。

今回は `make defconfig` したものをベースに `make menuconfig` で必要な設定を変更しました。
とはいっても生成された設定を見るとひとまずのデバッグ用途ではそのまま使用できそうだったので、カーネルバージョンの suffix だけの修正になりました。

変更した設定:

- `CONFIG_LOCALVERSION`: `-custom`

#### ビルド

カーネルをビルドします。

```
# `-j` オプションはジョブの並列実行数
$ make -j 2
```

完了すれば `arch/x86/boot/bzImage` にカーネルイメージが配置されます。

### 必要なソースの修正

これは v3.10 かつ今回用意した環境独自の問題かもしれませんが、落としてきたソースそのままからビルドしたカーネルだと起動時に「Booting the kernel.」出力後に停止してしまうので修正を加えます。

停止する原因は `early_idt_handlers` というブート時に一時的に使用する割り込みハンドラをまとめたテーブル内のデータが途中からずれることでした。各ハンドラは vector number や必要ならば error code 代わりの 0 を詰めたのちに共通処理へと jmp するという内容になっています。ブート処理中ではこのハンドラが 9 bytes であることを期待しているのですが、機械語を生成する際にハンドラによっては jmp 命令の offset 値が 4 bytes から 1 byte になってしまうことでズレが生じていました (恐らく vector number が大きいものについては offset 値が小さくなり、アセンブラがよきにはからうためかと)。

Linux カーネルのコミットを辿ると [こちら][7] で対処がされていたので、同様の方針でひとまず各ハンドラが 9 bytes となるように適当なバイトで埋めて対応しています。

```
# in arch/x86/kernel/head_64.S
        __INIT
        .globl early_idt_handlers
early_idt_handlers:
        # 104(%rsp) %rflags
        #  96(%rsp) %cs
        #  88(%rsp) %rip
        #  80(%rsp) error code
        i = 0
        .rept NUM_EXCEPTION_VECTORS
        .if (EXCEPTION_ERRCODE_MASK >> i) & 1
        ASM_NOP2
        .else
        pushq $0                # Dummy error code, to make stack frame uniform
        .endif
        pushq $i                # 72(%rsp) Vector number
        jmp early_idt_handler
        i = i + 1

        # この .fill ディレクティブが追加したもの
        # 各ハンドラが 9 bytes になるように 0xcc を埋める。
        .fill early_idt_handlers + i*9 - ., 1, 0xcc
        .endr
```

### ルートファイルシステムの用意

カーネルの正常起動を確認するため最低限のファイルシステムを [こちら][6] や [こちら][11] を元に用意します。

```
# VM 内での作成
# test.c は init として起動したい自作プログラム
# ここでは 1 秒ごとに printf("Hello") するだけ
$ gcc -static -o init test.c
$ echo init | cpio -o -H newc | gzip > initramfs.cpio.gz
```

### QEMU での実行

`bzImage` と `initramfs.cpio.gz` にはそれぞれ上記の手順内で用意したカーネルイメージとルートファイルシステムを指定します。

```
$ qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -initrd rootfs/initramfs.cpio.gz \
    -append "root=/dev/ram console=ttyS0 earlyprintk=ttyS0 nokaslr debug" \
    -serial stdio
```

また GDB デバッグを行うときは起動オプションに `-s -S` も追加して、別ターミナルから GDB でリモート接続します。

```
$ qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -initrd rootfs/initramfs.cpio.gz \
    -append "root=/dev/ram console=ttyS0 earlyprintk=ttyS0 nokaslr debug" \
    -serial stdio \
    -s \
    -S

# 別画面で
$ gdb vmlinux
...
Reading symbols from vmlinux...
(No debugging symbols found in vmlinux)

(gdb) target remote localhost:1234
Remote debugging using localhost:1234
0x000000000000fff0 in init_tss ()

# 例えばブート処理中の適当な関数にブレイクポイントを設定
(gdb) b x86_64_start_kernel
Breakpoint 1 at 0xffffffff81cd6573

(gdb) c
Continuing.

Breakpoint 1, 0xffffffff81cd6573 in x86_64_start_kernel ()
(gdb)
```

デバッグについては GDB よりも [QEMU monitor][8] の方が有用なこともあります。
例えば QEMU monitor では物理メモリアドレス指定でデータを確認することができます。

### 参考

- Linux ブートプロセスについて
  - 『詳解Linuxカーネル 第3版』 付録A システムの起動
  - [Linux Inside][3] の Booting, Initialization 項
  - [vmlinuxのヒミツ - valinux.hatenablog.com][4]
- Linux カーネルビルドについて
  - [Build a minimal Linux system and run it in QEMU - ibug.io][1]
  - [Linux カーネルを QEMU 上で起動する - kuniyu.jp][5]
- Linux カーネルデバッグについて
  - [Debugging kernel and modules via gdb - www.kernel.org][9]
  - [JOS から学ぶ GDB デバッグ手法][10]
  - [FreeDOS での開発にまつわる etc. - blog.tiqwab.com][8] の QEMU monitor 項

[1]: https://ibug.io/blog/2019/04/os-lab-1/
[2]: https://qiita.com/syo0901/items/3e03222bf4e79d22ccd1
[3]: https://0xax.gitbooks.io/linux-insides/content/
[4]: https://valinux.hatenablog.com/entry/20200910
[5]: https://kuniyu.jp/ja/blog/2/
[6]: https://ibug.io/blog/2019/04/os-lab-1/#preparing-the-root-filesystem---option-1-handcraft-init
[7]: https://github.com/torvalds/linux/commit/cdeb6048940fa4bfb429e2f1cba0d28a11e20cd5
[8]: https://blog.tiqwab.com/2018/10/21/freedos-dev.html#qemu-monitor
[9]: https://www.kernel.org/doc/html/latest/dev-tools/gdb-kernel-debugging.html
[10]: https://blog.tiqwab.com/2019/11/10/qemu-gdb.html
[11]: https://qiita.com/kaishuu0123/items/33621af2aca44d8d2c72#init-%E3%81%A7%E5%AE%9F%E8%A1%8C%E3%81%99%E3%82%8B%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%A0%E3%81%AE%E4%BD%9C%E6%88%90
[12]: https://blog.tiqwab.com/2018/07/21/build-linux.html
