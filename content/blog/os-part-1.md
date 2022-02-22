+++
title = "Rust OS - Part 1"
date = 2022-02-18
[taxonomies]
categories=["OS"]
tags=["Rust", "OS", "UEFI"]
+++

Welcome to part 1 of this blog series that will focus on creating an operating system kernel purely in rust.
This project was started as a means to deepen my knowledge of both Rust and operating system internals. The project can be found on [github](https://github.com/applepiesNX/uefi-bootloader-rust).

# Introduction #

Before we begin creating our kernel, we need to first make a few design decisions like what architecture
we are targeting and how our kernel will be booted. The present guide will focus on creating a kernel
targeting the x86_64 architecture. \
As for booting there are many paths we can take such as using BIOS, UEFI or
making our kernel multiboot2 compliant so that we can use any multiboot2 bootloader. Theres an argument
to be made for each of them ,but to keep things short we will be using a systems built-in UEFI firmware to boot
our kernel. Therefore, the first step in creating our operating system would be to a write an UEFI bootloader in Rust.

# UEFI Bootloader #
The first step would be to simply use cargo to create a new rust project.
```
cargo new bootloader-rust
```

## Building the Bootloader ##
In order to compile this project so that it can be executed by the  UEFI firmware, we need to tell cargo that we are building this project for a x86_64 uefi system that has no underlying operating system. By default cargo will compile the project to be run on the operating system and architecture that it is being built on. We can overwrite this behaviour by passing the `--target=TARGET` flag to cargo to specify which target we want to build for. Fortunately for us Rust has an inbuilt target that we can use to build our project to be run directly on the UEFI firmware. Rust also supports the creation of our own custom targets, however we will get to that in a later part when we are creating our kernel. For now, lets use the in-built ` x86_64-unknown-uefi` target to compile our project. We will also need to use the nightly compiler as we need to use a few unstable features. If the nightly compiler is not install it can be installed with the following command:
```
rustup install nightly
```

With the nightly compiler installed, In order to build our bootloader we will run the following command:
```
cargo +nightly build --target=x86_64-unknown-uefi
```

When we run this command we get the following error:

```
error[E0463]: can't find crate for `std`
  |
  = note: the `x86_64-unknown-uefi` target may not be installed
  = help: consider downloading the target with `rustup target add x86_64-unknown-uefi`
  = help: consider building the standard library from source with `cargo build -Zbuild-std`
```
This is because Rust does not provide prebuilt binaries for the UEFI target .The `std` crate depends on the underlying operating system to provide various features such as memory allocation. But we do not have an underlying operating system to provide these services therefore we need to tell the compiler that we do not intend to use the std crate. We do this by adding `#![no_std]` to the beginning of our `main.rs` file. 

Now if we try to build it again we get a different error. Its a very similar error however, this time its complaining about the lack of the `core` and `compiler_builtins` crates. Unlike the `std` crate , we dont have an option of not using these crates as they are the entire foundation the Rust language is built on. The reason for this error is once again because Rust doesnt provide any precompiled binaries for the UEFI target.
However, the nightly version of Rust does give us the option of compiling these crates on our own, and thats exactly what we'll be doing.

In order to compile these crates ourselves, we must first acquire the source for libstd. The following command can be used to download the source:
```
rustup component add rust-src
```

Once thats done we need to tell Cargo to use the unstable `build-std` feature to build the `core` and `compiler_builtins` crates. we do that by using the `-Z build-std` flag. Passing this flag will cause Cargo to implicitly build `core, std, alloc, and proc_macro` crates , however we will be specifying the exact crates we want to build by using the following syntax : `-Z build-std=<crates to build>`. With all that done our new build command looks like the following:

```
cargo +nightly build -Z build-std=core,compiler_builtins --target=x86_64-unknown-uefi
```

Now we get two more new errors. The first is about the `println!` macro. This is to be expected as the  `println!` macro is provided by the `std` crate that we do not have access to. We can simply remove the macro from our code and the error should disappear. 

The second error we have is complaining about the lack of a panic_handler function. Looking at the Rust documentation, it says that this funciton defines the behaviour of the program when it panics. This fucntionality is normally provided by the `std` crate. This means we will have to define our own panic handler if we want to compile our program.

The panic handler function must have the following signature `fn(&PanicInfo) -> !`, and it needs to be marked with the `#[panic_handler]` attribute. Sounds easy enough so lets add it to our `main.rs` file. First we need to import the `PanicInfo` struct which is defined in the `core` crate. Adding ` use core::panic::PanicInfo;` to our file will take care of that. Next, we need to define a function with the above signature.

```Rust
#[panic_handler]
fn panic(_info: &PanicInfo) ->!{
	loop{}
}
```
For the time being we will simply loop if our program panics. We will later expand the functionality of this as we become able to interact with the UEFI firmware.
With all of the above code added to our `main.rs` file, it  should look like the following:

```Rust
#![no_std]
use core::panic::PanicInfo;

fn main() {
}

#[panic_handler]
fn panic(_info: &PanicInfo) ->!{
	loop{}
}
```

Now we have a new error. The compiler is complaining about the lack of the `start` `lang_item`. The start lang item is what defines the entry point of our program. When a program is executed, the first function that is run is not the main funciton but an entry point function that serves a few purposes, like setting up the stack for the program. The UEFI specification defines a function called `EFI_IMAGE_ENTRY_POINT` , this function is what the UEFI firmware expects to handover execution to. For UEFI application written in Rust the linker looks for a function named `efi_main` as the entry point. So lets add one. First we need to add `#![no_main]` to the top of our `main.rs` file to notify the compiler that our program will not have a standard main function. Next we will remove our existing main function and replace it with a function named efi_main. 

However, simply doing so will not make our UEFI app compile. There are two issues here. Firstly, the name of our function gets mangled by the compiler to a less human readable name that contains information that makes compilation easier. This is a problem as the linker will be looking for a function with the name `efi_main`. The second problem is the calling convention of the function. The calling convention of a function defines many things including how parameters are passed to it , whether they are stored on the stack or passed through registers etc. Its very important that our `efi_main` function uses the calling convention the UEFI firmware expects us to.

The first issue can be solved by adding the  `#[no_mangle]` attribute to our `efi_main` function. This tells the compiler to not mangle its name.
For the second issue, Rust allows us to specify which calling convention we want to use with our functions through the following syntax: 

```Rust
extern "[calling convention]" fn my_function()->!{}
```

Fortunately for us Rust supports the calling convention used by the UEFI firmware. In order to use it we need to first add `#![feature(abi_efiapi)]` to our `main.rs` file. Next we need to edit our definition of the `efi_main` function and add `pub extern "efiapi" fn efi_main` to it. With all these changes made our main.rs file should now look like the following:


```Rust
#![no_std]
#![no_main]
#![feature(abi_efiapi)]
use core::panic::PanicInfo;

#[no_mangle]
pub extern "efiapi" fn efi_main() {	
}

#[panic_handler]
fn panic(_info: &PanicInfo) ->!{
	loop{}
}
```

And Finally attempting to compile it results in no errors! :smiley:

That all for part one. We have managed to create a basic UEFI app that we can build upon and turn into our bootloader. In the next part I will use the UEFI firmware's built in functions to print to the screen. 

