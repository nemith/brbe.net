---
title: "Don't (always) Panic (from libraries)"
date: 2022-08-01
tags: ["golang", "panic", "error-handling"]
---

I recently had a bit of a discussion on Mastodon about panics and their purpose in Go and have heard some interesting "rules" around panics that essentially comes down to:

> Never panic in libraries unless you follow the informal agreement of prefixing your functions with `Must`.

I think this is misguided rule and comes down to a misunderstanding of what `painc()` is in Go and confusing `panic/recover` with exceptions and error handing (usually coming from other languages that use exception).  You absolutely  ***should*** in some cases panic from libraries (with or without the `Must` prefix).  It is better to learn on ***why*** you would want to panic instead of having incomplete or misguided rules.

Also in a lot of cases just appending `Must` to a function that could error could also be just as wrong.

## `panic` is for Runtime Invariants

`panic()` in Go was never meant to handle errors.  If you are thinking that `panic()` (and it accompanied `recover`) is just some sort of `try/except` that is discouraged, then you are fundamentally thinking of the purpose of `panic()` incorrectly.

I think this is a bad comparison and a better comparison of `panic()` is with `assert`.  Assertions are a feature in a lot of languages ***assert*** runtime invariants.  Some languages perform `assert`s only when compiled in a  debug mode, but the idea is the same; you have an invariant in your code that cannot be checked by the compiler at compile time, so you add a `assert` (or in the case of Go a `panic()`) to validate (or assert) that invariant doesn't happen.

Another way to state this is to use `panic` only in conditions that *should never happen with proper use of the code*.  In an ideal world these conditions would be caught by the compiler, but that is not always the case.

On the other hand any thing that can error out due to environmental issues (network, disk, memory), external input (user inputs, rpc calls, etc), or anything else that could change at runtime should NEVER `panic`.  Use normal error handling (i.e returning errors as values from methods/functions and handling them at the call site)

However if it is a *clear* programming error or something that should be considered static in your code, then a panic is a good way to inform the programmer that something when wrong.

```go
// BAD: panic used for error handling and to avoid passing `errors` around.  The use here is essentially trying to replicate exceptions from other languages.
//
// Note: this would be bad even if you called this `MustOpenDB`.  Opening a database could fail by a lot of conditions that are not programmer error so using panic here doesn't indicate an invariant or something the programmer did "wrong" but instead just an error.
func OpenDB(dsn string) *DB {
    db, err := sql.Open(dsn)
    if err != nil {
        panic(err)
    }
    return db
}
```

```go
// Better:
func (b *Buffer) Grow(n int) {
    if n < 0 {
        panic("Grow called with negative size")
    }
    //...
}
```

## Why not just ALWAYS return errors

There may be a counter argument that even if you can `panic` why not making all invariants and errors just be normal errors handling?   For example in the above example why doesn't grow just return an `error` when given a negative number?

These invariants should caught and fixed as in the development cycle.  They, by default, provide a stack trace giving you an idea how the invariant was triggered with additional information for the programmer.  Making them regular errors would potentially hide these errors and cause unwanted behavior (like retrying something that will never pass).

For the few cases where you need to handle an error (i.e in a `http.Handler`) there is `recover` to allow you to ensure that one request doesn't affect other requests however this doesn't change the reason for `panic`.

## Implicit panics

There are a number of places where your code will panic without an explicit call to panic.  These are enforced by the compiler, however they serve the exact same purpose of explicit `panic` call.

In all these implicit cases the idea is the same as explicit `panic` statement.   There are invariants that is being enforced at runtime and deemed that they should be a panic to inform *the programmer** that something is wrong.  We can all agree that explicit error handling on these examples would be silly.  Why is library code any different?

#### Divide by zero

When dividing by zero go implicitly adds a panic if the divisor is `0` (Note: that if you use the literal `0` then it becomes a compiler error).

```go
a := 0
_ = 7/0
```

```plain
panic: runtime error: integer divide by zero

goroutine 1 [running]:
main.main()
	/tmp/sandbox3831422329/prog.go:7 +0x11

Program exited.
```

#### Indexing out of range

When you index a slice or array outside of the length Go will implicitly panic:

```go
a := []int{1, 2, 3}
_ = a[3]
```

```plain
  panic: runtime error: index out of range [3] with length 3

goroutine 1 [running]:
main.main()
	/tmp/sandbox497075684/prog.go:10 +0x5f
```

## Examples in the standard library

There are plenty of examples in the standard library of functions that ***can*** panic, but they are all to assert invariants that indicate bad use **by the programmer**.


* [io.CopyBuffer](https://pkg.go.dev/io#CopyBuffer) will panic if you give pass a zero length buffer. (However if you pass a `nil` buffer it creates one replicating the behavior of `io.Copy`).
* [bufio.Scanner.Buffer](https://pkg.go.dev/bufio#Scanner.Buffer) will panic if it is called after scanning has started (as setting the buffer makes no sense)
* [bytes.Repeat](https://pkg.go.dev/bytes@go1.20.6#Repeat) will panic if you give it a negative count or if the resulting length would overflow an integer.
* [bytes.Buffer.Grow](https://pkg.go.dev/bytes@go1.20.6#Buffer.Grow) will panic on a negative capacity or if the buffer cannot grow as it is too large (this may be better off an error...)
* [flag.*](https://pkg.go.dev/flag@go1.20.6) Defining a flag with the same name more than once in a given [`FlagSet`](https://pkg.go.dev/flag@go1.20.6#FlagSet) will cause a panic.
* Countless others.  The idea isn't for this list to be exhaustive but to illustrate valid uses of `panic` in libraries.


These are used sparingly but if you analyze the use of them they should all be programmer misuse.  Also they are all documented on the function/method.

It has been suggested that use of panic in the stdlib is wrong and I disagree.  I consider most of these proper use of `panic` as the fit the description above about being a programming error and not something that needs to be handled through regular error handling.

### The `Musts`

These are probably the most obvious example that people could think of for use of `panic` in the standard library. Examples include:

* [`regex.MustCompile`](https://pkg.go.dev/regexp#MustCompile)  and [`regexp.MustCompilePOSIX`](https://pkg.go.dev/regexp#MustCompilePOSIX)
* [`text/template.Must`](https://pkg.go.dev/text/template#Must) and [`html/template.Must`](https://pkg.go.dev/html/template#Must)
* [`net/netip.MustParseAddr`](https://pkg.go.dev/net/netip#MustParseAddr), [`net/netip.MustParseAddrPort`](https://pkg.go.dev/net/netip#MustParseAddrPort), and [`net/netip.MustParsePrefix`](https://pkg.go.dev/net/netip#MustParsePrefix)

The `Must` prefix here all functions that also have an version that also returns a error.  (For example `regex.MustCompile` is the panicy version of `regex.Compile` which can return an error).  But there is one thing with all these examples that you should keep in mind.

> **In all versions it is very likely that the input to these functions may be static!**

What does that mean?  Well it means that you should only use the `Must` versions of this functions when you, the programmer calling the function, can guarantee that the input ***SHOULD*** never fail and the way you do that is the input is hard coded into the binary itself.

You should ***NEVER*** use the `Must` versions of these functions with dyanmic/user input (i.e input from a config file, or from a API call, or even reading from a filesystem). In all these cases you are better off using the normal version (without the `Must` prefix) and handling your errors gracefully.

Where the `Must` versions come in handy is trying to define static content:

```go

// GOOD: Precompiling a Go regexp where the input is staticly defined.
var ipAddressRE = regexp.MustCompile(`\b(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?(\.|$)){4}\b`)

// GOOD: Compiling a template defined statically
var template.Must(var t = template.Must(template.New("name").Parse("Hello {{.First}} {{.Last}}"))

// GOOD: Parsing an statically defined IP address
var localhost = netip.MustParseAddr("127.0.0.1")

// BAD: Parsing IP address from a external source.
func ParseConfig(data []byte) (*Config, error) {
    var cfg struct {
        RemoteIP string `toml:"remote_ip"`
    }
    if err := toml.Unmarshal(&cfg, data); err != nil {
        return nil, err
    }

    // BAD: IP address is not static to the binary and could be wrong!
    // Should use `netip.ParseAddr()` instead and handle the error (return it, use a default value, log, whatever).
    remoteIP := netip.MustParseAddr(cfg.RemoteIP)
}
```

### Quick note on `json/encoding`

So not everything in the stdlib is "hunky dory".  There are some cases where `panic` is used incorrectly.  A great example of this is `encoding/json` package which does use `panic` and `recovery` as a way to avoid passing `errors` in return values.

 This was done to clean up the code somewhat, but I would argue that this is not something that should be emulated in your own code.  That being said it is not exposed to callers of the library and could (and should) be cleaned up without an API change.  The panic's being thrown are locked down to the `jsonError` type which means normal `panics` in the library would still bubble beyond the API (correctly!)

 https://github.com/golang/go/blob/master/src/encoding/json/encode.go#L280

## The rules on `panic`

So `panic` can be very useful and should be a tool that is available to you while you write your libraries.  Here are somethings to keep in mind if you looking at using `panic` in your Go libraries.

* Use `panic` sparingly and only when it is *always* a programmer error / invariant that cannot be caught by the compiler or linter.
* Only use `Must` prefixed alternatives for functions that could error only when you regularly expect static inputs.  Don't assume you always need a `Must*` version.
* ***NEVER*** user panic/recovery to avoid returning an `error` in normal error handling conditions.

![Green blob saying "Don't Panic" under it from Hitchhikers Guide to the Galaxy](dontpanic.jpg)

## Resources

* [Effective Go - Panic](https://go.dev/doc/effective_go#panic)
* [To panic! or Not to panic!]https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html(https://docrust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) Rust book.
* [Using unwrap() in Rust is Okay](https://blog.burntsushi.net/unwrap/) by Andrew Gallant (burntsushi)
