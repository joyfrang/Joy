# Joy: The Web Programming Framework and Language

> **Proposal Only:**
> This repository contains a proposal for **Joy**, a modern programming language and web framework. The project is not in active development and is currently in the feedback-gathering phase.
>
> **Note:** Joy is not scheduled for implementation in the near future. If you have thoughts or suggestions, contributions, and feedback are welcome!

---

## Philosophy

Joy is a complete ecosystem for building modern web applications and services, consisting of a new programming language and its integrated framework. Applications compile to WebAssembly (WASM) for the browser and to native binaries for the server, ensuring high performance. Joy is designed as a cohesive whole, providing a clear, productive, and reliable development experience out of the box.

Joy's design is guided by the "Joyful Programming" paradigm: a pragmatic approach focused on implementing whatever makes the most sense. This allows for flexibility, incorporating concepts from various programming paradigms—such as Functional Programming (FP) or Object-Oriented Programming (OOP)—without strict adherence to any single one.

---

## The Joy Language

### Data Modeling with `thing`

Joy's primary tool for data modeling is the `thing` keyword, which defines Algebraic Data Types (ADTs). Each variant can carry its own data, and exhaustive pattern matching via `defuse` ensures correctness.

```joy
thing User {
    Admin(u5 id, str name, u3 accessLevel)
    Viewer(u5 id, str name)
}

noth printUserDetails(User user) {
    defuse user {
        Admin(_, str name, u3 level) => print($"Admin: {name}, Level: {level}"),
        Viewer(_, str name)          => print($"Viewer: {name}")
    }
}
```

### Closures

Anonymous functions (closures) follow the same syntax as named functions but without a name. Variables from the parent scope must be explicitly brought in with `bring`:

```joy
str(bring str userName) {
    return $"Greetings, {userName}."
}
```

### Contracts and Implementations

Joy supports interface-like abstractions through **contracts** (`cont`) and their implementations (`impl`). Contracts define a set of function signatures that types must implement, enabling polymorphism and code reuse.

Instance methods take `self` as their first parameter, typed as the `thing` they belong to. They are called with dot syntax: `user.eat("hunger")`. The compiler recognizes the first `self` parameter and treats it as the receiver — no special keyword required.

```joy
// Define a contract (interface)
cont Eatable {
    bit eat(str reason)
    bit digest()
}

// Implement the contract for a specific type
impl User:Eatable {
    bit eat(User self, str reason) {
        print($"User ate something for reason: {reason}")
        return true
    }

    bit digest(User self) {
        print("User is digesting...")
        return true
    }
}

// Now User can be used wherever Eatable is expected
noth feedSomeone(Eatable hungry) {
    hungry.eat("hunger")
    hungry.digest()
}

noth example() {
    User user = Admin(id: 1, name: "Matin", accessLevel: 250)
    feedSomeone(user) // User implements Eatable, so this works
}
```

`impl` without a contract adds instance methods directly to the thing:

```joy
impl User {
    noth deactivate(User self) {
        // deactivate this user
    }
}

// Called as:
user.deactivate()
```

Contracts can be implemented by any `thing`, allowing for flexible and type-safe polymorphism without the complexity of traditional inheritance hierarchies.

### Generics with `mustbe`

Joy supports generic programming by constraining types to contracts using the `mustbe` keyword. This allows functions and `thing`s to operate on any type that fulfills a specific contract, ensuring both flexibility and type safety. This is the mechanism that powers built-in generic types like `maybe<T>`.

```joy
// A function that accepts any type implementing the Eatable contract
noth feed(mustbe Eatable food) {
    food.eat("Because it's dinner time!")
}
```

### Built-in `thing`s: `maybe<T>`

`maybe<T>` is a standard library `thing` representing a value that may or may not exist. It has two variants: `some(T value)` and `noth`. You `defuse` it like any other `thing`:

```joy
maybe<str> name = getNameFromCache()

defuse name {
    some(str n) => print(n)
    noth        => print("not found")
}
```

`maybe<T>` is not a special compiler construct — it is a regular `thing` that ships with Joy as a convention for nullable values.

### Validators

Validators are regular Joy functions prefixed with `v`. A parameter annotated with `->vName` means: run `vName` on the passed value before entering the function body. If the validator returns `noth`, the error propagates to the caller. The function body can assume the value is already valid.

```joy
maybe<str> vPermalink(str value) {
    if(value.length > 100) {
        return noth
    }
    return some(value)
}

// The caller of Page must handle the maybe<View> that results
// from the ->vPermalink annotation
View Page(str permalink->vPermalink) {
    return <p>Post at {permalink}</p>
}
```

Validators can also accept configuration parameters:

```joy
maybe<str> vMaxLength(str value, u5 max) {
    if(value.length > max) {
        return noth
    }
    return some(value)
}

// 1000 is passed as 'max'
noth createPost(str body->vMaxLength(1000)) {
    // body is guaranteed to be at most 1000 characters here
}
```

The validator name must match a function in scope. Unknown validator references are a compile error.

---

## Data Persistence and Serialization

### The `db` block

The `db` block declares how a `thing` maps to a database table. Only fields that need non-default behavior are listed — all other fields are persisted using their field name as the column name.

```joy
thing Post {
    Post(
        u5 id,
        str title,
        str body->vMaxLength(1000),
        str date,
        str author,
        str permalink->vPermalink
    )
}

db Post {
    table: "posts"       // override the default table name
    id: primaryKey, autoIncrement
    date: dbDate         // stored and queried as a date type
    permalink: unique
}
```

Valid `db` field options:

| Option | Meaning |
|---|---|
| `primaryKey` | Marks this field as the primary key |
| `autoIncrement` | Value is assigned by the database on insert |
| `unique` | Adds a unique constraint |
| `dbDate` | Stored as a date/timestamp type |
| `references(OtherThing.field)` | Foreign key relationship |
| `nullable` | Field may be null in the database |

### The `json` block

The `json` block declares serialization behavior. Only fields that deviate from defaults are listed. By default, all fields serialize using their field name as the JSON key.

```joy
json Post {
    id: ignore             // excluded from serialization entirely
    title: key("PostTitle") // serialized under a different key
}
```

Valid `json` field options:

| Option | Meaning |
|---|---|
| `ignore` | Field is excluded from serialization and deserialization |
| `key("name")` | Use a different key name in JSON |

---

## Memory Model and Concurrency

### Memory Model

* **Automatic ARC**: The compiler inserts `inc_ref` and `dec_ref` calls; developers never manage them manually.
* **Clone-by-default**: Assignments perform deep copies unless marked `share`:

  ```joy
  User a = b          // deep clone
  User a = share b    // shared reference via ARC
  ```

* **No Cycles**: Reference cycles are disallowed; the compiler rejects cyclic ownership.

  * Use alternative patterns (IDs, one-way ownership) to avoid cycles.

### Concurrent Blocks: `branch` and `server`

`branch` and `server` are not function calls — they are concurrent blocks, similar to how `for` and `if` are control flow constructs. They have their own scoping rules and the compiler handles their lifetime as part of structured concurrency.

#### `branch`

Spawns an async task tied to the current scope. Exiting the scope cancels all child branches. Variables from the parent scope must be explicitly captured with `bring`. By default, `bring` performs a deep clone. Use `bring share` for a shared ARC reference.

```joy
User user = User("Matin")

// Clone-by-value capture
branch(bring User user) {
    print(user.name)
}

// Shared (atomic ARC) capture
branch(bring share User user) {
    user.name = "Notmatin"
}
```

#### `server`

Spawns a server-side async task from within a client-side `Island`. Takes a `bucket` as its first argument, which it uses to send results back. Variables from the Island scope are captured with `bring`.

```joy
bucket<str> codenameB = (1, Wait)

server(codenameB, bring str userLocale) {
    str codename = generateCodename(userLocale)
    return codename
}
```

The `server` block initiates an RPC call stack on the server. Results are consumed via `<Wait>`.

#### Key Features

1. **Structured Concurrency**: Branches are tied to their parent scope. Exiting the scope cancels all child tasks.

2. **Backpressure & Flow Control**: Buckets support policies (`Wait`, `DropFirst`, `DropLast`, `SuspendSender`):

   ```joy
   bucket<bit> b = (5, Wait)
   // Sender can check b.isFull() or await b.available()
   ```

---

## Imports

Joy has three import namespaces, all using the same `bring` syntax:

```joy
bring joy:database/postgres          // Joy standard library
bring author:packageName/file        // approved third-party bundle
bring local:path/to/file             // project-local file (no .joy extension, path from project root)
```

---

## The Joy Web Framework

The framework extends Joy's language principles to web development. Functions are server-side by default for a secure-by-default architecture.

### Component Model

* **Layout**: Reusable wrappers for page structure.
* **View**: Static, server-rendered components.
* **Island**: Interactive client-side components with local state and events.

### Server RPC

Islands can call server closures via `server()`, passing in a pre-configured `bucket`:

```joy
bucket<str> codenameB = (3, DropLast)

server(codenameB) {
    return generateNewCodename()
}
```

### Asynchronous UI

The built-in `<Wait>` component declaratively consumes buckets with timeouts and fallbacks. The `as typename name` syntax binds the resolved value into the child scope:

```joy
<Wait for={codenameB(timeout: 5s) as str name}
      fallback={<p>Generating...</p>}
      timeout={<p>Timed out</p>}>
  <h2>Your new codename is: {name}</h2>
</Wait>
```

---

## CLI Tools

Joy uses `lets` as the command-line tool for running commands and managing projects:

```bash
# Create a new project named "joy-app"
$ lets make project joy-app

# Run the development server
$ lets run

# Build a production-ready web application
$ lets build --release

# Run tests
$ lets test
```

---

## A Complete Example: "Joyful Profile" App

```joy
// main.joy

thing User {
    Admin(u5 id, str name, u3 accessLevel)
    Viewer(u5 id, str name)
}

User getUserFromDb(u5 id) {
    return Admin(id: id, name: "Matin", accessLevel: 250)
}

str generateNewCodename() {
    return "Phoenix"
}

Layout MainLayout(Renderable children) {
    return <html>
        <head><title>Joyful Profile</title></head>
        <body>
            <div class="app-container">{children}</div>
        </body>
    </html>
}

#page("/")
View HomePage() {
    User user = getUserFromDb(id: 1)
    return <UserProfile user={user} />
}

Island UserProfile(User user) {
    str codename = "Nomad"
    bucket<str> codenameB = (1, Wait)

    server(codenameB) { return generateNewCodename() }

    return <MainLayout>
        {defuse user {
            Admin(_, str name, u3 level) => {
                <h1>Admin Panel: {name}</h1>
                <p>Access Level: {level}</p>
            },
            Viewer(_, str name) => {
                <h1>Welcome, {name}!</h1>
            }
        }}

        <hr />

        <Wait for={codenameB(timeout: 5s) as str newName}
              fallback={<p>Generating new codename...</p>}
              timeout={<p>Error: Request timed out.</p>}>
            <h2>Your new codename is: {newName}</h2>
        </Wait>
    </MainLayout>
}
```

## TODOs

* [x] Generic Types (proper implementation of `maybe` keyword depends on it)
* [x] Should there be implementations for `thing`s, like `user.add(...)`, or `user.remove(...)`
* [ ] JSON-like collections (e.g., for passing type-safe configurations around)
* [ ] It would be cool to have a name for each [Epoch release](https://antfu.me/posts/epoch-semver?utm_source=joyfrang#:~:text=The%20format%20is,compatible%20bug%20fixes.)
* [ ] How parameters should be passed in function calls?

**Proof of Concept:** You can view the Joy demo project, including example code and implementation details, [at the demo repository](https://github.com/joyfrang/Joy/tree/mom/Demo).
