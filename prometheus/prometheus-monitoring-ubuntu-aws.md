# Comprehensive Prometheus Monitoring for NS-3 on Ubuntu 24.04 AWS

## Overview

This document provides a complete Prometheus monitoring setup for NS-3 simulations running on Ubuntu 24.04 AWS instances, enabling deep performance analysis, debugging, and troubleshooting of both the simulator and MPI scaling issues.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     AWS EC2 Instances                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Node 1    │  │   Node 2    │  │   Node N    │   ...   │
│  │  NS-3 Sim   │  │  NS-3 Sim   │  │  NS-3 Sim   │         │
│  │  MPI Rank 0 │  │  MPI Rank 1 │  │  MPI Rank N │         │
│  │             │  │             │  │             │         │
│  │ Exporters:  │  │ Exporters:  │  │ Exporters:  │         │
│  │ - Node      │  │ - Node      │  │ - Node      │         │
│  │ - Process   │  │ - Process   │  │ - Process   │         │
│  │ - NVIDIA    │  │ - NVIDIA    │  │ - NVIDIA    │         │
│  │ - Custom    │  │ - Custom    │  │ - Custom    │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                 │                 │                │
│  ┌──────┴─────────────────┴─────────────────┴──────┐        │
│  │            Prometheus Server (Node 0)            │        │
│  │         - Scraping all exporters                 │        │
│  │         - Local time-series storage              │        │
│  │         - Alert manager integration              │        │
│  └──────────────────────┬──────────────────────────┘        │
│                         │                                    │
│  ┌──────────────────────┴──────────────────────────┐        │
│  │                    Grafana                       │        │
│  │         - Real-time dashboards                   │        │
│  │         - NS-3 specific panels                   │        │
│  │         - MPI performance analysis               │        │
│  └──────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

## Installation Script

```bash
#!/bin/bash
# install-monitoring.sh - Complete monitoring setup for Ubuntu 24.04

set -e

# Variables
PROMETHEUS_VERSION="2.47.0"
NODE_EXPORTER_VERSION="1.7.0"
PROCESS_EXPORTER_VERSION="0.7.10"
GRAFANA_VERSION="10.2.0"
LUSTRE_EXPORTER_VERSION="2.1.0"

# Update system
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install -y wget curl software-properties-common apt-transport-https \
    numactl linux-tools-common linux-tools-generic sysstat

# Create monitoring user
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Install Prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz
tar -xzf prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz
sudo cp prometheus-${PROMETHEUS_VERSION}.linux-amd64/{prometheus,promtool} /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp -r prometheus-${PROMETHEUS_VERSION}.linux-amd64/{consoles,console_libraries} /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# Install Node Exporter with extended collectors
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
tar -xzf node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo cp node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --system --no-create-home --shell /bin/false node_exporter

# Install Process Exporter
wget https://github.com/ncabatoff/process-exporter/releases/download/v${PROCESS_EXPORTER_VERSION}/process-exporter-${PROCESS_EXPORTER_VERSION}.linux-amd64.tar.gz
tar -xzf process-exporter-${PROCESS_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo cp process-exporter-${PROCESS_EXPORTER_VERSION}.linux-amd64/process-exporter /usr/local/bin/

# Install Lustre Exporter (if Lustre is used)
if [ -d "/proc/fs/lustre" ]; then
    wget https://github.com/HewlettPackard/lustre_exporter/releases/download/v${LUSTRE_EXPORTER_VERSION}/lustre_exporter-${LUSTRE_EXPORTER_VERSION}.linux-amd64.tar.gz
    tar -xzf lustre_exporter-${LUSTRE_EXPORTER_VERSION}.linux-amd64.tar.gz
    sudo cp lustre_exporter-${LUSTRE_EXPORTER_VERSION}.linux-amd64/lustre_exporter /usr/local/bin/
fi

# Install PCM (Intel Performance Counter Monitor) for memory bandwidth
if lscpu | grep -q "Intel"; then
    git clone https://github.com/opcm/pcm.git
    cd pcm
    make -j$(nproc)
    sudo make install
    cd ..
fi

# Install Grafana
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana

# Install additional exporters
pip3 install prometheus-aws-cost-exporter prometheus-pve-exporter

# Enable additional kernel metrics
echo 1 | sudo tee /proc/sys/vm/zone_reclaim_mode
echo always | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

## Core System Metrics

### 1. Node Exporter Configuration

```yaml
# /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Documentation=https://github.com/prometheus/node_exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter \
  --collector.filesystem.mount-points-exclude="^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)" \
  --collector.filesystem.fs-types-exclude="^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$$" \
  --collector.diskstats.ignored-devices="^(ram|loop|fd|(h|s|v|xv)d[a-z]|nvme\\d+n\\d+p)\\d+$$" \
  --collector.netclass.ignored-devices="^(veth.*|docker.*|cali.*|cilium.*)$$" \
  --collector.netdev.device-exclude="^(veth.*|docker.*|cali.*|cilium.*)$$" \
  --collector.cpu.info \
  --collector.loadavg \
  --collector.meminfo \
  --collector.diskstats \
  --collector.netdev \
  --collector.netstat \
  --collector.sockstat \
  --collector.vmstat \
  --collector.interrupts \
  --collector.ksmd \
  --collector.processes \
  --collector.tcpstat \
  --collector.buddyinfo \
  --collector.arp \
  --collector.ipvs \
  --collector.softnet \
  --collector.pressure \
  --collector.hwmon \
  --collector.edac \
  --collector.rapl \
  --collector.powersupplyclass \
  --collector.schedstat \
  --collector.zfs \
  --collector.nfs \
  --collector.nfsd \
  --collector.buddyinfo.device-include="^(sda|nvme).*" \
  --path.rootfs=/host \
  --collector.systemd \
  --collector.systemd.unit-whitelist="ns3.*|mpi.*" \
  --collector.qdisc \
  --collector.conntrack \
  --collector.filefd \
  --collector.netclass.netlink \
  --collector.nvme \
  --collector.bcache \
  --collector.bonding \
  --collector.infiniband

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 2. Process Exporter for NS-3 Monitoring

```yaml
# /etc/process-exporter/config.yml
process_names:
  - name: "{{.Matches}}"
    cmdline:
    - 'ns3-simulator'
    - 'waf'
    - 'python.*test.py'
    - 'mpirun'
    - 'mpiexec'
    - 'orted'
    
  - name: "{{.Comm}}"
    exe:
    - '/home/ubuntu/ns-3/build/src/.*'
    
  - name: ns3_workers
    cmdline:
    - 'ns-3.*worker.*'

  - name: system_critical
    comm:
    - 'systemd'
    - 'kernel'
    - 'sshd'
```

### 3. Custom NS-3 Metrics Exporter

```python
#!/usr/bin/env python3
# /usr/local/bin/ns3_exporter.py
import time
import psutil
import subprocess
from prometheus_client import start_http_server, Gauge, Counter, Histogram
import json
import re

# NS-3 Simulation Metrics
ns3_simulation_time = Gauge('ns3_simulation_time_seconds', 'Current simulation time')
ns3_wall_clock_time = Gauge('ns3_wall_clock_time_seconds', 'Wall clock time elapsed')
ns3_events_processed = Counter('ns3_events_processed_total', 'Total events processed')
ns3_packets_transmitted = Counter('ns3_packets_transmitted_total', 'Total packets transmitted', ['node_id', 'interface'])
ns3_packets_received = Counter('ns3_packets_received_total', 'Total packets received', ['node_id', 'interface'])
ns3_packets_dropped = Counter('ns3_packets_dropped_total', 'Total packets dropped', ['node_id', 'interface', 'reason'])

# MPI Metrics
mpi_rank_count = Gauge('mpi_rank_count', 'Number of MPI ranks')
mpi_messages_sent = Counter('mpi_messages_sent_total', 'MPI messages sent', ['source_rank', 'dest_rank'])
mpi_bytes_sent = Counter('mpi_bytes_sent_total', 'MPI bytes sent', ['source_rank', 'dest_rank'])
mpi_collective_time = Histogram('mpi_collective_time_seconds', 'Time spent in MPI collective operations', ['operation'])
mpi_barrier_time = Histogram('mpi_barrier_time_seconds', 'Time spent in MPI barriers')
mpi_allreduce_time = Histogram('mpi_allreduce_time_seconds', 'Time spent in MPI allreduce')

# Memory Metrics
ns3_heap_allocated = Gauge('ns3_heap_allocated_bytes', 'Heap memory allocated by NS-3')
ns3_heap_freed = Gauge('ns3_heap_freed_bytes', 'Heap memory freed by NS-3')
ns3_object_count = Gauge('ns3_object_count', 'Number of NS-3 objects', ['type'])
ns3_packet_pool_size = Gauge('ns3_packet_pool_size', 'Size of packet pool')
ns3_event_queue_size = Gauge('ns3_event_queue_size', 'Number of events in queue')

# Buffer/Queue Metrics
buffer_occupancy = Gauge('ns3_buffer_occupancy_bytes', 'Buffer occupancy', ['node_id', 'queue_id', 'traffic_class'])
buffer_drops = Counter('ns3_buffer_drops_total', 'Buffer drops', ['node_id', 'queue_id', 'reason'])
pfc_pause_sent = Counter('ns3_pfc_pause_sent_total', 'PFC pause frames sent', ['node_id', 'priority'])
pfc_pause_received = Counter('ns3_pfc_pause_received_total', 'PFC pause frames received', ['node_id', 'priority'])

# Network Performance Metrics
link_utilization = Gauge('ns3_link_utilization_percent', 'Link utilization', ['link_id'])
link_throughput = Gauge('ns3_link_throughput_bps', 'Link throughput', ['link_id', 'direction'])
link_latency = Histogram('ns3_link_latency_seconds', 'Link latency', ['link_id'])
flow_completion_time = Histogram('ns3_flow_completion_time_seconds', 'Flow completion time', ['flow_type'])

class NS3MetricsCollector:
    def __init__(self, stats_file='/tmp/ns3-stats.json'):
        self.stats_file = stats_file
        
    def collect_metrics(self):
        """Collect metrics from NS-3 stats file"""
        try:
            with open(self.stats_file, 'r') as f:
                stats = json.load(f)
                
            # Update metrics
            ns3_simulation_time.set(stats.get('simulation_time', 0))
            ns3_wall_clock_time.set(stats.get('wall_clock_time', 0))
            ns3_events_processed.inc(stats.get('events_processed', 0))
            
            # Per-node metrics
            for node_stats in stats.get('nodes', []):
                node_id = str(node_stats['id'])
                for iface_stats in node_stats.get('interfaces', []):
                    iface_id = str(iface_stats['id'])
                    ns3_packets_transmitted.labels(node_id=node_id, interface=iface_id).inc(
                        iface_stats.get('tx_packets', 0))
                    ns3_packets_received.labels(node_id=node_id, interface=iface_id).inc(
                        iface_stats.get('rx_packets', 0))
                        
            # MPI metrics
            mpi_stats = stats.get('mpi', {})
            mpi_rank_count.set(mpi_stats.get('rank_count', 0))
            
            # Memory metrics
            mem_stats = stats.get('memory', {})
            ns3_heap_allocated.set(mem_stats.get('heap_allocated', 0))
            ns3_heap_freed.set(mem_stats.get('heap_freed', 0))
            
        except Exception as e:
            print(f"Error collecting metrics: {e}")

    def collect_system_metrics(self):
        """Collect system-level metrics"""
        # Find NS-3 processes
        for proc in psutil.process_iter(['pid', 'name', 'cmdline']):
            try:
                if 'ns3' in ' '.join(proc.info['cmdline'] or []):
                    # Memory info
                    mem_info = proc.memory_info()
                    ns3_process_rss.labels(pid=proc.pid).set(mem_info.rss)
                    ns3_process_vms.labels(pid=proc.pid).set(mem_info.vms)
                    
                    # CPU info
                    cpu_percent = proc.cpu_percent(interval=0.1)
                    ns3_process_cpu.labels(pid=proc.pid).set(cpu_percent)
                    
            except (psutil.NoSuchProcess, psutil.AccessDenied):
                pass

if __name__ == '__main__':
    # Start Prometheus metrics server
    start_http_server(9100)
    
    collector = NS3MetricsCollector()
    
    # Collect metrics every 5 seconds
    while True:
        collector.collect_metrics()
        collector.collect_system_metrics()
        time.sleep(5)
```

## Prometheus Configuration

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'ns3-simulation'
    region: 'us-east-1'

# Alerting
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

# Load rules
rule_files:
  - "alerts/*.yml"

# Scrape configurations
scrape_configs:
  # Node Exporter - System Metrics
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: 'coordinator'
      - targets: ['worker1:9100', 'worker2:9100', 'worker3:9100']
        labels:
          role: 'worker'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):\d+'
        target_label: instance
        replacement: '${1}'

  # Process Exporter
  - job_name: 'process'
    static_configs:
      - targets: ['localhost:9256']

  # NS-3 Custom Metrics
  - job_name: 'ns3'
    static_configs:
      - targets: ['localhost:9101']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'ns3_.*'
        target_label: subsystem
        replacement: 'simulator'

  # AWS EC2 Metadata
  - job_name: 'ec2'
    ec2_sd_configs:
      - region: us-east-1
        port: 9100
        filters:
          - name: tag:Environment
            values: [ns3-simulation]
    relabel_configs:
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id
      - source_labels: [__meta_ec2_instance_type]
        target_label: instance_type
      - source_labels: [__meta_ec2_availability_zone]
        target_label: availability_zone

  # GPU Metrics (if using GPU instances)
  - job_name: 'gpu'
    static_configs:
      - targets: ['localhost:9835']
      
  # Lustre Metrics
  - job_name: 'lustre'
    static_configs:
      - targets: ['localhost:9169']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'lustre_.*'
        target_label: subsystem
        replacement: 'storage'
```

## Critical Metrics for NS-3 Performance Analysis

### 1. CPU and Scheduling Metrics

```promql
# CPU utilization per core
rate(node_cpu_seconds_total[5m]) * 100

# Context switches (indicator of MPI communication overhead)
rate(node_context_switches_total[5m])

# CPU steal time (AWS virtualization overhead)
rate(node_cpu_seconds_total{mode="steal"}[5m]) * 100

# Process CPU usage
rate(process_cpu_seconds_total{name="ns3-simulator"}[5m]) * 100

# Scheduler latency
node_schedstat_running_seconds_total - node_schedstat_waiting_seconds_total
```

### 2. Memory Metrics

```promql
# Memory usage breakdown
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Page faults (critical for large simulations)
rate(node_vmstat_pgfault[5m])

# Memory pressure
rate(node_pressure_memory_stalled_seconds_total[5m])

# Huge pages usage (important for 32TB simulations)
node_memory_HugePages_Total * node_memory_Hugepagesize_bytes

# NS-3 specific memory
ns3_heap_allocated_bytes - ns3_heap_freed_bytes

# Memory bandwidth utilization
rate(node_memory_numa_interleave_hit_total[5m])
```

### 3. Network/MPI Metrics

```promql
# Network throughput
rate(node_network_receive_bytes_total[5m]) * 8
rate(node_network_transmit_bytes_total[5m]) * 8

# Packet loss (critical for MPI)
rate(node_network_transmit_drop_total[5m])

# TCP retransmissions (MPI communication issues)
rate(node_netstat_Tcp_RetransSegs[5m])

# MPI-specific metrics
rate(mpi_messages_sent_total[5m])
rate(mpi_bytes_sent_total[5m])
histogram_quantile(0.99, mpi_collective_time_seconds)

# InfiniBand metrics (if using IB)
rate(node_infiniband_port_data_received_bytes_total[5m])
rate(node_infiniband_port_data_transmitted_bytes_total[5m])
```

### 4. Disk I/O and Storage Metrics

```promql
# Disk throughput
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])

# IOPS
rate(node_disk_reads_completed_total[5m])
rate(node_disk_writes_completed_total[5m])

# I/O latency
rate(node_disk_read_time_seconds_total[5m]) / rate(node_disk_reads_completed_total[5m])
rate(node_disk_write_time_seconds_total[5m]) / rate(node_disk_writes_completed_total[5m])

# Disk queue metrics (critical for performance)
node_disk_io_now                    # Number of I/Os currently in progress
node_disk_io_time_weighted_seconds_total  # Weighted time spent doing I/Os
node_disk_writes_merged_total       # Write merging efficiency
node_disk_reads_merged_total        # Read merging efficiency

# Average queue length
rate(node_disk_io_time_weighted_seconds_total[5m])

# Queue saturation
rate(node_disk_io_time_seconds_total[5m])  # Percentage of time disk was busy

# I/O pressure and stalls
rate(node_pressure_io_stalled_seconds_total[5m])
rate(node_pressure_io_waiting_seconds_total[5m])
node_pressure_io_full               # Full pressure stall information

# Block device stats
node_disk_discard_time_seconds_total  # Time spent discarding
node_disk_flush_requests_total        # Number of flush requests
node_disk_flush_requests_time_seconds_total  # Time spent in flush requests
```

### 5. Lustre Filesystem Metrics

```promql
# Lustre client stats (requires lustre_exporter)
lustre_client_operations_total{operation="read"}   # Read operations
lustre_client_operations_total{operation="write"}  # Write operations
lustre_client_bytes_total{operation="read"}        # Bytes read
lustre_client_bytes_total{operation="write"}       # Bytes written

# Lustre performance
lustre_client_latency_seconds{operation="read", quantile="0.99"}
lustre_client_latency_seconds{operation="write", quantile="0.99"}
lustre_client_rpcs_in_flight        # Outstanding RPCs
lustre_client_available_grant_bytes # Available grant space
lustre_client_dirty_pages_bytes     # Dirty pages in cache

# Lustre MDT (Metadata Target) metrics
lustre_mdt_operations_total{operation="getattr"}
lustre_mdt_operations_total{operation="setattr"}
lustre_mdt_operations_total{operation="open"}
lustre_mdt_operations_total{operation="close"}

# Lustre OST (Object Storage Target) metrics
lustre_ost_used_bytes               # Space used on OSTs
lustre_ost_free_bytes               # Space free on OSTs
lustre_ost_total_bytes              # Total space on OSTs
lustre_ost_used_inodes              # Inodes used
lustre_ost_free_inodes              # Inodes free

# Lustre network metrics
lustre_lnet_send_bytes_total        # LNet bytes sent
lustre_lnet_recv_bytes_total        # LNet bytes received
lustre_lnet_send_messages_total     # LNet messages sent
lustre_lnet_recv_messages_total     # LNet messages received
lustre_lnet_errors_total            # LNet errors by type

# Lustre cache metrics
lustre_page_cache_hits_total        # Page cache hits
lustre_page_cache_misses_total      # Page cache misses
lustre_page_cache_hit_ratio         # Cache hit ratio
```

### 6. Advanced Memory and NUMA Metrics

```promql
# Detailed memory metrics
node_memory_MemTotal_bytes          # Total memory
node_memory_MemFree_bytes           # Free memory
node_memory_MemAvailable_bytes      # Available memory
node_memory_Buffers_bytes           # Buffer cache
node_memory_Cached_bytes            # Page cache
node_memory_SwapCached_bytes        # Swap cache
node_memory_Active_bytes            # Active memory
node_memory_Inactive_bytes          # Inactive memory
node_memory_Active_anon_bytes       # Active anonymous memory
node_memory_Inactive_anon_bytes     # Inactive anonymous memory
node_memory_Active_file_bytes       # Active file-backed memory
node_memory_Inactive_file_bytes     # Inactive file-backed memory
node_memory_Unevictable_bytes       # Unevictable memory
node_memory_Mlocked_bytes           # Locked memory
node_memory_Dirty_bytes             # Dirty pages
node_memory_Writeback_bytes         # Pages being written back
node_memory_AnonPages_bytes         # Anonymous pages
node_memory_Mapped_bytes            # Memory-mapped files
node_memory_Shmem_bytes             # Shared memory
node_memory_Slab_bytes              # Kernel slab memory
node_memory_SReclaimable_bytes      # Reclaimable slab
node_memory_SUnreclaim_bytes        # Unreclaimable slab
node_memory_KernelStack_bytes       # Kernel stack
node_memory_PageTables_bytes        # Page table memory
node_memory_Bounce_bytes            # Bounce buffer memory
node_memory_WritebackTmp_bytes      # Writeback temp memory
node_memory_CommitLimit_bytes       # Commit limit
node_memory_Committed_AS_bytes      # Committed address space
node_memory_VmallocTotal_bytes      # Total vmalloc space
node_memory_VmallocUsed_bytes       # Used vmalloc space
node_memory_VmallocChunk_bytes      # Largest vmalloc chunk
node_memory_AnonHugePages_bytes     # Anonymous huge pages
node_memory_ShmemHugePages_bytes    # Shared memory huge pages
node_memory_ShmemPmdMapped_bytes    # PMD-mapped shared memory
node_memory_HugePages_Total         # Total huge pages
node_memory_HugePages_Free          # Free huge pages
node_memory_HugePages_Rsvd          # Reserved huge pages
node_memory_HugePages_Surp          # Surplus huge pages
node_memory_Hugepagesize_bytes      # Huge page size

# NUMA-specific metrics
node_memory_numa_hit_total          # NUMA hits by node
node_memory_numa_miss_total         # NUMA misses by node
node_memory_numa_foreign_total      # NUMA foreign accesses
node_memory_numa_interleave_total   # NUMA interleave allocations
node_memory_numa_local_total        # NUMA local allocations
node_memory_numa_other_total        # NUMA other node allocations

# Per-NUMA node memory info
node_memory_numa_Active_bytes{node="0"}
node_memory_numa_Inactive_bytes{node="0"}
node_memory_numa_Active_anon_bytes{node="0"}
node_memory_numa_Inactive_anon_bytes{node="0"}
node_memory_numa_Active_file_bytes{node="0"}
node_memory_numa_Inactive_file_bytes{node="0"}
node_memory_numa_Unevictable_bytes{node="0"}
node_memory_numa_Mlocked_bytes{node="0"}
node_memory_numa_Dirty_bytes{node="0"}
node_memory_numa_Writeback_bytes{node="0"}
node_memory_numa_FilePages_bytes{node="0"}
node_memory_numa_Mapped_bytes{node="0"}
node_memory_numa_AnonPages_bytes{node="0"}
node_memory_numa_Shmem_bytes{node="0"}
node_memory_numa_KernelStack_bytes{node="0"}
node_memory_numa_PageTables_bytes{node="0"}
node_memory_numa_Bounce_bytes{node="0"}
node_memory_numa_WritebackTmp_bytes{node="0"}
node_memory_numa_Slab_bytes{node="0"}
node_memory_numa_SReclaimable_bytes{node="0"}
node_memory_numa_SUnreclaim_bytes{node="0"}

# NUMA allocation preferences
node_memory_numa_prefered_node      # Preferred NUMA node
node_memory_numa_numastat_local_node # Local node allocations
node_memory_numa_numastat_other_node # Other node allocations

# Memory bandwidth metrics (requires pcm-exporter)
pcm_memory_bandwidth_read_bytes_per_second{socket="0"}
pcm_memory_bandwidth_write_bytes_per_second{socket="0"}
pcm_memory_bandwidth_total_bytes_per_second{socket="0"}

# Memory controller metrics
node_edac_correctable_errors_total   # ECC correctable errors
node_edac_uncorrectable_errors_total # ECC uncorrectable errors
node_edac_csrow_correctable_errors_total{csrow="0"}
node_edac_csrow_uncorrectable_errors_total{csrow="0"}

# THP (Transparent Huge Pages) metrics
node_vmstat_thp_fault_alloc         # THP fault allocations
node_vmstat_thp_fault_fallback      # THP allocation fallbacks
node_vmstat_thp_collapse_alloc      # THP collapse allocations
node_vmstat_thp_collapse_alloc_failed # Failed THP collapses
node_vmstat_thp_split_page          # THP page splits
node_vmstat_thp_split_page_failed   # Failed THP splits
node_vmstat_thp_zero_page_alloc     # THP zero page allocations
node_vmstat_thp_zero_page_alloc_failed # Failed THP zero allocations

# Memory compaction metrics
node_vmstat_compact_migrate_scanned  # Pages scanned for migration
node_vmstat_compact_free_scanned     # Free pages scanned
node_vmstat_compact_isolated         # Pages isolated for migration
node_vmstat_compact_stall            # Compaction stalls
node_vmstat_compact_fail             # Failed compaction attempts
node_vmstat_compact_success          # Successful compactions

# Page allocation metrics
node_vmstat_pgalloc_dma              # DMA zone allocations
node_vmstat_pgalloc_dma32            # DMA32 zone allocations
node_vmstat_pgalloc_normal           # Normal zone allocations
node_vmstat_pgalloc_movable          # Movable zone allocations
node_vmstat_pgfree                   # Page frees
node_vmstat_pgactivate               # Page activations
node_vmstat_pgdeactivate             # Page deactivations
node_vmstat_pgfault                  # Page faults
node_vmstat_pgmajfault               # Major page faults
node_vmstat_pgrefill                 # Page refills
node_vmstat_pgsteal_kswapd           # Pages stolen by kswapd
node_vmstat_pgsteal_direct           # Direct page steals
node_vmstat_pgscan_kswapd            # Pages scanned by kswapd
node_vmstat_pgscan_direct            # Direct page scans
node_vmstat_pgscan_direct_throttle   # Direct scan throttles
node_vmstat_pginodesteal             # Inode page steals
node_vmstat_slabs_scanned            # Slab objects scanned
node_vmstat_kswapd_inodesteal        # kswapd inode steals
node_vmstat_kswapd_low_wmark_hit_quickly # Quick low watermark hits
node_vmstat_kswapd_high_wmark_hit_quickly # Quick high watermark hits
node_vmstat_pageoutrun               # Pageout runs
node_vmstat_pgrotated                # Pages rotated to LRU tail

# Buddy allocator info
node_buddyinfo_blocks{node="0",zone="Normal",order="0"}  # Free blocks by order
node_buddyinfo_blocks{node="0",zone="Normal",order="1"}
node_buddyinfo_blocks{node="0",zone="Normal",order="2"}
# ... up to order 10

# Memory fragmentation index
node_memory_fragmentation_index{node="0",zone="Normal",order="0"}
```

### 7. NS-3 Simulation Metrics

```promql
# Simulation progress
ns3_simulation_time_seconds / ns3_wall_clock_time_seconds

# Event processing rate
rate(ns3_events_processed_total[5m])

# Packet processing rate
rate(ns3_packets_transmitted_total[5m])
rate(ns3_packets_received_total[5m])

# Buffer utilization
avg(buffer_occupancy_bytes) by (traffic_class)

# PFC activity
rate(pfc_pause_sent_total[5m])

# MPI scaling efficiency
(rate(ns3_events_processed_total[5m]) * mpi_rank_count) / sum(rate(process_cpu_seconds_total[5m]))
```

## Alert Rules

```yaml
# /etc/prometheus/alerts/ns3-alerts.yml
groups:
  - name: ns3_performance
    interval: 30s
    rules:
      - alert: HighCPUSteal
        expr: rate(node_cpu_seconds_total{mode="steal"}[5m]) * 100 > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU steal time on {{ $labels.instance }}"
          description: "CPU steal is {{ $value }}%, indicating AWS hypervisor contention"

      - alert: MPILoadImbalance
        expr: |
          (max(rate(ns3_events_processed_total[5m])) - min(rate(ns3_events_processed_total[5m]))) 
          / avg(rate(ns3_events_processed_total[5m])) > 0.3
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "MPI load imbalance detected"
          description: "Load imbalance is {{ $value }}, indicating poor partitioning"

      - alert: HighMemoryPressure
        expr: rate(node_pressure_memory_stalled_seconds_total[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory pressure on {{ $labels.instance }}"
          description: "Memory stall time is {{ $value }}, system is thrashing"

      - alert: NetworkBottleneck
        expr: |
          rate(node_network_transmit_drop_total[5m]) > 100 or
          rate(node_netstat_Tcp_RetransSegs[5m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Network bottleneck detected on {{ $labels.instance }}"

      - alert: SimulationStalled
        expr: rate(ns3_simulation_time_seconds[5m]) == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "NS-3 simulation appears stalled"
          description: "No simulation progress in 5 minutes"
          
      - alert: HighDiskQueueLength
        expr: rate(node_disk_io_time_weighted_seconds_total[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High disk queue length on {{ $labels.instance }}"
          description: "Average queue length is {{ $value }}, indicating I/O bottleneck"
          
      - alert: DiskSaturation
        expr: rate(node_disk_io_time_seconds_total[5m]) > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk saturated on {{ $labels.instance }}"
          description: "Disk busy {{ $value * 100 }}% of time"
          
      - alert: NUMAImbalance
        expr: |
          (max(node_memory_numa_hit_total) by (node) - min(node_memory_numa_hit_total) by (node)) 
          / avg(node_memory_numa_hit_total) by (node) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "NUMA memory access imbalance"
          description: "NUMA nodes have {{ $value }} imbalance in memory access"
          
      - alert: MemoryFragmentation
        expr: |
          node_buddyinfo_blocks{order="0"} == 0 and 
          sum(node_buddyinfo_blocks{order=~"[1-9]|10"}) > 1000
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Memory fragmentation detected"
          description: "No order-0 blocks available despite higher-order blocks"
          
      - alert: LustreHighLatency
        expr: |
          histogram_quantile(0.99, lustre_client_latency_seconds{operation="read"}) > 0.1 or
          histogram_quantile(0.99, lustre_client_latency_seconds{operation="write"}) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Lustre I/O latency"
          description: "P99 latency is {{ $value }}s for {{ $labels.operation }} operations"
          
      - alert: LustreRPCBacklog
        expr: lustre_client_rpcs_in_flight > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Lustre RPC backlog"
          description: "{{ $value }} RPCs in flight, possible Lustre congestion"
```

## Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "NS-3 MPI Performance Dashboard",
    "panels": [
      {
        "title": "MPI Scaling Efficiency",
        "targets": [
          {
            "expr": "(rate(ns3_events_processed_total[5m]) * mpi_rank_count) / sum(rate(process_cpu_seconds_total[5m]))"
          }
        ]
      },
      {
        "title": "Memory Usage by Component",
        "targets": [
          {
            "expr": "ns3_heap_allocated_bytes - ns3_heap_freed_bytes",
            "legendFormat": "NS-3 Heap"
          },
          {
            "expr": "node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes",
            "legendFormat": "System Total"
          }
        ]
      },
      {
        "title": "MPI Communication Overhead",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, mpi_collective_time_seconds)",
            "legendFormat": "P99 Collective Time"
          },
          {
            "expr": "rate(mpi_messages_sent_total[5m])",
            "legendFormat": "Message Rate"
          }
        ]
      },
      {
        "title": "Packet Processing Performance",
        "targets": [
          {
            "expr": "rate(ns3_packets_transmitted_total[5m])",
            "legendFormat": "TX Rate"
          },
          {
            "expr": "rate(ns3_packets_dropped_total[5m])",
            "legendFormat": "Drop Rate"
          }
        ]
      }
    ]
  }
}
```

## AWS-Specific Metrics

```yaml
# Additional CloudWatch exporter config
- job_name: 'cloudwatch'
  ec2_sd_configs:
    - region: us-east-1
  relabel_configs:
    - source_labels: [__meta_ec2_instance_id]
      target_label: instance_id
  metric_relabel_configs:
    - source_labels: [__name__]
      regex: 'aws_ec2_.*'
      target_label: subsystem
      replacement: 'infrastructure'
```

### EBS Metrics (Important for large simulations)
```promql
# EBS bandwidth utilization
aws_ec2_ebs_read_bytes / (125 * 1024 * 1024) * 100  # Percentage of 1000 Mbps
aws_ec2_ebs_write_bytes / (125 * 1024 * 1024) * 100

# EBS burst credits (critical for gp3)
aws_ec2_ebs_burst_balance
```

### Network Performance (Enhanced Networking)
```promql
# SR-IOV metrics
node_net_sriov_vf_tx_packets
node_net_sriov_vf_rx_packets

# Placement group metrics
aws_ec2_network_packets_per_second_allowance_utilization
```

## Debugging Queries

### Memory Leak Detection
```promql
# Heap growth rate
deriv(ns3_heap_allocated_bytes[1h]) > 0

# Memory not being freed
(ns3_heap_allocated_bytes - ns3_heap_freed_bytes) / ns3_heap_allocated_bytes > 0.9

# NUMA memory leak detection
rate(node_memory_numa_AnonPages_bytes[1h]) > 0

# Slab memory leak
deriv(node_memory_SUnreclaim_bytes[1h]) > 1048576  # Growing by 1MB/hour
```

### MPI Communication Patterns
```promql
# Communication matrix heatmap
sum(rate(mpi_messages_sent_total[5m])) by (source_rank, dest_rank)

# Identify communication hotspots
topk(10, sum(rate(mpi_bytes_sent_total[5m])) by (source_rank))
```

### Performance Regression Detection
```promql
# Compare current vs baseline performance
(rate(ns3_events_processed_total[5m]) / rate(ns3_events_processed_total[5m] offset 1d)) < 0.9
```

### Disk I/O Analysis
```promql
# Identify I/O bottlenecks
max(rate(node_disk_io_time_weighted_seconds_total[5m])) by (device) > 5

# Check for excessive queue depth
node_disk_io_now > 32

# Disk utilization by device
rate(node_disk_io_time_seconds_total[5m]) * 100

# Read/Write ratio
rate(node_disk_read_bytes_total[5m]) / (rate(node_disk_read_bytes_total[5m]) + rate(node_disk_written_bytes_total[5m]))
```

### NUMA Performance Analysis
```promql
# NUMA miss rate
rate(node_memory_numa_miss_total[5m]) / (rate(node_memory_numa_hit_total[5m]) + rate(node_memory_numa_miss_total[5m]))

# Cross-NUMA traffic
rate(node_memory_numa_foreign_total[5m])

# Memory bandwidth saturation (requires PCM)
pcm_memory_bandwidth_total_bytes_per_second / (272 * 1024 * 1024 * 1024)  # % of theoretical max

# NUMA allocation efficiency
node_memory_numa_local_total / (node_memory_numa_local_total + node_memory_numa_other_total)
```

### Lustre Performance Analysis
```promql
# Lustre cache efficiency
lustre_page_cache_hits_total / (lustre_page_cache_hits_total + lustre_page_cache_misses_total)

# OST imbalance
stddev(lustre_ost_used_bytes) by (ost) / avg(lustre_ost_used_bytes) by (ost)

# Metadata operation latency
histogram_quantile(0.99, rate(lustre_mdt_operations_seconds_bucket[5m])) by (operation)

# Network congestion indicators
rate(lustre_lnet_errors_total[5m]) > 0
```

### Memory Fragmentation Analysis
```promql
# Fragmentation score (lower is better)
sum(node_buddyinfo_blocks{order="0"}) / sum(node_buddyinfo_blocks)

# High-order allocation pressure
rate(node_vmstat_compact_stall[5m]) > 0

# THP allocation success rate
node_vmstat_thp_fault_alloc / (node_vmstat_thp_fault_alloc + node_vmstat_thp_fault_fallback)
```

## Usage Example

```bash
# Start monitoring stack
sudo systemctl start prometheus node_exporter process-exporter grafana-server

# Run NS-3 with metrics export
NS3_METRICS_FILE=/tmp/ns3-stats.json ./waf --run "my-simulation --enable-metrics"

# Access dashboards
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000 (admin/admin)

# Query examples
curl http://localhost:9090/api/v1/query?query=ns3_simulation_time_seconds
```

This comprehensive monitoring setup enables deep insights into NS-3 performance, MPI scaling issues, and AWS infrastructure utilization, facilitating optimization of large-scale network simulations.