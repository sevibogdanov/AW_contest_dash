Страница конкурса (1 место) https://analyticworkspace.ru/tpost/kzi80mbx01-podvedeni-itogi-konkursa-dashbordov-aw-b

# Структура
[Модель данных](#Модель-данных)  
[Вкладка "план-факт"](#Вкладка---план-факт)  
[Вкладка "рейтинг сотрудников"](#Вкладка---рейтинг-сотрудников)  
[Вкладка "детальный анализ сотрудников"](#Вкладка---детальный-анализ-сотрудников)  
[Вкладка "структура продаж"](#Вкладка---структура-продаж)  

Дашборд корректно отображается (без сокращений/обрезаний) при увеличении в браузере не более 100%.
Цветовая легенда: 
- красный - значения менее 85% или недовыполнение плана
- желтый - значения от 85-100% выполнения плана
- зеленый - значения 100% или более от плана
- серый - фактические значения
- голубой/синий - плановые значения

# Модель данных
![image](https://github.com/sevibogdanov/AW_contest_dash/assets/130535023/464feefe-b0a4-4929-9cdb-06fc08e3b30c)

<details>
<summary>Код fact</summary>
   
```   sql
with cte as (select 
    `ПЕРИОД`,
    `РЕГИОН`,
    `ФИО`,
    `ТОВАРНАЯ_ГРУППА`,
    sum(`ПРОДАНО__ШТ`) as `ПРОДАНО__ШТ`,
    sum(`ПРОДАНО__РУБ`) as `ПРОДАНО__РУБ`
    from `Лист1`
    group by 1,2,3,4)
--
,cte2 as (
    select *,
    sum(`ПРОДАНО__РУБ`) over(partition by `ПЕРИОД`,`ФИО`) as sold_within_month_per_employee,
    sum(`ПРОДАНО__РУБ`) over(partition by `ФИО`) as sold_total_per_employee
    from cte)
--
select *,
    concat(
        concat(case when length(cast(month(`ПЕРИОД`) as varchar(255))) = 1 then concat('0',cast(month(`ПЕРИОД`) as varchar(255))) else cast(month(`ПЕРИОД`) as varchar(255)) end,'-'),
        cast(year(`ПЕРИОД`) as varchar(255)))
    as data_filter, --формат даты
    1 ind_total, -- константа единица для джойнов и расчетов
    concat(concat(concat(
        concat(case when length(cast(month(`ПЕРИОД`) as varchar(255))) = 1 then concat('0',cast(month(`ПЕРИОД`) as varchar(255))) else cast(month(`ПЕРИОД`) as varchar(255)) end,'-'),
        cast(year(`ПЕРИОД`) as varchar(255))),'|'),`ТОВАРНАЯ_ГРУППА`) date_category,
    sum(`ПРОДАНО__РУБ`) over(partition by `РЕГИОН` order by `ПЕРИОД`) as running_fact, --накопительный факт
    substr(`ФИО`,1,position(' ' in `ФИО`)) as fio_surname,
    substr(`ФИО`,position(' ' in `ФИО`), length(`ФИО`)-position(' ' in `ФИО`)+1) as fio_name,
    row_number() over(partition by `ПЕРИОД`,    `РЕГИОН`,    `ФИО` order by `ТОВАРНАЯ_ГРУППА`) category_rn,
    row_number() over(partition by `ПЕРИОД`,`РЕГИОН` order by `ПЕРИОД`) filter_for_plan, --при сравнении с планом, будем суммировать только строки с 1, таким образом отсекаем дубли
    row_number() over(partition by `ПЕРИОД`,`РЕГИОН`,`ФИО` order by `ФИО`) filter_for_plan_emp,--при сравнении с планом, будем суммировать только строки с 1, таким образом отсекаем дубли на уровне сотрудника
    dense_rank() over(partition by `ПЕРИОД`,`РЕГИОН` order by `SOLD_TOTAL_PER_EMPLOYEE` desc) + 
    dense_rank() over(partition by `ПЕРИОД`,`РЕГИОН` order by `SOLD_TOTAL_PER_EMPLOYEE` asc) - 1 empl_per_region, --уникальное кол-во работников на реион
    dense_rank() over(partition by `ПЕРИОД`,`РЕГИОН` order by `SOLD_WITHIN_MONTH_PER_EMPLOYEE` desc) emp_rank_within_month_and_region,
    dense_rank() over(partition by `ПЕРИОД` order by `SOLD_WITHIN_MONTH_PER_EMPLOYEE` desc) emp_rank_within_month_total,
    dense_rank() over(order by `SOLD_TOTAL_PER_EMPLOYEE` desc) emp_rank_total
```
</details>

<details>
<summary>Код plan</summary>

```sql
select 
   `ПЕРИОД` `ПЛАН_ПЕРИОД`,
   `РЕГИОН` `ПЛАН_РЕГИОН`,
   `ПЛАН`,
   `ПЛАН` / sum(`ПЛАН`) over(partition by `ПЕРИОД`) as plan_share_within_month,
   sum(`ПЛАН`) over(partition by `РЕГИОН` order by `ПЕРИОД`) as plan_running_sum --накопительный план
from `Лист1`

```
</details>

<details>
<summary>Код okato_ref</summary>

``` sql
with cte as (select 
    replace(cast(`КОД` as varchar(255)),' ','') `OKATO_КОД`, -- подгон под формат
    `НАИМЕНОВАНИЕ` `OKATO_НАИМЕНОВАНИЕ`,
    `АДМИНИСТРАТИВНЫЙ_ЦЕНТР` `OKATO_АДМИНИСТРАТИВНЫЙ_ЦЕНТР`
    from `default`)
--
select
case 
    when length(`OKATO_КОД`)=11 then `OKATO_КОД`
    else concat('0',`OKATO_КОД`) end
    `OKATO_КОД`, -- подгон под формат
`OKATO_НАИМЕНОВАНИЕ`,
`OKATO_АДМИНИСТРАТИВНЫЙ_ЦЕНТР`
from cte
```
</details>

<details>
<summary>Код currency</summary>
   
```python
import requests
import datetime
from pyspark.sql import Row
def data_query():

    query = 'https://cbr.ru/key-indicators/'
    r = requests.get(query)

    upd_date = datetime.datetime.now()
    df = {'currency':[],
        'value':[],
        'upd_date':[]}

    # Ставка
    text = r.text
    text = text[text.find('лючевая ставка'):]
    text = text[text.find('value')+7:]
    percent_rate = float(text[:text.find('%')].replace(',','.'))
    #
    df['currency'].append('Ключевая ставка')
    df['value'].append(percent_rate)
    df['upd_date'].append(upd_date)

    #Юань
    text = text[text.find('Китайский юань'):]
    text = text[text.find('value td-w-4 _bold _end mono-num')+len('value td-w-4 _bold _end mono-num')+2:]
    #
    df['currency'].append('Китайский юань')
    df['value'].append(float(text[:text.find('</td')].replace(',','.')))
    df['upd_date'].append(upd_date)

    #Доллар США
    text = text[text.find('Доллар'):]
    text = text[text.find('value td-w-4 _bold _end mono-num')+len('value td-w-4 _bold _end mono-num')+2:]
    #
    df['currency'].append('Доллар США')
    df['value'].append(float(text[:text.find('</td')].replace(',','.')))
    df['upd_date'].append(upd_date)

    #Евро
    text = text[text.find('Евро'):]
    text = text[text.find('value td-w-4 _bold _end mono-num')+len('value td-w-4 _bold _end mono-num')+2:]
    #
    df['currency'].append('Евро')
    df['value'].append(float(text[:text.find('</td')].replace(',','.')))
    df['upd_date'].append(upd_date)
    print(df)
    return df

def after_load_virtual(df,spark,app,*args,**kwargs):
    data = data_query()
    rows=[]

    rows.append(Row(
        dollar = data['value'][2],
        upd_date = datetime.datetime.now(),
        yuan = data['value'][1],
        euro = data['value'][3],
        join_key = int(1)
    ))
    
    return spark.createDataFrame(rows, schema=df.schema)
```
</details>

# Вкладка - план-факт
### Вкладка предназначена для сравнения факта с планом в разрезе регионов, отображена динамика и накопительный факт/план по месяцам.  
Интересные особенности:
- план считается благодаря полю-фильтру, расчитанному через окнную функцию при загрузке  
```SUM_IF([plan],[filter_for_plan]=1)/1000000``` - поле агрегата
(```row_number() over(partition by `ПЕРИОД`,`РЕГИОН` order by `ПЕРИОД`) as filter_for_plan```sql при загрузке)
- факт в валютах пересчитывается по актуальным данным с сайта ЦБ (ежедневно в 6:00 по мск)
- на график "Выполнение плана" добавлена невидимая линия для выведения % выполнения в подсказку
- на графиках "Выполнение плана по периодам" и "Выполнение плана по регионам" использован дриллдаун для возможности быстрой отмены фильтра (появляется кнопка "назад")
- график "Накопительный процент выполнения использует предрасчитанные в источнике данные на уровне sql, поэтому он корректно отображает информацию при наложении фильтра по датам
```sum(`ПЛАН`) over(partition by `РЕГИОН` order by `ПЕРИОД`) as plan_running_sum``` - расчет накопительного плана
- агрегатная функция итога в таблице имеет другую разрядность, чтобы цифры отображались целиком
- кнопка "сбросить фильтры" ведет на страницу дашборда  (при обновлении фильтры сбрасываются)

# Вкладка - рейтинг сотрудников
### Вкладка предназначена для сравнения показателей выполнения плана и продаж сотрудников
Интересные особенности:
- план считается благодаря полю-фильтру и кол-ву сотрудников, расчитанным через окнные функции при загрузке   
```(SUM_IF([plan],[filter_for_plan_emp]=1) / max([empl_per_region]))``` - формула плана в виджете (план разбивается пропорционально кол-ву сотрудников в регионе)  
```row_number() over(partition by `ПЕРИОД`,`РЕГИОН`,`ФИО` order by `ФИО`) as filter_for_plan_emp``` - фильтр плана, но на уровне сотрудника  
```dense_rank() over(partition by `ПЕРИОД`,`РЕГИОН` order by `SOLD_TOTAL_PER_EMPLOYEE` desc) + 
dense_rank() over(partition by `ПЕРИОД`,`РЕГИОН` order by `SOLD_TOTAL_PER_EMPLOYEE` asc) - 1 empl_per_region``` - подсчет уникального кол-ва сотрудников через оконную функцию sql
- используется оконная функция внутри виджета для подсчета топа по факту продаж  
```RANK_DENSE(sum([prodano__rub]), desc total)```
- используется оконная функция внутри виджета для подсчета топа по выполнению своего плана (корректнее для отображения качества работы сотрудника, так как ситуация по регионам отличается)  
```RANK_DENSE(sum([prodano__rub]) / (SUM_IF([plan],[filter_for_plan_emp]=1) / max([empl_per_region])),desc total)```  

# Вкладка - детальный анализ сотрудников
### Вкладка предназначена для детального анализа конкретного сотрудника
Интересные особенности:
- все виджиты вкладки настроены отображать информацию только по 1 сотруднику за счет условия во всех агрегатах по типу
```if(countd([fio])>1,sum(NULL),SUM([prodano__rub]))```
# Вкладка - структура продаж
### Вкладка предназначена для анализа структуры продаж в динамике (по категориям и регионам)
Интересные особенности:
- для создания цветовой схемы для каждой категории и регионов создавался отдельный показатель
