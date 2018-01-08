---
layout: post
title: Kubernetes ことはじめ
tags: "elasticsearch, kubernetes"
comments: true
---

去年は [Docker が Kubernetes のネイティブサポート][21] を発表したり AWS にも Kubernetes をサポートする [EKS][22] がプレビュー入りしたりと Kubernetes が利用しやすくなる環境がどんどん整いつつあります。

ただ自身はこの流れに遅れて Kubernetes を触ってみたことが無かったので、ここで簡単に内容を整理してみることにしました。Kubernetes の基本的な要素である Pod, Deployment, Service を中心に見ていき、最後に既存の定義ファイルをもとに Elasticsearch クラスタを作成する、ということをやっていきます。

1. [Kubernetes とは](#what-is-kubernetes)
2. [Kubernetes を構成する主要な要素](#concepts)
3. [定義ファイルを使用したオブジェクトの作成](#definition-file)
4. [Elasticsearch クラスタのセットアップ](#es-setup)

環境:

使用している OS が ArchLinux なので kubectl と minikube はそれぞれ AUR からインストールしました。
kubctl で表示される client と server のバージョンが違うのは普通なんですかね...?

```bash
# Version of kubectl and cluster
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T21:07:38Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"8", GitVersion:"v1.8.0", GitCommit:"0b9efaeb34a2fc51ff8e4d34ad9bc6375459c4a4", GitTreeState:"clean", BuildDate:"2017-11-29T22:43:34Z", GoVersion:"go1.9.1", Compiler:"gc", Platform:"linux/amd64"}

# Version of minikube
$ minikube version
minikube version: v0.24.1
```

---

<div id="what-is-kubernetes" />

### 1. Kubernetes とは

- コンテナオーケストレーションツール
  - コンテナ化されたアプリケーションのデプロイやスケーリングといった Linux コンテナを本番運用する上で必要となる機能を提供するもの
- Kubernetes の操作には [kubectl][23] というコマンドラインインタフェースを使用することが多い
- Kubernetes の環境を一から自分で用意するのはけっこうタフ
  - クラウドベンダが (少なくとも一部分を) 管理した環境を利用するのが楽
    - Goole Kubernetes Engine
    - Azure Container Service
  - [Minikube][17] というローカルで簡易的な環境を作れるツールもある

Kubernetes をざっと理解するためには以下の内容を参考にしました。

- [Kubernetes とは][18]
- [GKE & Kubernetes のアーキテクチャ解説][24]
- [Picking the Right Solution][19]
- [Kubernetes API Overview][20]

以下では kubectl と Minikube はインストール済として進めていきます。

<div id="concepts" />

### 2. Kubernetes を構成する主要な要素

Kubernetes はコンテナオーケストレーションツールとして様々な概念を含んでいるのですが、その中でもまず理解しておきたい要素をいくつか取り上げて整理してみます。

#### Cluster

- 1 台以上の (物理的 or 仮想) マシンを束ねたもの
- cluster には master と呼ばれるノードとそれ以外のノードが存在する
  - 前者を master, 後者を単に node と呼ぶことが多いようなので、以後そう記述する
- master は cluster の管理を行う
- node にはアプリケーションコンテナをデプロイする

もう少し詳しく見ていくと master というのは以下の 3 つのプロセスを走らせているノードであると定義されているようです ([参考][1])。

- kube-apiserver (Kubernetes のオブジェクトを管理するための REST API を提供する)
- kube-controller-manager (cluster の状態を定期的にチェックし定義されたものに持っていく)
- kube-scheduler (どの node でコンテナを実行する等の決定を行う?)

一方 node では以下の 2 つのプロセスが実行されています。

- kubelet (master とのやり取りを行う)
- kube-proxy (ネットワーク設定のためのもの?)

(少なくとも本番環境では) master と node は別の (物理的 or 仮想) マシンに乗せるのが一般的みたいです。GKE (Google Kubernetes Engine) の最小構成も master 1, node 1 の 2 台構成になっています。

#### Pod

- Kubernetes ではコンテナを抽象化して Pod として扱う
- Pod には例えば以下のような情報が含まれる
  - 基にした image 名
  - マウントされた volume
  - 環境変数
  - 割り振られた private IP
- Pod とコンテナは 1 対 1 ではなく 1 対多の関係になる
  - 同一 Pod に含まれるコンテナは IP, ホスト名等のリソースを共有する

上では「node でコンテナを実行する」と言ったのですが、恐らく「node で Pod を走らせる」等言った方が正確ですね。

kubectl を使用して hello world を出力する簡単な Pod を実行してみます。

```bash
# Create Pod
$ kubectl run hello-world --image busybox --restart=Never -- echo hello world
pod "hello-world" created

# Get information of pods
$ kubectl get pods -a
NAME          READY     STATUS      RESTARTS   AGE
hello-world   0/1       Completed   0          1m

# Check output of Pod
$ kubectl logs hello-world
hello world

# Delete Pod
$ kubectl delete pod hello-world
pod "hello-world" deleted
```

#### Deployment

- クラスタ内である Pod をどのように実行するか、更新するかといったことを管理する
- Deployment には以下のような情報が含まれる
  - 実行したい Pod の情報
  - Pod のレプリカ数
  - Rollout 時の挙動 (rollout: 更新した Pod 定義の適用)

資料を見ている感じ、Pod を直接実行するより deployment を介して pod を実行するという方が多いように思います。

kubectl から [kubernetes-bootcamp][15] イメージを実行する簡単な Deployment を作成してみます。

```bash
# Create Deployment
$ kubectl run kubernetes-bootcamp --image docker.io/jocatalin/kubernetes-bootcamp:v1 --port=8080
deployment "kubernetes-bootcamp" created

# Get information of deployment
$ kubectl get deployment/kubernetes-bootcamp -o wide
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS            IMAGES                                       SELECTOR
kubernetes-bootcamp   1         1         1            1           17s       kubernetes-bootcamp   docker.io/jocatalin/kubernetes-bootcamp:v1   run=kubernetes-bootcamp

# Get information of pods
$ kubectl get pods -o wide
NAME                                   READY     STATUS    RESTARTS   AGE       IP           NODE
kubernetes-bootcamp-6db74b9f76-2f4xz   1/1       Running   0          2m        172.17.0.4   minikube

# Check output of Pod
$ kubectl logs $(kubectl get pods -o json | jq -r .items[0].metadata.name)
Kubernetes Bootcamp App Started At: 2018-01-01T11:27:29.059Z | Running On:  kubernetes-bootcamp-6db74b9f76-4d4q
```

上で Pod を実行するときにも同じ `kubectl run` を使用したのですが、こちらでは `--restart` オプションを指定しませんでした。`--restart` オプションは Pod が (正常、異常) 終了した際リスタートを行うかどうかを指定するためのもので、デフォルトは Always (常にリスタート) となります。バッチ的な処理を行うときには `--restart=Never` を指定することで (Deployment を作成せず) Pod のみを作成し、一度だけコンテナの処理を行うことができます。

```bash
$ kubectl run -h
...
      --restart='Always': The restart policy for this Pod.  Legal values [Always, OnFailure, Never].
If set to 'Always' a deployment is created, if set to 'OnFailure' a job is created, if set to
'Never', a regular pod is created. For the latter two --replicas must be 1.  Default 'Always', for
CronJobs `Never`.
...
```

これでアプリケーションは起動できているのですが、実はこのままだと cluster 外部からはアプリケーションにアクセスすることができません。外部からさくっとアクセス可能にするには `kubectl proxy` が使えます。

```bash
$ kubectl proxy -h
Creates a proxy server or application-level gateway between localhost and the Kubernetes API Server

$ kubectl proxy

(from another terminal)
$ curl http://localhost:8001/api/v1/proxy/namespaces/default/pods/$POD_NAME/
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-6db74b9f76-4d4qx | v=1
```

ただこれだと直接 Pod を指定しているので、複数台 (replicas >= 2) 走らせていてもアクセスを分散させることができません。この点を解消するためには Service を使用します。

#### Service

- Pod にアクセスするためのエンドポイントを定義する
- Service にはいくつかタイプが存在する
  - ClusterIP (cluster 内にサービスを公開する)
  - NodePort (cluster 外部からアクセスできるよう各 node の特定のポート上にサービスを公開する)
  - LoadBalancer (cluster 外部からアクセスできるようクラウドベンダの提供するロードバランサを使用してサービスを公開する)
  - ExternalName (指定した値を kube-dns の CNAME に登録する。ちょっと詳細と使いどころが?)

Deployment で定義したように同一アプリケーションが複数 Pod 走っていることがあるので、そこを意識せずにアクセスするためのオブジェクトという感じです。

他コンテナからアクセスできればいいという場合にはデフォルトの ClusterIP タイプで問題ありません。LoadBalancer は cluster 外部からアクセスしたい、かつ本番運用時にまず候補に上がるタイプだと思います。開発時やちょっと試したいというときには NodePort が使用できます。

ちなみにこれらのタイプはそれぞれ全く独立したものというわけではなく、ドキュメントによると NodePort を選択した際には自動的に ClusterIP も作成され、LoadBalancer を選択したときには NodePort, ClusterIP が自動的に作成されるという挙動になっているようです ([参考][2])。

kubectl からは `kubectl expose` で Service を作成することができます。

```bash
$ kubectl expose deployment kubernetes-bootcamp --port=80 --target-port=8080 --type=NodePort
service "kubernetes-bootcamp" exposed

$ kubectl get services -o wide
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE       SELECTOR
kubernetes            ClusterIP   10.96.0.1       <none>        443/TCP          13d       <none>
kubernetes-bootcamp   NodePort    10.105.32.106   <none>        80:31684/TCP     7m        run=kubernetes-bootcamp

$ kubectl describe service/kubernetes-bootcamp
Name:                     kubernetes-bootcamp
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=kubernetes-bootcamp
Type:                     NodePort
IP:                       10.110.121.164
Port:                     http  80/TCP
TargetPort:               8080/TCP
NodePort:                 http  32714/TCP
Endpoints:                172.17.0.4:8080,172.17.0.5:8080,172.17.0.6:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

`kubectl describe` の出力を見ると Port, TargetPort, NodePort とポート関連の設定がパット見掴みづらいのですが、それぞれ「Service が受け付けるポート」、「受け付けたリクエストをコンテナに流すときに使用するポート」、「Service(type=NodePort) を作成した場合に、node 上で外部からリクエストを受け付けるために公開したポート」ということになります。

これで cluster 外部から node の ポート 31684 を介してアクセスできるようになりました。Minikube を使用しているのであれば node の IP やサービスのエンドポイントは以下のようにして確認できます ([参考][16])。

```bash
# kubernetes-bootcamp is exposed as NodePort, so it shows the IP of the node is 192.168.99.100.
$ minikube service list
|-------------|----------------------|-----------------------------|
|  NAMESPACE  |         NAME         |             URL             |
|-------------|----------------------|-----------------------------|
| default     | kubernetes           | No node port                |
| default     | kubernetes-bootcamp  | http://192.168.99.100:31684 |
| kube-system | kube-dns             | No node port                |
| kube-system | kubernetes-dashboard | http://192.168.99.100:30000 |
|-------------|----------------------|-----------------------------|

$ minikube service kubernetes-bootcamp --url
http://192.168.99.100:31684

# Access through Service
$ curl http://192.168.99.100:31684/
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-6db74b9f76-4d4qx | v=1
```

ここまで見た Pod, Deployment, Service の関係を図にまとめると以下のようになります。図では master 1 台、node 2 台の cluster 内に 2 replicas Pod をデプロイする構成を模しています。

<img
  src="/images/get-started-kubernetes/logical-structure-inside-cluster.png"
  title="logical-structure-inside-cluster"
  alt="logical-structure-inside-cluster"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

<div id="definition-file" />

### 3. 定義ファイルを使用したオブジェクトの作成

kubectl を使用したオブジェクトの管理方法には [Kubernetes Object Management][3] で紹介されているようにいくつか種類があります。

今までは kubectl に適当なコマンド、オプションを渡してオブジェクトを作成していました (imperative command)。これは覚えやすくちょっと試したりする分にはいいのですが、コマンドの履歴が残らないので現在のオブジェクトの状態が把握しづらく実運用でメインに使用するには少し心もとない感じでした。

ここでは imperative object configuration と呼ばれる定義ファイルを使用したオブジェクトの管理を試してみます。こちらは imperative command と違い、必要なオブジェクトの仕様を Kubernetes に伝えるという形になり、オブジェクトの管理をより厳密に行えます。

オブジェクトの API 定義は Kubernetes の公式リファレンス [Reference Documentation][4] 内で提供されており、これを見ると定義ファイルが主に以下の要素で構成されることがわかります。

- `apiVersion`: 対象 REST API バージョン
- `kind`: オブジェクトの種類 (e.g. Pod, Deployment)
- `metadata`: オブジェクトのメタ情報 (e.g. name, namespace)
- `spec`: どういったオブジェクトが必要なのかを宣言的に指定

以下では Pod, Deployment, Service の定義ファイルを順番に見ていきます。

#### Pod

上で使用した hello wold をエコーする Pod を定義ファイルにしてみます。

```bash
$ cat pod-hello-world.yml 
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
spec:
  containers:
  - args:
    - echo
    - hello world
    image: busybox
    name: hello-world
  restartPolicy: Never
```

定義ファイルからは `kubectl create` でオブジェクトを作成できます。

```bash
$ kubectl create -f pod-hello-world.yml
pod "hello-world" created
$ kubectl logs hello-world
hello world
```

オブジェクトの削除も定義ファイルを指定すれば OK です。

```bash
$ kubectl delete -f pod-hello-world.yml 
pod "hello-world" deleted
```

#### Deployment

今度は kubernetes-bootcamp を 3 pods 動かす Deployment を定義してみます。

```bash
$ cat deployment-kubernetes-bootcamp.yml 
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  # Unique key of the Deployment instance
  name: kubernetes-bootcamp
spec:
  # 3 Pods should exist at all times.
  replicas: 3
  template:
    metadata:
      labels:
        app: kubernetes-bootcamp
    spec:
      containers:
      - name: kubernetes-bootcamp
        # Run this image
        image: docker.io/jocatalin/kubernetes-bootcamp:v1
        ports:
          - containerPort: 8080
            protocol: TCP

# Create Deployment
$ kubectl create -f deployment-kubernetes-bootcamp.yml
deployment "kubernetes-bootcamp" created

# Check created Deployment
$ kubectl get deployment/kubernetes-bootcamp
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   3         3         3            3           18s

# Three kubernetes-bootcamp pods are running
$ kubectl get pods -l "app=kubernetes-bootcamp"
NAME                                   READY     STATUS    RESTARTS   AGE
kubernetes-bootcamp-589df98666-bbbgd   1/1       Running   0          30s
kubernetes-bootcamp-589df98666-dctdh   1/1       Running   0          30s
kubernetes-bootcamp-589df98666-qr2p9   1/1       Running   0          30s
```

#### Service

上で作成した pods に対して Service を定義してみます。

```bash
$ cat service-kubernetes-bootcamp.yml
apiVersion: v1
kind: Service
metadata:
  # Unique key of the Service instance
  name: kubernetes-bootcamp
spec:
  ports:
    # Accept traffic sent to port 80
    - name: http
       port: 80
       targetPort: 8080
       protocol: TCP
  selector:
    app: kubernetes-bootcamp
  type: NodePort

# Create Service
$ kubectl create -f service-kubernetes-bootcamp.yml 
service "kubernetes-bootcamp" created

# Check created Service
$ kubectl get service/kubernetes-bootcamp
NAME                  TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes-bootcamp   NodePort   10.99.5.132   <none>        80:32057/TCP   13s

# Access through Service
$ curl http://192.168.99.100:32057
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-589df98666-c675q | v=1
```

このように作成したいオブジェクトの仕様がわかっていれば、あとは API リファレンスとにらめっこして定義ファイルを作成することができます。

もし既存のオブジェクトを参考にして定義ファイルを作成したい場合、 `kubectl describe -o yaml` が参考になるかもしれません。

```bash
$ kubectl get -o yaml deployment/kubernetes-bootcamp
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: 2018-01-06T02:03:12Z
  generation: 1
  labels:
    app: kubernetes-bootcamp
  name: kubernetes-bootcamp
  namespace: default
  resourceVersion: "136990"
  selfLink: /apis/extensions/v1beta1/namespaces/default/deployments/kubernetes-bootcamp
  uid: bffd72b7-f285-11e7-9eee-080027cdab72
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: kubernetes-bootcamp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: kubernetes-bootcamp
    spec:
      containers:
      - image: docker.io/jocatalin/kubernetes-bootcamp:v1
        imagePullPolicy: IfNotPresent
        name: kubernetes-bootcamp
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: 2018-01-06T02:03:14Z
    lastUpdateTime: 2018-01-06T02:03:14Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2018-01-06T02:03:12Z
    lastUpdateTime: 2018-01-06T02:03:14Z
    message: ReplicaSet "kubernetes-bootcamp-589df98666" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

<div id="es-setup" />

### 4. Elasticsearch クラスタのセットアップ

ここまでで一通り Kubernetes を使い始める上で大事な要素はおさえられたと思うので、最後に総まとめとして Kubernetes で Elasticsearch クラスタのセットアップを行ってみたいと思います。

とはいってもいきなり一から自分で必要な定義ファイルを作成するのはタフだったので、大部分は既存のものを参考にし、わからないところを随時確認するという形にしました。参考にしたリポジトリは [kubernetes/examples][5] と [pires/kubernetes-elasticsearch-cluster][6] です。このどちらも使用している Docker イメージは [pires/docker-elasticsearch-kubernetes][7] になります。

以下では 下図のように Minikube で作成した 1 node の Kubernetes cluster に 3 node (master, data 兼用) 構成の Elasticsearch cluster を作成していきます (Kubernetes と Elasticsearch で用語が似通っているのでちょっとややこしいですね...)。

<img
  src="/images/get-started-kubernetes/sample-es-cluster.png"
  title="create-elasticsearch-cluster-with-kubernetes"
  alt="create-elasticsearch-cluster-with-kubernetes"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

#### ServiceAccount の作成

まずは ServiceAccount オブジェクトを作成します。ServiceAccount は Kubernetes 内で Pod の認証認可のために使用されるものです。Pod 作成時に明示的に定義しない場合、デフォルトのもの (default) が割り当てられるのですが、ここでは参考元にならって新規 ServiceAccount を作成しそれを使用することにします。

```bash
$ cat service-account.yml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: elasticsearch

$ kubectl create -f ./service-account.yml 
serviceaccount "elasticsearch" created
```

なお ServiceAccount とはなんぞや、については以下を参考にしました。

- [Configure Service Accounts for Pods][10]
- [kubernetes の認証とアクセス制御を動かしてみる][11]
- [Kubernetes 1.8 のアクセス制御について。あと Dashboard][12]

#### Service の作成

次に Service を定義します。この Service では 9200, 9300 の 2 つを対象ポートにしています。これらはそれぞれ Elasticsearch の提供する外部 API にアクセスするためのもの、Elasticsearch の各ノードが相互にコミュニケーションをとるためのもの、となります。

参考元では `type=LoadBalancer` でしたが、Minikube 上では利用できないのでデフォルトの `ClusterIP` に変更しています。

```bash
$ cat ./es-svc.yml 
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  labels:
    component: elasticsearch
spec:
  # type: LoadBalancer
  selector:
    component: elasticsearch
  ports:
    - name: http
      port: 9200
      protocol: TCP
    - name: transport
      port: 9300
      protocol: TCP

$ kubectl create -f ./es-svc.yml
service "elasticsearch" created

$ kubectl get services -l "component=elasticsearch" -o wide
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE       SELECTOR
elasticsearch   ClusterIP   10.104.228.184   <none>        9200/TCP,9300/TCP   2m        component=elasticsearch
```

#### Deployment の作成

最後に少し長いですが Deployment の定義です。

```bash
$ cat es-deploy.yml 
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: es
  labels:
    component: elasticsearch
spec:
  replicas: 3
  template:
    metadata:
      labels:
        component: elasticsearch
    spec:
      serviceAccount: elasticsearch
      initContainers:
        - name: init-sysctl
          image: busybox
          imagePullPolicy: IfNotPresent
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
      containers:
        - name: es
          securityContext:
            capabilities:
              add:
                - IPC_LOCK
          image: quay.io/pires/docker-elasticsearch-kubernetes:5.6.2
          env:
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: "CLUSTER_NAME"
              value: "myesdb"
            - name: "DISCOVERY_SERVICE"
              value: "elasticsearch"
            - name: NODE_MASTER
              value: "true"
            - name: NODE_DATA
              value: "true"
            - name: HTTP_ENABLE
              value: "true"
            - name: NUMBER_OF_MASTERS
              value: "2"
            - name: ES_JAVA_OPTS
              value: "-Xms256m -Xmx256m"
          ports:
            - containerPort: 9200
              name: http
              protocol: TCP
            - containerPort: 9300
              name: transport
              protocol: TCP
          volumeMounts:
            - mountPath: /data
              name: storage
      volumes:
        - name: storage
          emptyDir: {}
```

いくつかこれまで扱っていない要素が出てきているので整理しておきます。

`.spec.template.spec.serviceAccount` には Pod に割り当てる ServiceAccount を定義しています。ここでは上で作成した `elasticsearch` を指定しています。

`.spec.template.spec.initContainers` はメインのアプリケーションコンテナを起動する前に動かしたいコンテナを定義するところで、何らかのセットアップに使用されることが多いようです。ここでは [Elasticsearch のドキュメント][8] にも言及されている `vm.max_map_count` を設定するために使用しています。

`.spec.template.spec.containers[0].securityContext` では `privileged` と `capabilities` という 2 つの属性を設定しています。これらは恐らく Docker の privileged オプションと同じ目的の設定で、コンテナが自身のネットワークやデバイスの操作を行うためのもののようです (ref. [Docker privileged オプションについて][13])。

`.spec.template.spec.volumes` と `.spec.template.spec.containers[0].volumeMounts` はコンテナにマウントする volume の設定です。前者でマウントする volume を、後者でマウント先を指定しています。ここでは volume の種類として `emptyDir` というのを使用しており、これはコンテナのクラッシュではデータが消えないが、Pod を削除するとデータが消えるというある種の一時ディレクトリのような性質を持つ volume です (ref. [Volumes][14])。

また `.spec.template.spec.containers[0].env` 内で `NAMESPACE` のような書き方をすると Pod のメタ情報を環境変数を介してアプリケーションに伝えることができます (ref. [Expose Pod Information to Containers Through Environment Variables][9])。

上の Deployment を `kubectl create` すると、ちゃんと Elasticsearch クラスタが構築できることが確認できます。

```bash
# Create Deployment
$ kubectl create -f es-deploy.yml 
deployment "es" created

# Get internal IP of Service
$ kubectl get services -l "component=elasticsearch" -o json | jq -r .items[0].spec.clusterIP
10.104.228.184

# SSH to (Kubernetes) node
$ minikube ssh

# Check (Elasticsearch) cluster status
$ curl http://10.104.228.184:9200 
{
  "name" : "98092ebc-6274-4a99-9588-a7e60abf29be",
  "cluster_name" : "myesdb",
  "cluster_uuid" : "Yq53jLFBTh-pKSAnkxxTBQ",
  "version" : {
    "number" : "5.6.2",
    "build_hash" : "57e20f3",
    "build_date" : "2017-09-23T13:16:45.703Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.1"
  },
  "tagline" : "You Know, for Search"
}

# Check number of (Elasticsearch) nodes
$ curl http://10.104.228.184:9200/_cluster/health?pretty
{
  "cluster_name" : "myesdb",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

[1]: https://kubernetes.io/docs/concepts/
[2]: https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services---service-types
[3]: https://kubernetes.io/docs/tutorials/object-management-kubectl/object-management/
[4]: https://kubernetes.io/docs/reference/
[5]: https://github.com/kubernetes/examples/tree/master/staging/elasticsearch
[6]: https://github.com/pires/kubernetes-elasticsearch-cluster
[7]: https://github.com/pires/docker-elasticsearch-kubernetes
[8]: https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html
[9]: https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/
[10]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
[11]: https://ishiis.net/2017/01/21/kubernetes-authentication-authorization/
[12]: https://www.kaitoy.xyz/2017/10/31/retry-dashboard-on-k8s-cluster-by-kubeadm/
[13]: https://qiita.com/muddydixon/items/d2982ab0846002bf3ea8
[14]: https://kubernetes.io/docs/concepts/storage/volumes/
[15]: https://github.com/kubernetes/kubernetes-bootcamp
[16]: https://blog.1q77.com/2017/01/minikube-part2/
[17]: https://github.com/kubernetes/minikube
[18]: https://www.redhat.com/ja/topics/containers/what-is-kubernetes
[19]: https://kubernetes.io/docs/setup/pick-right-solution/
[20]: https://kubernetes.io/docs/reference/api-overview/
[21]: http://www.publickey1.jp/blog/17/dockerkubernetesdockercon_eu_2017.html
[22]: https://aws.amazon.com/jp/blogs/news/amazon-elastic-container-service-for-kubernetes/
[23]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[24]: https://www.slideshare.net/HammoudiSamir/google-container-engine-gke-kubernetes
