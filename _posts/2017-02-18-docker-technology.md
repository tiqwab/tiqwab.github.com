---
layout: post
title: Docker技術入門
tags: "docker"
comments: true
---

最近になってようやく開発環境にDockerを使い始めたのですが、[この記事][3]を見てやっぱり一度Dockerの動作の仕組みをおさえておきたいよなと思いました。

馴染みのない部分も多く完璧におさえることは難しいので、ここではDockerの使用するLinuxの機能としてよく聞く3つ(overflayfs, namespaces, cgroups)が何なのかという点に注目して簡単にまとめてみました。

1. [Dockerとは](#anchor1)
2. [ハイパーバイザー型との比較](#anchor2)
3. [Dockerの使用するLinux技術](#anchor3)

---

<a id="anchor1"></a>

### 1. Dockerとは

一言でいえばコンテナ型の仮想化ツールです。

- コンテナ型
  - ホストOSのカーネルを使いゲスト (コンテナ)を動かす
  - それぞれのコンテナが独立したリソース (Filesystem, Processes, Memory, Devices, Network devices)を持つ (持っているように見せることができる)

- Dockerがコンテナ型の仮想化という方式を生み出したというわけではない
  - コンテナ型の仮想化自体はもっと前からある
  - DockerはLinuxの各種機能を使用しているだけで、仮想化に関して独自の技術を使っているわけではない (はず)

- Dockerが提供するのはコンテナを使用した開発やデプロイを行いやすくする仕組み
  - 直感的なコマンドライン操作
  - REST API
  - Docker Hubとの連携
  - etc.

<a id="anchor2"></a>

### 2. ハイパーバイザー型との比較

[ハイパーバイザー][8]が「ハードウェアごと仮想化」して仮想化機械(VM)を作成するものならば、コンテナ型は「OSレベルの仮想化」という表現になるかと思います。

- ハイパーバイザー型と比較したコンテナ型のメリット
  - 仮想化のために必要になるCPU, RAM, Storage等のオーバーヘッドが小さい
  - ホストOSから見るとコンテナはただのプロセスなので起動が早い
    - ハイパーバイザーの場合、それぞれのVMでOSを起動するような形になる

<a id="anchor3"></a>

### 3. Dockerの使用するLinux技術

Dockerはコンテナを作成するために種々のLinuxが持つ機能を利用しており、主要なものとして以下の3つがよく挙げられます。

- overlayfs
- namespaces
- cgroups

#### overlayfs

overlayfsはunion filesystem実装の一つ、と言われています。

詳細は下記の参考がわかりやすいのですが、union filesystemにはlowerdir, upperdirの概念があり、それらのディレクトリの重ね合わせでマウント先のディレクトリを表現します。
(ちなみにoverlayfsストレージドライバではlowerdirは1つしか指定できなかったが、overlayfs2では複数指定できる)

Dockerの場合、overlayfsがイメージやコンテナの管理に大きく関わっています。
この点に関しても下記の参考が詳しいのですが、こちらでも具体例を挙げて見てみようと思います。

コンテナのファイルシステム関連の内容ですがホストOS上で以下のように配置されています。

```
$ ls -al /var/lib/docker/overlay2/acc24111f7eae3c11f550840e56719a36a48c2ee39998be969e7290ebe4b7e53
total 28
drwx------ 5 root root 4096 Feb 18 14:32 .
drwx------ 7 root root 4096 Feb 18 14:45 ..
drwxr-xr-x 3 root root 4096 Feb 18 14:34 diff
-rw-r--r-- 1 root root   26 Feb 18 14:32 link
-rw-r--r-- 1 root root   57 Feb 18 14:32 lower
drwx------ 2 root root 4096 Feb 18 14:32 merged
drwx------ 3 root root 4096 Feb 18 14:45 work
```

- `/var/lib/docker/overlay2/<container id>` にコンテナのファイルシステムに関連するものがまとまっている
- `diff` がupperdir, `lower` がlowerdir, `work` がworkdirの役目
- 実際にコンテナ内で見えているのはそれらがmergeされた `merged`
  - `mount | grep overlay2` とかでこの様子は確認できる

overlayfsを使用した単純な構成ですね...と言いたいのですが、lowerdirに関してはコンテナ、イメージ間の関係を表現するために少しだけ複雑なことをしています。

`lower` は上のlsコマンドの結果で表示されているように、ディレクトリではなくファイルであり、その内容は以下のようになっています。

```
$ cat /var/lib/docker/overlay2/acc24111f7eae3c11f550840e56719a36a48c2ee39998be969e7290ebe4b7e53/lower
l/G2JXWQVKRWSPWCE6ENZPDHUKLQ:l/PBIUX5XLZZDDORB35GNALGUJYN
```
この場合lowerdirは `l/G2JXWQVKRWSPWCE6ENZPDHUKLQ` と `l/PBIUX5XLZZDDORB35GNALGUJYN`  だよ、ということを示しているのですが、これだけでは何を指しているのかわかりません。

これの指す先は `var/lib/docker/overlay2/l` を見ることでわかります。

```
$ ls -al /var/lib/docker/overlay2/l
total 24
drwx------ 2 root root 4096 Feb 18 14:45 .
drwx------ 7 root root 4096 Feb 18 14:45 ..
lrwxrwxrwx 1 root root   72 Feb 18 14:45 3IOH4LKZPYJ3V6ZKXYFMRFVWZS -> ../c9e0b0edab5243549ae43033f28ff0779a492c3810753b555102451c09a12f3a/diff
lrwxrwxrwx 1 root root   77 Feb 18 14:32 G2JXWQVKRWSPWCE6ENZPDHUKLQ -> ../acc24111f7eae3c11f550840e56719a36a48c2ee39998be969e7290ebe4b7e53-init/diff
lrwxrwxrwx 1 root root   72 Feb 13 22:52 PBIUX5XLZZDDORB35GNALGUJYN -> ../a0c4b6d93a9bb8c702d5afc517badc32ea72fa9949b28294486370b216014cbe/diff
lrwxrwxrwx 1 root root   72 Feb 18 14:32 QINFLIKX3HZUZMJSMJNDYENPPF -> ../acc24111f7eae3c11f550840e56719a36a48c2ee39998be969e7290ebe4b7e53/diff
```

どうやら `lower` の内容が指す先はこのイメージ、あるいはコンテナであるというのがこのディレクトリに存在するシンボリックリンクにより結びつけられていることがわかります。

同じ仕組みがコンテナ、イメージの差分にも、イメージ間の差分にも使用できるので、overlayfs2ではわかりやすい動きで省ディスクなコンテナ管理ができていると言えます。
(ちなみにoverlayfs1では複数lowerdirの指定ができないので、どうやらハードリンクで同じことをやっているようです)

なお `lower` で使われているIDは各々の `link` 内を見てもわかるようになっています。

```
$ cat /var/lib/docker/overlay2/acc24111f7eae3c11f550840e56719a36a48c2ee39998be969e7290ebe4b7e53/link
QINFLIKX3HZUZMJSMJNDYENPPF
```

参考

- [Docker and OverlayFS in practice][9]
- [LXCで学ぶコンテナ入門][10]

#### namespaces

namespaceはLinuxが管理するリソースに名前空間を導入することで、プロセス単位で独自の環境を作ることを可能にしている機能です。

namespaceが対象としているリソースの種類は6つです。

- mount (ファイルシステムのマウント)
- uts (ホスト名)
- ipc (プロセス間通信)
- user (UID, GID)
- pid (プロセスID)
- net (ネットワークデバイス)

namespaceを試しに扱う一番お手軽な方法はunshareコマンドを実行することです。
以下ではunshareを使用してuts namespaceの作成や削除を確認しています。

```
$ hostname (現在のhostnameの確認)
first
$ ps -o pid,utsns,args (現在のプロセスのuts namespace等を表示する)
  PID      UTSNS COMMAND
21929 4026531838 -bash
22126 4026531838 ps -o pid,utsns,args
$ sudo ls -l /proc/$$/ns (あるいは/proc/<PID>/nsで確認することも可能)
lrwxrwxrwx 1 nm nm 0 Feb 18 09:54 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 nm nm 0 Feb 18 09:54 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 nm nm 0 Feb 18 09:54 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 nm nm 0 Feb 18 09:54 net -> net:[4026531969]
lrwxrwxrwx 1 nm nm 0 Feb 18 09:54 pid -> pid:[4026531836]
lrwxrwxrwx 1 nm nm 0 Feb 18 09:54 uts -> uts:[4026531838]

$ sudo unshare --uts /bin/bash (新規uts namespaceでプロセスを開始する)
# echo $$
21939
# hostname
first
# lsns (lsnsで現在の名前空間一覧を表示する)
        NS TYPE   NPROCS   PID USER  COMMAND
4026531835 cgroup    149     1 root  /sbin/init
4026531836 pid       149     1 root  /sbin/init
4026531838 uts       147     1 root  /sbin/init
4026531839 ipc       149     1 root  /sbin/init
4026531840 mnt       145     1 root  /sbin/init
4026531857 mnt         1    32 root  kdevtmpfs
4026531969 net       148     1 root  /sbin/init
4026532307 uts         2 21939 root  /bin/bash (新規に追加されている)
# hostname second
# hostname (hostnameが変更されていることの確認)
second
# exit

$ sudo lsns (上で作成したuts namespaceは所属するプロセスが0になったので自動的に削除されていることの確認)
        NS TYPE   NPROCS   PID USER  COMMAND
4026531835 cgroup    146     1 root  /sbin/init
4026531836 pid       146     1 root  /sbin/init
4026531838 uts       146     1 root  /sbin/init
4026531839 ipc       146     1 root  /sbin/init
4026531840 mnt       142     1 root  /sbin/init
4026531857 mnt         1    32 root  kdevtmpfs
4026531969 net       145     1 root  /sbin/init
$ hostname (hostnameが初期のものであることの確認)
first
```

コンテナ型の仮想化では、コンテナ毎にnamespaceを割り当てることによってそれぞれで独立した環境を作成しているようです。

#### cgroups

cgroupsはプロセス単位でホストOSが持つリソースの制限を行うための機能です。
これにより例えばCPUやメモリの使用量の制限が行えます。

cgroupsの操作は `/sys/fs/croup` にマウントされている専用の仮想ファイルシステムを使用して行うことができます。
(感覚的には `/proc` 以下に対する操作の仕方と同様の話になる)

```
$ ls /sys/fs/cgroup/
blkio  cpu  cpuacct  cpu,cpuacct  cpuset  devices  freezer  memory  net_cls  perf_event  pids  systemd
```

プロセスの属するcgroupsは `cat /proc/<pid>/cgroup` で確認可能です。

```
$ cat /proc/$$/cgroup
10:blkio:/
9:pids:/user.slice/user-1000.slice/session-c2.scope
8:freezer:/
7:perf_event:/
6:cpuset:/
5:net_cls:/
4:cpu,cpuacct:/
3:devices:/user.slice
2:memory:/
1:name=systemd:/user.slice/user-1000.slice/session-c2.scope
```

簡単な操作の例は以下のページ等で解説されています。

参考

- [Linuxカーネルのコンテナ機能［2］ ─cgroupとは？（その1）][5]

---

以上の内容より、イメージからコンテナを作るとはざっくりと以下の操作を行うことになるのかなと考えています。
ちゃんと確認しているわけではなく、あくまで雰囲気はという程度なので、この点に関しては自身の理解が深まればちゃんと書き直したいところです。

- 自身のイメージとその `lower` に含まれるイメージからコンテナにとってのlowerdirを設定
- overlayfsにより、コンテナ用ファイルシステムを `merged` に用意
  - このときコンテナの `diff` がupperdirに指定される
- `merged` をルートにしたファイルシステムでコンテナ作成 (コンテナ用のプロセス開始)
  - mount namespaceの利用?
- namespaceで独立した環境の用意
- cgroupsでリソース使用量を適当に制限する
- 指定されたコマンドをコンテナ内で実行

### 参考

- [Docker][1]
  - 公式サイト
- [Docker Tutorial][4]
  - 公式のTutorial動画等
- [いまさら聞けないDocker入門][2]
  - Dockerの使用に関する解説記事
- [Linuxカーネルのコンテナ機能][6]
  - Dockerに関してではなくLNC(Linux Container)に関する詳細な記事
- [Dockerイメージの理解とコンテナのライフサイクル][7]
  - わかりやすさとスライドの質の高さがすごいです

[1]: https://www.docker.com/
[2]: http://www.atmarkit.co.jp/ait/articles/1405/16/news032.html
[3]: https://rimuru.lunanet.gr.jp/notes/post/how-to-root-from-inside-container/
[4]: https://www.docker.com/products/docker-toolbox#/tutorials
[5]: http://gihyo.jp/admin/serial/01/linux_containers/0003
[6]: http://gihyo.jp/admin/serial/01/linux_containers
[7]: http://www.slideshare.net/zembutsu/docker-images-containers-and-lifecycle
[8]: https://ja.wikipedia.org/wiki/%E3%83%8F%E3%82%A4%E3%83%91%E3%83%BC%E3%83%90%E3%82%A4%E3%82%B6
[9]: https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/
[10]: http://gihyo.jp/admin/serial/01/linux_containers/0018
