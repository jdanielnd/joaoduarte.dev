---
title: "Why I'm building a database from scratch (and why you should too)"
series: "How Databases Work: Building a PostgreSQL-Inspired DB in Rust"
date: 2025-09-17
draft: false
---

## **The production incident that ignited this idea**

A few months ago, we had a database performance crisis at work. It wasn't your typical "missing index" problem. The system would run fine for hours, handling thousands of queries across multiple indexes. Then, seemingly randomly, query times would degrade from milliseconds to seconds. The database wasn't out of memory. CPU wasn't maxed. Rows fetched peaking afterwards, which looked... weird, but not obviously wrong.

After days of investigation, we found the culprit hiding deep in PostgreSQL's internals. It was a combination of factors that created the perfect storm. We were able to handle it, but I walked away with an uncomfortable realization:

**I'd been using PostgreSQL for years, but I didn't actually understand how it worked.**

Sure, I knew the concepts. B+ trees, ACID properties, indexes. I could draw you a high-level diagram of how a query planner works. But when faced with real performance mysteries, my knowledge was too shallow. It was like being a pilot who knows which buttons to press but doesn't understand aerodynamics.

So I decided to fix that the only way I know how: by building my own database from scratch.

## **Why PostgreSQL?**

I've built simple key-value stores before. You know how it goes: throw a hash map at the problem, add some persistence, call it a day. They're fun weekend projects, but they don't teach you all the nuances that a large production database has to deal.

By following PostgreSQL's architecture, I'll learn:
- **Write-Ahead Logging (WAL)**: How databases survive crashes mid-write
- **Buffer Pool Management**: Why that 8GB of RAM matters more than disk speed
- **Page-based Storage**: The fundamental abstraction that makes everything work
- **System Catalogs**: How databases store information about themselves
- **Query Execution**: From SQL text to actual bytes on disk

Actually, these aren't PostgreSQL-specific ideas. They're the foundations of almost every relational database. Learn them once, understand databases forever.

## **Why Rust?**

I come from a web development background, so Ruby, Elixir, JavaScript, Python, the occasional Go service. These languages are fantastic for applications, but they hide the exact details I need to understand.

Rust forces you to think about things databases care about:

```rust
// In Python, this is just a string
data = "Hello, World!"

// In Rust, everything is explicit
let data: &[u8] = b"Hello, World!";  // Immutable bytes
let page: [u8; 8192] = [0; 8192];    // Exactly 8KB, stack allocated
let offset: usize = 42;               // Platform-specific size
unsafe {
    // Sometimes you need to cast bytes to structs
    let header = &*(page.as_ptr() as *const PageHeader);
}
```

This explicitness is perfect for database work. You're always thinking about:
- **Memory layout**: Is this struct packed correctly for disk storage?
- **Ownership**: Who owns this buffer? When can it be evicted?
- **Lifetimes**: How long does this page reference need to live?
- **Error handling**: Every disk operation can fail

Plus, Rust gives us modern tooling (cargo, great compiler errors, built-in testing) and safety guarantees that make development faster, not slower. No segfaults when manipulating page buffers. No data races when we eventually add concurrency.

Is Rust the "best" language for databases? Maybe not. But I think it's a good language for *learning* to build databases.

## **What we're building**

Meet **InoxDB**, the database we're building together! 

InoxDB is our PostgreSQL-inspired database written in Rust. The name? Stainless steel for a Rust project, because even our puns are over-engineered.

Over the next weeks, we'll build a real SQL database that:

### **Core Features**
- **Persistent Storage**: Your data survives process restarts  
- **SQL Interface**: Real CREATE TABLE, INSERT, SELECT, DELETE  
- **Crash Recovery**: Pull the power cord, don't lose committed data  
- **Buffer Pool**: Cache hot pages in memory for dramatic performance gains  
- **System Catalogs**: Database stores its own schema (it's meta all the way down)

### **Architecture Overview**
```
┌────────────────────────────────────────────┐
│              SQL Layer                     │
│ Parser (sqlparser-rs) → Planner → Executor │
└────────────────────┬───────────────────────┘
                     │
┌────────────────────▼───────────────────────┐
│          Access Methods                    │  
│   Sequential Scan, Index Scan (B+ Tree)    │
└────────────────────┬───────────────────────┘
                     │
┌────────────────────▼───────────────────────┐
│          Buffer Pool Manager               │
│   Page Cache, Eviction (LRU), Dirty Pages  │
└────────────────────┬───────────────────────┘
                     │
┌────────────────────▼───────────────────────┐
│           Storage Engine                   │
│   Heap Files, Pages (8KB), WAL, Records    │
└────────────────────────────────────────────┘
```

### **Smart decisions**
One key decision: we're using `sqlparser-rs` instead of building our own SQL parser. Why? Parsing is a fascinating but separate problem domain. By using a battle-tested parser, we can focus 100% on database internals: the storage engine, query execution, buffer management. That's where the real database magic happens.

### **Not building (yet)**
- Network protocol (we'll use a REPL)
- Concurrent transactions (single-writer is complex enough)
- Query optimization beyond basic index selection
- JOINs, views, stored procedures
- Replication or any distributed features

This scope is carefully chosen. Every feature teaches a fundamental concept without getting lost in complexity.

## **Demo: What InoxDB will look like**

Here's what we're working toward:

```bash
$ cargo run --release
InoxDB v0.1.0 (PostgreSQL-inspired database)
    
Type \h for help, \q to quit

inox> CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    department VARCHAR(50),
    salary INTEGER,
    active BOOLEAN DEFAULT true
);
CREATE TABLE
Time: 12ms

inox> INSERT INTO employees (id, name, department, salary, active) VALUES
    (1, 'Alice Johnson', 'Engineering', 95000, true),
    (2, 'Bob Smith', 'Sales', 65000, true),
    (3, 'Charlie Brown', 'Engineering', 85000, false);
INSERT 3
Time: 8ms

inox> SELECT name, salary FROM employees 
      WHERE department = 'Engineering' AND active = true
      ORDER BY salary DESC;
┌───────────────┬────────┐
│ name          │ salary │
├───────────────┼────────┤
│ Alice Johnson │ 95000  │
└───────────────┴────────┘
(1 row)
Time: 3ms

inox> \d employees
Table "employees"
┌────────────┬──────────────┬──────────┐
│ Column     │ Type         │ Nullable │
├────────────┼──────────────┼──────────┤
│ id         │ INTEGER      │ NOT NULL │
│ name       │ VARCHAR(100) │ NOT NULL │
│ department │ VARCHAR(50)  │ NULL     │
│ salary     │ INTEGER      │ NULL     │
│ active     │ BOOLEAN      │ NULL     │
└────────────┴──────────────┴──────────┘
Indexes:
  "employees_pkey" PRIMARY KEY (id)

inox> \q
Goodbye!
```

## **The Roadmap**

Each major milestone builds on the previous one:

**Milestone 1: Storage Foundations**  
Build heap files and page management. Watch your data survive a restart.

**Milestone 2: Write-Ahead Logging**  
Implement durability. Crash InoxDB mid-write and see it recover perfectly.

**Milestone 3: Buffer Pool Magic**  
Add caching. Watch query performance improve dramatically without changing query logic.

**Milestone 4: System Catalogs**  
Make the database self-aware. Store table definitions... in tables. (This one will bend your brain!)

**Milestone 5: SQL Comes Alive**  
Connect everything. Parse SQL, plan queries, execute them against real data.

## **Join the InoxDB journey**

Next week, we dive into the deep end:

**Storage Foundations: How to Persist a Row**

We'll answer questions like:
- Why do databases use fixed-size pages instead of just appending to a file?
- How do you store variable-length data (like strings) efficiently?
- What's a slot directory and why does every database have one?
- How do you handle deletes without actually deleting anything?

We'll write actual Rust code that takes a row and turns it into bytes on disk. Bytes that survive process restarts, system crashes, and even me accidentally deleting the wrong files (there will be stories).

The InoxDB code for each post will be available on [GitHub](https://github.com/jdanielnd/inoxdb), tagged by milestone so you can:
- Follow along and build your own version
- Jump to any milestone's starting point
- Submit issues when you find my inevitable bugs
- Contribute improvements (this is a learning project for all of us!)

**Ready to understand databases from the inside out? Welcome to the InoxDB project. See you next week.**

---

*Building InoxDB alongside me? Have questions or insights to share? Find me on [Twitter/X](https://twitter.com/jdanielnd) or jump into discussions on the [GitHub repo](https://github.com/jdanielnd/inoxdb). This is more fun when we learn together.*