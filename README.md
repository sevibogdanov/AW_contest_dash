# Структура
[Модель данных](#Модель-данных)  
[Вкладка "план-факт"](#Вкладка---план-факт)  
[Вкладка "рейтинг сотрудников"](#Вкладка---рейтинг-сотрудников)  
[Вкладка "детальный анализ сотрудников"](#Вкладка---детальный-анализ-сотрудников)  
[Вкладка "структура продаж"](#Вкладка---структура-продаж)  

# Модель данных
![image](https://github.com/sevibogdanov/AW_contest_dash/assets/130535023/464feefe-b0a4-4929-9cdb-06fc08e3b30c)
<details>
<summary><b>Код fact</b></summary>
```sql
with cte as (select 
    `ПЕРИОД`,
    `РЕГИОН`,
    `ФИО`,
    `ТОВАРНАЯ_ГРУППА`,
    sum(`ПРОДАНО__ШТ`) as `ПРОДАНО__ШТ`,
    sum(`ПРОДАНО__РУБ`) as `ПРОДАНО__РУБ`
    from `Лист1`
    group by 1,2,3,4)
,cte2 as (
    select *,
    sum(`ПРОДАНО__РУБ`) over(partition by `ПЕРИОД`,`ФИО`) as sold_within_month_per_employee,
    sum(`ПРОДАНО__РУБ`) over(partition by `ФИО`) as sold_total_per_employee
    from cte)
select *,
concat(
    concat(case when length(cast(month(`ПЕРИОД`) as varchar(255))) = 1 then concat('0',cast(month(`ПЕРИОД`) as varchar(255))) else cast(month(`ПЕРИОД`) as varchar(255)) end,'-'),
    cast(year(`ПЕРИОД`) as varchar(255)))
as data_filter,
1 ind_total,
concat(concat(concat(
    concat(case when length(cast(month(`ПЕРИОД`) as varchar(255))) = 1 then concat('0',cast(month(`ПЕРИОД`) as varchar(255))) else cast(month(`ПЕРИОД`) as varchar(255)) end,'-'),
    cast(year(`ПЕРИОД`) as varchar(255))),'|'),`ТОВАРНАЯ_ГРУППА`) date_category,
sum(`ПРОДАНО__РУБ`) over(partition by `РЕГИОН` order by `ПЕРИОД`) as running_fact,
substr(`ФИО`,1,position(' ' in `ФИО`)) as fio_surname,
substr(`ФИО`,position(' ' in `ФИО`), length(`ФИО`)-position(' ' in `ФИО`)+1) as fio_name,
row_number() over(partition by `ПЕРИОД`,    `РЕГИОН`,    `ФИО` order by `ТОВАРНАЯ_ГРУППА`) category_rn,
row_number() over(partition by `ПЕРИОД`,`РЕГИОН` order by `ПЕРИОД`) filter_for_plan, --при сравнении с планом, будем суммировать только строки с 1, таким образом отсекаем дубли
row_number() over(partition by `ПЕРИОД`,`РЕГИОН`,`ФИО` order by `ФИО`) filter_for_plan_emp,
dense_rank() over(partition by `ПЕРИОД`,`РЕГИОН` order by `SOLD_TOTAL_PER_EMPLOYEE` desc) + dense_rank() over(partition by `ПЕРИОД`,`РЕГИОН` order by `SOLD_TOTAL_PER_EMPLOYEE` asc) - 1 empl_per_region, --уникальное кол-во работников на реион
dense_rank() over(partition by `ПЕРИОД`,`РЕГИОН` order by `SOLD_WITHIN_MONTH_PER_EMPLOYEE` desc) emp_rank_within_month_and_region,
dense_rank() over(partition by `ПЕРИОД` order by `SOLD_WITHIN_MONTH_PER_EMPLOYEE` desc) emp_rank_within_month_total,
dense_rank() over(order by `SOLD_TOTAL_PER_EMPLOYEE` desc) emp_rank_total
from cte2```
</details>


# Вкладка - план-факт
# Вкладка - рейтинг сотрудников
# Вкладка - детальный анализ сотрудников
# Вкладка - структура продаж
