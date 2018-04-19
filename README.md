# `MariaDB 10.2.13`によるパーティショニング（PARTITION BY）の動作検証

## 環境
|＃|環境|概要|
|:--|:--|:--|
|0|IPアドレス|192.168.33.50|
|1|Vagrant|2.0.3|
|2|CentOS|7(config.vm.box = "centos/7")|
|3|MariaDB|10.2.13|
|4|MariaDBのrootパスワード|Password0419|

## 事前準備

以下の記事を参考に、CentOS7の環境が構築済みの状態であること。
[Vagrant2.0.3を使ったCentOS7.4の環境構築](https://qiita.com/You_name_is_YU/items/1bdf36270e9f594d1c7e)

## やりたいこと

1つのDB内で、複数のユーザを管理するシステムは数多く存在すると思います。
やりたいこととして、他のユーザが登録したデータを相互に参照することはしないため、
他ユーザの登録件数に依存して検索速度が低下しないように、パーティショニングの機能を使って実現できるのか。ということを検証してみたいと思います。

## 環境準備

- CentOS7にインストール済みのMariaDBをアンインストールする。

```shell
> yum -y remove mariadb-*
```

- CentOS7用のMariaDB 10.2をyumでインストールするため、リポジトリを追加する。

```shell
> vim /etc/yum.repos.d/MariaDB.repo

[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.2/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

- MariaDB 10.2をインストールする。

```shell
> yum -y install MariaDB-server MariaDB-client MariaDB-devel
```

- インストールが正常に完了しているか確認するため、以下のコマンドを実行する。

```shell
> mysql --version

mysql  Ver 15.1 Distrib 10.2.13-MariaDB, for Linux (x86_64) using readline 5.1
```

- MariaDBをインストールすると、設定ファイルは`/etc/my.cnf.d/`配下に配置される仕組みとなっているが、
実際にはほとんど設定ファイルが入っていない。
そのため、`/usr/share/mysql/`配下にあるサンプルのcnfファイルをコピーして流用する。

```shell
> ls -l /usr/share/mysql/my-*.cnf

-rw-r--r--. 1 root root  4920 Feb 12 16:58 /usr/share/mysql/my-huge.cnf
-rw-r--r--. 1 root root 20441 Feb 12 16:58 /usr/share/mysql/my-innodb-heavy-4G.cnf
-rw-r--r--. 1 root root  4907 Feb 12 16:58 /usr/share/mysql/my-large.cnf
-rw-r--r--. 1 root root  4920 Feb 12 16:58 /usr/share/mysql/my-medium.cnf
-rw-r--r--. 1 root root  2846 Feb 12 16:58 /usr/share/mysql/my-small.cnf

> cp -p /usr/share/mysql/my-medium.cnf /etc/my.cnf.d/server.cnf
```

- server.cnfを修正して文字コードをutf8に指定し、データファイル格納先ディレクトリを指定する。

```shell
> vim /etc/my.cnf.d/server.cnf

[client]
#password       = your_password
port            = 3306
socket          = /var/lib/mysql/mysql.sock
default-character-set = utf8  ### 追加

# Here follows entries for some specific programs

# The MariaDB server
[mysqld]
port            = 3306
socket          = /var/lib/mysql/mysql.sock
skip-external-locking
key_buffer_size = 16M
max_allowed_packet = 1M
table_open_cache = 64
sort_buffer_size = 512K
net_buffer_length = 8K
read_buffer_size = 256K
read_rnd_buffer_size = 512K
myisam_sort_buffer_size = 8M
datadir=/var/lib/mysql          #### 追加
character-set-server = utf8     #### 追加
```

- MariaDBのサービス自動起動を設定し、サービスを起動する。

```shell
> systemctl enable mariadb.service
> systemctl start mariadb.service
> systemctl status mariadb.service
```

- 初期設定（rootのパスワードなどを設定）

```shell
> /usr/bin/mysql_secure_installation

Enter current password for root (enter for none):｛何も入力せずEnter｝
Set root password? [Y/n] Y
New password: Password0419
Re-enter new password: Password0419
Remove anonymous users?[Y/n] Y
Disallow root login remotely? [Y/n] Y    ← rootユーザのリモートログイン禁止
Remove test database and access to it? [Y/n] Y
Reload privilege tables now? [Y/n] Y    ← 権限テーブルの再読み込み
Thanks for using MariaDB!
```

- DBを作成する。

```shell
> mysql -u root -p

MariaDB [(none)]> CREATE DATABASE sample DEFAULT CHARACTER SET utf8;
```

- 新規ユーザーを追加し、全てのIPアドレスからアクセス可能にする。

```sql
MariaDB [(none)]> CREATE USER 'test'@'%' IDENTIFIED BY 'test';
MariaDB [(none)]> GRANT ALL ON *.* TO test;
```

- 作成したユーザーでログインし、テーブルを作成する。

```sql
    MariaDB [(none)]> exit
    # mysql -u test -p
    MariaDB [(none)]> connect sample
    MariaDB [(none)]> create table users(user_id int auto_increment primary key, user_name text not null, create_dt date not null);
    MariaDB [sample]>
    create table emotion_dic(id int auto_increment primary key
     , user_id int not null
     , entry_word text not null
     , reading text not null
     , part text not null
     , emotion_real_number double(20,10) not null);
    MariaDB [sample]> insert into users(user_name, create_dt) values ('A', now());
    Query OK, 1 row affected, 1 warning (0.00 sec)

    MariaDB [sample]> insert into users(user_name, create_dt) values ('B', now());
    Query OK, 1 row affected, 1 warning (0.01 sec)

    MariaDB [sample]> insert into users(user_name, create_dt) values ('C', now());
    Query OK, 1 row affected, 1 warning (0.00 sec)

    MariaDB [sample]> insert into users(user_name, create_dt) values ('D', now());
    Query OK, 1 row affected, 1 warning (0.00 sec)
```

- 大量データをロード（emotion_dicテーブルのデータ）  
emotion_dicテーブルのデータは下記データを利用している。
https://github.com/Yu-Yamaguchi/mariadb-partitioning/blob/master/emotion_dic_data.csv

``` sql
MariaDB [sample]> LOAD DATA LOCAL INFILE '/home/vagrant/emotion_dic_data.csv'
    -> INTO TABLE emotion_dic
    -> FIELDS
    ->     TERMINATED BY ','
    ->     OPTIONALLY ENCLOSED BY '"'
    ->     ESCAPED BY ''
    -> LINES
    ->     STARTING BY ''
    ->     TERMINATED BY '\r\n'
    -> (
    ->  user_id,
    ->  entry_word,
    ->  reading,
    ->  part,
    ->  emotion_real_number
    -> );
    Query OK, 668980 rows affected, 1 warning (3.22 sec)
    Records: 668980  Deleted: 0  Skipped: 0  Warnings: 1
```

- ここまでの状態ではパーティショニングを実現していないため、1つのパーティションとなっている。

``` sql
SELECT TABLE_SCHEMA,TABLE_NAME,PARTITION_NAME,PARTITION_ORDINAL_POSITION,TABLE_ROWS
 FROM INFORMATION_SCHEMA.PARTITIONS
 WHERE TABLE_NAME='emotion_dic';
+--------------+-------------+----------------+----------------------------+------------+
| TABLE_SCHEMA | TABLE_NAME  | PARTITION_NAME | PARTITION_ORDINAL_POSITION | TABLE_ROWS |
+--------------+-------------+----------------+----------------------------+------------+
| sample       | emotion_dic | NULL           |                       NULL |     667329 |
+--------------+-------------+----------------+----------------------------+------------+
1 row in set (0.03 sec)
```

## パーティショニング前の負荷試験

上述している環境準備が完了した時点で、emotion_dicテーブルには66万件のデータが登録済みとなっている。
この状態のデータに対してMySQLが提供している負荷試験用のコマンド「mysqlslap」を利用して、
SELECTの負荷試験を行います。

ここでは、以下のSELECT文を1クライアントから1000回実行している。

```sql
SELECT * FROM emotion_dic WHERE user_id = 3 AND entry_word LIKE '%負%';
```

```
# mysqlslap --no-defaults --user=test --password=test --concurrency=1 --iterations=1000 --engine=innodb --create-schema=sample --no-drop --delimiter=";" --query="SELECT * FROM emotion_dic WHERE user_id = 3 AND entry_word LIKE '%負%';"
Benchmark
        Running for engine innodb
        Average number of seconds to run all queries: 0.325 seconds
        Minimum number of seconds to run all queries: 0.296 seconds
        Maximum number of seconds to run all queries: 0.548 seconds
        Number of clients running queries: 1
        Average number of queries per client: 1
```

## パーティショニング後の負荷試験

まずはemotion_dicテーブルにパーティションの設定
と思ったら、パーティション設定するカラムはPKを含んでいないといけない。というようなことを言われる
※正確にはUNIQUE KEY制約のカラムを1つ以上指定する必要がある。
  UNIQUE KEY (col1, col2)
  UNIQUE KEY (col3)
  の2つが指定されているテーブルが存在する場合は、
  partition by list(col3)はOKだけど、
  partition by list(col1)はNG。

```sql
mysql -u test -p
MariaDB [(none)]> connect sample
MariaDB [sample]> alter table emotion_dic partition by list(user_id) (
    ->     partition p_user1 values in (1),
    ->     partition p_user2 values in (2),
    ->     partition p_user3 values in (3),
    ->     partition p_user4 values in (4),
    ->     partition p_user5 values in (5),
    ->     partition p_user6 values in (6)
    -> );
ERROR 1503 (HY000): A PRIMARY KEY must include all columns in the table's partitioning function
```

仕方ないので、emotiono_dicのPKである「id」の列を削除してみる。

```sql
MariaDB [sample]> ALTER TABLE emotion_dic DROP COLUMN id;
Query OK, 668980 rows affected (1.80 sec)
Records: 668980  Duplicates: 0  Warnings: 0
```

もう一度パーティションの設定

```sql
MariaDB [sample]>
alter table emotion_dic partition by list(user_id) (
 partition p_user0 values in (0) engine = InnoDB,
 partition p_user1 values in (1) engine = InnoDB,
 partition p_user2 values in (2) engine = InnoDB,
 partition p_user3 values in (3) engine = InnoDB,
 partition p_user4 values in (4) engine = InnoDB,
 partition p_user5 values in (5) engine = InnoDB,
 partition p_user6 values in (6) engine = InnoDB,
 partition p_user7 values in (7) engine = InnoDB
);
Query OK, 668980 rows affected (1.75 sec)
Records: 668980  Duplicates: 0  Warnings: 0
```

パーティション設定が完了しているか確認する。

```sql
MariaDB [sample]>
SELECT
TABLE_SCHEMA,TABLE_NAME,PARTITION_NAME,PARTITION_ORDINAL_POSITION,TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_NAME='emotion_dic';
+--------------+-------------+----------------+----------------------------+------------+
| TABLE_SCHEMA | TABLE_NAME  | PARTITION_NAME | PARTITION_ORDINAL_POSITION | TABLE_ROWS |
+--------------+-------------+----------------+----------------------------+------------+
| sample       | emotion_dic | p_user0        |                          1 |          0 |
| sample       | emotion_dic | p_user1        |                          2 |       4734 |
| sample       | emotion_dic | p_user2        |                          3 |     109840 |
| sample       | emotion_dic | p_user3        |                          4 |     110187 |
| sample       | emotion_dic | p_user4        |                          5 |     110830 |
| sample       | emotion_dic | p_user5        |                          6 |     109890 |
| sample       | emotion_dic | p_user6        |                          7 |      58739 |
| sample       | emotion_dic | p_user7        |                          8 |      58828 |
+--------------+-------------+----------------+----------------------------+------------+
8 rows in set (0.00 sec)
```

性能評価

```shell
# mysqlslap --no-defaults --user=test --password=test --concurrency=1 --iterations=1000 --engine=innodb --create-schema=sample --no-drop --delimiter=";" --query="SELECT * FROM emotion_dic WHERE user_id = 3 AND entry_word LIKE '%負%';"
Benchmark
        Running for engine innodb
        Average number of seconds to run all queries: 0.068 seconds
        Minimum number of seconds to run all queries: 0.063 seconds
        Maximum number of seconds to run all queries: 0.229 seconds
        Number of clients running queries: 1
        Average number of queries per client: 1
```

## パーティショニングの設定に応じた負荷試験　結果まとめ

以下の条件で行った負荷試験の結果をまとめて記載する。

### 負荷試験の条件

|＃|クライアント数（concurrency）|繰り返し数（iterations）|
|:--|:--|:--|
|1|1|1000|
|2|10|100|
|3|100|10|

１の実行結果

```
    -- パーティショニング前
    Benchmark
            Running for engine innodb
            Average number of seconds to run all queries: 0.397 seconds
            Minimum number of seconds to run all queries: 0.305 seconds
            Maximum number of seconds to run all queries: 0.708 seconds
            Number of clients running queries: 1
            Average number of queries per client: 1

    -- パーティショニング後
    Benchmark
        Running for engine innodb
        Average number of seconds to run all queries: 0.063 seconds
        Minimum number of seconds to run all queries: 0.059 seconds
        Maximum number of seconds to run all queries: 0.126 seconds
        Number of clients running queries: 1
        Average number of queries per client: 1
```

２の実行結果

```
    -- パーティショニング前
    Benchmark
        Running for engine innodb
        Average number of seconds to run all queries: 3.309 seconds
        Minimum number of seconds to run all queries: 3.095 seconds
        Maximum number of seconds to run all queries: 4.576 seconds
        Number of clients running queries: 10
        Average number of queries per client: 1

    -- パーティショニング後
    Benchmark
        Running for engine innodb
        Average number of seconds to run all queries: 0.682 seconds
        Minimum number of seconds to run all queries: 0.613 seconds
        Maximum number of seconds to run all queries: 1.050 seconds
        Number of clients running queries: 10
        Average number of queries per client: 1
```

３の実行結果

```
    -- パーティショニング前
    Benchmark
        Running for engine innodb
        Average number of seconds to run all queries: 33.190 seconds
        Minimum number of seconds to run all queries: 31.768 seconds
        Maximum number of seconds to run all queries: 35.018 seconds
        Number of clients running queries: 100
        Average number of queries per client: 1

    -- パーティショニング後
    Benchmark
        Running for engine innodb
        Average number of seconds to run all queries: 6.686 seconds
        Minimum number of seconds to run all queries: 6.257 seconds
        Maximum number of seconds to run all queries: 7.533 seconds
        Number of clients running queries: 100
        Average number of queries per client: 1
```

## まとめ

他のユーザの件数増加に伴う検索性能の低下を抑えるため、
パーティショニングを使うことで解消することは可能だが、実際にはサロゲートキーのような
auto_incrementのid項目を設けたいケースも多く発生するため、UNIQUE KEY制約が含まれるテーブルをパーティショニングする場合、
そのキーをパーティショニングキーとして含めなければいけない制約は非常に苦しいところ。
条件によって使える。といった印象か。
