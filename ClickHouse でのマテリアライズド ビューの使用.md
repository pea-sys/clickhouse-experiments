# ClickHouse でのマテリアライズド ビューの使用

データのパフォーマンスと管理性を向上させるために ClickHouse に移動できる重要な処理ポイントがいくつかあります。ClickHouse で最も強力なツールの 1 つは、マテリアライズド ビューです。

### マテリアライズドビューとは何ですか?

SELECT マテリアライズド ビューは、データが挿入されると、そのデータに対するクエリの結果をターゲット テーブルに格納する特別なトリガーです

### Quick example

データ作成

```sql
:) CREATE TABLE wikistat
(
    `time` DateTime CODEC(Delta(4), ZSTD(1)),
    `project` LowCardinality(String),
    `subproject` LowCardinality(String),
    `path` String,
    `hits` UInt64
)
ENGINE = MergeTree
ORDER BY (path, time);

CREATE TABLE wikistat
(
    `time` DateTime CODEC(Delta(4), ZSTD(1)),
    `project` LowCardinality(String),
    `subproject` LowCardinality(String),
    `path` String,
    `hits` UInt64
)
ENGINE = MergeTree
ORDER BY (path, time)

Query id: 29584223-0b65-4f44-846e-5aea45f74192

Ok.

0 rows in set. Elapsed: 0.059 sec.

```

```sql
:) INSERT INTO wikistat SELECT *
FROM s3('https://ClickHouse-public-datasets.s3.amazonaws.com/wikistat/partitioned/wikistat*.native.zst') LIMIT 10000000

INSERT INTO wikistat SELECT *
FROM s3('https://ClickHouse-public-datasets.s3.amazonaws.com/wikistat/partitioned/wikistat*.native.zst')
LIMIT 10000000

Query id: 01d037d5-14a5-4a80-aabc-7c92731825ec

Ok.

0 rows in set. Elapsed: 103.389 sec. Processed 11.12 million rows, 24.66 MB (107.53 thousand rows/s., 238.53 KB/s.)
Peak memory usage: 773.09 MiB.
```

特定の日付に最も人気のあるプロジェクトを頻繁にクエリするとします。

```sql
:) SELECT
    project,
    sum(hits) AS h
FROM wikistat
WHERE date(time) = '2015-05-01'
GROUP BY project
ORDER BY h DESC
LIMIT 10;

SELECT
    project,
    sum(hits) AS h
FROM wikistat
WHERE date(time) = '2015-05-01'
GROUP BY project
ORDER BY h DESC
LIMIT 10

Query id: 56e07c27-3dce-4a89-9f39-a76b41ddb20f

    ┌─project─┬─────h─┐
 1. │ en      │ 14966 │
 2. │ it      │  3022 │
 3. │ zh      │  1193 │
 4. │ ja      │  1136 │
 5. │ es      │   872 │
 6. │ de      │   549 │
 7. │ fr      │   373 │
 8. │ tr      │   299 │
 9. │ id      │   281 │
10. │ ar      │   277 │
    └─────────┴───────┘

10 rows in set. Elapsed: 0.194 sec. Processed 7.36 million rows, 32.40 MB (37.87 million rows/s., 166.64 MB/s.)
Peak memory usage: 71.03 KiB.
```

このようなクエリが多数あり、パフォーマンス改善が必要な場合は、このクエリのマテリアライズド ビューを作成できます。

```sql
:) CREATE TABLE wikistat_top_projects
(
    `date` Date,
    `project` LowCardinality(String),
    `hits` UInt32
)
ENGINE = SummingMergeTree
ORDER BY (date, project);

:) CREATE MATERIALIZED VIEW wikistat_top_projects_mv TO wikistat_top_projects AS
SELECT
    date(time) AS date,
    project,
    sum(hits) AS hits
FROM wikistat
GROUP BY
    date,
    project;

CREATE TABLE wikistat_top_projects
(
    `date` Date,
    `project` LowCardinality(String),
    `hits` UInt32
)
ENGINE = SummingMergeTree
ORDER BY (date, project)

Query id: 9ec91b07-4814-4444-86d1-1dc69e21e3a1

Ok.

0 rows in set. Elapsed: 0.055 sec.

:) CREATE MATERIALIZED VIEW wikistat_top_projects_mv TO wikistat_top_projects AS
SELECT
    date(time) AS date,
    project,
    sum(hits) AS hits
FROM wikistat
GROUP BY
    date,
    project;

CREATE MATERIALIZED VIEW wikistat_top_projects_mv TO wikistat_top_projects
AS SELECT
    date(time) AS date,
    project,
    sum(hits) AS hits
FROM wikistat
GROUP BY
    date,
    project

Query id: 2fe5a3e1-b803-4f64-b406-f964886e3b76

Ok.

0 rows in set. Elapsed: 0.051 sec.
```

マテリアライズド ビューはいくつでも作成できますが、新しいマテリアライズド ビューごとに追加のストレージ負荷がかかるため、全体の数を適切な数に保ち、テーブルあたり 10 未満を目指します。

マテリアライズド ビューのターゲット テーブルにテーブルのデータを入力します。

```sql
:) INSERT INTO wikistat_top_projects SELECT
    date(time) AS date,
    project,
    sum(hits) AS hits
FROM wikistat
GROUP BY
    date,
    project;

INSERT INTO wikistat_top_projects SELECT
    date(time) AS date,
    project,
    sum(hits) AS hits
FROM wikistat
GROUP BY
    date,
    project

Query id: 2bdac308-39aa-4399-a95e-9bb09e65c7bc

Ok.

0 rows in set. Elapsed: 0.358 sec. Processed 10.00 million rows, 130.56 MB (27.90 million rows/s., 364.22 MB/s.)
Peak memory usage: 5.84 MiB.
```

### マテリアライズドビューテーブルのクエリ

```sql
:) SELECT
    project,
    sum(hits) hits
FROM wikistat_top_projects
WHERE date = '2015-05-01'
GROUP BY project
ORDER BY hits DESC
LIMIT 10;

SELECT
    project,
    sum(hits) AS hits
FROM wikistat_top_projects
WHERE date = '2015-05-01'
GROUP BY project
ORDER BY hits DESC
LIMIT 10

Query id: 9c331080-c0cd-482c-a909-82fb5c7273e3

    ┌─project─┬──hits─┐
 1. │ en      │ 14966 │
 2. │ it      │  3022 │
 3. │ zh      │  1193 │
 4. │ ja      │  1136 │
 5. │ es      │   872 │
 6. │ de      │   549 │
 7. │ fr      │   373 │
 8. │ tr      │   299 │
 9. │ id      │   281 │
10. │ ar      │   277 │
    └─────────┴───────┘

10 rows in set. Elapsed: 0.009 sec.
```

元のクエリの 1/10 以下の速度で応答が得られています  
SummingMergeTree エンジンは非同期であるため (これによりリソースが節約され、クエリ処理への影響が軽減されます)、一部の値は計算されない可能性があります。

### ディスク上のマテリアライズドビューのサイズを取得する

通常のテーブルと同様に取得できます

```sql
:) SELECT
    total_rows,
    formatReadableSize(total_bytes) AS total_bytes_on_disk
FROM system.tables
WHERE table = 'wikistat_top_projects';

SELECT
    total_rows,
    formatReadableSize(total_bytes) AS total_bytes_on_disk
FROM system.tables
WHERE `table` = 'wikistat_top_projects'

Query id: 4f8187fe-66f6-4f03-b072-75f4023ad48b

   ┌─total_rows─┬─total_bytes_on_disk─┐
1. │      14219 │ 48.38 KiB           │
   └────────────┴─────────────────────┘

1 row in set. Elapsed: 0.004 sec.
```

マテリアライズド ビューでデータを追加で更新する必要はありません。すべては ClickHouse によって自動的に実行されます

```sql
:) INSERT INTO wikistat
VALUES(now(), 'test', '', '', 10),(now(), 'test', '', '', 10),(now(), 'test', '', '', 20),(now(), 'test', '', '', 30);

INSERT INTO wikistat FORMAT Values

Query id: 19961e60-232e-4e36-86f7-78acca8ffb75

Ok.

4 rows in set. Elapsed: 0.015 sec.

:) SELECT hits
FROM wikistat_top_projects
FINAL
WHERE (project = 'test') AND (date = date(now()));

SELECT hits
FROM wikistat_top_projects
FINAL
WHERE (project = 'test') AND (date = date(now()))

Query id: 96c57664-4b29-4052-bd16-af9aba422503

   ┌─hits─┐
1. │   70 │
   └──────┘

1 row in set. Elapsed: 0.056 sec.
```

### マテリアライズドビューを使用して集計を高速化する

次のようなタイプのクエリが頻繁に実行されるとします。

```sql
:) SELECT
    toDate(time) AS date,
    min(hits) AS min_hits_per_hour,
    max(hits) AS max_hits_per_hour,
    avg(hits) AS avg_hits_per_hour
FROM wikistat
WHERE project = 'en'
GROUP BY date;

SELECT
    toDate(time) AS date,
    min(hits) AS min_hits_per_hour,
    max(hits) AS max_hits_per_hour,
    avg(hits) AS avg_hits_per_hour
FROM wikistat
WHERE project = 'en'
GROUP BY date

Query id: 872f0a8b-ccfb-4cde-bff0-5fe4d7695881

     ┌───────date─┬─min_hits_per_hour─┬─max_hits_per_hour─┬──avg_hits_per_hour─┐
  1. │ 2015-05-01 │                 1 │               156 │  3.213656860639897 │
  2. │ 2015-05-02 │                 1 │               137 │ 3.6442934405768965 │
・・・・・・・・・・
116. │ 2015-09-01 │                 1 │               111 │  3.443639874487087 │
     └───────date─┴─min_hits_per_hour─┴─max_hits_per_hour─┴──avg_hits_per_hour─┘

116 rows in set. Elapsed: 0.246 sec. Processed 3.80 million rows, 48.96 MB (15.43 million rows/s., 198.73 MB/s.)
Peak memory usage: 1.64 MiB.
```

より高速に取得するために、マテリアライズド ビューを使用してこれらの集計結果を保存しましょう。

```sql
:) CREATE MATERIALIZED VIEW wikistat_daily_summary_mv
TO wikistat_daily_summary AS
SELECT
    project,
    toDate(time) AS date,
    minState(hits) AS min_hits_per_hour,
    maxState(hits) AS max_hits_per_hour,
    avgState(hits) AS avg_hits_per_hour
FROM wikistat
GROUP BY project, date;

CREATE MATERIALIZED VIEW wikistat_daily_summary_mv TO wikistat_daily_summary
AS SELECT
    project,
    toDate(time) AS date,
    minState(hits) AS min_hits_per_hour,
    maxState(hits) AS max_hits_per_hour,
    avgState(hits) AS avg_hits_per_hour
FROM wikistat
GROUP BY
    project,
    date

Query id: 453ee4ad-6005-4243-96f6-a03049f0808f

Ok.

0 rows in set. Elapsed: 0.005 sec.
```

```sql
:) INSERT INTO wikistat_daily_summary SELECT
    project,
    toDate(time) AS date,
    minState(hits) AS min_hits_per_hour,
    maxState(hits) AS max_hits_per_hour,
    avgState(hits) AS avg_hits_per_hour
FROM wikistat
GROUP BY project, date;

INSERT INTO wikistat_daily_summary SELECT
    project,
    toDate(time) AS date,
    minState(hits) AS min_hits_per_hour,
    maxState(hits) AS max_hits_per_hour,
    avgState(hits) AS avg_hits_per_hour
FROM wikistat
GROUP BY
    project,
    date

Query id: 9c6e3df3-c07a-4619-a712-ff86701b3bc2

Ok.

0 rows in set. Elapsed: 0.342 sec. Processed 10.00 million rows, 131.44 MB (29.27 million rows/s., 384.75 MB/s.)
Peak memory usage: 16.37 MiB.
```

Merge コンビネータを使用して値を取得します  
まったく同じ結果が得られますが、処理速度は数千倍速いことに注意してください。

```sql
:) SELECT
    date,
    minMerge(min_hits_per_hour) min_hits_per_hour,
    maxMerge(max_hits_per_hour) max_hits_per_hour,
    avgMerge(avg_hits_per_hour) avg_hits_per_hour
FROM wikistat_daily_summary
WHERE project = 'en'
GROUP BY date;

SELECT
    date,
    minMerge(min_hits_per_hour) AS min_hits_per_hour,
    maxMerge(max_hits_per_hour) AS max_hits_per_hour,
    avgMerge(avg_hits_per_hour) AS avg_hits_per_hour
FROM wikistat_daily_summary
WHERE project = 'en'
GROUP BY date

Query id: 3e3dbc7b-2d61-4f98-addf-d27f961724c7

     ┌───────date─┬─min_hits_per_hour─┬─max_hits_per_hour─┬──avg_hits_per_hour─┐
  1. │ 2015-05-01 │                 1 │               156 │  3.213656860639897 │
  2. │ 2015-05-02 │                 1 │               137 │ 3.6442934405768965 │
・・・
116. │ 2015-09-01 │                 1 │               111 │  3.443639874487087 │
     └───────date─┴─min_hits_per_hour─┴─max_hits_per_hour─┴──avg_hits_per_hour─┘

116 rows in set. Elapsed: 0.009 sec.
```

### ストレージを最適化するためにデータを圧縮する

マテリアライズド ビューとソース テーブルの TTL を組み合わせることができます。
ここでは月ごとの集計データのみを保存するとします。

```sql
:) CREATE MATERIALIZED VIEW wikistat_monthly_mv TO
wikistat_monthly AS
SELECT
    toDate(toStartOfMonth(time)) AS month,
    path,
    sum(hits) AS hits
FROM wikistat
GROUP BY
    path,
    month;

CREATE MATERIALIZED VIEW wikistat_monthly_mv TO wikistat_monthly
AS SELECT
    toDate(toStartOfMonth(time)) AS month,
    path,
    sum(hits) AS hits
FROM wikistat
GROUP BY
    path,
    month

Query id: 158aa28b-2957-407b-a2f7-74e3618927f4

Ok.

```

元のテーブル (データは 1 時間ごとに保存されます) は、集計されたマテリアライズド ビューよりも 3 倍のディスク領域を必要とします。

ここで重要な注意点は、圧縮は結果の行数が少なくとも 10 倍減少する場合にのみ意味があるということです。その他の場合、ClickHouse の強力な圧縮およびエンコード アルゴリズムにより、集約なしで同等のストレージ効率が実現します。

月次集計ができたので、元のテーブルに TTL 式を追加して、1 週間後にデータが削除されるようにすることができます。

```Sql
:) ALTER TABLE wikistat MODIFY TTL time + INTERVAL 1 WEEK;

ALTER TABLE wikistat
    (MODIFY TTL time + toIntervalWeek(1))

Query id: 103f6d36-60b6-403e-bd14-a86c87333563

Ok.

0 rows in set. Elapsed: 0.913 sec.
```

### データの検証とフィルタリング

マテリアライズド ビューが使用されるもう 1 つの一般的な例は、挿入直後のデータの処理です。データ検証は良い例です。
不要な記号を含むすべての値を、結果のテーブルにクリーンなデータとして保存する前にフィルター処理するとします。

```sql
:) SELECT count(*)
FROM wikistat
WHERE NOT match(path, '[a-z0-9\\-]')
LIMIT 5;

SELECT count(*)
FROM wikistat
WHERE NOT match(path, '[a-z0-9\\-]')
LIMIT 5

Query id: 2f24b921-6417-4eb0-89ca-b5ff868e0596

   ┌─count()─┐
1. │       6 │
   └─────────┘

1 row in set. Elapsed: 0.007 sec.
```

検証フィルタリングを実装するには、すべてのデータを含むテーブルとクリーンなデータのみを含むテーブルの 2 つのテーブルが必要です。マテリアライズド ビューのターゲット テーブルは、クリーンなデータを含む最終テーブルの役割を果たし、ソース テーブルは一時的なものになります。

```sql
:) CREATE TABLE wikistat_src
(
    `time` DateTime,
    `project` LowCardinality(String),
    `subproject` LowCardinality(String),
    `path` String,
    `hits` UInt64
)
ENGINE = Null;

CREATE TABLE wikistat_src
(
    `time` DateTime,
    `project` LowCardinality(String),
    `subproject` LowCardinality(String),
    `path` String,
    `hits` UInt64
)
ENGINE = Null

Query id: 7b0936e3-4057-4a71-8f4b-0686dca1891b

Ok.

0 rows in set. Elapsed: 0.001 sec.
```

データ検証クエリを使用してマテリアライズド ビューを作成しましょう。

```Sql
:) CREATE TABLE wikistat_clean AS wikistat;;

CREATE TABLE wikistat_clean AS wikistat

Query id: dc874261-12ae-424f-b9c3-13e1ec83b123

Ok.

0 rows in set. Elapsed: 0.056 sec.

:) CREATE MATERIALIZED VIEW wikistat_clean_mv TO wikistat_clean
AS SELECT *
FROM wikistat_src
WHERE match(path, '[a-z0-9\\-]');

CREATE MATERIALIZED VIEW wikistat_clean_mv TO wikistat_clean
AS SELECT *
FROM wikistat_src
WHERE match(path, '[a-z0-9\\-]')

Query id: e81950f4-5ffa-4596-9792-61a112df4744

Ok.

0 rows in set. Elapsed: 0.004 sec.
```

データを挿入すると、wikistat_src 空のままになります。

```Sql
:) INSERT INTO wikistat_src SELECT * FROM s3('https://ClickHouse-public-datasets.s3.amazonaws.com/wikistat/partitioned/wikistat*.native.zst') LIMIT 100000;

INSERT INTO wikistat_src SELECT *
FROM s3('https://ClickHouse-public-datasets.s3.amazonaws.com/wikistat/partitioned/wikistat*.native.zst')
LIMIT 100000

Query id: e1677899-1f01-4fb2-8b82-d904cd9ea784

Ok.

0 rows in set. Elapsed: 53.873 sec. Processed 1.27 million rows, 7.59 MB (23.49 thousand rows/s., 140.82 KB/s.)
Peak memory usage: 388.19 MiB.


```

wikistat_clean マテリアライズド テーブルには有効な行のみが含まれます

```sql
:) SELECT count(*)
FROM wikistat_src;

SELECT count(*)
FROM wikistat_src

Query id: 720a8957-6963-4c7a-8336-9836ba9a8c44

   ┌─count()─┐
1. │       0 │
   └─────────┘

1 row in set. Elapsed: 0.003 sec.

:) SELECT count(*)
FROM wikistat_clean;

SELECT count(*)
FROM wikistat_clean

Query id: da966397-7081-425a-a987-a84f95db77d7

   ┌─count()─┐
1. │   73690 │
   └─────────┘

1 row in set. Elapsed: 0.004 sec.
```

検証ステートメントで 100000 - 73690 = 26310 行が削除されました。

### テーブルへのデータのルーティング

たとえば、無効なデータを削除するのではなく、別のテーブルにルーティングしたい場合があります。その場合は、別のクエリを使用して別のマテリアライズド ビューを作成します。

```Sql
:) CREATE TABLE wikistat_invalid AS wikistat;

CREATE TABLE wikistat_invalid AS wikistat

Query id: 9b1787c9-9203-43a1-91ab-430333c7deb0

Ok.

0 rows in set. Elapsed: 0.056 sec.

:) CREATE MATERIALIZED VIEW wikistat_invalid_mv TO wikistat_invalid
AS SELECT *
FROM wikistat_src
WHERE NOT match(path, '[a-z0-9\\-]');

CREATE MATERIALIZED VIEW wikistat_invalid_mv TO wikistat_invalid
AS SELECT *
FROM wikistat_src
WHERE NOT match(path, '[a-z0-9\\-]')

Query id: 5a455c1e-5b88-4ea6-97b7-5d2987a6faec

Ok.

0 rows in set. Elapsed: 0.004 sec.
```

無効な行が見つかります。

```sql
:) SELECT count(*)
FROM wikistat_invalid;

SELECT count(*)
FROM wikistat_invalid

Query id: eb14c197-7211-4a0e-8fca-8df925d8efec

   ┌─count()─┐
1. │   21987 │
   └─────────┘

1 row in set. Elapsed: 0.005 sec.
```

### データの変換

```sql
:) CREATE TABLE wikistat_human
(
    `date` Date,
    `hour` UInt8,
    `page` String
)
ENGINE = MergeTree
ORDER BY (page, date);

CREATE TABLE wikistat_human
(
    `date` Date,
    `hour` UInt8,
    `page` String
)
ENGINE = MergeTree
ORDER BY (page, date)

Query id: 6e57b9c7-6b33-470c-ae82-7ffbad267fe0

Ok.

0 rows in set. Elapsed: 0.056 sec.

:) CREATE MATERIALIZED VIEW wikistat_human_mv TO wikistat_human
AS SELECT
    date(time) AS date,
    toHour(time) AS hour,
    concat(project, if(subproject != '', '/', ''), subproject, '/', path) AS page,
    hits
FROM wikistat;

CREATE MATERIALIZED VIEW wikistat_human_mv TO wikistat_human
AS SELECT
    date(time) AS date,
    toHour(time) AS hour,
    concat(project, if(subproject != '', '/', ''), subproject, '/', path) AS page,
    hits
FROM wikistat

Query id: 47c325f3-fb2c-4ec8-b5d5-011814e4ce6d

Ok.

0 rows in set. Elapsed: 0.008 sec.
```
