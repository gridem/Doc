# Smart Pointers: Bad Design

[TOC]

## Null Pointers

Null pointers causes a crash. Thus we need to:

1. Introduce `NullPointerException`.
2. Use non-nullable smart pointers.

## Unique vs Shared

You don't know in advance the correct case. The common practice is to use `unique_ptr` and put it inside `shared_ptr` if it's needed. But it reduces the performance.

The idea is to introduce special class: `An<T>` as a smart pointer and use partial specialization to specialize it for the usage.

## Dependency Injection

The idea here is that you need to use smart pointers just for dependency injection. See my approach + god adapter.

Only in limited set of cases if you need to explicitly specify the shared behavior you should use it and specialize it explicitly.