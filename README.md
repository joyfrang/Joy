# Joy. The Web Programming Framework and Language

CLI Tools

Joy uses lets as the command-line tool for running commands. E.g.:

$ lets run
...

$ lets test
...

$ lets make project joy-eg
...

Philosophy

Joy is a modern web framework with its programming language, focusing on being a simple, feature-rich, and performant replacement for JavaScript in web development. The Joy programming language is designed to work only with the framework, so from now on, we'll use the name "Joy" interchangeably for both of them.

Joy compiles to WASM and runs both in the browser and on the server, with solid CPU-bound performance, efficient memory use, small binary size, fast startup time, and good concurrency support. Joy was built from the ground up to address performance issues in the JS ecosystem with real architectural solutions.

Its syntax is designed to be readable, maintainable, and a genuine joy to write. Joy aims to help JavaScript and TypeScript developers rediscover the fun in web programming.

Memory Model

🧠 Joy Memory Model Proposal (Updated July 2025)

This section outlines how Joy manages memory and concurrency using safe automatic reference counting (RC), clone-by-default semantics, and branch blocks for async.

✅ Core Memory Model

Joy uses automatic reference counting, handled entirely by the compiler.

By default, values are cloned when assigned (deep copy).

Sharing must be explicitly marked with the share keyword.

Developers never manually write inc_ref, dec_ref, or clone.


User a = b         // Default: clone b into a (deep copy)
User a = share b   // Opt-in: share reference (RC applies)

📦 Assignment Behavior

Case	Behavior

Primitive types	Value copied (no RC)
Object types	Cloned by default
share keyword used	Shared via RC


The compiler automatically inserts reference count updates (inc_ref, dec_ref) when using share.

🌀 Cycles and Memory Leaks

Cyclic references are disallowed. The compiler rejects reference cycles.

Data must be modeled without cyclic ownership.


Alternatives:

Use IDs instead of object references

Flatten nested data

Model parent-child with one-way ownership


thing Node {
    u5 id
    share Node[] children  // ok
    u5? parent_id           // avoids backref
}

🚀 branch and Concurrency

Joy introduces branch blocks for async and concurrency. They provide safe, structured, and ergonomic patterns for non-blocking code:

User user = User("Matin")

// Clone-by-value into the task (default)
branch(User user) {
    print(user.name)
}

// Shared reference with atomic RC
branch(share User user) {
    user.name = "Nitam"
}

Key Async Paradigm Features

1. Structured Concurrency

Branches are tied to the parent lexical scope or task. Compiler ensures child branches cancel when the scope exits.

withScope(scope) {
  branch(User user) { /*...*/ }
  branch(share User cfg) { /*...*/ }
} // auto-cancels both

2. Cancellation & Deadlines

Built-in timeout and cancellation token support:

branch(User user, timeout(2s)) { /* auto-cancel */ }
branch(User user, CancelCtx ctx) { /* cancel via ctx */ }

3. Select Expression for Concurrency

A concise syntax to await multiple channels or timeouts:

select {
  case msg = chan1(): print(msg)
  case _ = chan2(): doOther()
  case _ = timeout(1s): print("timed out")
}

4. Error Propagation & Aggregation

Branches return Result<T, E> values that can be awaited:

Result<Data, Error> branch fetchData() { /*...*/ }
Result<Data, Error> result = await fetchData().untill(5s)
match result {
  Ok(data)  => render(data)
  Err(err)  => handleError(err)
}

5. Backpressure & Flow Control

Channels support policies like Wait, DropFirst, DropLast, and SuspendSender. Producers can await chan.available() or check chan.isFull().

6. Diagnostic Tooling & Tracing

$ lets inspect async

Outputs live tasks, stack traces, and channel states. Developers can trace task lifecycles for performance profiling.

branch Argument Modes

Syntax	Behavior

branch(Type x)	Cloned into task (default)
branch(share Type x)	Shared, compiler applies Atomic RC
branch(read share Type x)	Shared, read-only


This explicit model avoids implicit data races. Shared references must be opt-in with share.

🔒 Developer Simplicity

No rc<T>, arc<T>, clone() needed in user code.

Compiler manages safety.

Shared state requires opt-in via share.

Clone-by-default guarantees pure semantics unless overridden.


Syntax

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

thing User {
    Admin(int level)
    User(int id)
}

noth sth() {
    User user = User(69)
    User anotherUser = Admin(20)

    know user {
        User(id) => print($$"User with id $$id, $ is the US Currency")
        Admin(_) => {
            int level = user.level
            print("Let's hack their account!")
        }
    }
}

drop unsafe rand

i5 rnd() {
    -> unsafe rand.num(1, 10)
}

cont Eatable {
    bit eat(str reason)
}

impl User:Eatable {
    bit eat(str reason) {
        print($"Matin ate user for reason: $reason")
        -> 1
    }
}

pub noth entry() {
    i5 num = readLine()
    num = double(num!)
    triple(num!!)
}

int double(int! input) {
    -> input! * 2
}

noth triple(int!! input) {
    input!! *= 3
}

View rich() {
    bit rich = getStatus()
    -> <p>Wow I'm {rich}!</p>
}

#client
bit getStatus() {
    noname()
    -> 1
}

#server
noth noname() {
    _ = 10
}

noth some() {
    chan<bit> channel = (5, Wait)

    branch() {
        channel(1)
    }

    bit res = channel().until(10s)
}

noth entry() {
    bit[] bits = [1, 0, 1]
    bit[...] moreBits = [1, 0, 0, 1]
    bit[100] evenMore = [1]

    bit firstTrue = bits.whr(b == 1).frst()?

    for(bit b in bits) {
        // loop body
    }

    for((bit b, u5 i) in bits) {
        // loop with index
    }
}

Island counter() {
    chan<str> placeholder = server(() => {
        str text = db.query(/*...*/)
        -> text
    })

    -> <Wait for={str ph = placeholder().untill(5s)}
             fallback={<p>Wait!</p>}
             timeout={<p>Oops!</p>}>
        <strong>I got the {ph}!</strong>
       </Wait>
}

