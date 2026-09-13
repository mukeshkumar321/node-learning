# Error Handling ⭐⭐⭐

## Topics Covered

- [1. try/catch](#1-trycatch)
- [2. Promise Errors](#2-promise-errors)
- [3. Async Errors](#3-async-errors)
- [4. Custom Errors](#4-custom-errors)
- [5. Global Error Handling](#5-global-error-handling)
- [6. Express Error Middleware](#6-express-error-middleware)
- [7. uncaughtException](#7-uncaughtexception)
- [8. unhandledRejection](#8-unhandledrejection)
- [9. Graceful Shutdown](#9-graceful-shutdown)

---

Error handling is important because interviewers often ask **"What happens
when an error occurs in async code?"**, **"What is the difference between
`uncaughtException` and `unhandledRejection`?"**, and **"How do you handle
errors in Express?"**

---

## 1. `try/catch`

`try/catch` is used to catch **synchronous errors** and errors thrown inside
an `async` function when `await` is used.

### Basic example

```js
try {
  const result = JSON.parse("invalid json");
} catch (error) {
  console.log("Something went wrong:", error.message);
}
```

If `JSON.parse()` throws an error, execution moves to `catch`.

### With async/await

```js
async function getUser() {
  try {
    const user = await fetchUser();
    return user;
  } catch (error) {
    console.log("Failed to fetch user:", error.message);
  }
}
```

### Interview point ⭐

`try/catch` **does not automatically catch errors from unrelated
asynchronous callbacks**.

For example:

```js
try {
  setTimeout(() => {
    throw new Error("Something went wrong");
  }, 1000);
} catch (error) {
  console.log(error);
}
```

The `catch` won't catch that error because the callback executes later,
outside the original `try/catch` execution.

---

## 2. Promise Errors

Promises can fail through:

- `throw`
- `reject()`

### Using `.catch()`

```js
fetchUser()
  .then((user) => {
    console.log(user);
  })
  .catch((error) => {
    console.log("Error:", error.message);
  });
```

### Promise rejection

```js
function getUser() {
  return Promise.reject(new Error("User not found"));
}

getUser().catch((error) => {
  console.log(error.message);
});
```

### `throw` inside `.then()`

```js
Promise.resolve()
  .then(() => {
    throw new Error("Something failed");
  })
  .catch((error) => {
    console.log(error.message);
  });
```

The thrown error becomes a **rejected Promise**.

### Interview question

**Q: How do you handle Promise errors?**

Answer:

> We handle Promise errors using `.catch()` or `try/catch` when using
> `async/await`. We should make sure rejected Promises are always handled to
> avoid `unhandledRejection`.

---

## 3. Async Errors

With `async/await`, use `try/catch`.

```js
async function getData() {
  try {
    const response = await fetchData();
    return response;
  } catch (error) {
    console.error(error);
  }
}
```

Without handling:

```js
async function getData() {
  const response = await fetchData();
}
```

If `fetchData()` rejects, `getData()` itself returns a rejected Promise.

So the caller must handle it:

```js
getData().catch((error) => {
  console.log(error);
});
```

### Important interview distinction

```js
try {
  await something();
} catch (error) {
  // handled
}
```

versus

```js
something().catch((error) => {
  // handled
});
```

Both can handle Promise errors.

---

## 4. Custom Errors

Instead of throwing generic `Error`, we can create custom error classes.

```js
class UserNotFoundError extends Error {
  constructor(message) {
    super(message);
    this.name = "UserNotFoundError";
  }
}
```

Then:

```js
throw new UserNotFoundError("User does not exist");
```

Handling:

```js
try {
  throw new UserNotFoundError("User does not exist");
} catch (error) {
  console.log(error.name);
  console.log(error.message);
}
```

### Practical API example

```js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
  }
}
```

Usage:

```js
throw new AppError("User not found", 404);
```

Then your global/Express error handler can use:

```js
error.statusCode;
```

to determine the HTTP response.

### Interview point ⭐

Custom errors are useful because they allow us to attach additional
information such as:

- HTTP status code
- error code
- error type
- metadata

### `error.cause` (ES2022)

`Error` supports an optional `cause` option for chaining/wrapping errors
while keeping the original error for debugging:

```js
try {
  await db.connect();
} catch (originalErr) {
  throw new Error("Failed to start application", { cause: originalErr });
}
```

`error.cause` gives you the original low-level error without losing the
higher-level context of where it was re-thrown.

### `Error.captureStackTrace`

When extending `Error` with custom classes, `Error.captureStackTrace` keeps
the stack trace clean by excluding the constructor call itself:

```js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;

    Error.captureStackTrace(this, this.constructor);
  }
}
```

This is a V8-specific API and is commonly used in custom error base classes.

---

## 5. Global Error Handling

A real application shouldn't have random error handling everywhere.

A common architecture is:

```text
Route
  ↓
Controller
  ↓
Service
  ↓
Error occurs
  ↓
Global Error Handler
  ↓
HTTP Response
```

For example:

```js
app.use((error, req, res, next) => {
  console.error(error);

  res.status(error.statusCode || 500).json({
    success: false,
    message: error.message || "Internal Server Error",
  });
});
```

This provides a centralized place to handle application errors.

### Important

Don't expose sensitive internal errors to clients.

Bad:

```json
{
  "error": "MongoDB connection failed at /internal/server/path..."
}
```

Better:

```json
{
  "success": false,
  "message": "Internal Server Error"
}
```

Log the detailed error internally.

---

## 6. Express Error Middleware

Express has special middleware for errors.

The signature is:

```js
(error, req, res, next);
```

Example:

```js
app.use((error, req, res, next) => {
  res.status(500).json({
    message: error.message,
  });
});
```

### Why four parameters?

Express recognizes middleware as an error handler when it has:

```js
(err, req, res, next);
```

### Route

```js
app.get("/users/:id", async (req, res, next) => {
  try {
    const user = await getUser(req.params.id);

    if (!user) {
      throw new AppError("User not found", 404);
    }

    res.json(user);
  } catch (error) {
    next(error);
  }
});
```

Then:

```js
app.use((error, req, res, next) => {
  res.status(error.statusCode || 500).json({
    message: error.message,
  });
});
```

### Interview question ⭐

**Q: What is the difference between normal middleware and error
middleware?**

Normal:

```js
(req, res, next);
```

Error:

```js
(err, req, res, next);
```

---

## 7. `uncaughtException`

This occurs when an exception is thrown synchronously and **nothing catches
it**.

Example:

```js
throw new Error("Unexpected error");
```

You can listen for it:

```js
process.on("uncaughtException", (error) => {
  console.error("Uncaught Exception:", error);
});
```

### Very important ⭐⭐⭐

You should **not treat `uncaughtException` as normal error handling**.

If an exception reaches this point, the application may be in an unreliable
state.

A common production strategy is:

```text
uncaughtException
       ↓
Log error
       ↓
Stop accepting new work
       ↓
Graceful shutdown
       ↓
Process exits
       ↓
Process manager restarts it
```

For example:

```js
process.on("uncaughtException", (error) => {
  console.error(error);

  server.close(() => {
    process.exit(1);
  });
});
```

### Interview answer

> `uncaughtException` occurs when a synchronous exception reaches the
> process level without being caught. It should generally be treated as a
> fatal condition; after logging and cleanup, the process should be
> restarted rather than continuing in an unknown state.

---

## 8. `unhandledRejection`

This happens when a Promise is rejected and there is no rejection handler
attached appropriately.

Example:

```js
Promise.reject(new Error("Database failed"));
```

You can listen for it:

```js
process.on("unhandledRejection", (error) => {
  console.error("Unhandled rejection:", error);
});
```

### Example

```js
async function test() {
  throw new Error("Something failed");
}

test();
```

`test()` returns a rejected Promise, but nobody handles it.

That's an **unhandled rejection**.

Correct:

```js
test().catch((error) => {
  console.error(error);
});
```

or:

```js
async function main() {
  try {
    await test();
  } catch (error) {
    console.error(error);
  }
}
```

### Interview difference ⭐⭐⭐

| `uncaughtException` | `unhandledRejection` |
| --- | --- |
| Synchronous exception | Promise rejection |
| No `try/catch` | No rejection handler |
| Process-level problem | Promise-level problem |
| Usually treated as fatal | Should be handled/reviewed; production strategy may terminate |

### Important behavior change (Node.js 15+) ⭐⭐⭐

Since **Node.js 15**, unhandled promise rejections **terminate the process by
default**. This is the `--unhandled-rejections=strict` behavior, which became
the default instead of just printing a warning.

```text
Node.js < 15  → unhandled rejection logs a warning, process keeps running
Node.js >= 15 → unhandled rejection throws and crashes the process (default)
```

This is a common interview trap: older answers that say "an unhandled
rejection just logs a warning" are **outdated** for current Node.js versions.
Always attach a `.catch()` or wrap `await` in `try/catch`.

### A note on domains (legacy) ⚠️

The old `domain` module was Node's original attempt at grouping async
operations for error handling. It is **deprecated** and should **not** be
used in new code — `async/await` with `try/catch`, plus
`uncaughtException`/`unhandledRejection` handlers, fully replace its use
cases.

---

## 9. Graceful Shutdown

Graceful shutdown means **stopping the application safely instead of
killing it immediately**.

For example, when the server receives:

```text
SIGTERM
```

we can:

1. Stop accepting new requests
2. Allow existing requests to finish
3. Close database connections
4. Close other resources
5. Exit the process

Example:

```js
const server = app.listen(3000);

process.on("SIGTERM", () => {
  console.log("SIGTERM received");

  server.close(() => {
    console.log("HTTP server closed");

    process.exit(0);
  });
});
```

For database connections:

```js
process.on("SIGTERM", async () => {
  console.log("Shutting down...");

  server.close(async () => {
    await database.disconnect();

    process.exit(0);
  });
});
```

### `process.exitCode` vs `process.exit()`

- `process.exitCode = 1` sets the exit code but lets the event loop finish
  naturally — pending I/O (like an in-flight `console.log` write or a
  response being flushed) completes before the process exits.
- `process.exit()` exits **immediately**, which can truncate pending
  writes/I/O and skip cleanup that hasn't finished yet.

For graceful shutdown, prefer setting `process.exitCode` and letting the
process exit naturally once resources are closed, only calling
`process.exit()` directly when you need to force termination (e.g. after a
timeout waiting for connections to close).

### Why is graceful shutdown important?

Without it, you could have:

- requests terminated halfway
- database operations interrupted
- connections left open
- incomplete file operations
- inconsistent application state

---

## 🔥 Most Important Interview Questions

Make sure you can answer these without looking at notes:

### 1. How does `try/catch` work with async/await?

Use:

```js
try {
  await operation();
} catch (error) {
  // handle error
}
```

---

### 2. How do you handle Promise errors?

```js
promise.catch((error) => {});
```

or:

```js
try {
  await promise;
} catch (error) {}
```

---

### 3. What happens when an async function throws an error?

It returns a **rejected Promise**.

```js
async function test() {
  throw new Error("Failed");
}
```

Conceptually:

```js
test(); // rejected Promise
```

---

### 4. What is custom error handling?

Creating application-specific errors:

```js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
  }
}
```

---

### 5. How does Express handle errors?

Pass the error to:

```js
next(error);
```

and handle it with:

```js
app.use((err, req, res, next) => {});
```

---

### 6. `uncaughtException` vs `unhandledRejection`?

Remember:

```text
uncaughtException  → uncaught synchronous exception
unhandledRejection → unhandled Promise rejection
```

---

### 7. What is graceful shutdown?

> Safely shutting down the application by stopping new work, finishing
> existing work, closing resources, and then exiting.

---

## 🧠 Final Mental Model

Remember Node.js error handling like this:

```text
                ERROR
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   Synchronous           Promise
        │                   │
    try/catch          catch / await
        │                   │
        └─────────┬─────────┘
                  ↓
          Application Error
                  │
                  ↓
        Express Error Handler
                  │
                  ↓
             HTTP Response

If NOT handled
       │
       ├── uncaughtException
       │
       └── unhandledRejection
                  │
                  ↓
          Log + Cleanup
                  ↓
          Graceful Shutdown
                  ↓
             Process Exit
```

### ⭐ Interview priority

**Must know extremely well:**

1. `try/catch`
2. Promise error handling
3. Async error handling
4. Express error middleware
5. Custom errors
6. `uncaughtException` vs `unhandledRejection`
7. Graceful shutdown

The **most important interview concept** here is understanding the
difference between **an error you can recover from inside application
flow** and a **process-level failure that should lead to controlled
shutdown/restart**.
