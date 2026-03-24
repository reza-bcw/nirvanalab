# Solana RPC Hardware Validation Report

## Overview

This document summarizes the hardware validation and runtime assessment of the Solana RPC node provided for testing. The goal of this review was to evaluate whether the current server configuration is suitable for running a heavy Solana RPC workload and to identify any operational risks or tuning considerations. The server under review is running `solana-rpc.service` on NixOS 25.11 inside a KVM virtualized environment. 

---

## Environment Summary

| Item            | Value                  |
| --------------- | ---------------------- |
| Hostname        | `solana`               |
| Service         | `solana-rpc.service`   |
| OS              | NixOS 25.11 (Xantusia) |
| Kernel          | `6.12.76`              |
| Virtualization  | KVM                    |
| Hardware Vendor | KubeVirt               |
| Architecture    | x86_64                 |

The workload is running on a virtual machine rather than bare metal, so these results should be interpreted as VM performance characteristics, not maximum physical-host performance. 

---

## Hardware Specifications

### CPU

| Item             | Value                    |
| ---------------- | ------------------------ |
| CPU Model        | AMD EPYC-Genoa Processor |
| vCPU Count       | 52                       |
| Threads per Core | 1                        |
| Sockets          | 1                        |
| NUMA Nodes       | 1                        |
| Hypervisor       | KVM                      |

### Memory

| Item                       | Value    |
| -------------------------- | -------- |
| Total RAM                  | 440 GiB  |
| Used at sampling time      | ~197 GiB |
| Available at sampling time | ~253 GiB |
| Swap                       | Disabled |

### Storage

| Mount   | Device      | Filesystem | Size | Used | Free | Use% |
| ------- | ----------- | ---------- | ---: | ---: | ---: | ---: |
| `/`     | `/dev/vda3` | ext4       |  92G |  12G |  76G |  14% |
| `/boot` | `/dev/vda2` | vfat       | 511M |  40M | 472M |   8% |
| `/data` | `/dev/vdb1` | ext4       | 2.7T | 1.8T | 780G |  71% |

A dedicated tmpfs mount is configured for `/data/solana/account-index` with a 200G memory-backed allocation. This improves account-index performance but shifts part of the workload pressure onto RAM.

---

## Solana Service Configuration

The node is configured as a heavy RPC node rather than a voting validator. Based on the active systemd process arguments, the service is running with the following key characteristics: 

### Runtime Profile

| Setting                        | Value                                        |
| ------------------------------ | -------------------------------------------- |
| Mode                           | RPC node                                     |
| Voting                         | Disabled (`--no-voting`)                     |
| RPC API                        | Full (`--full-rpc-api`)                      |
| Transaction History            | Enabled (`--enable-rpc-transaction-history`) |
| RPC Visibility                 | Private (`--private-rpc`)                    |
| Ledger Verification on Startup | Skipped                                      |
| WAL Recovery                   | `skip_any_corrupted_record`                  |
| Ledger Path                    | `/data/solana/ledger`                        |
| Accounts Path                  | `/data/solana/accounts`                      |
| Log File                       | `/home/data/solana-rpc.log`                  |
| Snapshot Interval              | 5000 slots                                   |
| Ledger Size Limit              | `2000000000`                                 |

### systemd Tuning

| Parameter      | Value                 |
| -------------- | --------------------- |
| Restart Policy | `on-failure`          |
| Restart Delay  | 1s                    |
| `LimitNOFILE`  | 1000000               |
| `LimitMEMLOCK` | 2000000000            |
| `Nice`         | -20                   |
| IO Scheduling  | realtime / priority 0 |

These settings are aligned with a performance-oriented Solana RPC deployment. In particular, the elevated file descriptor limit, negative nice level, and realtime I/O scheduling are all beneficial for this class of workload. 

---

## Solana Data Layout

| Path                         |            Size |
| ---------------------------- | --------------: |
| `/data`                      |            1.8T |
| `/data/solana`               |            1.8T |
| `/data/solana/ledger`        |            1.4T |
| `/data/solana/accounts`      |            457G |
| `/data/solana/account-index` | 0 (tmpfs mount) |
| `/home/data`                 |            4.7G |
| `/home/data/solana-rpc.log`  |            4.7G |

The ledger is the dominant storage consumer, followed by the accounts database. The current `/data` utilization is still acceptable, but capacity growth should be monitored closely due to the combination of full RPC API and transaction history retention.

---

## Runtime Resource Utilization

### CPU Behavior

At the time of sampling, the system load average was approximately `11.66 / 12.05 / 12.48`, which is acceptable for a 52-vCPU node. Aggregate CPU idle remained high at roughly 79–83%, indicating that the machine was not globally CPU-saturated. The main Solana process (`agave-validator`) was consuming the vast majority of the active CPU budget.

One notable observation is that `CPU0` stayed near 99–100% during the per-core sampling window, while most other cores retained substantial idle capacity. This suggests a hot core, IRQ imbalance, or a scheduler/thread-affinity concentration worth investigating further.

### Memory Behavior

| Metric                  |    Value |
| ----------------------- | -------: |
| System RAM              |  440 GiB |
| RAM used at sample time | ~197 GiB |
| Available RAM           | ~253 GiB |
| Solana process RSS      | ~175 GiB |
| Service memory current  | ~320 GiB |
| Service memory peak     | ~435 GiB |

The node has enough memory for the current workload, but Solana is clearly operating as a memory-intensive RPC service. Peak memory usage is high enough that future growth in ledger, history, or indexing behavior could make memory a limiting factor.

### Service-Level Counters

| Metric          |                   Value |
| --------------- | ----------------------: |
| Active Since    | 2026-03-23 13:26:10 UTC |
| Tasks           |                     525 |
| Network In      |                    1.1T |
| Network Out     |                  887.6G |
| Disk Read       |                      4T |
| Disk Written    |                   12.5T |
| Restart Counter |                       5 |

These counters show that the node is handling a substantial Solana workload and is actively reading/writing large volumes of data, which is expected for this role. However, the restart count should be investigated before considering the environment fully validated. 

---

## Operational Assessment

### What looks good

* CPU headroom is still available at the node level.
* Overall memory capacity is strong.
* Storage allocation is large enough for current testing.
* The service is tuned with sensible limits for a heavy Solana RPC role.
* The ledger and accounts are placed on the dedicated `/data` volume, which is appropriate.

### What needs attention

* `CPU0` remains heavily loaded compared to the rest of the cores.
* The service has restarted multiple times before reaching the current stable state.
* Peak memory usage is high and should be tracked over longer test windows.
* `/data` is already at 71% utilization and will continue to grow under this profile.
* This is a VM-based test, so bare-metal performance and latency characteristics are still unverified.

---

## Conclusion

The current server is **suitable for Solana RPC testing** and is capable of running the configured workload without signs of immediate system-wide CPU or memory exhaustion. The node is clearly provisioned with strong CPU and memory resources, and the service-level tuning is appropriate for high-throughput RPC usage.

That said, this setup should be treated as **conditionally healthy**, not fully production-cleared yet. The most important follow-up items are:

1. investigate the previous service restarts,
2. review the CPU0 hot-core behavior,
3. continue monitoring memory growth and `/data` capacity,
4. validate comparable behavior under longer-duration and possibly bare-metal testing.

---

## Final Verdict

| Area              | Status  | Notes                                         |
| ----------------- | ------- | --------------------------------------------- |
| CPU Capacity      | Good    | No global saturation observed                 |
| CPU Distribution  | Warning | CPU0 consistently hot                         |
| Memory Capacity   | Good    | Enough headroom, but peak usage is high       |
| Disk Capacity     | Warning | `/data` already at 71%                        |
| Service Tuning    | Good    | Limits and priorities are properly configured |
| Service Stability | Warning | Restart counter indicates prior failures      |
| Environment Type  | Note    | VM-based, not bare metal                      |

---

## Recommended Next Steps

* Review `journalctl` around the earlier failures to determine why the service restarted.
* Inspect IRQ affinity / CPU pinning / thread distribution to understand CPU0 saturation.
* Continue monitoring memory peak and account-index growth over longer runtime windows.
* Plan ahead for `/data` growth if transaction history remains enabled.
* Repeat the same test on bare metal if this server is being evaluated for production-grade Solana deployment.

