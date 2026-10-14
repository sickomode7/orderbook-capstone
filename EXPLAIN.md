# Explanation Document — what was built, how, and why

Read this if you need to explain the submission (viva, review, or just
revising). It covers every file, every design decision, and the questions
you're most likely to be asked.

## 1. The assignment in one paragraph

Build a resting order book (bids + asks, no trade matching) three times
with three genuinely different data structures, prove all three agree on
the same workload, benchmark them on 1M ops (70% add / 20% cancel / 10%
BBO + depth snapshots), and explain *why* the numbers differ. The price
rules (tick 0.01, bids 50.00–99.99, asks 100.01–150.00) mean there are only
10,000 possible price levels — that fact is what implementation C exploits.

## 2. Files and what each one does

### `common.hpp` — shared vocabulary

- `Side { Buy, Sell }`, `Level { price, qty }`, `BBO { ... }` — exactly the
  structs from the spec.
- `price_to_tick()` / `tick_to_price()` — the one interface change:
  `round(price * 100)` turns 99.99 into tick 9999. Tick size is fixed at
  0.01 so this conversion is exact (no precision lost). `llround` rounds
  to nearest instead of truncating, so 99.995-style float dust doesn't land
  on the wrong tick.
- Why ticks at all: floats can't be array indices or map keys reliably
  (`100.01` isn't exactly representable in binary), and integer compare is
  cheaper than float compare. If asked "why not just use double as the
  key" — that's the answer.

### `book_a.hpp` — `MapOrderBook` (the baseline)

- Two `std::map<int, long>`: tick → total quantity, one for bids, one for
  asks. `std::map` is a red-black tree: sorted, O(log M) insert/delete.
- One `unordered_map<id, OrderInfo>`: remembers each order's side/tick/qty
  so cancel can find it in O(1). Without this, cancel would have to search
  the tree — this is the "Hint 1" answer.
- `get_bbo()`: best bid = last element (`rbegin`), best ask = first
  (`begin`). No search — the tree maintains order for free.
- `add`: record id, `levels[tick] += qty`. `cancel`: look up id, subtract,
  erase the level if it hits zero (otherwise dead levels pile up and BBO
  reads garbage), return false for unknown/double-cancelled ids.
- If asked "where's the O(log M)": the `map::operator[]`/`find`/`erase` —
  each walks tree height ≈ log₂(levels).

### `book_b.hpp` — `HashScanOrderBook` (the deliberate bad one)

- Same id map, but levels live in `unordered_map<tick, qty>` — no ordering
  maintained at all.
- Every query re-derives order by scanning: `get_bbo` = one full scan per
  side; `get_bids(d)` = d scans ("find best tick below the previous one").
- This is genuinely different from A (hashing vs tree, lazy vs maintained
  order) and it exists to demonstrate the cost of *not* maintaining order:
  adds are O(1) but each BBO scans ~10,000 levels → ~104 µs vs ~24 ns.
- If asked "why is B so slow overall": 10% of 1M ops are BBO queries and
  there are 10,000 depth snapshots × 20 scans each. Queries dominate.

### `book_c.hpp` — `ArrayOrderBook` (own design)

- Core idea: only 10,000 possible ticks, so the tick *is* the index into
  `vector<long> bid_qty_ / ask_qty_` (size 15001, index 0 unused). Level
  lookup = one array read. No hashing, no tree, no per-level `new` ever.
- Same id `unordered_map` for cancels (all three share this — the brief
  says designs must differ in level storage / best-price finding, and they
  do; id lookup is an orthogonal problem and hash-by-id is the right
  answer in all three).
- `best_bid_` / `best_ask_` cache: add updates it with one compare;
  `get_bbo` just reads it (O(1), zero search). Cancel only scans when it
  empties the *exact best* level (`repair_best_bid/ask` walks to the next
  non-empty slot) — rare, and each step of the walk re-finds a level that
  later queries reuse, so it amortizes.
- Memory: 2 × 15001 × 8 bytes ≈ 240 KB — fits in L2/L3, which is the real
  reason it's 2–3× faster than A per op, not just Big-O.
- Limits to admit honestly: only works because the price range is bounded
  and known upfront; sparse usage wastes slots (fine at 10k); repair walk
  is O(range) in a pathological empty-the-book-top-down pattern.

### `bench.cpp` — workload, verify, benchmark in one program

- `build_workload(n, seed)`: generates the op stream *once* (mt19937, seed
  42). 70/20/10 split; cancel picks a uniformly random *live* id; if the
  book is empty a cancel slot becomes an add (can't cancel from nothing).
- `LiveSet`: dense id vector + position map → O(1) random pick and O(1)
  swap-with-last removal. Needed so workload *generation* isn't itself
  O(n²).
- `--verify`: runs the op stream through all three books, compares BBO +
  top-10 bids/asks every 1000 ops (plus edge cases first: empty BBO flags,
  unknown cancel → false, double cancel → false, depth > available →
  returns what's there). Prints PASS/FAIL per impl.
- Benchmark: `steady_clock` around each op, accumulated per type (timing
  each op individually would add noise; accumulating then dividing is the
  standard approach). `volatile sink` + final BBO/checksum print stop the
  compiler deleting "unused" work under `-O2`. `getrusage` gives peak RSS
  + page faults without `perf`.
- Median-of-5: run the binary 5 times, take the middle value (done in shell
  / `make measure`) — median resists one-off machine hiccups better than
  mean.

### `Makefile` — one `g++ -O2 -std=c++17 -Wall` line. `make verify` runs
the check, `make measure` runs all three impls 5×.

## 3. The numbers and the one-line story

Median of 5 runs, 1M ops (701,221 add / 198,995 cancel / 99,784 BBO):

| | A (map) | B (hash+scan) | C (array) |
|---|---|---|---|
| total | ~293 ms | ~18,387 ms | ~137 ms |
| add | ~239 ns | ~207 ns | ~85 ns |
| cancel | ~474 ns | ~589 ns | ~251 ns |
| BBO | ~24 ns | ~103,765 ns | ~24 ns |
| IPC | 0.74 | 0.27 | 0.84 |
| branch miss rate | 4.11% | 0.10% | 1.55% |
| cache misses | ~10M | ~1.60B | ~6M |

Story: C wins adds/cancels (array index beats tree walk and hashing) and
ties A on BBO (both O(1) reads). B's adds are fine but its queries scan
everything, so it loses 100×+ overall — it executes 22B instructions vs
~1.1B. Page faults/RSS are identical across impls (~20k minor, 0 major,
~64 MB) — the id map dominates memory in all three, so faults say nothing
and the report says so honestly.

## 4. Profiling situation (say this exactly if asked)

Real `perf` was installed (`sudo env OMARCHY_ALLOW_DIRECT_PACMAN=1
pacman -S perf` — plain `pacman -Syu` is blocked on Omarchy) and all 8
spec events were measured with `perf stat -r 5`; outputs in
`results/perf_*.txt`. For the `perf record` part, `gprof` was used
(`-pg` build) — outputs in `results/gprof_*.txt`. Key results: B spends
98% of time in the inlined query scans; in C the book is so cheap that
workload setup (25%) outweighs any single book op.

Tricky viva point: **A has the highest branch-miss *rate* (4.11%), not
B.** B's scan loops are long monotonous loops (easy to predict, 0.10%
rate) — B's problem is instruction *count*, not mispredicts. A's tree
comparisons branch on random keys, which the predictor can't learn.

## 5. Likely questions + short answers

- **"Why three designs, not three containers?"** They differ in *strategy*:
  maintained order (tree), no order (scan on demand), and constrained
  domain (index = price). Different storage, different best-price finding.
- **"Why is cancel the slowest op in C?"** It's the only op left doing a
  hash lookup + erase; add is one array write. Fix: open-addressing map or
  id-indexed vector (ids are dense, never reused).
- **"What if prices were unbounded?"** C breaks (can't index infinity) —
  fall back to A; the shared logical interface means callers don't change.
- **"Why median, not mean?"** One noisy run (background process, frequency
  scaling) skews a mean; the median ignores outliers.
- **"Why `volatile sink`?"** Under `-O2`, a query whose result is unused
  can be deleted entirely, benchmarking nothing. Forcing observable use
  keeps the work real.
- **"Empty-book BBO?"** `has_bid = has_ask = false`, prices left at 0 —
  callers must check the flags first.
- **"Multiple orders at one tick?"** Quantities sum in the level;
  individual orders tracked by id for exact cancel subtraction.
- **"Why single-threaded?"** Spec requirement; also keeps the benchmark a
  pure data-structure comparison with no locking noise.
- **"How did you install `perf` on Omarchy?"** `sudo env
  OMARCHY_ALLOW_DIRECT_PACMAN=1 pacman -S perf` — Omarchy blocks bare
  `pacman -Syu` (it wants upgrades via `omarchy update`), but a
  single-package `-S` with the bypass flag works fine.
