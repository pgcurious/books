# Inventing the LSM Tree

> A First-Principles Journey to One of Computing's Most Elegant Data Structures

## Overview

This mini-book takes you on a journey of discovery: **inventing the Log-Structured Merge (LSM) Tree from scratch**, as if it had never existed. Rather than explaining what LSM trees are, we'll derive them step-by-step through a natural problem-solving process, building intuition along the way.

## The Learning Philosophy

This book implements a novel learning approach: **guided invention**. Instead of:
- "Here's an LSM tree, let me explain how it works"

We ask:
- "Let's build a database. What problems will we encounter? How would we solve them?"

By following this thought process, you'll understand not just *what* LSM trees are, but *why* they must be this way—and you'll develop problem-solving intuition applicable far beyond this specific data structure.

## Core Journey

1. **The Initial Challenge**: We need to store and retrieve data
2. **First Attempt**: Simple in-memory solutions
3. **The Persistence Problem**: Making data survive crashes
4. **The Performance Wall**: Discovering that random writes are expensive
5. **The Sequential Insight**: Append-only logs are fast!
6. **The Read Problem**: How to find data quickly in a log
7. **The Memory Solution**: Keeping recent data accessible
8. **The Overflow Question**: What happens when memory fills?
9. **The Sorted Structure**: Discovering the power of order
10. **The Merge Revelation**: Combining files efficiently
11. **The Level Insight**: Organizing for optimal performance
12. **The LSM Tree**: Arriving at the complete solution

## Key Concepts Discovered

Throughout this journey, we'll naturally discover:

- **Write Amplification**: Why writing data multiple times can be acceptable
- **Read Amplification**: The cost of searching multiple locations
- **Trade-off Spaces**: How to navigate performance dimensions
- **Bloom Filters**: Probabilistic data structures for optimization
- **Compaction Strategies**: Different ways to organize data over time
- **Level-based Organization**: Why hierarchy emerges naturally

## What Makes This Book Different

Traditional explanations:
- Start with the solution
- Explain how it works
- You memorize the structure

This book:
- Starts with the problem
- Lets you discover the solution
- You understand why it must be this way

The result: **deep intuition** instead of **surface memorization**.

## Who This Book Is For

- Software engineers learning database internals
- Students studying data structures and algorithms
- Anyone curious about how modern databases work
- Developers who want to think from first principles
- People who prefer understanding "why" over memorizing "what"

## Prerequisites

You should be comfortable with:
- Basic data structures (arrays, hash tables, trees)
- Big-O notation and performance analysis
- The concept of disk vs. memory
- Basic file operations

No prior database knowledge required!

## Structure

The book is organized as a progressive discovery:

### Part 1: The Foundation (Chapters 1-3)
- The problem space
- First attempts and their limitations
- The performance reality of storage systems

### Part 2: The Breakthrough (Chapters 4-6)
- The append-only insight
- Solving the read problem
- Managing memory constraints

### Part 3: The Refinement (Chapters 7-9)
- Sorted structures and their power
- The merge operation
- Multi-level organization

### Part 4: The Complete Picture (Chapters 10-12)
- The full LSM tree structure
- Optimizations and variations
- Real-world applications

## Key Takeaways

After this journey, you'll understand:

- Why LSM trees optimize for write performance
- How they achieve competitive read performance
- The fundamental trade-offs in storage systems
- How to reason about data structure design from first principles
- The elegance of solving one problem at a time
- How constraints drive innovation

## Pedagogical Features

This book uses:

- **Semantic Formatting**: Visual hierarchy to guide attention
- **Progressive Complexity**: Each chapter builds naturally on the last
- **Problem-First Learning**: Encounter the problem before the solution
- **Concrete Examples**: Real numbers and scenarios, not abstractions
- **Thought Experiments**: "What if we tried...?" questions
- **Insight Boxes**: Highlighting key realizations
- **Trade-off Analysis**: Explicit cost-benefit reasoning

## Real-World Impact

LSM trees power many systems you use daily:
- **Apache Cassandra**: Distributed database
- **RocksDB**: Embedded key-value store (used by Facebook, LinkedIn)
- **LevelDB**: Google's storage library
- **HBase**: Hadoop's database
- **ScyllaDB**: High-performance NoSQL database

Understanding LSM trees means understanding a significant portion of modern data infrastructure.

## License

Creative Commons BY-SA 4.0

## Version

1.0 - November 2025

---

**Read the full book**: [book.md](./book.md)

**Repository**: [github.com/pgcurious/books](https://github.com/pgcurious/books)
