# ClickHouse で TTL を使用してデータ ライフサイクルを管理する

次のブログ記事をハンズオンで進めます。
https://clickhouse.com/blog/using-ttl-to-manage-data-lifecycles-in-clickhouse

### 期限切れデータを自動的に削除する

テーブル作成

```sql
a4c7df3750ff :) CREATE TABLE events
                (
                    `event` String,
                    `time` DateTime,
                    `value` UInt64
                )
                ENGINE = MergeTree
                ORDER BY (event, time)
                TTL time + INTERVAL 1 MONTH DELETE;

CREATE TABLE events
(
    `event` String,
    `time` DateTime,
    `value` UInt64
)
ENGINE = MergeTree
ORDER BY (event, time)
TTL time + toIntervalMonth(1)

Query id: 579fa4c4-15c9-400c-af3d-d590a8a7c340

Ok.

0 rows in set. Elapsed: 0.019 sec.
```

データ登録

```sql
a4c7df3750ff :) INSERT INTO events VALUES('error', now() - interval 2 month, 123), ('error', now(), 123);

INSERT INTO events FORMAT Values

Query id: cc8d5b9b-1428-425d-9627-d66a57a6da4b

Ok.

2 rows in set. Elapsed: 0.008 sec.
```

データ確認

```sql
a4c7df3750ff :) SELECT * FROM events;

SELECT *
FROM events

Query id: 2530f726-7135-4ca7-8bb2-5f3b57806c0c

┌─event─┬────────────────time─┬─value─┐
│ error │ 2024-05-21 03:37:26 │   123 │
│ error │ 2024-07-21 03:37:26 │   123 │
└───────┴─────────────────────┴───────┘

1 rows in set. Elapsed: 0.006 sec.
```

しばらく待つとデータが古い削除されます

```sql
a4c7df3750ff :) SELECT * FROM events;

SELECT *
FROM events

Query id: 0fb9e183-4b11-4cb6-95ab-43ce106e80a8

┌─event─┬────────────────time─┬─value─┐
│ error │ 2024-07-21 03:37:26 │   123 │
└───────┴─────────────────────┴───────┘

```

デフォルトでは４時間毎に削除チェックされます

```
SETTINGS merge_with_ttl_timeout = 1200
```

でインターバルを設定できますが３００秒未満に設定することは禁止されています

### 削除する行をフィルタリング

テーブル作成

```sql
a4c7df3750ff :) CREATE OR REPLACE TABLE events
                (
                    `event` String,
                    `time` DateTime,
                    `value` UInt64
                )
                ENGINE = MergeTree
                ORDER BY (event, time)
                TTL time + INTERVAL 1 MONTH DELETE WHERE event = 'error';

CREATE OR REPLACE TABLE events
(
    `event` String,
    `time` DateTime,
    `value` UInt64
)
ENGINE = MergeTree
ORDER BY (event, time)
TTL time + toIntervalMonth(1) WHERE event = 'error'

Query id: c07a9106-251f-44c7-84e7-594d86cc07a6

Ok.

0 rows in set. Elapsed: 0.024 sec.
```

データ登録

```Sql
a4c7df3750ff :) INSERT INTO events VALUES('not_error', now() - interval 2 month, 123), ('error', now(), 123);

INSERT INTO events FORMAT Values

Query id: 13835ab0-1110-41dc-ab5f-54ea6e93f859

Ok.

2 rows in set. Elapsed: 0.008 sec.

```

データ確認

```Sql
a4c7df3750ff :) SELECT * FROM events;

SELECT *
FROM events

Query id: 91ad5b7e-1f34-44fe-848d-c98de8c8430e

┌─event─────┬────────────────time─┬─value─┐
│ error     │ 2024-07-21 04:09:33 │   123 │
│ not_error │ 2024-05-21 04:09:33 │   123 │
└───────────┴─────────────────────┴───────┘

2 rows in set. Elapsed: 0.004 sec.
```

１月経過しても、not_error は削除されません

### 複数の削除条件

```sql
a4c7df3750ff :) CREATE OR REPLACE TABLE events
                (
                    `event` String,
                    `time` DateTime,
                    `value` UInt64
                )
                ENGINE = MergeTree
                ORDER BY (event, time)
                TTL time + INTERVAL 1 MONTH DELETE WHERE event != 'error',
                    time + INTERVAL 6 MONTH DELETE WHERE event = 'error';

CREATE OR REPLACE TABLE events
(
    `event` String,
    `time` DateTime,
    `value` UInt64
)
ENGINE = MergeTree
ORDER BY (event, time)
TTL time + toIntervalMonth(1) WHERE event != 'error', time + toIntervalMonth(6) WHERE event = 'error'

Query id: 71227cb3-dd18-4e60-8d5c-1fe645581421

Ok.

0 rows in set. Elapsed: 0.021 sec.
```

### 削除対象データを履歴テーブルに移動する

```Sql
a4c7df3750ff :) CREATE OR REPLACE TABLE errors_history (
                    `event` String,
                    `time` DateTime,
                    `value` UInt64
                )
                ENGINE = MergeTree
                ORDER BY (event, time);

CREATE OR REPLACE TABLE errors_history
(
    `event` String,
    `time` DateTime,
    `value` UInt64
)
ENGINE = MergeTree
ORDER BY (event, time)

Query id: 86e84f60-96c3-4989-a7fc-3528c30d6370

Ok.

0 rows in set. Elapsed: 0.017 sec.
```

削除する前にデータ移動

```sql
a4c7df3750ff :) CREATE MATERIALIZED VIEW errors_history_mv TO errors_history AS
                SELECT * FROM events WHERE event = 'error';

CREATE MATERIALIZED VIEW errors_history_mv TO errors_history AS
SELECT *
FROM events
WHERE event = 'error'

Query id: a232493b-1924-4aa3-96ac-d5f328fdbf1f

Ok.

0 rows in set. Elapsed: 0.010 sec.
```

### 集計を使用して履歴データを圧縮する

行を削除する代わりに集計して行数を減らす

```sql
a4c7df3750ff :) CREATE OR REPLACE TABLE events
                (
                    `event` String,
                    `time` DateTime,
                    `value` UInt64
                )
                ENGINE = MergeTree
                ORDER BY (toDate(time), event)
                TTL time + INTERVAL 1 MONTH GROUP BY toDate(time), event SET value = SUM(value);

CREATE OR REPLACE TABLE events
(
    `event` String,
    `time` DateTime,
    `value` UInt64
)
ENGINE = MergeTree
ORDER BY (toDate(time), event)
TTL time + toIntervalMonth(1) GROUP BY toDate(time), event SET value = SUM(value)

Query id: 72770f44-5d6e-42b7-8cbb-35eb7708b390

Ok.

0 rows in set. Elapsed: 0.023 sec.
```

データ登録

```sql
a4c7df3750ff :) INSERT INTO events VALUES('error', now() - interval 2 month, 123),
                                         ('error', now() - interval 2 month, 321);

INSERT INTO events FORMAT Values

Query id: bbd23c29-073f-4178-939a-52942afbbab4

Ok.

2 rows in set. Elapsed: 0.012 sec.
```

少し時間をおいてデータ確認すると集約されて行数が圧縮されている

```sql
a4c7df3750ff :) SELECT * FROM events;

SELECT *
FROM events

Query id: 05daee98-b853-48d1-a205-1ebe70831dc5

┌─event─┬────────────────time─┬─value─┐
│ error │ 2024-05-21 04:50:34 │   444 │
└───────┴─────────────────────┴───────┘

1 rows in set. Elapsed: 0.006 sec.
```

### 圧縮の変更

古いデータに対して、より高い圧縮をかける

```sql
a4c7df3750ff :) CREATE OR REPLACE TABLE events
                (
                    `event` String,
                    `time` DateTime,
                    `value` UInt64
                )
                ENGINE = MergeTree
                ORDER BY (toDate(time), event)
                TTL time + INTERVAL 1 MONTH RECOMPRESS CODEC(LZ4HC(10));

CREATE OR REPLACE TABLE events
(
    `event` String,
    `time` DateTime,
    `value` UInt64
)
ENGINE = MergeTree
ORDER BY (toDate(time), event)
TTL time + toIntervalMonth(1) RECOMPRESS CODEC(LZ4HC(10))

Query id: 97f7943d-51d4-4e78-9135-c2481d94c1bd

Ok.

0 rows in set. Elapsed: 0.016 sec.
```

### 列レベルの TTL

列レベルの TTL を設定して、データを単調にして圧縮率を上げます  
ライフサイクルの異なる列が対象になります

```sql
a4c7df3750ff :) CREATE OR REPLACE TABLE events
                (
                    `event` String,
                    `time` DateTime,
                    `value` UInt64,
                    `debug` String TTL time + INTERVAL 1 WEEK
                )
                ENGINE = MergeTree
                ORDER BY (event, time);

CREATE OR REPLACE TABLE events
(
    `event` String,
    `time` DateTime,
    `value` UInt64,
    `debug` String TTL time + toIntervalWeek(1)
)
ENGINE = MergeTree
ORDER BY (event, time)

Query id: d53dd3fb-eee1-4065-87aa-d3e4c4712303

Connecting to clickhouse-sample_server_1:9000 as user default.
Connected to ClickHouse server version 22.1.3 revision 54455.

Ok.

0 rows in set. Elapsed: 0.021 sec.
```

```sql
a4c7df3750ff :) INSERT INTO events VALUES('error', now() - interval 1 month, 45, 'a lot of details');

INSERT INTO events FORMAT Values

Query id: 91860498-2ead-40a8-bf99-058e906249ad

Ok.

1 rows in set. Elapsed: 0.014 sec.

a4c7df3750ff :) SELECT * FROM events;

SELECT *
FROM events

Query id: 6148dbc8-85bf-42f6-b3a6-bdd9fefe83ec

┌─event─┬────────────────time─┬─value─┬─debug─┐
│ error │ 2024-06-21 04:58:07 │    45 │       │
└───────┴─────────────────────┴───────┴───────┘

1 rows in set. Elapsed: 0.006 sec.
```
