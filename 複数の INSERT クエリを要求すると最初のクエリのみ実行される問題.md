# ClickHouse 複数の INSERT クエリを要求すると最初のクエリのみ実行される問題

テーブル作成

```sql
89772527cfbd :) CREATE TABLE images
                (
                    `id` String,
                    `timestamp` DateTime64(3),
                    `height` Int64,
                    `width` Int64,
                    `size` Int64
                )
                ENGINE = MergeTree
                ORDER BY (size, height, width);

CREATE TABLE images
(
    `id` String,
    `timestamp` DateTime64(3),
    `height` Int64,
    `width` Int64,
    `size` Int64
)
ENGINE = MergeTree
ORDER BY (size, height, width)

Query id: 4d63c401-6c5c-4993-8e6e-1a43abfe429c

Ok.

0 rows in set. Elapsed: 0.017 sec.
```

データ作成

```sql
89772527cfbd :) INSERT INTO images VALUES (1088619203512250448, '2023-03-24 00:24:03.684', 1536, 1536, 2207289);
                                INSERT INTO images VALUES (1088619204040736859, '2023-03-24 00:24:03.810', 1024, 1024, 1928974);
                                INSERT INTO images VALUES (1088619204749561989, '2023-03-24 00:24:03.979', 1024, 1024, 1275619);
                                INSERT INTO images VALUES (1088619206431477862, '2023-03-24 00:24:04.380', 2048, 2048, 5985703);
                                INSERT INTO images VALUES (1088619206905434213, '2023-03-24 00:24:04.493', 1024, 1024, 1558455);
                                INSERT INTO images VALUES (1088619208524431510, '2023-03-24 00:24:04.879', 1024, 1024, 1494869);
                                INSERT INTO images VALUES (1088619208425437515, '2023-03-24 00:24:05.160', 1024, 1024, 1538451);

INSERT INTO images FORMAT Values

Query id: e3e55d6e-78d0-4596-bc19-3a46f142eb7a

Ok.

1 rows in set. Elapsed: 0.051 sec.

89772527cfbd :) select * from images;

SELECT *
FROM images

Query id: 47e29a25-7bb6-420c-bd5d-6a64d71b154f

┌─id──────────────────┬───────────────timestamp─┬─height─┬─width─┬────size─┐
│ 1088619203512250448 │ 2023-03-24 00:24:03.684 │   1536 │  1536 │ 2207289 │
└─────────────────────┴─────────────────────────┴────────┴───────┴─────────┘

1 rows in set. Elapsed: 0.006 sec.
```

最初の一行のみ登録されている

別のクエリを実行すると、`Multi-statements are not allowed`とエラーが出た

```sql
89772527cfbd :) select 1;select 2;

Syntax error (Multi-statements are not allowed): failed at position 9 (end of query):

select 1;select 2;
```

どうも、`multiquery`パラメータをサーバに渡さないと、コマンドラインから複数のクエリは実行できない模様。

`multiquery`を指定して起動

```
sudo docker exec -it clickhouse-sample_client_1 /us
r/bin/clickhouse-client --host clickhouse-sample_server_1 --multiline --multiquery
```

複数のクエリが同時に処理できるようになった
`SELECT`複数クエリはエラーを通知して、`INSERT`複数クエリは 1 つ目のクエリだけ実行する動作は整合性がないと感じました。

```sql
89772527cfbd :) INSERT INTO images VALUES (1088619203512250448, '2023-03-24 00:24:03.684', 1536, 1536, 2207289);
                INSERT INTO images VALUES (1088619204040736859, '2023-03-24 00:24:03.810', 1024, 1024, 1928974);
                INSERT INTO images VALUES (1088619204749561989, '2023-03-24 00:24:03.979', 1024, 1024, 1275619);
                INSERT INTO images VALUES (1088619206431477862, '2023-03-24 00:24:04.380', 2048, 2048, 5985703);
                INSERT INTO images VALUES (1088619206905434213, '2023-03-24 00:24:04.493', 1024, 1024, 1558455);
                INSERT INTO images VALUES (1088619208524431510, '2023-03-24 00:24:04.879', 1024, 1024, 1494869);
                INSERT INTO images VALUES (1088619208425437515, '2023-03-24 00:24:05.160', 1024, 1024, 1538451);

INSERT INTO images FORMAT Values

Query id: ce97009f-fe1c-46fc-94d3-85f0d346930b

Ok.

1 rows in set. Elapsed: 0.008 sec.


INSERT INTO images FORMAT Values

Query id: 75cea100-5150-4d81-a6fe-337411545d19

Ok.

1 rows in set. Elapsed: 0.003 sec.


INSERT INTO images FORMAT Values

Query id: eabdcf56-7023-4687-ae13-05de6a7f5966

Ok.

1 rows in set. Elapsed: 0.004 sec.


INSERT INTO images FORMAT Values

Query id: aee37e01-be72-45ca-a4b9-f4d36010713a

Ok.

1 rows in set. Elapsed: 0.004 sec.


INSERT INTO images FORMAT Values

Query id: 33e35a8b-c111-4041-aace-f27b7ed43107

Ok.

1 rows in set. Elapsed: 0.004 sec.


INSERT INTO images FORMAT Values

Query id: ee5e3ea5-bda1-4fea-ab60-752b5e5d5dfe

Ok.

1 rows in set. Elapsed: 0.003 sec.


INSERT INTO images FORMAT Values

Query id: 140df42e-33ce-4707-ba2e-3acbc509d5bd

Ok.

1 rows in set. Elapsed: 0.002 sec.
```
