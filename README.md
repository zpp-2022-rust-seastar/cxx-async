# `cxx-async`

## Overview

`cxx-async` is a Rust crate that extends the [`cxx`](http://cxx.rs/) library to provide seamless
interoperability between asynchronous Rust code using `async`/`await` and [C++20 coroutines]
(https://en.cppreference.com/w/cpp/language/coroutines) using `co_await`. If your C++ code is
asynchronous, `cxx-async` can provide a more convenient, and potentially more efficient,
alternative to callbacks. You can freely convert between C++ coroutines and Rust futures and await
one from the other.

It's important to emphasize what `cxx-async` isn't: it isn't a C++ binding to Tokio or any other
Rust I/O library. Nor is it a Rust binding to `boost::asio` or similar. Such bindings could in
principle be layered on top of `cxx-async` if desired, but this crate doesn't provide them out of
the box. (Note that this is a tricky problem even in principle, since Rust async I/O code is
generally tightly coupled to a single library such as Tokio, in much the same way C++ async I/O
code tends to be tightly coupled to libraries like `boost::asio`.) If you're writing server code,
you can still use `cxx-async`, but you will need to ensure that both the Rust and C++ sides run
separate I/O executors.

`cxx-async` aims for compatibility with popular C++ coroutine support libraries. Right now,
both the lightweight [`cppcoro`](https://github.com/lewissbaker/cppcoro) and the more comprehensive
[Folly](https://github.com/facebook/folly/) are supported. Patches are welcome to support others.

## Quick tutorial

To use `cxx-async`, first start by adding `cxx` to your project. Then add the following to your
`Cargo.toml`:

```toml
[dependencies]
cxx-async = "0.1"
```

Now, inside your `#[cxx::bridge]` module, declare a future type and some methods like so:

```rust
#[cxx::bridge]
mod ffi {
    // Declare any future types you wish to bridge. They must begin with `RustFuture`.
    extern "Rust" {
        type RustFutureString;
    }

    // Async C++ methods that you wish Rust to call go here. Make sure they return one of the boxed
    // future types you declared above.
    unsafe extern "C++" {
        fn hello_from_cpp() -> Box<RustFutureString>;
    }

    // Async Rust methods that you wish C++ to call go here. Again, make sure they return one of the
    // boxed future types you declared above.
    extern "Rust" {
        fn hello_from_rust() -> Box<RustFutureString>;
    }
}
```

After the `#[cxx::bridge]` block, call the `define_cxx_future!` macro to define any `RustFuture`
types:

```rust
// The first argument is the name of the future type you declared, without the `RustFuture` prefix.
// The second argument is the Rust type that this future yields.
define_cxx_future!(String, String);
```

Now, in your C++ file, make sure to `#include` the right headers:

```cpp
#include "cxx_async.h"
#include "cxx_async_cppcoro.h"  // Or cxx_async_folly.h, depending on which library you're using.
#include "rust/cxx.h"
```

And add a call to the `CXXASYNC_DEFINE_FUTURE` macro to define the C++ side of the future:

```cpp
// Arguments are the same as `define_cxx_future!` on the Rust side. Note that the second argument
// is the C++ type that `cxx` maps your Rust type to: in this case, `String` mapped to
// `rust::String`, so we supply `rust::String` here.
CXXASYNC_DEFINE_FUTURE(String, rust::String);
```

You're all set! Now you can define asynchronous C++ code that Rust can call:

```cpp
rust::Box<RustFutureString> hello_from_cpp() {
    co_return std::string("Hello world!");
}
```

On the Rust side:

```rust
async fn call_cpp() -> String {
    // This returns a Result (with the error variant populated if C++ threw an exception), so you
    // need to unwrap it:
    ffi::hello_from_cpp().await.unwrap()
}
```

And likewise, define some asynchronous Rust code that C++ can call:

```rust
use cxx_async::CxxAsyncResult;
async fn hello_from_rust() -> CxxAsyncResult<String> {
    Ok("Hello world!".to_owned())
}
```

Over on the C++ side:

```cpp
cppcoro::task<rust::String> call_rust() {
    co_return hello_from_rust();
}
```

That's it!

## License

Licensed under either of Apache License, Version 2.0 or MIT license at your option.

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in
this project by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions. 
