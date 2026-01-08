# alloc-no-stdlib

[![Rust Version](https://img.shields.io/badge/rust-1.8%2B-blue.svg)](https://www.rust-lang.org)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-blue.svg)](LICENSE)
[![Build Status](https://travis-ci.org/dropbox/rust-alloc-no-stdlib.svg?branch=master)](https://travis-ci.org/dropbox/rust-alloc-no-stdlib)

A dynamic memory allocation framework for Rust `#![no_std]` environments.

## Overview

The `alloc-no-stdlib` crate provides a flexible memory allocation solution for Rust programs that cannot or do not want to use the standard library. This framework enables dynamic memory allocation in `#![no_std]` modules where no standard allocation method exists.

### Problem Statement

In `#![no_std]` environments, Rust programs face significant challenges:
- No access to the standard `Box`, `Vec`, or other heap-allocated types
- Limited dynamic memory allocation options
- Need for custom allocators that work without the standard library
- Requirement for safe memory management in constrained environments

This framework solves these problems by providing multiple allocation strategies that can work entirely without the standard library, while also being compatible with `std` when available.

## Features

- **Multiple Allocation Strategies**: Choose from stack-based, calloc-linked, or global variable allocations
- **No Standard Library Required**: Works in `#![no_std]` environments
- **stdlib Compatible**: Can integrate seamlessly with `std` when available
- **Zero-Copy Design**: Efficient memory management with minimal overhead
- **Application Isolation**: Security features using seccomp for sandboxing
- **Flexible**: Can be used with custom allocators, stack memory, or standard `Box<>`
- **Safe Abstractions**: Provides safe wrappers around unsafe operations

## Requirements

- **Rust**: 1.8 or later
- **Optional**: C standard library (for calloc/malloc/free linking)

## Installation

Add this to your `Cargo.toml`:

```toml
[dependencies]
alloc-no-stdlib = "2.0"
```

For `#![no_std]` projects, ensure you're not using the default features that require std.

## Allocation Methods

The framework provides three primary allocation strategies, each with different trade-offs:

### 1. Stack Allocation

Allocate memory directly on the stack using compile-time arrays.

**Pros:**
- ✅ Works without any standard library
- ✅ No external dependencies (no libc required)
- ✅ Deterministic performance
- ✅ Safe from heap fragmentation

**Cons:**
- ❌ Limited by stack size (typically a few megabytes)
- ❌ Affected by `ulimit` stack depth limits
- ❌ Cannot grow dynamically
- ❌ Risk of stack overflow with large allocations

**Example:**

```rust
#[macro_use]
extern crate alloc_no_stdlib;
use alloc_no_stdlib::{Allocator, SliceWrapper};

// Declare a stack-based allocator with 16 freelist entries
declare_stack_allocator_struct!(StackAllocator16, 16, stack);

fn main() {
    // Create a 1MB buffer on the stack
    let mut stack_buffer = define_allocator_memory_pool!(
        16, u8, [0; 1024 * 1024], stack
    );
    
    let mut allocator = StackAllocator16::<u8>::new_allocator(
        &mut stack_buffer, 
        alloc_no_stdlib::bzero
    );
    
    // Allocate memory
    let mut data = allocator.alloc_cell(1000);
    data[0] = 42;
    
    // Free memory (important to prevent leaks!)
    allocator.free_cell(data);
}
```

### 2. Calloc/Malloc Linking

Link against C standard library's `calloc` or `malloc` for dynamic heap allocation.

**Pros:**
- ✅ Large allocation sizes supported
- ✅ Dynamic growth capability
- ✅ No stack size limitations
- ✅ Pre-zeroed memory with calloc

**Cons:**
- ❌ Requires linking with libc
- ❌ Unsafe FFI boundary
- ❌ Must manually call `free_cell` to prevent leaks
- ❌ Less portable

**Example:**

```rust
#[macro_use]
extern crate alloc_no_stdlib;
use alloc_no_stdlib::Allocator;

extern "C" {
    fn calloc(n_elem: usize, el_size: usize) -> *mut u8;
    fn free(ptr: *mut u8);
}

declare_stack_allocator_struct!(CallocAllocator4, 4, calloc);

fn main() {
    // Allocate 200MB using calloc
    let mut global_buffer = unsafe {
        define_allocator_memory_pool!(
            4, u8, [0; 1024 * 1024 * 200], calloc
        )
    };
    
    let mut allocator = CallocAllocator4::<u8>::new_allocator(
        global_buffer.data,
        alloc_no_stdlib::bzero
    );
    
    let mut data = allocator.alloc_cell(10000);
    data[0] = 123;
    allocator.free_cell(data);
    
    // Buffer automatically freed when dropped
}
```

> ⚠️ **Warning**: Calloc allocation involves unsafe code and FFI. Memory leaks will occur if `free_cell` is not called.

### 3. Global Mutable Variable

Use a global mutable static variable for pre-allocated memory pools.

**Pros:**
- ✅ Fixed memory footprint
- ✅ Predictable allocation behavior
- ✅ Good for embedded systems
- ✅ No runtime allocation overhead

**Cons:**
- ❌ Requires unsafe code
- ❌ Fixed maximum size at compile time
- ❌ Global mutable state (not thread-safe without synchronization)
- ❌ Must manually manage lifetimes

**Example:**

```rust
#[macro_use]
extern crate alloc_no_stdlib;
use alloc_no_stdlib::Allocator;

// Define global memory pool
define_allocator_memory_pool!(4, u8, [0; 65536], global, GlobalBuffer);

fn main() {
    declare_stack_allocator_struct!(GlobalAllocator, 0, global);
    
    let mut allocator = GlobalAllocator::<u8>::new_allocator(
        alloc_no_stdlib::bzero
    );
    
    unsafe {
        bind_global_buffers_to_allocator!(allocator, GlobalBuffer, u8);
    }
    
    let mut data = allocator.alloc_cell(1024);
    data[0] = 99;
    allocator.free_cell(data);
}
```

> ⚠️ **Warning**: Global variables involve unsafe code and require careful lifetime management.

## Allocation Method Comparison

| Feature | Stack | Calloc/Malloc | Global Variable |
|---------|-------|---------------|-----------------|
| **Max Size** | ~Few MB | GBs | Fixed at compile-time |
| **Requires libc** | No | Yes | No |
| **Safety** | Safest | Unsafe FFI | Unsafe globals |
| **Performance** | Fastest | Fast | Fastest |
| **Memory Leaks** | Possible if not freed | Possible if not freed | Possible if not freed |
| **Portability** | Highest | Medium | High |
| **Use Case** | Small allocations | Large dynamic needs | Fixed embedded systems |

## stdlib Integration

When linked with the standard library, applications can leverage Rust's standard `Box<T>` type for automatic memory management:

```rust
extern crate alloc_no_stdlib;

// When using std, you can pass custom allocators but also use Box
fn use_with_std() {
    // Box provides automatic cleanup
    let data = Box::new([0u8; 1024]);
    // Automatically freed when data goes out of scope
}
```

The framework allows libraries that depend on `alloc-no-stdlib` to work seamlessly in both `#![no_std]` and `std` environments. When `std` is available, `Box<>` provides automatic memory deallocation, eliminating manual `free_cell` calls.

## Security: Application Isolation with seccomp

The library provides security features for isolating Rust applications that require dynamic allocations:

1. **Pre-allocation**: Allocate the maximum required memory upfront using `calloc`
2. **System Call Filtering**: Use `seccomp` (Linux) to prevent future system calls
3. **Sandboxing**: Restrict the application's capabilities after initialization

This approach is useful for:
- Sandboxed plugins
- Security-critical applications
- Untrusted code execution
- Defense-in-depth strategies

**Example seccomp usage pattern:**

```rust
// 1. Pre-allocate maximum memory
let mut buffer = unsafe {
    define_allocator_memory_pool!(4, u8, [0; 1024 * 1024 * 100], calloc)
};

// 2. Set up allocator
let mut allocator = CallocAllocator4::<u8>::new_allocator(
    buffer.data,
    alloc_no_stdlib::bzero
);

// 3. Enable seccomp filtering (platform-specific)
// After this point, no new syscalls like mmap, brk, etc.
#[cfg(target_os = "linux")]
unsafe {
    // Apply seccomp filters here
}

// 4. Run isolated code with pre-allocated memory
// Code can only use pre-allocated memory pool
```

## Quick Start

Here's a complete example showing basic usage:

```rust
#[macro_use]
extern crate alloc_no_stdlib;
use alloc_no_stdlib::Allocator;

// Declare allocator type
declare_stack_allocator_struct!(StackAlloc, 16, stack);

fn main() {
    // Create memory pool
    let mut buffer = define_allocator_memory_pool!(
        16, u8, [0; 8192], stack
    );
    
    // Initialize allocator
    let mut allocator = StackAlloc::<u8>::new_allocator(
        &mut buffer,
        alloc_no_stdlib::bzero
    );
    
    // Allocate and use memory
    let mut data1 = allocator.alloc_cell(100);
    data1[0] = 1;
    data1[99] = 255;
    
    let mut data2 = allocator.alloc_cell(200);
    data2[0] = 2;
    
    // Free memory (order doesn't matter)
    allocator.free_cell(data1);
    allocator.free_cell(data2);
}
```

## API Documentation

The framework provides the following key traits and types:

### Core Trait: `Allocator<T>`

```rust
pub trait Allocator<T> {
    type AllocatedMemory: AllocatedSlice<T>;
    
    fn alloc_cell(&mut self, len: usize) -> Self::AllocatedMemory;
    fn free_cell(&mut self, data: Self::AllocatedMemory);
}
```

### Allocator Types

- **`StackAllocator<T>`**: Stack-based memory allocator
- **`CallocBackingStore<T>`**: C library-backed allocator
- **`AllocatedStackMemory<T>`**: Stack-allocated memory handle

### Helper Functions

- **`bzero<T: Default>(data: &mut [T])`**: Zero-initialize memory
- **`uninitialized<T>(data: &mut [T])`**: Leave memory uninitialized (faster)

### Macros

- **`declare_stack_allocator_struct!`**: Declare allocator struct types
- **`define_allocator_memory_pool!`**: Create memory pool storage
- **`bind_global_buffers_to_allocator!`**: Connect global buffers to allocators

For detailed API documentation, see the [source code examples](tests/lib.rs).

## Safety Considerations

### Memory Leaks

> ⚠️ **Critical**: Always call `free_cell` for every `alloc_cell`. Memory will leak if not explicitly freed.

```rust
// ❌ BAD: Memory leak
let data = allocator.alloc_cell(100);
// data dropped without free_cell

// ✅ GOOD: Explicit cleanup
let data = allocator.alloc_cell(100);
allocator.free_cell(data);
```

### Use-After-Free

The type system helps prevent use-after-free, but be careful:

```rust
let mut x = allocator.alloc_cell(10);
x[0] = 5;
allocator.free_cell(x);
// x is moved and cannot be used anymore ✅
```

### Stack Overflow

Stack allocations can overflow if too large:

```rust
// ❌ Dangerous: May exceed stack size
let mut buffer = define_allocator_memory_pool!(
    16, u8, [0; 1024 * 1024 * 100], stack  // 100MB on stack!
);

// ✅ Better: Use calloc for large allocations
let mut buffer = unsafe {
    define_allocator_memory_pool!(
        16, u8, [0; 1024 * 1024 * 100], calloc
    )
};
```

### Thread Safety

The allocators are **not thread-safe** by default. For concurrent access:
- Use separate allocators per thread
- Wrap allocators in `Mutex` or other synchronization primitives
- Pre-allocate per-thread memory pools

## Limitations

- **Manual Memory Management**: Requires explicit `free_cell` calls
- **No Thread Safety**: Allocators are not `Send` or `Sync` by default
- **Fixed Pool Sizes**: Stack and global allocators have fixed capacity
- **No Growing**: Pre-allocated pools cannot expand
- **OOM Panics**: Allocation failures cause panics (no graceful degradation)
- **No Alignment Control**: Uses default alignment for types
- **Limited Debugging**: No built-in leak detection or memory profiling

## Performance Considerations

- **Stack allocations** are fastest but most limited
- **Calloc allocations** have syscall overhead but support large sizes
- Use `uninitialized` instead of `bzero` when zeroing isn't needed
- Freelist size affects allocation speed (larger = faster but more memory)
- Pre-allocate maximum memory upfront for predictable performance

## Examples

See the [`example.rs`](src/bin/example.rs) file for more comprehensive usage examples, including:
- Multiple allocator types in the same program
- Mixed stack and calloc allocations
- Custom type allocations (not just `u8`)
- Integration with stdlib's `Box`

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes with tests
4. Ensure all tests pass (`cargo test`)
5. Commit your changes (`git commit -m 'Add amazing feature'`)
6. Push to the branch (`git push origin feature/amazing-feature`)
7. Open a Pull Request

Please follow Rust coding conventions and include documentation for new features.

## License

This project is licensed under the **BSD 3-Clause License**. See the [LICENSE](LICENSE) file for details.

### BSD 3-Clause License Summary

- ✅ Commercial use allowed
- ✅ Modification allowed
- ✅ Distribution allowed
- ✅ Private use allowed
- ❌ No liability
- ❌ No warranty

---

**Minimum Rust Version**: 1.8+  
**Maintained by**: Daniel Reiter Horn  
**Repository**: https://github.com/dropbox/rust-alloc-no-stdlib
