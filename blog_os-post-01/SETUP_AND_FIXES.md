# Blog OS - Setup, Bugs Fixed, and Running Instructions

## Overview
This is a bare-metal operating system kernel written in Rust that runs without the standard library (`no_std`). This document details the issues encountered, fixes applied, and instructions for building and running the OS.

---

## Bugs Fixed

### 1. **Missing Custom Target Specification**
**Issue:** The project attempted to build for a standard target (x86_64-pc-windows-msvc) which includes the standard library and OS-specific features. A bare-metal OS requires a custom target.

**Fix:** Created `x86_64-blog_os.json` with the following configuration:
```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128",
  "arch": "x86_64",
  "target-endian": "little",
  "target-pointer-width": 64,
  "target-c-int-width": 32,
  "os": "none",
  "executables": true,
  "linker-flavor": "ld.lld",
  "linker": "rust-lld",
  "panic-strategy": "abort",
  "disable-redzone": true
}
```

**Key points:**
- `"os": "none"` indicates no operating system
- `"panic-strategy": "abort"` matches the Cargo.toml configuration
- `"disable-redzone"` is required for kernel-level code
- Removed SSE/MMX feature flags that caused ABI conflicts

### 2. **Missing Cargo Build Configuration**
**Issue:** Cargo didn't know to use the custom target or build the `core` library from source.

**Fix:** Created `.cargo/config.toml`:
```toml
[build]
target = "x86_64-blog_os.json"

[unstable]
build-std = ["core", "compiler_builtins"]
build-std-features = ["compiler-builtins-mem"]

[target.'cfg(target_os = "none")']
runner = "bootimage runner"
```

**Explanation:**
- `target` specifies our custom target file
- `build-std` enables building `core` and `compiler_builtins` from source
- This is necessary because pre-compiled standard library components don't exist for custom targets

### 3. **Unsafe Attribute Syntax for Nightly Compiler**
**Issue:** The original code used `#[no_mangle]` which is now considered an unsafe attribute in newer nightly Rust versions.

**Original code:**
```rust
#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

**Fixed code:**
```rust
#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

**Reason:** The `no_mangle` attribute can cause undefined behavior if multiple symbols have the same name, so the nightly compiler requires it to be marked as `unsafe`.

### 4. **Missing rust-src Component**
**Issue:** Building custom targets requires the Rust standard library source code.

**Fix:** Installed rust-src for the nightly toolchain:
```bash
rustup component add rust-src --toolchain nightly-x86_64-pc-windows-msvc
```

### 5. **Incompatible CPU Features**
**Issue:** Initial target specification included `-mmx,-sse,+soft-float` features which are incompatible with x86_64 ABI requirements.

**Error encountered:**
```
SSE register return with SSE disabled
target feature `soft-float` is incompatible with the ABI
```

**Fix:** Removed the incompatible feature flags from the target specification.

---

## Prerequisites

### Required Tools
1. **Rust Toolchain**
   - Nightly version required (for `build-std` feature)
   - Install: `rustup toolchain install nightly`

2. **rust-src Component**
   - Required for building core library
   - Install: `rustup component add rust-src --toolchain nightly`

3. **(Optional) QEMU** - For running the OS
   - Download from: https://www.qemu.org/download/
   - Required for testing the kernel

4. **(Optional) bootimage** - For creating bootable disk images
   - Install: `cargo install bootimage`

---

## Building the Project

### Basic Build
```bash
cargo +nightly build
```

### Release Build (Optimized)
```bash
cargo +nightly build --release
```

### Output Location
- Debug build: `target/x86_64-blog_os/debug/blog_os`
- Release build: `target/x86_64-blog_os/release/blog_os`

---

## Running the OS

⚠️ **Important:** This is a bare-metal kernel and **cannot** be run as a regular executable. It requires either:
1. A virtual machine (QEMU)
2. A bootable image
3. Real hardware (advanced)

### Method 1: Using QEMU (Direct Kernel Boot)
```bash
qemu-system-x86_64 -kernel target/x86_64-blog_os/debug/blog_os
```

### Method 2: Create Bootable Image with bootimage
1. Install bootimage:
   ```bash
   cargo install bootimage
   ```

2. Add llvm-tools-preview:
   ```bash
   rustup component add llvm-tools-preview
   ```

3. Create bootable image:
   ```bash
   cargo bootimage
   ```

4. Run with QEMU:
   ```bash
   qemu-system-x86_64 -drive format=raw,file=target/x86_64-blog_os/debug/bootimage-blog_os.bin
   ```

### Method 3: Write to USB Drive (Real Hardware)
⚠️ **WARNING:** This will erase all data on the target drive!

**Linux/macOS:**
```bash
dd if=target/x86_64-blog_os/debug/bootimage-blog_os.bin of=/dev/sdX && sync
```

**Windows:**
Use a tool like Rufus or Win32DiskImager to write the `.bin` file to a USB drive.

---

## Current Functionality

The current implementation provides:
- A minimal bare-metal entry point (`_start`)
- Basic panic handler
- Infinite loop (CPU halt state)

**Note:** This is a minimal kernel that doesn't do anything visible yet. To see output, you would need to:
1. Add VGA buffer text output
2. Implement serial port communication
3. Or add BIOS/UEFI print functionality

---

## Troubleshooting

### "can't find crate for `core`"
**Solution:** Install rust-src component:
```bash
rustup component add rust-src --toolchain nightly
```

### "unsafe attribute used without unsafe"
**Solution:** Ensure you're using the nightly toolchain and `#[unsafe(no_mangle)]` syntax.

### "SSE register return with SSE disabled"
**Solution:** Remove SSE-related feature flags from `x86_64-blog_os.json`.

### Build is slow
**Solution:** The first build compiles `core` and `compiler_builtins` from source, which takes time. Subsequent builds will be faster.

---

## Next Steps

To extend this OS, consider adding:
1. **VGA Text Buffer** - Display text on screen
2. **Serial Port Output** - Debug logging
3. **Interrupt Handling** - CPU exceptions and hardware interrupts
4. **Memory Management** - Paging and heap allocation
5. **Keyboard Input** - User interaction

---

## Resources

- [Rust OS Development Tutorial](https://os.phil-opp.com/)
- [OSDev Wiki](https://wiki.osdev.org/)
- [Rust Embedded Book](https://rust-embedded.github.io/book/)
- [Writing an OS in Rust (Blog Series)](https://os.phil-opp.com/)

---

## License

This project uses the same licenses as specified in LICENSE-APACHE and LICENSE-MIT.
