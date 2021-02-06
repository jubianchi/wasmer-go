<div align="center">
  <a href="https://wasmer.io" target="_blank" rel="noopener noreferrer">
    <img width="300" src="https://raw.githubusercontent.com/wasmerio/wasmer/master/assets/logo.png" alt="Wasmer logo">
  </a>
  
  <h1>Wasmer Go</h1>
  
  <p>
    <a href="https://github.com/wasmerio/wasmer-go/actions?query=workflow%3A%22Build+and+Test%22">
      <img src="https://github.com/wasmerio/wasmer-go/workflows/Build%20and%20Test/badge.svg" alt="Build Status">
    </a>
    <a href="https://github.com/wasmerio/wasmer-go/blob/master/LICENSE">
      <img src="https://img.shields.io/github/license/wasmerio/wasmer-go.svg" alt="License">
    </a>
    <a href="https://pkg.go.dev/github.com/wasmerio/wasmer-go/wasmer">
      <img src="https://img.shields.io/badge/go.dev-package-f06" alt="Go Package">
    </a> 
    <a href="https://pkg.go.dev/github.com/wasmerio/wasmer-go/wasmer?tab=doc">
      <img src="https://img.shields.io/badge/documentation-API-f06" alt="API Documentation">
    </a> 
  </p>

  <h3>
    <a href="https://wasmer.io/">Website</a>
    <span> • </span>
    <a href="https://docs.wasmer.io">Docs</a>
    <span> • </span>
    <a href="https://slack.wasmer.io/">Slack Channel</a>
  </h3>

</div>

<hr/>

A complete and mature WebAssembly runtime for Go based on [Wasmer].

**You are seeing the readme for the latest Wasmer Go version, if you are using an older version, please go to:**
* [0.3](https://github.com/wasmerio/wasmer-go/tree/v0.3.1/README.md)
* [0.2](https://github.com/wasmerio/wasmer-go/tree/0.2.0/README.md)
* [0.1](https://github.com/wasmerio/wasmer-go/tree/0.1.0/README.md)

# Features

  * **Easy to use**: The `wasmer` API mimics the standard WebAssembly API,
  * **Fast**: `wasmer` executes the WebAssembly modules as fast as
    possible, close to **native speed**,
  * **Safe**: All calls to WebAssembly will be fast, but more
    importantly, completely safe and sandboxed.

[Wasmer]: https://github.com/wasmerio/wasmer

# Install

To install the library, follow the classical:

```sh
$ go get github.com/wasmerio/wasmer-go/wasmer
```

> Note: Wasmer doesn't work on Windows yet.

If the pre-compiled shared libraries are not compatible with your
system, please wait, we are writing the documentation.

# Examples

## Basic example: Exported function

There is a toy program in `wasmer/testdata/tests.rs`, written in Rust
(or any other language that compiles to WebAssembly), which contains a
function like this one:

```rust
#[no_mangle]
pub extern fn sum(x: i32, y: i32) -> i32 {
    x + y
}
```

After compilation to WebAssembly, the
[`wasmer/testdata/tests.wasm`](https://github.com/wasmerio/wasmer-go/blob/master/wasmer/testdata/tests.wasm)
binary file is generated. ([Download
it](https://github.com/wasmerio/wasmer-go/raw/master/wasmer/testdata/tests.wasm)).

Then, we can execute it in Go:

```go
package main

import (
	"fmt"
    "io/ioutil"
	wasmer "github.com/wasmerio/wasmer-go/wasmer"
)

func main() {
    wasmBytes, _ := ioutil.ReadFile("path/to/tests.wasm")

    engine := wasmer.NewEngine()
    store := wasmer.NewStore(engine)

    // Compiles the module
    module, _ := wasmer.NewModule(store, wasmBytes)

    // Instantiates the module
    importObject := wasmer.NewImportObject()
    instance, _ := wasmer.NewInstance(module, importObject)

    // Gets the `sum` exported function from the WebAssembly instance.
    sum, _ := instance.Exports.GetFunction("sum")

    // Calls that exported function with Go standard values. The WebAssembly
    // types are inferred and values are casted automatically.
    result, _ := sum(5, 37)

    fmt.Println(result) // 42!
}
```

## Imported function

A WebAssembly module can export functions, this is how to run a
WebAssembly function, like we did in the previous
example. Nonetheless, a WebAssembly module can depend on “extern
functions”, then called imported functions. For instance, let's
consider the basic following Rust program:

```rust
extern {
    fn sum(x: i32, y: i32) -> i32;
}

#[no_mangle]
pub extern fn add1(x: i32, y: i32) -> i32 {
    unsafe { sum(x, y) + 1 }
}
```

In this case, the `add1` function is a WebAssembly exported function,
whilst the `sum` function is a WebAssembly imported function (the
WebAssembly instance needs to _import_ it to complete the
program). Good news: We can write the implementation of the `sum`
function directly in Go! Let's declare the complete host function:

```go
function := wasmer.NewFunction(
	store,
	wasmer.NewFunctionType(wasmer.NewValueTypes(I32, I32), wasmer.NewValueTypes(I32)),
	func(args []wasmer.Value) ([]wasmer.Value, error) {
		x := args[0].I32()
		y := args[1].I32()

		return []wasmer.Value{wasmer.NewI32(x + y)}, nil
	},
)
```

Then, we need to define an `ImportObject` and register the funtion
inside the correct namespace:

```go
importObject.Register(
	"math",
	map[string]IntoExtern{
		"sum": function,
	},
)
```

Here we are. Now let's instantiate the module with the import object,
and let's call the `add_one` function:

```go
instance, _ := wasmer.NewInstance(module, importObject)

addOne, _ := instance.Exports.GetFunction("add_one")

result, _ := addOne(41)
fmt.Println(result) # 42!
```

## Read the memory

A WebAssembly instance has a linear memory. Let's see how to read
it. Consider the following Rust program:

```rust
#[no_mangle]
pub extern fn return_hello() -> *const u8 {
    b"Hello, World!\0".as_ptr()
}
```

The `return_hello` function returns a pointer to a string. This string
is stored in the WebAssembly memory. Let's read it.

```go
wasmBytes, _ := wasm.ReadBytes("simple.wasm")

engine := wasmer.NewEngine()
store := wasmer.NewStore(engine)

// Compiles the module
module, _ := wasmer.NewModule(store, wasmBytes)

// Instantiates the module
importObject := wasmer.NewImportObject()
instance, _ := wasmer.NewInstance(module, importObject)

// Calls the `return_hello` exported function.
// This function returns a pointer to a string.
result, _ := instance.GetFunction("return_hello")

// Gets the pointer value as an integer.
pointer := result.ToI32()

// Reads the memory.
memory := instance.exports.GetMemory("memory").Data()

fmt.Println(string(memory[pointer : pointer+13])) // Hello, World!
```

In this example, we already know the string length, and we use a slice
to read a portion of the memory directly. Notice that the string
terminates by a null byte, which means we could iterate over the
memory starting from `pointer` until a null byte is met; that's a
similar approach.

# Documentation

[The documentation can be read online on pkg.go.dev][documentation]. It
contains function descriptions, short examples, long examples
etc. Everything one need to start using Wasmer with Go!

[documentation]: https://pkg.go.dev/github.com/wasmerio/wasmer-go/wasmer?tab=doc

# Testing

Once the library is build, run the following command:

```sh
$ just test
```

# Benchmarks

We compared Wasmer to [Wagon][wagon] and [Life][life]. The benchmarks
are in `benchmarks/`. The computer that ran these benchmarks is a
MacBook Pro 15" from 2016, 2.9Ghz Core i7 with 16Gb of memory. Here
are the results in a table (the lower the ratio is, the better):

<table>
  <thead>
    <tr>
      <th>Benchmark</th>
      <th>Runtime</th>
      <th align="right">Time (ms)</th>
      <th align="right">Ratio</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="3">N-Body</td>
      <td>Wasmer</td>
      <td align="right">42.078</td>
      <td align="right">1×</td>
    </tr>
    <tr>
      <td>Wagon</td>
      <td align="right">1841.950</td>
      <td align="right">44×</td>
    </tr>
    <tr>
      <td>Life</td>
      <td align="right">1976.215</td>
      <td align="right">47×</td>
    </tr>
    <tr>
      <td rowspan="3">Fibonacci (recursive)</td>
      <td>Wasmer</td>
      <td align="right">28.559</td>
      <td align="right">1×</td>
    </tr>
    <tr>
      <td>Wagon</td>
      <td align="right">3238.050</td>
      <td align="right">113×</td>
    </tr>
    <tr>
      <td>Life</td>
      <td align="right">3029.209</td>
      <td align="right">106×</td>
    </tr>
    <tr>
      <td rowspan="3">Pollard rho 128</td>
      <td>Wasmer</td>
      <td align="right">37.478</td>
      <td align="right">1×</td>
    </tr>
    <tr>
      <td>Wagon</td>
      <td align="right">2165.563</td>
      <td align="right">58×</td>
    </tr>
    <tr>
      <td>Life</td>
      <td align="right">2407.752</td>
      <td align="right">64×</td>
    </tr>
  </tbody>
</table>

While both Life and Wagon provide on average the same speed, Wasmer is
on average 72 times faster.

Put on a graph, it looks like this (reminder: the lower, the better):

![Benchmark results](https://cdn-images-1.medium.com/max/1200/1*08ymx9shShohcPCKi1XjlA.png)

[wagon]: https://github.com/go-interpreter/wagon
[life]: https://github.com/perlin-network/life

# What is WebAssembly?

Quoting [the WebAssembly site](https://webassembly.org/):

> WebAssembly (abbreviated Wasm) is a binary instruction format for a
> stack-based virtual machine. Wasm is designed as a portable target
> for compilation of high-level languages like C/C++/Rust, enabling
> deployment on the web for client and server applications.

About speed:

> WebAssembly aims to execute at native speed by taking advantage of
> [common hardware
> capabilities](https://webassembly.org/docs/portability/#assumptions-for-efficient-execution)
> available on a wide range of platforms.

About safety:

> WebAssembly describes a memory-safe, sandboxed [execution
> environment](https://webassembly.org/docs/semantics/#linear-memory) […].

# License

The entire project is under the MIT License. Please read [the
`LICENSE` file][license].


[license]: https://github.com/wasmerio/wasmer/blob/master/LICENSE
