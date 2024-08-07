# ClickHouse でサポートされている結合タイプ

https://clickhouse.com/blog/clickhouse-fully-supports-joins-part1

- INNER JOIN
- OUTER JOIN
- CROSS JOIN
- SEMI JOIN
- ANTI JOIN
- ANY JOIN
- ASOF JOIN

データ作成

```Sql
:) CREATE DATABASE imdb;

CREATE TABLE imdb.actors
(
    id         UInt32,
    first_name String,
    last_name  String,
    gender     FixedString(1)
) ENGINE = MergeTree ORDER BY (id, first_name, last_name, gender);

CREATE TABLE imdb.directors
(
    id         UInt32,
    first_name String,
    last_name  String
) ENGINE = MergeTree ORDER BY (id, first_name, last_name);

CRECREATE DATABASE imdb;

CREATE TABLE imdb.actors
(
    id         UInt32,
    first_name String,
    last_name  String,
    gender     FixedString(1)
) ENGINE = MergeTree ORDER BY (id, first_name, last_name, gender);

CREATE TABLE imdb.directors
(
    id         UInt32,
    first_name String,
    last_name  String
) ENGINE = MergeTree ORDER BY (id, first_name, last_name);

CREATE TABLE imdb.genres
(
    movie_id UInt32,
    genre    String
) ENGINE = MergeTree ORDER BY (movie_id, genre);

CREATE TABLE imdb.movie_directors
(
    director_id UInt32,
    movie_id    UInt64
) ENGINE = MergeTree ORDER BY (director_id, movie_id);

CREATE TABLE imdb.movies
(
    id   UInt32,
    name String,
    year UInt32,
    rank Float32 DEFAULT 0
) ENGINE = MergeTree ORDER BY (id, name, year);

CREATE TABLE imdb.roles
(
    actor_id   UInt32,
    movie_id   UInt32,
    role       String,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree ORDER BY (actor_id, movie_id);

CREATE DATABASE imdb

Query id: 5db89ece-665b-4ac1-8ce7-f41572c5e4e5

Ok.

0 rows in set. Elapsed: 0.009 sec.


CREATE TABLE imdb.actors
(
    `id` UInt32,
    `first_name` String,
    `last_name` String,
    `gender` FixedString(1)
)
ENGINE = MergeTree
ORDER BY (id, first_name, last_name, gender)

Query id: 875067a1-5088-4f80-9d94-a349580e0bee

Ok.

0 rows in set. Elapsed: 0.015 sec.


CREATE TABLE imdb.directors
(
    `id` UInt32,
    `first_name` String,
    `last_name` String
)
ENGINE = MergeTree
ORDER BY (id, first_name, last_name)

Query id: 74b2d42c-5e4f-47af-9619-6c9da91c629d

Ok.

0 rows in set. Elapsed: 0.014 sec.


CREATE TABLE imdb.genres
(
    `movie_id` UInt32,
    `genre` String
)
ENGINE = MergeTree
ORDER BY (movie_id, genre)

Query id: 3aa0206f-99f6-4333-93d5-fee5545973f9

Ok.

0 rows in set. Elapsed: 0.014 sec.


CREATE TABLE imdb.movie_directors
(
    `director_id` UInt32,
    `movie_id` UInt64
)
ENGINE = MergeTree
ORDER BY (director_id, movie_id)

Query id: 543f39f2-5e64-4876-8b02-ef1e5f258a2c

Ok.

0 rows in set. Elapsed: 0.015 sec.


CREATE TABLE imdb.movies
(
    `id` UInt32,
    `name` String,
    `year` UInt32,
    `rank` Float32 DEFAULT 0
)
ENGINE = MergeTree
ORDER BY (id, name, year)

Query id: 8fee0cc7-a1de-4c1d-86ba-bc6d015049d5

Ok.

0 rows in set. Elapsed: 0.015 sec.


CREATE TABLE imdb.roles
(
    `actor_id` UInt32,
    `movie_id` UInt32,
    `role` String,
    `created_at` DateTime DEFAULT now()
)
ENGINE = MergeTree
ORDER BY (actor_id, movie_id)

Query id: 9bd72f25-c63a-41b0-ba1f-097e650c69d3

Ok.

0 rows in set. Elapsed: 0.015 sec.

:) INSERT INTO imdb.actors
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_actors.tsv.gz',
'TSINSERT INTO imdb.actors
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_actors.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.directors
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_directors.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.genres
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies_genres.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.movie_directors
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies_directors.tsv.gz',
        'TSVWithNames');

INSERT INTO imdb.movies
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.roles(actor_id, movie_id, role)
SELECT actor_id, movie_id, role
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_roles.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.actors SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_actors.tsv.gz', 'TSVWithNames')

Query id: 43ebd680-0af4-471a-bb69-736d2de9ae3c

Ok.

0 rows in set. Elapsed: 7.912 sec. Processed 817.72 thousand rows, 25.60 MB (103.35 thousand rows/s., 3.24 MB/s.)
Peak memory usage: 44.43 MiB.


INSERT INTO imdb.directors SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_directors.tsv.gz', 'TSVWithNames')

Query id: 14866a4c-4265-4e23-b299-49e3d2428b26

Ok.

0 rows in set. Elapsed: 6.547 sec. Processed 86.88 thousand rows, 1.47 MB (13.27 thousand rows/s., 224.70 KB/s.)
Peak memory usage: 9.77 MiB.


INSERT INTO imdb.genres SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies_genres.tsv.gz', 'TSVWithNames')

Query id: 00464070-b705-4159-8a7d-4c099c7c7e7f

Ok.

0 rows in set. Elapsed: 7.308 sec. Processed 395.12 thousand rows, 6.81 MB (54.07 thousand rows/s., 931.22 KB/s.)
Peak memory usage: 19.01 MiB.


INSERT INTO imdb.movie_directors SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies_directors.tsv.gz', 'TSVWithNames')

Query id: 27932cef-1e22-485d-bb7e-7d6289d82cce

Ok.

0 rows in set. Elapsed: 6.933 sec. Processed 371.18 thousand rows, 4.05 MB (53.54 thousand rows/s., 584.24 KB/s.)
Peak memory usage: 8.99 MiB.


INSERT INTO imdb.movies SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies.tsv.gz', 'TSVWithNames')

Query id: f2bbc82c-b2f1-46ca-8097-7d0773c7caf6

Ok.

0 rows in set. Elapsed: 8.392 sec. Processed 388.27 thousand rows, 11.74 MB (46.27 thousand rows/s., 1.40 MB/s.)
Peak memory usage: 32.70 MiB.


INSERT INTO imdb.roles (actor_id, movie_id, role) SELECT
    actor_id,
    movie_id,
    role
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_roles.tsv.gz', 'TSVWithNames')

Query id: be26b7e8-1ae7-445c-8f9f-6893da35e505

Ok.

0 rows in set. Elapsed: 16.124 sec. Processed 3.43 million rows, 82.76 MB (212.85 thousand rows/s., 5.13 MB/s.)
Peak memory usage: 89.36 MiB.
```

データ確認

```Sql
:) SELECT id,
       any(actor_name)          as name,
       uniqExact(movie_id)    as num_movies,
       avg(rank)                as avg_rank,
       uniqExact(genre)         as unique_genres,
       uniqExact(director_name) as uniq_directors,
       max(created_at)          as updated_at
FROM (
         SELECT imdb.actors.id  as id,
                concat(imdb.actors.first_name, ' ', imdb.actors.last_name)  as actor_name,
                imdb.movies.id as movie_id,
                imdb.movies.rank as rank,
                genre,
                concat(imdb.directors.first_name, ' ', imdb.directors.last_name) as director_name,
                created_at
         FROM imdb.actors
                  JOIN imdb.roles ON imdb.roles.actor_id = imdb.actors.id
                  LEFT OUTER JOIN imdb.movies ON imdb.movies.id = imdb.roles.movie_id
                  LEFT OUTER JOIN imdb.genres ON imdb.genres.movie_id = imdb.movies.id
                  LEFT OUTER JOIN imdb.movie_directors ON imdb.movie_directors.movie_id = imdb.movies.id
                  LEFT OUTER JOIN imdb.directors ON imdb.directors.id = imdb.movie_directors.director_id
         )
GROUP BY id
ORDER BY num_movies DESC
LIMIT 5;

SELECT
    id,
    any(actor_name) AS name,
    uniqExact(movie_id) AS num_movies,
    avg(rank) AS avg_rank,
    uniqExact(genre) AS unique_genres,
    uniqExact(director_name) AS uniq_directors,
    max(created_at) AS updated_at
FROM
(
    SELECT
        imdb.actors.id AS id,
        concat(imdb.actors.first_name, ' ', imdb.actors.last_name) AS actor_name,
        imdb.movies.id AS movie_id,
        imdb.movies.rank AS rank,
        genre,
        concat(imdb.directors.first_name, ' ', imdb.directors.last_name) AS director_name,
        created_at
    FROM imdb.actors
    INNER JOIN imdb.roles ON imdb.roles.actor_id = imdb.actors.id
    LEFT JOIN imdb.movies ON imdb.movies.id = imdb.roles.movie_id
    LEFT JOIN imdb.genres ON imdb.genres.movie_id = imdb.movies.id
    LEFT JOIN imdb.movie_directors ON imdb.movie_directors.movie_id = imdb.movies.id
    LEFT JOIN imdb.directors ON imdb.directors.id = imdb.movie_directors.director_id
)
GROUP BY id
ORDER BY num_movies DESC
LIMIT 5

Query id: 2db60d40-e47b-4709-9772-4f580a827fa8

   ┌─────id─┬─name─────────┬─num_movies─┬───────────avg_rank─┬─unique_genres─┬─uniq_directors─┬──────────updated_at─┐
1. │  45332 │ Mel Blanc    │        909 │ 5.7884792542982515 │            19 │            148 │ 2024-07-27 17:50:01 │
2. │ 621468 │ Bess Flowers │        672 │  5.540605094212635 │            20 │            301 │ 2024-07-27 17:50:01 │
3. │ 283127 │ Tom London   │        549 │ 2.8057034230202023 │            18 │            208 │ 2024-07-27 17:50:01 │
4. │ 356804 │ Bud Osborne  │        544 │ 1.9575342420755093 │            16 │            157 │ 2024-07-27 17:50:01 │
5. │  89951 │ Edmund Cobb  │        544 │   2.72430730046193 │            17 │            203 │ 2024-07-27 17:50:01 │
   └────────┴──────────────┴────────────┴────────────────────┴───────────────┴────────────────┴─────────────────────┘

5 rows in set. Elapsed: 2.869 sec. Processed 5.49 million rows, 88.27 MB (1.91 million rows/s., 30.76 MB/s.)
Peak memory usage: 1.27 GiB.
```

データ規模

```Sql
SELECT
    `table`,
    formatReadableQuantity(sum(rows)) AS rows,
    formatReadableSize(sum(data_uncompressed_bytes)) AS data_uncompressed
FROM system.parts
WHERE (database = 'imdb') AND active
GROUP BY `table`
ORDER BY `table` ASC

Query id: 85f9ccd6-142c-4b44-a3c6-c5b245124ed4

   ┌─table───────────┬─rows────────────┬─data_uncompressed─┐
1. │ actors          │ 817.72 thousand │ 15.74 MiB         │
2. │ directors       │ 86.88 thousand  │ 1.62 MiB          │
3. │ genres          │ 395.12 thousand │ 4.29 MiB          │
4. │ movie_directors │ 371.18 thousand │ 4.25 MiB          │
5. │ movies          │ 388.27 thousand │ 11.72 MiB         │
6. │ roles           │ 3.43 million    │ 72.59 MiB         │
   └─────────────────┴─────────────────┴───────────────────┘

6 rows in set. Elapsed: 0.009 sec.
```

### INNER JOIN

```sql
:) SELECT
    m.name AS name,
    g.genre AS genre
FROM imdb.movies AS m
INNER JOIN imdb.genres AS g ON m.id = g.movie_id
ORDER BY
    m.year DESC,
    m.name ASC,
    g.genre ASC
LIMIT 10;

SELECT
    m.name AS name,
    g.genre AS genre
FROM imdb.movies AS m
INNER JOIN imdb.genres AS g ON m.id = g.movie_id
ORDER BY
    m.year DESC,
    m.name ASC,
    g.genre ASC
LIMIT 10

Query id: 2087b424-82e9-49ca-8437-3bc3dcc4652d

    ┌─name───────────────────────────────────┬─genre─────┐
 1. │ Harry Potter and the Half-Blood Prince │ Action    │
 2. │ Harry Potter and the Half-Blood Prince │ Adventure │
 3. │ Harry Potter and the Half-Blood Prince │ Family    │
 4. │ Harry Potter and the Half-Blood Prince │ Fantasy   │
 5. │ Harry Potter and the Half-Blood Prince │ Thriller  │
 6. │ DragonBall Z                           │ Action    │
 7. │ DragonBall Z                           │ Adventure │
 8. │ DragonBall Z                           │ Comedy    │
 9. │ DragonBall Z                           │ Fantasy   │
10. │ DragonBall Z                           │ Sci-Fi    │
    └────────────────────────────────────────┴───────────┘

10 rows in set. Elapsed: 0.104 sec. Processed 783.39 thousand rows, 21.50 MB (7.51 million rows/s., 206.01 MB/s.)
Peak memory usage: 68.34 MiB.
```

### (LEFT / RIGHT / FULL) OUTER JOIN

```sql
:) SELECT m.name
FROM imdb.movies AS m
LEFT JOIN imdb.genres AS g ON m.id = g.movie_id
WHERE g.movie_id = 0
ORDER BY
    m.year DESC,
    m.name ASC
LIMIT 10;

SELECT m.name
FROM imdb.movies AS m
LEFT JOIN imdb.genres AS g ON m.id = g.movie_id
WHERE g.movie_id = 0
ORDER BY
    m.year DESC,
    m.name ASC
LIMIT 10

Query id: 8f8d8a0f-c280-4ad4-bdf4-32db02e7c879

    ┌─name──────────────────────────────────────┐
 1. │ """Pacific War, The"""                    │
 2. │ """Turin 2006: XX Olympic Winter Games""" │
 3. │ Arthur, the Movie                         │
 4. │ Bridge to Terabithia                      │
 5. │ Mars in Aries                             │
 6. │ Master of Space and Time                  │
 7. │ Ninth Life of Louis Drax, The             │
 8. │ Paradox                                   │
 9. │ Ratatouille                               │
10. │ """American Dad"""                        │
    └───────────────────────────────────────────┘

10 rows in set. Elapsed: 0.098 sec.
```

### クロス結合

```sql
:) SELECT
    m.name,
    m.id,
    g.movie_id,
    g.genre
FROM imdb.movies AS m
CROSS JOIN imdb.genres AS g
LIMIT 10;

SELECT
    m.name,
    m.id,
    g.movie_id,
    g.genre
FROM imdb.movies AS m
CROSS JOIN imdb.genres AS g
LIMIT 10

Query id: ad6f4465-5235-4dd4-a9e3-3660facb8301

    ┌─name─┬─id─┬─movie_id─┬─genre───────┐
 1. │ #28  │  0 │        1 │ Documentary │
 2. │ #28  │  0 │        1 │ Short       │
 3. │ #28  │  0 │        2 │ Comedy      │
 4. │ #28  │  0 │        2 │ Crime       │
 5. │ #28  │  0 │        5 │ Western     │
 6. │ #28  │  0 │        6 │ Comedy      │
 7. │ #28  │  0 │        6 │ Family      │
 8. │ #28  │  0 │        8 │ Animation   │
 9. │ #28  │  0 │        8 │ Comedy      │
10. │ #28  │  0 │        8 │ Short       │
    └──────┴────┴──────────┴─────────────┘

10 rows in set. Elapsed: 0.021 sec.
```

```sql
:) SELECT
    m.name AS name,
    g.genre AS genre
FROM imdb.movies AS m
CROSS JOIN imdb.genres AS g
WHERE m.id = g.movie_id
ORDER BY
    m.year DESC,
    m.name ASC,
    g.genre ASC
LIMIT 10;

SELECT
    m.name AS name,
    g.genre AS genre
FROM imdb.movies AS m
CROSS JOIN imdb.genres AS g
WHERE m.id = g.movie_id
ORDER BY
    m.year DESC,
    m.name ASC,
    g.genre ASC
LIMIT 10

Query id: 21932af3-9b74-40f9-b179-14229424e954

    ┌─name───────────────────────────────────┬─genre─────┐
 1. │ Harry Potter and the Half-Blood Prince │ Action    │
 2. │ Harry Potter and the Half-Blood Prince │ Adventure │
 3. │ Harry Potter and the Half-Blood Prince │ Family    │
 4. │ Harry Potter and the Half-Blood Prince │ Fantasy   │
 5. │ Harry Potter and the Half-Blood Prince │ Thriller  │
 6. │ DragonBall Z                           │ Action    │
 7. │ DragonBall Z                           │ Adventure │
 8. │ DragonBall Z                           │ Comedy    │
 9. │ DragonBall Z                           │ Fantasy   │
10. │ DragonBall Z                           │ Sci-Fi    │
    └────────────────────────────────────────┴───────────┘

10 rows in set. Elapsed: 0.116 sec. Processed 624.50 thousand rows, 15.78 MB (5.36 million rows/s., 135.59 MB/s.)
Peak memory usage: 68.44 MiB.
```

クエリの WHERE セクションに結合式がある場合、ClickHouse は CROSS JOIN を INNER JOIN に書き換えます。

```sql
EXPLAIN SYNTAX
SELECT
    m.name AS name,
    g.genre AS genre
FROM imdb.movies AS m
CROSS JOIN imdb.genres AS g
WHERE m.id = g.movie_id
ORDER BY
    m.year DESC,
    m.name ASC,
    g.genre ASC
LIMIT 10

Query id: c83f3db8-f7f4-4d3f-85bf-dae81d576aec

    ┌─explain──────────────────────────────────────────┐
 1. │ SELECT                                           │
 2. │     name AS name,                                │
 3. │     genre AS genre                               │
 4. │ FROM imdb.movies AS m                            │
 5. │ ALL INNER JOIN imdb.genres AS g ON id = movie_id │
 6. │ WHERE id = movie_id                              │
 7. │ ORDER BY                                         │
 8. │     year DESC,                                   │
 9. │     name ASC,                                    │
10. │     genre ASC                                    │
11. │ LIMIT 10                                         │
    └──────────────────────────────────────────────────┘

11 rows in set. Elapsed: 0.054 sec.
```

### (LEFT / RIGHT) SEMI JOIN

LEFT SEMI JOIN クエリは、右側のテーブルに少なくとも 1 つの結合キーが一致する左側のテーブルの各行の列値を返します。最初に見つかった一致のみが返されます

```sql
:) SELECT
    a.first_name,
    a.last_name
FROM imdb.actors AS a
LEFT SEMI JOIN imdb.roles AS r ON a.id = r.actor_id
WHERE toYear(created_at) = '2024'
ORDER BY id ASC
LIMIT 10;

SELECT
    a.first_name,
    a.last_name
FROM imdb.actors AS a
SEMI LEFT JOIN imdb.roles AS r ON a.id = r.actor_id
WHERE toYear(created_at) = '2024'
ORDER BY id ASC
LIMIT 10

Query id: b23ff8a2-dbb9-487a-af76-0c484727f660

    ┌─first_name─┬─last_name──────────────┐
 1. │ Michael    │ 'babeepower' Viera     │
 2. │ Eloy       │ 'Chincheta'            │
 3. │ Dieguito   │ 'El Cigala'            │
 4. │ Antonio    │ 'El de Chipiona'       │
 5. │ José       │ 'El Francés'           │
 6. │ Félix      │ 'El Gato'              │
 7. │ Marcial    │ 'El Jalisco'           │
 8. │ José       │ 'El Morito'            │
 9. │ Francisco  │ 'El Niño de la Manola' │
10. │ Víctor     │ 'El Payaso'            │
    └────────────┴────────────────────────┘

10 rows in set. Elapsed: 0.325 sec. Processed 3.95 million rows, 45.61 MB (12.15 million rows/s., 140.32 MB/s.)
Peak memory usage: 123.67 MiB.
```

### (LEFT / RIGHT) ANTI JOIN

LEFT ANTI JOIN は、左側のテーブルから一致しないすべての行の列値を返します。

```sql
:) SELECT m.name
FROM imdb.movies AS m
LEFT ANTI JOIN imdb.genres AS g ON m.id = g.movie_id
ORDER BY
    year DESC,
    name ASC
LIMIT 10;

SELECT m.name
FROM imdb.movies AS m
ANTI LEFT JOIN imdb.genres AS g ON m.id = g.movie_id
ORDER BY
    year DESC,
    name ASC
LIMIT 10

Query id: 30fc1d8a-fe68-46d1-bc96-00c22a00c977

    ┌─name──────────────────────────────────────┐
 1. │ """Pacific War, The"""                    │
 2. │ """Turin 2006: XX Olympic Winter Games""" │
 3. │ Arthur, the Movie                         │
 4. │ Bridge to Terabithia                      │
 5. │ Mars in Aries                             │
 6. │ Master of Space and Time                  │
 7. │ Ninth Life of Louis Drax, The             │
 8. │ Paradox                                   │
 9. │ Ratatouille                               │
10. │ """American Dad"""                        │
    └───────────────────────────────────────────┘

10 rows in set. Elapsed: 0.145 sec. Processed 783.39 thousand rows, 15.42 MB (5.40 million rows/s., 106.29 MB/s.)
Peak memory usage: 34.44 MiB.
```

### (LEFT / RIGHT / INNER) ANY JOIN

左のテーブルの各行の列値を、右のテーブルの一致する行の列値と組み合わせるか、一致がない場合は右のテーブルのデフォルトの列値と組み合わせた値を返します。左のテーブルの行が右のテーブルに複数の一致がある場合、ClickHouse は最初に見つかった一致の結合された列値のみを返します

```Sql
:) WITH
    left_table AS (SELECT * FROM VALUES('c UInt32', 1, 2, 3)),
    right_table AS (SELECT * FROM VALUES('c UInt32', 2, 2, 3, 3, 4))
SELECT
    l.c AS l_c,
    r.c AS r_c
FROM left_table AS l
LEFT ANY JOIN right_table AS r ON l.c = r.c;

WITH
    left_table AS
    (
        SELECT *
        FROM VALUES('c UInt32', 1, 2, 3)
    ),
    right_table AS
    (
        SELECT *
        FROM VALUES('c UInt32', 2, 2, 3, 3, 4)
    )
SELECT
    l.c AS l_c,
    r.c AS r_c
FROM left_table AS l
ANY LEFT JOIN right_table AS r ON l.c = r.c

Query id: 3664f51f-61fe-4941-a77b-e9a045b30c22

   ┌─l_c─┬─r_c─┐
1. │   1 │   0 │
2. │   2 │   2 │
3. │   3 │   3 │
   └─────┴─────┘

3 rows in set. Elapsed: 0.005 sec.
```

```sql
:)
WITH
    left_table AS (SELECT * FROM VALUES('c UInt32', 1, 2, 3)),
    right_table AS (SELECT * FROM VALUES('c UInt32', 2, 2, 3, 3, 4))
SELECT
    l.c AS l_c,
    r.c AS r_c
FROM left_table AS l
RIGHT ANY JOIN right_table AS r ON l.c = r.c;

WITH
    left_table AS
    (
        SELECT *
        FROM VALUES('c UInt32', 1, 2, 3)
    ),
    right_table AS
    (
        SELECT *
        FROM VALUES('c UInt32', 2, 2, 3, 3, 4)
    )
SELECT
    l.c AS l_c,
    r.c AS r_c
FROM left_table AS l
ANY RIGHT JOIN right_table AS r ON l.c = r.c

Query id: 154bf805-306d-4eec-8b7a-2a532ece5a6c

   ┌─l_c─┬─r_c─┐
1. │   2 │   2 │
2. │   2 │   2 │
3. │   3 │   3 │
4. │   3 │   3 │
   └─────┴─────┘
   ┌─l_c─┬─r_c─┐
5. │   0 │   4 │
   └─────┴─────┘

5 rows in set. Elapsed: 0.053 sec.
```

### ASOF JOIN

ASOF JOIN は、非完全一致機能を提供します。左側のテーブルの行が右側のテーブルに完全に一致するものがない場合、代わりに右側のテーブルの最も近い一致する行が一致として使用されます。

# JOIN の内部

### Hash join

- 結合テーブルの右側の方が小さいテーブルのケース

```Sql
:) SELECT *
FROM imdb.roles AS r
JOIN imdb.actors AS a ON r.actor_id = a.id
FORMAT `Null`
SETTINGS join_algorithm = 'hash';

SELECT *
FROM imdb.roles AS r
INNER JOIN imdb.actors AS a ON r.actor_id = a.id
FORMAT `Null`
SETTINGS join_algorithm = 'hash'

Query id: 4e0fcd2f-cfc0-40c9-a90a-5ac36e995891

Ok.

0 rows in set. Elapsed: 0.405 sec. Processed 2.39 million rows, 77.11 MB (5.90 million rows/s., 190.23 MB/s.)
Peak memory usage: 179.14 MiB.
```

- 結合テーブルの左側の方が小さいテーブルのケース

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'hash';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'hash'

Query id: 8698be2f-6144-4776-8d86-75d97777b9d6

Ok.

0 rows in set. Elapsed: 0.447 sec. Processed 4.07 million rows, 126.68 MB (9.11 million rows/s., 283.57 MB/s.)
Peak memory usage: 377.06 MiB.
```

### Parallel hash join

- 結合テーブルの右側の方が小さいテーブルのケース

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'hash';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'hash'

Query id: 595edab4-c06a-4af3-beb3-063ded216ec6

Ok.

0 rows in set. Elapsed: 0.437 sec. Processed 4.17 million rows, 130.25 MB (9.53 million rows/s., 297.79 MB/s.)
Peak memory usage: 378.25 MiB.
```

- 結合テーブルの左側の方が小さいテーブルのケース

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'parallel_hash';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'parallel_hash'

Query id: d74e81fd-04fb-4ac6-9b4e-e68971e982be

Ok.

0 rows in set. Elapsed: 0.472 sec. Processed 4.23 million rows, 132.57 MB (8.97 million rows/s., 280.99 MB/s.)
Peak memory usage: 480.83 MiB.
```

### Grace ハッシュ結合

ハッシュ結合アルゴリズムと並列ハッシュ結合アルゴリズムはどちらも高速ですが、メモリに制約があります。右側のテーブルがメインメモリに収まらない場合、ClickHouse は OOM 例外を発生させます。このような状況では、ClickHouse ユーザーはパフォーマンスを犠牲にして、テーブルのデータをマージする前に外部ストレージに (部分的に) ソートする (部分的な) マージ アルゴリズムを使用できます。

右側の大きなテーブルとのハッシュ結合

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'grace_hash', grace_hash_join_initial_buckets = 3;

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'grace_hash', grace_hash_join_initial_buckets = 3

Query id: e8ada341-552d-4c6f-ace5-352c0c811f1e

Ok.

0 rows in set. Elapsed: 0.796 sec. Processed 4.25 million rows, 133.16 MB (5.34 million rows/s., 167.26 MB/s.)
Peak memory usage: 154.35 MiB.
```

右側の大きなテーブルとの Grace ハッシュ結合

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'grace_hash', grace_hash_join_initial_buckets = 3;

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'grace_hash', grace_hash_join_initial_buckets = 3

Query id: 7e68d27f-8d66-4595-8938-cd8560611e28

Ok.

0 rows in set. Elapsed: 0.766 sec. Processed 4.25 million rows, 133.16 MB (5.55 million rows/s., 173.92 MB/s.)
Peak memory usage: 156.81 MiB.
```

# 結合の内部 - フルソートマージ結合、部分マージ結合

https://clickhouse.com/blog/clickhouse-fully-supports-joins-full-sort-partial-merge-part3

### フルソートマージ結合

ClickHouse バージョンのソートマージ結合では、いくつかのパフォーマンス最適化が提供されます。

- 処理されるデータの量を最小限に抑えるために、並べ替えやマージ操作の前に、結合されたテーブルを互いの結合キーでフィルタリングすることができます。
- また、一方または両方のテーブルの物理的な行順序が結合キーのソート順序と一致する場合、対応するテーブルのソートフェーズはスキップされます。

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_rows_in_set_to_optimize_join = 0;

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_rows_in_set_to_optimize_join = 0

Query id: 05a67afc-1b53-4b8f-a818-98a670e8db8e

Ok.

0 rows in set. Elapsed: 5.958 sec. Processed 4.25 million rows, 133.16 MB (713.24 thousand rows/s., 22.35 MB/s.)
Peak memory usage: 252.61 MiB.
```

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_rows_in_set_to_optimize_join = 0, max_bytes_before_external_sort = '100M';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_rows_in_set_to_optimize_join = 0, max_bytes_before_external_sort = '100M'

Query id: 0c80ea82-772f-4be3-821b-eba0cc206d5d

Ok.

0 rows in set. Elapsed: 5.941 sec. Processed 4.25 million rows, 133.16 MB (715.30 thousand rows/s., 22.41 MB/s.)
Peak memory usage: 254.02 MiB.
```

### 最適化

```Sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge'

Query id: e52b6450-424d-4fc6-8093-f4f97cd2756f

Ok.

0 rows in set. Elapsed: 5.964 sec. Processed 4.25 million rows, 133.16 MB (712.57 thousand rows/s., 22.33 MB/s.)
Peak memory usage: 259.20 MiB.
```

```sql
:) SELECT countDistinct(first_name)
FROM imdb.actors;

SELECT countDistinct(first_name)
FROM imdb.actors

Query id: 85432634-519a-4176-96af-1cc837c56da6

   ┌─countDistinct(first_name)─┐
1. │                    110004 │
   └───────────────────────────┘

1 row in set. Elapsed: 0.062 sec.

:) SELECT countDistinct(role)
FROM imdb.roles;

SELECT countDistinct(role)
FROM imdb.roles

Query id: 934c95c0-3f76-47ea-8313-998c09618311

   ┌─countDistinct(role)─┐
1. │             1174675 │ -- 1.17 million
   └─────────────────────┘

1 row in set. Elapsed: 0.227 sec. Processed 3.43 million rows, 62.38 MB (15.13 million rows/s., 275.01 MB/s.)
Peak memory usage: 141.36 MiB.
```

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge'

Query id: da33e706-e730-4d30-b819-039fc633d04c

Ok.

0 rows in set. Elapsed: 5.933 sec. Processed 4.25 million rows, 133.16 MB (716.27 thousand rows/s., 22.44 MB/s.)
Peak memory usage: 258.45 MiB.
```

`max_rows_in_set_to_optimize_join`を使用

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_rows_in_set_to_optimize_join = 200_000;

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.first_name = r.role
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_rows_in_set_to_optimize_join = 200000

Query id: 2b79a916-3b52-4071-a674-452c9a411a46

Ok.

0 rows in set. Elapsed: 5.664 sec. Processed 4.25 million rows, 133.16 MB (750.36 thousand rows/s., 23.51 MB/s.)
Peak memory usage: 132.54 MiB.
```

物理的な行順序を利用する

```sql
:) SELECT
    name AS table,
    sorting_key
FROM system.tables
WHERE database = 'imdb';

SELECT
    name AS `table`,
    sorting_key
FROM system.tables
WHERE database = 'imdb'

Query id: d53319a2-718f-4cd7-b577-56787b19bda8

   ┌─table───────────┬─sorting_key───────────────────────┐
1. │ actors          │ id, first_name, last_name, gender │
2. │ directors       │ id, first_name, last_name         │
3. │ genres          │ movie_id, genre                   │
4. │ movie_directors │ director_id, movie_id             │
5. │ movies          │ id, name, year                    │
6. │ roles           │ actor_id, movie_id                │
   └─────────────────┴───────────────────────────────────┘

6 rows in set. Elapsed: 0.002 sec.
```

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_rows_in_set_to_optimize_join = 0;

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_rows_in_set_to_optimize_join = 0

Query id: 4a1a8d05-3d12-47b8-a2ac-f60c0bfa4147

Ok.

0 rows in set. Elapsed: 0.413 sec. Processed 4.25 million rows, 133.16 MB (10.29 million rows/s., 322.39 MB/s.)
Peak memory usage: 118.98 MiB.
```

`max_rows_in_set_to_optimize_join`を無効にせずに実行

```sql
:) SELECT *
FROM imdb.actors AS a
JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge'

Query id: 6dc0498c-fad0-45d1-a429-627dab5e4dcf

Ok.

0 rows in set. Elapsed: 0.414 sec. Processed 4.25 million rows, 133.16 MB (10.27 million rows/s., 321.65 MB/s.)
Peak memory usage: 115.32 MiB.
```

`max_bytes_before_external_sort`を制限して実行

```sql
:) SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_bytes_before_external_sort = '100M';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_bytes_before_external_sort = '100M'

Query id: 791ba4d5-64aa-47d1-9a87-81d9002f7a2b

Ok.

0 rows in set. Elapsed: 0.412 sec. Processed 4.25 million rows, 133.16 MB (10.31 million rows/s., 323.01 MB/s.)
Peak memory usage: 115.54 MiB.
```

### 部分マージ結合

部分マージ結合は、大きなテーブルを結合するときにメモリ使用量を最小限に抑えるように最適化されており、外部ソートによって最初に右側のテーブルのみを完全にソートします。メモリ内で処理されるデータの量を最小限に抑えるために、ディスク上に min-max インデックスを作成します。左側のテーブルは常にブロック単位でメモリ内でソートされます。ただし、左側のテーブルの物理的な行順序が結合キーのソート順序と一致する場合は、結合一致のメモリ内識別がより効率的になります。

```sql
:) SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'partial_merge';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'partial_merge'

Query id: 9768e90b-3084-4664-a089-c8307b2ff0b7

Ok.

0 rows in set. Elapsed: 0.720 sec. Processed 4.14 million rows, 129.36 MB (5.76 million rows/s., 179.76 MB/s.)
Peak memory usage: 295.93 MiB.
```

`max_bytes_before_external_sort`を制限して実行

```sql
:) SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_bytes_before_external_sort = '100M';

SELECT *
FROM imdb.actors AS a
INNER JOIN imdb.roles AS r ON a.id = r.actor_id
FORMAT `Null`
SETTINGS join_algorithm = 'full_sorting_merge', max_bytes_before_external_sort = '100M'

Query id: 9b1be53d-6408-4aaa-a069-c79ace956cb0

Ok.

0 rows in set. Elapsed: 0.416 sec. Processed 4.25 million rows, 133.16 MB (10.22 million rows/s., 320.12 MB/s.)
Peak memory usage: 118.97 MiB.
```
