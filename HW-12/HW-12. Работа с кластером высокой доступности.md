Конфигурация стенда:

1) 3 ВМ с ETCD (Ubuntu 22.04);
2) 3 ВМ с PG+Patroni (Ubuntu 22.04);
3) 1 ВМ с HAProxy (Ubuntu 22.04).
   
| **Hostname**           | **IP**      |
| ---------------------- | ----------- |
| mefanov-otus-etcd-1    | 10.92.5.31  |
| mefanov-otus-etcd-2    | 10.92.5.130 |
| mefanov-otus-etcd-3    | 10.92.5.12  |
| mefanov-otus-pg-1      | 10.92.5.134 |
| mefanov-otus-pg-2      | 10.92.5.143 |
| mefanov-otus-pg-3      | 10.92.5.159 |
| mefanov-otus-haproxy-1 | 10.92.5.132 |
Скриншот конфигурации стенда из Яндекс Облака:
   
![](<images/Pasted image 20260127141916.png>)

Устанавливаем и настраиваем etcd (на всех etcd-нодах):

![](<images/Pasted image 20260126195723.png>)

Соответствующие настройки с изменением ETCD_NAME и IP делаем на других нодах ETCD:

![](<images/Pasted image 20260126195801.png>)

Запускаем и проверяем состояние ETCD кластера:

```bash
sudo systemctl enable etcd
sudo systemctl restart etcd
sudo systemctl status etcd
```

ETCD кластер готов и корректно функционирует:

![](<images/Pasted image 20260126195734.png>)

Устанавливаем PostgreSQL + Patroni на всех pg-нодах:

```bash
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/postgresql.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/postgresql.gpg] \
http://apt.postgresql.org/pub/repos/apt \
$(lsb_release -cs)-pgdg main" \
| sudo tee /etc/apt/sources.list.d/pgdg.list
```

```bash
sudo apt update
```

```bash
sudo apt install -y postgresql-14 postgresql-client-14
```

Проверяем версию установленного PostgreSQL на pg-нодах:

![](<images/Pasted image 20260126200639.png>)

Далее останавливаем и отключаем PostgreSQL (чтобы в дальнейшем им управлял Patroni):

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

![](<images/Pasted image 20260126200639.png>)

![](<images/Pasted image 20260126200800.png>)

Устанавливаем Patroni на всех pg-нодах:

```bash
sudo apt install -y python3-pip python3-dev libpq-dev
sudo pip3 install patroni[etcd]
```

Проверяем версию установленного Patroni:

![](<images/Pasted image 20260126200933.png>)

Подготавливаем каталог данных PostgreSQL:

```bash
sudo mkdir -p /data/patroni
sudo chown postgres:postgres /data/patroni
sudo chmod 700 /data/patroni
```

Конфигурируем кластер Patroni:

```bash
sudo nano /etc/patroni.yml
```

На ВМ mefanov-otus-pg-1:

```yaml
scope: banana_cluster
namespace: /service/
name: pg-1

restapi:
  listen: 10.92.5.134:8008
  connect_address: 10.92.5.134:8008

etcd:
  hosts: 10.92.5.31:2379,10.92.5.130:2379,10.92.5.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 64MB

  initdb:
    - encoding: UTF8
    - data-checksums

  users:
    replicator:
      password: repl_pass
      options:
        - replication

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.92.5.134:5432
  data_dir: /data/patroni
  bin_dir: /usr/lib/postgresql/14/bin
  pg_hba:
    - host replication replicator 10.92.5.143/32 md5
    - host replication replicator 10.92.5.159/32 md5
    - host all all 10.92.5.0/24 md5
    - host all all 127.0.0.1/32 trust
  authentication:
    replication:
      username: replicator
      password: repl_pass
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: "/var/run/postgresql"

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

На ВМ mefanov-otus-pg-2 и mefanov-otus-pg-3 идентично, заменяя адреса и имена:

```yaml
scope: banana_cluster
namespace: /service/
name: pg-2

restapi:
  listen: 10.92.5.143:8008
  connect_address: 10.92.5.143:8008

etcd:
  hosts: 10.92.5.31:2379,10.92.5.130:2379,10.92.5.12:2379

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.92.5.143:5432
  data_dir: /data/patroni
  bin_dir: /usr/lib/postgresql/14/bin
  authentication:
    replication:
      username: replicator
      password: repl_pass
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: "/var/run/postgresql"

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

Конфигурируем systemd-сервис Patroni на всех pg-нодах:

```bash
sudo nano /etc/systemd/system/patroni.service
```

```bash
[Unit]
Description=Patroni PostgreSQL HA Cluster
After=network.target

[Service]
User=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
Restart=always
LimitNOFILE=10240

[Install]
WantedBy=multi-user.target
```

Рестартуем демон и стартуем Patroni кластер на всех нодах:

```bash
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
```

Проверяем состояние кластера:

```bash
patronictl -c /etc/patroni.yml list
```

Кластер инициализирован и работает успешно:
1) pg-1 -  лидер
2) pg-2, pg-3 -  реплики (репликация идёт в режиме streaming)

![](<images/Pasted image 20260126202646.png>)

Далее установим и настроим HAProxy:

```bash
sudo apt update
sudo apt install -y haproxy
```

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

Составляем содержимое конфигурационного файла:

![](<images/Pasted image 20260126202840.png>)

Запускаем HAproxy:

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

![](<images/Pasted image 20260126202858.png>)

![](<images/Pasted image 20260126203144.png>)

Протестируем отказоустойчивость кластера. Для этого на ноде pg-1 имитируем отказ и проверяем состояние кластера:

```bash
sudo systemctl stop patroni
```

Как видим, на скриншоте, бывший лидер pg-1 остановлен и его место заняла нода pg-3:

![](<images/Pasted image 20260126203225.png>)

Кластер остался в рабочем состоянии: 

![](<images/Pasted image 20260126203304.png>)