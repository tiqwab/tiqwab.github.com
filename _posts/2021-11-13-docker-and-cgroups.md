---
layout: post
title: Docker コンテナの cgroups 関連機能について
tags: "cgroups,container,youki,docker"
comments: true
---

Docker 等でコンテナを作成する際、Linux カーネル機能の一つである cgroups が使われます。youki という OCI Runtime を中心に色々見ていく中で実際にコンテナ作成時にどのような cgroup が用意されるのかといったことがわかってきたので知識の整理として吐き出してみました。

主な目的:

- コンテナでどのように cgroup, cgroup fs のマウント、および cgroup namespace が設定されるかの把握

なお cgroups 自体の説明はあまりここにはありません。
ページ末尾にいくつか参考になるリンクを記載しています。

### コンテナにおける cgroups の利用目的

cgroups はコンテナで利用できるリソース (e.g. CPU やメモリ) に制限をかけるために使用されます。

例えば

`docker container run --memory=200m -it --rm ubuntu:20.04 /bin/bash`

は Docker にメモリ使用量の上限を 200 MB に設定したコンテナを作成するよう指示します。

実際にコンテナ用に作成された cgroup の内容を確認すると `memory.max` に指定した値が設定されていることが確認できます (ここでは cgroup v2 環境を利用)。

```bash
$ cat /sys/fs/cgroup/system.slice/docker-0adcd26d10322b943d3a5d0e507831695b385d0116da3c52a62caecca32d5f07.scope/memory.max
209715200
```

### cgroup fs について

上記手順でさらっと `/sys/fs/cgroup` 以下のファイルにアクセスしていましたが、このように cgroup への操作は cgroup ファイルシステム (cgroupfs) を介して行います。

cgroup には v1, v2 がありそれによりファイルシステムの階層構造が異なります。

#### cgroup v1

cgroup v1 では cgroup で管理する各リソース (cgroup ではこれをコントローラあるいはサブシステムと呼ぶようです。ここでは以降コントローラと呼びます) 毎に cgroupfs をマウントします。

以下は CentOS 7 上で cgroup に関連するマウントポイントを表示した例です。
通常 cgroup 関連のマウント先には `/sys/fs/cgroup` 以下が使われます。

```bash
$ sudo mount -l | grep cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,seclabel,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,pids)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,net_prio,net_cls)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,devices)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuacct,cpu)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuset)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,perf_event)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,memory)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,blkio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,hugetlb)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,freezer)
```

cgroup は各コントローラで独立に管理されるので例えば cpuset コントローラでは A という cgroup に所属するが memory コントローラでは B に所属するといったことができます。

なお自身の所属する cgroup 一覧は `/proc/<pid>/cgroup` で確認できます。

```bash
$ cat /proc/self/cgroup
11:memory:/user.slice/user-1000.slice/session-6.scope
10:devices:/user.slice
9:net_cls,net_prio:/
8:cpu,cpuacct:/
7:hugetlb:/
6:perf_event:/
5:freezer:/
4:cpuset:/
3:pids:/user.slice/user-1000.slice/session-6.scope
2:blkio:/
1:name=systemd:/user.slice/user-1000.slice/session-6.scope
0::/user.slice/user-1000.slice/session-6.scope
```

#### cgroup v2

cgroup v2 では v1 と違い各コントローラ毎に cgroup に所属するのではなく単一の cgroup に所属しそこで各コントローラによるリソース制御を行います。

システムで cgroup v2 ファイルシステムを利用しているかは `cgroup2` タイプのマウントが存在するかで判断できます。

```bash
$ sudo mount -l | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime)
```

現在のプロセスが所属する cgroup を見ると一つしか表示されません。

```bash
$ cat /proc/self/cgroup
0::/user.slice/user-1000.slice/session-2.scope
```

ある cgroup でどのコントローラがサポートされているかは `cgroup.controllers` から判断できます。

```bash
$ cat /sys/fs/cgroup/cgroup.controllers
cpuset cpu io memory hugetlb pids rdma misc
```

また cgroup は階層構造を持つ (cgroup に親子関係が存在する) のですが、子の cgroup でどのコントローラを有効にするかを `cgroup.subtree_control` で制御できます。

```bash
$ cat /sys/fs/cgroup/cgroup.subtree_control
cpuset cpu io memory hugetlb pids rdma misc
```

### cgroup namespace について

namespace はコンテナという隔離された環境を作るために用いられる Linux カーネル機能の一つです。
扱いたいリソース毎に namespace が存在し、その中に cgroup namespace というのが存在します。

これは文字通り cgroup 用の namespace であり、実際のルートとは異なる cgroup をルートとみなした環境でプログラムを実行できます。

cgroup namespace の動作例としてここでは unshare を使ってみます。unshare コマンドでは `--cgroup` オプションにより、unshare を呼び出したプロセスが所属する cgroup をルートにした新しい cgroup namespace でコマンド実行を行います。

```bash
# with cgroup v2

# cgroup の作成
[sample]$ sudo mkdir /sys/fs/cgroup/mytest

# 作成した cgroup に所属
[sample]$ sudo bash -c "echo $$ > /sys/fs/cgroup/mytest/cgroup.procs"
[sample]$ cat /proc/self/cgroup
0::/mytest
[sample]$ ls /sys/fs/cgroup
cgroup.controllers      cgroup.threads         dev-mqueue.mount  io.stat           sys-fs-fuse-connections.mount
cgroup.max.depth        cpu.pressure           init.scope        memory.numa_stat  sys-kernel-config.mount
cgroup.max.descendants  cpuset.cpus.effective  io.cost.model     memory.pressure   sys-kernel-debug.mount
cgroup.procs            cpuset.mems.effective  io.cost.qos       memory.stat       sys-kernel-tracing.mount
cgroup.stat             cpu.stat               io.pressure       misc.capacity     system.slice
cgroup.subtree_control  dev-hugepages.mount    io.prio.class     mytest            user.slice

# 新しい mount, cgroup namespace でコマンド実行
[sample]$ sudo unshare --mount --cgroup /bin/bash

# cgroupfs のマウントし直し
[sample]# mount --make-rprivate /
[sample]# umount /sys/fs/cgroup
[sample]# mount -t cgroup2 none /sys/fs/cgroup

# cgroup の確認
# 自身の所属していた cgroup がルートになっている
[sample]# cat /proc/self/cgroup
0::/
[sample]# ls /sys/fs/cgroup
cgroup.controllers      cpuset.cpus               hugetlb.1GB.rsvd.current  io.stat              memory.swap.events
cgroup.events           cpuset.cpus.effective     hugetlb.1GB.rsvd.max      io.weight            memory.swap.high
cgroup.freeze           cpuset.cpus.partition     hugetlb.2MB.current       memory.current       memory.swap.max
cgroup.kill             cpuset.mems               hugetlb.2MB.events        memory.events        misc.current
cgroup.max.depth        cpuset.mems.effective     hugetlb.2MB.events.local  memory.events.local  misc.max
cgroup.max.descendants  cpu.stat                  hugetlb.2MB.max           memory.high          pids.current
cgroup.procs            cpu.uclamp.max            hugetlb.2MB.rsvd.current  memory.low           pids.events
cgroup.stat             cpu.uclamp.min            hugetlb.2MB.rsvd.max      memory.max           pids.max
cgroup.subtree_control  cpu.weight                io.bfq.weight             memory.min           rdma.current
cgroup.threads          cpu.weight.nice           io.latency                memory.numa_stat     rdma.max
cgroup.type             hugetlb.1GB.current       io.low                    memory.oom.group
cpu.max                 hugetlb.1GB.events        io.max                    memory.pressure
cpu.max.burst           hugetlb.1GB.events.local  io.pressure               memory.stat
cpu.pressure            hugetlb.1GB.max           io.prio.class             memory.swap.current
```

### Docker 経由のコンテナ作成における cgroup

Docker によって用意されるコンテナでここまで見てきた cgroup, cgroupfs のマウント、cgroup namespace の設定がどのように扱われるかを調べてみました。

Docker の実装ではコンテナの作成は最終的に OCI Runtime (e.g. runc) に委ねられます ([参考][6])。
このとき OCI Runtime にはどのようなコンテナを作成してほしいかを記述した [OCI Runtime Spec][7] を渡します。

Docker のソースを見ると [moby/oci/defaults.go][8] でデフォルトの spec が定義されています。コンテナ作成のリクエストハンドラである [api/server/router/container\_router.go][8] の `postContainersStart` から始まる処理も併せて確認すると、デフォルトでは cgroup 関連の設定は以下のようになります。

- コンテナ毎に cgroup は新しく切られる
- cgroup v1, v2 に依らず cgroupfs はマウントされる
- cgroup namespace については cgroup v1, v2 で異なる
  - cgroup v1 では cgroup namespace は新しく切られない
    - その場合 OCI Runtime 側で bind マウントにより cgroup の隔離が行われる
  - cgroup v2 では切られる

以下では cgroup v1, v2 それぞれについてより詳細を見ていきます。

### cgroup v1 環境 (legacy)

#### cgroup

cgroup v1 環境においては上述のように各コントローラ毎に cgroup が存在します。どこに cgroup を作成するかは OCI Runtime Spec の [linux.cgroupsPath][10] で指定することができ、Docker 越しだと `/docker/<container_id>` となっています。

下記は実際に Docker (with youki) で作成されたコンテナの cgroup 確認例です。

```bash
# OCI Runtime に youki を指定してコンテナ実行
# Runtime の追加は https://github.com/containers/youki に従って行った
$ sudo docker container run --rm -it --runtime=youki busybox /bin/sh

# 別ターミナルで (ホストからはコンテナが pid=12129 として見えている)
$ cat /proc/12129/cgroup
11:memory:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
10:devices:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
9:net_cls,net_prio:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
8:cpu,cpuacct:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
7:hugetlb:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
6:perf_event:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
5:freezer:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
4:cpuset:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
3:pids:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
2:blkio:/docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b
1:name=systemd:/user.slice/user-1000.slice/session-5.scope
0::/user.slice/user-1000.slice/session-5.scope
```

#### cgroupfs

Docker 経由で作成されるコンテナ内でも cgroupfs はマウントされます (あるいは少なくともそのようにコンテナには見せる、という方が正確かもしれません。詳細は次の cgroup namespace 項)。OCI Runtime Spec の [mounts][11] 内で cgroup type のマウントを含めることで作成を指示します。

```json
"mounts": [
  ...
  {
    "destination": "/sys/fs/cgroup",
    "type": "cgroup",
    "source": "cgroup",
    "options": [
      "ro",
      "nosuid",
      "noexec",
      "nodev"
    ]
  },
  ...
],
```

作成されたコンテナに相当するプロセスの mountinfo を見ると実際にマウントされた内容が確認できます。

```bash
$ cat /proc/12129/mountinfo | grep cgroup
567 566 0:60 / /sys/fs/cgroup rw,nosuid,nodev,noexec,relatime - tmpfs tmpfs rw,seclabel,mode=755
568 567 0:28 /user.slice/user-1000.slice/session-5.scope /sys/fs/cgroup/systemd rw,nosuid,nodev,noexec,relatime master:6 - cgroup cgroup rw,seclabel,xattr,name=systemd
569 567 0:31 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/blkio rw,nosuid,nodev,noexec,relatime master:7 - cgroup cgroup rw,seclabel,blkio
570 567 0:32 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/pids rw,nosuid,nodev,noexec,relatime master:8 - cgroup cgroup rw,seclabel,pids
571 567 0:33 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/cpuset rw,nosuid,nodev,noexec,relatime master:9 - cgroup cgroup rw,seclabel,cpuset
726 567 0:34 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/freezer rw,nosuid,nodev,noexec,relatime master:10 - cgroup cgroup rw,seclabel,freezer
727 567 0:35 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/perf_event rw,nosuid,nodev,noexec,relatime master:11 - cgroup cgroup rw,seclabel,perf_event
728 567 0:36 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/hugetlb rw,nosuid,nodev,noexec,relatime master:12 - cgroup cgroup rw,seclabel,hugetlb
729 567 0:37 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/cpu,cpuacct rw,nosuid,nodev,noexec,relatime master:13 - cgroup cgroup rw,seclabel,cpu,cpuacct
730 567 0:38 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/net_cls,net_prio rw,nosuid,nodev,noexec,relatime master:14 - cgroup cgroup rw,seclabel,net_cls,net_prio
731 567 0:39 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/devices rw,nosuid,nodev,noexec,relatime master:15 - cgroup cgroup rw,seclabel,devices
732 567 0:40 /docker/2f7f1f4fe4865f25c728dd6dbdcbf243b7ea968aef3902b90a2ca2580ea5324b /sys/fs/cgroup/memory rw,nosuid,nodev,noexec,relatime master:16 - cgroup cgroup rw,seclabel,memory
```

#### cgroup namespace

cgroup v1 ではどうやらコンテナ用に新しく cgroup namespace を切ることはしないようです。

実際にコンテナに相当するプロセスの cgroup namespace を確認するとホストのそれと一致します。

```bash
# namespaces of host
$ sudo ls -l /proc/self/ns
total 0
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 net -> 'net:[4026531992]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 pid -> 'pid:[4026531836]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 time -> 'time:[4026531834]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 user -> 'user:[4026531837]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 uts -> 'uts:[4026531838]'

# namespaes of container
$ sudo ls -l /proc/12129/ns
total 0
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 ipc -> 'ipc:[4026532178]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 mnt -> 'mnt:[4026532252]'
lrwxrwxrwx. 1 root root 0 Nov  5 20:59 net -> 'net:[4026532181]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 pid -> 'pid:[4026532177]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 pid_for_children -> 'pid:[4026532177]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 time -> 'time:[4026531834]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 user -> 'user:[4026531837]'
lrwxrwxrwx. 1 root root 0 Nov  5 21:32 uts -> 'uts:[4026532179]'
```

しかし一方でコンテナ向けに cgroupfs はマウントするよう指示しているので何も考慮しないとコンテナからホスト上の他の cgroup についての閲覧、操作ができてしまいそうです。

実際 unshare(1) を使ってそのような状況を再現してみました。
ここでは memory を `/mytest` という cgroup に所属させているのでそれをルートにしてコンテナ内では見せたいのですが、そうはなっておらずホストのルートがそのまま見えてしまっていることがわかります。

```bash
# memory コントローラ下に /mytest cgroup を作成し所属する
[mycontainer]$ sudo mkdir /sys/fs/cgroup/memory/mytest
[mycontainer]$ sudo bash -c "echo $$ > /sys/fs/cgroup/memory/mytest/tasks"
[mycontainer]$ cat /proc/$$/cgroup
11:memory:/mytest
10:devices:/user.slice
9:net_cls,net_prio:/
8:cpu,cpuacct:/
7:hugetlb:/
6:perf_event:/
5:freezer:/
4:cpuset:/
3:pids:/user.slice/user-1000.slice/session-6.scope
2:blkio:/
1:name=systemd:/user.slice/user-1000.slice/session-6.scope
0::/user.slice/user-1000.slice/session-6.scope

# 疑似コンテナ環境を用意する
# mount namespace のみを新しく切っている
[mycontainer]$ sudo unshare --mount /bin/bash
[mycontainer]# mount --make-rprivate /
[mycontainer]# mount --rbind ./rootfs ./rootfs
[mycontainer]# mount -t proc proc ./rootfs/proc

# cgroupfs のマウント
# cpuset, memory コントローラのみ用意
[mycontainer]# mkdir -p rootfs/sys/fs/cgroup
[mycontainer]# mount -t tmpfs cgroup_root rootfs/sys/fs/cgroup
[mycontainer]# mkdir rootfs/sys/fs/cgroup/cpuset
[mycontainer]# mount -t cgroup -ocpuset cpuset rootfs/sys/fs/cgroup/cpuset
[mycontainer]# mkdir rootfs/sys/fs/cgroup/memory
[mycontainer]# mount -t cgroup -omemory memory rootfs/sys/fs/cgroup/memory

# ルートファイルシステムを切り替え
[mycontainer]# mkdir rootfs/oldrootfs
[mycontainer]# pivot_root ./rootfs ./rootfs/oldrootfs
[mycontainer]# cd /
[/]# umount -l oldrootfs/

# コンテナ内からマウントポイント一覧の確認
[/]# cat /proc/self/mountinfo
540 504 8:1 /home/vagrant/sample/rootfs / rw,relatime - ext4 /dev/sda1 rw,seclabel
541 540 0:53 / /proc rw,relatime - proc proc rw
542 540 0:54 / /sys/fs/cgroup rw,relatime - tmpfs cgroup_root rw,seclabel
543 542 0:33 / /sys/fs/cgroup/cpuset rw,relatime - cgroup cpuset rw,seclabel,cpuset
544 542 0:40 / /sys/fs/cgroup/memory rw,relatime - cgroup memory rw,seclabel,memory

# コンテナ内から cgroup 一覧の確認
# (これはホストから見ても同じ)
[/]# cat /proc/self/cgroup
11:memory:/mytest
10:devices:/user.slice
9:net_cls,net_prio:/
8:cpu,cpuacct:/
7:hugetlb:/
6:perf_event:/
5:freezer:/
4:cpuset:/
3:pids:/user.slice/user-1000.slice/session-6.scope
2:blkio:/
1:name=systemd:/user.slice/user-1000.slice/session-6.scope
0::/user.slice/user-1000.slice/session-6.scope

[/]# ls -l /sys/fs/cgroup
total 0
dr-xr-xr-x    3 root     root             0 Nov  4 23:37 cpuset
dr-xr-xr-x    7 root     root             0 Nov  4 23:37 memory

# コンテナ内から cpuset コントローラのルート直下の一覧確認
[/]# ls -l /sys/fs/cgroup/cpuset
total 0
-rw-r--r--    1 root     root             0 Nov  5 21:41 cgroup.clone_children
-rw-r--r--    1 root     root             0 Nov  5 21:41 cgroup.procs
-r--r--r--    1 root     root             0 Nov  5 21:41 cgroup.sane_behavior
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.cpu_exclusive
-rw-r--r--    1 root     root             0 Nov  5 20:57 cpuset.cpus
-r--r--r--    1 root     root             0 Nov  5 21:41 cpuset.effective_cpus
-r--r--r--    1 root     root             0 Nov  5 21:41 cpuset.effective_mems
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.mem_exclusive
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.mem_hardwall
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.memory_migrate
-r--r--r--    1 root     root             0 Nov  5 21:41 cpuset.memory_pressure
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.memory_pressure_enabled
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.memory_spread_page
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.memory_spread_slab
-rw-r--r--    1 root     root             0 Nov  5 20:57 cpuset.mems
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.sched_load_balance
-rw-r--r--    1 root     root             0 Nov  5 21:41 cpuset.sched_relax_domain_level
drwxr-xr-x    2 root     root             0 Nov  5 20:59 docker
-rw-r--r--    1 root     root             0 Nov  5 21:41 notify_on_release
-rw-r--r--    1 root     root             0 Nov  5 21:41 release_agent
-rw-r--r--    1 root     root             0 Nov  5 21:41 tasks

# コンテナ内から memory コントローラのルート直下の一覧確認
# ここで本来は /mytest 下の情報だけ表示されてほしいが、docker 等がいるように
# ルートの情報が表示されている
[/]# ls -l /sys/fs/cgroup/memory
total 0
-rw-r--r--    1 root     root             0 Nov  5 23:52 cgroup.clone_children
--w--w--w-    1 root     root             0 Nov  5 23:52 cgroup.event_control
-rw-r--r--    1 root     root             0 Nov  5 23:52 cgroup.procs
-r--r--r--    1 root     root             0 Nov  5 23:52 cgroup.sane_behavior
drwxr-xr-x    2 root     root             0 Nov  5 20:59 docker
drwxr-xr-x    2 root     root             0 Nov  5 23:52 init.scope
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.failcnt
--w-------    1 root     root             0 Nov  5 23:52 memory.force_empty
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.kmem.failcnt
-rw-r--r--    1 root     root             0 Nov  5 20:59 memory.kmem.limit_in_bytes
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.kmem.max_usage_in_bytes
-r--r--r--    1 root     root             0 Nov  5 23:52 memory.kmem.slabinfo
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.kmem.tcp.failcnt
-rw-r--r--    1 root     root             0 Nov  5 20:59 memory.kmem.tcp.limit_in_bytes
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.kmem.tcp.max_usage_in_bytes
-r--r--r--    1 root     root             0 Nov  5 23:52 memory.kmem.tcp.usage_in_bytes
-r--r--r--    1 root     root             0 Nov  5 23:52 memory.kmem.usage_in_bytes
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.limit_in_bytes
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.max_usage_in_bytes
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.memsw.failcnt
-rw-r--r--    1 root     root             0 Nov  5 20:59 memory.memsw.limit_in_bytes
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.memsw.max_usage_in_bytes
-r--r--r--    1 root     root             0 Nov  5 23:52 memory.memsw.usage_in_bytes
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.move_charge_at_immigrate
-r--r--r--    1 root     root             0 Nov  5 23:52 memory.numa_stat
-rw-r--r--    1 root     root             0 Nov  5 20:59 memory.oom_control
----------    1 root     root             0 Nov  5 23:52 memory.pressure_level
-rw-r--r--    1 root     root             0 Nov  5 20:59 memory.soft_limit_in_bytes
-r--r--r--    1 root     root             0 Nov  5 23:52 memory.stat
-rw-r--r--    1 root     root             0 Nov  5 20:59 memory.swappiness
-r--r--r--    1 root     root             0 Nov  5 23:52 memory.usage_in_bytes
-rw-r--r--    1 root     root             0 Nov  5 23:52 memory.use_hierarchy
drwxr-xr-x    2 root     root             0 Nov  5 21:43 mytest
-rw-r--r--    1 root     root             0 Nov  5 23:52 notify_on_release
-rw-r--r--    1 root     root             0 Nov  5 23:52 release_agent
drwxr-xr-x   30 root     root             0 Nov  5 20:59 system.slice
-rw-r--r--    1 root     root             0 Nov  5 23:52 tasks
drwxr-xr-x    3 root     root             0 Nov  5 20:57 user.slice
```

しかし実際に Docker 経由で作成されるコンテナではちゃんと自分の cgroup をルートにした環境が整えられています。
youki の実装を見るとどうやら spec で cgroup namespace を切らないよう指示された場合には cgroup type のマウントを生やすのではなく bind マウントを使うようです。

作成されたコンテナの mountinfo をホストから見ると、

```
726 568 0:33 /docker/74029a18e504f42b1f743c88939c1dda81e94429c307bc2d551ab5cf1017858c /sys/fs/cgroup/cpuset ro,nosuid,nodev,noexec,relatime master:9 - cgroup cgroup rw,seclabel,cpuset
```

のように `master:x` という表記があり、これは [mount propagation type][12] のうち slave であることを示しています。
対応するマウントポイントはホストの mountinfo 内にある

```
36 29 0:33 / /sys/fs/cgroup/cpuset rw,nosuid,nodev,noexec,relatime shared:9 - cgroup cgroup rw,seclabel,cpuset
```

であり、ここから bind マウントによってコンテナの cgroup 用のディレクトリ構造が用意されていることがわかります。

実装は見ていませんが作成されるコンテナの mountinfo を見る限りは runc も同様の挙動のようです。

### cgroup v2 環境 (unified)

#### cgroup

cgroup v2 でもコンテナ毎に新しく cgroup が用意されます。

```bash
# OCI Runtime に youki を指定してコンテナ実行
$ sudo docker container run -it --rm --runtime=youki ubuntu:20.04 /bin/bash

# 別ターミナルで (ホストからはコンテナが pid=147356 として見えている)
$ cat /proc/147356/cgroup
0::/system.slice/docker-a2c88db75e2beb0a88972247854d51684dad061fdca888007ed04915976037c1.scope
```

#### cgroupfs

cgroup v2 でもコンテナ向けに cgroupfs が用意されます。

```bash
$ cat /proc/147356/mountinfo | grep cgroup
285 284 0:30 /system.slice/docker-a2c88db75e2beb0a88972247854d51684dad061fdca888007ed04915976037c1.scope /sys/fs/cgroup ro,nosuid,nodev,noexec,relatime - cgroup2 cgroup rw
```

#### cgroup namespace

cgroup v2 では v1 と違いデフォルトでは cgroup namespace が新しくコンテナ用に作られます。

OCI Runtime Spec 上では [linux.namespaces][15] に `cgroup` を含めて指示します。

```json
  "linux": {
    ...
    "namespaces": [
      {
        "type": "mount"
      },
      {
        "type": "network"
      },
      {
        "type": "uts"
      },
      {
        "type": "pid"
      },
      {
        "type": "ipc"
      },
      {
        "type": "cgroup"
      }
    ],
    ...
  }
```

その設定が加えられるのは恐らく [moby/daemon\_unix.go][14] です。
`cgroups.Mode() == cgroups.Unified`、つまり cgroup v2 であれば `hostConfig.CgroupnsMode = containertypes.CgroupnsPrivate` が設定されるようになっています。これがのちに OCI Runtime Spec を生成する際に cgroup namespace を作成するような指示に変換されます。

youki のソースを確認したところ、コンテナで新しく cgroup namespace を作成する手順はおおよそ以下のようになるという理解です。ここでは unshare(1) でできるだけ手順を再現してみています。新しく cgroup namespace を作成した際、unshare を呼んだプロセスが所属していた cgroup がルートに見えるので、先に希望の cgroup に所属させた上で unshare を呼んでいます。

```bash
# 事前にコンテナ用の cgroup を用意する
[mycontainer]$ sudo mkdir /sys/fs/cgroup/mytest

# 新しい mount, pid namespace を用意した /bin/bash を実行する
# --fork は pid namespace を作成するときには必要
[mycontainer]$ sudo unshare --mount --pid --fork /bin/bash

# 自身を用意した cgroup に加入させる
[mycontainer]# bash -c "echo $$ > /sys/fs/cgroup/mytest/cgroup.procs"

# 新しい cgroup namespace を用意した /bin/bash を実行する
# この中では unshare を呼んだプロセスが所属していた cgroup が root cgroup になる
[mycontainer]# unshare --cgroup /bin/bash

# ここではカレントディレクトリ下の rootfs をコンテナの rootfs として使用する想定
[mycontainer]# mount --make-rprivate /
[mycontainer]# mount --rbind ./rootfs ./rootfs
[mycontainer]# mount -t proc proc ./rootfs/proc

# cgroup2 fs をマウントする
[mycontainer]# mount -t cgroup2 none ./rootfs/sys/fs/cgroup

[mycontainer]# mkdir ./rootfs/oldrootfs
[mycontainer]# pivot_root ./rootfs ./rootfs/oldrootfs
[mycontainer]# cd /
[/]# umount -l oldrootfs
[/]# rmdir oldrootfs/
```

これでコンテナ自身からは自分がルートの cgroup に所属しているように見えます。
(ただしここでは手順上余計な bash プロセスが存在しています。これを避けるなら pid namespace の作成を 2 回目の unshare で行い、1 回目の unshare で作成されたプロセスを後で終了させればよいはず)

```bash
[/]# cat /proc/$$/cgroup
0::/

[/]# cat /sys/fs/cgroup/cgroup.procs
1
3
41

[/]# ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/bash
    3 root      0:00 /bin/bash
   42 root      0:00 ps aux
```

一方でホストからコンテナプロセスの cgroup を見るとルートではなく `/mytest` に所属しています。
(rootfs のパスを一部改変して記載)

```bash
[sample]$ cat /sys/fs/cgroup/mytest/cgroup.procs
105353
105370

[sample]$ cat /proc/105370/cgroup
0::/mytest

[sample]$ cat /proc/105370/mountinfo
265 233 0:25 <host_path_to_mycontainer>/rootfs / rw,relatime - btrfs /dev/nvme0n1p7 rw,compress=zstd:3,ssd,space_cache,subvolid=258,subvol=/@home
266 265 0:161 / /proc rw,relatime - proc proc rw
267 265 0:30 /mytest /sys/fs/cgroup rw,relatime - cgroup2 none rw
```

### 参考

- cgroups 概要
  - [Control Groups version 1 - www.kernel.org][1]
  - [Control Groups version 2 - www.kernel.org][1]
  - [LXCで学ぶコンテナ入門 －軽量仮想化環境を実現する技術 第37回][3]
    - cgroup v1 と v2 の差異が解説されている
- cgroup namespace 概要
  - [man 7 cgroup\_namespace][5]
  - [ControlGroups version2 の Namespace 項 - www.kernel.org][4]
- mount propagation type について
  - [Shared Subtrees - www.kernel.org][12]
  - [Mount namespaces, mount propagation, and unbindable mounts - lwn.net][13]

[1]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/index.html
[2]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html
[3]: https://gihyo.jp/admin/serial/01/linux_containers/0037
[4]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#namespace
[5]: https://man7.org/linux/man-pages/man7/cgroup_namespaces.7.html
[6]: https://medium.com/nttlabs/runc-overview-263b83164c98
[7]: https://github.com/opencontainers/runtime-spec/blob/master/spec.md
[8]: https://github.com/moby/moby/blob/c09789c1140a3b2e399b0c77c449b34d000e2362/oci/defaults.go#L16
[9]: https://github.com/moby/moby/blob/c09789c1140a3b2e399b0c77c449b34d000e2362/api/server/router/container/container_routes.go#L177
[10]: https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#cgroups-path
[11]: https://github.com/opencontainers/runtime-spec/blob/main/config.md#mounts
[12]: https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt 
[13]: https://lwn.net/Articles/690679/
[14]: https://github.com/moby/moby/blob/402d106142e77a092dc761b354fb43d6935aa4f9/daemon/daemon_unix.go#L357
[15]: https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md#namespaces
