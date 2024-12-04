# ISUCON13でのチューニングの流れ

## 環境構築

基本的には[private-isuで環境構築から最初のチューニングをするまで.md](./private-isuで環境構築から最初のチューニングをするまで.md)と同じ手順で行った。

以下、環境構築の情報を記載する。

- AMIは[こちら](https://github.com/matsuu/aws-isucon)を使用した
- 言語はRuby、ベンチマークとアプリケーションサーバーは同じEC2インスタンス上で実行した
- ログはNginxのアクセスログ(alp)、MySQLのスロークエリログ(pt-query-digest)、アプリケーションプロファイラ(estackprof)を使用した

## チューニングの流れ

### 初期状態

ベンチマークを実行して、ベンチマークやログの結果を確認した。

ベンチマークの結果は下記の通りだった。

```bash
スコア: 2485
```

htopを見ると、ベンチマークを除くと、MySQLのCPU利用率が支配的だった。そのため、pt-query-digestを使用してスロークエリを確認した。

```bash
# Profile
# Rank Query ID                       Response time Calls R/Call V/M   Ite
# ==== ============================== ============= ===== ====== ===== ===
#    1 0xF7144185D9A142A426A36DC55... 52.5397 29.6%   419 0.1254  0.01 SELECT livestream_tags
#    2 0x84B457C910C4A79FC9EBECB8B... 20.5223 11.6%   891 0.0230  0.02 SELECT icons
#    3 0x4ADE2DC90689F1C4891749AF5... 15.8311  8.9%  3485 0.0045  0.01 DELETE SELECT livecomments
#    4 0xF1B8EF06D6CA63B24BFF433E0... 14.5137  8.2%   213 0.0681  0.01 SELECT users livestreams livecomments
#    5 0xDB74D52D39A7090F224C4DEEA... 13.9470  7.9%   212 0.0658  0.01 SELECT users livestreams reactions
```

### スロークエリログを見ながらインデックスを貼っていく

一番上のスロークエリの具体的なクエリはクエリログから確認できるので、それに対してEXPLAINを実行し、それに関するインデックスを確認した。

```sql
mysql> EXPLAIN SELECT * FROM livestream_tags WHERE livestream_id = '7521';
+----+-------------+-----------------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table           | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+-----------------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | livestream_tags | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 11137 |    10.00 | Using where |
+----+-------------+-----------------+------------+------+---------------+------+---------+------+-------+----------+-------------+

mysql> SHOW INDEX FROM livestream_tags;
+-----------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table           | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| livestream_tags |          0 | PRIMARY  |            1 | id          | A         |       11137 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
```

`livestream_id`にインデックスが貼られていないことが分かったので、インデックスを貼る。今回は初期化スクリプト(`webapp/sql/initdb.d/10_schema.sql`)にインデックスを追加した。

```sql
CREATE TABLE `livestream_tags` (
  `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `livestream_id` BIGINT NOT NULL,
  `tag_id` BIGINT NOT NULL,
  INDEX `idx_livestream_id` (`livestream_id`)
) ENGINE=InnoDB CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```

初期化は[公式手順](https://github.com/isucon/isucon13/blob/c52b359fc6e733e1193ac8e9835bea23856566e7/docs/cautionary_note.md#isupipe-%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E3%81%AE%E3%82%B9%E3%82%AD%E3%83%BC%E3%83%9E%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)に従って行う。

```sql
-- DBにログイン
DROP DATABASE isupipe;
CREATE DATABASE isupipe;
```

```bash
# 初期化
cat webapp/sql/initdb.d/10_schema.sql | sudo mysql isupipe
```

スコアは3400点に上がった。

```bash
スコア: 3469
```

スロークエリのトップが変わったので、再度スロークエリを確認した。

```bash
# Profile
# Rank Query ID                       Response time Calls R/Call V/M   Ite
# ==== ============================== ============= ===== ====== ===== ===
#    1 0x84B457C910C4A79FC9EBECB8B... 11.3091 17.0%  1891 0.0060  0.01 SELECT icons
#    2 0x38BC86A45F31C6B1EE3246715... 10.6825 16.0%  1480 0.0072  0.00 SELECT themes
#    3 0x64CC8A4E8E4B390203375597C...  8.8201 13.2%    51 0.1729  0.01 SELECT ng_words
#    4 0x59F1B6DD8D9FEC059E55B3BFD...  6.2334  9.4%    87 0.0716  0.01 SELECT reservation_slots
#    5 0x22279D81D51006139E0C76405...  3.5973  5.4%  2591 0.0014  0.00 SELECT domains domainmetadata
```

一番上のスロークエリの具体的なクエリ例を確認し、それに対してEXPLAINを実行した。

```sql
mysql> EXPLAIN SELECT image FROM icons WHERE user_id = '1027';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | icons | NULL       | ALL  | NULL          | NULL | NULL    | NULL |  174 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

user_idに対して、インデックスが貼られていないため、こちらもインデックスを貼った。こちらはスコアがあまり変わらなかったが、インデックスが効くようになった。

```sql
mysql> EXPLAIN SELECT image FROM icons WHERE user_id = '1027';
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | icons | NULL       | ref  | idx_user_id   | idx_user_id | 8       | const |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
```

同様に下記のスロークエリも確認し、livecommentsにlivestream_idに対するインデックスを貼った。

```sql
mysql> EXPLAIN SELECT IFNULL(SUM(l2.tip), 0) FROM users u INNER JOIN livestreams l ON l.user_id = u.id INNER JOIN livecomments l2 ON l2.livestream_id = l.id
WHERE u.id = '6';
+----+-------------+-------+------------+--------+---------------------+---------+---------+--------------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys       | key     | key_len | ref                      | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------------+---------+---------+--------------------------+------+----------+-------------+
|  1 | SIMPLE      | u     | NULL       | const  | PRIMARY             | PRIMARY | 8       | const                    |    1 |   100.00 | Using index |
|  1 | SIMPLE      | l2    | NULL       | ALL    | NULL                | NULL    | NULL    | NULL                     | 1201 |   100.00 | NULL        |
|  1 | SIMPLE      | l     | NULL       | eq_ref | PRIMARY,idx_user_id | PRIMARY | 8       | isupipe.l2.livestream_id |    1 |     5.00 | Using where |
+----+-------------+-------+------------+--------+---------------------+---------+---------+--------------------------+------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)
```

しかし、これだと下記のようなインデックスが効かなかった。

```sql
mysql> EXPLAIN SELECT IFNULL(SUM(l2.tip), 0) FROM users u INNER JOIN livestreams l ON l.user_id = u.id INNER JOIN livecomments l2 ON l2.livestream_id = l.id
    -> WHERE u.id = '6';
+----+-------------+-------+------------+--------+-------------------+---------+---------+--------------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys     | key     | key_len | ref                      | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+-------------------+---------+---------+--------------------------+------+----------+-------------+
|  1 | SIMPLE      | u     | NULL       | const  | PRIMARY           | PRIMARY | 8       | const                    |    1 |   100.00 | Using index |
|  1 | SIMPLE      | l2    | NULL       | ALL    | idx_livestream_id | NULL    | NULL    | NULL                     | 1209 |   100.00 | NULL        |
|  1 | SIMPLE      | l     | NULL       | eq_ref | PRIMARY           | PRIMARY | 8       | isupipe.l2.livestream_id |    1 |    10.00 | Using where |
+----+-------------+-------+------------+--------+-------------------+---------+---------+--------------------------+------+----------+-------------+
```

とりあえrず、`FORCE INDEX`を使ってインデックスを強制的に使うようにした。

```sql
mysql> EXPLAIN SELECT IFNULL(SUM(l2.tip), 0) FROM users u INNER JOIN livestreams l ON l.user_id = u.id INNER JOIN livecomments l2 FORCE INDEX (idx_livestream_id) ON l2.livestream_id = l.id WHERE u.id = '6';
+----+-------------+-------+------------+-------+-------------------+-------------------+---------+--------------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys     | key               | key_len | ref          | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+-------------------+-------------------+---------+--------------+------+----------+-------------+
|  1 | SIMPLE      | u     | NULL       | const | PRIMARY           | PRIMARY           | 8       | const        |    1 |   100.00 | Using index |
|  1 | SIMPLE      | l     | NULL       | ALL   | PRIMARY           | NULL              | NULL    | NULL         | 7401 |    10.00 | Using where |
|  1 | SIMPLE      | l2    | NULL       | ref   | idx_livestream_id | idx_livestream_id | 8       | isupipe.l.id |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+-------+-------------------+-------------------+---------+--------------+------+----------+-------------+
```

しかし、これだとlivestreamsの方でインデックスが効かなくなった。こちらはlivestreamsのuser_idに対するインデックスを貼ることで、クエリを通してインデックスが使われるようになった。

```sql
mysql> EXPLAIN SELECT IFNULL(SUM(l2.tip), 0) FROM users u INNER JOIN livestreams l FORCE INDEX (idx_user_id) ON l.user_id = u.id INNER JOIN livecomments l2 FORCE INDEX (idx_livestream_id) ON l2.livestream_id = l.id WHERE u.id = '6';
+----+-------------+-------+------------+-------+-------------------+-------------------+---------+--------------+------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys     | key               | key_len | ref          | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------+-------------------+-------------------+---------+--------------+------+----------+--------------------------+
|  1 | SIMPLE      | u     | NULL       | const | PRIMARY           | PRIMARY           | 8       | const        |    1 |   100.00 | Using index              |
|  1 | SIMPLE      | l     | NULL       | ref   | idx_user_id       | idx_user_id       | 8       | const        |    8 |   100.00 | Using where; Using index |
|  1 | SIMPLE      | l2    | NULL       | ref   | idx_livestream_id | idx_livestream_id | 8       | isupipe.l.id |    1 |   100.00 | NULL                     |
+----+-------------+-------+------------+-------+-------------------+-------------------+---------+--------------+------+----------+--------------------------+
```

スロークエリから消えたが、スコアは3400点から3300点に少し下がった。ひとまず、スロークエリログを確認した。

```bash
# Profile
# Rank Query ID                       Response time Calls R/Call V/M   Ite
# ==== ============================== ============= ===== ====== ===== ===
#    1 0xDB74D52D39A7090F224C4DEEA... 24.4039 31.1%   427 0.0572  0.01 SELECT users livestreams reactions
#    2 0x38BC86A45F31C6B1EE3246715...  9.3098 11.8%  1180 0.0079  0.00 SELECT themes
#    3 0x64CC8A4E8E4B390203375597C...  6.3080  8.0%    33 0.1912  0.02 SELECT ng_words
#    4 0x59F1B6DD8D9FEC059E55B3BFD...  5.7216  7.3%    60 0.0954  0.01 SELECT reservation_slots
#    5 0x4ADE2DC90689F1C4891749AF5...  5.6502  7.2%  3460 0.0016  0.00 DELETE SELECT livecomments
```

トップのスロークエリが変わっていたので、具体的な原因となるクエリを確認した。

```sql
mysql> EXPLAIN SELECT COUNT(*) FROM users u
INNER JOIN livestreams l ON l.user_id = u.id
INNER JOIN reactions r ON r.livestream_id = l.id
WHERE u.id = '145';

+----+-------------+-------+------------+--------+---------------------+---------+---------+-------------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys       | key     | key_len | ref                     | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------------+---------+---------+-------------------------+------+----------+-------------+
|  1 | SIMPLE      | u     | NULL       | const  | PRIMARY             | PRIMARY | 8       | const                   |    1 |   100.00 | Using index |
|  1 | SIMPLE      | r     | NULL       | ALL    | NULL                | NULL    | NULL    | NULL                    | 1197 |   100.00 | NULL        |
|  1 | SIMPLE      | l     | NULL       | eq_ref | PRIMARY,idx_user_id | PRIMARY | 8       | isupipe.r.livestream_id |    1 |     5.00 | Using where |
+----+-------------+-------+------------+--------+---------------------+---------+---------+-------------------------+------+----------+-------------+
```

`reactions`の`livestream_id`に対するインデックスが貼られていないことが分かったので、インデックスを貼ってもう一度EXPLAINで確認した。

```sql
mysql> EXPLAIN SELECT COUNT(*) FROM users u
    -> INNER JOIN livestreams l ON l.user_id = u.id
    -> INNER JOIN reactions r ON r.livestream_id = l.id
    -> WHERE u.id = '145';
+----+-------------+-------+------------+-------+---------------------+-------------------+---------+--------------+------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys       | key               | key_len | ref          | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------+---------------------+-------------------+---------+--------------+------+----------+--------------------------+
|  1 | SIMPLE      | u     | NULL       | const | PRIMARY             | PRIMARY           | 8       | const        |    1 |   100.00 | Using index              |
|  1 | SIMPLE      | l     | NULL       | ref   | PRIMARY,idx_user_id | idx_user_id       | 8       | const        |    8 |   100.00 | Using where; Using index |
|  1 | SIMPLE      | r     | NULL       | ref   | idx_livestream_id   | idx_livestream_id | 8       | isupipe.l.id |    1 |   100.00 | Using index              |
+----+-------------+-------+------------+-------+---------------------+-------------------+---------+--------------+------+----------+--------------------------+
```

スコアは3300点から3700点に上がった。もう一度スロークエリログを確認した。

```bash
# Profile
# Rank Query ID                       Response time Calls R/Call V/M   Ite
# ==== ============================== ============= ===== ====== ===== ===
#    1 0x38BC86A45F31C6B1EE3246715... 11.2777 20.0%  1423 0.0079  0.00 SELECT themes
#    2 0x64CC8A4E8E4B390203375597C...  8.6589 15.3%    48 0.1804  0.02 SELECT ng_words
#    3 0x59F1B6DD8D9FEC059E55B3BFD...  5.8847 10.4%    83 0.0709  0.02 SELECT reservation_slots
#    4 0x84B457C910C4A79FC9EBECB8B...  4.4557  7.9%  1966 0.0023  0.01 SELECT icons
#    5 0x22279D81D51006139E0C76405...  3.6812  6.5%  2639 0.0014  0.00 SELECT domains domainmetadata
```

トップのスロークエリが変わったので、具体的な原因となるクエリを確認した。

```sql
mysql> EXPLAIN SELECT * FROM themes WHERE user_id = '1032';
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | themes | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1167 |    10.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

こちらもインデックスが貼られていなさそうだったのでインデックスを貼った。

```sql
mysql> EXPLAIN SELECT * FROM themes WHERE user_id = '1032';
+----+-------------+--------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
| id | select_type | table  | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+--------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | themes | NULL       | ref  | idx_user_id   | idx_user_id | 8       | const |    1 |   100.00 | Using index condition |
+----+-------------+--------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
```

スコアは3700点から4000点に上がった。もう一度スロークエリログのトップを確認し、それに対してEXPLAINで実行した。

```sql
mysql> EXPLAIN SELECT id, user_id, livestream_id, word FROM ng_words WHERE user_id = '1013' AND livestream_id = '7526';
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | ng_words | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 14484 |     1.00 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-------------+
```

上記のクエリログを見たうえで、下記のように複合インデックスを貼るようにスキーマを変えた。

```sql
CREATE TABLE `ng_words` (
  `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `user_id` BIGINT NOT NULL,
  `livestream_id` BIGINT NOT NULL,
  `word` VARCHAR(255) NOT NULL,
  `created_at` BIGINT NOT NULL,
  INDEX `idx_user_id_livestream_id` (`user_id`, `livestream_id`)
) ENGINE=InnoDB CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```

スロークエリログのトップからは当該クエリが消えた。

```sql
mysql> EXPLAIN SELECT id, user_id, livestream_id, word FROM ng_words WHERE user_id = '1013' AND livestream_id = '7526';
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------------+------+----------+-----------------------+
| id | select_type | table    | partitions | type | possible_keys             | key                       | key_len | ref         | rows | filtered | Extra                 |
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------------+------+----------+-----------------------+
|  1 | SIMPLE      | ng_words | NULL       | ref  | idx_user_id_livestream_id | idx_user_id_livestream_id | 16      | const,const |    1 |   100.00 | Using index condition |
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------------+------+----------+-----------------------+
```

再度スロークエリログを確認すると、下記のクエリが上の方にきていた。

```sql
DELETE FROM livecomments
WHERE
id = '890' AND
livestream_id = '7523' AND
(SELECT COUNT(*)
FROM
(SELECT 'フルアルバムを待ってるよ！' AS text) AS texts
INNER JOIN
(SELECT CONCAT('%', '霧降法式', '%') AS pattern) AS patterns
ON texts.text LIKE patterns.pattern) >= 1\G
```

こちらはDELETEのクエリなので、クエリログを見ても、DELETE対象が既に消えている。

```sql
-- pt-query-digestでDELETEをSELECTに変換して確認
EXPLAIN select * from  livecomments
WHERE
id = '890' AND
livestream_id = '7523' AND
(SELECT COUNT(*)
FROM
(SELECT 'フルアルバムを待ってるよ！' AS text) AS texts
INNER JOIN
(SELECT CONCAT('%', '霧降法式', '%')	AS pattern) AS patterns
ON texts.text LIKE patterns.pattern) >= 1;

+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                               |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------------------------------------------+
|  1 | DELETE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Impossible WHERE                                    |
|  2 | SUBQUERY    | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Impossible WHERE noticed after reading const tables |
|  4 | DERIVED     | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used                                      |
|  3 | DERIVED     | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------------------------------------------+
```

こういった際はクエリログではなく、実データを確認して、それに対するEXPLAINを実行する必要がある。実行すると下記のようになった。

```sql
-- 実データを確認
SELECT * From livecomments LIMIT 1;
+----+---------+---------------+--------------------------------------------------------------------------------------------------------------+-----+------------+
| id | user_id | livestream_id | comment                                                                                                      | tip | created_at |
+----+---------+---------------+--------------------------------------------------------------------------------------------------------------+-----+------------+
|  1 |     534 |          1532 | あっちの人、今めちゃくちゃ笑ってる場面で止まってるよ。何があったんだろ？                                     |   0 | 1732459603 |
+----+---------+---------------+--------------------------------------------------------------------------------------------------------------+-----+------------+

-- EXPLAINを実行
EXPLAIN SELECT * FROM livecomments WHERE     id = '1' AND     livestream_id = '1532' AND     (SELECT COUNT(*)     FROM         (SELECT comment AS text FROM
 livecomments WHERE id = '1' AND livestream_id = '1532') AS texts     INNER JOIN         (SELECT CONCAT('%', '笑ってる', '%') AS pattern) AS patterns     ON texts
.text LIKE patterns.pattern) >= 1;
+----+-------------+--------------+------------+--------+---------------------------+---------+---------+-------+------+----------+----------------+
| id | select_type | table        | partitions | type   | possible_keys             | key     | key_len | ref   | rows | filtered | Extra          |
+----+-------------+--------------+------------+--------+---------------------------+---------+---------+-------+------+----------+----------------+
|  1 | PRIMARY     | livecomments | NULL       | const  | PRIMARY,idx_livestream_id | PRIMARY | 8       | const |    1 |   100.00 | NULL           |
|  2 | SUBQUERY    | <derived4>   | NULL       | system | NULL                      | NULL    | NULL    | NULL  |    1 |   100.00 | NULL           |
|  2 | SUBQUERY    | livecomments | NULL       | const  | PRIMARY,idx_livestream_id | PRIMARY | 8       | const |    1 |   100.00 | NULL           |
|  4 | DERIVED     | NULL         | NULL       | NULL   | NULL                      | NULL    | NULL    | NULL  | NULL |     NULL | No tables used |
+----+-------------+--------------+------------+--------+---------------------------+---------+---------+-------+------+----------+----------------+
```

こちらはサブクエリが複雑なので、今後のチューニングがやりづらい。こちらは下記のようなクエリになるようにアプリケーションのコードを修正した。

```sql
EXPLAIN SELECT * FROM livecomments  WHERE id = '1'  AND livestream_id = '1532'  AND comment LIKE '%笑ってる%';
+----+-------------+--------------+------------+-------+---------------------------+---------+---------+-------+------+----------+-------+
| id | select_type | table        | partitions | type  | possible_keys             | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------------+------------+-------+---------------------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | livecomments | NULL       | const | PRIMARY,idx_livestream_id | PRIMARY | 8       | const |    1 |   100.00 | NULL  |
+----+-------------+--------------+------------+-------+---------------------------+---------+---------+-------+------+----------+-------+
```

ただ、、pt-query-digestでの計測タイミングにたまたま出てきただけなのか、あまり影響はなさそうだった。また、ちゃんとインデックスも効いていそうなことが分かった。

引き続き、スロークエリログのランキングとそのトップのクエリのログの具体例を確認した。

```bash
# Profile
# Rank Query ID                       Response time Calls R/Call V/M   Ite
# ==== ============================== ============= ===== ====== ===== ===
#    1 0x59F1B6DD8D9FEC059E55B3BFD...  5.5015 14.3%    87 0.0632  0.01 SELECT reservation_slots
#    2 0x84B457C910C4A79FC9EBECB8B...  4.3377 11.3%  2276 0.0019  0.00 SELECT icons
#    3 0x22279D81D51006139E0C76405...  4.1195 10.7%  3310 0.0012  0.00 SELECT domains domainmetadata
#    4 0x42EF7D7D98FBCC9723BF896EB...  3.5732  9.3%  2615 0.0014  0.00 SELECT records
#    5 0xFBC5564AE716EAE82F20BFB45...  3.5147  9.2%  5592 0.0006  0.00 SELECT tags
```

```sql
EXPLAIN SELECT slot FROM reservation_slots WHERE start_at = '1701223200' AND end_at = '1701226800';
+----+-------------+-------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table             | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | reservation_slots | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 8773 |     1.00 | Using where |
+----+-------------+-------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

reservation_slotsに対するインデックスを確認し、複合インデックスを貼るようにスキーマを変更した。

```sql
CREATE TABLE `reservation_slots` (
  `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `start_at` BIGINT NOT NULL,
  `end_at` BIGINT NOT NULL,
  `slot` INT NOT NULL,
  INDEX `idx_start_at_end_at` (`start_at`, `end_at`)
) ENGINE=InnoDB CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```

EXPLAINを実行すると、インデックスが効いていることが分かった。

```sql
mysql> EXPLAIN SELECT slot FROM reservation_slots WHERE start_at = '1701223200' AND end_at = '1701226800';
+----+-------------+-------------------+------------+------+---------------------+---------------------+---------+-------------+------+----------+-----------------------+
| id | select_type | table             | partitions | type | possible_keys       | key                 | key_len | ref         | rows | filtered | Extra                 |
+----+-------------+-------------------+------------+------+---------------------+---------------------+---------+-------------+------+----------+-----------------------+
|  1 | SIMPLE      | reservation_slots | NULL       | ref  | idx_start_at_end_at | idx_start_at_end_at | 16      | const,const |    1 |   100.00 | Using index condition |
+----+-------------+-------------------+------------+------+---------------------+---------------------+---------+-------------+------+----------+-----------------------+
```

この頃はスコアは上がっておらず、4000点から4500点の間でブレていた。

引き続き確認すると、下記のクエリがスロークエリとして出ていることが分かった。

```sql
mysql> EXPLAIN SELECT * FROM ng_words WHERE livestream_id = 2782;
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | ng_words | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 14213 |    10.00 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+-------+----------+-------------+
```

先ほど貼った下記の複合インデックスの順番が不適切だったと思われる。

```sql
-- 旧インデックス
INDEX `idx_user_id_livestream_id` (`user_id`, `livestream_id`)
-- 新インデックス
INDEX `idx_livestream_id_user_id` (`livestream_id`, `user_id`)
```

```sql
mysql> EXPLAIN SELECT * FROM ng_words WHERE livestream_id = 2782;
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------+------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys             | key                       | key_len | ref   | rows | filtered | Extra |
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | ng_words | NULL       | ref  | idx_livestream_id_user_id | idx_livestream_id_user_id | 8       | const |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------+------+----------+-------+

mysql> EXPLAIN SELECT id, user_id, livestream_id, word FROM ng_words WHERE user_id = '1013' AND livestream_id = '7526';
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------------+------+----------+-----------------------+
| id | select_type | table    | partitions | type | possible_keys             | key                       | key_len | ref         | rows | filtered | Extra                 |
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------------+------+----------+-----------------------+
|  1 | SIMPLE      | ng_words | NULL       | ref  | idx_livestream_id_user_id | idx_livestream_id_user_id | 16      | const,const |    1 |   100.00 | Using index condition |
+----+-------------+----------+------------+------+---------------------------+---------------------------+---------+-------------+------+----------+-----------------------+
```

上記の二つのように両方のクエリに対してインデックスが効いていることが分かった。

### アクセスログの確認

一旦ここまでのチューニングでインデックスはある程度貼ったので、MySQLのスロークエリログではなく、alpのアクセスログを確認することにした。

```text
+-------+-----+-----+-----+-----+-----+--------+----------------------------------------------+--------+--------+---------+--------+--------+--------+--------+--------+------------+------------+-------------+------------+
| COUNT | 1XX | 2XX | 3XX | 4XX | 5XX | METHOD |                     URI                      |  MIN   |  MAX   |   SUM   |  AVG   |  P90   |  P95   |  P99   | STDDEV | MIN(BODY)  | MAX(BODY)  |  SUM(BODY)  | AVG(BODY)  |
+-------+-----+-----+-----+-----+-----+--------+----------------------------------------------+--------+--------+---------+--------+--------+--------+--------+--------+------------+------------+-------------+------------+
| 69    | 0   | 67  | 0   | 2   | 0   | GET    | /api/livestream/search                       | 0.047  | 4.938  | 180.078 | 2.610  | 3.891  | 4.265  | 4.938  | 1.361  | 0.000      | 13349.000  | 520064.000  | 7537.159   |
| 152   | 0   | 148 | 0   | 3   | 1   | POST   | /api/register                                | 0.002  | 0.893  | 76.678  | 0.504  | 0.727  | 0.767  | 0.880  | 0.203  | 0.000      | 169878.000 | 232920.000  | 1532.368   |
| 122   | 0   | 118 | 0   | 4   | 0   | POST   | /api/livestream/reservation                  | 0.002  | 0.808  | 48.573  | 0.398  | 0.669  | 0.729  | 0.772  | 0.243  | 0.000      | 1103.000   | 109441.000  | 897.057    |
```

alpをみると上記のようになっていたが、その他のアクセスログをみていくと、同一エンドポイントにきていると思われる下記のようなログが別のログとして出ていた。

- `/api/livestream/7524/reaction`
- `/api/livestream/7524/livecomment`
- `/api/user/satomigoto0/statistics`

そのため、alpのmatching_groupsの設定を変更して、同一エンドポイントにきているものをまとめて表示するようにした。

```yaml
matching_groups: # array
  - /api/user/[a-zA-Z0-9]+/theme
  - /api/user/[a-zA-Z0-9]+/livestream
  - /api/livestream/[0-9]+/enter
  - /api/livestream/[0-9]+/exit
  - /api/livestream/[0-9]+
  - /api/livestream/[0-9]+/report
  - /api/livestream/[0-9]+/livecomment
  - /api/livestream/[0-9]+/ngwords
  - /api/livestream/[0-9]+/livecomment/[0-9]+/report
  - /api/livestream/[0-9]+/moderate
  - /api/livestream/[0-9]+/reaction
  - /api/user/[a-zA-Z0-9]+/icon
  - /api/user/[a-zA-Z0-9]+
  - /api/user/[a-zA-Z0-9]+/statistics
  - /api/livestream/[0-9]+/statistics
```

結果として、下記のように同一エンドポイントのクエリがまとめられるようになった。

```text
+-------+-----+------+-----+-----+-----+--------+-----------------------------------+-------+--------+---------+-------+--------+--------+--------+--------+-----------+------------+---------------+-----------+
| COUNT | 1XX | 2XX  | 3XX | 4XX | 5XX | METHOD |                URI                |  MIN  |  MAX   |   SUM   |  AVG  |  P90   |  P95   |  P99   | STDDEV | MIN(BODY) | MAX(BODY)  |   SUM(BODY)   | AVG(BODY) |
+-------+-----+------+-----+-----+-----+--------+-----------------------------------+-------+--------+---------+-------+--------+--------+--------+--------+-----------+------------+---------------+-----------+
| 641   | 0   | 634  | 0   | 7   | 0   | GET    | /api/livestream/[0-9]+            | 0.001 | 3.031  | 302.371 | 0.472 | 1.197  | 1.542  | 2.128  | 0.534  | 0.000     | 2214.000   | 467703.000    | 729.646   |
| 70    | 0   | 67   | 0   | 3   | 0   | GET    | /api/livestream/search            | 0.045 | 4.674  | 181.356 | 2.591 | 3.825  | 4.424  | 4.674  | 1.326  | 0.000     | 14203.000  | 517860.000    | 7398.000  |
| 1958  | 0   | 1956 | 0   | 2   | 0   | GET    | /api/user/[a-zA-Z0-9]+/icon       | 0.002 | 0.624  | 101.147 | 0.052 | 0.078  | 0.095  | 0.153  | 0.028  | 0.000     | 171652.000 | 106045557.000 | 54160.141 |
| 570   | 0   | 563  | 0   | 7   | 0   | POST   | /api/livestream/[0-9]+            | 0.003 | 0.928  | 82.627  | 0.145 | 0.213  | 0.237  | 0.320  | 0.075  | 0.000     | 2097.000   | 808213.000    | 1417.918  |
| 10    | 0   | 10   | 0   | 0   | 0   | GET    | /api/user/[a-zA-Z0-9]+            | 0.001 | 19.690 | 79.271  | 7.927 | 19.142 | 19.690 | 19.690 | 8.712  | 111.000   | 202.000    | 1454.000      | 145.400   |
| 141   | 0   | 139  | 0   | 1   | 1   | POST   | /api/register                     | 0.002 | 0.946  | 76.330  | 0.541 | 0.759  | 0.801  | 0.871  | 0.206  | 43.000    | 169839.000 | 229029.000    | 1624.319  |
| 122   | 0   | 118  | 0   | 4   | 0   | POST   | /api/livestream/reservation       | 0.002 | 0.973  | 48.864  | 0.401 | 0.655  | 0.706  | 0.779  | 0.245  | 0.000     | 1088.000   | 109083.000    | 894.123   |
| 140   | 0   | 130  | 0   | 1   | 9   | POST   | /api/icon                         | 0.021 | 1.434  | 27.254  | 0.195 | 0.222  | 0.769  | 1.339  | 0.240  | 0.000     | 170163.000 | 1532235.000   | 10944.536 |
| 147   | 0   | 144  | 0   | 3   | 0   | POST   | /api/login                        | 0.002 | 0.167  | 7.051   | 0.048 | 0.083  | 0.110  | 0.145  | 0.032  | 0.000     | 40.000     | 80.000        | 0.544     |
| 60    | 0   | 60   | 0   | 0   | 0   | GET    | /api/livestream                   | 0.004 | 0.492  | 5.135   | 0.086 | 0.210  | 0.228  | 0.492  | 0.108  | 2.000     | 2968.000   | 29874.000     | 497.900   |
| 101   | 0   | 101  | 0   | 0   | 0   | GET    | /api/tag                          | 0.001 | 0.168  | 4.627   | 0.046 | 0.068  | 0.103  | 0.149  | 0.027  | 1186.000  | 1186.000   | 119786.000    | 1186.000  |
| 1     | 0   | 1    | 0   | 0   | 0   | POST   | /api/initialize                   | 2.653 | 2.653  | 2.653   | 2.653 | 2.653  | 2.653  | 2.653  | 0.000  | 19.000    | 19.000     | 19.000        | 19.000    |
| 4     | 0   | 4    | 0   | 0   | 0   | GET    | /api/user/[a-zA-Z0-9]+/livestream | 0.235 | 0.696  | 1.796   | 0.449 | 0.696  | 0.696  | 0.696  | 0.166  | 1014.000  | 1530.000   | 5096.000      | 1274.000  |
| 26    | 0   | 26   | 0   | 0   | 0   | POST   | /api/livestream/[0-9]+/enter      | 0.002 | 0.186  | 1.307   | 0.050 | 0.085  | 0.108  | 0.186  | 0.036  | 0.000     | 0.000      | 0.000         | 0.000     |
| 16    | 0   | 16   | 0   | 0   | 0   | DELETE | /api/livestream/[0-9]+/exit       | 0.003 | 0.102  | 0.826   | 0.052 | 0.089  | 0.102  | 0.102  | 0.024  | 0.000     | 0.000      | 0.000         | 0.000     |
| 5     | 0   | 5    | 0   | 0   | 0   | GET    | /api/user/[a-zA-Z0-9]+/theme      | 0.002 | 0.123  | 0.386   | 0.077 | 0.123  | 0.123  | 0.123  | 0.044  | 59.000    | 59.000     | 295.000       | 59.000    |
| 1     | 0   | 1    | 0   | 0   | 0   | GET    | /api/payment                      | 0.023 | 0.023  | 0.023   | 0.023 | 0.023  | 0.023  | 0.023  | 0.000  | 15.000    | 15.000     | 15.000        | 15.000    |
+-------+-----+------+-----+-----+-----+--------+-----------------------------------+-------+--------+---------+-------+--------+--------+--------+--------+-----------+------------+---------------+-----------+
```

`/api/livestream/[0-9]+`のクエリがボトルネックになっていることが分かった。estackprofでコードを確認した。

```ruby
                                  |   425  |     # get livestream
                                  |   426  |     get '/api/livestream/:livestream_id' do
                                  |   427  |       verify_user_session!
                                  |   428  |
                                  |   429  |       livestream_id = cast_as_integer(params[:livestream_id])
                                  |   430  |
    2    (0.0%)                   |   431  |       livestream = db_transaction do |tx|
                                  |   432  |         livestream_model = tx.xquery('SELECT * FROM livestreams WHERE id = ?', livestream_id).first
                                  |   433  |         unless livestream_model
                                  |   434  |           raise HttpError.new(404)
                                  |   435  |         end
                                  |   436  |
    2    (0.0%)                   |   437  |         fill_livestream_response(tx, livestream_model)
                                  |   438  |       end
                                  |   439  |
                                  |   440  |       json(livestream)
                                  |   441  |     end
```

内部で呼び出しているコードに原因がありそうに見える。ひとまず`fill_livestream_response`関数のコードを確認した。

```ruby
                                  |   109  |       def fill_livestream_response(tx, livestream_model)
  286    (2.4%) /     7   (0.1%)  |   110  |         owner_model = tx.xquery('SELECT * FROM users WHERE id = ?', livestream_model.fetch(:user_id)).first
 2266   (19.2%) /     2   (0.0%)  |   111  |         owner = fill_user_response(tx, owner_model)
                                  |   112  |
 1262   (10.7%) /     8   (0.1%)  |   113  |         tags = tx.xquery('SELECT * FROM livestream_tags WHERE livestream_id = ?', livestream_model.fetch(:id)).map do |livestream_tag_model|
  928    (7.8%) /    12   (0.1%)  |   114  |           tag_model = tx.xquery('SELECT * FROM tags WHERE id = ?', livestream_tag_model.fetch(:tag_id)).first
    7    (0.1%) /     7   (0.1%)  |   115  |           {
    6    (0.1%) /     4   (0.0%)  |   116  |             id: tag_model.fetch(:id),
    2    (0.0%) /     1   (0.0%)  |   117  |             name: tag_model.fetch(:name),
                                  |   118  |           }
                                  |   119  |         end
                                  |   120  |
   13    (0.1%) /     4   (0.0%)  |   121  |         livestream_model.slice(:id, :title, :description, :playlist_url, :thumbnail_url, :start_at, :end_at).merge(
                                  |   122  |           owner:,
                                  |   123  |           tags:,
                                  |   124  |         )
                                  |   125  |       end
```

`fill_livestream_response`は複数箇所で呼ばれているとはいえ、あきらかに負荷になっている。とりあえずJOINでN+1問題の解決を図った。

```ruby
    get '/api/livestream/:livestream_id' do
      verify_user_session!

      livestream_id = cast_as_integer(params[:livestream_id])

      livestream = db_transaction do |tx|
        query = <<~SQL
          SELECT
            l.id, l.title, l.description, l.playlist_url, l.thumbnail_url, l.start_at, l.end_at,
            u.id AS user_id, u.name, u.display_name, u.description AS user_description,
            t.id AS tag_id, t.name AS tag_name,
            th.id AS theme_id, th.dark_mode,
            ic.image AS icon_image
          FROM livestreams l
          LEFT JOIN users u ON l.user_id = u.id
          LEFT JOIN livestream_tags lt ON lt.livestream_id = l.id
          LEFT JOIN tags t ON lt.tag_id = t.id
          LEFT JOIN themes th ON u.id = th.user_id
          LEFT JOIN icons ic ON u.id = ic.user_id
          WHERE l.id = ?
        SQL

        results = tx.xquery(query, livestream_id).to_a

        unless results.any?
          raise HttpError.new(404)
        end

        batch_livestream_response(results)
      end

      json(livestream)
    end

    def batch_livestream_response(results)
      tags = results.map do |row|
        { id: row[:tag_id], name: row[:tag_name] }.compact
      end.uniq

      owner = {
        id: results.first[:user_id],
        name: results.first[:name],
        display_name: results.first[:display_name],
        description: results.first[:user_description],
        theme: {
          id: results.first[:theme_id],
          dark_mode: results.first[:dark_mode]
        },
        icon_hash: results.first[:icon_image] ? Digest::SHA256.hexdigest(results.first[:icon_image]) : Digest::SHA256.hexdigest(File.binread(FALLBACK_IMAGE))
      }

      results.first.slice(:id, :title, :description, :playlist_url, :thumbnail_url, :start_at, :end_at).merge(
        owner: owner,
        tags: tags,
      )
    end
```

スコアとしては変わらなかったが、estackprof上でみると、ひとまず負荷は下がっていた。ICONの取得が結局遅そうなので、こちらの方針で進めるのではなく、画像を一旦静的ファイルにする方向に変えた。

```ruby
      # 読み取るコードの一例
      def fill_user_response(tx, user_model)
        theme_model = tx.xquery('SELECT * FROM themes WHERE user_id = ?', user_model.fetch(:id)).first

        # icon_model = tx.xquery('SELECT image FROM icons WHERE user_id = ?', user_model.fetch(:id)).first
        icon_path = "../img/#{user_model.fetch(:id)}.jpg"
        image =
          # 元々のコード
          # if icon_model
          #   icon_model.fetch(:image)
          # else
          #   File.binread(FALLBACK_IMAGE)
          # end
          if File.exist?(icon_path)
            File.binread(icon_path)
          else
            FALLBACK_IMAGE_BIN
          end
        icon_hash = Digest::SHA256.hexdigest(image)
```

```ruby
    # 書き込む側の一例
    post '/api/icon' do
      verify_user_session!

      sess = session[DEFAULT_SESSION_ID_KEY]
      unless sess
        raise HttpError.new(401)
      end
      user_id = sess[DEFAULT_USER_ID_KEY]
      unless user_id
        raise HttpError.new(401)
      end

      req = decode_request_body(PostIconRequest)
      image = Base64.decode64(req.image)

      # 元々のコード
      # icon_id = db_transaction do |tx|
      #   tx.xquery('DELETE FROM icons WHERE user_id = ?', user_id)
      #   tx.xquery('INSERT INTO icons (user_id, image) VALUES (?, ?)', user_id, image)
      #   tx.last_id
      # end

      File.open("../img/#{user_id}.jpg", mode = "w") do |f|
        f.write(image)
      end
```

ここでalpの設定が間違っていることに気がついたので修正した。

```yaml
matching_groups: # array
  - /api/user/[a-zA-Z0-9]+/theme
  - /api/user/[a-zA-Z0-9]+/livestream
  - /api/user/[a-zA-Z0-9]+/icon
  - /api/user/[a-zA-Z0-9]+/statistics
  - /api/user/[a-zA-Z0-9]+ # これは他のAPI(/api/user/~)よりも下に書かないと全部これとマッチしてしまう
  - /api/livestream/[0-9]+/enter
  - /api/livestream/[0-9]+/exit
  - /api/livestream/[0-9]+/report
  - /api/livestream/[0-9]+/livecomment/[0-9]+/report
  - /api/livestream/[0-9]+/livecomment
  - /api/livestream/[0-9]+/ngwords
  - /api/livestream/[0-9]+/moderate
  - /api/livestream/[0-9]+/reaction
  - /api/livestream/[0-9]+/statistics
  - /api/livestream/[0-9]+ # これは他のAPI(/api/user/~)よりも下に書かないと全部これとマッチしてしまう
```

上位のクエリが下記のように変わった。

```text
+-------+-----+------+-----+-----+-----+--------+--------------------------------------------------+-------+--------+---------+--------+--------+--------+--------+--------+-----------+------------+---------------+-----------+
| COUNT | 1XX | 2XX  | 3XX | 4XX | 5XX | METHOD |                       URI                        |  MIN  |  MAX   |   SUM   |  AVG   |  P90   |  P95   |  P99   | STDDEV | MIN(BODY) | MAX(BODY)  |   SUM(BODY)   | AVG(BODY) |
+-------+-----+------+-----+-----+-----+--------+--------------------------------------------------+-------+--------+---------+--------+--------+--------+--------+--------+-----------+------------+---------------+-----------+
| 246   | 0   | 240  | 0   | 6   | 0   | GET    | /api/livestream/search                           | 0.040 | 3.500  | 367.396 | 1.493  | 2.276  | 2.440  | 3.200  | 0.654  | 0.000     | 13946.000  | 1918891.000   | 7800.370  |
| 947   | 0   | 942  | 0   | 5   | 0   | GET    | /api/livestream/[0-9]+/livecomment               | 0.004 | 2.020  | 330.488 | 0.349  | 0.804  | 0.976  | 1.292  | 0.317  | 0.000     | 2242.000   | 1068818.000   | 1128.636  |
| 1015  | 0   | 1010 | 0   | 5   | 0   | GET    | /api/livestream/[0-9]+/reaction                  | 0.004 | 1.780  | 308.196 | 0.304  | 0.728  | 0.912  | 1.292  | 0.306  | 0.000     | 1585.000   | 806502.000    | 794.583   |
| 22    | 0   | 9    | 0   | 13  | 0   | GET    | /api/user/[a-zA-Z0-9]+/statistics                | 0.996 | 20.004 | 227.448 | 10.339 | 20.000 | 20.000 | 20.004 | 7.004  | 0.000     | 130.000    | 1048.000      | 47.636    |
| 467   | 0   | 459  | 0   | 6   | 2   | POST   | /api/register                                    | 0.028 | 1.684  | 189.820 | 0.406  | 0.660  | 0.788  | 1.200  | 0.217  | 0.000     | 169880.000 | 535918.000    | 1147.576  |
```
