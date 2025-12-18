# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*

   | QUERY PLAN |
   | :--- |
   | Bitmap Heap Scan on t\_books  \(cost=12.00..16.01 rows=1 width=33\) \(actual time=0.009..0.010 rows=0 loops=1\) |
   |   Recheck Cond: \(category IS NULL\) |
   |   -&gt;  Bitmap Index Scan on t\_books\_brin\_cat\_idx  \(cost=0.00..12.00 rows=1 width=0\) \(actual time=0.007..0.008 rows=0 loops=1\) |
   |         Index Cond: \(category IS NULL\) |
   | Planning Time: 0.280 ms |
   | Execution Time: 0.026 ms |

   
   *Объясните результат:*

   План использует **Bitmap Index Scan по BRIN-индексу**, который отбирает диапазоны страниц, где потенциально могут находиться `NULL` значения. Далее выполняется **Bitmap Heap Scan** с повторной проверкой условия (`Recheck Cond`), так как BRIN хранит агрегированную информацию по блокам, а не по строкам. Фактически подходящих строк не найдено, но индекс позволил быстро исключить большинство страниц.


6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*

   | QUERY PLAN |
   | :--- |
   | Bitmap Heap Scan on t\_books  \(cost=12.00..16.02 rows=1 width=33\) \(actual time=9.724..9.726 rows=0 loops=1\) |
   |   Recheck Cond: \(\(category\)::text = 'INDEX'::text\) |
   |   Rows Removed by Index Recheck: 150000 |
   |   Filter: \(\(author\)::text = 'SYSTEM'::text\) |
   |   Heap Blocks: lossy=1224 |
   |   -&gt;  Bitmap Index Scan on t\_books\_brin\_cat\_idx  \(cost=0.00..12.00 rows=1 width=0\) \(actual time=0.081..0.082 rows=12240 loops=1\) |
   |         Index Cond: \(\(category\)::text = 'INDEX'::text\) |
   | Planning Time: 0.208 ms |
   | Execution Time: 9.754 ms |

   
   *Объясните результат (обратите внимание на bitmap scan):*
   Используется **BRIN-индекс только по `category`**, поэтому по индексу отбираются все блоки, где категория может быть `'INDEX'`. Это приводит к формированию **lossy bitmap** (укрупнённые отметки блоков), из-за чего при чтении heap-таблицы происходит массовая повторная проверка строк (`Rows Removed by Index Recheck`). Условие по `author` применяется уже после чтения данных, что делает запрос относительно медленным.


8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*

   | QUERY PLAN |
   | :--- |
   | Sort  \(cost=3099.11..3099.12 rows=5 width=7\) \(actual time=22.216..22.218 rows=6 loops=1\) |
   |   Sort Key: category |
   |   Sort Method: quicksort  Memory: 25kB |
   |   -&gt;  HashAggregate  \(cost=3099.00..3099.05 rows=5 width=7\) \(actual time=22.180..22.182 rows=6 loops=1\) |
   |         Group Key: category |
   |         Batches: 1  Memory Usage: 24kB |
   |         -&gt;  Seq Scan on t\_books  \(cost=0.00..2724.00 rows=150000 width=7\) \(actual time=0.006..6.215 rows=150000 loops=1\) |
   | Planning Time: 0.079 ms |
   | Execution Time: 22.272 ms |

   
   *Объясните результат:*
   Выполняется **последовательное сканирование всей таблицы**, так как BRIN-индекс не подходит для операций `DISTINCT`. Уникальные значения формируются через **HashAggregate**, после чего результат сортируется. Для малой кардинальности поля (несколько категорий) это нормальное и ожидаемое поведение.


9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*

   | QUERY PLAN |
   | :--- |
   | Aggregate  \(cost=3099.03..3099.05 rows=1 width=8\) \(actual time=14.664..14.666 rows=1 loops=1\) |
   |   -&gt;  Seq Scan on t\_books  \(cost=0.00..3099.00 rows=14 width=0\) \(actual time=14.659..14.660 rows=0 loops=1\) |
   |         Filter: \(\(author\)::text \~\~ 'S%'::text\) |
   |         Rows Removed by Filter: 150000 |
   | Planning Time: 0.387 ms |
   | Execution Time: 14.691 ms |

   
   *Объясните результат:*
   Используется **Seq Scan**, поскольку BRIN-индекс по `author` неэффективен для префиксного поиска (`LIKE 'S%'`). Условие проверяется для каждой строки, и почти все строки отфильтровываются. Для такого запроса больше подошёл бы B-tree индекс.


10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*

   | QUERY PLAN |
   | :--- |
   | Aggregate  \(cost=3475.88..3475.89 rows=1 width=8\) \(actual time=26.425..26.427 rows=1 loops=1\) |
   |   -&gt;  Seq Scan on t\_books  \(cost=0.00..3474.00 rows=750 width=0\) \(actual time=26.417..26.419 rows=1 loops=1\) |
   |         Filter: \(lower\(\(title\)::text\) \~\~ 'o%'::text\) |
   |         Rows Removed by Filter: 149999 |
   | Planning Time: 0.292 ms |
   | Execution Time: 26.446 ms |

   
   *Объясните результат:*
   Несмотря на наличие функционального индекса `LOWER(title)`, планировщик выбирает **последовательное сканирование**. Причина — крайне низкая селективность (найдена всего одна строка) и относительно высокая стоимость использования индекса по сравнению с полным сканированием небольшой таблицы. PostgreSQL считает Seq Scan более выгодным.


12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*

   | QUERY PLAN |
   | :--- |
   | Bitmap Heap Scan on t\_books  \(cost=12.00..16.02 rows=1 width=33\) \(actual time=0.615..0.616 rows=0 loops=1\) |
   |   Recheck Cond: \(\(\(category\)::text = 'INDEX'::text\) AND \(\(author\)::text = 'SYSTEM'::text\)\) |
   |   Rows Removed by Index Recheck: 8811 |
   |   Heap Blocks: lossy=72 |
   |   -&gt;  Bitmap Index Scan on t\_books\_brin\_cat\_auth\_idx  \(cost=0.00..12.00 rows=1 width=0\) \(actual time=0.022..0.022 rows=720 loops=1\) |
   |         Index Cond: \(\(\(category\)::text = 'INDEX'::text\) AND \(\(author\)::text = 'SYSTEM'::text\)\) |
   | Planning Time: 0.213 ms |
   | Execution Time: 0.638 ms |

   
   *Объясните результат:*
   Составной BRIN-индекс значительно сузил количество подходящих блоков: bitmap стал намного компактнее (`lossy=72` вместо 1224). Это резко сократило объём повторных проверок и время выполнения запроса. Условия по `category` и `author` теперь применяются на уровне индекса, что демонстрирует преимущество составного BRIN-индекса для коррелированных данных.
