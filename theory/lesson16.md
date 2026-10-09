
# Пользовательские функции в SQL

## Превью / аннотация / краткое содержание

Занятие посвящено пользовательским функциям в реляционных СУБД: их назначению, созданию, вызову, параметрам, возвращаемым значениям и характеристикам выполнения.

Пользовательские функции позволяют оформлять повторно используемые вычисления и операции с данными в самостоятельные объекты базы данных.

При этом функции разных СУБД существенно различаются. PostgreSQL позволяет создавать скалярные и табличные функции на SQL и PL/pgSQL, SQL Server поддерживает скалярные и табличные UDF с определёнными ограничениями, MySQL использует хранимые функции со скалярным результатом, а Oracle предоставляет функции PL/SQL и механизмы возврата коллекций и табличных результатов.

На занятии рассматриваются:

- назначение пользовательских функций;
- отличия функций от процедур;
- скалярные и табличные функции;
- создание функций в PostgreSQL, SQL Server, MySQL и Oracle;
- входные, выходные и комбинированные параметры;
- позиционные и именованные вызовы;
- значения параметров по умолчанию;
- `RETURNS`, `RETURNS TABLE`, `SETOF`, `RECORD` и `VOID`;
- характеристики `IMMUTABLE`, `STABLE`, `VOLATILE`;
- параметры параллельного выполнения;
- безопасность `SECURITY INVOKER` и `SECURITY DEFINER`;
- практическое применение функций для логирования и расчётов;
- производительность и ограничения пользовательских функций.

Основная практика выполняется в PostgreSQL с использованием SQL и PL/pgSQL.

## Основные понятия

**Пользовательская функция (User-Defined Function, UDF)** — именованный программный объект базы данных, который принимает аргументы, выполняет заданную логику и возвращает результат.

**Скалярная функция** — функция, возвращающая одно значение: число, строку, дату, логическое значение, JSON или другой скалярный тип.

**Табличная функция (Table-Valued Function)** — функция, возвращающая набор строк с определённой структурой.

**CREATE FUNCTION** — команда создания пользовательской функции.

**CREATE OR REPLACE FUNCTION** — команда создания или замены определения функции в поддерживающих её СУБД.

**CREATE OR ALTER FUNCTION** — синтаксис SQL Server для создания или изменения функции.

**Параметр IN** — входной параметр функции.

**Параметр OUT** — параметр, используемый для формирования результата функции в поддерживающих его СУБД.

**Параметр INOUT** — параметр, который одновременно принимает исходное значение и участвует в формировании возвращаемого результата.

**RETURNS** — описание возвращаемого типа функции.

**RETURNS TABLE** — форма объявления табличного результата функции.

**RETURNS SETOF** — конструкция PostgreSQL для функций, возвращающих множество значений указанного типа.

**RECORD** — тип PostgreSQL для записи, структура которой определяется контекстом.

**SETOF RECORD** — множество записей, структура которых может задаваться при вызове функции.

**VOID** — тип PostgreSQL, обозначающий отсутствие содержательного возвращаемого значения.

**RETURN** — оператор завершения выполнения функции с возвратом результата либо завершения процедуры в зависимости от языка и контекста.

**RETURN NEXT** — оператор PL/pgSQL, добавляющий строку в результат функции, возвращающей множество.

**RETURN QUERY** — оператор PL/pgSQL, добавляющий результат SQL-запроса в набор возвращаемых строк.

**Позиционный вызов** — передача аргументов в порядке их объявления.

**Именованный вызов** — передача аргументов с указанием имён параметров.

**DEFAULT** — значение, используемое, если соответствующий аргумент не передан.

**PL/pgSQL** — процедурный язык PostgreSQL.

**T-SQL** — расширение SQL для Microsoft SQL Server.

**PL/SQL** — процедурный язык Oracle.

**IMMUTABLE** — характеристика функции PostgreSQL, означающая независимость результата от изменяемого состояния при одинаковых аргументах.

**STABLE** — характеристика функции PostgreSQL, допускающая зависимость от состояния базы в пределах согласованного представления данных одного SQL-оператора.

**VOLATILE** — характеристика функции PostgreSQL, указывающая, что результат или побочные эффекты могут изменяться между вызовами.

**PARALLEL SAFE** — функция, которую разрешено выполнять в параллельных рабочих процессах при соблюдении соответствующих требований.

**PARALLEL RESTRICTED** — функция, которая в параллельном плане должна выполняться в ведущем процессе.

**PARALLEL UNSAFE** — функция, наличие которой требует отказа от параллельного плана соответствующего запроса.

**SECURITY INVOKER** — режим выполнения функции с правами вызывающего пользователя.

**SECURITY DEFINER** — режим выполнения функции с правами её владельца.

**STRICT / RETURNS NULL ON NULL INPUT** — характеристика функции PostgreSQL, при которой функция не вызывается при наличии `NULL` среди входных аргументов и возвращает SQL `NULL`.

**LEAKPROOF** — специальная характеристика функции PostgreSQL, связанная с безопасностью раскрытия данных через её поведение; предназначена для доверенных функций.

**Инициализация функции** — подготовка параметров и локальных переменных при вызове функции.

**Инкапсуляция** — выделение реализации некоторой логики в самостоятельный объект с определённым интерфейсом.

**Побочный эффект (Side Effect)** — изменение состояния вне возвращаемого значения, например запись в таблицу или получение значения последовательности.

**Inlining** — оптимизация, при которой тело подходящей функции включается в план вызывающего запроса.

## Основная часть

### Тема 1. Пользовательская функция как объект базы данных

В SQL часто встречаются повторяющиеся вычисления.

Например, приложение рассчитывает цену товара с учётом скидки:

```sql
SELECT
    product_id,
    price,
    discount_percent,
    ROUND(
        price * (1 - discount_percent / 100.0),
        2
    ) AS final_price
FROM products;
```

Если аналогичное выражение используется в нескольких запросах, его можно вынести в пользовательскую функцию.

#### Создание простой функции PostgreSQL

```sql
CREATE OR REPLACE FUNCTION calculate_discount_price(
    p_price NUMERIC,
    p_discount_percent NUMERIC
)
RETURNS NUMERIC
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT ROUND(
        p_price * (1 - p_discount_percent / 100),
        2
    );
$$;
```

Вызов:

```sql
SELECT calculate_discount_price(1500, 10);
```

Результат:

```text
1350.00
```

Теперь функцию можно использовать в SQL-запросах:

```sql
SELECT
    product_id,
    calculate_discount_price(
        price,
        discount_percent
    ) AS final_price
FROM products;
```

#### Зачем нужны пользовательские функции

Основные задачи:

- устранение повторяющихся вычислений;
- преобразование значений;
- реализация проверок;
- нормализация данных;
- формирование табличных результатов;
- выполнение логики, тесно связанной с данными;
- предоставление контролируемого интерфейса доступа;
- организация служебных операций.

Функция создаётся как объект базы данных и может использоваться разными клиентами.

Однако создание функции само по себе не делает запрос быстрее.

В некоторых случаях вызов функции создаёт дополнительные накладные расходы или ограничивает оптимизацию.

Поэтому функции следует рассматривать прежде всего как инструмент организации логики, а не как универсальный способ повышения производительности.

#### Функция и процедура

Функции и процедуры — разные программные объекты.

В PostgreSQL:

```sql
SELECT function_name(...);
```

используется для вызова функции.

А:

```sql
CALL procedure_name(...);
```

для вызова процедуры.

Функция обязательно имеет объявленный тип результата.

Процедура может использовать выходные параметры, но не применяется как обычное скалярное выражение внутри `SELECT`.

Функции PostgreSQL не могут самостоятельно выполнять `COMMIT` и `ROLLBACK`.

Процедуры могут управлять транзакциями при выполнении специальных условий, рассмотренных на предыдущем занятии.

В других СУБД различия также существуют, но конкретные возможности зависят от реализации.

### Тема 2. Типология пользовательских функций

Функции различаются прежде всего формой возвращаемого результата.

#### Скалярные функции

Скалярная функция возвращает одно значение.

Например:

```sql
CREATE OR REPLACE FUNCTION square_number(
    p_value NUMERIC
)
RETURNS NUMERIC
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT p_value * p_value;
$$;
```

Вызов:

```sql
SELECT square_number(5);
```

Результат:

```text
25
```

Скалярная функция может использоваться в выражениях:

```sql
SELECT square_number(10);
```

```sql
SELECT
    product_id,
    square_number(quantity)
FROM products;
```

```sql
SELECT *
FROM products
WHERE square_number(quantity) > 100;
```

Однако последний пример не обязательно эффективно использует обычный индекс по `quantity`.

Оборачивание индексируемого столбца в функцию может затруднить индексный поиск.

Для подходящих функций PostgreSQL поддерживает индексы по выражениям, но это не означает, что индекс следует создавать для любого вычисления.

#### Табличные функции

Табличная функция возвращает множество строк.

Например, создадим функцию генерации чисел:

```sql
CREATE OR REPLACE FUNCTION generate_even_numbers(
    p_limit INTEGER
)
RETURNS TABLE (
    number_value INTEGER
)
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT n
    FROM generate_series(1, p_limit) AS n
    WHERE n % 2 = 0;
$$;
```

Вызов:

```sql
SELECT *
FROM generate_even_numbers(10);
```

Результат:

| number_value |
|---:|
| 2 |
| 4 |
| 6 |
| 8 |
| 10 |

Табличную функцию можно использовать в `FROM`:

```sql
SELECT number_value
FROM generate_even_numbers(20)
WHERE number_value > 10;
```

Она может участвовать в соединениях и дальнейшей SQL-обработке.

#### Функции, возвращающие множество скалярных значений

В PostgreSQL можно использовать `SETOF`.

```sql
CREATE OR REPLACE FUNCTION generate_numbers(
    p_start INTEGER,
    p_end INTEGER
)
RETURNS SETOF INTEGER
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT generate_series(p_start, p_end);
$$;
```

Вызов:

```sql
SELECT *
FROM generate_numbers(1, 5);
```

Результат:

```text
1
2
3
4
5
```

`SETOF INTEGER` означает, что функция возвращает множество значений типа `INTEGER`.

Это не обязательно набор с несколькими столбцами.

#### Функции без содержательного результата

PostgreSQL поддерживает `RETURNS void`.

Например:

```sql
CREATE TABLE function_log (
    log_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    message TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

```sql
CREATE OR REPLACE FUNCTION write_log(
    p_message TEXT
)
RETURNS void
LANGUAGE plpgsql
VOLATILE
AS $$
BEGIN
    INSERT INTO function_log (message)
    VALUES (p_message);
END;
$$;
```

Вызов:

```sql
SELECT write_log('Пользователь выполнил действие');
```

Функция не возвращает содержательного результата, но изменяет данные.

Это пример функции с побочным эффектом.

В PostgreSQL такие функции допустимы, но их необходимо объявлять `VOLATILE`.

Также важно учитывать, что запись в журнал выполняется в транзакционном контексте вызывающего запроса.

Если транзакция откатится, запись в таблице `function_log` также будет отменена.

### Тема 3. Создание функций в PostgreSQL и других СУБД

#### PostgreSQL

Общая форма:

```sql
CREATE [ OR REPLACE ] FUNCTION function_name(
    parameters
)
RETURNS result_type
LANGUAGE language_name
AS $$
    function_body
$$;
```

Дополнительные характеристики определяют:

- изменчивость функции;
- поведение при `NULL`;
- параллельную безопасность;
- режим проверки прав;
- стоимость выполнения;
- ожидаемое количество возвращаемых строк.

Например:

```sql
CREATE OR REPLACE FUNCTION add_numbers(
    p_a INTEGER,
    p_b INTEGER
)
RETURNS INTEGER
LANGUAGE plpgsql
IMMUTABLE
PARALLEL SAFE
AS $$
BEGIN
    RETURN p_a + p_b;
END;
$$;
```

Вызов:

```sql
SELECT add_numbers(10, 20);
```

Результат:

```text
30
```

Для простых выражений не всегда нужен PL/pgSQL.

Можно использовать `LANGUAGE sql`:

```sql
CREATE OR REPLACE FUNCTION add_numbers_sql(
    p_a INTEGER,
    p_b INTEGER
)
RETURNS INTEGER
LANGUAGE sql
IMMUTABLE
PARALLEL SAFE
AS $$
    SELECT p_a + p_b;
$$;
```

SQL-функция часто проще, если вся её логика выражается одним запросом.

#### CREATE OR REPLACE и сигнатура

PostgreSQL идентифицирует перегруженные функции по имени, схеме и типам входных аргументов.

Например, могут существовать:

```sql
calculate_value(INTEGER)
```

и:

```sql
calculate_value(NUMERIC)
```

Это разные перегрузки.

`CREATE OR REPLACE FUNCTION` заменяет определение подходящей существующей функции, но не позволяет произвольно изменить тип её возвращаемого значения.

Для изменения интерфейса иногда требуется создать новую перегрузку или удалить старую функцию с учётом зависимостей.

Использовать `DROP FUNCTION ... CASCADE` без анализа зависимых объектов опасно.

#### SQL Server

В Microsoft SQL Server применяется:

```sql
CREATE OR ALTER FUNCTION
```

Пример:

```sql
CREATE OR ALTER FUNCTION dbo.add_numbers(
    @a INT,
    @b INT
)
RETURNS INT
AS
BEGIN
    RETURN @a + @b;
END;
GO
```

Вызов:

```sql
SELECT dbo.add_numbers(10, 20);
```

`GO` — разделитель пакетов в некоторых SQL-клиентах Microsoft, а не команда T-SQL, выполняемая сервером.

SQL Server поддерживает:

- скалярные функции;
- inline table-valued functions;
- multi-statement table-valued functions.

Однако пользовательские T-SQL-функции имеют ограничения на побочные эффекты.

Например, обычная T-SQL UDF не может изменять произвольные постоянные пользовательские таблицы так же, как функция PostgreSQL с `VOLATILE`.

Для операций изменения состояния обычно применяются хранимые процедуры.

#### MySQL/InnoDB

В MySQL пользовательская хранимая функция обычно возвращает одно значение.

Пример:

```sql
DELIMITER //

CREATE FUNCTION add_numbers(
    p_a INT,
    p_b INT
)
RETURNS INT
DETERMINISTIC
NO SQL
BEGIN
    RETURN p_a + p_b;
END//

DELIMITER ;
```

Вызов:

```sql
SELECT add_numbers(10, 20);
```

`DELIMITER` — команда клиента MySQL.

Важно учитывать, что характеристики `DETERMINISTIC`, `NO SQL`, `READS SQL DATA` и другие являются декларациями о свойствах функции.

MySQL не доказывает автоматически, что функция действительно детерминированная.

Обычная хранимая функция MySQL не возвращает произвольный набор строк как табличная функция PostgreSQL.

Для табличных результатов используются другие механизмы, включая запросы, представления, процедуры и встроенные табличные функции.

#### Oracle

В Oracle функции часто пишутся на PL/SQL.

```sql
CREATE OR REPLACE FUNCTION add_numbers(
    p_a IN NUMBER,
    p_b IN NUMBER
)
RETURN NUMBER
IS
BEGIN
    RETURN p_a + p_b;
END;
/
```

Вызов:

```sql
SELECT add_numbers(10, 20)
FROM dual;
```

Oracle поддерживает функции, возвращающие скалярные значения, коллекции и другие типы.

Для возврата набора строк могут применяться pipelined table functions и другие механизмы.

Но они не являются синтаксически идентичными `RETURNS TABLE` PostgreSQL.

#### Сравнение

| Возможность | PostgreSQL | SQL Server | MySQL/InnoDB | Oracle |
|---|---|---|---|---|
| Скалярные функции | Да | Да | Да | Да |
| Пользовательские табличные функции | Да | Да | Нет прямого аналога в stored functions | Да, через специальные механизмы |
| SQL внутри функции | Да | С ограничениями | С ограничениями | Да, с ограничениями вызова |
| Изменение таблиц из функции | Допустимо при соответствующих свойствах | Обычным T-SQL UDF запрещено | Допустимо с ограничениями | Возможно, но имеются ограничения при вызове из SQL |
| Транзакционный COMMIT внутри обычной функции | Нет | Нет | Нет | Обычно недопустим в функции, вызываемой из SQL; есть специальные автономные транзакции |
| Перегрузка функций | Да | Нет обычной перегрузки T-SQL UDF по типам | Нет | Да, в соответствующих контекстах |

### Тема 4. Параметры функций

Параметры определяют интерфейс функции.

В PostgreSQL поддерживаются:

- `IN`;
- `OUT`;
- `INOUT`;
- `VARIADIC`.

#### IN

`IN` — входной параметр.

```sql
CREATE OR REPLACE FUNCTION greet_student(
    IN p_name TEXT
)
RETURNS TEXT
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT 'Здравствуйте, ' || p_name;
$$;
```

Вызов:

```sql
SELECT greet_student('Анна');
```

Результат:

```text
Здравствуйте, Анна
```

Если не указать направление параметра, PostgreSQL по умолчанию использует `IN`.

#### OUT

`OUT` используется для формирования результата.

```sql
CREATE OR REPLACE FUNCTION calculate_rectangle(
    IN p_width NUMERIC,
    IN p_height NUMERIC,
    OUT area NUMERIC,
    OUT perimeter NUMERIC
)
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    area := p_width * p_height;
    perimeter := 2 * (p_width + p_height);
END;
$$;
```

Вызов:

```sql
SELECT *
FROM calculate_rectangle(5, 3);
```

Результат:

| area | perimeter |
|---:|---:|
| 15 | 16 |

При нескольких `OUT`-параметрах PostgreSQL формирует составной результат.

Не требуется возвращать его отдельным выражением `RETURN area, perimeter`.

Значения задаются присваиванием выходным параметрам.

#### INOUT

Параметр `INOUT` получает входное значение и становится частью результата.

```sql
CREATE OR REPLACE FUNCTION increase_counter(
    INOUT p_counter INTEGER
)
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    p_counter := p_counter + 1;
END;
$$;
```

Вызов:

```sql
SELECT increase_counter(10);
```

Результат:

```text
11
```

#### Различия с процедурами

В MySQL хранимые функции принимают только входные параметры.

`OUT` и `INOUT` допускаются у процедур, но не у обычных хранимых функций.

В SQL Server функции также не предоставляют модель выходных параметров, аналогичную процедурам с `OUTPUT`.

В Oracle OUT/IN OUT возможны в PL/SQL-функциях, но для функций, вызываемых из SQL-выражений, действуют дополнительные ограничения. Поэтому такие интерфейсы чаще встречаются у процедур.

#### Значения по умолчанию

В PostgreSQL можно объявить:

```sql
CREATE OR REPLACE FUNCTION calculate_price(
    p_price NUMERIC,
    p_discount NUMERIC DEFAULT 0
)
RETURNS NUMERIC
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT ROUND(
        p_price * (1 - p_discount / 100),
        2
    );
$$;
```

Вызов с двумя аргументами:

```sql
SELECT calculate_price(1000, 10);
```

Результат:

```text
900.00
```

Вызов с одним аргументом:

```sql
SELECT calculate_price(1000);
```

Результат:

```text
1000.00
```

В PostgreSQL после входного параметра, имеющего значение по умолчанию, последующие входные параметры также должны иметь значения по умолчанию.

Это необходимо учитывать при проектировании интерфейса функции.

#### Позиционный вызов

```sql
SELECT calculate_price(1000, 10);
```

Порядок аргументов соответствует порядку параметров.

#### Именованный вызов

```sql
SELECT calculate_price(
    p_discount => 10,
    p_price => 1000
);
```

Именованный вызов повышает читаемость, особенно когда функция имеет много аргументов одинаковых типов.

Однако изменение имени параметра может повлиять на вызывающий код, использующий именованные аргументы.

Поэтому имена параметров также следует рассматривать как часть публичного интерфейса.

### Тема 5. Типы возвращаемого результата

Тип результата определяет, как функцию можно использовать в SQL.

#### Простой скалярный результат

```sql
RETURNS INTEGER
```

```sql
RETURNS NUMERIC
```

```sql
RETURNS TEXT
```

```sql
RETURNS BOOLEAN
```

```sql
RETURNS JSONB
```

Функция может возвращать и составной пользовательский тип.

Но составной результат не следует автоматически считать табличной функцией, возвращающей несколько строк.

#### RETURNS TABLE

Пример:

```sql
CREATE OR REPLACE FUNCTION get_expensive_products(
    p_min_price NUMERIC
)
RETURNS TABLE (
    product_id BIGINT,
    product_name TEXT,
    price NUMERIC
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT
        p.product_id,
        p.product_name,
        p.price
    FROM products AS p
    WHERE p.price >= p_min_price;
END;
$$;
```

Здесь предполагается наличие соответствующей таблицы `products`.

Вызов:

```sql
SELECT *
FROM get_expensive_products(1000);
```

Структура результата известна при создании функции.

Это позволяет:

- использовать имена столбцов;
- соединять результат с таблицами;
- фильтровать значения;
- применять сортировку и агрегацию.

#### RETURNS SETOF

`SETOF` означает множество значений заданного типа.

```sql
CREATE OR REPLACE FUNCTION get_positive_numbers(
    p_limit INTEGER
)
RETURNS SETOF INTEGER
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT generate_series(1, p_limit);
$$;
```

Вызов:

```sql
SELECT *
FROM get_positive_numbers(5);
```

#### RETURN NEXT

В PL/pgSQL можно формировать результат построчно.

```sql
CREATE OR REPLACE FUNCTION get_numbers_loop(
    p_limit INTEGER
)
RETURNS SETOF INTEGER
LANGUAGE plpgsql
IMMUTABLE
AS $$
DECLARE
    i INTEGER;
BEGIN
    FOR i IN 1..p_limit LOOP
        RETURN NEXT i;
    END LOOP;

    RETURN;
END;
$$;
```

Вызов:

```sql
SELECT *
FROM get_numbers_loop(5);
```

Результат:

```text
1
2
3
4
5
```

Важно: `RETURN NEXT` добавляет значение в формируемый результат, но не завершает выполнение функции.

После цикла используется `RETURN`, завершающий функцию.

В текущей реализации PL/pgSQL результаты функций, возвращающих множество, могут накапливаться до передачи вызывающему запросу.

Поэтому `RETURN NEXT` не следует автоматически воспринимать как механизм потоковой передачи каждой строки клиенту.

#### RETURN QUERY

Если набор строк уже можно получить SQL-запросом, предпочтительнее использовать `RETURN QUERY`.

```sql
CREATE OR REPLACE FUNCTION get_numbers_query(
    p_limit INTEGER
)
RETURNS SETOF INTEGER
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT generate_series(1, p_limit);
END;
$$;
```

В этом примере `RETURN QUERY` добавляет результаты запроса в возвращаемый набор.

При этом исполнение PL/pgSQL-функции не обязательно прекращается сразу после `RETURN QUERY`.

#### SETOF RECORD

Для результата с описываемой при вызове структурой используется `SETOF RECORD`.

```sql
CREATE OR REPLACE FUNCTION get_demo_record()
RETURNS SETOF RECORD
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT
        1::INTEGER AS id,
        'SQL'::TEXT AS title;
$$;
```

Вызов:

```sql
SELECT *
FROM get_demo_record() AS t(
    id INTEGER,
    title TEXT
);
```

В данном случае вызывающая сторона должна указать структуру записи.

Но `SETOF RECORD` не означает, что одна функция может без ограничений возвращать произвольную структуру, не согласованную с ожидаемым описанием результата.

Структура фактически возвращаемых значений должна соответствовать объявленной при вызове.

#### RETURNS TABLE или SETOF RECORD

Если результат стабилен и имеет известные столбцы, чаще удобнее `RETURNS TABLE`.

Если требуется универсальный интерфейс с описанием структуры при вызове, можно использовать `SETOF RECORD`.

Однако нет общего правила, согласно которому `RETURNS TABLE` обязательно выполняется быстрее.

Фактическая стоимость зависит от тела функции, способа её вызова, возможности оптимизации и объёма данных.

#### VOID

Тип `VOID` используется, когда функция не возвращает содержательного значения.

```sql
CREATE OR REPLACE FUNCTION print_message(
    p_message TEXT
)
RETURNS void
LANGUAGE plpgsql
VOLATILE
AS $$
BEGIN
    RAISE NOTICE '%', p_message;
END;
$$;
```

Вызов:

```sql
SELECT print_message('Тестовое сообщение');
```

Важно: `VOID` не является синонимом полностью отсутствующего результата вызова в SQL-протоколе.

Функция имеет возвращаемый тип `void`, но не предоставляет прикладного значения.

### Тема 6. Характеристики выполнения функций PostgreSQL

PostgreSQL позволяет задавать функции свойства, влияющие на оптимизатор.

Основные:

- `IMMUTABLE`;
- `STABLE`;
- `VOLATILE`.

Эти характеристики необходимо указывать в соответствии с фактическим поведением функции.

#### IMMUTABLE

`IMMUTABLE` означает, что результат функции полностью определяется входными аргументами.

Пример:

```sql
CREATE OR REPLACE FUNCTION cube_number(
    p_value NUMERIC
)
RETURNS NUMERIC
LANGUAGE sql
IMMUTABLE
PARALLEL SAFE
AS $$
    SELECT p_value * p_value * p_value;
$$;
```

При одинаковом аргументе результат не зависит от:

- времени;
- состояния таблиц;
- пользователя;
- содержимого сессии;
- изменяемых настроек, влияющих на вычисление.

Это позволяет оптимизатору применять дополнительные преобразования, включая предварительное вычисление выражений с константами.

Но характеристика не проверяется автоматически.

Если объявить функцию `IMMUTABLE`, хотя она читает изменяемые данные, можно получить некорректную оптимизацию.

Особенно опасно использовать неправильно объявленную функцию в индексах по выражениям.

#### STABLE

`STABLE` предназначен для функций, результат которых может зависеть от состояния базы данных или контекста выполнения, но остаётся согласованным в пределах соответствующего SQL-оператора.

Например, функция чтения настройки из таблицы:

```sql
CREATE TABLE course_settings (
    setting_name TEXT PRIMARY KEY,
    setting_value TEXT NOT NULL
);
```

```sql
CREATE OR REPLACE FUNCTION get_course_setting(
    p_name TEXT
)
RETURNS TEXT
LANGUAGE sql
STABLE
AS $$
    SELECT setting_value
    FROM course_settings
    WHERE setting_name = p_name;
$$;
```

Поскольку данные таблицы могут изменяться, функция не является `IMMUTABLE`.

Важная особенность PostgreSQL: `STABLE` и `IMMUTABLE` функции используют снимок данных, установленный на начало вызывающего SQL-оператора, тогда как `VOLATILE`-функции при выполнении собственных SQL-запросов могут получать новые снимки в соответствии с правилами PostgreSQL.

Поэтому характеристика влияет не только на оптимизацию, но и на семантику видимости данных.

#### VOLATILE

`VOLATILE` — значение по умолчанию.

Оно подходит для функций, которые:

- изменяют данные;
- обращаются к последовательностям;
- используют изменяющиеся значения;
- имеют другие побочные эффекты.

Например:

```sql
CREATE OR REPLACE FUNCTION get_random_value()
RETURNS DOUBLE PRECISION
LANGUAGE sql
VOLATILE
AS $$
    SELECT random();
$$;
```

Разные вызовы могут возвращать разные значения.

Функция записи в журнал также должна быть `VOLATILE`.

#### Сравнение

| Характеристика | Основная гарантия |
|---|---|
| IMMUTABLE | Один результат для одинаковых аргументов независимо от изменяемого контекста |
| STABLE | Стабильность в пределах одного SQL-оператора и его снимка |
| VOLATILE | Результат и побочные эффекты могут меняться между вызовами |

Нельзя считать `IMMUTABLE` просто вариантом более быстрой функции.

Неправильный выбор характеристики способен нарушить корректность результатов.

### Тема 7. Параллельное выполнение функций

PostgreSQL поддерживает параллельные планы некоторых SQL-запросов.

При этом пользовательские функции могут ограничивать допустимость параллельного выполнения.

#### PARALLEL SAFE

Функция разрешена для выполнения в параллельных рабочих процессах.

```sql
CREATE OR REPLACE FUNCTION multiply_numbers(
    p_a INTEGER,
    p_b INTEGER
)
RETURNS INTEGER
LANGUAGE sql
IMMUTABLE
PARALLEL SAFE
AS $$
    SELECT p_a * p_b;
$$;
```

Такая функция не обращается к изменяемому состоянию и не выполняет операций, несовместимых с параллельным выполнением.

#### PARALLEL UNSAFE

`PARALLEL UNSAFE` означает, что функция не должна использоваться внутри параллельного плана.

Это значение по умолчанию для пользовательских функций PostgreSQL.

К небезопасным ситуациям относятся, в частности:

- изменение данных;
- некоторые изменения состояния сессии;
- операции с последовательностями;
- управление транзакционным состоянием;
- иные действия, несовместимые с параллельными рабочими процессами.

Функция записи в таблицу аудита должна быть `PARALLEL UNSAFE`.

#### PARALLEL RESTRICTED

`PARALLEL RESTRICTED` позволяет использовать функцию в параллельном плане, но она должна выполняться в ведущем процессе.

Такое ограничение требуется, например, при работе с некоторыми объектами и состояниями, доступными только ведущему процессу.

#### Важное правило

Нельзя присваивать всем функциям `PARALLEL SAFE` только ради ускорения запросов.

PostgreSQL доверяет указанным разработчиком свойствам.

Ошибочная маркировка может приводить к неверным результатам или ошибкам выполнения.

Также даже корректная `PARALLEL SAFE` функция не гарантирует, что весь запрос будет выполняться параллельно.

Решение зависит от:

- стоимости запроса;
- размера таблиц;
- настроек параллелизма;
- операторов плана;
- других вызываемых функций.

### Тема 8. Безопасность пользовательских функций

Пользовательские функции являются частью системы разграничения доступа.

В PostgreSQL существуют режимы:

```sql
SECURITY INVOKER
```

и:

```sql
SECURITY DEFINER
```

#### SECURITY INVOKER

Используется по умолчанию.

Функция выполняется с правами вызывающего пользователя.

Если внутри функции выполняется запрос к таблице, вызывающему пользователю обычно нужны соответствующие права на неё.

#### SECURITY DEFINER

Функция выполняется с правами владельца функции.

Это позволяет предоставить пользователю ограниченный интерфейс к данным без прямого доступа к таблицам.

Например, приложение может иметь право вызвать функцию изменения статуса заказа, но не иметь права выполнять произвольный UPDATE таблицы заказов.

Однако такой режим создаёт границу привилегий.

Если функция написана неправильно, пользователь может получить доступ к действиям, которые ему не предназначены.

#### Безопасное использование search_path

В PostgreSQL особенно важно исключить подмену объектов через `search_path`.

Небезопасная функция может обращаться к таблице или другой функции по неуточнённому имени.

Если злоумышленник способен создавать объекты в одной из просматриваемых схем, существует риск подмены разрешения имени.

Поэтому для `SECURITY DEFINER` необходимо:

- явно задавать безопасный `search_path`;
- не включать недоверенные схемы;
- использовать квалифицированные имена объектов;
- ограничивать права `CREATE` в схемах;
- контролировать динамический SQL;
- ограничивать доступ к самой функции.

Пример определения:

```sql
CREATE SCHEMA IF NOT EXISTS app_private;

CREATE TABLE app_private.audit_events (
    event_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    message TEXT NOT NULL
);
```

```sql
CREATE OR REPLACE FUNCTION app_private.write_audit_event(
    p_message TEXT
)
RETURNS void
LANGUAGE plpgsql
VOLATILE
PARALLEL UNSAFE
SECURITY DEFINER
SET search_path = pg_catalog, app_private, pg_temp
AS $$
BEGIN
    INSERT INTO app_private.audit_events (message)
    VALUES (p_message);
END;
$$;
```

Этот пример предполагает, что владельцем функции является доверенная роль, а недоверенные пользователи не могут создавать или изменять объекты в `app_private`.

Кроме того, на функцию нужно настроить права выполнения.

#### Права EXECUTE

В PostgreSQL новые функции по умолчанию обычно получают право `EXECUTE` для `PUBLIC`.

Поэтому для функции с повышенными привилегиями необходимо явно контролировать доступ.

Например:

```sql
REVOKE ALL
ON FUNCTION app_private.write_audit_event(TEXT)
FROM PUBLIC;
```

Затем:

```sql
GRANT EXECUTE
ON FUNCTION app_private.write_audit_event(TEXT)
TO application_role;
```

Предполагается, что `application_role` уже существует.

Также вызывающей роли необходим `USAGE` на схему, в которой размещена функция.

Для защищённых функций удобно создавать функцию и настраивать права в одной миграционной транзакции, чтобы не возникало нежелательного промежутка доступности.

#### Динамический SQL

Динамические SQL-команды требуют отдельного внимания.

Нельзя просто конкатенировать пользовательские значения в текст SQL.

В PL/pgSQL для динамического исполнения существует `EXECUTE`.

Для подстановки значений предпочтительно использовать параметры через `USING`, а для идентификаторов — безопасное форматирование, например `%I` в `format()`.

Но даже при технически безопасном построении запроса необходимо проверять, разрешена ли вызывающему пользователю запрошенная операция.

#### Различия других СУБД

| СУБД | Основные механизмы |
|---|---|
| PostgreSQL | SECURITY INVOKER / SECURITY DEFINER |
| SQL Server | EXECUTE AS, ownership chaining, подпись модулей сертификатами |
| MySQL | SQL SECURITY DEFINER / INVOKER |
| Oracle | AUTHID DEFINER / CURRENT_USER |

Модели различаются, поэтому нельзя считать их полностью эквивалентными.

### Тема 9. Практика: функция расчёта корней квадратного уравнения

Рассмотрим математический пример, демонстрирующий:

- входные параметры;
- локальные переменные;
- условные конструкции;
- табличный результат;
- обработку разных вариантов вычисления.

Квадратное уравнение:

\[
ax^2 + bx + c = 0
\]

Дискриминант:

\[
D = b^2 - 4ac
\]

При `D > 0` существуют два различных действительных корня.

При `D = 0` существует один действительный корень.

При `D < 0` действительных корней нет.

Предполагаем, что `a` не равно нулю.

```sql
CREATE OR REPLACE FUNCTION solve_quadratic(
    p_a DOUBLE PRECISION,
    p_b DOUBLE PRECISION,
    p_c DOUBLE PRECISION
)
RETURNS TABLE (
    root_number INTEGER,
    root_value DOUBLE PRECISION
)
LANGUAGE plpgsql
IMMUTABLE
PARALLEL SAFE
AS $$
DECLARE
    discriminant DOUBLE PRECISION;
BEGIN
    IF p_a IS NULL
       OR p_b IS NULL
       OR p_c IS NULL THEN
        RAISE EXCEPTION
            'Коэффициенты не должны быть NULL';
    END IF;

    IF p_a = 0 THEN
        RAISE EXCEPTION
            'Коэффициент a должен быть отличен от нуля';
    END IF;

    discriminant := p_b * p_b - 4 * p_a * p_c;

    IF discriminant < 0 THEN
        RETURN;

    ELSIF discriminant = 0 THEN
        root_number := 1;
        root_value := -p_b / (2 * p_a);

        RETURN NEXT;

    ELSE
        root_number := 1;
        root_value :=
            (-p_b + sqrt(discriminant)) / (2 * p_a);

        RETURN NEXT;

        root_number := 2;
        root_value :=
            (-p_b - sqrt(discriminant)) / (2 * p_a);

        RETURN NEXT;
    END IF;

    RETURN;
END;
$$;
```

Вызов:

```sql
SELECT *
FROM solve_quadratic(1, -3, 2);
```

Результат:

| root_number | root_value |
|---:|---:|
| 1 | 2 |
| 2 | 1 |

Один корень:

```sql
SELECT *
FROM solve_quadratic(1, -2, 1);
```

Результат:

| root_number | root_value |
|---:|---:|
| 1 | 1 |

Отрицательный дискриминант:

```sql
SELECT *
FROM solve_quadratic(1, 0, 1);
```

Результат — пустой набор строк.

Это сознательно выбранный интерфейс: функция возвращает только действительные корни.

Важное ограничение: использование `DOUBLE PRECISION` означает приближённую арифметику.

Сравнение дискриминанта с нулём и формула вычисления корней могут быть численно неустойчивыми для некоторых коэффициентов.

Для задач, требующих высокой численной точности, необходима отдельная стратегия вычислений.

### Тема 10. Практика: учебная модель студентов и курсов

Рассмотрим функцию расчёта прогресса студента.

Создадим учебные таблицы.

#### Студенты

```sql
CREATE TABLE students (
    student_id BIGINT PRIMARY KEY,
    student_name TEXT NOT NULL
);
```

#### Курсы

```sql
CREATE TABLE courses (
    course_id BIGINT PRIMARY KEY,
    course_name TEXT NOT NULL
);
```

#### Запись студентов на курсы

```sql
CREATE TABLE enrollments (
    student_id BIGINT NOT NULL
        REFERENCES students(student_id),
    course_id BIGINT NOT NULL
        REFERENCES courses(course_id),
    PRIMARY KEY (student_id, course_id)
);
```

#### Задания

```sql
CREATE TABLE assignments (
    assignment_id BIGINT PRIMARY KEY,
    course_id BIGINT NOT NULL
        REFERENCES courses(course_id),
    assignment_name TEXT NOT NULL,
    max_score NUMERIC(8, 2) NOT NULL
        CHECK (max_score > 0)
);
```

#### Отправленные решения

```sql
CREATE TABLE submissions (
    submission_id BIGINT PRIMARY KEY,
    assignment_id BIGINT NOT NULL
        REFERENCES assignments(assignment_id),
    student_id BIGINT NOT NULL
        REFERENCES students(student_id),
    score NUMERIC(8, 2)
        CHECK (score >= 0),
    status TEXT NOT NULL
        CHECK (status IN (
            'submitted',
            'reviewed',
            'rejected'
        ))
);
```

В этой упрощённой модели допустимы несколько попыток выполнения одного задания.

Для расчёта прогресса будем использовать максимальный подтверждённый результат студента по каждому заданию.

В реальной модели также необходимо обеспечить, чтобы студент отправлял решения только по курсам, на которые он записан, а баллы не превышали максимума задания. Здесь эти правила будут дополнительно учитываться в запросе, но полноценную целостность модели следует обеспечивать отдельно.

#### Журнал действий

```sql
CREATE TABLE student_action_log (
    log_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    student_id BIGINT NOT NULL
        REFERENCES students(student_id),
    action_type TEXT NOT NULL,
    details JSONB,
    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);
```

#### Данные

```sql
INSERT INTO students (
    student_id,
    student_name
)
VALUES
    (1, 'Анна'),
    (2, 'Борис');
```

```sql
INSERT INTO courses (
    course_id,
    course_name
)
VALUES
    (10, 'Основы SQL');
```

```sql
INSERT INTO enrollments (
    student_id,
    course_id
)
VALUES
    (1, 10),
    (2, 10);
```

```sql
INSERT INTO assignments (
    assignment_id,
    course_id,
    assignment_name,
    max_score
)
VALUES
    (101, 10, 'SELECT', 20),
    (102, 10, 'JOIN', 30),
    (103, 10, 'Транзакции', 50);
```

```sql
INSERT INTO submissions (
    submission_id,
    assignment_id,
    student_id,
    score,
    status
)
VALUES
    (1, 101, 1, 15, 'reviewed'),
    (2, 102, 1, 20, 'reviewed'),
    (3, 102, 1, 25, 'reviewed'),
    (4, 103, 1, NULL, 'submitted'),
    (5, 101, 2, 20, 'reviewed');
```

Для студента Анны максимальное количество баллов по курсу:

```text
20 + 30 + 50 = 100
```

Подтверждённые баллы:

```text
15 + 25 + 0 = 40
```

Прогресс:

```text
40%
```

### Тема 11. Функция логирования действия

Создадим функцию, которая записывает событие в таблицу.

```sql
CREATE OR REPLACE FUNCTION log_student_action(
    p_student_id BIGINT,
    p_action_type TEXT,
    p_details JSONB DEFAULT '{}'::jsonb
)
RETURNS void
LANGUAGE plpgsql
VOLATILE
PARALLEL UNSAFE
AS $$
BEGIN
    INSERT INTO student_action_log (
        student_id,
        action_type,
        details
    )
    VALUES (
        p_student_id,
        p_action_type,
        p_details
    );
END;
$$;
```

Вызов:

```sql
SELECT log_student_action(
    1,
    'course_opened',
    '{"course_id":10}'::jsonb
);
```

Проверка:

```sql
SELECT
    student_id,
    action_type,
    details,
    created_at
FROM student_action_log
ORDER BY log_id;
```

Здесь функция:

- принимает три аргумента;
- использует значение по умолчанию;
- выполняет INSERT;
- не возвращает содержательного результата.

Важно, что журнал записывается в той же транзакции.

Если транзакция будет отменена, запись лога также отменится.

Такая функция подходит для прикладного журнала подтверждённых действий, но не заменяет независимый аудит безопасности.

### Тема 12. Функция расчёта прогресса студента

Создадим скалярную функцию, возвращающую процент набранных баллов от максимально возможной суммы по курсу.

Для каждого задания учитывается лучший проверенный результат.

#### Реализация PL/pgSQL

```sql
CREATE OR REPLACE FUNCTION calculate_student_progress(
    p_student_id BIGINT,
    p_course_id BIGINT
)
RETURNS NUMERIC
LANGUAGE plpgsql
STABLE
PARALLEL SAFE
AS $$
DECLARE
    total_possible NUMERIC := 0;
    total_earned NUMERIC := 0;
    result_percent NUMERIC := 0;
BEGIN
    IF NOT EXISTS (
        SELECT 1
        FROM enrollments
        WHERE student_id = p_student_id
          AND course_id = p_course_id
    ) THEN
        RETURN NULL;
    END IF;

    SELECT
        COALESCE(SUM(a.max_score), 0),
        COALESCE(
            SUM(
                LEAST(
                    COALESCE(best.best_score, 0),
                    a.max_score
                )
            ),
            0
        )
    INTO
        total_possible,
        total_earned
    FROM assignments AS a
    LEFT JOIN LATERAL (
        SELECT MAX(s.score) AS best_score
        FROM submissions AS s
        WHERE s.assignment_id = a.assignment_id
          AND s.student_id = p_student_id
          AND s.status = 'reviewed'
    ) AS best ON TRUE
    WHERE a.course_id = p_course_id;

    IF total_possible = 0 THEN
        RETURN 0;
    END IF;

    result_percent := ROUND(
        total_earned * 100 / total_possible,
        2
    );

    RETURN result_percent;
END;
$$;
```

Вызов:

```sql
SELECT calculate_student_progress(1, 10);
```

Результат:

```text
40.00
```

Для Бориса:

```sql
SELECT calculate_student_progress(2, 10);
```

Результат:

```text
20.00
```

#### Особенности реализации

Функция:

1. Проверяет запись студента на курс.
2. Получает задания курса.
3. Находит лучший проверенный результат по каждому заданию.
4. Ограничивает учтённый результат максимальным баллом задания.
5. Вычисляет суммарный прогресс.
6. Возвращает процент.

Если студент не записан на курс, функция возвращает `NULL`.

Если курс не содержит заданий, функция возвращает `0`.

Это выбранные бизнес-правила, а не обязательное поведение всех функций расчёта прогресса.

В реальном приложении нужно явно определить, чем отличаются:

- студент не записан на курс;
- курс не содержит заданий;
- студент ещё не отправлял решений;
- все работы ожидают проверки;
- работы отклонены;
- есть несколько проверенных попыток.

#### Почему функция STABLE

Функция читает таблицы, состояние которых может изменяться.

Следовательно, она не является `IMMUTABLE`.

В рамках одного SQL-оператора функция может использовать согласованный снимок данных.

Поэтому `STABLE` соответствует выбранной реализации.

Функция не изменяет данные, не использует последовательности и не обращается к состоянию, которое запрещено для параллельных работников, поэтому для данного варианта подходит `PARALLEL SAFE`.

Но при дальнейшем изменении тела функции эти характеристики необходимо пересматривать.

### Тема 13. Композиция функций и побочные эффекты

Исходная идея учебного примера предполагает, что функция расчёта прогресса также вызывает функцию логирования.

Технически в PostgreSQL это возможно.

Например:

```sql
PERFORM log_student_action(
    p_student_id,
    'progress_calculated',
    jsonb_build_object(
        'course_id', p_course_id
    )
);
```

Однако здесь возникает архитектурная проблема.

Функция расчёта больше не является функцией только чтения.

Она начинает изменять таблицу журнала.

Следовательно, объявлять её `STABLE` нельзя.

Она должна быть `VOLATILE` и `PARALLEL UNSAFE`.

Кроме того, вызов:

```sql
SELECT calculate_student_progress(1, 10);
```

теперь не только возвращает результат, но и записывает событие в таблицу.

Это может быть неожиданно для вызывающего кода.

#### Почему лучше разделить ответственность

В большинстве прикладных сценариев удобнее иметь:

- функцию расчёта прогресса без побочных эффектов;
- отдельную функцию записи события;
- процедуру или прикладной сервис, который объединяет эти действия при необходимости.

Например:

```sql
BEGIN;

SELECT calculate_student_progress(1, 10);

SELECT log_student_action(
    1,
    'progress_viewed',
    '{"course_id":10}'::jsonb
);

COMMIT;
```

Здесь обе операции выполняются в одной транзакции.

При этом нужно помнить, что `SELECT` для получения прогресса в общем случае не обязан означать факт просмотра пользователем.

Именно приложение знает, была ли информация действительно предоставлена пользователю.

Поэтому логирование события `progress_viewed` обычно лучше выполнять в момент соответствующего прикладного действия.

#### Когда побочные эффекты допустимы

Функции PostgreSQL могут изменять данные.

Это полезно для определённых:

- триггерных функций;
- служебных операций;
- атомарных действий;
- внутренних API базы данных.

Но необходимо явно отражать побочные эффекты в интерфейсе, документации и характеристиках функции.

### Тема 14. Производительность пользовательских функций

Пользовательская функция может сделать SQL-запрос проще, но не обязательно быстрее.

Рассмотрим:

```sql
SELECT
    student_id,
    calculate_student_progress(student_id, 10)
FROM students;
```

Для каждого студента вызывается функция, внутри которой выполняются дополнительные SQL-запросы.

Если студентов несколько, это может быть приемлемо.

Если студентов сотни тысяч, стоимость может оказаться значительной.

#### Проблема построчных вызовов

Один из распространённых источников неэффективности — выполнение сложной пользовательской функции для каждой строки большого набора.

Например:

```sql
SELECT
    order_id,
    expensive_user_function(order_id)
FROM orders;
```

В зависимости от реализации функция может выполнять дополнительные обращения к таблицам для каждого заказа.

Это похоже на проблему N+1 запросов на уровне приложения.

#### Множественная альтернатива

Иногда вычисление лучше выразить одним запросом с группировкой.

Например, прогресс всех студентов курса можно рассчитать через CTE и JOIN без отдельного вызова функции на каждого студента.

```sql
WITH best_scores AS (
    SELECT
        s.student_id,
        a.course_id,
        a.assignment_id,
        LEAST(MAX(s.score), a.max_score) AS earned_score
    FROM submissions AS s
    JOIN assignments AS a
        ON a.assignment_id = s.assignment_id
    WHERE s.status = 'reviewed'
    GROUP BY
        s.student_id,
        a.course_id,
        a.assignment_id,
        a.max_score
),
course_totals AS (
    SELECT
        course_id,
        SUM(max_score) AS max_total
    FROM assignments
    GROUP BY course_id
),
student_totals AS (
    SELECT
        student_id,
        course_id,
        SUM(earned_score) AS earned_total
    FROM best_scores
    GROUP BY student_id, course_id
)
SELECT
    e.student_id,
    e.course_id,
    CASE
        WHEN COALESCE(ct.max_total, 0) = 0
            THEN 0
        ELSE ROUND(
            COALESCE(st.earned_total, 0) * 100 /
            ct.max_total,
            2
        )
    END AS progress_percent
FROM enrollments AS e
LEFT JOIN course_totals AS ct
    ON ct.course_id = e.course_id
LEFT JOIN student_totals AS st
    ON st.student_id = e.student_id
   AND st.course_id = e.course_id;
```

Этот вариант вычисляет прогресс сразу для всех записанных студентов.

При больших объёмах данных он может оказаться эффективнее многократных вызовов процедурной функции.

Но результат следует проверять на реальном наборе данных и плане выполнения.

#### Inlining

PostgreSQL может встраивать тело некоторых SQL-функций в план вызывающего запроса.

Это называется function inlining.

Например, простая `LANGUAGE sql` функция при соблюдении условий может быть раскрыта оптимизатором.

Но не каждая функция подлежит такому преобразованию.

Ограничения зависят от:

- языка;
- конструкции тела;
- характеристик функции;
- параметров;
- режима безопасности;
- типа результата.

В SQL Server также существуют механизмы scalar UDF inlining для подходящих функций.

Они могут существенно уменьшать накладные расходы, но применимы не ко всем T-SQL UDF.

Поэтому утверждение «пользовательские функции всегда выполняются отдельно для каждой строки» не является универсальным.

#### COST и ROWS

PostgreSQL позволяет задавать оценки стоимости.

Например:

```sql
ALTER FUNCTION calculate_student_progress(
    BIGINT,
    BIGINT
)
COST 100;
```

Для функций, возвращающих набор строк, может задаваться оценка `ROWS`.

Эти параметры помогают оптимизатору оценивать стоимость вызова и ожидаемое количество результатов.

Но они не ограничивают фактическое время выполнения и не заставляют функцию возвращать заданное количество строк.

Это оценки для планирования.

#### EXPLAIN ANALYZE

Пример:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    student_id,
    calculate_student_progress(student_id, 10)
FROM students;
```

План позволяет анализировать выполнение основного запроса.

Для подробной диагностики SQL-команд внутри PL/pgSQL-функций могут понадобиться дополнительные средства мониторинга, например `pg_stat_statements` и `auto_explain` при соответствующей настройке.

Важно учитывать, что не все внутренние операции пользовательской функции обязательно детально отображаются в обычном плане вызывающего запроса.

### Тема 15. Проектирование и сопровождение функций

Функции — часть программного интерфейса базы данных.

Поэтому к ним следует применять обычные принципы разработки.

#### Ясная ответственность

Желательно, чтобы функция решала одну понятную задачу.

Например:

```text
calculate_discount_price
```

— вычисление цены.

```text
calculate_student_progress
```

— расчёт прогресса.

```text
log_student_action
```

— запись события.

Если одна функция одновременно выполняет сложную аналитику, изменяет десятки таблиц, обращается к внешним системам и логирует каждое действие, её становится трудно тестировать и сопровождать.

#### Предсказуемый результат

Необходимо определить:

- возвращаемый тип;
- поведение при `NULL`;
- допустимость отсутствующих данных;
- обработку ошибок;
- точность числовых вычислений;
- порядок строк табличного результата;
- наличие побочных эффектов.

Табличная функция не гарантирует порядок результата без соответствующего внешнего `ORDER BY`.

Даже если строки внутри функции получены с сортировкой, вызывающий запрос должен явно задавать порядок, если он существенен.

#### Обработка NULL

В PostgreSQL можно объявить:

```sql
RETURNS NULL ON NULL INPUT
```

или эквивалентное:

```sql
STRICT
```

Для такой функции PostgreSQL автоматически возвращает SQL `NULL`, если хотя бы один входной аргумент равен `NULL`.

Тело функции в этом случае не вызывается.

Но `STRICT` подходит не для всех функций.

Например, функция, которая должна заменять `NULL` значением по умолчанию, не должна быть объявлена таким способом.

#### Тестирование

Необходимо проверять:

1. Обычные корректные аргументы.
2. Граничные значения.
3. Нулевые значения.
4. `NULL`.
5. Отсутствующие связанные записи.
6. Некорректные типы и диапазоны.
7. Большие наборы данных.
8. Конкурентные вызовы.
9. Права доступа.
10. Поведение внутри транзакций.

Особенно важно тестировать функции с побочными эффектами.

#### Миграции

Определения функций должны храниться в системе контроля версий.

Например:

```text
database/
  migrations/
    V001__create_tables.sql
    V002__create_student_functions.sql
    V003__update_progress_calculation.sql
```

Изменение сигнатуры функции может повлиять на:

- приложения;
- представления;
- процедуры;
- другие функции;
- триггеры;
- права EXECUTE;
- подготовленные запросы.

Поэтому изменение функции — полноценное изменение интерфейса базы данных.

#### Выбор между SQL-функцией, PL/pgSQL и процедурой

| Задача | Подход |
|---|---|
| Простое вычисление одного выражения | SQL-функция или выражение непосредственно в SQL |
| Параметризованная выборка | SQL-функция RETURNS TABLE |
| Несколько условий и переменных | PL/pgSQL |
| Циклы и обработка исключений | PL/pgSQL |
| Операция с контролируемыми побочными эффектами | Функция VOLATILE либо процедура |
| Управление транзакциями внутри серверной логики | Процедура с учётом ограничений СУБД |
| Массовое обновление строк | В первую очередь множественный SQL |
| Многократно используемый бизнес-инвариант | Ограничение БД, функция или процедура — по характеру правила |

Главный критерий: функция должна уменьшать сложность системы и делать поведение более предсказуемым, а не скрывать неконтролируемую последовательность действий.

## Итоговые выводы

1. **Пользовательские функции оформляют повторно используемую логику в самостоятельные объекты СУБД.** Они позволяют централизовать вычисления, преобразования и определённые операции с данными.

2. **Функции отличаются формой возвращаемого результата.** Скалярные функции возвращают одиночное значение, а табличные — набор строк.

3. **Синтаксис и возможности функций зависят от СУБД.** PostgreSQL, SQL Server, MySQL и Oracle имеют разные модели создания, выполнения и ограничения пользовательских функций.

4. **Параметры определяют интерфейс функции.** В PostgreSQL поддерживаются `IN`, `OUT`, `INOUT` и другие варианты, но этот набор нельзя механически переносить на функции MySQL или SQL Server.

5. **Именованные аргументы и значения по умолчанию повышают удобство использования функций.** При этом имена, типы и порядок параметров являются частью интерфейса, который необходимо сопровождать.

6. **RETURNS TABLE, SETOF и RECORD предназначены для разных форм результата.** Функции с известной структурой обычно удобнее объявлять через `RETURNS TABLE`.

7. **RETURN NEXT и RETURN QUERY позволяют формировать множества строк в PL/pgSQL.** Они не должны автоматически рассматриваться как потоковая передача результата клиенту.

8. **IMMUTABLE, STABLE и VOLATILE влияют на оптимизацию и семантику выполнения.** Их необходимо объявлять в соответствии с реальными свойствами функции.

9. **Параллельная безопасность является свойством корректности, а не только производительности.** `PARALLEL SAFE` нельзя назначать без проверки выполняемых операций.

10. **SECURITY DEFINER предоставляет возможности выполнения с правами владельца.** Такой режим требует ограничения EXECUTE, защиты `search_path` и внимательной проверки логики функции.

11. **Функции PostgreSQL могут иметь побочные эффекты, но не все СУБД позволяют это одинаково.** Функции, изменяющие данные, требуют особого внимания к транзакциям и архитектуре.

12. **Вызов функции для каждой строки большого набора может оказаться дорогим.** При массовых вычислениях следует рассматривать множественный SQL и возможности оптимизатора.

13. **Пользовательские функции необходимо хранить в Git и тестировать.** Изменение сигнатуры или поведения функции способно повлиять на все приложения и объекты базы данных, которые её используют.

Главный результат занятия — понимание того, как проектировать пользовательские функции с предсказуемым интерфейсом, корректными характеристиками выполнения, контролируемыми правами и разумными границами ответственности.

## Дополнительные материалы

### PostgreSQL

- [CREATE FUNCTION](https://www.postgresql.org/docs/current/sql-createfunction.html)
- [User-Defined Functions](https://www.postgresql.org/docs/current/xfunc.html)
- [SQL Functions](https://www.postgresql.org/docs/current/xfunc-sql.html)
- [PL/pgSQL Functions](https://www.postgresql.org/docs/current/plpgsql.html)
- [Control Structures](https://www.postgresql.org/docs/current/plpgsql-control-structures.html)
- [Function Volatility Categories](https://www.postgresql.org/docs/current/xfunc-volatility.html)
- [Parallel Safety](https://www.postgresql.org/docs/current/parallel-safety.html)
- [Function Security](https://www.postgresql.org/docs/current/perm-functions.html)
- [Function Optimization Information](https://www.postgresql.org/docs/current/xfunc-optimization.html)

### Microsoft SQL Server

- [CREATE FUNCTION](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-function-transact-sql)
- [User-Defined Functions](https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/user-defined-functions)
- [Scalar UDF Inlining](https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/scalar-udf-inlining)
- [EXECUTE AS](https://learn.microsoft.com/en-us/sql/t-sql/statements/execute-as-clause-transact-sql)

### MySQL/InnoDB

- [CREATE PROCEDURE and CREATE FUNCTION](https://dev.mysql.com/doc/refman/8.4/en/create-procedure.html)
- [Stored Function Restrictions](https://dev.mysql.com/doc/refman/8.4/en/stored-program-restrictions.html)
- [Stored Object Access Control](https://dev.mysql.com/doc/refman/8.4/en/stored-objects-security.html)
- [Stored Program Binary Logging](https://dev.mysql.com/doc/refman/8.4/en/stored-programs-logging.html)

### Oracle

- [CREATE FUNCTION](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/CREATE-FUNCTION.html)
- [PL/SQL Subprograms](https://docs.oracle.com/en/database/oracle/oracle-database/23/lnpls/plsql-subprograms.html)
- [PL/SQL Functions That SQL Statements Can Invoke](https://docs.oracle.com/en/database/oracle/oracle-database/23/lnpls/pl-sql-functions-that-sql-statements-can-invoke.html)
- [Pipelined Table Functions](https://docs.oracle.com/en/database/oracle/oracle-database/23/addci/using-pipelined-and-parallel-table-functions.html)
