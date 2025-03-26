### **📌 Что такое оконные функции?**

**Оконные функции (Window Functions)** в SQL позволяют выполнять **агрегатные вычисления** (такие как `SUM()`, `AVG()`, `ROW_NUMBER()` и др.) **по группам строк**, но **без группировки всей таблицы**, в отличие от `GROUP BY`.

📌 Главное отличие от `GROUP BY`:

- `GROUP BY` **схлопывает строки** в одну (результат всего запроса сокращается).
- Оконные функции **оставляют исходные строки**, добавляя **агрегированные значения**.

---

## **🔹 Общий синтаксис**

```sql
<window_function>() OVER (
    PARTITION BY column_name   -- Разбивает данные на группы (по аналогии с GROUP BY)
    ORDER BY column_name       -- Определяет порядок обработки строк
    ROWS BETWEEN ...           -- Определяет "окно" строк для вычисления
)
```

- **`PARTITION BY`** (необязательно) — делит данные на группы (подобно `GROUP BY`).
- **`ORDER BY`** (необязательно) — задает порядок обработки строк.
- **`ROWS BETWEEN ...`** (необязательно) — определяет размер "окна" данных.

---

## **🔹 Виды оконных функций**

Они делятся на **три типа**:

1. **Ранжирование (Ranking Functions)** → `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE()`
2. **Агрегатные функции (Aggregate Window Functions)** → `SUM()`, `AVG()`, `MIN()`, `MAX()`, `COUNT()`
3. **Функции смещения (Offset Functions)** → `LAG()`, `LEAD()`, `FIRST_VALUE()`, `LAST_VALUE()`

---

## **1️⃣ Функции ранжирования**

Используются для **пронумерования строк** внутри окна.

### **📌 `ROW_NUMBER()`**

Выдает **уникальный номер** для каждой строки в группе, начиная с `1`.

```sql
SELECT employee_id, department, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
FROM employees;
```

🔹 **Применение:** если нужно выбрать **топ-N записей в каждой группе** (например, **самые высокооплачиваемые сотрудники в каждом отделе**).

---

### **📌 `RANK()`**

Похож на `ROW_NUMBER()`, но если есть **одинаковые значения**, он **пропускает номера**.

```sql
SELECT employee_id, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS ranking
FROM employees;
```

📌 Если два человека получают одинаковую зарплату, они **получат один и тот же ранг**, а следующий **будет пропущен**.  
Пример: `1, 2, 2, 4` (пропущен `3`).

---

### **📌 `DENSE_RANK()`**

Как `RANK()`, но **не пропускает номера**.

```sql
SELECT employee_id, department, salary,
       DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_ranking
FROM employees;
```

📌 Если зарплаты одинаковые, ранги тоже одинаковые, но **следующий номер не пропущен**:  
Пример: `1, 2, 2, 3`.

---

### **📌 `NTILE(N)`**

Делит строки в каждой группе на `N` равных частей (примерно).

```sql
SELECT employee_id, department, salary,
       NTILE(4) OVER (PARTITION BY department ORDER BY salary DESC) AS quartile
FROM employees;
```

📌 Разделит данные на 4 **равные группы** (квартели).

---

## **2️⃣ Агрегатные функции в окнах**

Работают как стандартные `SUM()`, `AVG()`, `COUNT()`, но **не группируют строки**.

### **📌 `SUM() OVER()`**

Накапливает сумму от **начала окна**.

```sql
SELECT employee_id, department, salary,
       SUM(salary) OVER (PARTITION BY department ORDER BY employee_id) AS running_total
FROM employees;
```

📌 Позволяет **считать нарастающий итог (cumulative sum)**.

---

### **📌 `AVG() OVER()`**

Считает **скользящее среднее**.

```sql
SELECT employee_id, department, salary,
       AVG(salary) OVER (PARTITION BY department ORDER BY employee_id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM employees;
```

📌 **Анализ временных рядов** — например, средняя зарплата за последние 3 сотрудника.

---

## **3️⃣ Функции смещения**

Позволяют **обращаться к предыдущим и следующим строкам**.

### **📌 `LAG()`**

Возвращает значение **из предыдущей строки**.

```sql
SELECT employee_id, department, salary,
       LAG(salary, 1, 0) OVER (PARTITION BY department ORDER BY employee_id) AS prev_salary
FROM employees;
```

📌 **Полезно для анализа изменений** (например, зарплата сотрудника по сравнению с предыдущим).

---

### **📌 `LEAD()`**

Возвращает значение **из следующей строки**.

```sql
SELECT employee_id, department, salary,
       LEAD(salary, 1, 0) OVER (PARTITION BY department ORDER BY employee_id) AS next_salary
FROM employees;
```

📌 **Полезно для прогнозирования** (например, зарплата следующего сотрудника).

---

### **📌 `FIRST_VALUE()`**

Возвращает **первое значение в окне**.

```sql
SELECT employee_id, department, salary,
       FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) AS highest_salary
FROM employees;
```

📌 **Определение минимального или максимального значения в группе**.

---

### **📌 `LAST_VALUE()`**

Возвращает **последнее значение в окне**.

```sql
SELECT employee_id, department, salary,
       LAST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS lowest_salary
FROM employees;
```

📌 **Получение последнего элемента в окне**.

---

## **🔹 Оконные функции vs. `GROUP BY`**

||Оконные функции|`GROUP BY`|
|---|---|---|
|**Сохраняет строки?**|✅ Да|❌ Нет|
|**Работает с `ORDER BY`?**|✅ Да|⚠️ Не всегда|
|**Можно считать скользящие суммы?**|✅ Да|❌ Нет|
|**Можно работать с предыдущими/следующими строками?**|✅ Да (`LAG()`, `LEAD()`)|❌ Нет|

---

## **🔹 Где применяются оконные функции?**

✔️ **Финансовая аналитика** → накопительные суммы, средние значения.  
✔️ **Ранжирование (рейтинги, лидерборды)** → `ROW_NUMBER()`, `RANK()`.  
✔️ **Предсказательный анализ** → `LAG()`, `LEAD()`.  
✔️ **Временные ряды** → скользящее среднее.  
✔️ **Оптимизация SQL-запросов** → заменяет `GROUP BY` в сложных запросах.

---

## **🔹 Итог**

📌 **Оконные функции** — мощный инструмент SQL, который позволяет выполнять агрегатные вычисления **без потери строк**.  
📌 Они **гибче `GROUP BY`** и отлично подходят для **анализа временных рядов, ранжирования и предсказаний**.  
📌 Использование **`PARTITION BY` + `ORDER BY`** позволяет разбивать данные на **группы** и выполнять сложные вычисления.

Если есть вопросы или нужен разбор конкретного кейса — спрашивай! 🚀
