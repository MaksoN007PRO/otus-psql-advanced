Задание: Развернуть Managed PostgreSQL в Yandex Cloud.

Активируем выданный грант и создаем облако в Yandex Cloud:

![](<images/Pasted image 20260305124736.png>)

После этого переходим в сервис Managed Service for PostgreSQL и создаем кластер, указывая необходимые настройки:

![](<images/Pasted image 20260305125056.png>)

Задаем минимальные значения ресурсов, как требуется в задании:

![](<images/Pasted image 20260305125157.png>)

Разрешаем подключения через psql извне:

![](<images/Pasted image 20260305125319.png>)

Результат созданного и работаюшего кластера в Yandex Cloud:

![](<images/Pasted image 20260305130652.png>)

При помощи клиента psql пробуем подключиться к кластеру в Yandex Cloud извне:

```sql
psql "host=rc1b-8ilts24tc6kmhs1t.mdb.yandexcloud.net port=6432 sslmode=require dbname=db1 user=mefanov"
```

![[Pasted image 20260305130637.png]]
![](<images/Pasted image 20260218154543.png>)

Сделаем запрос на проверку версии кластера:

![](<images/Pasted image 20260305130756.png>)

Сделаем тестовые запросы по созданию таблицы и извлечению данных:

![](<images/Pasted image 20260305130857.png>)

Итого: кластер создан с минимальными параметрами, подключение через psql успешно извне.