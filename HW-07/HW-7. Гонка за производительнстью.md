Конфигурация стенда:

1) ВМ с Ubuntu 22.04, RAM 4 GB, CPU 4;

| **Hostname**       | **IP**      |
| ------------------ | ----------- |
| mefanov-otus-hw6-1 | 10.92.5.78  |

Создаем БД для тестирования:

```bash
sudo -u postgres createdb pgbench
```

Воспользуемся [рекомендациями от PostgresPRO](https://postgrespro.ru/docs/postgrespro/14/pgbench) для тестирования утилитой pgbench.
Ключевое правило - scale factor должен быть не меньше количества клиентов:

$$
-s ≥ -c
$$
где:
`-s` — scale factor (размер тестовой базы)
`-c` — количество клиентов (concurrent connections)

Инициализируем pgbench со scale factor = 50 (5 млн. строк):

```bash
sudo -u postgres pgbench -i -s 50 pgbench
```

![](<images/Pasted image 20260211140417.png>)

Размер созданной БД - 756 МБ:

![](<images/Pasted image 20260211140537.png>)

Базовое тестирование с параметрами PostgreSQL по умолчанию:

```bash
sudo -u postgres pgbench -c 32 -j 4 -T 60 pgbench
```

-c 32 - количество параллельных соединение;
-j 4 - количество worker-потоков pgbench (CPU).

Результат базового тестирования (3 раза подряд):

![](<images/Pasted image 20260211141420.png>)

Результаты наглядно:

|Прогон|TPS|Latency|
|---|---|---|
|1|1316|24.3 ms|
|2|1056|30.3 ms|
|3|1575|20.3 ms|
Среднее значение:

```
(1316 + 1056 + 1575) / 3 ≈ 1316 TPS
```

Будем менять параметры по блокам в postgresql.conf от самого сильного фактора производительности к менее значимым.

1) Первый блок параметров (убираем задержку записи WAL):

```sql
ALTER SYSTEM SET fsync = 'off';
ALTER SYSTEM SET synchronous_commit = 'off';
ALTER SYSTEM SET full_page_writes = 'off';
```

Результат тестирования:

```bash
latency average = 4.947 ms
initial connection time = 23.971 ms
tps = 6468.031341 (without initial connection time)
```

2) Второй блок параметров (checkpoint):

```sql
ALTER SYSTEM SET checkpoint_timeout = '30min';
ALTER SYSTEM SET max_wal_size = '8GB';
ALTER SYSTEM SET checkpoint_completion_target = '0.9';
```

Результат тестирования:

```bash
latency average = 4.935 ms
initial connection time = 23.956 ms
tps = 6484.894867 (without initial connection time)
```

3) Третий блок параметров (буферный кэш, память на сортировки и т.д.):

```sql
ALTER SYSTEM SET shared_buffers = '2GB';
ALTER SYSTEM SET work_mem = '64MB';
ALTER SYSTEM SET maintenance_work_mem = '512MB';
ALTER SYSTEM SET effective_cache_size = '6GB';
```

Результат тестирования:

```bash
latency average = 4.800 ms
initial connection time = 24.132 ms
tps = 6666.741292 (without initial connection time)
```

4) Четвертый блок параметров (планировщик):

```sql
ALTER SYSTEM SET random_page_cost = '1.0';
ALTER SYSTEM SET seq_page_cost = '1.0';
```

Результат тестирования:

```bash
latency average = 4.741 ms
initial connection time = 27.348 ms
tps = 6749.385133 (without initial connection time)
```

5) Пятый блок параметров (параллелизм):

```sql
ALTER SYSTEM SET max_worker_processes = '4';
ALTER SYSTEM SET max_parallel_workers = '4';
ALTER SYSTEM SET max_parallel_workers_per_gather = '2';
```

Результат тестирования:

```bash
latency average = 4.779 ms
initial connection time = 26.459 ms
tps = 6695.367136 (without initial connection time)
```


Итоговая сводная таблица:

| Этап   | Изменения                                                                   | TPS      | Latency (ms) | Изменение по отношению к Default |
| ------ | --------------------------------------------------------------------------- | -------- | ------------ | -------------------------------- |
| Блок 0 | Дефолтные настройки                                                         | **1316** | 24.3         | -                                |
| Блок 1 | fsync, synchronous_commit, full_page_writes                                 | **6468** | 4.95         | +392%                            |
| Блок 2 | checkpoint_timeout, max_wal_size, completion_target                         | **6484** | 4.93         | +0.2%                            |
| Блок 3 | shared_buffers, work_mem, maintenance_work_mem, effective_cache_size        | **6666** | 4.80         | +2.8%                            |
| Блок 4 | random_page_cost, seq_page_cost                                             | **6749** | 4.74         | +1.2%                            |
| Блок 5 | max_worker_processes, max_parallel_workers, max_parallel_workers_per_gather | **6695** | 4.78         | −0.8%                            |

Итого, общий прирост составил более чем в 5 раз:

```
6749 / 1316 ≈ 5.1 раза
```

Задержка снизилась тоже в 5 раз:

```
24.3 ms -> 4.7 ms ≈ в 5 раз
```