---
title: "SFINAE in C++"
author: jieqiboh
pubDatetime: 2026-04-18T00:00:00Z
description: "-"
draft: false
tags: []
---

The previous post ended with a problem: `T&&` with a deduced `T` accepts every type, so templates can show up in overload resolution where they don't belong. SFINAE is how you control that.

## The Core Rule

When the compiler resolves a call, it builds a candidate list by substituting actual types into each template's signature. If substitution produces an invalid expression, the candidate is silently dropped — no error. This is SFINAE: Substitution Failure Is Not An Error.

The important word is *signature*. SFINAE only applies to failures in the function signature — return type, parameter types, template parameters. Failures in the function body are always hard errors.

```cpp
template <typename T>
void foo(typename T::value_type x);  // signature mentions T::value_type

foo<int>(42);  // int has no ::value_type -- dropped silently
```

```cpp
template <typename T>
void bar(T val) {
    typename T::value_type x;  // body -- hard error
}

bar<int>(42);  // error -- SFINAE doesn't reach the body
```

If every candidate drops out, you get "no matching function." If at least one survives, the failures of the others are invisible.

## The Simplest Fallback Pattern

The earliest SFINAE idiom uses `...` — C's variadic argument list — as a catch-all:

```cpp
template <typename T>
void process(T val, typename T::iterator* = nullptr) {
    std::cout << "has iterator\n";
}

void process(...) {   // matches anything -- last resort
    std::cout << "no iterator\n";
}

process(std::vector<int>{});  // first overload -- vector has ::iterator
process(42);                  // first overload dropped (int has no ::iterator), fallback wins
```

`...` means "accept any number of arguments of any type." Overload resolution ranks it dead last, so it only wins when everything else was dropped. It's different from C++11 variadic templates (`typename... Args`), which are type-safe and compile-time. The C-style `...` predates the type system entirely — in modern C++ its only real job is this kind of fallback.

## `std::enable_if` — The Standard Tool

Raw SFINAE like above is fragile. `std::enable_if` is the standard way to engineer a substitution failure on purpose:

```cpp
// Primary template -- empty, no ::type
template <bool Condition, typename T = void>
struct enable_if {};

// Specialization -- only when Condition is true
template <typename T>
struct enable_if<true, T> {
    using type = T;
};
```

That's the whole thing — partial specialization on a bool. When `Condition` is false, the struct is empty. Accessing `::type` on it is invalid, which triggers substitution failure in the signature, which drops the overload.

`enable_if_t` just hides the `typename ... ::type` noise:

```cpp
template <bool Condition, typename T = void>
using enable_if_t = typename enable_if<Condition, T>::type;
```

In use:

```cpp
template <typename T>
std::enable_if_t<std::is_integral_v<T>, void>
process(T val) {
    std::cout << "integer: " << val << "\n";
}

template <typename T>
std::enable_if_t<std::is_floating_point_v<T>, void>
process(T val) {
    std::cout << "float: " << val << "\n";
}

process(42);    // first overload
process(3.14);  // second overload
process("hi");  // hard error -- both drop out
```

`std::enable_if_t<..., void>` is the return type here. The return type is just an expression — it can be a template that fails to resolve. When the condition is false, the return type is invalid, the signature fails, and the overload is dropped before the body is touched.

## Where `enable_if` Can Live

Three equivalent placements:

```cpp
// 1. Return type
template <typename T>
std::enable_if_t<std::is_integral_v<T>, void>
foo(T val);

// 2. Extra template parameter -- most common
template <typename T,
          typename = std::enable_if_t<std::is_integral_v<T>>>
void foo(T val);

// 3. Parameter -- useful for constructors, which have no return type
template <typename T>
void foo(T val, std::enable_if_t<std::is_integral_v<T>>* = nullptr);
```

All three drop the overload when the condition is false. Option 2 is what you'll see most often.

## Using SFINAE to Fight SFINAE

The language rule is that substitution failure in a signature is silent. That rule exists so template overloads can coexist without accidentally erroring on types they don't apply to.

`enable_if` turns this around — it deliberately constructs a substitution failure to eliminate overloads you don't want:

```
SFINAE as safety net:
    "don't error when a template happens to be invalid for this T"

SFINAE as weapon (enable_if):
    "make this overload deliberately invalid for T when my condition is false"
```

The compiler can't tell the difference between an accidental and an intentional failure. So you pick the exact moment you want an overload gone — `::type` on an empty struct, guaranteed invalid when the condition is false — and plant the failure there.

## The Full Picture

```
Compiler sees foo(arg)
        |
        v
Build candidate list -- for each template overload:
    evaluate the SIGNATURE only
        |
        ├── valid   --> add to candidate list
        └── invalid --> drop silently (SFINAE), body never touched
        |
        v
Pick best candidate
        |
        ├── one match   --> instantiate, compile body
        ├── no matches  --> hard error: no matching function
        └── ambiguous   --> hard error: ambiguous call
```

## SFINAE vs Concepts

C++20 concepts do the same thing with readable syntax:

```cpp
// SFINAE
template <typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
void process(T val) { ... }

// Concepts
template <std::integral T>
void process(T val) { ... }
```

The main practical difference is error messages:

```
// SFINAE:
error: no matching function for call to 'process'
note: substitution failure: no type named 'type' in enable_if<false>

// Concepts:
error: no matching function for call to 'process'
note: constraint 'std::integral<double>' was not satisfied
```

Concepts tell you exactly what failed. SFINAE errors send you hunting through template internals. For new C++20 code, prefer concepts. SFINAE still matters for reading older codebases and understanding what concepts are doing under the hood.
