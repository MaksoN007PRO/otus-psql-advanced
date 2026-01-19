Установим PostgreSQL 18 на ОС Ubuntu 24.04 по официальной инструкции с postgresql.org:

```bash
sudo apt install -y postgresql-common  
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

```bash
sudo apt install postgresql-18
```

Проверка корректности работы службы:

![](<images/Pasted image 20260119152121.png>)

Подключаемся к БД, создаем таблицу и заполняем её данными:

![](<images/Pasted image 20260119152230.png>)

Проверяем изначальный список дисков:

![](<images/Pasted image 20260119152407.png>)

Создаем диск в ЯО:

![](<images/Pasted image 20260119152738.png>)

Присоединяем созданный диск:

![](<images/Pasted image 20260119152810.png>)

После рестарта ВМ появился новый присоединенный диск на 45 гб:

![](<images/Pasted image 20260119152929.png>)

Форматируем диск в ext4 и монтируем его:

![](<images/Pasted image 20260119153443.png>)

Переносом данных данных остановим PostgreSQL:

```bash
sudo systemctl stop postgresql
```

Копируем данные на новый диск:

```bash
sudo rsync -av /var/lib/postgresql/18/main/ /mnt/pgdata/main/
```

Назначаем права:

```bash
sudo chown -R postgres:postgres /mnt/pgdata
sudo chmod 700 /mnt/pgdata/main
```

Настраиваем PSQL для работы с новым диском. Для этого редактируем файл конфигурации и прописываем туда новую директорию PostgeSQL:

```bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

![](<images/Pasted image 20260119153819.png>)

Стартуем сервис PosgreSQL и проверяем сохранность данных:

![](<images/Pasted image 20260119153911.png>)