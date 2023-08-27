# Портфолио: Аналитик данных
## Обо мне: Приветствую! Меня зовут Михаил, я начинающий аналитик данных. В этом репозитирии вы можете найти некоторые из моих проектов, выполненных в ходе обучения и практики.
# Навыки и технологии
## Инструменты анализа данных: SQL, Excel, Google sheets 
## Языки программирования и библиотеки: Python
## Системы управления базами данных: MYSQL, PostgreSQL
## Средства визуализации данных: Power BI
## Мои проекты: Kалькулятор юнит-экономики, Моделирование изменения балансов студентов (SQL-запрос и анализ результата запроса)
```PostgreSQL
--------------Шаг 1----------------------------------------------------------------------------------------------------
-- Узнаем, когда была первая транзакция для каждого студента. Начиная с этой даты, мы будем собирать его баланс уроков. 
-- **Создадим CTE** `first_payments` с двумя полями: `user_id` и `first_payment_date` (дата первой успешной транзакции).
With first_payment_date as
(Select user_id
, date_trunc ('day', min(transaction_datetime)) as first_payment_date
from skyeng_db.payments
where status_name = 'success'
group by user_id
order by first_payment_date
)
------------- Шаг 2---------------------------------------------------------------------------------------------------
-- Соберем таблицу с датами за каждый календарный день 2016 года. Есть разные способы это сделать, но мы воспользуемся тем, который уже знаем.
-- Выберем все даты из таблицы `classes`, **создадим CTE** `all_dates` с полем `dt`, где будут храниться уникальные даты (без времени) уроков.
, all_dates as (
Select distinct date_trunc('day', class_start_datetime) as dt
from skyeng_db.classes
where (class_start_datetime between '2016-01-01' and '2017-01-01')),
------------- ШАГ 3---------------------------------------------------------------------------------------------------
-- Узнаем, за какие даты имеет смысл собирать баланс для каждого студента. Для этого объединим таблицы и создадим CTE all_dates_by_user, 
-- где будут храниться все даты жизни студента после того, как произошла его первая транзакция. 
-- В таблице должны быть такие поля: user_id, dt. 
all_dates_by_user as(
Select fp.user_id, 
    ad.dt
from all_dates as ad
left join first_payment_date as fp
on fp.first_payment_date <= ad.dt
order by user_id, dt),
--------------ШАГ 4--------------------------------------------------------
-- Найдем все изменения балансов, связанные с успешными транзакциями. 
-- Выберем все транзакции из таблицы payments, сгруппируем их по user_id и дате транзакции (без времени) и найдем сумму по полю classes. 
-- В результате получим CTE payments_by_dates с полями: user_id, payment_date, transaction_balance_change (сколько уроков было начислено или списано в этот день). 
payments_by_dates as (
Select user_id
, date_trunc('day', transaction_datetime) as payment_date
, sum(classes) as transaction_balance_change
from skyeng_db.payments
where status_name = 'success'
group by user_id, payment_date),
-------------- Шаг 5------------------------------------------------------
-- Найдем баланс студентов, который сформирован только транзакциями. Для этого объединим `all_dates_by_user` и `payments_by_dates` так, чтобы совпадали даты и `user_id`. 
-- Используем оконные выражения (функцию `sum`), чтобы найти кумулятивную сумму по полю `transaction_balance_change` для всех строк до текущей включительно с разбивкой по `user_id`
--и сортировкой по `dt`. 
-- **В результате** получим CTE `payments_by_dates_cumsum` с полями: `user_id`, `dt`, `transaction_balance_change` — `transaction_balance_change_cs` 
--(кумулятивная сумма по `transaction_balance_change`). 
-- При подсчете кумулятивной суммы можно заменить пустые значения нулями.
payments_by_dates_cumsum as (
Select a2.user_id
, a2.dt
, a1.transaction_balance_change
, sum(coalesce(transaction_balance_change,0)) over (partition by a2.user_id order by dt) as transaction_balance_change_cs
from payments_by_dates as a1
right join all_dates_by_user as a2
on a1.user_id = a2.user_id
and a1.payment_date=a2.dt),
---------------Шаг 6-------------------------------------------------------
-- Найдем изменения балансов из-за прохождения уроков. 
-- Создадим CTE `classes_by_dates`, посчитав в таблице `classes` количество уроков за каждый день для каждого ученика. 
-- Нас не интересуют вводные уроки и уроки со статусом, отличным от `success` и `failed_by_student`. 
-- **Получим результат** с такими полями: `user_id`, `class_date`, `classes` (количество пройденных в этот день уроков).
-- Причем `classes` мы умножим на `-1`, чтобы отразить, что `-` — это списания с баланса.
classes_by_dates as (
Select user_id
, date_trunc('day', class_start_datetime) as class_date
, count(id_class)*(-1) as classes
from skyeng_db.classes
where class_status in ('success', 'failed_by_student')
and class_type <> 'trial'
group by user_id, date_trunc('day', class_start_datetime)),
---------------Шаг 7-------------------------------------------------------
-- По аналогии с уже проделанным шагом для оплат создадим CTE для хранения кумулятивной суммы количества пройденных уроков. 
-- Для этого объединим таблицы `all_dates_by_user` и `classes_by_dates` так, чтобы совпадали даты и `user_id`. 
-- Используем оконные выражения (функцию `sum`), чтобы найти кумулятивную сумму по полю `classes` для всех строк до текущей включительно с разбивкой по `user_id` и сортировкой по `dt`. 
-- **В результате** получим CTE `classes_by_dates_dates_cumsum`с полями: `user_id`, `dt`, `classes` — `classes_cs`(кумулятивная сумма по `classes`).
-- При подсчете кумулятивной суммы обязательно нужно заменить пустые значения нулями.
classes_by_dates_dates_cumsum as (Select b2.user_id
, b2.dt
, b1.classes
, sum(coalesce(classes,0)) over (partition by b2.user_id order by dt) as classes_cs
from classes_by_dates b1
right join all_dates_by_user b2
on b1.user_id=b2.user_id
and b1.class_date=b2.dt
order by dt),
--------------Шаг 8--------------------------------------------------------
-- Создадим CTE `balances` ****с вычисленными балансами каждого студента. 
-- Для этого объединим таблицы `payments_by_dates_cumsum` ****и `classes_by_dates_dates_cumsum` так, чтобы совпадали даты и `user_id`.
-- **Получим такие поля:** `user_id`, `dt`, `transaction_balance_change`, `transaction_balance_change_cs`, `classes`, `classes_cs`, `balance` (`classes_cs` + `transaction_balance_change_cs`).
balances as (Select c1.user_id
, c1.dt
, transaction_balance_change
, transaction_balance_change_cs 
, c2.classes
, c2.classes_cs
, classes_cs + transaction_balance_change_cs  as balance
from payments_by_dates_cumsum c1
join classes_by_dates_dates_cumsum c2
on c1.user_id=c2.user_id
and c1.dt = c2.dt)
---### Задание 1
---Выберите топ-1000 строк из CTE `balances` с сортировкой по `user_id` и `dt`. Посмотрите на изменения балансов студентов.
Select *
from balances
order by user_id, dt
limit 1000
```

