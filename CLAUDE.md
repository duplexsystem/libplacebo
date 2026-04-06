# CLAUDE.md - libplacebo Development Guide

## Project Overview

libplacebo is a GPU-accelerated video/image rendering library (LGPLv2.1+), originally extracted from mpv. It provides a cross-platform GPU abstraction with backends for Vulkan, OpenGL, and Direct3D 11, along with high-quality image processing shaders for scaling, tone mapping, color management, and more.

**Version**: 7.362.0 (Major.API.Fix)

## Build System

**Meson** (>= 1.3.0), with C11 and C++20.

### Quick Build

```bash
meson setup build
ninja -C build
```

### Build with Tests

```bash
meson setup build -Dtests=true
ninja -C build
meson test -C build -v --num-processes=1
```

### Common Build Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `-Dtests=true` | bool | false | Build test suite |
| `-Dbench=true` | bool | false | Build benchmarks |
| `-Dfuzz=true` | bool | false | Build fuzzer binaries (requires AFL) |
| `-Dvulkan=enabled` | feature | auto | Vulkan backend |
| `-Dopengl=enabled` | feature | auto | OpenGL backend |
| `-Dd3d11=enabled` | feature | auto | D3D11 backend |
| `-Dshaderc=enabled` | feature | auto | libshaderc SPIR-V compiler |
| `-Dglslang=enabled` | feature | auto | glslang SPIR-V compiler |
| `-Dlcms=enabled` | feature | auto | ICC color management |
| `-Ddebug-abort=true` | bool | false | abort() on runtime errors |

### CI-style Release Build

```bash
meson setup build --buildtype release --werror -Dtests=true -Dshaderc=enabled -Dglslang=enabled
ninja -C build
```

## Repository Structure

```
src/                    # Main source code
  include/libplacebo/   # Public API headers (installed)
    shaders/            # Shader API headers
    utils/              # Utility API headers
  vulkan/               # Vulkan backend (~17 files)
  opengl/               # OpenGL backend (~14 files)
  d3d11/                # Direct3D 11 backend (~11 files)
  glsl/                 # SPIR-V compilation wrappers
  shaders/              # Shader implementations (color, sampling, film grain, etc.)
  utils/                # Utility implementations (frame_queue, upload, dolbyvision)
  tests/                # Test suite (~24 files)
    fuzz/               # Fuzzer test inputs
  *.c, *.h              # Core library (gpu, renderer, dispatch, colorspace, etc.)
3rdparty/               # Git submodules (Vulkan-Headers, jinja, glad, fast_float)
demos/                  # Example applications
docs/                   # MkDocs documentation
tools/                  # Build utility scripts
```

## Architecture (Tiered API)

The API is organized in progressive tiers of abstraction:

- **Tier 0**: Math primitives, color science (`colorspace.h`, `common.h`, `filters.h`, `tone_mapping.h`, `gamut_mapping.h`)
- **Tier 1**: GPU abstraction (`gpu.h`, `swapchain.h`, `vulkan.h`, `opengl.h`, `d3d11.h`)
- **Tier 2**: Shader generation (`shaders.h`, `shaders/*.h`)
- **Tier 3**: Shader dispatch (`dispatch.h`)
- **Tier 4**: High-level renderer (`renderer.h`, `options.h`, `utils/*.h`)

Each backend (Vulkan, OpenGL, D3D11) implements the same `pl_gpu` interface.

## Code Conventions

### Naming

- Public functions: `pl_` prefix (e.g., `pl_render_image`, `pl_log_create`)
- Public types: `pl_` prefix (e.g., `pl_gpu`, `pl_tex`, `pl_buf`)
- Enums/constants: `PL_` prefix (e.g., `PL_LOG_DEBUG`, `PL_COLOR_TRC_LINEAR`)
- Internal macros: `PL_` prefix (e.g., `PL_MIN`, `PL_MAX`, `PL_DEF`, `PL_PRIV`)

### Style

- **Indentation**: 4 spaces (no tabs)
- **Braces**: Opening brace on same line for control flow, next line for function definitions
- **Line length**: No strict limit, but generally kept reasonable (~80-100 chars)
- **Designated initializers**: Used extensively for struct initialization
- **Trailing commas**: Used in multi-line initializer lists

### API Patterns

- Opaque types via typedef: `typedef const struct pl_log_t *pl_log`
- Create/destroy pairs: `pl_*_create()` / `pl_*_destroy()` (destroy takes `**ptr` and NULLs it)
- Parameter structs with macro defaults: `pl_log_params(...)`
- Thread-safety documented per function
- API versioning via `PL_API_VER` linker glue macros

### File Headers

Every source file starts with the LGPLv2.1+ license header:

```c
/*
 * This file is part of libplacebo.
 *
 * libplacebo is free software; you can redistribute it and/or
 * modify it under the terms of the GNU Lesser General Public
 * License as published by the Free Software Foundation; either
 * version 2.1 of the License, or (at your option) any later version.
 * ...
 */
```

## Testing

Tests live in `src/tests/`. Key test files:

- `gpu_tests.c` - Comprehensive GPU backend tests (shared across all backends)
- `vulkan.c`, `opengl_surfaceless.c`, `d3d11.c` - Backend-specific test runners
- `colorspace.c`, `tone_mapping.c` - Color science tests
- `bench.c` - Benchmarks

Run tests:
```bash
meson test -C build -v --num-processes=1
```

Run with timeout multiplier (for CI/slow systems):
```bash
meson test -C build -t 5 -v --num-processes=1
```

## CI Pipeline

Primary CI is GitLab (`.gitlab-ci.yml`) with stages: compile -> test -> sanitize.

Key jobs:
- **linux/static/aarch64/macos**: Compilation on multiple platforms with `--werror`
- **win32/win64**: Cross-compilation via MinGW
- **scan**: Clang static analyzer (`scan-build`)
- **llvmpipe**: Software renderer tests with shaderc+glslang
- **gpu**: Hardware GPU tests with code coverage
- **sanitize**: ASan + UBSan (leak detection disabled due to Mesa driver leaks)

## Dependencies

**Required**: Meson >= 1.3.0, C11 compiler, C++20 compiler

**Bundled** (git submodules in `3rdparty/`): Vulkan-Headers, Jinja2, glad, fast_float, MarkupSafe

**Optional external**: libvulkan, libGL/libEGL, lcms2 (>= 2.9), libdovi (>= 1.6.7), libxxhash, libunwind, libshaderc, glslang

## Key Things to Know

- Submodules are required: use `git submodule update --init --recursive` after cloning
- The Vulkan backend uses code generation from Vulkan XML registry via Jinja2 templates
- GLSL shaders in `src/shaders/` use a custom preprocessing step during build
- The `renderer.c` file is large and is the main high-level rendering pipeline
- `gpu_tests.c` (~68KB) is the central GPU test framework used by all backend tests
- Windows builds use MinGW cross-compilation in CI
- Always build with `--werror` to match CI expectations
