---
layout: post
title: SoftEther VPN のクライアントセットアップ for Linux
tags: "linux, vpn, softethervpn"
comments: true
---

最近 [SoftEhter VPN][1] というソフトウェアを使用して自宅 VPN を構築しました。

自分で VPN 環境を作成するのは初めてだったのですが、VPN サーバについては公式ドキュメントであったり いくつかのブログ記事であったりを参考に比較的スムーズに用意できました。

クライアントについても大体は難なく進み、例えば L2TP/IPSec を利用した Windows, Android, iPad からの接続や SoftEther VPN Client を使用する Windows からの接続はすぐに確認できました。

ただ Linux をクライアントにした接続についてはかなり苦戦をし、VPN に不慣れなことと情報があまり出てこないことから軽く一日以上を溶かしてしまいました。

ということで一例として自分が確認した接続方法 2 通りを紹介しておこうと思います。

検証に使用した PC の OS と関連パッケージ情報は以下の通りです。

```
$ uname -a
Linux sd004 5.2.14-arch2-1-ARCH #1 SMP PREEMPT Thu Sep 12 10:42:38 UTC 2019 x86_64 GNU/Linux

$ yay -Qs networkmanager
...
local/networkmanager 1.20.2-1 (gnome)
    Network connection manager and user applications
local/networkmanager-l2tp 1.2.12-2
    L2TP support for NetworkManager
...

$ yay -Qs strongswan
local/strongswan 5.8.1-1
    Open source IPsec implementation

$ yay -Qs softethervpn
local/softethervpn v4.29_9680-1
    Multi-protocol VPN Program from University of Tsukuba
```

### L2TP/IPSec による Linux クライアントからの接続

自身の Linux 環境ではネットワークまわりの管理に [NetworkManager][2] を使用しています。
NetworkManager ではプラグインで VPN がサポートされています。ここでは L2TP/IPSec 接続を行いたいので [NetworkManager-l2tp][3] というプラグインをインストールして設定してみました。

例えば Arch Linux 環境であれば以下のパッケージをインストールします。

```
# networkmanager-l2tp は現状 AUR パッケージ
# strongswan (or libreswan) は IPSec を使用する場合に必要
$ yay -S networkmanager-l2tp strongswan
```

GUI で NetworkManager 設定を触れるならば、恐らく VPN Connections のあたりから新規に VPN 接続設定を追加することができるはずです。このあたりは [Ubuntu 18.04をL2TPのVPNクライアントにする][4] を参考にしました。

ここで自分が躓いたのは IPSec Settings のアルゴリズム設定でした。参考元のブログでも苦労していたっぽいのですが、VPN サーバの情報を確認しつつアルゴリズムを記載してもどうも接続確立中に失敗するようで、ログを見ても原因がピンと来ず悩みました。

わからなすぎて一度諦めていたのですが、改めて NetworkManager-L2TP の README を読むと [Issue with VPN servers only proposing IPSec IKEv1 weak legacy algorithms][5] という項があり、SHA1 だしどうもいまのケースは該当していそうな気がするな、と思い記述に従い Legacy Proposals オプションを使用したところうまくいきました。

参考までに接続時に作成されたインタフェース、変更されたルーティングテーブル、NetworkManager の connection 設定ファイルをまとめておきます (IP アドレス等一部修正)。

```
$ ip address show ppp0
28: ppp0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 3
    link/ppp 
    inet 192.168.1.12 peer 1.0.0.1/32 scope global ppp0
       valid_lft forever preferred_lft forever
```

- ppp とは？
  - Point-to-Poiint Protocol
  - ここでは自端末と 1.0.0.1/32 が振られた端末との peer-to-peer 通信が確立されている
    - ただ 1.0.0.1 自体にはどうやら意味がないらしい？
    - ppp のネットワークインタフェースだけ欲しいみたいなことなのか？

```
$ ip route

default dev ppp0 proto static scope link metric 50 # added
default via 192.168.40.1 dev wlp2s0 proto dhcp metric 600 
1.0.0.1 dev ppp0 proto kernel scope link src 192.168.1.12 metric 50 # added
<vpn server ip> via 192.168.40.1 dev wlp2s0 proto static metric 600  # added
192.168.40.0/24 dev wlp2s0 proto kernel scope link src 192.168.40.218 metric 600 
192.168.40.1 dev wlp2s0 proto static scope link metric 600 # added
```

- デフォルトゲートウェイについての設定が 2 つある？
  - この場合 metric 値が小さい方が優先されるので追加したものが使用される

```
$ cat /etc/NetworkManager/system-connections/'VPN connection 1.nmconnection'
 [connection]
 id=vpnconnection1
 uuid=<uuid>
 type=vpn
 autoconnect=false
 permissions=
 
 [vpn]
 gateway=<vpn server host name>
 ipsec-enabled=yes
 ipsec-esp=aes256-sha1,aes128-sha1,3des-sha1!
 ipsec-ike=aes256-sha1-ecp384,aes128-sha1-ecp256,3des-sha1-modp1024!
 ipsec-psk=<psk>
 lcp-echo-failure=5
 lcp-echo-interval=30
 mru=1500
 mtu=1500
 password-flags=1
 refuse-eap=yes
 require-mppe=yes
 user=<username>
 service-type=org.freedesktop.NetworkManager.l2tp
 
 [ipv4]
 dns-search=
 method=auto
 
 [ipv6]
 addr-gen-mode=stable-privacy
 dns-search=
 ip6-privacy=0
 method=auto
 
 [proxy]
```

- connection って？
  - NetworkManager において接続設定を管理するもの
  - device (ネットワークインタフェース) に connection をアタッチする
- connection.type が vpn
- VPN に必要な設定が vpn 設定下に書かれる

まだ Phase 1, 2 の違いが理解できていなかったり VPN サーバ側で使用するアルゴリズムを修正するべきだったりと疑問や課題は残りますが、ともあれ Linux で NetworkManager を使用した L2TP/IPSec 接続を行うことができました。

(追記)

暗号化アルゴリズムですが NetworkManager-l2tp の [Known Issues][9] により詳細な情報が載っていました。これを見るとまず [Querying VPN server for its IKEv1 algorithm proposals][10] に書かれるようにまず ike-scan を利用して VPN サーバの対応する暗号化アルゴリズムを一覧し、その中から希望のアルゴリズムを設定するのが良さそうです。

最終的には以下のように設定しました。本当はハッシュアルゴリズムは SHA-2 を指定したかったのですが、何故か Phase 2 に指定できずまだもやもやします。

```
ipsec-esp=aes256-sha1            # Phase 2
ipsec-ike=aes256-sha384-modp2048 # Phase 1
```

### SoftEther VPN Client for Linux を使用した接続

上とは別に SoftEther VPN Client を使用した接続の確認も行ったので簡単に紹介しておきます。

インストールは Arch Linux の場合 AUR のパッケージを利用することができます。
それ以外の場合公式ページからダウンロードすることになります。

```
$ yay -S softethervpn
```

VPN Client の設定は Windows 版と違い GUI で行えないので vpccmd を利用してコマンドラインから行いました。このあたりは [SoftEther VPN でPC間接続VPNをする (Client編)][6] が参考になりました。

上の手順により tun ネットワークインタフェースが追加される (ここでは vpn\_vpn) ので専用の NetworkManager connection を用意します。はじめどういう設定にすればいいのかわからなかったのですが、ひとまず `ip address add dev vpn_vpn <ip>` で静的に IP を追加したところ自動的に connection が作成されたので、それを `nmcli connection show <connection>` で確認して参考にしました。

```
$ cat /etc/NetworkManager/system-connections/tun-vpn_vpn.nmconnection
 [connection]
 id=tun-vpn_vpn
 uuid=<uuid>
 type=tun
 interface-name=vpn_vpn
 permissions=
 
 [tun]
 mode=2 # added
 
 [ipv4]
 dns-search=
 method=auto
 route-metric=500 # added
 
 [ipv6]
 addr-gen-mode=stable-privacy
 dns-search=
 method=auto
 
 [proxy]
```

- route-metric
  - この connection で追加するルーティング情報の metric 値を指定する

作成した connection を NetworkManager に認識してもらうには `nmcli connection reload` を実行します。

最後に以下のようなスクリプトを用意して VPN の開始、修了を操作できるようにしました。

```bash
#!/bin/bash

set -eu

readonly NM_CONNECTION=tun-vpn_vpn # 作成した NetworkManager の connection
readonly VPN_SERVER_IP=$(dig +short <vpn_server_host>) # VPN サーバの IP
readonly DEFAULT_INTERFACE=<interface> # 通常時 (VPN 未使用時) インターネット接続に使用しているインタフェース
readonly DEFAULT_GATEWAY_IP=<gateway_ip> # 通常時のデフォルトゲートウェイ

USAGE="Usage: vpn (start|end)"

command=$1

case "$command" in
    "start" )
        systemctl start softethervpn-client
        nmcli connection up $NM_CONNECTION
        ip route add $VPN_SERVER_IP via $DEFAULT_GATEWAY_IP dev $DEFAULT_INTERFACE
        ;;
    "end" )
        ip route del $VPN_SERVER_IP via $DEFAULT_GATEWAY_IP dev $DEFAULT_INTERFACE
        nmcli connection down $NM_CONNECTION
        systemctl stop softethervpn-client
        ;;
    * )
        echo "$USAGE"
        exit 1
        ;;
esac
```

当初の構想では [NetworkManager の dispatcher][7] や connection 中の ipv4.route-data 設定を利用して softethervpn-client が起動すると自動で設定を行うようにしようと思っていたのですが、管理するファイルが増えたり、設定の記述方法が謎だったりしたので単一スクリプトで完結させました。

VPN 開始時は `./vpn start`, 終了時は `./vpn end` を実行します。

VPN 接続確立後、ルーティングテーブルはこんな感じになります。
前節で紹介した方法と同様、インターネットへのトラフィックが VPN 接続先のデフォルトゲートウェイから出ていくようにしています。

```
$ ip route
default via 192.168.1.1 dev vpn_vpn proto dhcp metric 500 # added
default via 192.168.40.1 dev wlp2s0 proto dhcp metric 600 
<vpn server ip> via 192.168.40.1 dev wlp2s0 # added
192.168.1.0/24 dev vpn_vpn proto kernel scope link src 192.168.1.12 metric 500 # added
192.168.40.0/24 dev wlp2s0 proto kernel scope link src 192.168.40.218 metric 600 
```

[1]: https://ja.softether.org/
[2]: https://wiki.archlinux.jp/index.php/NetworkManager
[3]: https://github.com/nm-l2tp/NetworkManager-l2tp
[4]: https://www.komee.org/category/Network?page=1549508400
[5]: https://github.com/nm-l2tp/NetworkManager-l2tp
[6]: https://qiita.com/Daisuke-Otaka/items/b9d99c9dcbb84cf813d7#%E3%82%AF%E3%83%A9%E3%82%A4%E3%82%A2%E3%83%B3%E3%83%88%E3%81%AE%E8%A8%AD%E5%AE%9A%E7%AE%A1%E7%90%86
[7]: https://wiki.archlinux.jp/index.php/NetworkManager#.E3.83.8D.E3.83.83.E3.83.88.E3.83.AF.E3.83.BC.E3.82.AF.E3.82.B5.E3.83.BC.E3.83.93.E3.82.B9.E3.81.A8_NetworkManager_dispatcher
[8]: https://github.com/nm-l2tp/NetworkManager-l2tp/wiki/Known-Issues
[9]: https://github.com/nm-l2tp/NetworkManager-l2tp/wiki/Known-Issues
[10]: https://github.com/nm-l2tp/NetworkManager-l2tp/wiki/Known-Issues#querying-vpn-server-for-its-ikev1-algorithm-proposals
