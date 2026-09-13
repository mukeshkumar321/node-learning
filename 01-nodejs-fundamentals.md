# Node.js Fundamentals ⭐⭐⭐

## Topics Covered

- [1. What Is Node.js?](#1-what-is-nodejs)
- [2. Node.js Architecture](#2-nodejs-architecture)
- [3. V8 Engine](#3-v8-engine)
- [4. Single-Threaded and Non-Blocking I/O](#4-single-threaded-and-non-blocking-io)
- [5. Synchronous vs Asynchronous](#5-synchronous-vs-asynchronous)
- [6. Node.js Use Cases](#6-nodejs-use-cases)

---

## 1. What Is Node.js?

**Node.js is a JavaScript runtime environment that allows JavaScript to run
outside the browser.**

Normally:

```text
JavaScript
    ↓
Browser
    ↓
Chrome / Firefox / Edge
```

With Node.js:

```text
JavaScript
    ↓
Node.js Runtime
    ↓
Operating System
```

Node.js is built on Google's **V8 JavaScript engine**.

### Why was Node.js created?

JavaScript was originally designed primarily for browsers. Node.js made it
possible to use JavaScript for:

- Backend APIs
- Web servers
- CLI applications
- Real-time applications
- Microservices
- Automation scripts

### Important distinction

```text
JavaScript → Programming language
Node.js    → Runtime environment
V8         → JavaScript engine
Express.js → Web framework built for Node.js
```

Node.js is not a programming language and it is not a framework.

### Practical example

This is a complete Node.js HTTP server without Express:

```js
const http = require("node:http");

const server = http.createServer((request, response) => {
  response.writeHead(200, { "content-type": "application/json" });
  response.end(JSON.stringify({ message: "Hello from Node.js" }));
});

server.listen(3000, () => {
  console.log("Server running on http://localhost:3000");
});
```

Run it with:

```bash
node server.js
```

### Interview answer

> Node.js is a JavaScript runtime built on Chrome's V8 engine that allows
> JavaScript to execute outside the browser. It uses an event-driven,
> non-blocking I/O model, making it well suited for I/O-intensive applications.

---

## 2. Node.js Architecture

A simplified Node.js architecture looks like this:

```text
             Your JavaScript Code
                     │
                     ▼
                Node.js APIs
             (fs, http, crypto)
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
         V8                  libuv
   JavaScript Engine      Event loop and I/O
          │                     │
          └──────────┬──────────┘
                     ▼
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
     OS / Kernel           Thread Pool
```

There are several important pieces.

## V8

V8 executes JavaScript code. It also manages memory and garbage collection.

## Node.js APIs

Node.js provides APIs that are not available in browser JavaScript in the same
way:

```js
const fs = require("node:fs");
const http = require("node:http");
const path = require("node:path");
const crypto = require("node:crypto");
```

## Event loop

The event loop allows Node.js to handle asynchronous operations without
blocking the main JavaScript thread.

## libuv

**libuv is a C library used by Node.js to provide the event loop and
cross-platform asynchronous I/O.**

It also provides a worker pool for operations that cannot be done
asynchronously by the OS, such as:

- Some `fs` operations (e.g. `fs.readFile`, `fs.stat`)
- `dns.lookup`
- CPU-heavy `crypto` functions (`pbkdf2`, `scrypt`, `randomBytes`)
- `zlib` compression

The worker pool defaults to **4 threads** and is configurable with an
environment variable:

```bash
UV_THREADPOOL_SIZE=8 node server.js
```

If an app fires off more than 4 concurrent thread-pool-bound operations (say,
6 `fs.readFile` calls at once), only 4 run at a time — the rest queue until a
thread frees up. This is a common follow-up question once someone claims
"Node.js handles concurrent I/O."

You do not need to memorize the internal C implementation. You should
understand the responsibility of each component.

### What happens during a database request?

```text
Request arrives
    ↓
JavaScript handler starts
    ↓
Database operation is requested
    ↓
Node.js handles other work
    ↓
Database result is ready
    ↓
Callback / promise continuation runs
    ↓
Response is sent
```

The JavaScript handler does not sit idle waiting for the database. This is why
one Node.js process can handle many concurrent I/O requests.

---

## 3. V8 Engine

**V8 is Google's open-source JavaScript engine.**

It was originally developed for Google Chrome and is also used by Node.js.

Its primary job is:

```text
JavaScript Code
      ↓
      V8
      ↓
Executable instructions
```

For example:

```js
const x = 10;
const y = 20;

console.log(x + y);
```

V8 parses and executes this JavaScript.

### What V8 also manages

- Memory allocation
- Garbage collection
- Runtime optimization
- Execution of JavaScript functions

### Important distinction

Do not say:

> V8 is Node.js.

Instead:

```text
Node.js
 ├── V8
 ├── libuv
 ├── Node.js APIs
 └── Other runtime components
```

### Practical memory example

Objects that are no longer referenced can eventually be garbage collected:

```js
function createTemporaryData() {
  return {
    createdAt: Date.now(),
    values: new Array(100_000).fill("data"),
  };
}

createTemporaryData();
```

However, keeping references can cause a memory leak:

```js
const requests = [];

setInterval(() => {
  requests.push({
    receivedAt: Date.now(),
    payload: "This remains referenced",
  });
}, 1000);
```

The array keeps growing, so the objects cannot be collected.

Common sources of memory leaks include:

- Unbounded arrays or caches
- Global references
- Event listeners that are never removed
- Closures retaining large objects

Useful diagnostic commands:

```bash
node --inspect server.js
node --trace-gc server.js
```

### Interview answer

> V8 is Google's open-source JavaScript engine, written primarily in C++. It
> executes JavaScript code and is used by both Chrome and Node.js.

---

## 4. Single-Threaded and Non-Blocking I/O

## Single-threaded

Node.js executes JavaScript callbacks primarily on a **single main thread**.

```js
console.log("A");
console.log("B");
console.log("C");
```

Output:

```text
A
B
C
```

But saying that Node.js is completely single-threaded is not accurate.

```text
Main JavaScript thread
        │
        ├── Executes JavaScript
        ├── Runs callbacks
        └── Runs promise continuations

libuv / operating system
        │
        ├── Handles suitable asynchronous I/O
        └── Uses a worker pool for some operations
```

Worker Threads can also be used when an application needs parallel JavaScript
execution for CPU-heavy work.

## Non-blocking I/O

Suppose the application needs to read a large file.

Blocking approach:

```text
Read file
   ↓
Wait
   ↓
File completed
   ↓
Continue
```

Non-blocking approach:

```text
Start file read ─────────────► OS / libuv
       │
       └── Continue other JavaScript work

When the file completes ─────► Callback runs
```

### Practical example

```js
const fs = require("node:fs");

console.log("1. Before reading");

fs.readFile("large-file.txt", "utf8", (error, contents) => {
  if (error) {
    console.error(error);
    return;
  }

  console.log("3. File length:", contents.length);
});

console.log("2. After starting read");
```

Typical output:

```text
1. Before reading
2. After starting read
3. File length: ...
```

Node.js starts the file operation and continues executing JavaScript instead
of waiting synchronously.

### What blocks Node.js?

Any long-running JavaScript operation blocks the main thread:

```js
function expensiveCalculation() {
  let result = 0;

  for (let index = 0; index < 5_000_000_000; index += 1) {
    result += index;
  }

  return result;
}
```

While this function runs, other callbacks cannot execute in that process.

### Important interview clarification

Do not say:

> Node.js is single-threaded, so it cannot do multiple things at once.

Say:

> JavaScript callbacks run primarily on the main thread, while Node.js uses
> the event loop, operating-system APIs, and the libuv worker pool to handle
> concurrent I/O work.

---

## 5. Synchronous vs Asynchronous

## Synchronous

Synchronous code waits for the current operation to finish:

```js
const fs = require("node:fs");

const data = fs.readFileSync("data.txt", "utf8");

console.log(data);
console.log("Done");
```

Execution:

```text
Read file
   ↓
WAIT
   ↓
Read complete
   ↓
console.log(data)
   ↓
console.log("Done")
```

During `readFileSync`, the event loop cannot process other JavaScript
callbacks.

## Asynchronous

Asynchronous code starts an operation and handles its result later:

```js
const fs = require("node:fs");

fs.readFile("data.txt", "utf8", (error, data) => {
  if (error) {
    console.error(error);
    return;
  }

  console.log(data);
});

console.log("Done");
```

Execution:

```text
Start reading
     │
     ├──────────────► File system
     │
     ▼
console.log("Done")
     │
     ▼
File completed
     │
     ▼
Callback executes
```

### Promise example

```js
const { readFile } = require("node:fs/promises");

async function loadConfig() {
  const data = await readFile("config.json", "utf8");
  return JSON.parse(data);
}

loadConfig()
  .then((config) => console.log(config))
  .catch((error) => console.error("Could not load config:", error));
```

`async/await` makes asynchronous code easier to read, but it does not make
CPU-heavy code run on another thread.

### Synchronous vs asynchronous in APIs

Prefer asynchronous APIs inside request handlers because synchronous APIs can
increase latency for every request sharing that process.

Synchronous APIs can be reasonable for:

- Small startup configuration reads
- One-time initialization
- Controlled command-line scripts

Asynchronous APIs are preferred for:

- Database calls
- File operations during requests
- External HTTP calls
- Network operations

### Concurrency vs parallelism

- **Concurrency:** multiple operations make progress during overlapping time.
- **Parallelism:** multiple operations execute at the same instant on different
  threads or CPU cores.

Node.js commonly provides concurrency for I/O. Worker Threads can provide
parallel JavaScript execution for CPU-heavy work.

---

## 6. Node.js Use Cases

Node.js is particularly good for **I/O-intensive applications**.

## REST APIs

```text
Frontend
   ↓
Node.js API
   ↓
Database
```

## Real-time applications

Examples:

- Chat applications
- Notifications
- Live dashboards
- Collaboration tools

## Scaling across CPU cores

A single Node.js process only uses one CPU core for executing JavaScript.
To take advantage of multi-core machines, run multiple Node.js processes:

```js
const cluster = require("node:cluster");
const os = require("node:os");

if (cluster.isPrimary) {
  os.cpus().forEach(() => cluster.fork());
} else {
  require("./server");
}
```

The built-in `cluster` module forks one worker process per core, and each
worker shares the same server port. This is a common answer to "Node.js is
single-threaded, so how does it use multiple cores?" — the answer is
multiple processes, not multiple threads executing the same JS.

## Microservices

```text
API Gateway
     ↓
Node.js Services
 ┌──────────────┬──────────────┐
 │ User Service │ Order Service│
 └──────────────┴──────────────┘
```

## Streaming

Node.js streams are useful for:

- Large files
- Video and audio
- Data processing
- Uploads and downloads

## CLI tools and automation

Node.js can also be used to build:

- Project generators
- Build tools
- Deployment scripts
- File-processing utilities

## When Node.js is not ideal

Node.js is less suitable when CPU-intensive work runs directly on the main
JavaScript thread:

- Large image processing
- Video encoding
- Complex mathematical calculations
- Machine-learning inference
- Long-running CPU-heavy algorithms

Possible solutions include:

- Worker Threads
- Child Processes
- Background jobs and queues
- A separate service designed for the workload

Node.js can still remain the API layer; the CPU-heavy work should not block the
request-handling event loop.

---

# ⭐ The Most Important Mental Model

Remember this:

```text
                 NODE.JS
                    │
             ┌──────┴──────┐
             │             │
            V8           libuv
             │             │
      Executes JS      Async I/O
                           │
                    ┌──────┴──────┐
                    │             │
               Event Loop    Thread Pool
```

Overall request flow:

```text
Request
   ↓
Node.js handler
   ↓
JavaScript / Event Loop
   ↓
I/O operation
   ↓
libuv / OS
   ↓
Operation completes
   ↓
Callback / Promise
   ↓
Response
```

The main limitation is equally important:

```text
Long-running JavaScript
          ↓
Main thread blocked
          ↓
Other callbacks wait
          ↓
Request latency increases
```

---

# 🎯 Interview Questions

After finishing this chapter, you should be able to answer these without
looking at your notes:

1. What is Node.js?
2. Is Node.js a language or a runtime?
3. What is V8?
4. Why is Node.js called single-threaded?
5. Is Node.js completely single-threaded?
6. What does non-blocking I/O mean?
7. What is the difference between synchronous and asynchronous code?
8. What is the event loop's role?
9. What is libuv?
10. How does Node.js handle concurrent requests?
11. Why is Node.js good for I/O-intensive applications?
12. When should you avoid running work directly in Node.js?
13. Does `async/await` make CPU-heavy code non-blocking?

## One-line interview summary

> Node.js is a V8-based JavaScript runtime with an event-driven, non-blocking
> I/O architecture. JavaScript runs primarily on a single main thread, while
> asynchronous I/O is coordinated through the event loop, libuv, the operating
> system, and its worker pool, allowing Node.js to efficiently handle many
> concurrent I/O operations.
