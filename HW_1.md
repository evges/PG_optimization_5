# Домашнее задание 1

## Цель работы

1. Развернуть PostgreSQL любым удобным способом на Linux
2. Провести первичный бенчмарк (простой и расширенный вариант)
3. Настроить на оптимальную производительность
4. Настроить на максимальную производительность, не обращая внимание на ACID

---

## Часть 1: Развёртывание инфраструктуры

### 1.1 Создание виртуальной машины в Yandex Cloud

Для выполнения работы была создана виртуальная машина в Yandex Cloud:

```bash
yc compute instance create \
  --name postgres-bench \
  --hostname postgres-bench \
  --zone ru-central1-a \
  --cores 4 \
  --memory 16 \
  --core-fraction 100 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2404-lts,size=100,type=network-ssd \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/id_ed25519.pub
```

**Характеристики VM:**

| Параметр | Значение |
|----------|----------|
| CPU | 4 ядра (Intel Xeon Cascadelake) |
| RAM | 16 GB |
| Диск | 100 GB network-ssd |
| ОС | Ubuntu 24.04.3 LTS |
| Зона | ru-central1-a |

### 1.2 Установка PostgreSQL 18

```bash
# Обновление системы
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y

# Добавление официального репозитория PostgreSQL
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Установка PostgreSQL 18 и утилит мониторинга
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-18 postgresql-contrib-18
sudo apt -y install htop atop iotop sysstat
```

**Проверка установки:**

```bash
sudo -u postgres psql -c "SELECT version();"
```
```
PostgreSQL 18.1 (Ubuntu 18.1-1.pgdg24.04+2) on x86_64-pc-linux-gnu, 
compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, 64-bit
```

**Параметры системы:**

```
CPU:    Intel Xeon Processor (Cascadelake), 4 cores
Memory: 16 GB (15Gi available)
Disk:   97 GB SSD (network-ssd)
Swap:   0 B (отключён)
```

---

## Часть 2: Первичный бенчмарк (Baseline)

### 2.1 Инициализация тестовой базы

```bash
sudo su - postgres
pgbench -i -s 50 postgres
```

**Результат инициализации:**
```
done in 24.73 s (drop tables 0.00 s, create tables 0.01 s, 
client-side generate 18.58 s, vacuum 0.38 s, primary keys 5.75 s)
```


### 2.2 Простой бенчмарк (1 клиент)

```bash
pgbench -T 10 -P 1 postgres
```

**Результат:**
```
progress: 1.0 s, 193.0 tps, lat 4.936 ms stddev 4.975, 0 failed
progress: 2.0 s, 173.0 tps, lat 5.979 ms stddev 11.213, 0 failed
progress: 3.0 s, 144.0 tps, lat 6.961 ms stddev 10.104, 0 failed
progress: 4.0 s, 178.0 tps, lat 5.651 ms stddev 7.313, 0 failed
progress: 5.0 s, 161.0 tps, lat 6.126 ms stddev 8.492, 0 failed
progress: 6.0 s, 229.0 tps, lat 4.428 ms stddev 7.135, 0 failed
progress: 7.0 s, 182.0 tps, lat 5.153 ms stddev 8.724, 0 failed
progress: 8.0 s, 209.0 tps, lat 5.074 ms stddev 8.753, 0 failed
progress: 9.0 s, 249.0 tps, lat 3.961 ms stddev 7.283, 0 failed
progress: 10.0 s, 319.0 tps, lat 3.181 ms stddev 6.429, 0 failed

tps = 203.628667 (without initial connection time)
latency average = 4.910 ms
```

### 2.3 Расширенный бенчмарк (10 клиентов)

```bash
pgbench -c 10 -j 4 -T 60 -P 5 postgres
```

**Результат:**
```
progress: 5.0 s, 1636.8 tps, lat 6.075 ms stddev 11.358, 0 failed
progress: 10.0 s, 903.6 tps, lat 11.077 ms stddev 15.895, 0 failed
progress: 15.0 s, 1320.0 tps, lat 7.569 ms stddev 15.280, 0 failed
progress: 20.0 s, 1489.6 tps, lat 6.707 ms stddev 12.756, 0 failed
progress: 25.0 s, 1415.6 tps, lat 7.060 ms stddev 11.697, 0 failed
progress: 30.0 s, 1405.6 tps, lat 7.106 ms stddev 12.915, 0 failed
progress: 35.0 s, 1068.8 tps, lat 9.342 ms stddev 13.466, 0 failed
progress: 40.0 s, 572.8 tps, lat 17.413 ms stddev 23.350, 0 failed
progress: 45.0 s, 251.0 tps, lat 39.686 ms stddev 29.379, 0 failed  -- провал
progress: 50.0 s, 1307.2 tps, lat 7.702 ms stddev 14.134, 0 failed
progress: 55.0 s, 1154.0 tps, lat 8.653 ms stddev 18.788, 0 failed
progress: 60.0 s, 635.4 tps, lat 15.738 ms stddev 23.277, 0 failed

tps = 1096.886270 (without initial connection time)
latency average = 9.110 ms
```

**Наблюдение:** На 45-й секунде произошёл провал до 251 TPS. 
Причины:
- `max_wal_size = 1GB` (дефолт) — частые checkpoint при активной записи
- `shared_buffers = 128MB` (дефолт) — быстро накапливаются dirty pages

### 2.4 Бенчмарк только на чтение (SELECT)

```bash
pgbench -c 10 -j 4 -T 60 -P 5 -S postgres
```

**Результат:**
```
progress: 5.0 s, 25688.9 tps, lat 0.386 ms stddev 0.218, 0 failed
progress: 10.0 s, 26664.0 tps, lat 0.373 ms stddev 0.179, 0 failed
...
progress: 60.0 s, 27404.9 tps, lat 0.362 ms stddev 0.166, 0 failed

tps = 27281.881699 (without initial connection time)
latency average = 0.364 ms
```

### 2.5 Итоги Baseline

| Тест | TPS | Latency avg |
|------|-----|-------------|
| 1 клиент | **204** | 4.91 ms |
| 10 клиентов | **1 097** | 9.11 ms |
| SELECT only | **27 282** | 0.36 ms |

---

## Часть 3: Оптимальная настройка производительности

### 3.1 Настройка операционной системы

```bash
# Уменьшение swappiness (чтобы PostgreSQL не свопился)
sudo sysctl vm.swappiness=1
echo "vm.swappiness=1" | sudo tee -a /etc/sysctl.conf

# Отключение Transparent Huge Pages (вызывает latency spikes)
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

### 3.2 Настройка PostgreSQL

```sql
-- Память (для 16 GB RAM)
ALTER SYSTEM SET shared_buffers = '4GB';           -- 25% RAM
ALTER SYSTEM SET effective_cache_size = '12GB';    -- 75% RAM
ALTER SYSTEM SET work_mem = '64MB';                -- для сортировок и хэшей
ALTER SYSTEM SET maintenance_work_mem = '512MB';   -- для VACUUM, CREATE INDEX
ALTER SYSTEM SET wal_buffers = '64MB';             -- буфер WAL

-- Checkpoint (размазывание нагрузки)
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET min_wal_size = '1GB';
ALTER SYSTEM SET max_wal_size = '4GB';

-- Оптимизация для SSD
ALTER SYSTEM SET random_page_cost = 1.1;           -- SSD быстрый на random I/O
ALTER SYSTEM SET effective_io_concurrency = 200;   -- параллельный I/O

-- Параллелизм
ALTER SYSTEM SET max_parallel_workers_per_gather = 2;
ALTER SYSTEM SET max_parallel_workers = 4;
```

```bash
sudo systemctl restart postgresql
```

**Проверка:**
```bash
sudo -u postgres psql -c "SHOW shared_buffers;"
 shared_buffers 
----------------
 4GB
```

### 3.3 Бенчмарк после оптимизации

**Инициализация:**
```
done in 20.25 s (drop tables 0.12 s, create tables 0.01 s, 
client-side generate 17.64 s, vacuum 0.35 s, primary keys 2.13 s)
```
*Было 24.73 с -> стало 20.25 с*

**1 клиент:**
```
tps = 267.996693 (without initial connection time)
latency average = 3.731 ms
```

**10 клиентов:**
```
progress: 5.0 s, 1163.2 tps, lat 8.564 ms stddev 13.380, 0 failed
progress: 10.0 s, 1170.2 tps, lat 8.538 ms stddev 42.789, 0 failed
progress: 15.0 s, 1958.2 tps, lat 5.104 ms stddev 9.160, 0 failed
progress: 20.0 s, 1548.4 tps, lat 6.450 ms stddev 11.393, 0 failed
progress: 25.0 s, 1489.2 tps, lat 6.714 ms stddev 11.363, 0 failed
progress: 30.0 s, 1825.2 tps, lat 5.474 ms stddev 10.275, 0 failed
progress: 35.0 s, 1340.4 tps, lat 7.441 ms stddev 11.485, 0 failed
progress: 40.0 s, 1245.0 tps, lat 8.035 ms stddev 12.563, 0 failed
progress: 45.0 s, 1851.0 tps, lat 5.403 ms stddev 9.166, 0 failed  -- нет провала!
progress: 50.0 s, 1375.6 tps, lat 7.242 ms stddev 11.999, 0 failed
progress: 55.0 s, 1821.4 tps, lat 5.500 ms stddev 9.454, 0 failed
progress: 60.0 s, 1738.4 tps, lat 5.751 ms stddev 10.173, 0 failed

tps = 1544.164979 (without initial connection time)
latency average = 6.471 ms
```

**SELECT only:**
```
tps = 32893.018745 (without initial connection time)
latency average = 0.302 ms
```

### 3.4 Итоги оптимизации

| Тест | Baseline | Optimized | Прирост |
|------|----------|-----------|---------|
| 1 клиент | 204 TPS | **268 TPS** | **+31%** |
| 10 клиентов | 1 097 TPS | **1 544 TPS** | **+41%** |
| SELECT only | 27 282 TPS | **32 893 TPS** | **+21%** |

Провал до 251 TPS на 45-й секунде исчез — `max_wal_size = 4GB` уменьшил частоту checkpoint.

---

## Часть 4: Максимальная производительность (без ACID)

### 4.1 Отключение durability

```sql
-- Асинхронный commit (не ждём записи WAL на диск)
ALTER SYSTEM SET synchronous_commit = 'off';

-- ОПАСНО! Полное отключение fsync
ALTER SYSTEM SET fsync = 'off';
ALTER SYSTEM SET full_page_writes = 'off';

-- Редкие checkpoint (раз в 30 минут)
ALTER SYSTEM SET checkpoint_timeout = '30min';
ALTER SYSTEM SET max_wal_size = '16GB';
```

```bash
sudo systemctl restart postgresql
```

### 4.2 Бенчмарк

**Инициализация (в 4 раза быстрее!):**
```
done in 6.03 s (drop tables 0.12 s, create tables 0.00 s, 
client-side generate 4.00 s, vacuum 0.33 s, primary keys 1.59 s)
```
*Было 24.73 с → стало 6.03 с*

**1 клиент:**
```
progress: 1.0 s, 1802.9 tps, lat 0.552 ms stddev 0.112, 0 failed
progress: 2.0 s, 1604.0 tps, lat 0.623 ms stddev 0.120, 0 failed
progress: 3.0 s, 1796.0 tps, lat 0.556 ms stddev 0.106, 0 failed
progress: 4.0 s, 1762.9 tps, lat 0.567 ms stddev 0.109, 0 failed
progress: 5.0 s, 1470.0 tps, lat 0.680 ms stddev 0.092, 0 failed
progress: 6.0 s, 1509.1 tps, lat 0.662 ms stddev 0.091, 0 failed
progress: 7.0 s, 1549.0 tps, lat 0.646 ms stddev 0.105, 0 failed
progress: 8.0 s, 2007.9 tps, lat 0.498 ms stddev 0.070, 0 failed
progress: 9.0 s, 470.6 tps, lat 0.480 ms stddev 0.029, 0 failed
progress: 10.0 s, 403.3 tps, lat 4.400 ms stddev 77.744, 0 failed

tps = 1438.209558 (without initial connection time)
latency average = 0.695 ms
```

**10 клиентов:**
```
progress: 5.0 s, 4675.6 tps, lat 2.129 ms stddev 0.653, 0 failed
progress: 10.0 s, 4821.4 tps, lat 2.072 ms stddev 0.650, 0 failed
progress: 15.0 s, 4773.2 tps, lat 2.093 ms stddev 0.639, 0 failed
progress: 20.0 s, 4816.5 tps, lat 2.073 ms stddev 0.637, 0 failed
progress: 25.0 s, 4820.1 tps, lat 2.072 ms stddev 0.655, 0 failed
progress: 30.0 s, 4798.5 tps, lat 2.081 ms stddev 0.645, 0 failed
progress: 35.0 s, 4900.7 tps, lat 2.038 ms stddev 0.624, 0 failed
progress: 40.0 s, 4821.4 tps, lat 2.072 ms stddev 0.668, 0 failed
progress: 45.0 s, 4847.3 tps, lat 2.060 ms stddev 0.652, 0 failed
progress: 50.0 s, 4806.4 tps, lat 2.078 ms stddev 0.664, 0 failed
progress: 55.0 s, 4840.8 tps, lat 2.063 ms stddev 0.690, 0 failed
progress: 60.0 s, 4834.6 tps, lat 2.066 ms stddev 0.667, 0 failed

tps = 4813.850833 (without initial connection time)
latency average = 2.075 ms
```

**SELECT only:**
```
tps = 32383.714645 (without initial connection time)
latency average = 0.307 ms
```

### 4.3 Итог максимальной производительности

| Тест | Baseline | Optimized | Max Perf | Прирост vs Baseline |
|------|----------|-----------|----------|---------------------|
| 1 клиент | 204 TPS | 268 TPS | **1 438 TPS** | **+605%** |
| 10 клиентов | 1 097 TPS | 1 544 TPS | **4 814 TPS** | **+339%** |
| SELECT only | 27 282 TPS | 32 893 TPS | 32 384 TPS | +19% |

**Наблюдения:**
- SELECT практически не изменился — `fsync=off` влияет только на запись
- Производительность стабильная (stddev ~0.65 вместо ~15)
- Latency снизилась с 9.1 ms до 2.1 ms (в 4.4 раза)

---

## Сравнение результатов

### Таблица результатов

| Конфигурация | 1 клиент | 10 клиентов | SELECT only | Init time |
|--------------|----------|-------------|-------------|-----------|
| Baseline  | 204 TPS | 1 097 TPS | 27 282 TPS | 24.73 s |
| Optimized | 268 TPS | 1 544 TPS | 32 893 TPS | 20.25 s |
| Max performance | **1 438 TPS** | **4 814 TPS** | 32 384 TPS | **6.03 s** |


### Изменённые параметры

| Параметр | Default | Optimized | Max Perf |
|----------|---------|-----------|----------|
| shared_buffers | 128 MB | 4 GB | 4 GB |
| effective_cache_size | 4 GB | 12 GB | 12 GB |
| work_mem | 4 MB | 64 MB | 64 MB |
| maintenance_work_mem | 64 MB | 512 MB | 512 MB |
| random_page_cost | 4.0 | 1.1 | 1.1 |
| effective_io_concurrency | 1 | 200 | 200 |
| max_wal_size | 1 GB | 4 GB | 16 GB |
| synchronous_commit | on | on | **off** |
| fsync | on | on | **off** |
| full_page_writes | on | on | **off** |

---

## Выводы

1. **Дефолтные настройки PostgreSQL консервативны** — рассчитаны на минимальное потребление ресурсов (shared_buffers = 128 MB при 16 GB RAM). Для выделенного сервера требуется тюнинг.

2. **Безопасная оптимизация даёт +41%** на записи. Основной вклад:
   - `shared_buffers = 25% RAM` — данные кэшируются в памяти
   - `max_wal_size = 4GB` — реже checkpoint, меньше провалов
   - `random_page_cost = 1.1` — планировщик учитывает скорость SSD

3. **`synchronous_commit = off`** — значительный прирост при минимальных рисках.

4. **`fsync = off` даёт большой прирост**, но ценой полной потери durability.

5. **SELECT не зависит от настроек записи** — прирост всего +19% за счёт оптимизации памяти.
