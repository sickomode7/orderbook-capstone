# Low-Latency Order Book — Capstone

Name: Tanishka Mangure

Roll Number: 24bcs10264

Single-instrument resting order book (no matching), implemented three ways
and benchmarked with the same 1M-op workload. C++17, stdlib only.

## Files

| File | What it is |
|---|---|
| `common.hpp` | Shared types (`Side`, `Level`, `BBO`) + tick helpers |
| `book_a.hpp` | Impl A: `std::map` levels + `unordered_map` id lookup |
| `book_b.hpp` | Impl B: `unordered_map` levels + linear scan for best price |
| `book_c.hpp` | Impl C: fixed price-indexed arrays + cached best bid/ask |
| `bench.cpp` | One benchmark program (`--impl A/B/C`) + `--verify` correctness check |
| `Makefile` | Build with `g++ -O2 -std=c++17 -Wall` |
| `REPORT.md` | Results table + performance analysis |
| `EXPLAIN.md` | Plain-English explanation of everything (viva prep) |
| `results/` | Saved outputs of verify + one benchmark run per impl |

## Build

```sh
make
```

Needs only `g++` (C++17). No external libraries.

## Correctness check

```sh
./bench --verify
```

Runs the same random workload through A, B and C and compares BBO +
top-10 bids/asks every 1000 ops, plus edge cases (empty book, unknown and
double cancel, shallow depth). Expected output is in `results/verify.txt`.

## Benchmark

```sh
./bench --impl A
./bench --impl B
./bench --impl C
```

1,000,000 ops, seed 42 (70% add / 20% cancel / 10% BBO, depth-10 snapshot
every 100 ops). Flags: `--ops N`, `--seed N`. Each run prints per-op-type
timings, final BBO + checksum (so the work can't be optimized away), and
peak RSS / page faults from `getrusage`. Raw outputs: `results/bench_*.txt`.

## Interface change (documented as required)

Public API keeps `double price`, but internally prices are integer ticks
(`tick = round(price * 100)`). Tick size is fixed at 0.01, so this is
exact — it avoids float rounding problems and makes C's array indexing
possible. Conversion helpers live in `common.hpp`.

## Data-structure choices (short version)

- **A (`std::map`):** the obvious ordered container. Best bid/ask at the
  tree ends, O(log M) updates. Baseline.
- **B (`unordered_map` + scan):** keeps no ordering at all; every BBO/depth
  query scans all live levels. Fast adds, terrible queries — included to
  show what happens when you skip maintaining order.
- **C (price-indexed arrays):** the price range is fixed (10,000 ticks), so
  the tick *is* the array index. O(1) add, cached best bid/ask, no
  per-level allocations (~240 KB total, cache-resident).

Full complexity table + measured numbers + analysis: see `REPORT.md`.

## Profiling with `perf`

`perf` counters (spec's exact event list, 5 runs each); outputs in
`results/perf_*.txt`:

```sh
perf stat -r 5 -e task-clock,cycles,instructions,branches,branch-misses,cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults ./bench --impl A
```

Repeat for B and C. Note `perf stat` prints to stderr, so capture with
`> file 2>&1`. All 8 counters work on this machine (Arch/Omarchy:
`sudo env OMARCHY_ALLOW_DIRECT_PACMAN=1 pacman -S perf`;
`perf_event_paranoid` is already 2, so no extra setup needed).

For the `perf record` part (where time is spent), `gprof` was used:

```sh
g++ -O2 -pg -std=c++17 -o bench_pg bench.cpp
./bench_pg --impl A && gprof bench_pg gmon.out
```

Outputs in `results/gprof_*.txt`.

<!-- roll number / email: TODO - add yours before submitting -->
<!-- submit via PR to https://github.com/KnightKnight27/scaler-HFT-C--2028 -->
