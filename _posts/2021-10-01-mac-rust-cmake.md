---
layout:             post
title:              "Install Rust and CMake on an Intel MacBook"
category:           "Programming Languages"
tags:               rust
permalink:          /posts/mac-rust-cmake-installation
last_modified_at:   "2023-02-02"
---

A reproducible record of how to install [Rust](https://www.rust-lang.org/) and [CMake](https://cmake.org/) on macOS. Rust is a memory-safe system programming language that has gained popularity over recent years. CMake is a cross-platform, compiler-independent, and open-source build system generator.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## macOS Rust Installation

It is suggested to use `rustup` as it will (among other things) allow us to switch between versions of Rust without having to download anything additional:

```bash
# Follow this link: https://www.rust-lang.org/tools/install
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Or, we can simply do:

```bash
brew install rustup
```

After a while, we should see a prompt:

```console
1) Proceed with installation (default)
2) Customize installation
3) Cancell installation
>
```

We can just choose `1` and follow the default installation process. If in the former step we use Homebrew to install `rustup`, we need to run:

```bash
rustup-init
```

If everything goes well, we can restart the terminal window and check the version of Rust installed on our machine:

```console
lochlomond:~$ rustup --version
rustup 1.25.2 (2023-02-01)
info: This is the version for the rustup toolchain manager, not the rustc compiler.
info: The currently active `rustc` version is `rustc 1.67.1 (d5a82bbd2 2023-02-07)`
lochlomond:~$ rustc --version
rustc 1.67.1 (d5a82bbd2 2023-02-07)
lochlomond:~$ cargo --version
cargo 1.67.1 (8ecd4f20a 2023-01-10)
```

Now, we should see two newly created directories inside our `$HOME` directory: `.cargo` and `.rustup`. Of course we can move them into another directory, e.g., `$HOME/Library/Rust`. To make that happen, we need to overwrite the default values of certain environment variables in our `.bash_profile`:

```bash
# Add Rust
export CARGO_HOME=$HOME/Library/Rust/.cargo
export RUSTUP_HOME=$HOME/Library/Rust/.rustup
export PATH="$CARGO_HOME/bin:$PATH"
. "$CARGO_HOME/env"
```

We can observe their locations as shown below:

```console
lochlomond:~$ which rustc
/Users/xingjianxuanyuan/Library/Rust/.cargo/bin/rustc
lochlomond:~$ which rustup
/Users/xingjianxuanyuan/Library/Rust/.cargo/bin/rustup
lochlomond:~$ which cargo
/Users/xingjianxuanyuan/Library/Rust/.cargo/bin/cargo
```

### My First Rust Program

This example is taken from [Mara Bos, *Rust Atomics and Locks: Low-Level Concurrency in Practice*, O'Reilly, 2023](https://marabos.nl/atomics/).

```rust
use std::thread;

fn main() {
    let t1 = thread::spawn(f);
    let t2 = thread::spawn(f);

    println!("Hello from the main thread.");

    t1.join().unwrap();
    t2.join().unwrap();
}

fn f() {
    println!("Hello from another thread!");

    let id = thread::current().id();
    println!("This is my thread id: {id:?}");
}
```

The standard library's `std::thread::spawn` function[^1] takes a single argument: the function the thread to be spawned will execute. The thread stops once this function returns. We let both spawned threads print a message and show their thread ID, which is a unique identifier for each thread and is of type `ThreadID`. The `println` macro uses `std::io::Stdout::lock()` to make sure its output does not get interrupted. A `println!()` expression will wait until any concurrently running one is finished before writing any output. The `.join()` method waits until the thread has finished executing and returns a `std::thread::Result`. The `.unwrap()` method handles the situation when the thread did not successfully finish its function because it panicked.

```console
$ rustc hello.rs
$ ./hello
Hello from the main thread.
Hello from another thread!
This is my thread id: ThreadId(3)
Hello from another thread!
This is my thread id: ThreadId(2)
```

## macOS CMake Installation

I include below a copy of the instructions found [here](https://gist.github.com/fscm/29fd23093221cf4d96ccfaac5a1a5c90) for easy future reference. All the credit goes to [Frederico Martins](https://gist.github.com/fscm).

If we already have CMake installed, we should first do the following:

```bash
sudo find /usr/local/bin -type l -lname '/Applications/CMake.app/*' -delete

sudo rm -rf /Applications/CMake.app
```

The official download page of CMake is [here](https://cmake.org/download/). We can choose the version we would like to install and copy the corresponding link:

```bash
mkdir ~/Downloads/CMake

# The latest release is 3.25.2
curl --silent --location --retry 3 "https://github.com/Kitware/CMake/releases/download/v3.25.2/cmake-3.25.2-Darwin-x86_64.dmg" --output ~/Downloads/CMake/cmake-Darwin-x86_64.dmg
```

Mount the downloaded installation image using the following command:

```bash
yes | PAGER=cat hdiutil attach -quiet -mountpoint /Volumes/cmake-Darwin-x86_64 ~/Downloads/CMake/cmake-Darwin-x86_64.dmg
```

Copy `CMake.app` to the `/Applications` folder:

```bash
cp -R /Volumes/cmake-Darwin-x86_64/CMake.app /Applications/
```

Unmount the image:

```bash
hdiutil detach /Volumes/cmake-Darwin-x86_64
```

Add the CMake tool to the `$PATH` variable:

```bash
sudo "/Applications/CMake.app/Contents/bin/cmake-gui" --install=/usr/local/bin
```

Verify our installation:

```console
lochlomond:~$ cmake --version
cmake version 3.25.2

CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

Finally, we can remove the downloaded installation image:

```bash
rm -rf ~/Downloads/CMake
```

[^1]: The `std::thread::spawn` function is actually just a convenient shorthand for `std::thread::Builder::new().spawn().unwrap()`. A `std::thread::Builder` allows us to set some settings for the new thread before spawning it.