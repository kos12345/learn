**-- Создать триггер (на таблице sales) для поддержки.  
-- Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE**  

--создаем индекс для упрощения работы триггера на команде UPDATE  
create unique index i_idx on good_sum_mart using btree (good_name);  

--создаем функцию  
CREATE OR REPLACE FUNCTION summary_sales() RETURNS TRIGGER AS «$»summary_sales«$»  
    BEGIN  
        --при удалении вычисляем сумму удаляемого товара и вычитаем из витрины  
        IF (TG_OP = 'DELETE') THEN  
        with cte as (  
        select (G.good_price * old.sales_qty ) as sales_diff,G.good_name as good_name  
        FROM goods G  
        where G.goods_id = old.good_id  
        )  
        update good_sum_mart  
        set sum_sale = good_sum_mart.sum_sale - cte.sales_diff  
        from cte  
        where good_sum_mart.good_name = cte.good_name;  
        --при вставе вычисляем сумму добавляемого товара и вставляем в витрину  
        --если такой товар уже есть, то суммируем с ним  
        ELSIF (TG_OP = 'INSERT') then  
        INSERT INTO good_sum_mart  
        select G.good_name as good_name,(G.good_price * new.sales_qty ) as sales_diff  
        FROM public.goods G  
        where G.goods_id = new.good_id  
        ON CONFLICT (good_name) DO UPDATE  
        SET good_name = EXCLUDED.good_name, sum_sale = good_sum_mart.sum_sale + EXCLUDED.sum_sale;  
        ELSIF (TG_OP = 'UPDATE') then  
        --при апдейте если это произошла смена товара  
        IF ( OLD.good_id != NEW.good_id) THEN
        --вставляем новый товар или апдейтим существующий  
		     	INSERT INTO good_sum_mart  
	        select G.good_name as good_name,(G.good_price * new.sales_qty ) as sales_diff  
	        FROM public.goods G  
	        where G.goods_id = new.good_id  
	        ON CONFLICT (good_name) DO UPDATE  
	        SET good_name = EXCLUDED.good_name, sum_sale = good_sum_mart.sum_sale + EXCLUDED.sum_sale;  
	        --уменьшаем сумму старого товара  
	        with cte as (  
	        select   -1 * old.sales_qty  * G.good_price as sales_diff,G.good_name as good_name  
	        FROM goods G  
	        where G.goods_id = old.good_id  
	        )  
	        update good_sum_mart  
	        set sum_sale = good_sum_mart.sum_sale + cte.sales_diff  
	        from cte  
	        where good_sum_mart.good_name = cte.good_name;  
        END IF;  
        --если сменили количество товара, то вычисляем разницу и суммируем с витриной  
        IF ( OLD.good_id = NEW.good_id) THEN  
	        with cte as (  
	        select ( new.sales_qty - old.sales_qty ) * G.good_price as sales_diff,G.good_name as good_name  
	        FROM goods G  
	        where G.goods_id = old.good_id  
	        )  
	        update good_sum_mart  
	        set sum_sale = good_sum_mart.sum_sale + cte.sales_diff  
	        from cte  
	        where good_sum_mart.good_name = cte.good_name;  
	     END IF;  
       END IF;  
       RETURN NULL;  
    END;  
«$»summary_sales«$» LANGUAGE plpgsql;  

--добавляем триггер на таблицу sales  
CREATE TRIGGER summary_sales AFTER INSERT OR UPDATE OR DELETE ON sales FOR EACH ROW EXECUTE PROCEDURE summary_sales();  



**-- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?  
-- Подсказка: В реальной жизни возможны изменения цен.**

--единственный минус старого отчета  
--если будет происходить смена цены товара через update существующего в таблице goods  
update goods set good_price = 195000000.01 where good_name = 'Автомобиль Ferrari FXX K'  
--то отчет станет работать неправильно, так как у него нет фиксации по какой цене производилась продажа. и он будет производить перерасчет для всех продаж товара  

--если изменение цен будет происходить таким образом  
INSERT INTO goods (goods_id, good_name, good_price) VALUES  (3, 'Автомобиль Ferrari FXX K', 195000000.01);  
INSERT INTO sales (good_id, sales_qty) VALUES  (3, 1);  
--то отчет будет работать нормально.  

--а витрина в таком случае будет работать по прежнему правильно, так как она изменяется только при изменении продажи.  
--и если появится новая продажа неважно с каким id товара и его ценой, главное чтобы совпадало наименование товара.  

--недостаток витрины более критичен!:  
--при массовой вставке продаж одного товара - витрина будет узким местом, так как будет изменяться одна строка и будут блокировки и выстраиваться очередь.  

--вывод. вместо "живого" отчета сделать лучше материализованно представление, которое будет по расписанию обновляться. да данные будут слегка устаревшими  но в любом случае даже результат выполнение "живого" отчета моментально устаревает, так как продажи идут постоянно  


