# Asynchronous Programming ⭐⭐⭐

## Topics Covered

- [1. Callbacks](#1-callbacks)
- [2. Promises](#2-promises)
- [3. async/await](#3-asyncawait)
- [4. Promise.all()](#4-promiseall)
- [5. Promise.allSettled()](#5-promiseallsettled)
- [6. Promise.race() and Promise.any()](#6-promiserace-and-promiseany)
- [7. Sequential vs Parallel Execution](#7-sequential-vs-parallel-execution)
- [8. Async Error Handling](#8-async-error-handling)
- [9. Unhandled Rejections and Uncaught Exceptions](#9-unhandled-rejections-and-uncaught-exceptions)

---

This is a **very important Node.js interview chapter**. Since you already
covered the Event Loop, focus on understanding **how async operations are
written, executed, combined, and handled when they fail**.

---

## 1. Callbacks ⭐⭐⭐

A **callback** is a function passed to another function and executed later.

### Basic example

```js
function fetchData(callback) {
  setTimeout(() => {
    callback("Data received");
  }, 1000);
}

fetchData((data) => {
  console.log(data);
});
```

### Node.js callback pattern

Node traditionally uses the **error-first callback** pattern:

```js
fs.readFile("file.txt", "utf8", (err, data) => {
  if (err) {
    console.error(err);
    return;
  }

  console.log(data);
});
```

Remember:

```text
callback(error, result)
```

### Callback Hell

Nested callbacks become difficult to maintain:

```js
getUser((err, user) => {
  if (err) {
    console.error(err);
    return;
  }

  getOrders(user, (err, orders) => {
    if (err) {
      console.error(err);
      return;
    }

    getPayment(orders, (err, payment) => {
      if (err) {
        console.error(err);
        return;
      }

      console.log(payment);
    });
  });
});
```

This is one reason Promises and `async/await` became popular.

**Interview question:**

> What is callback hell and how can you avoid it?

Answer: Use Promises, `async/await`, and modularize asynchronous operations.

---

## 2. Promises ⭐⭐⭐

A Promise represents the eventual result of an asynchronous operation.

It has three states:

```text
Pending
   ↓
Fulfilled
   OR
Rejected
```

### Creating a Promise

```js
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Success");
  }, 1000);
});
```

### Consuming it

```js
promise
  .then((result) => {
    console.log(result);
  })
  .catch((error) => {
    console.error(error);
  })
  .finally(() => {
    console.log("Finished");
  });
```

### Important methods

```text
.then()
.catch()
.finally()
```

### Promise chaining

```js
getUser()
  .then((user) => getOrders(user.id))
  .then((orders) => getPayment(orders))
  .then((payment) => console.log(payment))
  .catch((error) => console.error(error));
```

**Interview focus:**

- Promise states
- `resolve()` vs `reject()`
- `.then()`
- `.catch()`
- `.finally()`
- Promise chaining

---

## 3. `async/await` ⭐⭐⭐

`async/await` provides cleaner syntax for working with Promises.

### Promise version

```js
getUser()
  .then((user) => getOrders(user.id))
  .then((orders) => console.log(orders));
```

### async/await version

```js
async function getData() {
  const user = await getUser();
  const orders = await getOrders(user.id);

  console.log(orders);
}
```

### Important rule

An `async` function **always returns a Promise**.

```js
async function test() {
  return "Hello";
}

console.log(test());
```

Conceptually:

```text
Promise.resolve("Hello")
```

### `await`

`await` waits for a Promise to settle **inside the async function**.

It does **not block the Node.js event loop**.

This is a very common interview question.

---

## 4. `Promise.all()` ⭐⭐⭐

Use when multiple independent async operations can run **in parallel** and
you need **all results**.

```js
const [users, products, orders] = await Promise.all([
  getUsers(),
  getProducts(),
  getOrders(),
]);
```

Conceptually:

```text
getUsers()    ────────┐
getProducts() ────────┼──→ Promise.all()
getOrders()   ────────┘
```

### Important behavior

If **one Promise rejects**, `Promise.all()` rejects.

```js
try {
  const results = await Promise.all([task1(), task2(), task3()]);
} catch (error) {
  console.log(error);
}
```

You don't get successful results from the other promises through the
`Promise.all()` result once one rejects.

---

## 5. `Promise.allSettled()` ⭐⭐

Use when you want the result of **every operation**, regardless of whether
individual operations succeed or fail.

```js
const results = await Promise.allSettled([task1(), task2(), task3()]);

console.log(results);
```

Example result:

```js
[
  { status: "fulfilled", value: "Task 1 done" },
  { status: "rejected", reason: "Task 2 failed" },
  { status: "fulfilled", value: "Task 3 done" },
];
```

### Key difference

| `Promise.all()` | `Promise.allSettled()` |
| --- | --- |
| Fails when one rejects | Waits for all |
| Gives results only if all succeed | Gives every result |
| Good when all operations are required | Good when operations are independent |
| Rejects on first rejection | Never rejects because of individual Promise rejection |

**Interview question:**

> When would you use `Promise.allSettled()` instead of `Promise.all()`?

When you need to know the outcome of **every operation**, even if some fail.

---

## 6. `Promise.race()` and `Promise.any()` ⭐⭐

These are less commonly used than `Promise.all()`/`Promise.allSettled()`, but
they come up often as follow-up interview questions.

### `Promise.race()`

Settles as soon as the **first** Promise settles — whether it fulfills or
rejects.

```js
const result = await Promise.race([
  fetchFromPrimary(),
  fetchFromBackup(),
]);
```

Use it for things like timeouts or "whichever source responds first":

```js
const timeout = (ms) =>
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timed out")), ms)
  );

const data = await Promise.race([fetchData(), timeout(2000)]);
```

If the fastest Promise rejects, `Promise.race()` rejects with that reason,
even if other Promises would have eventually fulfilled.

### `Promise.any()`

Settles as soon as the **first** Promise **fulfills**. It ignores rejections
unless every Promise rejects.

```js
const result = await Promise.any([
  fetchFromMirrorA(),
  fetchFromMirrorB(),
  fetchFromMirrorC(),
]);
```

If **all** Promises reject, `Promise.any()` rejects with an
`AggregateError` that contains all the individual rejection reasons:

```js
try {
  await Promise.any([task1(), task2()]);
} catch (error) {
  console.log(error instanceof AggregateError); // true
  console.log(error.errors); // array of individual errors
}
```

### Summary

| Method | Settles when | Typical use case |
| --- | --- | --- |
| `Promise.all()` | All fulfill, or first rejection | Need every result; all required |
| `Promise.allSettled()` | All settle (success or failure) | Need every outcome, failures allowed |
| `Promise.race()` | First settle (fulfil or reject) | Timeouts, "fastest wins" |
| `Promise.any()` | First fulfillment, or all reject (`AggregateError`) | Fastest successful result, ignore failures |

---

## 7. Sequential vs Parallel Execution ⭐⭐⭐

This is **very important**.

### Sequential

```js
const user = await getUser();
const orders = await getOrders();
const products = await getProducts();
```

Execution:

```text
getUser
   ↓
getOrders
   ↓
getProducts
```

If each takes 1 second:

```text
≈ 3 seconds
```

---

### Parallel

If operations are independent:

```js
const [user, orders, products] = await Promise.all([
  getUser(),
  getOrders(),
  getProducts(),
]);
```

Execution:

```text
getUser     ───────┐
getOrders   ───────┼──→ result
getProducts ───────┘
```

If each takes 1 second:

```text
≈ 1 second
```

### Important interview rule

Ask:

> **Does operation B depend on the result of operation A?**

If **yes → sequential**

```js
const user = await getUser();
const orders = await getOrders(user.id);
```

If **no → parallel**

```js
const [users, products] = await Promise.all([getUsers(), getProducts()]);
```

---

## 8. Async Error Handling ⭐⭐⭐

### With `async/await`

Use `try/catch`.

```js
async function getData() {
  try {
    const data = await fetchData();
    console.log(data);
  } catch (error) {
    console.error(error);
  }
}
```

### With Promises

```js
fetchData()
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error(error);
  });
```

### Handling `Promise.all()`

```js
try {
  const results = await Promise.all([task1(), task2(), task3()]);
} catch (error) {
  console.error("One task failed:", error);
}
```

### `finally`

Use when something should happen regardless of success/failure:

```js
try {
  await saveData();
} catch (error) {
  console.error(error);
} finally {
  console.log("Request completed");
}
```

---

## 9. Unhandled Rejections and Uncaught Exceptions ⭐⭐⭐

Sometimes an error escapes every `.catch()` and `try/catch` you wrote. Node
gives you process-level hooks as a last line of defense.

### `unhandledRejection`

Fires when a Promise rejects and **no** `.catch()`/`try-catch` handled it:

```js
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection:", reason);
});

Promise.reject(new Error("Something failed"));
```

**Important:** since **Node.js 15**, an unhandled rejection **terminates the
process by default** (it used to just print a warning). Treat
`unhandledRejection` as a place to log and clean up, not as a substitute for
proper error handling.

### `uncaughtException`

Fires when a synchronous error escapes all the way to the top without being
caught:

```js
process.on("uncaughtException", (error) => {
  console.error("Uncaught exception:", error);
  process.exit(1);
});
```

`uncaughtException` leaves the process in a potentially inconsistent state,
so the recommended practice is to log the error and exit rather than trying
to keep running.

### Interview question

**Q: What happens to an unhandled Promise rejection in modern Node.js?**

> By default, since Node.js 15, an unhandled Promise rejection crashes the
> process (similar to an uncaught exception), instead of only logging a
> warning as older versions did.

---

## 🔥 Most Important Interview Concepts

For this chapter, make sure you can explain these **without looking at
notes**:

1. **Callback vs Promise**
2. **Callback hell**
3. **Promise states**
4. **Promise chaining**
5. **Why use `async/await`?**
6. **Does `await` block Node.js?**
7. **`Promise.all()` vs `Promise.allSettled()`**
8. **`Promise.race()` vs `Promise.any()`**
9. **Sequential vs parallel execution**
10. **When should you use `Promise.all()`?**
11. **How do you handle errors with `async/await`?**
12. **What happens when one Promise in `Promise.all()` rejects?**
13. **Why does an `async` function return a Promise?**
14. **What happens to an unhandled Promise rejection?**
15. **`unhandledRejection` vs `uncaughtException`**

### ⭐ One pattern you should remember

```js
async function getDashboard() {
  try {
    const [user, products, notifications] = await Promise.all([
      getUser(),
      getProducts(),
      getNotifications(),
    ]);

    return {
      user,
      products,
      notifications,
    };
  } catch (error) {
    console.error("Failed to load dashboard:", error);
  }
}
```

This single example combines **async/await + parallel execution +
Promise.all + error handling**, so it's excellent interview practice.
