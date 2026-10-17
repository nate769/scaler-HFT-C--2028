# Low-Latency Order Book (C++17)

**Name:** Nathan Joseph Diniz  **Roll number:** 24bcs10342  **SST email:** <your SST email>
**Team (max 2):** <partner name + SST email, or "solo">

## Build

```bash
g++ -std=c++17 -O2 -g -fno-omit-frame-pointer -Wall -Wextra -o bench src/bench.cpp
# or
cmake -S . -B build && cmake --build build   # binary: build/bench
```

## Correctness test

```bash
./bench --verify
```
Runs 200,000 seeded random operations (adds, valid cancels, cancels of unknown/already-cancelled ids, BBO queries)
through A, B and C in lock-step, and after every operation compares add/cancel return values, `get_bbo()`,
`get_bids(10)`, `get_asks(10)` (and the full book every 1000 ops). It also runs edge-case tests (empty book,
unknown cancel, double cancel, insufficient depth, price-band checks, duplicate ids).

## Benchmark

```bash
./bench --impl A      # also B, C
```
Pre-generates 1,000,000 ops (seed 42; 70% add / 20% cancel / 10% BBO; `get_bids(10)` + `get_asks(10)` every 100 ops;
50/50 side, qty 1–100, random valid tick in the side's band, cancels always target a live order). The identical
stream is used for every implementation. Runs 5 times and reports the median. Each run does:
1. a clean pass (no per-call timers) → **total time**
2. an instrumented pass with `steady_clock::now()` around each call → **ns/add, ns/cancel, ns/BBO**
   (the measured cost of one `now()` pair is subtracted and printed).

A checksum of every result is printed so the work cannot be optimised away; it is identical across A/B/C.

Options: `--ops N`, `--seed S`, `--runs R`, `--no-breakdown`.

### perf
```bash
for i in A B C; do
  perf stat -r 5 -e task-clock,cycles,instructions,branches,branch-misses,cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
    ./bench --impl $i --runs 1 --no-breakdown
done
perf record -g ./bench --impl A --runs 1 --no-breakdown && perf report     # repeat for B, C
```
`--runs 1 --no-breakdown` keeps perf focused on one clean pass. Workload generation (~24 MB of ops) is common
to all implementations and contributes equally to page faults.

## Interface changes
- Prices are **integer ticks** (`10001` = 100.01) instead of `double`; `to_ticks()/to_price()/fmt_price()` helpers are in `include/types.hpp`.
- `add_order` returns `bool` (false if qty ≤ 0, price outside its band, or duplicate id). The book is unchanged on rejection.
- Depth `Level.price`, `BBO.*_price` are ticks as well.

## Data-structure choices

| | A | B | C |
|---|---|---|---|
| Price levels | `std::map<tick, qty>` (red-black tree) | sorted `std::vector<Level>`, best at back | flat `long qty[10001]` indexed by tick |
| Orders | `std::unordered_map<id, Order>` | `std::vector<Order>` sorted by id, tombstoned on cancel | slot pool + free list |
| Best bid/ask | `begin()` of map | `back()` of vector | cached `best_bid_/best_ask_`, rescan only if best level empties |
| Find order on cancel | hash lookup | binary search on id | custom open-addressing hash (backward-shift delete) → slot |

L = number of non-empty price levels (≤ 5,000 per side), N = orders.

| Operation | A | B | C |
|---|---|---|---|
| add_order | O(log L) avg + O(1) hash | O(log L + L) shift; order append O(1) if ids increase (O(N) otherwise) | O(1) avg |
| cancel_order | O(log L) avg | O(log N + log L + L) shift | O(1) avg; O(L) worst case on best-level rescan |
| get_bbo | O(1) | O(1) | O(1) |
| get_bids / get_asks | O(depth) | O(depth) | O(depth + empty ticks skipped), worst O(L) |

## Correctness results
<paste output of ./bench --verify>

## Performance results
<paste perf stat output for A, B, C from YOUR machine>

### Results table (median of 5, your machine)

| Metric | A | B | C |
|---|---|---|---|
| Total time (ms) | | | |
| ns / add | | | |
| ns / cancel | | | |
| ns / BBO | | | |
| IPC | | | |
| Branch misses (%) | | | |
| L1-dcache-load-misses | | | |
| Cache misses | | | |
| Page faults | | | |

## Performance analysis (1–2 pages)
<answer 11.1 – 11.6 from your own measurements>
