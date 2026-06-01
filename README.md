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

### Comments

Joy uses arrow-based comment syntax. The arrow points at the commented text.

```joy
This is a leading comment <~
noth something(Eatable food) {
    User u = getUser() ~> this is a trailing comment
}
```

For multi-line block comments, use `~>>` and `<<~`. Everything between them is commented:

```joy
~>>
This entire block is commented out.
noth oldImplementation() {
    ...
}
<<~
```

### Data Modeling with `thing`

Joy's primary tool for data modeling is the `thing` keyword, which defines Algebraic Data Types (ADTs). Each variant can carry its own data, and exhaustive pattern matching via `defuse` ensures correctness.

```joy
thing User {
    Admin(u5 id, str name, u3 accessLevel)
    Viewer(u5 id, str name)
}

noth printUserDetails(User user) {
    defuse(user) {
        Admin(_, str name, u3 level) => print($"Admin: {name}, Level: {level}"),
        Viewer(_, str name)          => print($"Viewer: {name}")
    }
}
```

A variant can reference an already-defined `thing` as one of its cases using `bring`:

```joy
thing DatabaseError {
    ConnectionFailed()
    Timeout()
}

thing AppError {
    NotFound(str resource)
    bring DatabaseError
}
```

When defusing an `AppError`, `ConnectionFailed` and `Timeout` are available as flat variants alongside `NotFound`.

### Validation

Joy has no magic annotation-based validator system. Validation is done two ways:

**Simple validation — plain function calls.** Write a function that returns `bomb` or `maybe` and call it explicitly. No ceremony, no new concepts:

```joy
bomb<str, PermalinkError> makePermalink(str value) {
    if(value.length > 100) { return TooLong(value) }
    if(!hasValidChars(value)) { return InvalidCharacter(value) }
    return fine(value)
}

bomb<Post, PostError> createPost(str permalink) {
    str validPermalink = rise makePermalink(permalink)
    ~> ... build the post
}
```

**Construction-time invariants — `validate` blocks.** A constructor can have a `validate(ErrorThing)` block that runs when the `thing` is constructed. Construction then returns `bomb<ThingType, ErrorThing>` instead of `ThingType` directly.

```joy
thing DateRangeError {
    InvalidRange(str start, str end)
    EmptyRange(str start)
}

thing DateRange {
    DateRange(str start, str end) validate(DateRangeError) {
        if(start >= end) { return InvalidRange(start, end) }
        if(start == end) { return EmptyRange(start) }
        return fine()
    },
    Forever() ~> no validate block — constructing this returns DateRange directly
}
```

Rules for `validate` blocks:
- `fine()` signals success — the compiler infers the constructed type from context
- Every variant you `return` must exist in the specified `ErrorThing` — compile error otherwise
- Not all variants of `ErrorThing` need to be used in a given block
- Multiple constructors in the same `thing` can share the same `ErrorThing`
- Constructors without a `validate` block return the type directly, not a `bomb`

### Closures

Anonymous functions (closures) follow the same syntax as named functions but without a name. Variables from the parent scope must be explicitly captured with `bring`:

```joy
str(bring str userName) {
    return $"Greetings, {userName}."
}
```

### Contracts and Implementations

Joy supports interface-like abstractions through **contracts** (`cont`) and their implementations (`impl`). Contracts define a set of function signatures that types must implement, enabling polymorphism and code reuse.

Instance methods take `self` as their first parameter, typed as the `thing` they belong to. They are called with dot syntax: `user.eat("hunger")`. The compiler recognizes the first `self` parameter and treats it as the receiver — no special keyword required.

```joy
cont Eatable {
    bit eat(str reason)
    bit digest()
}

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

noth feedSomeone(Eatable hungry) {
    hungry.eat("hunger")
    hungry.digest()
}

noth example() {
    User user = Admin(id: 1, name: "Matin", accessLevel: 250)
    feedSomeone(user) ~> User implements Eatable, so this works
}
```

`impl` without a contract adds instance methods directly to the thing:

```joy
impl User {
    noth deactivate(User self) {
        deactivate this user <~
    }
}

user.deactivate() ~> called as
```

### Generics

Joy supports generic data types through parameterized `thing`s.

```joy
thing Pair<A, B> {
    Pair(A first, B second)
}

thing Box<T> {
    Box(T value)
}
```

Generic types may also have implementations:

```joy
impl Box<T> {
    T get(Box<T> self) {
        return self.value
    }
}
```

Generic parameters may be used anywhere a normal type can be used:

```joy
Pair<str, u5> pair = (
    first: "hello",
    second: 42
)

maybe<Pair<str, u5>> result = some(pair)
```

### Contracts vs Generics

Contracts remain Joy's primary mechanism for polymorphism.

```joy
cont Eatable {
    bit eat(str reason)
}

noth feed(Eatable food) {
    food.eat("Because it's dinner time!")
}
```

Unlike some languages, generic parameters are not required merely to accept multiple types implementing the same contract.

```joy
noth feed(Eatable food) { ... }
```

is preferred over:

```joy
noth feed<Eatable T>(T food) { ... }
```

(which Joy does not currently support).

Generics are intended primarily for parameterizing data structures rather than functions.

### Built-in `thing`s: `maybe<T>` and `bomb<T, E>`

`maybe<T>` and `bomb<T, E>` are implemented as generic things and follow the same rules as user-defined generic types. They exist as shared vocabulary and convention. The type system is powerful enough to express them without any compiler magic; they ship with Joy so everyone agrees on the same pattern.

#### `maybe<T>`

Represents a value that may or may not exist. Variants: `some(T value)` and `noth`.

```joy
maybe<str> name = getNameFromCache()

defuse(name) {
    some(str n) => print(n)
    noth        => print("not found")
}
```

#### `bomb<T, E>`

Represents an operation that can either succeed or fail. The success variant is `fine(T value)`. The error variants come directly from whatever `thing` is passed as `E` — they appear flat alongside `fine` in a `defuse`, with no extra wrapping.

```joy
thing PostError {
    NotFound(str permalink)
    Unauthorized()
}

bomb<Post, PostError> getPost(str permalink) {
    ~> ...
}

defuse(getPost(permalink: "hello")) {
    fine(Post p)    => renderPost(p)
    NotFound(str s) => notFound()
    Unauthorized()  => forbidden()
}
```

### Error Handling Model

Joy's error handling is built around `bomb<T, E>` and `defuse`. There is no exception system and no hidden control flow.

**Returning errors:** A function signals failure by returning an error variant of its `bomb` type:

```joy
bomb<str, PostError> findTitle(str permalink) {
    maybe<Post> post = db.posts.first(p => p.permalink == permalink)
    defuse(post) {
        some(Post(str title, _)) => return fine(title)
        noth                     => return NotFound(permalink)
    }
}
```

**Handling errors:** Callers use `defuse` and must handle every variant. There is no way to silently ignore a `bomb`:

```joy
defuse(findTitle(permalink: "hello")) {
    fine(str title)  => print(title)
    NotFound(str s)  => print($"No post at {s}")
}
```

**Propagating errors:** If a function wants to pass a `bomb` up to its own caller without handling it, it uses `rise`:

```joy
bomb<str, PostError> getPostTitle(str permalink) {
    str title = rise findTitle(permalink) ~> if findTitle detonates, the error rises to our caller
    return fine(title.uppercase())
}
```

`rise` is the Joy equivalent of `?` in Rust or `try` in Zig. The error type of the current function must be compatible with the error type of the expression being risen.

---

## Configuration System

Joy provides a general `setup` block for attaching metadata and behavior to `thing`s. Rather than one-off language features, `setup` is an extensible protocol: the namespace before the `/` identifies the bundle providing the behavior, and the path after identifies the configuration type.

```joy
setup(joy:database/table for Post) { ... }      ~> built-in Joy database ORM
setup(joy:json/object for Post) { ... }         ~> built-in Joy JSON serialization
setup(someBundle:graphql/type for User) { ... } ~> a third-party bundle's config
```

Generic types may also be configured:

```joy
setup(someBundle:schema/type for Response<T>) {
    ...
}
```

### `setup(joy:database/table for ...)`

Declares how a `thing` maps to a database table. Only fields that need non-default behavior are listed — all other fields are persisted using their field name as the column name.

```joy
thing Post {
    Post(
        u5 id,
        str title,
        str body,
        str date,
        str author,
        str permalink
    ) validate(PostError) {
        if(body.length > 1000) { return BodyTooLong(body) }
        if(!isValidPermalink(permalink)) { return InvalidPermalink(permalink) }
        return fine()
    }
}

setup(joy:database/table for Post) {
    table: "posts"
    id: primaryKey, autoIncrement
    date: dbDate
    permalink: unique
}
```

Valid field options:

| Option | Meaning |
|---|---|
| `primaryKey` | Marks this field as the primary key |
| `autoIncrement` | Value is assigned by the database on insert |
| `unique` | Adds a unique constraint |
| `dbDate` | Stored as a date/timestamp type |
| `references(OtherThing.field)` | Foreign key relationship |
| `nullable` | Field may be null in the database |

### `setup(joy:json/object for ...)`

Declares JSON serialization behavior. Only fields that deviate from defaults are listed. By default, all fields serialize using their field name as the JSON key.

```joy
setup(joy:json/object for Post) {
    id: ignore
    title: key("PostTitle")
}
```

Valid field options:

| Option | Meaning |
|---|---|
| `ignore` | Field is excluded from serialization and deserialization |
| `key("name")` | Use a different key name in JSON |

---

## Queries

Because Joy owns the language, ORM, and framework, queries are a first-class language construct rather than a library API.

```joy
Post[] posts = query(db.posts) {
    where(author == "Matin")
    orderBy(date descending)
    take(10)
}
```

Queries are compiler-understood and may be translated directly into optimized database operations.

Queries also work on in-memory collections:

```
maybe<Post> post = query(posts) {
    where(author == "Matin")
    first()
}
```

Common operations:

```
where(...)
orderBy(...)
take(...)
skip(...)
first()
single()
count()
any()
all(...)
```

---

## Memory Model and Concurrency

### Memory Model

Joy's memory model is designed to be simple and correct by default, with zero manual memory management.

* **Automatic ARC**: The compiler inserts `inc_ref` and `dec_ref` calls; developers never manage them manually.
* **Clone-by-default with structural sharing**: Assignment performs a logical clone. Internally, the compiler implements this via persistent data structures and copy-on-write (COW), so unchanged parts of a structure are shared rather than copied. The result is value semantics without the cost of always copying everything.

  ```joy
  User a = b   ~> logical clone — a is independent from b
  ```

* **No Ownership Cycles**: Reference cycles in the ownership graph are disallowed; the compiler rejects them. This is rarely a practical constraint for web app data models, which are naturally trees or DAGs. When entities need to reference each other (e.g. a `Post` referencing its `User` author), the correct model is ID-based rather than pointer-based:

  ```joy
  thing User {
      User(u5 id, str name)
  }

  thing Post {
      Post(u5 id, str title, u5 authorId)   ~> not User author — just an ID
  }
  ```

  The compiler rejecting ownership cycles is a feature, not a limitation — it pushes models toward the correct, cache-friendly, serialization-friendly ID-based representation.

### Concurrent Blocks: `branch` and `server`

`branch` and `server` are not function calls — they are concurrent blocks, similar to how `for` and `if` are control flow constructs. The compiler manages their lifetime as part of structured concurrency.

#### `branch`

Spawns an async task tied to the current scope. Exiting the scope cancels all child branches. Variables from the parent scope must be explicitly captured with `bring`. `bring` performs a logical clone (COW).

```joy
User user = User("Matin")

branch(bring User user) {
    print(user.name)
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
   ```

---

## Cache

Joy provides first-class cache support using the same `setup` protocol as the database and JSON systems, keeping it backend-agnostic.

### Provider Configuration

```joy
bring joy:cache/redis

Cache configureCache() {
    return (cacheProvider: redis.provider(connectionString: "..."))
}
```

### Output Cache

Output caching is a property of a `View`, declared via `setup`. The compiler wraps the View transparently — no `cache.get()`/`cache.set()` noise in the View body.

```joy
View Page(str permalink) {
    maybe<Post> post = query(db.posts) {
        where(it.permalink == permalink)
        first()
    }
    ~> ...
}

setup(joy:cache/output for Page) {
    key: permalink          ~> cache is keyed by the route param
    ttl: 5m
    vary: ["Accept-Language"]
    tags: ["posts"]         ~> for tag-based invalidation
}
```

### KV Cache

KV stores are declared as named, typed stores via `setup`, then accessed through the `cache` global — mirroring how `db` works for database tables.

```joy
thing CachePolicy {
    Ttl(u5 seconds)
    SlidingTtl(u5 seconds)     ~> resets TTL on each access
    Eternal()                  ~> never evicts
}

setup(joy:cache/kv for SessionStore) {
    key: str
    value: str
    policy: Ttl(900)           ~> 15 min TTL
}
```

Usage:

```joy
maybe<str> session = cache.SessionStore.get("user:123")

defuse(session) {
    some(str s) => useSession(s)
    noth        => forbidden()
}

cache.SessionStore.set("user:123", token)
cache.SessionStore.delete("user:123")
```

The type safety comes for free — `SessionStore` only accepts `str` keys and `str` values, enforced at compile time.

### Tag-based Invalidation

```joy
cache.invalidate(tag: "posts")
cache.invalidate(store: SessionStore, key: "user:123")
```

Because the compiler knows the shape of every `setup(joy:cache/...)` block, it can statically validate that tags exist and warn on dead invalidation calls.

### Server Block Cache

The `server` block accepts an optional `cache:` argument, keyed automatically by the captured `bring` variables:

```joy
bucket<str> profile = ()

server(profile, cache: Ttl(60s), bring str userId) {
    str data = db.users.get(userId).name
    return data
}
```

---

## Imports

Joy has three import namespaces, all using the same `bring` syntax:

```joy
bring joy:database/postgres     ~> Joy standard library
bring author:packageName/file   ~> approved third-party bundle
bring local:path/to/file        ~> project-local file (no .joy extension, path from project root)
```

---

## The Joy Web Framework

The framework extends Joy's language principles to web development. Functions are server-side by default for a secure-by-default architecture.

### Component Model

* **Layout**: Reusable wrappers for page structure.
* **View**: Static, server-rendered components.
* **Island**: Interactive client-side components with local state and events.

### Server RPC

Islands can call server closures via `server`, passing in a pre-configured `bucket`:

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
$ lets make project joy-app   # Create a new project named "joy-app"
$ lets run                    # Run the development server
$ lets build --release        # Build a production-ready web application
$ lets test                   # Run tests
```

---

## A Complete Example: "Joyful Profile" App

```joy
main.joy <~

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
        {defuse(user) {
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

* [x] Generic Types (proper implementation of `maybe` and `bomb` depends on it)
* [x] Should there be implementations for `thing`s, like `user.add(...)`, or `user.remove(...)`
* [x] Error handling model (`bomb`, `defuse`, `rise`)
* [x] Validation system (`validate` blocks on constructors, plain function calls)
* [x] Cache system (`setup(joy:cache/output ...)`, `setup(joy:cache/kv ...)`, `server` block cache)
* [ ] JSON-like collections (e.g., for passing type-safe configurations around)
* [ ] It would be cool to have a name for each [Epoch release](https://antfu.me/posts/epoch-semver?utm_source=joyfrang#:~:text=The%20format%20is,compatible%20bug%20fixes.)
* [ ] How parameters should be passed in function calls?

**Proof of Concept:** You can view the Joy demo project, including example code and implementation details, [at the demo repository](https://github.com/joyfrang/Joy/tree/mom/Demo).