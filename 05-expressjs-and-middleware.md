# Express.js & Middleware ⭐⭐⭐

## Topics Covered

- [1. Express Architecture](#1-express-architecture)
- [2. Routing](#2-routing)
- [3. Request/Response](#3-requestresponse)
- [4. Middleware](#4-middleware)
- [5. next()](#5-next)
- [6. Custom Middleware](#6-custom-middleware)
- [7. Error-Handling Middleware](#7-error-handling-middleware)
- [8. Router](#8-router)

---

This is a **very important Node.js interview chapter**. It covers
interview-focused explanations, examples, and common questions for every
topic in the list.

---

## 1. Express Architecture ⭐⭐⭐

### What is Express.js?

**Express.js** is a lightweight web framework built on top of Node.js that
makes it easier to build:

- REST APIs
- Web servers
- Backend applications
- Middleware pipelines
- Routing systems

Without Express, you can create a server using Node's built-in `http`
module:

```js
const http = require("node:http");

const server = http.createServer((req, res) => {
  res.end("Hello");
});

server.listen(3000);
```

With Express:

```js
const express = require("express");

const app = express();

app.get("/", (req, res) => {
  res.send("Hello");
});

app.listen(3000);
```

Express provides abstractions around things like **routing, middleware,
request handling, and responses**.

### Basic Express architecture

Think of Express like:

```text
Client
   ↓
HTTP Request
   ↓
Express Application
   ↓
Middleware 1
   ↓
Middleware 2
   ↓
Router
   ↓
Controller / Handler
   ↓
Response
   ↓
Client
```

A typical application:

```js
const express = require("express");

const app = express();

app.use(express.json());

app.get("/users", (req, res) => {
  res.json([{ id: 1, name: "John" }]);
});

app.listen(3000);
```

### Interview question

**Q: What is Express.js?**

> Express.js is a minimal and flexible web framework for Node.js that
> provides features such as routing, middleware handling, request/response
> utilities, and API development.

---

## 2. Routing ⭐⭐⭐

Routing determines **which code should execute for a particular HTTP method
and URL**.

```js
app.get("/users", (req, res) => {
  res.send("Get users");
});

app.post("/users", (req, res) => {
  res.send("Create user");
});

app.put("/users/:id", (req, res) => {
  res.send("Update user");
});

app.delete("/users/:id", (req, res) => {
  res.send("Delete user");
});
```

The route is generally:

```text
HTTP Method + URL → Handler
```

For example:

```text
GET /users
```

matches:

```js
app.get("/users", handler);
```

### Route Parameters

You can define dynamic values using `:`.

```js
app.get("/users/:id", (req, res) => {
  console.log(req.params.id);
});
```

Request:

```text
GET /users/123
```

Then:

```js
req.params.id;
```

is:

```text
123
```

### Query Parameters

Request:

```text
GET /users?page=2&limit=10
```

Access them using:

```js
req.query;
```

Example:

```js
app.get("/users", (req, res) => {
  console.log(req.query);
});
```

Result:

```json
{
  "page": "2",
  "limit": "10"
}
```

### Important difference

```text
/users/123
```

→ `req.params`

```text
/users?page=2
```

→ `req.query`

---

## 3. Request / Response ⭐⭐⭐

Express provides `req` and `res` objects.

```js
app.get("/users", (req, res) => {
  // req → incoming request
  // res → outgoing response
});
```

### Important `req` properties

#### `req.params`

Route parameters:

```js
app.get("/users/:id", (req, res) => {
  console.log(req.params.id);
});
```

#### `req.query`

Query parameters:

```js
app.get("/users", (req, res) => {
  console.log(req.query);
});
```

#### `req.body`

Request body:

```js
app.post("/users", (req, res) => {
  console.log(req.body);
});
```

For JSON requests, you normally need:

```js
app.use(express.json());
```

before the route.

#### `req.headers`

Access HTTP headers:

```js
const token = req.headers.authorization;
```

#### `req.method`

```js
console.log(req.method);
```

Example:

```text
GET
```

#### `req.url`

```js
console.log(req.url);
```

### Important `res` methods

#### `res.send()`

```js
res.send("Hello");
```

#### `res.json()`

Used frequently in APIs:

```js
res.json({
  message: "Success",
});
```

#### `res.status()`

Set HTTP status:

```js
res.status(201).json({
  message: "User created",
});
```

You can chain them:

```js
res.status(200).json({
  message: "Success",
});
```

#### `res.sendStatus()`

```js
res.sendStatus(404);
```

This sends the status code and its standard text.

---

## 4. Middleware ⭐⭐⭐⭐⭐

This is probably the **most important Express topic in this chapter**.

### What is middleware?

Middleware is a function that runs **between the incoming request and the
final response**.

Basic structure:

```js
app.use((req, res, next) => {
  console.log("Middleware executed");

  next();
});
```

Think:

```text
Request
   ↓
Middleware
   ↓
Middleware
   ↓
Route Handler
   ↓
Response
```

Middleware can:

- Execute code
- Modify `req`
- Modify `res`
- End the request
- Pass control to the next middleware

### Example

```js
app.use((req, res, next) => {
  console.log("Request received");

  next();
});

app.get("/", (req, res) => {
  res.send("Home");
});
```

Request:

```text
GET /
```

Execution:

```text
Request
  ↓
Middleware
  ↓
next()
  ↓
Route handler
  ↓
Response
```

---

## 5. `next()` ⭐⭐⭐⭐⭐

`next()` passes control to the **next middleware or handler** in the
Express middleware chain.

```js
app.use((req, res, next) => {
  console.log("Middleware 1");

  next();
});

app.use((req, res, next) => {
  console.log("Middleware 2");

  next();
});

app.get("/", (req, res) => {
  console.log("Route");
  res.send("Hello");
});
```

Execution:

```text
Middleware 1
      ↓
    next()
      ↓
Middleware 2
      ↓
    next()
      ↓
Route
      ↓
Response
```

### What happens if you don't call `next()`?

Example:

```js
app.use((req, res, next) => {
  console.log("Middleware");

  // next() missing
});
```

The request will generally remain hanging because neither the next handler
nor a response was reached.

You can either:

```js
next();
```

or terminate the request:

```js
res.send("Done");
```

### `next(error)`

This is important for error handling.

```js
app.use((req, res, next) => {
  const error = new Error("Something went wrong");

  next(error);
});
```

Express will move into error-handling middleware.

---

## 6. Custom Middleware ⭐⭐⭐

You can create your own reusable middleware.

Example:

```js
function logger(req, res, next) {
  console.log(`${req.method} ${req.url}`);

  next();
}

app.use(logger);
```

Now every request goes through `logger`.

### Route-specific middleware

Middleware doesn't have to apply to every route.

```js
function auth(req, res, next) {
  const token = req.headers.authorization;

  if (!token) {
    return res.status(401).json({
      message: "Unauthorized",
    });
  }

  next();
}
```

Use it only for a particular route:

```js
app.get("/profile", auth, (req, res) => {
  res.json({
    message: "Profile",
  });
});
```

Flow:

```text
GET /profile
      ↓
auth middleware
      ↓
next()
      ↓
/profile handler
```

But:

```text
GET /public
```

doesn't go through `auth`.

---

## 7. Error-Handling Middleware ⭐⭐⭐⭐⭐

Express has a special middleware format for errors:

```js
(err, req, res, next);
```

Notice it has **4 parameters**.

```js
app.use((err, req, res, next) => {
  console.error(err);

  res.status(500).json({
    message: "Internal Server Error",
  });
});
```

### Very important interview point

Normal middleware:

```js
(req, res, next);
```

Error middleware:

```js
(err, req, res, next);
```

The **four arguments identify it as error-handling middleware**.

### How errors reach it

```js
app.get("/users", (req, res, next) => {
  try {
    throw new Error("Database failed");
  } catch (error) {
    next(error);
  }
});
```

Then:

```js
app.use((err, req, res, next) => {
  res.status(500).json({
    message: err.message,
  });
});
```

Flow:

```text
Request
   ↓
Route
   ↓
Error occurs
   ↓
next(error)
   ↓
Error Middleware
   ↓
Response
```

### Important

Error middleware should generally be registered **after your routes and
other middleware**.

```js
app.use(express.json());

app.get("/users", handler);

app.use(errorHandler);
```

---

## 8. Router ⭐⭐⭐⭐⭐

As your application grows, putting all routes inside `app.js` becomes
difficult.

Instead, use `express.Router()`.

### Without Router

```js
app.get("/users", getUsers);
app.post("/users", createUser);

app.get("/products", getProducts);
app.post("/products", createProduct);
```

Everything is inside the main application.

### With Router

`user.routes.js`

```js
const express = require("express");

const router = express.Router();

router.get("/", (req, res) => {
  res.json({ message: "Get users" });
});

router.post("/", (req, res) => {
  res.json({ message: "Create user" });
});

module.exports = router;
```

Then in `app.js`:

```js
const express = require("express");
const userRoutes = require("./user.routes");

const app = express();

app.use("/users", userRoutes);

app.listen(3000);
```

Now:

```text
GET /users
```

matches:

```js
router.get("/");
```

and:

```text
POST /users
```

matches:

```js
router.post("/");
```

### Router-Level Middleware

You can also attach middleware to a router.

```js
const router = express.Router();

router.use(auth);

router.get("/profile", (req, res) => {
  res.send("Profile");
});
```

Every route in that router will pass through `auth`.

---

## Complete Example

Here's how these concepts work together:

```js
const express = require("express");

const app = express();

app.use(express.json());

// Logger middleware
app.use((req, res, next) => {
  console.log(req.method, req.url);
  next();
});

// Authentication middleware
function auth(req, res, next) {
  const token = req.headers.authorization;

  if (!token) {
    return res.status(401).json({
      message: "Unauthorized",
    });
  }

  next();
}

// Public route
app.get("/", (req, res) => {
  res.json({
    message: "Home",
  });
});

// Protected route
app.get("/profile", auth, (req, res) => {
  res.json({
    message: "Profile",
  });
});

// Error middleware
app.use((err, req, res, next) => {
  console.error(err);

  res.status(500).json({
    message: "Internal Server Error",
  });
});

app.listen(3000);
```

The architecture is:

```text
                  Request
                     │
                     ▼
             express.json()
                     │
                     ▼
              Logger Middleware
                     │
                     ▼
                 Routing
                /       \
               /         \
          Public        Protected
            │              │
            │           auth()
            │              │
            ▼              ▼
        Handler          Handler
               \          /
                \        /
                 Response

              Errors
                 │
                 ▼
          Error Middleware
```

---

## 🔥 Interview Questions You Must Know

For this chapter, make sure you can answer these without looking at notes:

### Express

1. What is Express.js?
2. Why do we use Express instead of the Node `http` module?
3. Explain Express architecture.
4. How does an Express request flow through the application?

### Routing

1. What is routing?
2. Difference between `req.params` and `req.query`.
3. How do you access request body?
4. How do you define dynamic routes?
5. Difference between `app.get()` and `app.use()`.

### Middleware

1. What is middleware?
2. Why is middleware used?
3. What does `next()` do?
4. What happens if `next()` isn't called?
5. Can middleware modify `req` and `res`?
6. How do you create custom middleware?
7. What is application-level vs route-level middleware?

### Error Handling

1. What is error-handling middleware?
2. Why does error middleware have 4 parameters?
3. How do you pass an error to error middleware?
4. Where should error middleware be registered?
5. What is the difference between `next()` and `next(error)`?

### Router

1. Why do we use `express.Router()`?
2. Difference between `app` and `router`.
3. How do you mount a router?
4. What is router-level middleware?

---

## 🎯 What you should be able to explain in an interview

The most important thing is not memorizing definitions. You should be able
to explain this flow:

> **Request → Middleware → `next()` → Router → Controller/Handler →
> Response**

And for failures:

> **Request → Middleware/Route → `next(error)` → Error-handling middleware
> → Error Response**

If you understand those two flows properly, you've covered the **core
Express.js & Middleware interview concepts** in your list.
