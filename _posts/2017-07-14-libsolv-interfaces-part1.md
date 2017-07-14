---
layout: post
title:  "libsolv - A case study in interfacing with C - Part 1"
date:   2017-07-14 15:00:00 -0500
categories: rust libsolv bindgen
---

This is the first of several series detailing how one would use Rust in real world scenarios. In this series, I will detail the trials, tribulations, and (hopefully) triumphs in creating a Rust interface to a reasonably complex C library: [libsolv](https://github.com/opensuse/libsolv) and, by extension, libsolvext. 

## Choosing A Scaffold
"Every step of progress the world has made has been from scaffold to scaffold, and from stake to stake." - Wendell Phillips

When contemplating at any extraordinarily daunting task, it's useful to identify some measuring stick by which one gauges progress. Initially, at least, the measuring stick for libsolv's interface needs to be reasonably simple and straightforward with few external dependencies. For our purposes, [depchase](https://github.com/fedora-modularity/depchase) fills this role. Depchase is a simple python tool used for hunting down RPM dependencies. It primarily depends on libsolv & libsolvext. It needs only simple inputs and has well-defined outputs.

As I progress, I will adapt other scaffolds.

[Let's begin](https://github.com/ambaxter/libsolv-rs/tree/rewiring)

## Goals

### Project Goals
* A 1:1 port of depchase in Rust, down to the output formatting, if possible
* A reasonably Rusty interface for libsolv and other necessary dependencies

### Project Non-Goals
* rewrite libsolv, etc.. - one day...
* performance
* cross-architecture support - I have but one x86_64 machine

## Library Overview
Libsolv is package dependency solver used by some Linux distributions to install software via various front-ends (DNF for Fedora).  It is a rather expansive codebase split across two libraries (libsolv and libsolvext) with roughly 70k lines of C. The library exports an impressive array of types and functions.



| Library | Functions* | Types* |
| :---- | ----: | ----: |
| libsolv | 398 | 23 |
| libsolvext | 24 | 2 |


*represents what is currently wrapped by bindgen. These numbers are expected to increase
 
The code has no inline documentation, though documentation exists as both man pages and the project website. The API is very clean and, once you get used to C idioms and behaviors (it's been a few years for me), reading the source is straightforward.
 
libsolv focuses on the core functionality of solving dependencies whereas libsolvext provides handling for test cases, rpm\|deb\|arch\|haiku files, and distribution-specific features.

## Project Organization
A crate is Rust's unit of compilation and is defined by a Cargo.toml file, akin to a cmake/makefile for C, Gemfile for Ruby, or setup.py for Python. It describes project metadata (authors, documentation location, project site), dependencies, project settings, etc. To learn more about the crates and Rust's ecosystem, check out [https://crates.io/](https://crates.io/)

libsolv-rs is split (at time of writing) into four crates - one parent and three children.
* libsolv-rs - the library other Rust applications and libraries will depend on. All of our Rust code will go here
* libsolv-sys - child library - bindings for the libsolv library
* libsolvext-sys - child library - bindings for the libsolvext library
* systest - to be used in the future

## On C Bindings
Mozilla's [Servo](https://github.com/servo) project has spun off many high-quality Rust libraries and tools used across the Rust ecosystem. The most valuable tool that this project uses is [bindgen](https://github.com/servo/rust-bindgen). Bindgen automagically generates C-to-Rust library bindings for us with minimal configuration. It's a major time saver and kudos to the Servo team for developing such an amazing product. 

My first attempt at manually binding libsolv in Rust was less than ideal. The process was slow and extremely error prone. Thankfully, Mozilla has already handled that particular issue. 

## libsolv-sys
Take a look at the [libsolv-sys crate](https://github.com/ambaxter/libsolv-rs/tree/rewiring/libsolv-sys) and I'll explain the major features.
* Cargo.toml - Defines the crate and its dependencies.
* wrapper.h - Bindgen requires a .h file to specify what C functionality it's going to import
* build.rs - Defines a custom build script for Cargo, Rust's build tool. I'll go into this in more detail in a bit.
* src/lib.rs - the main library file. The include! macro will dump everything bindgen finds here
* static/* - bindgen can't resolve C static inline functions because they don't exist in the library nor are they available in the compilation unit. While possibly slower, we'll compile these so Rust can see the inline symbols we can use.
* src/export - the Rust side of the static binding

Let's look at build.rs in a bit more detail.

```rust
fn main() {
   // Compile the static inline functions into an archive
    // and directs Cargo to link it
    gcc::Config::new()
        .file("static/queue.c")
        .file("static/bitmap.c")
          // snip
        .compile("libsolv-static-functions.a");

    // Direct Cargo to link the libsolv library
    println!("cargo:rustc-link-lib=solv");

    let bindings = bindgen::Builder::default()
        // The input header we would like to generate
        // bindings for.
        .header("wrapper.h")

        // Prefer the libc crate's definitions for libc types
        .ctypes_prefix("libc")

        // Whitelist libsolv's functions, types, and variables,
        // otherwise bindgen will bind all of libc
        .whitelisted_type("Solver")
        .whitelisted_function("solv.*")
          //snip

        // Hide FILE from bindgen's output
        // Otherwise we get the OS's private file implementation
        .hide_type("FILE")
        .raw_line("use libc::FILE;")

        .generate()
        // Unwrap the Result and panic on failure.
        .expect("Unable to generate bindings");

    // Write the bindings to the $OUT_DIR/bindings.rs file.
    let out_path = PathBuf::from(env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings!");
}
```

We don't have to define all of libsolv's types or functions. Not only does bindgen's whitelist functions accept regular expressions, it also will automatically recursively resolve referenced types if you don't specify them. This feature can be turned off if necessary.
 
## libsolvext-sys
The [libsolvext-sys crate](https://github.com/ambaxter/libsolv-rs/tree/rewiring/libsolvext-sys) is similar to libsolv-sys, but lacks the need for static functions (for now). Libsolvext makes use of C #defines to configure optional includes and features. Libsolvext-sys accounts for this with a slightly more complex wrapper.h based off the cmake includes. 

```c
// Defines libsolvext compiled features
#include <solv/solvversion.h>

// Always included
#include <solv/testcase.h>
#include <solv/solv_xfopen.h>
#include <solv/tools_util.h>

// Scraped from CMAKE

#if defined(LIBSOLVEXT_FEATURE_RPMDB) || defined(LIBSOLVEXT_FEATURE_RPMPKG)
#include <solv/pool_fileconflicts.h>
#include <solv/repo_rpmdb.h>
#endif

//snip...
```

Build.rs is similar to libsolv-rs's, but we additionally need to account for libsolv-rs already supplying libsolv's structs.

```rust
fn main() {
    let bindings = bindgen::Builder::default()
        // The input header we would like to generate
        // bindings for.
        .header("wrapper.h")

        // Prefer the libc crate's definitions for libc types
        .ctypes_prefix("libc")

        // <solv/testcase.h>
        .whitelisted_var("TESTCASE.*")
        .whitelisted_function("testcase.*")

        // <solv/solv_xfopen.h>
        .whitelisted_function("solv_xfopen.*")

        // <solv/pool_fileconflicts.h>
        .whitelisted_var("FINDFILECONFLICTS.*")
        .whitelisted_function("pool_findfileconflicts")
    //snip

        // Don't let bindgen recreate libsolv's types
        .hide_type("Chksum")
        .hide_type("DUChanges")
        .hide_type("Dataiterator")
        .hide_type("Datamatcher")
    // snip

        // Import necessary structs from libsolv_sys
        .raw_line("use libsolv_sys::{Chksum, DUChanges, Dataiterator, Datamatcher, Datapos, Dirpool};")
    //snip 

}
```


## Results
After some relatively minor configuration, Rust bindings for libsolv and libsolvext will be auto-generated for us. This provides a nice base for all future development.
