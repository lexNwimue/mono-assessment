# Mono Assessment

## Problem

In a banking application that integrates with multiple external banks, each outgoing transaction to a third-party bank returns a status code indicating success or failure. For operational visibility and decision support (confidence indicators), we need to track and display the **success rate** of transactions to each destination bank. We assume there is an existing transactions table/collection & transactions are fetched and pushed into the system sequentially, say, via a queue. On that note, the success rate should:

- Reflect recent behavior (e.g., last 15 minutes)
- Be efficiently computed in real-time
- Support long-term trend analysis (e.g., last week, last month)

---

## High-Level Solution

We use a **two-tiered architecture** to handle both real-time and historical needs. These two data can be created concurrently (dual write):

1. **Real-Time Tier (Redis)**  
   - Aggregate transactions in, say, 1-minute buckets using Redis
   - Keys expire after 15 minutes
   - Allows fast calculation of success rate over sliding window

2. **Persistent Tier (Database)**  
   - Periodically (e.g., every 15 minutes) aggregate older transactions into longer-term summaries
   - Store aggregates in a database for historical queries
   - Granularity (e.g., 15-min, hourly, daily) is configurable

---

## Low-Level Overview

### Recording a Transaction

When a transaction is received (via a queue), the system:

1. Calculates the minute-level time bucket:
   ```js
   const bucket = Math.floor(Date.now() / 60000) * 60;
   const redisKey = `success:${destinationBank}:${bucket}`;


2. Increments counters in Redis:
   ```js
   HINCRBY redisKey total 1
   if (statusCode === 200) {
      HINCRBY redisKey success 1
   }
   EXPIRE redisKey 900  // 15 minutes TTL

This ensures Redis only contains the last 15 minutes of metrics, bucketed by bank and minute.


### Computing Recent Success Rate
To compute the success rate over the last 15 minutes:

- Generate all Redis keys for the last 15 one-minute buckets
- For each key, fetch success and total counters
- Sum them to compute the final rate:

   ```js
   const successRate = totalSuccess / totalCount

### Long-Term Aggregation (CRON)
Every 15 minutes, a scheduled job:
- Queries recent transactions from the existing transactions table
- Groups/Batches them by destination bank and time interval (e.g., 15 min, hour, etc.)
- Writes/updates entries in an aggregate_metrics table that might look like:

   ```js
   {
      destination_bank: 'GTBank',
      interval_start: '2025-06-14T12:00:00Z',
      interval_unit: '15min',
      success_count: 135,
      total_count: 142
  }

For faster queries, we might have a Primary Composite Index on `(destination_bank, interval_unit, interval_start)` because you almost always filter by bank, you may filter by interval_unit (e.g., show daily trends vs 15-min chunks), and you often query by time range (interval_start BETWEEN ...). 

After aggregation, older raw transactions can be handled in a number of ways, depding on business requirements, including:

- Streamed to another service (e.g., Kafka → Lambda → S3) that stores the transactions in compressed .csv/.parquet format for analytical or audit purposes
- Archived via another CRON job that:
  - Selects and exports transactions older than N days
  - Uploads the files to cold storage (e.g., S3)
  - Deletes the exported data from the database

### Possible Optimizations

- For Redis (Real-Time Layer):
  -  Pipeline writes to reduce Redis round-trips
  -  Sharded Redis (horizontal scaling) to support very high traffic
  -  Smaller keys (e.g., short bank codes) to reduce memory usage
  -  Dynamic TTLs based on traffic to avoid unnecessary retention

- For Database (Persistence Layer)
  -  Batched inserts/updates for better DB write throughput
  -  Indexing on (bank, interval_start) to speed up aggregations
  -  Partitioning large tables by time (e.g., by month) to keep queries fast

- For Queries
  -  Materialized views for fixed window analytics (e.g., daily bank performance)
 
- #### Stream Processing (Optional or Alternative)
  Use a stream processor (e.g., Apache Flink, Kafka Streams, Apache Beam) to:
- Consume transactions as they arrive
- Compute success metrics in near-real-time
- Emit aggregate results directly into Redis and/or the aggregate DB table

This would considerably lower latency and sliding windows would be more accurate, eliminating the need for batch CRON jobs since windows are continuously computed. 
Although it introduces added setup complexity and operational costs

### Assumptions
- All Redis operations are atomic and idempotent
- The aggregation granularity for persistent metrics is configurable
- A successful transaction is identified using a known status code (e.g., 200)
- Duplicate transactions are not possible
