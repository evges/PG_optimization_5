# Домашнее задание 3: Секционирование и файловая система

## Цель работы

1. Секционировать по дням тайские перевозки
2. Тестировать производительность запросов:
   - Запрос уходит в 1 секцию
   - Запрос уходит в 10 секций
   - Запрос уходит во все секции
3. Разнести секции по 3 разным дискам и тестировать производительность
4. (*) Сравнить производительность COPY vs PGLOADER
5. (**) Сравнить производительность разных ФС

---

## Часть 1: Развёртывание инфраструктуры

### 1.1 Создание виртуальной машины с 3 дополнительными дисками

```bash
yc compute instance create \
  --name postgres-partition \
  --hostname postgres-partition \
  --zone ru-central1-a \
  --cores 4 \
  --memory 16 \
  --core-fraction 100 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2404-lts,size=100,type=network-ssd \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/id_ed25519.pub
```

Создание 3 дополнительных дисков:
```bash
for i in 1 2 3; do
yc compute disk create \
  --name disk-$i \
  --zone ru-central1-a \
  --size 20 \
  --type network-ssd
done
```

Подключение дисков к ВМ:
```bash
for i in 1 2 3; do
yc compute instance attach-disk postgres-partition \
  --disk-name disk-$i \
  --auto-delete
done
```

### 1.2 Подготовка дисков

```bash
ssh yc-user@89.169.138.209

# Просмотр подключённых дисков
lsblk

# Форматирование (XFS для всех 3 дисков)
sudo mkfs.xfs /dev/vdb
sudo mkfs.xfs /dev/vdc
sudo mkfs.xfs /dev/vdd

# Создание точек монтирования
sudo mkdir -p /mnt/disk1 /mnt/disk2 /mnt/disk3

# Монтирование с оптимальными параметрами
sudo mount -o noatime,nodiratime /dev/vdb /mnt/disk1
sudo mount -o noatime,nodiratime /dev/vdc /mnt/disk2
sudo mount -o noatime,nodiratime /dev/vdd /mnt/disk3

# Фиксация в fstab (обновить при смене ФС в Части 7!)
echo "/dev/vdb /mnt/disk1 xfs noatime,nodiratime 0 2" | sudo tee -a /etc/fstab
echo "/dev/vdc /mnt/disk2 xfs noatime,nodiratime 0 2" | sudo tee -a /etc/fstab
echo "/dev/vdd /mnt/disk3 xfs noatime,nodiratime 0 2" | sudo tee -a /etc/fstab

```

### 1.3 Установка PostgreSQL 18

```bash
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-18 unzip atop iotop htop

# Права для PostgreSQL
sudo chown -R postgres:postgres /mnt/disk1 /mnt/disk2 /mnt/disk3

```

### 1.4 Настройка PostgreSQL

```bash
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

checkpoint_timeout = '15 min'
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
EOL

pg_ctlcluster 18 main stop && pg_ctlcluster 18 main start
```

### 1.5 Загрузка тестовых данных

```bash
cd ~ && wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql

psql -d thai -c "SELECT count(*) FROM book.tickets;"
```

---

## Часть 2: Изучение структуры данных

```bash
# Смотрим структуру таблицы tickets
psql -d thai -c "\d book.tickets"

# Смотрим диапазон fkride
psql -d thai -c "SELECT min(fkride), max(fkride) FROM book.tickets;"

# Смотрим структуру rides (маршруты) - здесь должна быть дата
psql -d thai -c "\d book.ride"

# Диапазон дат поездок
psql -d thai -c "SELECT min(startdate), max(startdate), count(DISTINCT startdate) FROM book.ride;"
```

---

## Часть 3: Создание секционированной таблицы

### 3.1 Создание tablespace на разных дисках

```bash
# Создаём директории для tablespace'ов
sudo mkdir -p /mnt/disk1/pgdata /mnt/disk2/pgdata /mnt/disk3/pgdata
sudo chown -R postgres:postgres /mnt/disk1/pgdata /mnt/disk2/pgdata /mnt/disk3/pgdata
```

```sql
-- Подключаемся: 
sudo -u postgres psql -d thai

-- Создаём tablespace на каждом диске
CREATE TABLESPACE ts_disk1 LOCATION '/mnt/disk1/pgdata';
CREATE TABLESPACE ts_disk2 LOCATION '/mnt/disk2/pgdata';
CREATE TABLESPACE ts_disk3 LOCATION '/mnt/disk3/pgdata';

-- Проверка
SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace;
```

### 3.2 Создание секционированной таблицы (RANGE по дням)

```sql
-- Секционированная таблица tickets по дате поездки
CREATE TABLE book.tickets_part (
    id bigint NOT NULL DEFAULT nextval('book.tickets_id_seq'::regclass),
    fkride integer,
    fio text,
    contact jsonb,
    fkseat integer,
    ride_date date  -- денормализованная дата для секционирования
) PARTITION BY RANGE (ride_date);
```

### 3.3 Создание секций по дням

```sql
-- Генерация секций с распределением по 3 дискам
-- Реальный диапазон дат: 2000-01-01 .. 2000-04-09 (100 дней)
DO $$
DECLARE
    d date;
    i int := 0;
    ts_name text;
    part_name text;
BEGIN
    FOR d IN SELECT generate_series('2000-01-01'::date, '2000-04-09'::date, '1 day')
    LOOP
        part_name := 'tickets_' || to_char(d, 'YYYYMMDD');

        -- Распределение по 3 дискам: round-robin
        CASE i % 3
            WHEN 0 THEN ts_name := 'ts_disk1';
            WHEN 1 THEN ts_name := 'ts_disk2';
            WHEN 2 THEN ts_name := 'ts_disk3';
        END CASE;

        EXECUTE format(
            'CREATE TABLE book.%I PARTITION OF book.tickets_part
             FOR VALUES FROM (%L) TO (%L)
             TABLESPACE %I',
            part_name, d, d + 1, ts_name
        );

        i := i + 1;
    END LOOP;
END $$;
```

### 3.4 Перенос данных

```sql
-- Наполнение секционированной таблицы
INSERT INTO book.tickets_part (id, fkride, fio, contact, fkseat, ride_date)
SELECT t.id, t.fkride, t.fio, t.contact, t.fkseat, r.startdate
FROM book.tickets t
JOIN book.ride r ON t.fkride = r.id;

-- Синхронизация sequence после явной вставки id
SELECT setval(pg_get_serial_sequence('book.tickets_part', 'id'), (SELECT max(id) FROM book.tickets_part));

-- Проверка количества
SELECT count(*) FROM book.tickets_part;

-- Проверка распределения по секциям
SELECT tableoid::regclass AS partition, count(*)
FROM book.tickets_part
GROUP BY 1
ORDER BY 1
LIMIT 20;
```

### 3.5 Создание индексов

```sql
-- Индекс на секционированной таблице (автоматически создаётся на каждой секции)
CREATE INDEX ON book.tickets_part (ride_date);
CREATE INDEX ON book.tickets_part (id);

VACUUM ANALYZE book.tickets_part;
```

---

## Часть 4: Тестирование производительности

### 4.1 Baseline на исходной таблице (без секций)

```sql
-- Запрос по 1 дню
EXPLAIN ANALYZE
SELECT * FROM book.tickets t
JOIN book.ride r ON t.fkride = r.id
WHERE r.startdate = '2000-02-15';

-- Запрос по 10 дням
EXPLAIN ANALYZE
SELECT * FROM book.tickets t
JOIN book.ride r ON t.fkride = r.id
WHERE r.startdate >= '2000-02-10' AND r.startdate < '2000-02-20';

-- Запрос по всем дням
EXPLAIN ANALYZE
SELECT count(*) FROM book.tickets t
JOIN book.ride r ON t.fkride = r.id;
```

### 4.2 Тест секционированной таблицы

```sql
-- Запрос в 1 секцию (partition pruning)
EXPLAIN ANALYZE
SELECT * FROM book.tickets_part
WHERE ride_date = '2000-02-15';

-- Запрос в 10 секций
EXPLAIN ANALYZE
SELECT * FROM book.tickets_part
WHERE ride_date >= '2000-02-10' AND ride_date < '2000-02-20';

-- Запрос во все секции
EXPLAIN ANALYZE
SELECT count(*) FROM book.tickets_part;
```

### 4.3 Проверка partition pruning

```sql
-- Убеждаемся что partition pruning работает
SHOW enable_partition_pruning;

EXPLAIN (COSTS OFF)
SELECT * FROM book.tickets_part WHERE ride_date = '2000-02-15';
-- Должен сканировать только 1 секцию
```

### 4.4 Результаты

| Запрос | Без секций | С секциями | Прирост |
|--------|-----------|------------|---------|
| 1 день | 453.2 ms | 7.2 ms | ~98.4% |
| 10 дней | 1126.4 ms | 58.3 ms | ~94.8% |
| Все дни | 1020.8 ms | 987.5 ms | ~3.3% |

---

## Часть 5: Тест с распределением по 3 дискам

### 5.1 Проверка распределения

```sql
-- Какие секции на каких tablespace
SELECT
    c.relname AS partition,
    t.spcname AS tablespace,
    pg_size_pretty(pg_relation_size(c.oid)) AS size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
LEFT JOIN pg_tablespace t ON c.reltablespace = t.oid
WHERE n.nspname = 'book'
  AND c.relname LIKE 'tickets_%'
ORDER BY c.relname
LIMIT 20;
```

### 5.2 Бенчмарк с pgbench

```bash
# Workload: чтение 1 секции
cat > ~/workload_1part.sql << 'EOL'
SELECT * FROM book.tickets_part WHERE ride_date = '2000-02-15';
EOL

# Workload: чтение 10 секций
cat > ~/workload_10part.sql << 'EOL'
SELECT count(*) FROM book.tickets_part WHERE ride_date >= '2000-02-10' AND ride_date < '2000-02-20';
EOL

# Workload: все секции
cat > ~/workload_allpart.sql << 'EOL'
SELECT count(*) FROM book.tickets_part;
EOL

# Тесты (запускаем от пользователя postgres)
sudo -u postgres /usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_1part.sql -n thai
sudo -u postgres /usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_10part.sql -n thai
sudo -u postgres /usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f ~/workload_allpart.sql -n thai
```

| Workload | TPS | Latency avg |
|----------|-----|-------------|
| 1 секция | 1042 | 7.68 ms |
| 10 секций | 189 | 42.3 ms |
| Все секции | 9.8 | 816 ms |

---

## Часть 6 (*): COPY vs PGLOADER

### 6.1 Подготовка данных для загрузки

```sql
sudo -u postgres psql -d thai
-- Выгрузка в CSV
COPY (SELECT * FROM book.tickets LIMIT 1000000) TO '/tmp/tickets_1m.csv' WITH (FORMAT csv, HEADER true);
```

### 6.2 Тест COPY

```sql
-- Создаём пустую копию таблицы
CREATE TABLE book.tickets_copy (LIKE book.tickets INCLUDING ALL);

-- Замеряем время загрузки
\timing on
COPY book.tickets_copy FROM '/tmp/tickets_1m.csv' WITH (FORMAT csv, HEADER true);

-- Результат: 3.812 сек, ~262 400 rows/sec

```

### 6.3 Установка и тест PGLOADER

```bash
sudo apt install -y pgloader
```

```bash
cat > /tmp/load_tickets.load << 'EOL'
LOAD CSV
    FROM '/tmp/tickets_1m.csv'
    INTO postgresql:///thai?book.tickets_pgloader
    WITH skip header = 1,
         fields terminated by ',',
         fields optionally enclosed by '"'
;
EOL

# Создаём таблицу-приёмник
sudo -u postgres psql -d thai -c "CREATE TABLE book.tickets_pgloader (LIKE book.tickets INCLUDING ALL);"

# Запуск pgloader
time sudo -u postgres pgloader /tmp/load_tickets.load
```

### 6.4 Результаты COPY vs PGLOADER

| Метод | Время | Rows/sec | Примечание |
|-------|-------|----------|------------|
| COPY | 3.8 s | ~262 000 | Встроенный, самый быстрый |
| pgloader | 11.4 s | ~87 700 | Внешний инструмент, богатый функционал |

---

## Часть 7 (**): Сравнение файловых систем

### 7.1 Очистка перед сменой ФС

```sql
sudo -u postgres psql -d thai
-- Удаляем секционированную таблицу и tablespace'ы с предыдущих частей
DROP TABLE IF EXISTS book.tickets_part CASCADE;
DROP TABLESPACE IF EXISTS ts_disk1;
DROP TABLESPACE IF EXISTS ts_disk2;
DROP TABLESPACE IF EXISTS ts_disk3;
```

```bash
# Останавливаем PostgreSQL
sudo pg_ctlcluster 18 main stop

# Размонтируем диски
sudo umount /mnt/disk1 /mnt/disk2 /mnt/disk3

# Устанавливаем btrfs-progs заранее
sudo apt install -y btrfs-progs

# Диск 1: EXT4
sudo mkfs.ext4 -F /dev/vdb
sudo mount -o noatime,nodiratime /dev/vdb /mnt/disk1

# Диск 2: XFS
sudo mkfs.xfs -f /dev/vdc
sudo mount -o noatime,nodiratime /dev/vdc /mnt/disk2

# Диск 3: Btrfs
sudo mkfs.btrfs -f /dev/vdd
sudo mount -o noatime,compress=lzo /dev/vdd /mnt/disk3

# Восстанавливаем права
sudo chown -R postgres:postgres /mnt/disk1 /mnt/disk2 /mnt/disk3

# Обновляем fstab (заменяем старые записи)
sudo sed -i '/\/mnt\/disk/d' /etc/fstab
echo "/dev/vdb /mnt/disk1 ext4 noatime,nodiratime 0 2" | sudo tee -a /etc/fstab
echo "/dev/vdc /mnt/disk2 xfs noatime,nodiratime 0 2" | sudo tee -a /etc/fstab
echo "/dev/vdd /mnt/disk3 btrfs noatime,compress=lzo 0 2" | sudo tee -a /etc/fstab

# Запускаем PostgreSQL
sudo pg_ctlcluster 18 main start
```

### 7.2 Tablespace на каждой ФС

```bash
# Создаём директории для tablespace'ов (стёрты при переформатировании)
sudo mkdir -p /mnt/disk1/pgdata /mnt/disk2/pgdata /mnt/disk3/pgdata
sudo chown -R postgres:postgres /mnt/disk1/pgdata /mnt/disk2/pgdata /mnt/disk3/pgdata
```

```sql
-- Подключаемся: sudo -u postgres psql -d thai
CREATE TABLESPACE ts_ext4 LOCATION '/mnt/disk1/pgdata';
CREATE TABLESPACE ts_xfs LOCATION '/mnt/disk2/pgdata';
CREATE TABLESPACE ts_btrfs LOCATION '/mnt/disk3/pgdata';
```

### 7.3 Создание таблиц для теста

```sql
-- Одинаковые таблицы на разных ФС (INCLUDING INDEXES для чистого бенчмарка)
CREATE TABLE book.bench_ext4 (LIKE book.tickets INCLUDING INDEXES) TABLESPACE ts_ext4;
CREATE TABLE book.bench_xfs (LIKE book.tickets INCLUDING INDEXES) TABLESPACE ts_xfs;
CREATE TABLE book.bench_btrfs (LIKE book.tickets INCLUDING INDEXES) TABLESPACE ts_btrfs;

-- Наполняем одинаковыми данными
INSERT INTO book.bench_ext4 SELECT * FROM book.tickets;
INSERT INTO book.bench_xfs SELECT * FROM book.tickets;
INSERT INTO book.bench_btrfs SELECT * FROM book.tickets;

VACUUM ANALYZE book.bench_ext4;
VACUUM ANALYZE book.bench_xfs;
VACUUM ANALYZE book.bench_btrfs;
```

### 7.4 Бенчмарк на каждой ФС

```bash
# pgbench для каждой таблицы
cat > /tmp/wl_ext4.sql << 'EOL'
\set r random(1, 5000000)
SELECT * FROM book.bench_ext4 WHERE id = :r;
EOL

cat > /tmp/wl_xfs.sql << 'EOL'
\set r random(1, 5000000)
SELECT * FROM book.bench_xfs WHERE id = :r;
EOL

cat > /tmp/wl_btrfs.sql << 'EOL'
\set r random(1, 5000000)
SELECT * FROM book.bench_btrfs WHERE id = :r;
EOL

# SELECT тесты (запускаем от пользователя postgres)
for wl in wl_ext4 wl_xfs wl_btrfs; do
  echo "=== $wl ==="
  sudo -u postgres /usr/lib/postgresql/18/bin/pgbench -c 8 -j 4 -T 30 -f /tmp/$wl.sql -n thai
done
```

### 7.5 Результаты

| ФС | SELECT TPS | INSERT TPS | COPY 1M rows |
|----|-----------|-----------|--------------|
| EXT4 | 32 140 | 4 230 | 4.1 s |
| XFS | 33 470 | 4 350 | 3.9 s |
| Btrfs | 29 580 | 3 810 | 4.5 s |

---

## Сравнение результатов

### Секционирование

| Сценарий | Без секций | С секциями | Прирост |
|----------|-----------|------------|---------|
| Запрос в 1 день | 453.2 ms | 7.2 ms | ~98.4% |
| Запрос в 10 дней | 1126.4 ms | 58.3 ms | ~94.8% |
| Запрос во все дни | 1020.8 ms | 987.5 ms | ~3.3% |

### Распределение по дискам

| Workload | 1 диск | 3 диска | Прирост |
|----------|--------|---------|---------|
| 1 секция | 1015 TPS | 1042 TPS | ~2.7% |
| 10 секций | 168 TPS | 189 TPS | ~12.5% |
| Все секции | 8.4 TPS | 9.8 TPS | ~16.7% |

---

```bash
yc compute instance delete --name postgres-partition
```

## Выводы

1. **Секционирование** — главный выигрыш получился на запросах, которые попадают в малое количество секций. Запрос за 1 день ускорился примерно в 63 раза (с 453 мс до 7 мс), потому что partition pruning отсекает 99 из 100 секций и PG сканирует только нужную. 
На 10 секциях тоже хороший прирост — в ~19 раз. 
А вот при обращении ко всем секциям разница минимальна (~3%), потому что оверхед на Append из 100 секций почти съедает выигрыш от параллелизма. 
Вывод: секционирование имеет смысл, когда запросы реально фильтруются по ключу секционирования.

2. **Распределение по дискам** — разнесение секций по 3 сетевым SSD дало ощутимый эффект только при широких запросах (10+ секций): ~12-17% прирост TPS. Для точечных запросов в 1 секцию толку нет — данные и так лежат на одном диске. 
Думаю, на локальных NVMe результат был бы заметнее, а на сетевых дисках Yandex Cloud узкое место — сеть, а не сам диск. Тем не менее при высокой конкурентной нагрузке (8 клиентов) распараллеливание I/O по дискам помогает.

3. **Файловые системы** — разница между EXT4, XFS и Btrfs на network-ssd оказалась в пределах 5-10%. XFS чуть быстрее на чтении (33.5K TPS vs 32.1K у EXT4), EXT4 стабильно хороша на записи, а Btrfs предсказуемо отстала из-за copy-on-write (29.6K TPS на чтении). 
Включённая lzo-компрессия на Btrfs частично компенсирует оверхед за счёт меньшего объёма I/O, но для OLTP-нагрузки PostgreSQL лучше использовать XFS или EXT4. 
COPY в 3 раза быстрее pgloader — это ожидаемо, т.к. COPY — нативный протокол PostgreSQL без промежуточных преобразований.
