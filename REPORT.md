# Results + Performance Analysis

Machine: AMD Ryzen 5 6600H, 14 GB RAM, Linux. Built with
`g++ -O2 -std=c++17 -Wall` (g++ 16.2.1). Workload: 1,000,000 ops,
seed 42 → 701,221 adds / 198,995 cancels / 99,784 BBO queries, plus a
depth-10 bid+ask snapshot every 100 ops (10,000 snapshots).

## Correctness

```
Edge cases (empty BBO, unknown/double cancel, shallow depth): PASS
Implementation A: PASS
Implementation B: PASS
Implementation C: PASS
All implementations produce identical results.
```

All three also agree on the final state (identical checksum `304767386`,
final BBO `Bid: 99.99 x 3059 / Ask: 100.01 x 3152`). Full outputs in
`results/`.

## Design summary + complexity

| | A: `std::map` | B: hash + scan | C: price-indexed array |
|---|---|---|---|
| Orders stored | `unordered_map<id, info>` | `unordered_map<id, info>` | `unordered_map<id, info>` |
| Levels stored | `map<tick, qty>` (sorted) | `unordered_map<tick, qty>` | `vector<long>` indexed by tick |
| Best bid/ask found | tree ends (`rbegin`/`begin`) | full scan of all levels | cached variables |
| Order found on cancel | hash lookup by id | hash lookup by id | hash lookup by id |
| `add_order` | O(log M) | O(1) avg | O(1) |
| `cancel_order` | O(log M) | O(1) avg | O(1) avg |
| `get_bbo` | O(1) | O(M) | O(1) |
| `get_bids`/`get_asks` | O(depth) | O(M·depth) | O(depth + skipped empties) |

M = live price levels (up to 10,000 here).

## Results table

Per-op timings: median of 5 `bench` runs (`steady_clock`). Counters: mean
of 5 runs from real `perf stat -r 5` with the spec's exact event list.
Raw outputs: `results/perf_{A,B,C}.txt`.

| Metric | A (map) | B (hash+scan) | C (array) |
|---|---|---|---|
| Total time (ms) | 292.81 | 18387.49 | 136.88 |
| ns / add | 239.4 | 206.8 | 85.0 |
| ns / cancel | 474.4 | 589.0 | 251.0 |
| ns / BBO | 24.4 | 103765.2 (~104 µs) | 23.6 |
| ns / depth snapshot (10+10) | ~330 | ~783,000 (~0.78 ms) | ~178 |
| task-clock (ms) | 418.89 | 18555.43 | 355.47 |
| cycles | 1,671,325,866 | 80,903,192,991 | 1,354,690,528 |
| instructions | 1,231,510,976 | 22,203,959,932 | 1,143,051,092 |
| IPC | 0.74 | 0.27 | 0.84 |
| branches | 230,750,507 | 5,102,223,627 | 202,628,648 |
| branch-misses | 9,474,510 | 5,217,853 | 3,135,574 |
| Branch misses (%) | 4.11 | 0.10 | 1.55 |
| L1-dcache-load-misses | 22,501,248 | 2,646,565,132 | 12,550,277 |
| Cache misses | 10,476,146 | 1,599,523,977 | 6,095,288 |
| dTLB-load-misses | 1,257,563 | 2,568,373 | 1,165,261 |
| Page faults | 20,071 | 20,012 | 19,972 |
| Peak RSS (KB) | 64,320 | 64,416 | 64,496 |

Note: task-clock (whole process incl. workload generation) is higher than
the benchmark's own total (ops loop only) — that gap is workload setup,
and `gprof` confirms it (see below).

## Hotspot profile (`gprof`, stands in for `perf record`)

`results/gprof_{A,B,C}.txt`. Summary:

- **A:** `add_order` 36%, `cancel_order` 24% of runtime — the tree ops are
  the program. Workload generation is 27%.
- **B:** 98% of time inside `run_bench` itself — the scan loops got
  inlined into the benchmark loop, which *is* the finding: the program is
  one giant query scan. `add`/`cancel` are each under 0.3%.
- **C:** book ops shrink to `add` 10% + `cancel` 15% (+15% id-hash insert);
  workload generation (25%) is now the biggest single item. The order book
  got so cheap that test setup costs more than the thing tested.

## 11.1 Which implementation is fastest?

- **`add_order`: C (85 ns)** — one hash insert + one array write, no tree
  rebalancing, no per-level allocation. A pays for red-black
  insert (~239 ns); B's add is also just a hash insert (~207 ns) but
  hashing + bucket pointer chasing costs more than C's direct index.
- **`cancel_order`: C (251 ns)** — hash lookup + array write; the
  cached-best repair walk almost never triggers (only when the cancelled
  order empties the exact best level). A pays tree delete (~474 ns);
  B pays two hash ops plus erase (~589 ns).
- **`get_bbo`: A and C tie (~24 ns)** — both are pointer/variable reads.
  B takes ~104 µs because it scans all ~10,000 live levels per query, and
  BBO queries are 10% of the workload, so B's total time (18.4 s) is
  ~135× worse than C's. B's depth snapshot is even worse (~0.78 ms: 20
  full scans per snapshot).

Why C wins overall: fewest instructions per op (1.14B vs A's 1.22B vs B's
22.2B — B executes ~20× more instructions), least memory touched (240 KB
of arrays stay cache-resident), zero per-level allocations.

## 11.2 Branch prediction

**A has the most branch misses: 9.47M (4.11% miss rate)**, then B (5.22M,
but only 0.10%), then C (3.14M, 1.55%). The surprise — B has the *lowest*
miss rate — makes sense on inspection: B's scan loops are long,
monotonous loops whose back-edge branch is taken 10,000× in a row; the
predictor loves that. A's misses come from red-black tree comparisons:
each insert/delete walks ~log M nodes with data-dependent
go-left/go-right branches on essentially random keys — the hardest pattern
to predict. C's hot path is nearly branchless (array index + one min/max
compare), hence the lowest absolute count.

## 11.3 Cache behavior

**B is by far the worst: 1.60B cache misses and 2.65B L1 misses** —
~150× A's and ~260× C's. Every query streams whole hash tables (bucket
array + nodes scattered across the heap): ~10k pointer hops per BBO,
~200k per depth snapshot, and nothing is reused between queries. Its IPC
of 0.27 shows the CPU stalled on memory most of the time.

**A (10.5M misses, IPC 0.74):** each level is a separately allocated tree
node, so inserts/deletes chase pointers to random heap addresses. Fine at
this size, but every op touches several cache lines across nodes.

**C (6.1M misses, IPC 0.84 — best):** contiguous `vector<long>` slots, one
hash lookup per mutation, sequential access in repair walks and depth
snapshots. The 240 KB of level arrays fits in L2/L3. This is why C beats A
2–3× per op even where both are "O(1)-ish": the constant is cache lines,
not Big-O.

## 11.4 Page faults

All three: ~20k faults, 0 major, ~64 MB peak RSS — practically identical.
The book itself is small; everything is first-touched at startup (minor
faults = fresh page mapping) and nothing is paged out on a 14 GB machine.
The id `unordered_map` (~500k live orders at peak) dominates memory in all
three designs equally, which is why RSS barely differs. Conclusion: the
designs differ in *access pattern*, not footprint — faults say nothing
beyond "allocation behavior is equivalent."

## 11.5 Production choice

**C**, with one reservation. Latency: best on every op type, and more
importantly the most *predictable* — add/BBO/depth are flat O(1) array
ops with the lowest miss counts, while A's tree ops vary with depth and B
degrades as the book fills. Memory: fixed ~240 KB + the id map every
design needs; zero per-level allocation means zero allocator jitter on
the hot path. Complexity/maintainability is the trade-off: tick-indexing
plus best-repair logic is subtler than `std::map`, and only works because
the price range is bounded and known upfront. For this instrument (fixed
tick bands) that constraint is given, so C is the right call. If the book
ever needed unbounded prices, A would be the fallback — same logical
interface, no caller changes.

## 11.6 What would you try next?

1. **Integer ticks in the public API too** — internals already use ticks;
   pushing them outward kills the `llround` per add.
2. **Faster id lookup** — `gprof` shows the id hash insert is now 15% of
   C's runtime and cancel (251 ns) is its slowest op. Ids are dense and
   never reused, so a growing `vector` indexed by id would make lookup
   nearly free.
3. **Pre-allocation** — `reserve()` the id map (~700k adds known upfront)
   to avoid rehash pauses; same for workload vectors.
4. **Combined top-of-book call** — `get_bids`/`get_asks` walk adjacent
   regions; one call returning both sides halves the passes.
5. **Huge pages / pinning** — only if major faults ever appear; they
   don't here, so this stays last.
