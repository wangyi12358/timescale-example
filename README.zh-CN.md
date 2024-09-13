# Timescale DB Example

[English](./README.md) | 简体中文

这个示例演示了如何使用 TimescaleDB 扩展来管理时序数据。TimescaleDB 是一个开源的 PostgreSQL 扩展，用于处理大规模时序数据。  

## Timescale DB 特征

* **自动分片**：自动将数据分片存储，提高查询性能。
* **连续聚合**：支持连续聚合查询，减少数据量。
* **空间分片**：支持基于时间和其他列的空间分片。
* **SQL 兼容**：支持标准 SQL 查询。
* **自动压缩**：自动压缩历史数据，减少存储空间。

## 使用 Timescale DB 自动分片

### 1. 启用 TimescaleDB 扩展

确保在数据库中启用了 TimescaleDB 扩展。你可以通过以下 SQL 命令启用它：

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

这一步只需要在安装 TimescaleDB 后执行一次。

### 2. 创建一张普通表

```sql
CREATE TABLE sensor_data (
  time        TIMESTAMPTZ       NOT NULL,
  sensor_id   INT               NOT NULL,
  temperature DOUBLE PRECISION  NULL,
  humidity    DOUBLE PRECISION  NULL
);
```

### 3. 将表转换为 Hypertable

为了使用 TimescaleDB 的功能，需要将表转换为 hypertable。Hypertable 是一种特殊的表，用于分片和管理时序数据。使用 create_hypertable 函数将现有表转换为 hypertable：

```
SELECT create_hypertable('sensor_data', 'time', chunk_time_interval => interval '1 hour');
```

* 'sensor_data' 是要转换的表名。
* 'time' 是你选择的时间列，它会被用作 TimescaleDB 的分片依据。
* chunk_time_interval 是分片的时间间隔。在这个例子中，我们将数据按小时分片。

### 4.  (可选) 指定空间分片

如果你的数据量很大，除了基于时间的分片之外，还可以基于其他列进行空间分片。比如，按 sensor_id 进行空间分片：

```
SELECT create_hypertable('sensor_data', 'time', 'sensor_id');
```

这种方式将根据 time 和 sensor_id 进行双层分片，进一步优化查询性能。

### 5. 插入数据

之后，你就可以像往常一样往 sensor_data 表中插入数据了。TimescaleDB 会自动管理分片和存储优化。

```sql
INSERT INTO sensor_data (time, sensor_id, temperature, humidity)
VALUES (NOW(), 1, 22.5, 60.1);
```

### 6. 查看 Hypertable 和 Chunk 信息

你可以通过以下命令来查看数据库中现有的 hypertable：

```sql
SELECT * FROM timescaledb_information.hypertables;
```

你还可以查看分片（chunk）的详细信息：

```sql
SELECT * FROM timescaledb_information.chunks;
```

## 使用 Timescale DB 自动压缩（默认不开启）

TimescaleDB 的自动压缩功能可以帮助节省存储空间并提高查询效率，特别是在处理大规模时序数据时。默认情况下，TimescaleDB 并不会自动启用数据压缩，你需要手动配置和设置压缩策略。

### 1. 启用压缩

你需要为某个 hypertable 启用压缩功能，并定义压缩的策略。压缩通常应用于历史数据，因为它会使写入稍微变慢，但查询和存储效率会提高。
启用压缩的命令如下：

```sql
ALTER TABLE sensor_data SET (timescaledb.compress);
```

### 2. 定义何时自动压缩数据

通常，你希望压缩的是历史数据，即最近的数据不压缩，过了一段时间后才进行压缩。可以通过 add_compression_policy 来定义何时触发压缩。
例如，设置为 7 天后自动压缩数据：

```sql
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');
```

### 3. 定义具体的压缩列

你可以指定需要压缩的列，TimescaleDB 会根据列的数据类型使用不同的压缩算法。压缩列可以通过以下方式设置：

```sql
ALTER TABLE sensor_data
ALTER COLUMN temperature SET COMPRESSION DELTA,  -- 适合数值型列
ALTER COLUMN humidity SET COMPRESSION GORILLA;   -- 适合浮点数或时间序列数据
```

TimescaleDB 提供了多种压缩算法，针对不同类型的数据列（如数值型、时间戳等）进行优化。

### 4. TimescaleDB 默认的压缩行为

默认情况下，TimescaleDB 并不会自动启用压缩。你必须手动启用压缩策略，并指定具体的压缩触发条件（比如对历史数据压缩）。在没有启用压缩之前，所有数据都会保持未压缩状态。

一旦压缩策略被启用：

* 数据写入仍然会被存储在非压缩的 chunk 中。
* 当达到你定义的时间间隔（如 7 天 后），数据将自动压缩为较小的存储格式。

### 5. 监控压缩状态

你可以通过以下查询来查看哪些 chunk 已经被压缩：

```sql
SELECT chunk_name, compression_status
FROM timescaledb_information.chunks
WHERE hypertable_name = 'sensor_data';
```

这将告诉你哪些 chunk 已经被压缩，以及它们的压缩状态。

### 6. 总结

1. 默认情况下，TimescaleDB 不自动启用压缩，你需要手动设置。
2. 使用 ALTER TABLE 为 hypertable 启用压缩，然后通过 add_compression_policy 设置自动压缩的策略。
3. 你可以定义哪些列应该被压缩，并选择合适的压缩算法。
4. 通常压缩策略会应用于 历史数据，可以根据你的需要灵活设置压缩时间。

通过合理设置自动压缩策略，TimescaleDB 能够极大节省存储空间并提高查询效率，特别是对于时序数据或 IoT 数据应用场景。

## 参考链接

* [安装 TimescaleDB](https://docs.timescale.com/self-hosted/latest/install/)
