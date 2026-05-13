# Домашнее задание 4: Настройка репликации и сравнение производительности

## Цель работы

1. Развернуть 2 варианта конфигурации реплик:
   - Синхронная + асинхронная реплика
   - Синхронная + асинхронная каскадная (снимаемая с синхронной)
2. Сравнить производительность (TPS) в разных режимах

---

## Часть 1: Развёртывание инфраструктуры

### 1.1 Создание 3 виртуальных машин

```bash
# Мастер
yc compute instance create \
  --name postgres-master \
  --hostname postgres-master \
  --zone ru-central1-a \
  --cores 4 \
  --memory 16 \
  --core-fraction 100 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2404-lts,size=50,type=network-ssd \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/id_ed25519.pub

# Реплика 1 (синхронная)
yc compute instance create \
  --name postgres-replica1 \
  --hostname postgres-replica1 \
  --zone ru-central1-a \
  --cores 4 \
  --memory 16 \
  --core-fraction 100 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2404-lts,size=50,type=network-ssd \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/id_ed25519.pub

# Реплика 2 (асинхронная / каскадная)
yc compute instance create \
  --name postgres-replica2 \
  --hostname postgres-replica2 \
  --zone ru-central1-a \
  --cores 4 \
  --memory 16 \
  --core-fraction 100 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2404-lts,size=50,type=network-ssd \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/id_ed25519.pub
```

Смотрим внутренние IP-адреса (они понадобятся для настройки репликации):
```bash
yc compute instance list
```

```
+----------------------+-------------------+---------------+---------+----------------+-------------+
|          ID          |       NAME        |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP |
+----------------------+-------------------+---------------+---------+----------------+-------------+
| fhm10pbpqk4nae45utp8 | postgres-replica1 | ru-central1-a | RUNNING | 93.77.179.247  | 10.128.0.19 |
| fhmhk6u6o0rbr5vockp0 | postgres-master   | ru-central1-a | RUNNING | 89.169.152.107 | 10.128.0.34 |
| fhmm6rrd6cb4vj5mpg9v | postgres-replica2 | ru-central1-a | RUNNING | 111.88.246.94  | 10.128.0.4  |
+----------------------+-------------------+---------------+---------+----------------+-------------+
```


### 1.2 Установка PostgreSQL 18 на всех 3 машинах

```bash
ssh yc-user@89.169.152.107
ssh yc-user@93.77.179.247
ssh yc-user@111.88.246.94
```

На каждой из 3 VM выполняем:
```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-18 unzip atop iotop htop
```

### 1.3 Настройка PostgreSQL на мастере

```bash
ssh yc-user@89.169.152.107

sudo su postgres

cat >> /etc/postgresql/18/main/postgresql.conf << EOL
shared_buffers = '4096 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '11 GB'
effective_io_concurrency = 100
random_page_cost = 1.1

shared_preload_libraries = 'pg_stat_statements'
track_io_timing = on
track_functions = pl

checkpoint_timeout  = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

wal_compression = on
wal_buffers = -1

max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
parallel_leader_participation = on

enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on

# Репликация
wal_level = replica
max_wal_senders = 4
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
synchronous_commit = on
EOL

# listen_addresses нельзя задать в конфиге до рестарта через postgresql.conf,
# т.к. pg_ctlcluster от postgres не работает (systemd). Делаем через ALTER SYSTEM:
psql -c "ALTER SYSTEM SET listen_addresses = '*';"

# Разрешаем подключение реплик (репликация)
cat >> /etc/postgresql/18/main/pg_hba.conf << EOL
host replication replicator 0.0.0.0/0 scram-sha-256
host all replicator 0.0.0.0/0 scram-sha-256
EOL

# Рестарт через systemd (из-под yc-user)
exit
sudo systemctl restart postgresql@18-main
sudo ss -tlnp | grep 5432
# Должно быть: 0.0.0.0:5432
```

> **Важный нюанс:** PostgreSQL, запущенный через systemd, нельзя рестартовать из-под postgres (`pg_ctlcluster` ругается). Нужно выходить в `yc-user` и делать `sudo systemctl restart postgresql@18-main`.

### 1.4 Загрузка тестовых данных

```bash
# На мастере (из-под postgres)
cd ~ && wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql

psql -d thai -c "SELECT count(*) FROM book.tickets;"
```

### 1.5 Создание пользователя репликации и слотов

```bash
# На мастере
psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'ReplicaPass123';"

# Слоты для каждой реплики (WAL не удалятся пока реплика не получит их)
psql -c "SELECT pg_create_physical_replication_slot('replica1_slot');"
psql -c "SELECT pg_create_physical_replication_slot('replica2_slot');"

# Проверка
psql -c "SELECT slot_name, active FROM pg_replication_slots;"
```

```
   slot_name   | active 
---------------+--------
 replica1_slot | f
 replica2_slot | f
```

### 1.6 Дополнительные настройки на мастере

```bash
sudo su postgres

# Даём replicator права на чтение (для тестов SELECT с реплик)
psql -d thai -c "GRANT USAGE ON SCHEMA book TO replicator;"
psql -d thai -c "GRANT SELECT ON ALL TABLES IN SCHEMA book TO replicator;"

# Настраиваем .pgpass (чтобы pgbench не спрашивал пароль при подключении к репликам)
echo "*:5432:*:replicator:ReplicaPass123" > ~/.pgpass
chmod 600 ~/.pgpass
```

---

## Часть 2: Подготовка workload-скриптов

На мастере создаём файлы нагрузки:

```bash
# Workload: INSERT (основной тест для репликации)
cat > ~/workload_insert.sql << 'EOL'
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
    ceil(random()*100),
    (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text,
    ('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb,
    ceil(random()*100)
);
EOL

# Workload: SELECT (для проверки чтения с реплик)
cat > ~/workload_select.sql << 'EOL'
\set r random(1, 5000000)
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
EOL
```

---

## Часть 3: Baseline (без реплик)

Замеряем производительность мастера в одиночку:

```bash
# INSERT: 8 клиентов, 30 секунд
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai

# SELECT: 8 клиентов, 30 секунд
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_select.sql -n thai
```

| Workload | TPS | Latency avg |
|----------|-----|-------------|
| INSERT (без реплик) | 2817 | 2.839 ms |
| SELECT (без реплик) | 28427 | 0.281 ms |

---

## Часть 4: Вариант 1 — Синхронная + Асинхронная реплика

```
Master ──sync──> Replica1 (синхронная)
   └────async──> Replica2 (асинхронная)
```

### 4.1 Настройка реплик (pg_basebackup)

На **обеих репликах** настраиваем .pgpass и снимаем базовый бэкап:

```bash
ssh yc-user@93.77.179.247
sudo su postgres

# Файл с паролем для автоподключения к мастеру
cat > ~/.pgpass << EOL
89.169.152.107:5432:*:replicator:ReplicaPass123
EOL
chmod 0600 ~/.pgpass

# Останавливаем, удаляем данные, снимаем бэкап
pg_ctlcluster 18 main stop
rm -rf /var/lib/postgresql/18/main

time pg_basebackup -h 89.169.152.107 -p 5432 -U replicator -R -S replica1_slot -D /var/lib/postgresql/18/main
```

> Ключ `-R` автоматически создаёт `standby.signal` и прописывает `primary_conninfo` в `postgresql.auto.conf`.

Аналогично на **Replica2** (с `replica2_slot`):

```bash
ssh yc-user@111.88.246.94
sudo su postgres

cat > ~/.pgpass << EOL
89.169.152.107:5432:*:replicator:ReplicaPass123
EOL
chmod 0600 ~/.pgpass

pg_ctlcluster 18 main stop
rm -rf /var/lib/postgresql/18/main

time pg_basebackup -h 89.169.152.107 -p 5432 -U replicator -R -S replica2_slot -D /var/lib/postgresql/18/main
```

### 4.2 Задаём application_name на репликах

Без `application_name` мастер не сможет идентифицировать реплику для синхронного режима.

На **Replica1** перед стартом:
```bash
# Добавляем application_name в primary_conninfo
# Смотрим текущий conninfo
cat /var/lib/postgresql/18/main/postgresql.auto.conf

# Дописываем application_name
sed -i "s/primary_conninfo = '\(.*\)'/primary_conninfo = '\1 application_name=replica1'/" /var/lib/postgresql/18/main/postgresql.auto.conf

# Проверяем
cat /var/lib/postgresql/18/main/postgresql.auto.conf
```

На **Replica2**:
```bash
# Добавляем application_name в primary_conninfo
# Смотрим текущий conninfo
cat /var/lib/postgresql/18/main/postgresql.auto.conf

sed -i "s/primary_conninfo = '\(.*\)'/primary_conninfo = '\1 application_name=replica2'/" /var/lib/postgresql/18/main/postgresql.auto.conf

# Проверяем
cat /var/lib/postgresql/18/main/postgresql.auto.conf

```

### 4.3 Настройка конфига реплик и запуск

На **обеих репликах** задаём параметры:
```bash
sudo su postgres

cat >> /etc/postgresql/18/main/postgresql.conf << EOL
hot_standby = on
hot_standby_feedback = on
max_wal_senders = 4
EOL

# Разрешаем внешние подключения (для pgbench с мастера и каскадной репликации)
echo "host all replicator 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/18/main/pg_hba.conf
echo "host replication replicator 0.0.0.0/0 scram-sha-256" >> /etc/postgresql/18/main/pg_hba.conf

# listen_addresses = '*' уже задан через ALTER SYSTEM (скопировался с мастера при pg_basebackup)

pg_ctlcluster 18 main start
```

Проверяем что реплики работают:
```bash
psql -d thai -c "SELECT pg_is_in_recovery();"
# Должно быть: t

psql -d thai -c "SELECT count(*) FROM book.tickets;"
# Должно совпасть с мастером
```

### 4.4 Включение синхронной репликации на мастере

```bash
# На мастере (под postgres)
# Replica1 — синхронная, Replica2 — асинхронная
psql -c "ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (replica1)';"
psql -c "SELECT pg_reload_conf();"
```

Проверяем:
```sql
-- Статус репликации
psql -c "SELECT application_name, sync_state, state, sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;"
```

Ожидаемый вывод:
```
 application_name | sync_state |   state   |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn 
------------------+------------+-----------+------------+------------+------------+------------
 replica2         | async      | streaming | 0/26000060 | 0/26000060 | 0/26000060 | 0/26000060
 replica1         | sync       | streaming | 0/26000060 | 0/26000060 | 0/26000060 | 0/26000060
(2 rows)
```

### 4.5 Бенчмарк Варианта 1

```bash
# INSERT на мастере (синхронная запись на replica1)
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai
# Результат: tps = 1283 (latency 6.233 ms)

# SELECT с реплики 1 (проверяем что читать можно)
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_select.sql -n -h 10.128.0.19 -U replicator thai
# Результат: tps = 11725 (latency 0.682 ms)
```

| Workload | TPS | Latency avg |
|----------|-----|-------------|
| INSERT на мастер (sync replica1) | 1283 | 6.233 ms |
| SELECT с реплики 1 | 11725 | 0.682 ms |

### 4.6 Тест разных режимов synchronous_commit

```bash
# === remote_apply (самый надёжный, самый медленный) ===
psql -c "ALTER SYSTEM SET synchronous_commit = 'remote_apply';"
psql -c "SELECT pg_reload_conf();"
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai

# === on (ждём записи WAL на диск реплики) ===
psql -c "ALTER SYSTEM SET synchronous_commit = 'on';"
psql -c "SELECT pg_reload_conf();"
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai

# === local (только локальный диск мастера) ===
psql -c "ALTER SYSTEM SET synchronous_commit = 'local';"
psql -c "SELECT pg_reload_conf();"
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai

# === off (асинхронная запись, максимальная скорость) ===
psql -c "ALTER SYSTEM SET synchronous_commit = 'off';"
psql -c "SELECT pg_reload_conf();"
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai
```

### 4.7 Результаты Варианта 1

| synchronous_commit | TPS (INSERT) | Latency avg | Описание |
|-------------------|-------------|-------------|----------|
| Без реплик (baseline) | 2817 | 2.839 ms | Мастер один |
| remote_apply | 1249 | 6.401 ms | Ждём применения WAL на реплике |
| on | 1284 | 6.228 ms | Ждём записи WAL на диск реплики |
| local | 2635 | 3.035 ms | Только локальный fsync |
| off | 10841 | 0.738 ms | Полностью асинхронно |

---

## Часть 5: Вариант 2 — Синхронная + Каскадная асинхронная

```
Master ──sync──> Replica1 (синхронная)
                     └──async──> Replica2 (каскадная)
```

Идея: Replica2 реплицируется не с мастера, а с Replica1. Это снижает нагрузку на мастер (один WAL sender вместо двух).

### 5.1 Перенастройка Replica2 на каскад

На **мастере** удаляем слот replica2 (больше не нужен):
```bash
# На мастере (под postgres)
psql -c "SELECT pg_drop_replication_slot('replica2_slot');"
```

На **Replica1** создаём слот для replica2 и разрешаем входящую репликацию:
```bash
# На Replica1
sudo su postgres

psql -c "SELECT pg_create_physical_replication_slot('replica2_slot');"

cat >> /etc/postgresql/18/main/pg_hba.conf << EOL
host replication replicator 10.128.0.0/16 scram-sha-256
EOL

psql -c "SELECT pg_reload_conf();"
```

На **Replica2** переключаем источник на Replica1 (полный переналив):
```bash
# На Replica2
sudo su postgres

pg_ctlcluster 18 main stop
rm -rf /var/lib/postgresql/18/main

# Настраиваем .pgpass для подключения к Replica1
echo "10.128.0.19:5432:*:replicator:ReplicaPass123" > ~/.pgpass
chmod 0600 ~/.pgpass

# Basebackup С REPLICA1 (не с мастера!)
pg_basebackup -h 10.128.0.19 -p 5432 -U replicator -R -S replica2_slot -D /var/lib/postgresql/18/main

# Дописываем application_name
sed -i "s/primary_conninfo = '\(.*\)'/primary_conninfo = '\1 application_name=replica2'/" /var/lib/postgresql/18/main/postgresql.auto.conf

pg_ctlcluster 18 main start
```

Проверяем цепочку:

На **мастере:**
```sql
psql -c "SELECT application_name, sync_state, state FROM pg_stat_replication;"
-- Должна быть только replica1
```

На **Replica1:**
```sql
psql -c "SELECT application_name, state FROM pg_stat_replication;"
-- Должна быть replica2 (каскадная)
```

На **Replica2:**
```sql
psql -c "SELECT pg_is_in_recovery();"
-- t

psql -d thai -c "SELECT count(*) FROM book.tickets;"
-- Должно совпасть
```

### 5.2 Бенчмарк Варианта 2

```bash
# На мастере — возвращаем synchronous_commit для полноценного сравнения
psql -c "ALTER SYSTEM SET synchronous_commit = 'remote_apply';"
psql -c "SELECT pg_reload_conf();"
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai

psql -c "ALTER SYSTEM SET synchronous_commit = 'on';"
psql -c "SELECT pg_reload_conf();"
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai

psql -c "ALTER SYSTEM SET synchronous_commit = 'off';"
psql -c "SELECT pg_reload_conf();"
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n thai
```

### 5.3 Результаты Варианта 2 (каскад)

| synchronous_commit | TPS (INSERT) | Latency avg |
|-------------------|-------------|-------------|
| remote_apply | 1347 | 5.937 ms |
| on | 1370 | 5.839 ms |
| off | 11128 | 0.719 ms |

---

## Часть 6: Сравнение результатов

### Влияние synchronous_commit на INSERT TPS

| synchronous_commit | Без реплик | Вариант 1 (sync+async) | Вариант 2 (sync+cascade) |
|-------------------|-----------|------------------------|--------------------------|
| remote_apply | — | 1249 TPS | 1347 TPS |
| on | — | 1284 TPS | 1370 TPS |
| local | 2817 TPS | 2635 TPS | — |
| off | — | 10841 TPS | 11128 TPS |
| baseline (нет реплик) | 2817 TPS | — | — |

### Структура репликации

| Параметр | Вариант 1 | Вариант 2 |
|----------|-----------|-----------|
| WAL senders на мастере | 2 | 1 |
| Нагрузка на сеть мастера | Выше | Ниже |
| Задержка Replica2 | Минимальная | Выше (через Replica1) |

---

## Удаление инфраструктуры

```bash
yc compute instance delete --name postgres-master
yc compute instance delete --name postgres-replica1
yc compute instance delete --name postgres-replica2
```

---

## Выводы

1. synchronous_commit -- это главный фактор, влияющий на скорость записи. 
Между remote_apply (1249 TPS) и off (10841 TPS) разница почти в 9 раз. 
Всё потому, что в синхронном режиме каждая транзакция ждёт подтверждения от реплики через сеть.

2. Каскадная схема оказалась чуть быстрее прямой при синхронных режимах: 1347 vs 1249 для remote_apply (+8%), 1370 vs 1284 для on (+7%). 
Логично -- мастер обслуживает 1 WAL sender вместо 2, меньше нагрузка на сеть и диск.

3. В асинхронном режиме (off) разница между вариантами минимальна: 11128 vs 10841 TPS (~3%). 
Мастер не ждёт реплик, поэтому количество WAL sender-ов роли не играет.

4. Режим local - компромисс: 2635 TPS -- близко к baseline без реплик (2817), при этом данные гарантированно на диске мастера, а реплики получают WAL асинхронно.

5. Чтение с реплики через сеть примерно в 2.4 раза медленнее, чем локально с мастера: 11725 vs 28427 TPS. 
Это overhead на сетевое подключение (latency 0.682 ms vs 0.281 ms).

6. Каскадная репликация снижает нагрузку на мастер (1 WAL sender вместо 2), но увеличивает задержку для replica2 -- данные идут через промежуточный узел.
