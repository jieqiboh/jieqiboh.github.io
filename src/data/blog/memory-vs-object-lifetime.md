---
title: "std::launder"
author: jieqiboh
pubDatetime: 2026-04-19T00:00:00Z
description: "-"
draft: false
tags: []
---

I was recently implementing an `inplace_vector` — a fixed-capacity vector backed by aligned stack storage, no heap allocation — and ran into `std::launder`. It's a standard library addition that looks pointless - it takes a pointer and returns a pointer to the same address. No allocation, no transformation, no runtime effect whatsoever. On its face, it does nothing.

The reason it exists is a subtle gap in the C++ object model: a pointer to an address is NOT the same thing as a pointer to the object living at that address. Most of the time this distinction doesn't matter. In a narrow set of cases — specifically when you're using placement new to construct objects into raw storage — it matters a lot, and `std::launder` is the only way to close the gap.

## The setup

Placement new lets you construct an object into memory you already own:

```cpp
alignas(T) unsigned char buf[sizeof(T)];
::new (buf) T(args...);
```

To access the object afterwards, you cast the storage pointer:

```cpp
T* p = reinterpret_cast<T*>(buf);
p->member;
```

For most types this works fine. The problem shows up when `T` has `const` or reference members.

## Replaceable vs non-replaceable

The standard has a concept called *replaceability* ([object lifetime](https://en.cppreference.com/w/cpp/language/lifetime), [std::is_replaceable](https://en.cppreference.com/w/cpp/types/is_replaceable)). A type is replaceable if it has no `const` members and no reference members.

For a replaceable type, if you destroy an object and placement-new a new one into the same storage, any existing pointer to that storage is implicitly repointed to the new object. The replacement is transparent — the pointer just follows.

For non-replaceable types the rule is different. Consider:

```cpp
struct Fixed { const int id; };
```

`Fixed` is non-replaceable because `id` is `const`. If you placement-new two `Fixed` objects into the same storage in sequence:

```cpp
alignas(Fixed) unsigned char buf[sizeof(Fixed)];

::new (buf) Fixed{1};
::new (buf) Fixed{2};

Fixed* p = reinterpret_cast<Fixed*>(buf);
p->id;  // UB
```

The standard says `p` does not legally point to the `Fixed{2}` that now lives in `buf`. The cast gives you a pointer to the address — not to the object. Accessing through it is undefined behaviour, and the compiler treats it as such: it is free to assume that UB never occurs, so it may reason as if the second placement new never happened, reuse a value it loaded from `p->id` earlier, or do something else entirely. In practice it usually just reads the correct bytes — but at higher optimisation levels, where the compiler's alias analysis and constant propagation get more aggressive, it stops being reliable.

The issue is pointer provenance, not caching. `p` is derived from `buf` via a cast — but the standard says that for non-replaceable types, a pointer derived from the original storage does not automatically follow a replacement object. It is still considered to point at the old, now-destroyed `Fixed{1}`. Accessing a destroyed object is UB, and that's the actual problem.

The reason `const` and reference members make a type non-replaceable is to protect the optimizer within a single object's lifetime: the compiler is allowed to cache a `const` member read and reuse it later, knowing the value can't change while the object is alive. Allowing a raw pointer to silently repoint to a replacement object would undermine that — the cached value might now be stale. Making such types non-replaceable means you have to opt in to the repoint explicitly, via `std::launder`.

## What `std::launder` does

`std::launder` is the opt-in escape hatch. It takes a pointer to an address and produces a pointer that legally refers to the object currently living there:

```cpp
Fixed* p = std::launder(reinterpret_cast<Fixed*>(buf));
p->id;  // fine — p points to Fixed{2}
```

It doesn't emit any instructions at runtime. It's a signal to the compiler: don't apply any provenance-based assumptions through this pointer — treat it as freshly obtained.

## Where this matters in practice

Any code that manages raw storage and constructs objects into it via placement new needs to account for this. A generic fixed-capacity container like `inplace_vector<T, N>` can't know at compile time whether `T` is replaceable, so it routes all element access through a laundered pointer:

```cpp
T* data_ptr() noexcept {
    return std::launder(reinterpret_cast<T*>(storage_));
}
```

The same applies to `std::optional` and `std::variant`, both of which hold objects in internal aligned storage and must handle arbitrary `T`.

If you know `T` is always replaceable — no `const` members, no references — the launder is technically unnecessary. But for generic code, it's the only correct default.
