---
layout: post
title: Growi を構築する Ansible Playbook 作成
tags: "ansible"
comments: true
---

これまで雰囲気で使用していた Ansible についてもう少し整理された理解をしたかったので、[単一ホストに Growi を構築する Playbook][12] を作成してみました。以下はそこで得た理解をまとめたものです。

Growi インストールの手順は [Growi のインストール on CentOS][2] を参考にしました。

### プロジェクトのディレクトリ構成

今回のプロジェクトは以下のようなディレクトリ構成にしました。これは [Sample Ansible setup][1] を元に必要ないものを省いた形になっています。

```
.
├── group_vars  # Group 変数置場
│   ├── all.yaml
│   └── sample.yaml
│
├── hosts  # Inventory ファイル
│
├── host_vars  # Host 変数置場
│   └── growi.example.com.yaml
│
├── roles  # Role 置場
│   ├── common  # 共通 Role, 内部省略
│   │
│   ├── elasticsearch  # Elasticsearch 用 Role, 内部省略
│   │
│   ├── growi  # Growi 用 Role, 内部省略
│   │
│   ├── mongodb  # MongoDB 用 Role, 内部省略
│   │
│   └── proxy  # Reverse Proxy 用 Role
│       ├── defaults
│       │   └── main.yaml
│       ├── handlers
│       │   └── main.yaml
│       ├── tasks
│       │   └── main.yaml
│       └── templates
│           └── etc
│               ├── nginx
│               │   └── conf.d
│               │       └── growi.conf
│               └── yum.repos.d
│                   └── nginx.repo
│
├── site.yaml  # Playbook
```

プロジェクトの一番上の階層に注目すると以下の 4 種の Ansible リソースが含まれています。

- Inventory (`hosts` ファイル)
- Role (`roles` ディレクトリ)
- Playbook (`site.yaml` ファイル)
- Variable (`group_vars`, `host_vars` ディレクトリ)

#### Inventory

- Ansible でプロビジョニングしたい Host の一覧
- 各 Host は 1 以上の Group に所属される
  - Host は必ず `all` という Group に含まれる
  - ユーザ独自に作成した Group にも所属できる
- 静的に定義する場合は環境ごとに作成されることが多そう (e.g. staging, production)
- プラグインを使用すれば動的に定義することもできる (ref. [Working with dynamic inventory][10])

今回作成したプロジェクトでは `hosts` ファイルで Inventory を定義しました。

```
[sample]
growi.example.com ansible_host=192.168.33.10
```

ここでは `growi.example.com` という Host を `all` と `sample` Group に所属させています。

Host 名として直接 IP を記述することもできますが、それよりもわかりやすい名前をつけて `ansible_host` で IP を指定する方が管理しやすいかと思います。

#### Role

- プロビジョニングしたい内容を記述する
- 例えば以下の要素で構成される
  - Task
    - 一連のプロビジョニング処理
    - やりたい処理は Module とそれに渡す引数という形で記述する
  - Handler
    - プロビジョニング中に (別 Task から) notify された際に実行する Task を記述する
    - デフォルトでは Handler は他の Task が終了したあとに実行される
  - Template
    - プロビジョニング中に使用する設定ファイル等のための Jinja2 テンプレート
  - defaults
    - Role のデフォルト変数
    - Role で共通に使用する値や外から変更したい値を設定する

以下に今回 proxy Role で使用した Task を記載しました。

```yaml
# roles/proxy/tasks/main.yaml
---

- name: Setup Nginx repository
  ansible.builtin.template:
    src: etc/yum.repos.d/nginx.repo
    dest: /etc/yum.repos.d/nginx.repo
    owner: root
    group: root
    mode: '0644'

- name: Install Nginx
  ansible.builtin.yum:
    name: nginx
    enablerepo: nginx
    state: present

# ... (省略)

- name: Add Nginx conf
  ansible.builtin.template:
    src: etc/nginx/conf.d/growi.conf
    dest: /etc/nginx/conf.d/growi.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart Nginx

- name: Start Nginx
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: yes
```

Module というのはここでいう `ansible.builtin.template` や `ansible.builtin.yum` のことで、このように Ansible では Module を選択することでやりたい処理を記述します。

Template は `ansible.builtin.template` Module を使用して、予め用意した Jinja2 テンプレートファイルを元にプロビジョニング先のホストにファイルを配置する仕組みです。ここでは Yum のリポジトリファイルや Nginx の設定ファイルを用意するために使用しています。

Handler はある Task を実行して何らかの変更が加えられた (changed が返された) ときにだけ処理を実行するための仕組みです。ここでは `Add Nginx conf` Task で `Restart Nginx` Task に notify することで、Nginx の設定ファイルを書き換えたときに Nginx をリスタートするようにしています。

```
# roles/proxy/handlers/main.yaml 
---

- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```

Role をどういった単位でまとめるべきか自分の中で整理できていないのですが、最低限意味のある単位で細かく分けるのがいいのかなと考えています。今回は Elasticsearch, MongoDB, proxy (Nginx), Growi のようにミドルウェア毎に Role を分けてみました。各サービスのホスト、ポートを変数として共有できれば全ミドルウェアを単一ホストにインストールすることも、DB だけ別のホストにインストールする、といったこともできそうなので程良い粒度ではないかと思っています。

#### Playbook

Ansible における Playbook というのはけっこう意味が広いというか「Playbook を適用する」みたいなことはよく言うけど具体的に Playbook で何を指しているのかわかりづらいという印象でした。ただ [Intro to playbooks][11] をちゃんと読んで自分がわかりにくく感じているのは Playbook として何か画一的なものをイメージしているからであって、実際は「Group と (Role を介した間接的な) Task の結びつけ」を Playbook と呼べばいいのかなと思いました。

なので以下のような Group と Task をひとまとめにした形も、

```yaml
---

- name: update web servers
  hosts: webservers
  remote_user: root

  tasks:
  - name: ensure apache is at the latest version
    yum:
      name: httpd
      state: latest
  - name: write the apache config file
    template:
      src: /srv/httpd.j2
      dest: /etc/httpd.conf

- name: update db servers
  hosts: databases
  remote_user: root

  tasks:
  - name: ensure postgresql is at the latest version
    yum:
      name: postgresql
      state: latest
  - name: ensure that postgresql is started
    service:
      name: postgresql
      state: started
```

あるいは今回のプロジェクトで採用したように Group と Role を結びつけたような形も、

```yaml
# site.yaml
---

- hosts: sample
  roles:
    - common
    - mongodb
    - elasticsearch
    - growi
    - proxy
  become: True
```

一見別のフォーマットに見えますがどちらもこれを使用して `ansible-playbook` コマンドで実行できる Playbook ということなんだと思います。というか試してみたら後者のファイルでも `tasks` を生やして Task を記述できたので結局フォーマットも同じですね。ちょっとしたプロビジョニングなら前者のように直接 Task を書けばいいし、複雑なものになってくると Role を使用した後者のような形で書きたくなるというだけの話っぽいです。

#### Variable

[Using Variables][9] に Playbook で設定できる箇所や優先度についてまとめられています。
今回のプロジェクトでは以下のような箇所では変数を管理しました。

- Role のデフォルト値として設定する (e.g. `roles/proxy/defaults/main.yaml` ファイル)
- Group に設定する (`group_vars` ディレクトリ)
- Host に設定する (`host_vars` ディレクトリ)
- Runtime 時に (CLI の `--extra-vars` オプションとして) 設定する

{% raw %}
また Runtime 時の設定でいうと環境変数を `{{ lookup('env', 'PWD') }}` のように参照することもできます。
{% endraw %}

[1]: https://docs.ansible.com/ansible/latest/user_guide/sample_setup.html
[2]: https://docs.growi.org/en/admin-guide/getting-started/centos.html
[3]: https://note.com/shift_tech/n/ne0e1ccec6fbd
[4]: https://archlinux.org/packages/community/any/ansible/
[5]: https://www.archlinux.jp/packages/community/any/ansible-base/
[6]: https://www.redhat.com/ja/explore/ansible/getting-started/with-ansible-content-collections
[7]: https://docs.ansible.com/ansible/latest/user_guide/collections_using.html
[8]: http://expressjs.com/en/guide/behind-proxies.html
[9]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html
[10]: https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html
[11]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html
[12]: https://github.com/tiqwab/example/tree/master/ansible-growi
