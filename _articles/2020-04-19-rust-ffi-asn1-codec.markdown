---
layout: single 
title:  "Rust FFI - Building an ASN1 codec - Part1"
date:   2020-04-19 20:07:00 +0530
categories: Rust programming FFI
---

The Rust ecosystem comes with all the tools you need to call into a C Library. This is a great way to use
existing libraries in your Rust application or library and also a nice way to introduce Rust to parts of your application.

In this tutorial, I will show you all you need to do in order to plug a C Library into a Rust library which you can then use
in your application. The source code of the library is included in the project and the tutorial also covers
building C sources from the Rust build system.

I recently had to parse some ASN.1 encoded data. The Rust crate for ASN.1 
[rust-asn1](https://github.com/alex/rust-asn1) was unable to parse my ASN.1 specification. The [asn1c](https://github.com/vlm/asn1c) project
could handle the ASN1 specifications I wanted to use. Asn1c generates standalone C code without any additional library dependencies. This is a 
fantastic approach that makes it very portable. Kudos to Lev Walkin for this nice project.

The specification I want to encode and decode is IEEE 1609.3. I won't go into the details of this specification as it is not important for the tutorial. I found the ASN.1 specification for this specification at [https://github.com/libv2x/v2x/tree/master/libv2x/asn1](https://github.com/libv2x/v2x/tree/master/libv2x/asn1). 

Our goal now is to build a Rust library that can encode and decode IEEE 1609.3 UPER packets. You should be able to use the same steps for any ASN.1 specification you may
have, as long as asn1c can handle it. Let's get started.

# Project set up
Let's create the project. Remember, we are creating a library.

```shell
$ cargo new ieee1609dot3codec-rs --lib
  Created library `ieee1609dot3codec-rs` package
```
Install asn1c. We could get fancy and download and build asn1c, but that will
overload this tutorial.  Follow instructions [here](https://github.com/vlm/asn1c/blob/master/INSTALL.m). Ensure that asn1c is in your path.

```shell
$ asn1c  -h
ASN.1 Compiler, v0.9.29
Copyright (c) 2003-2017 Lev Walkin <vlm@lionet.info> and contributors.
Usage: asn1c [options] file ...
```

Copy the ASN1 definition into our project. 
```shell
$ mkdir asn1 ; cd asn1
$ wget https://raw.githubusercontent.com/libv2x/v2x/master/libv2x/asn1/1609dot3all.asn
```
Your project directory should look like this.

```shell
.
├── asn1
│   └── 1609dot3all.asn
├── Cargo.toml
└── src
    └── lib.rs
```
# Generate the C codec source code
We could generate the code separately and check-in the generated source into 
the tree. A cleaner way is to check-in just the ASN1 specification and then generate and compile the source programmatically.

We need a build.rs to set up things before we start compiling Rust code.

```rust
// build.rs
use std::env;
use std::path::{Path, PathBuf};
use std::process::Command;

fn main() {
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());

    // Generate the codec source files from the ASN.1 specification.
    Command::new("asn1c")
        .args(&[
            "asn1/1609dot3all.asn",
            "-D",
            out_dir.to_str().unwrap(),
            "-fcompound-names",
            "-no-gen-example",
        ])
        .status()
        .unwrap();
}
```
Build this using 'cargo build'. If the build is successful, you will find a bunch
of generated C source files and headers in the OUT directory. The actual folder name 
will vary on your computer of course.

```shell
./target/debug/build/ieee1609dot3codec-rs-236ec4bb8be59d14/out
```

Let's extend the build.rs to now build these files so that they can be linked in to your Rust application.

First, update Cargo.toml to include cc in the build-dependencies section. 
```toml
[build-dependencies]
cc = { version = "1.0", features = ["parallel"]}
```

We now use the cc crate to compile the C sources. We write a little helper function
to find all the generated files to avoid listing out the generated files manually.

The build.rs now looks like this
```rust
use cc;
use std::env;
use std::fs::{self, DirEntry};
use std::path::{Path, PathBuf};
use std::process::Command;

/// return a tuple of lists. The first entry contains the list of .c files and the
/// second entry is the list of headers in the specified directory.
fn find_generated_sources_and_headers(out_dir: &Path) -> (Vec<PathBuf>, Vec<PathBuf>) {
    let mut sources = Vec::new();
    let mut headers = Vec::new();

    if out_dir.is_dir() {
        for entry in fs::read_dir(out_dir).unwrap() {
            let entry = entry.unwrap();
            let path = entry.path();
            if path.is_file() {
                if let Some(extension) = path.extension() {
                    match extension.to_str().unwrap() {
                        "c" => sources.push(PathBuf::from(path)),
                        "h" => headers.push(PathBuf::from(path)),
                        _ => {}
                    }
                }
            }
        }
    }
    (sources, headers)
}

fn main() {
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());

    // Generate the codec source files from the ASN.1 specification.
    Command::new("asn1c")
        .args(&[
            "asn1/1609dot3all.asn",
            "-D",
            out_dir.to_str().unwrap(),
            "-fcompound-names",
            "-no-gen-example",
        ])
        .status()
        .unwrap();

    let (sources, headers) = find_generated_sources_and_headers(&out_dir);

    let mut cc_builder = cc::Build::new();
    cc_builder.include(&out_dir.to_str().unwrap());

    for source in sources {
        cc_builder.file(&source);
    }

    cc_builder.flag("-Wno-missing-field-initializers");
    cc_builder.flag("-Wno-missing-braces");
    cc_builder.flag("-Wno-unused-parameter");
    cc_builder.flag("-Wno-unused-const-variable");
    cc_builder.compile("libasn1codec");
}
```
With these few lines of code, you have generated and compiled the C code. The library 
will get linked with any application that uses this library.  Now we need to figure out how to call the C Codec APIs from Rust.  

All we need to do is define each C Api and type using Rust as explained in [https://rust-embedded.github.io/book/interoperability/c-with-rust.html](https://rust-embedded.github.io/book/interoperability/c-with-rust.html). Writing this manually is a pain when you have hundreds of functions and types. Here comes [bindgen](https://rust-lang.github.io/rust-bindgen/) to the rescue. 

# Bindgen to generate unsafe Rust bindings to your C files
Start by adding bindgen to Cargo.toml
```toml
[build-dependencies]
cc = { version = "1.0", features = ["parallel"]}
bindgen = "0.53"
```

Add the following lines to use bindgen to generate the bindings. You may need to
add additional flags and headers.  Here, for example, we add the out_dir into the
include path. I also had to forcefully include sys/types.h and stdio.h.  Probably
due to a bug in the generated code.

```rust
   // Generate Bindings
    let mut builder = bindgen::Builder::default()
        .clang_arg(format!("-I{}", &out_dir.to_str().unwrap()))
        .header("sys/types.h")
        .header("stdio.h");

    for header in headers {
        builder = builder.header(String::from(header.to_str().unwrap()));
    }
    let bindings = builder.generate().expect("Unable to generate bindings");
    bindings
        .write_to_file(out_dir.join("bindings.rs"))
        .expect("Couldn't write bindings!");
```

Run the code now and the build should succeed. You will also find a bindings.rs created in the build folder. This contains
the generated rust bindings.

We're now done with build.rs, and are now ready to start writing some code for the library.

## Include the generated bindings into the library
So far, we have compiled the C sources and generated Rust bindings to call into the C Api. Before we can start making call
via the bindings, we need to include it into the library sources. Rust has an include! macro that allows you to include
other files into your source code. Add this line into the lib.rs
```rust
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]
#![allow(dead_code)]

include!(concat!(env!("OUT_DIR"), "/bindings.rs"));

```
Thats it! The generated bindings are now part of lib.rs.  We can start writing some tests to ensure that the library works
as expected. The allow macros are to disable warnings for the generated types which may not use Rust's preferred naming
conventions.

# Writing the first test
Let's take the generated bindings for a spin by writing some tests. For now, lets start with a single test to encode some data
into a 1609.3 frame. lib.rs now looks like this.

```rust
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]
#![allow(dead_code)]

include!(concat!(env!("OUT_DIR"), "/bindings.rs"));

#[cfg(test)]
mod tests {
    use super::*;
    use std::ffi::c_void;
    fn new_context() -> asn_struct_ctx_t {
        let ctx_struct: asn_struct_ctx_t = asn_struct_ctx_t {
            phase: 0,
            step: 0,
            context: 0,
            ptr: std::ptr::null_mut(),
            left: 0i64,
        };
        ctx_struct
    }

    fn create_short_msg_npdu(
        mut data: &mut [u8],
        version: i64,
        dest_address: i64,
    ) -> ShortMsgNpdu_t {
        let mut short_msg = ShortMsgNpdu_t {
            subtype: ShortMsgSubtype_t {
                present: ShortMsgSubtype_PR_ShortMsgSubtype_PR_nullNetworking,
                choice: ShortMsgSubtype_ShortMsgSubtype_u {
                    nullNetworking: NullNetworking_t {
                        version: version,
                        nExtensions: std::ptr::null_mut(),
                        _asn_ctx: new_context(),
                    },
                },
                _asn_ctx: new_context(),
            },
            transport: ShortMsgTpdus_t {
                present: ShortMsgTpdus_PR_ShortMsgTpdus_PR_bcMode,
                choice: ShortMsgTpdus_ShortMsgTpdus_u {
                    bcMode: ShortMsgBcPDU_t {
                        destAddress: VarLengthNumber_t {
                            present: VarLengthNumber_PR_VarLengthNumber_PR_content,
                            choice: VarLengthNumber_VarLengthNumber_u {
                                content: dest_address,
                            },
                            _asn_ctx: new_context(),
                        },
                        tExtensions: std::ptr::null_mut(),
                        _asn_ctx: new_context(),
                    },
                },
                _asn_ctx: new_context(),
            },
            body: ShortMsgData_t {
                /* OCTET_DATA */
                buf: data.as_mut_ptr(),
                size: data.len() as u64,
                _asn_ctx: new_context(),
            },
            _asn_ctx: new_context(),
        };
        short_msg
    }

    #[test]
    fn encode_short_msg_n_pdu() {
        let mut data = vec![0xFEu8; 10];
        let version = 2;
        let dest_address = 12;
        let mut encoded_data = vec![0u8; 128];

        let mut short_msg = create_short_msg_npdu(&mut data, version, dest_address);

        let message_ptr: *mut c_void = &mut short_msg as *mut _ as *mut c_void;
        let encode_buffer_ptr: *mut c_void = encoded_data.as_mut_ptr() as *mut _ as *mut c_void;

        unsafe {
            let enc_rval = uper_encode_to_buffer(
                &asn_DEF_ShortMsgNpdu,
                std::ptr::null(),
                message_ptr,
                encode_buffer_ptr,
                encoded_data.len() as u64,
            );
            if enc_rval.encoded > 0 {
                println!(
                    "Success! encoded ShortMsgNpdu, {} bytes {} bits",
                    enc_rval.encoded / 8,
                    enc_rval.encoded % 8,
                );
                let num_bytes = enc_rval.encoded / 8;
                encoded_data.resize(num_bytes as usize, 0);
            } else {
                panic!("Encode ShortMsgNpdu failed,  {}", enc_rval.encoded);
            }
        }
    }
}
```
This seems like a lot of code, but most of this is just initializing the structure before it is encoded.
Initializing a complex structure like this in C can be error prone. With Rust, it is impossible to 
create a structure without initializing all its members.  The function create_short_msg_npdu on line 23
creates the structure for the PDU and populates all the fields. The test function encode_short_msg_n_pdu
encodes the structure using the uper_encode_to_buffer api from the codec. This is a C function. The binding for
this function is part of the lib module that is included by the include! macro in line 6.

FFI calls are unsafe as Rust cannot enforce safety inside another language. So any calls to C functions 
have to be in unsafe blocks. Usually a Rust safe wrapper is written to wrap around all the unsafe C calls. It is possible
to combine the safe wrapper and the unsafe bindings into one crate, but the convention is to split them into separate crates.
A crate name postfixed with "-sys" usually has the generated bindings and the "-rs" is used to signify the safe Rust wrappers
around the unsafe bindings.

We have now created a Rust library that exposes all the C APIs as unsafe Rust functions. Encoding seems to work fine, but
we have one big problem to solve before we implement the decoding.

The asn1c codec allocates memory using the C malloc api. The decoded structure we get back from the asn1c is thus not allocated
by Rust and this is going to cause errors when a decoded structure goes out of scope. We need to do something so that the asn1 APIs
are called for freeing structures created by the library instead of the Rust deallocator.

# Solving the memory de-allocation problem
In this second part of the tutorial, we will ensure that structures that are allocated by the asn1c library can be safely
used in Rust code.  The correct deallocator for the structure will be used when it goes out of scope.

Each asn1c generated type has q type descriptor. The de-allocation function for the type
is part of the descriptor.  

## The syn crate
The syn crate parses Rust code and allows you to iterate over the token tree. We want to generate some additional code in bindings.rs. 

First, add syn to Cargo.toml
```toml
[build-dependencies]
cc = { version = "1.0", features = ["parallel"]}
bindgen = "0.53"
syn = {version = "1.0", features = ["full", "printing"]}
```



































