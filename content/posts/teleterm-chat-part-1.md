+++
title = 'Teleterm Chat Part 1'
date = 2024-02-02T21:30:26+08:00
draft = false
series = 'Teleterm Chat'
tags = ["terminal", "rust"]
+++

The first thing I learnt from this project has nothing to do with software. It's that asking the right questions can make a huge difference to how I feel and how I approach the problem at hand. I eventually landed on figuring out how to call C code from Rust as the first problem to solve.

## Asking better questions

Having no experience creating command-line applications, I was at a loss for where to start. As per the previous post, I thought I could try to figure out authentication first, but even that was too difficult of a problem for me to tackle. What I realised was the kind of questions I ask myself makes a huge difference.

When I set the goal of creating this app, I was asking myself "where can I start?" My answer was "I don't know". Even if I pointed at something seemingly obvious, such as authentication, I wouldn't have an answer for the next "so where can I start for authentication?" This felt very demoralising.

What helped was changing the question that I asked myself. I tried to instead ask "what smaller problem can I solve?" It seems like a small change, but it is this change that focused my mind on breaking down the problem into smaller pieces, as opposed to trying to search for non-existent places in my mind (as prompted by the "where" keyword).

Repeated asking "what smaller problem can I solve" is like recursion. The base case here is arriving at a problem I think I can solve. If I don't yet have the base case, I ask myself the same question again, breaking down the already broken-down problem into ever smaller pieces, until I eventually arrive at the base case.

My recursive process went something like this, where each point is a breakdown from the previous point.

- How to build Teleterm Chat?
- I can break the app down into 3 key parts: login, create command, send message. Which can I focus on first?
- How to just send a message programmatically? (as this is the core function I want)
- How to use [TDLib's JSON interface](https://core.telegram.org/tdlib/docs/#using-json) from Rust?
- How do I call C code from Rust, i.e. use FFIs (Foreign Function Interface)?

There we go, something I think I can solve! Prior to this, I had zero experience with FFIs of any kind. I did use [napi-rs](https://napi.rs/) to write a Rust library for NodeJS before, but I don't think that counts because all the FFIs were handled by napi-rs. I knew it was possible to call C code from Rust, I just didn't know how. But at least I felt like I could figure it out!

_As an aside, there are actually [Rust libraries](https://github.com/tdlib/td/blob/master/example/README.md#rust) that implement the interface between Rust and TDLib. However, I'm choosing not to use them for two reasons. First, one big goal I have with this project is to learn as much as I can about Rust. This means trying to keep the libraries and abstractions that I use to a minimum. Second, I don't foresee my app needing every single API that TDLib has to offer, so I don't want to include large libraries that cover all functionality._

## Rust FFI

Where do I even begin for this one? I did so much googling and experimenting in a disorganised fashion, yet I still somehow managed to get the outcome I desired. Let's see...

I didn't want to just call C from Rust; I wanted to ensure that I was calling the code in the same way that I would be calling TDLib. So I built the library as per the [instructions for macOS](https://tdlib.github.io/td/build.html?language=Rust) and installed it as a subdirectory in the root of my project folder, i.e. `project/td/tdlib`. There were two things I learnt before even trying to write Rust to call C.

The first was that my Rust code would have to interact with a file that has a `.dylib` extension, specifically `libtdjson.dylib`. I'm not sure what the exact details are, but the way I understand it is that the file is a dynamic library that would be searched up by my program at runtime; contrast this with static libraries that are linked into the program at compile time.

The one new term that I learnt was linked libraries. I think these have a similar idea to `node_modules` in the JavaScript ecosystem or crates in the Rust ecosystem, where a library is a bunch of code that we can import to utilise in our program.

The second was that `libtdjson` really means the name of the library is just `tdjson`. This seems to be a common naming convention. I suppose that's why the C programming language is also called `libc`? Something good to know, so that I don't get confused in the future.

### Calling a basic C library first

With these two points in hand, I set out to first call a basic C library from Rust. TDLib is a large and complex library, that's why I chose to start small with C library that only had a single function.

```C
# hello.c

int square(int val) {
    return val * val;
}
```

I then had to compile it to a dynamic library with a C compiler (I used [clang](https://clang.llvm.org/) as I'm on MacOS). This was simple enough[^1] to do with `clang -dynamiclib -o libhello.dylib hello.c`, where `-dynamiclib` means I want to create a dynamic library, and `-o filename.ext` specifies that I want to write the output of compiling the file `hello.c` to the file `filename.ext`.

[^1]: Simple enough means some googling to find some [sample commands](https://stackoverflow.com/a/21909880), then reading man pages and experimenting a bit more to find out which are the key flags and arguments to achieve my desired outcome.

That's the basic C library ready! Next up, calling this library from Rust.

Calling the basic C library was quite straightforward because there is plenty of documentation with similarly simple examples, such as the [Rustonomicon](https://doc.rust-lang.org/nomicon/ffi.html). The main thing I had to learn was the use of the [`extern` keyword](https://doc.rust-lang.org/std/keyword.extern.html) to declare function interfaces that our Rust code can call the C from. The other is the [link attribute](https://doc.rust-lang.org/reference/items/external-blocks.html#the-link-attribute) that is added above the `extern "C"` block, which specifies the name of the library that the Rust compiler should look for, which in this case is a file named `libhello`. So my basic Rust code looks like this.

```rust
// src/main.rs

#[link(name = "hello")]
extern "C" {
    fn square(val: i32) -> i32;
}

fn main() {
    let r = unsafe { square(3) };
    println!("3 squared is {r}");
}
```

However, the compiler [doesn't know where to find this custom library](https://stackoverflow.com/questions/26246849/how-to-i-tell-rust-where-to-look-for-a-static-library), as trying to run the program as is fails with ``error: linking with `cc` failed: exit status: 1`` and the linker error `ld: library 'hello' not found`, so we have to tell it with a [build script](https://doc.rust-lang.org/cargo/reference/build-scripts.html).

```rust
// build.rs

fn main() {
    println!("cargo:rustc-link-search=.")
}
```

The `rustc-link-search` flag tells cargo to include the path indicated after the equals sign in the list of paths that the system linker will search for libraries, which in this case is the root directory of the project because that's where I chose to write the built dynamic library to.

Running the program gave me the value `9` as expected. Great! That was my first time doing any form of FFI, and I'm sure it won't be the last.

### Actually calling TDLib

Calling the simple libhello library was simple; calling TDLib wasn't. The main I faced was that the dynamic library wasn't located in the project's root directory. Instead, it was located in `td/tdlib/lib`. After I moved my `libhello.dylib` to a similar path, simply changing the path for the `rustc-link-search` flag to `td/tdlib/lib` returned the error `dyld[49929]: Library not loaded: libhello.dylib` along with a list of paths that the linker tried to search for the library, which (unsurprisingly) doesn't include `td/tdlib/lib`.

At this point, I decided to switch over to trying to execute code from TDLib instead, mainly because I already have a basic understanding of Rust's FFI from the basic C library example. I mimicked this [python example](https://github.com/tdlib/td/blob/437c2d0c6e0ad104022d5ad86ddc8aedc41cb7a8/example/python/tdjson_example.py#L79) in the TDLib repo, calling the [`execute` method](https://core.telegram.org/tdlib/docs/td__json__client_8h.html#aff3d28f8896cf0b74f7825e3198f5b3e) in TDLib which will return a sensible value when called successfully. I referenced the extern function definitions from the [rust-tdlib project](https://github.com/antonio-antuan/rust-tdlib/blob/master/src/tdjson.rs) for the TDLib API. My initial test code looked something like this:

```rust
// src/main.rs

#[link(name = tdjson)] // the name of the library file is `libtdjson.dylib`
extern "C" {
	fn execute(request: *const c_char) -> *const c_char;
}

fn main() {
	let test_str = execute("{\"@type\": \"getTextEntities\", \"text\": \"@telegram /test_command https://telegram.org telegram.me\", \"@extra\": [\"5\", 7.0, \"a\"]}").unwrap();
	println!("From telegram: {}", test_str);
}
```

As I noted above, trying to call the library with the same setup as I had for `libhello` doesn't work, so I had to try other things.

My first thought was to somehow try to include the library into the `target/` directory where the final binary is built, because [that's a path where cargo will look](https://doc.rust-lang.org/cargo/reference/environment-variables.html#dynamic-library-paths) when searching for dynamic libraries. I don't think plainly copying the library folder into the target directory is a wise choice, as the target directory is [managed by cargo](https://doc.rust-lang.org/cargo/guide/build-cache.html).

Instead, I wanted to try doing it like the [example in the cargo book](https://doc.rust-lang.org/cargo/reference/build-script-examples.html#building-a-native-library), where I would use a `build.rs` script to build and include the library into our program. I tried to use the [cmake crate](https://docs.rs/cmake/0.1.50/cmake/) to mimic the build script used to build TDLib. However, this didn't work for reasons that are still unclear to me, as I couldn't verify whether the script successfully built the library or not.[^2]

[^2]: Interestingly, after writing the script, rust-analyzer (Rust's LSP) held a lock on the build file for exceptionally long, so I had to wait a while before I could test the output. I suspect rust-analyzer was building the library in the background, as it used [cargo check](https://doc.rust-lang.org/cargo/commands/cargo-check.html) which compiles the packages, but I couldn't confirm as I wasn't able to get any logs from the LSP during that time.

What ended up making the difference for me was searching for the right terms. In this case, it was the difference between static and dynamic libraries. When I fine-tuned my search to be about setting custom directories for linking **dynamic** libraries, I came across [this stackoverflow thread](https://stackoverflow.com/questions/40602708/linking-rust-application-with-a-dynamic-library-not-in-the-runtime-linker-search) that eventually led me to a working solution. The key code is in the `build.rs` file, but another important change was removing the `link` attribute from the `extern "C"` block in the main program. You can see the rest of the code on [GitHub](https://github.com/darricheng/teleterm-chat/tree/c34efefc80d8fa602ca323cf038243e3b45395a2).

```rust
// build.rs

use std::env;

fn main() {
    let root_dir = env::current_dir().unwrap();

    println!("cargo:rustc-link-lib=dylib=tdjson");
    println!(
        "cargo:rustc-link-search=native={}/td/tdlib/lib/",
        root_dir.display()
    );
    println!(
        "cargo:rustc-link-arg=-Wl,-rpath,{}/td/tdlib/lib/",
        root_dir.display()
    );
}
```

I honestly still do not know why or how this works, so that is something I'll have to look into more in time to come. The one thing I did learn has to do with `cargo:rustc-link-arg=-Wl,-rpath,{}/td/tdlib/lib/`. Whatever argument we're passing to cargo with that line is actually being passed first to the compiler, which I think is LLVM but invoked through clang[^3], which then passes the `-rpath` flag through to the linker using the `-Wl` flag. Still, very cool stuff! I'm excited to learn more about how all these software work, and then build even cooler stuff with them.

[^3]: I say this because both clang and gcc have the `-Wl` flag, which according to clang's documentation, is to "Pass the comma separated arguments to the linker" ([gcc](https://gcc.gnu.org/onlinedocs/gcc-3.2/gcc/Link-Options.html) has a similar description).

_Side note: When building TDLib, I could choose where to install the built library. The two options suggested were `td/tdlib` or `/usr/local`, of which I chose the former (of course, you can install it anywhere else too). I think installing it to the latter option might mitigate some of the issues I faced (I didn't try this though), but I wanted to isolate this library as development dependency for this project only, so I pushed through with getting the library installation in the project root directory working._

And with that, I have successfully called TDLib from Rust!

Next up, I'll be learning more about asynchronous programming in Rust with the [tokio crate](https://tokio.rs/). Why? Because most of TDLib's API involves asynchronous function calls via the `td_send` method on the JSON interface. When we send a request to `td_send`, TDLib will prepare the response, but we'll have to retrieve it via a call to the `td_receive` method. In the [python example](https://github.com/tdlib/td/blob/d79bd4b69403868897496da39b773ab25c69f6af/example/python/tdjson_example.py#L85), they use a while loop to keep receiving events from `td_receive`, then handle the event accordingly.

However, I don't think this is the way I want my program to work. I briefly recall from one of Jon Gjengset's[^4] [videos about async Rust](https://www.youtube.com/@jonhoo/search?query=async) that the async runtime can register event handlers with the operating system to be woken up when specific asynchronous functions return, which is what I want to happen in the final program. I'm currently thinking that I might need a daemon running in the background, so that when I invoke the one liner command to send a message, I won't need to go through the entire authentication process again, then the daemon will handle the sending of the message. So I will need an async runtime to handle the interactions with TDLib.

[^4]: I think he is hands down the best educational content creator for Rust (and really just programming in general too because of how close to the metal Rust is) that I have found thus far. His videos are long, but he goes deep and takes the time to explain difficult concepts in a clear manner. Highly recommend!

Anyway, plenty of this is still speculation by me at this point. Maybe the async runtime will be absolutely unnecessary in the final implementation. But what the heck, a big reason I'm doing this project is to learn Rust. So I'll have fun learning more with tokio first, then worry about whether it is necessary or not later on! I will only really know once I try building out the app with this hypothesis.

Onwards to learning async Rust with tokio!
