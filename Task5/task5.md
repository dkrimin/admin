### Этап 1: Установка ClickHouse на локальной машине

1. **Устанавливаем ClickHouse:**
   ```bash
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates
   sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv E0C56BD4
   echo "deb https://repo.clickhouse.com/deb/stable/ main/" | sudo tee /etc/apt/sources.list.d/clickhouse.list
   sudo apt-get update
   sudo apt-get install -y clickhouse-server clickhouse-client
   ```

2. **Запускаем ClickHouse-сервер:**
   ```bash
   sudo service clickhouse-server start
   ```

---

### Этап 2: Запуск ClickHouse в Docker

1. **Создаём контейнер с ClickHouse:**
   ```bash
   docker run -d --name clickhouse_container -p 8123:8123 -v /data/clickhouse:/var/lib/clickhouse yandex/clickhouse-server
   ```

   - **Порт `8123`** позволяет подключаться к ClickHouse через HTTP.
   - **Директория `/data/clickhouse`** используется для сохранения данных.

---

### Этап 3: Конфигурация ClickHouse для репликации

#### 3.1 Запуск ZooKeeper для репликации

ClickHouse использует **ZooKeeper** для управления реплицируемыми таблицами.

1. **Запускаем ZooKeeper-контейнер:**
   ```bash
   docker run -d --name zookeeper -p 2181:2181 confluentinc/cp-zookeeper:latest
   ```

#### 3.2 Настройка ClickHouse для взаимодействия с ZooKeeper

1. **Изменяем файл конфигурации ClickHouse** (обычно находится в `/etc/clickhouse-server/config.xml`):
   ```xml
   <zookeeper>
       <node>
           <host>localhost</host>
           <port>2181</port>
       </node>
   </zookeeper>
   ```

2. **Перезапускаем ClickHouse-сервер:**
   ```bash
   sudo service clickhouse-server restart
   ```

---

### Этап 4: Создание реплицируемых таблиц

#### 4.1 Настройка таблиц на разных узлах

1. **На основном сервере:**
   ```sql
   CREATE TABLE replicated_table (
       id UInt64,
       value String
   ) ENGINE = ReplicatedMergeTree('/clickhouse/tables/replicated_table', 'replica_1')
   ORDER BY id;
   ```

2. **На другом узле или контейнере:**
   ```sql
   CREATE TABLE replicated_table (
       id UInt64,
       value String
   ) ENGINE = ReplicatedMergeTree('/clickhouse/tables/replicated_table', 'replica_2')
   ORDER BY id;
   ```

#### 4.2 Создание общей таблицы для распределения данных

Объединяем данные из всех реплик в одну таблицу:
```sql
CREATE TABLE distributed_table AS replicated_table
ENGINE = Distributed('cluster_name', 'default', 'replicated_table', rand());
```

---

### Этап 5: Проверка репликации

1. **Добавляем данные на одной из реплик:**
   ```sql
   INSERT INTO replicated_table VALUES (1, 'value1'), (2, 'value2');
   ```

2. **Проверяем данные на других репликах:**
   ```sql
   SELECT * FROM replicated_table;
   ```

