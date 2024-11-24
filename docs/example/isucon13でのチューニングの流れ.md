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
# TODO: 更新内容を記載
```
