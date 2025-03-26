В SQL оператор GROUP BY используется для группировки строк, имеющих одинаковые значения в указанных столбцах. Часто используется вместе с агрегатными функциями, такими как COUNT, SUM, AVG, MAX, и MIN, чтобы выполнить операции на каждой из групп.

Пример общего синтаксиса:
```sql
SELECT столбец1, агрегатная_функция(столбец2)
FROM таблица
GROUP BY столбец1;
```
Вот несколько примеров для разных сценариев:
1. Группировка по одному столбцу и использование функции COUNT:
```sql
SELECT department, COUNT(*)
FROM employees
GROUP BY department;
```
2. Группировка по нескольким столбцам и использование функции SUM:
```sql
SELECT department, job_title, SUM(salary)
FROM employees
GROUP BY department, job_title;
```
3. Группировка с использованием HAVING для фильтрации групп:
```sql
SELECT department, AVG(salary)
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000;
```   
4. Группировка с дополнительными условиями WHERE:
```sql
SELECT department, COUNT(*)
FROM employees
WHERE hire_date >= '2020-01-01'
GROUP BY department;
```   
