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
        Viewer(_, str name)        => print($"Viewer: {name}")
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

```joy
// Define a contract (interface)
cont Eatable {
    bit eat(str reason)
    bit digest()
}

// Implement the contract for a specific type
impl User:Eatable {
    bit eat(str reason) {
        print($"User ate something for reason: {reason}")
        return true
    }
    
    bit digest() {
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

Contracts can be implemented by any `thing`, allowing for flexible and type-safe polymorphism without the complexity of traditional inheritance hierarchies.

### Generics with `mustbe`

Joy supports generic programming by constraining types to contracts using the `mustbe` keyword. This allows functions and `thing`s to operate on any type that fulfills a specific contract, ensuring both flexibility and type safety. This is the mechanism that powers built-in generic types like `maybe<T>` and `bomb<T, E>`.

```joy
// A function that accepts any type implementing the Eatable contract
noth feed(mustbe Eatable food) {
    food.eat("Because it's dinner time!")
}
```

### Static Dependency Injection with `settype`

Joy includes a special internal keyword, `settype`, to support advanced compile-time features like static dependency injection. It allows the compiler to assign a concrete type to a specially-designed typeless parameter, which is essential for configuring bundles without a runtime overhead.

This feature is used internally by the framework and is not part of the public API available to developers.

```joy
// Example from a bundle definition
Bundle load(str licenseKey) {
    // 'settype' assigns the 'Settings' type to the 'settings' parameter
    // The compiler resolves this based on a predefined usage
    return (settings: settype Settings(licenseKey: licenseKey, apiKey: "def"))
}
```

---

## Memory Model and Concurrency

### Memory Model

* **Automatic ARC**: The compiler inserts `inc_ref` and `dec_ref` calls; developers never manage them manually.
* **Clone‑by‑default**: Assignments perform deep copies unless marked `share`:

  ```joy
  User a = b          // deep clone
  User a = share b    // shared reference via ARC
  ```

* **No Cycles**: Reference cycles are disallowed; the compiler rejects cyclic ownership.

  * Use alternative patterns (IDs, one‑way ownership) to avoid cycles.

### Branch and Concurrency

Joy introduces `branch` blocks for async tasks with safe, structured concurrency:

```joy
User user = User("Matin")

// Default (clone-by-value) task
branch(bring User user) {
    print(user.name)
}

// Shared (atomic ARC) task
branch(bring share User user) {
    user.name = "Notmatin"
}
```

#### Key Features

1. **Structured Concurrency**: Branches are tied to their parent scope. Exiting the scope cancels all child tasks.

2. **Backpressure & Flow Control**: Buckets support policies (`Wait`, `DropFirst`, `DropLast`, `SuspendSender`):

   ```joy
   bucket<bit> b = (5, Wait)
   // Sender can check `b.isFull()` or await `b.available()`
   ```

---

## The Joy Web Framework

The framework extends Joy’s language principles to web development. Functions are server‑side by default for a secure‑by‑default architecture.

### Component Model

* **Layout**: Reusable wrappers for page structure.
* **View**: Static, server‑rendered components.
* **Island**: Interactive client‑side components with local state and events.

### Server RPC

Islands can call server closures via `server()`, passing in a pre-configured `bucket`:

```joy
bucket<str> codenameB = (3, DropLast)
server(codenameB) {
    return generateNewCodename()}
```

### Asynchronous UI

The built‑in `<Wait>` component declaratively consumes buckets with timeouts and fallbacks:

```joy
<Wait for={codenameB(timeout: 5s)}
      fallback={<p>Generating...</p>}
      timeout={<p>Timed out</p>}>
  {(str name) => <h2>Your new codename is: {name}</h2>}
</Wait>
```

---

## CLI Tools

Joy uses `lets` as the command‑line tool for running commands and managing projects:

```bash
# Create a new project named "joy-app"
$ lets make project joy-app

# Run the development server
echo "Starting dev server..."

$ lets run

# Build a production‑ready web application
$ lets build --release

# Run tests
echo "Running tests..."
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

        <Wait for={codenameB(timeout: 5s)}
              fallback={<p>Generating new codename...</p>}
              timeout={<p>Error: Request timed out.</p>}>
            {(str newName) => {
                codename = newName
                return <h2>Your new codename is: {codename}</h2>
            }}
        </Wait>
    </MainLayout>
}
```

## TODOs

There are numerous TODOs in the demos. However, these features need to be planned before they can make their way into the proposal:

* [x] Generic Types (proper implementation of `maybe` and `bomb` keywords depends on it)
* [ ] JSON-like collections (e.g., for passing type-safe configurations around)
* [ ] It would be cool to have a name for each [Epoch release](https://antfu.me/posts/epoch-semver?utm_source=joyfrang#:~:text=The%20format%20is,compatible%20bug%20fixes.)
* [x] Should there be implementations for `thing`s, like `user.add(...)`, or `user.remove(...)`
* [ ] How parameters should be passed in function calls?

**Proof of Concept:** You can view the Joy demo project, including example code and implementation details, [at the demo repository](https://github.com/joyfrang/Joy/tree/mom/Demo).q
