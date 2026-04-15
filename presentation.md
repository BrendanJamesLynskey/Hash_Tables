# Hash Tables

**Computer Science Fundamentals Series**

Hash functions · Collision resolution · Open addressing · Cuckoo hashing · Load factor · Concurrent maps

*Mid-level software engineer track -- 20 slides*

---

## Table of Contents

1. [Dictionary / Map ADT](#slide-02--dictionary--map-adt)
2. [Direct Addressing](#slide-03--direct-addressing)
3. [Hash Functions](#slide-04--hash-functions)
4. [Universal Hashing](#slide-05--universal-hashing)
5. [Separate Chaining](#slide-06--separate-chaining)
6. [Open Addressing -- Linear & Quadratic Probing](#slide-07--open-addressing--linear--quadratic-probing)
7. [Double Hashing](#slide-08--double-hashing)
8. [Load Factor & Performance](#slide-09--load-factor--performance)
9. [Resizing & Rehashing](#slide-10--resizing--rehashing)
10. [Robin Hood Hashing](#slide-11--robin-hood-hashing)
11. [Cuckoo Hashing](#slide-12--cuckoo-hashing)
12. [Hopscotch Hashing](#slide-13--hopscotch-hashing)
13. [Swiss Table](#slide-14--swiss-table)
14. [Perfect Hashing](#slide-15--perfect-hashing)
15. [Hash Tables in Practice -- Python & Java](#slide-16--hash-tables-in-practice--python--java)
16. [Hash Tables in Practice -- Go & C++](#slide-17--hash-tables-in-practice--go--c)
17. [Thread-Safe Hash Maps](#slide-18--thread-safe-hash-maps)
18. [Applications & Limitations](#slide-19--applications--limitations)
19. [Summary & Further Reading](#slide-20--summary--further-reading)

---

## Slide 02 -- Dictionary / Map ADT

### The abstract interface

A dictionary (also called map, associative array) supports three core operations on key-value pairs:

- **INSERT(k, v)** -- associate value `v` with key `k`
- **SEARCH(k)** -- return the value associated with key `k`, or report absent
- **DELETE(k)** -- remove the pair with key `k`

### Naive implementations

| Data Structure | INSERT | SEARCH | DELETE |
|---------------|--------|--------|--------|
| Unsorted array | `O(1)` | `O(n)` | `O(n)` |
| Sorted array | `O(n)` | `O(log n)` | `O(n)` |
| Balanced BST | `O(log n)` | `O(log n)` | `O(log n)` |
| **Hash table** | **O(1) expected** | **O(1) expected** | **O(1) expected** |

> Hash tables achieve amortised constant time by trading deterministic guarantees for expected-case performance. The key insight: convert the key to an array index via a hash function.

---

## Slide 03 -- Direct Addressing

### When keys are small integers

If every possible key is in `{0, 1, ..., U-1}` and `U` is small, store values in a plain array `T[0..U-1]`.

```
T[k] = v       -- INSERT
T[k]           -- SEARCH
T[k] = NIL     -- DELETE
```

All operations are `O(1)` worst case.

### The problem

- When `U` is large (e.g. 64-bit integers, strings), allocating an array of size `U` is infeasible
- When only `n << U` keys are actually used, most of the array is wasted
- Solution: **hash the key** into a smaller index range `[0, m-1]` where `m = O(n)`

> Direct addressing is the conceptual foundation. A hash table generalises it by compressing the key space.

---

## Slide 04 -- Hash Functions

### Requirements

A hash function `h(k)` maps a key from a universe `U` to an index in `{0, 1, ..., m-1}`.

- **Deterministic** -- same key always produces the same index
- **Uniform distribution** -- keys should spread evenly across slots
- **Fast to compute** -- ideally `O(1)` or `O(|key|)` for variable-length keys

### Division method

```
h(k) = k mod m
```

Simple but sensitive to choice of `m`. Avoid powers of 2 (only low-order bits used). Choose `m` as a prime not close to a power of 2.

### Multiplication method

```
h(k) = floor(m * (k * A mod 1))       where 0 < A < 1
```

Knuth suggests `A = (sqrt(5) - 1) / 2 ≈ 0.6180339887`. Less sensitive to choice of `m`.

### Practical hash functions

| Function | Use case |
|----------|----------|
| **MurmurHash3** | General-purpose, fast, good distribution |
| **xxHash** | Extremely fast, widely used in checksums |
| **SipHash** | Hash-flooding resistant -- default in Python, Rust |
| **FNV-1a** | Simple, byte-at-a-time, good for short keys |
| **CityHash / FarmHash** | Google's fast hashes for strings |

---

## Slide 05 -- Universal Hashing

### The adversary problem

Any single deterministic hash function has a worst case: an adversary can craft `n` keys that all hash to the same slot, degrading every operation to `O(n)`.

### Universal hash families

A family `H` of hash functions is **universal** if for any two distinct keys `x ≠ y`:

```
Pr[h(x) = h(y)] ≤ 1/m       where h is chosen randomly from H
```

### Carter-Wegman construction

For prime `p ≥ |U|`:

```
h_{a,b}(k) = ((a * k + b) mod p) mod m
```

where `a ∈ {1, ..., p-1}` and `b ∈ {0, ..., p-1}` are chosen randomly.

- Expected chain length is `n/m` regardless of key distribution
- No adversary can force worst-case behaviour without knowing `a, b`

> **Cryptographic hashes** (SHA-256) provide stronger guarantees but are much slower. Use them for security, not hash tables.

---

## Slide 06 -- Separate Chaining

### Collision resolution with linked lists

Each slot `T[i]` points to a linked list of all elements whose keys hash to `i`.

```
T[0] -> [k3,v3] -> [k7,v7] -> NIL
T[1] -> NIL
T[2] -> [k1,v1] -> NIL
T[3] -> [k5,v5] -> [k9,v9] -> [k2,v2] -> NIL
T[4] -> [k8,v8] -> NIL
```

### Operations

- **INSERT** -- compute `h(k)`, prepend to list at `T[h(k)]` -- `O(1)`
- **SEARCH** -- compute `h(k)`, traverse list at `T[h(k)]` -- `O(chain length)`
- **DELETE** -- search then unlink -- `O(chain length)`

### Performance

With load factor `α = n/m` and a good hash function:

- Expected chain length = `α`
- Expected search time = `O(1 + α)`
- If `n = O(m)`, all operations are `O(1)` expected

> Chaining is simple and tolerates `α > 1` gracefully. Downside: pointer chasing is cache-unfriendly. Each node is a separate heap allocation.

---

## Slide 07 -- Open Addressing -- Linear & Quadratic Probing

### Open addressing principle

All elements stored directly in the table array. No pointers, no linked lists. On collision, **probe** a sequence of alternative slots until an empty one is found.

### Linear probing

```
h(k, i) = (h'(k) + i) mod m       for i = 0, 1, 2, ...
```

Slots are checked sequentially. Simple and cache-friendly but suffers from **primary clustering** -- runs of occupied slots grow and merge.

### Quadratic probing

```
h(k, i) = (h'(k) + c1*i + c2*i^2) mod m
```

Spreads probes more than linear probing. Avoids primary clustering but can cause **secondary clustering** -- keys with the same initial hash follow the same probe sequence.

### Comparison

| Method | Cache behaviour | Clustering | Table coverage |
|--------|----------------|------------|---------------|
| **Linear probing** | Excellent | Primary clustering | Visits all slots |
| **Quadratic probing** | Good | Secondary clustering | May miss slots if `m` not prime |

> Linear probing is used in practice more than theory suggests -- modern CPUs love sequential memory access. Swiss Table proves this decisively.

---

## Slide 08 -- Double Hashing

### Two independent hash functions

```
h(k, i) = (h1(k) + i * h2(k)) mod m
```

where `h2(k)` must never return 0 (otherwise the probe sequence degenerates to a single slot).

Common choice: `h2(k) = 1 + (k mod (m - 1))` with `m` prime.

### Why it works

- Each key generates a **distinct probe sequence** -- no clustering
- Approximates the ideal of uniform probing
- Expected probes for unsuccessful search: `1 / (1 - α)`
- Expected probes for successful search: `(1/α) * ln(1 / (1 - α))`

### Probe count at various load factors

| Load factor α | Unsuccessful search | Successful search |
|--------------|-------------------|-----------------|
| 0.50 | 2.0 probes | 1.4 probes |
| 0.75 | 4.0 probes | 1.8 probes |
| 0.90 | 10.0 probes | 2.6 probes |
| 0.95 | 20.0 probes | 3.2 probes |

> As `α → 1`, performance degrades sharply. Open-addressing tables should resize well before reaching `α = 1` -- typically at `α ∈ [0.5, 0.75]`.

---

## Slide 09 -- Load Factor & Performance

### Load factor

```
α = n / m       (number of stored elements / table capacity)
```

- **Chaining:** `α` can exceed 1.0. Performance degrades linearly: expected search = `O(1 + α)`
- **Open addressing:** `α` must stay below 1.0. Performance degrades non-linearly

### Expected chain length (chaining with universal hashing)

For `n` keys in `m` slots, the expected longest chain is `Θ(log n / log log n)`.

### Amortised O(1) argument

If we maintain `α ≤ c` for some constant `c < 1` by doubling the table when `α` exceeds the threshold:

- Each insertion is `O(1)` expected
- Resizing costs `O(n)` but happens only every `O(n)` insertions
- **Amortised cost per insertion:** `O(1)`

### Space-time trade-off

| Load factor target | Memory overhead | Probe performance |
|-------------------|----------------|------------------|
| `α ≤ 0.50` | 2x actual data | Excellent |
| `α ≤ 0.75` | 1.33x actual data | Good |
| `α ≤ 0.90` | 1.11x actual data | Acceptable (chaining) / poor (open addr) |

> Most real-world hash tables target `α ∈ [0.5, 0.75]`. Python's dict resizes at `α = 2/3`.

---

## Slide 10 -- Resizing & Rehashing

### When to resize

Typically when `α` exceeds a threshold (e.g. 0.75). Some implementations also shrink when `α` drops below a lower bound (e.g. 0.25).

### The rehashing process

1. Allocate a new table of size `m' = 2m` (or next prime > `2m`)
2. For every element in the old table, compute `h(k) mod m'` and insert into the new table
3. Free the old table

Cost: `O(n)` -- must rehash every element because the modulus changed.

### Incremental rehashing (Redis approach)

Instead of rehashing all `n` keys at once (which causes a latency spike):

- Maintain **two tables** simultaneously
- On each operation, migrate a few keys from the old table to the new
- Lookups check both tables during the transition
- After all keys are migrated, free the old table

### Growth strategies

| Strategy | New size | Amortised insert |
|----------|---------|-----------------|
| **Doubling** | `2m` | `O(1)` |
| **1.5x growth** | `1.5m` | `O(1)` (lower constant) |
| **Prime stepping** | next prime > `2m` | `O(1)`, better distribution with division hashing |

> Shrinking is often omitted in practice -- the memory cost of an oversized table is usually acceptable, and shrink-grow oscillation near the threshold is expensive.

---

## Slide 11 -- Robin Hood Hashing

### Equalising probe distances

A variant of open-addressing (typically linear probing) where the **probe distance** of each element is tracked.

### The Robin Hood rule

During insertion, if the new element's probe distance exceeds the probe distance of the element currently occupying a slot, **swap them** -- "steal from the rich, give to the poor."

```
Insert key K with probe distance d:
  While slot is occupied:
    If d > occupant's probe distance:
      Swap K with occupant
      Continue inserting the displaced element
    Advance to next slot, increment d
  Place K in empty slot
```

### Benefits

- **Variance of probe distances is dramatically reduced** -- all elements have similar probe lengths
- Expected maximum probe distance: `O(log log n)` vs `O(log n)` for standard linear probing
- Lookup can terminate early: if current probe distance exceeds the occupant's, the key is absent
- Deletion is simpler than standard open addressing -- backward-shift instead of tombstones

> Robin Hood hashing makes linear probing practical at higher load factors. Rust's `HashMap` used Robin Hood hashing (before switching to `hashbrown` / Swiss Table).

---

## Slide 12 -- Cuckoo Hashing

### Worst-case O(1) lookup

Uses **two hash functions** `h1` and `h2` and two tables `T1`, `T2`.

- **SEARCH:** check `T1[h1(k)]` and `T2[h2(k)]` -- exactly 2 lookups, always
- **DELETE:** check both locations, remove -- `O(1)` worst case

### Insertion algorithm

```
Insert key k:
  If T1[h1(k)] is empty -> place k there, done.
  Evict occupant x from T1[h1(k)], place k.
  If T2[h2(x)] is empty -> place x there, done.
  Evict occupant y from T2[h2(x)], place x.
  ... repeat (with a max iteration limit)
  If cycle detected -> rehash with new functions.
```

### Properties

| Property | Value |
|----------|-------|
| **Lookup** | `O(1)` worst case |
| **Deletion** | `O(1)` worst case |
| **Insertion** | `O(1)` amortised (with rehash) |
| **Max load factor** | ~50% (two tables) or higher with buckets |
| **Rehash trigger** | Insertion cycle detected |

> Cuckoo hashing guarantees constant-time lookups, making it ideal for network routers, hardware implementations, and read-heavy workloads.

---

## Slide 13 -- Hopscotch Hashing

### Bounded neighbourhood

Each slot has a "neighbourhood" of `H` consecutive slots (typically `H = 32`). An element hashing to slot `i` must reside within `i` to `i + H - 1`.

### Lookup

Check all `H` slots in the neighbourhood -- a single cache line or two. Uses a bitmap per slot to record which of the `H` neighbours are occupied by elements hashing to that slot.

### Insertion

1. Find an empty slot (may be beyond the neighbourhood)
2. Repeatedly swap elements to move the empty slot closer, into the neighbourhood
3. If no swap chain can bring the empty slot within range, resize

### Advantages over linear probing

- **Bounded worst-case probe distance:** at most `H` slots
- **Cache-friendly:** neighbourhood fits in 1--2 cache lines
- **Concurrency-friendly:** lock only the neighbourhood, not the whole table
- Deletion needs no tombstones -- just clear the bitmap bit and the slot

> Hopscotch hashing combines the cache-friendliness of linear probing with a bounded probe guarantee. Used in Cliff Click's lock-free concurrent hash map.

---

## Slide 14 -- Swiss Table

### Google's flat_hash_map

Open-addressing with linear probing, but with SIMD-accelerated metadata lookups. Used in Abseil (`absl::flat_hash_map`) and Rust's `hashbrown`.

### Architecture

- Table divided into **groups** of 16 slots
- Each group has a 16-byte **control byte array** (one byte per slot)
- Control byte stores the top 7 bits of the hash (H2) + 1 bit for empty/deleted/full
- Probe sequence walks over groups, not individual slots

### SIMD-powered lookup

```
1. Compute h1 = hash(key)
2. Group index = h1 mod (num_groups)
3. H2 = top 7 bits of hash
4. Load 16 control bytes into a SIMD register
5. Compare all 16 against H2 in ONE instruction (SSE2 _mm_cmpeq_epi8)
6. Bitmask of matches -> check only matching slots for key equality
```

### Performance characteristics

| Property | Value |
|----------|-------|
| **Probe unit** | 16 slots at once (SIMD) |
| **Empty check** | Single SIMD comparison |
| **Load factor** | Up to ~87.5% efficiently |
| **Deletion** | Uses "deleted" sentinel -- no tombstone clustering |
| **Memory layout** | Keys and values stored flat, no pointers |

> Swiss Table is the state of the art for general-purpose hash tables. It proves that linear probing + SIMD beats more complex schemes in practice.

---

## Slide 15 -- Perfect Hashing

### Zero collisions by construction

A **perfect hash function** maps `n` keys to `m` slots with **no collisions**. A **minimal** perfect hash uses `m = n`.

### FKS two-level scheme (Fredman, Komlos, Szemeredi)

**Level 1:** Hash `n` keys into `m` primary slots using a universal hash function.

**Level 2:** For each primary slot `j` with `n_j` keys, build a secondary table of size `m_j = n_j^2` with its own universal hash function.

```
Level 1:  h1(k) -> primary slot j
Level 2:  h_j(k) -> position in secondary table for slot j
```

### Space and time

| Property | Value |
|----------|-------|
| **Lookup** | `O(1)` worst case -- two hash computations, two array lookups |
| **Total space** | `O(n)` expected (sum of `n_j^2` is `O(n)` when `m = n`) |
| **Construction** | `O(n)` expected -- retry level-2 functions if collisions |
| **Static only** | Keys must be known in advance; no insertions |

### Practical use

- Compiler keyword tables
- Routing tables with fixed prefix sets
- Tools: `gperf`, `CMPH`, `BBHash`

> Perfect hashing is optimal for static dictionaries. For dynamic sets, cuckoo hashing is the closest analogue with `O(1)` worst-case lookup.

---

## Slide 16 -- Hash Tables in Practice -- Python & Java

### Python dict (CPython 3.6+)

- **Open addressing** with a compact layout
- Separate **hash array** (indices) and **dense entries array** (preserves insertion order)
- Hash function: **SipHash** (hash-flooding resistant)
- Resize at `α = 2/3`; growth factor is roughly `3x` for small dicts, `2x` for large
- Compact dict saves ~25% memory vs the pre-3.6 implementation

### Java HashMap

- **Separate chaining** with linked list per bucket
- At `α = 0.75` (default), resize to `2 * capacity`
- When a chain exceeds **8 elements**, the chain is **treeified** -- converted to a red-black tree (`O(log n)` worst-case lookup per bucket)
- When the chain shrinks below **6**, it reverts to a linked list
- Hash function: `(h = key.hashCode()) ^ (h >>> 16)` -- mixes high bits into low bits

### Comparison

| Feature | Python dict | Java HashMap |
|---------|------------|-------------|
| **Strategy** | Open addressing | Separate chaining |
| **Order preserved** | Yes (insertion order) | No (use LinkedHashMap) |
| **Treeification** | No | Yes (chains > 8) |
| **Hash function** | SipHash | hashCode() XOR-shifted |
| **Default load factor** | 0.67 | 0.75 |

---

## Slide 17 -- Hash Tables in Practice -- Go & C++

### Go map

- **Separate chaining with buckets** -- each bucket holds 8 key-value pairs
- Overflow buckets linked when a bucket is full
- **Incremental rehashing** -- grows and evacuates lazily during map operations
- Hash function: architecture-dependent (AES instructions on amd64, fallback otherwise)
- **Not thread-safe** -- concurrent read/write causes a fatal panic (by design)
- Iteration order is **randomised** to prevent reliance on ordering

### C++ std::unordered_map

- **Separate chaining** with linked list per bucket (mandated by the standard's iterator invalidation rules)
- Default `max_load_factor = 1.0`
- Poor cache locality due to pointer-based chains -- widely considered a performance anti-pattern
- **Alternatives:** `absl::flat_hash_map` (Swiss Table), `robin_map` (Tessil), `ankerl::unordered_dense`

### Why std::unordered_map is slow

| Issue | Cause |
|-------|-------|
| **Pointer chasing** | Each node is a separate heap allocation |
| **Cache misses** | Nodes scattered across memory |
| **High load factor** | Default `1.0` means long chains |
| **Iterator stability** | Standard requires references stay valid -- prevents open addressing |

> The C++ standard's iterator stability requirement forces `std::unordered_map` into separate chaining. For performance-critical code, use a third-party flat hash map.

---

## Slide 18 -- Thread-Safe Hash Maps

### The problem

Concurrent reads and writes to a hash table require synchronisation. A global lock serialises all operations and destroys throughput.

### Lock striping (Java ConcurrentHashMap)

- Partition the table into **segments** (e.g. 16)
- Each segment has its own lock
- Operations on different segments proceed in parallel
- Java 8+: replaced segments with per-bucket CAS + `synchronized` on the bucket head

### Lock-free approaches

- **Read-Copy-Update (RCU):** readers proceed without locks; writers copy the affected structure, modify the copy, atomically swap the pointer
- **Harris-style lock-free linked lists:** CAS-based insertion and deletion on chains
- **Cliff Click's NonBlockingHashMap:** lock-free open-addressing using CAS on state transitions

### Concurrent hash map comparison

| Implementation | Read | Write | Resize |
|---------------|------|-------|--------|
| **Global lock** | Serialised | Serialised | Blocking |
| **Lock striping** | Parallel (per segment) | Parallel (per segment) | Per-segment |
| **CAS per bucket** | Lock-free | Lock-free | Cooperative |
| **RCU** | Wait-free | Lock on write side | Copy-on-write |

> For read-heavy workloads, RCU or lock-free maps offer the best throughput. For mixed workloads, CAS-based per-bucket locking (Java 8 ConcurrentHashMap) is the industry standard.

---

## Slide 19 -- Applications & Limitations

### Applications

- **Symbol tables** -- compilers, interpreters, debuggers
- **Caching** -- memoisation, LRU caches (hash map + doubly-linked list)
- **Databases** -- hash indexes, hash joins, in-memory tables
- **Networking** -- connection tracking, NAT tables, routing tables
- **Deduplication** -- seen-set for crawlers, stream processing
- **Counting** -- frequency tables, word counts, histograms
- **Sets** -- membership testing (hash set = hash map with no values)

### Limitations

- **No ordering** -- no efficient range queries, min/max, successor/predecessor (use a BST or B-tree instead)
- **Hash function quality matters** -- poor hash functions cause clustering and `O(n)` degradation
- **Hash-flooding attacks** -- adversarial inputs can force worst-case behaviour if hash function is predictable (mitigated by SipHash / random seeds)
- **Memory overhead** -- open addressing wastes empty slots; chaining wastes pointers
- **Resizing latency spikes** -- rehashing `n` elements pauses the system (mitigated by incremental rehashing)
- **Not cache-optimal for scans** -- iterating all elements is slower than a flat array

> When you need sorted iteration or range queries, a balanced BST (`std::map`, `TreeMap`) or B-tree is the right choice. Hash tables dominate point queries.

---

## Slide 20 -- Summary & Further Reading

### Key takeaways

- Hash tables provide `O(1)` expected time for insert, search, and delete -- the fastest dictionary for point queries
- The choice of hash function determines distribution quality and security against adversarial inputs
- Separate chaining is simple but cache-unfriendly; open addressing keeps data in contiguous memory
- Robin Hood hashing equalises probe distances; cuckoo hashing guarantees `O(1)` worst-case lookup
- Swiss Table (SIMD + linear probing) is the current state of the art for general-purpose hash maps
- Perfect hashing achieves `O(1)` worst case for static key sets
- Load factor management and amortised resizing keep operations constant-time in practice
- Thread safety requires lock striping, CAS-based techniques, or read-copy-update -- never a single global lock

### Recommended reading

| Source | Description |
|--------|------------|
| **CLRS** | *Introduction to Algorithms*, chapters 11--12 -- hash tables and BSTs |
| **Knuth** | *The Art of Computer Programming*, Vol. 3 -- Sorting and Searching, Section 6.4 |
| **Herlihy & Shavit** | *The Art of Multiprocessor Programming* -- concurrent hash maps |
| **Abseil Swiss Table** | [abseil.io/about/design/swisstables](https://abseil.io/about/design/swisstables) -- Swiss Table design doc |
| **Pagh & Rodler** | "Cuckoo Hashing" -- *Journal of Algorithms*, 2004 |
| **Celis et al.** | "Robin Hood Hashing" -- FOCS 1985 |
