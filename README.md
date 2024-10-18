# Oracle_labs

## Вариант 7
## Лабораторная 3: Пакет с функцией и процедурой

```sql
-- Создаем пакет
CREATE OR REPLACE PACKAGE SEMESTER_PACKAGE AS
  FUNCTION count_subjects_avg(semester_number NUMBER) RETURN NUMBER;
  PROCEDURE log_function_calls;
END SEMESTER_PACKAGE;
/
-- Создаем тело пакета
CREATE OR REPLACE PACKAGE BODY SEMESTER_PACKAGE AS
  v_function_calls NUMBER := 0; -- Переменная для подсчета вызовов функции
  
  -- Функция подсчета количества предметов и среднего балла по семестру
  FUNCTION count_subjects_avg(semester_number NUMBER) RETURN NUMBER IS
    v_subject_count NUMBER;
    v_avg_score NUMBER;
  BEGIN
    -- Подсчитываем количество предметов и средний балл за указанный семестр
    SELECT COUNT(*), AVG(score)
    INTO v_subject_count, v_avg_score
    FROM EXAM_MARKS
    WHERE semester = semester_number;

    v_function_calls := v_function_calls + 1; -- Увеличиваем счетчик вызовов функции

    RETURN v_avg_score;
  END count_subjects_avg;

  -- Процедура для логирования вызовов функции
  PROCEDURE log_function_calls IS
  BEGIN
    INSERT INTO FUNCTION_LOGS(semester_number, avg_score, subject_count, function_calls)
    SELECT semester, AVG(score), COUNT(*), v_function_calls
    FROM EXAM_MARKS
    GROUP BY semester;
  END log_function_calls;

END SEMESTER_PACKAGE;
/
```

## Лабораторная 4: Триггер для проверки стипендии

```sql
-- Создаем триггер для контроля стипендии
CREATE OR REPLACE TRIGGER check_scholarship
  BEFORE INSERT OR UPDATE ON STUDENT_SCHOLARSHIPS
  FOR EACH ROW
DECLARE
  v_max_scholarship NUMBER;
  scholarship_deviation EXCEPTION;
BEGIN
  -- Определяем максимальную стипендию
  SELECT MAX(scholarship)
  INTO v_max_scholarship
  FROM STUDENT_SCHOLARSHIPS;

  -- Проверяем отклонение от максимальной стипендии
  IF :NEW.scholarship > v_max_scholarship * 1.1 THEN
    RAISE scholarship_deviation;
  END IF;

EXCEPTION
  WHEN scholarship_deviation THEN
    DBMS_OUTPUT.PUT_LINE('Ошибка: превышение порога стипендии.');
    -- Записываем данные о превышении в таблицу
    INSERT INTO SCHOLARSHIP_LOG(student_id, old_scholarship, new_scholarship)
    VALUES (:NEW.student_id, v_max_scholarship, :NEW.scholarship);
END;
/
```

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
