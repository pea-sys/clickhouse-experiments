# ClickHouse を使ってみる

ClickHouse は列指向 DB です。

以下のようなユースケースに適しています。

- ログ分析
- ビジネスインテリジェンス
- リアルタイムモニタリング
- アドテック
- 時系列データ分析

---

公式の docker image を使います

- docker-compose.yml

```
version: "3.7"

services:
  client:
    image: yandex/clickhouse-client
    entrypoint:
      - /bin/sleep
    command:
      - infinity
  server:
    image: yandex/clickhouse-server
    ports:
      - 8123:8123
    volumes:
      - ./volume:/var/lib/clickhouse
```

```
sudo docker-compose up -d
docker image ls
REPOSITORY                 TAG       IMAGE ID       CREATED       SIZE
yandex/clickhouse-server   latest    c739327b5607   2 years ago   826MB
yandex/clickhouse-client   latest    8208fbe345cd   2 years ago   805MB
```

コンテナにアクセスし、クライアントを起動しサーバーにアクセスします

```
docker exec -it clickhouse-sample_client_1 /usr/bin/clickhouse-client --host clickhouse-sample_server_1 --multiline
ClickHouse client version 22.1.3.7 (official build).
Connecting to clickhouse-sample_server_1:9000 as user default.
Connected to ClickHouse server version 22.1.3 revision 54455.
```

データベース一覧の確認

```sql
b61e6fae1579 :) show databases;

SHOW DATABASES

Query id: 3f1b4960-3008-4f06-aba9-6ed76e8052f3

┌─name───────────────┐
│ INFORMATION_SCHEMA │
│ default            │
│ information_schema │
│ system             │
└────────────────────┘

4 rows in set. Elapsed: 0.002 sec.
```

サンプルデータを作成します

テーブル作成

```sql
CREATE TABLE uk_price_paid
(
    price UInt32,
    date Date,
    postcode1 LowCardinality(String),
    postcode2 LowCardinality(String),
    type Enum8('terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4, 'other' = 0),
    is_new UInt8,
    duration Enum8('freehold' = 1, 'leasehold' = 2, 'unknown' = 0),
    addr1 String,
    addr2 String,
    street LowCardinality(String),
    locality LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String),
    county LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (postcode1, postcode2, addr1, addr2);
```

データを挿入

```sql
INSERT INTO uk_price_paid
WITH
   splitByChar(' ', postcode) AS p
SELECT
    toUInt32(price_string) AS price,
    parseDateTimeBestEffortUS(time) AS date,
    p[1] AS postcode1,
    p[2] AS postcode2,
    transform(a, ['T', 'S', 'D', 'F', 'O'], ['terraced', 'semi-detached', 'detached', 'flat', 'other']) AS type,
    b = 'Y' AS is_new,
    transform(c, ['F', 'L', 'U'], ['freehold', 'leasehold', 'unknown']) AS duration,
    addr1,
    addr2,
    street,
    locality,
    town,
    district,
    county
FROM url(
    'http://prod.publicdata.landregistry.gov.uk.s3-website-eu-west-1.amazonaws.com/pp-complete.csv',
    'CSV',
    'uuid_string String,
    price_string String,
    time String,
    postcode String,
    a String,
    b String,
    c String,
    addr1 String,
    addr2 String,
    street String,
    locality String,
    town String,
    district String,
    county String,
    d String,
    e String'
) SETTINGS max_http_get_redirects=10;
```

行数確認

```sql
b61e6fae1579 :) SELECT count()FROM uk_price_paid;

SELECT count()
FROM uk_price_paid

Query id: 7473736f-5d1c-4a40-bdcf-8ccb0e1c6e69

┌──count()─┐
│ 29212384 │
└──────────┘

1 rows in set. Elapsed: 0.005 sec.
```

サイズ確認

```sql
b61e6fae1579 :) SELECT formatReadableSize(total_bytes)FROM system.tables WHERE name = 'uk_price_paid';

SELECT formatReadableSize(total_bytes)
FROM system.tables
WHERE name = 'uk_price_paid'

Query id: b69e2ed3-abe0-4ce0-be3c-d21a9ea9d99e

┌─formatReadableSize(total_bytes)─┐
│ 246.56 MiB                      │
└─────────────────────────────────┘

1 rows in set. Elapsed: 0.013 sec.
```

いくつかクエリを実行

```sql
b61e6fae1579 :) SELECT
                   toYear(date) AS year,
                   round(avg(price)) AS price,
                   bar(price, 0, 1000000, 80
                )
                FROM uk_price_paid
                GROUP BY year
                ORDER BY year;

SELECT
    toYear(date) AS year,
    round(avg(price)) AS price,
    bar(price, 0, 1000000, 80)
FROM uk_price_paid
GROUP BY year
ORDER BY year ASC

Query id: fb5d58ef-7252-406d-92e2-0176f0801d3e

┌─year─┬──price─┬─bar(round(avg(price)), 0, 1000000, 80)─┐
│ 1995 │  67943 │ █████▍                                 │
│ 1996 │  71517 │ █████▋                                 │
│ 1997 │  78547 │ ██████▎                                │
│ 1998 │  85446 │ ██████▋                                │
│ 1999 │  96047 │ ███████▋                               │
│ 2000 │ 107497 │ ████████▌                              │
│ 2001 │ 118896 │ █████████▌                             │
│ 2002 │ 137962 │ ███████████                            │
│ 2003 │ 155903 │ ████████████▍                          │
│ 2004 │ 178896 │ ██████████████▎                        │
│ 2005 │ 189369 │ ███████████████▏                       │
│ 2006 │ 203541 │ ████████████████▎                      │
│ 2007 │ 219382 │ █████████████████▌                     │
│ 2008 │ 217141 │ █████████████████▎                     │
│ 2009 │ 213431 │ █████████████████                      │
│ 2010 │ 236117 │ ██████████████████▊                    │
│ 2011 │ 232810 │ ██████████████████▌                    │
│ 2012 │ 238398 │ ███████████████████                    │
│ 2013 │ 256954 │ ████████████████████▌                  │
│ 2014 │ 280081 │ ██████████████████████▍                │
│ 2015 │ 297381 │ ███████████████████████▋               │
│ 2016 │ 313616 │ █████████████████████████              │
│ 2017 │ 346808 │ ███████████████████████████▋           │
│ 2018 │ 351212 │ ████████████████████████████           │
│ 2019 │ 354990 │ ████████████████████████████▍          │
│ 2020 │ 378416 │ ██████████████████████████████▎        │
│ 2021 │ 388747 │ ███████████████████████████████        │
│ 2022 │ 410924 │ ████████████████████████████████▋      │
│ 2023 │ 390283 │ ███████████████████████████████▏       │
│ 2024 │ 365375 │ █████████████████████████████▏         │
└──────┴────────┴────────────────────────────────────────┘

30 rows in set. Elapsed: 0.219 sec. Processed 29.21 million rows, 175.27 MB (133.23 million rows/s., 799.36 MB/s.)
```

いやいや、実行速度は体感でもかなり早いと感じます。

```sql
b61e6fae1579 :) SELECT
                   toYear(date) AS year,
                   round(avg(price)) AS price,
                   bar(price, 0, 2000000, 100
                )
                FROM uk_price_paid
                WHERE town = 'LONDON'
                GROUP BY year
                ORDER BY year;

SELECT
    toYear(date) AS year,
    round(avg(price)) AS price,
    bar(price, 0, 2000000, 100)
FROM uk_price_paid
WHERE town = 'LONDON'
GROUP BY year
ORDER BY year ASC

Query id: 67205b0b-36c3-4515-825a-5597b2592d41

┌─year─┬───price─┬─bar(round(avg(price)), 0, 2000000, 100)───────────────┐
│ 1995 │  109157 │ █████▍                                                │
│ 1996 │  118685 │ █████▊                                                │
│ 1997 │  136546 │ ██████▋                                               │
│ 1998 │  153008 │ ███████▋                                              │
│ 1999 │  180669 │ █████████                                             │
│ 2000 │  215872 │ ██████████▋                                           │
│ 2001 │  233006 │ ███████████▋                                          │
│ 2002 │  263699 │ █████████████▏                                        │
│ 2003 │  278430 │ █████████████▊                                        │
│ 2004 │  304666 │ ███████████████▏                                      │
│ 2005 │  322920 │ ████████████████▏                                     │
│ 2006 │  356212 │ █████████████████▋                                    │
│ 2007 │  404081 │ ████████████████████▏                                 │
│ 2008 │  420784 │ █████████████████████                                 │
│ 2009 │  427803 │ █████████████████████▍                                │
│ 2010 │  480318 │ ████████████████████████                              │
│ 2011 │  496307 │ ████████████████████████▋                             │
│ 2012 │  519541 │ █████████████████████████▊                            │
│ 2013 │  616257 │ ██████████████████████████████▋                       │
│ 2014 │  724292 │ ████████████████████████████████████▏                 │
│ 2015 │  792531 │ ███████████████████████████████████████▋              │
│ 2016 │  843637 │ ██████████████████████████████████████████▏           │
│ 2017 │  983701 │ █████████████████████████████████████████████████▏    │
│ 2018 │ 1016806 │ ██████████████████████████████████████████████████▋   │
│ 2019 │ 1050022 │ ████████████████████████████████████████████████████▌ │
│ 2020 │ 1058345 │ ████████████████████████████████████████████████████▊ │
│ 2021 │  968428 │ ████████████████████████████████████████████████▍     │
│ 2022 │ 1008966 │ ██████████████████████████████████████████████████▍   │
│ 2023 │  957661 │ ███████████████████████████████████████████████▊      │
│ 2024 │  876848 │ ███████████████████████████████████████████▋          │
└──────┴─────────┴───────────────────────────────────────────────────────┘

30 rows in set. Elapsed: 0.106 sec. Processed 29.21 million rows, 86.02 MB (276.07 million rows/s., 812.93 MB/s.)
```

```sql

b61e6fae1579 :) SELECT
                    town,
                    district,
                    count() AS c,
                    round(avg(price)) AS price,
                    bar(price, 0, 5000000, 100)
                FROM uk_price_paid
                WHERE date >= '2020-01-01'
                GROUP BY
                    town,
                    district
                HAVING c >= 100
                ORDER BY price DESC
                LIMIT 100;

SELECT
    town,
    district,
    count() AS c,
    round(avg(price)) AS price,
    bar(price, 0, 5000000, 100)
FROM uk_price_paid
WHERE date >= '2020-01-01'
GROUP BY
    town,
    district
HAVING c >= 100
ORDER BY price DESC
LIMIT 100

Query id: 9ab1a1a8-9139-4080-8ce6-ee8f8e6b4a88

┌─town─────────────────┬─district───────────────┬─────c─┬───price─┬─bar(round(avg(price)), 0, 5000000, 100)─────────────────────────────────┐
│ LONDON               │ CITY OF LONDON         │  1106 │ 3544256 │ ██████████████████████████████████████████████████████████████████████▊ │
│ LONDON               │ CITY OF WESTMINSTER    │ 13855 │ 2850124 │ █████████████████████████████████████████████████████████               │
│ LONDON               │ KENSINGTON AND CHELSEA │  9172 │ 2487287 │ █████████████████████████████████████████████████▋                      │
│ LEATHERHEAD          │ ELMBRIDGE              │   319 │ 2125817 │ ██████████████████████████████████████████▌                             │
│ VIRGINIA WATER       │ RUNNYMEDE              │   521 │ 2010814 │ ████████████████████████████████████████▏                               │
│ LONDON               │ CAMDEN                 │ 10270 │ 1647863 │ ████████████████████████████████▊                                       │
│ HENLEY-ON-THAMES     │ BUCKINGHAMSHIRE        │   121 │ 1556673 │ ███████████████████████████████▏                                        │
│ NORTHWOOD            │ THREE RIVERS           │   217 │ 1435417 │ ████████████████████████████▋                                           │
│ COBHAM               │ ELMBRIDGE              │  1217 │ 1302091 │ ██████████████████████████                                              │
│ WINDSOR              │ BRACKNELL FOREST       │   147 │ 1284415 │ █████████████████████████▋                                              │
│ BARNET               │ ENFIELD                │   499 │ 1262372 │ █████████████████████████▏                                              │
│ LONDON               │ RICHMOND UPON THAMES   │  2298 │ 1253569 │ █████████████████████████                                               │
│ OXFORD               │ SOUTH OXFORDSHIRE      │  1120 │ 1234820 │ ████████████████████████▋                                               │
│ BEACONSFIELD         │ BUCKINGHAMSHIRE        │  1241 │ 1210925 │ ████████████████████████▏                                               │
│ WATFORD              │ HERTSMERE              │   108 │ 1200997 │ ████████████████████████                                                │
│ LONDON               │ ISLINGTON              │  9897 │ 1186390 │ ███████████████████████▋                                                │
│ WINDLESHAM           │ SURREY HEATH           │   305 │ 1182143 │ ███████████████████████▋                                                │
│ WEYBRIDGE            │ ELMBRIDGE              │  2086 │ 1170469 │ ███████████████████████▍                                                │
│ RICHMOND             │ RICHMOND UPON THAMES   │  2842 │ 1150622 │ ███████████████████████                                                 │
│ LONDON               │ HOUNSLOW               │  2220 │ 1133236 │ ██████████████████████▋                                                 │
│ SURBITON             │ ELMBRIDGE              │   298 │ 1126724 │ ██████████████████████▌                                                 │
│ ASCOT                │ WINDSOR AND MAIDENHEAD │  1444 │ 1122850 │ ██████████████████████▍                                                 │
│ ESHER                │ ELMBRIDGE              │  1590 │ 1115838 │ ██████████████████████▎                                                 │
│ RADLETT              │ HERTSMERE              │   869 │ 1069515 │ █████████████████████▍                                                  │
│ LONDON               │ HAMMERSMITH AND FULHAM │ 11039 │ 1047704 │ ████████████████████▊                                                   │
│ LEATHERHEAD          │ GUILDFORD              │   628 │ 1046559 │ ████████████████████▊                                                   │
│ CHALFONT ST GILES    │ BUCKINGHAMSHIRE        │   453 │ 1041965 │ ████████████████████▋                                                   │
│ SALCOMBE             │ SOUTH HAMS             │   361 │ 1026832 │ ████████████████████▌                                                   │
│ BROCKENHURST         │ NEW FOREST             │   370 │ 1025876 │ ████████████████████▌                                                   │
│ READING              │ WINDSOR AND MAIDENHEAD │   107 │ 1018512 │ ████████████████████▎                                                   │
│ FARNHAM              │ EAST HAMPSHIRE         │   162 │ 1006601 │ ████████████████████▏                                                   │
│ IVER                 │ BUCKINGHAMSHIRE        │   687 │ 1000530 │ ████████████████████                                                    │
│ GUILDFORD            │ WAVERLEY               │   448 │  994865 │ ███████████████████▊                                                    │
│ GERRARDS CROSS       │ BUCKINGHAMSHIRE        │  1401 │  990181 │ ███████████████████▋                                                    │
│ WELWYN               │ EAST HERTFORDSHIRE     │   119 │  987638 │ ███████████████████▋                                                    │
│ LONDON               │ TOWER HAMLETS          │ 17727 │  986222 │ ███████████████████▋                                                    │
│ BARNSLEY             │ ROTHERHAM              │   105 │  985232 │ ███████████████████▋                                                    │
│ BURFORD              │ WEST OXFORDSHIRE       │   346 │  973262 │ ███████████████████▍                                                    │
│ PETERSFIELD          │ CHICHESTER             │   151 │  967000 │ ███████████████████▎                                                    │
│ LONDON               │ KINGSTON UPON THAMES   │   149 │  960033 │ ███████████████████▏                                                    │
│ SUTTON COLDFIELD     │ LICHFIELD              │   180 │  954493 │ ███████████████████                                                     │
│ FARNHAM              │ HART                   │   195 │  952853 │ ███████████████████                                                     │
│ COVENTRY             │ WARWICK                │   153 │  944277 │ ██████████████████▊                                                     │
│ EGHAM                │ RUNNYMEDE              │  1637 │  943424 │ ██████████████████▋                                                     │
│ WEMBLEY              │ BRENT                  │  3332 │  937495 │ ██████████████████▋                                                     │
│ HARTFIELD            │ WEALDEN                │   137 │  934447 │ ██████████████████▋                                                     │
│ EAST MOLESEY         │ ELMBRIDGE              │   594 │  930040 │ ██████████████████▌                                                     │
│ INGATESTONE          │ CHELMSFORD             │   209 │  918918 │ ██████████████████▍                                                     │
│ LONDON               │ MERTON                 │  7778 │  908802 │ ██████████████████▏                                                     │
│ NEWPORT PAGNELL      │ MILTON KEYNES          │  1301 │  904795 │ ██████████████████                                                      │
│ KINGSTON UPON THAMES │ RICHMOND UPON THAMES   │   292 │  902050 │ ██████████████████                                                      │
│ HARPENDEN            │ ST ALBANS              │  2177 │  898048 │ █████████████████▊                                                      │
│ LONDON               │ WANDSWORTH             │ 23929 │  895107 │ █████████████████▊                                                      │
│ HENLEY-ON-THAMES     │ SOUTH OXFORDSHIRE      │  1813 │  893812 │ █████████████████▊                                                      │
│ HASSOCKS             │ LEWES                  │   150 │  890944 │ █████████████████▋                                                      │
│ POTTERS BAR          │ WELWYN HATFIELD        │   485 │  880695 │ █████████████████▌                                                      │
│ KINGSTON UPON THAMES │ KINGSTON UPON THAMES   │  3197 │  877000 │ █████████████████▌                                                      │
│ BILLINGSHURST        │ CHICHESTER             │   417 │  872776 │ █████████████████▍                                                      │
│ STOCKBRIDGE          │ TEST VALLEY            │   533 │  865871 │ █████████████████▎                                                      │
│ THAMES DITTON        │ ELMBRIDGE              │   761 │  860227 │ █████████████████▏                                                      │
│ LUTON                │ CENTRAL BEDFORDSHIRE   │   684 │  855495 │ █████████████████                                                       │
│ ALRESFORD            │ EAST HAMPSHIRE         │   106 │  855276 │ █████████████████                                                       │
│ CROYDON              │ SUTTON                 │   254 │  853841 │ █████████████████                                                       │
│ EAST GRINSTEAD       │ TANDRIDGE              │   184 │  852793 │ █████████████████                                                       │
│ LONDON               │ SOUTHWARK              │ 13992 │  851381 │ █████████████████                                                       │
│ TONBRIDGE            │ SEVENOAKS              │   177 │  850622 │ █████████████████                                                       │
│ HASLEMERE            │ CHICHESTER             │   395 │  846303 │ ████████████████▊                                                       │
│ LECHLADE             │ COTSWOLD               │   313 │  820660 │ ████████████████▍                                                       │
│ LONDON               │ BARNET                 │ 14544 │  813354 │ ████████████████▎                                                       │
│ TRING                │ BUCKINGHAMSHIRE        │   119 │  812011 │ ████████████████▏                                                       │
│ TWICKENHAM           │ RICHMOND UPON THAMES   │  3812 │  810941 │ ████████████████▏                                                       │
│ SOLIHULL             │ STRATFORD-ON-AVON      │   225 │  808039 │ ████████████████▏                                                       │
│ CHIGWELL             │ EPPING FOREST          │   753 │  807960 │ ████████████████▏                                                       │
│ MARLOW               │ BUCKINGHAMSHIRE        │  1284 │  807753 │ ████████████████▏                                                       │
│ TEDDINGTON           │ RICHMOND UPON THAMES   │  1867 │  807231 │ ████████████████▏                                                       │
│ LONDON               │ BRENT                  │  7388 │  797854 │ ███████████████▊                                                        │
│ LONDON               │ EALING                 │ 10688 │  797785 │ ███████████████▊                                                        │
│ PETWORTH             │ CHICHESTER             │   481 │  796118 │ ███████████████▊                                                        │
│ UPMINSTER            │ THURROCK               │   161 │  794506 │ ███████████████▊                                                        │
│ LONDON               │ HACKNEY                │ 12069 │  794332 │ ███████████████▊                                                        │
│ PULBOROUGH           │ CHICHESTER             │   147 │  792150 │ ███████████████▋                                                        │
│ WOKING               │ GUILDFORD              │   600 │  790322 │ ███████████████▋                                                        │
│ BERKHAMSTED          │ DACORUM                │  1775 │  788544 │ ███████████████▋                                                        │
│ READING              │ SOUTH OXFORDSHIRE      │  1035 │  786824 │ ███████████████▋                                                        │
│ KESTON               │ BROMLEY                │   276 │  783628 │ ███████████████▋                                                        │
│ HAYWARDS HEATH       │ WEALDEN                │   100 │  775449 │ ███████████████▌                                                        │
│ MUCH HADHAM          │ EAST HERTFORDSHIRE     │   148 │  769420 │ ███████████████▍                                                        │
│ SOLIHULL             │ WARWICK                │   193 │  758253 │ ███████████████▏                                                        │
│ BETCHWORTH           │ MOLE VALLEY            │   270 │  756630 │ ███████████████▏                                                        │
│ GREAT MISSENDEN      │ BUCKINGHAMSHIRE        │   753 │  756002 │ ███████████████                                                         │
│ RICKMANSWORTH        │ THREE RIVERS           │  2443 │  755854 │ ███████████████                                                         │
│ HINDHEAD             │ WAVERLEY               │   379 │  755027 │ ███████████████                                                         │
│ BOURNE END           │ BUCKINGHAMSHIRE        │   489 │  753022 │ ███████████████                                                         │
│ WELWYN               │ WELWYN HATFIELD        │   640 │  752550 │ ███████████████                                                         │
│ OXFORD               │ OXFORD                 │  6450 │  747390 │ ██████████████▊                                                         │
│ TUNBRIDGE WELLS      │ WEALDEN                │   321 │  746192 │ ██████████████▊                                                         │
│ ASCOT                │ BRACKNELL FOREST       │   550 │  746045 │ ██████████████▊                                                         │
│ AMERSHAM             │ BUCKINGHAMSHIRE        │  1577 │  744706 │ ██████████████▊                                                         │
│ INGATESTONE          │ BRENTWOOD              │   486 │  743476 │ ██████████████▋                                                         │
│ MAYFIELD             │ WEALDEN                │   293 │  742601 │ ██████████████▋                                                         │
└──────────────────────┴────────────────────────┴───────┴─────────┴─────────────────────────────────────────────────────────────────────────┘

100 rows in set. Elapsed: 0.354 sec. Processed 29.21 million rows, 310.35 MB (82.51 million rows/s., 876.55 MB/s.)
```

Projection を使った高速化  
事前に集計されたデータを任意の形式で保存することで、クエリ速度を向上させることができます  
マテビューみたいなものですね

```sql
b61e6fae1579 :) ALTER TABLE uk_price_paid
                    ADD PROJECTION projection_by_year_district_town
                    (
                        SELECT
                            toYear(date),
                            district,
                            town,
                            avg(price),
                            sum(price),
                            count()
                        GROUP BY
                            toYear(date),
                            district,
                            town
                    );

ALTER TABLE uk_price_paid
    ADD PROJECTION projection_by_year_district_town
    (
        SELECT
            toYear(date),
            district,
            town,
            avg(price),
            sum(price),
            count()
        GROUP BY
            toYear(date),
            district,
            town
    )

Query id: cb0ee654-0e25-456f-a20b-a1e7335ec311

Ok.

0 rows in set. Elapsed: 0.021 sec.
```

マテビュー作成には２秒かかりました

```
b61e6fae1579 :) ALTER TABLE uk_price_paid
                    MATERIALIZE PROJECTION projection_by_year_district_town
                SETTINGS mutations_sync = 1;

ALTER TABLE uk_price_paid
    MATERIALIZE PROJECTION projection_by_year_district_town
SETTINGS mutations_sync = 1

Query id: 57037b95-8472-401b-ac1b-de728dac2e16

Ok.

0 rows in set. Elapsed: 2.897 sec.
```

同じ 3 つのクエリをもう一度実行してみましょう。

比較  
何故か遅くなってしまいました

```
#１つ目のクエリ
no projection:30 rows in set. Elapsed: 0.219 sec. Processed 29.21 million rows, 175.27 MB (133.23 million rows/s., 799.36 MB/s.)
projection:30 rows in set. Elapsed: 0.208 sec. Processed 29.21 million rows, 175.27 MB (140.75 million rows/s., 844.47 MB/s.)
# ２つ目のクエリ
no projection:30 rows in set. Elapsed: 0.106 sec. Processed 29.21 million rows, 86.02 MB (276.07 million rows/s., 812.93 MB/s.)
projection:30 rows in set. Elapsed: 0.156 sec. Processed 29.21 million rows, 86.02 MB (187.07 million rows/s., 550.88 MB/s.)
# ３つ目のクエリ
no projection:100 rows in set. Elapsed: 0.354 sec. Processed 29.21 million rows, 310.35 MB (82.51 million rows/s., 876.55 MB/s.)
projection:100 rows in set. Elapsed: 0.372 sec. Processed 29.21 million rows, 310.35 MB (78.47 million rows/s., 833.67 MB/s.)
```

特定のバージョン以降は`Projection`を使用するとクエリ解釈に時間が掛かるようになったという Issue もあるので、`Projection`も慎重に適用したほうが良さそうですね。
