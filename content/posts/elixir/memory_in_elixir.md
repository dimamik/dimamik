---
title: "How memory is managed in Elixir/Erlang"
date: 2024-11-23T00:00:00Z
description: Garbage collection, loss of sharing, how memory leaks can happen, and how memory in organized in ETSes and DETSes.
tags:
  - elixir
  - memory
  - low-level
categories:
  - Elixir
draft: false
cover:
  image: /img/posts/elixir/memory_in_elixir.webp
---

## TL;DR

- **Stack vs Heap:** Stack is for local variables (LIFO), heap for dynamic memory. Erlang processes have separate stacks and heaps, with garbage collection triggered when they meet.

- **Garbage Collection:** Short-lived data stays in the young heap; long-lived data moves to the old heap, collected less frequently.

- **Memory:** Isolated per process, immutable, and copied when shared. Large binaries (64+ bytes) are exceptions.

- **ETS/DETS:** ETS is in-memory; DETS is on-disk. ETS manages its memory separately and supports concurrency.

- **Leaks:** Occur in process states/ETS. Use ETS for large or frequently updated data. Debug with :erlang.garbage_collect().

## Stack vs Heap

- The **stack** is memory used for local variables and control flow, automatically managed and limited in size. It follows a last-in, first-out (LIFO) order.

- The **heap** is memory used for dynamic allocation, managed manually by the programmer, allowing for flexible but potentially more complex usage.

Each Erlang process has its own stack and heap which are allocated in the same memory block and grow towards each other. When the stack and the heap [meet](https://github.com/erlang/otp/blob/OTP-18.0/erts/emulator/beam/beam_emu.c#L387), the garbage collector is triggered and memory is reclaimed. If not enough memory was reclaimed, the heap will grow.

## Generational garbage collection

- **Young Heap**: This is the memory area where new data is initially allocated. When a process in Erlang creates new data, it goes into the young heap. Garbage collection occurs more frequently here because data is short-lived; if an object survives multiple garbage collections in the young heap, it is considered more likely to be long-lived.
- **Old Heap**: Data that survives a certain number of garbage collections in the young heap is promoted to the old heap. This is intended for long-lived data, which is collected less frequently. The idea is that old heap garbage collections are more costly, so they occur less often.

## Memory in erlang/elixir

- Memory is not shared between processes (one notable exception):
  - Data is always deep copied:
    - When sent between processes
    - When given аs initial args to а new process (ex. spawn)
    - When written to / from ETS
  - Exception is with literals and big strings (64+ bytes binaries), those are shared between processes
  - BUT, memory is actively shared inside a process
    - For example to reference the same nested struct multiple times
- All values are immutable, hence modification means creation of new value \* often data structures are only partially recreated
- Memory is owned (and collected) per process, including ETS tables
- ETS tables manage their memory by themselves, no garbage collection

## Loss of sharing

BEAM does not expect you to send messages with shared sub terms (like nested repeated struct)

- Recap: Memory is not shared between processes
- Recap: Data is copied to another process or ETS
- But copying data to another process or ETS involves “loss of sharing”
- It can be helpful to use `:erts_debug.size()` and `:erts_debug.flat_size()` to get real size and flat size оf a term
- This optimization can be turned off with “—enable-sharing-preserving” (experimental)

## Memory leaks

- Mainly happen in process states and ETSes
- Sometimes things only look like leaks - GC will not always collect your garbage (ex. if it is in “old” heap) - `:erlang.garbage_collect()` can be helpful for debugging, but probably not the solution you are looking for
- If you store a lot of data in process state and modify it frequently, consider keeping it in ETS
- You can choose to allocate memory for messages in process heap or off it `+hmad off_heaplon_heap` VM argument or `:message_queue_data` process flag

## ETS and DETS

- ETS in [Erlang](https://www.erlang.org/docs/23/man/ets)/[Elixir](https://elixirschool.com/en/lessons/storage/ets) - a robust in-memory store for Elixir and Erlang objects that comes included. ETS is capable of storing large amounts of data and offers constant time data access.

- [DETS](https://www.erlang.org/docs/22/man/dets) - same as ETS, but stores data on disk and almost never fully loads it into RAM

### How ETS manages its own memory?

1. **Separate from Process Heaps**: ETS tables are not part of the normal Erlang process heaps. Instead, they are allocated outside the process heaps in a separate memory area. This allows ETS tables to store large volumes of data without affecting the garbage collection of Erlang processes.
2. **Fixed and Dynamic Allocation**: ETS allocates memory in blocks. It starts with a fixed block size, but as the table grows, new blocks are allocated dynamically. This enables efficient growth of tables and avoids frequent memory reallocation.
3. **Table Types**: ETS supports several table types such as `set`, `ordered_set`, `bag`, and `duplicate_bag`. Each type influences how memory is organized and accessed.
4. **Garbage Collection**: ETS tables do not undergo the same garbage collection as Erlang processes. Since ETS tables are independent, memory usage must be managed explicitly by creating and deleting tables as needed.
5. **Ownership and Concurrency**: An ETS table is owned by a process, and when this owning process terminates, the ETS table is automatically deleted, freeing up its memory. ETS provides concurrency controls allowing multiple processes to read from a table, with configurable options for write concurrency.

### How (and when) ETS frees its memory

1. Table Deletion
2. Owner Process Termination
3. Manual Cleanup

## Sources

[Memory leaks in Elixir](https://www.youtube.com/watch?v=DIEgEo1egJE)
[Erlang Garbage Collector — erts v15.1.2](https://www.erlang.org/doc/apps/erts/garbagecollection.html)
