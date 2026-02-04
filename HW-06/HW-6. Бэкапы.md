Конфигурация стенда:

1) 2 ВМ с Ubuntu 22.04;
   
![](<images/Pasted image 20260204140505.png>)

| **Hostname**       | **IP**      |
| ------------------ | ----------- |
| mefanov-otus-hw6-1 | 10.92.5.78  |
| mefanov-otus-hw6-2 | 10.92.5.192 |

Установка PostgreSQL 14

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
```

Проверка версии psql:

```bash
psql --version
psql (PostgreSQL) 14.20 (Ubuntu 14.20-0ubuntu0.22.04.1)
```


Установка pg_probackup

```bash
sudo apt install gpg wget
wget -qO - https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG-PROBACKUP | \
sudo tee /etc/apt/trusted.gpg.d/pg_probackup.
```

```bash
. /etc/os-release
echo "deb [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb $VERSION_CODENAME main-$VERSION_CODENAME" | \
sudo tee /etc/apt/sources.list.d/pg_probackup.list
```

```bash
sudo apt update
apt search pg_probackup
```

```bash
sudo apt install pg-probackup-16
```

Проверка версии pg_probackup:

```bash
mefanov@mefanov-otus-hw6-1:~$ /usr/bin/pg_probackup-16 --version
pg_probackup-16 2.5.16 (PostgreSQL 16.11)
```

Проверка статуса службы Postgresql:

![](<images/Pasted image 20260204131346.png>)

Создаем БД, таблицу "Лояльность поставщиков" и наполняем её данными:

```sql
CREATE DATABASE suppliers_loyalty;
\c suppliers_loyalty
```

```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    name TEXT,
    bonus_points INT
);
```

```sql
INSERT INTO clients (name, bonus_points)
VALUES
 ('ООО Поставщик1', 1200),
 ('ИП Поставщик2', 450);
```

Вывод содержимого таблицы:

![](<images/Pasted image 20260204132143.png>)

Создаем каталог для бекапов:

```bash
sudo mkdir -p /var/lib/pg_probackup
sudo chown postgres:postgres /var/lib/pg_probackup
```

Инициализируем каталог бекапов:

```bash
sudo -u postgres /usr/bin/pg_probackup-16 init -B /var/lib/pg_probackup
```

![](<images/Pasted image 20260204132557.png>)

Добавляем инстанс и проверяем, что он добавился:

```bash
sudo -u postgres /usr/bin/pg_probackup-16 add-instance -B /var/lib/pg_probackup -D /var/lib/postgresql/14/main --instance=loyalty
```

![](<images/Pasted image 20260204132757.png>)

![](<images/Pasted image 20260204132833.png>)

Настраиваем архивирование WAL в PostgreSQL:

```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

Выставляем следующие параметры:

```bash
wal_level = replica
archive_mode = on
archive_command = '/usr/bin/pg_probackup-16 archive-push -B /var/lib/pg_probackup --instance=loyalty %p'
archive_timeout = 60
max_wal_senders = 5
```

Рестартуем PostgreSQL:

```bash
sudo systemctl restart postgresql
```

Делаем FULL BACKUP:

```bash
sudo -u postgres /usr/bin/pg_probackup-16 backup \
  -B /var/lib/pg_probackup \
  --instance=loyalty \
  -b FULL \
  --stream
```

Столкнулся с ошибкой, что у нас используется PSQL 14, а версия pg_probackup собрана под PSQL 16.

![](<images/Pasted image 20260204134023.png>)

Устанавливаем pg_probackup-14:

```bash
sudo apt install pg-probackup-14
```

Проверяем создание бекапа уже на новой версии:

```bash
sudo -u postgres /usr/bin/pg_probackup-14 backup \
  -B /var/lib/pg_probackup \
  --instance=loyalty \
  -b FULL \
  --stream
```

![](<images/Pasted image 20260204133704.png>)

Список бекапов:

![](<images/Pasted image 20260204133728.png>)

Удалим бекап с ошибкой и сделаем валидацию корректного бекапа:

```bash
sudo -u postgres /usr/bin/pg_probackup-14 delete \
  -B /var/lib/pg_probackup \
  --instance=loyalty \
  --backup-id=T9XK1P
```

```bash
sudo -u postgres /usr/bin/pg_probackup-14 validate \
  -B /var/lib/pg_probackup \
  --instance=loyalty
```

![](<images/Pasted image 20260204134234.png>)

Восстанавливать бекап будем на другом кластере на другой ВМ. Для этого сначала подготовим окружение (установка psql и pg_probackup):

![](<images/Pasted image 20260204134754.png>)

Создаём каталог для бэкапов:

```bash
sudo mkdir -p /var/lib/pg_probackup
sudo chown postgres:postgres /var/lib/pg_probackup
```

Через rsync передаем файлы бекапа на vm-2:

![](<images/Pasted image 20260204135506.png>)

Проверяем, что бекап передался и корректно отображается на новом кластере:

```bash
sudo -u postgres /usr/bin/pg_probackup-14 show \
  -B /var/lib/pg_probackup
```

![](<images/Pasted image 20260204135554.png>)

Далее будем восстанавливать данные из бекапа. Для этого останавливаем PostgreSQL на новом кластере:

```bash
sudo systemctl stop postgresql
```

Очищаем дата-директорию PostgreSQL:

```bash
sudo rm -rf /var/lib/postgresql/14/main/*
```

Восстанавливаемся из бекапа:

```bash
sudo -u postgres /usr/bin/pg_probackup-14 restore \
  -B /var/lib/pg_probackup \
  --instance=loyalty \
  -D /var/lib/postgresql/14/main \
  --recovery-target=latest
```

![](<images/Pasted image 20260204140207.png>)

Запускаем службу PostgreSQL:

```bash
sudo systemctl start postgresql
```

Подключаемся и проверяем целостность данных:

![](<images/Pasted image 20260204140353.png>)

Данные есть, восстановление прошло успешно.