# OpenTelemetry C++/Rust Plan

# Executive Summary

Add OpenTelemetry export to the C++ network simulator to enable real-time telemetry visualization in a system like Grafana or Honecomb (specific system is outside this doc), complementing existing CSV output.

### Critical Constraint

**Telemetry data IS the simulation product.** Losing telemetry data is unacceptable.

If the OTLP backend can't keep up, the system applies **backpressure** to slow the simulation rather than drop data. This ensures every CSV row that would be written is also exported to OTLP—no silent data loss. However, this condition constitutes an operational event; OTLP should never be the simulation bottleneck.

### Approach

- **Standard OpenTelemetry API** exposed to C++ via Rust FFI
- **C++ remains simple:** One-line span creation calls, no error handling
- **Rust handles complexity:** Batching, async export, backpressure, saturation metrics
- **Backend:** The backend is left unspecified intentionally. It may be Grafana, Honeycomb, both or something else.

## Problem Statement

**Current State:** The simulator outputs telemetry via line-by-line CSV writes:

- nd-stats.csv (network device stats)
- tcp-stats.csv (TCP socket state)
- pfc-ecn-stats.csv (PFC pause events)
- ... 6+ CSV files total

**Limitations:**

- Schema drift issues (positional CSV columns, separate schema definitions)
- Requires Python scripts for basic queries
- Hard to correlate across multiple CSV files

**Goal:** Enable real-time querying and visualization while preserving CSV output for offline analysis.

## Architecture

### High-Level Design

```
┌─────────────────────────┐
│   C++ Network Simulator │
│                         │
│  ┌──────────────────┐   │
│  │ Instrumentation  │   │
│  │   Points         │   │
│  └────────┬─────────┘   │
└───────────┼─────────────┘
            │ FFI Calls
            ▼
┌─────────────────────────┐
│   Rust Telemetry Layer  │
│                         │
│  ┌──────────────────┐   │
│  │ OpenTelemetry    │   │
│  │ Client           │   │
│  └────────┬─────────┘   │
└───────────┼─────────────┘
            │
            ▼
    OTLP Exporters
    (gRPC/HTTP)

```

### Component Breakdown

### C++ Side

- **Instrumentation macros/functions** - Lightweight wrappers around FFI calls
- **Span lifecycle management** - Start/end trace spans at key simulation points
- **Attribute collection** - Gather context (packet IDs, node names, event types)

### Rust Side (FFI Library)

- **OpenTelemetry SDK** - Initialize tracer provider, exporters, and processors
- **C-compatible API** - Expose `extern "C"` functions for span creation, events, and attributes
- **Memory safety** - Handle C++ strings and data structures safely across FFI boundary
- **Configuration** - Parse environment variables/config files for backend endpoints

### FFI Boundary

- **Span operations**: `otel_span_start()`, `otel_span_end()`, `otel_span_add_event()`
- **Attributes**: `otel_set_attribute_string()`, `otel_set_attribute_int64()`
- **Context propagation**: `otel_context_attach()`, `otel_context_detach()`

## Current Output System

The network simulator currently outputs telemetry data via **periodic CSV writes**:

### Existing Stats Output

The simulator uses a callback-based tracing system (`ScalaTracedCallbackHandler`) that periodically writes CSV rows:

**Key Output Files:**

- `nd-stats.csv` - Network device TX/RX stats (packets, bytes, drops)
- `tcp-stats.csv` - TCP socket state (cwnd, rtt, retransmissions)
- `pfc-ecn-stats.csv` - PFC pause events, ECN marking
- `net-buff-stats.csv` - Buffer occupancy, enqueue/dequeue rates
- `pcie-stats.csv` - PCIe layer statistics
- `tier-hop-stats.csv` - Multi-tier packet hop tracking

**Current Architecture:**

```
Simulation Events → TracedCallbacks → ScalaTracedCallbackHandler
                                            ↓
                    Periodic Reports (e.g., ReportNetDeviceStats())
                                            ↓
                            ScalaStatFile → CsvWriter → Disk
                                            ↓
                          (CSV row-by-row, flushed with endl)

```

**Report Functions (src/scala-traced-callback-handling/model/scala-traced-callback-handler.cc):**

- `ReportNetDeviceStats()` - Called every `m_ndStatsReportInterval` (lines 1773-1831)
- `ReportTcpStats()` - Called every `m_tcpStackReportInterval` (lines 1833-1839)
- `ReportPFCECNData()` - Called every `m_pfcEcnStatsReportInterval` (lines 2014-2068)
- `ReportNetBuffStats()` - Buffer statistics reporting

### Data Flow Example (Network Device Stats)

For reference, presently the code looks similar to:

```cpp
void ScalaTracedCallbackHandler::ReportNetDeviceStats() {
  for (auto& entry : m_deviceLoggingSources) {
    NDStatsMeta& meta = entry.second;
    // Compute interval stats
    m_ndStatsStrm << now;
    m_ndStatsStrm << meta.nodeId;
    m_ndStatsStrm << meta.txBytes.GetDelta();
    m_ndStatsStrm << meta.rxBytes.GetDelta();
    m_ndStatsStrm << meta.txDrops.GetDelta();
    // ... more columns
    m_ndStatsStrm.EndOfRow();  // Writes CSV row + endl flush
  }
  Simulator::Schedule(m_ndStatsReportInterval, &ScalaTracedCallbackHandler::ReportNetDeviceStats, this);
}

```

## Proposed OpenTelemetry Integration

### Strategy: Dual-Path Output

Instead of replacing CSV output entirely, we'll add a **parallel OpenTelemetry export path**:

```
Simulation Events → TracedCallbacks → ScalaTracedCallbackHandler
                                            ↓
                    Periodic Reports (ReportNetDeviceStats, etc.)
                           ↙                      ↘
                    CSV Writer                FFI → Rust OTel Client
                    (existing)                     (new)
                         ↓                              ↓
                      Disk                         OTLP Exporter
                                                         ↓
                                                  Grafana/Honecomb/etc

```

This de-risks adoption, as both paths are supported, preventing the need for coordinating changes with any and all consumers at once.

### FFI Integration: Standard OpenTelemetry API

We'll expose the standard OpenTelemetry tracing API to C++ via FFI. This is the same pattern used in Python, Go, Java, etc:

**Standard Pattern:** `get_tracer()` → `start_span()` → `set_attribute()` → `end_span()`

### C++ Header (scala-otel.h)

```cpp
#ifndef SCALA_OTEL_H
#define SCALA_OTEL_H

#include <stdint.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef void* OtelTracer;
typedef void* OtelSpan;

// Initialization
bool otel_init(const char* endpoint, const char* service_name);
void otel_shutdown();

// Tracer
OtelTracer otel_get_tracer(const char* name);

// Span lifecycle (standard OTel API)
OtelSpan otel_tracer_start_span(OtelTracer tracer, const char* span_name);
void otel_span_end(OtelSpan span);

// Set attributes (type-specific for FFI)
void otel_span_set_attribute_i64(OtelSpan span, const char* key, int64_t value);
void otel_span_set_attribute_f64(OtelSpan span, const char* key, double value);
void otel_span_set_attribute_string(OtelSpan span, const char* key, const char* value);

#ifdef __cplusplus
}
#endif

#endif // SCALA_OTEL_H

```

### C++ Usage (Standard OTel Pattern)

```cpp
// In ScalaTracedCallbackHandler::DoInitialize()
std::string endpoint = ScalaConfig::GetString("OtelEndpoint", "");
if (!endpoint.empty() && otel_init(endpoint.c_str(), "network-simulator")) {
  m_otelTracer = otel_get_tracer("network-simulator");
  m_otelEnabled = true;
}

// In ReportNetDeviceStats()
void ScalaTracedCallbackHandler::ReportNetDeviceStats() {
  double now = Simulator::Now().GetSeconds();

  for (auto& entry : m_deviceLoggingSources) {
    // Calculate stats (existing code)
    double txTP = ((intervalBytesTx * 8) / ndStatsReportInterval / G);
    double rxTP = ((intervalBytesRx * 8) / ndStatsReportInterval / G);
    // ...

    // Write to CSV (existing)
    m_ndStatsStrm << CurrentISO8601TimeUTC().c_str() << now
                  << entry.second.compName.c_str() << /* ... */;
    m_ndStatsStrm.EndOfRow();

    // NEW: Send to OpenTelemetry (standard API)
    if (m_otelEnabled) {
      OtelSpan span = otel_tracer_start_span(m_otelTracer, "network.device.interval");

      otel_span_set_attribute_f64(span, "sim_time_sec", now);
      otel_span_set_attribute_string(span, "component_name", entry.second.compName.c_str());
      otel_span_set_attribute_string(span, "nd_name", entry.second.ndType.c_str());
      otel_span_set_attribute_string(span, "component_type", entry.second.compType.c_str());
      otel_span_set_attribute_i64(span, "ifid", entry.second.devId);
      otel_span_set_attribute_string(span, "uldl", entry.second.uldl.c_str());
      otel_span_set_attribute_i64(span, "node_id", entry.second.node_id);
      otel_span_set_attribute_string(span, "tier", std::to_string(entry.second.tier).c_str());
      otel_span_set_attribute_i64(span, "total_packets_tx", txCalc->getCount());
      otel_span_set_attribute_i64(span, "total_packets_rx", rxCalc->getCount());
      otel_span_set_attribute_f64(span, "total_bytes_tx_gb", txCalc->getSum() / G);
      otel_span_set_attribute_f64(span, "total_bytes_rx_gb", rxCalc->getSum() / G);
      otel_span_set_attribute_f64(span, "interval_bytes_tx_mb", intervalBytesTx / M);
      otel_span_set_attribute_f64(span, "interval_bytes_rx_mb", intervalBytesRx / M);
      otel_span_set_attribute_f64(span, "tx_rate_gbps", txTP);
      otel_span_set_attribute_f64(span, "rx_rate_gbps", rxTP);
      otel_span_set_attribute_f64(span, "tx_link_util_percent", txLinkUtil);
      otel_span_set_attribute_f64(span, "rx_link_util_percent", rxLinkUtil);
      otel_span_set_attribute_i64(span, "tx_drops", txDrop->getCount());
      otel_span_set_attribute_i64(span, "rx_drops", rxDrop->getCount());

      otel_span_end(span);
    }
  }

  Simulator::Schedule(m_ndStatsReportInterval, &ScalaTracedCallbackHandler::ReportNetDeviceStats, this);
}

```

**Key Design Principles:**

- **Standard API**: Familiar to anyone who's used OpenTelemetry
- **Self-describing**: Attribute names travel with values, no schema drift
- **Type-safe**: Separate functions for int64/float64/string prevent type errors
- **Batching hidden**: Rust worker batches transparently (C++ doesn't need to know)

### Rust Implementation

The Rust FFI layer exposes standard OpenTelemetry APIs and handles all complexity:

- Bounded queue with backpressure
- Background worker thread for batching
- Saturation metrics emitted for alerting
- OTLP gRPC export to Grafana/Honeycomb, etc

**Full implementation details:** See [opentelemetry-rust-implementation.md](https://www.notion.so/OpenTelemetry-Rust-FFI-Implementation-284da38a46628012aa7df72ac4b5f7d5?pvs=21)

### Data Representation

Each CSV row becomes one OpenTelemetry span:

- **Span name**: Indicates stats type (`"network.device.interval"`, `"tcp.stats.interval"`)
- **Span attributes**: All CSV columns as key-value pairs (self-describing, no schema drift)
- **Span time**: Set to simulation time from the CSV row
- **Span duration**: Arbitrary small value

## C++ Usage Example

Adding OpenTelemetry export alongside existing CSV output:

```cpp
// In ReportNetDeviceStats() - existing CSV reporting function
void ScalaTracedCallbackHandler::ReportNetDeviceStats() {
  for (auto& entry : m_deviceLoggingSources) {
    // Compute stats (existing code)
    double txRateGbps = ...;
    double rxRateGbps = ...;

    // Write to CSV (existing)
    m_ndStatsStrm << now << meta.componentName << meta.ndName << ...;
    m_ndStatsStrm.EndOfRow();

    // NEW: Export to OpenTelemetry
    if (m_otelEnabled) {
      OtelSpan span = otel_tracer_start_span(m_tracer, "network.device.interval");

      // Add all 21 CSV columns as attributes
      otel_span_set_attribute_f64(span, "sim_time_sec", now);
      otel_span_set_attribute_string(span, "component_name", meta.componentName.c_str());
      otel_span_set_attribute_string(span, "nd_name", meta.ndName.c_str());
      otel_span_set_attribute_i64(span, "node_id", meta.nodeId);
      otel_span_set_attribute_f64(span, "tx_rate_gbps", txRateGbps);
      otel_span_set_attribute_f64(span, "rx_rate_gbps", rxRateGbps);
      otel_span_set_attribute_i64(span, "tx_drops", meta.txDrops.GetDelta());
      // ... 14 more attributes (one per CSV column)

      otel_span_end(span);  // May block if queue full (backpressure)
    }
  }
}

```

**Result in Grafana:**

- Each CSV row becomes one searchable span
- All 21 columns become span attributes (self-describing, no schema drift)
- Query with TraceQL: `{ node.id = 42 && tx_rate_gbps > 90 }`

## Design Decisions

### 1. Standard OpenTelemetry API

Use industry-standard OpenTelemetry tracing API (not custom protocol):

- Familiar to anyone who's used OTel in Python/Go/Java
- Future-proof: easy migration to native C++ OTel SDK if needed
- Self-describing data: attribute names travel with values (no schema drift)

### 2. Backpressure (Not Load Shedding)

**Decision:** Block simulation if OTLP backend can't keep up.

**Implementation:**

- Bounded queue in Rust
- When full, `otel_span_end()` blocks C++ thread
- Simulation slows down, but **zero data loss**
- Saturation metric (`otel.queue.saturation`) emitted every 10s for alerting

**Rationale:** Telemetry data IS the product—dropping it silently defeats the purpose of running the simulation.

### 3. Spans (Not Metrics)

Each CSV row → one OpenTelemetry span with interval data as attributes.

**Why spans?**

- Works with Tempo/Grafana trace viewer
- TraceQL querying in Grafana Explore
- Can correlate different stat types in same trace
- Metrics API can be added later if needed

### 4. Configuration

- **Runtime toggle:** Only exports if `OtelEndpoint` config is set (zero overhead otherwise)
- **Compile-time:** `-enable-otel` waf flag (enabled if Rust toolchain detected)
- **Sampling:** `OtelSamplingRate` config to export subset of intervals for high-volume sims

## References

- [OpenTelemetry Rust SDK](https://github.com/open-telemetry/opentelemetry-rust)
- [OTLP Specification](https://opentelemetry.io/docs/specs/otlp/)
- [Grafana Tempo Documentation](https://grafana.com/docs/tempo/latest/)
- [TraceQL Query Language](https://grafana.com/docs/tempo/latest/traceql/)
- [Rust FFI Guide](https://doc.rust-lang.org/nomicon/ffi.html)

[OpenTelemetry Rust FFI Implementation](https://www.notion.so/OpenTelemetry-Rust-FFI-Implementation-284da38a46628012aa7df72ac4b5f7d5?pvs=21)