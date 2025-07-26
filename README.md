Joy: The Web Programming Framework and Language

Philosophy

Joy is a complete ecosystem for building modern web applications and services, consisting of a new programming language and its integrated framework.

Applications compile to WebAssembly (WASM) for the browser and native binaries for the server, ensuring high performance. Joy is designed as a cohesive whole, providing a clear, productive, and reliable development experience out of the box.


---

CLI Tools

Joy uses lets as the command‑line tool for running commands and managing projects:

# Create a new project named "joy-app"
$ lets make project joy-app

# Run the development server
echo "Starting dev server..."
$ lets run

# Build a production‑ready web application
$ lets build --release

# Run tests
$ lets test


---

The Joy Language

Data Modeling with thing

Joy's primary tool for data modeling is the thing keyword, which creates robust Algebraic Data Types (ADTs). This allows you to define an object that can exist in one of several distinct states.

The know expression provides exhaustive pattern matching. To ensure clarity and correctness, the types of all bound variables must be explicitly declared within the pattern.

// A User can be one of two variants: Admin or Viewer.
thing User {
    Admin(u5 id, str name, u3 accessLevel)
    Viewer(u5 id, str name)
}

// In this `know` block, the types for `name` and `level` are explicitly
// declared, preventing ambiguity and errors.
noth printUserDetails(User user) {
    know user {
        Admin(_, str name, u3 level) => print($"Admin: {name}, Level: {level}"),
        Viewer(_, str name)        => print($"Viewer: {name}")
    }
}

Closures

Anonymous functions (closures) follow the same declaration syntax as named functions, just without a name. The compiler identifies them by the context in which they are defined.

Syntax: ReturnType(params...) { ... }

Variables from the parent scope are not captured automatically; they must be explicitly brought into the closure's scope using the bring keyword within the parameter list.

// A closure that brings `userName` into its scope and returns a `str`.
str(bring str userName) {
    return $"Greetings, {userName}."
}


---

Memory Model and Concurrency

Joy manages memory and concurrency using safe Automatic Reference Counting (ARC), clone‑by‑default semantics, and structured concurrency constructs.

Memory Model

Automatic ARC: The compiler inserts inc_ref and dec_ref calls; developers never manage them manually.

Clone‑by‑default: Assignments perform deep copies unless marked share:

User a = b          // deep clone
User a = share b    // shared reference via ARC

No Cycles: Reference cycles are disallowed; the compiler rejects cyclic ownership.

Use alternative patterns (IDs, one‑way ownership) to avoid cycles.



Branch and Concurrency

branch takes a noth() returning closure or function reference as an argument, improving composability and alignment with other constructs:

// External function
noth doAsync() {
    print("running async")
}

// Using an external function
branch(doAsync)

// Inline closure for brevity
branch(noth() {
    print("hi")
})

Key Features:

1. Structured Concurrency: Branches are tied to their parent scope. Exiting the scope cancels all child tasks.

withScope(scope) {
  branch(doCleanup)
} // auto-cancels


2. Cancellation & Deadlines: Built‑in support for timeouts and cancellation tokens:

branch(timeout: 2s, doTimeoutTask)


3. Select Expression: Await multiple buckets or timeouts concisely:

select {
  case msg = bucket1():     print(msg)
  case _   = bucket2():     doOther()
  case _   = timeout(1s):   print("timed out")
}


4. Error Propagation: Branch returns Result<T, E>, which can be awaited and matched.




---

Buckets and Server RPC

Buckets are now the primary async primitive. They encapsulate queueing, backpressure policy, and producer logic.

Defining a Bucket

// Create a bucket with capacity 3 and DropLast policy, attaching a producer
bucket<str> codenameB = bucket(3, DropLast, generateNewCodename)

Attaching Producers

Function reference:

bucket<str> b = bucket(1, Wait, generateCodename)

Inline closure:

bucket<str> b = bucket(2, DropFirst, str() { return "hello" })


Server RPC

Islands trigger server-side producers by the context they run in; no separate server() call is needed. Producers defined on the server execute accordingly.


---

The Joy Web Framework

The framework extends Joy’s language principles to web development. Functions are server‑side by default for a secure‑by‑default architecture.

Component Model

Layout: Reusable wrappers for page structure.

View: Static, server‑rendered components.

Island: Interactive client‑side components with local state and events.


Asynchronous UI

The built‑in <Wait> component declaratively consumes buckets with timeouts and fallbacks:

<Wait for={codenameB(timeout: 5s)}
      fallback={<p>Generating...</p>}
      timeout={<p>Timed out</p>}>
  {(str name) => <h2>Your new codename is: {name}</h2>}
</Wait>


---

A Complete Example: "Joyful Profile" App

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
    bucket<str> codenameB = bucket(1, Wait, generateNewCodename)

    return <MainLayout>
        {know user {
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


---

Tooling: the lets CLI

# Create a new project
$ lets make project joy-app

# Run development server
$ lets run

# Inspect async tasks
$ lets inspect async

# Build production release
$ lets build --release

# Run tests
$ lets test

