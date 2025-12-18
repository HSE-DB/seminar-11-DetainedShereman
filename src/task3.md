## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*

    | QUERY PLAN |
    | :--- |
    | Bitmap Heap Scan on test\_cluster  \(cost=59.17..7696.73 rows=5000 width=68\) \(actual time=10.652..88.745 rows=500201 loops=1\) |
    |   Recheck Cond: \(category = 'A'::text\) |
    |   Heap Blocks: exact=8334 |
    |   -&gt;  Bitmap Index Scan on test\_cluster\_cat\_idx  \(cost=0.00..57.92 rows=5000 width=0\) \(actual time=9.622..9.623 rows=500201 loops=1\) |
    |         Index Cond: \(category = 'A'::text\) |
    | Planning Time: 0.240 ms |
    | Execution Time: 100.866 ms |

    
    *Объясните результат:*

    Используется **Bitmap Index Scan** по индексу `category`, так как условие возвращает около 50% строк таблицы. Индекс формирует bitmap из большого числа TID, после чего выполняется **Bitmap Heap Scan** с чтением множества разрозненных страниц (`Heap Blocks: exact=8334`).

    Так как строки с `category = 'A'` физически распределены по всей таблице случайным образом, чтение сопровождается большим количеством I/O, что и приводит к высокому времени выполнения (~100 мс).


4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*

        [2025-12-18 14:59:20] workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
        [2025-12-18 14:59:21] completed in 656 ms

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*

    | QUERY PLAN |
    | :--- |
    | Bitmap Heap Scan on test\_cluster  \(cost=5574.13..20156.04 rows=499833 width=39\) \(actual time=7.658..44.866 rows=500201 loops=1\) |
    |   Recheck Cond: \(category = 'A'::text\) |
    |   Heap Blocks: exact=4169 |
    |   -&gt;  Bitmap Index Scan on test\_cluster\_cat\_idx  \(cost=0.00..5449.17 rows=499833 width=0\) \(actual time=7.166..7.168 rows=500201 loops=1\) |
    |         Index Cond: \(category = 'A'::text\) |
    | Planning Time: 0.292 ms |
    | Execution Time: 57.375 ms |

    
    *Объясните результат:*

    После выполнения `CLUSTER` строки с одинаковым значением `category` размещены **физически подряд**. В результате при том же плане выполнения:

    * количество читаемых heap-блоков сократилось почти вдвое (`4169` вместо `8334`);
    * чтение данных стало более последовательным.

    Это значительно уменьшило затраты на I/O и сократило общее время выполнения запроса до ~57 мс, несмотря на то, что возвращается то же количество строк.


6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*


    | Параметр       | До кластеризации | После кластеризации |
    | -------------- | ---------------- | ------------------- |
    | Heap Blocks    | 8334             | 4169                |
    | Execution Time | ~101 ms          | ~57 ms              |
    | План           | Bitmap Heap Scan | Bitmap Heap Scan    |

    Кластеризация **не изменила план выполнения**, но существенно улучшила **локальность данных**, что уменьшило количество чтений страниц и ускорило выполнение запроса почти в 2 раза.
    Это наглядно демонстрирует, что `CLUSTER` эффективен для запросов, регулярно фильтрующих данные по одному и тому же признаку с низкой селективностью.
