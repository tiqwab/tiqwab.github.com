---
layout: post
title: docker-compose で Elasticsearch の split brain を体験する
tags: "elasticsearch, split brain, docker-compose, docker"
comments: true
---

[Elasticsearch][22] はログのストアや分析、サイト内検索機能の実装等に使用されることの多い全文検索エンジンです。特徴の一つとして分散型のドキュメントストアであり、これにより高いパフォーマンス、可用性を実現しているのですが、同時に正しく運用しないと split brain と呼ばれる障害を起こしてしまう危険性もあります。

split brain に関しては [適切な設定][5] を行えば避けることができるので、一度自分で試してみたいと思いつつ複数台構成のクラスタを用意するのは面倒だなと思っていました。最近 Docker のネットワークまわりの知識を手に入れ、docker-compose を使用すればさくっとクラスタを作り split brain を再現できそうだと見通しがたったので、ローカル環境で split brain とその対応設定を試してみることにしました。

1. [split brain とは](#split-brain)
2. [作成する Elasticsearch クラスタの構成](#cluster-structure)
3. [最低限必要な Docker ネットワーク周りの知識](#docker-network)
4. [使用する docker-compose.yml の解説](#docker-compose-yml)
5. [split brain 体験](#split-brain-experience)
6. [minimum\_master\_nodes 設定の登場](#minimum-master-nodes)


環境情報:

- Docker Client, Server: 18.01.0-ce
- Elasticsearch: 6.1.2

---

<div id="split-brain" />

### split brain とは

Elasticsearch において split brain とは「クラスタ内でネットワーク分断が起きた際、分断後のノード間で別々にクラスタを組み直してしまう状況」のことです。

例えば以下のようなネットワーク間にまたがった 5 ノードで構成する Elasticsearch クラスタを考えてみます。

<img
  src="/images/understanding-split-brain-with-docker-compose/split-brain1.png"
  title="sample of elasticseach cluster"
  alt="sample of elasticsearch cluster"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

ここでネットワーク A, B 間で分断が起き、かつ Node 1, Node 2 間、Node 3, Node 4 , Node 5 間で別々のクラスタを組んでしまうと split brain 状態となります。

<img
  src="/images/understanding-split-brain-with-docker-compose/split-brain2.png"
  title="cluster in split brain"
  alt="cluster in split brain"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

この状態だとデータの書き込みは片方のクラスタにしか反映されないためデータの整合性を保証できなくなります。結果、Elasticsearch ユーザから見ると例えばアクセスするノードに応じて結果が見えたり見えなかったりするという混乱する状況になってしまいます。

(補足) ちなみにここでは特に Elasticsearch を取り上げましたが、[split brain][1] 自体は分散システム一般に存在する概念です。

(補足) また上で異なるネットワーク間で Elasicsearch クラスタを組むという例を上げましたが、[Elasticsearch はデータセンターをまたぐようなクラスタの組み方を推奨していない][3] とのことなので、ここでのイメージはローカルネットワーク、あるいは AWS でならリージョン内の異なる Availability Zone 間で、という感じに考えてもらえればと思います。

Elasticsearch の文脈における split brain に関して以下の記事を参考にしました。

- [elasticseachでノード障害が起きたときの動作][2]

<div id="cluster-structure" />

### 作成する Elasticsearch クラスタの構成

今回 split brain 体験用に作成する Elasticsearch クラスタは以下のような構成にしました。

<img
  src="/images/understanding-split-brain-with-docker-compose/cluster-design.png"
  title="the design of elasticsearch cluster"
  alt="the design of elasticsearch cluster"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

上述した例と同様 5 ノードから成る Elasticsearch クラスタです。各ノードはいずれもマスタかつデータノードとして機能させます。各ノードは Docker コンテナとして動いており、その host が図の中心に配置されています。ネットワーク的にはこの host がルータとして働くという感じになるかと思います。

<div id="docker-network" />

### 最低限必要な Docker ネットワーク周りの知識

クラスタ構成を考えたので、あとは `docker-compose.yml` でコンテナ定義を用意するだけなのですが、とはいえ構成図通りのネットワークを作成するには多少 Docker のネットワーク周りの知識も必要になります。

ここでは Docker コンテナとホスト間、コンテナとコンテナ間の通信を理解する上で重要な要素についていくつか見ていきたいと思います。

#### veth

Docker を使用し始めたときに最初に触れるであろうコマンドの一つに `docker container run` が挙げられます。例えば

```
docker container run -d -p 8888:80 dockersamples/static-site
```

を実行すると、コンテナ上でサンプルの Web サーバが動き、ホストからは `http://localhost:8888` でサンプルページにアクセスできます。また

```
docker container exec -it <container name> ping -c 3 google.co.jp
```

を実行するとコンテナ内からホストを介して外部のネットワークにアクセスすることができます。

このようにただ Docker にコンテナ起動を依頼するだけでコンテナとホスト間が何らかの仕組みで通信可能になるわけですが、その正体は veth という仮想的なネットワークインタフェースです。veth は Linux カーネルに組み込まれている機能の一つで、作成するには以下のコマンドを実行します。

```
# Create veth
$ ip link add veth-sideA type veth peer name veth-sideB

# Show the created veth
$ ip link show type veth
13: veth-sideB@veth-sideA: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether a2:f8:0f:83:0a:6b brd ff:ff:ff:ff:ff:ff
14: veth-sideA@veth-sideB: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 46:a7:7a:df:28:ea brd ff:ff:ff:ff:ff:ff
```

上のように veth はネットワークインタフェース 2 つのペアとして作成されます。

veth のペアはお互いにのみ通信可能であるという特徴があり、コンテナとホスト間はこの仕組みにより仮想的に繋がっています。イメージ的には物理マシン間を LAN ケーブルで繋いでいるという感じでしょうか。

veth については以下の内容を参考にしました。

- [networking:bridge (Linux Foundation Wiki)][11]
- [第6回 Linux カーネルのコンテナ機能][12]

#### netns

先程のコンテナを起動すると veth が作成されるのですが、実際にホスト側でネットワークインタフェースを一覧してみると、その veth は片割れしか見えません。

```
$ ip link show type veth
12: vethccd8eab@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default 
    link/ether 42:79:e4:41:ba:cb brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

対応する veth が何故見えないのかというとそれは起動したコンテナと同じ netns (ネットワーク名前空間) に移動させられたから、ということになります。Linux カーネルのコンテナ実装を支える機能の一つに種々の名前空間がありますが、その中で netns はネットワークインタフェースのようなネットワーク関連を担当しています。netns の概要に関しては ip netns の man ページが簡潔でわかりやすいと思います。

起動したコンテナの netns は以下のようにそのプロセス ID から確認することができます。

```
# Get pid of the running container (in the host)
$ CONTAINER_PID=$(docker container inspect heuristic_chatterjee | jq .[0].State.Pid)

# Check netns of the process
$ ls -l /proc/$CONTAINER_PID/ns/net
lrwxrwxrwx 1 root root 0 Jan 22 21:58 /proc/2524/ns/net -> 'net:[4026532398]'
```

netns は慣習的に `/var/run/netns` 下にファイルディスクリプタを配置するようですが、Docker コンテナの netns はここには作成されず、`ip netns list` でも確認できません。`ip netns` の対象とするには例えばここに上記ファイルを指すシンボリックリンクを作成します。

```
# Create symbolic link to the target netns
$ ln -s /proc/2524/ns/net /var/run/netns/heuristic_ns

# List netns
$ ip netns list
heuristic_ns

# Execute command in the specified netns
$ ip netns exec heuristic_ns ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

ここで見えている `eth0@if12` が目的とするもう片方の veth です。

netns に関しては以下の内容を参考にしました。

- [Dockerのネットワーク管理とnetnsの関係][13]
- [Linux Network Namespace で遊んで見る][14]
- [ip netns コマンドが以外にきめ細やかにコンテナを作ってくれる][15]

#### bridge

veth, netns によりコンテナ、ホスト間の通信についてはイメージができたので、次にコンテナ、コンテナ間を見ていきます。

上で取り上げたコンテナと同様のものをもう一台作成します。

```
$ docker container run -d -p 8889:80 dockersamples/static-site
```

次に立ち上げたコンテナ 2 台に割り当てられた IP を見てみると、それぞれの eth0 が同一ネットワークに属していることがわかります。

```
# List containers
$ docker container ls
CONTAINER ID        IMAGE                       COMMAND                  CREATED              STATUS              PORTS                           NAMES
c237b12a9243        dockersamples/static-site   "/bin/sh -c 'cd /usr…"   About a minute ago   Up About a minute   443/tcp, 0.0.0.0:8889->80/tcp   romantic_curran
c7300a8e0672        dockersamples/static-site   "/bin/sh -c 'cd /usr…"   24 hours ago         Up 24 hours         443/tcp, 0.0.0.0:8888->80/tcp   heuristic_chatterjee

# Check the assigned IP of a container
$ docker container exec -it heuristic_chatterjee ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

# Check the assigned IP of another
$ docker container exec -it romantic_curran ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
17: eth0@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

`ip route get` してみると確かにお互い直接通信ができるようです。

```
$ docker container exec -it heuristic_chatterjee ip route get 172.17.0.3
172.17.0.3 dev eth0  src 172.17.0.2 
    cache 
$ docker container exec -it romantic_curran ip route get 172.17.0.2
172.17.0.2 dev eth0  src 172.17.0.3 
    cache 
```

このようなコンテナ間通信は [Linux bridge][10] を使用した [ブリッジ接続][16] により実現されています。bridge には veth も含めたネットワークインタフェースをアタッチでき、それらを同一セグメントして扱うことを可能にします。

Docker ネットワークを扱うコマンド `docker network` でいうと、`DRIVER=bridge` なものがいま話題にしている bridge を使用したネットワークです。詳細を見るとネットワーク内の各コンテナの IP 等を確認することもできます。

```
# List network managed by Docker
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
59a4e3dcca41        bridge              bridge              local
ea85a1861566        host                host                local
c47a9a8af292        none                null                local

# Show detail of the network
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "59a4e3dcca41f5345c7d19c9f73ddcec62be7e70433fd4f28130ddb3b4bf5e72",
        "Created": "2018-01-22T21:48:40.06518534+09:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "c237b12a924330c28c13acdf5338cb9d7e4f5f228176709482d77239f083e6d7": {
                "Name": "romantic_curran",
                "EndpointID": "c0bbcad8587534e1b0a5e5ef2accb3213cf5fd46d8b7a5e852ff809629be8914",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "c7300a8e06724aecffd55c8e5c23d1f2517fc4c84bc46fb98b71fa167cfe0a3e": {
                "Name": "heuristic_chatterjee",
                "EndpointID": "9ac301887858aadbc2441b182cbe17233444debcb839fe7f4cdf43a8919092ae",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

単純に `docker container run` した場合は Docker がデフォルトで用意する `docker0` という名前の bridge に属することになります。これはコンテナ起動時に変更可能であり、このあと Elasticsearch クラスタを作成する場合には別の bridge を使用することになります。

また Linux 的には bridge を管理するコマンド (の一つ) として `brctl` があり、こちらでも一覧で docker0 を確認することができます。

```
$ brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242eda31f33	no		veth1cd036c
							vethccd8eab
```

Linux bridge に関しては以下の内容も参考にしました。

- [ネットワークブリッジ  - ArchWiki][17]

#### iptables

[iptables][18] は Linux でファイアウォールやルータ設定を行うために使用するコマンド (正確には Netfilter というパケット処理用モジュールのフロントエンドという感じ？) です。Docker はデーモン起動時やネットワーク作成時等にホストの iptables を変更します。設定内容は多岐に渡りますが、今回の目的に関わるものとして Docker ネットワーク間の通信を制限するというのがあります。

```
$ iptables -nvL

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   15  1480 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
   15  1480 DOCKER-ISOLATION  all  --  *      *       0.0.0.0/0            0.0.0.0/

...

Chain DOCKER-ISOLATION (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DROP       all  --  br-6fd7eba4e53b docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 DROP       all  --  docker0 br-6fd7eba4e53b  0.0.0.0/0            0.0.0.0/0           
    0     0 DROP       all  --  br-2fbacd7bd31e docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 DROP       all  --  docker0 br-2fbacd7bd31e  0.0.0.0/0            0.0.0.0/0           
    0     0 DROP       all  --  br-2fbacd7bd31e br-6fd7eba4e53b  0.0.0.0/0            0.0.0.0/0           
    0     0 DROP       all  --  br-6fd7eba4e53b br-2fbacd7bd31e  0.0.0.0/0            0.0.0.0/0           
   15  1480 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   15  1480 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0  

...
```

FORWARD チェインで 2 番目に参照される DOCKER-ISOLATION チェインで bridge 間のパケットを DROP するような設定が定義されています。

Docker により追加されている挙動は DOCKER-ISOLATION より先に参照される DOCKER-USER チェインを使用することで上書きできます。例えば br-2fbacd7bd31e, br-6fd7eba4e53b 間の通信を有効にしたければ以下のルールを DOKCER-USER に追加します。

```
# Insert a new rule before existing rules
$ iptables -I DOCKER-USER -p all -i br-2fbacd7bd31e -o br-6fd7eba4e53b -j ACCEPT
$ iptables -I DOCKER-USER -p all -i br-6fd7eba4e53b -o br-2fbacd7bd31e -j ACCEPT
```

Elasticsearch クラスタ作成時にはこれを使用してネットワーク間の通信をコントロールします。

iptables に関しては以下の内容も参考にしました。

- [Docker container networking][20]
- [iptables - ArchWiki][19]

<div id="docker-compose-yml" />

### 使用する docker-compose.yml の解説

ここまでの内容を踏まえて上述の構成図通りに `docker-compose.yml` を作成します。

以下は作成した `docker-compose.yml` のうち、es-node1, es-node3 に当たるコンテナ定義とネットワーク定義の抜粋です。ファイル全体は [こちら][6] に配置しています。

```yaml
version: '2.2'

# The definition of containers
services:
  es-node1:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.1.2
    container_name: es-node1
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=172.18.0.2,172.18.0.3,172.19.0.2,172.19.0.3,172.19.0.4"
      - "node.attr.network=es-netA"
      - "cluster.routing.allocation.awareness.attributes=network"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - 9200:9200
    networks:
      es-netA:
        ipv4_address: 172.18.0.2

...

  es-node3:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.1.2
    container_name: es-node3
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=172.18.0.2,172.18.0.3,172.19.0.2,172.19.0.3,172.19.0.4"
      - "node.attr.network=es-netB"
      - "cluster.routing.allocation.awareness.attributes=network"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - 9201:9200
    networks:
      es-netB:
        ipv4_address: 172.19.0.2

...


# The definition of networks
networks:
  es-netA:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/24
          gateway: 172.18.0.1
  es-netB:
    driver: bridge
    ipam:
      config:
        - subnet: 172.19.0.0/24
          gateway: 172.19.0.1
```

この定義のベースは公式ドキュメント [Install Elasticsearch with Docker][4] から持ってきているので、ここではそこから変更した部分を中心に取り上げます。

まずネットワーク定義についてですが、構成図の通りに es-netA, es-netB という 2 つを作成しています。見たままですが、以下の定義で es-netA という名前の docker network 設定がサブネット `172.18.0.0/24`, デフォルトゲートウェイ `172.18.0.1` で作成されます。

```
  es-netA:
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/24
          gateway: 172.18.0.1
```

コンテナ定義では各コンテナをいずれかのネットワークに追加します。ここでは es-node1, es-node2 には es-netA を, es-node3, es-node4, es-node5 には es-netB を割り当てます。またこのとき docker-compose v2 では明示的に IP を割り当てることができるのですが、どうやらこれは v3 ではできないようなので注意です ([Compose file version 3 reference][7])。

```
  es-node1:
    ...
    networks:
      es-netA:
        ipv4_address: 172.18.0.2
```

次に `discovery.zen.ping.unicast.hosts` の設定を行います。これは [Discovery][8] と呼ばれる Elasticsearch における新規ノードの追加やマスタの決定に関わるモジュールの設定です。公式ドキュメントでは

> It is recommended that the unicast hosts list be maintained as the list of master-eligible nodes in the cluster

という記述 (from [Zen Discovery][5]) があり、マスタになり得るノードのリストを指定するべきなようです。作成中のクラスタでは全ノードがマスタ候補なので 5 台の IP をカンマ区切りで設定しています。

```
- "discovery.zen.ping.unicast.hosts=172.18.0.2,172.18.0.3,172.19.0.2,172.19.0.3,172.19.0.4"
```

最後に [Shard Allocation Awareness][9] の設定です。今回クラスタ内に 2 つのネットワークを用意していますが、もしプライマリ、レプリカシャードともに同一ネットワークに配置された場合、何らかの原因でネットワーク全体に障害が発生した場合にそのデータにアクセスできなくなってしまいます。このようなケースを避けるために、プライマリ、レプリカシャードの配置をある程度コントロールするための設定として `cluster.routing.allocation.awareness` を使用します。

```
# for nodes in es-netA
- "node.attr.network=es-netA"
- "cluster.routing.allocation.awareness.attributes=network"

# for nodes in es-netB
- "node.attr.network=es-netB"
- "cluster.routing.allocation.awareness.attributes=network"
```

上の YAML では各ノードが属するネットワークに応じてノード属性を `network=es-netA` のように持つようにしました。この属性を `cluster.routing.allocation.awareness.attributes` で指定することで、どのシャードもプライマリとレプリカが別ネットワークに属すように配置させることができます。今回の場合もし es-netA, es-netB の片方のネットワークに全くアクセスできなくなったとしても、データが失われるということは防ぐことができます。

なお volume 設定は今回特に必要が無いため用意しませんでした。

作成した `docker-compose.yml` を使用して実際にクラスタが構築できることを確認します。

```
$ docker-compose up -d

$ curl http://localhost:9200/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 5,
  "active_primary_shards" : 1,
  "active_shards" : 2,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

`number_of_nodes` からちゃんと 5 台で構成できていることが、`status` が green であることから正常にクラスタが構築できていることがわかります。

<div id="split-brain-experience" />

### split brain 体験

長かったですが下準備が終わったので、あとは Elasticsearch を叩いていくだけです。

まずテスト用に適当なデータを投入します。

```
$ curl -XPUT -H "Content-Type: application/json" http://localhost:9200/persons/doc/1?pretty -d '{"name": "Alice", "age": 20}'
{
  "_index" : "persons",
  "_type" : "doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}

$ curl http://localhost:9200/persons/doc/1?pretty
{
  "_index" : "persons",
  "_type" : "doc",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "name" : "Alice",
    "age" : 20
  }
}
```

この状態で es-netA, es-netB ネットワーク間を分断するために追加した iptables ルールを削除します。

```
# Delete rules
$ iptables -D DOCKER-USER 2
$ iptables -D DOCKER-USER 1
```

少し待つと、es-netA, es-netB それぞれに属するノード間でクラスタが作成されてしまいました。これが split brain 状態ということですね。

```
# Access node in es-netA
$ curl http://localhost:9200/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 2,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 6,
  "active_shards" : 12,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}

# Access node in es-netB
$ curl http://localhost:9201/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 6,
  "active_shards" : 12,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

何も知らないユーザが更新をかけにきますがその結果は各クラスタ内にしか反映されません。クラスタ自体は正常に動いているので異常にも気づきにくいでしょうし、こうなるとかなりやっかいな状況になると思います。

```
# Partially update to the cluster in es-netA
$ curl -H "Content-Type: application/json" http://localhost:9200/persons/doc/1/_update?pretty -d '{"doc": {"tag": {"tag1": "foo"}}}'

$ curl http://localhost:9200/persons/doc/1?pretty
{
  "_index" : "persons",
  "_type" : "doc",
  "_id" : "1",
  "_version" : 2,
  "found" : true,
  "_source" : {
    "name" : "Alice",
    "age" : 20,
    "tag" : {
      "tag1" : "foo"
    }
  }
}

# Partially update to the cluster in es-netB
$ curl -H "Content-Type: application/json" http://localhost:9201/persons/doc/1/_update?pretty -d '{"doc": {"tag": {"tag2": "bar"}}}'

$ curl http://localhost:9201/persons/doc/1?pretty
{
  "_index" : "persons",
  "_type" : "doc",
  "_id" : "1",
  "_version" : 2,
  "found" : true,
  "_source" : {
    "name" : "Alice",
    "age" : 20,
    "tag" : {
      "tag2" : "bar"
    }
  }
}
```

(ちなみにここでネットワークの分断が解消されるとどうなるかも見たのですが、自動では一つのクラスタに統合されないようでした。無理矢理片方の master ノードを落とすと統合されますが、その場合同一 ID のデータはいずれかのクラスタのものを使用するしかないと思うので、結果として書き込みの一部が消失することになります)

<div id="minimum-master-nodes" />

### minimum\_master\_nodes 設定の登場

最後に Elasticsearch で split brain を避けるために提供されている `minimum_master_nodes` 設定を確認します。これはクラスタ内にマスタ候補ノード (`node.master` が true) が最低限何台存在しないといけないかを示す値で、`(クラスタ内マスタ候補ノード数 / 2) + 1` を指定するべきとされています。この値は言い換えれば「マスタ候補ノード数の過半数を獲得するために必要な最低ノード数」であり、クラスタ分断後に稼働し続けるクラスタを決定するために使用されます。

今回の場合、5 台のノード全てがマスタ候補になるのでコンテナ定義の環境変数に `minimum_master_nodes` として 3 を設定します。

```yaml
    environment:
      # 追加
      - "discovery.zen.minimum_master_nodes=3"
```

クラスタ作成後、たしかに設定が反映されていることを確認します。

```
$ curl http://localhost:9200/_nodes?pretty
...
        "discovery" : {
          "zen" : {
            "minimum_master_nodes" : "3",
            "ping" : {
              "unicast" : {
                "hosts" : "172.18.0.2,172.18.0.3,172.19.0.2,172.19.0.3,172.19.0.4"
              }
            }
          }
        },
...
```

上と同様にネットワーク分断後、それぞれのノードにアクセスしてみると、3 台構成の es-netB 内のノードからは正常にレスポンスが返ってくる一方で、2 台しか存在しない es-netA からはマスタが存在しないというエラーが返ってきます。

```
# Access node in es-netA
$ curl http://localhost:9200/_cluster/health?pretty
{
  "error" : {
    "root_cause" : [
      {
        "type" : "master_not_discovered_exception",
        "reason" : null
      }
    ],
    "type" : "master_not_discovered_exception",
    "reason" : null
  },
  "status" : 503
}

# Access node in es-netB
$ curl http://localhost:9201/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 6,
  "active_shards" : 12,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

es-netA 内のノードからは以下のような警告ログが出ていることも確認でき、想定通りの挙動をしていることがわかります。

```
[2018-01-20T11:55:26,325][WARN ][o.e.d.z.ZenDiscovery     ] [JMEliYg] not enough master nodes discovered during pinging (found [[Candidate{node={AkJXTpn}{AkJXTpnzSVKH3Lmn8g99kA}{ZIjhHqqWTDuMGk4DmyVsHg}{172.18.0.3}{172.18.0.3:9300}{ml.machine_memory=16612593664, ml.max_open_jobs=20, ml.enabled=true, network=es-netA}, clusterStateVersion=24}, Candidate{node={JMEliYg}{JMEliYgYT-qEoXojYll_SA}{Cumq9Af8SKyp7et6r1RKLg}{172.18.0.2}{172.18.0.2:9300}{ml.machine_memory=16612593664, ml.max_open_jobs=20, ml.enabled=true, network=es-netA}, clusterStateVersion=24}]], but needed [3]), pinging again
```

この状態でも es-netA 内のノードに read はできるのですが、write は怒られます (ただし [No master block][21] によれば read もできないように設定することが可能なよう)。

ネットワーク分断が解消されると、クラスタは自動的にマージされ、5 台構成に戻ります。もちろんその際には es-netB クラスタに書き込まれた内容は保持されます。

[1]: https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%97%E3%83%AA%E3%83%83%E3%83%88%E3%83%96%E3%83%AC%E3%82%A4%E3%83%B3%E3%82%B7%E3%83%B3%E3%83%89%E3%83%AD%E3%83%BC%E3%83%A0
[2]: https://www.creationline.com/blog/15831
[3]: https://www.elastic.co/blog/clustering_across_multiple_data_centers
[4]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[5]: https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html
[6]: https://github.com/tiqwab/example/blob/master/elasticsearch-with-docker/five-nodes/docker-compose.yml
[7]: https://docs.docker.com/compose/compose-file/#ipam
[8]: https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery.html
[9]: https://www.elastic.co/guide/en/elasticsearch/reference/current/allocation-awareness.html
[10]: https://wiki.linuxfoundation.org/networking/bridge
[11]: https://wiki.linuxfoundation.org/networking/bridge
[12]: http://gihyo.jp/admin/serial/01/linux_containers/0006
[13]: http://enakai00.hatenablog.com/entry/20140424/1398321672
[14]: http://momijiame.tumblr.com/post/48994420012/linux-network-namespace-%E3%81%A7%E9%81%8A%E3%82%93%E3%81%A7%E3%81%BF%E3%82%8B
[15]: http://tenforward.hatenablog.com/entry/20160726/1469533536
[16]: https://ja.wikipedia.org/wiki/%E3%83%96%E3%83%AA%E3%83%83%E3%82%B8%E6%8E%A5%E7%B6%9A
[17]: https://wiki.archlinux.jp/index.php/%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AF%E3%83%96%E3%83%AA%E3%83%83%E3%82%B8
[18]: https://ja.wikipedia.org/wiki/Iptables
[19]: https://wiki.archlinux.jp/index.php/Iptables
[20]: https://docs.docker.com/engine/userguide/networking/
[21]: https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html#no-master-block
[22]: https://www.elastic.co/jp/products/elasticsearch
