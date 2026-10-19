# Kafka Metrics — Alert vs Monitoring (Spring Boot 4 / Spring Kafka, observation enabled)

Legend: **🚨 Alert** = page or notify someone · **📊 Monitor** = dashboard and trends only.
Thresholds are starting points; tune them to your own baseline.

---

## Consumer

| Metric | What it tells you | Type & when |
|---|---|---|
| `kafka_consumer_fetch_manager_records_lag_max` | Messages behind the latest offset (worst partition) | 🚨 Lag growing steadily for 5–10 min, or above your SLA in messages or time |
| `spring_kafka_listener_seconds_count{error!="none"}` | Failed listener invocations | 🚨 Error ratio > 1–5% over 5 min, or any error on critical flows |
| `kafka_consumer_coordinator_rebalance_total` | Group rebalances | 🚨 More than a few per hour (rebalance storm). 📊 Otherwise, to correlate with deployments |
| `kafka_consumer_coordinator_failed_rebalance_total` | Rebalances that failed | 🚨 Any increase |
| `kafka_consumer_time_between_poll_max` | Longest gap between two `poll()` calls | 🚨 Above 80% of `max.poll.interval.ms`, because the consumer is about to be evicted |
| `kafka_consumer_last_poll_seconds_ago` | Time since last poll | 🚨 Above `max.poll.interval.ms` (consumer stuck or dead) |
| `spring_kafka_listener_seconds` (p95/p99 from `_bucket`) | Processing time per record or batch | 📊 Trend. 🚨 Only if p99 breaks an SLA or nears the poll interval |
| `spring_kafka_listener_seconds_max` | Slowest recent processing | 📊 Spotting outliers |
| `kafka_consumer_fetch_manager_records_consumed_rate` | Throughput (records/s) | 📊 Capacity view. 🚨 Rate drops to 0 while lag rises |
| `kafka_consumer_coordinator_commit_latency_avg` | Offset commit time | 📊 Broker health signal |
| `kafka_consumer_poll_idle_ratio_avg` | Share of time spent waiting in poll | 📊 Near 0 means saturated (scale out). Near 1 means idle |
| `kafka_consumer_coordinator_assigned_partitions` | Partitions per instance | 📊 Checks balance. 🚨 Value of 0 on an instance that should consume |

---

## Producer

| Metric | What it tells you | Type & when |
|---|---|---|
| `kafka_producer_record_error_total` | Sends that failed for good (after retries) | 🚨 Any increase, because it can mean data loss unless you handle the failure |
| `spring_kafka_template_seconds_count{error!="none"}` | Failed `send()` as seen by your code | 🚨 Error ratio > 0 on critical topics |
| `kafka_producer_record_retry_total` | Retried sends | 📊 Trend. 🚨 Sustained spike (broker or network trouble) |
| `kafka_producer_buffer_available_bytes` | Free memory in the send buffer | 🚨 Below about 10% of `buffer.memory`, because `send()` will soon block |
| `kafka_producer_bufferpool_wait_ratio` | Time threads spend blocked waiting for buffer | 🚨 Sustained > 0 (back-pressure on your app) |
| `kafka_producer_request_latency_avg` / `_max` | Broker round-trip time | 📊 Trend. 🚨 `_max` near `request.timeout.ms` |
| `spring_kafka_template_seconds` (p95/p99) | End-to-end send latency from your code | 📊 Trend. 🚨 If it breaks an SLA |
| `kafka_producer_record_send_rate` | Throughput (records/s) | 📊 Capacity view. 🚨 Falls to 0 when traffic is expected |
| `kafka_producer_batch_size_avg` | Average batch size | 📊 Tuning `linger.ms` and `batch.size` |
| `kafka_producer_records_per_request_avg` | Batching efficiency | 📊 Tuning |
| `kafka_producer_compression_rate_avg` | Compression ratio | 📊 Tuning |
| `kafka_producer_connection_count` | Open broker connections | 🚨 Drops to 0. 📊 Otherwise |

---

## Required configuration

Percentile histograms (needed for the p95/p99 rows):

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        spring.kafka.listener: true
        spring.kafka.template: true
    tags:
      region: ${REGION}   # split every panel per cluster/region
```

## Example queries (PromQL)

```promql
# Listener error ratio
sum by (spring_kafka_listener_id)(rate(spring_kafka_listener_seconds_count{error!="none"}[5m]))
/ sum by (spring_kafka_listener_id)(rate(spring_kafka_listener_seconds_count[5m]))

# p99 processing time
histogram_quantile(0.99, sum by (le, spring_kafka_listener_id)(rate(spring_kafka_listener_seconds_bucket[5m])))

# Lag
max by (client_id)(kafka_consumer_fetch_manager_records_lag_max)
```

> Label names can vary slightly between versions; check `/actuator/prometheus` for the exact names.

---

## Rules of thumb

- **Alert on symptoms.** Lag growing, errors, a stuck consumer and a blocked producer are the core set; everything else is for diagnosing once an alert has fired.
- **Combine signals to avoid noise.** For example, alert on "lag rising AND consumed_rate = 0", not on a lag spike alone during a batch load.
- **Back client-side lag with broker-side lag.** The client metric vanishes when the consumer dies, so the lag alert should also come from kafka-exporter or Burrow.
