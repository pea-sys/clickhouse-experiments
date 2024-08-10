# ClickHouse での時系列データの操作

### ClickHouse で利用可能な日付と時刻の種類

ClickHouse には、いくつかの日付と時刻のタイプがあります。ユースケースに応じて、さまざまなタイプを適用できます。ほとんどの場合、日付に Date タイプを使用すれば十分です。

```sql
:) CREATE TABLE dates
(
    `date` Date,
    `datetime` DateTime,
    `precise_datetime` DateTime64(3),
    `very_precise_datetime` DateTime64(9)
)
ENGINE = MergeTree
ORDER BY tuple();

CREATE TABLE dates
(
    `date` Date,
    `datetime` DateTime,
    `precise_datetime` DateTime64(3),
    `very_precise_datetime` DateTime64(9)
)
ENGINE = MergeTree
ORDER BY tuple()

Query id: 3825165b-ec54-4402-b8d6-4c9e914e2746

Ok.

0 rows in set. Elapsed: 0.054 sec.
```

```sql
:) INSERT INTO dates SELECT NOW(), NOW(), NOW64(3), NOW64(9);

INSERT INTO dates SELECT
    NOW(),
    NOW(),
    NOW64(3),
    NOW64(9)

Query id: 1d7979d4-4b09-4520-8937-9c9ef6cb03bc

Ok.

0 rows in set. Elapsed: 0.070 sec.

:) SELECT * FROM dates;

SELECT *
FROM dates

Query id: 7a483bb5-09ff-4e32-b463-18fcfc98efd3

   ┌───────date─┬────────────datetime─┬────────precise_datetime─┬─────────very_precise_datetime─┐
1. │ 2024-07-28 │ 2024-07-28 10:11:48 │ 2024-07-28 10:11:48.042 │ 2024-07-28 10:11:48.091853385 │
   └────────────┴─────────────────────┴─────────────────────────┴───────────────────────────────┘

1 row in set. Elapsed: 0.055 sec.
```

### Timezones

```Sql
:) CREATE TABLE dtz
(
    `id` Int8,
    `t` DateTime('Europe/Berlin')
)
ENGINE = MergeTree
ORDER BY tuple();

CREATE TABLE dtz
(
    `id` Int8,
    `t` DateTime('Europe/Berlin')
)
ENGINE = MergeTree
ORDER BY tuple()

Query id: bb623976-f172-4718-942c-e7147d3f33aa

Ok.

0 rows in set. Elapsed: 0.079 sec.
```

```sql
:) INSERT INTO dtz SELECT 1, toDateTime('2022-12-12 12:13:14', 'America/New_York');

INSERT INTO dtz SELECT
    1,
    toDateTime('2022-12-12 12:13:14', 'America/New_York')

Query id: 9791584b-eb90-45ff-8415-a4ed6bf16f0a

Ok.

0 rows in set. Elapsed: 0.030 sec.

:) INSERT INTO dtz SELECT 2, toDateTime('2022-12-12 12:13:14');

INSERT INTO dtz SELECT
    2,
    toDateTime('2022-12-12 12:13:14')

Query id: fa72df48-13b2-4b9f-9cb0-1f61c042e707

Ok.

0 rows in set. Elapsed: 0.006 sec.

:) SELECT * FROM dtz;

SELECT *
FROM dtz

Query id: b6f914cc-defd-4019-859e-b1b208109550

   ┌─id─┬───────────────────t─┐
1. │  2 │ 2022-12-12 04:13:14 │
   └────┴─────────────────────┘
   ┌─id─┬───────────────────t─┐
2. │  1 │ 2022-12-12 18:13:14 │
   └────┴─────────────────────┘

2 rows in set. Elapsed: 0.004 sec.
```

### Querying

テーブル準備

```sql
:) CREATE TABLE wikistat
(
    `time` DateTime,
    `project` String,
    `subproject` String,
    `path` String,
    `hits` UInt64
)
ENGINE = MergeTree
ORDER BY (time);

CREATE TABLE wikistat
(
    `time` DateTime,
    `project` String,
    `subproject` String,
    `path` String,
    `hits` UInt64
)
ENGINE = MergeTree
ORDER BY time

Query id: e17bd88e-f3e7-44a7-beed-84d315ce98c6

Ok.

0 rows in set. Elapsed: 0.059 sec.

:) INSERT INTO wikistat SELECT *
FROM s3('https://ClickHouse-public-datasets.s3.amazonaws.com/wikistat/partitioned/wikistat*.native.zst') LIMIT 50000000;

INSERT INTO wikistat SELECT *
FROM s3('https://ClickHouse-public-datasets.s3.amazonaws.com/wikistat/partitioned/wikistat*.native.zst')
LIMIT 50000000

Query id: 7b4f0ca4-461a-4fe3-b1cb-5aa96bf302f6

Ok.

0 rows in set. Elapsed: 69.949 sec. Processed 51.15 million rows, 103.30 MB (731.18 thousand rows/s., 1.48 MB/s.)
Peak memory usage: 858.28 MiB.
```

### 期間に基づいて集計する

各日のヒットの合計数を取得します。

```sql
:) SELECT
    sum(hits) AS h,
    toDate(time) AS d
FROM wikistat
GROUP BY d
ORDER BY d ASC
LIMIT 5;

SELECT
    sum(hits) AS h,
    toDate(time) AS d
FROM wikistat
GROUP BY d
ORDER BY d ASC
LIMIT 5

Query id: 37d5df48-7ac2-49b5-9d39-f3eaf092c927

   ┌───────h─┬──────────d─┐
1. │ 2141934 │ 2015-05-01 │
2. │ 3398699 │ 2015-05-02 │
3. │ 3378380 │ 2015-05-03 │
4. │ 3380495 │ 2015-05-04 │
5. │ 3319123 │ 2015-05-05 │
   └─────────┴────────────┘

5 rows in set. Elapsed: 0.403 sec. Processed 42.98 million rows, 515.80 MB (106.54 million rows/s., 1.28 GB/s.)
Peak memory usage: 1.07 MiB.
```

特定の日付でフィルターすることもできます

```sql
:) SELECT
    sum(hits) AS v,
    toStartOfHour(time) AS h
FROM wikistat
WHERE date(time) = '2015-05-01'
GROUP BY h
ORDER BY h ASC
LIMIT 5;

SELECT
    sum(hits) AS v,
    toStartOfHour(time) AS h
FROM wikistat
WHERE date(time) = '2015-05-01'
GROUP BY h
ORDER BY h ASC
LIMIT 5

Query id: 399fd296-dc77-42c0-a86e-1130921fca1a

   ┌──────v─┬───────────────────h─┐
1. │ 136833 │ 2015-05-01 10:00:00 │
2. │ 139670 │ 2015-05-01 11:00:00 │
3. │ 135468 │ 2015-05-01 12:00:00 │
4. │ 146586 │ 2015-05-01 13:00:00 │
5. │ 140633 │ 2015-05-01 14:00:00 │
   └────────┴─────────────────────┘

5 rows in set. Elapsed: 0.071 sec.
```

### カスタムグループ化間隔

4 時間間隔でグループ化するとします。

```Sql
:) SELECT
    sum(hits) AS v,
    toStartOfInterval(time, INTERVAL 4 HOUR) AS h
FROM wikistat
WHERE date(time) = '2015-05-01'
GROUP BY h
ORDER BY h ASC
LIMIT 6;

SELECT
    sum(hits) AS v,
    toStartOfInterval(time, toIntervalHour(4)) AS h
FROM wikistat
WHERE date(time) = '2015-05-01'
GROUP BY h
ORDER BY h ASC
LIMIT 6

Query id: a6f6dfac-66bc-400e-ba26-6c9825e1efaf

   ┌──────v─┬───────────────────h─┐
1. │ 276503 │ 2015-05-01 08:00:00 │
2. │ 562129 │ 2015-05-01 12:00:00 │
3. │ 610280 │ 2015-05-01 16:00:00 │
4. │ 693022 │ 2015-05-01 20:00:00 │
   └────────┴─────────────────────┘

4 rows in set. Elapsed: 0.061 sec.
```

### Filling empty groups

```sql
:) SELECT
    toStartOfHour(time) AS h,
    sum(hits)
FROM wikistat
WHERE (project = 'it') AND (subproject = 'm') AND (date(time) = '2015-06-12')
GROUP BY h
ORDER BY h ASC;

SELECT
    toStartOfHour(time) AS h,
    sum(hits)
FROM wikistat
WHERE (project = 'it') AND (subproject = 'm') AND (date(time) = '2015-06-12')
GROUP BY h
ORDER BY h ASC

Query id: 1b3577ca-118c-471d-9c6b-dcb941351418

    ┌───────────────────h─┬─sum(hits)─┐
 1. │ 2015-06-12 00:00:00 │      2558 │
 2. │ 2015-06-12 01:00:00 │      2329 │
 3. │ 2015-06-12 08:00:00 │      2503 │
 4. │ 2015-06-12 09:00:00 │      1453 │
 5. │ 2015-06-12 10:00:00 │       835 │
 6. │ 2015-06-12 11:00:00 │       672 │
 7. │ 2015-06-12 12:00:00 │       374 │
 8. │ 2015-06-12 13:00:00 │       359 │
 9. │ 2015-06-12 14:00:00 │       575 │
10. │ 2015-06-12 15:00:00 │       851 │
11. │ 2015-06-12 16:00:00 │      1148 │
12. │ 2015-06-12 17:00:00 │      1572 │
13. │ 2015-06-12 18:00:00 │      1803 │
14. │ 2015-06-12 19:00:00 │      2177 │
15. │ 2015-06-12 20:00:00 │      2298 │
16. │ 2015-06-12 21:00:00 │      2659 │
    └─────────────────────┴───────────┘

16 rows in set. Elapsed: 0.236 sec.

```

`WITH FILL`で欠損値を補完します

```Sql
:) SELECT
    toStartOfHour(time) AS h,
    sum(hits)
FROM wikistat
WHERE (project = 'it') AND (subproject = 'm') AND (date(time) = '2015-06-12')
GROUP BY h
ORDER BY h ASC WITH FILL STEP toIntervalHour(1);

SELECT
    toStartOfHour(time) AS h,
    sum(hits)
FROM wikistat
WHERE (project = 'it') AND (subproject = 'm') AND (date(time) = '2015-06-12')
GROUP BY h
ORDER BY h ASC WITH FILL STEP toIntervalHour(1)

Query id: efddf322-8cbd-457f-a704-5685e7524053

    ┌───────────────────h─┬─sum(hits)─┐
 1. │ 2015-06-12 00:00:00 │      2558 │
 2. │ 2015-06-12 01:00:00 │      2329 │
 3. │ 2015-06-12 02:00:00 │         0 │
 4. │ 2015-06-12 03:00:00 │         0 │
 5. │ 2015-06-12 04:00:00 │         0 │
 6. │ 2015-06-12 05:00:00 │         0 │
 7. │ 2015-06-12 06:00:00 │         0 │
 8. │ 2015-06-12 07:00:00 │         0 │
 9. │ 2015-06-12 08:00:00 │      2503 │
10. │ 2015-06-12 09:00:00 │      1453 │
11. │ 2015-06-12 10:00:00 │       835 │
12. │ 2015-06-12 11:00:00 │       672 │
13. │ 2015-06-12 12:00:00 │       374 │
14. │ 2015-06-12 13:00:00 │       359 │
15. │ 2015-06-12 14:00:00 │       575 │
16. │ 2015-06-12 15:00:00 │       851 │
17. │ 2015-06-12 16:00:00 │      1148 │
18. │ 2015-06-12 17:00:00 │      1572 │
19. │ 2015-06-12 18:00:00 │      1803 │
20. │ 2015-06-12 19:00:00 │      2177 │
21. │ 2015-06-12 20:00:00 │      2298 │
22. │ 2015-06-12 21:00:00 │      2659 │
    └─────────────────────┴───────────┘

22 rows in set. Elapsed: 0.054 sec.
```

### Rolling time windows

```sql
:) SELECT
    sum(hits),
    dateDiff('day', toDateTime('2015-05-01 18:00:00'), time) AS d
FROM wikistat
GROUP BY d
ORDER BY d ASC
LIMIT 5;

SELECT
    sum(hits),
    dateDiff('day', toDateTime('2015-05-01 18:00:00'), time) AS d
FROM wikistat
GROUP BY d
ORDER BY d ASC
LIMIT 5

Query id: 0aa0a230-f360-4c3c-9f17-28aa371038d8

   ┌─sum(hits)─┬─d─┐
1. │   2141934 │ 0 │
2. │   3398699 │ 1 │
3. │   3378380 │ 2 │
4. │   3380495 │ 3 │
5. │   3319123 │ 4 │
   └───────────┴───┘

5 rows in set. Elapsed: 0.346 sec. Processed 44.42 million rows, 533.09 MB (128.27 million rows/s., 1.54 GB/s.)
Peak memory usage: 61.02 KiB.
```

### 視覚分析

```sql
:) SELECT
    toHour(time) AS h,
    sum(hits) AS t,
    bar(t, 0, max(t) OVER (), 50) AS bar
FROM wikistat
GROUP BY h
ORDER BY h ASC;

SELECT
    toHour(time) AS h,
    sum(hits) AS t,
    bar(t, 0, max(t) OVER (), 50) AS bar
FROM wikistat
GROUP BY h
ORDER BY h ASC

Query id: 11a73151-5dab-4cf7-b286-91f27e0b4404

    ┌──h─┬────────t─┬─bar────────────────────────────────────────────────┐
 1. │  0 │ 17222936 │ ██████████████████████████████████████████████████ │
 2. │  1 │ 17069062 │ █████████████████████████████████████████████████▌ │
 3. │  2 │ 16194528 │ ███████████████████████████████████████████████    │
 4. │  3 │ 15945924 │ ██████████████████████████████████████████████▎    │
 5. │  4 │ 15862771 │ ██████████████████████████████████████████████     │
 6. │  5 │ 15658224 │ █████████████████████████████████████████████▍     │
 7. │  6 │ 15308238 │ ████████████████████████████████████████████▍      │
 8. │  7 │ 14508582 │ ██████████████████████████████████████████         │
 9. │  8 │ 13327580 │ ██████████████████████████████████████▋            │
10. │  9 │ 12803118 │ █████████████████████████████████████▏             │
11. │ 10 │ 13164982 │ ██████████████████████████████████████▏            │
12. │ 11 │ 13276069 │ ██████████████████████████████████████▌            │
13. │ 12 │ 13302575 │ ██████████████████████████████████████▌            │
14. │ 13 │ 13298108 │ ██████████████████████████████████████▌            │
15. │ 14 │ 12717758 │ ████████████████████████████████████▉              │
16. │ 15 │ 12500229 │ ████████████████████████████████████▎              │
17. │ 16 │ 12722021 │ ████████████████████████████████████▉              │
18. │ 17 │ 12955038 │ █████████████████████████████████████▌             │
19. │ 18 │ 13357064 │ ██████████████████████████████████████▊            │
20. │ 19 │ 13844351 │ ████████████████████████████████████████▏          │
21. │ 20 │ 14255431 │ █████████████████████████████████████████▍         │
22. │ 21 │ 15261393 │ ████████████████████████████████████████████▎      │
23. │ 22 │ 16397055 │ ███████████████████████████████████████████████▌   │
24. │ 23 │ 17164467 │ █████████████████████████████████████████████████▊ │
    └────┴──────────┴────────────────────────────────────────────────────┘

24 rows in set. Elapsed: 0.360 sec. Processed 42.07 million rows, 504.79 MB (116.93 million rows/s., 1.40 GB/s.)
```

### カウンターとゲージのメトリック

時系列を扱うときに遭遇するメトリックには、基本的に 2 つの種類があります。

- カウンターは、属性別に分割され、時間枠別にグループ化された追跡イベントの合計数をカウントするために使用されます。ここでの一般的な例は、Web サイト訪問者の追跡です。
- ゲージは、時間の経過とともに変化する傾向があるメトリック値を設定するために使用されます。良い例としては、CPU 負荷の追跡が挙げられます。

`INTERPOLATE`を使用することで欠落データを補完できます

```Sql
:) CREATE TABLE metrics
( `time` DateTime, `name` String, `value` UInt32 )
ENGINE = MergeTree ORDER BY tuple();

INSERT INTO metrics VALUES
('2022-12-28 06:32:16', 'cpu', 7), ('2022-12-28 14:31:22', 'cpu', 50), ('2022-12-28 14:30:30', 'cpu', 25), ('2022-12-28 14:25:36', 'cpu', 10), ('2022-12-28 11:32:08', 'cpu', 5), ('2022-12-28 10:32:12', 'cpu', 5);

CREATE TABLE metrics
(
    `time` DateTime,
    `name` String,
    `value` UInt32
)
ENGINE = MergeTree
ORDER BY tuple()

Query id: 3d007d74-613b-4f64-9a3a-3992ec47dc52

Ok.

0 rows in set. Elapsed: 0.008 sec.


INSERT INTO metrics FORMAT Values

Query id: 2c129202-1c5f-4d30-984d-8254b777b48b

Ok.

6 rows in set. Elapsed: 0.005 sec.
:) SELECT
    toStartOfHour(time) AS h,
    any(value) AS v
FROM metrics
GROUP BY h
ORDER BY h ASC WITH FILL STEP toIntervalHour(1)
INTERPOLATE ( v AS v );

SELECT
    toStartOfHour(time) AS h,
    any(value) AS v
FROM metrics
GROUP BY h
ORDER BY h ASC WITH FILL STEP toIntervalHour(1)
INTERPOLATE ( v AS v )

Query id: 5d972317-727e-4287-ab73-d9f0d6b0c5c6

   ┌───────────────────h─┬──v─┐
1. │ 2022-12-28 06:00:00 │  7 │
2. │ 2022-12-28 07:00:00 │  7 │
3. │ 2022-12-28 08:00:00 │  7 │
4. │ 2022-12-28 09:00:00 │  7 │
5. │ 2022-12-28 10:00:00 │  5 │
6. │ 2022-12-28 11:00:00 │  5 │
7. │ 2022-12-28 12:00:00 │  5 │
8. │ 2022-12-28 13:00:00 │  5 │
9. │ 2022-12-28 14:00:00 │ 50 │
   └─────────────────────┴────┘

9 rows in set. Elapsed: 0.055 sec.
```

### ヒストグラム

```sql
:) WITH histogram(10)(hits) AS h
SELECT
    round(arrayJoin(h).1) AS l,
    round(arrayJoin(h).2) AS u,
    arrayJoin(h).3 AS w,
    bar(w, 0, max(w) OVER (), 20) AS b
FROM
(
    SELECT
        path,
        sum(hits) AS hits
    FROM wikistat
    WHERE date(time) = '2015-06-09'   GROUP BY path
    HAVING hits > 500.
);

WITH histogram(10)(hits) AS h
SELECT
    round(arrayJoin(h).1) AS l,
    round(arrayJoin(h).2) AS u,
    arrayJoin(h).3 AS w,
    bar(w, 0, max(w) OVER (), 20) AS b
FROM
(
    SELECT
        path,
        sum(hits) AS hits
    FROM wikistat
    WHERE date(time) = '2015-06-09'
    GROUP BY path
    HAVING hits > 500.
)

Query id: c9394b67-308f-4b1e-a161-7a3dd186fa8c

    ┌───────l─┬───────u─┬──────w─┬─b────────────────────┐
 1. │     501 │    1663 │    366 │ ████████████████████ │
 2. │    1663 │    3342 │ 85.125 │ ████▋                │
 3. │    3342 │    5009 │  9.625 │ ▌                    │
 4. │    5009 │    6940 │      3 │ ▏                    │
 5. │    6940 │    9540 │   1.25 │                      │
 6. │    9540 │   11802 │      1 │                      │
 7. │   11802 │   13141 │      1 │                      │
 8. │   13141 │   14236 │  1.125 │                      │
 9. │   14236 │ 1298456 │   1.75 │                      │
10. │ 1298456 │ 2582073 │  1.125 │                      │
    └─────────┴─────────┴────────┴──────────────────────┘

10 rows in set. Elapsed: 0.108 sec. Processed 933.89 thousand rows, 35.23 MB (8.65 million rows/s., 326.20 MB/s.)
Peak memory usage: 32.06 MiB.
```

### トレンド

```sql
:) SELECT
    toDate(time) AS d,
    sum(hits) AS h,
    lagInFrame(h) OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS p,
    h - p AS trend
FROM wikistat
WHERE path = '077'
GROUP BY d
ORDER BY d ASC
LIMIT 15;

SELECT
    toDate(time) AS d,
    sum(hits) AS h,
    lagInFrame(h) OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS p,
    h - p AS trend
FROM wikistat
WHERE path = '077'
GROUP BY d
ORDER BY d ASC
LIMIT 15

Query id: 4131bd3d-bd72-488e-9e78-67b97a10bcd5

    ┌──────────d─┬─h─┬─p─┬─trend─┐
 1. │ 2015-05-01 │ 2 │ 0 │     2 │
 2. │ 2015-05-02 │ 1 │ 2 │    -1 │
 3. │ 2015-05-04 │ 2 │ 1 │     1 │
 4. │ 2015-05-05 │ 2 │ 2 │     0 │
 5. │ 2015-05-06 │ 1 │ 2 │    -1 │
 6. │ 2015-05-07 │ 1 │ 1 │     0 │
 7. │ 2015-05-11 │ 4 │ 1 │     3 │
 8. │ 2015-05-12 │ 1 │ 4 │    -3 │
 9. │ 2015-05-13 │ 2 │ 1 │     1 │
10. │ 2015-05-14 │ 4 │ 2 │     2 │
11. │ 2015-05-15 │ 1 │ 4 │    -3 │
12. │ 2015-05-17 │ 2 │ 1 │     1 │
13. │ 2015-05-18 │ 1 │ 2 │    -1 │
14. │ 2015-05-19 │ 3 │ 1 │     2 │
15. │ 2015-05-20 │ 3 │ 3 │     0 │
    └────────────┴───┴───┴───────┘

15 rows in set. Elapsed: 0.677 sec. Processed 44.49 million rows, 1.10 GB (65.71 million rows/s., 1.63 GB/s.)
Peak memory usage: 4.12 MiB.
```

### 累積値

```sql
:) SELECT
    toDate(time) AS d,
    sum(hits) AS h,
    sum(h) OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND 0 FOLLOWING) AS c,
    bar(c, 0, 50, 25) AS b
FROM wikistat
WHERE path = '077'
GROUP BY d
ORDER BY d ASC
LIMIT 15;

SELECT
    toDate(time) AS d,
    sum(hits) AS h,
    sum(h) OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND 0 FOLLOWING) AS c,
    bar(c, 0, 50, 25) AS b
FROM wikistat
WHERE path = '077'
GROUP BY d
ORDER BY d ASC
LIMIT 15

Query id: b70c5a91-9ec7-4c7b-aa21-884493cdce30

    ┌──────────d─┬─h─┬──c─┬─b───────────────┐
 1. │ 2015-05-01 │ 2 │  2 │ █               │
 2. │ 2015-05-02 │ 1 │  3 │ █▌              │
 3. │ 2015-05-04 │ 2 │  5 │ ██▌             │
 4. │ 2015-05-05 │ 2 │  7 │ ███▌            │
 5. │ 2015-05-06 │ 1 │  8 │ ████            │
 6. │ 2015-05-07 │ 1 │  9 │ ████▌           │
 7. │ 2015-05-11 │ 4 │ 13 │ ██████▌         │
 8. │ 2015-05-12 │ 1 │ 14 │ ███████         │
 9. │ 2015-05-13 │ 2 │ 16 │ ████████        │
10. │ 2015-05-14 │ 4 │ 20 │ ██████████      │
11. │ 2015-05-15 │ 1 │ 21 │ ██████████▌     │
12. │ 2015-05-17 │ 2 │ 23 │ ███████████▌    │
13. │ 2015-05-18 │ 1 │ 24 │ ████████████    │
14. │ 2015-05-19 │ 3 │ 27 │ █████████████▌  │
15. │ 2015-05-20 │ 3 │ 30 │ ███████████████ │
    └────────────┴───┴────┴─────────────────┘

15 rows in set. Elapsed: 0.629 sec. Processed 48.47 million rows, 1.20 GB (77.00 million rows/s., 1.91 GB/s.)
Peak memory usage: 4.12 MiB.
```

### Rates

```Sql
:) SELECT
    toStartOfHour(time) AS t,
    sum(hits) AS h,
    round(h / (10 * 10), 2) AS rate,
    bar(rate * 10, 0, max(rate * 10) OVER (), 25) AS b
FROM wikistat
WHERE path = '077'
GROUP BY t
ORDER BY t ASC
LIMIT 23;

SELECT
    toStartOfHour(time) AS t,
    sum(hits) AS h,
    round(h / (10 * 10), 2) AS rate,
    bar(rate * 10, 0, max(rate * 10) OVER (), 25) AS b
FROM wikistat
WHERE path = '077'
GROUP BY t
ORDER BY t ASC
LIMIT 23

Query id: 86150c2e-4686-42dd-ab36-189f16830177

    ┌───────────────────t─┬─h─┬─rate─┬─b──────────┐
 1. │ 2015-05-01 20:00:00 │ 1 │ 0.01 │ █████      │
 2. │ 2015-05-01 21:00:00 │ 1 │ 0.01 │ █████      │
 3. │ 2015-05-02 05:00:00 │ 1 │ 0.01 │ █████      │
 4. │ 2015-05-04 16:00:00 │ 2 │ 0.02 │ ██████████ │
 5. │ 2015-05-05 02:00:00 │ 1 │ 0.01 │ █████      │
 6. │ 2015-05-05 10:00:00 │ 1 │ 0.01 │ █████      │
 7. │ 2015-05-06 19:00:00 │ 1 │ 0.01 │ █████      │
 8. │ 2015-05-07 20:00:00 │ 1 │ 0.01 │ █████      │
 9. │ 2015-05-11 02:00:00 │ 1 │ 0.01 │ █████      │
10. │ 2015-05-11 03:00:00 │ 1 │ 0.01 │ █████      │
11. │ 2015-05-11 04:00:00 │ 1 │ 0.01 │ █████      │
12. │ 2015-05-11 12:00:00 │ 1 │ 0.01 │ █████      │
13. │ 2015-05-12 04:00:00 │ 1 │ 0.01 │ █████      │
14. │ 2015-05-13 04:00:00 │ 1 │ 0.01 │ █████      │
15. │ 2015-05-13 05:00:00 │ 1 │ 0.01 │ █████      │
16. │ 2015-05-14 00:00:00 │ 2 │ 0.02 │ ██████████ │
17. │ 2015-05-14 01:00:00 │ 1 │ 0.01 │ █████      │
18. │ 2015-05-14 19:00:00 │ 1 │ 0.01 │ █████      │
19. │ 2015-05-15 23:00:00 │ 1 │ 0.01 │ █████      │
20. │ 2015-05-17 10:00:00 │ 2 │ 0.02 │ ██████████ │
21. │ 2015-05-18 09:00:00 │ 1 │ 0.01 │ █████      │
22. │ 2015-05-19 04:00:00 │ 1 │ 0.01 │ █████      │
23. │ 2015-05-19 13:00:00 │ 1 │ 0.01 │ █████      │
    └─────────────────────┴───┴──────┴────────────┘

23 rows in set. Elapsed: 0.634 sec. Processed 48.24 million rows, 1.19 GB (76.12 million rows/s., 1.88 GB/s.)
Peak memory usage: 81.35 KiB.
```

### 時系列ストレージの効率性の向上

```sql
:) SELECT
    uniq(project),
    uniq(subproject)
FROM wikistat;

SELECT
    uniq(project),
    uniq(subproject)
FROM wikistat

Query id: dbb84692-d868-4212-8772-ff514a5c6208

   ┌─uniq(project)─┬─uniq(subproject)─┐
1. │           601 │               45 │
   └───────────────┴──────────────────┘

1 row in set. Elapsed: 1.169 sec. Processed 47.47 million rows, 969.73 MB (40.61 million rows/s., 829.59 MB/s.)
Peak memory usage: 723.39 KiB.
```

```sql
:) ALTER TABLE wikistat
    MODIFY COLUMN `project` LowCardinality(String),
    MODIFY COLUMN `subproject` LowCardinality(String);

ALTER TABLE wikistat
    (MODIFY COLUMN `project` LowCardinality(String)),
    (MODIFY COLUMN `subproject` LowCardinality(String))

Query id: 1404f45b-c1bf-4182-88dc-fbecfa84594b

Ok.

0 rows in set. Elapsed: 5.086 sec.

:) SELECT max(hits)
FROM wikistat;

SELECT max(hits)
FROM wikistat

Query id: 3f6727ef-2977-4f08-8331-759eedb25859

   ┌─max(hits)─┐
1. │     71925 │
   └───────────┘

1 row in set. Elapsed: 0.121 sec. Processed 43.44 million rows, 347.48 MB (359.12 million rows/s., 2.87 GB/s.)
Peak memory usage: 107.03 KiB.
```

### シーケンスの保存を最適化するコーデック

時系列データのような連続したデータを扱う場合、特別なコーデックを使用することでストレージ効率をさらに向上させることができます。基本的な考え方は、絶対値そのものではなく、値間の変化を保存することです。これにより、ゆっくり変化するデータを扱うときに必要なスペースが大幅に削減されます。

```sql
:) ALTER TABLE wikistat
MODIFY COLUMN `time` CODEC(Delta, ZSTD);

ALTER TABLE wikistat
    (MODIFY COLUMN `time` CODEC(Delta, ZSTD))

Query id: 869fd5bb-69be-48da-8063-152bbd7015ad

Ok.

0 rows in set. Elapsed: 0.003 sec.
```
