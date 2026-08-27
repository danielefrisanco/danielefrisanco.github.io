---
layout: post
title: "Why You Should Care About DLinked — A Simple, Powerful Ruby Gem"
date: 2026-08-27 00:00:00 +0200
description: "Ruby's Array hits a performance wall on insertion and deletion near the head of a large list. DLinked brings the doubly-linked list to Ruby with an idiomatic interface and O(1) operations at both ends."
tags: [Ruby, Data Structures, Performance, Open Source]
---

## Introduction: beyond the Array

When we think of data structures in Ruby, the first thing that comes to mind is the Array. It's powerful, flexible, and fast for most tasks. But what happens when you need blazing-fast insertion or deletion, especially near the beginning or middle of a large list? That's where the standard Ruby Array hits a performance wall.

Introducing [DLinked](https://github.com/danielefrisanco/dlinked), a small but mighty Ruby gem that brings the classic doubly-linked list data structure into your toolkit. It's designed to solve specific performance problems that arrays struggle with, offering a clean, idiomatic Ruby interface.

## What is a doubly-linked list? (And why it matters)

Unlike an Array, which stores elements sequentially in memory, a doubly-linked list stores elements as "Nodes." Each node holds the data and has two crucial pointers:

- A pointer to the next node.
- A pointer to the previous node.

This simple difference leads to a profound performance advantage. Here is a quick look at the time complexity comparison between DLinked and the standard Ruby Array:

| Operation | DLinked (linked list) | Ruby Array |
|-----------|-----------------------|------------|
| Insertion at head/tail | O(1) — constant time | O(N) — linear time |
| Deletion (if node is found) | O(1) — constant time | O(N) — linear time |
| Access/lookup by index | O(N) — linear time | O(1) — constant time |

## The pro of DLinked: speed where arrays slow down

DLinked is built for two primary use cases where array performance degrades rapidly:

### 1. High-speed queues (FIFO) and stacks (LIFO)

If you are constantly adding and removing elements from the ends of a list (like managing a job queue or a command history), the DLinked gem shines. Because it only needs to update two pointers at the head or tail, operations are instantaneous regardless of the list size.

### 2. Custom caches and priority management

Linked lists are the foundation of sophisticated data structures like Least Recently Used (LRU) caches. With DLinked, you can:

- Insert a new item in O(1).
- Remove an old item (the tail) in O(1).
- Quickly "promote" an accessed item to the head in O(1) time by adjusting its two surrounding pointers.

This makes DLinked a perfect starting point for building your own high-performance, in-memory cache solution in Ruby.

## How to get started

The `dlinked` gem is easy to install and use:

```
gem install dlinked
```

You can then initialize and use it like a standard collection:

```ruby
list = DLinked::List.new
list.prepend(10) # List: [10] (Fast O(1) insertion)
list.append(20)  # List: [10, 20] (Fast O(1) insertion)

# Use #first, #last, #shift, and #pop for O(1) queue/stack operations:
list.shift       # Returns 10, List: [20]
list.pop         # Returns 20, List: []

list.prepend(30) # List: [30]
# DLinked::List: [30]
```

## Resources and links

Ready to integrate the power of the doubly-linked list into your next Ruby project?

- [View the source code and contribute on GitHub](https://github.com/danielefrisanco/dlinked)
- [Detailed class and method documentation](https://danielefrisanco.github.io/dlinked/)
- [Install the gem from RubyGems](https://rubygems.org/gems/dlinked)

Happy coding!
