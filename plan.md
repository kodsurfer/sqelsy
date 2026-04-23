Вот примерный список правил (чек-лист), которые такой инструмент мог бы проверять:
1. Индексы и производительность

    Отсутствующие индексы на Foreign Keys: PostgreSQL не создает их автоматически, что приводит к медленным JOIN и блокировкам при удалении в родительских таблицах.
    Неиспользуемые индексы: Найти индексы с нулевым idx_scan (через pg_stat_user_indexes).
    Дублирующиеся индексы: Когда один индекс является префиксом другого (например, индекс по (a, b) делает индекс по (a) избыточным).

    Таблицы без Primary Key: Плохая практика, затрудняющая репликацию и уникальную идентификацию строк.
    Отсутствие Foreign Keys: Если имена колонок намекают на связь (например, user_id), но явного FK нет.
    Использование Enum vs Reference Tables: Проверка, что для часто меняющихся списков не используются жестко заданные типы ENUM.

    Использование UUID вместо TEXT: Проверка, что идентификаторы хранятся в эффективном бинарном формате.
    JSONB vs JSON: Рекомендация использовать JSONB для возможности индексации.
    Text/Varchar без лимитов: Предупреждение, если везде используется VARCHAR(255) без реальной необходимости (в Postgres TEXT часто лучше).
    Булевы флаги: Проверка на возможность использования TIMESTAMPTZ (например, is_deleted -> deleted_at), что дает больше информации.

4. Нейминг и стиль

    Регистрозависимые имена: Поиск таблиц или колонок в кавычках ("UserName"), что крайне неудобно в SQL.
    Плюрализация: Проверка единообразия имен таблиц (users vs user).
    Зарезервированные слова: Использование имен вроде order, user, group без кавычек.

Как это реализовать технически?
Лучше всего использовать системные каталоги (information_schema или pg_catalog). 
Пример запроса для поиска FK без индексов:
sql

SELECT
    rel.relname AS table_name,
    fk.conname AS foreign_key_name,
    con.attname AS column_name
FROM pg_constraint fk
JOIN pg_class rel ON rel.oid = fk.conrelid
JOIN pg_attribute con ON con.attrelid = rel.oid AND con.attnum = ANY(fk.conkey)
WHERE fk.contype = 'f'
AND NOT EXISTS (
    SELECT 1 FROM pg_index i 
    WHERE i.indrelid = rel.oid 
    AND (indkey::int2[])[0:cardinality(fk.conkey)-1] @> fk.conkey
);

Кстати, подобные инструменты уже начинают появляться (например, SchemaSpy для визуализации или sql-lint), но кастомный чекер под специфику может быть полезнее.


Для реализации CLI-инструмента на Python или Go вам понадобятся SQL-запросы к системным таблицам pg_catalog. Вот три базовых правила, с которых стоит начать:
1. Поиск Foreign Keys без индексов
Запрос для чекера:
sql

SELECT
    c.relname AS table_name,
    con.conname AS constraint_name,
    confrel.relname AS referenced_table
FROM pg_constraint con
JOIN pg_class c ON c.oid = con.conrelid
JOIN pg_class confrel ON confrel.oid = con.confrelid
WHERE con.contype = 'f'
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i
    WHERE i.indrelid = con.conrelid
      AND (i.indkey::int2[])[0:cardinality(con.conkey)-1] @> con.conkey
  );

Используйте код с осторожностью.
2. Поиск дублирующих или избыточных индексов
Часто разработчики создают индекс на (a, b) и отдельно на (a). Второй индекс в Postgres избыточен, так как первый может обслуживать запросы по префиксу.
Запрос для чекера:
sql

SELECT
    ind1.relname AS table_name,
    i1.relname AS redundant_index,
    i2.relname AS covering_index
FROM pg_index x1
JOIN pg_class ind1 ON ind1.oid = x1.indrelid
JOIN pg_class i1 ON i1.oid = x1.indexrelid
JOIN pg_index x2 ON x2.indrelid = x1.indrelid AND x1.indexrelid <> x2.indexrelid
JOIN pg_class i2 ON i2.oid = x2.indexrelid
WHERE (x2.indkey::int2[])[0:cardinality(x1.indkey)-1] = x1.indkey
  AND x1.indisunique IS FALSE; -- Уникальные индексы не трогаем

Используйте код с осторожностью.
3. Поиск неиспользуемых индексов
Индексы замедляют вставку и обновление. Если индекс не использовался с момента запуска БД, его стоит удалить.
sql

SELECT
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS scan_count
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indisunique IS FALSE -- Не предлагаем удалять PK или Unique
  AND schemaname = 'public';

Используйте код с осторожностью.
Архитектура CLI-инструмента
Чтобы инструмент был удобным, я рекомендую такую структуру вывода:

    Название проверки (например, [MISSING_FK_INDEX]).
    Объект: Таблица и колонка.
    Рекомендация: Готовый SQL-запрос CREATE INDEX ....
    Обоснование: Почему это важно (например, "Ускорит JOIN с таблицей X").
