Конфигурация стенда:

1) ВМ с Ubuntu 22.04, RAM 4 GB, CPU 4;

| **Hostname**       | **IP**      |
| ------------------ | ----------- |
| mefanov-otus-hw6-2 | 10.92.5.192 |

Установим minikube и cubectl:

![](<images/Pasted image 20260218154543.png>)

Запускаем minikube:

![](<images/Pasted image 20260218155241.png>)

Создаем файл манифеста postgres.yaml, описывая в нем ресурсы для развертывания:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              value: "postgres"
            - name: POSTGRES_PASSWORD
              value: "postgres"
            - name: POSTGRES_DB
              value: "postgres"
---
apiVersion: v1
kind: Service
metadata:
  name: postgres        
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

Сохраняем файл и применяем манифест:

```bash
kubectl apply -f postgres.yaml
```

Проверяем результат:

![](<images/Pasted image 20260218155628.png>)

Делаем проброс портов, чтобы подключиться к базе и локальной ВМ, потому что изначально "Service" типа "ClusterIP" доступен только внутри кластера:

![](<images/Pasted image 20260218160031.png>)

В номом окне терминала подключаемся к БД и проверяем версию PostgreSQL:

![](<images/Pasted image 20260218160131.png>)

Теперь сделаем масштабирование (увеличение количества подов) и проверяем результат:

```bash
kubectl scale deployment postgres --replicas=3
```

Kubernetes создал ещё 2 пода и теперь их стало 3:

![](<images/Pasted image 20260218160258.png>)

Теперь развернем PostgreSQL через Helm. Для начала скачаем и установим Helm:

![](<images/Pasted image 20260218160616.png>)

Установим PostgreSQL через Helm из хелм-чарта bitnamicharts/postgresql. Создаем файл values.yaml. Параметры architecture=replication - включает режим primary + replica, а readReplicas.replicaCount=2 - создаёт 2 read-реплики:

```bash
auth:
  username: postgres
  password: postgres
  database: postgres

architecture: replication

readReplicas:
  replicaCount: 2
```

Установим хелм-чарт:

```bash
helm install postgres oci://registry-1.docker.io/bitnamicharts/postgresql -f values.yaml
```

Проверяем поды:

![](<images/Pasted image 20260218162418.png>)