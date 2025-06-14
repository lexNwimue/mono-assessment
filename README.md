# Mono Assessment

## Problem

In a banking application that integrates with multiple external banks, each outgoing transaction to a third-party bank returns a status code indicating success or failure. For operational visibility and decision support (confidence indicators), we need to track and display the **success rate** of transactions to each destination bank.

This success rate should:
- Reflect recent behavior (e.g., last 15 minutes)
- Be efficiently computed in real-time
- Support long-term trend analysis (e.g., last week, last month)

---

## High-Level Solution

We use a **two-tiered architecture** to handle both real-time and historical needs:

1. **Real-Time Tier (Redis)**  
   - Aggregate transactions in 1-minute buckets using Redis
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
- Queries recent transactions from persistent storage
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

For faster queries, we might have a Primary Composite Index on `(destination_bank, interval_unit, interval_start)` because you almost always filter by bank, you may filter by interval_unit (e.g., show daily trends vs 15-min chunks), and you often query by time range (interval_start BETWEEN ...)

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

### Assumptions
- 
- Transactions are pushed into the system via a queue
- Redis stores per-minute time buckets and expires each after 15 minutes (or whatever the desired window is)
- All Redis operations are atomic and idempotent
- The aggregation granularity for persistent metrics is configurable
- A successful transaction is identified using a known status code (e.g., 200)
- Duplicate transactions are not possible
