---
title: "(Bad?) Experiments with Go generics"
date: 2020-06-21
tags: ["golang"]
draft: true
---

The Go team just posted about *[The Next Step for Generics](https://blog.golang.org/generics-next-step)* whiches shows the current progess for Generics in go which includes an updated draft which removed contracts completely and replaces them just with interfaces a long with a lot of different examples.  

In addition the go team created a translation tool that will take the experimental generic code and convert it to concreate types via monophorphization so that you can play with and experiment with the feature.  This tool has also been exposed at [go2goplay.golang.org](https://go2goplay.golang.org) making it really easy to play around with the new syntax.

So I starting thinking of ways generics could be used mostly based off of what is found in other langages and I implemented a few of them here.  Please note that these are probably **horrible ideas** and probably shouldn't be taken as what *should* be done.

## Rust-style `Result` object
With Go you can return multiple values from a function and errors must be passed between functions as one of those values (typically the last one) as there are no exceptions that bubble up behind the scenes.   

```go
func doThing(fail bool) (int, error) {
    if fail {
        // we explitly return the error here
        return 0, errors.New("failed to do thing")
    }
    return 42, 0
}

func main() {
    x, err := doThing(true)
    if err != nil {
        panic(err)
    }
    fmt.Println(x)
}
```

Rust is similar in making errors explit return values, however instead of returning multiple errors an [`enum`](https://doc.rust-lang.org/rust-by-example/custom_types/enum.html) called [`Result`](https://doc.rust-lang.org/std/result/index.html) with generic type parameters is passed back with a value of either a `Ok` value or and `Err` value (never both!). Both `Ok` and `Err` can have any type. 

```rust
// Define both the Ok type and the Err type.  For simplicty i am returning a
// static string slice
fn do_thing(fail: bool) Result<i32, &'static str> {
    if fail {
        return Err("failed to do thing")
    }
    return Ok(42)
}

fn main() {
    // unwrap gets the Result value for Ok or panics if Result is an Err
    println!(do_thing.unwrap())
}
```
In this case `unwrap` will panic if the result enum is the Err value, but there are other ways of checking the error from Rust.

Although Go doesn't not support a `enum` type we can use a type paramaterized struct which can contain the return type and/or an `error` value.

```go
// Result can hold _both_ a Err and Result but only one is used.
type Result(type T) struct {
	Err   error
	Value T
}

// Ok will create a new result with a value of any type and no error.
func Ok(type T)(value T) Result(T) {
	return Result(T){ Value: value }
}	
    
// Err will create a new result with only an error
func Err(type T)(err error) Result(T) {
	return Result(T){ Err: err }
}

// Expect will panic if there is an error, else it returns the underlying
// type.
func (r Result(T)) Expect(msg string) T {
	if r.Err != nil {
		panic(msg)
	}
	return r.Value
}

// Unwrap will just return the value and ingore the error.  This is unlike
// rust which will panic here.  If there is an error the zero value for T
// is returned instead.
func (r Result(T)) Unwrap() T {
	return r.Value
}

// IsOk returns true if there is no error
func (r Result(T)) IsOk() bool {
	return r.Err == nil 
}

// IsErr returns true if there is an error
func (r Result(T)) IsErr() bool {
	return r.Err != nil 
}

// ErrIs is a wrapper for errors.Is
func (r Result(T)) ErrIs(target error) bool {
	return errors.Is(r.Err, target)
}

// ErrAs is a wrapper for errors.As
func (r Result(T)) ErrAs(target interface{}) bool {
	return errors.As(r.Err, target)
}

var ErrBadValue = errors.New("bad value")

func myFunc(input string) Result(int) {
	if input == "bad" {
        // note we have to explitly state the return type here as Go cannot 
        // (yet) infer the type from the return type.
		return Err(int)(ErrBadValue)
	}

    // type inference works ok here as we are passing an int.
	return Ok(42)
}

func main() {
    // panic if there is an error
	fmt.Println(myFunc("good").Expect("value is bad"))
//	fmt.Println(myFunc("bad").Expect("value is bad"))
	
	res := myFunc("wut")
    // replacement for if err != nil
    if res.IsErr() {
		panic("we had any error")
	}
	
	res = myFunc("hmm")
	if res.ErrIs(ErrBadValue) {
		panic("we got a bad value")		
	}
	
	res = myFunc("bad")
    // ignore the error
    fmt.Println(res.Unwrap())
}
```

So with generic Go we can create a `Result` like object, but besides having a way to panic on error with  `Expect` we actually don't gain much as there still ins't a way to easily return an Result error back up to the higher called in the stack (like `?` in Rust).  

So this probably isn't the best idea, but it is possible.

[Play with this in the go2go playground](https://go2goplay.golang.org/p/_UYH9bIKSGu)

## Optional objects
Go only supports `nil` on certain values (maps, slices, channels, interfaces, and pointers) which can be used to denote the "lack of value".  Other types like `int` or `string` cannot be null and instead are instantiated as their *default* or *zero* value.  Like `0` for `int` and `""` (empty string) for `string`.

Often times the zero value can be used to see if a value is set or not (i.e: `if myString == ""`) however
there are times where the zero value is valid but you still also want to check the existence for a value as well.

There are really two way in Go to handle this situation.  One way is to create a pointer to the value.  Since pointers are *nillable* you can check the value for nil.  The downside is if you forget to check for nil you will panic at runtime.  The other way is to wrap the value in a struct with a bool to denote it exists or not

```golang
// using a pointer to the value
type Foo struct {
    quuz *int
}

func doThingFoo(foo Foo) {
    if foo.quuz == nil {
        fmt.Println("value not set")
        return 
    }
    // Note we have to deference the value back to an int here for functions
    // that expect int and not *int. Also will panic if we forgot to nill check
    // first
    doThingWithValue(*foo.quuz)
}

// We need a object to hold the value and it's current state
func OptionalInt struct {
    Value int
    Valid bool
}

func Bar struct {
    quux OptionalInt
}

func doThingBar(bar Bar) {
    if !foo.quux.Valid {
        fmt.Println("value not set")
        return 
    }
    // Here we can get the Value directly from the struct.  We have no chance
    // of panicing here as even with quux being invalid this will still be the
    // zero value or `0` as it is an int.
    doThingWithValue(foo.quux.Value)
}
```
You can find examples of both in the standard library.  `database/sql` has functions for `NullBool`, `NullFloat64`, `NullInt32`, `NillInt64`, `NullString`, and (recently) `NullTime`.  The `flag` package on the other hand has functions that either take in a pointer (i.e `BoolVar`, `DurationVar`, `Float64`, `Int64` etc ) or return a pointer (`Bool`, `Duration`, `Float`, `Int`).

In both cases you can see a lot of repitition and that functions and types are replicated for all the basic types.   This sounds exactly what generics in Go are for!

```golang
package main

import (
	"fmt"
)

// Optional type can be valid or not and store the value
type Optional(type T) struct {
	valid bool
	value T
}

// Empty returns a new Optional that is not valid
func Empty(type T)() Optional(T) {
	return Optional(T){}
}

// New createa new option with the given value and is implictly valid
func New(type T)(value T) Optional(T) {
	return Optional(T){
		valid: true,
		value: value,
	}
}

// FromPtr will take a point and create a new valid Optional if it is not
// nil and will automatically dereference the value.
func FromPtr(type T)(ptr *T) Optional(T) {
	if ptr == nil {
		return Optional(T){} // valid is automatically false
	}
	return Optional(T){
		value: *ptr,
		valid: true,
	}
}

// Valid returns if the there is an optional.
func (o *Optional(T)) Valid() bool { return o.valid }

// Get will return the value and and bool if it is valid or not.  This is 
// similar to getting a value from a map.
func (o *Optional(T)) Get() (T, bool) {
	return o.value, o.valid
}

// Must will return the valid or panic if it doesn't exist
func (o *Optional(T)) Must() T {
	if !o.valid {
		panic(fmt.Sprintf("%T is not valid", o.value))
	}
	return o.value
}

// OrZero will return a valid value or it's zero value
func (o *Optional(T)) OrZero() T {
	return o.value
}

// OrElse will return a valid value or a different value
func (o *Optional(T)) OrElse(other T) T {
	if o.valid {
		return o.value
	}
	return other
}

// Ptr will return a pointer to the value or nil if it is not valid.
func (o *Optional(T)) Ptr() *T {
	if !o.valid {
		return nil
	}
	return &o.value
}

// Ptr is a function to return a pointer to a value.  Also made generic :)
func Ptr(type T)(val T) *T {
	return &val
}

func main() {
	a := New(42)
	if v, ok := a.Get(); ok {
		fmt.Printf("a is valid and set to: %v\n", v)
	}
	
	// Create empty optional
	b := Empty(string)()
	// print if b is valid if not return "<unknown>"
	fmt.Println("b value:", b.OrElse("<unknown>"))
	

	// function call needed as you cannot take the address of a literal
	c := FromPtr(Ptr(42)) 
	fmt.Println("c valid:", c.Valid())
	cv, _ := c.Get()
	fmt.Println("c value:", cv) // value already derefed	
	
	// Create nil *int
	var nilInt *int
	d := FromPtr(nilInt)
	fmt.Println("d valid:", d.Valid())
	dv, _ := d.Get()
	fmt.Println("d value:", dv) // value already derefed	
}
```

Unlike `Result` type playing around with an Optional wrapper seems like it could be useful in a post-generic world.  I am sure there is other functionality that could be added to a wrapper as well. 

[Play with this in the go2go playground](https://go2goplay.golang.org/p/P8REnlsuld4)


## Summary
I did this examples as a though expirement with "what could be done with generics" and not so much "what *should* be done".  There are a couple take aways I had with this.  


1. Working with generics is pretty straight forward and easy.  Writing seemed to be easy even with the extra params, however, reading may hurt a bit.
2. There are a lot of questions about the future of Go in a post generic world.  What code will be idomatic and what won't be?  What is ok to have a feature-rich wrapper around, what won't be?  I cannot say for certain but experimentation will be key!

What else can be done with the generic code in Go?  What do you use in other languages you wish you could with Go that could be enabled by paramaterized types?