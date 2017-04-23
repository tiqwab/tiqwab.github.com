---
layout: post
title: Spring Boot ハンズオンと学ぶ 2
tags: "spring boot, java"
comments: true
---

[前回][13]に引き続き [Spring Boot ハンズオン][7]を行いながら、理解の足りないところ、気になるところをいくつかまとめてみました。

最終的に実装したものは [jsug-shop-zero][8] に置いています。

1. [Repository with Spring Data JPA](#anchor1)
2. [Paging](#anchor2)
3. [Session storage with Redis](#anchor3)

### 環境

- Spring Boot: 1.5.2.RELEASE

<a id="anchor1"></a>

### 1. Repository with Spring Data JPA

[Spring Data JPA][2] では [Repository][3] インタフェースに特定の命名規則に従ってメソッドを定義することで、自動的に実装が補完されるという仕組みがあります。

自動で導出可能なメソッドは以下のようなものです。

- `find...By`
- `read...By`
- `query...By`
- `count...By`
- `get...By`

`find...By` 以外はあまり見慣れないものだったので、少し試してみます。

```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface PersonRepository extends JpaRepository<Person, Long> {
    List<Person> findByFirstName(String firstName);
    List<Person> readByFirstName(String firstName);
    List<Person> queryByFirstName(String firstName);
    List<Person> countByFirstName(String firstName);
    List<Person> getByFirstName(String firstName);
}
```

```
# find...By
Hibernate: select person0_.id as id1_5_, person0_.age as age2_5_, person0_.birth_day as birth_da3_5_, person0_.first_name as first_na4_5_, person0_.last_name as last_nam5_5_ from person person0_ where person0_.first_name=?
# read...By
Hibernate: select person0_.id as id1_5_, person0_.age as age2_5_, person0_.birth_day as birth_da3_5_, person0_.first_name as first_na4_5_, person0_.last_name as last_nam5_5_ from person person0_ where person0_.first_name=?
# query...By
Hibernate: select person0_.id as id1_5_, person0_.age as age2_5_, person0_.birth_day as birth_da3_5_, person0_.first_name as first_na4_5_, person0_.last_name as last_nam5_5_ from person person0_ where person0_.first_name=?
# count...By
Hibernate: select count(person0_.id) as col_0_0_ from person person0_ where person0_.first_name=?
# get...By
Hibernate: select person0_.id as id1_5_, person0_.age as age2_5_, person0_.birth_day as birth_da3_5_, person0_.first_name as first_na4_5_, person0_.last_name as last_nam5_5_ from person person0_ where person0_.first_name=?
```

`count...By` は予想通り取得件数の表示なのですが、他は発行されたクエリ上はここで見る限り違いがありません。

これらのメソッドは適当な場所に 'distinct', 'and', 'or', 'order' 等のキーワードを加えるとその内容を踏まえたクエリが生成されます。

またメソッドの引数に `Sort` や `Pageable` といったクラスを設定することでそれぞれ取得順のソート、ページングを考慮した取得を行うことができます。

```java
public interface PersonRepository extends JpaRepository<Person, Long> {
    List<Person> findByFirstName(String firstName, Sort sort);
    Page<Person> findByFirstName(String firstName, Pageable pageable);
}
```

```
# find..By with Sort
Hibernate: select person0_.id as id1_5_, person0_.age as age2_5_, person0_.birth_day as birth_da3_5_, person0_.first_name as first_na4_5_, person0_.last_name as last_nam5_5_ from person person0_ where person0_.first_name=? order by person0_.first_name asc
# find..By with Pageable
Hibernate: select person0_.id as id1_5_, person0_.age as age2_5_, person0_.birth_day as birth_da3_5_, person0_.first_name as first_na4_5_, person0_.last_name as last_nam5_5_ from person person0_ where person0_.first_name=? limit ? offset ?
Hibernate: select count(person0_.id) as col_0_0_ from person person0_ where person0_.first_name=?
```

これらの方法で自動導出できないクエリも自分で定義することができます。
方法としてはアノテーションで指定する、プロパティファイル内に定義するといったものがあります ([参考][1])。

#### 参考

- [Spring Data JPA Reference Documentation][4]

<a id="anchor2"></a>

### 2. Paging

Spring Boot ハンズオンでは [GoodsController][17] にページング機能を見越した実装を行っています。
しかしハンズオンの中では実際に機能を追加するところまで行っていないので、ここでその先の手順を整理しておきたいと思います。

#### Controller

ページングを実装するため、Controller に以下の内容を加えます (この内容はハンズオンの実装に含まれています)。

- (1) Controller のメソッド引数に `Pageable pageable` を指定する
  - この際、 `@PageableDefault` でページングに関するデフォルト値を設定できる
  - グローバルなデフォルト値の設定は `PageableHandlerMethodArgumentResolver` で ([参考][5])
- (2) Service (最終的に Repositroy) に `Paging` オブジェクトを渡して情報を取得する
  - 上述のように Spring Data Jpa を使用すれば楽に実装できる
- (3) Model に `Pageable` オブジェクトを設定して、 view から使用できるようにする

```java
@Controller
public class GoodsController {
    ...

    @Autowired
    GoodsService goodsService;

    @RequestMapping(value = "/")
    public String showGoods(@RequestParam(value = "categoryId", defaultValue = "1") Long categoryId,
                            // (1)
                            @PageableDefault(
                                page = 0,
                                size = 3,
                                sort = {"id"},
                                direction = Sort.Direction.ASC
                                ) Pageable pageable,
                            Model model) {
        // (2)
        Page<Goods> page = goodsService.findByCategoryId(categoryId, pageable);
        // (3)
        model.addAttribute("page", page);
        model.addAttribute("categoryId", categoryId);
        return "goods/showGoods";
    }

    ...
}
```

これで `http://localhost:8080/?page=1&size=5&sort=id` のようにリクエストパラメータを介して Paging 用の情報を取得することができます。

#### View

サーバ側で用意したページング処理を利用する形で `goods/showGoods.html` に変更を加えます。
ここではシンプルに先頭、末尾、一つ前、一つ後のページへ遷移するような機能を考えます。

- (1) `th:fragment` は他から参照して使用するための仕組み
  - ページングとは直接関係しないが、どこか共用のものとして定義しておくのが良きかと
- (2) `${page.first}` と `${page.last}` で表示中のページが先頭、末尾なのかを確認できる
  - ここではそれに応じて class を設定している

```html
    <!-- (1) -->
    <div th:fragment="pagination" style="display: inline-block;">
        <ul>
            <!-- Go to the first -->
            <!-- (2) -->
            <li th:class="${page.first} ? 'disabled' : ''" style="display: inline;">
                <span th:if="${page.first}">&lt;&lt;</span>
                <a th:if="${not page.first}" th:href="@{${url}(page=0)}">&lt;&lt;</a>
            </li>
            <!-- Go to the prev -->
            <li th:class="${page.first} ? 'disabled' : ''" style="display: inline;">
                <span th:if="${page.first}">&lt;</span>
                <a th:if="${not page.first}" th:href="@{${url}(page=(${page.number}-1))}">&lt;</a>
            </li>
            <!-- Go to the next -->
            <li th:class="${page.last} ? 'disabled' : ''" style="display: inline;">
                <span th:if="${page.last}">&gt;</span>
                <a th:if="${not page.last}" th:href="@{${url}(page=(${page.number}+1))}">&gt;</a>
            </li>
            <!-- Go to the last -->
            <li th:class="${page.last} ? 'disabled' : ''" style="display: inline;">
                <span th:if="${page.last}">&gt;&gt;</span>
                <a th:if="${not page.last}" th:href="@{${url}(page=(${page.totalPages}-1))}">&gt;&gt;</a>
            </li>
        </ul>
        <!-- Show the current page number -->
        <div><span th:text="${page.number+1} + '/' + ${page.totalPages}">1/3</span></div>
    </div>
```

#### 参考

- [ページネーション - TERASOLUNA][5]
- [Spring Boot + Thymeleaf でページング機能を実装する][6]

<a id="anchor3"></a>

### 3. Session storage with Redis

Spring Boot ハンズオン中では、アプリケーションを複数台のサーバにスケールアウトさせた際のセッション管理に [Redis][16] を使用しています。

このとき Spring では [Spring Data Redis][14] や [Spring Session][15] が活躍してくれるようなので、その手順をまとめます。

#### RedisTemplate による操作

Spring Boot から Redis を使用する場合、最低限必要な設定は以下の 2 つです。

- [Spring Data Redis][14] を依存先の設定に追加する
- 接続する Redis サーバの情報を設定する

今回はビルドツールに Gradle を使用しているので、 `build.gradle` に `spring-boot-starter-data-redis` への依存設定を追加します。

```groovy
dependencies {
    compile('org.springframework.boot:spring-boot-starter-data-redis')
    ...
}
```

接続する Redis サーバの情報は例えば `application.properties` に設定します。

```
spring.redis.host=localhost
spring.redis.port=6379
```

アプリケーションから Redis への接続を確認するために、 Spring が提供する `RedisTemplate` クラスを使用してデータの設定、取得を行います。

`RedisTemplate` でデータの保存を行う例は以下のようになります。

```java
@Slf4j
@Component
public class RedisSampleRunner implements CommandLineRunner {

    @Autowired
    RedisTemplate<String, String> redisTemplate;

    @Override
    public void run(String... args) throws Exception {
        // Save value of STRING type
        redisTemplate.opsForValue().set("str1", "val1");

        // Save value of LIST type
        redisTemplate.opsForList().leftPush("list1", "1");
        redisTemplate.opsForList().leftPushAll("list1", "1", "2", "3");

        // Save value of SET type
        redisTemplate.opsForSet().add("set1", "1", "2", "3");
        redisTemplate.opsForSet().add("set1", "1");

        // Save value of HASH type
        redisTemplate.opsForHash().put("hash1", "firstName", "Taro");
        redisTemplate.opsForHash().put("hash1", "lastName", "Yamada");

        // Save value of ZSET type
        redisTemplate.opsForZSet().add("zset1", "history1", 1);
        redisTemplate.opsForZSet().add("zset1", "history2", 2);
        redisTemplate.opsForZSet().add("zset1", "history1", 3);
    }

}
```

このとき Redis では以下のようなコマンドが実行されていることが確認できます。

```
> redis-cli monitor
1492920298.682293 [0 172.20.0.1:49036] "SET" "str1" "val1"
1492920298.684174 [0 172.20.0.1:49036] "LPUSH" "list1" "1"
1492920298.684987 [0 172.20.0.1:49036] "LPUSH" "list1" "1" "2" "3"
1492920298.686585 [0 172.20.0.1:49036] "SADD" "set1" "1" "2" "3"
1492920298.686907 [0 172.20.0.1:49036] "SADD" "set1" "1"
1492920298.688316 [0 172.20.0.1:49036] "HSET" "hash1" "firstName" "Taro"
1492920298.688669 [0 172.20.0.1:49036] "HSET" "hash1" "lastName" "Yamada"
1492920298.690467 [0 172.20.0.1:49036] "ZADD" "zset1" "1.0" "history1"
1492920298.690849 [0 172.20.0.1:49036] "ZADD" "zset1" "2.0" "history2"
1492920298.691120 [0 172.20.0.1:49036] "ZADD" "zset1" "3.0" "history1"
```

次に格納したデータを取得してみます。

```java
        // Get value of STRING type
        log.info("get str1: {}", redisTemplate.opsForValue().get("str1"));

        // Get value of LIST type
        log.info("lrange list1 0 -1: {}", redisTemplate.opsForList().range("list1", 0, -1));

        // Get value of SET type
        log.info("smembers set1: {}", redisTemplate.opsForSet().members("set1"));

        // Get value of HASH type
        log.info("hget hash1 firstname: {}", redisTemplate.opsForHash().get("hash1", "firstName"));

        // Get value of ZSET type
        // How to specify '+inf' and '-inf'?
        log.info("zrangebyscore zset1 -inf +inf withscores:");
        redisTemplate.opsForZSet().rangeByScoreWithScores("zset1", 0, Integer.MAX_VALUE)
                .stream()
                .forEach(t -> log.info("({},{})", t.getValue(), t.getScore()));
```

アプリケーションのログを見ると、確かにデータが取得されていることがわかります。

```
com.tiqwab.example.RedisSampleRunner     : get str1: val1
com.tiqwab.example.RedisSampleRunner     : lrange list1 0 -1: [3, 2, 1, 1]
com.tiqwab.example.RedisSampleRunner     : smembers set1: [1, 2, 3]
com.tiqwab.example.RedisSampleRunner     : hget hash1 firstname: Taro
com.tiqwab.example.RedisSampleRunner     : zrangebyscore zset1 -inf +inf withscores:
com.tiqwab.example.RedisSampleRunner     : (history2,2.0)
com.tiqwab.example.RedisSampleRunner     : (history1,3.0)
```

このとき Redis に発行されるクエリは以下のようになります。

```
> redis-cli monitor
1492921478.550161 [0 172.20.0.1:49334] "GET" "str1"
1492921478.554306 [0 172.20.0.1:49334] "LRANGE" "list1" "0" "-1"
1492921478.558467 [0 172.20.0.1:49334] "SMEMBERS" "set1"
1492921478.561299 [0 172.20.0.1:49334] "HGET" "hash1" "firstName"
1492921478.569065 [0 172.20.0.1:49334] "ZRANGEBYSCORE" "zset1" "0.0" "2.147483647E9" "withscores"
```

#### セッションを Redis へ保存

Spring Boot で作成したアプリケーションから Redis への接続が行えることを確認したので、次にセッションを Redis に保存するよう変更します。

セッション管理するクラス例として以下の `Cart` クラスを用意しました。

- (1) `@Scope` で `Cart` オブジェクトのコンテナ上でのライフサイクルを設定する
  - ここではセッションを指定し、 (デフォルトでは) `HttpSession` に格納されるようにする
- (2) `Cart` クラスに `Serializable` を実装させる
  - セッション管理という点では必要ではない
  - このあと Redis に登録するために必要となる

```java
@Component
// (1)
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
@ToString
// (2)
public class Cart implements Serializable {

    private OrderLines orderLines;

    public Cart() {
        this.orderLines = new OrderLines();
    }

    public List<OrderLine> getLines() {
        return orderLines.getLines();
    }

}
```

Redis をセッションの格納先とするために以下の 2 つを行います。

- 依存先に [Spring Session][15] の追加
- `@EnableRedisHttpSession` の追加

`build.gradle` に Spring Session への依存を追加します。

```groovy
dependencies {
    compile('org.springframework.session:spring-session')
    ...
}
```

Configuration クラスに `@EnableRedisHttpSession` を付加します。

```java
import org.springframework.session.data.redis.config.annotation.web.http.EnableRedisHttpSession;

@Configuration
@EnableRedisHttpSession
public class CacheConfig {

}
```

最低限必要な手順はこれで終わりです。
他に格納する値を json 化するといった設定もできるようです (これは自分で `RedisTemplate` や `RedisCacheManager` を 明示的に DI コンテナに登録するというのが必要になるが、特定のクラスだけではなく汎用的に行う方法がうまく確認できなかった)。

#### 参考

- [Redis の使い方][9]
- [Spring Boot で Spring Data Redis を利用する][10]
- [Spring Boot で Redis を使う！][11]
- [Spring Boot で Redis にキャッシュする際の抽象化について][12]

[1]: http://qiita.com/tag1216/items/55742fdb442e5617f727 "Spring Data JPA クエリ実装方法まとめ"
[2]: http://projects.spring.io/spring-data-jpa/ "Spring Data JPA"
[3]: https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/Repository.html
[4]: https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.query-creation
[5]: https://terasolunaorg.github.io/guideline/public_review/ArchitectureInDetail/Pagination.html
[6]: http://qiita.com/KevinFQ/items/ca68a3001bae19f92879
[7]: http://jsug-spring-boot-handson.readthedocs.io/en/latest/index.html
[8]: https://github.com/tiqwab/example/tree/master/jsug-shop-zero
[9]: http://promamo.com/?p=3358
[10]: http://qiita.com/yoshidan/items/f7c10a43d2a40c3ce8df
[11]: http://blog.rakugakibox.net/entry/2015/07/27/spring-boot-with-redis
[12]: https://ishiis.net/2016/09/02/spring-redis-cache/
[13]: {% post_url 2017-04-15-think-with-spring-boot-handson %} "Spring Boot ハンズオンと学ぶ 1"
[14]: http://projects.spring.io/spring-data-redis/ "Spring Data Redis"
[15]: http://projects.spring.io/spring-session/ "Spring Session"
[16]: https://redis.io/ "Redis"
[17]: https://github.com/making/jsug-shop/blob/ch06/src/main/java/jsug/app/goods/GoodsController.java
