---
layout: post
title: Btrfs on NVMe SSD のセットアップ
tags: "btrfs, linux"
comments: true
---

最近 DELL XPS13 (9380) を購入し Arch Linux をインストールする際にファイルシステムとして Btrfs を採用しました。初めての作業でけっこう悩む部分もあったのでセットアップ例の 1 つとして記録を残します。

### 環境

PC:

- DELL XPS13 9380 (メモリ 16GB だとスペックは挙げるまでもなく下記の一種のみ?)
  - cpu: Intel(R) Core(TM) i7-8565U CPU @ 1.80GHz
  - memory: 16GB LPDDR3 2133MHz
  - disk: PC401 NVMe SK hynix 512 GB

Arch Linux & Linux Kernel バージョン:

```shell
$ uname -a
... 5.0.10-arch1-1-ARCH ...
```

### 参考元

Btrfs 自体開発がまだ活発なのでここで書いた情報もすぐに古くなるかもしれません。
実際に Btrfs のセットアップを行うなら以下の 2 つは眺めておくとよさそうです。

- [Btrfs - ArchWiki][3]
- [btrfs Wiki][4]

### Btrfs の特徴

- Cow (Copy on Write)
- Compression
  - 透過的なファイル圧縮のサポート
- Subvolume
  - マウント可能な POSIX filetree とみなせる
- Snapshot
  - ある時点での subvolume の保存を CoW を利用して効率よく実現
  - snapshot もまた subvolume
  - subvolume を再帰的には対象にしない

### セットアップ手順

Arch Linux インストールの大まかな手順については [インストールガイド - ArchWiki][5] と同じです。
以下では Btrfs に関する部分だけ取り上げます。

### パーティションのフォーマット

`mkfs.btrfs` を使用します。

```shell
$ lsblk # check name of the target partition
$ mkfs.btrfs /dev/nvme0n1pX # specify the name
```

### パーティションのマウント

自分が初めて Btrfs のセットアップをして悩んだのが

- ファイルレイアウト
- マウントオプション

でした。

#### ファイルレイアウト

ファイルレイアウトについては [Btrfs#Mounting subvolumes - ArchWiki][6] に書かれている Tip,

> ... consider creating a subvolume to house your actual data and mounting it as /.

に従うのがいいと思います。こうすることで例えばスナップショットの取りやすさや過去のスナップショットからの復元といった操作がやりやすくなるのだと思います。

具体的なファイルレイアウト例としては以下のものが見つかりました。

- [Snapper#推奨ファイルシステムレイアウト - ArchWiki][7]
- [SysadminGuide - btrfs Wiki][8]

実際にこの時点で設定したレイアウトでは上のようにスナップショット用の subvolume を root と同じ階層に作成したりはせず、単に @ (`/` へのマウント用) と @home (`/home/` へのマウント用) の 2 つを作成、マウントしただけにしました。

```
toplevel (subvolume id = 5)
  +-- @ (subvolume to be mounted at /)
  +-- @home (subvolume to be mounted at /home)
```

あとで出ますがスナップショット用の subvolume は [Snapper][9] 設定時に作成されるそれをただ使用しています。

#### マウントオプション

ググって他の人の設定を参考にしたり [Manpage/btrfs(5) - btrfs Wiki][10] を見ながら迷った末以下のようにしました。

```shell
$ cat /etc/fstab | grep btrfs
UUID=13f3bba6-a01e-4b0a-bb84-b2bed4bb3827	/         	btrfs     	rw,relatime,compress=zstd,ssd,space_cache,subvolid=256,subvol=/@,subvol=@	0 0
UUID=13f3bba6-a01e-4b0a-bb84-b2bed4bb3827	/home     	btrfs     	rw,relatime,compress=zstd,ssd,space_cache,subvolid=258,subvol=/@home,subvol=@home	0 0
```

Btrfs 特有のものでいうと

- compress
  - ディスク保存時の圧縮アルゴリズムの指定
- space\_cache
  - ディスクの空き領域をキャッシュしてすぐに探せるようにするみたいな?
- subvolid
  - subvolume に振られる id
  - subvol と併用する必要は無い? 併用する場合両者が指す subvolume が異なるとエラーらしい
- subvol
  - subvolume の toplevel から見た path
  - これ 2 回指定されているのは謎
  - 後述の mount 実行後に genfstab しているだけなので変なことしているつもりは無いが...?

subvolid, subvol あたりは `btrfs subvolume list` 等で確認できます。

```shell
$ btrfs subvolume list -a -p .
ID 256 gen 3280 parent 5 top level 5 path <FS_TREE>/@
ID 258 gen 3280 parent 5 top level 5 path <FS_TREE>/@home
...
```

あと discard を指定する例もあるのですが、NVMe の場合入れるべきでない (参考元: [NVMe - ArchWiki][13]) とのことで含めませんでした。

ということで以上を踏まえて実際の作業は以下のようになりました。これをあとで `genfstab` すれば上述したものができます。

```shell
 $ mount -o defaults,relatime,ssd,compress=zstd /dev/nvme0n1pX /mnt
 $ cd /mnt
 $ btrfs subvolume create @ # to be mounted at /
 $ btrfs subvolume create @home # to be mounted at /home
 $ btrfs subvolume list -p . # to check created subvolumes
 
 $ cd /
 $ umount /mnt
 $ mount -o defaults,relatime,ssd,compress=zstd,subvol=@ /dev/nvme0n1pX /mnt
 
 $ cd /mnt
 $ mkdir -p home
 $ mount -o defaults,relatime,ssd,compress=zstd,subvol=@home /dev/nvme0n1pX home
```

### Snapper による自動スナップショット取得

Btrfs でスナップショットを取得するのは簡単で `btrfs subvolume snapshot` コマンドを使用します。

ただ運用的には定期的に取得したい、一定以上古いものは消したいといった要求があると思います。そうした用途に使用できるツールとして [Snapper][9] というのがあります。

Snapper 用の設定を @, @home subvolume に作成します。

```shell
 $ snapper -c root create-config / # for /
 $ snapper -c home create-config /home # for /home
```

これにより `/.snapshots`, `/home/.snapshots` にスナップショット用の subvolume がそれぞれ作成されます。

スナップショットを取得するタイミングや保持する個数は `/etc/snapper/configs/` 以下に作成される設定ファイルで調整できます。

```
$ grep "timeline cleanup" -A 7 /etc/snapper/configs/home
# limits for timeline cleanup
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="0"
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_YEARLY="0"
```

自動スナップショットを有効化するには対応する systemd ユニットを有効化します。

```shell
$ systemctl enable snapper-timeline.timer snapper-cleanup.timer
```

### あとから subvolume にしたいとき

あるディレクトリをスナップショットから除きたいという場合 subvolume を作成します。これは Btrfs では subvolume を再帰的にスナップショットを取らないという挙動になっているためです。

除外したいディレクトリの候補になるのは例えば

- キャッシュのような保存する必要がないもの
- `/var/log`
  - 何らかのトラブルで `/` を過去のスナップショットに戻したときにログを維持する方がトラブルシュートしやすいかも

といったものです。一例として [こちら][11]。

はじめから除外したいディレクトリがわかっているならば上でパーティションのマウントを行っているときに一緒に `btrfs subvolume create` すればいいのですが、既存のディレクトリを subvolume 化したい場合は少し手間がかかります。

[btrfs-subvolume-questions][12] によれば Btrfs においてお行儀のよいやり方は

1. subvolume にしたいディレクトリが含まれる subvolume の snapshot を作成する (Btrfs においては snapshot も subvolume)
2. 作成した snapshot から目的のディレクトリ外のものを消す
3. 目的のディレクトリを消し作成した snapshot をそこに mv する

という感じのようです。例えば `/var/log` を subvolume 化するには以下のようにしました。

```shell
$ btrfs subvolume snapshot / /var/log.new # temporal name
$ cd /var/log.new
... (remove other than /var/log.new/var/log)
$ mv var/log/* . && rm -rf var
# この時点で /var/log.new と /var/log の中身が (作業中に発生しうる /var/log への変更を無視すれば) 一致
$ cd /var
$ rm -rf log
$ mv log.new log
```

`/var/log` なのであまり気にしなかったですが、より丁寧にやるなら USB フラッシュドライブ等に入れたインストールメディアで起動したシステムから上の操作をやるべきだと思います。

なんにせよ手間ですし割と怖い操作なので可能ならば事前に subvolume を作成しておきたいです。

### Linux ブートオプション

ブートオプションに `rootflags=subvol=@` の追加が必要でした。

自分は systemd-boot をブートローダに使用しているので、以下のファイルを変更しました。

```shell
$ cat /boot/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=PARTUUID=<UUID> rootflags=subvol=@ rw
```

### Btrfs の注意点

- スワップファイルの対応がないのでハイバネートできない
  - 正確にいうとある (参考 [btrfsでswapfile + hibernation][1]) がまだ少し手間がかかるらしい
    - Arch Linux はもちろんすでに Linux kernel 5.0
  - 個人的にはサスペンドでいいのであまり困らない
    - 両者の違いについてはこちら: [サスペンドとハイバネート - ArchWiki][2]

[1]: https://qiita.com/tmsn/items/41bf294728a4d2c0d4b8
[2]: https://wiki.archlinux.jp/index.php/%E3%82%B5%E3%82%B9%E3%83%9A%E3%83%B3%E3%83%89%E3%81%A8%E3%83%8F%E3%82%A4%E3%83%90%E3%83%8D%E3%83%BC%E3%83%88
[3]: https://wiki.archlinux.org/index.php/Btrfs
[4]: https://btrfs.wiki.kernel.org/index.php/Main_Page#Guides_and_usage_information
[5]: https://wiki.archlinux.jp/index.php/%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%82%AC%E3%82%A4%E3%83%89
[6]: https://wiki.archlinux.org/index.php/Btrfs#Mounting_subvolumes
[7]: https://wiki.archlinux.jp/index.php/Snapper#.E6.8E.A8.E5.A5.A8.E3.83.95.E3.82.A1.E3.82.A4.E3.83.AB.E3.82.B7.E3.82.B9.E3.83.86.E3.83.A0.E3.83.AC.E3.82.A4.E3.82.A2.E3.82.A6.E3.83.88
[8]: https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Subvolumes
[9]: https://wiki.archlinux.jp/index.php/Snapper
[10]: https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs(5)
[11]: https://www.ncaq.net/2019/01/28/13/37/05/
[12]: https://unix.stackexchange.com/questions/63606/btrfs-subvolume-questions
[13]: https://wiki.archlinux.jp/index.php/%E3%82%BD%E3%83%AA%E3%83%83%E3%83%89%E3%82%B9%E3%83%86%E3%83%BC%E3%83%88%E3%83%89%E3%83%A9%E3%82%A4%E3%83%96/NVMe
