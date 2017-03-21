---
layout: post
title: Spring Boot with Docker な開発環境を考える
tags: "java, mysql, spring, gradle, docker"
comments: true
---

Java で Web Application を作成するときに何が気が重いって、環境や設定、クラス等必要なものをごちゃごちゃ揃える必要のあることだと思っています。

[Spring Boot][6] と [Docker][7] を使えばもっと身軽に開発ができるだろうなということで少し内容を整理してみます。
ここではサンプルとして単純な REST API を Spring Boot で実装し、アプリケーションの実行環境とデータベースを Docker Container で準備するという形を目指します。

<img
  src="/images/docker-java/docker-java.png"
  title="spring boot with docker"
  alt="spring boot with docker"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

- [データベース準備](#anchor1)
- [REST API 実装 with Spring Boot](#anchor2)
- [Java Application 実行用 Image 作成](#anchor3)
- [docker-compose でまとめて管理](#anchor4)

### 環境

- OS: Arch Linux (linux kernel: 4.9.3)
- Java: openjdk version "1.8.0\_121"
- docker: Docker version 17.03.0-ce, build 60ccb2265b

---

<a id="anchor1"></a>

### データベース準備

データベースとしては公式に Docker image を提供している [MySQL][1] を使用します。 image の説明を見ると、データベースのセットアップを容易にする設定が用意されていることがわかります。

#### Container 作成

MySQL image からの container 作成は通常通り `docker container run` で行います。その際に以下の環境変数を定義することで、自動的にデータベースの設定を行うことができます。

- `MYSQL_DATABASE` : database の作成
- `MYSQL_USER` : 新規ユーザの作成
- `MYSQL_PASSWORD` : 上記ユーザのパスワード設定
- `MYSQL_ROOT_PASSWORD` : root のパスワード設定

```
$ docker container run --name mysqld -e MYSQL_DATABASE=mydb -e MYSQL_USER=foo -e MYSQL_PASSWORD=foo-pass -e MYSQL_ROOT_PASSWORD=root-pass -d mysql
```

起動後、設定した情報でデータベースへ接続可能なことが確認できます。

```
$ docker container exec -it mysqld /bin/sh
$ mysql -u $MYSQL_USER -p $MYSQL_DATABASE
Enter password:
mysql>
...
```

#### データベースのセットアップ

データベース起動時、 DDL を実行してテーブル等の定義を行いたい場合も多いと思います。 `/docker-entrypoint-initdb.d` に `*.sql` ファイルをマウントしておけば、任意の SQL を初回 container 起動時に実行することができます。

```
docker run -v $(PWD)/sql:/docker-entrypoint-initdb.d -d --name mysqld mysql
```

<a id="anchor2"></a>

### REST API 実装 with Spring Boot

Spring Boot を使用すれば 簡単な REST API はさくっと作成することができます。Spring Boot は tutorial が豊富なのですが、今回はその中の [React.js and Spring Data REST][3] を参考にしています。

#### `build.gradle` の編集

今回のアプリケーションではビルドや依存ライブラリの管理に [Gradle][4] を使用します。 ビルドスクリプト `build.gradle` の雛形は  [Spring Initializer][8] から用意できるので、それを元に以下のように編集します。

```groovy
buildscript {
    ext {
        springBootVersion = '1.5.2.RELEASE'
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'

jar {
    baseName = 'boot-get-started'
    version = '0.0.1-SNAPSHOT'
}

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

dependencies {
    compile('org.springframework.boot:spring-boot-starter-web')
    compile("org.springframework.boot:spring-boot-starter-data-jpa")
    compile("org.springframework.boot:spring-boot-starter-data-rest")
    compile('mysql:mysql-connector-java:6.0.6')
    compileOnly('org.projectlombok:lombok')
    testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

`gradlew clean build` なんかを一回実行し、2杯ぐらいお茶を味わい時間を潰せば、必要なライブラリのインストールが完了します。

#### Entity の作成

今回は適当な User クラスを Entity として定義します。

```java
package com.tiqwab.example;

import lombok.Data;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

@Data
@Entity
public class User {

    @Id
    @GeneratedValue
    private Integer id;
    private String firstName;
    private String lastName;

    private User() {
    }

    public User(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

}
```

#### Repository の作成

User クラスと対応する Repository を作成します。
`CrudRepository` を継承した interface を定義することで、自動的に CRUD に対応する実装が提供されます。

また依存ライブラリに `org.springframework.boot:spring-boot-starter-data-rest` が定義されているため、 この Repository を元に User 用の REST API も定義されます (後述)。

```java
package com.tiqwab.example;

import org.springframework.data.repository.CrudRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserRepository extends CrudRepository<User, Integer> {
}
```

#### `application.yml` の編集

アプリケーションの設定ファイル `application.yml` を作成します。この中にデータベースまわりや REST 関連でいくつか設定を追加します。

- `spring.datasource`
  - データベース接続情報の設定
- `spring.jpa.hibernate.ddl-auto`
  - Entity をもとに 起動時に DDL を実行するか
    - update: なければ作る
    - create: 起動時に存在すれば破棄、その後作成
    - create-drop: 起動時に作成、終了時に破棄
    - none: 実行しない
- `spring.data.rest.base-path`
  - REST API の base になるパス

```
spring:
    datasource:
        driverClassName: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://localhost/mydb
        username: sboot
        password: sboot
    jpa:
        database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
        show-sql: true
        hibernate:
            ddl-auto: update
    data:
        rest:
            base-path: /api
```

ちなみにここで定義している設定をはじめ、 `application.yml` に書くような設定は全て環境変数で上書くことができます。 (例えばアプリケーション実行時、 `spring.jpa.hibernate.ddl-auto=None` という環境変数があれば update ではなく none が設定される)

`spring.jpa.hibernate.ddl-auto` に関しては今回 MySQL image 側の仕組みでセットアップを行っているので none でも可です。 アプリケーション側でセットアップをやる場合、以下の2つのアプローチが可能です ([参考][9])。

- Hibernate (ORM) の仕組みを利用
  - ddl-auto を update (or create) に
  - 初期データの投入は `hibernate.hbm2ddl.import_files` プロパティで
- ORM と Spring Data の仕組みを利用
  - ddl-auto を update (or create) に
  - 初期データの投入は `DataSourceInitializer` クラスを利用して

開発の観点でいえば、下の方法でいくのが一番直感的かつ簡単でいいかなと思います。

<a id="anchor3"></a>

### Java Application 実行用 Image 作成

Spring Boot で作成したアプリケーションの Docker Image 作成例は[公式][5]で紹介されています。
上のものを参考にしつつ、開発時を想定した image を作成します。

作成例との主な差異は以下の2点です。

- base image に Alpine Linux の使用
  - image のサイズを小さくできる
- jar を ADD するのではなくアプリケーションのルートディレクトリをマウントする
  - ビルド後に毎回 image から作り直す必要を無くす

```
# use alpine as base image
FROM openjdk:jdk-alpine
# recommended by spring boot
VOLUME /tmp
# create directory for application
RUN mkdir /app
WORKDIR /app
# add user for application
RUN adduser -S sboot
USER sboot
# set entrypoint to execute spring boot application
ENV JAVA_OPTS=""
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar build/libs/boot-get-started-0.0.1-SNAPSHOT.jar" ]
```

通常はこの後 `docker image build .` で image を作成しますが、今回は下記の docker-compose 内で自動で作成されるので、ここでは作成しません。

<a id="anchor4"></a>

### docker-compose でまとめて管理

上2つの container を別々に管理するのは面倒なので、docker-compose でまとめて管理します。

アプリケーションと同じディレクトリに `docker-compose.yml` ファイルを作成し、実行したいコンテナ情報を記述します。

```
version: '2'
services:
    db:
        image: mysql
        ports:
            - "3306:3306"
        volumes:
            - ./sql:/docker-entrypoint-initdb.d
        environment:
            MYSQL_DATABASE: mydb
            MYSQL_USER: sboot
            MYSQL_PASSWORD: sboot
            MYSQL_ROOT_PASSWORD: root
    app:
        build: .
        image: tiqwab/boot:0.0.1
        depends_on: 
            - db
        ports:
            - "8080:8080"
        volumes:
            - .:/app
        environment:
            spring.datasource.driverClassName: "com.mysql.cj.jdbc.Driver"
            spring.datasource.url: "jdbc:mysql://db/mydb"
            spring.datasource.username: "sboot"
            spring.datasource.password: "sboot"
```

これでアプリケーションを実行する環境が整いました。
まだ上で作成したアプリケーション用の image をビルドしていないため、 `docker-compose up --build` で image を作成しつつコンテナを起動してみます。

```
$ docker-compose up --build
```

起動後、User に対する REST API が実装されていることが確認できます。

```
$ curl http://localhost:8080/api/users
{
  "_embedded" : {
    "users" : [ {
      "firstName" : "Taro",
      "lastName" : "Yamada",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/api/users/1"
        },
        "user" : {
          "href" : "http://localhost:8080/api/users/1"
        }
      }
    }, {
      "firstName" : "Hanako",
      "lastName" : "Tanaka",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/api/users/2"
        },
        "user" : {
          "href" : "http://localhost:8080/api/users/2"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/users"
    },
    "profile" : {
      "href" : "http://localhost:8080/api/profile/users"
    }
  }
}

$ curl -X POST http://localhost:8080/api/users -d '{"firstName": "Jiro", "lastName": "Ohta"}' -H "Content-Type:application/json"
{
  "firstName" : "Jiro",
  "lastName" : "Ohta",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/users/5"
    },
    "user" : {
      "href" : "http://localhost:8080/api/users/5"
    }
  }
}

$ curl -X PUT http://localhost:8080/api/users/5 -d '{"firstName": "Jiro", "lastName": "Iida"}' -H "Content-Type:application/json"
{
  "firstName" : "Jiro",
  "lastName" : "Iida",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/api/users/5"
    },
    "user" : {
      "href" : "http://localhost:8080/api/users/5"
    }
  }
}

$ curl-X DELETE http://localhost:8080/api/users/6 -w %{http_code}
> 204
```

というわけで、 Spring Boot と Docker で簡単に REST API の開発環境を構築できました。

これ以後コンテナの実行、停止は docker-compose で以下のようにできます (もちろんこれらもただのコンテナなので `docker container ...` でも管理できます)。

```
# create and start containers in background
$ docker-compose up -d

# stop containers
$ docker-compose stop

# remote containers
$ docker-compose rm
```

### Summary

- Spring Boot で REST API をさくっと実装できる
- Docker で開発環境をさくっと用意できる
- docker-compose で複数の container をさくっとまとめられる

### 参考

- [boot-get-started][10]
  - 今回作成したサンプルコード
- [Spring Boot With Docker][5]
  - Spring Boot の実行環境を Docker で用意する公式のチュートリアル
- [React.js and Spring Data REST][3]
  - React.js 、Spring Boot を使用したアプリケーション開発の公式チュートリアル

[1]: https://hub.docker.com/_/mysql/ "library/mysql - Docker Hub"
[2]: http://dqn.sakusakutto.jp/2015/10/docker_mysqld_tutorial.html "tutorial of mysql image"
[3]: https://spring.io/guides/tutorials/react-and-spring-data-rest/ "React.js and Spring Data REST"
[4]: https://gradle.org/ "Gradle"
[5]: https://spring.io/guides/gs/spring-boot-docker/ "Spring Boot with Docker"
[6]: https://projects.spring.io/spring-boot/ "Spring Boot"
[7]: https://www.docker.com/ "Docker"
[8]: https://start.spring.io/ "Spring Initializer"
[9]: http://stackoverflow.com/questions/24508223/multiple-sql-import-files-in-spring-boot "multiple sql import files in spring boot"
[10]: https://github.com/tiqwab/example/tree/master/boot-get-started "Spring Boot Get Started With Docker"
