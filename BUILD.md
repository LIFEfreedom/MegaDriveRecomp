# Build Guide

## Requirements

### Compiler

**Clang** with `libc++` is required. GCC is not supported — the CMake config explicitly sets `clang++` and links against `libc++`/`libc++abi`.

- Verified: **Clang 18** (C++26 support is needed for the main project)
- **Clang 21 does not work.** The pinned `fmt` 11.0.2 fails there — `FMT_STRING` stops
  being a constant expression (fixed upstream in fmt 11.1). Stay on Clang 18, or bump
  `fmt` to >= 11.1 and `spdlog` to >= 1.15 in `src/CMakeLists.txt`.

### System packages

**Linux (Ubuntu 24.04):**
```sh
sudo apt-get install \
  cmake ninja-build clang libc++-dev libc++abi-dev \
  git pkg-config python3-jinja2 \
  libwayland-dev mesa-common-dev xorg-dev libxkbcommon-dev
```

**Docker** — the included `Dockerfile` builds that environment (dev tools included). It does
not copy the sources; mount the repository into the container:
```sh
docker build -t segacxx-dev .
```
```sh
docker run --rm -it -v "$PWD:/work" -w /work segacxx-dev
```

**Windows:** not supported as-is. `src/CMakeLists.txt` passes GNU-ld style link flags
(`-Wl,-Bstatic -lc++ -lc++abi -Wl,-Bdynamic`), which a Clang targeting MSVC does not accept,
and libc++ is not shipped for that target. Use WSL or the Docker image above.

**macOS:**
```sh
brew install cmake ninja llvm
```
Then set `CC`/`CXX` to the Homebrew Clang, not Apple Clang:
```sh
export CC=$(brew --prefix llvm)/bin/clang
export CXX=$(brew --prefix llvm)/bin/clang++
```

---

## External dependencies

All external libraries are **downloaded automatically** by CMake via `FetchContent` — no manual installation needed.

| Library | Version | Purpose |
|---------|---------|---------|
| [fmt](https://github.com/fmtlib/fmt) | 11.0.2 | String formatting |
| [spdlog](https://github.com/gabime/spdlog) | 1.14.1 | Logging |
| [GLFW](https://github.com/glfw/glfw) | 3.4 | Window and input |
| [Dear ImGui](https://github.com/ocornut/imgui) | 1.91.4 | Debug UI |
| [GLAD](https://github.com/Dav1dde/glad) | v2.0.8 | OpenGL loader |
| [nlohmann/json](https://github.com/nlohmann/json) | 3.11.3 | JSON parsing |
| [magic_enum](https://github.com/Neargye/magic_enum) | 0.9.6 | Compile-time enum reflection |
| [stb](https://github.com/nothings/stb) | latest | Image read/write |
| [miniaudio](https://github.com/mackron/miniaudio) | latest | Audio output |
| [Nuked-OPN2](https://github.com/nukeykt/Nuked-OPN2) | latest | YM2612 FM emulation |

---

## Build

```sh
cmake -S src -B build -G Ninja -DCMAKE_EXE_LINKER_FLAGS=-L/usr/lib/llvm-18/lib
```
```sh
cmake --build build
```

The `-L` is required on Ubuntu: `libc++abi.a` is installed only under `/usr/lib/llvm-18/lib`,
so the `-Wl,-Bstatic -lc++abi` in `src/CMakeLists.txt` cannot find it. Without it CMake's own
probe compilations fail and configure aborts on `Could NOT find Threads`. Adjust the path if
your libc++ lives elsewhere.

Binaries are placed in `build/bin/`:
- `sega_emulator` — Sega Genesis emulator
- `smd_recomp` — static recompiler tool
- `m68k_emulator` — standalone M68K emulator
- `m68k_test` — M68K instruction tests
- `sega_video_test` — video rendering test

---

## Debug build

Edit `src/CMakeLists.txt` and swap the build type lines:

```cmake
# set(CMAKE_BUILD_TYPE Release)   ← comment out
set(CMAKE_BUILD_TYPE Debug)       ← uncomment
```

Also uncomment the sanitizer flags if needed:
```cmake
add_compile_options(-fsanitize=undefined,address)
add_link_options(-fsanitize=undefined,address)
```

---

## Running the emulator

```sh
./build/bin/sega_emulator/sega_emulator sonic.gen
```

Exactly one argument — the path to the ROM. Controls: arrow keys for the D-pad, `A`/`S`/`D`
for buttons A/B/C, `Enter` for Start. Gamepads are supported as well.

---

## Running the recompiler

`smd_recomp` takes a TOML config — see `example.toml` for the format, including the
`[[switch_tables]]` and `[[functions]]` entries needed to resolve indirect jumps.

```sh
./build/bin/smd_recomp/smd_recomp example.toml
```

It writes into the configured output directory:
- `func_table.h` / `func_table.cpp` — forward declarations and the address → function dispatch table
- one `.cpp` per function (`split_files = true`), or a single `all_functions.cpp`

**The generated code is not buildable on its own.** It includes `recomp_runtime.h` and every
function takes a `RecompContext&`. That runtime — and the `output/` CMake project that would
link the recompiled game together with a ROM — are not part of this repository yet and have to
be supplied by hand.

---

## Tests

`m68k_test` expects the [SingleStepTests 680x0](https://github.com/SingleStepTests/680x0) JSON
suite at the hardcoded path `/usr/src/680x0/68000/v1/` (see `src/bin/m68k_test/main.cpp`); it
aborts if the directory is missing.

For `sega_video_test`, see `src/bin/sega_video_test/README.md`.
