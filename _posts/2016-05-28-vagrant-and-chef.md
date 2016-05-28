---
layout: post
title:  "VagrantとChef連携"
tags: "vagrant, chef"
---

VagrantとChefを連携して使用するための手順をまとめる。

1. [Vagrantプラグインのインストール](#anchor1)
2. [仮想マシン作成からchef-clientインストールまで](#anchor2)
3. [cookbookの用意](#anchor3)
4. [vagrantとchefの連携](#anchor4)

### 環境

- OS: Mac OS 10.11.4
- VirtualBox: 5.0.20
- Vagrant: 1.8.1
- Chef Development Kit: 0.13.21

---

<a id="anchor1"></a>

### 1. Vagrantプラグインのインストール

[こちら][1]に従い、Vagrantプラグインのインストールを行う。

```bash
$ vagrant plugin install vagrant-omnibus
$ vagrant plugin install vagrant-chef-zero
$ vagrant plugin install vagrant-vbox-snapshot

$ vagrant plugin list
vagrant-chef-zero (2.0.0)
vagrant-omnibus (1.4.1)
vagrant-share (1.1.5, system)
vagrant-vbox-snapshot (0.0.10)
```

- vagrant-omnibus
  - 任意のversionのChefを任意の仮想マシンproviderにインストールするためのプラグイン
- vagrant-chef-zero
  - Vagrant起動、終了時にローカル(ホスト側)でのChef-zeroサーバの自動的な立ち上げ、終了を行う。
- vagrant-vbox-snapshot
  - Vagrantで作成した仮想マシンのsnapshotを簡単に管理するためのもの
  - Chefとは関係の無いプラグインだが便利そうなために追加
  - `vagrant snapshot take <snapshot_name>`でsnapshot作成
  - `vagrant snapshot go <snapshot_name>`で仮想マシンを指定のsnapshot状態に戻す

<a id="anchor2"></a>

### 2. 仮想マシン作成からchef-clientインストールまで

Boxのインストール後、`Vagrantfile`に`config.omnibus.chef_version=:latest`を追記し、仮想マシン上にchef-clientがインストールされていることを確認する。

```bash
# Use local box as listed
$ vagrant box list
CentOS_6.4_x86_64_Minimal (virtualbox, 0)
centos-7.2                (virtualbox, 0)

# Generate Vagrantfile
$ vagrant init centos-7.2

$ vim Vagrantfile
$ tail -n 5 Vagrantfile
Vagrant.configure(2) do |config|
  config.vm.box = 'centos-7.2'
  config.vm.network :private_network, ip: '192.168.33.33'
  # Comment out since vagrant cannot up with the below line...
  # config.omnibus.chef_version = :latest
end

# Create vm
$ vagrant up

# Provision after removing comment out above
$ vagrant provision

# SSH to vm
$ vagrant ssh

[vagrant@localhost ~]$ chef-client -v
Chef: 12.10.24
```

<a id="anchor3"></a>

### 3. cookbookの用意

仮想マシンに適用するためのcookbookを作成する。今回の例では新規に`my_cookbook`というcookbookを作成し、その内部でChef Supermarket上の`nodejs` cookbookを使用し、Node.jsのインストールを行わせる。

- 自作cookbookの依存関係は`metadata.rb`で記述
- 依存関係上のrecipeの実行は`include_recipe <hoge>`という感じ

```bash
# Create chef repository
# 'cookbooks' directory stores installed cookbooks, whereas 'site-cookbooks' original
$ chef generate repo chef-repo
$ mkdir chef-repo/site-cookbooks
$ cd chef-repo/site-cookbooks

# Create cookbook
$ chef generate cookbook my_cookbook
$ cd my_cookbook

# Set dependency
$ vim metadata.rb
$ cat !$
name 'my_cookbook'
maintainer 'The Authors'
maintainer_email 'you@example.com'
license 'all_rights'
description 'Installs/Configures my_cookbook'
long_description 'Installs/Configures my_cookbook'
version '0.1.0'

depends 'nodejs', '~> 2.4.4'

# Edit recipe
$ vim recipes/default.rb
$ cat !$
include_recipe 'nodejs::nodejs_from_package'
```

<a id="anchor4"></a>

### 4. vagrantとchefの連携

上記で作成したcookbookをVagrantで作成した仮想マシンに適用した後、Node.jsが仮想マシンにインストールされていることを確認する。

- ルート(`Vagrantfile`と同じ階層)に`Berksfile`を作成する。
  - cookbookのダウンロードに使用。
  - provision実行時に使用するわけではないので、依存分も含めてcookbookを用意できるなら無くてもよい。
- `Vagrantfile`でprovisionの設定を行う。
  - `cookbooks_path`プロパティでchef-zeroサーバにcookbookリポジトリのパスを認識させる。
  - `nodes_path`プロパティは、(詳しくないけれど)今回の用途では特に気にしなくてよい。
  - 実行したいroleやrecipeは`roles_path`, `add_recipe`で指定する。

```bash
# Create Berksfile
$ vim Berksfile
$ cat !$
source 'https://supermarket.chef.io'

cookbook 'my_cookbook', path: './chef-repo/site-cookbooks/my_cookbook'

# Install cookbooks
$ berks vendor chef-repo/cookbooks

# Edit Vagrantfile to configure provision
$ vim Vagrantfile
$ cat !$
Vagrant.configure(2) do |config|
  config.vm.box = 'centos-7.2'
  config.vm.network :private_network, ip: '192.168.33.33'

  config.omnibus.chef_version = :latest

  config.vm.provision :chef_zero do |chef|
    chef.cookbooks_path = ['./chef-repo/cookbooks']
    chef.nodes_path = './chef-repo/nodes'
    chef.roles_path = './chef-repo/roles'
    chef.add_recipe 'my_cookbook::default'
  end
end

# '--provision' option required! (refer to [1])
# Use ulimit command if EMFILE error occurs (refer to Memo below)
$ vagrant reload --provision

# Confirm installation of nodejs
$ vagrant ssh
[vagrant@localhost ~]$ node -v
v0.10.42

# Directory structure (exclude 'cookbooks' dir)
$ tree -I cookbooks .
.
├── Berksfile
├── Berksfile.lock
├── Vagrantfile
└── chef-repo
    ├── LICENSE
    ├── README.md
    ├── chefignore
    ├── data_bags
    │   ├── README.md
    │   └── example
    │       └── example_item.json
    ├── environments
    │   ├── README.md
    │   └── example.json
    ├── nodes
    │   └── vagrant-50d182c8.json
    ├── roles
    │   ├── README.md
    │   └── example.json
    └── site-cookbooks
        └── my_cookbook
            ├── Berksfile
            ├── Berksfile.lock
            ├── README.md
            ├── chefignore
            ├── metadata.rb
            ├── recipes
            │   └── default.rb
            ├── spec
            │   ├── spec_helper.rb
            │   └── unit
            │       └── recipes
            │           └── default_spec.rb
            └── test
                └── integration
                    ├── default
                    │   └── serverspec
                    │       └── default_spec.rb
                    └── helpers
                        └── serverspec
                            └── spec_helper.rb
```

### 印象

- Vagrantで仮想マシンを簡単に作成できる。
- Chefで既存のcookbook等を使いながら簡単に仮想マシンのprovisionを行うことができる。
- 加えてvagrant-vbox-snapshotプラグインを使えば、気軽に仮想マシンの状態を管理できる。

ということで、軽く触っている感じ、とても使い易い連携になっている。

### Memo

- `vagrant reload`時に`Too many open files - getcwd (Errno::EMFILE)`エラーが出た場合は、`ulimit -n 1024`とか値を増やして実行する。[参照][3]

### 参考

- [初めての Vagrant + Chef zero + Berkshelf][1]
- [VagrantとChefによる仮想環境構築の自動化 (VirtualBox編) \| オブジェクトの広場][2]

[1]: http://qiita.com/tsuyopooon/items/d90679d9f8b5ccfcfde5
[2]: https://www.ogis-ri.co.jp/otc/hiroba/technical/vagrant-chef/chap1.html
[3]: http://spring-mt.tumblr.com/post/27041672218/too-many-open-files-getcwd-errnoemfile%E3%81%A7%E3%83%8F%E3%83%9E%E3%81%A3%E3%81%9F%E4%BB%B6
