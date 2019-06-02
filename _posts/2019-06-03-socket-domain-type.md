---
layout: post
title: socket 関数の domain, type パラメータ
tags: "socket, posix"
comments: true
---

最近『基礎からわかるTCP/IP ネットワーク実験プログラミング』という本を読んでいます。TCP, UDP, IP, Ethernet といったプロトコルのパケット構造を理解し簡単なツールを作って実験するというような内容なのですが、その中で socket 関数というのが多様なパラメータとともに出てきます。socket 関数は `socket(int domain, int type, int protocol)` というように 3 つのパラメータを受けとり、それらは man ページで以下のように説明されています。

> domain : Specifies the communications domain in which a socket is to be created.
> 
> type: Specifies the type of socket to be created.
> 
> protocol: Specifies a particular protocol to be used with the socket. 

ただの int 型に多様な意味を持たせ機能豊富な 1 関数を作ってしまうというのは C 言語あるあるなのかもしれませんが初見わかりにくいので、本に出てくるような使い方を元にパラメータの組み合わせを整理してみました。取り上げたのは UDP 通信を想定して以下の 4 つです。

- 1 `socket(PF_PACKET, SOCK_DGRAM, protocol)`
  - `man 7 packet`
  - データリンクヘッダ (例えば Ethernet ヘッダ) を除いたパケットを扱える (例えば IP ヘッダ以降)
  - root 権限が必要
  - protocol には `ether_header` 構造体の `ehter_type` に当たるものを入れる
- 2 `socket(PF_PACKET, SOCK_RAW, protocol)`
  - `man 7 packet`
  - データリンクヘッダを含めたパケットを扱える
  - root 権限が必要
  - protocol には `ether_header` 構造体の `ehter_type` に当たるものを入れる
- 3 `socket(PF_INET, SOCK_DGRAM, 0)`
  - `man 7 udp`
  - UDP で通信する場合に通常使用するのはこれ
  - ユーザとしてはデータのみ気にすればいいい
  - protocol には `ip` 構造体の `ip_p` に当たるものを入れる
- 4 `socket(PF_INET, SOCK_RAW, protocol)`
  - `man 7 ip`
  - IP ヘッダから触ることができる
  - root 権限が必要
  - protocol には `ip` 構造体の `ip_p` に当たるものを入れる

各 socket の使用例は [こちら][2] に置いています。`PF_PACKET` や `SOCK_RAW` を使用した 1, 2, 4 については (他アプリケーションが) 受信した UDP を read して IP とポートを表示するというプログラムになっています。例えば本でいう udpc, udps プログラムがローカルホスト上で通信している内容を読めるという感じです。 3 はポート 5320 で待ち受ける UDP サーバです。

1, 2, 4 のように `PF_PACKET` や `SOCK_RAW` を指定して作成した socket は、あらゆる port 宛のパケットを読めてしまうので root 権限が必要になるという理解です (ポートという概念が TCP, UDP にしかないので、それより下位のレイヤをユーザに触らせる socket だと必然的にそうなるということかと)。

また UDP ではなく TCP を扱いたい場合には type の `SOCK_DGRAM` 指定を `SOCK_STREAM` に変更すればいいはずです。

[1]: https://www.amazon.co.jp/%E5%9F%BA%E7%A4%8E%E3%81%8B%E3%82%89%E3%82%8F%E3%81%8B%E3%82%8BTCP-IP-%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AF%E5%AE%9F%E9%A8%93%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0%E2%80%95Linux-FreeBSD%E5%AF%BE%E5%BF%9C-%E6%9D%91%E5%B1%B1/dp/4274065847
[2]: https://github.com/tiqwab/example/tree/master/sample-socket
