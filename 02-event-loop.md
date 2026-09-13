# Node.js Event Loop ⭐⭐⭐

## Topics Covered

- [1. Call Stack](#1-call-stack)
- [2. Event Loop](#2-event-loop)
- [3. Callback Queue](#3-callback-queue)
- [4. Microtasks vs Macrotasks](#4-microtasks-vs-macrotasks)
- [5. Node.js Event Loop Phases](#5-nodejs-event-loop-phases)
- [6. process.nextTick()](#6-processnexttick)
- [7. setImmediate()](#7-setimmediate)
- [8. setTimeout()](#8-settimeout)
- [9. setTimeout() vs setImmediate()](#9-settimeout-vs-setimmediate)
- [10. Very Important Interview Example](#10-very-important-interview-example)

---

The most important thing to understand first:

> **Node.js runs JavaScript on a single main thread, but it can handle many
> asynchronous operations without blocking that thread.**

The **Event Loop** is the mechanism that coordinates this asynchronous
execution.

---

## 1. Call Stack

The **Call Stack** keeps track of the JavaScript functions currently being
executed.

Example:

```js
function first() {
  second();
}

function second() {
  console.log("Hello");
}

first();
```

Execution:

```text
Call Stack

┌─────────────┐
│   second()  │
├─────────────┤
│   first()   │
├─────────────┤
│   global    │
└─────────────┘
```

JavaScript follows **LIFO**:

> Last In → First Out

So:

```text
first()
   ↓
second()
   ↓
console.log()
```

`second()` finishes first, then `first()`.

### Interview point

**Q: What is the Call Stack?**

> The Call Stack is a data structure used by JavaScript to keep track of
> function execution. Functions are pushed onto the stack when called and
> removed when they complete.

---

## 2. Event Loop

Now the important part.

Suppose:

```js
console.log("A");

setTimeout(() => {
  console.log("B");
}, 0);

console.log("C");
```

Output:

```text
A
C
B
```

You might ask:

> If timeout is `0`, why doesn't B execute immediately?

Because `setTimeout()` is asynchronous.

Conceptually:

```text
                 ┌─────────────────┐
                 │   Call Stack    │
                 └────────┬────────┘
                          │
                          ↓
                 ┌─────────────────┐
                 │   Event Loop    │
                 └────────┬────────┘
                          │
             ┌────────────┴────────────┐
             ↓                         ↓
       Microtask Queue           Callback Queue
             │                         │
       Promise callbacks        setTimeout callback
       process.nextTick()
```

The Event Loop keeps checking whether the Call Stack is free and determines
which queued callbacks can run.

### Important

`setTimeout(fn, 0)` means:

> **Run this callback after the minimum timer delay and when the event loop
> is able to execute it.**

It does **not** mean "run immediately."

---

# 3. Callback Queue

When an asynchronous callback becomes ready, it can wait in a queue until
JavaScript is ready to execute it.

Example:

```js
setTimeout(() => {
  console.log("Timer");
}, 0);

console.log("End");
```

Output:

```text
End
Timer
```

Conceptually:

```text
setTimeout()
     ↓
Timer expires
     ↓
Callback becomes eligible
     ↓
Queue
     ↓
Event Loop
     ↓
Call Stack
     ↓
console.log("Timer")
```

---

## 4. Microtasks vs Macrotasks ⭐⭐⭐

This is **very important for interviews**.

### Microtasks

Common examples:

```text
Promise.then()
Promise.catch()
Promise.finally()
queueMicrotask()
process.nextTick() // Node-specific, special priority
```

### Macrotask / task examples

```text
setTimeout()
setInterval()
setImmediate()
I/O callbacks
```

The simplified priority is:

```text
Synchronous code
      ↓
process.nextTick()
      ↓
Microtasks
      ↓
Macrotasks / Event Loop phases
```

Consider:

```js
console.log("1");

setTimeout(() => {
  console.log("2");
}, 0);

Promise.resolve().then(() => {
  console.log("3");
});

console.log("4");
```

Output:

```text
1
4
3
2
```

Why?

### Step 1 — synchronous code

```text
1
4
```

### Step 2 — microtask

```text
3
```

### Step 3 — timer callback

```text
2
```

Therefore:

> **Microtasks generally execute before the next macrotask/event-loop phase
> continues.**

---

# 5. Node.js Event Loop Phases ⭐⭐⭐

Node.js has several event-loop phases.

A simplified representation:

```text
        ┌───────────────┐
        │    Timers     │
        └───────┬───────┘
                ↓
        ┌───────────────┐
        │ Pending       │
        │ Callbacks     │
        └───────┬───────┘
                ↓
        ┌───────────────┐
        │     Idle /    │
        │    Prepare    │
        └───────┬───────┘
                ↓
        ┌───────────────┐
        │      Poll     │
        └───────┬───────┘
                ↓
        ┌───────────────┐
        │     Check     │
        │ setImmediate  │
        └───────┬───────┘
                ↓
        ┌───────────────┐
        │     Close     │
        │   Callbacks   │
        └───────────────┘
                ↓
              repeat
```

The important phases for interviews are:

| Phase | Purpose |
| --- | --- |
| **Timers** | `setTimeout()` / `setInterval()` callbacks |
| **Pending callbacks** | Certain deferred I/O callbacks |
| **Poll** | Retrieve/process I/O events |
| **Check** | `setImmediate()` callbacks |
| **Close callbacks** | Close events such as socket close |

You don't need to memorize every internal detail initially.

Focus on:

```text
Timers → Poll → Check
```

---

## 6. `process.nextTick()` ⭐⭐⭐

Node.js provides:

```js
process.nextTick();
```

Example:

```js
console.log("A");

process.nextTick(() => {
  console.log("B");
});

console.log("C");
```

Output:

```text
A
C
B
```

`process.nextTick()` schedules a callback to execute **after the current
operation completes, before the event loop proceeds to the next phase**.

### Important distinction

`process.nextTick()` is **not technically an event-loop phase**.

Node maintains a special **nextTick queue**.

Conceptually:

```text
Current synchronous operation
          ↓
process.nextTick() queue
          ↓
Microtask queue
          ↓
Event loop
```

Be careful with wording here: Node's exact scheduling behavior has nuances,
but for interviews this priority model is the useful mental model.

---

# 7. `setImmediate()` ⭐⭐⭐

`setImmediate()` schedules a callback for the **Check phase**.

```js
setImmediate(() => {
  console.log("Immediate");
});
```

It is particularly useful when working with I/O.

For example:

```js
const fs = require("node:fs");

fs.readFile(__filename, () => {
  setImmediate(() => {
    console.log("Immediate");
  });
});
```

The callback can run in the **Check phase after the poll phase**.

---

# 8. `setTimeout()`

Example:

```js
setTimeout(() => {
  console.log("Timeout");
}, 0);
```

This schedules a timer callback.

Important:

```js
setTimeout(fn, 0);
```

doesn't mean:

```text
Execute immediately
```

It means approximately:

```text
Don't execute before the timer threshold.
Execute when the event loop gets to the timer callback.
```

Also, actual execution can be delayed if the Call Stack is busy.

---

## 9. `setTimeout()` vs `setImmediate()`

This is a **classic Node.js interview question**.

### Outside an I/O callback

Example:

```js
setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});
```

The order can be **non-deterministic** depending on timing.

You should **not** confidently say:

```text
timeout
immediate
```

every time.

### Inside an I/O callback

Example:

```js
const fs = require("node:fs");

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log("timeout");
  }, 0);

  setImmediate(() => {
    console.log("immediate");
  });
});
```

Typically:

```text
immediate
timeout
```

Why?

Because after the I/O callback runs in the **poll phase**, Node proceeds to
the **check phase**, where `setImmediate()` callbacks execute.

```text
Poll
 ↓
setImmediate()
 ↓
Timers
 ↓
setTimeout()
```

---

# 10. Very Important Interview Example ⭐⭐⭐

Understand this completely:

```js
console.log("1");

setTimeout(() => {
  console.log("2");
}, 0);

setImmediate(() => {
  console.log("3");
});

Promise.resolve().then(() => {
  console.log("4");
});

process.nextTick(() => {
  console.log("5");
});

console.log("6");
```

The guaranteed initial part is:

```text
1
6
5
4
```

Then the event-loop callbacks execute.

The relative ordering of `setTimeout(..., 0)` and `setImmediate()` **when
scheduled from the main module** should not be treated as universally
guaranteed.

So don't memorize:

```text
1
6
5
4
2
3
```

as an absolute rule.

Instead remember:

```text
Synchronous
    ↓
nextTick
    ↓
microtasks
    ↓
event-loop callbacks
```

and understand that `setImmediate()` vs `setTimeout(0)` depends on context.

---

# 🧠 The Mental Model You Should Remember

Think of Node.js like this:

```text
             JavaScript
                 │
                 ↓
          ┌─────────────┐
          │ Call Stack  │
          └──────┬──────┘
                 │
        synchronous execution
                 │
                 ↓
        ┌─────────────────┐
        │ Async operation │
        └────────┬────────┘
                 │
          ┌──────┴───────┐
          ↓              ↓
     nextTick        Microtasks
      Queue             Queue
          │              │
          └──────┬───────┘
                 ↓
          Event Loop
                 │
       ┌─────────┼─────────┐
       ↓         ↓         ↓
    Timers      Poll      Check
       │         │         │
 setTimeout     I/O    setImmediate
```

---

# 🎯 Interview Questions

After studying this chapter, make sure you can explain:

1. What is the Event Loop?
2. Why is Node.js called single-threaded?
3. What is the Call Stack?
4. What happens when Node encounters asynchronous code?
5. What is the Callback Queue?
6. What are microtasks and macrotasks?
7. Which executes first: Promise or `setTimeout()`?
8. What is `process.nextTick()`?
9. How is `process.nextTick()` different from Promise microtasks?
10. What is `setImmediate()`?
11. What is the difference between `setImmediate()` and `setTimeout()`?
12. What are Node.js event-loop phases?
13. What is the Poll phase?
14. Why doesn't `setTimeout(fn, 0)` execute immediately?
15. Predict the output of asynchronous Node.js code.

## ⭐ Most important

For your interview preparation, spend the most time on these four:

```text
1. Call Stack + Event Loop
2. Microtasks vs Macrotasks
3. process.nextTick()
4. setTimeout() vs setImmediate()
```

Once these are clear, **Node.js asynchronous behavior becomes much easier to
reason about.**
