# Unix Command-Line Pattern: MapReduce File Processing

A concise reference on implementing parallel MapReduce architectures directly in the shell using standard Unix utilities (`find`, `xargs`, and `awk`).

---

## 1. Architecture Overview

```
Input Generation            Parallel Mapper Stage                Serial Reducer Stage
=================          ======================               ====================
[ find . -type f ]  --->   [ xargs -P<cores> awk '...' ]  --->  [ awk '{sum += $1} ...' ]
  Emits file paths           Spawns parallel workers              Aggregates subtotals
  (null-delimited)           Outputs intermediate tuples          Emits grand totals

```

* **Input Generator (`find`)**: Identifies file partitions and streams paths safely.
* **Map Phase (`xargs -P<N> + awk`)**: Distributes batches of files across $N$ CPU cores, running local parsing and filtering to produce intermediate summary metrics.
* **Reduce Phase (`awk`)**: Consumes the streaming subtotal lines from all workers and aggregates them into final global sums.

---

## 2. Reference Command

```bash
find . -type f -name '*.pgn' -print0 | \
  xargs -0 -n4 -P$(nproc) awk '
    /Result/ {
      split($0, a, "-");
      res = substr(a[1], length(a[1]), 1);
      if (res == 1) white++;
      if (res == 0) black++;
      if (res == 2) draw++;
    }
    END {
      print white+black+draw, white, black, draw
    }
  ' | \
  awk '
    {
      games += $1;
      white += $2;
      black += $3;
      draw  += $4;
    }
    END {
      print "Total Games:", games, "| White Wins:", white, "| Black Wins:", black, "| Draws:", draw
    }
  '

```

---

## 3. Flag & Parameter Mechanics

| Pipeline Component | Argument / Flag | Engineering Purpose |
| --- | --- | --- |
| **`find`** | `-type f` | Restricts discovery strictly to regular files, ignoring directory nodes and symlinks. |
|  | `-print0` | Emits paths terminated with a null character (`\0`) to protect against paths containing spaces, tabs, or newlines. |
| **`xargs`** | `-0` | Reads null-delimited input from `find -print0` without breaking on whitespace. |
|  | `-n4` | **Batch size (chunking):** Hands 4 files per process invocation, amortizing process `fork`/`exec` overhead. |
|  | `-P$(nproc)` | **Concurrency factor:** Spawns up to the machine's maximum logical core count of concurrent worker processes. |
| **Mapper `awk**` | `/Result/ { ... }` | In-engine pattern filter (replaces piping through `grep`), keeping data movement in a single process memory space. |
|  | `split()` / `substr()` | Parses local records and extracts target state indicators. |
|  | `END { print ... }` | Emits a single whitespace-delimited intermediate tuple (`Total White Black Draw`) per batch. |
| **Reducer `awk**` | `{ games += $1 ... }` | Consumes incoming batch lines sequentially and sums columnar fields. |
|  | `END { print ... }` | Formats and emits final aggregated corpus counts. |

---

## 4. Key Performance Optimizations

* **Pipe Elimination**: Replaces `cat file | grep pattern | awk` with a native `awk '/pattern/ { ... }'` rule, eliminating two inter-process pipe transfers and scheduler context switches.
* **Core Saturation**: Standard `awk` or `grep` is strictly single-threaded (max 100% on one core). Introducing `xargs -P` saturates multi-socket/multi-core systems up to $N \times 100\%$ CPU utilization.
* **Batch Tuning (`-n`)**:
* Passing `-n1` creates one process per file—inefficient for thousands of small files due to process startup latency.
* Setting `-n4` or `-n8` minimizes startup overhead while keeping worker load balanced across cores.



---

## 5. Verification Checklist

1. **Verify logical core count before sizing `-P`:**
```bash
nproc

```


2. **Observe live multi-core parallel scaling:**
```bash
htop

```


3. **Inspect intermediate mapper output (prior to reduce aggregation):**
```bash
find . -type f -name '*.pgn' -print0 | xargs -0 -n4 -P4 awk '/Result/ { split($0, a, "-"); res = substr(a[1], length(a[1]), 1); if (res == 1) white++; if (res == 0) black++; if (res == 2) draw++ } END { print white+black+draw, white, black, draw }'

```
