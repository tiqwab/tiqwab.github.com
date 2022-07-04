---
layout: post
title: Snap がどのようにサンドボックスを用意するか
tags: "apparmor,seccomp,cgroup,snap"
comments: true
---

- [Snap とは](#what-is-snap)
- [自作 Snap パッケージの用意](#sample-snap)
- [Mount namespace](#mount-namespace)
- [Seccomp](#seccomp)
- [AppArmor](#apparmor)
- [Device cgroup](#device-cgroup)
- [自作 Snap パッケージで ping を許可する](#sample-snap-with-ping)

<div id="what-is-snap" />

### Snap とは

Snap は各種 Linux ディストリビューション、macOS 向けにいわゆるユニバーサルパッケージを提供するシステムです。パッケージ内に必要な依存関係がすべて含まれており、ホストとは独立したサンドボックス環境でアプリケーションを実行することができます。

同様の機能を持つパッケージシステムとして Flatpak もありますが、Snap は Canonical 社製ということもあり Ubuntu 関連で特に耳にする気がします。最近では Ubuntu 22.04 LTS がリリースされた際に [Firefox パッケージが Snap に移行した][10] というニュースが話題になっていました。

前々から Linux でサンドボックスを作成するのによく用いられる AppArmor や Seccomp といった技術に触れるきっかけがほしいと思っていたので、Snap を題材にどのようにアプリケーションの実行がされているのかを整理してみました。

以後のプログラムやコマンド実行は Ubuntu 22.04 LTS で行っています。

<div id="sample-snap" />

### 自作 Snap パッケージの用意

まず Snap パッケージがどのようなものなのか、また気軽に動かして触れる環境を用意するために、単に bash を実行するだけの自作パッケージを用意してみます。

パッケージの作成には公式で提供される Snapcraft を利用します。Snapcraft 自体も Snap パッケージとして提供されています。

```
$ sudo snap install snapcraft --classic
```

インストール後、`snapcraft init` によりパッケージの初期化を行います。

```
$ mkdir sample-snap && cd !$

$ snapcraft init
Created snap/snapcraft.yaml.
Go to https://docs.snapcraft.io/the-snapcraft-format/8337 for more information about the snapcraft.yaml format.

$ tree .
.
└── snap
    └── snapcraft.yaml

1 directory, 1 file
```

上記の出力で表示されている `snapcraft.yaml` は Snap パッケージのビルド定義であり、これがパッケージに最低限必要なファイルになります。今回は bash を実行するだけのパッケージなので内容は以下のように編集しました。

```yaml
name: sample-snap
base: core20
version: '0.1'
summary: Execute bash
description: |
  Execute bash in Sandbox

grade: devel # must be 'stable' to release into candidate/stable channels
confinement: devmode # use 'strict' once you have the right plugs and slots

# apps 以下で実行したいコマンドを指定する
apps:
  bash:
    command: bin/main.sh

# parts で環境に必要なものを用意する
# ここでは 予め用意した bash を実行するだけのシェルスクリプトを配置している
parts:
  bash:
    plugin: dump
    source: .
    organize:
      main.sh: bin/
```

ビルドは以下のコマンドで行います。
自分は VM 内に作業環境を用意していたからというのもあるでしょうが完了まで 15 分程度かかりました。

```bash
$ snapcraft --debug
```

ビルド完了するとカレントディレクトリに `sample-snap_0.1_amd64.snap` というような squashfs ファイルイメージが生成されるので、これを Snap でインストールします。

```bash
$ file sample-snap_0.1_amd64.snap
sample-snap_0.1_amd64.snap: Squashfs filesystem, little endian, version 4.0, xz compressed, 983 bytes, 8 inodes, blocksize: 131072 bytes, created: Fri Jul  1 06:49:54 2022

# アクセス制御を厳密に行うためにビルド時に設定した devmode ではなく jailmode で実行する
# devmode ではログのみが記録される
$ sudo snap install sample-snap_0.1_amd64.snap --dangerous --jailmode
```

インストールしたパッケージが提供するコマンドは `<package_name>.<app_name>` のようにして実行できます。
これは実際には `/usr/bin/snap` へのシンボリックリンクになっており、snap 側でそれを検知して適切なコマンドを隔離した環境で実行します。

```bash
$ which sample-snap.bash
/snap/bin/sample-snap.bash

$ ls -l /snap/bin/sample-snap.bash
lrwxrwxrwx 1 root root 13 Jul  1 07:42 /snap/bin/sample-snap.bash -> /usr/bin/snap

$ sample-snap.bash

$ echo hello
hello
```

デフォルトの環境はできることがかなり制限されており、例えば外部との通信も許可されていません。

```bash
$ ping -c 3 8.8.8.8
bash: /usr/bin/ping: Permission denied
```

Snap パッケージの動作が確認できたので、ここから先は [Security policy and sandboxing][11] を参考にしながらアプリケーション用の隔離環境がどのように用意されているのかを見ていきます。

<div id="mount-namespace" />

### Mount namespace

先程の自作パッケージで実行した bash が所属する namespace を確認すると mount だけホストとは異なる namespace が作成されていることがわかります。

```bash
# bash の namespaces
$ ls -l /proc/18647/ns
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 mnt -> 'mnt:[4026532193]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 net -> 'net:[4026531840]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 time -> 'time:[4026531834]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 user -> 'user:[4026531837]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 02:01 uts -> 'uts:[4026531838]'

# ホストの namespaces
$ ls -l /proc/$$/ns 
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 net -> 'net:[4026531840]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 time -> 'time:[4026531834]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 user -> 'user:[4026531837]'
lrwxrwxrwx 1 vagrant vagrant 0 Jul  1 01:59 uts -> 'uts:[4026531838]'
```

隔離環境での mountinfo をホストから確認してみます。

```bash
$ cat /proc/9619/mountinfo
310 304 7:2 / / ro,nodev,relatime master:47 - squashfs /dev/loop2 ro,errors=continue
311 310 0:5 / /dev rw,nosuid,relatime master:2 - devtmpfs udev rw,size=1989976k,nr_inodes=497494,mode=755,inode64
312 311 0:23 / /dev/pts rw,nosuid,noexec,relatime master:3 - devpts devpts rw,gid=5,mode=620,ptmxmode=000
313 312 0:40 / /dev/pts rw,relatime - devpts devpts rw,gid=5,mode=620,ptmxmode=666
314 311 0:26 / /dev/shm rw,nosuid,nodev master:4 - tmpfs tmpfs rw,inode64
315 311 0:32 / /dev/hugepages rw,relatime master:14 - hugetlbfs hugetlbfs rw,pagesize=2M
316 311 0:19 / /dev/mqueue rw,nosuid,nodev,noexec,relatime master:15 - mqueue mqueue rw
317 311 0:40 /ptmx /dev/ptmx rw,relatime - devpts devpts rw,gid=5,mode=620,ptmxmode=666
318 310 8:1 /etc /etc rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
319 318 7:2 /etc/nsswitch.conf /etc/nsswitch.conf ro,nodev,relatime master:47 - squashfs /dev/loop2 ro,errors=continue
320 318 7:2 /etc/apparmor /etc/apparmor ro,nodev,relatime master:47 - squashfs /dev/loop2 ro,errors=continue
336 318 7:2 /etc/apparmor.d /etc/apparmor.d ro,nodev,relatime master:47 - squashfs /dev/loop2 ro,errors=continue
337 310 8:1 /home /home rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
338 310 8:1 /root /root rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
339 310 0:22 / /proc rw,nosuid,nodev,noexec,relatime master:12 - proc proc rw
340 339 0:31 / /proc/sys/fs/binfmt_misc rw,relatime master:13 - autofs systemd-1 rw,fd=29,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=15599
341 310 0:21 / /sys rw,nosuid,nodev,noexec,relatime master:7 - sysfs sysfs rw
342 341 0:6 / /sys/kernel/security rw,nosuid,nodev,noexec,relatime master:8 - securityfs securityfs rw
343 341 0:28 / /sys/fs/cgroup rw,nosuid,nodev,noexec,relatime master:9 - cgroup2 cgroup2 rw,nsdelegate,memory_recursiveprot
344 341 0:29 / /sys/fs/pstore rw,nosuid,nodev,noexec,relatime master:10 - pstore pstore rw
345 341 0:30 / /sys/fs/bpf rw,nosuid,nodev,noexec,relatime master:11 - bpf bpf rw,mode=700
346 341 0:7 / /sys/kernel/debug rw,nosuid,nodev,noexec,relatime master:16 - debugfs debugfs rw
348 341 0:12 / /sys/kernel/tracing rw,nosuid,nodev,noexec,relatime master:17 - tracefs tracefs rw
349 341 0:33 / /sys/fs/fuse/connections rw,nosuid,nodev,noexec,relatime master:18 - fusectl fusectl rw
356 341 0:20 / /sys/kernel/config rw,nosuid,nodev,noexec,relatime master:19 - configfs configfs rw
357 310 8:1 /tmp /tmp rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
358 357 8:1 /tmp/snap.sample-snap/tmp /tmp rw,relatime - ext4 /dev/sda1 rw,discard,errors=remount-ro
359 310 8:1 /var/snap /var/snap rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
360 310 8:1 /var/lib/snapd /var/lib/snapd rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
590 360 8:1 /var/lib/snapd/hostfs /var/lib/snapd/hostfs rw,relatime - ext4 /dev/sda1 rw,discard,errors=remount-ro
591 590 8:1 / /var/lib/snapd/hostfs rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1126 591 0:24 / /var/lib/snapd/hostfs/run rw,nosuid,nodev,noexec,relatime master:5 - tmpfs tmpfs rw,size=401996k,mode=755,inode64
1127 1126 0:27 / /var/lib/snapd/hostfs/run/lock rw,nosuid,nodev,noexec,relatime master:6 - tmpfs tmpfs rw,size=5120k,inode64
1128 1126 0:34 / /var/lib/snapd/hostfs/run/credentials/systemd-sysusers.service ro,nosuid,nodev,noexec,relatime master:20 - ramfs none rw,mode=700
1129 1126 0:24 /snapd/ns /var/lib/snapd/hostfs/run/snapd/ns rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,size=401996k,mode=755,inode64
1130 1126 0:45 / /var/lib/snapd/hostfs/run/user/1000 rw,nosuid,nodev,relatime master:340 - tmpfs tmpfs rw,size=401992k,nr_inodes=100498,mode=700,uid=1000,gid=1000,inode64
1131 591 7:0 / /var/lib/snapd/hostfs/snap/bpftrace/160 ro,nodev,relatime master:30 - squashfs /dev/loop0 ro,errors=continue
1132 591 7:1 / /var/lib/snapd/hostfs/snap/core20/1494 ro,nodev,relatime master:45 - squashfs /dev/loop1 ro,errors=continue
1133 591 7:2 / /var/lib/snapd/hostfs/snap/core20/1518 ro,nodev,relatime master:47 - squashfs /dev/loop2 ro,errors=continue
1134 591 7:5 / /var/lib/snapd/hostfs/snap/my-snap-name/x1 ro,nodev,relatime master:49 - squashfs /dev/loop5 ro,errors=continue
1135 591 7:4 / /var/lib/snapd/hostfs/snap/multipass/7416 ro,nodev,relatime master:51 - squashfs /dev/loop4 ro,errors=continue
1136 591 7:6 / /var/lib/snapd/hostfs/snap/snapcraft/7717 ro,nodev,relatime master:53 - squashfs /dev/loop6 ro,errors=continue
1137 591 7:3 / /var/lib/snapd/hostfs/snap/multipass/7372 ro,nodev,relatime master:55 - squashfs /dev/loop3 ro,errors=continue
1138 591 7:10 / /var/lib/snapd/hostfs/snap/lxd/22923 ro,nodev,relatime master:57 - squashfs /dev/loop10 ro,errors=continue
1139 591 7:7 / /var/lib/snapd/hostfs/snap/vault/2012 ro,nodev,relatime master:59 - squashfs /dev/loop7 ro,errors=continue
1140 591 7:9 / /var/lib/snapd/hostfs/snap/snapd/15904 ro,nodev,relatime master:61 - squashfs /dev/loop9 ro,errors=continue
1141 591 7:11 / /var/lib/snapd/hostfs/snap/snapd/16010 ro,nodev,relatime master:63 - squashfs /dev/loop11 ro,errors=continue
1142 591 7:8 / /var/lib/snapd/hostfs/snap/core18/2409 ro,nodev,relatime master:65 - squashfs /dev/loop8 ro,errors=continue
1143 591 0:38 / /var/lib/snapd/hostfs/vagrant rw,nodev,relatime master:109 - vboxsf vagrant rw,iocharset=utf8,uid=1000,gid=1000
1144 1143 0:46 / /var/lib/snapd/hostfs/vagrant rw,nodev,relatime master:348 - vboxsf vagrant rw,iocharset=utf8,uid=1000,gid=1000
1145 591 7:12 / /var/lib/snapd/hostfs/snap/sample-snap/x1 ro,nodev,relatime master:168 - squashfs /dev/loop12 ro,errors=continue
1146 591 7:13 / /var/lib/snapd/hostfs/snap/sample-snap/x2 ro,nodev,relatime master:200 - squashfs /dev/loop13 ro,errors=continue
1147 310 8:1 /var/tmp /var/tmp rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1148 310 0:24 / /run rw,nosuid,nodev,noexec,relatime master:5 - tmpfs tmpfs rw,size=401996k,mode=755,inode64
1149 1148 0:27 / /run/lock rw,nosuid,nodev,noexec,relatime master:6 - tmpfs tmpfs rw,size=5120k,inode64
1150 1148 0:34 / /run/credentials/systemd-sysusers.service ro,nosuid,nodev,noexec,relatime master:20 - ramfs none rw,mode=700
1151 1148 0:24 /snapd/ns /run/snapd/ns rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,size=401996k,mode=755,inode64
1152 1148 0:45 / /run/user/1000 rw,nosuid,nodev,relatime master:340 - tmpfs tmpfs rw,size=401992k,nr_inodes=100498,mode=700,uid=1000,gid=1000,inode64
1153 1148 0:24 /netns /run/netns rw,nosuid,nodev,noexec,relatime shared:5 - tmpfs tmpfs rw,size=401996k,mode=755,inode64
1154 310 8:1 /usr/lib/modules /usr/lib/modules rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1155 310 8:1 /usr/src /usr/src rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1156 310 8:1 /var/log /var/log rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1157 310 8:1 /media /media rw,relatime shared:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1158 310 8:1 /mnt /mnt rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1159 310 7:11 /usr/lib/snapd /usr/lib/snapd ro,nodev,relatime master:63 - squashfs /dev/loop11 ro,errors=continue
1160 310 8:1 /snap /snap rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1161 1160 7:0 / /snap/bpftrace/160 ro,nodev,relatime master:30 - squashfs /dev/loop0 ro,errors=continue
1162 1160 7:1 / /snap/core20/1494 ro,nodev,relatime master:45 - squashfs /dev/loop1 ro,errors=continue
1163 1160 7:2 / /snap/core20/1518 ro,nodev,relatime master:47 - squashfs /dev/loop2 ro,errors=continue
1164 1160 7:5 / /snap/my-snap-name/x1 ro,nodev,relatime master:49 - squashfs /dev/loop5 ro,errors=continue
1165 1160 7:4 / /snap/multipass/7416 ro,nodev,relatime master:51 - squashfs /dev/loop4 ro,errors=continue
1166 1160 7:6 / /snap/snapcraft/7717 ro,nodev,relatime master:53 - squashfs /dev/loop6 ro,errors=continue
1167 1160 7:3 / /snap/multipass/7372 ro,nodev,relatime master:55 - squashfs /dev/loop3 ro,errors=continue
1168 1160 7:10 / /snap/lxd/22923 ro,nodev,relatime master:57 - squashfs /dev/loop10 ro,errors=continue
1169 1160 7:7 / /snap/vault/2012 ro,nodev,relatime master:59 - squashfs /dev/loop7 ro,errors=continue
1170 1160 7:9 / /snap/snapd/15904 ro,nodev,relatime master:61 - squashfs /dev/loop9 ro,errors=continue
1171 1160 7:11 / /snap/snapd/16010 ro,nodev,relatime master:63 - squashfs /dev/loop11 ro,errors=continue
1172 1160 7:8 / /snap/core18/2409 ro,nodev,relatime master:65 - squashfs /dev/loop8 ro,errors=continue
1173 1160 7:12 / /snap/sample-snap/x1 ro,nodev,relatime master:168 - squashfs /dev/loop12 ro,errors=continue
1174 1160 7:13 / /snap/sample-snap/x2 ro,nodev,relatime master:200 - squashfs /dev/loop13 ro,errors=continue
```

mountinfo 全体はかなり長いのでいくつか Snap 特有であろう点に着目してみます。

#### base イメージ

まずはベースとなるファイルシステムイメージについてです。Snap でアプリケーションを実行する際にルートファイルシステムとして [base snaps][3] と呼ばれる特別な snap が利用されます。例えば先程自作した Snap パッケージでは `snapcraft.yaml` で `base: core20` のように指定することで Ubuntu 20.04 LTS をベースにした環境を用意しています。

```
# mountinfo からの抜粋
310 304 7:2 / / ro,nodev,relatime master:47 - squashfs /dev/loop2 ro,errors=continue
1163 1160 7:2 / /snap/core20/1518 ro,nodev,relatime master:47 - squashfs /dev/loop2 ro,errors=continue
```

mountinfo を見るとベースイメージは squashfs ファイルイメージであり、ループバックデバイスとしてマウントされていることがわかります。このうち後者のマウントについてはパッケージインストール時に systemd の mount ユニットを作成して永続化しており、それをアプリケーション実行時に適切な場所にマウントしています。

```
$ cat /etc/systemd/system/snap-core20-1518.mount
[Unit]
Description=Mount unit for core20, revision 1518
Before=snapd.service
After=zfs-mount.service

[Mount]
What=/var/lib/snapd/snaps/core20_1518.snap
Where=/snap/core20/1518
Type=squashfs
Options=nodev,ro,x-gdu.hide,x-gvfs-hide
LazyUnmount=yes

[Install]
WantedBy=multi-user.target
```

オリジナルのファイルイメージは `/var/lib/snapd/snaps/` に存在しています。

```
$ sudo file /var/lib/snapd/snaps/core20_1518.snap
/var/lib/snapd/snaps/core20_1518.snap: Squashfs filesystem, little endian, version 4.0, xz compressed, 64933075 bytes, 11789 inodes, blocksize: 131072 bytes, created: Fri May 27 06:47:13 2022
```

#### パッケージ特有の環境

自作 Snap ビルド時に確認したように、パッケージの実態は squashfs ファイルイメージです。これは base イメージと同様、インストール時に `/var/lib/snapd/snaps/` 以下に配置され、`/snap/<snap_name>` 下にループバックマウントされます。アプリケーション実行時にはそれが利用できるように隔離環境にマウントされます。

```
360 310 8:1 /var/lib/snapd /var/lib/snapd rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1160 310 8:1 /snap /snap rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
1174 1160 7:13 / /snap/sample-snap/x2 ro,nodev,relatime master:200 - squashfs /dev/loop13 ro,errors=continue
```

Snap パッケージの展開場所はアプリケーションからは `SNAP` 環境変数で確認できます。

```bash
# サンドボックス内の bash から
$ env | grep SNAP
...
SNAP=/snap/sample-snap/x2
...
```

また Snap パッケージで [layout][15] 設定を追加しているとそれに対応したマウントも行われます。layout は主にパッケージ内で使用するコマンドやライブラリが何らか固定されたパスに依存している (ベースのパスを環境変数やパラメータとして取得する仕組みになっていない) 場合の回避策として利用できる仕組みです。

例えば multipass というアプリケーションでは `snapcraft.yaml` に以下のような layout が設定されています。

```yaml
layout:
  /usr/lib/x86_64-linux-gnu/qemu:
    bind: $SNAP/usr/lib/x86_64-linux-gnu/qemu
  /usr/share/X11:
    bind: $SNAP/usr/share/X11
  /etc/fonts:
    bind: $SNAP/etc/fonts
  /usr/share/fonts:
    bind: $SNAP/usr/share/fonts
  /usr/share/icons:
    bind: $SNAP/usr/share/icons
```

実際に multipass プロセスの mountinfo で関連していそうなものを見てみます。

```
# ここでは /etc/fonts についてだけ抜粋
#
# 見方としては以下でいいはず...
# - マウントID: 803 がマウントID: 100 の MS_SLAVE マウントになっている
# - マウントID: 622 ではマウント元のパス /etc/fonts を /etc/fonts にバインドマウントしている
#   - マウント元、というのはここでいうマウントID: 803 と考えて良いはず。ひいてはマウントID: 100 

# multipass プロセス
$ cat /proc/695/mountinfo
...
803 798 7:4 / /snap/multipass/7416 ro,nodev,relatime master:51 - squashfs /dev/loop4 ro,errors=continue
622 673 7:4 /etc/fonts /etc/fonts ro,nodev,relatime master:51 - squashfs /dev/loop4 ro,errors=continue
...

# ホスト
$ cat /proc/$$/mountinfo
...
100 29 7:4 / /snap/multipass/7416 ro,nodev,relatime shared:51 - squashfs /dev/loop4 ro,errors=continue
...
```

この情報は `/var/lib/snapd/mount/*.fstab` として Snap 内では管理されているようです。
上の mountinfo よりはこちらを見るほうがわかりやすいですね。

```bash
$ cat /var/lib/snapd/mount/snap.multipass.fstab
/snap/multipass/7416/etc/fonts /etc/fonts none rbind,rw,x-snapd.origin=layout 0 0
/snap/multipass/7416/usr/lib/x86_64-linux-gnu/qemu /usr/lib/x86_64-linux-gnu/qemu none rbind,rw,x-snapd.origin=layout 0 0
/snap/multipass/7416/usr/share/X11 /usr/share/X11 none rbind,rw,x-snapd.origin=layout 0 0
/snap/multipass/7416/usr/share/fonts /usr/share/fonts none rbind,rw,x-snapd.origin=layout 0 0
/snap/multipass/7416/usr/share/icons /usr/share/icons none rbind,rw,x-snapd.origin=layout 0 0
/var/lib/snapd/hostfs/var/lib/dhcp /var/lib/dhcp none bind,rw,x-snapd.ignore-missing 0 0
/var/lib/snapd/hostfs/tmp/.X11-unix /tmp/.X11-unix none bind,ro 0 0
```

#### パッケージのデータ置き場

先述のパッケージから展開されたディレクトリは read-only でマウントされるため、アプリケーションで永続化された状態を管理したい場合はデータ用に用意されたディレクトリを利用する必要があります。アプリケーションに対しては `SNAP_DATA` や `SNAP_USER_DATA` といった環境変数でそのパスが渡されます (アプリケーションに渡される環境変数については [Environment variables][4] を参照)。

```bash
# サンドボックス内の bash から
$ env | grep -e '^SNAP_.*DATA.*$'
SNAP_USER_DATA=/home/vagrant/snap/sample-snap/x2
SNAP_DATA=/var/snap/sample-snap/x2
```

これらのディレクトリを隔離環境で見せるためのマウントが設定されています。

```
359 310 8:1 /var/snap /var/snap rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
337 310 8:1 /home /home rw,relatime master:1 - ext4 /dev/sda1 rw,discard,errors=remount-ro
```

#### ホストのルートファイルシステムの再配置

サンドボックス内では元のルートファイルシステムが `/var/lib/snapd/hostfs` として配置されていることがわかります。

```
590 360 8:1 /var/lib/snapd/hostfs /var/lib/snapd/hostfs rw,relatime - ext4 /dev/sda1 rw,discard,errors=remount-ro
```

ただこれはアプリケーションのマウント設定の中で `pivot_root` を行うときの `put_old` パラメータのために用意された箇所というだけでアプリケーションが利用することは想定されていないようです。環境内では AppArmor の設定でこのディレクトリへのアクセスは制限されています。

hostfs についての詳細は [フォーラムの質問][5] で回答がされていました。

<div id="seccomp" />

### Seccomp

Snap ではサンドボックス内で発行できるシステムコールを制限するために Seccomp を利用しています。Seccomp が動作していることは `/proc/<pid>/status` から確認できます。

```bash
# 隔離環境内の bash から
# Seccomp: 2 が filter mode で動作していることを示す
$ cat /proc/$$/status | grep -i seccomp
Seccomp:        2
Seccomp_filters:        2
```

一般的に動作中のプロセスに設定されている Seccomp ルールを確認する術があるのかはわからないですが、Snap に関していうと `/var/lib/snapd/seccomp/bpf/snap` 以下を見ることで元になる設定とそこから生成された bpf が確認できます。内容を見ると特別に設定を加えていないアプリケーションでは Snap ソースコード上の [interfaces/seccomp/template.go][12] 内の `defaultTemplate` や `basePrivDropSyscalls` あたりの内容が使われるようです。

```bash
# src ファイルが許可されるシステムコールを羅列したもの
# bin ファイルがそれから生成した BPF コード
$ ls -l /var/lib/snapd/seccomp/bpf/snap.sample-snap.bash.*
-rw-r--r-- 1 root root 6168 Jul  1 07:07 /var/lib/snapd/seccomp/bpf/snap.sample-snap.bash.bin
-rw-r--r-- 1 root root 9790 Jul  1 07:07 /var/lib/snapd/seccomp/bpf/snap.sample-snap.bash.src
```

<div id="apparmor" />

### AppArmor

Snap ではアプリケーションのファイルアクセスを AppArmor により制御しています。適用されるルールは `/var/lib/snapd/apparmor/profiles` 以下で管理されており、Snap ソースコード上では [interfaces/apparmor][13] 下でテンプレート定義やルール生成の処理が行われています。

```bash
$ ls -l /var/lib/snapd/apparmor/profiles/snap.sample-snap.bash
-rw-r--r-- 1 root root 23063 Jul  1 07:07 /var/lib/snapd/apparmor/profiles/snap.sample-snap.bash
```

ルールの中身を見ると、例えば `/home` 以下のアクセスについて制限はかけつつデータ用のパスだったりのアクセスを許可している様子がわかります。

```
  # Allow read-access on /home/ for navigating to other parts of the
  # filesystem. While this allows enumerating users, this is already allowed
  # via /etc/passwd and getent.
  @{HOMEDIRS}/ r,

  # Read-only home area for other versions
  # bind mount *not* used here (see 'parallel installs', above)
  owner @{HOME}/snap/@{SNAP_INSTANCE_NAME}/                  r,
  owner @{HOME}/snap/@{SNAP_INSTANCE_NAME}/**                mrkix,

  # Writable home area for this version.
  # bind mount *not* used here (see 'parallel installs', above)
  owner @{HOME}/snap/@{SNAP_INSTANCE_NAME}/@{SNAP_REVISION}/** wl,
  owner @{HOME}/snap/@{SNAP_INSTANCE_NAME}/common/** wl,
```

```bash
# サンドボックス内の bash から
$ ls /home
ubuntu  vagrant

$ ls /home/vagrant
ls: cannot open directory '/home/vagrant': Permission denied

$ ls /home/vagrant/snap/sample-snap/
common  current  x1

$ touch /home/vagrant/snap/sample-snap/current/foo.txt

$ ls -l /home/vagrant/snap/sample-snap/current/
total 0
-rw-rw-r-- 1 vagrant vagrant 0 Jul  1 07:48 foo.txt
```

<div id="device-cgroup" />

### Device cgroup

アプリケーションのデバイスアクセスの制御として通常 AppArmor によるファイルパスをベースにした制御が行われていますが、どうやらそれでは不十分な場合には Device cgroup による制御も設定されるようです。なお公式ドキュメントではデバイスの制御について Device cgroup と総称されていますが、実際は cgroup v2 では device controller は消えているので代わりに BPF による制御が行われています (参考: [Linux kernel doc の cgroup-v2.txt][6] の Device controller 項)。

このあたりは実際に動作するアプリケーションで実行の様子を試したりがうまく行かなかったので、ソースコード上での理解だけになりますが、Snap ソースコード上 [cmd/snap-confine/udev-support.c][14] 内の `sc_setup_device_cgroup` 関数を中心に見ていくと、Device cgroup による制御ルールは udev ルールからタグで対象を絞り込んだものを利用しているようです。

Snap では [interface][7] によりサンドボックス外のリソースへのアクセスを提供する仕組みがあり、そこで設定されるルールの一つに udev があります。例えば Snap でインストールした multipass で設定された udev には以下のように kvm や network-control という interface 由来のルールがありました。最後のルールではアクセスを許可したいデバイスにタグを付与した上で、`snap-device-helper` を実行しているのだと推測できます。ソースコードを読むと `snap-device-helper` で Device cgroup へのデバイス追加を行っているようです。

```bash
$ cat /etc/udev/rules.d/70-snap.multipass.rules
# This file is automatically generated.
# kvm
KERNEL=="kvm", TAG+="snap_multipass_multipassd"
# network-control
KERNEL=="rfkill", TAG+="snap_multipass_multipassd"
# network-control
KERNEL=="tun", TAG+="snap_multipass_multipassd"
TAG=="snap_multipass_multipassd", RUN+="/usr/lib/snapd/snap-device-helper $env{ACTION} snap_multipass_multipassd $devpath $major:$minor"
```

<div id="sample-snap-with-ping" />

### 自作 Snap パッケージで ping を許可する

はじめに自作したパッケージでは ping 実行時に以下のパーミッションエラーが発生しました。

```bash
$ ping -c 3 8.8.8.8
bash: /usr/bin/ping: Permission denied
```

ここまで見てきたようにアプリケーションが実行されるサンドボックスでは Seccomp や AppArmor が動作しておりデフォルトの設定では ping 実行に必要な機能が制限されています。ログを見ると実際 AppArmor による制限を受けていることがわかります。

```bash
$ sudo journalctl
...
Jul 02 01:23:03 ubuntu-jammy audit[6013]: AVC apparmor="DENIED" operation="exec" profile="snap.sample-snap.bash" name="/usr/bin/ping" pid=6013 comm="bash" requested_mask="x" denied_mask="x" fsuid=1000 ouid=0
Jul 02 01:23:03 ubuntu-jammy audit[6013]: AVC apparmor="DENIED" operation="open" profile="snap.sample-snap.bash" name="/usr/bin/ping" pid=6013 comm="bash" requested_mask="r" denied_mask="r" fsuid=1000 ouid=0
...
```

ping を許可するためには Snap 標準で用意されている network-observe という interface が利用できます。network-observe により追加される AppArmor や Seccomp 設定は [interfaces/builtin/network\_observe.go][8] より確認できます。

自作 Snap の `snapcraft.yaml` に `plugs` 設定を追加し、同様の手順でビルド、インストールします。

```yaml
name: sample-snap
base: core20
version: '0.1'
summary: Execute bash
description: |
  Execute bash in Sandbox

grade: devel # must be 'stable' to release into candidate/stable channels
confinement: devmode # use 'strict' once you have the right plugs and slots

apps:
  bash:
    command: bin/main.sh
    # plugs の追加
    plugs:
      - network-observe

parts:
  bash:
    plugin: dump
    source: .
    organize:
      main.sh: bin/
```

インストール後、sample-snap パッケージが持つ plug と結びつける slot を明示的に指定する必要があります。Snap interface では機能を提供する側の口を slot, 利用者側を plug という概念で表します。

```bash
$ sudo snap connections sample-snap
Interface        Plug                         Slot  Notes
network-observe  sample-snap:network-observe  -     -
```

network-observe の slot は snapd により提供されています。

```bash
$ sudo snap interface network-observe
name:    network-observe
summary: allows querying network status
plugs:
  - multipass
  - my-snap-name
  - sample-snap
slots:
  - snapd

$ sudo snap connect sample-snap:network-observe snapd

$ sudo snap connections sample-snap
Interface        Plug                         Slot              Notes
network-observe  sample-snap:network-observe  :network-observe  manual
```

これで ping が動作することが確認できました。

```bash
$ sample-snap.bash

$ ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=8.14 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=63 time=11.6 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=63 time=8.49 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2054ms
rtt min/avg/max/mdev = 8.142/9.407/11.595/1.553 ms
```

### 参考

- Snap の作成について
  - [Snapcraft ovewview][2]
  - [第654回 snapパッケージング入門 Ubuntu Weekly Recipe - gihyo.jp][1]

[1]: https://gihyo.jp/admin/serial/01/ubuntu-recipe/0654
[2]: https://snapcraft.io/docs/snapcraft-overview
[3]: https://snapcraft.io/docs/base-snaps
[4]: https://snapcraft.io/docs/environment-variables
[5]: https://forum.snapcraft.io/t/documentation-of-var-lib-snapd-hostfs/7223
[6]: https://www.kernel.org/doc/Documentation/cgroup-v2.txt
[7]: https://snapcraft.io/docs/interface-management
[8]: https://github.com/snapcore/snapd/blob/master/interfaces/builtin/network_observe.go
[9]: https://snapcraft.io/
[10]: https://gihyo.jp/admin/serial/01/ubuntu-recipe/0714
[11]: https://snapcraft.io/docs/security-sandboxing
[12]: https://github.com/snapcore/snapd/blob/master/interfaces/seccomp/template.go
[13]: https://github.com/snapcore/snapd/tree/master/interfaces/apparmor
[14]: https://github.com/snapcore/snapd/blob/master/cmd/snap-confine/user-support.c
[15]: https://snapcraft.io/docs/snap-layouts
