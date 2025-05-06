---
title: "The Semantics of Move and Copy"
date: 2025-05-05
description: "-"
---

I recently completed my parallel programming module, and while it wasn't a particularly pleasant experience, I think I got the gist of ownership, as well as the concepts surrounding move and copy, which are important to internalise. This is a distillation of what I've learned for future me's reference.

## Ownership ##

Unless you've touched lower level languages such as C++ (and particularly Rust), ownership would not be something that you would necessarily know.
Broadly speaking, ownership is about who controls a certain value stored on the stack or heap.  
In C++, an owner is an [object](https://en.cppreference.com/w/cpp/language/object) containing a pointer to an object allocated by new. (for which a delete is required)
An important part of ownership is determining when and who should delete an object.

Say we write the following in C++:
```cpp
std::string s1 = "hello";
```

{{< figure src="/blog/copy-move-semantics/images/1.png" width="300">}}

As can be seen, s1 is stored on the stack, and contains a pointer to the string "hello" stored on the heap, and thus is considered its owner. As std::string follows [RAII](https://en.cppreference.com/w/cpp/language/raii), once s1 goes out of scope, its heap allocated "hello" gets deleted as well.

### Move (C++ std::move as compared to Rust)

Now, say we want to copy the contents of s1 to s2, and don't need s1 to contain the value "hello" anymore.
We could naively do the following:

```cpp
std::string s1 = "hello";
std::string s2 = s1;
```

{{< figure src="/blog/copy-move-semantics/images/2.png" width="300">}}
C++ actually performs a deep copy, allocating more space on the heap and copying the contents of s1 over!
(Note that whether a deep or shallow copy is performed depends on the type and how its copy assignment operator is implemented. For strings a deep copy is done.)

Instead, what we hope to achieve is a _transferral of ownership_, where s2 points to the original dynamically allocated string on the heap, and s1 loses its pointer to it.

{{< figure src="/blog/copy-move-semantics/images/3.png" width="300">}}

```cpp
std::string s1 = "hello";
std::string s2 = std::move(s1);
```

[`std::move`](https://en.cppreference.com/w/cpp/utility/move) helps to achieve this behaviour.
#### More: Value Categories

For those wondering how `std::move` works, we first need to understand [value categories](https://en.cppreference.com/w/cpp/language/value_category) in C++.