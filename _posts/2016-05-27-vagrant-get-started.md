---
layout: post
title:  "Vagrant入門"
tags: "vagrant"
---

仮想マシンを手軽に作成、管理するためにVagrantを使用してみる。

1. [VirtualBox, Vagrantのインストール](#anchor1)
2. [ボックスファイルの入手](#anchor2)
3. [Vagrantfileの作成](#anchor3)
4. [仮想マシン停止、削除](#anchor4)

### 環境

- OS: Mac OS 10.11.4
- VirtualBox: 5.0.20
- Vagrant: 1.8.1

---

<a id="anchor1"></a>

### 1. VirtualBox, Vagrantのインストール

homebrewでインストール。

```bash
$ brew cask install virtualbox
$ brew cask install vagrant
```

<a id="anchor2"></a>

### 2. ボックスファイルの入手

Vagrantで仮想マシンを用意するときは入手した(あるいは作成した)ボックスファイルを用意する必要がある。これは、[こちら][1]の「物理PCにおけるインストールディスクを用意する、に近い」という表現がしっくりくる。
オーソドックスな手順としては、以下のように目的のボックスファイルを[Atlas by HasiCorp][3]等から探し、URLを添えて`vagrant box add`する。

```bash
vagrant box add CentOS_6.4_x86_64_Minimal http://developer.nrel.gov/downloads/vagrant-boxes/CentOS-6.4-x86_64-v20131103.box
```

上記コマンドで対象ボックスファイルのダウンロードが開始される。
ただし、ダウンロードに時間がかかりタイムアウトすることがあるので、そのときは直接アドレス叩いてローカルに保存したboxを、`vagrand box add`するのが吉。  

完了後、`vagrant box list`でダウンロード済みのbox一覧が確認でき、それらは`~/.vagrant.d/boxes`に保存されていることも確認できる。

```bash
$ vagrant box list
CentOS_6.4_x86_64_Minimal (virtualbox, 0)

$ ls ~/.vagrant.d/boxes
CentOS_6.4_x86_64_Minimal
```

<a id="anchor3"></a>

### 3. Vagrantfileの作成

仮想マシンの設定は`Vagrantfile`というファイルに記述する。仮想マシンを作成したいディレクトリに移動し、下記コマンドで`Vagrantfile`を作成する。

```bash
$ vagrant init CentOS_6.4_x86_64_Minimal
```

これで以下のような`Vagrantfile`が作成され、指定のboxをもとに仮想マシンが作成されることになる。

```ruby
Vagrant.configure(2) do |config|
  config.vm.box = "CentOS_6.4_x86_64_Minimal"
end
```

<a id="anchor4"></a>

### 4. 仮想マシン起動

上で作成した`Vagrantfile`を使用して、仮想マシンを作成、起動、そして接続してみる。  

```bash
# Create vm
$ vagrant up

# SSH to vm
$ vagrant ssh
Welcome to your Vagrant-built virtual machine.
[vagrant@localhost ~]$
```

恐ろしく簡単に仮想マシンを用意することができた。

<a id="anchor5"></a>

### 5. 仮想マシン停止、削除

仮想マシンの停止、削除は以下のコマンドで行うことができる。

```bash
# Stop vm
$ vagrant halt

# Destroy vm
$ vagrant destroy
```

### 印象

- 仮想マシンを作成、管理する負担が激減して素晴らしい。
- Chefのようなprovision toolと連携させることも多いらしいので、それも試してみる。

### 参考

- [仮想環境を CUI（コマンドライン）でいじれる Vagrant を試してみた][1]
- [Vagrant入門。簡単にローカル環境に仮想マシン(VM)を作る][2]

[1]: http://girigiribauer.com/archives/966
[2]: http://ruby-rails.hatenadiary.com/entry/20150719/1437249020
[3]: https://atlas.hashicorp.com/boxes/search
