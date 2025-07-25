# Joy. The Web Programming Framework and Language

## Philosophy

Joy is a modern web development framework paired with its own programming language. It's designed to replace JavaScript and TypeScript with a simpler, more efficient, and developer-friendly alternative.

Joy applications compile to WebAssembly (WASM) for the web and can also run as native binaries on servers via LLVM. This results in efficient performance across CPU usage, memory consumption, binary size, concurrency, and startup time.

Built from scratch to address JavaScript’s pain points, Joy offers a clean and enjoyable developer experience.

## Memory Model

Joy uses automatic reference counting (RC) to manage memory safely. The compiler handles all details, so developers don’t need to worry about manual memory management.

* By default, values are cloned on assignment.
* Shared access requires the `share` keyword.
* No manual `inc_ref`/`dec_ref` or `clone()` calls in user code.

```c
User a = b         // Deep copy clone
User a = share b   // Shared reference via RC
```

### Assignment Rules

| Type             | Behavior          |
| ---------------- | ----------------- |
| Primitive values | Value is copied   |
| Objects          | Cloned by default |
| `share` keyword  | Shared via RC     |

The compiler inserts reference count updates as needed.

### Preventing Cycles

Circular RC is disallowed. The compiler rejects reference cycles. Use IDs, flattened structures, or one-way ownership instead:

```c
thing Node {
    u5 id
    share Node[] children
    u5 parent_id   // avoids backref
}
```

## Async Concurrency: `branch` Blocks

`branch` creates structured, safe async tasks:

```c
User user = ("Matin")

// Cloned copy
branch(User user) {
    print(user.name)
}

// Shared reference
branch(share User user) {
    user.name = "Notmatin"
}
```

### Features

1. Structured Concurrency: Children cancel when their parent scope exits.
2. Cancellation & Timeouts: Built-in token and timeout support.
3. Select Expression: Wait on multiple channels or timers.
4. Result Propagation: `branch` can return `Result<T, E>`.
5. Flow Control: Channels support `Wait`, `DropFirst`, `DropLast`, `SuspendSender`.
6. Diagnostics: `$ lets inspect async` shows live tasks and traces.

#### Argument Modes

| Syntax                      | Behavior                 |
| --------------------------- | ------------------------ |
| `branch(Type x)`            | By-value clone (default) |
| `branch(share Type x)`      | Shared via atomic RC     |
| `branch(read share Type x)` | Shared, read-only        |

### Example: Web Build

```bash
$ lets build --target web --release
```

Produces a `.wasm` module plus JS loader. Integrates with Webpack/Vite via the Joy plugin.

## CLI Tools

Joy includes a command-line interface (CLI) tool called `lets`. This tool is used to manage and run your projects. For example:

```bash
$ lets run          # Start the app
$ lets test         # Run all tests
$ lets make project joy-eg  # Create a new project
```

## Syntax Examples

```c
Wapp entry() {
    Wapp wapp = (8080, false)
    -> wapp
}

View index() {
    -> <h1>Wow!</h1><Counter sth="85"/>
}

Island counter(int sth) {
    i5 num = 0
    -> <button @click=(num+=1)>Number is {num}</button>
}
```
