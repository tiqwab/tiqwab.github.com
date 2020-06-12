---
layout: post
title: 自宅 Wi-Fi ネットワークのパケットキャプチャ
tags: "wifi, network"
comments: true
---

IoT だったりスマート家電だったりといった概念が普及するにつれ、自宅 Wi-Fi ネットワークに新しいデバイスをお出迎えする機会が増えていると思います。そうするとときどき「このデバイスはいつどんな通信を行っているのだろうか」ということが気になってきます。デバイスの通信の挙動を理解する方法の一つはパケットキャプチャだと思うのですが、それを Wi-Fi ネットワークを対象におこなったことがなかったので試しにやってみることにしました。

### Wi-Fi アダプタの用意

Wi-Fi ネットワークを流れる、かつ自分が送信元、送信先ではないパケットをキャプチャするためには、「モニターモード」と呼ばれる設定をサポートする Wi-Fi カードが必要になります。

イーサネット LAN 内で同様のパケットキャプチャを行いたいという場合、[tcpdump がそうする][6]ようにネットワークインタフェースを「[プロミスキャスモード][5]」に設定します。通常ネットワークインタフェースは自分宛ではないフレームをドロップしますが、プロミスキャスモードでは受信した全てのパケットを OS やアプリケーションへ引き渡すようになります。

例えば Linux 系列の OS では以下のコマンドで対象ネットワークインタフェースをプロミスキャスモードに設定できます。

```shell
# ref. https://www.thegeekdiary.com/how-to-configure-interface-in-promiscuous-mode-in-centos-rhel/
$ ip link set <interface> promisc on
```

同様のコマンドで Wi-Fi ネットワークインタフェースもプロミスキャスモードにできれば話は単純なのですが、実際は大抵の Wi-Fi チップやドライバではプロミスキャスモードがサポートされていない or 想定した通りには動かないようです。

代わりに必要となるのがモニターモードと呼ばれる設定です。Wi-Fi ネットワークインタフェースをモニターモードに設定することでイーサネット LAN におけるプロミスキャスモードと同様にネットワークを流れるパケットのキャプチャが可能になります (Wi-Fi インタフェースにおいてプロミスキャスモードとモニタモードがどう違うかはあまり明確ではなく使用する Wi-Fi カード依存っぽいです。基本的にはモニターモードを有効にしないといけない、という感じでプロミスキャスモードについては気にしなくて良さそうに思っています)。

モニターモードを有効にする方法については [CaputreSetup/WLAN - The Wireshark Wiki][7] で記述されているように、環境に強く依存します。自分は Dell の XPS13 (9380) に Arch Linux を入れて使用しており、付属の Wi-Fi カードではモニタモードがサポートされていないので、[WIFIアダプターをモニターモードにする(WIFIクラッキングに必須)][2] を見ておすすめされていた [Alfa AWUS036NHA][1] という Wi-Fi アダプタを購入しました。この Wi-Fi アダプタは自分の用途では十分なものだったのですが、あとから気付いた注意点も 2 つほどありました。

- 対応している Wi-Fi 規格が IEEE 802.11 b/g/n で 2.4 Ghz のみ
  - なので対象ネットワークが 5.0 Ghz なら用途に合わない
- MIMO 対応は 1x1 のみ (恐らく)
  - なので例えば対象 Wi-Fi ルータ、クライアントデバイスが 2x2 で通信しているとうまくキャプチャができない (後述)

参考:

- [WIFIアダプターをモニターモードにする(WIFIクラッキングに必須)][2]
  - モニターモードについてや適切な Wi-Fi アダプタの選定について
- [CaputreSetup/WLAN - The Wireshark Wiki][7]
  - Turning on monitor mode 項がモニターモードを設定する方法について詳しい

### 環境構築

Wi-Fi ネットワークのパケットキャプチャを行う環境は必要なツールが揃っている Kali Linux が便利そうなので Vagrant で用意しました。公式で box も用意されています。

```ruby
# Vagrantfile

# -*- mode: ruby -*-
# vi: set ft=ruby :

# ref. https://www.kali.org/news/announcing-kali-for-vagrant/

Vagrant.configure("2") do |config|
  config.vm.box = "kalilinux/rolling"
end
```

VM に Wi-Fi アダプタを認識させるためには VirtualBox の extension pack が必要 (正しくは USB2.0/3.0 デバイスを認識させるために必要?) なのでインストールしておきます。自分は Arch Linux の AUR として提供されているものがあったのでそちらから用意しました (ref. [VirtualBox - ArchWiki][8])

VM を起動したあとは [WIFIアダプターをモニターモードにする(WIFIクラッキングに必須)][2] に従ってモニタモードの動作を確認します。ここでは内容が重複するので使用したコマンドだけまとめておきます。

```bash
# set wlan0 as monitor mode
$ sudo airmon-ng check kill
$ sudo airmon-ng start wlan0

# check whether wlan0 is in monitor mode
# in this case, wlan0 interface disappears and wlan0mon appears instead
$ sudo iwconfig

# list available APs (Access Points)
$ sudo airodump-ng wlan0mon

# monitor the paticular AP
$ sudo airodump-ng --bssid <bssid> wlan0mon
```

これでモニタモードへの移行が確認できれば準備 OK です。

ちなみに全く原因がわからないのですが、Wi-Fi アダプタをゲスト OS で認識させているとゲスト OS 終了時にホスト OS がフリーズすることがありました。どうしようもなくてゲスト OS 終了前に物理的に Wi-Fi アダプタを引っこ抜いて対応していましたが、荒療治すぎるのでどうにかしたいところです。

参考:

- [WIFIアダプターをモニターモードにする(WIFIクラッキングに必須)][2]
  - 環境構築、モニタモードの設定方法について

### パケットキャプチャ手順

パケットキャプチャには Wireshark を使用します。上で用意した wlan0mon のようなモニタモードを有効化したインタフェースを選択すれば Probe Request, Probe Response のような暗号化されていない通信はすぐに見られるようになります。

ただネットワーク上を流れるデータパケットは通常暗号化されているので、その内容を見るためには復号化のための情報を Wireshark に教える必要があります。細かい部分は Wireshark のバージョンや OS で異なりますが、以下のような操作を行います。

1. \[View\] -> \[Wireless Toolbar\] を選択
2. 表示されたツールバーから \[802.11 Preferences\] を選択
3. \[Enable Decryption\]  にチェックを入れ、\[Decryption Keys\] を選択
4. キーの情報を入力

入力するキーの情報は Wi-Fi ネットワークで使用されている暗号方式に依ります。今回は WPS2-PSK だったので、key type に wpa-psk、key に [こちら][9] で生成した情報を設定しました。

また WPA-PSK では対象クライアント毎に WPA キー (上で用意した) や MAC アドレス、乱数から暗号化鍵を生成する (ref. [Wiresharkで公衆無線LANのヤバさを確認してみた][11]) ので、対象デバイスが送受信するパケットを復号化するためにはルータとの間で行われる EAPOL と呼ばれる 4-way handshake をキャプチャしてその情報を取得する必要があります。EAPOL でフィルタリングして Message 1 of 4 から 4 of 4 までのパケットが確認できれば OK です。

復号できればあとは流れるパケットを眺めます。フィルタとしては以下の 2 つをよく使いました。

- `wlan.bssid == <bssid>`
  - AP 毎に持つユニークな ID
  - このフィルタを入れないとあらゆる AP からのパケットをキャプチャしてしまう
  - 確認方法は色々あると思うが、自分は airodump-ng を使用した
  - ちなみに普段 SSID と呼ぶものは essid であり、ユニークとは限らない
- `wlan.addr == <mac>`
  - パケットの送受信先によるフィルタリング
  - 通信を観察したいクライアントの MAC アドレスを指定する

便利なツールのおかげで、 Wi-Fi ネットワークであってもパケットキャプチャリングはそんなに難しくないのですが、2 つハマった点があったので記述しておきます。

- チャンネルを合わせる
  - EAPOL パケットが取れなくて困っていたらチャンネルがずれていた
  - AP が使用するチャンネルはルータの設定画面だったり airodump-ng からわかる
    - なおパケットキャプチャリング中は airodump-ng を止めるべき。というのはデフォルトだと (チャンネル指定しないと) AP を検出するためにチャンネルホッピングするので。
  - `iwconfig <interface> channel <ch>` で設定する
- クライアント、ルータの MIMO 対応規格を確認する
  - はじめ iPad Pro をクライアントにして動作確認していたが、なぜかブロードキャストパケットしか見えなかった
  - [全く同じ現象][13]を見つけて、それに従い古いスマートフォンで試すとうまくいった
  - 確証はないが状況証拠からクライアント、ルータ間は MIMO (2x2) で通信しており、一方で Wi-Fi アダプタが 1x1 にしか対応していなかったので、うまくキャプチャができなかったのだと考えている

参考:

- [WLAN (IEEE 802.11) capture setup - The Wireshark Wiki][4]
  - Wireshark を使用した Wi-Fi ネットワーク上のパケットキャプチャ方法について
- [How to Decrypt 802.11 - The Wireshark Wiki][10]
  - Wireshark での WPA パケットの復号化方法について
- [Tutorial: WPA Packet Capture Explained - Aircrack-ng][3]
  - WPA パケットのキャプチャ例。このうち EAPOL パケット 4 つが復号化のために重要
- [Wiresharkで公衆無線LANのヤバさを確認してみた][11]
  - パケットキャプチャの具体例
- [無線LANのパケットを手っ取り早くキャプチャする方法メモ][12]
  - パケットキャプチャの具体例

### Nature Remo の初期設定処理の観察

最後に具体例として先日自宅にお出迎えした [Nature Remo mini][14] (以下 Remo) の初期設定処理をパケットキャプチャで観察します。[Nature Remo 初期設定マニュアル][15]でいうと「2. Nature Remoの初期設定」にあたる部分で、アプリを起動したスマートフォンと Remo 間の通信を観察したものになります。

まずは Remo が AP を提供しスマートフォンから接続します (マニュアルの「4. Nature Remoのネットワークに接続する」)。このときパケットキャプチャができていれば以下のように一連の EAPOL パケットを確認できます (MAC アドレスを伏せていますが、ピンクがルータ、黄土色がスマートフォンのそれです)。

<figure>
  <img
    src="/images/packet-capture-wifi/eapol.png"
    title="eapol packets"
    alt="eapol packets"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>

EAPOL がうまく取得できたので以降の通信の内容を確認できるようになりました。確認された通信の一部は以下のようになっています (一応 authorization は伏せています)。

<figure>
  <img
    src="/images/packet-capture-wifi/http.png"
    title="http packets between the app and Nature Remo"
    alt="http packets between the app and Nature Remo"
    style="display: block; margin: 0 auto; border: 1px solid #eee"
    width="75%"
  />
</figure>

アプリ側からはまず `http://remo.local/scan` に対して HTTP POST リクエストを送信していました。レスポンスから推測するとどうやら Remo から検出できる AP の一覧を JSON として提供しているようです。
またアプリ側から定期的に `http://remo.local/status` に GET リクエストを送信しているのも確認できました。これは Remo の生存確認をしているのだと思います。

マニュアルの 6, 7 ではスマートフォンから自宅の Wi-Fi を選択します。そのあとに TLSv1.2 による通信が発生します。この内容は暗号化されているので確認できませんが、恐らくここで Remo に接続すべき SSID やその認証情報を渡していそうです。TLS で使用される証明書は issuer が ca.nature.global, subject が remo.local となっているのでプライベート認証局 (ca.nature.global) で署名したサーバ証明書を Remo に持たせているという感じだと思います。

いまどき外部サーバとの通信は TLS 等で暗号化されているのでパケットキャプチャしてもあまり多くの情報は得られませんが、少なくとも通信するタイミングや頻度ぐらいは理解できるはずです。

[1]: https://www.ebay.com/itm/Alfa-AWUS036NHA-802-11n-Wireless-USB-Adapter-Atheros-chip-AR9271L-Suction-CLIP-/202524651963
[2]: https://medium.com/ethical-hacking-%E3%83%9B%E3%83%AF%E3%82%A4%E3%83%88%E3%83%8F%E3%83%83%E3%82%AB%E3%83%BC/wifi%E3%82%A2%E3%83%80%E3%83%97%E3%82%BF%E3%83%BC%E3%82%92%E3%83%A2%E3%83%8B%E3%82%BF%E3%83%BC%E3%83%A2%E3%83%BC%E3%83%89%E3%81%AB%E3%81%99%E3%82%8B-%E6%9C%80%E9%87%8D%E8%A6%81%E3%82%B9%E3%82%AD%E3%83%AB-5ee7a57e5165
[3]: https://www.aircrack-ng.org/doku.php?id=wpa_capture
[4]: https://wiki.wireshark.org/CaptureSetup/WLAN
[5]: https://ja.wikipedia.org/wiki/%E3%83%97%E3%83%AD%E3%83%9F%E3%82%B9%E3%82%AD%E3%83%A3%E3%82%B9%E3%83%BB%E3%83%A2%E3%83%BC%E3%83%89
[6]: https://thinkit.co.jp/article/721/1?nopaging=1
[7]: https://wiki.wireshark.org/CaptureSetup/WLAN#Turning_on_monitor_mode
[8]: https://wiki.archlinux.jp/index.php/VirtualBox#.E3.82.A8.E3.82.AF.E3.82.B9.E3.83.86.E3.83.B3.E3.82.B7.E3.83.A7.E3.83.B3.E3.83.91.E3.83.83.E3.82.AF
[9]: https://www.wireshark.org/tools/wpa-psk.html
[10]: https://wiki.wireshark.org/HowToDecrypt802.11
[11]: https://uwazumi.honeniq.net/2013/11/21/wireshark-public-wifi.html
[12]: http://blog.bogus.jp/archives/993/comment-page-1
[13]: https://superuser.com/questions/1474962/only-seeing-broadcast-traffic-in-wifi-captures
[14]: https://nature.global/jp/nature-remo
[15]: https://nature.global/jp/faq/setup-manual
