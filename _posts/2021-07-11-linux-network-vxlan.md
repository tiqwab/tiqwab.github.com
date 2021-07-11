---
layout: post
title: Linux の仮想ネットワーク機能を利用した VXLAN 体験
tags: "linux,vxlan"
comments: true
---

Docker や Kubernetes 等のネットワーク周りでたまに聞く [VXLAN][5] というプロトコルについて、どうやら Linux の仮想ネットワーク機能を使って試してみることができそうなのでやってみました。

### Linux bridge を介したコンテナ間の通信

VXLAN を実際に作成し始める前に iproute2 や Linux bridge に親しむため、以下のような 2 Docker コンテナ間を bridge でつないだ構成のネットワークを作成してみます。

<figure>
  <img
    src="/images/linux-network-vxlan/figure1.png"
    title="network-linux-bridge"
    alt="network-linux-bridge"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>


#### Docker コンテナの用意

```
$ sudo docker container run -d --name C0 --net=none debian tail -f /dev/null
$ sudo docker container run -d --name C1 --net=none debian tail -f /dev/null
```

`--net=none` を指定することで lo しかネットワークインタフェースを持たないコンテナを作成することができます。今回はこれを仮想ネットワーク上の Linux ホストだと想定して使用します。

それぞれのコンテナに紐づく netns を記録しておきます。
これらは Docker が作成するもので `/var/run/docker/netns/` 以下に用意されます。
後半のシンボリックリンク作成は iproute2 ツールでは `/var/run/netns/` 以下にファイルが存在することを期待するのでその調整のためです。

{% raw %}
```
$ netns_c0=$(sudo docker container inspect C0 -f "{{.NetworkSettings.SandboxKey}}")
$ netns_c1=$(sudo docker container inspect C1 -f "{{.NetworkSettings.SandboxKey}}")
$ sudo ln -sf $netns_c0 /var/run/netns/${netns_c0##*/}
$ sudo ln -sf $netns_c1 /var/run/netns/${netns_c1##*/}
```
{% endraw %}

#### Linux bridge 作成

```
$ sudo ip link add dev br0 type bridge
$ sudo ip addr add dev br0 192.168.0.1/24
$ sudo ip link set br0 up
```

#### veth の作成

作成した br0 と各コンテナをつなぐために veth の作成、設定を行ないます。veth はペア間で通信が可能な仮想ネットワークインタフェースであり、ここでは netns を横断したインタフェース間の接続のために利用します。

```
# 2 つの veth を作成
$ sudo ip link add dev veth0 type veth peer name veth1
$ sudo ip link add dev veth2 type veth peer name veth3

# 各 veth の片ペアをコンテナ内に持っていく
$ sudo ip link set dev veth0 netns $netns_c0
$ sudo ip link set dev veth2 netns $netns_c1

# br0 につなぐ
$ sudo ip link set veth1 master br0
$ sudo ip link set veth3 master br0

# コンテナ内に持っていった veth を eth0 として MAC, IP アドレスを割り当てる
# ip netns exec ... により指定した netns 内での操作が可能
$ sudo ip netns exec ${netns_c0##*/} ip link set dev veth0 name eth0 address 02:42:c0:a8:00:02
$ sudo ip netns exec ${netns_c0##*/} ip addr add dev eth0 192.168.0.2/24
$ sudo ip netns exec ${netns_c1##*/} ip link set dev veth2 name eth0 address 02:42:c0:a8:00:03
$ sudo ip netns exec ${netns_c1##*/} ip addr add dev eth0 192.168.0.3/24
$ sudo ip netns exec ${netns_c0##*/} ip link set dev eth0 up
$ sudo ip netns exec ${netns_c1##*/} ip link set dev eth0 up
```

これにより C0, C1 間で通信が可能になります。

```
$ sudo docker container exec C0 ping -c 1 192.168.0.3
```

### 2 台の Linux ホスト間での VXLAN

ここから実際に VXLAN 通信を試してみます。

本節の内容の多くは [Deep dive into Docker Overlay Networks][1] を参考にしています。
この記事自体は Docker overlay network driver の動作を解説したものですが、その中心に VXLAN があるので結果として VXLAN の理解という観点でも有用なものになっています。

VXLAN は L3 ネットワーク上に仮想的な L2 ネットワークを構築するトンネリングプロトコルであり、そのトンネリングのエンドポイントは VTEP (VXLAN Tunneling End Point) と呼ばれます。VTEP はお互いの存在やその配下のノードの情報をやり取りする必要があり、その実現方法はいくつか考えられますが (ref. [VXLAN & Linux][2])、 ここではひとまず静的なネットワークを扱うという前提で単純に手動で設定を行なうこととします。

ここで構築するネットワークの概要図です。
node0, node1 の Linux ホスト間で VXLAN を構成します。
それにより C0, C1 コンテナがお互いにあたかも同一 L2 ネットワークに属しているかのように見せます。

<figure>
  <img
    src="/images/linux-network-vxlan/figure2.png"
    title="network-linux-vxlan-two-nodes"
    alt="network-linux-vxlan-two-nodes"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>

node0, node1 で行なう操作はほぼ同一なので以下のようなシェルスクリプトを用意しました。

{% raw %}
```
#!/bin/bash

set -eux -o pipefail

BRIDGE_IP=$1
CONTAINER_NAME=$2
CONTAINER_MAC=$3
CONTAINER_IP=$4

# ネットワーク上のホストとして Docker コンテナを起動
sudo docker container run -d --name $CONTAINER_NAME --net=none debian tail -f /dev/null
netns_ctn=$(sudo docker container inspect $CONTAINER_NAME -f "{{.NetworkSettings.SandboxKey}}")
sudo ln -sf $netns_ctn /var/run/netns/${netns_ctn##*/}

# overlay 用の netns を作成
sudo ip netns add overns

# Linux bridge の作成 (in overns)
sudo ip netns exec overns ip link add dev br0 type bridge
sudo ip netns exec overns ip addr add dev br0 $BRIDGE_IP/24
sudo ip netns exec overns ip link set br0 up

# vxlan interface の作成
# 作成コマンドは https://blog.revolve.team/2017/08/20/deep-dive-3-into-docker-overlay-networks-part-3/ に従った
# proxy は vxlan interface が ARP リクエストに答えるためのもの
# leaning は vxlan interface が受信したフレームをもとに bridge fdb を更新するためのもの
# また vxlan interface は直接 overns 内で作成するのではなくホストで作成してから持っていく必要があるとのこと
sudo ip link add dev vxlan0 type vxlan id 42 proxy learning dstport 4789
sudo ip link set vxlan0 netns overns
sudo ip netns exec overns ip link set vxlan0 master br0
sudo ip netns exec overns ip link set vxlan0 up

# Docker コンテナを br0 に接続
sudo ip link add dev veth0 mtu 1450 type veth peer name veth1 mtu 1450
sudo ip link set dev veth0 netns overns
sudo ip netns exec overns ip link set veth0 master br0
sudo ip netns exec overns ip link set veth0 up
sudo ip link set dev veth1 netns ${netns_ctn##*/}
sudo ip netns exec ${netns_ctn##*/} ip link set dev veth1 name eth0 address $CONTAINER_MAC
sudo ip netns exec ${netns_ctn##*/} ip addr add dev eth0 $CONTAINER_IP/24
sudo ip netns exec ${netns_ctn##*/} ip link set dev eth0 up
```
{% endraw %}

```
# on node0
$ ./vxlan_setup.sh 192.168.0.1 C0 02:42:c0:a8:00:2 192.168.0.2

# on node1
$ ./vxlan_setup.sh 192.168.0.11 C1 02:42:c0:a8:00:12 192.168.0.12
```

この時点だと

- C0, C1 コンテナではお互いの IP はわかっても MAC アドレスがわからない
- VTEP はお互いの存在を知らないので受け取ったパケットをトンネリングできない

ので、その情報を overns netns 内の ARP テーブル、FDB に追加します。

```
# on node0
$ sudo ip netns exec overns ip neighbor add 192.168.0.12 lladdr 02:42:c0:a8:00:12 dev vxlan0
$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:12 dev vxlan0 self dst 172.31.29.77 vni 42 port 4789

# on node1
$ sudo ip netns exec overns ip neighbor add 192.168.0.2 lladdr 02:42:c0:a8:00:2 dev vxlan0
$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:2 dev vxlan0 self dst 172.31.18.142 vni 42 port 4789
```

FDB というのは個人的にはあまり馴染みがないものなのですが、物理スイッチでいうと受信フレームの転送先ポートを決定するために使用されるデータベースを指すようです。ここでもやはり同様に Linux bridge が受信フレームの MAC アドレスからどのポートに転送するかを決定するために使用されるテーブルということだと思います。

例えば上の node0 の例だと 02:42:c0:a8:00:12 を送信先 MAC アドレスとするフレームを受け取ったら vxlan0 インタフェースから 172.31.29.77 宛にパケットを転送するべし... と解釈できます。

ARP, FDB 情報を追加したので、これにより C0, C1 間で通信が可能になります (なお VXLAN 通信に UDP port 4789 を利用しているのでそこを空けておく必要がある)。

```
# on node0
$ sudo docker container exec C0 ping -c 1 192.168.0.12
PING 192.168.0.12 (192.168.0.12) 56(84) bytes of data.
64 bytes from 192.168.0.12: icmp_seq=1 ttl=64 time=0.593 ms
```

また tcpdump でパケットキャプチャすることで実際に VXLAN パケットが流れていることも確認できます。

```
# on node0
$ sudo tcpdump -pni eth0 "port 4789"
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
03:20:31.449219 IP 172.31.18.142.46464 > 172.31.29.77.4789: VXLAN, flags [I] (0x08), vni 42
IP 192.168.0.2 > 192.168.0.12: ICMP echo request, id 55, seq 1, length 64
03:20:31.449752 IP 172.31.29.77.40025 > 172.31.18.142.4789: VXLAN, flags [I] (0x08), vni 42
IP 192.168.0.12 > 192.168.0.2: ICMP echo reply, id 55, seq 1, length 64
```

参考までに node0 の overns 内の各ネットワークインタフェース、ARP, FDB 情報を載せておきます。

```
$ sudo ip netns exec overns ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether c6:7e:1c:8a:1e:60 brd ff:ff:ff:ff:ff:ff
90: vxlan0@if90: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether e2:33:da:a2:cb:ff brd ff:ff:ff:ff:ff:ff link-netnsid 0
92: veth0@if91: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether c6:7e:1c:8a:1e:60 brd ff:ff:ff:ff:ff:ff link-netns 2c756f89987f

$ sudo ip netns exec overns ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether c6:7e:1c:8a:1e:60 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::18f6:dcff:fe08:7144/64 scope link
       valid_lft forever preferred_lft forever
90: vxlan0@if90: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UNKNOWN group default qlen 1000
    link/ether e2:33:da:a2:cb:ff brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::e033:daff:fea2:cbff/64 scope link
       valid_lft forever preferred_lft forever
92: veth0@if91: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default qlen 1000
    link/ether c6:7e:1c:8a:1e:60 brd ff:ff:ff:ff:ff:ff link-netns 2c756f89987f
    inet6 fe80::c47e:1cff:fe8a:1e60/64 scope link
       valid_lft forever preferred_lft forever

$ sudo ip netns exec overns ip neigh
192.168.0.12 dev vxlan0 lladdr 02:42:c0:a8:00:12 PERMANENT

$ sudo ip netns exec overns bridge fdb
33:33:00:00:00:01 dev br0 self permanent
01:00:5e:00:00:6a dev br0 self permanent
33:33:00:00:00:6a dev br0 self permanent
01:00:5e:00:00:01 dev br0 self permanent
33:33:ff:08:71:44 dev br0 self permanent
e2:33:da:a2:cb:ff dev vxlan0 vlan 1 master br0 permanent
e2:33:da:a2:cb:ff dev vxlan0 master br0 permanent
02:42:c0:a8:00:12 dev vxlan0 dst 172.31.29.77 link-netnsid 0 self permanent
c6:7e:1c:8a:1e:60 dev veth0 vlan 1 master br0 permanent
c6:7e:1c:8a:1e:60 dev veth0 master br0 permanent
33:33:00:00:00:01 dev veth0 self permanent
01:00:5e:00:00:01 dev veth0 self permanent
33:33:ff:8a:1e:60 dev veth0 self permanent
```

後片付けは以下のコマンドで行ないます。

```
$ sudo docker container stop C0
$ sudo docker container rm C0
$ sudo unlink /var/run/netns/2c756f89987f
$ sudo ip netns del overns
```

### 複数 VTEP での VXLAN

前節の構成の拡張で、3 台の Linux ホスト (3 VTEP) で VXLAN を組んでみます。

まず各 VTEP の情報を前節同様手動登録する方法を確認し、そのあと IP マルチキャストを利用した手動登録の必要ない手法も確認します。

構築するネットワークの概要図 (node0, node1 は前節と同じなので省略)

<figure>
  <img
    src="/images/linux-network-vxlan/figure3.png"
    title="network-linux-vxlan-three-nodes"
    alt="network-linux-vxlan-three-nodes"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>

#### 手動登録

静的なネットワークで L2 ネットワークを構成するノードに増減が無いならば、各 VTEP でお互いの情報を事前に登録するのが最も簡単な方法になります。

まず node0, node1, node2 をこれまでと同様にセットアップします。

```
# on node0
$ ./vxlan_setup.sh 192.168.0.1 C0 02:42:c0:a8:00:2 192.168.0.2
# on node1
$ ./vxlan_setup.sh 192.168.0.11 C1 02:42:c0:a8:00:12 192.168.0.12
# on node2
$ ./vxlan_setup.sh 192.168.0.21 C2 02:42:c0:a8:00:22 192.168.0.22
```

そしてお互いの情報を登録します。

```
# on node0
$ sudo ip netns exec overns ip neighbor add 192.168.0.12 lladdr 02:42:c0:a8:00:12 dev vxlan0
$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:12 dev vxlan0 self dst 172.31.29.77 vni 42 port 4789
$ sudo ip netns exec overns ip neighbor add 192.168.0.22 lladdr 02:42:c0:a8:00:22 dev vxlan0
$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:22 dev vxlan0 self dst 172.31.19.240 vni 42 port 4789

# on node1
$ sudo ip netns exec overns ip neighbor add 192.168.0.2 lladdr 02:42:c0:a8:00:2 dev vxlan0
$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:2 dev vxlan0 self dst 172.31.18.142 vni 42 port 4789
$ sudo ip netns exec overns ip neighbor add 192.168.0.22 lladdr 02:42:c0:a8:00:22 dev vxlan0
$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:22 dev vxlan0 self dst 172.31.19.240 vni 42 port 4789

# on node2
$ sudo ip netns exec overns ip neighbor add 192.168.0.2 lladdr 02:42:c0:a8:00:2 dev vxlan0
$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:2 dev vxlan0 self dst 172.31.18.142 vni 42 port 4789
$ sudo ip netns exec overns ip neighbor add 192.168.0.12 lladdr 02:42:c0:a8:00:12 dev vxlan0
$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:12 dev vxlan0 self dst 172.31.29.77 vni 42 port 4789
```

これで C0, C1, C2 コンテナ間でお互いに通信ができるようになります。

#### IP マルチキャストの利用

IP マルチキャストを利用する方法では、`vxlan_setup.sh` 中の vxlan インタフェースの作成時のオプションを変更します。
proxy と learning を外して使用する IP マルチキャストアドレスとその送受信に使用するネットワークインタフェース (?) を設定します。

```
- sudo ip link add dev vxlan0 type vxlan id 42 proxy learning dstport 4789
+ sudo ip link add vxlan0 type vxlan id 42 group 239.1.1.1 dev eth0 dstport 4789
```

これだけであとは各ノードで `vxlan_setup.sh` を実行すればコンテナ間の通信が確認できます。

```
# on node0
$ ./vxlan_setup.sh 192.168.0.1 C0 02:42:c0:a8:00:2 192.168.0.2
# on node1
$ ./vxlan_setup.sh 192.168.0.11 C1 02:42:c0:a8:00:12 192.168.0.12
# on node2
$ ./vxlan_setup.sh 192.168.0.21 C2 02:42:c0:a8:00:22 192.168.0.22
```

どのように IP マルチキャストが利用されているかを大まかにでも把握するため、C2 でのセットアップ時や通信時のパケットを node1 の eth0 でキャプチャしてみます。

```
# on node1
$ tcpdump -npi eth0
```

まず node2 での vxlan インタフェース作成時に、以下の IGMPv3 パケットが発生します。

```
07:31:41.550426 IP 172.31.19.240.59740 > 239.1.1.1.4789: VXLAN, flags [I] (0x08), vni 42
IP 192.168.0.21 > 224.0.0.22: igmp v3 report, 1 group record(s)
```

続いて C2 -> C1 へ ping を飛ばすと ARP による MAC アドレスの解決が行われます。
このとき VXLAN でカプセル化された ARP リクエストの送信先 IP には 239.1.1.1 が使われており、このことから L2 のブロードキャストを IP マルチキャストにより実現しているということがわかります。

```
# ARP
07:37:41.947768 IP 172.31.19.240.42237 > 239.1.1.1.4789: VXLAN, flags [I] (0x08), vni 42
ARP, Request who-has 192.168.0.12 tell 192.168.0.22, length 28
07:37:41.947946 ARP, Request who-has 172.31.19.240 tell 172.31.29.77, length 28
07:37:41.948130 ARP, Reply 172.31.19.240 is-at 08:00:27:27:f1:fa, length 46
07:37:41.948147 IP 172.31.29.77.48382 > 172.31.19.240.4789: VXLAN, flags [I] (0x08), vni 42
ARP, Reply 192.168.0.12 is-at 02:42:c0:a8:00:12, length 28

# ICMP
07:37:41.948318 IP 172.31.19.240.45027 > 172.31.29.77.4789: VXLAN, flags [I] (0x08), vni 42
IP 192.168.0.22 > 192.168.0.12: ICMP echo request, id 7, seq 1, length 64
07:37:41.948356 IP 172.31.29.77.48991 > 172.31.19.240.4789: VXLAN, flags [I] (0x08), vni 42
IP 192.168.0.12 > 192.168.0.22: ICMP echo reply, id 7, seq 1, length 64
```

もちろん C2 -> C0 への初回 ping でも node1 のパケットキャプチャに ARP が引っかかってきます。

```
07:39:57.988680 IP 172.31.19.240.42237 > 239.1.1.1.4789: VXLAN, flags [I] (0x08), vni 42
ARP, Request who-has 192.168.0.2 tell 192.168.0.22, length 28
```

2 回目以降の通信では ARP は行われません。これは FDB にどの VTEP へパケットを流せばいいか判断するための情報が保存されるためです。

```
# C2 は 192.168.0.2 が MAC アドレス 02:42:c0:a8:00:02 であることを学習している
$ sudo docker container exec C2 ip neigh
192.168.0.12 dev eth0 lladdr 02:42:c0:a8:00:12 STALE
192.168.0.2 dev eth0 lladdr 02:42:c0:a8:00:02 STALE

# FDB には 02:42:c0:a8:00:02 に対しては 172.31.18.142 に送信すればいいことがわかっている
$ sudo ip netns exec overns bridge fdb
33:33:00:00:00:01 dev br0 self permanent
01:00:5e:00:00:6a dev br0 self permanent
33:33:00:00:00:6a dev br0 self permanent
01:00:5e:00:00:01 dev br0 self permanent
33:33:ff:3f:2b:0e dev br0 self permanent
02:42:c0:a8:00:02 dev vxlan0 master br0
0a:2c:07:da:17:96 dev vxlan0 master br0
8e:ae:fa:ff:aa:b7 dev vxlan0 vlan 1 master br0 permanent
8e:ae:fa:ff:aa:b7 dev vxlan0 master br0 permanent
0a:2c:07:da:17:96 dev vxlan0 dst 172.31.18.142 link-netnsid 0 self
00:00:00:00:00:00 dev vxlan0 dst 239.1.1.1 via ifindex 3 link-netnsid 0 self permanent
02:42:c0:a8:00:02 dev vxlan0 dst 172.31.18.142 link-netnsid 0 self
02:42:c0:a8:00:22 dev veth0 master br0
d6:98:d3:b5:20:6b dev veth0 vlan 1 master br0 permanent
d6:98:d3:b5:20:6b dev veth0 master br0 permanent
33:33:00:00:00:01 dev veth0 self permanent
01:00:5e:00:00:01 dev veth0 self permanent
33:33:ff:b5:20:6b dev veth0 self permanent
```

参考までに、node2 の overns 内の各ネットワークインタフェース、ARP, FDB 情報を載せておきます。

```
$ sudo ip netns exec overns ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 8e:ae:fa:ff:aa:b7 brd ff:ff:ff:ff:ff:ff
5: vxlan0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 8e:ae:fa:ff:aa:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
7: veth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether d6:98:d3:b5:20:6b brd ff:ff:ff:ff:ff:ff link-netns 0f8c05851d02

$ sudo ip netns exec overns ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 8e:ae:fa:ff:aa:b7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.21/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::7cb4:e8ff:fe3f:2b0e/64 scope link
       valid_lft forever preferred_lft forever
5: vxlan0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN group default qlen 1000
    link/ether 8e:ae:fa:ff:aa:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::8cae:faff:feff:aab7/64 scope link
       valid_lft forever preferred_lft forever
7: veth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default qlen 1000
    link/ether d6:98:d3:b5:20:6b brd ff:ff:ff:ff:ff:ff link-netns 0f8c05851d02
    inet6 fe80::d498:d3ff:feb5:206b/64 scope link
       valid_lft forever preferred_lft forever

$ sudo ip netns exec overns ip neigh

$ sudo ip netns exec overns bridge fdb
33:33:00:00:00:01 dev br0 self permanent
01:00:5e:00:00:6a dev br0 self permanent
33:33:00:00:00:6a dev br0 self permanent
01:00:5e:00:00:01 dev br0 self permanent
33:33:ff:3f:2b:0e dev br0 self permanent
02:42:c0:a8:00:12 dev vxlan0 master br0
02:42:c0:a8:00:02 dev vxlan0 master br0
8e:ae:fa:ff:aa:b7 dev vxlan0 vlan 1 master br0 permanent
8e:ae:fa:ff:aa:b7 dev vxlan0 master br0 permanent
00:00:00:00:00:00 dev vxlan0 dst 239.1.1.1 via ifindex 3 link-netnsid 0 self permanent
02:42:c0:a8:00:12 dev vxlan0 dst 172.31.29.77 link-netnsid 0 self
02:42:c0:a8:00:02 dev vxlan0 dst 172.31.18.142 link-netnsid 0 self
02:42:c0:a8:00:22 dev veth0 master br0
d6:98:d3:b5:20:6b dev veth0 vlan 1 master br0 permanent
d6:98:d3:b5:20:6b dev veth0 master br0 permanent
33:33:00:00:00:01 dev veth0 self permanent
01:00:5e:00:00:01 dev veth0 self permanent
33:33:ff:b5:20:6b dev veth0 self permanent

$ sudo docker container exec C2 ip neigh
192.168.0.12 dev eth0 lladdr 02:42:c0:a8:00:12 STALE
192.168.0.2 dev eth0 lladdr 02:42:c0:a8:00:02 STALE
```

#### その他の手法

ここで試したもの以外にも etcd のような外部ストレージに情報を保存し、[Netlink][3] を利用して MAC アドレスの解決等必要なときに必要な情報を適宜設定する、といったアプローチもあるようです。[FlannelのVXLANバックエンドの仕組み][4] に依れば Flannel はこのような一例になると思います。

[1]: https://blog.revolve.team/2017/04/25/deep-dive-into-docker-overlay-networks-part-1/
[2]: https://vincent.bernat.ch/en/blog/2017-vxlan-linux
[3]: https://en.wikipedia.org/wiki/Netlink
[4]: https://enakai00.hatenablog.com/entry/2015/04/02/173739
[5]: https://www.infraexpert.com/study/virtual3.html
