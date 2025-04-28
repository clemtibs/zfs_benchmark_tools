# ZFS Benchmarking for RAIDZ2 vs dRAID2

These tests are designed to explore how ZFS performs under different storage
topologies, hardware configurations, and I/O workloads. Particularly, the
differences between RAIDZ2 and dRAID2 as documented in this
[question](https://serverfault.com/q/1150567/182812) with my 5 drive NAS.

The suite of tests are also to measure the real-world impact of SLOGs, special
vdevs, and layout types like dRAID and RAIDZ2 on throughput, latency,
durability, and metadata responsiveness by systematically running key FIO
benchmarks across these combinations. This testing aims to reveal strengths,
bottlenecks, and tradeoffs in realistic deployment scenarios.

Note: I determined that L2Arc wasn't going to be implemented on my NAS, so it's
listed here as optional and wasn't included in my tests.

Please excuse the hardcoded and rough nature of the script there are some features
that are half implemented, and some that are obsolete. I tried to make configuration
obvious in case you wanted to reuse it to tweak it. If you wanted 
to run it on your system, make note of the requirements (`zfsutils-linux, fio, arcstat, smartmontools, nvme-cli`),
and particularly the configuration options under "Script Config" at the top.

One handy tool is the `histogram` subcommand which I found on ZFS forums for determining
the amount of files in a given directory that that are of varous sizes. This
helps determine recordsize for a dataset.

My first run of test data is included here for analysis. As I get better visualizations and
draw conclusions, I'll continue to update this repository with
[findings](#findings).

## How to Help

**FIO Experts**

Please double check my FIO tests to make sure that they test what they intended.

**ZFS experts**

I'm pretty sure my tuning was sane and pretty close to what my intended setup
would look like, but always worth another look.

**Data Analysis/Visualization**

- Charts and graphs so that the rest of us can draw conclusions and reference these tests.
- Better yet, if you can contribute a script to automate that process, I'll
  include it here.

---

## Hardware Specs

- 5x TOSHIBA X300 Pro HDWR51CXZSTB 12TB 7200 RPM 512MB Cache SATA 6.0Gb/s 3.5" HDD
  - main pool
- TOPTON / CWWK CW-5105NAS w/ N6005 (CPUN5105-N6005-6SATA) NAS mainboard
- 64GB RAM
- 1x SAMSUNG 870 EVO Series 2.5" 500GB SATA III V-NAND SSD MZ-77E500B/AM
  - Operating system. XFS on LVM
- 2x SAMSUNG 870 EVO Series 2.5" 500GB SATA III V-NAND SSD MZ-77E500B/AM
  - mirrored for special metadata vdevs
- Nextorage Japan 2TB NVMe M.2 2280 PCIe Gen.4 Internal SSD
  - Reformatted to 4096b sector size
  - 3 GPT partitions
    - volatile OS files
    - SLOG special device
    - L2Arc (was considering, but decided to not use on this machine)

## ⚖️ Pool Configurations

| Pool Label                   | Topology | Special VDEV | SLOG | Cache  | Description                                |
|------------------------------|----------|--------------|------|--------|--------------------------------------------|
| `tank_raidz2_special_slog`   | RAIDZ2   |      ✅      |  ✅  |   ❔   | 🛠️ Production RAIDZ2 with all extras       |
| `tank_raidz2_nospecial_slog` | RAIDZ2   |      ❌      |  ✅  |   ❔   | 🛠️ RAIDZ2 with HDDs only                   |
| `tank_draid2_special_slog`   | dRAID2   |      ✅      |  ✅  |   ❔   | 🛠️ Production dRAID2 with all extras       |
| `tank_draid2_nospecial_slog` | dRAID2   |      ❌      |  ✅  |   ❔   | 🛠️ dRAID2 with HDDs only                   |
| `tank_raidz2_special_noslog` | RAIDZ2   |      ✅      |  ❌  |   ❔   | 🧪 Special vdev but no SLOG                |
| `tank_draid2_special_noslog` | dRAID2   |      ✅      |  ❌  |   ❔   | 🧪 Special vdev but no SLOG                |

## 🔢 FIO Workload Summary

| FIO Test Name         | Description                                           | Uses SLOG?   | Uses Special VDEV?    |
|-----------------------|-------------------------------------------------------|--------------|-----------------------|
| `async_seq_write`     | 📦 Bulk sequential write (e.g., large file upload)    | ❌           | ❌                    |
| `sync_seq_write`      | 🔒 Sync large file write (NFS, VM durability)         | ✅           | ❌                    |
| `async_rand_write`    | 🎯 Random writes under async context                  | ❌           | ⚠️ Partial (if small)  |
| `sync_rand_write`     | 🔒🎯 Random sync writes (worst-case fsync)            | ✅           | ⚠️ Partial (if small)  |
| `metadata_stress`     | 🧬 Small files and metadata-heavy ops                 | ⚠️  Sometimes | ✅                    |
| `mixed_rw`            | 🔄 70/30 random read/write blend                      | ⚠️  Sometimes | ✅                    |
| `large_read`          | 📖 Sequential read (e.g., ML dataset staging)         | ❌           | ❌                    |
| `slog_punisher`       | 🔥 Stress test sync durability throughput             | ✅           | ❌                    |

## ⚙️ Variables Under Test

| Parameter           | Values                 | Purpose                                   |
|---------------------|------------------------|-------------------------------------------|
| 🧱 Topology         | RAIDZ2, dRAID2         | Assess layout impact on performance       |
| 🧩 Special VDEV     | Present, Absent        | Offload metadata/small files              |
| 🪵 SLOG (log vdev)  | Present, Absent        | Reduce fsync() latency                    |
| 💾 Cache VDEV       | Optional               | Helps ARC pressure, omitted for now       |
| 🧪 Workloads        | 8 FIO tests            | Simulate various real-world scenarios     |

## 🔄 Test Matrix

- ✅ **Run all 8 FIO tests** on these 4 main configurations:
  - 🧪 `tank_raidz2_special`
  - 🧪 `tank_raidz2_nospecial`
  - 🧪 `tank_draid2_special`
  - 🧪 `tank_draid2_nospecial`

- ⚠️ **Run sync-focused tests only** on these 2 variants (omit SLOG):
  - 🧪 `tank_raidz2_special_noslog`
  - 🧪 `tank_draid2_special_noslog`

- 📌 Relevant tests for "no SLOG" setups:
  - 🔒 `sync_seq_write`
  - 🔒🎯 `sync_rand_write`
  - 🔥 `slog_punisher`

## 🔎 Summary

- 📦 **6 unique ZFS pools**
- 📈 **40+ test runs** across all combinations
- 📊 Logged: ARC stats, I/O metrics, SMART status, memory usage
- 🎯 Focus: durability, metadata performance, read/write throughput, topology comparison

## Notes

- You will need to increase ulimit for metadata tests above the default 1024

## 🧪 Test Schedule

Pool name uniqueness is for the sake of avoiding collisions in the logging
output. These commands can help you run all the tests.

- Baseline (no special, no slog)
  - tank_raidz2_nospecial_noslog
    - setup:
      - `zfs_benchmark_tool --name=tank_raidz2_nospecial_noslog setup raidz2`
    - tests
      - sync_seq_write
        - `zfs_benchmark_tool --name=tank_raidz2_nospecial_noslog --type=raidz2 test sync_seq_write`
      - sync_rand_write
        - `zfs_benchmark_tool --name=tank_raidz2_nospecial_noslog --type=raidz2 test sync_rand_write`
      - slog_punisher
        - `zfs_benchmark_tool --name=tank_raidz2_nospecial_noslog --type=raidz2 test slog_punisher`
    - teardown:
      - `zfs_benchmark_tool --name=tank_raidz2_nospecial_noslog destroy`
  - tank_draid2_nospecial_noslog
    - setup:
      - `zfs_benchmark_tool --name=tank_draid2_nospecial_noslog setup draid2`
    - tests
      - sync_seq_write
        - `zfs_benchmark_tool --name=tank_draid2_nospecial_noslog --type=draid2 test sync_seq_write`
      - sync_rand_write
        - `zfs_benchmark_tool --name=tank_draid2_nospecial_noslog --type=draid2 test sync_rand_write`
      - slog_punisher
        - `zfs_benchmark_tool --name=tank_draid2_nospecial_noslog --type=draid2 test slog_punisher`
    - teardown:
      - `zfs_benchmark_tool --name=tank_draid2_nospecial_noslog destroy`
- With Special (no slog)
  - tank_raidz2_special_noslog
    - setup:
      - `zfs_benchmark_tool --name=tank_raidz2_special_noslog setup raidz2`
      - `zfs_benchmark_tool --name=tank_raidz2_special_noslog add special`
    - tests
      - sync_seq_write
        - `zfs_benchmark_tool --name=tank_raidz2_special_noslog --type=raidz2 test sync_seq_write`
      - sync_rand_write
        - `zfs_benchmark_tool --name=tank_raidz2_special_noslog --type=raidz2 test sync_rand_write`
      - slog_punisher
        - `zfs_benchmark_tool --name=tank_raidz2_special_noslog --type=raidz2 test slog_punisher`
    - teardown:
      - `zfs_benchmark_tool --name=tank_raidz2_special_noslog destroy`
  - tank_draid2_special_noslog
    - setup:
      - `zfs_benchmark_tool --name=tank_draid2_special_noslog setup draid2`
      - `zfs_benchmark_tool --name=tank_draid2_special_noslog add special`
    - tests
      - sync_seq_write
        - `zfs_benchmark_tool --name=tank_draid2_special_noslog --type=draid2 test sync_seq_write`
      - sync_rand_write
        - `zfs_benchmark_tool --name=tank_draid2_special_noslog --type=draid2 test sync_rand_write`
      - slog_punisher
        - `zfs_benchmark_tool --name=tank_draid2_special_noslog --type=draid2 test slog_punisher`
    - teardown:
      - `zfs_benchmark_tool --name=tank_draid2_special_noslog destroy`
- With Slog (no special)
  - tank_raidz2_nospecial_slog
    - setup:
      - `zfs_benchmark_tool --name=tank_raidz2_nospecial_slog setup raidz2`
      - `zfs_benchmark_tool --name=tank_raidz2_nospecial_slog add log`
    - all 8 fio tests
      - `zfs_benchmark_tool --name=tank_raidz2_nospecial_slog --type=raidz2 test all-fio`
    - teardown:
      - `zfs_benchmark_tool --name=tank_raidz2_nospecial_slog destroy`
  - tank_raidz2_special_slog
    - setup:
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog setup raidz2`
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog add special`
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog add log`
    - all 8 fio tests
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog --type=raidz2 test all-fio`
    - teardown:
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog destroy`
  - tank_draid2_nospecial_slog
    - setup:
      - `zfs_benchmark_tool --name=tank_draid2_nospecial_slog setup draid2`
      - `zfs_benchmark_tool --name=tank_draid2_nospecial_slog add log`
    - all 8 fio tests
      - `zfs_benchmark_tool --name=tank_draid2_nospecial_slog --type=draid2 test all-fio`
    - teardown:
      - `zfs_benchmark_tool --name=tank_draid2_nospecial_slog destroy`
  - tank_draid2_special_slog
    - setup:
      - `zfs_benchmark_tool --name=tank_draid2_special_slog setup draid2`
      - `zfs_benchmark_tool --name=tank_draid2_special_slog add special`
      - `zfs_benchmark_tool --name=tank_draid2_special_slog add log`
    - all 8 fio tests
      - `zfs_benchmark_tool --name=tank_draid2_special_slog --type=draid2 test all-fio`
    - teardown:
      - NONE, reuse data and proceed to resilver tests
- With Special and Slog
  - tank_draid2_special_slog
    - setup:
      - reuse previous pool and data
    - resilver test
      - `zfs_benchmark_tool --name=tank_draid2_special_slog --type=draid2 test resilver`
    - teardown:
      - `zfs_benchmark_tool --name=tank_draid2_special_slog destroy`
  - tank_raidz2_special_slog
    - setup:
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog setup raidz2`
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog add special`
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog add log`
    - resilver test
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog --type=raidz2 test resilver`
    - teardown:
      - `zfs_benchmark_tool --name=tank_raidz2_special_slog destroy`

---

## Findings

### Layout Effect on Resilver Times

**Conclusion** dRAID2 resilvers faster with default settings, and potentially only
marginally faster when using tuning tactings on both (tunings were not used on
these tests). dRAID handles fragmentation a bit better (not sure about resource
use), but when actively rebuilding, both setups achieve that ~10GB/min number which 
is pretty much the max write speed the HDD is capable of.

#### 5×12TB Pool (80% full, ~29TB of data) — Resilver Rates and Times

Assuming that resilvering scales *mostly* linearly with pool usage, I
extrapolated what resilvers would look like with my pool at 80% usage.

*Disclosure: ChatGPT was used to help create this chart and summary from my two
summary.log files*

|Pool Type | Tuning | Active Resilver Rate | Real-World Wallclock Rate | Active Resilver Time (Ideal) | Wall Clock Time (Real) |
|----------|--------|----------------------|---------------------------|------------------------------|------------------------|
|  RAIDZ2  | Default (untuned) | ~10 GB/min | ~4–5 GB/min | ~48h | ~100h (4+ days)
|  RAIDZ2  | Tuned (delay=0, max_active++) | ~16–20 GB/min | ~12–16 GB/min | ~30–36h | ~55–70h (2.5–3 days)
|  dRAID2  | Default (untuned) | ~10.9 GB/min | ~9–10 GB/min | ~44h | ~56–60h (2.5 days)
|  dRAID2  | Tuned (delay=0, max_active++) | ~12–14 GB/min | ~11–13 GB/min | ~36–40h | ~48–50h (2 days)

|       | RAIDZ2 Untuned | RAIDZ2 Tuned | dRAID2 Untuned | dRAID2 Tuned |
|-------|----------------|--------------|----------------|--------------|
Active GB/min | ~10 | ~16–20 | ~10.9 | ~12–14
Wallclock GB/min | ~4–5 | ~12–16 | ~9–10 | ~11–13
Wallclock Time | ~100h | ~55–70h | ~56–60h | ~48–50h
