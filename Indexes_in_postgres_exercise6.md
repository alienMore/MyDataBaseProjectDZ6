<h1 align="center">ДЗ 6</h1>
<h1 align="center">Индексы в postgreSQL</h1>

---
1. ### Индекс по полю + Индекс на несколько полей ###

```sql
--таблица
CREATE TABLE IF NOT EXISTS my_otus_project_schema.data_from_elasticsearch
(
date_error_appeared timestamp,
warning_type VARCHAR(50),
host VARCHAR(50),
date_inser_in_table timestamp DEFAULT CURRENT_TIMESTAMP
) TABLESPACE my_otus_project_tablespace;

--INSERT в таблицу
INSERT INTO data_from_elasticsearch
(date_error_appeared, warning_type, host, date_inser_in_table)
SELECT a.date_error_appeared::date, my_random_string(10), my_random_string(10), b.date_inser_in_table
FROM generate_series(
    '2017-02-07'::date,
    '2017-05-17'::date,
    '1 day'::interval
) AS a(date_error_appeared),
generate_series(
    '2019-02-07'::date,
    '2019-05-17'::date,
    '1 day'::interval
) AS b(date_inser_in_table)
ORDER BY random();
--таблица содержит 40000 строк
SELECT count(*) FROM data_from_elasticsearch;

--индекса нет
EXPLAIN (analyze) SELECT warning_type FROM data_from_elasticsearch
WHERE warning_type = 'СщИвКФоЗвО';
--Seq Scan on data_from_elasticsearch  (cost=0.00..955.00 rows=1 width=21) (actual time=0.029..32.541 rows=1 loops=1)
--  Filter: ((warning_type)::text = 'СщИвКФоЗвО'::text)
--  Rows Removed by Filter: 39999
--Planning Time: 0.168 ms
--Execution Time: 32.567 ms

--с индексом по полю warning_type
CREATE INDEX IF NOT EXISTS data_from_elasticsearch_index ON data_from_elasticsearch (warning_type);

EXPLAIN (analyze) SELECT warning_type FROM data_from_elasticsearch
WHERE warning_type = 'СщИвКФоЗвО';
--Index Only Scan using data_from_elasticsearch_index on data_from_elasticsearch  (cost=0.41..4.43 rows=1 width=21) (actual --time=0.137..0.139 rows=1 loops=1)
--  Index Cond: (warning_type = 'СщИвКФоЗвО'::text)
--  Heap Fetches: 0
--Planning Time: 0.091 ms
--Execution Time: 0.159 ms
```
**Без индеса** ~ время поиска от **0.029..32.541**

**C индексом** ~ время поиска меньше, от **0.41..4.43**

P.S. без условия WHERE **индекс не исп.** при его наличии.

| Если сделать индексы так (два с разными именами на одну таблицу и на два разных поля):              |
| --------------------------------------------------------------------------------------------------- |
| CREATE INDEX IF NOT EXISTS data_from_elasticsearch_index ON data_from_elasticsearch (warning_type); |
| CREATE INDEX IF NOT EXISTS data_from_elasticsearch_host_index ON data_from_elasticsearch (host);    |

то

**index применяется только по полю host в выражении AND и по host,warning_type в выражении OR**
```sql
EXPLAIN SELECT warning_type, host FROM data_from_elasticsearch
WHERE warning_type = 'дтюспаБХрЛ'
AND host = 'ююеОЭБАГХд';

--Index Scan using data_from_elasticsearch_host_index on data_from_elasticsearch  (cost=0.41..8.43 rows=1 width=42)
--  Index Cond: ((host)::text = 'ююеОЭБАГХд'::text)
--  Filter: ((warning_type)::text = 'дтюспаБХрЛ'::text)

EXPLAIN SELECT warning_type, host FROM data_from_elasticsearch
WHERE warning_type = 'дтюспаБХрЛ'
OR host = 'ююеОЭБАГХд';

--Bitmap Heap Scan on data_from_elasticsearch  (cost=8.85..16.55 rows=2 width=42)
--  Recheck Cond: (((warning_type)::text = 'дтюспаБХрЛ'::text) OR ((host)::text = 'ююеОЭБАГХд'::text))
--  ->  BitmapOr  (cost=8.85..8.85 rows=2 width=0)
--        ->  Bitmap Index Scan on data_from_elasticsearch_index  (cost=0.00..4.42 rows=1 width=0)
--              Index Cond: ((warning_type)::text = 'дтюспаБХрЛ'::text)
--        ->  Bitmap Index Scan on data_from_elasticsearch_host_index  (cost=0.00..4.42 rows=1 width=0)
--              Index Cond: ((host)::text = 'ююеОЭБАГХд'::text)
```

| Если индекс сделатьтак(составной индекс):                                                                |
| -------------------------------------------------------------------------------------------------------- |
| CREATE INDEX IF NOT EXISTS data_from_elasticsearch_index ON data_from_elasticsearch (warning_type,host); |

то

**index применяется только по двум полям host,warning_type в выражении AND и вообще без индекса в выражении OR(но set enable_seqscan=off см. ниже)**
```sql
EXPLAIN SELECT warning_type, host FROM data_from_elasticsearch
WHERE warning_type = 'дтюспаБХрЛ'
AND host = 'ююеОЭБАГХд';

--Index Only Scan using data_from_elasticsearch_index on data_from_elasticsearch  (cost=0.41..4.43 rows=1 width=42)
--  Index Cond: ((warning_type = 'дтюспаБХрЛ'::text) AND (host = 'ююеОЭБАГХд'::text))

EXPLAIN SELECT warning_type, host FROM data_from_elasticsearch
WHERE warning_type = 'дтюспаБХрЛ'
OR host = 'ююеОЭБАГХд';

--Seq Scan on data_from_elasticsearch  (cost=0.00..1582.00 rows=2 width=42)
--  Filter: (((warning_type)::text = 'дтюспаБХрЛ'::text) OR ((host)::text = 'ююеОЭБАГХд'::text))

--НО, если отключить set enable_seqscan=off; то OR тоже начинает искать по обоим индесам
set enable_seqscan=off;

--Bitmap Heap Scan on data_from_elasticsearch  (cost=2446.84..2454.54 rows=2 width=42)
--  Recheck Cond: (((warning_type)::text = 'дтюспаБХрЛ'::text) OR ((host)::text = 'ююеОЭБАГХд'::text))
--  ->  BitmapOr  (cost=2446.84..2446.84 rows=2 width=0)
--        ->  Bitmap Index Scan on data_from_elasticsearch_index  (cost=0.00..4.42 rows=1 width=0)
--              Index Cond: ((warning_type)::text = 'дтюспаБХрЛ'::text)
--        ->  Bitmap Index Scan on data_from_elasticsearch_index  (cost=0.00..2442.41 rows=1 width=0)
--              Index Cond: ((host)::text = 'ююеОЭБАГХд'::text)

```


2. ### Индекс для полнотекстового поиска ###

```sql
EXPLAIN SELECT * FROM data_from_elasticsearch
WHERE to_tsvector('russian', warning_type) @@ to_tsquery('ИИЧХЭФМДЮЩ');

--Без индекса
--Seq Scan on data_from_elasticsearch  (cost=0.00..10478.00 rows=100 width=58)
--  Filter: (to_tsvector('russian'::regconfig, warning_type) @@ to_tsquery('ИИЧХЭФМДЮЩ'::text))

CREATE INDEX IF NOT EXISTS full_text_search ON data_from_elasticsearch
USING gin(to_tsvector('russian', warning_type));

--С индексом
--Bitmap Heap Scan on data_from_elasticsearch  (cost=13.03..246.04 rows=100 width=58)
--  Recheck Cond: (to_tsvector('russian'::regconfig, warning_type) @@ to_tsquery('ИИЧХЭФМДЮЩ'::text))
--  ->  Bitmap Index Scan on full_text_search  (cost=0.00..13.00 rows=100 width=0)
--        Index Cond: (to_tsvector('russian'::regconfig, warning_type) @@ to_tsquery('ИИЧХЭФМДЮЩ'::text))
```

3. ### Индекс на часть таблицы или индекс на поле с функцией ###

```sql
--Нет индекса
EXPLAIN SELECT * FROM data_from_elasticsearch
WHERE warning_type = 'дтюспаБХрЛ';

--Seq Scan on data_from_elasticsearch  (cost=0.00..478.00 rows=1 width=58)
--  Filter: ((warning_type)::text = 'дтюспаБХрЛ'::text)

--Есть индекс по полю warning_type = 'УЭфПиюВагс';
CREATE INDEX IF NOT EXISTS data_from_elasticsearch_index
ON data_from_elasticsearch (warning_type)
WHERE warning_type = 'УЭфПиюВагс';

--Index Scan using data_from_elasticsearch_index on data_from_elasticsearch  (cost=0.12..8.14 rows=1 width=58)

--Без функционального индекса
EXPLAIN SELECT * FROM data_from_elasticsearch
WHERE (warning_type || ' ' || host) = 'ЯХСрнЩтсИб уЖпвГаИИсф';
--Seq Scan on data_from_elasticsearch  (cost=0.00..578.00 rows=100 width=58)
--  Filter: ((((warning_type)::text || ' '::text) || (host)::text) = 'ЯХСрнЩтсИб уЖпвГаИИсф'::text)

--С функциональным индексом
CREATE INDEX IF NOT EXISTS functional_index
ON data_from_elasticsearch ((warning_type || ' ' || host));

--Bitmap Heap Scan on data_from_elasticsearch  (cost=5.19..188.70 rows=100 width=58)
--  Recheck Cond: ((((warning_type)::text || ' '::text) || (host)::text) = 'ЯХСрнЩтсИб уЖпвГаИИсф'::text)
--  ->  Bitmap Index Scan on functional_index  (cost=0.00..5.16 rows=100 width=0)
--        Index Cond: ((((warning_type)::text || ' '::text) || (host)::text) = 'ЯХСрнЩтсИб уЖпвГаИИсф'::text)

```



| Database   | ver |
| -----      | --- |
| PostgreSQL | 13.3|
