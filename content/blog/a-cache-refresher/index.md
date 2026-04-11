---
title: "A Cache Refresher"
date: 2026-03-30
description: "-"
---

A dive into CPU caches for those who superficially understand spatial/temporal locality.

### Direct Mapped Caches 

Recall that a cache line size is the size of a unit of transfer between the DRAM and cache.

Core idea: every word in memory can be mapped onto a single cache line.  
How: Given the setup (8 cache lines, 4-byte cache line size, 1-byte word, 32-bit address)  

- **Offset**: log₂(4) = **2 bits** — selects byte within a cache line
- **Index**: log₂(8) = **3 bits** — selects which cache line
- **Tag**: 32 − 3 − 2 = **27 bits** — identifies which memory block is cached

**Example: Fetching 0x0E8 from Physical Memory**
{{< figure src="/images/dm-cache.png" width="">}}

| Field  | Bits    | Value | Meaning                             |
|--------|---------|-------|-------------------------------------|
| Tag    | [31:5]  | 7     | Which block from memory             |
| Index  | [4:2]   | 2     | Maps to cache line #2               |
| Offset | [1:0]   | 0     | Byte 1 within the 4-byte cache line |

So this address looks up cache line 2. If the tag stored there is 7, it's a **hit** and byte 1 is returned. Otherwise it's a **miss** — the 4-byte block starting at address `0x0E8` is fetched from DRAM into cache line 2.

### Direct Mapped Caches - Increasing Block Size

Block size is always a power of 2.  
We can take advantage of spatial locality (probability that memory we subsequently access is within the same region of physical memory as the memory we are currently accessing)
This incurs a larger penalty on misses, since it takes longer to transfer the block into cache.

{{< figure src="/images/dm-cache-largerblock.png" width="">}}

### Direct Mapped Caches - Inflexible Mapping

Consider a 1024-line DM Cache with block size = 1 word. If we have a loop in a program that accesses 2 regions in physical memory that correspond to the same cache line, they will be competing for the same cache line slot,
leading to repeated misses!
Hit rate = 0%

{{< figure src="/images/dm-conflict.png" width="400">}}

### Fully Associative Caches

The address splits into just two parts - no index bits since there's no fixed slot assignment:
```
| ---- Tag bits ---- | Offset bits |
```
Example: 32-bit address, 4-byte cache lines (so 2 offset bits), say 4 cache lines total.

{{< figure src="/images/fa-cache.png" width="">}}

On a read: e.g. address = 0x00000009  
Offset (2 bits): 01 - byte 1 within the cache line  
Tag (30 bits): 0x000002

- The tag (0x000002) is broadcast to all 4 comparators simultaneously
- Every cache line checks its stored tag against 0x000002 in parallel
- If any line matches AND its valid bit is 1 - cache hit, use the offset to pick the right byte from that line's data
- If nothing matches - cache miss, fetch from DRAM, evict some line (LRU etc.), store it anywhere

Why no index? In a direct-mapped or set-associative cache, the index bits tell you exactly which slot(s) to check. Here you just broadcast to everyone and let them all raise their hand simultaneously

Observe that it resembles having multiple 1-line DM Caches running in parallel.

### N-Way Set Associative Cache

```
| Tag bits | Index bits (3) | Offset bits (2) |
```

Example:
Address: 0x00000029 = ...0000 0010 1001

- Offset: 01 (last 2 bits) byte 1 within the cache line  
- Index: 010 -- go to set 2  
- Tag: everything left = 0x000002

On a read:

- Index bits select set 2 - narrows you to one row across all 4 ways
- The tag 0x000002 is broadcast to all 4 comparators for that set simultaneously
- If any way's tag matches AND valid bit is 1 - hit, use offset to return the right byte
- If no way matches - miss, fetch from DRAM, place in any way within set 2 (e.g. LRU eviction)

{{< figure src="/images/nsa-cache-1.png" width="400">}}
{{< figure src="/images/nsa-cache-2.png" width="">}}

Why it's a compromise:

- vs direct-mapped (1-way): multiple ways per set means fewer conflict misses -- two addresses mapping to the same set can coexist
- vs fully associative: you only compare 4 tags instead of all 32, so much cheaper hardware

### Resources

[MIT OpenCourseware 6004 Computation Structures](https://ocw.mit.edu/courses/6-004-computation-structures-spring-2017/pages/c14/)
