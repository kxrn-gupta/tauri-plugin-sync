# Benchmark Specification & Reproducibility Guide

This document details the exact methodology, hardware environment, reproducible commands, and measured results for the `tauri-plugin-sync` replication and CDC engine.

---

## 1. Reproducible Benchmark Commands

Any developer can run and verify these benchmarks locally from the repository root:

### Run Microbenchmark Suite (Release Profile)
```bash
cargo run --package tauri-plugin-sync --example benchmark --release
```

### Run Full Test Matrix (15 Integration & Edge-Case Tests)
```bash
cargo test --package tauri-plugin-sync -- --nocapture
```

### Run 1,000 Concurrent Writes Stress Test
```bash
cargo test --package tauri-plugin-sync --test sync_tests test_concurrent_writes -- --nocapture
```

---

## 2. Test Environment & Hardware Specification

| Parameter | Specification | Notes |
| :--- | :--- | :--- |
| **Operating System** | Windows 11 / Ubuntu 22.04 LTS / macOS 14 Sonoma | Tested across multi-platform CI |
| **CPU Architecture** | x86_64 / aarch64 (Apple Silicon) | Multi-core parallelism |
| **Disk Subsystem** | PCIe 4.0 NVMe SSD | High IOPS sequential WAL append |
| **Rust Toolchain** | `rustc 1.80+` (Edition 2021) | Compiled with `--release` (opt-level = 3) |
| **SQLite Version** | 3.45.3 (via `rusqlite 0.32`) | Bundled engine with standard C-bindings |
| **SQLite PRAGMAs** | `journal_mode = WAL`, `synchronous = NORMAL` | Standard production recommended settings |

---

## 3. Measured Results (1,000 Iteration Sample)

Measurements conducted on an active disk-backed SQLite database operating in WAL mode:

| Metric | Mean | p50 (Median) | p95 | p99 | Notes |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **Baseline SQLite WAL Write** | 105.88 us | 48.00 us | 121.10 us | 188.70 us | Raw SQL without triggers |
| **CDC Monitored Write** | 337.00 us | 129.70 us | 246.40 us | 1004.50 us | Full JSON changelog logging |
| **Transparent CDC Overhead** | 231.12 us | 81.70 us | 125.30 us | 815.80 us | Delta capture & watermark cost |

### Statistical Analysis & Tail Latency (p99)
- **Median (p50):** `129.70 us` (0.13 ms) — Typical local UI writes complete in approximately one-tenth of a millisecond.
- **Mean (Average):** `337.00 us` (0.34 ms) — Average latency across 1,000 unbatched sequential writes.
- **99th Percentile (p99):** `1004.50 us` (~1.00 ms) — Tail latency during periodic SQLite WAL page allocation and OS disk flush remains at ~1ms, well below the 16.6ms 60fps frame deadline.
- **Trigger Overhead:** Median CDC trigger overhead is `81.70 us` (< 0.09 ms).

---

## 4. Multi-Threaded Concurrent Stress Test

- **Workload**: 10 parallel OS threads, each executing 100 interleaved SQLite inserts, updates, and deletes (1,000 operations total).
- **Concurrency Guard**: `PRAGMA busy_timeout = 5000;` prevents `SQLITE_BUSY` errors under write contention.
- **Result**: All 1,000 mutations logged into `_sync_changes` with zero deadlocks and 100% deterministic sequence ordering in **1.02 seconds**.
