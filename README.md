# Joy: The Web Programming Framework and Language

## Philosophy

Joy is a complete ecosystem for building modern web applications and services, consisting of a new programming language and its integrated framework.

Applications compile to WebAssembly (WASM) for the browser and native binaries for the server, ensuring high performance. Joy is designed as a cohesive whole, providing a clear, productive, and reliable development experience out of the box.

## The Joy Language

### Data Modeling with `thing`

Joy's primary tool for data modeling is the `thing` keyword, which creates robust Algebraic Data Types (ADTs). This allows you to define an object that can exist in one of several distinct states.

The `know` expression provides exhaustive pattern matching. To ensure clarity and correctness, the types of all bound variables must be explicitly declared within the pattern.

```c
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
        Viewer(_, str name) => print($"Viewer: {name}")
    }
}
```

### Closures

Anonymous functions, or closures, follow the same declaration syntax as named functions, just without a name. The compiler identifies them by the context in which they are defined.

**Syntax:** `ReturnType(params...) { ... }`

Variables from the parent scope are not captured automatically; they must be explicitly brought into the closure's scope using the `bring` keyword within the parameter list.

```c
// A closure that brings `userName` into its scope and returns a `str`.
str(bring str userName) {
    return $"Greetings, {userName}."
}
```

### Concurrency and Memory

Memory management is automatic via ARC, with values being cloned by default and shared explicitly with the `share` keyword.

Asynchronous tasks are launched with `branch` blocks. These tasks are tied to their parent scope's lifecycle and are automatically cleaned up. For long-running background tasks that must outlive their creator (e.g., sending a post-registration email), an explicit `timeout` parameter can be provided to the `branch` itself.

## The Joy Web Framework

The framework extends the language's principles to web development. Functions are server-side by default, establishing a secure-by-default architecture.

*   **Component Model:**
    *   **`Layout`:** A reusable wrapper for page structure.
    *   **`View`:** A static, server-rendered component.
    *   **`Island`:** An interactive client-side component for state and events.
*   **Server RPC:** An `Island` can call back to the server using the `server()` function. This function takes a closure which it executes on the server. The type of the returned channel is inferred directly from the closure's explicit return type declaration.
    *   `server( str() { ... } )` returns a `chan<str>`.
    *   `server( User() { ... } )` returns a `chan<User>`.
*   **Asynchronous UI:** The `<Wait>` component declaratively handles async operations by consuming a channel. Timeouts are specified at the point of consumption.

## A Complete Example: The "Joyful Profile" App

This application demonstrates the seamless integration of Joy's language and framework features.

```c
// main.joy - Application Entry Point & Components

// --- Data Models and Server-Side Logic ---

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

// --- UI Components ---

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

    // The type of `codenameChannel` is `chan<str>` because the closure
    // passed to `server()` is explicitly defined to return a `str`.
    chan<str> codenameChannel = server( str() {
        return generateNewCodename()
    })

    // The component's render output.
    return <div>
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

        // We consume the channel, specifying a 5-second timeout.
        <Wait for={codenameChannel(timeout: 5s)}
              fallback={<p>Generating new codename...</p>}
              timeout={<p>Error: Request timed out.</p>}>

            // This closure runs on the client when data is received.
            {(str newName) => {
                codename = newName
                return <h2>Your new codename is: {codename}</h2>
            }}
        </Wait>
    </div>
}
```

## Tooling: The `lets` CLI

Joy includes `lets`, a command-line tool for the entire development workflow.

```bash
# Create a new project named "joy-app"
$ lets make project joy-app

# Run the development server
$ lets run

# Build a production-ready web application
$ lets build --release
```
