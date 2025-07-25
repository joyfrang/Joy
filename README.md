# Joy. The Web Programming Framework and Language

## CLI Tools

Joy uses `lets` as the command-line tool for running commands. E.g.:
```bash
$ lets run
...

$ lets test
...

$ lets make project joy-eg
...
```

## Philosophy

Joy is a modern web framework with its programming language, focusing on being a simple, feature-rich, and performant replacement for JavaScript in web development. The Joy programming language is designed to work only with the framework, so from now on, we'll use the name "Joy" interchangeably for both of them.

Joy compiles to WASM and runs both in the browser and on the server, with decent CPU-bound performance, RAM usage, binary size, concurrency, and startup time. Generally, Joy was engineered from the ground up to address the JS ecosystem's massive performance issues, with an optimized infrastructure, instead of temporary band-aids.

Joy's syntax is also made to be a Joy to read, write, and maintain.

Joy was made to make JS/TS devs enjoy programming again.

## Syntax

```c
// Wapp is a Web App type
Wapp entry() {
	Wapp wapp = (port = 8080, tls = false) // Initialize with params
	-> wapp // Return
}

// A static component
View index() {
	-> <h1>Wow!</h1><Counter sth="85"/>
}

Island counter(int sth) {
	i5 num = 0
	-> <button @click=(num+=1)>Number is {num}</button>
}

/// A shell for the island Counter, which will be sent by the server
View _counter(int sth) {
	-> <p>Loading...</p>
}
```

```c
thing User {
	Admin(int level)
	User(int id)
}

noth sth() {
	User user = (69) // User is User, because the names are the same 
	User anotherUser = Admin(20)

	know user {
		User(id) => print($$"User with id $$id, $ is the US Currency")
		Admin(_) => {
			int level = user.level // Because we "know" this is an admin, it's fine
			print("Let's hack their account!")
		}
	}
}
```

```c
drop unsafe rand // Importing a non-approved package requires "unsafe"

// i5 is 2^5 = 32 bit intager
i5 rnd() {
	// Interacting with non-approved libs also needs unsafe, to prevent
	// "The NPM Effect"
	-> unsafe rand.num(1, 10)
}
```

```c
// Contracts are Interfaces
cont Eatable {
	bit eat(str why)
}

impl User:Eatable {
	bit eat(str reason) {
		print($"Matin ate user for reason: $reason")
		-> 1
	}
}
```

```c
// A public function
pub noth entry() {
	i5 num = readLine() // ReadLine from stdin
	num = double(num!) // Passing a refrence to the int
	triple(num!!) // Passing a mutable refrence to the int
}

int double(int! input) {
	-> input! * 2 /* Derefrencing has the same syntax */ 
}

noth triple(int!! input) {
	input!! *= 3
}
```

```c
// A function that returns a type View, named rich, with no arguments
View rich() {
	bit rich = getStatus()
	-> <p>Wow I'm {rich}!</p>
}

/// Runs on client, can use client-side APIs
#client
bit getStatus() {
	// Calls a server-side function
	noname()
	-> 1
}

/// A server function can use DB, internal networking, etc., but can't directly run code in the client for security reasons
#server
noth noname() {
	// You can't remove "_ =" and you have to explicitly ignore the return value
	_ = 10
}

```

```c
noth some() {
	chan<bit> channel = (5, Wait) // make a channel with type int, buffered simply, you can don't specify the capacity to make it an unbuffered channel. Should also take an optional third argument (or second if capacity is not set) for behaviour when the channel is full, use C# channel things: Wait, DropFirst, DropLast, DropNew
	// TOF means "Take Off". Async in Joy is like a plane
	tof() {
		// Do async work
		channel(1) // Add 1 to the channel
	}

	bit res = channel().until(10s/ms/us(micro)/ns/s/m/h) // empty call on a channel means wait until someone adds sth to this chan
}
```

```c
noth entry() {
	bit[] bits = [1, 0, 1] // Make a dynamically sized array
	bit[...] moreBits = [1, 0, 0, 1] // Equalivent to bit[4]
	bit[100] evenMore = [1] // Room for appending without resizing

	// We have Linq!
	bit firstTrue = bits.whr(b == 1).frst()? // ? means get the value from the result, I'm fine with crashing my program if it doesn't return Ok

	for(bit b in bits) {
		// Sth
	}

	// Ints and Unsigned Ints numbers are powers of two
	// u5 is a 2^5 = 32 bit unsigned intager
	for((bit b, u5 i) in bits) {
		// Do something with it
	}
}
```

```c
// ...

Island counter() {
	// An inline server anonymous server function. Returns a channel 
	// We don't pass the channel in, so the runtime will make the channel for us,
	// So it can't be used for anything other than this problem
	// The runtime makes the channel this way:
	// chan<str> = (1, Wait)
	// TODO: Also handle streaming responses, and also one-directional channels,
	// Use HTTP streaming under the hood
	chan<str> placeholder = server(() => {
		str text = db.query(/*...*/)
		-> text
	})

	// Do other stuff

	// str text = placeholder().untill(5s).unwrap()

	// or consume it within a Wait element

	// Wait is basically Suspense in React
	-> <Wait for={str ph = placeholder().untill(5s)}
			 fallback={<p>Wait!</p>}
			 timeout={<p>Oops!</p>}>
			 
		<strong>I got the {ph}!</strong>
	   </Wait>
}
```
