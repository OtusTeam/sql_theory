
# Хранимые процедуры в SQL

## Превью / аннотация / краткое содержание

Занятие посвящено хранимым процедурам — программным объектам реляционных СУБД, предназначенным для выполнения последовательностей операций над данными.

Хранимые процедуры позволяют объединять SQL-команды, проверки, вычисления, обработку ошибок и служебные действия в один вызываемый интерфейс. Они применяются при реализации бизнес-операций, массовой обработке данных, административных задачах и создании контролируемого доступа к базе.

Однако хранимая процедура не является автоматически независимой транзакцией. Её поведение зависит от СУБД, контекста вызова и способа управления ошибками.

На занятии рассматриваются:

- назначение хранимых процедур;
- отличия процедур от пользовательских функций;
- создание процедур в четырёх СУБД курса;
- входные и выходные параметры;
- вызов процедур и получение результатов;
- диагностические сообщения;
- генерация и обработка исключений;
- управление транзакциями;
- динамический SQL и параметризация;
- защита от SQL-инъекций;
- практическая процедура расчёта прогресса студента;
- логирование ошибок и ограничение побочных эффектов;
- безопасность, производительность и архитектурные границы процедурной логики.

Основные практические примеры приведены для PostgreSQL и PL/pgSQL. Дополнительно рассматриваются Microsoft SQL Server, MySQL/InnoDB и Oracle.

## Основные понятия

**Хранимая процедура (Stored Procedure)** — именованный программный объект базы данных, выполняемый через специальную команду вызова.

**CREATE PROCEDURE** — SQL-команда создания процедуры.

**CREATE OR REPLACE PROCEDURE** — команда создания или замены определения процедуры в PostgreSQL и других поддерживающих её СУБД.

**CREATE OR ALTER PROCEDURE** — синтаксис создания или изменения процедуры в SQL Server.

**CALL** — команда вызова процедуры в PostgreSQL, MySQL и Oracle.

**EXEC / EXECUTE** — команды выполнения процедур в SQL Server и некоторых других контекстах.

**IN-параметр** — входной параметр процедуры.

**OUT-параметр** — параметр, через который процедура передаёт результат вызывающей стороне.

**INOUT-параметр** — параметр, используемый для передачи входного значения и возврата результата.

**Позиционный вызов** — передача аргументов в порядке их объявления.

**Именованный вызов** — передача аргументов с указанием имён параметров.

**Диагностическое сообщение** — служебное сообщение, формируемое процедурным кодом.

**RAISE NOTICE** — средство PL/pgSQL для выдачи информационного сообщения.

**RAISE EXCEPTION** — средство PL/pgSQL для генерации ошибки.

**PRINT** — команда T-SQL для вывода диагностического сообщения.

**DBMS_OUTPUT.PUT_LINE** — процедура Oracle для вывода сообщения в буфер DBMS_OUTPUT.

**SIGNAL** — конструкция MySQL для генерации условия ошибки.

**Исключение (Exception)** — ошибка или иное условие, изменяющее обычный ход исполнения программы.

**EXCEPTION** — раздел обработки исключений PL/pgSQL и PL/SQL.

**TRY / CATCH** — механизм обработки ошибок SQL Server.

**DECLARE HANDLER** — конструкция объявления обработчика условий в MySQL.

**WHEN OTHERS** — обработчик большинства исключений PL/pgSQL и PL/SQL.

**SQLSTATE** — пятисимвольный код SQL-ошибки.

**P0001** — стандартный код пользовательского исключения, генерируемого `RAISE EXCEPTION` без явного кода в PostgreSQL.

**SQLERRM** — специальное значение PL/pgSQL, позволяющее получить сообщение об ошибке внутри обработчика.

**GET STACKED DIAGNOSTICS** — конструкция PL/pgSQL для получения подробных сведений об исключении.

**THROW** — команда SQL Server для генерации или повторной передачи исключения.

**COMMIT** — подтверждение изменений транзакции.

**ROLLBACK** — отмена изменений транзакции.

**SAVEPOINT** — точка сохранения внутри транзакции.

**Подтранзакция (Subtransaction)** — механизм частичного отката, применяемый, в частности, при обработке исключений PL/pgSQL.

**Динамический SQL (Dynamic SQL)** — SQL-код, формируемый во время выполнения программы.

**EXECUTE ... USING** — конструкция PL/pgSQL для выполнения динамического SQL с отдельно переданными значениями параметров.

**format()** — функция PostgreSQL для форматирования строк, включая безопасное представление идентификаторов и SQL-литералов.

**quote_ident()** — функция PostgreSQL для корректного заключения идентификатора в кавычки при необходимости.

**QUOTENAME()** — функция SQL Server для экранирования идентификаторов.

**sp_executesql** — системная процедура SQL Server для выполнения динамического параметризованного T-SQL.

**PREPARE / EXECUTE** — механизмы подготовки и выполнения динамических SQL-операторов в MySQL и других СУБД.

**EXECUTE IMMEDIATE** — конструкция выполнения динамического SQL в Oracle.

**SQL-инъекция (SQL Injection)** — изменение структуры исполняемого SQL посредством небезопасного включения недоверенных данных.

**SECURITY DEFINER** — режим выполнения процедуры PostgreSQL с правами её владельца.

**SECURITY INVOKER** — режим выполнения процедуры с правами вызывающего пользователя.

**Идемпотентность** — свойство операции, при котором повторное применение не изменяет результат после первого успешного применения, в пределах определённой семантики.

**Транзакционный outbox** — механизм надёжной подготовки событий для последующей передачи внешним системам в рамках транзакции изменения бизнес-данных.

## Основная часть

### Тема 1. Хранимая процедура как объект базы данных

Хранимая процедура представляет собой именованный объект СУБД, содержащий одну или несколько операций.

Она вызывается отдельной командой и может:

- читать данные;
- изменять таблицы;
- выполнять проверки;
- использовать условные конструкции;
- вызывать другие процедуры и функции;
- формировать диагностические сообщения;
- обрабатывать ошибки;
- возвращать выходные параметры;
- управлять транзакциями при соблюдении ограничений конкретной СУБД.

Например, оформление заказа может включать несколько действий:

1. Проверить наличие заказа.
2. Проверить возможность изменения его статуса.
3. Изменить статус.
4. Сохранить сведения об операции.
5. Вернуть итог вызывающему приложению.

Все эти действия можно оформить в одну процедуру.

#### Простейшая процедура PostgreSQL

```sql
CREATE OR REPLACE PROCEDURE hello_procedure()
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Процедура выполнена';
END;
$$;
```

Вызов:

```sql
CALL hello_procedure();
```

Процедура выполняет действие, но не возвращает скалярное значение как SQL-функция.

#### Процедура и серверное выполнение

Хранимая процедура выполняется непосредственно в контексте сервера базы данных.

Это позволяет сократить число обращений приложения к СУБД, если вместо нескольких последовательных запросов приложение отправляет один вызов процедуры.

Однако нельзя утверждать, что процедура всегда работает быстрее обычного SQL.

Если внутри неё выполняется цикл с тысячами отдельных запросов, результат может оказаться хуже одного множественного SQL-оператора.

Главное преимущество процедуры — возможность оформить последовательность связанных действий в самостоятельный интерфейс.

#### Процедура как граница бизнес-операции

Процедура может представлять определённую бизнес-операцию, например:

```text
reserve_inventory
```

```text
complete_order
```

```text
recalculate_course_progress
```

Но граница процедуры не обязательно совпадает с границей транзакции.

Процедура может вызываться внутри уже открытой транзакции приложения.

В некоторых СУБД она также может сама управлять транзакциями при разрешённых условиях.

Эти механизмы необходимо различать.

### Тема 2. Отличие хранимых процедур от пользовательских функций

Функции и процедуры похожи тем, что обе позволяют хранить исполняемый код на стороне базы данных.

Но они имеют различные интерфейсы и особенности выполнения.

#### Пользовательская функция

Функция используется как выражение или источник данных.

Например:

```sql
SELECT calculate_student_progress(1, 10);
```

Она имеет объявленный тип возвращаемого результата.

В PostgreSQL функция может возвращать:

- скалярное значение;
- составное значение;
- множество строк;
- `void`.

Также PostgreSQL допускает функции с побочными эффектами.

Поэтому утверждение «функция всегда только читает данные» неверно.

Однако функции PostgreSQL не могут самостоятельно завершать транзакцию командами `COMMIT` и `ROLLBACK`.

#### Хранимая процедура

Процедура вызывается отдельной командой:

```sql
CALL recalculate_student_progress(1, 10);
```

Она предназначена для выполнения определённой операции.

Результат может возвращаться через выходные параметры или другие поддерживаемые СУБД механизмы.

Процедура не используется как скалярное выражение внутри обычного `SELECT`.

#### Сравнение

| Характеристика | Функция PostgreSQL | Процедура PostgreSQL |
|---|---|---|
| Создание | CREATE FUNCTION | CREATE PROCEDURE |
| Вызов | SELECT / использование в выражении | CALL |
| Объявленный возвращаемый тип | Да | Нет RETURNS как у функции |
| OUT / INOUT | Да | Да |
| Изменение таблиц | Допустимо для подходящих функций | Да |
| COMMIT / ROLLBACK внутри объекта | Нет | При выполнении специальных условий |
| Применение в WHERE | Да, если возвращает подходящий тип | Нет |
| Применение в FROM | Да, в частности для табличных функций | Нет |
| Типичная роль | Вычисление или получение данных | Выполнение операции |

#### Особенность процедур SQL Server

В SQL Server процедура может возвращать несколько наборов строк посредством `SELECT`.

Например:

```sql
CREATE OR ALTER PROCEDURE dbo.get_customer_orders
    @customer_id BIGINT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT *
    FROM dbo.orders
    WHERE customer_id = @customer_id;
END;
GO
```

Вызов:

```sql
EXEC dbo.get_customer_orders
    @customer_id = 1001;
```

Здесь процедура возвращает набор строк клиенту.

Но этот механизм отличается от табличной функции, которую можно непосредственно использовать в `FROM`.

#### PostgreSQL и наборы результатов

В PostgreSQL процедура с `OUT` или `INOUT` возвращает строку значений этих параметров.

Это не то же самое, что произвольная последовательность табличных результатов через обычные `SELECT` внутри процедуры.

В PL/pgSQL для SQL-запроса, возвращающего строки, необходимо указать назначение результата, например:

```sql
SELECT ... INTO ...
```

или использовать `PERFORM`, если результат не нужен.

Если процедура должна возвращать большой табличный результат, иногда лучше использовать функцию `RETURNS TABLE`.

### Тема 3. Синтаксис создания процедур

#### PostgreSQL

Общий вид:

```sql
CREATE OR REPLACE PROCEDURE procedure_name(
    IN p_input INTEGER,
    OUT p_output TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- тело процедуры
END;
$$;
```

Создадим пример:

```sql
CREATE OR REPLACE PROCEDURE calculate_sum(
    IN p_a INTEGER,
    IN p_b INTEGER,
    OUT p_result INTEGER
)
LANGUAGE plpgsql
AS $$
BEGIN
    p_result := p_a + p_b;
END;
$$;
```

Вызов на уровне SQL:

```sql
CALL calculate_sum(10, 20, NULL);
```

Результат:

| p_result |
|---:|
| 30 |

При обычном SQL-вызове PostgreSQL для `OUT`-параметра указывается аргумент-позиция, например `NULL`.

Переданное туда значение не используется как входное.

Однако при вызове из PL/pgSQL для выходного параметра требуется подходящая переменная, принимающая результат.

Это различие важно учитывать.

#### INOUT

```sql
CREATE OR REPLACE PROCEDURE increase_counter(
    INOUT p_counter INTEGER
)
LANGUAGE plpgsql
AS $$
BEGIN
    p_counter := p_counter + 1;
END;
$$;
```

Вызов:

```sql
CALL increase_counter(10);
```

Результат:

| p_counter |
|---:|
| 11 |

`INOUT` получает начальное значение и возвращает изменённое.

#### Вызов из PL/pgSQL

```sql
DO $$
DECLARE
    v_counter INTEGER := 10;
BEGIN
    CALL increase_counter(v_counter);

    RAISE NOTICE 'Counter = %', v_counter;
END;
$$;
```

В этом случае изменение `INOUT`-параметра присваивается переменной `v_counter`.

#### SQL Server

```sql
CREATE OR ALTER PROCEDURE dbo.calculate_sum
    @a INT,
    @b INT,
    @result INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @result = @a + @b;
END;
GO
```

Вызов:

```sql
DECLARE @sum INT;

EXEC dbo.calculate_sum
    @a = 10,
    @b = 20,
    @result = @sum OUTPUT;

SELECT @sum AS result;
```

Результат:

```text
30
```

В SQL Server выходной параметр объявляется через `OUTPUT`.

Также процедуры могут возвращать целочисленный код состояния через `RETURN`, но такой код не заменяет произвольный набор выходных значений.

#### MySQL

```sql
DELIMITER //

CREATE PROCEDURE calculate_sum(
    IN p_a INT,
    IN p_b INT,
    OUT p_result INT
)
BEGIN
    SET p_result = p_a + p_b;
END//

DELIMITER ;
```

Вызов:

```sql
CALL calculate_sum(10, 20, @result);

SELECT @result;
```

В MySQL `OUT`-параметр можно передавать через пользовательскую переменную сессии.

`DELIMITER` используется клиентом для корректной отправки многострочного определения процедуры.

#### Oracle

```sql
CREATE OR REPLACE PROCEDURE calculate_sum(
    p_a IN NUMBER,
    p_b IN NUMBER,
    p_result OUT NUMBER
)
AS
BEGIN
    p_result := p_a + p_b;
END;
/
```

Вызов из PL/SQL:

```sql
DECLARE
    v_result NUMBER;
BEGIN
    calculate_sum(
        10,
        20,
        v_result
    );

    DBMS_OUTPUT.PUT_LINE(v_result);
END;
/
```

Oracle поддерживает процедуры со входными, выходными и комбинированными параметрами.

Способы возврата наборов строк включают, в частности, `REF CURSOR` и другие механизмы.

### Тема 4. Параметры и интерфейс процедуры

Параметры определяют, какие данные принимает процедура и какие результаты возвращает.

#### IN

Входной параметр используется для передачи значения.

```sql
CREATE OR REPLACE PROCEDURE print_student_id(
    IN p_student_id BIGINT
)
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Student ID = %', p_student_id;
END;
$$;
```

Вызов:

```sql
CALL print_student_id(100);
```

#### OUT

Выходной параметр предназначен для формирования результата.

Например:

```sql
CREATE OR REPLACE PROCEDURE get_number_status(
    IN p_number INTEGER,
    OUT p_status TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    IF p_number > 0 THEN
        p_status := 'positive';
    ELSIF p_number < 0 THEN
        p_status := 'negative';
    ELSE
        p_status := 'zero';
    END IF;
END;
$$;
```

Вызов:

```sql
CALL get_number_status(-5, NULL);
```

Результат:

```text
negative
```

#### INOUT

Используется, когда параметр должен принять начальное значение и вернуть новое.

Например, можно изменить накопленный счётчик:

```sql
CALL increase_counter(50);
```

Результат:

```text
51
```

#### Значения по умолчанию

PostgreSQL позволяет задавать значения входных параметров по умолчанию.

```sql
CREATE OR REPLACE PROCEDURE print_message(
    IN p_message TEXT,
    IN p_prefix TEXT DEFAULT 'INFO'
)
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE '[%] %', p_prefix, p_message;
END;
$$;
```

Вызов:

```sql
CALL print_message('Операция выполнена');
```

Или:

```sql
CALL print_message(
    p_prefix => 'DEBUG',
    p_message => 'Проверка'
);
```

Имена и типы параметров являются частью интерфейса процедуры.

При изменении сигнатуры необходимо учитывать зависимые приложения и другие программные объекты.

#### Перегрузка процедур

PostgreSQL допускает несколько процедур с одинаковым именем, если различаются их входные типы аргументов.

Однако перегрузка не должна усложнять понимание вызова.

Сочетание параметров по умолчанию, перегрузок и неявных преобразований типов может приводить к неоднозначному разрешению процедуры.

Поэтому для публичных API базы рекомендуется проектировать простые и предсказуемые сигнатуры.

### Тема 5. Диагностические сообщения

Диагностические сообщения полезны при разработке и отладке процедур.

Они позволяют выводить:

- значения переменных;
- достигнутые этапы;
- информацию о выбранной ветке;
- результат промежуточного вычисления;
- сведения об ошибках.

#### PostgreSQL: RAISE NOTICE

```sql
CREATE OR REPLACE PROCEDURE demo_diagnostics(
    IN p_value INTEGER
)
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Начало обработки';

    RAISE NOTICE 'Получено значение: %', p_value;

    IF p_value > 0 THEN
        RAISE NOTICE 'Число положительное';
    ELSE
        RAISE NOTICE 'Число неположительное';
    END IF;

    RAISE NOTICE 'Конец обработки';
END;
$$;
```

Вызов:

```sql
CALL demo_diagnostics(10);
```

#### Уровни сообщений PostgreSQL

PL/pgSQL поддерживает:

```sql
RAISE DEBUG
RAISE LOG
RAISE INFO
RAISE NOTICE
RAISE WARNING
RAISE EXCEPTION
```

Они различаются назначением и правилами обработки.

`RAISE EXCEPTION` создаёт ошибку.

Остальные перечисленные уровни предназначены для сообщений и не обязательно прекращают выполнение.

Отображение и регистрация сообщений зависят от настроек PostgreSQL и клиента.

Например, `client_min_messages` и `log_min_messages` влияют на разные направления вывода.

#### SQL Server

В T-SQL используется:

```sql
PRINT 'Операция выполнена';
```

Например:

```sql
CREATE OR ALTER PROCEDURE dbo.demo_diagnostics
AS
BEGIN
    PRINT 'Начало обработки';

    PRINT 'Операция выполнена';
END;
GO
```

`PRINT` удобен для простого вывода, но не следует использовать его как надёжный протокол доставки статусов клиенту.

В SQL Server также используется `RAISERROR` для сообщений с определёнными параметрами, хотя для генерации новых ошибок обычно предпочтительнее `THROW`.

#### MySQL

В MySQL нет прямого универсального аналога `RAISE NOTICE`, имеющего полностью идентичную семантику PostgreSQL.

Для простых учебных сообщений процедура может использовать `SELECT`:

```sql
SELECT 'Начало обработки' AS message;
```

Однако такой `SELECT` формирует результат, возвращаемый клиенту.

Это не то же самое, что отдельное диагностическое сообщение PostgreSQL.

#### Oracle

Oracle:

```sql
DBMS_OUTPUT.PUT_LINE('Операция выполнена');
```

Для отображения сообщений в SQL-клиенте обычно необходимо включить соответствующий вывод.

#### Сообщения и журналирование

Диагностические сообщения не должны заменять структурированный результат процедуры.

Если вызывающей стороне необходимо узнать:

- успешно ли завершилась операция;
- какой объект был создан;
- сколько строк изменено;
- какой статус получен,

лучше использовать выходные параметры, возвращаемые наборы данных либо предусмотренные механизмы ошибок.

Для постоянного аудита нужны специально спроектированные журналы.

### Тема 6. Обработка исключений

Во время выполнения процедуры могут возникать ошибки:

- нарушение уникальности;
- нарушение внешнего ключа;
- отсутствие ожидаемой строки;
- неверный тип;
- деление на ноль;
- недостаток прав;
- конфликт конкурентных транзакций.

Процедура должна различать ожидаемые бизнес-условия и непредвиденные технические ошибки.

#### Генерация исключения PostgreSQL

```sql
CREATE OR REPLACE PROCEDURE check_positive_number(
    IN p_value INTEGER
)
LANGUAGE plpgsql
AS $$
BEGIN
    IF p_value <= 0 THEN
        RAISE EXCEPTION
            'Число должно быть положительным: %',
            p_value;
    END IF;
END;
$$;
```

Вызов:

```sql
CALL check_positive_number(-10);
```

приведёт к ошибке.

#### SQLSTATE

Для пользовательской ошибки можно указать код:

```sql
RAISE EXCEPTION USING
    MESSAGE = 'Недопустимая операция',
    ERRCODE = 'P0001';
```

`P0001` — общий код пользовательского исключения PostgreSQL.

Для прикладного API можно разработать собственную систему допустимых SQLSTATE-кодов, не конфликтующую с уже определёнными кодами СУБД.

При этом код ошибки следует выбирать осознанно: клиент может использовать его для определения способа реакции.

#### Обработка ожидаемой ошибки

```sql
DO $$
BEGIN
    PERFORM 10 / 0;

EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Обнаружено деление на ноль';
END;
$$;
```

Обработчик перехватывает конкретную ошибку.

#### WHEN OTHERS

```sql
DO $$
BEGIN
    RAISE EXCEPTION 'Тестовая ошибка';

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Ошибка: %', SQLERRM;
        RAISE;
END;
$$;
```

`SQLERRM` содержит текст текущей ошибки.

`RAISE;` передаёт ошибку вызывающему уровню.

`WHEN OTHERS` не охватывает абсолютно все возможные условия PostgreSQL: некоторые специальные ошибки, например `QUERY_CANCELED`, не перехватываются этой конструкцией без явного указания.

Нежелательно перехватывать все ошибки и просто возвращать сообщение об успехе.

Такое поведение способно скрывать нарушения целостности и сбои.

#### GET STACKED DIAGNOSTICS

Более подробная диагностика:

```sql
DO $$
DECLARE
    v_message TEXT;
    v_state TEXT;
    v_detail TEXT;
BEGIN
    RAISE EXCEPTION 'Демонстрационная ошибка';

EXCEPTION
    WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_message = MESSAGE_TEXT,
            v_state = RETURNED_SQLSTATE,
            v_detail = PG_EXCEPTION_DETAIL;

        RAISE NOTICE
            'SQLSTATE = %, message = %, detail = %',
            v_state,
            v_message,
            v_detail;
END;
$$;
```

Это позволяет получить структурированную информацию об ошибке.

#### SQL Server: TRY/CATCH

```sql
BEGIN TRY
    SELECT 10 / 0;
END TRY
BEGIN CATCH
    PRINT ERROR_MESSAGE();
    THROW;
END CATCH;
```

Внутри `CATCH` доступны функции:

- `ERROR_NUMBER()`;
- `ERROR_MESSAGE()`;
- `ERROR_SEVERITY()`;
- `ERROR_STATE()`;
- `ERROR_LINE()`;
- `ERROR_PROCEDURE()`.

Не все ошибки SQL Server обрабатываются `TRY/CATCH` одинаково.

Например, отдельные ошибки компиляции на том же уровне выполнения и критические сбои имеют особое поведение.

#### MySQL: DECLARE HANDLER

В MySQL внутри сохранённой процедуры:

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    ROLLBACK;
    RESIGNAL;
END;
```

Обработчик позволяет определить реакцию на SQL-ошибку.

Для формирования собственного условия используется `SIGNAL`.

Например:

```sql
SIGNAL SQLSTATE '45000'
SET MESSAGE_TEXT = 'Недопустимая операция';
```

Код `45000` применяется для пользовательских исключений общего назначения в MySQL.

#### Oracle: EXCEPTION

В Oracle:

```sql
BEGIN
    -- основной код
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Данные не найдены');

    WHEN OTHERS THEN
        RAISE;
END;
/
```

Для генерации пользовательской ошибки может использоваться `RAISE_APPLICATION_ERROR`.

#### Ошибка не всегда требует подавления

Для неожиданных ошибок предпочтительно сохранить их для вызывающего приложения.

Процедура может:

- добавить диагностический контекст;
- освободить ресурсы;
- выполнить допустимую компенсирующую логику;
- повторно передать исключение.

При этом необходимо учитывать транзакционность побочных действий.

### Тема 7. Транзакционная логика процедур

Хранимые процедуры часто реализуют несколько связанных изменений.

Например:

```text
Проверить заказ
    ↓
Изменить статус
    ↓
Создать запись истории
    ↓
Вернуть результат
```

Эти действия должны выполняться согласованно.

Но в разных СУБД процедуры по-разному взаимодействуют с транзакциями.

#### PostgreSQL: CALL и транзакционный контекст

Процедура вызывается через:

```sql
CALL procedure_name(...);
```

Обычно её SQL-команды выполняются в транзакционном контексте вызова.

Если приложение открыло транзакцию:

```sql
BEGIN;

CALL procedure_name(...);

COMMIT;
```

изменения процедуры входят в эту транзакцию.

Это удобно, когда приложение должно объединить несколько действий в одну операцию.

#### COMMIT внутри процедуры PostgreSQL

В определённых условиях процедура PostgreSQL может выполнять:

```sql
COMMIT;
```

или:

```sql
ROLLBACK;
```

Но такое управление разрешено не всегда.

В частности, завершение транзакции недопустимо, если процедура вызвана внутри явного транзакционного блока, которым управляет вызывающий код.

Также существуют ограничения для:

- процедур `SECURITY DEFINER`;
- процедур с прикреплённой секцией `SET`;
- цепочек вызовов;
- блоков с обработкой исключений.

В PL/pgSQL нельзя завершить транзакцию внутри блока, содержащего обработчик `EXCEPTION`.

Поэтому процедура с самостоятельным `COMMIT` не является универсально вызываемой частью любой внешней транзакции.

#### Процедура без собственного COMMIT

Для многих прикладных сценариев удобнее, чтобы процедура выполняла SQL-операции, но не завершала транзакцию.

Тогда приложение может использовать:

```sql
BEGIN;

CALL update_order_status(1001, 'completed');

CALL create_delivery_task(1001);

COMMIT;
```

Если одна операция завершится ошибкой, приложение сможет отменить транзакцию целиком.

Это повышает композиционность серверной логики.

#### Обработка исключений и частичный откат

В PostgreSQL блок с `EXCEPTION` создаёт механизм под-транзакции.

Если внутри него происходит ошибка, изменения данных, выполненные в соответствующем блоке до ошибки, откатываются.

Пример:

```sql
CREATE TABLE transaction_demo (
    id INTEGER PRIMARY KEY,
    value TEXT NOT NULL
);
```

```sql
CREATE OR REPLACE PROCEDURE demo_exception_rollback()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO transaction_demo (id, value)
    VALUES (1, 'outer');

    BEGIN
        INSERT INTO transaction_demo (id, value)
        VALUES (2, 'inner');

        RAISE EXCEPTION 'Ошибка внутреннего блока';

    EXCEPTION
        WHEN OTHERS THEN
            RAISE NOTICE 'Внутренний блок отменён';
    END;
END;
$$;
```

Вызов:

```sql
CALL demo_exception_rollback();
```

После успешного завершения:

```sql
SELECT *
FROM transaction_demo
ORDER BY id;
```

вернёт строку с `id = 1`.

Строка с `id = 2` будет отменена.

Пример предполагает первоначально пустую таблицу.

#### SQL Server

В SQL Server часто используется явная транзакция:

```sql
BEGIN TRY
    BEGIN TRANSACTION;

    -- операции

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0
        ROLLBACK TRANSACTION;

    THROW;
END CATCH;
```

Но такая процедура должна учитывать, не была ли транзакция уже открыта вызывающим кодом.

Вложенные `BEGIN TRANSACTION` не создают независимо фиксируемые транзакции.

Обычный `ROLLBACK TRANSACTION` может отменить всю активную транзакцию.

Поэтому владение транзакцией необходимо проектировать явно.

#### MySQL/InnoDB

Процедура MySQL может выполнять:

```sql
START TRANSACTION;
COMMIT;
ROLLBACK;
```

с учётом ограничений и контекста.

Но `START TRANSACTION` не создаёт вложенную транзакцию: при уже активной транзакции соответствующее поведение может вызвать неявное завершение предыдущей транзакции.

MySQL не поддерживает обычные независимые вложенные транзакции.

#### Oracle

В Oracle процедура может выполнять `COMMIT` и `ROLLBACK`.

Но такая команда воздействует на текущую транзакцию сессии, включая изменения, которые вызывающий код мог выполнить ранее.

Поэтому самостоятельная фиксация внутри вспомогательной процедуры может неожиданно разрушить атомарность более крупной операции.

#### Практический принцип

Если процедура является частью транзакции, управляемой приложением, обычно не следует добавлять в неё собственный `COMMIT` без специальной причины.

Транзакционной границей должен владеть определённый уровень системы.

### Тема 8. Динамический SQL

Динамический SQL — это запрос, текст которого формируется во время исполнения.

Он нужен, когда структура запроса зависит от параметров.

Например, процедура должна обращаться к одной из нескольких таблиц, имя которой передаётся при вызове.

Или требуется создать таблицу с динамически определяемым именем.

#### Когда динамический SQL оправдан

Типичные сценарии:

- административные операции;
- работа с разными таблицами;
- динамический выбор столбцов;
- генерация DDL;
- универсальные отчёты;
- построение переменной структуры SQL-запроса.

Но динамический SQL не требуется только потому, что значение фильтра передаётся параметром.

Например:

```sql
SELECT *
FROM orders
WHERE customer_id = p_customer_id;
```

внутри PL/pgSQL уже использует значение переменной.

Не нужно превращать такой запрос в строку.

#### Небезопасная конкатенация

Опасный пример:

```sql
v_sql := 'SELECT * FROM orders WHERE status = '''
         || p_status || '''';
```

Если `p_status` содержит управляющие SQL-символы, текст запроса может измениться.

Кроме того, возможно некорректное экранирование кавычек.

При работе с недоверенными значениями такой код создаёт риск SQL-инъекции.

### Тема 9. Безопасная параметризация динамического SQL

#### PostgreSQL: EXECUTE USING

Рассмотрим процедуру с динамическим именем таблицы и параметром фильтрации.

```sql
CREATE OR REPLACE PROCEDURE count_table_rows(
    IN p_schema TEXT,
    IN p_table TEXT,
    IN p_status TEXT,
    OUT p_count BIGINT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_sql TEXT;
BEGIN
    v_sql := format(
        'SELECT COUNT(*) FROM %I.%I WHERE status = $1',
        p_schema,
        p_table
    );

    EXECUTE v_sql
    INTO p_count
    USING p_status;
END;
$$;
```

Вызов:

```sql
CALL count_table_rows(
    'public',
    'orders',
    'completed',
    NULL
);
```

Здесь:

- `%I` используется для идентификаторов;
- `$1` — параметр динамического запроса;
- `USING` передаёт значение отдельно от текста SQL;
- `INTO` получает результат.

Важно: `%I` обеспечивает корректное экранирование идентификатора, но не проверяет, разрешено ли пользователю обращаться к выбранной таблице.

Для привилегированной процедуры необходимо также ограничивать список доступных схем и объектов.

#### format()

В PostgreSQL:

| Форматтер | Назначение |
|---|---|
| `%I` | SQL-идентификатор |
| `%L` | SQL-литерал |
| `%s` | Необработанное представление значения |

Пример:

```sql
SELECT format(
    'SELECT * FROM %I.%I',
    'public',
    'orders'
);
```

Результат:

```sql
SELECT * FROM public.orders
```

При необходимости `%I` добавляет кавычки и экранирует специальные символы имени.

Однако `%I` не позволяет передавать полноценные произвольные SQL-выражения.

Например, пользовательский ввод для `ORDER BY` нельзя безопасно принимать как готовую строку с именами столбцов, направлениями сортировки и операторами.

Такие структурные элементы следует выбирать из явно разрешённого набора.

#### Динамический UPDATE

```sql
CREATE OR REPLACE PROCEDURE update_text_column(
    IN p_schema TEXT,
    IN p_table TEXT,
    IN p_column TEXT,
    IN p_id BIGINT,
    IN p_value TEXT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_sql TEXT;
BEGIN
    v_sql := format(
        'UPDATE %I.%I SET %I = $1 WHERE id = $2',
        p_schema,
        p_table,
        p_column
    );

    EXECUTE v_sql
    USING p_value, p_id;
END;
$$;
```

Этот пример демонстрирует параметризацию.

Но он **не является готовым безопасным универсальным API**.

Для реального применения необходимо:

- проверять допустимые таблицы;
- проверять допустимые столбцы;
- контролировать права;
- проверять типы;
- определять ожидаемое количество изменённых строк;
- исключать изменение системных или чувствительных данных;
- учитывать конкурентные операции.

#### SQL Server: sp_executesql

```sql
DECLARE @sql NVARCHAR(MAX);
DECLARE @table SYSNAME = N'orders';
DECLARE @status NVARCHAR(30) = N'completed';

SET @sql =
    N'SELECT COUNT(*) FROM dbo.'
    + QUOTENAME(@table)
    + N' WHERE status = @p_status';

EXEC sys.sp_executesql
    @sql,
    N'@p_status NVARCHAR(30)',
    @p_status = @status;
```

`QUOTENAME` используется для идентификатора.

Значение статуса передаётся через параметры `sp_executesql`.

Для идентификаторов и значений требуются разные механизмы обработки.

#### MySQL: PREPARE

В MySQL динамический SQL может использовать:

```sql
PREPARE
EXECUTE
DEALLOCATE PREPARE
```

Пример:

```sql
SET @sql = 'SELECT COUNT(*) FROM orders WHERE status = ?';
SET @status = 'completed';

PREPARE stmt FROM @sql;

EXECUTE stmt USING @status;

DEALLOCATE PREPARE stmt;
```

Подстановка значений через placeholders не означает возможность таким же способом подставлять имена таблиц и столбцов.

Идентификаторы необходимо формировать отдельно с обязательной проверкой допустимости.

В MySQL нет прямого аналога PostgreSQL `format('%I', ...)`, полностью решающего эту задачу.

#### Oracle: EXECUTE IMMEDIATE

Oracle поддерживает:

```sql
EXECUTE IMMEDIATE
```

Например:

```sql
DECLARE
    v_count NUMBER;
BEGIN
    EXECUTE IMMEDIATE
        'SELECT COUNT(*) FROM orders WHERE status = :status'
    INTO v_count
    USING 'completed';

    DBMS_OUTPUT.PUT_LINE(v_count);
END;
/
```

Для проверки и безопасной работы с динамическими идентификаторами Oracle предоставляет, в частности, пакет `DBMS_ASSERT`.

Но и в Oracle безопасное экранирование имени не заменяет проверку прав и допустимости операции.

#### Общий принцип безопасности

Параметризация защищает **значения** от интерпретации как SQL-кода.

Корректное экранирование защищает **идентификаторы** от нарушения синтаксиса и внедрения дополнительных SQL-конструкций.

Проверка допустимости определяет, **разрешено ли вообще выполнять операцию над указанным объектом**.

Для безопасного динамического SQL нужны все три аспекта.

### Тема 10. Практика: учебная база для процедуры расчёта прогресса

Рассмотрим учебную систему, содержащую:

- студентов;
- курсы;
- записи на курсы;
- задания;
- результаты проверок;
- журнал активности.

Для воспроизводимости примера используем отдельную схему.

```sql
CREATE SCHEMA IF NOT EXISTS lesson17;
```

#### Студенты

```sql
CREATE TABLE lesson17.students (
    student_id BIGINT PRIMARY KEY,
    student_name TEXT NOT NULL
);
```

#### Курсы

```sql
CREATE TABLE lesson17.courses (
    course_id BIGINT PRIMARY KEY,
    course_name TEXT NOT NULL
);
```

#### Запись на курс

```sql
CREATE TABLE lesson17.enrollments (
    student_id BIGINT NOT NULL
        REFERENCES lesson17.students(student_id),
    course_id BIGINT NOT NULL
        REFERENCES lesson17.courses(course_id),
    PRIMARY KEY (student_id, course_id)
);
```

#### Задания

```sql
CREATE TABLE lesson17.assignments (
    assignment_id BIGINT PRIMARY KEY,
    course_id BIGINT NOT NULL
        REFERENCES lesson17.courses(course_id),
    assignment_name TEXT NOT NULL,
    max_score NUMERIC(10, 2) NOT NULL
        CHECK (max_score > 0)
);
```

#### Результаты

```sql
CREATE TABLE lesson17.submissions (
    submission_id BIGINT PRIMARY KEY,
    student_id BIGINT NOT NULL
        REFERENCES lesson17.students(student_id),
    assignment_id BIGINT NOT NULL
        REFERENCES lesson17.assignments(assignment_id),
    score NUMERIC(10, 2),
    status TEXT NOT NULL
        CHECK (
            status IN ('submitted', 'reviewed', 'rejected')
        ),
    CHECK (score IS NULL OR score >= 0)
);
```

#### Прогресс студентов

```sql
CREATE TABLE lesson17.student_progress (
    student_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    progress_percent NUMERIC(5, 2) NOT NULL
        CHECK (progress_percent BETWEEN 0 AND 100),
    calculated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id, course_id)
        REFERENCES lesson17.enrollments(student_id, course_id)
);
```

#### Журнал активности

```sql
CREATE TABLE lesson17.activity_log (
    log_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    student_id BIGINT NOT NULL
        REFERENCES lesson17.students(student_id),
    action_type TEXT NOT NULL,
    details JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);
```

#### Исходные данные

```sql
INSERT INTO lesson17.students
VALUES
    (1, 'Анна'),
    (2, 'Борис');

INSERT INTO lesson17.courses
VALUES
    (10, 'Основы SQL');

INSERT INTO lesson17.enrollments
VALUES
    (1, 10),
    (2, 10);

INSERT INTO lesson17.assignments
VALUES
    (101, 10, 'SELECT', 20),
    (102, 10, 'JOIN', 30),
    (103, 10, 'Транзакции', 50);

INSERT INTO lesson17.submissions
VALUES
    (1, 1, 101, 15, 'reviewed'),
    (2, 1, 102, 20, 'reviewed'),
    (3, 1, 102, 25, 'reviewed'),
    (4, 1, 103, NULL, 'submitted'),
    (5, 2, 101, 20, 'reviewed');
```

Для каждого задания будем учитывать лучший проверенный результат студента.

Максимум по курсу:

```text
20 + 30 + 50 = 100
```

Анна получила:

```text
15 + 25 + 0 = 40
```

Прогресс:

```text
40%
```

Борис получил:

```text
20 + 0 + 0 = 20
```

Прогресс:

```text
20%
```

### Тема 11. Процедура расчёта и сохранения прогресса

Создадим процедуру, которая:

1. Проверяет существование студента.
2. Проверяет существование курса.
3. Проверяет запись студента на курс.
4. Рассчитывает прогресс.
5. Сохраняет результат.
6. Записывает событие.
7. Возвращает процент через `OUT`-параметр.

```sql
CREATE OR REPLACE PROCEDURE lesson17.recalculate_student_progress(
    IN p_student_id BIGINT,
    IN p_course_id BIGINT,
    OUT p_progress_percent NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_total_possible NUMERIC := 0;
    v_total_earned NUMERIC := 0;
BEGIN
    IF NOT EXISTS (
        SELECT 1
        FROM lesson17.students
        WHERE student_id = p_student_id
    ) THEN
        RAISE EXCEPTION
            'Студент % не существует',
            p_student_id
            USING ERRCODE = 'P0001';
    END IF;

    IF NOT EXISTS (
        SELECT 1
        FROM lesson17.courses
        WHERE course_id = p_course_id
    ) THEN
        RAISE EXCEPTION
            'Курс % не существует',
            p_course_id
            USING ERRCODE = 'P0001';
    END IF;

    IF NOT EXISTS (
        SELECT 1
        FROM lesson17.enrollments
        WHERE student_id = p_student_id
          AND course_id = p_course_id
    ) THEN
        RAISE EXCEPTION
            'Студент % не записан на курс %',
            p_student_id,
            p_course_id
            USING ERRCODE = 'P0001';
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
        v_total_possible,
        v_total_earned
    FROM lesson17.assignments AS a
    LEFT JOIN LATERAL (
        SELECT MAX(s.score) AS best_score
        FROM lesson17.submissions AS s
        WHERE s.assignment_id = a.assignment_id
          AND s.student_id = p_student_id
          AND s.status = 'reviewed'
    ) AS best ON TRUE
    WHERE a.course_id = p_course_id;

    IF v_total_possible = 0 THEN
        p_progress_percent := 0;
    ELSE
        p_progress_percent := ROUND(
            v_total_earned * 100 / v_total_possible,
            2
        );
    END IF;

    INSERT INTO lesson17.student_progress (
        student_id,
        course_id,
        progress_percent,
        calculated_at
    )
    VALUES (
        p_student_id,
        p_course_id,
        p_progress_percent,
        clock_timestamp()
    )
    ON CONFLICT (student_id, course_id)
    DO UPDATE
    SET
        progress_percent = EXCLUDED.progress_percent,
        calculated_at = EXCLUDED.calculated_at;

    INSERT INTO lesson17.activity_log (
        student_id,
        action_type,
        details
    )
    VALUES (
        p_student_id,
        'progress_recalculated',
        jsonb_build_object(
            'course_id', p_course_id,
            'progress_percent', p_progress_percent
        )
    );

    RAISE NOTICE
        'Прогресс студента % по курсу %: % процентов',
        p_student_id,
        p_course_id,
        p_progress_percent;
END;
$$;
```

#### Вызов

```sql
CALL lesson17.recalculate_student_progress(
    1,
    10,
    NULL
);
```

Результат:

| p_progress_percent |
|---:|
| 40.00 |

Для Бориса:

```sql
CALL lesson17.recalculate_student_progress(
    2,
    10,
    NULL
);
```

Результат:

| p_progress_percent |
|---:|
| 20.00 |

#### Проверка сохранённых результатов

```sql
SELECT *
FROM lesson17.student_progress
ORDER BY student_id, course_id;
```

#### Проверка журнала

```sql
SELECT
    student_id,
    action_type,
    details,
    created_at
FROM lesson17.activity_log
ORDER BY log_id;
```

#### Почему используется ON CONFLICT

Если прогресс студента уже рассчитан, повторный вызов должен обновить существующую запись, а не создавать конфликт первичного ключа.

Поэтому используется:

```sql
INSERT ... ON CONFLICT ... DO UPDATE
```

Однако процедура в целом не полностью идемпотентна.

Хотя итоговая запись прогресса заменяется новым значением, повторный вызов создаёт новую запись активности и обновляет время расчёта.

Если бизнес-операция должна быть полностью идемпотентной, необходимо дополнительно определить идентификатор операции и правила обработки повторов.

#### Конкурентное выполнение

Если два процесса одновременно вызовут процедуру для одного студента и курса, `ON CONFLICT` обеспечит корректное разрешение конфликта уникального ключа при записи результата.

Но это не гарантирует, что оба процесса вычисляли прогресс на одинаковом снимке исходных данных.

Если за время выполнения изменяются результаты заданий, итоговый прогресс может зависеть от порядка транзакций.

Для критичных сценариев необходимо отдельно определить требования к изоляции и согласованности расчёта.

### Тема 12. Логирование ошибок и ограничения транзакционного аудита

Исходный сценарий предусматривает сохранение ошибок процедуры в журнал.

Для этого можно создать таблицу:

```sql
CREATE TABLE lesson17.error_log (
    error_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    operation_name TEXT NOT NULL,
    error_state TEXT,
    error_message TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
        DEFAULT clock_timestamp()
);
```

Рассмотрим обработчик:

```sql
EXCEPTION
    WHEN OTHERS THEN
        INSERT INTO lesson17.error_log (
            operation_name,
            error_state,
            error_message
        )
        VALUES (
            'recalculate_student_progress',
            SQLSTATE,
            SQLERRM
        );

        RAISE;
```

На первый взгляд такая конструкция позволяет сохранить ошибку.

Но существует важная особенность.

Если после записи выполняется `RAISE`, ошибка передаётся вызывающему коду.

При откате внешней транзакции запись в `error_log` также будет отменена.

Поэтому нельзя утверждать, что этот вариант гарантированно сохраняет журнал неуспешных операций.

#### Когда запись может сохраниться

Если обработчик перехватывает ошибку, записывает событие и завершает процедуру без повторной генерации исключения, запись может быть подтверждена вместе с транзакцией.

Но в этом случае вызывающий код должен явно получить информацию о неуспешном результате.

Иначе возможна ситуация, когда процедура фактически не выполнила операцию, но приложение считает вызов успешным.

#### Почему нельзя просто выполнить COMMIT в обработчике

В PostgreSQL блок с `EXCEPTION` использует механизм под-транзакции.

Транзакционное завершение внутри такого блока запрещено.

Поэтому конструкция вида:

```sql
EXCEPTION
    WHEN OTHERS THEN
        INSERT INTO error_log (...);
        COMMIT;
```

не является допустимым универсальным способом независимого сохранения ошибок.

#### Как организовать надёжное журналирование

Возможные варианты:

- сохранить событие об успешной операции в той же транзакции;
- передать ошибку приложению для внешнего логирования;
- использовать журнал PostgreSQL;
- применять централизованную систему логирования;
- формировать события через transactional outbox;
- использовать специализированный аудит.

Выбор зависит от того, какие именно события необходимо сохранять.

**Аудит подтверждённых изменений и журнал всех попыток выполнения операций — разные задачи.**

### Тема 13. Безопасность хранимых процедур

Процедуры могут предоставлять контролируемый интерфейс доступа к данным.

Например, приложение может иметь право вызвать:

```sql
CALL complete_order(1001);
```

без разрешения выполнять произвольные UPDATE в таблице заказов.

Для этого в PostgreSQL может использоваться `SECURITY DEFINER`.

Однако выполнение с правами владельца требует особого внимания.

#### SECURITY INVOKER

Это режим PostgreSQL по умолчанию.

Процедура использует права вызывающего пользователя.

#### SECURITY DEFINER

Процедура выполняется с правами владельца.

Это позволяет ограничить прямой доступ к таблицам и предоставить пользователю только конкретные операции.

Но при ошибке проектирования такой интерфейс может расширить привилегии вызывающего пользователя сверх необходимого.

#### Основные требования

Для привилегированных процедур необходимо:

1. Использовать минимально необходимые права владельца.
2. Ограничивать `EXECUTE`.
3. Исключать недоверенные схемы из `search_path`.
4. Использовать квалифицированные имена объектов.
5. Защищать динамический SQL.
6. Проверять идентификаторы и допустимость операций.
7. Не доверять произвольным значениям, переданным пользователем.
8. Контролировать изменение самой процедуры.

Пример ограничения прав:

```sql
REVOKE ALL
ON PROCEDURE lesson17.recalculate_student_progress(
    BIGINT,
    BIGINT
)
FROM PUBLIC;
```

Выдача конкретной роли:

```sql
GRANT EXECUTE
ON PROCEDURE lesson17.recalculate_student_progress(
    BIGINT,
    BIGINT
)
TO application_role;
```

Предполагается, что роль `application_role` существует и имеет необходимые права на схему.

Если процедура работает в режиме `SECURITY INVOKER`, одной выдачи EXECUTE недостаточно для доступа к базовым таблицам без соответствующих прав.

Также важно: PostgreSQL запрещает транзакционное управление внутри процедур `SECURITY DEFINER`.

Поэтому нельзя одновременно считать такую процедуру универсальным привилегированным интерфейсом и предполагать, что она будет самостоятельно выполнять COMMIT.

### Тема 14. Архитектурные границы процедур

Хранимые процедуры позволяют переносить часть прикладной логики на уровень СУБД.

Это оправданно не во всех случаях.

#### Когда процедура подходит

Хранимая процедура полезна, если:

- операция требует нескольких взаимосвязанных изменений;
- важно уменьшить число сетевых обращений;
- несколько приложений должны использовать одинаковые правила;
- требуется единый транзакционный контекст;
- операция тесно связана с ограничениями и состоянием данных;
- нужен ограниченный интерфейс для изменения таблиц;
- выполняется специализированная административная операция.

#### Когда процедура не нужна

Если задача решается одним SQL-запросом, создание процедуры может добавить ненужную сложность.

Например:

```sql
UPDATE products
SET price = price * 1.10
WHERE category_id = 5;
```

обычно не требует цикла внутри процедуры.

Простое вычисление:

```sql
price * quantity
```

также не обязательно выносить в отдельную хранимую процедуру.

#### Процедуры и внешние сервисы

Нежелательно держать длительную транзакцию открытой во время ожидания:

- HTTP-запроса;
- подтверждения оплаты;
- пользовательского действия;
- ответа удалённого сервиса;
- длительного вычисления вне СУБД.

Такие действия могут увеличивать время удержания блокировок и усложнять восстановление после сбоев.

Для распределённых процессов лучше использовать подходящую архитектуру с событиями, очередями, идемпотентностью и явным управлением состояниями.

#### Производительность

Процедура может ускорить операцию за счёт уменьшения сетевых обращений.

Но стоимость SQL-команд внутри процедуры никуда не исчезает.

Например, процедура:

```text
Получить 100 000 строк
    ↓
Для каждой выполнить UPDATE
```

может работать значительно хуже одного множественного UPDATE.

Поэтому необходимо анализировать:

- количество выполняемых SQL-команд;
- использование индексов;
- планы выполнения;
- блокировки;
- объём журналирования;
- длительность транзакции;
- конкурентную нагрузку.

В PostgreSQL для диагностики могут использоваться `pg_stat_statements`, `auto_explain` и другие инструменты при соответствующей настройке.

#### Сопровождение

Определения процедур должны храниться в Git.

Например:

```text
database/
  migrations/
    V001__create_tables.sql
    V002__create_progress_procedure.sql
    V003__add_error_handling.sql
  routines/
    recalculate_student_progress.sql
  tests/
    test_progress.sql
```

Для процедуры необходимо тестировать:

- корректные входные данные;
- отсутствующие объекты;
- нарушение бизнес-условий;
- повторные вызовы;
- конкурентное выполнение;
- ошибки SQL;
- права доступа;
- откат транзакции;
- поведение параметров результата.

Процедура является частью публичного интерфейса базы данных, поэтому изменение её сигнатуры и поведения необходимо сопровождать как изменение программного API.

## Итоговые выводы

1. **Хранимая процедура объединяет последовательность действий в именованный объект СУБД.** Она подходит для многошаговых операций, связанных с данными и транзакционной логикой.

2. **Процедуры и функции различаются интерфейсом и способом вызова.** PostgreSQL использует `CALL` для процедур и SQL-выражения для функций, однако функции также могут иметь побочные эффекты.

3. **Параметры формируют контракт процедуры.** Входные, выходные и комбинированные параметры позволяют передавать данные и возвращать результаты. Реализация этих механизмов различается между СУБД.

4. **Диагностические сообщения полезны при отладке, но не заменяют формализованный результат.** Для передачи статуса операции лучше использовать выходные параметры или предусмотренные механизмы ошибок.

5. **Обработка исключений должна сохранять корректную семантику операции.** Непредвиденные ошибки не следует подавлять без явного решения.

6. **Транзакционные возможности процедур зависят от СУБД и контекста вызова.** Процедура не получает автоматически независимую транзакцию.

7. **В PostgreSQL транзакционное управление внутри процедур имеет ограничения.** В частности, оно недопустимо в определённых контекстах вызова, в процедурах `SECURITY DEFINER` и внутри блоков с обработчиками исключений.

8. **Обычное логирование ошибки в таблицу транзакционно.** Если транзакция откатывается, соответствующая запись журнала может быть потеряна вместе с изменениями.

9. **Динамический SQL нужен для изменяемой структуры запроса, а не для обычной передачи значений.** В PostgreSQL значения следует передавать через `EXECUTE ... USING`, а идентификаторы безопасно форматировать.

10. **Экранирование и параметризация не заменяют проверку полномочий.** Даже синтаксически безопасный динамический запрос может предоставлять недопустимую операцию над защищёнными объектами.

11. **Процедуры могут уменьшать сетевые обращения, но не гарантируют ускорения SQL.** Для массовой обработки по возможности следует использовать множественные операции вместо построчных циклов.

12. **Хранимые процедуры требуют тестирования, документирования и версионирования.** Они должны быть частью управляемых миграций и общей архитектуры приложения.

Главный результат занятия — понимание хранимой процедуры как управляемого интерфейса выполнения операций внутри СУБД, с чётко определёнными параметрами, транзакционными границами, обработкой ошибок и требованиями безопасности.

## Дополнительные материалы

### PostgreSQL

- [CREATE PROCEDURE](https://www.postgresql.org/docs/current/sql-createprocedure.html)
- [CALL](https://www.postgresql.org/docs/current/sql-call.html)
- [PL/pgSQL — SQL Procedural Language](https://www.postgresql.org/docs/current/plpgsql.html)
- [Basic Statements](https://www.postgresql.org/docs/current/plpgsql-statements.html)
- [Control Structures and Exception Handling](https://www.postgresql.org/docs/current/plpgsql-control-structures.html)
- [Transaction Management](https://www.postgresql.org/docs/current/plpgsql-transactions.html)
- [Errors and Messages](https://www.postgresql.org/docs/current/plpgsql-errors-and-messages.html)
- [Function and Procedure Security](https://www.postgresql.org/docs/current/perm-functions.html)

### Microsoft SQL Server

- [CREATE PROCEDURE](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-procedure-transact-sql)
- [EXECUTE](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/execute-transact-sql)
- [sp_executesql](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-executesql-transact-sql)
- [QUOTENAME](https://learn.microsoft.com/en-us/sql/t-sql/functions/quotename-transact-sql)
- [TRY...CATCH](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/try-catch-transact-sql)
- [THROW](https://learn.microsoft.com/en-us/sql/t-sql/language-elements/throw-transact-sql)
- [XACT_STATE](https://learn.microsoft.com/en-us/sql/t-sql/functions/xact-state-transact-sql)

### MySQL/InnoDB

- [CREATE PROCEDURE](https://dev.mysql.com/doc/refman/8.4/en/create-procedure.html)
- [CALL](https://dev.mysql.com/doc/refman/8.4/en/call.html)
- [Stored Program Syntax](https://dev.mysql.com/doc/refman/8.4/en/sql-compound-statements.html)
- [DECLARE HANDLER](https://dev.mysql.com/doc/refman/8.4/en/declare-handler.html)
- [SIGNAL](https://dev.mysql.com/doc/refman/8.4/en/signal.html)
- [Prepared Statements](https://dev.mysql.com/doc/refman/8.4/en/sql-prepared-statements.html)
- [Stored Program Restrictions](https://dev.mysql.com/doc/refman/8.4/en/stored-program-restrictions.html)

### Oracle

- [PL/SQL Language Reference](https://docs.oracle.com/en/database/oracle/oracle-database/23/lnpls/)
- [PL/SQL Subprograms](https://docs.oracle.com/en/database/oracle/oracle-database/23/lnpls/plsql-subprograms.html)
- [PL/SQL Error Handling](https://docs.oracle.com/en/database/oracle/oracle-database/23/lnpls/plsql-error-handling.html)
- [Dynamic SQL](https://docs.oracle.com/en/database/oracle/oracle-database/23/lnpls/dynamic-sql.html)
- [DBMS_ASSERT](https://docs.oracle.com/en/database/oracle/oracle-database/23/arpls/DBMS_ASSERT.html)
