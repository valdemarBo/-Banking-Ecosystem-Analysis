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

### Анализ прибыльности и эффективности отделений
```sql
WITH emp_sal AS (
    SELECT branch_id, SUM(salary) AS total_salary
    FROM employees
    GROUP BY branch_id
),
loan_interest AS (
    SELECT branch_id, 
           SUM((loan_amount * interest_rate) / 100) AS annual_interest_income
    FROM loans
    WHERE status = 'Active'
    GROUP BY branch_id
),
transaction_fees AS (
    SELECT a.branch_id,
           SUM(t.amount * 0.005) AS estimated_fee_income
    FROM transactions t
    JOIN accounts a ON t.account_id = a.account_id
    GROUP BY a.branch_id
),
branch_metrics AS (
    SELECT 
        b.branch_id,
        b.branch_name,
        b.city,
        COALESCE(emp.total_salary, 0) AS salary_cost,
        COALESCE(li.annual_interest_income, 0) AS interest_income,
        COALESCE(tf.estimated_fee_income, 0) AS fee_income,
        COALESCE(li.annual_interest_income, 0) + COALESCE(tf.estimated_fee_income, 0) AS total_income,
        COALESCE(li.annual_interest_income, 0) + COALESCE(tf.estimated_fee_income, 0) - COALESCE(emp.total_salary, 0) AS net_profit
    FROM branches b
    LEFT JOIN emp_sal emp ON b.branch_id = emp.branch_id
    LEFT JOIN loan_interest li ON b.branch_id = li.branch_id
    LEFT JOIN transaction_fees tf ON b.branch_id = tf.branch_id
)
SELECT 
    branch_id,
    branch_name,
    city,
    ROUND(salary_cost::NUMERIC, 2) AS salary_cost,
    ROUND(interest_income::NUMERIC, 2) AS interest_income,
    ROUND(fee_income::NUMERIC, 2) AS fee_income,
    ROUND(total_income::NUMERIC, 2) AS total_income,
    ROUND(net_profit::NUMERIC, 2) AS net_profit,
    ROUND((net_profit / NULLIF(salary_cost, 0) * 100)::NUMERIC, 2) AS profit_to_salary_ratio
FROM branch_metrics
ORDER BY net_profit DESC
LIMIT 10;
```
<img width="1234" height="220" alt="branch_revenue_analysis" src="https://github.com/user-attachments/assets/1562bc7b-f82c-4a2d-92ef-c6c2ab66f28e" />

### Описание метрик

| Столбец | Перевод | Описание |
|---------|---------|----------|
| `salary_cost` | Расходы на зарплату | Общий фонд оплаты труда сотрудников отделения |
| `interest_income` | Процентный доход | Доход от процентов по активным кредитам |
| `fee_income` | Комиссионный доход | Доход от комиссий за транзакции (0.5% от суммы) |
| `total_income` | Общий доход | Сумма процентного и комиссионного дохода |
| `net_profit` | Чистая прибыль | Общий доход минус зарплатные расходы |
| `profit_to_salary_ratio` | Эффективность (ROI) | Сколько прибыли приносит 1 рубль зарплаты, % |

### ТОP 10 по прибыли и эффективности

<img width="864" height="499" alt="branch_effectiveness" src="https://github.com/user-attachments/assets/a7bcf9a2-9313-4557-95bf-b6632d481d22" />

<img width="649" height="290" alt="roi_branches" src="https://github.com/user-attachments/assets/39409373-9454-4fae-8cda-2745f5c44d1b" />



### Ключевые выводы

| Показатель | Значение |
|------------|----------|
| **Общая прибыль топ-10** | ~53.4 млн |
| **Средняя прибыль на отделение** | ~5.34 млн |
| **Средняя эффективность (ROI)** | ~611% |

####  Лучшие по прибыли
- **Bengaluru Branch 4** — 6.02 млн (лидер)
- **Mumbai Branch 3** — 5.77 млн
- **Bhopal Branch 4** — 5.75 млн

####  Лучшие по эффективности
- **Mumbai Branch 3** — 957% (1 рубль зарплаты приносит 9.57 рубля прибыли)
- **Bengaluru Branch 7** — 833%
- **Bengaluru Branch 4** — 824%

####  По городам
| Город | Количество | Суммарная прибыль |
|-------|------------|-------------------|
| Bengaluru | 3 | ~17.0 млн |
| Delhi | 2 | ~10.7 млн |
| Bhopal | 2 | ~10.9 млн |
| Mumbai | 1 | ~5.8 млн |
| Jaipur | 1 | ~5.0 млн |
| Chandigarh | 1 | ~5.0 млн |

**Вывод:** Bengaluru — самый сильный регион (3 отделения в топ-10, суммарная прибыль 17 млн).

#### Ключевые инсайты

1. **Разрыв в прибыли минимален** — все отделения топ-10 показывают стабильно высокий результат (4.98–6.02 млн). Это говорит о **равномерной эффективности** сети.

2. **Высокая эффективность труда** — средний ROI 611% означает, что каждый рубль, вложенный в зарплаты, приносит **6.11 рубля прибыли**.

3. **Bengaluru — лидирующий регион** — три отделения в топ-10 приносят ~17 млн прибыли, что составляет почти **треть** всей прибыли.

4. **Mumbai Branch 3 — лучший по эффективности** — при самой низкой зарплате среди топ-3 (602K) показывает максимальный ROI (957%).

5. **Delhi Branch 3 — аутсайдер** — самая высокая зарплата (1.21 млн) при самой низкой прибыли (5.04 млн) и ROI (416%). 


#### Рекомендации

| Отделение | Рекомендация |
|-----------|--------------|
| **Mumbai Branch 3** | Масштабировать модель управления как лучшую практику |
| **Delhi Branch 3** | Провести аудит расходов на персонал и пересмотреть эффективность сотрудников |
| **Bengaluru** | Усилить присутствие — регион показывает стабильно высокие результаты |
| **Все отделения** | Поддерживать текущий уровень эффективности, так как разрыв между отделениями минимален |

---

### Анализ использования кредитных кард. Utilization Ratio Analysis

```sql
SELECT 
    c.card_type,
    ROUND(SUM(t.amount)::NUMERIC / NULLIF(SUM(c.credit_limit), 0)::NUMERIC, 2) AS spending_ratio,
    COUNT(DISTINCT c.card_id) AS active_cards,
    COUNT(t.transaction_id) AS total_transactions,
    ROUND(SUM(t.amount)::NUMERIC, 2) AS total_spent,
    ROUND(SUM(c.credit_limit)::NUMERIC, 2) AS total_limit
FROM cards c
JOIN transactions t ON c.account_id = t.account_id
WHERE c.status = 'Active'
GROUP BY c.card_type
ORDER BY spending_ratio DESC;
```

<img width="833" height="110" alt="utilization_ratio_analysis" src="https://github.com/user-attachments/assets/1a005d92-5f63-4a36-83cb-62f13cf9c017" />

#### Описание метрик

| Метрика  | Перевод  |
|-----------------|----------------|
| `card_type` | Тип карты |
| `spending_ratio` | Коэффициент использования лимита / Коэффициент трат |
| `active_cards` | Активные карты / Количество активных карт |
| `total_transactions` | Всего транзакций / Количество транзакций |
| `total_spent` | Общая сумма трат / Всего потрачено |
| `total_limit` | Общий кредитный лимит / Суммарный лимит |

<img width="720" height="316" alt="анализ использования кредитных кард" src="https://github.com/user-attachments/assets/6ce81144-940e-423a-9379-9a3ab7fc2891" />

### Выводы
#### Все типы карт демонстрируют практически одинаковый уровень использования кредитного лимита — от 14% до 16%. Это указывает на то, что клиенты используют карты умеренно, без перегрузки лимитов. Debit-карты лидируют по объёму трат и количеству транзакций, что говорит об их популярности как основного платёжного инструмента. Credit - Classic занимает второе место по всем показателям, что может указывать на его привлекательность для широкой аудитории. Credit - Platinum показывает самые низкие показатели использования, что может быть связано с более высокими требованиями к клиентам или узким сегментом пользователей. В целом, данные свидетельствуют о равномерном распределении использования карт без явных аномалий.

---

### Классический доходный анализ для банковской экосистемы.

```sql
SELECT
    EXTRACT(MONTH FROM start_date::DATE) AS month_num,
    ROUND(SUM(loan_amount * (interest_rate / 100.0 / 12.0))::NUMERIC, 2) AS total_interest_income,
    ROUND(SUM(loan_amount / term_months)::NUMERIC, 2) AS total_principal_repaid
FROM loans
WHERE
    status = 'Active'
    AND EXTRACT(YEAR FROM start_date::DATE) = (
        SELECT EXTRACT(YEAR FROM MAX(start_date::DATE)) FROM loans
    )
GROUP BY month_num
ORDER BY month_num ASC;
```
<img width="695" height="592" alt="анализ использования кредитов" src="https://github.com/user-attachments/assets/1bdda5e2-4241-4571-8ce8-a6d1315c70aa" />

### Выводы по анализу использования кредитов
#### Март стал пиковым месяцем по процентному доходу (409 422 руб.), что может указывать на сезонное увеличение кредитной активности после февраля.

#### Апрель показал максимальное погашение основного долга (1 054 558 руб.), что говорит о высокой платёжной дисциплине клиентов в этот период.

#### Февраль и май демонстрируют наименьший процентный доход (320 590 и 322 514 руб.), что может быть связано с сезонными факторами или снижением выдачи кредитов.

#### Средний процентный доход за месяц составляет 355 220 руб., а среднее погашение основного долга — 933 765 руб., что подтверждает стабильность кредитного портфеля.

#### Объём погашений превышает процентный доход в 2.6 раза (4.67 млн против 1.78 млн), что говорит о здоровой структуре портфеля с преобладанием возврата тела кредита над процентными платежами.




