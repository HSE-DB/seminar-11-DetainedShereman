## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*

     | QUERY PLAN |
     | :--- |
     | Bitmap Heap Scan on t\_books  \(cost=21.03..1336.08 rows=750 width=33\) \(actual time=0.029..0.030 rows=1 loops=1\) |
     |   Recheck Cond: \(to\_tsvector\('english'::regconfig, \(title\)::text\) @@ '''expert'''::tsquery\) |
     |   Heap Blocks: exact=1 |
     |   -&gt;  Bitmap Index Scan on t\_books\_fts\_idx  \(cost=0.00..20.84 rows=750 width=0\) \(actual time=0.013..0.014 rows=1 loops=1\) |
     |         Index Cond: \(to\_tsvector\('english'::regconfig, \(title\)::text\) @@ '''expert'''::tsquery\) |
     | Planning Time: 1.086 ms |
     | Execution Time: 0.141 ms |

    
    *Объясните результат:*
    
    Запрос использует **GIN-индекс по `to_tsvector`**, что позволяет быстро найти документы, содержащие нужный лексем. Выполняется **Bitmap Index Scan** для получения подходящих TID, после чего **Bitmap Heap Scan** извлекает строки из таблицы с повторной проверкой условия (`Recheck Cond`). Найдена одна строка, индекс полностью оправдан и обеспечивает минимальное время выполнения.


6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*

     | QUERY PLAN |
     | :--- |
     | Index Scan using t\_lookup\_pk on t\_lookup  \(cost=0.42..8.44 rows=1 width=23\) \(actual time=0.016..0.016 rows=1 loops=1\) |
     |   Index Cond: \(\(item\_key\)::text = '0000000455'::text\) |
     | Planning Time: 0.185 ms |
     | Execution Time: 0.032 ms |

     
     *Объясните результат:*
     Используется **Index Scan по B-tree первичному ключу**, что является оптимальным планом для точечного поиска по уникальному значению. Индекс сразу указывает на нужную строку, чтение минимально, время выполнения крайне низкое.


14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*

     | QUERY PLAN |
     | :--- |
     | Index Scan using t\_lookup\_clustered\_pkey on t\_lookup\_clustered  \(cost=0.42..8.44 rows=1 width=23\) \(actual time=0.063..0.064 rows=1 loops=1\) |
     |   Index Cond: \(\(item\_key\)::text = '0000000455'::text\) |
     | Planning Time: 0.153 ms |
     | Execution Time: 0.081 ms |

     
     *Объясните результат:*
     План выполнения идентичен — **Index Scan по первичному ключу**. Кластеризация не даёт преимущества для одиночного точечного поиска, так как доступ к строке всё равно осуществляется через индекс. Незначительное отличие во времени связано с кэшированием и физическим расположением страниц.


15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*

     | QUERY PLAN |
     | :--- |
     | Index Scan using t\_lookup\_value\_idx on t\_lookup  \(cost=0.42..8.44 rows=1 width=23\) \(actual time=0.019..0.019 rows=0 loops=1\) |
     |   Index Cond: \(\(item\_value\)::text = 'T\_BOOKS'::text\) |
     | Planning Time: 0.238 ms |
     | Execution Time: 0.037 ms |

     
     *Объясните результат:*
     Используется **Index Scan по вторичному B-tree индексу**. Значение `'T_BOOKS'` отсутствует в таблице, поэтому индекс быстро подтверждает отсутствие строк без необходимости сканировать таблицу целиком. Это демонстрирует эффективность индекса даже при нулевом результате.


18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*

     | QUERY PLAN |
     | :--- |
     | Index Scan using t\_lookup\_clustered\_value\_idx on t\_lookup\_clustered  \(cost=0.42..8.44 rows=1 width=23\) \(actual time=0.020..0.020 rows=0 loops=1\) |
     |   Index Cond: \(\(item\_value\)::text = 'T\_BOOKS'::text\) |
     | Planning Time: 0.267 ms |
     | Execution Time: 0.038 ms |

     
     *Объясните результат:*
     
     План аналогичен предыдущему: **Index Scan по индексу `item_value`**. Кластеризация по первичному ключу не влияет на производительность поиска по другому столбцу. Отсутствие результата подтверждается быстро и с минимальными затратами.


19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Для поиска по `item_value` **разницы в производительности между обычной и кластеризованной таблицами практически нет**. В обоих случаях используется B-tree индекс, который напрямую указывает на строки (или подтверждает их отсутствие).
     
     Кластеризация влияет преимущественно на **диапазонные и последовательные чтения по кластерному ключу**, но не даёт преимуществ для точечных запросов по вторичным индексам.
