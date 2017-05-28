---
layout: post
title: Elasticsearch の Discovery について
tags: "elasticsearch, aws"
comments: true
---

[Elasticsearch][8] では複数の node から cluster を構成し、データを永続化します。
Elasticsearch には稼働中にも新規に追加される node を 'discovery' して cluster へ追加する仕組みがあるようなのですが、その挙動がピンと来なかったので簡単な cluster を EC2 インスタンスで構成して確認しました。

1. [CloudFormation でネットワーク構築](#anchor1)
2. [Elasticsearch のインストール](#anchor2)
3. [Discovery](#anchor3)
  - 3.1 [Zen Discovery](#anchor3_1)
  - 3.2 [EC2 Discovery](#anchor3_2)
  - 3.3 [Shared Allocation Awareness](#anchor3_3)

### 環境

- Elasticsearch: 2.4.5

---

<div id="anchor1" />

### 1. CloudFormation でサンプルネットワークの構築

今回のサンプルでは AWS の EC2 インスタンス上で Elasticsearch を動かしたいので、そのための VPC やらを CloudFormation で作成しました。使用した [テンプレート][9] はこちらです。

テンプレートの概要は以下の通りです。

- VPC 内に subnet を 2 つ作成
- Subnet の AZ は互いに別にしている (ap-northeast-1a, ap-northeast-1c)
- Security Group では Elasticsearch 用に port 番号 9200, 9300 を開いている

<div id="anchor2" />

### 2. Elasticsearch のインストール

Elasticsearch を EC2 インスタンスにインストールしていきます。
なお現在の Elasticsearch の最新版は 5.x ですが今回は事情によりバージョン 2.4.5 を使用します。

```
$ sudo rpm -i https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/rpm/elasticsearch/2.4.5/elasticsearch-2.4.5.rpm
```

`/etc/elasticsearch/elasticsearch.yml` にいくつか Elasticsearch の設定を行います。

```yml
# --- Cluster ---
# Specify the name of cluster which this node belongs to.
cluser.name: cluster-sample

# --- Node ---
# Specify node name. Here use hostname of the node
node.name: ${HOSTNAME}
# Specify whether this node is master node or not. Default is true.
# node.master: true
# Specify whether this node is dada node or not. Default is true.
# node.data: true

# --- Network ---
# Specify hostname of IP address used to communicate nodes within this host.
# '_site_' means private network such as '192.168.0.1'.
network.host: [_site_]
```

この中で重要なのは `cluster.name` で、cluster はここで設定したクラスタ名により識別されます。
新しく追加する node は同一のクラスタ名を持つ必要がありますし、逆に既存の cluster とは別の cluster を作りたい場合は `cluster.name` は別の値にするべきです。

次に cluster の状況を手軽に GUI で見るために [head プラグイン][2]を使用します。
最新版では deprecated なのですが v2.4.5 では `plugin install` による手順を行います。

```
$ cd /usr/share/elasticsearch/
$ sudo bin/plugin install mobz/elasticsearch-head
```

インストール後、`http://<ip_address>:9200/_plugin/head/` で head プラグインが提供する UI にアクセスできるようになります。

Elasticsearch を自動起動するサービスとして設定します。

```
$ sudo chkconfig --add elasticsearch
```

Elasticsearch を起動します。

```
$ sudo service elasticsearch start
```

適当に Cluster, Index を作ると以下のように sharding が行われている状態が確認できます。
sharding の設定はデフォルトのものを使用しているので、1 つの index に対し 5 つの shards が作られます。
また各 primary shard に replica が 1 つ作られるので、計 10 shards を作ろうとします。

<img
  src="/images/elasticsearch-cluster/cluster-with-one-node.png"
  title="cluster-with-one-node"
  alt="cluster-with-one-node"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

ただ現時点では cluster に存在する node が 1 つのみであり、primary shards と replica shards は同一 node 上には置けないので、5 つの replica shards がいずれも配置できないという状態になっています。

<div id="anchor3" />

### 3. Discovery

Elasticsearch では、新しく追加した node が cluster に追加される過程を [Discovery][3] と呼びます。
Cluster は `/etc/elasicsearch/elasticsearch.yml` で指定する `cluster.name` により識別されますが、それだけでは 既存の cluster に入ることはできず、対象 cluster 中の node (master node?) に 'discovery' してもらう必要があるようです。

ここでは Discovery の実装として Zen Discovery と EC2 Discovery について取り上げます。

<div id="anchor3_1" />

#### 3.1 Zen Discovery

[Zen Discovery][4] は Elasticsearch の用意するデフォルト実装であり、 unicast に基づいたものになります (multicast で行う仕組みもあったようですが既に deprecated)。

設定としては `/etc/elasticsearch/elasticsearch.yml` の以下のフィールドに追加したい node の ホスト名あるいは IP アドレスを指定するというものになります。

例えば 上と同様な EC2 インスタンスをもう一台用意し、それぞれの IP アドレスが '10.0.0.36', '10.0.0.160' の場合、それぞれに以下の記述を加えることで node 2 台構成の cluster が作成されます。

```
# --- Discovery ---
# Specify the list of (at least some of) other nodes in the cluster.
discovery.zen.ping.unicast.hosts: ['10.0.0.36', '10.0.0.160']
```

<img
  src="/images/elasticsearch-cluster/cluster-with-two-nodes.png"
  title="cluster-with-two-nodes"
  alt="cluster-with-two-nodes"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

なおデフォルトでは port 9300 が node 間の連携に使用されるので、ファイアウォールやセキュリティグループの設定でこの port を空けておく必要があります (ちなみに外部に提供する REST API として使用している port は 9200)。

<div id="anchor3_2" />

#### 3.2 EC2 Discovery

EC2 インスタンスを node として使用する場合、Discovery の実装として EC2 Discovery を利用することができます。
Elasticsearch v5.x では [EC2 Discovery Plugin][6] が提供されているのですが、今回は v2.x を対象としているため、[AWS Cloud Plugin][5] を使用します。

AWS Cloud Plugin を各 node にインストールします。

```
$ sudo bin/plugin install cloud-aws
```

`/etc/elasticsearch/elasticsearch.yml` を以下のように変更します。
設定の概要は

- `discovery.ec2.groups` で指定する Security Group を持つ node を discovery する 
  - 何か勘違いしているかも? 追記参照
- `cloud.node.auto_attributes` を true にし、`aws_availability_zone` を参照可能にする
  - 設定中の node の属する AZ が取得できる
  - この値は `http://<hostname>:9200/_nodes/transport?pretty` で確認できる
  - のちの [Shared Allocation Awareness](#anchor3_3) で使用

という感じです。

(追記: 試しに わざと異なる Security Group に属するインスタンスを追加したのですが、それでも cluster に入ってしまったので、思っているのと挙動が違う気がしています)

```
# Comment out
# discovery.zen.ping.unicast.hosts: ['10.0.0.1', '10.0.0.2']

# --- AWS Cloud Plugin ---
discovery.type: ec2
cloud.aws.region: 'ap-northeast-1'
cloud.node.auto_attributes: true
discovery.ec2.groups: sg-acc73fca
```

またこのプラグインは EC2 インスタンスの情報を得るために EC2 サービスへのアクセスを必要とするため、その IAM ロール (with AmazonEC2ReadOnlyAccess Policy) もインスタンスに設定しておきます。

上の Zen Discovery の unicast の挙動と異なり、こちらは (少なくともデフォルトの挙動だと) 条件を満たす node は勝手に cluster に追加されるようなので、便利ですが少し注意が必要です。

<div id="anchor3_3" />

#### 3.3 Shared Allocation Awareness

EC2 インスタンス上で Elasticsearch を動かしている場合、primary と replica shards は別の AZ 上の data node に保存したいと思うのが自然だと思います。

[Shared Allocation Awareness][7] で紹介されている機能によれば、以下の 2 つの設定を `/etc/elasticsearch/elasticsearch.yml` に記載するとそういった挙動を実現することができるようです。

```yml
cluster.routing.allocation.awareness.attributes: aws_availability_zone
cluster.routing.allocation.awareness.force.aws_availability_zone.values: ap-northeast-1a, ap-northeast-1c
```

2 台の EC2 インスタンスが両方とも同一の AZ (ap-northeast-1a) にいる状態でシステムの再起動を行うと、replica shards を配置する場所が無くなるため、cluster health が yellow となります。

<img
  src="/images/elasticsearch-cluster/cluster-with-two-nodes-same-az.png"
  title="cluster-with-two-nodes-same-az"
  alt="cluster-with-two-nodes-same-az"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

もう 1 台 EC2 インスタンスを 別の AZ (ap-northeast-1c) に生やすと、そちらに replica shards が作成され、cluster health が green に戻ります。

<img
  src="/images/elasticsearch-cluster/cluster-with-two-nodes-different-az.png"
  title="cluster-with-two-nodes-different-az"
  alt="cluster-with-two-nodes-different-az"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

ここでは AZ を意識した sharding を行うようにしましたが、設定する値として別のものを使用すれば、違った基準に基づいて primary と replica の sharding 先を制御することもできます。

参考:

- [Amazon EC2 を使用して Elasticsearch クラスタをセットアップする][1]
- [Elasticsearch AWS で Multi-AZ 配置][10]

[1]: https://www.elastic.co/jp/blog/setting-up-es-cluster-on-ec2
[2]: https://github.com/mobz/elasticsearch-head
[3]: https://www.elastic.co/guide/en/elasticsearch/reference/2.4/modules-discovery.html
[4]: https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html
[5]: https://www.elastic.co/guide/en/elasticsearch/plugins/2.4/cloud-aws.html
[6]: https://www.elastic.co/guide/en/elasticsearch/plugins/5.4/discovery-ec2.html
[7]: https://www.elastic.co/guide/en/elasticsearch/reference/2.4/allocation-awareness.html
[8]: https://www.elastic.co/products/elasticsearch
[9]: https://github.com/tiqwab/example/blob/master/sample-elasticsearch-cluster/es-cfn-template.yml
[10]: https://medium.com/hello-elasticsearch/elasticsearch-aws-%E3%81%A7-multi-az-%E9%85%8D%E7%BD%AE-e638a0262916
