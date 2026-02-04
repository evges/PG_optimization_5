## Цель работы

1. Сравнить производительность при 500+ коннектах:
   - Ванильный PostgreSQL (без пулера)
   - PgBouncer

## Настройка

```bash
yc compute instance create \
  --name postgres-pooler \
  --hostname postgres-pooler \
  --zone ru-central1-a \
  --cores 4 \
  --memory 16 \
  --core-fraction 100 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2404-lts,size=100,type=network-ssd \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/id_ed25519.pub
```


```bash
ssh yc-user@89.169.145.39

sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-18 unzip atop iotop htop
```

```bash
sudo su postgres

cat >> /etc/postgresql/18/main/postgresql.conf << EOL
shared_buffers = '4096 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '11 GB'
effective_io_concurrency = 100 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.1 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements'    # per statement resource usage stats
track_io_timing=on        # measure exact block IO times
track_functions=pl        # track execution times of pl-language procedures if any

# Checkpointing: 
checkpoint_timeout  = '15 min' 
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

# WAL writing
wal_compression = on
wal_buffers = -1    # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB

# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries: 
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
parallel_leader_participation = on

# Advanced features 
enable_partitionwise_join = on 
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 100

max_connections = 600
EOL

pg_ctlcluster 18 main stop && pg_ctlcluster 18 main start

cd ~ && wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql

psql -d thai -c "SELECT count(*) FROM book.tickets;"
  count  
---------
 5185505
(1 row)

**Workload 1: SELECT (чтение)**
```bash
cat > ~/workload_select.sql << 'EOL'
\set r random(1, 5000000)
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
EOL
```

```bash
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
```

```bash
sudo su postgres

# Задаём пароль пользователю postgres
psql -c "ALTER USER postgres WITH PASSWORD 'postgres123';"

# Создаём админ-пользователя для PgBouncer
psql -c "CREATE USER admindb WITH PASSWORD 'admin123';"

# Проверяем хеши паролей
psql -c "SELECT usename, passwd FROM pg_shadow;"
```
 usename  |                                                                passwd                                                                 
----------+---------------------------------------------------------------------------------------------------------------------------------------
 postgres | SCRAM-SHA-256$4096:syP2as/2n25IB4zV28DPLg==$hpzZ4TAAO2Cesqg0dq0VZaxXuhBubbqexkFL3O02qmo=:uqPcj1JjfEc4iFvl+Jlyfq/imKf5oOkkvQinssbNXtY=
 admindb  | SCRAM-SHA-256$4096:t5F/CCwD7ie2H46NRVKi8g==$egYFoXA3Lho+VCnb1Q50wW/7FJQgO0IzHK23xkRq2Xc=:uk214wL0tUfT9pDkrItuGLYrKHB7ut0qwIuIqQFsKOI=
(2 rows)

echo "host    all    all    127.0.0.1/32    scram-sha-256" | sudo tee -a /etc/postgresql/18/main/pg_hba.conf

```bash
# Для пользователя postgres
echo "localhost:5432:*:postgres:postgres123" | sudo tee /var/lib/postgresql/.pgpass
echo "127.0.0.1:5432:*:postgres:postgres123" | sudo tee -a /var/lib/postgresql/.pgpass
echo "127.0.0.1:6432:*:postgres:postgres123" | sudo tee -a /var/lib/postgresql/.pgpass
sudo chmod 600 /var/lib/postgresql/.pgpass
sudo chown postgres:postgres /var/lib/postgresql/.pgpass
```

### Предварительный VACUUM

```bash
sudo su postgres
psql -c "VACUUM ANALYZE book.tickets;" -d thai
```

## 1 Baseline тесты (без пулера)

### Сравнение Unix Socket vs TCP

```bash
sudo su postgres

# Unix socket (самый быстрый, без сетевого стека)
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload_select.sql -n -U postgres thai
tps = 31625.099756 (without initial connection time)

# TCP через localhost (добавляется сетевой стек)
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload_select.sql -n -U postgres -h localhost -p 5432 thai
tps = 21314.935031 (without initial connection time)

```
TCP добавляет ~30% overhead

## 1.1 Тест SELECT 
```bash
sudo su postgres
# 8 клиентов (baseline)
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 21855.483179 (without initial connection time)

# 50 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 50 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 22745.548857 (without initial connection time)

# 100 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 100 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 20517.437637 (without initial connection time)

# 200 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 200 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 17670.606673 (without initial connection time)

# 500 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 500 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 16019.396214 (without initial connection time)
```
| Клиентов | TPS | Latency avg | Initial conn time |
|----------|-----|-------------|-------------------|
| 8 (socket) | **31 625** | 0.253 ms | 11 ms |
| 8 (TCP) | **21 315** | 0.375 ms | 65 ms |
| 50 | **22 746** | 2.198 ms | 400 ms |
| 100 | **20 517** | 4.874 ms | 785 ms |
| 200 | **17 671** | 11.318 ms | 1 546 ms |
| 500 | **16 019** | 31.212 ms | 3 766 ms |

## 1.1 Тест INSERT 
```bash
# 8 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 2372.375328 (without initial connection time)

# 100 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 100 -j 4 -T 30 -f ~/workload_insert.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 5810.467371 (without initial connection time)

# 500 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 500 -j 4 -T 30 -f ~/workload_insert.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 5637.659750 (without initial connection time)
```
| Клиентов | TPS | Latency avg |
|----------|-----|-------------|
| 8 | **2 372** | 3.372 ms |
| 100 | **5 810** | 17.210 ms |
| 500 | **5 638** | 88.689 ms |

## Установка и настройка PgBouncer
```bash
sudo DEBIAN_FRONTEND=noninteractive apt install -y pgbouncer
sudo systemctl stop pgbouncer
```
```bash
sudo tee /etc/pgbouncer/pgbouncer.ini << 'EOF'
[databases]
thai = host=127.0.0.1 port=5432 dbname=thai

[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid

listen_addr = *
listen_port = 6432

# Аутентификация SCRAM (как в PostgreSQL)
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Режим пулинга
pool_mode = transaction

# Размеры пулов
max_client_conn = 2000
default_pool_size = 100
min_pool_size = 10
reserve_pool_size = 20
reserve_pool_timeout = 3

# Таймауты
server_connect_timeout = 15
server_idle_timeout = 600
query_wait_timeout = 120

# Сброс состояния
server_reset_query = DISCARD ALL
server_reset_query_always = 1

# Логирование
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

# Админка
admin_users = admindb
EOF
```

```bash
sudo systemctl enable pgbouncer
sudo systemctl start pgbouncer
sudo systemctl status pgbouncer

# Проверка логов
sudo tail -20 /var/log/postgresql/pgbouncer.log
```

```bash
# Подключение к базе через PgBouncer
sudo -u postgres psql -p 6432 -h 127.0.0.1 -d thai -U postgres -c 'SELECT 1;'

# Подключение к админке PgBouncer
sudo -u postgres psql -p 6432 -h 127.0.0.1 -d pgbouncer -U admindb
```

SHOW POOLS;      -- статистика пулов
SHOW CLIENTS;    -- клиентские соединения  
SHOW SERVERS;    -- серверные соединения к PostgreSQL
SHOW STATS;      -- общая статистика
SHOW CONFIG;     -- текущая конфигурация

PAUSE thai;      -- приостановить пул (новые запросы ждут)
RESUME thai;     -- возобновить
RELOAD;          -- перечитать конфиг без рестарта

```bash
echo "127.0.0.1:6432:*:postgres:postgres123" | sudo tee -a /var/lib/postgresql/.pgpass
```

```bash
sudo su postgres

# Напрямую через TCP (baseline)
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 5432 thai
tps = 21605.364615 (without initial connection time)

# Через PgBouncer
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 6432 thai
tps = 9193.087959 (without initial connection time)
```

При малом количестве коннектов PgBouncer добавляет overhead!

## 1.2 Тест SELECT 

```bash
sudo su postgres

# 50 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 50 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 6432 thai
tps = 6794.783410 (without initial connection time)

# 100 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 100 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 6432 thai
tps = 6150.812989 (without initial connection time)

# 200 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 200 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 6432 thai
tps = 6006.861721 (without initial connection time)

# 500 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 500 -j 4 -T 30 -f ~/workload_select.sql -n -U postgres -h 127.0.0.1 -p 6432 thai
tps = 5661.679039 (without initial connection time)
```

| Клиентов | TPS | Latency avg | Initial conn time |
|----------|-----|-------------|-------------------|
| 8 | **9 193** | 0.870 ms | 42 ms |
| 50 | **6 795** | 7.359 ms | 271 ms |
| 100 | **6 151** | 16.258 ms | 560 ms |
| 200 | **6 007** | 33.295 ms | 1 065 ms |
| 500 | **5 662** | 88.313 ms | 2 695 ms |

### 1.2  Тест INSERT 
```bash
# 8 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_insert.sql -n -U postgres -h 127.0.0.1 -p 6432 thai
tps = 1817.141767 (without initial connection time)

# 100 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 100 -j 4 -T 30 -f ~/workload_insert.sql -n -U postgres -h 127.0.0.1 -p 6432 thai
tps = 3402.864979 (without initial connection time)

# 500 клиентов
/usr/lib/postgresql/18/bin/pgbench -c 500 -j 4 -T 30 -f ~/workload_insert.sql -n -U postgres -h 127.0.0.1 -p 6432 thai
tps = 3319.004409 (without initial connection time)
```

| Клиентов | TPS | Latency avg |
|----------|-----|-------------|
| 8 | **1 817** | 4.403 ms |
| 100 | **3 403** | 29.387 ms |
| 500 | **3 319** | 150.648 ms |
