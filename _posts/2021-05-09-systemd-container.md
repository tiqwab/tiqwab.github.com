---
layout: post
title: cgroup v2 環境下で systemd を Docker コンテナ内で動かしたい
tags: "cgroup,systemd,docker"
comments: true
---

### systemd を Docker コンテナ内で動かしたい理由

あまり本番環境へデプロイするコンテナで systemd を動かすというのは無いのですが、開発環境でちょっと何かミドルウェアを動かしたい、かつそのミドルウェアにそこまで詳しくないといった状況のときに、「パッケージのインストール -> systemctl start ...」でコンテナ起動ができると楽だなというのがモチベーションです。

便宜上、この記事では以降、こうしたコンテナを systemd container と呼ぶことにします。

ちなみに [Running systemd in a non-privileged container][2] という記事 (2016 年なので少し古い) を見ると systemd 開発者の中にはむしろコンテナでも systemd を PID=1 として実行するべきと考える人もいるみたいです。

### 以前理解していた方法

自分が以前 systemd container を動かすために使っていた方法は、上でも触れた [Running systemd in a non-privileged container][2] 内で紹介されているものでした。例えば Apache HTTP Server を動かすためには Dockerfile として以下のようなものを用意し、

```
FROM         fedora:24
ENV container docker
RUN dnf -y install httpd; dnf clean all; systemctl enable httpd
STOPSIGNAL SIGRTMIN+3
EXPOSE 80
CMD [ "/sbin/init" ]
```

Docker イメージをビルド後、以下のコマンドでコンテナ起動するというものでした。

```
# 記事内からの引用。ただし自分の環境では --privileged も必要だった
$ docker run -d --tmpfs /tmp --tmpfs /run -v /sys/fs/cgroup:/sys/fs/cgroup:ro httpd
```

このように cgroup v1 環境下では `/sys/fs/cgroup` の read-only なマウントが必要でした。実際には `/sys/fs/cgroup/systemd` の read-write 許可したマウントも必要なはずですが、cgroup v1 を利用するシステムでは以下のようなマウントが設定されていること、[docker/for-linux #788][4] で言及されるようにネストされたマウント元のパーミッションは維持されること (いまの場合 `/sys/fs/cgroup/systemd` には ro 指定が反映されないということ) から Docker には `/sys/fs/cgroup` のマウントのみ伝えればいいです。

```
$ mount -l | grep cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,seclabel,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
...
```

### cgroup v2 環境下での方法

Docker バージョン 20.10 からは cgroup v2 に対応したらしく、そのおかげか cgroup v2 を利用するシステムでは `/sys/fs/cgroup` のマウントなしにコンテナ起動ができるようになりました。というよりマウントすると起動できません。

```
$ docker run -d --tmpfs /tmp --tmpfs /run --privileged httpd
```

自環境が cgroup v2 かどうか確認する方法は色々あると思いますが、例えば mount コマンドの出力から判断できます。

```
$ mount -l | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot)
```

一点注意点としては起動するコンテナのベースイメージもある程度新しくないといけない (恐らく systemd のバージョンの問題) ので、例えば上の Dockerfile で使用した `fedora:24` や `centos:7` ではなく、`fedora:32` や `centos:8` を利用する必要があります。

今回ちょこちょこ調べていてはじめて知りましたが、Docker は cgroup の制御を systemd と連携して行なうようです。systemd-cgls により表示できる cgroup の構造を見ると Docker が動的に作成した scope が確認できます。

```
$ sudo systemd-cgls
Control group /:
-.slice
├─user.slice
...
├─init.scope
│ └─1 /sbin/init
└─system.slice
...
  ├─docker-9ccfea8d3468bb79584f45815936551a61900cadd66eb13ef66928789067b044.scope
  │ ├─init.scope
  │ │ └─76397 /sbin/init
  │ └─system.slice
  │   ├─dbus-broker.service
  │   │ ├─76462 /usr/bin/dbus-broker-launch --scope system --audit
  │   │ └─76463 dbus-broker --log 4 --controller 9 --machine-id 7241c28441a64de78739c60b92fecab7 --max-bytes 5368709>
  │   ├─systemd-homed.service
  │   │ └─76460 /usr/lib/systemd/systemd-homed
  │   ├─systemd-journald.service
  │   │ └─76446 /usr/lib/systemd/systemd-journald
  │   ├─httpd.service
  │   │ ├─76465 /usr/sbin/httpd -DFOREGROUND
  │   │ ├─76467 /usr/sbin/httpd -DFOREGROUND
  │   │ ├─76468 /usr/sbin/httpd -DFOREGROUND
  │   │ ├─76469 /usr/sbin/httpd -DFOREGROUND
  │   │ └─76471 /usr/sbin/httpd -DFOREGROUND
  │   ├─systemd-oomd.service
  │   │ └─76456 /usr/lib/systemd/systemd-oomd
  │   ├─systemd-resolved.service
  │   │ └─76457 /usr/lib/systemd/systemd-resolved
  │   ├─system-getty.slice
  │   │ └─getty@tty1.service
  │   │   └─76466 /sbin/agetty -o -p -- \u --noclear tty1 linux
  │   └─systemd-logind.service
  │     └─76461 /usr/lib/systemd/systemd-logind
...
```

cgroup v2 で何が変わったのか、また systemd が Docker のような container manager に対して何を提供できるのかといったことについては以下の記事が参考になりそうです (流し読みしかしていないですが)。

- [Control Group APIs and Delegation][5]
- [The New Control Group Interfaces][6]

[1]: https://www.docker.com/blog/introducing-docker-engine-20-10/
[2]: https://developers.redhat.com/blog/2016/09/13/running-systemd-in-a-non-privileged-container/
[3]: https://github.com/docker/for-linux/issues/219
[4]: https://github.com/docker/for-linux/issues/788
[5]: https://systemd.io/CGROUP_DELEGATION/
[6]: https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/
