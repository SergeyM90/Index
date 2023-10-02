# Index
# Реляционные базы данных. Индексы - Sergey Mironov -SYS-20

# Задание 1
Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

Решение.  

Общий размер всех таблиц - размер данных + размер индексов, поэтому:  

SELECT sum(INDEX_LENGTH)/sum(DATA_LENGTH + INDEX_LENGTH)*100 as index_percentage   
FROM information_schema.TABLES    
WHERE TABLE_SCHEMA = 'sakila';    

![task1](https://github.com/SergeyM90/Index/assets/84016375/d578ced7-4ea2-411a-870d-9fa5702b47f6)  



# Задание 2  
Выполните explain analyze следующего запроса:  

select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)  
from payment p, rental r, customer c, inventory i, film f  
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id  

  
перечислите узкие места;  
оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.  

Решение.  

Explain analyze:  

-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=4170..4170 rows=391 loops=1)  
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=4170..4170 rows=391 loops=1)  
        -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=1856..4023 rows=642000 loops=1)  
            -> Sort: c.customer_id, f.title  (actual time=1856..1913 rows=642000 loops=1)  
                -> Stream results  (cost=10.1e+6 rows=15.6e+6) (actual time=0.364..1213 rows=642000 loops=1)  
                    -> Nested loop inner join  (cost=10.1e+6 rows=15.6e+6) (actual time=0.359..1038 rows=642000 loops=1)  
                        -> Nested loop inner join  (cost=8.51e+6 rows=15.6e+6) (actual time=0.357..914 rows=642000 loops=1)  
                            -> Nested loop inner join  (cost=6.95e+6 rows=15.6e+6) (actual time=0.353..777 rows=642000 loops=1)  
                                -> Inner hash join (no condition)  (cost=1.54e+6 rows=15.4e+6) (actual time=0.344..40.4 rows=634000 loops=1)  
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.61 rows=15400) (actual time=0.0306..5.11 rows=634 loops=1)  
                                        -> Table scan on p  (cost=1.61 rows=15400) (actual time=0.0233..3.11 rows=16044 loops=1)  
                                    -> Hash  
                                        -> Covering index scan on f using idx_title  (cost=112 rows=1000) (actual time=0.106..0.247 rows=1000 loops=1)  
                                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.25 rows=1.01) (actual time=735e-6..0.00106 rows=1.01 loops=634000)  
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=250e-6 rows=1) (actual time=95.1e-6..113e-6 rows=1 loops=642000)  
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=250e-6 rows=1) (actual time=78.5e-6..96.2e-6 rows=1 loops=642000)  

Бросается в глаза количество строк, которые необходимо просканировать, чтобы выполнить запрос.  
Оконная функция сканирует таблицу с 642000 строк. А сама таблица при объединении и выборке по предварительным рассчётам explain analyze должна сканировать 15,4*10^6 строк, а фактически - 642000. Это происходит из-за того, что таблица film присоединяется кросс-джойном, соответственно таблицы перемножаются.  

Если изначальной задачей было посмотреть данные по каждому пользователю на определенную дату (2005-07-30) и какие фильмы на какую сумму он оплатил, то можно переписать запрос следующим образом:  

select concat(c.last_name, ' ', c.first_name), f.title, sum(p.amount) over (partition by c.customer_id, f.title)  
from payment p, rental r, customer c, inventory i, film f  
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id and i.film_id = f.film_id  

Добавив поле с названием фильма f.title, и указав условие по которому добавляется таблица film i.film_id = f.film_id  
В этом случае получаем такую выборку:  

![task1](https://github.com/SergeyM90/Index/assets/84016375/6a16c4ef-ad85-4aa9-90f9-e6386de77f28)

Время выполнения запроса (согласно explain analyze) сокращается с 3741 до 7, а фактическое в IDE с 3с до 38мс.  

Если всё-таки задача - получить данные по количеству оплат от каждого пользователя в конкретный день (что и происходит при выполнении изначального запроса), то таблицы film и inventory вообще не нужны. Кроме того, логичнее и нагляднее все необходимые таблицы присоединить джойнами.  

select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id)  
from payment p  
join rental r on r.rental_date = p.payment_date  
join customer c on r.customer_id = c.customer_id  
where date(p.payment_date) = '2005-07-30'  

Время выполнения запроса (согласно explain analyze) 5.51, а фактическое в IDE 31 мс  

При дальнейшей оптимизации можно добавить индексы для first name (last name и так проиндексировано)  

alter table customer  
add index idx_first_name(first_name)  

Время выполнения запроса (согласно explain analyze) 5.39, а фактическое в IDE 29 мс  

Если к базе часто поступают запросы в которых используется фильтр по date(p.payment_date), то имеет смысл выделить поле payment_date_only. Но естественно это нужно учесть при дальнейшем внесении / изменении записей в БД. Это актуально именно в случае MySQL, постгрес может создавать индексы по выражениям.  

ALTER TABLE payment  
ADD COLUMN payment_date_only DATE;  

UPDATE payment  
SET payment_date_only = DATE(payment_date);  

CREATE INDEX date_payment_date_index  
ON payment (payment_date_only);  

Сам запрос:  

select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id)  
from payment p  
join rental r on r.rental_date = p.payment_date  
join customer c on r.customer_id = c.customer_id  
where p.payment_date_only = '2005-07-30'  

Время выполнения запроса (согласно explain analyze) 2.65 (в два раза меньше)  

И оконная функция тут не нужна, она добавляет лишние вычисления. Для каждой оплаты (т.к. мы начинаем строить с таблицы payments) подсчитывается общая сумма по покупателю, а потом лишние убираются (distinct)  

select  concat(c.last_name, ' ', c.first_name), sum(p.amount)  
from payment p  
join rental r on r.rental_date = p.payment_date  
join customer c on r.customer_id = c.customer_id  
where p.payment_date_only = '2005-07-30'  
group by concat(c.last_name, ' ', c.first_name)  

Время выполнения запроса (согласно explain analyze) 1.94  









