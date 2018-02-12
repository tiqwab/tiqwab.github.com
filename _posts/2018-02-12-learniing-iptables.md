---
layout: post
title: Docker の iptables 設定
tags: "docker, iptables"
comments: true
---

Docker のネットワークまわりの理解を深める一環として iptables について整理します。
全体として [Iptables Tutorial][2] がとても参考になりました。

1. [iptables とは](#iptables-general)
2. [iptables の構成要素](#iptables-concept)
3. [Docker デーモン起動時の iptables](#iptables-docker-started)
4. [Docker コンテナ起動時の iptables](#iptables-container-started)

環境:

```
# OS
$ cat /proc/version 
Linux version 4.14.13-1-ARCH (builduser@heftig-32336) (gcc version 7.2.1 20171224 (GCC)) #1 SMP PREEMPT Wed Jan 10 11:14:50 UTC 2018

# iptables
$ iptables --version
iptables v1.6.1

# Docker
$ docker version
Client:
 Version:	18.01.0-ce
 API version:	1.35
 Go version:	go1.9.2
 Git commit:	03596f51b1
 Built:	Sun Jan 14 23:10:39 2018
 OS/Arch:	linux/amd64
 Experimental:	false
 Orchestrator:	swarm

Server:
 Engine:
  Version:	18.01.0-ce
  API version:	1.35 (minimum version 1.12)
  Go version:	go1.9.2
  Git commit:	03596f51b1
  Built:	Sun Jan 14 23:11:14 2018
  OS/Arch:	linux/amd64
  Experimental:	false
```

---

<div id="iptables-general" />

### iptables とは

[iptables][1] は Linux でファイアウォールやルータ設定を行うために使用するツールであり、Netfilter というパケット処理用モジュールのフロントエンドとして働くものです。

iptables に変わるものとして nftables や CentOS 7 以降では firewalld が使用されたりもしますが、まだ一般的には iptables が主流なのかなと思います (例えば Docker もまだ nftables サポートは [feature request][9] 状態みたいですし)。

<div id="iptables-concept" />

### iptables の構成要素

iptables の理解に重要な概念としてルール、チェイン、テーブルがあります。

ルールとは iptables におけるファイアウォールの具体的な設定です。例えば「ループバックインタフェースから送信されたパケットならば全て許可する」といったものです。ルールは

- プロトコル (prot)
- 受信元インタフェース (in)
- 送信先インタフェース (out)
- 受信元 IP (の範囲) (source)
- 送信先 IP (の範囲) (destination)
- パケットに対するアクション (e.g. 許可する、許可しない) (target)

といったもので構成されます。上で例に挙げたルールの場合はそれぞれ

- prot: all
- in: lo
- out: \*
- source: 0.0.0.0/0
- destination: 0.0.0.0/0
- target: ACCEPT

のようになります。言い換えればルールはパケットに対するアクション (target) とそれを適用するための条件をセットにしたものです。

ルール単体でも簡単なパケットの制御はできそうですが、iptables ではテーブルとチェインという概念により更に複雑な設定が可能になります。テーブルは iptables を構成する最も大きな括りであり、filter、nat、raw、mangle、security の 5 つが存在します。よく使用されるのは filter と nat で、それぞれパケットフィルタリング、NAT (Network Address Translation) ルールを定義するためのものです。テーブルは複数のチェインを持ち、チェインによりどのタイミングでパケットにルールを適用するかが決定されます。テーブル、チェイン、ルールの構造は以下のようにまとめられます。

<img
  src="/images/learn-iptables/iptables-structure.png"
  title="iptables-structure"
  alt="iptables structure"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

実際に各チェイン、ルールがどのように適用されるかは以下のフローチャートでまとめられます。ここでは ArchWiki の [iptables][1] に倣って filter, nat テーブルに関連するチェインのみを取り上げています。

<img
  src="/images/learn-iptables/iptables.png"
  title="flow-chart-of-iptables"
  alt="flow chart of iptables"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

パケットはこのフローチャートにそって各チェインを通過し、チェインで定義されたルールが順番に評価、適用されます。具体的にはあるマシン上で考えられるパケットの送受信 3 パターンについてはそれぞれ以下のようになります。

- (1) 自分から外部にパケットを送信する (下図 blue)
- (2) 外部からパケットを受信しかつそれが自分宛て (下図 red)
- (3) 外部からパケットを受信したが自分宛てではなく外部へフォワーディングする (下図 green)

<img
  src="/images/learn-iptables/iptables-3way.png"
  title="flow-chart-of-iptables-with-three-way"
  alt="flow chart of iptables specifying three way of packets"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

ルールを設定する際には、この図を見ることでどのチェインに追加するべきかがわかります。例えば自分宛てのパケットに対してフィルタリングをかけたい際には filter テーブルの INPUT にルールを設定します。

iptables に関しては以下の資料を参考にしました。

- [iptables - ArchWiki][1]
- [Traversing of tables and chains - Iptables Tutorial][3]

<div id="iptables-docker-started" />

### Docker デーモン起動時の iptables

Docker ではコンテナ間、コンテナホスト間、コンテナから外部ネットワーク、といった通信の制御を iptables で行っています。Docker デーモン起動前後に iptables を確認すると filter, nat テーブルにいくつか新規にルールが定義されることがわかります。

#### filter テーブル

Docker デーモン起動後の filter テーブルでは主に FORWARD チェインへルールが定義されます。

```
# Show filter table after starting Docker daemon.
$ iptables -nvL --line-number
Chain INPUT (policy ACCEPT 309 packets, 49021 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num pkts bytes target     prot opt in     out     source               destination         
1      0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
2      0     0 DOCKER-ISOLATION  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
3      0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
4      0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
5      0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
6      0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 277 packets, 21141 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER (1 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER-ISOLATION (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           
```

FORWARD チェインに定義されたルールを上から順番に見ていきます。ホスト上の FORWARD チェインに定義されるようなルールなので、主に管理するコンテナにどのような通信を許可するか、を定めたものになります。

- num 1: まずは必ず DOCKER-USER チェインにいく
  - ターゲットで Docker が追加したチェインが指定されているので、そちらでルールの評価が行われる
  - 今回の場合 DOCKER-USER チェインはすぐに元のチェインに戻る RETURN ルールのみが定義されている
    - つまり何もしない
  - もし Docker が追加したルールを上書きするような挙動を追加したい場合、ここに定義するのがよい
- num 2: DOCKER-ISOLATION チェインにいく
  - Docker ネットワーク間の通信を制限するためのルールがここに定義される
  - いまは docker0 ブリッジネットワークしか存在しないので何もしない
- num 3: docker0 インタフェース行きで ctstate RELATED, ESTABLISHED なパケットは許可
  - ctstate については後述
- num 4: それ以外の docker0 インタフェース行きは DOCKER チェインで判断
  - いまはここでも何もしない
- num 5: docker0 インタフェースから来てそれ以外のインタフェースに行くパケットは許可
  - 例えばコンテナから外部ネットワークに行くようなもの
  - ただし異なる Docker ネットワークに行くようなものは DOCKER-ISOLATION チェインで制限される
- num 6: docker0 インタフェースから来て docker0 インタフェースに行くパケットは許可する
  - いわばコンテナ間通信
- 上記のいずれにも当てはまらない場合、チェインで定義したデフォルトポリシーが適用される
  - ここでは DROP

もう少しルール 3 の定義に含まれる ctstate を見ていきます。ctstate は iptables 拡張の一つ conntrack により提供される (というより Linux カーネルの一部？) 機能で、通信状態による条件付けを提供します。ctstate が管理する状態には NEW, ESTABLISHED, RELATED, INVALID といったものがあります。

iptables がどのように接続状態を判断するのかですが、どうやら主要な通信プロトコル (e.g. TCP, UDP, and ICMP) についてはこの状態なら ctstate は NEW, ESTABLISHED, あるいは RELATED であるという定義をそれぞれで与えているようです。それ以外のプロコルに対しても初回の通信なのか両方向で送受信したものかで ctstate を設定するような挙動になるとされています。

ここでいう接続状態については `/proc/net/nf_conntrack` で確認できます (Linux OS のバージョンによって多少パスが異なりそう)。iptables は nat テーブルの PREROUTING, OUTPUT チェインで適宜この情報の更新を行います。

```
# To see ICMP connetion in conntrack
$ ping -c 3 google.co.jp
PING google.co.jp (216.58.220.227) 56(84) bytes of data.
64 bytes from nrt13s37-in-f3.1e100.net (216.58.220.227): icmp_seq=1 ttl=54 time=27.4 ms
...

$ cat /proc/net/nf_conntrack | grep icmp
ipv4     2 icmp     1 27 src=192.168.11.10 dst=216.58.220.227 type=8 code=0 id=2090 src=216.58.220.227 dst=192.168.11.10 type=0 code=0 id=2090 mark=0 zone=0 use=2
```

上のルールでは ctstate が ESTABLISHED, RELATED のときに ACCEPT するとしていますが、これは既に自分が知っている接続に関するものなので許可してもいいよ、ということを定義するのだと考えれば良さそうです。

ctstate については以下の内容を参考にしました。

- [iptables を使用してファイアウォール機能を稼働させ、セキュリティーを制御する][4]
- [The state machine - Iptables Tutorial][5]

#### nat テーブル

続いて Docker が nat テーブルに定義するルールについて見ていきます。

```
# Show nat table
$ iptables -nvL -t nat
Chain PREROUTING (policy ACCEPT 34 packets, 3186 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   17  1007 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 84 packets, 5316 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 84 packets, 5316 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0 
```

- PREROUTING チェイン
  - ADDRTYPE match dst-type LOCAL のときに DOCKER チェインへいく
    - addrtype については後述
- OUTPUT チェイン
  - PREROUTING チェインと同様
  - ただしパケットの行き先がループバックアドレスならば該当しない
- POSTROUTING チェイン
  - 172.17.0.0/16 は docker0 ブリッジネットワークのセグメント
  - このネットワークから外部のネットワークに行くものに対し MASQUERADE する
    - MASQUERADE については後述

ADDRTYPE についてですが、これはカーネルがパケットの分類に使用する情報であり、例えば LOCAL, UNICAST, BLOADCAST といったタイプがあるようです。調べたり触っている感じ LOCAL というのはループバックアドレスのような自身から自身への通信のようなものなのかなと思うのですがちょっと自信がないです。また各タイプの意味はレイヤー 3 プロトコルによって異なるらしいのでそこも注意が必要になります。

ADDRTYPE の概要は以下が参考になりました。

- [Addrtype match - Iptables Tutorial][6]

MASQUERADE (IP マスカレード) ですがこれは Linux で実装された場合の名称で、一般的には NAPT (Network Address and Port Translation) と呼ばれます。NAT には送信元アドレスを変換する SNAT, 送信先を返還する DNAT がありますが、マスカレードは SNAT の特殊なケースになります。

iptables 上では SNAT (MASQUERADE) は nat テーブルの POSTROUTING チェイン、DNAT は PREROUTING チェインで定義します。

NAT, NAPT については以下を参考にしました。

- [NAT と SNAT と DNAT][7]
- [パケットの料理法の解説][8]

<div id="iptables-container-started" />

### Docker コンテナ起動時の iptables

次にコンテナを起動した際の iptables への変化を見ていきます。ただ単純にコンテナを起動するだけだと特に変わりがないので、 `-p` オプション設定時の挙動を見ることにします。

```
# Run container with -p option
$ docker container run -d -p 80:80 dockersamples/static-site
$ docker container ls -a
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                         NAMES
4be67466de54        dockersamples/static-site   "/bin/sh -c 'cd /usr…"   17 seconds ago      Up 16 seconds       0.0.0.0:80->80/tcp, 443/tcp   mystifying_poincare
```

これによりホスト上のポート 80 にアクセスすることでコンテナ上ポート 80 で公開している Web サーバにアクセスできます。

このようにコンテナを起動すると、filter テーブルの DOCKER チェインに新規ルールが追加されます。

```
$ iptables -nvL
...
Chain DOCKER (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     tcp  --  !docker0 docker0  0.0.0.0/0            172.17.0.2           tcp dpt:80
...
```

このルールでは「TCP で docker0 以外のインタフェースから docker0 インタフェースへ行くパケットで、172.17.0.2:80 が送信先のもの」が許可されています。

filter テーブルの DOCKER チェインは FORWARD チェイン内

```
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0   
```

のルールで参照されるチェインです。

コンテナ起動前は docker0 ブリッジネットワーク外部から docker0 行きのパケットは ctstate RELATED, ESTABLISHED なものしか許可していなかったので、コンテナ 80 番ポートへのアクセスを許可するにはこのようなルールが必要になることがわかります。

また nat テーブルにも POSTROUTING, DOCKER チェインに 2 つのルールが追加されます。

```
$ iptables -nvL -t nat
Chain PREROUTING (policy ACCEPT 30 packets, 1761 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 1265 74405 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 226 packets, 14411 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 2023  121K DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 253 packets, 15551 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 1021 61260 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
    0     0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:80 <- 増えた

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
    3   180 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.2:80 ← 増えた
```

このうち DOCKER チェインに追加された DNAT ルールが実際にコンテナ 80 番ポートにポートフォワーディングするルールになります。ルール自体を見ると TCP で送信先ポートが 80 というけっこうざっくりしたルールに見えるのですが、これは DOCKER チェインを参照するルール側で条件やタイミングを絞っているからなのかなと思います。

POSTROUTING チェインの方に追加された方は具体的にどの場合に対応するのかよくわかりません。見た感じ Web サーバを動かすコンテナ (IP 172.17.0.2 が割り当てられているもの) 自身からホストのポート 80 を叩きに来た場合なんかが当てはまるのかなと思ったのですが、実際試してみると違いそうでした。

[1]: https://wiki.archlinux.jp/index.php/Iptables
[2]: https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html
[3]: https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#TRAVERSINGOFTABLES
[4]: https://www.ibm.com/developerworks/jp/opensource/library/os-iptables/
[5]: https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE
[6]: https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#ADDRTYPEMATCH
[7]: http://yoru9zine.hatenablog.com/entry/2015/12/19/072304
[8]: https://linuxjf.osdn.jp/JFdocs/NAT-HOWTO-6.html
[9]: https://github.com/moby/moby/issues/26824
