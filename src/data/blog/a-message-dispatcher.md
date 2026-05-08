---
title: "A Message Dispatcher"
author: jieqiboh
pubDatetime: 2026-05-08T00:00:00Z
description: "-"
draft: false
tags: []
---

I was trying to understand std::variant and std::visit, so I made a simple project with the following premise: You have a stream of heterogeneous messages arriving at runtime, which have to be dispatched to the right handler.

## The Messages

```cpp
struct NewOrder { uint64_t order_id;  double price;    int qty;     char side; };
struct Cancel   { uint64_t order_id; };
struct Modify   { uint64_t order_id;  double new_price; int new_qty; };
```

Exact sizes and memory layout:

```
NewOrder:
[ order_id: 8 ][ price: 8 ][ qty: 4 ][ side: 1 ][ pad: 3 ] = 24 bytes

Cancel:
[ order_id: 8 ] = 8 bytes

Modify:
[ order_id: 8 ][ new_price: 8 ][ new_qty: 4 ][ pad: 4 ] = 24 bytes
```

## Approach 1 — `std::variant` + `std::visit`

### Union vs Variant

A raw `union` stores one of several types in shared memory — sized for the largest member:

```cpp
union Raw {
    NewOrder new_order;
    Cancel   cancel;
    Modify   modify;
};
// sizeof(Raw) = 24 — always pays for the largest member
// nothing stops you from reading new_order when cancel is active → undefined behaviour
```

`std::variant` wraps the union and adds a type index, making access safe:

```
std::variant<NewOrder, Cancel, Modify>:

[ index: 0|1|2 (1 byte) ][ padding (7 bytes) ][ data (24 bytes) ]
                                                ^^^^^^^^^^^^^^^^^
                                                sized for largest member (NewOrder/Modify)
                                                a Cancel stored here wastes 16 bytes

Total: 32 bytes, regardless of which type is active
```

`variant.index()` tells you which type is live. `std::get<NewOrder>(v)` throws if the wrong type is active. The raw union gives none of this — you have to track it yourself, and getting it wrong is UB.

### The Overload Pattern

`std::visit` takes a single callable and dispatches based on the active type. The overload pattern bundles multiple lambdas into one object:

```cpp
template<typename... Ts>
struct overload : Ts... {
    using Ts::operator()...;
};
template<typename... Ts> overload(Ts...) -> overload<Ts...>;
```

Each non-capturing lambda is its own anonymous struct type with a single `operator()` and no data members:

```cpp
struct __lambda_0 { void operator()(const NewOrder& o) const { ... } };
struct __lambda_1 { void operator()(const Cancel& c)   const { ... } };
struct __lambda_2 { void operator()(const Modify& m)   const { ... } };
```

When you write:

```cpp
overload{
    [](const NewOrder& o) { ... },
    [](const Cancel& c)   { ... },
    [](const Modify& m)   { ... },
}
```

CTAD deduces `Ts... = __lambda_0, __lambda_1, __lambda_2` and instantiates:

```cpp
struct overload<__lambda_0, __lambda_1, __lambda_2>
    : __lambda_0, __lambda_1, __lambda_2
{
    using __lambda_0::operator();
    using __lambda_1::operator();
    using __lambda_2::operator();
};
```

The three `using` declarations pull all three `operator()`s into one overload set, so a single call resolves to the right handler based on argument type. Because none of the lambdas capture anything, every base class is an empty struct. Empty Base Optimization (EBO) collapses them — the derived struct also holds no data, and constructing it emits no instructions at runtime.

### What the Compiler Emits

At `-O2`, `std::visit` with stateless lambdas compiles to a direct jump table — no function pointer indirection, handler code inlined at each target:

```
handle_visit:
    load index from variant         ; 0, 1, or 2

    jump to table[index]

    table[0]:                       ; NewOrder
        load order_id, price, qty
        add to g_sink
        return

    table[1]:                       ; Cancel
        load order_id
        sub from g_sink
        return

    table[2]:                       ; Modify
        load order_id, new_price
        add to g_sink
        return
```

The overload struct disappears entirely. The abstraction costs nothing.

## Approach 2 — Virtual Dispatch

### The Vtable for Each Class

Every class with a virtual function gets a vtable in static memory. Every instance gets a hidden `vptr` pointing to its class's vtable:

```
IMessage vtable:
  [0]  ~IMessage()       → default destructor
  [1]  handle()          → pure virtual

VNewOrder vtable:
  [0]  ~VNewOrder()
  [1]  VNewOrder::handle → g_sink += order_id + price + qty

VCancel vtable:
  [0]  ~VCancel()
  [1]  VCancel::handle   → g_sink -= order_id

VModify vtable:
  [0]  ~VModify()
  [1]  VModify::handle   → g_sink += order_id + new_price
```

The vtables are shared — one per class, not per instance. But every instance pays 8 bytes for the `vptr`.

### Object Layout in Memory

```
VNewOrder:
[ vptr (8) ][ order_id (8) ][ price (8) ][ qty (4) ][ pad (4) ] = 32 bytes
     |
     └──→ VNewOrder vtable:  [ ~VNewOrder | VNewOrder::handle ]

VCancel:
[ vptr (8) ][ order_id (8) ] = 16 bytes
     |
     └──→ VCancel vtable:    [ ~VCancel | VCancel::handle ]

VModify:
[ vptr (8) ][ order_id (8) ][ new_price (8) ][ new_qty (4) ][ pad (4) ] = 32 bytes
     |
     └──→ VModify vtable:    [ ~VModify | VModify::handle ]
```

### The Dispatch Sequence

```
m->handle():
  1. load vptr from *m              (memory read — follows pointer to object)
  2. load fn ptr from vptr[1]       (memory read — follows vptr to vtable)
  3. call through function pointer  (indirect branch)
```

Two memory reads before any handler logic runs.

### The Real Cost — Heap Scatter

The benchmark stores messages as `vector<unique_ptr<IMessage>>`. The vector itself is contiguous, but each pointer leads to a separate heap allocation:

```
vector (contiguous):
[ ptr0 ][ ptr1 ][ ptr2 ][ ptr3 ] ...
    |        |        |        |
    ↓        ↓        ↓        ↓
 0x55a0   0xb3f2   0x1c44   0x9a01   ← scattered across heap
 VNewOrder VCancel  VModify  VNewOrder
```

The CPU prefetcher can predict the sequential vector reads, but cannot predict where each pointer leads. Nearly every dereference is a cache miss.

## Approach 3 — Hand-Rolled Vtable

### The Tagged Union

Instead of `std::variant`'s hidden index, we make the tag explicit:

```cpp
enum class MsgKind : uint8_t { NewOrder, Cancel, Modify };

struct RawMessage {
    MsgKind kind;
    union {
        NewOrder new_order;
        Cancel   cancel;
        Modify   modify;
    } data;
};
```

Memory layout:

```
RawMessage:
[ kind (1) ][ pad (7) ][ data (24 bytes) ]
                        ^^^^^^^^^^^^^^^^^
                        union — largest member wins
Total: 32 bytes — same as std::variant, just made explicit
```

Objects live directly in a `vector<RawMessage>` — no pointers, no heap scatter, fully contiguous.

### The Explicit Function Pointer Table

```cpp
using HandlerFn = void(*)(const RawMessage&);

constexpr std::array<HandlerFn, 3> g_vtable = {
    handle_new_order_raw,   // index 0 — MsgKind::NewOrder
    handle_cancel_raw,      // index 1 — MsgKind::Cancel
    handle_modify_raw,      // index 2 — MsgKind::Modify
};
```

This is what the compiler builds for you invisibly with `virtual`. Here we build it ourselves — same structure, fully visible.

### The Dispatch Sequence

```
handle_vtable(msg):
  1. cast msg.kind to size_t            (integer op, no memory read)
  2. load fn ptr from g_vtable[kind]    (one memory read — table is small, lives in L1)
  3. call through function pointer      (indirect branch)
```

One fewer memory read than virtual dispatch (no vptr load per object). The handlers are not inlined — unlike `std::visit`.

## Results

```
std::visit              27.52 ms     5.50 ns/msg
virtual dispatch        30.07 ms     6.01 ns/msg
hand-rolled vtable      27.56 ms     5.51 ns/msg
```

## What the Results Actually Tell You

**`std::visit` ≈ hand-rolled vtable.** The compiler generates equivalent code. The overload pattern, CTAD, and `std::visit` all compile away at `-O2`. The abstraction is free.

**Virtual dispatch is slower because of memory layout, not vtable overhead.** The vtable lookup itself costs one extra memory read. But the real penalty is the heap-scattered objects — a cache miss on nearly every message.

**The dispatch mechanism barely matters. Data layout dominates.**

If the virtual objects were stored contiguously (e.g. in a pool allocator), the pointer targets would be sequential and the prefetcher could handle them. Virtual dispatch would close the gap significantly.

Pick your data layout before picking your dispatch mechanism.

## Full Code

```cpp
// message_dispatcher.cpp
// Benchmarks: std::visit vs virtual dispatch vs manual vtable
// Compile: g++ -O2 -std=c++20 -o dispatcher message_dispatcher.cpp

#include <variant>
#include <string>
#include <functional>
#include <chrono>
#include <vector>
#include <cstdio>
#include <cstdint>
#include <memory>
#include <random>
#include <array>

// ============================================================
// 1. MESSAGE TYPES
// ============================================================

struct NewOrder {
    uint64_t order_id;
    double   price;
    int      qty;
    char     side; // 'B' or 'S'
};

struct Cancel {
    uint64_t order_id;
};

struct Modify {
    uint64_t order_id;
    double   new_price;
    int      new_qty;
};

using Message = std::variant<NewOrder, Cancel, Modify>;

// ============================================================
// 2. OVERLOAD PATTERN
// ============================================================

template<typename... Ts>
struct overload : Ts... {
    using Ts::operator()...;
};
template<typename... Ts> overload(Ts...) -> overload<Ts...>;


// ============================================================
// 3. HANDLER USING std::visit + overload
// ============================================================

static volatile int64_t g_sink = 0;

void handle_visit(const Message& msg) {
    std::visit(overload{
        [](const NewOrder& o) {
            g_sink += o.order_id + static_cast<int64_t>(o.price) + o.qty;
        },
        [](const Cancel& c) {
            g_sink -= static_cast<int64_t>(c.order_id);
        },
        [](const Modify& m) {
            g_sink += static_cast<int64_t>(m.order_id) + static_cast<int64_t>(m.new_price);
        },
    }, msg);
}


// ============================================================
// 4. VIRTUAL DISPATCH
// ============================================================

struct IMessage {
    virtual ~IMessage() = default;
    virtual void handle() const = 0;
};

struct VNewOrder : IMessage {
    uint64_t order_id; double price; int qty;
    VNewOrder(uint64_t id, double p, int q) : order_id(id), price(p), qty(q) {}
    void handle() const override {
        g_sink += order_id + static_cast<int64_t>(price) + qty;
    }
};

struct VCancel : IMessage {
    uint64_t order_id;
    explicit VCancel(uint64_t id) : order_id(id) {}
    void handle() const override { g_sink -= static_cast<int64_t>(order_id); }
};

struct VModify : IMessage {
    uint64_t order_id; double new_price;
    VModify(uint64_t id, double p) : order_id(id), new_price(p) {}
    void handle() const override {
        g_sink += static_cast<int64_t>(order_id) + static_cast<int64_t>(new_price);
    }
};


// ============================================================
// 5. HAND-ROLLED VTABLE
// ============================================================

enum class MsgKind : uint8_t { NewOrder, Cancel, Modify };

struct RawMessage {
    MsgKind kind;
    union {
        NewOrder new_order;
        Cancel   cancel;
        Modify   modify;
    } data;
};

using HandlerFn = void(*)(const RawMessage&);

void handle_new_order_raw(const RawMessage& m) {
    const auto& o = m.data.new_order;
    g_sink += o.order_id + static_cast<int64_t>(o.price) + o.qty;
}
void handle_cancel_raw(const RawMessage& m) {
    g_sink -= static_cast<int64_t>(m.data.cancel.order_id);
}
void handle_modify_raw(const RawMessage& m) {
    const auto& mo = m.data.modify;
    g_sink += static_cast<int64_t>(mo.order_id) + static_cast<int64_t>(mo.new_price);
}

constexpr std::array<HandlerFn, 3> g_vtable = {
    handle_new_order_raw,
    handle_cancel_raw,
    handle_modify_raw,
};

void handle_vtable(const RawMessage& msg) {
    g_vtable[static_cast<size_t>(msg.kind)](msg);
}


// ============================================================
// 6. BENCHMARK HARNESS
// ============================================================

constexpr size_t N = 5'000'000;

using Clock = std::chrono::steady_clock;

double bench_visit(const std::vector<Message>& msgs) {
    auto t0 = Clock::now();
    for (const auto& m : msgs) handle_visit(m);
    auto t1 = Clock::now();
    return std::chrono::duration<double, std::milli>(t1 - t0).count();
}

double bench_virtual(const std::vector<std::unique_ptr<IMessage>>& msgs) {
    auto t0 = Clock::now();
    for (const auto& m : msgs) m->handle();
    auto t1 = Clock::now();
    return std::chrono::duration<double, std::milli>(t1 - t0).count();
}

double bench_vtable(const std::vector<RawMessage>& msgs) {
    auto t0 = Clock::now();
    for (const auto& m : msgs) handle_vtable(m);
    auto t1 = Clock::now();
    return std::chrono::duration<double, std::milli>(t1 - t0).count();
}


// ============================================================
// 7. MAIN
// ============================================================

int main() {
    std::mt19937_64 rng(42);
    std::uniform_int_distribution<int> kind_dist(0, 2);

    std::vector<Message> var_msgs;
    var_msgs.reserve(N);

    std::vector<std::unique_ptr<IMessage>> virt_msgs;
    virt_msgs.reserve(N);

    std::vector<RawMessage> raw_msgs;
    raw_msgs.reserve(N);

    for (size_t i = 0; i < N; ++i) {
        int k = kind_dist(rng);
        uint64_t id = rng();
        double price = static_cast<double>(rng() % 100000) / 100.0;
        int qty = static_cast<int>(rng() % 1000) + 1;

        switch (k) {
        case 0:
            var_msgs.emplace_back(NewOrder{id, price, qty, 'B'});
            virt_msgs.push_back(std::make_unique<VNewOrder>(id, price, qty));
            raw_msgs.push_back(RawMessage{MsgKind::NewOrder, {.new_order = {id, price, qty, 'B'}}});
            break;
        case 1:
            var_msgs.emplace_back(Cancel{id});
            virt_msgs.push_back(std::make_unique<VCancel>(id));
            raw_msgs.push_back(RawMessage{MsgKind::Cancel, {.cancel = {id}}});
            break;
        case 2:
            var_msgs.emplace_back(Modify{id, price, qty});
            virt_msgs.push_back(std::make_unique<VModify>(id, price));
            raw_msgs.push_back(RawMessage{MsgKind::Modify, {.modify = {id, price, qty}}});
            break;
        }
    }

    // Warmup pass
    for (const auto& m : var_msgs)  handle_visit(m);
    for (const auto& m : raw_msgs)  handle_vtable(m);
    for (const auto& m : virt_msgs) m->handle();

    double t_visit   = bench_visit(var_msgs);
    double t_virtual = bench_virtual(virt_msgs);
    double t_vtable  = bench_vtable(raw_msgs);

    printf("\n=== Message Dispatcher Benchmark (N=%zu) ===\n\n", N);
    printf("  %-20s  %8.2f ms   %6.2f ns/msg\n",
           "std::visit",  t_visit,   t_visit   * 1e6 / N);
    printf("  %-20s  %8.2f ms   %6.2f ns/msg\n",
           "virtual dispatch", t_virtual, t_virtual * 1e6 / N);
    printf("  %-20s  %8.2f ms   %6.2f ns/msg\n",
           "hand-rolled vtable", t_vtable, t_vtable  * 1e6 / N);
    printf("\n  (sink=%lld -- prevents dead-code elimination)\n\n",
           (long long)g_sink);

    return 0;
}
```
