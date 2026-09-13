# REST API & HTTP ⭐⭐⭐

## Topics Covered

- [1. HTTP Methods](#1-http-methods)
- [2. Status Codes](#2-status-codes)
- [3. Headers](#3-headers)
- [4. Request/Response](#4-requestresponse)
- [5. Query Params vs Path Params](#5-query-params-vs-path-params)
- [6. JSON](#6-json)
- [7. REST Principles](#7-rest-principles)
- [8. Pagination/Filtering](#8-paginationfiltering)
- [9. API Versioning](#9-api-versioning)

---

This is an important Node.js interview chapter. It covers every point in
the list, with interview-focused explanations and examples.

---

## 1. HTTP Methods ⭐⭐⭐

HTTP methods tell the server what operation the client wants to perform.

| Method | Purpose | Example |
| --- | --- | --- |
| GET | Retrieve data | Get all users |
| POST | Create data | Create a user |
| PUT | Replace/update entire resource | Replace user |
| PATCH | Partially update resource | Update user's email |
| DELETE | Delete data | Delete a user |
| HEAD | Same as GET but without response body | Check resource |
| OPTIONS | Get supported operations | CORS/preflight |

### GET

```http
GET /users
```

Used to retrieve users.

Important: GET should generally be safe and idempotent.

### POST

```http
POST /users
Content-Type: application/json

{
  "name": "Rahul",
  "email": "rahul@example.com"
}
```

Creates a new resource.

POST is generally not idempotent.

Calling it twice can create two users.

### Idempotency ⭐⭐⭐

A very common interview question: **which HTTP methods are idempotent?**

**Idempotent** means: making the same request one time or many times
produces the same result on the server — the state doesn't change further
after the first successful call.

| Method | Idempotent? |
| --- | --- |
| GET | Yes |
| HEAD | Yes |
| OPTIONS | Yes |
| PUT | Yes |
| DELETE | Yes |
| POST | No (generally) |
| PATCH | No (generally) |

Notes:

- `GET`, `HEAD`, `OPTIONS` — read-only, so repeating them changes nothing.
- `PUT` — replaces a resource with the same payload; doing it once or ten
  times leaves the resource in the same final state.
- `DELETE` — deleting an already-deleted resource still leaves it deleted
  (even if the server returns `404` on the second call, the resource state
  is unchanged).
- `POST` — typically creates a new resource each time, so calling it twice
  usually creates two resources. Not idempotent.
- `PATCH` — depends on the semantics of the patch, but a partial update
  (e.g., "increment counter by 1") produces a different result each time
  it's called, so it's generally treated as **not** idempotent.

### PUT vs PATCH ⭐⭐⭐

This is a common interview question.

PUT → replace the complete resource.

```http
PUT /users/10
Content-Type: application/json

{
  "name": "Rahul",
  "email": "new@example.com",
  "age": 25
}
```

PATCH → update only specific fields.

```http
PATCH /users/10
Content-Type: application/json

{
  "email": "new@example.com"
}
```

**Interview answer**

> PUT is generally used for complete replacement of a resource, while PATCH
> is used for partial updates.

---

## 2. Status Codes ⭐⭐⭐

Status codes tell the client what happened with the request.

### 1xx — Informational

Not commonly used in normal REST API development.

### 2xx — Success

| Code | Meaning |
| --- | --- |
| 200 | OK |
| 201 | Created |
| 202 | Accepted |
| 204 | No Content |

Examples:

```text
GET /users       → 200 OK
POST /users      → 201 Created
DELETE /users/10 → 204 No Content
```

### 3xx — Redirection

| Code | Meaning |
| --- | --- |
| 301 | Permanently moved |
| 302 | Found/temporary redirect |
| 304 | Not Modified |

#### ETags & conditional requests

`304 Not Modified` usually works together with **ETags** — a version
identifier the server attaches to a response so the client can ask "has
this changed since I last fetched it?" instead of re-downloading it.

```text
Response:
ETag: "33a64df551"

Next request:
If-None-Match: "33a64df551"
```

If the resource hasn't changed, the server replies `304 Not Modified` with
no body, saving bandwidth. A similar mechanism exists based on timestamps:

```text
Response:
Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT

Next request:
If-Modified-Since: Wed, 21 Oct 2025 07:28:00 GMT
```

`If-None-Match` (ETag-based) is generally preferred since it's exact,
while `If-Modified-Since` only has second-level precision.

### 4xx — Client Errors ⭐⭐⭐

| Code | Meaning |
| --- | --- |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 405 | Method Not Allowed |
| 409 | Conflict |
| 422 | Unprocessable Entity (also seen as "Unprocessable Content") |
| 429 | Too Many Requests |

#### 400 vs 422 ⭐⭐⭐

Another common interview question.

**400 Bad Request** — the request itself is malformed. The server can't
even parse it.

```text
Invalid JSON syntax
Missing required Content-Type
Malformed request body
```

**422 Unprocessable Entity** — the request is well-formed (valid JSON,
correct structure) but semantically invalid — it fails validation rules.

```json
{
  "email": "not-an-email",
  "age": -5
}
```

This is syntactically valid JSON, so it's not a `400`. But it fails
business/validation rules, so the server responds with `422`.

**Interview answer**

> 400 means the request couldn't be understood (malformed syntax), while
> 422 means the request was understood but the data failed validation.

#### 401 vs 403 ⭐⭐⭐

**401** — The client has not provided valid authentication credentials.

```http
GET /profile
Authorization: Bearer invalid-token
```

**403** — The client is authenticated but does not have permission.

```text
Admin endpoint
    ↓
Normal authenticated user
    ↓
403 Forbidden
```

### 5xx — Server Errors

| Code | Meaning |
| --- | --- |
| 500 | Internal Server Error |
| 502 | Bad Gateway |
| 503 | Service Unavailable |
| 504 | Gateway Timeout |

---

## 3. Headers ⭐⭐⭐

Headers contain additional metadata about the request or response.

Example:

```text
Content-Type: application/json
Authorization: Bearer token123
Accept: application/json
```

### Important request headers

**`Content-Type`** — Tells the server what format the request body uses.

```text
Content-Type: application/json
```

**`Accept`** — Tells the server what response format the client expects.

```text
Accept: application/json
```

**`Authorization`** — Used for authentication.

```text
Authorization: Bearer eyJhbGci...
```

**`User-Agent`** — Identifies the client.

### Important response headers

```text
Content-Type: application/json
Cache-Control: no-cache
Location: /users/101
```

### Interview question

**Q: `Content-Type` vs `Accept`?**

> `Content-Type` describes the format of the data being sent, while
> `Accept` specifies the format the client wants in the response.

---

## 4. Request/Response ⭐⭐⭐

Every HTTP interaction consists of a request and a response.

### Request

```text
Client
   ↓
HTTP Request
   ↓
Server
```

A request can contain:

- Method
- URL
- Headers
- Query parameters
- Path parameters
- Body

Example:

```http
POST /users?source=mobile
Authorization: Bearer token
Content-Type: application/json

{
  "name": "Rahul"
}
```

### Response

```text
Server
   ↓
HTTP Response
   ↓
Client
```

A response contains:

- Status code
- Headers
- Body

Example:

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": 101,
  "name": "Rahul"
}
```

---

## 5. Query Params vs Path Params ⭐⭐⭐

Very common interview question.

### Path Parameters

Used to identify a specific resource.

```http
GET /users/101
```

Here, `101` is a path parameter.

Express:

```js
app.get("/users/:id", (req, res) => {
  console.log(req.params.id);
});
```

### Query Parameters

Usually used for filtering, searching, sorting, pagination, etc.

```http
GET /users?page=2&limit=10
```

Express:

```js
app.get("/users", (req, res) => {
  console.log(req.query.page);
  console.log(req.query.limit);
});
```

### Easy way to remember

```text
/users/101
        ↑
    Path parameter

/users?page=2
       ↑
   Query parameter
```

### Interview answer

> Path parameters identify a specific resource, while query parameters
> modify or filter the request.

---

## 6. JSON ⭐⭐

JSON stands for JavaScript Object Notation.

It is commonly used to exchange data between frontend and backend.

Example:

```json
{
  "id": 101,
  "name": "Rahul",
  "skills": ["Node.js", "React"],
  "active": true
}
```

JSON supports:

```text
String
Number
Boolean
Array
Object
null
```

JSON does not support:

```text
undefined
function
Date object
Map
Set
```

### Express

To parse JSON request bodies:

```js
app.use(express.json());
```

Then:

```js
app.post("/users", (req, res) => {
  console.log(req.body);
});
```

---

## 7. REST Principles ⭐⭐⭐

REST = Representational State Transfer

REST is an architectural style for designing network APIs.

### Important principles

#### 1. Client-Server

Frontend and backend are separated.

```text
Client → API → Server
```

#### 2. Stateless ⭐⭐⭐

Each request should contain the information required to process it.

The server should not depend on previous requests to understand the
current request.

For example:

```http
GET /profile
Authorization: Bearer token
```

The server can authenticate the request using the token.

#### 3. Resource-based URLs

Prefer:

```text
GET  /users
GET  /users/10
POST /users
```

Instead of:

```text
GET  /getUsers
POST /createUser
```

The URL represents the resource, while HTTP methods represent the
operation.

#### 4. Uniform Interface

APIs should follow consistent conventions.

For example:

```text
GET    /users
GET    /users/10
POST   /users
PATCH  /users/10
DELETE /users/10
```

#### 5. Cacheable

Responses should indicate whether they can be cached.

Example:

```text
Cache-Control: max-age=3600
```

#### 6. Layered System

The client doesn't necessarily know whether it is communicating directly
with the application server.

There may be:

```text
Client
  ↓
Load Balancer
  ↓
API Gateway
  ↓
Application
  ↓
Database
```

#### 7. HATEOAS

**HATEOAS** = Hypermedia As The Engine Of Application State.

This is one of the constraints from Roy Fielding's original REST
dissertation, and it's a common follow-up when an interviewer asks "is your
API truly RESTful?"

The idea: a response shouldn't just return data — it should also include
**links** describing what the client can do next, so the client doesn't
need to hard-code URLs elsewhere in the application.

Example response with hypermedia links:

```json
{
  "id": 101,
  "name": "Rahul",
  "status": "pending",
  "links": [
    { "rel": "self", "href": "/users/101", "method": "GET" },
    { "rel": "cancel", "href": "/users/101/cancel", "method": "POST" },
    { "rel": "orders", "href": "/users/101/orders", "method": "GET" }
  ]
}
```

In practice, most "REST" APIs (including most Node.js APIs you'll build in
interviews and on the job) skip HATEOAS entirely and are really just
JSON-over-HTTP APIs following REST-ish conventions, not fully RESTful in
the strict Fielding sense. It's fine to say so — just be able to explain
what the constraint means and that it's commonly omitted.

---

## 8. Pagination/Filtering ⭐⭐⭐

Very important when APIs return large datasets.

Instead of:

```http
GET /users
```

returning 1 million users, use pagination.

### Page-based pagination

```http
GET /users?page=2&limit=20
```

Meaning:

```text
page = 2
limit = 20
```

Response:

```json
{
  "data": [],
  "page": 2,
  "limit": 20,
  "total": 1000
}
```

### Filtering

```http
GET /users?role=admin
```

Multiple filters:

```http
GET /users?role=admin&status=active
```

### Searching

```http
GET /users?search=rahul
```

### Sorting

```http
GET /users?sort=name&order=asc
```

### Pagination + filtering + sorting

A realistic API might look like:

```http
GET /users?page=2&limit=20&role=admin&sort=name&order=asc
```

You should be comfortable explaining how each parameter works in an
interview.

---

## 9. API Versioning ⭐⭐⭐

API versioning allows you to change an API without immediately breaking
existing clients.

### URL versioning

Most common and easy to understand:

```http
GET /api/v1/users
```

Later:

```http
GET /api/v2/users
```

### Header versioning

Version can be specified through headers.

```text
Accept: application/vnd.myapi.v2+json
```

### Query parameter versioning

```http
GET /users?version=2
```

### Which one should you use?

For most Node.js projects, URL versioning is straightforward:

```text
/api/v1/users
/api/v2/users
```

The key interview point is:

> API versioning allows us to introduce breaking changes while keeping
> older clients working.

---

# 🔥 Most Important Interview Questions

Make sure you can answer these without looking at notes:

1. What is REST?
2. What is the difference between PUT and PATCH?
3. What is the difference between GET and POST?
4. What does idempotent mean?
5. Which HTTP methods are idempotent?
6. Difference between 401 and 403?
7. Difference between 400 and 422?
8. What does 201 mean?
9. What does 204 mean?
10. What are HTTP headers?
11. `Content-Type` vs `Accept`?
12. What is the `Authorization` header?
13. Path params vs query params?
14. How do you implement pagination in an API?
15. How do filtering and sorting work?
16. What does stateless mean in REST?
17. Why should REST URLs represent resources rather than actions?
18. What is API versioning?
19. How would you version a Node.js API?
20. Design REST APIs for CRUD operations on users.

## 🎯 Interview-ready CRUD example

You should be able to immediately explain:

```text
GET     /api/v1/users
GET     /api/v1/users/:id
POST    /api/v1/users
PUT     /api/v1/users/:id
PATCH   /api/v1/users/:id
DELETE  /api/v1/users/:id
```

And know what should go in:

- Path params
- Query params
- Headers
- Request body
- Response body
- Status codes
