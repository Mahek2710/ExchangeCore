# ExchangeCore
### A Low-Latency Limit Order Book & Matching Engine in C++

> Microsecond-level order matching. Exchange-grade data structures. Built from scratch.

---

## What is this?

ExchangeCore is a high-performance limit order book engine that replicates the core infrastructure of a real financial exchange. It ingests a simulated market feed over UDP, matches buy and sell orders using strict price-time priority, and tracks PnL — all with nanosecond-precision latency benchmarking on Linux.

This is not a toy. The architecture mirrors what production HFT systems use: a red-black tree of price levels, a FIFO queue of orders per level, and an O(1) cancellation path via HashMap.

---

## Performance

| Metric | Value |
|---|---|
| Throughput | 1M+ orders/sec |
| Median latency (p50) | < 1 µs |
| p99 latency | < 5 µs |
| Cancel complexity | O(1) |
| Insert complexity | O(log P) new price level / O(1) existing |
| Match complexity | O(log P) |

*Benchmarked on Linux x86-64. P = number of distinct price levels.*

---

## Architecture

```
UDP Feed (simulated exchange)
        │
        ▼
┌───────────────────┐
│   Feed Handler    │  ← Parses raw order messages over POSIX UDP socket
└────────┬──────────┘
         │
         ▼
┌───────────────────────────────────────────────┐
│                  Order Book                   │
│                                               │
│  BID SIDE                   ASK SIDE          │
│  std::map (desc)            std::map (asc)    │
│  ┌─────────────────┐        ┌───────────────┐ │
│  │ 101.00 → [A][B] │        │ 102.00 → [C] │ │
│  │ 100.50 → [D]    │        │ 103.00 → [E] │ │
│  └─────────────────┘        └───────────────┘ │
│                                               │
│  unordered_map<order_id, OrderNode*>          │
│  └── O(1) pointer into DLL for cancels        │
└────────┬──────────────────────────────────────┘
         │
         ▼
┌───────────────────┐
│  Matching Engine  │  ← Price-time priority, handles GTC / IOC / FOK / Market
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   PnL Tracker     │  ← Fill price × qty, realised vs unrealised
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Latency Logger   │  ← clock_gettime, outputs p50/p99 histogram to CSV
└───────────────────┘
```

---

## Data Structures

### Price Level Map — `std::map`
- Bids sorted **descending** (best bid = highest price = `begin()`)
- Asks sorted **ascending** (best ask = lowest price = `begin()`)
- Insert/delete a price level: **O(log P)**
- Best bid/ask access: **O(1)**

### Per-Level Order Queue — Doubly Linked List
- FIFO within each price level (time priority)
- Append: **O(1)**
- Remove by pointer: **O(1)**

### Order Lookup — `unordered_map<uint64_t, OrderNode*>`
- Maps order ID directly to a pointer into the DLL
- Cancel by ID: **O(1)** — no scan needed

---

## Order Types

| Type | Behaviour |
|---|---|
| `GoodTillCancel` | Rests in book until filled or explicitly cancelled |
| `ImmediateOrCancel` | Fills whatever it can instantly, cancels the rest |
| `FillOrKill` | Must fill completely or not at all |
| `Market` | Converted to limit at extreme price, reuses matching logic |

---

## Matching Logic

Price-time priority:
1. Best price is matched first (highest bid vs lowest ask)
2. At equal price, earliest order in queue fills first
3. Partial fills are supported — remaining quantity rests in book (GTC) or is cancelled (IOC/FOK)

```cpp
// Simplified hot path
while (!asks_.empty() && bid.price >= asks_.begin()->first) {
    auto& [ask_price, ask_level] = *asks_.begin();
    fill(bid, ask_level);
    if (ask_level.empty()) asks_.erase(asks_.begin());
}
```

---

## Feed Simulation

ExchangeCore listens on a UDP socket for a binary order message stream. A companion feed generator sends synthetic order flow modelled on real market microstructure (random arrivals, price clustering near mid).

```
Message format (32 bytes):
┌────────┬────────┬────────┬────────┬────────┬────────┐
│ msg_type│order_id│  side  │  price │  qty   │  flags │
│  1 byte │ 8 bytes│ 1 byte │ 8 bytes│ 8 bytes│ 6 bytes│
└────────┴────────┴────────┴────────┴────────┴────────┘
```

---

## Latency Benchmarking

Every order processed on the hot path is timed with `clock_gettime(CLOCK_MONOTONIC)`. Results are written to `bench/latency.csv`:

```
percentile, latency_ns
p50,        420
p90,        810
p99,        3200
p99.9,      9100
```

Run the benchmark:
```bash
./build/bench --orders 1000000 --output bench/latency.csv
```

---

## PnL Tracker

Tracks fills in real time:

- **Realised PnL** — closed positions (buy filled then sell filled)
- **Unrealised PnL** — open positions marked to current mid price
- Per-instrument breakdown

---

## Build

**Requirements:** GCC 11+ or Clang 14+, CMake 3.20+, Linux

```bash
git clone https://github.com/yourusername/exchangecore.git
cd exchangecore
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

Compiler flags used for the hot path:
```
-O3 -march=native -fno-exceptions -fno-rtti
```

---

## Run

**Start the matching engine:**
```bash
./build/exchangecore --port 9001
```

**Start the feed generator (separate terminal):**
```bash
./build/feed_gen --host 127.0.0.1 --port 9001 --rate 500000
```

**Run benchmarks:**
```bash
./build/bench --orders 1000000
```

**Run tests:**
```bash
ctest --test-dir build -v
```

---

## Project Structure

```
exchangecore/
├── include/
│   ├── order.hpp           # Order, OrderType, Side definitions
│   ├── price_level.hpp     # Doubly linked list per price level
│   ├── order_book.hpp      # Core book: std::map + unordered_map
│   ├── matching_engine.hpp # Matching logic, order type handling
│   ├── feed_handler.hpp    # UDP socket ingestion
│   ├── pnl_tracker.hpp     # Realised/unrealised PnL
│   └── latency_logger.hpp  # clock_gettime benchmarking
├── src/
│   ├── order_book.cpp
│   ├── matching_engine.cpp
│   ├── feed_handler.cpp
│   ├── pnl_tracker.cpp
│   └── latency_logger.cpp
├── tools/
│   ├── bench.cpp           # Standalone benchmark tool
│   └── feed_gen.cpp        # Synthetic UDP feed generator
├── tests/
│   ├── test_matching.cpp   # Price-time priority correctness
│   ├── test_cancel.cpp     # O(1) cancel correctness
│   ├── test_order_types.cpp# IOC, FOK, GTC edge cases
│   └── test_pnl.cpp        # Fill accounting correctness
├── bench/
│   └── latency.csv         # Benchmark output
├── CMakeLists.txt
└── README.md
```

---

## Tests

Tests use synthetic order sequences with known expected outcomes — not just "does it run" but "does order A fill before order B at the same price level":

```bash
ctest --test-dir build -v

# Example assertions:
# - Two bids at same price: earlier one fills first
# - IOC order: unfilled portion is never added to book
# - FOK order: zero partial fills, all-or-nothing
# - Cancel: order_id removed from book and lookup map
```

---

## Stack

| Layer | Technology |
|---|---|
| Language | C++17 |
| OS | Linux (Ubuntu 22.04) |
| Networking | POSIX UDP sockets |
| Threading | `std::thread`, `std::mutex`, `std::atomic` |
| Build | CMake |
| Timing | `clock_gettime(CLOCK_MONOTONIC)` |
| Testing | Catch2 |
| Profiling | `perf`, `valgrind --tool=callgrind` |

---

## Why this matters

Every exchange in the world runs something like this at its core. The latency between an order arriving and a trade executing is measured in microseconds — sometimes nanoseconds. The data structures chosen here are not academic: red-black trees, FIFO queues, and hash maps are the actual building blocks of production matching engines at NSE, BSE, CME, and every HFT firm's internal infrastructure.

---

*Built by Mahek Hingorani*
# ExchangeCore
### A Low-Latency Limit Order Book & Matching Engine in C++

> Microsecond-level order matching. Exchange-grade data structures. Built from scratch.

---

## What is this?

ExchangeCore is a high-performance limit order book engine that replicates the core infrastructure of a real financial exchange. It ingests a simulated market feed over UDP, matches buy and sell orders using strict price-time priority, and tracks PnL — all with nanosecond-precision latency benchmarking on Linux.

This is not a toy. The architecture mirrors what production HFT systems use: a red-black tree of price levels, a FIFO queue of orders per level, and an O(1) cancellation path via HashMap.

---

## Performance

| Metric | Value |
|---|---|
| Throughput | 1M+ orders/sec |
| Median latency (p50) | < 1 µs |
| p99 latency | < 5 µs |
| Cancel complexity | O(1) |
| Insert complexity | O(log P) new price level / O(1) existing |
| Match complexity | O(log P) |

*Benchmarked on Linux x86-64. P = number of distinct price levels.*

---

## Architecture

```
UDP Feed (simulated exchange)
        │
        ▼
┌───────────────────┐
│   Feed Handler    │  ← Parses raw order messages over POSIX UDP socket
└────────┬──────────┘
         │
         ▼
┌───────────────────────────────────────────────┐
│                  Order Book                   │
│                                               │
│  BID SIDE                   ASK SIDE          │
│  std::map (desc)            std::map (asc)    │
│  ┌─────────────────┐        ┌───────────────┐ │
│  │ 101.00 → [A][B] │        │ 102.00 → [C] │ │
│  │ 100.50 → [D]    │        │ 103.00 → [E] │ │
│  └─────────────────┘        └───────────────┘ │
│                                               │
│  unordered_map<order_id, OrderNode*>          │
│  └── O(1) pointer into DLL for cancels        │
└────────┬──────────────────────────────────────┘
         │
         ▼
┌───────────────────┐
│  Matching Engine  │  ← Price-time priority, handles GTC / IOC / FOK / Market
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   PnL Tracker     │  ← Fill price × qty, realised vs unrealised
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Latency Logger   │  ← clock_gettime, outputs p50/p99 histogram to CSV
└───────────────────┘
```

---

## Data Structures

### Price Level Map — `std::map`
- Bids sorted **descending** (best bid = highest price = `begin()`)
- Asks sorted **ascending** (best ask = lowest price = `begin()`)
- Insert/delete a price level: **O(log P)**
- Best bid/ask access: **O(1)**

### Per-Level Order Queue — Doubly Linked List
- FIFO within each price level (time priority)
- Append: **O(1)**
- Remove by pointer: **O(1)**

### Order Lookup — `unordered_map<uint64_t, OrderNode*>`
- Maps order ID directly to a pointer into the DLL
- Cancel by ID: **O(1)** — no scan needed

---

## Order Types

| Type | Behaviour |
|---|---|
| `GoodTillCancel` | Rests in book until filled or explicitly cancelled |
| `ImmediateOrCancel` | Fills whatever it can instantly, cancels the rest |
| `FillOrKill` | Must fill completely or not at all |
| `Market` | Converted to limit at extreme price, reuses matching logic |

---

## Matching Logic

Price-time priority:
1. Best price is matched first (highest bid vs lowest ask)
2. At equal price, earliest order in queue fills first
3. Partial fills are supported — remaining quantity rests in book (GTC) or is cancelled (IOC/FOK)

```cpp
// Simplified hot path
while (!asks_.empty() && bid.price >= asks_.begin()->first) {
    auto& [ask_price, ask_level] = *asks_.begin();
    fill(bid, ask_level);
    if (ask_level.empty()) asks_.erase(asks_.begin());
}
```

---

## Feed Simulation

ExchangeCore listens on a UDP socket for a binary order message stream. A companion feed generator sends synthetic order flow modelled on real market microstructure (random arrivals, price clustering near mid).

```
Message format (32 bytes):
┌────────┬────────┬────────┬────────┬────────┬────────┐
│ msg_type│order_id│  side  │  price │  qty   │  flags │
│  1 byte │ 8 bytes│ 1 byte │ 8 bytes│ 8 bytes│ 6 bytes│
└────────┴────────┴────────┴────────┴────────┴────────┘
```

---

## Latency Benchmarking

Every order processed on the hot path is timed with `clock_gettime(CLOCK_MONOTONIC)`. Results are written to `bench/latency.csv`:

```
percentile, latency_ns
p50,        420
p90,        810
p99,        3200
p99.9,      9100
```

Run the benchmark:
```bash
./build/bench --orders 1000000 --output bench/latency.csv
```

---

## PnL Tracker

Tracks fills in real time:

- **Realised PnL** — closed positions (buy filled then sell filled)
- **Unrealised PnL** — open positions marked to current mid price
- Per-instrument breakdown

---

## Build

**Requirements:** GCC 11+ or Clang 14+, CMake 3.20+, Linux

```bash
git clone https://github.com/yourusername/exchangecore.git
cd exchangecore
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

Compiler flags used for the hot path:
```
-O3 -march=native -fno-exceptions -fno-rtti
```

---

## Run

**Start the matching engine:**
```bash
./build/exchangecore --port 9001
```

**Start the feed generator (separate terminal):**
```bash
./build/feed_gen --host 127.0.0.1 --port 9001 --rate 500000
```

**Run benchmarks:**
```bash
./build/bench --orders 1000000
```

**Run tests:**
```bash
ctest --test-dir build -v
```

---

## Project Structure

```
exchangecore/
├── include/
│   ├── order.hpp           # Order, OrderType, Side definitions
│   ├── price_level.hpp     # Doubly linked list per price level
│   ├── order_book.hpp      # Core book: std::map + unordered_map
│   ├── matching_engine.hpp # Matching logic, order type handling
│   ├── feed_handler.hpp    # UDP socket ingestion
│   ├── pnl_tracker.hpp     # Realised/unrealised PnL
│   └── latency_logger.hpp  # clock_gettime benchmarking
├── src/
│   ├── order_book.cpp
│   ├── matching_engine.cpp
│   ├── feed_handler.cpp
│   ├── pnl_tracker.cpp
│   └── latency_logger.cpp
├── tools/
│   ├── bench.cpp           # Standalone benchmark tool
│   └── feed_gen.cpp        # Synthetic UDP feed generator
├── tests/
│   ├── test_matching.cpp   # Price-time priority correctness
│   ├── test_cancel.cpp     # O(1) cancel correctness
│   ├── test_order_types.cpp# IOC, FOK, GTC edge cases
│   └── test_pnl.cpp        # Fill accounting correctness
├── bench/
│   └── latency.csv         # Benchmark output
├── CMakeLists.txt
└── README.md
```

---

## Tests

Tests use synthetic order sequences with known expected outcomes — not just "does it run" but "does order A fill before order B at the same price level":

```bash
ctest --test-dir build -v

# Example assertions:
# - Two bids at same price: earlier one fills first
# - IOC order: unfilled portion is never added to book
# - FOK order: zero partial fills, all-or-nothing
# - Cancel: order_id removed from book and lookup map
```

---

## Stack

| Layer | Technology |
|---|---|
| Language | C++17 |
| OS | Linux (Ubuntu 22.04) |
| Networking | POSIX UDP sockets |
| Threading | `std::thread`, `std::mutex`, `std::atomic` |
| Build | CMake |
| Timing | `clock_gettime(CLOCK_MONOTONIC)` |
| Testing | Catch2 |
| Profiling | `perf`, `valgrind --tool=callgrind` |

---

## Why this matters

Every exchange in the world runs something like this at its core. The latency between an order arriving and a trade executing is measured in microseconds — sometimes nanoseconds. The data structures chosen here are not academic: red-black trees, FIFO queues, and hash maps are the actual building blocks of production matching engines at NSE, BSE, CME, and every HFT firm's internal infrastructure.

---

*Built by Mahek Hingorani*
