# High-Frequency Monitoring Best Practices for NS-3 Simulations

## Executive Summary

Modern monitoring replaces RRD-style databases with:
- **Multi-tier scraping**: 1s, 5s, 15s intervals based on metric criticality
- **Push Gateway**: Sub-second metrics (100ms-1s) for critical events
- **Recording Rules**: Pre-aggregation at ingestion time
- **Remote Storage**: Automatic downsampling for long-term retention
- **Streaming Aggregation**: Real-time rollups without data loss

## 1. Optimal Sampling Rate Strategy

### 1.1 Tiered Scrape Configuration

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s  # Default for low-frequency metrics
  evaluation_interval: 5s  # Recording rules evaluation
  
  # Scrape timeout must be less than interval
  scrape_timeout: 10s
  
  # External labels for federation
  external_labels:
    cluster: 'ns3-hpc'
    region: 'us-east-1'

# Different scrape intervals for different metric types
scrape_configs:
  # Critical NS-3 metrics - 1 second interval
  - job_name: 'ns3_critical'
    scrape_interval: 1s
    scrape_timeout: 800ms
    static_configs:
      - targets: ['localhost:9101']
    metric_relabel_configs:
      # Only keep critical metrics at 1s resolution
      - source_labels: [__name__]
        regex: 'ns3_(simulation_time|events_processed|mpi_sync_time|buffer_drops).*'
        action: keep
  
  # Standard NS-3 metrics - 5 second interval  
  - job_name: 'ns3_standard'
    scrape_interval: 5s
    scrape_timeout: 4s
    static_configs:
      - targets: ['localhost:9101']
    metric_relabel_configs:
      # Exclude critical metrics (collected at 1s)
      - source_labels: [__name__]
        regex: 'ns3_(simulation_time|events_processed|mpi_sync_time|buffer_drops).*'
        action: drop
      # Keep all other ns3_ metrics
      - source_labels: [__name__]
        regex: 'ns3_.*'
        action: keep
        
  # System metrics - 15 second interval
  - job_name: 'node'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9100']
      
  # AWS/Cloud metrics - 60 second interval
  - job_name: 'aws'
    scrape_interval: 60s
    ec2_sd_configs:
      - region: us-east-1
        port: 9100
```

### 1.2 Push Gateway for Sub-Second Metrics

```yaml
# For events that need < 1s resolution
- job_name: 'pushgateway'
  scrape_interval: 1s
  honor_labels: true
  static_configs:
    - targets: ['localhost:9091']
```

```cpp
// NS-3 code for sub-second metrics
class HighFrequencyMetrics
{
private:
    // Batch metrics to reduce overhead
    struct MetricBatch {
        std::vector<std::pair<double, double>> samples;
        std::chrono::high_resolution_clock::time_point start;
    };
    
    std::unordered_map<std::string, MetricBatch> m_batches;
    const size_t BATCH_SIZE = 100;
    const auto BATCH_TIMEOUT = std::chrono::milliseconds(100);
    
public:
    void RecordHighFrequencyEvent(const std::string& metric, double value)
    {
        auto& batch = m_batches[metric];
        auto now = std::chrono::high_resolution_clock::now();
        
        batch.samples.push_back({
            Simulator::Now().GetSeconds(),
            value
        });
        
        // Push when batch is full or timeout
        if (batch.samples.size() >= BATCH_SIZE ||
            now - batch.start > BATCH_TIMEOUT) {
            PushBatch(metric, batch);
            batch.samples.clear();
            batch.start = now;
        }
    }
    
    void PushBatch(const std::string& metric, const MetricBatch& batch)
    {
        // Use libcurl to push to gateway
        std::stringstream payload;
        
        // Prometheus exposition format with timestamps
        for (const auto& [timestamp, value] : batch.samples) {
            payload << metric << " " << value << " " 
                   << static_cast<int64_t>(timestamp * 1000) << "\n";
        }
        
        // Async push to avoid blocking simulation
        AsyncPushToGateway(payload.str());
    }
};
```

## 2. Recording Rules for Pre-Aggregation

### 2.1 Recording Rules Configuration

```yaml
# /etc/prometheus/recording_rules.yml
groups:
  - name: ns3_aggregations
    interval: 5s  # How often to evaluate rules
    rules:
      # 5-second rate calculations
      - record: ns3:events_per_second
        expr: rate(ns3_events_processed[5s])
        
      - record: ns3:packets_per_second
        expr: rate(ns3_packets_transmitted_total[5s])
        
      # 1-minute moving averages
      - record: ns3:events_per_second:avg1m
        expr: avg_over_time(ns3:events_per_second[1m])
        
      # 5-minute aggregations
      - record: ns3:simulation_efficiency:5m
        expr: |
          increase(ns3_simulation_time_seconds[5m]) / 
          (5 * 60)
          
      # Per-rank aggregations
      - record: ns3:mpi_efficiency_by_rank:5m
        expr: |
          1 - (
            sum by (rank) (rate(ns3_mpi_sync_time_seconds_sum[5m])) /
            sum by (rank) (rate(ns3_mpi_compute_time_seconds_sum[5m]))
          )
          
      # Hierarchical aggregations
      - record: ns3:packets_by_traffic_class:rate5m
        expr: |
          sum by (traffic_class) (
            rate(ns3_packets_transmitted_total[5m])
          )
```

### 2.2 Downsampling Rules for Long-Term Storage

```yaml
  - name: ns3_downsampling
    interval: 1m
    rules:
      # 1-minute aggregations for week-old data
      - record: ns3:simulation_time:1m
        expr: avg_over_time(ns3_simulation_time_seconds[1m])
        
      # 5-minute aggregations for month-old data
      - record: ns3:events_processed:5m
        expr: increase(ns3_events_processed[5m])
        
      # 1-hour aggregations for year-old data
      - record: ns3:simulation_time:1h
        expr: avg_over_time(ns3:simulation_time:1m[1h])
```

## 3. Modern Storage Architecture for Multi-Day Simulations

### 3.1 VictoriaMetrics for Long-Term Storage

```yaml
# docker-compose.yml for VictoriaMetrics
version: '3.8'

services:
  # VictoriaMetrics single-node (or cluster for larger deployments)
  victoriametrics:
    image: victoriametrics/victoria-metrics:latest
    ports:
      - "8428:8428"
    volumes:
      - vmdata:/storage
    command:
      - '-storageDataPath=/storage'
      - '-retentionPeriod=2y'  # 2 year maximum retention
      - '-dedup.minScrapeInterval=1s'
      - '-search.maxPointsPerTimeseries=10000000'  # Support long simulations
      - '-search.maxSamplesPerQuery=1000000000'
      
  # Prometheus with remote write
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - promdata:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=2d'  # Local buffer
      - '--storage.tsdb.min-block-duration=2h'
      - '--storage.tsdb.max-block-duration=1d'
      
  # vmagent for streaming aggregation
  vmagent:
    image: victoriametrics/vmagent:latest
    ports:
      - "8429:8429"
    volumes:
      - ./vmagent.yml:/etc/vmagent/vmagent.yml
    command:
      - '-promscrape.config=/etc/vmagent/vmagent.yml'
      - '-remoteWrite.url=http://victoriametrics:8428/api/v1/write'
      - '-remoteWrite.maxDiskUsagePerURL=100GB'  # Large buffer for multi-day sims

volumes:
  vmdata:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/nvme/victoria-metrics  # Use fast NVMe storage
  promdata:
```

### 3.2 Tiered Retention Policy for Long Simulations

```yaml
# prometheus.yml addition
remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"
    queue_config:
      capacity: 1000000  # Large queue for burst handling
      max_shards: 50
      min_shards: 10
      max_samples_per_send: 10000
      batch_send_deadline: 5s
      min_backoff: 30ms
      max_backoff: 1s
    
    # Retention policies based on metric type
    write_relabel_configs:
      # Critical metrics (1s resolution) - keep raw for 1 week
      - source_labels: [__name__]
        regex: 'ns3_(simulation_time|events_processed|mpi_sync_time|buffer_drops|packet_latency).*'
        target_label: __retention__
        replacement: '168h'  # 1 week
        
      # Standard metrics (5s resolution) - keep raw for 1 week
      - source_labels: [__name__]
        regex: 'ns3_(packets_.*|queue_depth|link_utilization).*'
        target_label: __retention__
        replacement: '168h'  # 1 week
        
      # System metrics (15s resolution) - keep raw for 2 weeks
      - source_labels: [__name__]
        regex: 'node_.*'
        target_label: __retention__
        replacement: '336h'  # 2 weeks
        
      # MPI communication matrix - keep raw for 3 days (high cardinality)
      - source_labels: [__name__]
        regex: 'mpi_(messages|bytes)_sent_total'
        target_label: __retention__
        replacement: '72h'  # 3 days
```

## 4. Storage Requirements for r7a.48xlarge Nodes

### 4.1 Metrics Volume Estimation

```bash
# r7a.48xlarge specifications:
# - 192 vCPUs (can run ~180 MPI ranks efficiently)
# - 1536 GB RAM
# - 10-100K nodes per rank typical for large simulations

# Metric categories and estimated volume:

# 1. NS-3 Core Metrics (per rank)
# - simulation_time, events_processed: 2 metrics
# - event_queue_size, memory_usage: 2 metrics
# - Total: ~4 metrics/rank @ 1s = 4 samples/sec

# 2. Per-Node Metrics (assuming 50K nodes/rank average)
# - packets_sent/received/dropped: 3 metrics/node
# - queue_depth per interface: ~2 metrics/node
# - With cardinality reduction: ~100 aggregate metrics/rank @ 5s

# 3. MPI Metrics
# - rank-to-rank communication matrix: 180*180 = 32,400 potential pairs
# - With sparse communication: ~1,000 active pairs
# - 2 metrics/pair (messages, bytes) @ 5s = 400 samples/sec

# 4. System Metrics
# - CPU, memory, network, disk: ~200 metrics @ 15s = 13 samples/sec

# Total estimated samples per r7a.48xlarge per second:
# Core: 180 ranks * 4 metrics * 1 Hz = 720 samples/sec
# Nodes: 180 ranks * 100 metrics * 0.2 Hz = 3,600 samples/sec  
# MPI: 1,000 pairs * 2 metrics * 0.2 Hz = 400 samples/sec
# System: 200 metrics * 0.067 Hz = 13 samples/sec
# TOTAL: ~4,733 samples/sec

# Storage calculation (with compression):
# Prometheus/VictoriaMetrics typically achieve 1-2 bytes/sample with compression
# Conservative estimate: 1.5 bytes/sample
# 4,733 samples/sec * 1.5 bytes = 7,100 bytes/sec = 7.1 KB/sec

# Per hour: 7.1 KB/sec * 3600 sec = 25.6 MB/hour
# Per day: 25.6 MB/hour * 24 hours = 614 MB/day
# Per week (raw): 614 MB/day * 7 days = 4.3 GB/week per node
```

### 4.2 Storage Planning Table

| Simulation Duration | Raw Storage (1 week retention) | Downsampled (1 year) | Total per Node |
|--------------------|--------------------------------|----------------------|----------------|
| 1 hour | 26 MB | 5 MB | 31 MB |
| 6 hours | 154 MB | 30 MB | 184 MB |
| 24 hours (1 day) | 614 MB | 120 MB | 734 MB |
| 72 hours (3 days) | 1.8 GB | 360 MB | 2.2 GB |
| 168 hours (1 week) | 4.3 GB | 840 MB | 5.1 GB |

### 4.3 Multi-Node Cluster Storage

```bash
# For a typical HPC cluster running NS-3:
# 10 x r7a.48xlarge nodes = 1,800 MPI ranks total

# Storage requirements:
# Per simulation day: 10 nodes * 614 MB = 6.1 GB/day
# 1 week raw retention: 10 nodes * 4.3 GB = 43 GB
# 1 year downsampled: 10 nodes * 840 MB = 8.4 GB

# Recommended storage allocation:
# - Fast NVMe for recent data: 100 GB (2+ weeks raw data)
# - Standard SSD for older data: 500 GB (2+ years historical)
```

## 5. Optimized Configuration for Multi-Day Simulations

### 5.1 Adaptive Sampling for Long Runs

```cpp
class LongRunningSimulationMetrics
{
private:
    // Reduce metric frequency as simulation progresses
    struct AdaptiveSchedule {
        double sim_hours;
        std::chrono::seconds interval;
    };
    
    std::vector<AdaptiveSchedule> m_schedule = {
        {0.0, 1s},      // First hour: 1s resolution
        {1.0, 5s},      // Hours 1-6: 5s resolution  
        {6.0, 30s},     // Hours 6-24: 30s resolution
        {24.0, 60s},    // Days 2-7: 1m resolution
        {168.0, 300s}   // Week 2+: 5m resolution
    };
    
public:
    std::chrono::seconds GetCurrentInterval()
    {
        double hours = Simulator::Now().GetHours();
        
        for (auto it = m_schedule.rbegin(); it != m_schedule.rend(); ++it) {
            if (hours >= it->sim_hours) {
                return it->interval;
            }
        }
        
        return m_schedule[0].interval;
    }
};
```

### 5.2 Checkpoint-Based Metric Persistence

```cpp
class MetricCheckpointer
{
public:
    void CreateCheckpoint()
    {
        // Flush all pending metrics before checkpoint
        PrometheusExporter::Get()->FlushAll();
        
        // Create metric snapshot
        std::string checkpoint_id = GenerateCheckpointId();
        
        // Tag metrics with checkpoint for easy retrieval
        PrometheusExporter::Get()->SetGlobalLabel(
            "checkpoint_id", checkpoint_id
        );
        
        // Log checkpoint creation
        NS_LOG_INFO("Metrics checkpoint created: " << checkpoint_id);
    }
    
    void RestoreFromCheckpoint(const std::string& checkpoint_id)
    {
        // Query historical metrics from this checkpoint
        auto metrics = QueryMetricsFromCheckpoint(checkpoint_id);
        
        // Restore internal state
        RestoreSimulationState(metrics);
        
        // Continue metrics collection
        PrometheusExporter::Get()->SetGlobalLabel(
            "checkpoint_id", checkpoint_id + "_resumed"
        );
    }
};
```

## 6. Best Practices for Multi-Day Simulations

### 6.1 Retention and Downsampling Strategy

| Data Age | Resolution | Use Case | Storage Impact |
|----------|------------|----------|----------------|
| 0-1 week | Raw (1s/5s) | Debugging, detailed analysis | 100% |
| 1-4 weeks | 1 minute | Performance trends | 10% |
| 1-6 months | 5 minutes | Long-term patterns | 2% |
| 6-24 months | 1 hour | Historical comparison | 0.2% |

### 6.2 Query Optimization for Long Time Ranges

```promql
# Use recording rules for common long-range queries
# Bad: Calculate on demand over 1 week of 1s data
avg_over_time(ns3_simulation_time_seconds[1w])

# Good: Use pre-calculated 1h averages
avg_over_time(ns3:simulation_time:1h[1w])

# For multi-day analysis, use step parameter
# Query 1 week of data with 1-hour resolution
ns3_events_processed[1w:1h]

# Efficient percentile calculation over long ranges
histogram_quantile(0.95, 
  sum(rate(ns3_packet_latency_bucket[1h])) by (le)
)
```

### 6.3 Monitoring Pipeline Health

```yaml
# Alert on metric ingestion issues
groups:
  - name: metric_pipeline
    rules:
      - alert: MetricIngestionDelay
        expr: |
          prometheus_remote_storage_highest_timestamp_in_seconds -
          prometheus_remote_storage_queue_highest_sent_timestamp_seconds > 300
        for: 5m
        annotations:
          summary: "Metrics delayed by {{ $value }}s"
          
      - alert: HighMetricCardinality
        expr: |
          prometheus_tsdb_symbol_table_size_bytes > 1e9
        annotations:
          summary: "Metric cardinality too high"
          
      - alert: StorageSpaceRunningOut
        expr: |
          predict_linear(prometheus_tsdb_storage_blocks_bytes[6h], 7*24*3600) >
          node_filesystem_avail_bytes{mountpoint="/prometheus"}
        annotations:
          summary: "Storage will fill in < 7 days"
```

## 7. Cost-Optimized Storage for AWS

### 7.1 Tiered Storage on AWS

```yaml
# Storage configuration for different data ages
storage_tiers:
  hot:  # 0-2 weeks
    type: "gp3"
    size: "500GB"
    iops: 16000
    throughput: "1000MB/s"
    
  warm:  # 2 weeks - 3 months
    type: "gp3"
    size: "2TB"
    iops: 3000
    throughput: "250MB/s"
    
  cold:  # 3+ months
    type: "sc1"  # Throughput optimized HDD
    size: "10TB"
    min_size: "500GB"  # sc1 minimum
    
  archive:  # 1+ year
    type: "s3"
    storage_class: "GLACIER_IR"  # Instant retrieval
    lifecycle_days: 365
```

### 7.2 Estimated Monthly Costs

```bash
# For 10-node cluster with 1-week continuous simulation:
# Hot storage: 500GB gp3 = $40/month
# Warm storage: 2TB gp3 = $160/month  
# Cold storage: 10TB sc1 = $250/month
# S3 Glacier IR: 1TB = $4/month
# Total: ~$454/month for comprehensive metric retention

# Data transfer (assuming 10% accessed):
# 430GB * $0.09/GB = $38.70/month
# Total with transfer: ~$493/month
```

This configuration ensures that multi-day NS-3 simulations have full-fidelity metrics available for at least a week, with intelligent downsampling for long-term analysis, while keeping storage costs reasonable for HPC-scale deployments.