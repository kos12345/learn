**В - вопрос, О - ответ**  

**В: Что изменилось и почему?**  
О: увеличили shared_buffers,effective_io_concurrency.количество обработанных транзакций увеличилось.  

**В: Объясните полученный результат**  
О: размер таблицы вырос. мертвых строк стало 10 миллионов. потому что автовакуум отключен и мертвые строки не удалялись.  

**В: Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.Не забыть вывести номер шага цикла.**  
О:  
do  
$$  
declare j integer;  
BEGIN  
	FOR j IN 1..10 LOOP  
		update mytable set mystring = left(md5(random()::text),5);  
		RAISE NOTICE 'step: %', j;  
	END LOOP;  
end;  
$$;  

**Вывод результата в psql:**  

NOTICE:  step: 1  
NOTICE:  step: 2  
NOTICE:  step: 3  
NOTICE:  step: 4  
NOTICE:  step: 5  
NOTICE:  step: 6  
NOTICE:  step: 7  
NOTICE:  step: 8  
NOTICE:  step: 9  
NOTICE:  step: 10  
DO  

