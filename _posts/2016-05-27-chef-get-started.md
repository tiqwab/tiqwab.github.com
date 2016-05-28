---
layout: post
title:  "Chef入門"
tags: "chef"
---

Vagrantでのprovisionerとしての使用を想定しつつ、Chefの基本的な操作を[チュートリアル][1]に従って確認。また、公式のドキュメントを流し見して、全体の概要をまとめてみる。

1. [環境確認](#anchor1)
2. [とりあえず動かしてみる](#anchor2)
3. [とりあえずパッケージをインストールしてみる](#anchor3)
4. [Chefに関するMemo](#anchor4)

### 環境

- OS: Mac OS 10.11.4
- VirtualBox: 5.0.20
- Vagrant: 1.8.1
- chefdk: 0.13.21

---

<a id="anchor1"></a>

### 1. 環境確認

Vagrantで構築したCentOSの仮想マシンを使用する。今回はすでにChefのインストールされているボックスファイルを選択。

```bash
[vagrant@localhost ~]$ uname -a
Linux localhost.localdomain 2.6.32-358.el6.x86_64 #1 SMP Fri Feb 22 00:31:26 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

[vagrant@localhost ~]$ cat /etc/redhat-release
CentOS release 6.4 (Final)

[vagrant@localhost ~]$ chef-client -v
Chef: 11.8.0
```

<a id="anchor2"></a>

### 2. とりあえず動かしてみる

[公式のチュートリアル][1]を見ながら進めようとしたが、`chef-client --local-mode hello.rb`を実行しても`hello.rb`が認識されなくてしょっぱなから躓く。色々調べて以下のようなディレクトリ構成、コマンドを書き換えてから動作することを確認。

```bash
# Cookbooks structure
$ tree cookbooks
cookbooks
├── helloworld (called as 'Cookbook')
│   └── recipes
│       ├── default.rb (called as 'Recipe')
│       └── renamed.rb (called as 'Recipe')
└── motd (called as 'Cookbook')
    └── recipes
        └── default.rb (called as 'Recipe')
```

```
# First recipe
[vagrant@localhost cookbooks]$ cat helloworld/recipes/default.rb
file '/tmp/motd' do
  content 'hello world'
end

# '-z' option means '--local-mode', '-o' name of recipe
[vagrant@localhost cookbooks]$ chef-client -z -o helloworld
[2016-05-18T13:12:43+00:00] WARN: No config file found or specified on command line, using command line options.
Starting Chef Client, version 11.8.0
[2016-05-18T13:12:43+00:00] WARN: Run List override has been provided.
[2016-05-18T13:12:43+00:00] WARN: Original Run List: [recipe[helloworld]]
[2016-05-18T13:12:43+00:00] WARN: Overridden Run List: [recipe[helloworld]]
resolving cookbooks for run list: ["helloworld"]
Synchronizing Cookbooks:
  - helloworld
Compiling Cookbooks...
Converging 1 resources
Recipe: helloworld::default
  * file[/tmp/motd] action create
    - create new file /tmp/motd
    - update content in file /tmp/motd from none to b94d27
        --- /tmp/motd	2016-05-18 13:12:43.722662694 +0000
        +++ /tmp/.motd20160518-4058-19d2ftv	2016-05-18 13:12:43.724662693 +0000
        @@ -1 +1,2 @@
        +hello world

Chef Client finished, 1 resources updated

# Second recipe
[vagrant@localhost cookbooks]$ cat helloworld/recipes/renamed.rb
file '/tmp/motd' do
  content 'hello hello world'
end

[vagrant@localhost cookbooks]$ chef-client -z -o helloworld::renamed
[2016-05-18T13:16:36+00:00] WARN: No config file found or specified on command line, using command line options.
Starting Chef Client, version 11.8.0
[2016-05-18T13:16:37+00:00] WARN: Run List override has been provided.
[2016-05-18T13:16:37+00:00] WARN: Original Run List: [recipe[helloworld]]
[2016-05-18T13:16:37+00:00] WARN: Overridden Run List: [recipe[helloworld::renamed]]
resolving cookbooks for run list: ["helloworld::renamed"]
Synchronizing Cookbooks:
  - helloworld
Compiling Cookbooks...
Converging 1 resources
Recipe: helloworld::renamed
  * file[/tmp/motd] action create
    - update content in file /tmp/motd from b94d27 to 1324b3
        --- /tmp/motd	2016-05-18 13:12:43.724662693 +0000
        +++ /tmp/.motd20160518-4313-yq22rf	2016-05-18 13:16:37.179662468 +0000
        @@ -1,2 +1,2 @@
        -hello world
        +hello hello world

Chef Client finished, 1 resources updated
```

<a id="anchor3"></a>

### 3. とりあえずパッケージをインストールしてみる

チュートリアルに従って'httpd'パッケージをインストールしてみる。

```
[vagrant@localhost cookbooks]$ cat helloworld/recipes/webserver.rb
package 'httpd' do
  action :install
end

[vagrant@localhost cookbooks]$ sudo chef-client -z -o helloworld::webserver
[2016-05-18T13:43:12+00:00] WARN: No config file found or specified on command line, using command line options.
Starting Chef Client, version 11.8.0
[2016-05-18T13:43:13+00:00] WARN: Run List override has been provided.
[2016-05-18T13:43:13+00:00] WARN: Original Run List: [recipe[helloworld]]
[2016-05-18T13:43:13+00:00] WARN: Overridden Run List: [recipe[helloworld::webserver]]
resolving cookbooks for run list: ["helloworld::webserver"]
Synchronizing Cookbooks:
  - helloworld
Compiling Cookbooks...
Converging 1 resources
Recipe: helloworld::webserver
  * package[httpd] action install
    - install version 2.2.15-47.el6.centos.4 of package httpd

Chef Client finished, 1 resources updated

[vagrant@localhost cookbooks]$ httpd -v
Server version: Apache/2.2.15 (Unix)
Server built:   Mar 22 2016 19:03:53
```

簡単にインストールできたので、起動、`index.html`表示と流れるようにいけると思いきややはり躓く。どうやらiptables周りが原因なようだったので、一旦OFFにしてhttpサーバの動作を確認することにする。

```
[vagrant@localhost cookbooks]$ cat helloworld/recipes/webserver.rb
service 'iptables' do
  action [:stop, :disable]
end

package 'httpd' do
  action :install
end

# chkconfig httpd on
# service httpd start
service 'httpd' do
  action [:enable, :start]
end

# Source file located in helloworld/templates/default
template '/var/www/html/index.html' do
  source 'index.html.erb'
  mode 0644
end

[vagrant@localhost cookbooks]$ sudo chef-client -z -o helloworld::webserver
...
```

httpサーバが起動しており、ホスト側から`curl 192.168.33.10`で目的の`index.html`にアクセスできることを確認。

---

<a id="anchor4"></a>

### 4. Chefに関するMemo

公式のチュートリアルやドキュメントを触り読みながらChefに関する概要をメモ書き。

### Chef全体

- 処理の単位には`Cookbook`, `Recipe`, `Resource`というものがある。
  - `Resource`が最も最小単位であり、1つの動作を表す。上でいう`file`
  - 'Recipe'は'Resource'をまとめたもの。上でいう`default.rb`, `renamed.rb`
  - 'Cookbook'は`Recipe`をまとめたもの。上でいう`helloworld`
- recipe名のデフォルトは`default.rb`のよう。違うrecipeを指定したい時は`<cookbook>::<recipe>`という記述となる。

### Chefを構成するマシン

基本的にChefはクライアント-サーバ構成を持つ。マシンの主要な役割と関係するツールは以下のような感じ。

- Chef server
  - clientが使用するcookbooksなどを管理するマシン
- Nodes(Chef client)
  - Chefによるprovisionを行いたい対象マシン
  - chef-clientを使用
- Workstation
  - cookbooksの作成などを行い、サーバにアップする作業用マシン
  - (単一、あるいは複数の)cookbookをリポジトリ(chef-repoと表記されることが多い)にまとめ、これをgit等でバージョン管理を行うのを推奨
  - [Chef Supermarket][3]に多数のcookbookが登録されている
  - chef, kitchen, knife, chefspec, berkshelf等を使用
  - chefdkをインストールすれば一式揃う

### Workstation上での主要な操作

#### chef

- `chef generate repo <repository_name>`
  - chef-repoの作成。`cookbooks/`を含むディレクトリ構成を持つ。
- `chef generate cookbook <cookbook_name>`
  - cookbookの作成。様々なファイルを作って含めてくれる。recipeとしては`recipes/default.rb`が自動で生成される。
- `chef generate recipe <path_to_cookbook> <recipe_name>`
  - recipeの作成。`<path_to_cookbook>/recipes/<recipe_name>.rb`等が作成される。
- `chef generate template <path_to_cookbook> <template_name>`
  - templateの作成。 `<path_to_cookbook>/templates/default/<template_name>.erb`が作成される。
  - `.erb`形式なので`<%= var %>`, `<% proc %>`等の記述で変数の埋込や処理の実行を行える。
- `chef generate attribute <path_to_cookbook> <attribute_name>`
  - attributeファイルの作成。`<path_to_cookbook>/attributes/<attribute_name>.rb`が作成される。
  - Chefでのattributeの概念は結構複雑で、異なる箇所に、幾つかの優先度を持たせて書くことができる。
  - とりあえずrecipeに値をベタ書きしたくない、という程度なら`default.rb`でattributeを定義してあげれば十分そう。

#### kitchen

テスト用仮想マシンの管理を行うツール。自作のcookbookの動作確認を行うために使用する。`.kitchen.yml`が設定ファイルとなる。

- `kitchen list`
  - テスト用仮想マシン一覧の表示。
- `kitchen converge`
  - テスト用仮想マシンの作成。
  - `<path_to_cookbook>/.kitchen.yml`を参照して仮想マシンを作成し、cookbookを適用する。
- `kitchen login`
  - 作成したテスト用仮想マシンへのログイン。
- `kitchen destroy`
  - テスト用仮想マシンの破棄。

`.kitchen.yml`の記述例は以下の通り。

```yml
driver:
  name: vagrant
  network:
      - ["private_network", {ip: "192.168.33.33"}]

provisioner:
  name: chef_zero

# Set driver.box if want to use local box
platforms:
  - name: centos-7.2
    driver:
        box: centos-7.2

suites:
  - name: default
    run_list:
      - recipe[awesome_customers::default]
    attributes:
```

#### Berkshelf

cookbookのインストールおよび依存関係を解決するためのツール。  
`Berksfile`が設定ファイル。

- `berks install`
  - `Berksfile`に記載したcookbookを依存関係を解決しつつダウンロードする。デフォルトのダウンロード先は`~/.berkshelf`となる。
  - 依存cookbookは`Berksfile`ではなく`metadata.rb`に記載するべき？ [参照][5]
    - 多分そもそも2つは用途が異なる。以下は現在の認識であり、間違いがあるかも。
    - `metadata.rb`に含めるということは、cookbookのメタデータの一つになるということだといえる。もしcookbookのディレクトリ下で`berks`コマンドを実行すれば、その依存関係を解決して全ての必要なcookbookをダウンロードしてくれるが、それが主目的で依存関係を`metadata.rb`に書くわけではない(と思う)。
    - `Berksfile`は純粋にcookbookを集めるためのもの。特定のcookbookに関わったりするものではない(はず)。
- `berks vendor <path_to_repo>`
  - 指定したパスにcookbookをダウンロードする。
  - Vagrantのprovisionとして用いたりするときに使っているのをよく見る。

### Policyに関して

Chefを実行する際の設定等を管理する仕組みというイメージ。例えば、あるnodeに対して'web server'というroleを与え、"development"というenvironmentでChefを実行させることで、開発時のWeb Server用マシンを構築する、というような。  
Policyとしては以下の3つが含まれる。

- Role
  - 対象のサーバタイプを決定する。 e.g. 'web server', 'database server'
  - attributeやrun-listが定義できる
- Environment
  - 対象の開発プロセスを決定する。 e.g. 'development', 'production'
  - attributeをプロセスによって書き換えるような使い方
- Data bags
  - パスワードのようなセキュアに管理したい設定を保管する仕組み

#### Role

- 設定ファイルは`<path_to_cookbook>/roles/<name_of_role>.json`
- `rb`形式でも設定できるらしいが、未確認。
- roleの定義ファイルは以下のような感じ。

```bash
$ cat roles/web.json
{
    "name": "web",
    "description": "Web role",
    "chef_type": "role",
    "json_class": "Chef::Role",
    "default_attributes": {},
    "override_attributes": {},
    "run_list": [
        "recipe[env_test::web]"
    ]
}
```

#### Environment

- 設定ファイルは`<path_to_cookbook>/environments/<name_of_environment>.json`
- `rb`形式でも設定できるらしいが、試しにkitchenで動かしている際には確認できず。
- 以下のようにdefault attributeで設定した値をenvironmentで定義した値で上書きすることができる。
- [こちら][6]を参考に。

attributeのみ定義

```bash
$ cat recipes/default.rb
puts '##########'
puts "node['env_test']['foo'] is #{node['env_test']['foo']}."
puts '##########'

$ cat attributes/default.rb
default['env_test']['foo'] = "This is foo"

$ kitchen converge
...
##########
node['env_test']['foo'] is This is foo.
##########
...
```

environment定義後

```bash
$ cat environments/development.json
{
  "name": "development",
  "description":"variables depends on environment",
  "chef_type": "environment",
  "json_class": "Chef::Environment",
  "default_attributes": {
    "env_test": {
      "foo": "This is overwritten"
    }
  },
  "override_attributes": {}
}

$ kitchen converge
...
##########
node['env_test']['foo'] is This is overwritten.
##########
...
```

### 参考

- [Tutorials - Learn Chef][4]
- [An Overview of Chef - Chef Docs][2]

[1]: https://learn.chef.io/learn-the-basics/rhel/
[2]: https://docs.chef.io/chef_overview.html
[3]: https://supermarket.chef.io/
[4]: https://learn.chef.io/tutorials/
[5]: http://qiita.com/mather314/items/a10ac4f9eeb68023623d
[6]: http://d.hatena.ne.jp/naoya/20131222/1387700058
