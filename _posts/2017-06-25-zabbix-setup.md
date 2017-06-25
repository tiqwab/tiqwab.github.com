---
layout: post
title: Zabbix のセットアップ
tags: "zabbix"
comments: true
---

現在仕事では一部のシステムを [Zabbix][5] によって監視しているのですが、自身で Zabbix のインストールを行ったことがなかったので一度試してみたいと思いました。

最終的に Zabbix Server, Agent をセットアップする Chef cookbook を作成できればという感じですが、そのためには一度手順を確認しておかないと厳しそうなのでまずはマニュアルでインストールを行ってみます。

環境

AWS 上に 以下の 2 台の EC2 インスタンスをたてています。

- Zabbix Server
  - version: 3.2.6
  - インストール先 OS: CentOS 7.3
  - 開放 port: 22 (SSH), 80 (HTTP), 10051 (zabbix)
- Zabbix Agent
  - version: 3.2.6
  - インストール先 OS: Amazon Linux AMI release 2017.03
  - 開放 port: 22 (SSH), 10050 (zabbix)

---

### Zabbix Server の用意

CentOS 7 に Zabbix Server をインストールします。

```
# Check distribution and version 
$ cat /etc/centos-release
CentOS Linux release 7.3.1611 (Core)

# Set timezone
$ timedatectl set-timezone Asia/Tokyo

# Disable SELinux
# ...
# SELINUX=disabled
# ...
$ vi /etc/selinux/config

# Register a repository of zabbix 3.2
$ rpm -ivh http://repo.zabbix.com/zabbix/3.2/rhel/7/x86_64/zabbix-release-3.2-1.el7.noarch.rpm

# Install zabbix server (zabbix-get is optional)
# Assume that zabbix uses mysql (or mariadb) as database.
$ yum -y install zabbix-server-mysql zabbix-web-mysql zabbix-get

# Edit timezone of zabbix web interface
# ...
# php_value date.timezone Asia/Tokyo
# ...
$ vi /etc/httpd/conf.d/zabbix.conf
```

次に Zabbix が使用する DB を用意します。
ここでは Zabbix server と MariaDB を同一のホスト上に配置させています。

```
# Install mariadb
$ yum -y install mariadb-server 

# Settings of mariadb
# ...
# [mysqld]
# character-set-server = utf8
# collation-server = utf8_bin
# skip-character-set-client-handshake
# innodb_file_per_table
# ...
$ vi /etc/my.cnf.d/server.cnf

# Start and enable mariadb
$ systemctl start mariadb
$ systemctl enable mariadb

# 'root' should have the password, but ignore here
$ mysql -uroot
> create database zabbix character set utf8 collate utf8_bin;
> grant all privileges on zabbix.* to zabbix@`%` identified by 'password';
> grant all privileges on zabbix.* to zabbix@`localhost` identified by 'password';
> exit

# Set up tables for zabbix
$ zcat /usr/share/doc/zabbix-server-mysql-3.2.6/create.sql.gz | mysql -uroot -Dzabbix
```

最後に Zabbix server の設定ファイルに DB への接続設定を追記し、Zabbix を起動します。

```
# Edit database configuration of zabbix server
# ...
# DBHost=localhost
# ...
# DBPassword=password
# ...
$ vi /etc/zabbix/zabbix_server.conf

# Start and enable zabbix-server and httpd
$ systemctl start zabbix-server
$ systemctl enable zabbix-server
$ systemctl start httpd
$ systemctl enable httpd
```

これで `http://<zabbix-server>/zabbix` にアクセスすると setup の画面が表示されます。
Setup を指示に従って進めていくとログイン画面が表示されるのでアカウント 'Admin' パスワード 'zabbix' でログインできます。

### Zabbix Agent の用意

Zabbix で監視対象となるサーバを用意します。
Zabbix Server と違い、こちらは Amazon Linux を使用することにします。

```
# Check distribution and version 
$ cat /etc/system-release
Amazon Linux AMI release 2017.03

# Register a repository of zabbix 3.2
# Use 'el6' not 'el7'
$ rpm -ivh http://repo.zabbix.com/zabbix/3.2/rhel/6/x86_64/zabbix-release-3.2-1.el6.noarch.rpm

# Install zabix agent
$ yum -y install zabbix-agent

# Add settings of zabbix server ip
# ...
# Server=<ip of zabbix server>
# ...
# ServerActive=<ip of zabbix server>
# ...
$ vi /etc/zabbix/zabbix_agentd.conf

# Start and enable zabbix agent
$ service zabbix-agent start
$ chkconfig zabbix-agent on
```

これで Zabbix Agent のセットアップができたので Server との疎通を確認します。

```
# Execute on zabbix server
$ zabbix_get -s <zabbix agent ip> -k agent.version
3.2.6
```

確認できればあとは Zabbix の Web インタフェースからホストを登録して監視ができます。

参考:

- [Zabbix ダウンロード][4]
- [AWS 上で CentOS 7 に Zabbix 3.2 を構築してみた][2]
- [Zabbix 3.0 を CentOS 7 にインストール][3]


[1]: https://docs.aws.amazon.com/ja_jp/opsworks/latest/userguide/best-practices-packaging-cookbooks-locally.html
[2]: http://dev.classmethod.jp/cloud/aws/aws-centos7-zabbix3_2-setup/
[3]: http://qiita.com/atanaka7/items/294a639effdb804cfdaa
[4]: http://www.zabbix.com/jp/download
[5]: http://www.zabbix.com/jp/
