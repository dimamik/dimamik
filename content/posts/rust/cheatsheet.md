---
title: "Rust cheatsheet"
date: 2026-04-26T00:00:00Z
description: Rust thingies ;)
tags:
  - rust
  - low-level
categories:
  - Rust
draft: false
---

# Rust Cheat Sheet

## Variables

```rust
let x = 5;              // immutable
let mut y = 5;          // mutable
let x = x + 1;         // shadowing (rebinding)
```

## Types

```rust
i8/i16/i32/i64/i128    // signed integers (default: i32)
u8/u16/u32/u64/u128    // unsigned integers
usize                  // pointer-sized int, used for indices/lengths
f32/f64                // floats (default: f64)
bool                   // true/false
char                   // 4 bytes, Unicode scalar value

&str                   // borrowed string slice (read-only view, lives in binary)
String                 // owned, heap-allocated, growable

(i32, f64, &str)       // tuple - fixed size, mixed types, stack
[i32; 3]               // array - fixed size, same type, stack
Vec<i32>               // vector - growable, same type, heap
```

## Functions

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b               // no semicolon = return value
}

let double = |x: i32| x * 2;   // closure (anonymous fn)
let add = |a, b| a + b;        // closure, types inferred
```

## Control Flow

```rust
if x > 5 { ... } else { ... }  // expression, returns value

// match = case in Elixir, exhaustive
match x {
    1 => "one",
    2 | 3 => "two or three",
    4..=9 => "four to nine",
    _ => "other",               // catch-all
}

for item in &vec { ... }
while condition { ... }
loop { break value; }           // infinite loop, can return value
```

## Ownership

```rust
let s1 = String::from("hi");
let s2 = s1;            // s1 MOVED to s2, s1 invalid
let s2 = s1.clone();    // explicit deep copy, both valid

// i32, f64, bool, char - COPIED automatically (stack types)
let a = 5;
let b = a;              // both valid
```

## References & Borrowing

```rust
&T                      // immutable reference (borrow)
&mut T                  // mutable reference (exclusive borrow)

// Rules:
// - any number of &T at once
// - OR exactly one &mut T
// - never both at the same time
// - references must always be valid

fn foo(s: &String) { }         // borrows, doesn't take ownership
fn foo(s: &mut String) { }     // borrows mutably
```

## Structs

```rust
struct User {
    name: String,       // private by default
    pub age: u32,       // public
}

impl User {
    fn new(name: &str) -> User { User { name: String::from(name), age: 0 } }  // associated fn
    fn greet(&self) { }         // immutable method
    fn rename(&mut self) { }    // mutable method
    fn consume(self) { }        // takes ownership
}
```

## Enums

```rust
enum Shape {
    Circle(f64),                        // tuple variant
    Rectangle(f64, f64),
    Triangle { base: f64, height: f64 }, // struct variant
    None,                               // unit variant
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(r) => PI * r * r,
            Shape::Rectangle(w, h) => w * h,
            Shape::Triangle { base, height } => 0.5 * base * height,
            Shape::None => 0.0,
        }
    }
}
```

## Option and Result

```rust
// Option - no null in Rust
Option<T>  =  Some(value)  |  None

// Result - no exceptions in Rust
Result<T, E>  =  Ok(value)  |  Err(error)

// Pattern match
match result {
    Ok(v) => ...,
    Err(e) => ...,
}

// Shortcuts
result.unwrap()             // Ok(v) -> v, panics on Err
result.unwrap_or(default)   // Ok(v) -> v, Err -> default
result.map(|v| v * 2)       // transform Ok value
if let Ok(v) = result { }   // handle one variant

// ? operator - propagate error up (like Elixir's `with`)
fn foo() -> Result<i32, SomeError> {
    let n = fallible_fn()?;  // returns Err early if Err
    Ok(n * 2)
}
```

## Traits

```rust
trait Area {
    fn area(&self) -> f64;              // required
    fn describe(&self) -> String {      // default impl
        format!("area: {}", self.area())
    }
}

impl Area for Circle {
    fn area(&self) -> f64 { PI * self.radius * self.radius }
}

fn print_area(shape: &impl Area) { }       // trait as param
fn foo<T: Area>(shape: &T) { }             // generic + trait bound
fn foo<A: Area, B: Area>(a: &A, b: &B) { } // multiple types

#[derive(Debug, Clone, PartialEq)]  // auto-implement common traits
struct Point { x: f64, y: f64 }
```

## Modules

```rust
mod geometry {          // inline module
    pub struct Circle { pub radius: f64 }
}

mod geometry;           // load from src/geometry.rs or src/geometry/mod.rs

use geometry::Circle;           // alias (like Elixir's alias)
use geometry::{Circle, utils};  // multiple
use geometry::utils::*;         // glob import
```

## Visibility

```rust
// private by default (opposite of Elixir)
pub fn foo() { }        // public
pub struct Foo { }
pub field: i32,         // struct fields also need pub
```

## Iterators (like Elixir's Enum)

```rust
vec.iter()              // borrows elements as &T
vec.into_iter()         // consumes vec, yields T
vec.iter_mut()          // yields &mut T

.map(|x| x * 2)
.filter(|x| x > 0)
.fold(0, |acc, x| acc + x)     // like Enum.reduce
.any(|x| x > 5)
.all(|x| x > 0)
.find(|x| x > 5)
.count()
.collect::<Vec<i32>>()         // consume iterator into collection
.copied()                       // &T -> T for Copy types
```

## Printing

```rust
println!("{x}");        // variable
println!("{}", x);      // argument (needed for expressions/fields)
println!("{x:?}");      // debug format (needs #[derive(Debug)])
println!("{x:.2}");     // 2 decimal places
format!("...")          // returns String instead of printing
```

## Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn my_test() {
        assert_eq!(2 + 2, 4);
        assert!(condition);
        assert_ne!(a, b);
    }

    #[test]
    #[should_panic]
    fn panics() { panic!("oh no"); }
}
```

Doc tests:

````rust
/// # Examples
/// ```
/// let x = 1 + 1;
/// assert_eq!(x, 2);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
````

## Concurrency

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel::<String>();
let tx2 = tx.clone();   // multiple producers
drop(tx);               // drop original - channel closes when all tx clones drop

thread::spawn(move || { // `move` transfers ownership into thread (required)
    tx2.send("hello".to_string()).unwrap();
});

for msg in rx { }       // blocks until all senders dropped
```

## Async / Concurrency Options

| Approach            | When to use                           | Elixir analogy                |
| ------------------- | ------------------------------------- | ----------------------------- |
| `std::thread`       | CPU-bound work, true parallelism      | `Task.async` on separate node |
| `tokio` async tasks | I/O-bound work (HTTP, DB, sockets)    | BEAM processes for I/O        |
| `Arc<Mutex<T>>`     | shared mutable state across threads   | GenServer holding state       |
| `mpsc::channel`     | message passing between threads/tasks | `send/2` + `receive`          |
| Actix actors        | actor model, supervision trees        | GenServer + OTP               |

**Key async primitives (tokio):**

```rust
#[tokio::main]
async fn main() { }            // entry point, starts runtime

async fn foo() -> i32 { 42 }  // async function, returns Future
foo().await                    // drive future to completion

tokio::spawn(async { ... })    // spawn async task (like Process.spawn)
tokio::join!(a(), b())         // run concurrently, wait for both
tokio::select! {               // wait for whichever completes first
    v = a() => ...,            // like selective receive in Erlang
    v = b() => ...,
}
tokio::time::sleep(Duration::from_millis(100)).await  // async sleep

// Async channel (tokio) - like std::mpsc but for async
let (tx, mut rx) = tokio::sync::mpsc::channel::<String>(32); // 32 = buffer
tx.send("hello".to_string()).await.unwrap();
while let Some(msg) = rx.recv().await { ... }  // None when all senders dropped
```

**Cooperative vs preemptive:**

- Tokio tasks yield only at `.await` points - CPU-bound work can starve others
- For CPU-bound work in async context: `tokio::task::spawn_blocking(|| { ... })`
- OS threads (`std::thread`) are preemptive - kernel can pause them anywhere
- No BEAM-style preemptive green threads exist in Rust (by design - zero cost)

## Arc and Reference Counting

```rust
use std::sync::Arc;

let data = Arc::new(String::from("hello")); // ref_count: 1
let data2 = Arc::clone(&data);             // ref_count: 2, same heap data
drop(data);                                // ref_count: 1
drop(data2);                               // ref_count: 0, data freed
```

- `Arc` = thread-safe reference counting (like BEAM large binary ref counting)
- `Rc` = same but single-threaded only, cheaper
- Clone increments counter, drop decrements - freed when count hits 0
- Thread panic runs destructors - `Arc` count always decremented, no leak
- Only leak risk: reference cycles (`Arc<A>` points to `Arc<B>` points to `Arc<A>`)
- Fix cycles with `Weak<T>` - weak reference that doesn't increment count

**Channel fan-out options:**

```rust
// mpsc - one consumer, ownership moves (default)
let (tx, rx) = mpsc::channel(32);

// broadcast - clones to every subscriber (requires T: Clone)
let (tx, _) = broadcast::channel(32);
let rx1 = tx.subscribe();
let rx2 = tx.subscribe();

// watch - multiple consumers, only latest value (like an Agent)
let (tx, rx) = watch::channel("initial");
let rx2 = rx.clone();

// Arc in channel - fan-out without copying heap data
tx.send(Arc::new(msg)).await;  // consumers get Arc<T>, cheap clone
```

## Personal Notes (from questions during learning)

**Strings - two types, always confusing at first:**

- `"hello"` is `&str` - baked into the binary, you borrow it, zero cost
- `String::from("hello")` copies to heap, you own it, can mutate/grow
- Functions should take `&str` as params (accepts both), return `String` when producing
- `s.to_string()` and `String::from(s)` both convert `&str` -> `String`

**`&` is always a reference, everywhere, no exceptions:**

- `&Task` = borrowed Task, read-only
- `&mut Task` = borrowed Task, writable
- `Vec<&Task>` = vector of pointers to Tasks (tasks live elsewhere)
- `.` auto-derefs: `task.id` works even if `task` is `&&mut Task`

**Iterators and double-references:**

- `.iter()` yields `&T`, closure gets `&&T` - need `*x` for non-auto-deref ops like `%`
- `.iter_mut()` yields `&mut T`, `find` returns `Option<&mut T>` - just use `Some(task)`
- `.copied()` early converts `&T -> T` for Copy types, cleaner than manual deref
- `.collect()` not `.vec!()` - easy to forget

**Stack vs heap - the mental model:**

- Stack: fixed size, auto-freed when function returns, pointer just moves up
- Heap: dynamic size, freed when owner drops, `String`/`Vec` live here
- Can't return `&local_var` - local is on stack, reference outlives the frame

**Ownership gotchas:**

- Passing `String` to a function moves it - use `&String` or `&str` to borrow
- `let s2 = s1` moves heap types, copies stack types (i32, bool, char, f64)
- Mutable borrow ends at last use, not end of scope - lets you reborrow sooner

**No function overloading** - use enums + match, or traits instead

**println! quirk** - `{x}` works for variables, not field access:

```rust
println!("{}", user.name);   // correct
println!("{user.name}");     // COMPILE ERROR
```

**Elixir protocol vs behaviour (clarified):**

- Protocol: dispatch on data type at runtime (`String.Chars.to_string(x)` - x's type decides)
- Behaviour: module contract (`@behaviour GenServer` - module promises to implement callbacks)
- Rust traits cover both with one concept

## Elixir -> Rust Quick Map

| Elixir                     | Rust                                      |
| -------------------------- | ----------------------------------------- |
| `def foo(a, b)`            | `fn foo(a: T, b: T) -> T`                 |
| `\|>` pipe                 | `.method()` chaining                      |
| `case`                     | `match` (exhaustive)                      |
| `with`                     | `?` operator                              |
| `{:ok, v}` / `{:error, e}` | `Ok(v)` / `Err(e)`                        |
| `nil` / value              | `None` / `Some(value)`                    |
| `defprotocol` / `defimpl`  | `trait` / `impl Trait for Type`           |
| `@behaviour`               | `trait` as bound                          |
| `defstruct`                | `struct` + `impl`                         |
| `%Struct{}`                | `Struct { field: value }`                 |
| `alias Foo.Bar`            | `use foo::Bar`                            |
| `inspect(x)`               | `{x:?}` or `{x:#?}`                       |
| `mix test`                 | `cargo test`                              |
| `mix deps.get`             | `cargo build`                             |
| `GenServer.call`           | actor + `oneshot` channel                 |
| `:gen_tcp.recv(s, N)`      | `reader.read_exact(&mut buf).await`       |
| `<<n::big-32>>`            | `reader.read_u32().await` (BE by default) |

## Slice Patterns

```rust
match parts.as_slice() {                // Vec<T> doesn't match - need &[T]
    ["GET", key]        => ...,         // exact 2 elements
    ["SET", key, value] => ...,         // exact 3
    [cmd, ..]           => ...,         // first + rest (rest ignored)
    []                  => ...,
}
```

- `Vec<T>` is the owner; `&[T]` is the borrowed view. Pattern syntax works on `&[T]`.
- Three equivalent ways: `parts.as_slice()`, `&parts[..]`, `&*parts`.
- `[head, ..]` is the rest pattern, like Elixir's `[head | _tail]`.

## Match Guards

```rust
match result {
    Err(e) if e.kind() == io::ErrorKind::UnexpectedEof => Ok(None),
    Err(e) => Err(e),                   // catch-all must come AFTER guarded arm
    Ok(v)  => Ok(Some(v)),
}
```

- `if <bool>` after a pattern. Pattern + guard must both hold.
- Same as Elixir's `case x do {:error, e} when e.kind == :foo -> ...`.

## The actor pattern (toy GenServer)

```rust
use tokio::sync::{mpsc, oneshot};

pub struct Request {
    cmd: Command,
    reply: oneshot::Sender<Response>,    // single-use reply channel
}

#[derive(Clone)]                         // mpsc::Sender is Clone, so this is too
pub struct StoreHandle { tx: mpsc::Sender<Request> }

impl StoreHandle {
    pub fn spawn() -> Self {
        let (tx, mut rx) = mpsc::channel::<Request>(64);
        tokio::spawn(async move {
            let mut store = Store::new();    // owned by ONE task; no Arc/Mutex
            while let Some(Request { cmd, reply }) = rx.recv().await {
                let response = store.apply(cmd);
                let _ = reply.send(response);     // ignore: caller may have hung up
            }
        });
        Self { tx }
    }

    pub async fn call(&self, cmd: Command) -> Response {
        let (reply_tx, reply_rx) = oneshot::channel();
        if self.tx.send(Request { cmd, reply: reply_tx }).await.is_err() {
            return Response::Error("store unavailable".into());
        }
        reply_rx.await.unwrap_or_else(|_| Response::Error("store unavailable".into()))
    }
}
```

- `oneshot::Sender::send` is NOT async (unlike mpsc). Just `.send(value)`, no await.
- `mpsc::Sender::send` IS async, returns `Err` when receiver dropped (actor died).
- The actor task ends when all `Sender` clones drop -> `rx.recv()` returns `None`.
- One owner of state, no locks, message-passing -> Rust GenServer.

## Mutex vs actor (when to pick which)

|                              | `Arc<Mutex<T>>`                             | actor                                  |
| ---------------------------- | ------------------------------------------- | -------------------------------------- |
| Lines of code                | ~10                                         | ~40                                    |
| Fast (no contention)         | ✓                                           | slower (channel hop x2)                |
| Async work in operations     | awkward (don't `.await` while holding lock) | natural                                |
| Backpressure                 | implicit                                    | explicit (`mpsc::channel(N)`)          |
| Snapshot/clone state cleanly | blocks all readers                          | natural (handle in actor between msgs) |

- Mutex is the idiomatic default in Rust. Reach for actor when state has async work, or you want backpressure, or "atomic across multiple ops" matters.
- Critical: `tokio::sync::Mutex` (not `std::sync::Mutex`) inside async code. Holding `std::sync::Mutex` across `.await` can deadlock.

## TCP / async I/O

```rust
let (stream, addr) = listener.accept().await?;
let (read_half, mut write_half) = stream.into_split();   // owned halves, separate tasks
let mut reader = BufReader::new(read_half);              // BufReader consumes its arg
let mut writer = BufWriter::new(write_half);

// AsyncReadExt - tokio extension methods on AsyncRead:
reader.read_u8().await?;             // 1 byte -> u8
reader.read_u32().await?;            // 4 bytes BE -> u32 (LE variants too)
reader.read_exact(&mut buf).await?;  // fill buf or err

// AsyncWriteExt:
writer.write_all(&buf).await?;       // write all bytes
writer.flush().await?;               // FORCE buffered bytes to syscall
```

- `into_split` consumes stream, gives `OwnedReadHalf` + `OwnedWriteHalf` (independently movable).
- `tokio::io::split(&mut stream)` exists for borrowed split.
- BufReader/BufWriter wrap raw halves and add buffering. Always `flush` after a logical message.

## Generic async I/O bound

```rust
async fn read<R: AsyncRead + Unpin>(reader: &mut R) -> io::Result<Frame> { ... }
async fn write<W: AsyncWrite + Unpin>(&self, writer: &mut W) -> io::Result<()> { ... }
```

- `+ Unpin` is required ceremony for using `AsyncReadExt`/`AsyncWriteExt` extension methods.
- Almost everything is `Unpin` automatically (Vec, String, your structs, TcpStream, BufReader).
- Treat as "takes any async reader/writer." Pin/Unpin is an advanced topic; skip the deep dive.

## Length-prefixed binary framing

Wire format pattern:

```
[4 bytes BE: length][N bytes: payload]
```

```rust
async fn read_len_prefixed_bytes<R: AsyncRead + Unpin>(reader: &mut R) -> io::Result<Vec<u8>> {
    let len = reader.read_u32().await?;
    if len > MAX_FRAME_LEN { return Err(io::Error::new(io::ErrorKind::InvalidData, "too large")); }
    let mut buf = vec![0u8; len as usize];      // pre-fill with zeros - read_exact OVERWRITES
    reader.read_exact(&mut buf).await?;
    Ok(buf)
}

fn push_len_prefixed(buf: &mut Vec<u8>, payload: &[u8]) {
    buf.extend_from_slice(&(payload.len() as u32).to_be_bytes());
    buf.extend_from_slice(payload);
}
```

- `vec![0u8; n]` not `Vec::with_capacity(n)`: capacity ≠ length. `read_exact` looks at len.
- Always cap the length BEFORE allocating. A malicious peer can send `u32::MAX` and OOM you.
- Reuse the on-wire format as the on-disk format (M4). Don't invent two framings for the same data.

## Clean-EOF idiom

```rust
async fn read_opcode_or_eof<R: AsyncRead + Unpin>(reader: &mut R) -> io::Result<Option<u8>> {
    match reader.read_u8().await {
        Ok(byte) => Ok(Some(byte)),
        Err(e) if e.kind() == io::ErrorKind::UnexpectedEof => Ok(None),
        Err(e) => Err(e),
    }
}
```

- "Peer closed cleanly between frames" -> `Ok(None)`.
- "Peer closed mid-frame" or other I/O error -> `Err`.
- Caller distinguishes the two cases:
  ```rust
  match read_opcode_or_eof(reader).await? {
      Some(b) => /* parse rest of frame */,
      None    => break,                // shutdown gracefully
  }
  ```
- This pattern (`Result<Option<T>>`) appears anywhere "operation can fail OR data can be absent."

## Numeric casts

- No `From<u32> for usize` or vice versa - they're different sizes on different platforms.
- Use `as` for numeric casts: `len as usize`, `n as u32` (truncates if too big).
- Use `try_into()` for fallible numeric conversion (returns `Result`).
- `.into()` is for STRUCTURAL conversions (`&str -> String`, `T -> Box<T>`); NOT for numbers.

## io::Error construction

```rust
io::Error::new(io::ErrorKind::InvalidData, "frame too large")
io::Error::new(io::ErrorKind::InvalidData, format!("unknown opcode: {x:#x}"))
io::Error::new(io::ErrorKind::InvalidData, utf8_err)        // anything : Into<Box<dyn Error>>
```

- `InvalidData` = bytes parsed but didn't make sense (unknown opcode, bad UTF-8, length cap exceeded).
- `UnexpectedEof` = tokio gives you this; usually you don't construct it.
- Don't `panic!` for runtime conditions. `panic!` is for "this is unreachable / invariant violated."

## Bytes & strings - going both ways

| From            | To             | How                                                   |
| --------------- | -------------- | ----------------------------------------------------- |
| `&str`          | `&[u8]`        | `s.as_bytes()` (free)                                 |
| `String`        | `Vec<u8>`      | `s.into_bytes()` (no copy)                            |
| `Vec<u8>`       | `String`       | `String::from_utf8(v)` -> `Result`                    |
| `&[u8]`         | `&str` (lossy) | `String::from_utf8_lossy(b)` (good for printing)      |
| `u32`           | `[u8; 4]` BE   | `n.to_be_bytes()`                                     |
| `[u8; 4]` BE    | `u32`          | `u32::from_be_bytes(arr)` (or use tokio's `read_u32`) |
| byte literal    | `u8`           | `b'\n'`, `b'a'` (ASCII value as u8)                   |
| byte string lit | `&[u8; N]`     | `b"hello"`                                            |

## tokio::spawn - what to know

```rust
tokio::spawn(async move {
    // captures by move; runs concurrently; can outlive the caller
    // returns ()  ->  no `?`; use `match` + `break` for errors
});
```

- `move` is required when you capture owned values (most cases).
- If you're spawning per-iteration in a loop, clone shared resources FIRST and shadow:
  ```rust
  loop {
      let (stream, _) = listener.accept().await?;
      let store = store.clone();           // shadow - fresh handle per task
      tokio::spawn(async move { /* uses store */ });
  }
  ```
- Without the shadow-clone, `store` would be moved away on iteration 1 and gone for iteration 2.

## "Receipt" cost notes (Rust trade-offs that surprised me)

- `&str` is free; `String` is heap. Conversion is one allocation - that's the receipt.
- `Vec::with_capacity(n)` allocates space but length is still 0. Use `vec![0u8; n]` if you need length.
- Channel send is two allocs + an atomic. Mutex lock is one atomic CAS. Mutex is faster for tiny critical sections.
- Sync `flush` doesn't fsync - bytes are still in kernel buffers. Use `sync_all` for power-loss durability.
- Cloning `Arc` or `mpsc::Sender` is one atomic increment - cheap. Clone freely; don't avoid it.
