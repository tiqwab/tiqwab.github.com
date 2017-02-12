---
layout: post
title: Jenkins入門
tags: "jenkins"
comments: true
---

Jenkinsを自分で用意して動かしてというのをやったことがなかったので、簡単なことならちゃちゃっと用意できるぐらいになっておきたいです。

1. [やることのイメージ](#anchor1)
2. [Jenkinsのインストール](#anchor2)
3. [SSHの設定](#anchor3)
4. [gitレポジトリの準備](#anchor4)
5. [PipelineによるJenkinsジョブの設定](#anchor5)
6. [Jenkinsジョブのリモート起動](#anchor6)

### 環境

Jenkinsの動作環境

- OS: CentOS Linux release 7.3.1611
- Java: openjdk version "1.8.0\_121"
- Jenkins: 2.32.2

---

<a id="anchor1"></a>

### 1. やることのイメージ

Jenkinsの利用形態でよく見るのはgithubのhookを利用してpushやpull requestに応答してテストやビルドを行うというものだと思います。
しかしここではよりJenkins自体の動作に注目するため (そして何よりJenkinsを動かすグローバルな環境が手元になかったので)、gitとJenkinsのみを使用したシンプルなローカル環境でJenkinsを動作させたいと思います。

- 目的: リモートgitリポジトリへのpushをトリガーにテストを実行する
  - host (VM含め) 3台
  - host1にgitのリモートリポジトリを用意
  - host2からhost1のリモートリポジトリにpushする
  - リモートリポジトリに設定したhookによりhost3上のJenkinsジョブを起動し、テストを実行する

```
+-----+                  +-----+ -(2. ジョブキック by hook)->   +-----+
|host2| -(1. git push)-> |host1|                              |host3|
+-----+                  +-----+  <-(3. checkout and test)-   +-----+
```

<a id="anchor2"></a>

### 2. Jenkinsのインストール

Jenkinsをインストールするhost3はVMで用意したのでその設定です。

- VagrantとVirtualBoxでJenkinsをインストールするVMの用意
  - box: [centos/7][3]
  - networkはport forwarding
    - Jenkinsはデフォルトではport: 8080を使用するので、それに合わせておく
- Jenkinsのインストール
  - まずJavaの実行環境が必要
  - その他必要になるパッケージのインストール (Jenkinsの動作に必須なものではない)
    - wget (下記でwgetコマンドを使用するため。デフォルトで存在するcurlでもよい)
    - git (ジョブの中でgitを使用したいため)
    - epel-release (python-pipのインストールのため)
    - python-pip (テストでPythonを使用するため)
  - Jenkinsリポジトリを新たにyumのリポジトリとして追加する
    - `sudo wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo`
    - `sudo rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key`
  - Jenkinsのインストール
    - `sudo yum install jenkins`

以上の内容をまとめたVagrantfileは以下のようになります。

```ruby
Vagrant.configure("2") do |config|
  # Specify box
  config.vm.box = "centos/7"
  # Specify network
  config.vm.network "forwarded_port", guest: 8080, host: 8080
  # Provisioning shell script
  config.vm.provision "shell", inline: <<-SHELL
    yum install -y java-1.8.0-openjdk
    yum install -y wget
    yum install -y git
    yum install -y epel-release
    yum install -y python-pip
    wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo
    rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
    yum install -y jenkins
    systemctl enable jenkins
    systemctl start jenkins
  SHELL
end
```

これを配置したディレクトリで`vagrant up`したあとに、`http://127.0.0.1:8080`にアクセスするとJenkinsが起動していることが確認できます。
このあとJenkinsのUnlockだったり色々ありますが、画面に表示される内容に素直に従う感じで進めればOKです。

[参考]

- [Installing jenkins][1]
- [JenkinsでCI環境構築チュートリアル][2]

<a id="anchor3"></a>

### 3. SSHの設定

1.で示した内容に従い、gitリモートリポジトリを管理するhost1にローカルリポジトリを持つhost2、Jenkinsをインストールしたhost3からそれぞれssh接続できるようにします。

SSHの設定なので特にJenkinsに限ったような話ではないのですが、一点気をつけることとしてはJenkinsのジョブはシステム上jenkinsユーザによって実行されるため、鍵生成もjenkinsユーザとして行うようにします ([参考: Jenkinsの設定を最小限でJenkinsfile(Pipeline)を使う][8])。

```bash
$ sudo su
$ su -s /bin/bash - jenkins
$ ssh-keygen -t rsa
...
```

またSSH先のport番号の設定等はssh config(`~/.ssh/config`)に記載しておけば、Jenkinsジョブで使用するときも問題なく機能します。

鍵生成やssh configといったあたりの内容はここでは省略しますが、手順に関しては下記のサイトがわかりやすく解説されていると思います。

- [Raspberry Piに公開鍵認証を使ってssh接続する][4]
- [複数の公開鍵を使い分ける][5]

<a id="anchor4"></a>

### 4. gitリポジトリの準備

3.でhost間での接続を確立したので、ローカル、リモートそれぞれにgitリポジトリを作成します。

host1上でgitリモートリポジトリを作成します。

```bash
$ mkdir jenkins-beginner.git && cd jenkins-beginner.git
$ git init --bare --share
```

host2上にgitリポジトリを作成し、remote先に上で作成したものを指定します。

```bash
$ mkdir jenkins-beginner && cd jenkins-beginner
$ git init
$ git remote add origin ssh://<host1>/<path_to_repo>
```

試しにgit pushしてみてうまく動いていることを確認します。

```bash
$ git push -u origin master
Counting objects: 3, done.
Writing objects: 100% (3/3), 233 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To ssh://pi@rasp1/home/pi/Workspace/jenkins-beginner.git
 * [new branch]      master -> master
Branch master set up to track remote branch master from origin.
```

<a id="anchor5"></a>

### 5. PipelineによるJenkinsジョブの作成

Jenkinsのジョブは以前はGUIで作成していたようですが、ver.2からは標準の機能としてPipelineというGroovyスクリプトでジョブを定義できるようになりました。
スクリプトでジョブを記述することで内容を確認しやすくなり、またスクリプト自体をバージョン管理することができるので扱いやすくなっています。

ここではそのスクリプトをJenkinsfileとして開発中のリソースと一緒にリポジトリに配置し、そのJenkinsfileを使用してジョブが実行されるようにしたいと思います。
主な手順は以下の3つです。

- Credentialの設定
- Jenkinsfileの作成
- Jenkinsジョブの作成

#### Credentialの設定

これはJenkinsfileによるジョブの実行に必須の内容ではないですが、SSHによりリモートのgitリポジトリからJenkinsfileを取得してくるために、秘密鍵の情報をJenkinsに教えてあげる必要があります。

Jenkinsではジョブによらず一元的に認証情報を管理する機能があるため、この機能を利用してジョブ中に使用するパスワードや秘密鍵といった情報をJenkinsに登録します。
登録の手順等は参考のページに書かれているために省略し、今回の登録内容のみ下記にまとめておきます。

```
- Scope: Global
- Username: jenkins (SSHで使用するユーザ名? ただそうだとしてもssh configで上書きされているようなので今回はあまり意味がない)
- Private Key: From the jenkins master ~/.ssh (つまり通常は/var/lib/jenkins/.sshのことになるはず)
- Passphrase: (上で鍵生成時に設定しなかったためなし)
- ID: jenkins-private-key (Jenkins内で使用されるID。Credential中で他と被るものはつけられない)
- Description: jenkins-private-key
```

#### Jenkinsfileの作成

ジョブで実行したい内容をリポジトリに配置したJenkinsfileに記述します。

Jenkinsfileの書き方は下記の参考や公式のリファレンスが参考になるかと思います。例えばgitやgithubを利用したいのであればテンプレート化されたコードが使用できますし、ssh認証を利用した処理を書きたいのであればSSH Agent Pluginが助けになります。また`sh`メソッドを使用すればシェルコマンドを実行できるので、比較的何でもできます。

ちなみにジョブを実行する際のカレントディレクトリは`/var/lib/jenkins/workspace/<job-name>`になり、またJenkinsfileを持ってくるときは`var/lib/jenkins/workspace/<job-name>@script`にcheckoutしたものが配置されるようです。

なのでテストを実行したいリポジトリにJenkinsfileを配置した場合、Jenkinsfile内で再度checkoutするか、script用のディレクトリをコピーして使用するかになるのかなと思います。
ここでは再度checkoutしましたが、これだとリポジトリの情報がジョブの定義時とで2回書くことになっておりイマイチかもしれません。

```groovy
node {
    stage('checkout') {
        if (!fileExists('jenkins-beginner')) {
            // clone repository if this is the first time to execute the job
            sh 'git clone <url_to_repo>'
        } else {
            // pull resources from the repository
            sh 'cd jenkins-beginner && git pull'
        }
        sh 'cd jenkins-beginner && ls -al'
    }
    stage('test') {
        sh 'cd jenkins-beginner && python setup.py test'
    }
}
```

#### Jenkinsジョブの作成

Jenkinsのジョブ定義から「Pipeline script from SCM」を指定して以下のように設定します。

- SCM: Git
- Repositories
  - Repository URL: `ssh://<user>@<host1>/<path_to_repo>`
  - Credentials: jenkins (上で設定したもの)
- Script Path: Jenkinsfile

これにより上で作成したJenkinsfileを含むリポジトリの内容がcheckoutされ、ジョブが実行されることが確認できます。

```bash
 > git rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url ssh://pi@rasp1/home/pi/Workspace/jenkins-beginner.git # timeout=10
Fetching upstream changes from ssh://pi@rasp1/home/pi/Workspace/jenkins-beginner.git
 > git --version # timeout=10
using GIT_SSH to set credentials jenkins private key
 > git fetch --tags --progress ssh://pi@rasp1/home/pi/Workspace/jenkins-beginner.git +refs/heads/*:refs/remotes/origin/*
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
Checking out Revision 31167790b68e05d7ebd05231a45f13ec63f8047e (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 31167790b68e05d7ebd05231a45f13ec63f8047e
 > git rev-list 211883ee28529581b7d74ce5638c534f31fd6021 # timeout=10
[Pipeline] node
Running on master in /var/lib/jenkins/workspace/jenkins-sample
[Pipeline] {
[Pipeline] stage
[Pipeline] { (checkout)
[Pipeline] fileExists
[Pipeline] sh
[jenkins-sample] Running shell script
+ git clone ssh://pi@rasp1/home/pi/Workspace/jenkins-beginner.git
Cloning into 'jenkins-beginner'...
[Pipeline] sh
[jenkins-sample] Running shell script
+ cd jenkins-beginner
+ ls -al
total 16
drwxr-xr-x. 5 jenkins jenkins  114 Feb 11 20:00 .
drwxr-xr-x. 3 jenkins jenkins   30 Feb 11 20:00 ..
drwxr-xr-x. 8 jenkins jenkins  163 Feb 11 20:00 .git
-rw-r--r--. 1 jenkins jenkins 1060 Feb 11 20:00 .gitignore
-rw-r--r--. 1 jenkins jenkins  496 Feb 11 20:00 Jenkinsfile
-rw-r--r--. 1 jenkins jenkins   74 Feb 11 20:00 README.md
drwxr-xr-x. 2 jenkins jenkins   40 Feb 11 20:00 sample
-rw-r--r--. 1 jenkins jenkins  168 Feb 11 20:00 setup.py
drwxr-xr-x. 2 jenkins jenkins   45 Feb 11 20:00 test
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (test)
[Pipeline] sh
[jenkins-sample] Running shell script
+ cd jenkins-beginner
+ python setup.py test
running test
running egg_info
creating sample.egg-info
writing sample.egg-info/PKG-INFO
writing top-level names to sample.egg-info/top_level.txt
writing dependency_links to sample.egg-info/dependency_links.txt
writing manifest file 'sample.egg-info/SOURCES.txt'
reading manifest file 'sample.egg-info/SOURCES.txt'
writing manifest file 'sample.egg-info/SOURCES.txt'
running build_ext
test_echo (test.echo_test.EchoTest) ... ok

----------------------------------------------------------------------
Ran 1 test in 0.000s

OK
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

[参考]

- [Jenkins 2の新機能「Pipeline」を使ってみよう][6]
- [Jenkinsを使った自動テスト環境を作る (前編)][7]

<a id="anchor6"></a>

### 6. Jenkinsジョブのリモート起動

リモートからジョブを起動するためにJenkins上で以下の作業を行います。

- ジョブ定義から「Build Triggers -> Trigger builds remotely」をチェック
  - Authentication Tokenに適当なキーワードを入力
- Jenkinsホームから「People -> ジョブを定義しているユーザ -> Configure -> API Token」でUser IDとAPI Tokenを確認
- Jenkinsホームから「Manage Jenkins -> Configure Global Security -> Prevent Cross Site Request Forgery exploits」のチェックを外す
  - ローカルなのでまあ問題は無いけれど、そうでない場合どうするべき?

gitリモートリポジトリに行き、`<repository>/hooks/post-update`に以下のようにジョブをキックするためのコマンドを追加します。
ここでUser ID, API Token, job-tokenはそれぞれ上記で定義したものです。

```
#!/usr/bin/env bash
url=http://<host2>/job/<job-name>/build
credential=<User ID>:<API Token>
token=<job-token>
curl -u $credential $url -X POST -d token=$token
```

これでリモートリポジトリにpushを行った際に、ジョブが実行されるようになります。

### Summary

- JenkinsはけっこうGUIでごちゃごちゃやらないといけないイメージがあったけどPipelineを使えばそんなことはなかった
- 簡単なテストやビルドジョブであれば簡単にかつ直感的に書くことができる
  - 個人的にはJavaのbuildツールのGradleと似て、スクリプトで書けるのがわかりやすくて好き

次に考えたいのは以下の2点です。

- いまだとテストを行う際に、Jenkinsを動かしているサーバ上に色々と用意しないといけなくなっているのでどうにかしたい
  - Dockerの使用?
- GitHub上で連携させたい
  - ただこれだとクラウドツールで色々ありそうなのでそちらを使うのが一般的?

### 参考

- [Installing jenkins][1]
- [JenkinsでCI環境構築チュートリアル][2]
- [Jenkins 2の新機能「Pipeline」を使ってみよう][6]
- [Jenkinsを使った自動テスト環境を作る (前編)][7]

[1]: https://jenkins.io/doc/book/getting-started/installing
[2]: https://ics.media/entry/14273
[3]: https://atlas.hashicorp.com/centos/boxes/7
[4]: https://tool-lab.com/2013/11/raspi-key-authentication-over-ssh/
[5]: http://d.hatena.ne.jp/MIZUNO/20080705/1215238138
[6]: http://www.buildinsider.net/enterprise/jenkins/005
[7]: http://knowledge.sakura.ad.jp/knowledge/5293/
[8]: http://qiita.com/comefigo/items/fc7aad310235caf89fcd
