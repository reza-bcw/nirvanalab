## Infrastructure Performance Comparison Report: Local PCIe vs. Network-Attached Storage
We have conducted a series of benchmark tests to evaluate the performance delta between our current Dedicated Infrastructure and the provided Networked Infrastructure. Both systems were tested under active load to simulate real-world production environments.

### The primary variable in this comparison is the storage architecture:

Our Current Infrastructure: Utilizes Direct Attached Storage (DAS) via the PCIe bus/Motherboard. This eliminates network overhead and provides the lowest possible latency.
Tested Infrastructure: Utilizes Network-Based Storage. Data traversal occurs over a network fabric, introducing encapsulation and switching overhead.
3. Objective of the Analysis
The goal of these tests—accompanied by the attached screenshots and output logs—is not merely to confirm that local storage is faster, but to quantify the performance gap. We aim to measure how the transition from a local PCIe interface to a networked environment impacts:

# Storage Performance Comparison Report
## Latitude vs NirvanaLab

## 1) Objective

This document compares the storage behavior of the two environments under live workload conditions using the same family of operational tools:

- `iostat`
- `ioping`
- `dstat`
- `iotop`

The goal is to evaluate the primary application storage path in each environment and determine which datacenter currently provides the stronger storage profile for the active Solana/Agave workload.

---

## 2) Scope of Comparison

To keep the comparison consistent, only the primary workload storage target was evaluated on each system:

| Datacenter | Target Volume | Mount Path | Device Used for Analysis |
|---|---|---|---|
| **Latitude** | main data/workload array | `/home` | **`md1`** |
| **NirvanaLab** | main data volume | `/data` | **`vdb` / `/dev/vdb1`** |

### Additional normalization rules
- For **`iostat`**, only the primary data device was considered:
  - Latitude: `md1`
  - NirvanaLab: `vdb`
- For **`iotop`**, only the **`agave-validator`** process was considered as the application-level storage consumer.
- For **`ioping`**, the direct test target was:
  - Latitude: `/home` on `md1`
  - NirvanaLab: `/data` on `/dev/vdb1`

---

## 3) Test Methodology

| Tool | Purpose | Interpretation |
|---|---|---|
| `ioping` | Measures storage latency and short-block response time | Best indicator for responsiveness and consistency |
| `iostat` | Measures sustained device-level read/write behavior | Best indicator for operational throughput, queue depth, and device pressure |
| `dstat` | Captures host-level disk burst behavior over time | Useful for identifying bursty write/read patterns |
| `iotop` | Identifies the top live I/O consumer process | Used here to isolate the `agave-validator` workload |

### Important note
These were **live operational measurements**, not a synthetic `fio` benchmark.  
Accordingly, this report reflects **real workload behavior**, which is often more useful operationally, but it should not be interpreted as a maximum hardware ceiling benchmark.

---

## 5) Detailed Results

## 5.1 `ioping` Comparison
Primary responsiveness test on the target data path.

| Metric | Latitude (`/home` on `md1`) | NirvanaLab (`/data` on `/dev/vdb1`) | Better |
|---|---:|---:|---|
| Min latency | **114.6 us** | 392.6 us | **Latitude** |
| Avg latency | **121.2 us** | 2.32 ms | **Latitude** |
| Max latency | **139.9 us** | 8.93 ms | **Latitude** |
| Latency deviation (mdev) | **8.17 us** | 3.06 ms | **Latitude** |
| IOPS | **8.25k** | 431 | **Latitude** |
| Throughput | **32.2 MiB/s** | 1.68 MiB/s | **Latitude** |

### Interpretation
Latitude delivered a substantially stronger latency profile:
- ~19x lower average latency
- much lower tail latency
- materially higher small-block IOPS

This indicates a far more responsive and stable storage path for the workload volume.

![](img/lati_ioping.png)

![](img/nirvana_ioping.png)

---

## 5.2 `iostat` Comparison
Device-level view of sustained read/write behavior on the primary workload device.

### Read-side comparison

| Metric | Latitude (`md1`) | NirvanaLab (`vdb`) | Better |
|---|---:|---:|---|
| Avg `r/s` | **985.71** | 44.70 | **Latitude** |
| Peak `r/s` | **3304.00** | 252.54 | **Latitude** |
| Avg `rkB/s` | **64407.56** | 4400.65 | **Latitude** |
| Peak `rkB/s` | **224218.00** | 40580.53 | **Latitude** |
| Avg `r_await` | **0.20 ms** | 0.89 ms | **Latitude** |
| Max `r_await` | **0.29 ms** | 1.49 ms | **Latitude** |

### Write-side comparison

| Metric | Latitude (`md1`) | NirvanaLab (`vdb`) | Better |
|---|---:|---:|---|
| Avg `w/s` | **2115.59** | 76.85 | **Latitude** |
| Peak `w/s` | **2734.00** | 382.00 | **Latitude** |
| Avg `wkB/s` | **207163.98** | 41719.49 | **Latitude** |
| Peak `wkB/s` | **266830.00** | 217698.00 | **Latitude** |
| Avg `w_await` | **3.26 ms** | 3.94 ms | **Latitude** |
| Max `w_await` | **4.94 ms** | 18.03 ms | **Latitude** |

### Queue and utilization comparison

| Metric | Latitude (`md1`) | NirvanaLab (`vdb`) | Interpretation |
|---|---:|---:|---|
| Avg `aqu-sz` | **7.32** | 0.89 | Latitude is servicing a much deeper queue |
| Max `aqu-sz` | **11.88** | 5.11 | Latitude carried more concurrent pressure |
| Avg `%util` | **19.23%** | 5.47% | Latitude is doing substantially more work |
| Max `%util` | **50.45%** | 18.20% | Latitude was more heavily exercised |

### Interpretation
Latitude is not merely “busier”; it is **handling materially more read and write traffic while maintaining better latency characteristics**.

This is the most important finding in the report:

> **Latitude sustained much higher storage throughput at better latency than NirvanaLab on the primary workload path.**

### nirvanalab
![](img/nirvana_iostate.png)

![](img/nirvana_iostate2.png)


### latitude
![](img/lati_iostat.png)

![](img/lati_iostat2.png)


---

## 5.3 `dstat` Comparison
Host-level burst behavior over the sampling window.

| Metric | Latitude | NirvanaLab | Observation |
|---|---:|---:|---|
| Peak read | 8648k/s | **332M/s** | NirvanaLab showed larger read bursts |
| Peak write | 369M/s | **1374M/s** | NirvanaLab showed much larger write bursts |
| Average read (window) | 1653.7 kB/s | **54537.3 kB/s** | NirvanaLab was burstier at host level |
| Average write (window) | 40668.6 kB/s | **379748.3 kB/s** | NirvanaLab showed much heavier short-window write bursts |

### Interpretation
`dstat` reflects total host-level disk behavior and captures short-lived bursts.  
In this data set, **NirvanaLab showed significantly larger burst spikes**, especially on writes.

However, this does **not** mean NirvanaLab has the better storage platform.  
When correlated with `ioping` and `iostat`, the evidence suggests:

- NirvanaLab experiences larger burst behavior
- but its target storage path (`vdb` / `/data`) is less responsive and less consistent
- while Latitude sustains heavier workload more efficiently on the target data volume

### nirvanalab
![](img/nirvana_dstat.png)

### latitude
![](img/lati_dstat.png)

---

## 5.4 `iotop` Comparison
Application-level view focused on `agave-validator`.

| Metric | Latitude (`agave-validator`) | NirvanaLab (`agave-validator`) | Better |
|---|---:|---:|---|
| Process disk read | **40.32 M** | 25.48 M | **Latitude** |
| Process disk write | **336.82 M** | 208.61 M | **Latitude** |

### System totals observed in the same snapshots

| Metric | Latitude | NirvanaLab |
|---|---:|---:|
| Total disk read | 5.15 M/s | 272.28 K/s |
| Total disk write | 14.93 M/s | **22.80 M/s** |

### Interpretation
The `agave-validator` workload on Latitude is pushing more I/O than on NirvanaLab, yet the primary device still demonstrates better latency and stronger sustained throughput.

This is an indicator that Latitude has **more operational storage headroom** for the workload.

---

## 6) Operational Interpretation

### Latitude
Strengths observed:
- excellent small-block responsiveness on `md1`
- much higher sustained read/write rates
- better read latency and better worst-case write latency
- capable of servicing deeper queues without obvious storage distress

Operationally, this suggests Latitude is the stronger platform for the current Agave workload.

### NirvanaLab
Observations:
- larger short-lived write bursts at host level
- materially weaker latency profile on `/data`
- lower sustained throughput on the target device
- worse worst-case write latency

Operationally, NirvanaLab appears more sensitive to burst pressure and shows a less consistent storage response on the primary workload volume.



