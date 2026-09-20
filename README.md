<!--
Copyright 2026 林博仁(Buo-ren Lin) <buo.ren.lin@gmail.com>
SPDX-License-Identifier: MIT
-->

# Bug reproduction of the Whisper.cpp armhf build virtual memory exhaustion problem

This document describes how to reproduce the `virtual memory exhausted: Cannot allocate memory` compiler crash encountered when building `whisper.cpp` with the Vulkan backend (`-DGGML_VULKAN=ON`) on 32-bit ARM (`armhf`), using QEMU user-mode emulation on an AMD64 (x86_64) host.

<https://gitlab.com/brlin/bug-repro-whisper.cpp-armhf-build-virtual-memory-exhaustion>

## 1. Problem Description & Root Cause

When building `whisper.cpp` with `-DGGML_VULKAN=ON` on 32-bit architectures such as `armhf`, the compilation of `ggml-vulkan` fails with:

```text
[ 70%] Building CXX object ggml/src/ggml-vulkan/CMakeFiles/ggml-vulkan.dir/mul_mm.comp.cpp.o
virtual memory exhausted: Cannot allocate memory
make[2]: *** [ggml/src/ggml-vulkan/CMakeFiles/ggml-vulkan.dir/build.make:...: ggml/src/ggml-vulkan/CMakeFiles/ggml-vulkan.dir/mul_mm.comp.cpp.o] Error 1
```

### Root Cause

The shader code generator (`ggml/src/ggml-vulkan/vulkan-shaders/vulkan-shaders-gen.cpp`) outputs compiled SPIR-V shader bytecode as arrays of comma-separated hex numbers:

```cpp
const unsigned char mul_mm_xxx_data[1600] = {
    0x03, 0x02, 0x23, 0x00, ...
};
```

In `mul_mm.comp.cpp`, there are over **1,000 shaders**, generating over **30,000,000 numeric byte literals** across 2.54 million lines of code.

When GCC (`cc1plus`) parses an initializer list of numbers (`{ 0x..., 0x... }`), its C++ front-end creates an `INTEGER_CST` AST node and a `CONSTRUCTOR` element node for every single byte constant. Constructing this AST requires **over 3.4 GiB of heap memory** during parsing alone.

Because 32-bit Linux architectures enforce a **hard 3 GiB user-space process virtual address space limit** (`CONFIG_VMSPLIT_3G`), `g++` runs out of virtual address space and terminates with `virtual memory exhausted: Cannot allocate memory` during the AST parsing phase, before optimization or code generation can even begin.

## 2. Prerequisites on AMD64 Host

Install QEMU user-mode emulation and Docker (or Podman) on the AMD64 host:

```bash
sudo apt update
sudo apt install -y qemu-user-static docker.io
```

Drop the `docker.io` package if you already have another compatible container runtime installed.

Verify that ARMv7 (`armhf`) user-space emulation is active:

```bash
docker run --rm --platform linux/arm/v7 -v "${PWD}:/pwd" ubuntu:24.04 uname -m
# Expected output: armv7l
```

## 3. Step-by-Step Reproduction Instructions

### Step 1: Start an emulated `armhf` container

From the root of this repository:

```bash
docker run --rm -it \
    --platform linux/arm/v7 \
    -v "$(pwd)":/workspace \
    ubuntu:24.04 bash
```

### Step 2: Install build dependencies inside the container

```bash
apt update && apt install -y \
    build-essential \
    cmake \
    git \
    glslc \
    libvulkan-dev \
    pkg-config \
    spirv-headers
```

### Step 3: Clone `whisper.cpp` at release `v1.9.4`

```bash
git clone https://github.com/ggerganov/whisper.cpp.git /tmp/whisper.cpp
cd /tmp/whisper.cpp
git checkout v1.9.4
```

### Step 4: Apply upstream 32-bit Vulkan-Hpp compatibility fix

Tagged release `v1.9.4` requires upstream commit `69fcec3bf11e54825edda9ea2c11e33f5cb9652c` (addressing issue [#4036](https://github.com/ggml-org/whisper.cpp/issues/4036) / [patch 0001](file:///workspace/snap/local/patches/0001-Fix-Vulkan-Hpp-handle-usage-on-32-bit-targets.patch)), which fixes `vk::Buffer` handle types so compilation does not fail prematurely before reaching the shader compilation stage:

```bash
git -c user.email="dummy@example.com" -c user.name="Dummy User" cherry-pick 69fcec3bf11e54825edda9ea2c11e33f5cb9652c
```

### Step 5: Configure and trigger the build

```bash
cmake -B build -DGGML_VULKAN=ON
cmake --build build --target ggml-vulkan -j1
```

### Expected Failure Result

When `g++` compiles `mul_mm.comp.cpp.o`, virtual memory is exhausted:

```text
[ 70%] Building CXX object ggml/src/ggml-vulkan/CMakeFiles/ggml-vulkan.dir/mul_mm.comp.cpp.o
virtual memory exhausted: Cannot allocate memory
make[2]: *** [ggml/src/ggml-vulkan/CMakeFiles/ggml-vulkan.dir/build.make:...: ggml/src/ggml-vulkan/CMakeFiles/ggml-vulkan.dir/mul_mm.comp.cpp.o] Error 1
```

## 4. Verifying the Fix

### Step 6: Apply the string literal patch

From within `/tmp/whisper.cpp` inside the container:

```bash
git apply </pwd/Embed-Vulkan-shaders-as-string-literals-to-reduce-compilation-memory.patch
```

This changes the generator to output concatenated octal string literals (`alignas(4) const unsigned char name_data[] = "\003\002\043...";`) instead of numeric lists. A string literal is parsed as a single `STRING_CST` AST node, reducing AST node allocation by >99.9%.

### Step 7: Re-generate the shaders and rebuild

```bash
# Rebuild vulkan-shaders-gen and remove the old generated source file
cmake --build build --target vulkan-shaders-gen
rm -f build/ggml/src/ggml-vulkan/mul_mm.comp.cpp

# Rebuild ggml-vulkan
cmake --build build --target ggml-vulkan
```

### Expected Success Result

* `mul_mm.comp.cpp.o` compiles in **under 2 seconds** using only **~530 MiB** peak RSS (down from >3.5 GiB crash).
* `libggml-vulkan.so` links and completes with 100% success.
