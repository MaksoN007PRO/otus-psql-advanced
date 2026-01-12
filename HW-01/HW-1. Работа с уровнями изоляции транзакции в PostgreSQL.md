1. Создаем файл `docker-compose.yml`:
   
![](images/Pasted%20image%20260112134118.png)

2. Поднимаем контейнер `otus-psql-mefanov` с PostgreSQL контейнера с PostgreSQL:
   
![[Pasted image 20260112134544.png]]

3. Подключаемся к созданной БД `postgres` от имени пользователя `postgres` при помощи pgAdmin с использованием двух терминалов:
   
![[Pasted image 20260112135357.png]]
4. Выключаем autocommit в обеих сессиях:
   
![[Pasted image 20260112134803.png]]

5. В первой сессии создаем таблицу shipments (перевозки) и добавляем в неё данные:
   
```sql
create table shipments(id serial, product_name text, quantity int, destination text); 
insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe'); 
insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA'); commit;
```

![[Pasted image 20260112135448.png]]

6. Проверяем текущий уровень изоляции:
   
```sql
show transaction isolation level;
```

По умолчанию read commited:

![[Pasted image 20260112135623.png]]

7. Начинаем новую транзакцию в обеих сессиях с уровнем изоляции по умолчанию и добавляем новую запись в первой сессии:

```sql
insert into shipments(product_name, quantity, destination) values('sugar', 300, 'Asia');
```

![[Pasted image 20260112140009.png]]

Во второй сессии выполняем запрос:
   
```sql
select * from shipments;
```

![[Pasted image 20260112140116.png]]
Новая строка с `'sugar', 300, 'Asia'` не видна, т.к. могут быть видны только примененные изменения, что соответствовало бы уровню изоляции `read committed`. 

Завершаем первую транзакцию с помощью `commit;` и снова выполняем `select * from shipments` во второй сессии:

![[Pasted image 20260112140509.png]]
![[Pasted image 20260112140528.png]]
Теперь новая строка видна, так как изменения были подтверждены в первой сессии к моменту запуска запроса во второй сессии.

10. Далее начинаем новые транзакции в обеих сессиях с уровнем изоляции `repeatable read`:

```sql
begin;
set transaction isolation level repeatable read;
```

![[Pasted image 20260112141058.png]]

В первой сессии добавляем новую запись:

```sql
insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');
```

![[Pasted image 20260112141243.png]]

Во второй сессии выполняем:

```sql
select * from shipments;
```

![[Pasted image 20260112141253.png]]

Новая запись во второй сессии не видна, т.к. согласно уровню изоляции `Repeatable Read` видны только те данные, которые были зафиксированы до начала транзакции.

Завершаем транзакцию в первой сессии и повторяем запрос во второй.

![[Pasted image 20260112141253.png]]
Новая строка снова не видна, т.к. вторая сессия продолжает видеть данные на момент создания собственной транзакции.

Завершаем транзакцию во второй сессии и повторяем запрос:

![[Pasted image 20260112141919.png]]

Данные стали видны, т.к. новый запрос получает новый снимок и видит закоммиченные изменения.
