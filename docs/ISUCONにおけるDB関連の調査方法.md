# ISUCONにおけるDB関連の調査方法

この資料では、競技開始時にDB関連の情報を把握するための調査方法をまとめている。

## sql関連のフォルダの確認

例年通りならば`webapp/sql`に置いてある。

- 初期化コマンドやテーブル構造が書いてある場合があるので見ること
- serviceを再起動したら初期化プログラムを毎回走らせるようになっている場合がある(ISUCON11)

## テーブルのデータの確認

- どんなテーブル/データがあるか分からないと調査出来ないので調べる。

```sql
-- データベース一覧を表示
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| isuports           |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

-- データベースを選択
use isuports;

-- テーブル一覧を確認
show tables;
+--------------------+
| Tables_in_isuports |
+--------------------+
| id_generator       |
| tenant             |
| visit_history      |
+--------------------+

-- テーブルの中身を確認(先頭10個)
SELECT * From tenant LIMIT 10;

-- スキーマの確認
show create table tenant;

-- テーブル定義の確認
DESCRIBE tenant;

-- テーブルの行数を確認
SELECT table_name, engine, table_rows, avg_row_length, floor((data_length+index_length)/1024/1024) as allMB, floor((data_length)/1024/1024) as dMB, floor((index_length)/1024/1024) as iMB FROM information_schema.tables WHERE table_schema=database() ORDER BY (data_length+index_length) DESC;
-- 一応単体で確認するコマンドも載せておく
select count(*) from tenant;

-- インデックスの確認
-- 一つずつ簡易的に確認する
SHOW INDEX FROM tenant;
-- 全てのテーブルをまとめて確認する
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
    table_schema = 'データベース名';
```
