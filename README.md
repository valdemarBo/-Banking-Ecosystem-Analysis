# Анализ банковской экосистемы

Комплексный SQL-анализ банковской экосистемы: 10 таблиц (клиенты, счета, транзакции, кредиты, карты, отделения, сотрудники, поддержка). Оценка CLV, кредитных рисков (DTI/NPL), эффективности отделений, мошенничества и качества поддержки.

**Инструменты:** PostgreSQL, DBeaver, Power BI

---

## Введение

Данный проект представляет собой комплексный анализ реалистичной синтетической банковской экосистемы с использованием SQL. Структурированный по десяти основным сущностям — охватывающим демографические данные клиентов, счета, транзакции, кредитные карты, кредиты, сотрудников, отделения и операции поддержки — набор данных моделирует сложные транзакционные рабочие процессы и институциональные иерархии. Путём выполнения запросов к этим взаимосвязанным сущностям данный портфолио-проект демонстрирует практические навыки построения запросов к базам данных, преобразования данных и применения методов бизнес-аналитики для извлечения практически значимых финансовых инсайтов.

---

## Цели исследования

### 1. Исследовательский анализ данных (EDA)
Проверка качества данных и целостности связей: валидация данных, обработка пропущенных значений (NULL) и проверка связей по внешним ключам для обеспечения точности данных перед последующей визуализацией.

### 2. Анализ транзакций и денежных потоков
Написание сложных SQL-запросов с использованием агрегатных функций, оконных функций и обобщённых табличных выражений (CTE) для оценки структуры депозитов, снятий и погашений кредитов в разрезе отделений.

### 3. Сегментация клиентов и отслеживание поведения
Выявление высокодоходных клиентов, анализ тенденций использования кредитных карт и оценка уровней активности по счетам с использованием многотабличных соединений и фильтрации.

### 4. Управление рисками и кредитным портфелем
Запросы по распределению кредитов, остаткам задолженности и графикам погашения для оценки уровня риска и доли просроченной задолженности.

### 5. Эффективность отделений и сотрудников
Агрегация объёмов транзакций и метрик по обработке обращений в службу поддержки для измерения операционной эффективности в различных отделениях.
## Введение: Загрузка данных и создание базы данных

### Исходные данные

Проект основан на **10 CSV-файлах**, которые моделируют ключевые бизнес-сущности розничного банка:

| # | Название файла | Содержание |
|---|----------------|------------|
| 1 | `customers.csv` | Демографические данные клиентов |
| 2 | `accounts.csv` | Банковские счета клиентов |
| 3 | `transactions.csv` | Транзакции по счетам |
| 4 | `loans.csv` | Кредитные договоры |
| 5 | `loan_payments.csv` | Платежи по кредитам |
| 6 | `cards.csv` | Банковские карты |
| 7 | `card_transactions.csv` | Транзакции по картам |
| 8 | `branches.csv` | Отделения банка |
| 9 | `employees.csv` | Сотрудники банка |
| 10 | `support_tickets.csv` | Обращения в поддержку |

### Процесс загрузки данных

1. **Импорт в DBeaver**  
   Все CSV-файлы загружены в базу данных **PostgreSQL** через встроенный инструмент импорта DBeaver (`Import Data → CSV`). Каждый файл стал отдельной таблицей в схеме `public`.

2. **Создание структуры**  
   На основе полей из CSV-файлов автоматически определены типы данных (`INTEGER`, `VARCHAR`, `NUMERIC`, `DATE`) и созданы соответствующие таблицы.

3. **Установление связей**  
   Между таблицами установлены **первичные** и **внешние ключи** для обеспечения целостности данных и поддержки многотабличных запросов.

### Схема базы данных

Ниже представлена ER-диаграмма, отображающая связи между всеми таблицами:

<img width="721" height="626" alt="sql_schema" src="https://github.com/user-attachments/assets/47deeaa1-de2c-4593-abe8-01640675329c" />



*Рисунок 1. Реляционная схема банковской базы данных с указанием первичных и внешних ключей.*

**Ключевые связи:**

| Связь | Описание |
|-------|----------|
| `accounts.customer_id` → `customers.customer_id` | Счета принадлежат клиентам |
| `transactions.account_id` → `accounts.account_id` | Транзакции привязаны к счетам |
| `loans.customer_id` → `customers.customer_id` | Кредиты выданы клиентам |
| `cards.customer_id` → `customers.customer_id` | Карты оформлены на клиентов |
| `employees.branch_id` → `branches.branch_id` | Сотрудники работают в отделениях |
| `support_tickets.customer_id` → `customers.customer_id` | Обращения созданы клиентами |

### Исследовательский анализ данных. Поиск и удаление дубликатов

Запрос находит дублирующиеся записи по комбинации полей: `name`, `gender`, `date_of_birth`.

```sql
-- Поиск дубликатов по комбинации полей
SELECT *
FROM (
    SELECT *,
        COUNT(*) OVER(PARTITION BY name, gender, date_of_birth) AS duplicate_count
    FROM customers
) p
WHERE duplicate_count > 1
ORDER BY name, gender, date_of_birth;
```


<img width="1164" height="450" alt="duplicates" src="https://github.com/user-attachments/assets/bd2ff024-ab76-4d87-890c-d444267c5480" />

### Поиск дублирующихся групп с определением действия

Запрос находит все дублирующиеся записи (74 дубликата, 37 повторных записей) в таблице `customers` по комбинации полей `name`, `gender`, `date_of_birth` и помечает, какая запись будет оставлена, а какая удалена.

```sql
-- Находим все дублирующиеся группы
WITH duplicate_groups AS (
    SELECT 
        customer_id,
        name,
        gender,
        date_of_birth,
        COUNT(*) OVER(PARTITION BY name, gender, date_of_birth) AS duplicate_count,
        ROW_NUMBER() OVER(
            PARTITION BY name, gender, date_of_birth 
            ORDER BY customer_id ASC  -- Оставляем запись с наименьшим ID
        ) AS row_num
    FROM customers
)
SELECT 
    customer_id,
    name,
    gender,
    date_of_birth,
    duplicate_count,
    CASE 
        WHEN row_num = 1 THEN 'ОСТАВЛЯЕМ'
        ELSE 'УДАЛЯЕМ'
    END AS action
FROM duplicate_groups
WHERE duplicate_count > 1
ORDER BY name, gender, date_of_birth, row_num;
```


<img width="751" height="147" alt="double_group" src="https://github.com/user-attachments/assets/03a66587-dd41-4f9c-898d-6fc677397b49" />


### Создание таблицы соответствия для удаления дубликатов

```sql
-- Создаём таблицу соответствия: старый customer_id → новый (оставляемый)
CREATE TABLE customer_id_mapping AS
WITH duplicates AS (
    SELECT 
        customer_id,
        FIRST_VALUE(customer_id) OVER(
            PARTITION BY name, gender, date_of_birth 
            ORDER BY customer_id ASC
        ) AS keep_customer_id
    FROM customers
)
SELECT 
    customer_id AS old_id,
    keep_customer_id AS new_id
FROM duplicates
WHERE customer_id != keep_customer_id;
```

<img width="460" height="249" alt="table_duplicates" src="https://github.com/user-attachments/assets/e70070c8-d00a-49c2-a2c8-44d065e8c041" />

### Обновление внешних ключей и удаление дубликатов

####  Обновление связанных таблиц

Перед удалением дубликатов из таблицы `customers` необходимо перенаправить внешние ключи во всех связанных таблицах на сохраняемые записи (`new_id`).

```sql
-- Обновляем accounts
UPDATE accounts 
SET customer_id = m.new_id
FROM customer_id_mapping m
WHERE accounts.customer_id = m.old_id;

-- Обновляем loans
UPDATE loans 
SET customer_id = m.new_id
FROM customer_id_mapping m
WHERE loans.customer_id = m.old_id;

-- Обновляем cards
UPDATE cards 
SET customer_id = m.new_id
FROM customer_id_mapping m
WHERE cards.customer_id = m.old_id;

-- Обновляем support_tickets
UPDATE support_tickets 
SET customer_id = m.new_id
FROM customer_id_mapping m
WHERE support_tickets.customer_id = m.old_id;
```

####  Удаление дубликатов
```sql
-- Удаляем все дубликаты, кроме оставляемых
DELETE FROM customers
WHERE customer_id IN (
    SELECT old_id FROM customer_id_mapping
);
```
#### Проверка отсутствия дубликатов и валидности ключей
```sql
-- Проверка на отсутствие дубликатов
SELECT 
    name,
    gender,
    date_of_birth,
    COUNT(*) AS duplicate_count
FROM customers
GROUP BY name, gender, date_of_birth
HAVING COUNT(*) > 1;
```
```sql
-- Проверка валидности внешних ключей
SELECT 'accounts' AS table_name, COUNT(*) AS orphans
FROM accounts a
LEFT JOIN customers c ON a.customer_id = c.customer_id
WHERE c.customer_id IS NULL
UNION ALL
SELECT 'loans', COUNT(*)
FROM loans l
LEFT JOIN customers c ON l.customer_id = c.customer_id
WHERE c.customer_id IS NULL
UNION ALL
SELECT 'cards', COUNT(*)
FROM cards ca
LEFT JOIN customers c ON ca.customer_id = c.customer_id
WHERE c.customer_id IS NULL
UNION ALL
SELECT 'support_tickets', COUNT(*)
FROM support_tickets st
LEFT JOIN customers c ON st.customer_id = c.customer_id
WHERE c.customer_id IS NULL;
```
#### Проверка таблицы **accounts** на **NULL** значения
```sql
SELECT *
FROM accounts
WHERE (account_type IS NULL OR account_type = '' OR
      (status IS NULL OR status = ''));
```
#### Поиск дубликатов в таблице branches
```sql
-- 2.1 Поиск дубликатов
SELECT *
FROM (
    SELECT *,
        COUNT(*) OVER(PARTITION BY branch_name, city, state) AS duplicate_count
    FROM branches
) p
WHERE duplicate_count > 1
ORDER BY branch_name, city, state;
```
#### Проверяем дубликаты
```sql
-- 2.3 Проверка: это действительно разные отделения?
SELECT 
    branch_name,
    city,
    state,
    branch_id,
    ifsc_code,
    opened_date
FROM branches
WHERE branch_name IN (
    SELECT branch_name
    FROM branches
    GROUP BY branch_name, city, state
    HAVING COUNT(*) > 1
)
ORDER BY branch_name, city, state, opened_date;
```

<img width="724" height="180" alt="check_branch_dublicates" src="https://github.com/user-attachments/assets/2aae9c1a-d171-41e1-a77d-cdcfdaeb04d7" />


#### Дубликаты НЕ удаляем, потому что это разные отделения с разными ifsc_code и opened_date
#### Используем branch_id для идентификации в JOIN-ах
---
#### Проверяем сотрудников на дубликаты
```sql
--Проверка дублирующихся комбинаций (имя + branch_id + роль) + зарплата + дата трудоустройства:
SELECT *
FROM (
    SELECT *,
           COUNT(*) OVER(PARTITION BY name, branch_id, role) AS count
    FROM employees
) p
WHERE count > 1;
```

<img width="796" height="224" alt="employee_dublicates" src="https://github.com/user-attachments/assets/1350b867-64cf-4028-816b-3c3ccc8ff665" />

#### Дата трудоустройства и зарплата разные, дубликатами не являются. Оставляем записи. Это разные люди

### Предварительный анализ качества данных и структуры всех таблиц завершён. Целостность первичных ключей подтверждена. Обнаруженные дубликаты имён среди клиентов, филиалов и сотрудников проверены и признаны уникальными синтетическими записями, различающимися по ключевым атрибутам. Связи по внешним ключам валидны, особенности данных задокументированы. Схема данных чиста, реляционная целостность обеспечена, данные полностью готовы к сложным многотабличным запросам, агрегациям и бизнес-аналитике.

### Ниже представлены основные бизнес-выводы для банка как ключевого заказчика.
---
### Анализ кредитного портфеля: уровень дефолтов по типам кредитов

Запрос оценивает риск-профиль различных кредитных продуктов, рассчитывая процент дефолтов и объём просроченной задолженности по каждому типу кредита.

```sql
SELECT 
    l.loan_type,
    COUNT(DISTINCT l.customer_id) AS total_borrowers,
    COUNT(l.loan_id) AS total_loans,
    ROUND(AVG(c.credit_score)::NUMERIC, 2) AS avg_credit_score,
    ROUND(AVG(l.loan_amount)::NUMERIC, 2) AS avg_loan_amount,
    SUM(CASE WHEN l.status = 'Defaulted' THEN 1 ELSE 0 END) AS default_loans,
    ROUND(
        SUM(CASE WHEN l.status = 'Defaulted' THEN 1 ELSE 0 END) * 100.0 / COUNT(l.loan_id), 
        2
    ) AS default_percent,
    ROUND(
        SUM(CASE WHEN l.status = 'Defaulted' THEN l.loan_amount ELSE 0 END)::NUMERIC, 
        2
    ) AS defaulted_amount
FROM loans l
JOIN customers c ON l.customer_id = c.customer_id
GROUP BY l.loan_type
ORDER BY default_percent DESC;
```

<img width="1199" height="148" alt="loan_defolts" src="https://github.com/user-attachments/assets/048511a7-be1a-497d-8781-cfee9379b824" />


### Описание метрик

| Метрика | Описание |
|---------|----------|
| `total_borrowers` | Количество уникальных клиентов, взявших кредит данного типа |
| `total_loans` | Общее количество кредитов данного типа |
| `avg_credit_score` | Средний кредитный рейтинг клиентов, взявших такой кредит |
| `avg_loan_amount` | Средняя сумма кредита для данного типа |
| `default_loans` | Количество дефолтных кредитов |
| `default_percent` | % дефолтов — доля просроченных кредитов (ключевая метрика риска) |
| `defaulted_amount` | Общая сумма просроченной задолженности по данному типу |

### Вывод по кредитному портфелю

Все типы кредитов демонстрируют практически идентичные показатели: средний кредитный рейтинг заёмщиков (~600), средняя сумма кредита (~420K) и доля дефолтов (~7.2%) почти не отличаются между продуктами.

Это связано с тем, что данные являются **синтетическими** и сгенерированы с равномерным распределением параметров, без заложенных различий между типами кредитов.

Таким образом, содержательных бизнес-выводов о риск-профиле отдельных продуктов на основе данного датасета сделать нельзя. Для реального анализа необходимы данные с естественной вариативностью и дополнительными метриками (срок, регион, обеспечение).

---





