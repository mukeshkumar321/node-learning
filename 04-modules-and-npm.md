# Modules & npm ⭐⭐⭐

## Topics Covered

- [1. CommonJS](#1-commonjs)
- [2. ES Modules](#2-es-modules)
- [3. require vs import](#3-require-vs-import)
- [4. module.exports vs exports](#4-moduleexports-vs-exports)
- [5. Module Resolution and Caching](#5-module-resolution-and-caching)
- [6. npm](#6-npm)
- [7. package.json](#7-packagejson)
- [8. Dependencies vs devDependencies](#8-dependencies-vs-devdependencies)
- [9. npm Scripts](#9-npm-scripts)
- [10. Semantic Versioning](#10-semantic-versioning)

---

## 1. CommonJS

CommonJS (CJS) is the traditional Node.js module system.

### Export

```js
// math.js

const add = (a, b) => a + b;

module.exports = add;
```

### Import

```js
// app.js

const add = require("./math");

console.log(add(2, 3)); // 5
```

### Exporting multiple values

```js
// math.js

const add = (a, b) => a + b;
const subtract = (a, b) => a - b;

module.exports = {
  add,
  subtract,
};
```

```js
const { add, subtract } = require("./math");
```

### Interview point

CommonJS uses:

```text
module.exports → export
require()     → import
```

CommonJS modules are loaded **synchronously**, which fits Node's server-side
environment well.

---

## 2. ES Modules

ES Modules (ESM) are the standardized JavaScript module system.

### Export

```js
// math.js

export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
```

### Import

```js
import { add, subtract } from "./math.js";
```

You can also use default exports:

```js
// math.js

export default function add(a, b) {
  return a + b;
}
```

```js
import add from "./math.js";
```

### Node.js and ESM

Node.js supports ESM when you configure it appropriately, commonly using:

```json
{
  "type": "module"
}
```

Then `.js` files are treated as ES modules.

Alternatively, `.mjs` explicitly represents an ES module.

---

## 3. `require` vs `import`

This is a **very common interview question**.

| CommonJS | ES Modules |
| --- | --- |
| `require()` | `import` |
| `module.exports` | `export` |
| Traditional Node.js | JavaScript standard |
| Usually synchronous loading | Designed around static module structure |
| Can be conditionally called | Static `import` must be at module top level |
| `.cjs` | `.mjs` |
| `require('./file')` | `import x from './file.js'` |

### CommonJS

```js
const fs = require("fs");
```

### ESM

```js
import fs from "fs";
```

### Important interview distinction

Don't say:

> "`import` is asynchronous and `require` is synchronous."

That's an oversimplification.

A better answer:

> **CommonJS uses synchronous `require()` loading, while ES modules have a
> static module structure and an asynchronous-capable module
> loading/linking model.**

Also, modern Node.js supports both systems.

---

## 4. `module.exports` vs `exports`

This is another **very important interview topic**.

Initially:

```text
exports === module.exports
```

Both point to the same object.

So this works:

```js
exports.add = (a, b) => a + b;
```

because you're adding a property to the same exported object.

This also works:

```js
module.exports.add = (a, b) => a + b;
```

### The common mistake

```js
exports = function () {
  console.log("Hello");
};
```

This **does not change what the module exports**.

Why?

Initially:

```text
exports ────────┐
               ↓
          module.exports
               ↓
             {}
```

When you do:

```js
exports = function () {};
```

you only reassign the local `exports` variable.

```text
exports → function()

module.exports → {}
```

Node returns:

```text
module.exports
```

not the reassigned `exports`.

### Correct approach

If you want to export a function directly:

```js
module.exports = function () {
  console.log("Hello");
};
```

Then:

```js
const greet = require("./greet");

greet();
```

### Interview question

**Q: What is the difference between `exports` and `module.exports`?**

Good answer:

> `exports` is initially a reference to `module.exports`. Adding properties
> through `exports` works, but reassigning `exports` does not change the
> actual exported value. For replacing the entire export, use
> `module.exports`.

---

## 5. Module Resolution & Caching

When you write:

```js
const math = require("./math");
```

Node needs to figure out **which module you mean**.

For CommonJS, simplified resolution includes:

```text
1. Core/built-in module
        ↓
2. Relative/absolute file
        ↓
3. Directory/package resolution
        ↓
4. node_modules
```

Example:

```js
const fs = require("fs");
```

Node recognizes `fs` as a built-in module.

```js
const math = require("./math");
```

Node resolves the local module.

```js
const express = require("express");
```

Node looks for the installed package through `node_modules` and its package
metadata.

### What is `node_modules`?

`node_modules` is the directory where installed packages actually live on
disk. When you run `npm install`, packages are downloaded into
`node_modules`, and Node's module resolution algorithm walks up through
`node_modules` directories (starting from the requiring file's own folder
and moving up toward the filesystem root) until it finds a match. This is
also why `node_modules` is normally excluded from version control — it's a
derived artifact that can be recreated from `package.json` /
`package-lock.json`.

### Module caching

Once a CommonJS module is loaded, Node caches it.

Example:

```js
// counter.js

console.log("Counter module loaded");

module.exports = {
  count: 0,
};
```

```js
// app.js

const counter1 = require("./counter");
const counter2 = require("./counter");
```

The module's initialization code generally runs **once per resolved
module**, and subsequent `require()` calls return the cached export.

Therefore:

```js
counter1 === counter2;
```

is typically:

```text
true
```

### Interview question

**Q: Why does Node.js cache modules?**

To avoid repeatedly loading and executing the same module, improving
performance and preserving module state.

### Caching is keyed by resolved file path, not package name

Node's module cache key is the **fully resolved absolute file path**, not
the package name you typed in `require()`. This matters because it means two
`require("some-package")` calls can load **different instances** of what
looks like "the same" module if they resolve to different files on disk —
for example:

- Nested `node_modules` folders (a dependency ships its own copy of a
  package instead of sharing the top-level one)
- `npm link` / symlinked packages resolving to a path outside the project's
  own `node_modules`

```text
/project/node_modules/some-lib          ← instance A
/project/node_modules/dep/node_modules/some-lib  ← instance B (different file path)
```

If `some-lib` keeps module-level state (a singleton, a cache, a connection
pool), instance A and instance B are **not the same object**, even though
they're "the same package" — a classic source of "why is my singleton
state duplicated?" bugs.

---

## 6. npm

**npm = Node Package Manager**

It is used to:

- install packages
- manage dependencies
- publish packages
- run scripts
- manage package versions

Example:

```bash
npm install express
```

This typically:

1. Downloads the package
2. Places it in `node_modules`
3. Records it in `package.json`
4. Updates the lockfile

### What is `package-lock.json`?

`package.json` records dependency ranges like `^5.1.0`, which can resolve to
different exact versions over time as new releases come out. `package-lock.json`
locks in the **exact resolved version** (and the resolved versions of every
transitive dependency) that was installed. Committing it ensures that
`npm install` produces the **same dependency tree** on every machine and in
CI, not just something compatible with the version ranges.

### Important commands

```bash
npm init
```

Creates a `package.json`.

```bash
npm install express
```

Installs a production dependency.

```bash
npm install -D nodemon
```

Installs a development dependency.

```bash
npm uninstall express
```

Removes a dependency.

```bash
npm install
```

Installs dependencies described by the project metadata/lockfile.

```bash
npm update
```

Updates packages within the ranges allowed by the project's dependency
specifications.

---

## 7. `package.json`

`package.json` describes your Node.js project.

Example:

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^5.1.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}
```

Important fields:

```text
name
version
type
scripts
dependencies
devDependencies
engines
```

### `type`

```json
{
  "type": "module"
}
```

Tells Node to treat `.js` files as ES modules by default within that package
scope.

Without that, `.js` is normally treated as CommonJS unless another package
configuration applies.

---

## 8. Dependencies vs devDependencies

### `dependencies`

Packages required by the application at runtime.

Example:

```bash
npm install express
```

Produces something like:

```json
"dependencies": {
  "express": "^5.1.0"
}
```

### `devDependencies`

Packages needed mainly during development/build/testing.

Example:

```bash
npm install -D nodemon
```

```json
"devDependencies": {
  "nodemon": "^3.1.0"
}
```

Examples:

**dependencies**

```text
express
mongoose
jsonwebtoken
```

**devDependencies**

```text
nodemon
eslint
prettier
jest
typescript
```

### Interview question

**Q: Why shouldn't Nodemon usually be in dependencies?**

Because Nodemon is a development tool and isn't required for the
application to run in production.

---

## 9. npm Scripts

Scripts are commands defined inside `package.json`.

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "jest"
  }
}
```

Run them using:

```bash
npm start
```

or:

```bash
npm run dev
```

or:

```bash
npm test
```

### Why use scripts?

Instead of remembering:

```bash
node server.js
```

everyone on the team can simply run:

```bash
npm start
```

This gives the project a consistent interface.

### Special npm scripts

Some scripts can be run without `npm run`.

```bash
npm start
npm test
```

For custom scripts:

```bash
npm run dev
npm run build
```

---

## 10. Semantic Versioning

Semantic Versioning = **SemVer**

Version format:

```text
MAJOR.MINOR.PATCH
```

Example:

```text
5.2.3
```

Meaning:

```text
5 → MAJOR
2 → MINOR
3 → PATCH
```

### MAJOR

Breaking/incompatible changes.

```text
4.x.x → 5.x.x
```

Your existing code may stop working.

### MINOR

New backward-compatible functionality.

```text
5.1.0 → 5.2.0
```

### PATCH

Backward-compatible bug fixes.

```text
5.2.3 → 5.2.4
```

---

## Version ranges — VERY IMPORTANT

Suppose:

```json
"express": "^5.1.0"
```

`^` means approximately:

> Allow compatible updates according to SemVer, generally without changing
> the major version.

So the allowed range can include:

```text
5.1.1
5.2.0
5.x.x
```

but not:

```text
6.0.0
```

### `~`

```json
"express": "~5.1.0"
```

Generally allows patch-level updates:

```text
5.1.1
5.1.2
```

but not:

```text
5.2.0
```

### Exact version

```json
"express": "5.1.0"
```

Requests that exact version.

---

## ⭐ Interview Questions You Must Know

For this chapter, make sure you can answer these **without looking at
notes**:

### Modules

1. What is a Node.js module?
2. What is CommonJS?
3. What are ES modules?
4. CommonJS vs ES Modules?
5. `require` vs `import`?
6. `module.exports` vs `exports`?
7. Why does reassigning `exports` not work?
8. How does Node resolve a module?
9. What is module caching?
10. Does Node execute a required module every time?

### npm

1. What is npm?
2. What is `package.json`?
3. What is `package-lock.json`?
4. `dependencies` vs `devDependencies`?
5. What is `node_modules`?
6. What does `npm install` do?
7. `npm install package` vs `npm install -D package`?
8. What are npm scripts?
9. `npm start` vs `npm run start`?
10. Why shouldn't `node_modules` usually be committed to Git?

### SemVer

1. What is Semantic Versioning?
2. What are MAJOR, MINOR and PATCH?
3. What does `^5.1.0` mean?
4. What does `~5.1.0` mean?
5. Difference between `^`, `~`, and an exact version?
6. Why can dependency updates sometimes break an application?

---

## 🎯 What I would prioritize for your interview

**Must know extremely well:**

```text
CommonJS
        ↓
ES Modules
        ↓
require vs import
        ↓
module.exports vs exports
        ↓
Module resolution
        ↓
Module caching
        ↓
npm
        ↓
package.json
        ↓
dependencies vs devDependencies
        ↓
npm scripts
        ↓
Semantic Versioning
```

The **highest-value interview traps** are:

> `exports` vs `module.exports`
>
> `require()` vs `import`
>
> module caching
>
> `dependencies` vs `devDependencies`
>
> `package.json` vs `package-lock.json`
>
> `^` vs `~`
>
> CommonJS vs ES Modules

If you can explain these with a small code example, you're at a **good
interview-ready level for this chapter**.
