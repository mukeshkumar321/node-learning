# Performance & Scaling ⭐⭐⭐

## Topics Covered

- [1. Event-Loop Blocking](#1-event-loop-blocking)
- [2. CPU-Bound vs I/O-Bound Tasks](#2-cpu-bound-vs-io-bound-tasks)
- [3. Memory Leaks](#3-memory-leaks)
- [4. Caching](#4-caching)
- [5. Redis Basics](#5-redis-basics)
- [6. Streams and Backpressure](#6-streams-and-backpressure)
- [7. Worker Threads](#7-worker-threads)
- [8. Cluster](#8-cluster)
- [9. Horizontal Scaling](#9-horizontal-scaling)

---

This is an **important Node.js interview chapter**. It covers every point in
the list, with interview-focused explanations and examples.

---

## 1. Event-Loop Blocking ⭐⭐⭐

You should know:

- What event-loop blocking means
- Why blocking is dangerous in Node.js
- Synchronous vs asynchronous operations
- CPU-intensive operations that block the event loop
- Common blocking APIs:
  - `fs.readFileSync()`
  - `crypto` synchronous APIs
  - large loops
  - expensive JSON operations
- How to identify blocking code
- How to avoid blocking
- Moving heavy work to Worker Threads

**Interview questions:**

- Why is blocking the event loop bad?
- Give an example of event-loop blocking.
- How would you identify and fix event-loop blocking?

---

## 2. CPU-Bound vs I/O-Bound Tasks ⭐⭐⭐

Understand the difference between:

### I/O-bound

Tasks waiting for external resources:

- Database queries
- File operations
- HTTP requests
- Network operations

Node.js is **very good at handling I/O-bound work** because of its
asynchronous, non-blocking model.

### CPU-bound

Tasks requiring significant CPU processing:

- Large calculations
- Image/video processing
- Encryption
- Data processing
- Large loops

These can block the event loop.

**Interview question:**

> Why is Node.js suitable for I/O-heavy applications but less suitable for
> CPU-heavy tasks?

You should be able to answer this clearly.

---

## 3. Memory Leaks ⭐⭐⭐

Know:

- What a memory leak is
- How memory is managed in Node.js
- Garbage collection basics
- Common causes of memory leaks:
  - Global variables
  - Unremoved event listeners
  - Timers
  - Large objects kept in memory
  - Closures
  - Caches without limits
- Symptoms of memory leaks
- How to investigate memory leaks
  - `process.memoryUsage()`
  - Heap snapshots
  - Chrome DevTools / Node.js profiling basics

**Important interview concept:**

> Garbage collection does not mean memory leaks cannot happen.

If your application still holds references to objects that are no longer
needed, garbage collection cannot remove them.

---

## 4. Caching ⭐⭐⭐

Understand:

- What caching is
- Why caching improves performance
- Cache hit vs cache miss
- In-memory caching
- External caching
- Cache invalidation
- TTL — Time To Live
- Cache-aside pattern
- When caching should/shouldn't be used

Example:

```text
Request
   ↓
Check Cache
   ↓
Cache Hit → Return data
   ↓
Cache Miss
   ↓
Database
   ↓
Store in Cache
   ↓
Return data
```

**Interview questions:**

- Why use caching?
- What is cache invalidation?
- What is TTL?
- What is cache-aside?
- What happens when cached data becomes stale?

---

## 5. Redis Basics ⭐⭐⭐

You don't need to become a Redis expert for a Node.js interview.

Know:

- What Redis is
- Why Redis is fast
- In-memory data store
- Key-value concept
- Common data types:
  - String
  - List
  - Set
  - Hash
- TTL / expiration
- Caching with Redis
- Redis for sessions
- Redis for rate limiting
- Redis in distributed applications
- Basic Node.js + Redis usage

Typical architecture:

```text
Node.js
   ↓
Redis
   ↓
Database
```

Know why Redis is often placed between the application and database.

---

## 6. Streams and Backpressure ⭐⭐⭐

Streams are an important Node.js interview topic.

Understand:

- What streams are
- Why streams are useful
- Processing data chunk-by-chunk
- Streams vs loading entire data into memory
- Types of streams:
  - Readable
  - Writable
  - Duplex
  - Transform
- `pipe()`
- Backpressure
- Practical examples

Example:

Instead of:

```text
Large File
     ↓
Load Entire File
     ↓
Memory
```

Streams allow:

```text
Large File
 ↓
Chunk
 ↓
Process
 ↓
Chunk
 ↓
Process
```

This reduces memory usage.

**Interview questions:**

- What are streams in Node.js?
- Why are streams useful?
- What is backpressure?
- Difference between readable and writable streams?
- What is a Transform stream?

---

## 7. Worker Threads ⭐⭐⭐

Understand:

- Why Worker Threads exist
- Worker Threads vs main thread
- CPU-intensive tasks
- `worker_threads`
- `Worker`
- `parentPort`
- Sending messages between threads
- When to use Worker Threads
- When not to use them

Architecture:

```text
Main Thread
     │
     ├── Normal API requests
     │
     └── CPU-heavy task
              ↓
        Worker Thread
```

Example use cases:

- Image processing
- Large calculations
- Encryption
- Data processing

**Key interview point:**

> Worker Threads are mainly useful for CPU-intensive JavaScript work that
> would otherwise block the event loop.

---

## 8. Cluster ⭐⭐⭐

Understand:

- What Node.js Cluster is
- Why Cluster exists
- Multiple Node.js processes
- Using multiple CPU cores
- Primary process
- Worker processes
- Load distribution
- Cluster vs Worker Threads

Basic architecture:

```text
                Primary
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    Worker 1    Worker 2    Worker 3
       │           │           │
       └────────── API ────────┘
```

**Important distinction:**

| Cluster | Worker Threads |
| --- | --- |
| Multiple processes | Multiple threads |
| Separate memory | Can share memory via `SharedArrayBuffer` |
| Useful for scaling Node processes | Useful for CPU-heavy tasks |
| Uses multiple CPU cores | Uses multiple CPU cores |

You should be able to explain this difference in an interview.

---

## 9. Horizontal Scaling ⭐⭐⭐

Understand:

- Vertical scaling
- Horizontal scaling
- Load balancing
- Multiple Node.js instances
- Stateless applications
- Shared session storage
- Redis in distributed systems
- Database considerations
- Reverse proxy/load balancer basics

### Vertical scaling

```text
1 Node.js server
      ↓
More CPU / RAM
```

### Horizontal scaling

```text
             Load Balancer
              /    |    \
             ↓     ↓     ↓
          Node 1 Node 2 Node 3
             \     |     /
                Redis
                  ↓
              Database
```

**Interview questions:**

- What is horizontal scaling?
- Horizontal vs vertical scaling?
- How would you scale a Node.js application?
- Why should horizontally scaled applications generally be stateless?
- Where would you store sessions when you have multiple Node.js servers?

---

## ⭐ What You Must Be Able to Explain

After completing this chapter, you should confidently answer this scenario:

> **"Your Node.js API is getting slow as traffic increases. How would you
> improve its performance and scale it?"**

A strong answer should touch on:

```text
1. Identify event-loop blocking
        ↓
2. Separate CPU-bound and I/O-bound work
        ↓
3. Optimize database queries
        ↓
4. Add caching
        ↓
5. Use Redis where appropriate
        ↓
6. Use streams for large data
        ↓
7. Move CPU-heavy work to Worker Threads
        ↓
8. Run multiple Node.js processes/instances
        ↓
9. Load balance traffic
        ↓
10. Horizontally scale
```

### Priority for interviews

**Must know very well:**

- Event-loop blocking
- CPU-bound vs I/O-bound
- Caching
- Redis basics
- Streams
- Worker Threads
- Cluster
- Horizontal scaling

**Good supporting knowledge:**

- Memory leaks
- Backpressure
- Load balancing
- Stateless architecture
- Cache invalidation

This chapter is especially important because interviewers often combine
**Node.js internals + performance + system design** into one scenario.
