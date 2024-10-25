# Oracle_labs

## Вариант 7
## Лабораторная 3: Пакет с функцией и процедурой
Создать пакет, состоящий из функции с параметрами, процедуры без параметров. Функция подсчитывает количество предметов, прочитанных в
семестре, номер которого задан параметром и средний балл, полученный
по этому предмету. Процедура подсчитывает количество обращений к
функции и заносит это количество, номер семестра, средний балл и количество предметов в новую таблицу, созданную заранее. 

## Шаг 1: Создаем таблицы
Таблица предметов (SUBJECTS) – хранит информацию о предметах, пройденных в разных семестрах, и средний балл по каждому предмету.
Таблица логирования обращений к функции (FUNCTION_CALL_LOG) – хранит данные о количестве вызовов функции, номере семестра, среднем балле и количестве предметов.
```sql
-- Создаем таблицу SUBJECTS
CREATE TABLE SUBJECTS (
    subject_id NUMBER PRIMARY KEY,
    subject_name VARCHAR2(50) NOT NULL,
    semester NUMBER NOT NULL,
    average_score NUMBER(3, 2) NOT NULL
);

-- Заполняем таблицу SUBJECTS тестовыми данными
INSERT INTO SUBJECTS (subject_id, subject_name, semester, average_score) VALUES (1, 'Mathematics', 1, 4.5);
INSERT INTO SUBJECTS (subject_id, subject_name, semester, average_score) VALUES (2, 'Physics', 1, 3.8);
INSERT INTO SUBJECTS (subject_id, subject_name, semester, average_score) VALUES (3, 'Chemistry', 2, 4.2);
INSERT INTO SUBJECTS (subject_id, subject_name, semester, average_score) VALUES (4, 'Biology', 2, 4.7);
INSERT INTO SUBJECTS (subject_id, subject_name, semester, average_score) VALUES (5, 'Computer Science', 3, 4.9);

-- Создаем таблицу FUNCTION_CALL_LOG для записи обращений к функции
CREATE TABLE FUNCTION_CALL_LOG (
    log_id NUMBER PRIMARY KEY,
    semester NUMBER,
    item_count NUMBER,
    average_score NUMBER(3, 2),
    call_count NUMBER
);

```
## Шаг 2: Создаем пакет с функцией и процедурой
Функция count_subjects_in_semester – принимает номер семестра и возвращает количество предметов в этом семестре и средний балл.
Процедура log_function_call – записывает количество вызовов функции, номер семестра, средний балл и количество предметов в таблицу FUNCTION_CALL_LOG.
```sql
-- Создаем пакет
CREATE OR REPLACE PACKAGE subject_package AS
    FUNCTION count_subjects_in_semester(p_semester NUMBER) RETURN NUMBER;
    PROCEDURE log_function_call;
END subject_package;
/

-- Реализуем тело пакета
CREATE OR REPLACE PACKAGE BODY subject_package AS
    -- Переменная для отслеживания количества вызовов функции
    call_counter NUMBER := 0;

    -- Функция для подсчета предметов и среднего балла по заданному семестру
    FUNCTION count_subjects_in_semester(p_semester NUMBER) RETURN NUMBER IS
        v_item_count NUMBER := 0;
        v_total_score NUMBER := 0;
        v_avg_score NUMBER := 0;
    BEGIN
        -- Подсчет количества предметов и суммирование баллов
        SELECT COUNT(*), NVL(SUM(average_score), 0)
        INTO v_item_count, v_total_score
        FROM SUBJECTS
        WHERE semester = p_semester;

        -- Вычисление среднего балла
        IF v_item_count > 0 THEN
            v_avg_score := v_total_score / v_item_count;
        END IF;

        -- Увеличение счетчика вызова функции
        call_counter := call_counter + 1;

        RETURN v_item_count;
    END count_subjects_in_semester;

    -- Процедура для записи информации о вызове функции в таблицу
    PROCEDURE log_function_call IS
        v_last_semester NUMBER := 1;  -- Начальное значение
        v_item_count NUMBER := 0;
        v_avg_score NUMBER := 0;
    BEGIN
        -- Получение информации о последнем вызове функции
        SELECT semester, COUNT(*), AVG(average_score)
        INTO v_last_semester, v_item_count, v_avg_score
        FROM SUBJECTS
        WHERE semester = v_last_semester;

        -- Запись данных в таблицу FUNCTION_CALL_LOG
        INSERT INTO FUNCTION_CALL_LOG (log_id, semester, item_count, average_score, call_count)
        VALUES (FUNCTION_CALL_LOG_SEQ.NEXTVAL, v_last_semester, v_item_count, v_avg_score, call_counter);

        -- Сброс счетчика вызова функции
        call_counter := 0;
    END log_function_call;

END subject_package;
/
```

## Шаг 3: Проверка работы
Теперь, когда таблицы и пакет созданы, можно протестировать функцию и процедуру.
```sql
-- Вызов функции для подсчета предметов в семестре
DECLARE
    item_count NUMBER;
BEGIN
    item_count := subject_package.count_subjects_in_semester(1);
    DBMS_OUTPUT.PUT_LINE('Количество предметов в семестре: ' || item_count);
END;
/

-- Вызов процедуры для записи информации о вызове функции
BEGIN
    subject_package.log_function_call;
END;
/

```

## Лабораторная 4: Триггер для проверки стипендии

Создать триггер, который считает максимальный стипендию и выдает
диагностическое сообщение при превышении заданного порога
уклонения вводимого значения атрибута от максимальной стипендии,
при этом происходит заполнение некоторой таблицы.

## Шаг 1: Создаем таблицы
```sql
-- Создаем таблицу для хранения данных о стипендиях
CREATE TABLE STIPENDS (
    student_id NUMBER PRIMARY KEY,
    student_name VARCHAR2(50) NOT NULL,
    stipend_amount NUMBER(10, 2) NOT NULL
);

-- Создаем таблицу для записи диагностических сообщений
CREATE TABLE STIPEND_LOG (
    log_id NUMBER PRIMARY KEY,
    student_id NUMBER,
    entered_stipend NUMBER(10, 2),
    max_stipend NUMBER(10, 2),
    deviation_percentage NUMBER(5, 2),
    log_message VARCHAR2(255),
    log_date DATE DEFAULT SYSDATE
);

```
## Шаг 2: Создаем триггер для проверки порога отклонения
Для определения порога отклонения будем использовать переменную stipend_threshold (например, 20%). Если значение новой стипендии превышает текущую максимальную стипендию на указанный процент, триггер добавит запись в таблицу STIPEND_LOG с диагностическим сообщением.
```sql
-- Устанавливаем порог отклонения для проверки стипендии
DECLARE
    stipend_threshold CONSTANT NUMBER := 20; -- 20% отклонения
END;
/

-- Создаем триггер, который срабатывает при добавлении или изменении данных в таблице STIPENDS
CREATE OR REPLACE TRIGGER check_stipend_threshold
BEFORE INSERT OR UPDATE OF stipend_amount ON STIPENDS
FOR EACH ROW
DECLARE
    v_max_stipend NUMBER(10, 2);
    v_deviation_percentage NUMBER(5, 2);
BEGIN
    -- Определяем текущую максимальную стипендию в таблице STIPENDS
    SELECT NVL(MAX(stipend_amount), 0) INTO v_max_stipend FROM STIPENDS;

    -- Проверяем, если введенная стипендия превышает максимальную на установленный порог
    IF :NEW.stipend_amount > v_max_stipend THEN
        v_deviation_percentage := ((:NEW.stipend_amount - v_max_stipend) / v_max_stipend) * 100;

        IF v_deviation_percentage > stipend_threshold THEN
            -- Запись в таблицу STIPEND_LOG диагностического сообщения
            INSERT INTO STIPEND_LOG (log_id, student_id, entered_stipend, max_stipend, deviation_percentage, log_message)
            VALUES (
                STIPEND_LOG_SEQ.NEXTVAL,
                :NEW.student_id,
                :NEW.stipend_amount,
                v_max_stipend,
                v_deviation_percentage,
                'Warning: Stipend exceeds maximum by more than ' || stipend_threshold || '%'
            );

            -- Вывод диагностического сообщения
            DBMS_OUTPUT.PUT_LINE('Warning: Entered stipend exceeds maximum stipend by more than ' || stipend_threshold || '%');
        END IF;
    END IF;
END;
/
```
## Шаг 3: Проверка работы триггера
Заполним таблицу тестовыми данными и проверим работу триггера.
```sql
-- Вставка данных в таблицу STIPENDS, чтобы протестировать триггер
INSERT INTO STIPENDS (student_id, student_name, stipend_amount) VALUES (1, 'Alice', 5000);
INSERT INTO STIPENDS (student_id, student_name, stipend_amount) VALUES (2, 'Bob', 6000);

-- Вставка значения стипендии, превышающего порог отклонения от максимальной
INSERT INTO STIPENDS (student_id, student_name, stipend_amount) VALUES (3, 'Charlie', 8000);
```
В случае, если стипендия превышает установленный порог отклонения от максимальной, триггер запишет соответствующую информацию в таблицу STIPEND_LOG и выведет диагностическое сообщение в DBMS_OUTPUT.

## Лабораторная 5: Создание таблицы с использованием последовательностей

```sql
-- Создаем последовательность
CREATE SEQUENCE student_seq
  START WITH 1
  INCREMENT BY 1
  NOCACHE;

-- Создаем таблицу
CREATE TABLE STUDENTS (
  student_id NUMBER PRIMARY KEY,
  first_name VARCHAR2(50),
  last_name VARCHAR2(50),
  enrollment_date DATE
);

-- Вставляем данные с использованием последовательности
INSERT INTO STUDENTS (student_id, first_name, last_name, enrollment_date)
VALUES (student_seq.NEXTVAL, 'Иван', 'Иванов', SYSDATE);

INSERT INTO STUDENTS (student_id, first_name, last_name, enrollment_date)
VALUES (student_seq.NEXTVAL, 'Петр', 'Петров', SYSDATE);

```
