# Домашнее задание №13

Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ
В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).
Есть запрос для генерации отчета – сумма продаж по каждому товару.
БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.
Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)
Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE
•	Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.

## Ход работы

Создадим объекты согласно файлу «hw_triggers.sql» и заполним таблицы тестовыми данными.
Таблица «товары»:

![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/45c8e4b3-8d89-449f-87cb-bda98f1b0705)


Таблица «продажи»:

![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/fabfa211-ebef-4b6f-9b6a-ba762f55f2bf)

Посмотрим отчет о продажах:

 ![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/d933660d-6820-4e3d-be53-6bb81ce9008a)

С увеличением объёма данных отчет стал создаваться медленно, поэтому принято решение денормализовать БД: создать таблицу good_sum_mart и заполнить ее данными.

![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/7cfa21ca-1d5a-4119-bff2-493cc36df17c)

Создадим триггер для витрины:

>CREATE OR REPLACE FUNCTION fn_good_sum_mart() RETURNS TRIGGER AS $tg_good_sum_mart$
>
>   DECLARE
>
>     new_good_name  varchar(63);
> 
>   new_good_price numeric(12, 2);
>
>   old_good_name  varchar(63);
>
>   old_good_price numeric(12, 2);
>
>begin
>
>  -- Определяем товар и текущую цену
>
>  IF NEW is not null THEN
>
>  SELECT good_name, good_price FROM pract_functions.goods WHERE goods_id = NEW.good_id
>
 > INTO new_good_name, new_good_price;
>
 > END IF;
>
 
 > IF OLD is not null THEN
> 
 > SELECT good_name, good_price FROM pract_functions.goods WHERE goods_id  = OLD.good_id
> 
 > INTO old_good_name, old_good_price;
> 
  >END IF;
> 

 

  
 >IF (tg_op = 'DELETE') THEN
>
 >   -- Считаем дельту на удаление по изменению количества продаж
>
>  UPDATE pract_functions.good_sum_mart
>
 >  SET sum_sale = s2.sum_sale - old_good_price * OLD.sales_qty
>
>  FROM pract_functions.good_sum_mart s2
>
>  WHERE s2.good_name = old_good_name;
>
  
> ELSEIF (tg_op = 'UPDATE') THEN
> 
>   -- Считаем дельту на обновление по изменению количества продаж
> 
 > UPDATE pract_functions.good_sum_mart
> 
 >  SET sum_sale = s2.sum_sale + new_good_price * (NEW.sales_qty - OLD.sales_qty)
> 
 > FROM pract_functions.good_sum_mart s2
> 
 > WHERE s2.good_name = new_good_name;
> 
     
 >ELSEIF (tg_op = 'INSERT') THEN
>
 >   IF NOT EXISTS (SELECT * FROM pract_functions.good_sum_mart m WHERE m.good_name  = new_good_name)
>
 > THEN
>
 >      -- Считаем продажи по новым товарам
>
 >     INSERT INTO pract_functions.good_sum_mart (good_name, sum_sale)
>
 >       SELECT good_name , good_price * NEW.sales_qty
>
 >     FROM goods
>
  >    WHERE goods_id = NEW.good_id;
>
  >   ELSE
>
  >    -- Считаем дельту на обновление по изменению количества продаж
>
  >    UPDATE pract_functions.good_sum_mart
>
  >     SET sum_sale = s2.sum_sale + new_good_price * NEW.sales_qty
>
  >    FROM pract_functions.good_sum_mart  s2
>
   >   WHERE s2.good_name = new_good_name;
>
  >END IF;
>
  
 >END IF;
>
   > RETURN null;
>
>END;
>
>$ tg_good_sum_mart $
>
>language plpgsql;
>
>
>CREATE or replace TRIGGER tg_sales
>
>after insert or delete or update on pract_functions.sales
>
>FOR EACH ROW EXECUTE function fn_good_sum_mart();
>

![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/b14890c8-00ab-4d85-a6ca-b2f63ba29593)

 

Проверим работу триггера.
Просмотрим продажи товара «Ряженка» и суммы в витрине, добавим продажу и убедимся, что новая продажа появилась и в витрине изменилась сумма:

![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/c24f2d94-9cdf-47ca-97e5-7c33eb7a690b)


Отредактируем у последней продажи количество – данные в витрине изменились:

  ![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/b2503731-1d35-429f-bbb3-bfa33ffe190f)


Удалим первую продажу – данные в витрине изменились:

   ![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/18f7b9b2-6da7-4959-8333-29d7cb082fc7)

Изменим цену в таблице товаров и добавим новую продажу – видим, что в витрине сумма изменилась на нужную сумму:

 ![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/aa096fe2-3d26-40ae-b8c3-e0b67a4dcbb7)


Снова отредактируем у последней продажи количество – данные в витрине изменились некорректно, т.к. товар был продан по другой цене, но это нигде не зафиксировано:

 ![image](https://github.com/blaidex2/Postgres_Homework-13/assets/130083589/2b0bf946-131e-4c66-a22c-299e6faa2ec3)

## *
Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности) Подсказка: В реальной жизни возможны изменения цен.

>При  использовании витрины, данные в которой изменяются триггером при каком-либо изменении продажи конкретного товара, выборка данных происходит значительно быстрее, чем
> в отчете. Это позволяет иметь актуальные на текущий момент данные: при большом объеме данных за время расчета отчета могут быть выполнены новые продажи или возвраты, а
> также новая партия товара может быть продана по другой цене.
> 
>Таким образом, использование витрины в предложенном виде имеет ряд существенных недостатков, из-за которых данные в витрине будут некорректны и связано это с изменением
>цены товара во времени (для корректного подсчета стоимости продаж нам нужно иметь какой-либо справочник цен или фиксировать цену на момент продажи, например цена в таблице sales) и наличием имени
>товара, а не ссылки на него (имя товара могут изменить)
>
