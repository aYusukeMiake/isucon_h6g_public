# private-isu-training

## 概要

- [private-isu](https://github.com/catatsuy/private-isu)を用いて環境構築からインデックスを貼るまで

## 注意事項

- AWSを利用するため、料金が請求される方法である
- EC2インスタンスは適切に停止/削除すること
  - 一旦作業を終了する場合は停止すること(IPアドレスが起動のたびに変わるので、そこは注意すること)
  - もう使わない場合は削除すること

## 前提

- AWSのアカウントを持っていること
- EC2インスタンスの作成/SSH接続/削除に関する基本的な知識があること

## 練習時の環境構築メモ

[AMI]((https://github.com/catatsuy/private-isu#ami))を用いて環境構築をした。

- AWSのセキュリティグループとキーペアは事前に作成しておいた
  - セキュリティグループ
    - インバウンドルールを自身の手元のIPアドレスを許可するようにしておく
    - アウトバウンドルールは全て許可するようにしておく
    - インスタンスを作成後に、各EC2インスタンス同士のアクセスも許可するようにしておく
    - 面倒くさければ、全てのIPアドレスを許可するようにしておく
  - キーペア
    - 事前に作成し、pemファイルをダウンロードしておく
    - 後でSSH接続する際に使用する
- AWSマネジメントコンソールを開き、上記のAMI(x86)を利用してEC2インスタンスを2個作成した
  - 用途は「ベンチマーク実行用」と「アプリケーションサーバ+DBサーバー用」
    - アプリケーションサーバとDBサーバを分離する工夫をする場合は、追加でインスタンスを作成する必要
  - 終了後はインスタンスを削除すること
  - コストを確認の上、インスタンスを選択すること
    - 初めは`t2.micro`(1時間約$0.01)で試した
      - ただし、動作確認レベル(EC2の立ち上げ、接続確認、ベンチマークの初回実行)でしか使えない
      - ベンチマークを実行し続けるとフリーズするので、本格的にやる場合は`c5.large`(1時間約$0.1)
    - 推奨されているのは`c7a.large`(1時間約$0.12)

### SSH接続

~/.ssh/configに下記の設定を追記した

```bash
Host 適当な名前(例: private-isu-bench)
  HostName インスタンスのパブリックIPv4 DNS(ec2-111-222-3-4.ap-northeast-1.compute.amazonaws.com)
  IdentityFile ~/.ssh/key_name.pem
  User ubuntu(マニュアルに書いてある)
```

vscodeにRemote-SSHをインストールし、上記の設定のHostにSSH接続した。

参考: [マニュアル](https://github.com/catatsuy/private-isu/blob/master/manual.md)

### アプリの起動確認

http://[インスタンスのパブリックIP]にアクセスし、アプリが起動していることを確認した

### ベンチマークの実行

[private-isuのマニュアル](https://github.com/catatsuy/private-isu/blob/17a5955243cb638cfaf7b901a4ad8ee74402576d/README.md?plain=1#L64-L69)に従って、ベンチマークを実行した。

- まずは競技者用インスタンス上でのベンチマーカーを実行(アプリケーションが動いているかの確認)
- それが上手くいったら、ベンチマーカーをベンチマーク用インスタンス上で実行(インスタンス同士の接続確認)

### ログの準備

詳細は`isucon_share/docs`の「ログの取り方」を参照

- dstat
- Nginxのアクセスログとalp
- MySQLのスロークエリログ
- estackprof

## パフォーマンスチューニング

### 初期状態

ベンチマークの実行結果は以下の通り

```shell
{"pass":true,"score":329,"success":410,"fail":8,"messages":["リクエストがタイムアウトしました (GET /favicon.ico)","リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}
```

ログは以下の通り

- htopで確認すると、MySQL関連のCPU使用率が90%以上になっていた。
- alpのログは下記の通り

  ```shell
  +-------+-----+------+-----+-----+-----+--------+----------------------+-------+--------+----------+-------+-------+-------+--------+--------+-----------+-------------+---------------+------------+
  | COUNT | 1XX | 2XX  | 3XX | 4XX | 5XX | METHOD |         URI          |  MIN  |  MAX   |   SUM    |  AVG  |  P90  |  P95  |  P99   | STDDEV | MIN(BODY) |  MAX(BODY)  |   SUM(BODY)   | AVG(BODY)  |
  +-------+-----+------+-----+-----+-----+--------+----------------------+-------+--------+----------+-------+-------+-------+--------+--------+-----------+-------------+---------------+------------+
  | 1055  | 0   | 1028 | 0   | 27  | 0   | GET    | /image/[a-zA-Z0-9]+  | 0.018 | 8.170  | 1505.287 | 1.427 | 3.324 | 4.064 | 6.381  | 1.483  | 0.000     | 1056749.000 | 205795325.000 | 195066.659 |
  | 223   | 0   | 188  | 0   | 35  | 0   | GET    | /                    | 0.508 | 9.978  | 741.479  | 3.325 | 6.452 | 7.806 | 9.473  | 2.010  | 0.000     | 5049.000    | 910940.000    | 4084.933   |
  | 176   | 0   | 0    | 176 | 0   | 0   | POST   | /login               | 0.001 | 8.077  | 391.262  | 2.223 | 6.222 | 6.893 | 8.067  | 2.445  | 0.000     | 0.000       | 0.000         | 0.000      |
  | 82    | 0   | 78   | 0   | 4   | 0   | GET    | /favicon.ico         | 0.024 | 10.001 | 306.151  | 3.734 | 8.028 | 9.527 | 10.001 | 3.068  | 43.000    | 43.000      | 3354.000      | 40.902     |
  ```

- MySQLのスロークエリログは以下の通り

  ```shell
  Count: 63  Time=0.05s (3s)  Lock=0.00s (0s)  Rows=7.3 (460), isuconp[isuconp]@localhost
    SELECT * FROM `comments` WHERE `post_id` = N ORDER BY `created_at` DESC

  Count: 5639  Time=0.05s (290s)  Lock=0.00s (0s)  Rows=2.8 (15602), isuconp[isuconp]@localhost
    SELECT * FROM `comments` WHERE `post_id` = N ORDER BY `created_at` DESC LIMIT N

  Count: 10  Time=0.02s (0s)  Lock=0.00s (0s)  Rows=9986.4 (99864), isuconp[isuconp]@localhost
    SELECT `id`, `user_id`, `body`, `mime`, `created_at` FROM `posts` WHERE `created_at` <= 'S' ORDER BY `created_at` DESC

  Count: 1  Time=0.02s (0s)  Lock=0.00s (0s)  Rows=1.0 (1), isuconp[isuconp]@localhost
    SELECT COUNT(*) AS count FROM `comments` WHERE `post_id` IN (N,N,N,N,N,N,N,N,N,N,N,N,N,N,N,N,N,N,N,N)
  ```

あきらかにボトルネックがMySQLにあることがわかる。また、明確に遅いクエリがあることも分かる。また、画像のレスポンスが遅いこともわかる。

### DBの情報を確認

ひとまず現状のDBの情報を確認する。やり方は`isucon_share/docs/ISUCONにおけるDB関連の調査方法.md`に記載している。

```shell
# DBに接続
mysql -u${DB_ROOT_USER} -p${DB_ROOT_PASSWORD} ${DB_NAME}

# 以降、mysql> でのコマンドを実行
# データベース一覧
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| isuconp            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

# データベースを選択
use isuconp;

# テーブル一覧
show tables;
+-------------------+
| Tables_in_isuconp |
+-------------------+
| comments          |
| posts             |
| users             |
+-------------------+

# テーブルの中身を確認(先頭10個)
SELECT * FROM comments LIMIT 10;
+----+---------+---------+-----------------------------------------------------------------------------+---------------------+
| id | post_id | user_id | comment                                                                     | created_at          |
+----+---------+---------+-----------------------------------------------------------------------------+---------------------+
|  1 |    5162 |     533 | ○Ｏo。.（T￢T)/~~~オヤスミナサイ                                            | 2016-01-03 09:00:01 |
|  2 |    9722 |     669 | ♪ﾊﾛo(･x･o) !!o(･x･)/ﾊﾛ!! (o･x･)oﾊﾛ♪                                         | 2016-01-03 09:00:02 |
|  3 |    9627 |     806 | (｡-_-)ﾉ☆･ﾟ::ﾟﾖﾛｼｸ♪                                                          | 2016-01-03 09:00:03 |
|  4 |    5258 |     424 | (´∇｀)ﾉω"ﾀﾏｷﾝﾌﾞﾗﾌﾞﾗﾊﾞｲﾊﾞｲ                                                   | 2016-01-03 09:00:04 |
|  5 |    1073 |     875 | (ﾒﾟ皿ﾟ)ﾉ"▼▼ｻﾝｸﾞﾗｽﾌﾘﾌﾘﾊﾞｲﾊﾞｲ                                                 | 2016-01-03 09:00:05 |
|  6 |    6484 |     220 | こんにち▼･。･▼｣｣｣｣ーﾜﾝﾜﾝ!!                                                  | 2016-01-03 09:00:06 |
|  7 |    2557 |     152 | (´∇｀)ﾉω"ﾀﾏｷﾝﾌﾞﾗﾌﾞﾗﾊﾞｲﾊﾞｲ                                                   | 2016-01-03 09:00:07 |
|  8 |    6741 |     351 | (｡･~･｡)ﾉ▽" ｵｼﾒﾌﾘﾌﾘﾊﾞﾌﾞﾊﾞﾌﾞ♪                                                 | 2016-01-03 09:00:08 |
|  9 |    4906 |     247 | ★*♪｡☆*★*♪｡☆*★*♪｡☆*(^∇ﾟ*)ﾉ" ｵﾔｽﾐｨ♪                                           | 2016-01-03 09:00:09 |
| 10 |     238 |     192 | ★*♪｡☆*★*♪｡☆*★*♪｡☆*(^∇ﾟ*)ﾉ" ｵﾔｽﾐｨ♪                                           | 2016-01-03 09:00:10 |
+----+---------+---------+-----------------------------------------------------------------------------+---------------------+

# テーブル定義の確認
DESCRIBE comments;
+------------+-----------+------+-----+-------------------+-------------------+
| Field      | Type      | Null | Key | Default           | Extra             |
+------------+-----------+------+-----+-------------------+-------------------+
| id         | int       | NO   | PRI | NULL              | auto_increment    |
| post_id    | int       | NO   |     | NULL              |                   |
| user_id    | int       | NO   |     | NULL              |                   |
| comment    | text      | NO   |     | NULL              |                   |
| created_at | timestamp | NO   |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
+------------+-----------+------+-----+-------------------+-------------------+

# テーブルの行数を確認
SELECT table_name, engine, table_rows, avg_row_length, floor((data_length+index_length)/1024/1024) as allMB, floor((data_length)/1024/1024) as dMB, floor((index_length)/1024/1024) as iMB FROM information_schema.tables WHERE table_schema=database() ORDER BY (data_length+index_length) DESC;
+------------+--------+------------+----------------+-------+------+------+
| TABLE_NAME | ENGINE | TABLE_ROWS | AVG_ROW_LENGTH | allMB | dMB  | iMB  |
+------------+--------+------------+----------------+-------+------+------+
| posts      | InnoDB |       9672 |         136114 |  1255 | 1255 |    0 |
| comments   | InnoDB |      99666 |            110 |    10 |   10 |    0 |
| users      | InnoDB |       1000 |            212 |     0 |    0 |    0 |
+------------+--------+------------+----------------+-------+------+------+

SELECT
    table_name,
    index_name,
    non_unique,
    seq_in_index,
    column_name,
    collation,
    cardinality,
    sub_part,
    packed,
    nullable,
    index_type,
    comment,
    index_comment
FROM
    information_schema.statistics
WHERE
    table_schema = 'isuconp';
+------------+--------------+------------+--------------+--------------+-----------+-------------+----------+--------+----------+------------+---------+---------------+
| TABLE_NAME | INDEX_NAME   | NON_UNIQUE | SEQ_IN_INDEX | COLUMN_NAME  | COLLATION | CARDINALITY | SUB_PART | PACKED | NULLABLE | INDEX_TYPE | COMMENT | INDEX_COMMENT |
+------------+--------------+------------+--------------+--------------+-----------+-------------+----------+--------+----------+------------+---------+---------------+
| comments   | PRIMARY      |          0 |            1 | id           | A         |       99666 |     NULL |   NULL |          | BTREE      |         |               |
| posts      | PRIMARY      |          0 |            1 | id           | A         |        9672 |     NULL |   NULL |          | BTREE      |         |               |
| users      | account_name |          0 |            1 | account_name | A         |        1000 |     NULL |   NULL |          | BTREE      |         |               |
| users      | PRIMARY      |          0 |            1 | id           | A         |        1000 |     NULL |   NULL |          | BTREE      |         |               |
+------------+--------------+------------+--------------+--------------+-----------+-------------+----------+--------+----------+------------+---------+---------------+
```

### indexを貼る

comments関連のスロークエリがあるのでインデックスを貼る。

```sql
ALTER TABLE comments ADD INDEX idx_post_id (post_id);
```

スコアは以下のようになり、だいぶ上がる。

```shell
{"pass":true,"score":7952,"success":6787,"fail":0,"messages":[]}
```

また、MySQLのスロークエリログは以下のようになり、post_idに関するクエリがなくなっている。

```shell
Count: 1  Time=0.10s (0s)  Lock=0.00s (0s)  Rows=0.0 (0), isuconp[isuconp]@localhost
  DELETE FROM posts WHERE id > N

Count: 58  Time=0.02s (1s)  Lock=0.00s (0s)  Rows=9935.2 (576242), isuconp[isuconp]@localhost
  SELECT `id`, `user_id`, `body`, `mime`, `created_at` FROM `posts` WHERE `created_at` <= 'S' ORDER BY `created_at` DESC

Count: 125  Time=0.02s (2s)  Lock=0.00s (0s)  Rows=1.0 (125), isuconp[isuconp]@localhost
  SELECT COUNT(*) AS count FROM `comments` WHERE `user_id` = N
```
