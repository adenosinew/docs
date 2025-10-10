# DSP v2 initiative

URL: https://scalacomputing.atlassian.net/browse/NS3-4909
Created by: Kelvin Woo
Created time: May 2, 2025 9:56 AM

# Current Pain Points (“Why refactor?”)

| Issue | Impact |
| --- | --- |
| **EOL base image** – Ubuntu 20.04 left standard support in 2025, so we miss security patches and newer kernels | Higher CVE exposure; no modern glibc |
| **Legacy Python 3.7** – lacks of faster CPython optimizations (3.12 is ~10‑15 % quicker on many workloads)   | Slower start‑up, deprecation warnings |
| **Fragmented dependency tree** – hand‑rolled requirements, multiple duplicate libs, and no lock file  | Repro builds are brittle |
| **Stale Dask** – latest releases added a lot of optimizations  ([Changelog - Dask documentation](https://docs.dask.org/en/stable/changelog.html?utm_source=chatgpt.com)) | Inefficient parallel I/O, slow shuffle, less performant dataframe operations |
| **Ad‑hoc process orchestration** | Poor node utilization, harder debug |
| **Five‑minute cold starts** | DSP looks very bad for small simulation |
| **Slow iteration** | Adding new charts take days  |

## Specifications
- Fault tolerance: If the DSP process fails or stuck, the job should be able to recover.
- Telemetry: We should have a way to track the performance of the DSP process.
- Version & Deployment:

## Issues & Bugs
- DSP hangs in the middle of the simulation. We should have a way to recover.
- Mixing patterns, such as client-specific patterns which group multiple charts together. We have a filename-regex pattern pipeline too, which generates all the charts for a specific type of files. Or should we adopt a output-specific pattern? Such as for the RTT charts, they can be generated from Roce data or UET data.
- Five‑minute cold starts. DSP looks very bad for small simulation 
- Efficient utilization of the resources.

## Design Questions:
- Do we need to add DAG abstraction? Between our tasks, there is almost no dependency.
- Do we treat it as a CRON job or web service?
- Do we need a OLAP database?

# How to refactor?
- What the realistic goal for this refactor?



# Goals for *v2* Refactor

## Decouple DSP with NS3 version

When:

- Deprecate <4.0.0
- 4.0.0 official release

Problem:

- NS3 team to provide feature yaml
    - DSP team to define schema
- NS3 software + platform name
- Orchestrator to check model
- RoCE, Coordinator

**Data Model Design**

The data model is an interface between ns3 and DSP that gives ns3 team more flexibility to configure common charts (e.g. line chart). This relaxes the coupling between ns3 and DSP versioning; a minor schema change in ns3 side may no longer requires DSP version upgrade. However, more flexibility comes with more complexity at DSP side; DSP needs to parse the data model and error handling. The idea is to config the JSON in terms of ‘What’ DSP to do, rather than ‘How’ DSP to do; such that DSP can upgrade the ‘How’ internally without breaking the ‘What’. 

### Nd Stats

```json
{
    "name": "nd-stats",
    "regex": ".*nd-stats-[0-9]*$",
    "columns": [
        {
            "name": "sim_time_sec",
            "dtype": "float64"
        },
        {
            "name": "uldl",
            "dtype": "object"
        },
        {
            "name": "tier",
            "dtype": "int32"
        },
        {
            "name": "tx_rate_Gbps",
            "dtype": "float64"
        },
        {
            "name": "rx_rate_Gbps",
            "dtype": "float64"
        },
        {
            "name": "tx_drops",
            "dtype": "int64"
        },
        {
            "name": "rx_drops",
            "dtype": "int64"
        }
    ],
    "groupby": [
        "tier",
        "uldl",
        "sim_time_sec"
    ],
    "agg": {
        "tx_rate_Gbps": [
            "max",
            "min",
            "mean"
        ],
        "rx_rate_Gbps": [
            "max",
            "min",
            "mean"
        ],
        "tx_drops": [
            "sum"
        ],
        "rx_drops": [
            "sum"
        ]
    },
    "plots": [
        {
            "type": "line",
            "key": "nic-rx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) NIC RX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "nic-tx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) NIC TX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-ul-rx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) RSW UL RX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul",
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-ul-tx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) RSW UL TX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": null
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-dl-rx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) RSW DL RX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-dl-tx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) RSW DL TX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "fsw-dl-rx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) FSW DL RX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        1
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "fsw-dl-tx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) FSW DL TX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        1
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "nic-rx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) NIC RX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "nic-tx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) NIC TX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-ul-rx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) RSW UL RX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-ul-tx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) RSW UL TX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-dl-rx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) RSW DL RX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-dl-tx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) RSW DL TX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "fsw-dl-rx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) FSW DL RX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        1
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "fsw-dl-tx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) FSW DL TX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        1
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        }
    ]
}
```

```json
{
    "dsp_tasks": [
        {
            "name": "global-sim-report",
            "regex": ".*global-sim-report-[0-9]*$",
            "plot": [
                {
                    "key": "net-device",
                    "name": "global-network-interface-table",
                    "title": "Network Interface Statistics",
                    "rows": [
                        {
                            "name": "Global",
                            "key": "global"
                        },
                        {
                            "name": "Spine Switch",
                            "key": "ssw-ul"
                        },
                        {
                            "name": "Spine Switch",
                            "key": "ssw-dl"
                        },
                        {
                            "name": "Fabric Switch",
                            "key": "fsw-ul"
                        },
                        {
                            "name": "Fabric Switch",
                            "key": "fsw-dl"
                        },
                        {
                            "name": "Rack Switch",
                            "key": "rsw-ul"
                        },
                        {
                            "name": "Rack Switch",
                            "key": "rsw-dl"
                        },
                        {
                            "name": "NIC",
                            "key": "nic-ul"
                        },
                        {
                            "name": "NIC",
                            "key": "nic-dl"
                        }
                    ],
                    "columns": [
                        {
                            "name": "Packets RX",
                            "key": "packets-rx"
                        },
                        {
                            "name": "Packets TX",
                            "key": "packets-tx"
                        },
                        {
                            "name": "Bytes RX (GB)",
                            "key": "bytes_rx_gb"
                        },
                        {
                            "name": "Bytes TX (GB)",
                            "key": "bytes_tx_gb"
                        },
                        {
                            "name": "Ingress Drops (RX)",
                            "key": "ingress-drops-rx"
                        },
                        {
                            "name": "Egress Drops (TX)",
                            "key": "egress-drops-tx"
                        },
                        {
                            "name": "Direction (UL/DL)",
                            "key": "direction"
                        }
                    ]
                },
                {
                    "key": "pfc",
                    "name": "global-pfc-table",
                    "title": "PFC Statistics",
                    "plot": [
                        {
                            "rows": [
                                {
                                    "name": "Global",
                                    "key": "global"
                                },
                                {
                                    "name": "Spine Switch",
                                    "key": "ssw-ul"
                                },
                                {
                                    "name": "Spine Switch",
                                    "key": "ssw-dl"
                                },
                                {
                                    "name": "Fabric Switch",
                                    "key": "fsw-ul"
                                },
                                {
                                    "name": "Fabric Switch",
                                    "key": "fsw-dl"
                                },
                                {
                                    "name": "Rack Switch",
                                    "key": "rsw-ul"
                                },
                                {
                                    "name": "Rack Switch",
                                    "key": "rsw-dl"
                                },
                                {
                                    "name": "NIC",
                                    "key": "nic-ul"
                                },
                                {
                                    "name": "NIC",
                                    "key": "nic-dl"
                                }
                            ],
                            "columns": [
                                {
                                    "name": "XOff RX",
                                    "key": "total-xoff-rx"
                                },
                                {
                                    "name": "XOff TX",
                                    "key": "total-xoff-tx"
                                },
                                {
                                    "name": "XOn RX",
                                    "key": "total-xon-rx"
                                },
                                {
                                    "name": "XOn TX",
                                    "key": "total-xon-tx"
                                },
                                {
                                    "name": "Total Time Paused (usec)",
                                    "key": "time-paused-usec"
                                },
                                {
                                    "name": "Direction (UL/DL)",
                                    "key": "direction"
                                }
                            ]
                        }
                    ]
                }
            ]
        },
        {
            "name": "tier-hop",
            "regex": ".*tier-hop-[0-9]*$",
            "columns": [
                {
                    "name": "sim_time_sec",
                    "dtype": "float64"
                },
                {
                    "name": "spine",
                    "dtype": "int"
                },
                {
                    "name": "fabric",
                    "dtype": "int"
                },
                {
                    "name": "rack",
                    "dtype": "int"
                }
            ],
            "plot": [
                {
                    "key": "hop-line",
                    "title": "Number of Packets per Tier vs Simulation Time",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": [
                        "spine",
                        "fabric",
                        "rack"
                    ],
                    "yaxis_title": "Number of Packets per Tier"
                },
                {
                    "key": "hop-ratio",
                    "title": "Ratio of Packets per Tier vs Simulation Time",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": [
                        "spine",
                        "fabric",
                        "rack"
                    ],
                    "yaxis_title": "Ratio of Packets per Tier (%)"
                },
                {
                    "key": "hop-interval",
                    "title": "Number of Packets per Tier per Interval vs Simulation Time",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": [
                        "spine",
                        "fabric",
                        "rack"
                    ],
                    "yaxis_title": "Number of Packets"
                }
            ]
        },
        {
            "name": "cpu-mem-stats",
            "regex": ".*cpu-mem-stats*",
            "columns": [
                {
                    "name": "time_utc",
                    "dtype": "string"
                },
                {
                    "name": "sim_time_sec",
                    "dtype": "float64"
                },
                {
                    "name": "host_name",
                    "dtype": "string"
                },
                {
                    "name": "core_rss_M",
                    "dtype": "float64"
                },
                {
                    "name": "sys_total_mem_M",
                    "dtype": "float64"
                },
                {
                    "name": "sys_used_mem_M",
                    "dtype": "float64"
                },
                {
                    "name": "sys_mem_usage_pct",
                    "dtype": "float64"
                },
                {
                    "name": "total_time_pct",
                    "dtype": "float64"
                }
            ],
            "plot": [
                {
                    "key": "system-memory-usage-vs-utc-time",
                    "title": "Number of Packets per Tier vs Simulation Time",
                    "xaxis": "time_utc",
                    "xaxis_title": "UTC Time",
                    "yaxis": "sys_mem_usage_pct",
                    "yaxis_title": "Memory Usage (%)",
                    "trace_column": "host_name"
                },
                {
                    "key": "system-memory-usage-vs-simulation-time",
                    "title": "System Memory Usage (%) vs Simulation Time",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": "sys_mem_usage_pct",
                    "yaxis_title": "Memory Usage (%)",
                    "trace_column": "host_name"
                },
                {
                    "key": "system-memory-usage-per-host",
                    "title": "System Memory Usage Per Core.",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": "core_rss_M",
                    "yaxis_title": "RSS Memory Usage (MB)",
                    "trace_column": "core"
                },
                {
                    "key": "system-cpu-usage-per-host",
                    "title": "Simulation CPU Usage Per Core.",
                    "xaxis": "time_utc",
                    "xaxis_title": "UTC Time",
                    "yaxis": "total_time_pct",
                    "yaxis_title": "CPU Usage (%)",
                    "trace_column": "core"
                }
            ]
        },
        {
    "name": "nd-stats",
    "regex": ".*nd-stats-[0-9]*$",
    "columns": [
        {
            "name": "sim_time_sec",
            "dtype": "float64"
        },
        {
            "name": "uldl",
            "dtype": "object"
        },
        {
            "name": "tier",
            "dtype": "int32"
        },
        {
            "name": "tx_rate_Gbps",
            "dtype": "float64"
        },
        {
            "name": "rx_rate_Gbps",
            "dtype": "float64"
        },
        {
            "name": "tx_drops",
            "dtype": "int64"
        },
        {
            "name": "rx_drops",
            "dtype": "int64"
        }
    ],
    "groupby": [
        "tier",
        "uldl",
        "sim_time_sec"
    ],
    "agg": {
        "tx_rate_Gbps": [
            "max",
            "min",
            "mean"
        ],
        "rx_rate_Gbps": [
            "max",
            "min",
            "mean"
        ],
        "tx_drops": [
            "sum"
        ],
        "rx_drops": [
            "sum"
        ]
    },
    "plots": [
        {
            "type": "line",
            "key": "nic-rx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) NIC RX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "nic-tx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) NIC TX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-ul-rx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) RSW UL RX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul",
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-ul-tx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) RSW UL TX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": null
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-dl-rx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) RSW DL RX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-dl-tx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) RSW DL TX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "fsw-dl-rx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) FSW DL RX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        1
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "fsw-dl-tx-throughput-vs-simulation-time",
            "title": "Aggregated (Max Min Mean) FSW DL TX Throughput vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_rate_Gbps": [
                    "max",
                    "min",
                    "mean"
                ]
            },
            "yaxis_title": "Throughput (Gbps)",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        1
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "nic-rx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) NIC RX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "nic-tx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) NIC TX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-ul-rx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) RSW UL RX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-ul-tx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) RSW UL TX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-dl-rx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) RSW DL RX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "rsw-dl-tx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) RSW DL TX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        2
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "fsw-dl-rx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) FSW DL RX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "rx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        1
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "fsw-dl-tx-drops-vs-simulation-time",
            "title": "Aggregated (Sum) FSW DL TX Drops vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "tx_drops": [
                    "sum"
                ]
            },
            "yaxis_title": "Drops",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        1
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "dl"
                    ]
                }
            ]
        }
    ]
},
        {
            "name": "perf",
            "regex": "*perf-[0-9]*$",
            "columns": [
                {
                    "name": "sim_time_sec",
                    "dtype": "float64"
                },
                {
                    "name": "metric_name",
                    "dtype": "object"
                },
                {
                    "name": "perf_metric_usec",
                    "dtype": "float64"
                },
                {
                    "name": "metric_path",
                    "dtype": "object"
                },
                {
                    "name": "metric_aggregation_id",
                    "dtype": "object"
                }
            ],
            "plot": [
                {
                    "key": "filter-agg",
                    "pre-defined": "true"
                },
                {
                    "key": "perf-agg",
                    "pre-defined": "true"
                },
                {
                    "key": "kde-chart",
                    "pre-defined": "true"
                },
                {
                    "key": "violin-chart",
                    "pre-defined": "true"
                },
                {
                    "key": "scatter-chart",
                    "pre-defined": "true"
                }
            ]
        },
        {
            "name": "ecn-pfc-stats",
            "regex": ".*ecn-pfc-stats-[0-9]*$",
            "columns": [
                {
                    "name": "sim_time_sec",
                    "dtype": "float64"
                },
                {
                    "name": "uldl",
                    "dtype": "object"
                },
                {
                    "name": "tier",
                    "dtype": "int32"
                },
                {
                    "name": "priority",
                    "dtype": "int32"
                },
                {
                    "name": "pfcxoff_rx",
                    "dtype": "int32"
                },
                {
                    "name": "pfcxoff_tx",
                    "dtype": "int32"
                },
                {
                    "name": "pfcxon_rx",
                    "dtype": "int32"
                },
                {
                    "name": "pfcxon_tx",
                    "dtype": "int32"
                }
            ],
            "plot": [
                {
                    "key": "pfc-vs-simulation-time",
                    "title": "PFC vs Simulation Time.",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": [
                        "pfcxoff_tx",
                        "pfcxoff_rx",
                        "pfcxon_tx",
                        "pfcxon_rx"
                    ],
                    "yaxis_title": "Count",
                    "agg_methods": [
                        "sum"
                    ]
                }
            ]
        },
        {
            "name": "qmon-fap",
            "regex": "qmon(?!.*log).*fap[0-9]*",
            "columns": [
                {
                    "name": "sim_time_sec",
                    "dtype": "float64"
                },
                {
                    "name": "queue_type",
                    "dtype": "object"
                },
                {
                    "name": "max_queue_size_interval",
                    "dtype": "int32"
                },
                {
                    "name": "max_queue_size",
                    "dtype": "int32"
                }
            ],
            "groupby": [
                "sim_time_sec",
                "queue_type"
            ],
            "agg_methods": [
                "max",
                "mean"
            ],
            "plot": [
                {
                    "key": "qmnfap-fapfifq-interval-agg",
                    "title": "DNX Switch Interval Cell Queue Sizes vs Simulation Time",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": [
                        "max_queue_size_interval"
                    ],
                    "yaxis_title": "Queue Size (Cells)",
                    "filters": [
                        {
                            "column": "queue_type",
                            "type": "==",
                            "value": [
                                "fapfifq"
                            ]
                        }
                    ]
                },
                {
                    "key": "qmnfap-voq-egrq-interval-agg",
                    "title": "DNX Switch Interval VOQ and Egress Queue Sizes vs Simulation Time",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": [
                        "max_queue_size_interval"
                    ],
                    "yaxis_title": "Queue Size (Bytes)",
                    "filters": [
                        {
                            "column": "queue_type",
                            "type": "==",
                            "value": [
                                "egrq",
                                "voq"
                            ]
                        }
                    ]
                },
                {
                    "key": "qmnfap-voq-egrq-agg",
                    "title": "DNX Switch VOQ and Ingress Queue Sizes vs Simulation Time",
                    "xaxis": "sim_time_sec",
                    "xaxis_title": "Simulation Time (sec)",
                    "yaxis": [
                        "max_queue_size"
                    ],
                    "yaxis_title": "Queue Size (Bytes)",
                    "filters": [
                        {
                            "column": "queue_type",
                            "type": "==",
                            "value": [
                                "egrq",
                                "voq"
                            ]
                        }
                    ]
                }
            ]
        },
        {
            "name": "qmon-fe",
            "regex": "qmon(?!.*log).*fe[0-9]*",
            "columns": [
                {
                    "name": "sim_time_sec",
                    "dtype": "float64"
                },
                {
                    "name": "queue_type",
                    "dtype": "object"
                },
                {
                    "name": "max_queue_size",
                    "dtype": "int32"
                }
            ],
            "groupby": [
                "sim_time_sec"
            ],
            "agg_methods": [
                "max",
                "mean"
            ],
            "plot": {
                "key": "qmnfe-agg",
                "title": "DNX Switch Cell Queue Sizes vs Simulation Time",
                "xaxis": "sim_time_sec",
                "xaxis_title": "Simulation Time (sec)",
                "yaxis": [
                    "max_queue_size"
                ],
                "yaxis_title": "Queue Size (Cells)"
            }
        }
    ]
}
```

DSP Architecture 

![image.png](DSP%20v2%20initiative%201e7da38a46628080856bde121a6ec4d2/image.png)

Orchestrator pulls tasks JSON from db by filtering on *ns3_version* AND *platform;* and modifies the JSON according to the simulation at Run when necessary (e.g. perf). DSP runs tasks according to the JSON. This design pushes the judging logic from the late stage (DSP) to earlier stages (device version deployment, platform deployment, and at simulation Run in orchestrator); thus, DSP becomes more independent. 

### Ecn-Pfc

```json
{
    "name": "ecn-pfc-stats",
    "regex": ".*ecn-pfc-stats-[0-9]*$",
    "columns": [
        {
            "name": "sim_time_sec",
            "dtype": "float64"
        },
        {
            "name": "uldl",
            "dtype": "object"
        },
        {
            "name": "tier",
            "dtype": "int32"
        },
        {
            "name": "priority",
            "dtype": "int32"
        },
        {
            "name": "pfcxoff_rx",
            "dtype": "int32"
        },
        {
            "name": "pfcxoff_tx",
            "dtype": "int32"
        },
        {
            "name": "pfcxon_rx",
            "dtype": "int32"
        },
        {
            "name": "pfcxon_tx",
            "dtype": "int32"
        }
    ],
    "groupby": [
        "tier",
        "priority",
        "uldl",
        "sim_time_sec"
    ],
    "agg": {
        "pfcxoff_tx": [
            "sum"
        ],
        "pfcxoff_rx": [
            "sum"
        ],
        "pfcxon_tx": [
            "sum"
        ],
        "pfcxon_rx": [
            "sum"
        ]
    },
    "plots": [
        {
            "type": "line",
            "key": "nic-pfc-xon-rx-vs-simulation-time",
            "title": "NIC PFC XON RX vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "pfcxon_rx": [
                    "sum"
                ]
            },
            "yaxis_title": "Count",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "priority",
                    "values": null
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "nic-pfc-xoff-rx-vs-simulation-time",
            "title": "NIC PFC XOFF RX vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "pfcxoff_rx": [
                    "sum"
                ]
            },
            "yaxis_title": "Count",
            "select": [
                {
                    "column": "tier",
                    "values": [
                        3
                    ]
                },
                {
                    "column": "priority",
                    "values": null
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "pfc-xon-tx-vs-simulation-time",
            "title": "PFC XON TX vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "pfcxon_tx": [
                    "sum"
                ]
            },
            "yaxis_title": "Count",
            "select": [
                {
                    "column": "tier",
                    "values": null
                },
                {
                    "column": "priority",
                    "values": [
                        0
                    ]
                },
                {
                    "column": "uldl",
                    "values": [
                        "ul"
                    ]
                }
            ]
        },
        {
            "type": "line",
            "key": "pfc-xon-rx-vs-simulation-time",
            "title": "PFC XON RX vs Simulation Time",
            "xaxis_column": "sim_time_sec",
            "xaxis_title": "Simulation Time (sec)",
            "yaxis_columns": {
                "pfcxon_rx": [
                    "sum"
                ]
            },
            "yaxis_title": "Count",
            "select": [
                {
                    "column": "tier",
                    "values": null
                },
                {
                    "column": "priority",
                    "values": [
                        0
                    ]
                },
                {
                    "column": "uldl",
                    "values": null
                }
            ]
        }
    ]
}
```

### Global Report Table

```json
{
    "name": "global-sim-report",
    "regex": ".*global-sim-report-[0-9]*$",
    "plots": [
        {
            "type": "table",
            "key": "net-device",
            "name": "global-network-interface-table",
            "title": "Network Interface Statistics",
            "rows": [
                {
                    "name": "Global",
                    "key": "global"
                },
                {
                    "name": "Spine Switch",
                    "key": "ssw-ul"
                },
                {
                    "name": "Spine Switch",
                    "key": "ssw-dl"
                },
                {
                    "name": "Fabric Switch",
                    "key": "fsw-ul"
                },
                {
                    "name": "Fabric Switch",
                    "key": "fsw-dl"
                },
                {
                    "name": "Rack Switch",
                    "key": "rsw-ul"
                },
                {
                    "name": "Rack Switch",
                    "key": "rsw-dl"
                },
                {
                    "name": "NIC",
                    "key": "nic-ul"
                },
                {
                    "name": "NIC",
                    "key": "nic-dl"
                }
            ],
            "columns": [
                {
                    "name": "Tier",
                    "key": "tier"
                },
                {
                    "name": "Packets RX",
                    "key": "packets-rx"
                },
                {
                    "name": "Packets TX",
                    "key": "packets-tx"
                },
                {
                    "name": "Bytes RX (GB)",
                    "key": "bytes-rx-gb"
                },
                {
                    "name": "Bytes TX (GB)",
                    "key": "bytes-tx-gb"
                },
                {
                    "name": "Ingress Drops (RX)",
                    "key": "ingress-drops-rx"
                },
                {
                    "name": "Egress Drops (TX)",
                    "key": "egress-drops-tx"
                },
                {
                    "name": "Direction (UL/DL)",
                    "key": "direction"
                }
            ]
        },
        {
            "type": "table",
            "key": "pfc",
            "name": "global-pfc-table",
            "title": "PFC Statistics",
            "rows": [
                {
                    "name": "Global",
                    "key": "global"
                },
                {
                    "name": "Spine Switch",
                    "key": "ssw-ul"
                },
                {
                    "name": "Spine Switch",
                    "key": "ssw-dl"
                },
                {
                    "name": "Fabric Switch",
                    "key": "fsw-ul"
                },
                {
                    "name": "Fabric Switch",
                    "key": "fsw-dl"
                },
                {
                    "name": "Rack Switch",
                    "key": "rsw-ul"
                },
                {
                    "name": "Rack Switch",
                    "key": "rsw-dl"
                },
                {
                    "name": "NIC",
                    "key": "nic-ul"
                },
                {
                    "name": "NIC",
                    "key": "nic-dl"
                }
            ],
            "columns": [
                {
                    "name": "Tier",
                    "key": "tier"
                },
                {
                    "name": "XOff RX",
                    "key": "total-xoff-rx"
                },
                {
                    "name": "XOff TX",
                    "key": "total-xoff-tx"
                },
                {
                    "name": "XOn RX",
                    "key": "total-xon-rx"
                },
                {
                    "name": "XOn TX",
                    "key": "total-xon-tx"
                },
                {
                    "name": "Total Time Paused (usec)",
                    "key": "time-paused-usec"
                },
                {
                    "name": "Direction (UL/DL)",
                    "key": "direction"
                }
            ]
        }
    ]
}
```

## Modernize DSP application

### 2.1 Platform & Runtime

- **Move to Ubuntu 24.04 LTS**
- **Upgrade to Python 3.12** (or 3.13)
    - Maybe enable JIT
- **Switch package manager to `uv`**
- **Install docker compose**

### 2.2 Dependency Hygiene

- Ship a *minimal* runtime image:
    - core scientific stack (**NumPy, Pandas, Dask**)
    - **Plotly** (totally remove bokeh, datashader, hvplot and etc)
- Generate a deterministic `uv.lock` file; ban unpinned transitive deps

### 2.3 Execution & Post‑processing

- Isolate heavy DSP post‑processing behind a celery/Dask‑distributed worker pool

# Success Criteria

| KPI | Target |
| --- | --- |
| **Cold start** | **< 30 s** from `docker run` to ready probe (was 3‑5 min)   |
| **Max problem size** | Handle ≥ 2× larger simulations without out‑of‑memory errors (300GB) |

---

### Next Steps

1. **Dependency audit**: review the code mark for removal.