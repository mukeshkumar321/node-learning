# Authentication & Security ⭐⭐⭐

## Topics Covered

- [1. Authentication vs Authorization](#1-authentication-vs-authorization)
- [2. JWT](#2-jwt)
- [3. Access and Refresh Tokens](#3-access-and-refresh-tokens)
- [4. Sessions vs JWT](#4-sessions-vs-jwt)
- [5. Password Hashing](#5-password-hashing)
- [6. CORS](#6-cors)
- [7. CSRF](#7-csrf)
- [8. XSS](#8-xss)
- [9. Rate Limiting](#9-rate-limiting)
- [10. Input Validation](#10-input-validation)
- [11. HTTPS & Security Headers](#11-https--security-headers)

---

This is an **important Node.js interview chapter**. It covers every point
in the list, with practical Node.js/Express examples and the interview
questions you should be ready for.

---

## 1. Authentication vs Authorization

These are commonly confused.

### Authentication — "Who are you?"

Authentication verifies the identity of a user.

Example:

```text
User → email + password → Server
                         ↓
                    Verify credentials
                         ↓
                    User authenticated
```

Example:

```http
POST /login

{
  "email": "john@example.com",
  "password": "secret123"
}
```

If credentials are correct, the server authenticates the user.

### Authorization — "What are you allowed to do?"

Authorization determines what an authenticated user can access.

Example:

```text
Admin → Can delete users
User  → Cannot delete users
```

Example middleware:

```js
function requireAdmin(req, res, next) {
  if (req.user.role !== "admin") {
    return res.status(403).json({
      message: "Access denied",
    });
  }

  next();
}
```

### Interview question

**Q: What's the difference between authentication and authorization?**

> Authentication verifies **who the user is**, while authorization
> determines **what that authenticated user is allowed to access or
> perform**.

---

## 2. JWT

JWT = **JSON Web Token**.

It is commonly used for stateless authentication.

A JWT looks roughly like:

```text
xxxxx.yyyyy.zzzzz
```

It has three parts:

```text
Header.Payload.Signature
```

### Header

Contains information about the token.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload

Contains claims.

```json
{
  "userId": 123,
  "role": "admin"
}
```

### Signature

Used to verify that the token hasn't been modified.

Conceptually:

```text
signature =
HMAC(
  base64(header) + "." + base64(payload),
  secret
)
```

### Important

JWT payload is **encoded, not encrypted**.

Therefore:

- ❌ Don't put passwords in JWT.
- ❌ Don't put sensitive information in the payload.

### Algorithm confusion attacks

A well-known JWT vulnerability class. Two things to guard against:

- **`alg: none`** — the JWT spec technically allows an "unsecured" token
  with no signature. Never accept `alg: none` when verifying a token; a
  poorly configured verifier might treat it as valid.
- **RS256/HS256 confusion** — if your server expects tokens signed with
  RS256 (asymmetric: private key signs, public key verifies), an attacker
  who knows your public key can craft a token signed with **HS256** using
  the public key as the HMAC secret. If the server's verification code
  just uses whatever `alg` the token header claims, it may incorrectly
  verify the attacker's forged HS256 token as valid using that public key
  as the shared secret.

**Fix:** always explicitly specify the allowed algorithm(s) when verifying,
instead of trusting the token's own header:

```js
jwt.verify(token, publicKey, { algorithms: ["RS256"] });
```

This way, a token forged with a different algorithm is rejected outright.

### JWT storage: localStorage vs httpOnly cookies

Where you store the JWT on the client matters:

- **`localStorage`** — accessible to any JavaScript running on the page,
  so it's vulnerable to theft via **XSS** (see the [XSS](#8-xss) section).
  If an attacker injects a script, they can simply read the token.
- **`httpOnly` cookies** — not accessible to JavaScript at all, so XSS
  can't directly read the token. But because the browser attaches cookies
  automatically, you now need **CSRF protection** for state-changing
  requests (see the [CSRF](#7-csrf) section).

The tradeoff, in short: `localStorage` trades CSRF risk for XSS risk;
`httpOnly` cookies trade XSS-based token theft for the need to defend
against CSRF. Neither is a silver bullet — you still need to defend
against XSS and CSRF regardless of where the token lives.

### JWT Authentication Flow

```text
             Login
               ↓
       Email + Password
               ↓
          Server verifies
               ↓
          Create JWT
               ↓
          Send JWT
               ↓
       Client stores token
               ↓
      Request with JWT
               ↓
       Server verifies JWT
               ↓
          Allow request
```

Example:

```js
const token = jwt.sign(
  {
    userId: user.id,
    role: user.role,
  },
  process.env.JWT_SECRET,
  {
    expiresIn: "15m",
  },
);
```

Middleware:

```js
function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;

  const token = authHeader?.split(" ")[1];

  if (!token) {
    return res.status(401).json({
      message: "Authentication required",
    });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    req.user = decoded;

    next();
  } catch {
    return res.status(401).json({
      message: "Invalid or expired token",
    });
  }
}
```

---

## 3. Access & Refresh Tokens

A common production authentication architecture uses **two tokens**.

### Access Token

Short-lived.

Example:

```text
15 minutes
```

Used for accessing APIs.

```text
Authorization: Bearer <access_token>
```

### Refresh Token

Longer-lived.

Example:

```text
7 days
30 days
```

Used to obtain a new access token.

### Why two tokens?

Suppose the access token lasts 30 days.

If someone steals it, they potentially have access for 30 days.

Instead:

```text
Access Token
    ↓
15 minutes
```

If stolen, the attack window is much smaller.

When it expires:

```text
Refresh Token
      ↓
Generate new Access Token
```

### Flow

```text
Login
  ↓
Access Token + Refresh Token
  ↓
Access Token used for API requests
  ↓
Access Token expires
  ↓
Refresh Token sent to refresh endpoint
  ↓
New Access Token
```

### Revoking & rotating refresh tokens

A common follow-up: **how would you revoke/rotate refresh tokens?**

Since refresh tokens are long-lived, they need their own protections:

- **Store a hash, not the raw token.** Persist a hash (e.g., SHA-256) of
  each refresh token server-side, similar to password hashing — if the
  database is compromised, attackers don't get usable tokens directly.
- **Rotate on every use.** Each time a refresh token is used to get a new
  access token, issue a **new** refresh token and invalidate the old one.
  The client stores the new one and the old one can never be used again.
- **Detect reuse.** If a refresh token that was already rotated out (i.e.,
  already used once) is presented again, that's a strong signal it was
  stolen — treat it as a compromise, and revoke the **entire token
  family** (all tokens descended from that original login), forcing the
  user to re-authenticate.

```text
Login → Refresh Token v1
           ↓ (used)
        Refresh Token v2 (v1 now invalid)
           ↓ (used)
        Refresh Token v3 (v2 now invalid)

If v1 or v2 is presented again → reuse detected → revoke whole family
```

### Interview question

**Q: Why should access tokens be short-lived?**

> To reduce the impact of token theft. A stolen short-lived access token
> becomes useless relatively quickly.

---

## 4. Sessions vs JWT

Another very common interview question.

### Session-based authentication

Server stores session information.

```text
Client
  ↓
Login
  ↓
Server creates session
  ↓
Session ID → Client
```

Example:

```text
Cookie:
sessionId=abc123
```

Server:

```text
abc123 → userId: 123
```

The server maintains the session.

### JWT authentication

The token itself contains claims.

```text
Client
  ↓
JWT
  ↓
Server verifies JWT
```

The server doesn't necessarily need to maintain authentication state for
every token.

### Comparison

| Sessions | JWT |
| --- | --- |
| Server stores session | Token carries claims |
| Usually cookie-based | Often `Authorization` header or cookie |
| Easy server-side invalidation | Stateless invalidation is harder |
| Requires session storage in distributed setups | Easier to distribute |
| Session ID itself contains little information | JWT contains claims |
| Common for traditional web apps | Common for APIs/microservices |

### Important interview point

Don't say:

> JWT is always better than sessions.

That's incorrect.

The appropriate choice depends on the application architecture.

---

## 5. Password Hashing

**Never store passwords as plain text.**

Bad:

```json
{
  "password": "mypassword123"
}
```

If the database is compromised, passwords are immediately exposed.

Instead:

```text
password
   ↓
bcrypt / Argon2
   ↓
password hash
   ↓
database
```

Example with bcrypt:

```js
const hash = await bcrypt.hash(password, 12);
```

During login:

```js
const isValid = await bcrypt.compare(password, user.passwordHash);
```

### Hashing vs Encryption

**Hashing**

```text
password → hash
```

One-way operation.

**Encryption**

```text
data → encrypted data
```

Can be decrypted using the appropriate key.

Passwords should generally be **hashed**, not reversibly encrypted.

### Interview question

**Q: Why do we hash passwords?**

> Password hashing protects passwords even if the database is compromised.
> A password hash is designed to be computationally difficult to reverse.

---

## 6. CORS

CORS = **Cross-Origin Resource Sharing**.

It controls which origins are allowed to make browser-based cross-origin
requests to your server.

Example:

```text
Frontend:
https://myapp.com

Backend:
https://api.myapp.com
```

These are different origins.

Express:

```js
const cors = require("cors");

app.use(
  cors({
    origin: "https://myapp.com",
  }),
);
```

You may also see:

```js
app.use(cors());
```

This allows broad cross-origin access and isn't automatically appropriate
for production.

### Important

CORS is primarily a **browser security mechanism**.

It does not stop:

- Postman
- curl
- server-to-server requests

### Interview question

**Q: Does CORS protect your API from attackers using Postman?**

> No. CORS is enforced by browsers. An attacker can still send requests
> using tools such as curl or Postman, so authentication and authorization
> are still required.

---

## 7. CSRF

CSRF = **Cross-Site Request Forgery**.

It happens when an attacker tricks a user's browser into sending an
authenticated request to your application.

It's particularly relevant when authentication credentials are
automatically sent by the browser, such as cookies.

Example:

```text
User logged into bank.com
        ↓
Visits malicious-site.com
        ↓
Malicious site causes request to bank.com
        ↓
Browser automatically sends bank cookie
```

Potentially:

```http
POST /transfer
```

### Common CSRF defenses

#### CSRF token

Server generates a token.

```text
Request
   ↓
CSRF token
   ↓
Server validates token
```

#### SameSite cookies

For example:

```js
res.cookie("session", token, {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
});
```

`sameSite: "lax"` still allows the cookie to be sent on **top-level
cross-site GET navigations** (e.g., a link on another site that navigates
the browser to yours) — it only blocks it on cross-site subrequests like
form POSTs, images, and fetch/XHR from another origin. That means `lax`
alone is **not a complete CSRF defense**. For state-changing requests
(POST/PUT/PATCH/DELETE), combine `sameSite` with CSRF tokens, or use
`sameSite: "strict"` where your application's UX can tolerate it (it
prevents the cookie from being sent even on those top-level navigations).

### Important distinction

```text
CORS ≠ CSRF protection
```

They solve different problems.

---

## 8. XSS

XSS = **Cross-Site Scripting**.

An attacker injects malicious JavaScript into content that gets executed in
another user's browser.

Example malicious input:

```html
<script>
  alert("Hacked");
</script>
```

If your application renders untrusted input as HTML without proper
escaping/sanitization, it can lead to XSS.

### Types

Know these three:

```text
Stored XSS
Reflected XSS
DOM-based XSS
```

### Prevention

- Escape output
- Sanitize untrusted HTML where HTML is actually allowed
- Avoid unsafe DOM APIs such as `innerHTML` with untrusted content
- Use Content Security Policy where appropriate
- Don't put sensitive secrets in browser-accessible storage unnecessarily

### Interview question

**Q: What's the difference between XSS and CSRF?**

> XSS involves injecting malicious script into a page so it executes in a
> user's browser. CSRF tricks a user's browser into making an unwanted
> authenticated request.

---

## 9. Rate Limiting

Rate limiting controls how many requests a client can make within a given
period.

Example:

```text
100 requests / minute / IP
```

This helps protect against:

- Brute-force attacks
- Login abuse
- API abuse
- Denial-of-service attempts
- Excessive resource consumption

Express example:

```js
import rateLimit from "express-rate-limit";

const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
});

app.use(limiter);
```

For login endpoints, you generally want **stricter limits** than for
ordinary endpoints.

In distributed systems, rate limiting often needs a shared store such as
Redis rather than relying only on in-memory counters.

### IP-based limiting isn't enough on its own

Rate-limiting auth endpoints **by IP alone is insufficient** — botnets and
rotating proxies can spread login attempts across many IPs, each staying
under the per-IP limit while collectively hammering the same account. For
login/auth endpoints specifically, also rate-limit **by account/username**
(and consider combining both: per-IP and per-account limits), so an
attacker can't just rotate IPs to bypass the limit.

---

## 10. Input Validation

Never blindly trust user input.

Example:

```http
POST /users
```

Input:

```json
{
  "email": "invalid",
  "age": "hello"
}
```

Your API should validate it.

Using a validation library such as Zod:

```js
const userSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().int().positive(),
});
```

Then:

```js
const result = userSchema.safeParse(req.body);

if (!result.success) {
  return res.status(400).json({
    message: "Invalid input",
  });
}
```

### Validate things like

```text
Body
Query parameters
Path parameters
Headers
File uploads
```

### Important

Input validation helps prevent:

- Invalid data
- Unexpected application behavior
- Injection vulnerabilities
- Resource abuse

But validation **alone isn't enough**. Database queries should still use
parameterization/prepared statements, and output should still be handled
safely.

---

## 11. HTTPS & Security Headers

Beyond application-level auth logic, a few baseline platform-level
protections are expected in any production Node.js app.

### HTTPS/TLS

All traffic — especially login, tokens, and cookies — should be served
over **HTTPS**, not plain HTTP. Without TLS, credentials, tokens, and
session cookies travel in plaintext and can be intercepted (e.g., on
public Wi-Fi, or by anything sitting on the network path). Cookies marked
`secure: true` are only ever sent over HTTPS, which is another reason TLS
is a baseline requirement, not an optional extra.

### Security headers (Helmet, HSTS)

Sensible HTTP response headers reduce the attack surface for things like
XSS, clickjacking, and protocol downgrade attacks. In Express, the
[`helmet`](https://www.npmjs.com/package/helmet) middleware sets a
reasonable set of these by default:

```js
const helmet = require("helmet");

app.use(helmet());
```

This includes headers such as:

- **`Strict-Transport-Security` (HSTS)** — tells browsers to only ever
  connect to your site over HTTPS for a given period, even if the user
  types `http://` or clicks an `http://` link, preventing downgrade
  attacks.
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options` / frame-ancestors (clickjacking protection)
- A baseline `Content-Security-Policy`

None of this replaces the auth/validation/rate-limiting work covered
above — it's a complementary, cheap-to-add baseline layer.

---

## 🔥 Most Important Interview Questions

For this chapter, make sure you can answer these without notes:

### Authentication

1. What is authentication?
2. What is authorization?
3. Authentication vs authorization?
4. How would you implement authentication in Express?

### JWT

1. What is JWT?
2. What are the three parts of JWT?
3. Is JWT encrypted?
4. What is the purpose of the JWT signature?
5. How do you verify a JWT?
6. Where should JWT secrets be stored?

### Access/Refresh Tokens

1. What is an access token?
2. What is a refresh token?
3. Why use both?
4. Why should access tokens be short-lived?
5. What happens when an access token expires?
6. How would you revoke/rotate refresh tokens?

### Sessions

1. JWT vs session authentication?
2. What are the advantages of sessions?
3. What are the advantages of JWT?
4. When would you choose sessions over JWT?

### Passwords

1. Why shouldn't passwords be stored directly?
2. Hashing vs encryption?
3. What is bcrypt?
4. Why is password hashing intentionally slow?

### Web Security

1. What is CORS?
2. Is CORS an authentication mechanism?
3. What is CSRF?
4. How do you prevent CSRF?
5. What is XSS?
6. XSS vs CSRF?

### API Security

1. What is rate limiting?
2. How would you protect a login endpoint from brute-force attacks?
3. Why is input validation important?
4. Where should input validation happen?
5. How would you secure an Express REST API?

---

## ⭐ What you should be able to explain in an interview

Don't just memorize definitions. Be able to explain this complete flow:

```text
                USER
                  │
                  ▼
              POST /login
                  │
                  ▼
          Validate input
                  │
                  ▼
       Find user in database
                  │
                  ▼
        Compare password hash
                  │
             ┌────┴────┐
             │         │
           Invalid    Valid
             │         │
             ▼         ▼
           401     Generate tokens
                       │
                ┌──────┴──────┐
                ▼             ▼
          Access Token   Refresh Token
                │             │
                ▼             ▼
          API requests    Refresh endpoint
                │             │
                ▼             ▼
          Verify token    Issue new access
                │             │
                ▼             ▼
          Authorization
                │
                ▼
          Protected API
```

If you understand this flow, you have the **core practical
authentication/security knowledge expected for a Node.js backend
interview**.
