# 📚 **Ultimate Beginner-Friendly Guide to Move on Sui**

---

## 🗂️ Table of Contents

> Click any header in most Markdown viewers to jump.

1. [Intro: Packages & Project Layout](#1-intro-packages--project-layout)
2. [Module Anatomy](#2-module-anatomy)
3. [Imports with `use`](#3-imports-with-use)
4. [Objects & Struct Basics](#4-objects--struct-basics)
5. [Variables (`let`) & Constants (`const`)](#5-variables-let--constants-const)
6. [Control-Flow: `if`, `assert!`, Recursion](#6-control-flow-if-assert-recursion)
7. [Expressions, Scope & Operators](#7-expressions-scope--operators)
8. [Functions: Args, Return, Visibility, `entry`](#8-functions-args-return-visibility-entry)
9. [Abilities (`key`, `store`, `copy`, `drop`)](#9-abilities-key-store-copy-drop)
10. [Deep-Dive on Objects](#10-deep-dive-on-objects)
11. [Using Objects: Reference vs Value](#11-using-objects-reference-vs-value)
12. [Object Ownership Models](#12-object-ownership-models)
13. \[Complete *Hello World* Explained]\(#13-complete-hello world-explained)
14. [Mini-Challenge: Build `AnimalObject`](#14-mini-challenge-build-animalobject)
15. [Cheat-Sheet & Quick Reference](#15-cheat-sheet--quick-reference)

---

## 1️⃣ Intro: Packages & Project Layout

### Creating a package

```bash
sui move new hello_world
```

* Creates a directory named **`hello_world`**
* `sources/` → Move source files
* `Move.toml` → dependencies & aliases

<details><summary>🔍 Tip: Editing <code>Move.toml</code></summary>

```toml
[addresses]
sui = "0x2"          # built-in Sui framework
std = "0x1"          # standard library
my_friend = "0xabc…" # custom alias

[dependencies]
Sui = { local = "../sui-framework" }
```

</details>

---

## 2️⃣ Module Anatomy

Every on-chain program is a **module**.

```move
module <package_name>::<module_name> {
    // 1. Imports
    // 2. Struct/object definitions
    // 3. Optional init()   (constructor)
    // 4. Private / public / entry functions
}
```

*🔑 Why?* — Modules group code and can hold multiple structs & functions.

---

## 3️⃣ Imports with `use`

Syntax → `use <address_or_alias>::<ModuleName>[::{Subitem}];`

```move
use sui::transfer;            // whole module
use std::string::{Self, String}; // bring struct String + alias Self
```

Common imports:

| Statement                                 | Purpose                                    |
| ----------------------------------------- | ------------------------------------------ |
| `use std::string;`                        | Work with UTF-8 strings                    |
| `use sui::transfer;`                      | Move objects between owners                |
| `use sui::object;`                        | Low-level object helpers (`new`, `delete`) |
| `use sui::tx_context::{Self, TxContext};` | Access transaction info                    |

---

## 4️⃣ Objects & Struct Basics

### Why “objects”?

On Sui, **objects = on-chain data units**. They always carry:

* A unique ID (`UID`)
* One or more abilities (`key` required)

```move
use sui::object::UID;

struct HelloWorldObject has key {
    id: UID,             // mandatory for key objects
    text: string::String // custom field
}
```

### Adding an `init` (constructor)

Runs **once** at publish time:

```move
fun init(ctx: &mut TxContext) {
    transfer::transfer(
        HelloWorldObject { id: object::new(ctx), text: string::utf8(b"Hi") },
        tx_context::sender(ctx)
    );
}
```

---

## 5️⃣ Variables (`let`) & Constants (`const`)

```move
// ── Variables ─────────────────────────────
let x;              // type inferred later
let y: u8;          // declare type
let z = false;      // infer + init
let k: u16 = 42;

// ── Constants ─────────────────────────────
const MAX_SUPPLY: u64 = 1_000_000;
```

*Constants live at module level, never change, and cannot leak outside.*

---

## 6️⃣ Control-Flow: `if`, `assert!`, Recursion

### `if` as an expression

```move
let fee = if (is_vip) { 0 } else { 10 }; // fee is u8
```

*Both branches must return the same type.*

### `assert!`

```move
assert!(amount > 0, 0); // abort tx if condition false
```

### Recursion (Move avoids loops)

```move
fun fib(n: u64): u64 {
    if (n <= 1) { n } else { fib(n-1) + fib(n-2) }
}
```

---

## 7️⃣ Expressions, Scope & Operators

* **Expression** = any code that *returns* something OR ends with `;`.
* `{ ... }` introduces a **scope**. Inner vars shadow outer ones.

```move
{
    let inner = 9;
    // inner visible only here
}
```

### Integer operators

`+  -  *  /  %  <<  >>  &  ^`

### Ignoring results

```move
let _ = expensive_call(); // silence “unused” warning
```

---

## 8️⃣ Functions: Args, Return, Visibility, `entry`

```move
fun add(a: u8, b: u8): u8 {
    a + b                // last expr is return
}
```

### Visibility table

| Keyword            | Where callable?                                                                 |
| ------------------ | ------------------------------------------------------------------------------- |
| *(none)* (private) | Same module only                                                                |
| `public`           | Any module                                                                      |
| `entry`            | Directly by transactions <br>*(no return allowed; last arg = `&mut TxContext`)* |

```move
public entry fun mint(ctx: &mut TxContext) { … }
```

---

## 9️⃣ Abilities (`key`, `store`, `copy`, `drop`)

| Ability | Grants                                    |
| ------- | ----------------------------------------- |
| `key`   | Can live in global storage, needs `UID`   |
| `store` | Data inside object can also live on chain |
| `copy`  | Value can be duplicated with `copy`       |
| `drop`  | Value can be dropped automatically        |

**Rule of thumb:**
`struct MyObj has key, store { … }` covers 90 % of contracts.

---

## 10️⃣ Deep-Dive on Objects

### Creating

```move
fun new(a: u8, b: u8, ctx: &mut TxContext): SumObject {
    SumObject {
        id: object::new(ctx),
        number_1: a,
        number_2: b
    }
}
```

### Storing (making global)

```move
public entry fun create(a: u8, b: u8, ctx: &mut TxContext) {
    let obj = new(a, b, ctx);
    transfer::public_transfer(obj, tx_context::sender(ctx));
}
```

*No `print` in Move → viewing happens via Explorer.*

---

## 11️⃣ Using Objects: Reference vs Value

### Pass **by reference**

```move
public entry fun copy_into(src: &SumObject, dest: &mut SumObject) {
    dest.number_1 = src.number_1;
    dest.number_2 = src.number_2;
}
```

### Pass **by value**

```move
public entry fun update(n: u8, obj: SumObject): SumObject {
    let mut tmp = obj;
    tmp.number_1 = n;
    tmp          // returned; caller decides where to store
}
```

#### Deleting a by-value object

```move
public entry fun burn(obj: SumObject) {
    let SumObject { id, number_1: _, number_2: _ } = obj;
    object::delete(id);
}
```

---

## 12️⃣ Object Ownership Models

| Model             | Who can touch it? | How to create                                       |
| ----------------- | ----------------- | --------------------------------------------------- |
| **Address-owned** | That address only | `transfer::transfer` or `transfer::public_transfer` |
| **Immutable**     | Read-only for all | `transfer::public_freeze_object(obj)`               |
| **Shared**        | Everyone          | `transfer::share_object(obj)`                       |
| **Wrapped**       | Nested in parent  | `struct Parent { child: Child }`                    |
| **Dynamic Field** | Key/value pairs   | (Advanced — not covered here)                       |

---

## 13️⃣ Complete *Hello World* Explained

```move
// Copyright (c) 2022, Sui Foundation
// SPDX-License-Identifier: Apache-2.0
module hello_world::hello_world {

    use std::string;
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};

    struct HelloWorldObject has key, store {
        id: UID,
        text: string::String
    }

    /// Mint one object containing “Hello World!”
    public entry fun mint(ctx: &mut TxContext) {
        let object = HelloWorldObject {
            id: object::new(ctx),                     // unique UID
            text: string::utf8(b"Hello World!")       // UTF-8 string
        };
        transfer::public_transfer(object, tx_context::sender(ctx));
    }
}
```

**Flow**

1. **`mint`** is an `entry` → anyone may call.
2. `object::new` anchors a fresh on-chain UID.
3. Object is transferred to caller’s address.
4. Caller now owns it; viewable in Explorer.

---

## 14️⃣ Mini-Challenge: Build `AnimalObject`

Create a new module and:

1. Define `AnimalObject`

   ```move
   struct AnimalObject has key, store {
       id: UID,
       name: string::String,
       no_of_legs: u8,
       favorite_food: string::String,
   }
   ```
2. Write `public entry fun create(...)` that:

   * accepts `name_bytes`, `legs`, `food_bytes`
   * builds the object (string via `string::utf8`)
   * transfers it to `tx_context::sender(ctx)`

👉 **Deploy** to testnet, call `create`, and share the resulting Object ID!

---

## 15️⃣ Cheat-Sheet & Quick Reference

| Topic        | One-liner                                   |
| ------------ | ------------------------------------------- |
| New package  | `sui move new pkg_name`                     |
| Publish      | `sui client publish --gas-budget 100000000` |
| Call `entry` | `sui client call --function mint ...`       |
| Module decl. | `module pkg::mod { … }`                     |
| Import       | `use sui::transfer;`                        |
| Variable     | `let x: u8 = 5;`                            |
| Constant     | `const PI: u64 = 314;`                      |
| If           | `let y = if (flag) { 1 } else { 0 };`       |
| Assert       | `assert!(x > 0, 1);`                        |
| New UID      | `object::new(ctx)`                          |
| Transfer     | `transfer::transfer(obj, addr)`             |
| Delete UID   | `object::delete(id)`                        |

---

### 🎉 You’re All Set!
