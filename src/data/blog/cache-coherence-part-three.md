---
title: "Cache Coherence and the MESI Protocol: Part 3"
author: jieqiboh
pubDatetime: 2026-04-13T00:00:00Z
description: "-"
draft: false
tags: []
---

This is a continuation from part 2, which covers measurements of cache effects under sequential and random access patterns.
We explore cache coherence, which is the uniformity of shared resource data that is stored in multiple local caches. This is something to consider when multiple execution contexts (threads or processes) use the
same region of memory.

## Write-Through vs Write-Back

These are two strategies for keeping cache and main memory in sync when you write data.

**Write-through** is simple: every write to cache immediately also writes to main memory. Cache and RAM are always identical.  
The problem is performance — if you're incrementing a counter in a loop, every single increment fires a write to main memory over the FSB, even though nobody else needs that value yet.

**Write-back** is smarter: writes only update the cache line and mark it dirty. The write to main memory is deferred until the cache line gets evicted (LRU etc). A tight loop modifying the same variable fires one eventual write to RAM instead of thousands — much better for FSB bandwidth.

## The Problem: Multiple Caches, One Memory

Write-back creates a problem on multi-processor systems. Each CPU has its own cache, so if CPU 1 has a dirty cache line (modified but not yet written back to RAM) and CPU 2 tries to read that same address, CPU 2 fetches from main memory and gets the stale old value. The correct value is sitting in CPU 1's cache, unreachable.

This is the cache coherence problem. The hardware must solve it: when CPU 2 requests an address, the system needs to know that CPU 1 holds a dirty version and intercept the read to supply the correct data. This is what cache coherence protocols like MESI handle.

## The MESI Protocol

Each cache line in every processor's cache is tagged with one of four states (2 bits are used):

- **Modified (M):** Present only in this cache, and dirty — it has been modified from the value in main memory. The cache must write the data back to main memory before any other cache can read it. The write-back transitions the line to Shared.
- **Exclusive (E):** Present only in this cache, but clean — it matches main memory. It may become Shared in response to a read request from another CPU, or Modified when written to locally.
- **Shared (S):** May exist in multiple caches and is clean — it matches main memory. The line may be discarded (→ Invalid) at any time.
- **Invalid (I):** The cache line is invalid (unused).

For any given pair of caches, the permitted states of a given cache line are:

|   | M | E | S | I |
|---|---|---|---|---|
| **M** | ✗ | ✗ | ✗ | ✓ |
| **E** | ✗ | ✗ | ✗ | ✓ |
| **S** | ✗ | ✗ | ✓ | ✓ |
| **I** | ✓ | ✓ | ✓ | ✓ |

When a line is M or E in one cache, all other caches must hold it as I.

## State Transitions

The MESI protocol is a finite-state machine driven by two types of stimuli.

**Processor requests** — issued by the local CPU:
- **PrRd:** processor requests to read a cache block
- **PrWr:** processor requests to write a cache block

**Bus-side requests** — snooped from other processors via the shared bus:
- **BusRd:** another processor wants to read a block
- **BusRdX:** another processor wants to write a block it doesn't currently hold. The X stands for Exclusive — the requester needs the data *and* exclusive ownership, so all other caches must invalidate their copies. It's not called BusWrite because the processor doesn't have the data yet and must fetch it first; the write comes after.
- **BusUpgr:** another processor wants to write a block it already holds in Shared state. Since it already has the data, only invalidation is needed — no data transfer, cheaper than BusRdX.
- **Flush:** a cache is writing a dirty block back to main memory.
- **FlushOpt:** a cache supplies a block directly to another cache on the bus, bypassing main memory (cache-to-cache transfer). The Opt stands for Optional — main memory may or may not update itself by snooping this transfer. It's faster because cache-to-cache latency is lower than a round-trip through RAM.

**Snooping:** every cache monitors all bus transactions. When a bus request appears, each cache controller checks whether it holds that line and reacts — updating state, supplying data, or invalidating its copy. The memory controller also snoops, stepping in when no cache can supply the line.

![](/images/cache-coherence-part-three/cache-shared-bus.png)

All cache lines start **Invalid**.

**Processor-initiated transitions**

| State | Operation | New State | What happens |
|---|---|---|---|
| Invalid | PrRd | E or S | Issues BusRd; → E if no other cache responds, → S if others have a copy; data comes from another cache or main memory |
| Invalid | PrWr | M | Issues BusRdX; others invalidate their copies; data fetched from another cache or main memory; then written |
| Exclusive | PrRd | E | Cache hit — no bus traffic |
| Exclusive | PrWr | M | Cache hit — silent upgrade, no bus traffic |
| Shared | PrRd | S | Cache hit — no bus traffic |
| Shared | PrWr | M | Issues BusUpgr; all other copies → Invalid |
| Modified | PrRd | M | Cache hit — no bus traffic |
| Modified | PrWr | M | Cache hit — no bus traffic |

**Bus-initiated transitions**

| State | Operation | New State | What happens |
|---|---|---|---|
| Invalid | BusRd | I | Signal ignored |
| Invalid | BusRdX / BusUpgr | I | Signal ignored |
| Exclusive | BusRd | S | Put FlushOpt on bus with block contents |
| Exclusive | BusRdX | I | Put FlushOpt on bus with block contents |
| Shared | BusRd | S | May put FlushOpt on bus (design choice — one Shared cache supplies the data) |
| Shared | BusRdX / BusUpgr | I | May put FlushOpt on bus (design choice) |
| Modified | BusRd | S | Put FlushOpt on bus; memory controller snoops and writes to main memory |
| Modified | BusRdX | I | Put FlushOpt on bus; memory controller snoops and writes to main memory |

Key insight: E→M is silent — a PrWr on an Exclusive line needs no bus transaction. S→M requires a BusUpgr broadcast to every CPU holding the line. This is why the CPU tries to keep lines Exclusive — it avoids the broadcast on the next write.

<img src="/images/cache-coherence-part-three/mesi-protocol-transitions.png" width="300" />

**Not all transitions generate bus traffic.** PrRd and PrWr that hit a cached line (M, E, or S) are resolved entirely within the local cache — no bus activity. Bus transactions only occur when:

- The line is Invalid — a BusRd or BusRdX goes on the bus
- A Shared line is written — BusUpgr invalidates all other copies
- A Modified line is snooped — the owner flushes to RAM before the requesting CPU can proceed

## Implications for Performance

### False Sharing

### Write Amplification

## Variants: MESIF and MOESI

## Resources
