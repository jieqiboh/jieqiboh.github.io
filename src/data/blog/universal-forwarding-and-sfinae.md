---
title: "Universal Forwarding in C++"
author: jieqiboh
pubDatetime: 2026-04-18T00:00:00Z
description: "-"
draft: false
tags: []
---

Quick notes on C++ value categories, template argument deduction, and how `std::forward` lets you preserve move/copy semantics across a function call boundary.

## A Concrete T

For the examples below, `T` is a type that owns a resource with explicit copy and move constructors:

```cpp
struct Widget {
    std::string data;

    Widget(const std::string& s) : data(s) {
        std::cout << "copy ctor\n";
    }

    Widget(std::string&& s) : data(std::move(s)) {
        std::cout << "move ctor\n";
    }
};
```

When `create` calls `T(arg)`, overload resolution picks between these two based on the value category of `arg`. That's the whole problem.

## The Problem: Unnecessary Copies

Consider a simple wrapper that constructs a `T`:

```cpp
template <typename T, typename Arg>
T create(Arg arg) {        // takes by value
    return T(arg);
}
```

Call it with a string:

```cpp
std::string s = create<std::string>(std::string("hello"));
```

What happens:

1. `std::string("hello")` — temporary created
2. Copied into `arg` — copy 1
3. `arg` copied into `T(arg)` — copy 2

Two copies of a string that never needed to be copied. The original was a temporary — it could have been moved both times.

## Naive Fix: Take by Rvalue Reference

```cpp
template <typename T, typename Arg>
T create(Arg&& arg) {
    return T(std::move(arg));
}
```

Now it moves. But:

```cpp
std::string existing = "hello";
auto s = create<std::string>(existing);  // moves from existing!
// existing is now in a valid but unspecified state -- you just silently destroyed it
```

Unconditionally moving is wrong when the caller passed an lvalue they still need.

## The Real Problem

Inside `create`, `arg` is always an lvalue — it has a name, you can take its address. The information about whether the caller passed an lvalue or rvalue is gone the moment it becomes a named parameter.

What you actually need:

- Move if the caller passed a temporary (rvalue)
- Copy if the caller passed a named variable (lvalue)

And this has to happen at compile time.

## The Solution: Universal References + `std::forward`

```cpp
template <typename T, typename Arg>
T create(Arg&& arg) {                        // universal reference
    return T(std::forward<Arg>(arg));        // preserves value category
}

std::string existing = "hello";
auto s1 = create<std::string>(existing);              // Arg=string&,  copies
auto s2 = create<std::string>(std::string("hello"));  // Arg=string&&, moves
```

`Arg&&` is a universal reference — not a plain rvalue ref — because `Arg` is a deduced template parameter. Reference collapsing determines what `Arg` actually becomes:

```cpp
// caller passes lvalue:  Arg deduced as string&,  Arg&& collapses to string&
// caller passes rvalue:  Arg deduced as string,   Arg&& stays as string&&
```

`std::forward<Arg>(arg)` casts `arg` back to whatever `Arg` was deduced as. It's a conditional `std::move` — moves for rvalues, copies for lvalues.

Without it:

```cpp
return T(arg);                     // always copies -- arg is an lvalue inside the function
return T(std::move(arg));          // always moves  -- wrong for lvalue inputs
return T(std::forward<Arg>(arg));  // correct       -- moves xor copies based on caller
```

## What's Left Unsolved

`Arg&&` accepts any type — including ones `T` can't be constructed from, and including `T` itself, which can silently hijack copy construction. That's a separate problem, covered in the next post on SFINAE.
