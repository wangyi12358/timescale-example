# TimescaleDB Example

English | [简体中文](./README.zh-CN.md)

This example demonstrates how to use the TimescaleDB extension to manage time-series data. TimescaleDB is an open-source PostgreSQL extension designed for handling large-scale time-series data.

## TimescaleDB Features

* **Automatic Partitioning**: Automatically partitions data for improved query performance.
* **Continuous Aggregation**: Supports continuous aggregate queries to reduce data volume.
* **Space Partitioning**: Supports partitioning based on time and other columns.
* **SQL Compatibility**: Compatible with standard SQL queries.
* **Automatic Compression**: Automatically compresses historical data to save storage space.

## Using TimescaleDB Automatic Partitioning

### 1. Enable the TimescaleDB Extension

Ensure that the TimescaleDB extension is enabled in the database. You can enable it with the following SQL command:

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

This step only needs to be executed once after installing TimescaleDB.

### 2. Create a Regular Table

```sql
CREATE TABLE sensor_data (
  time        TIMESTAMPTZ       NOT NULL,
  sensor_id   INT               NOT NULL,
  temperature DOUBLE PRECISION  NULL,
  humidity    DOUBLE PRECISION  NULL
);
```

### 3. Convert the Table into a Hypertable

To use TimescaleDB features, the table needs to be converted into a hypertable. A hypertable is a special type of table used for partitioning and managing time-series data. You can use the create_hypertable function to convert an existing table:

```
SELECT create_hypertable('sensor_data', 'time', chunk_time_interval => interval '1 hour');
```

* ‘sensor_data’ is the name of the table to be converted.
* ‘time’ is the time column you choose, which will be used for partitioning by TimescaleDB.
* chunk_time_interval specifies the time interval for partitioning. In this example, the data is partitioned hourly.

### 4. (Optional) Specify Space Partitioning

If your dataset is large, you can partition by columns in addition to time-based partitioning. For example, partition by sensor_id:

```
SELECT create_hypertable('sensor_data', 'time', 'sensor_id');
```

This will partition data by both time and sensor_id, further optimizing query performance.

### 5. Inserting Data

After that, you can insert data into the sensor_data table as usual. TimescaleDB will automatically manage partitioning and storage optimization.

```sql
INSERT INTO sensor_data (time, sensor_id, temperature, humidity)
VALUES (NOW(), 1, 22.5, 60.1);
```

### 6. View Hypertable and Chunk Information

You can view the existing hypertables in the database using the following command:

```sql
SELECT * FROM timescaledb_information.hypertables;
```

You can also view detailed information about chunks (partitions):

```sql
SELECT * FROM timescaledb_information.chunks;
```

## Using TimescaleDB Automatic Compression (Not Enabled by Default)

TimescaleDB’s automatic compression feature can help save storage space and improve query efficiency, especially when dealing with large-scale time-series data. By default, TimescaleDB does not enable data compression automatically, so you need to configure and set up compression manually.

### 1. Enable Compression

You need to enable compression for a hypertable and define a compression policy. Compression is typically applied to historical data because it can slow down writes slightly, but it improves query and storage efficiency.
The command to enable compression is:

```sql
ALTER TABLE sensor_data SET (timescaledb.compress);
```

### 2. Define When to Automatically Compress Data

Usually, you want to compress historical data, i.e., data that is not recent. You can define when to trigger compression using add_compression_policy.
For example, set the policy to compress data after 7 days:

```sql
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');
```

### 3. Define Specific Columns to Compress

You can specify which columns to compress, and TimescaleDB will use different compression algorithms depending on the column’s data type. You can set compression for columns like this:

```sql
ALTER TABLE sensor_data
ALTER COLUMN temperature SET COMPRESSION DELTA,  -- 适合数值型列
ALTER COLUMN humidity SET COMPRESSION GORILLA;   -- 适合浮点数或时间序列数据
```

TimescaleDB offers several compression algorithms optimized for different types of data (e.g., numeric, timestamp, etc.).

### 4. Default Compression Behavior of TimescaleDB

By default, TimescaleDB does not enable compression automatically. You need to manually enable a compression policy and specify the conditions for triggering compression (e.g., compressing historical data). Without compression enabled, all data remains uncompressed.

Once compression is enabled:

* Data writes are still stored in non-compressed chunks.
* When the defined time interval is reached (e.g., 7 days), data is automatically compressed into a smaller storage format.

### 5. Monitoring Compression Status

You can check which chunks have been compressed by running the following query:

```sql
SELECT chunk_name, compression_status
FROM timescaledb_information.chunks
WHERE hypertable_name = 'sensor_data';
```

This will show you which chunks have been compressed and their compression status.

### 6. Summary

1. By default, TimescaleDB does not enable compression automatically; you need to configure it manually.
2. Use ALTER TABLE to enable compression for a hypertable, and then set an automatic compression policy using add_compression_policy.
3. You can specify which columns to compress and select an appropriate compression algorithm.
4. Compression policies typically apply to historical data, and you can set the compression timing according to your needs.

By setting up an automatic compression policy properly, TimescaleDB can greatly reduce storage space and improve query efficiency, especially in time-series or IoT data applications.

## Reference Links

* [Install TimescaleDB](https://docs.timescale.com/self-hosted/latest/install/)
