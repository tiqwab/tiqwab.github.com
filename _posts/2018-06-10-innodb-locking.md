---
layout: post
title: MySQL のロック概観
tags: "mysql"
comments: true
---

MySQL を扱っているとたまに「このクエリだと何に対してロックを取るんだっけ？」、「デッドロックのエラー出てるけど原因がわからない」等悩むことがあるので、MySQL におけるロックの挙動を雰囲気だけでも理解したいなと思いました。

MySQL のロックまわりに関しては検索すれば既に多くの解説が得られるので、ここではそれらを参考にしながら自分にとってポイントになる部分を適宜まとめたりサンプルを実行したりしています。

なお途中出てくるサンプルは MySQL 5.7 (not 8.0) の REPEATABLE READ で確認したものです。

- [トランザクション分離レベル](#transaction-isolation-level)
- [consistent read と locking read](#consistent-locking-read)
- [shared lock と exclusive lock](#shared-exclusive-lock)
- [InnoDB Locking](#innodb-locking)
- [DML 文は暗黙的に exclusive lock を取る](#dml-exlusive-lock)
- [検索に使用された行がロックの対象になる](#lock-not-only-result-rows)
- [ロック取得状況確認方法](#show-engine-innodb-status)
- [各記事で見る (MySQL) ロックのハマりどころ](#other-posts)

<div id="transaction-isolation-level" />

### トランザクション分離レベル

MySQL に限った話ではありませんが、ロックの目的の一つは ACID 特性の I (isolation) を実現することなので、まずはトランザクション分離レベルについて概要をおさらいするのがいいかなと思います。下記のトランザクション分離レベルの項が簡潔にまとめられています。

- [MySQL の InnoDB のロック挙動調査][12]

特に重要な以下の表を抜き出します。

```
| 分離レベル      | ダーティリード | ファジーリード | ファントムリード |
------------------|----------------|----------------|------------------|
| READ UNCOMMITED | あり           | あり           | あり             |
| READ COMMITED   | なし           | あり           | あり             |
| REPEATABLE READ | なし           | なし           | あり             |
| SELIALIZABLE    | なし           | なし           | なし             |
```

MySQL 特有の話をすると

- デフォルトの分離レベルは REPEATABLE READ
- REPEATABLE READ では consistent read においてはファントムリードも防止する (後述)

というのがポイントです。

<div id="consistent-locking-read" />

### consistent read と locking read

トランザクション中の読み取り操作は大別すると 2 種類あります。

- [MySQL 5.7 Reference Manual :: 14.5.2.3 Consistent Nonlocing Reads][13]
- [MySQL 5.7 Reference Manual :: 14.5.2.5 Locking Reads][14]

#### consistent read

- トランザクション内での (locking read ではない) 読み取りは、そのトランザクション中はじめに read した時点のスナップショットを参照するような挙動になる
- スナップショットは全テーブル対象 (クエリしたテーブルのみではない)
- スナップショットの仕組みは undo ログに基づくので lock は取得しない

#### locking read

- shared lock をとるクエリと exclusive lock をとるクエリがある
- locking read したレコードに関してはスナップショットではなくコミット済みの最新の値が取得される

上の内容を確認するために以下の 2 テーブル用意します。

```
mysql> SHOW CREATE TABLE lock_sample;
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------+
| Table       | Create Table                                                                                                                                     |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------+
| lock_sample | CREATE TABLE `lock_sample` (
  `id` bigint(20) NOT NULL,
  `val1` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> SELECT * FROM lock_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |   10 |
|  4 |   10 |
|  5 |    4 |
|  6 |   10 |
+----+------+
6 rows in set (0.00 sec)

mysql> SHOW CREATE TABLE another_sample;
+----------------+-----------------------------------------------------------------------------------------------------------------------------------------------------+
| Table          | Create Table                                                                                                                                        |
+----------------+-----------------------------------------------------------------------------------------------------------------------------------------------------+
| another_sample | CREATE TABLE `another_sample` (
  `id` bigint(20) NOT NULL,
  `val1` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 |
+----------------+-----------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> SELECT * FROM another_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
+----+------+
3 rows in set (0.00 sec)
```

2 つのトランザクション TA, TB で以下の操作を行います。

```
TA> begin;

# TB がレコード更新
TB> begin;
Query OK, 0 rows affected (0.00 sec)

TB> UPDATE lock_sample SET val1 = 3 WHERE id = 2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

TB> commit;
Query OK, 0 rows affected (0.01 sec)

# TA がトランザクション初 read。
# スナップショットはトランザクション開始時ではなくこのタイミングに基づく。
# なので TB で commit したものが読める。
TA> SELECT * FROM lock_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    3 |
|  3 |   10 |
|  4 |   10 |
|  5 |    4 |
|  6 |   10 |
+----+------+
6 rows in set (0.00 sec)

# 次の TB による変更は TA からは読まれない。
TB> UPDATE lock_sample SET val1 = 6 WHERE id = 2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

TB> UPDATE another_sample SET val1 = 10 WHERE id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

TB> commit;
Query OK, 0 rows affected (0.01 sec)

TA> SELECT * FROM lock_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    3 |
|  3 |   10 |
|  4 |   10 |
|  5 |    4 |
|  6 |   10 |
+----+------+
6 rows in set (0.00 sec)

TA> SELECT * FROM another_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
+----+------+
3 rows in set (0.00 sec)

# ただし locking read するとその値が読まれる
TA> SELECT * FROM lock_sample WHERE id = 2 FOR UPDATE;
+----+------+
| id | val1 |
+----+------+
|  2 |    6 |
+----+------+
1 row in set (0.00 sec)
```

<div id="shared-exclusive-lock" />

### shared lock と exclusive lock

- shared lock は read 用、exclusive lock は write 用と思えばよい
- shared lock は `SELECT ... LOCK IN SHARE MODE` で取得できる
- exclusive lock は `SELECT ... FOR UPDATE` で習得できる
- shared lock 同士は conflict しない
- それ以外の組み合わせは conflict する

参考:

- [MySQL 5.7 Reference Manual :: InnoDB Locking][5]

<div id="innodb-locking" />

### InnoDB Locking

- ロックには粒度が色々ある (e.g. レコード、テーブル)
- ざっくりとはここで挙げる種類と上の shared or exlusive の組み合わせで普段扱うロックを捉えられるはず
  - 有り得ない組み合わせとかフラグみたいな概念もありそうなのであくまでざっくりと

#### intention lock

- あるテーブルのデータにロックを取りに行く前にそのテーブル自体に対して取るロック
- IS (intention shared lock), IX (intention exclusive lock) と略される
- `LOCK TABLE ... WRITE` のようなテーブル自体への明示的な lock 以外に対して intention lock がブロックすることはない
  - と [リファレンス][4] は言っているように読めるけど [下で](#conflicted-example) わざとロック競合したとき intention lock も取得待ちになっているように見える
  - intention lock 同士は conflict しない
- あまり話も聞かないしこいつのせいで何かのトラブルに巻き込まれるということは少なそう

#### record lock

- index record へのロック
- record をロックするという場合、実際はインデックス上のレコードをロックしている
  - インデックスが定義されていないテーブルでも内部で作成したインデックスを使用する

record lock の場合、`SHOW ENGINE INNODB STATUS` では以下のように表示されます (`SHOW ENGINE INNODB STATUS` でのロック確認については [下記参照](#show-engine-innodb-status))。
`RECORD LOCKS` や `locks rec but not gap` が書かれている行から、ここでは lock\_sample テーブルの PRIMARY インデックス 上には record exlusive lock を取得していることがわかります。

```
mysql> SELECT * FROM lock_sample WHERE id = 2 FOR UPDATE;
mysql> SHOW ENGINE INNODB STATUS \G;
...
------------
TRANSACTIONS
------------
Trx id counter 2329
Purge done for trx's n:o < 2324 undo n:o < 0 state: running but idle
History list length 4
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 0, not started
MySQL thread id 4, OS thread handle 0x7f9625d74700, query id 50 localhost root
---TRANSACTION 2328, ACTIVE 8 sec
2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 3, OS thread handle 0x7f9625db5700, query id 62 localhost root init
SHOW ENGINE INNODB STATUS
TABLE LOCK table `sample`.`lock_sample` trx id 2328 lock mode IX
RECORD LOCKS space id 6 page no 3 n bits 72 index `PRIMARY` of table `sample`.`lock_sample` trx id 2328 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex 870000013a0110; asc     :  ;;
 3: len 4; hex 80000002; asc     ;;
 ...
```

参考:

- [MySQL のロックについて][6]
- [MySQL 5.7 Reference Manual :: 14.5.3 Locks Set by Different SQL Statements in InnoDB][7]

#### gap lock

- index records 間のスペースに対するロック
- ファントムリードの防止
  - なので (MySQL の) `REPEATABLE READ` では必要だが `READ COMMITED` では発生しない

`SHOW ENGINE INNODB STATUS` では `RECORD LOCKS ... locks gap before rec` と表示されます。

```
mysql> SELECT * FROM lock_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    3 |
|  5 |    4 |
|  8 |    8 |
+----+------+
6 rows in set (0.00 sec)

mysql> SELECT * FROM lock_sample WHERE id = 6 FOR UPDATE;
Empty set (0.00 sec)

mysql> SHOW ENGINE INNODB STATUS \G;
...
------------
TRANSACTIONS
------------
Trx id counter 2331
Purge done for trx's n:o < 2324 undo n:o < 0 state: running but idle
History list length 4
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 0, not started
MySQL thread id 4, OS thread handle 0x7f9625d74700, query id 50 localhost root
---TRANSACTION 2330, ACTIVE 36 sec
2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 3, OS thread handle 0x7f9625db5700, query id 70 localhost root init
SHOW ENGINE INNODB STATUS
Trx read view will not see trx with id >= 2331, sees < 2331
TABLE LOCK table `sample`.`lock_sample` trx id 2330 lock mode IX
RECORD LOCKS space id 6 page no 3 n bits 80 index `PRIMARY` of table `sample`.`lock_sample` trx id 2330 lock_mode X locks gap before rec
Record lock, heap no 7 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000008; asc         ;;
 1: len 6; hex 000000000919; asc       ;;
 2: len 7; hex 94000001420110; asc     B  ;;
 3: len 4; hex 80000008; asc     ;;
 ...
```

#### next-key lock

- record lock と gap lock の組み合わせ

`SHOW ENGINE INNODB STATUS` 上の表示は以下の感じです。`RECORD LOCKS` の行で `lock_mode X` で終わっているのが record lock, gap lock との違いになります。

```
mysql> SELECT * FROM lock_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    3 |
|  5 |    4 |
|  8 |    8 |
+----+------+
6 rows in set (0.00 sec)

mysql> SELECT * FROM lock_sample WHERE id BETWEEN 6 AND 7 FOR UPDATE;
Empty set (0.00 sec)

mysql> SHOW ENGINE INNODB STATUS
...
------------
TRANSACTIONS
------------
Trx id counter 2332
Purge done for trx's n:o < 2324 undo n:o < 0 state: running but idle
History list length 4
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 0, not started
MySQL thread id 4, OS thread handle 0x7f9625d74700, query id 50 localhost root
---TRANSACTION 2331, ACTIVE 24 sec
2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 3, OS thread handle 0x7f9625db5700, query id 74 localhost root init
SHOW ENGINE INNODB STATUS
Trx read view will not see trx with id >= 2332, sees < 2332
TABLE LOCK table `sample`.`lock_sample` trx id 2331 lock mode IX
RECORD LOCKS space id 6 page no 3 n bits 80 index `PRIMARY` of table `sample`.`lock_sample` trx id 2331 lock_mode X
Record lock, heap no 7 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000008; asc         ;;
 1: len 6; hex 000000000919; asc       ;;
 2: len 7; hex 94000001420110; asc     B  ;;
 3: len 4; hex 80000008; asc     ;;
 ...
```

#### insert intention lock

- insert 時に取る gap lock
- insert であるというフラグがたてられる
- お互いに競合しなければ同じ範囲の gap lock でも構わず insert が行える
- こいつに関しては exclusive しかありえなそう

<div id="dml-exclusive-lock" />

### DML 文は暗黙的に exclusive lock を取る

```
TA> UPDATE lock_sample SET val1 = 10 WHERE id = 2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

TA> SHOW ENGINE INNODB STATUS \G;
...
------------
TRANSACTIONS
------------
Trx id counter 1819
Purge done for trx's n:o < 1812 undo n:o < 0 state: running but idle
History list length 4
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421169626372856, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 1818, ACTIVE 7 sec
2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 2, OS thread handle 139694349264640, query id 55 localhost root
TABLE LOCK table `sample`.`lock_sample` trx id 1818 lock mode IX
RECORD LOCKS space id 24 page no 3 n bits 72 index PRIMARY of table `sample`.`lock_sample` trx id 1818 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 00000000071a; asc       ;;
 2: len 7; hex 36000001800110; asc 6      ;;
 3: len 4; hex 8000000a; asc     ;;
 ...
```

### 検索に使用された行がロックの対象になる

例えば以下のようにインデックスを設定していない列を条件に指定すると、検索はテーブル全体を対象にしないといけないために各レコードと supremum に next-key lock を取得するようです。ここで出てくる supremum とは MySQL が内部的に持つ上限値を表すレコードです。このため他トランザクションからはこのテーブルに update, insert が一切行えない状況になります。

```
mysql> SHOW CREATE TABLE lock_sample \G;
*************************** 1. row ***************************
       Table: lock_sample
Create Table: CREATE TABLE `lock_sample` (
  `id` bigint(20) NOT NULL,
  `val1` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

mysql> SELECT * FROM lock_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    3 |
|  5 |    4 |
|  8 |    8 |
+----+------+
6 rows in set (0.00 sec)

mysql> SELECT * FROM lock_sample WHERE val1 = 2 FOR UPDATE;
+----+------+
| id | val1 |
+----+------+
|  2 |    2 |
+----+------+
1 row in set (0.01 sec)

mysql> SHOW ENGINE INNODB STATUS \G;
...
------------
TRANSACTIONS
------------
Trx id counter 2333
Purge done for trx's n:o < 2324 undo n:o < 0 state: running but idle
History list length 4
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 0, not started
MySQL thread id 4, OS thread handle 0x7f9625d74700, query id 76 localhost root
---TRANSACTION 2332, ACTIVE 19 sec
2 lock struct(s), heap size 360, 7 row lock(s)
MySQL thread id 3, OS thread handle 0x7f9625db5700, query id 79 localhost root init
SHOW ENGINE INNODB STATUS
Trx read view will not see trx with id >= 2333, sees < 2333
TABLE LOCK table `sample`.`lock_sample` trx id 2332 lock mode IX
RECORD LOCKS space id 6 page no 3 n bits 80 index `PRIMARY` of table `sample`.`lock_sample` trx id 2332 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 6; hex 000000000906; asc       ;;
 2: len 7; hex 86000001390110; asc     9  ;;
 3: len 4; hex 80000001; asc     ;;

Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex 870000013a0110; asc     :  ;;
 3: len 4; hex 80000002; asc     ;;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000003; asc         ;;
 1: len 6; hex 00000000090c; asc       ;;
 2: len 7; hex 8a0000013d0110; asc     =  ;;
 3: len 4; hex 80000003; asc     ;;

Record lock, heap no 5 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000004; asc         ;;
 1: len 6; hex 00000000090d; asc       ;;
 2: len 7; hex 8b0000013e0110; asc     >  ;;
 3: len 4; hex 80000003; asc     ;;

Record lock, heap no 6 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000005; asc         ;;
 1: len 6; hex 00000000090e; asc       ;;
 2: len 7; hex 8c0000013f0110; asc     ?  ;;
 3: len 4; hex 80000004; asc     ;;

Record lock, heap no 7 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000008; asc         ;;
 1: len 6; hex 000000000919; asc       ;;
 2: len 7; hex 94000001420110; asc     B  ;;
 3: len 4; hex 80000008; asc     ;;
...
```

このあたりについては下記の資料が図とともに解説されていてわかりやすいと思いました。

- [MySQL のロックについて][6]

<div id="show-engine-innodb-status" />

### 取得ロックの確認方法

各トランザクションが現在どのようなロックを取得しているか、またどのロックを取得待ちしているかといった情報は `SET GLOBAL INNODB_STATUS_OUTPUT_LOCKS=ON` からの `SHOW ENGINE INNODB STATUS` で確認できます。

- [なぜあなたは SHOW ENGINE INNODB STATUS を読まないのか][3]

```
mysql> SET GLOBAL innodb_status_output_locks=ON;
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW VARIABLES LIKE '%innodb_status_output_locks%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| innodb_status_output_locks | ON    |
+----------------------------+-------+
```

適当にロック競合を起こすためのテーブルを用意します。

```
mysql> SHOW CREATE TABLE lock_sample \G;
*************************** 1. row ***************************
       Table: lock_sample
Create Table: CREATE TABLE `lock_sample` (
  `id` bigint(20) NOT NULL,
  `val1` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
1 row in set (0.00 sec)

mysql> SELECT * FROM lock_sample;
+----+------+
| id | val1 |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    3 |
|  5 |    4 |
+----+------+
5 rows in set (0.00 sec)
```

2 つの異なるトランザクション TA, TB で以下の操作を行うことでロック競合を起こします。
まず TA で id=3 のレコードへの exlusive lock を取得します。

```
TA> SET autocommit = 0;
TA> SELECT * FROM lock_sample WHERE id = 3 FOR UPDATE;
```

このとき `SHOW ENGINE INNODB STATUS` の表示は以下のようになります。

```
mysql> SHOW ENGINE INNODB STATUS \G;
...

------------
TRANSACTIONS
------------
Trx id counter 2325
Purge done for trx's n:o < 2324 undo n:o < 0 state: running but idle
History list length 4
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 2319, not started
MySQL thread id 2, OS thread handle 0x7f9625d74700, query id 33 localhost root
---TRANSACTION 2324, ACTIVE 7 sec
2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 1, OS thread handle 0x7f9625db5700, query id 35 localhost root init
SHOW ENGINE INNODB STATUS
TABLE LOCK table `sample`.`lock_sample` trx id 2324 lock mode IX
RECORD LOCKS space id 6 page no 3 n bits 72 index `PRIMARY` of table `sample`.`lock_sample` trx id 2324 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex 870000013a0110; asc     :  ;;
 3: len 4; hex 80000002; asc     ;;

...
```

全てを理解するのは難しいのですが、TA (trxid=2324) が

- `lock_sample` テーブルの IX (intention exclusive lock) を取得
- `lock_sample` テーブルの プライマリインデックス に対する record exclusive lock を取得

していることがわかります。

そして TB で同じ id=2 のレコードに対するロック取得を試みるとロック競合します。

```
TB> SET autocommit = 0;
TB> SELECT * FROM lock_sample WHERE id = 3 FOR UPDATE;
(blocked and timeout)
```

<div id="conflicted-example" />

ブロック中に取得した `SHOW ENGINE INNODB STATUS` は以下のようになりました。

```
------------
TRANSACTIONS
------------
Trx id counter 2326
Purge done for trx's n:o < 2324 undo n:o < 0 state: running but idle
History list length 4
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 2325, ACTIVE 1 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 2, OS thread handle 0x7f9625d74700, query id 38 localhost root statistics
SELECT * FROM lock_sample WHERE id = 2 FOR UPDATE
------- TRX HAS BEEN WAITING 1 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 6 page no 3 n bits 72 index `PRIMARY` of table `sample`.`lock_sample` trx id 2325 lock_mode X locks rec but not gap waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex 870000013a0110; asc     :  ;;
 3: len 4; hex 80000002; asc     ;;

------------------
TABLE LOCK table `sample`.`lock_sample` trx id 2325 lock mode IX
RECORD LOCKS space id 6 page no 3 n bits 72 index `PRIMARY` of table `sample`.`lock_sample` trx id 2325 lock_mode X locks rec but not gap waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex 870000013a0110; asc     :  ;;
 3: len 4; hex 80000002; asc     ;;

---TRANSACTION 2324, ACTIVE 618 sec
2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 1, OS thread handle 0x7f9625db5700, query id 39 localhost root init
SHOW ENGINE INNODB STATUS
TABLE LOCK table `sample`.`lock_sample` trx id 2324 lock mode IX
RECORD LOCKS space id 6 page no 3 n bits 72 index `PRIMARY` of table `sample`.`lock_sample` trx id 2324 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex 870000013a0110; asc     :  ;;
 3: len 4; hex 80000002; asc     ;;
```

TB の trxid は 2325 です。出力を見ると

- `lock_sample` テーブルへの intention exclusive lock 取得待ち
- `lock_sample` テーブルのプライマリインデックスに対する record exclusive lock 取得待ち

の 2 つが新たに出現しているがわかります。

このように `SHOW INNODB ENGINE STATUS` で各トランザクションのロックの挙動を把握することができます。

ちなみに `innodb_lock_monitor` を create して確認するという方法を紹介している場合もあるみたいですが、5.6 で非推奨、5.7 系以降消えた機能なので注意です。

- [MySQL 5.6 Reference Manula :: 14.17.2 Enabling InnoDB Monitors][4]

5.7, 8,0 系でも同様の手順で確認できますが、下の記事によると別途ロックに関する情報を参照するためのビューが提供されているようです。ちょこちょこビューの名前が変わったりしているのが混乱しますが、見易さを重視するならこちらをメインに使用するのもありかもしれません。

- [MySQL 5.7 と 8.0 でロック状態を確認する][2]

<div id="other-posts" />

### 各記事で見る (MySQL) ロックのハマりどころ

- [InnoDB の REPEATABLE READ における Locking Read についての注意点][8]
  - ロストアップデート
  - MySQL に限った話ではないはず
- [世界の何処かで MySQL (InnoDB) の REPEATABLE READ に嵌まる人を1人でも減らすために][9]
  - スナップショットを確立するタイミングに関して
- [挿入と参照ロックに疲れ果てた俺達は][10]
  - なかったら INSERT したいし、あるならロック取りたいよねという話
  - insert 時の gap lock 同士は範囲が同じでも一意制約等に違反しなければブロックされない
- [MySQL の外部キーとデッドロック][11]
  - 外部キー制約とデッドロック

### 参考

- [MySQL のロックについて][6]
  - 各クエリについてどのようにロックが取られるかを示す図がかなり役立ちました
- [MySQL の InnoDB のロック挙動調査][12]
  - ロック周りの挙動を幅広く理解するために

[1]: https://spring-mt.hatenablog.com/entry/2016/02/02/000145
[2]: https://qiita.com/hmatsu47/items/607d176e885f098262e8
[3]: https://soudai.hatenablog.com/entry/2017/12/20/030013
[4]: https://dev.mysql.com/doc/refman/5.7/en/innodb-enabling-monitors.html
[5]: https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html
[6]: https://dbstudy.info/files/20140907/mysql_lock_r2.pdf
[7]: https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html
[8]: http://nippondanji.blogspot.jp/2013/12/innodbrepeatable-readlocking-read.html
[9]: http://techblog.kayac.com/repeatable_read.html
[10]: https://ichirin2501.hatenablog.com/entry/2015/08/23/191500
[11]: http://www.tree-tips.com/mysql/deadlock/foreignkey/
[12]: https://github.com/ichirin2501/doc/blob/master/innodb.md
[13]: https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html
[14]: https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-reads.html
[15]: 
