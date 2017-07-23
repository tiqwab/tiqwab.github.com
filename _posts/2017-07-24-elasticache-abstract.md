---
layout: post
title: Performance At Scale With Amazon Elasticache のまとめ
tags: "redis, aws, elasticache"
comments: true
---

[Performance At Scale With Amazon Elasticache][2] は Amazon が書いている Elasticache に関するドキュメントなのですが、Elasticache に限らずインメモリキャッシュを使用するときのベストプラクティスを学ぶ参考になりそうだったので、ざっと読んでまとめてみました。

Elasticache 自体は ストレージエンジンとして Memcached と Redis を用意していますが、ここでは主に Redis に注目しています。

### Abstract

- キャッシュ用途としてのインメモリキーストア
  - 頻繁にアクセスされるようなデータをキャッシュすればアプリケーションのパフォーマンスを上げることができる
  - 正しく使えばスケールアップ時のコストを抑えることができる
  - キャッシュだけではなく analytics や recommendation engine といった用途にも使える

### Alternatives to Elasticache

Elasticache 以外のキャッシュ実装方法についての長所と短所。

#### Amazon CloudFront (CDN)

- Pro: Web ページ, 画像等静的なデータのキャッシュができる
- Con: 動的コンテンツもキャッシュできるけど生成後の結果しか保存できない
  - (あとは少し前話題になったように動的コンテンツへのキャッシュは色々注意が必要。[こちら][1] に書いてあるように完全にパーソナライズされたコンテンツだと厳しい)

#### Amazon RDS (RDB) の Read Replicas

- Con: インメモリキャッシュほど速くない
- Con: Primary のデータを replicate するだけなので計算、集計した結果を使用するようなことはできない

#### アプリケーションサーバ上に持つ

- Pro: 実装がシンプルで容易
- Con: スケールアップ時に空のキャッシュからスタートしてしまう
  - データ層へ急激な負担がかかる
- Con: アプリケーションサーバが複数台あるときにそれぞれで別のキャッシュを持つことになる

### Memcached vs Redis

どういうときに Memcached あるいは Redis が適しているのか。

- 単純なデータのキャッシュなら Memcached
- データが増加していく傾向にありスケールアウトさせたいなら Memcached
- マルチスレッドの性能を望むなら Memcached
- 保存するデータに複雑なデータ構造 (list, hash, set etc.) を望むなら Redis
- Pub/sub な使い方を望むなら Redis
- ディスクへの永続化を考えるなら Redis
- Multi AZ で動かして failover させたいなら Redis

Memcached が multi thread であり Redis が single thread なのは大きな違いの一つかなと思います。

### Elasticache for Memcached

- キャッシュの主要な目的は RDB 等の一次データストアへの read 量をおさえること
- 頻繁に read されるが write はそこまでではないデータはキャッシュのよい対象になる
  - RDB 等をスケールアップするより安く済み容易なことが多い
- Read request が spike するような場合、キャッシュはその対策になる

### Consistent Hashing (Sharding)

キャッシュノードが複数台で構成される場合に、キャッシュを載せるノードをどう決定するか。

- キーのハッシュ値を取りノード台数の modulo を計算するのはわかりやすいけど、台数が変更になったときに大きな影響を受ける
- [Consistent Hashing][3] のような remap される key が少なくなるような実装が望ましく、実際多くの client library はそうしたものを使用しているはず

### Caching Design Patterns

特にキャッシュ実装でよく見る 2 つのパターンについてここでは抜き出します。

#### Be Lazy

Lazy caching, または Lazy population と呼ばれるキャッシュ形式。

1. アプリケーションがキャッシュ対象なデータへのリクエストを受け取る
2. キャッシュを探しにいく
3. キャッシュがあればそのデータを返す
4. 無ければ一次データストアへ取りに行き結果を返す。このとき取得したデータはキャッシュにも入れておき次回のリクエスト時にはキャッシュから取得できるようにする。

#### Write On Through

一次データストアにデータを write したときに、キャッシュへもそれを反映させるという方法。
先回りしてデータをキャッシュするということになるので Pros, Cons もそれに対応したものになる。

- Pro: Cache miss を防止できる
- Pro: 常に最新のデータがキャッシュされていることが期待できる
- Con: 実際はリクエストされないデータを入れてしまい無駄にメモリを消費することになるかも
- Con: キャッシュへの書き込み頻度が増える
- Con: キャッシュノードが落ちた場合、立ち上げ時にそのノードにデータを supply する必要がある

Be Lazy 方式と Write On Through 方式はそれぞれ Read 時にキャッシュを用意するもの、Write 時にキャッシュを用意するものなのでお互いに補完し合えるものであり、組み合わせて使用できます。

なので基本的には Be Lazy でキャッシュを実装し、その中で必要なデータに対しては Write On Through も組み合わせるというのが一つの戦略として推奨されています。

### Expiration Date

キャッシュの expiration date をどんな値に設定するか。
あらゆるデータ、シチュエーションにこうすべきというものはないが、一つの考え方としては以下のものがある。

- あらゆるキャッシュデータに time to live (TTL) を設定する (ただし Write On Through なもの以外)
  - 数時間、数日間といった単位
- 頻繁に更新されるようなデータであれば数秒程度の TTL を設定する
  - 一次データストアへの急激なリクエスト増加に対応できる
- Russian doll caching
  - Ruby on Rails で使用されているキャッシュ実装
    - [Caching with Rails][4]
    - [The performance impact of "Russian doll" caching][5]

### The Thundering Herd

キャッシュ実装における Thundering herd, あるいは dog piling とも呼ばれる問題は expire したキャッシュデータに対して同時に大量のリクエストがとんだ場合に一次データストアへ急激な負荷をかけてしまうというものです。

これと同じ問題は新しく空のキャッシュノードを追加した場合にも起こり得ます。

Thundering herd 問題を解決するための一つの方法としては事前にキャッシュを用意しておく (キャッシュノードを prewarm する) というものがあります。

1. 問題となるキャッシュリクエストと同様のことを行うスクリプトを用意する
2. もし lazy caching を実装しているならば、上のスクリプトでヒットしなかったキャッシュデータは自動的にキャッシュに入れられる
3. 新規キャッシュノードを追加する前に 1. のスクリプトを実行する
4. もし定期的にキャッシュノードの増減を行う場合でも、何らか自動化できるはず

TTL に関してはある程度ランダムな値にすることでキャッシュデータが同一タイミングで expire されるのを防ぐという設定方法もある。

### Advanced Datasets with Redis

Redis を使用したキャッシュ設計を具体的なユースケースを挙げつつ考えています。

- Game Leaderboards
- Recommendation Engines
- Chat and Messaging
- Queues

### Monitoring Cache Efficiency

Elasticache のモニタリングについて。

- CPU
  - 高い CPU 使用率は性能に対して大量のリクエストをさばいている、あるいは Redis の場合データセットに対する操作によるもの (多分 sort とか) と考えられる
  - CloudWatch のモニターの場合、CPU コア 1 つに対しての値として表示されることに注意。Redis は single thread なのでもし 2 core のインスタンスで 45 % と表示されているなら、実際は 90 % と捉えるべき
- Evictions
  - メモリ使用量が上限に達した場合、キーを evict して必要なスペースを作ろうとする
  - あまりに eviction が多いときはスケールアップ、スケールアウトを考えるべき
- CacheMisses
  - キャッシュが見つからなかった回数のモニタ
  - これ自体は問題ではないが、頻繁に起こり、かつ Evictions も多い場合はメモリが足りていないと考えられる
- BytesUsedForCacheItems
  - Elasticache で管理されているインメモリデータストアが使用しているメモリの量
- SwapUsage
  - 発生していると何かまずいことになっていそう

正常なキャッシュノードでは cache bytes はほぼ "max memory" パラメータに設定された値と等しくなります。
また Cache hits の方が cache misses よりも速く増加していくはずです。

[1]: https://blogs.akamai.com/jp/2015/09/bestpractice-web-perfromance02.html
[2]: https://d0.awsstatic.com/whitepapers/performance-at-scale-with-amazon-elasticache.pdf
[3]: http://www.tom-e-white.com/2007/11/consistent-hashing.html
[4]: http://edgeguides.rubyonrails.org/caching_with_rails.html#russian-doll-caching
[5]: https://signalvnoise.com/posts/3690-the-performance-impact-of-russian-doll-caching
