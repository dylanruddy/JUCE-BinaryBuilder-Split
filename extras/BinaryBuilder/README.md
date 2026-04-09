# BinaryBuilder

BinaryBuilder is a command-line utility (part of the JUCE extras) that encodes
any collection of binary files into C++ source files so they can be compiled
directly into your application or plug-in executable — no file-system access
required at runtime.

---

## Building BinaryBuilder

Xcode and Visual Studio project files are located in
[`Builds/`](Builds/).  You can also build with CMake from the repository root:

```sh
cmake . -B cmake-build -DJUCE_BUILD_EXTRAS=ON
cmake --build cmake-build --target BinaryBuilder
```

The resulting binary is placed in the build output directory for your platform
(e.g. `Builds/MacOSX/build/Debug/BinaryBuilder` on macOS).

---

## Basic Usage

```
BinaryBuilder  <sourcedirectory>  <targetdirectory>  <classname>  [--split]  [wildcard]
```

| Argument           | Required | Description |
|--------------------|----------|-------------|
| `sourcedirectory`  | ✅        | Folder containing the files to embed |
| `targetdirectory`  | ✅        | Folder where the generated files will be written |
| `classname`        | ✅        | C++ namespace / file-name prefix for the generated code |
| `--split`          | ❌        | Generate one `.cpp` per source file instead of a single combined `.cpp` |
| `wildcard`         | ❌        | Glob pattern to filter source files (default: `*`) |

`--split` and the wildcard pattern are **order-independent** — you can pass
them in either order after `classname`.

---

## Standard Mode (no `--split`)

```sh
BinaryBuilder "/path/to/assets" "/path/to/output" "BinaryData"
```

This produces **two files** in the target directory:

```
BinaryData.h      ← shared header (extern declarations for every asset)
BinaryData.cpp    ← all asset data combined into a single translation unit
```

Add both files to your project, `#include "BinaryData.h"` wherever you need
them, and access assets like this:

```cpp
// Play a sound that was embedded from "click.wav"
auto* stream = new MemoryInputStream (BinaryData::click_wav,
                                      BinaryData::click_wavSize, false);
```

---

## Split Mode (`--split`)

For large asset sets, compiling a single `BinaryData.cpp` can be very slow and
memory-intensive because every change forces a full recompile of all embedded
data.  The `--split` flag solves this by writing **one `.cpp` file per source
asset**.

```sh
BinaryBuilder "/path/to/assets" "/path/to/output" "BinaryData" --split
```

### Example — Dylan's Desktop workflow

```sh
"/Users/dylanruddy/Documents/Programming/JUCE-BinaryBuilder-Split/extras/BinaryBuilder/Builds/MacOSX/build/Debug/BinaryBuilder" \
    "/Users/dylanruddy/Desktop/TEST" \
    "/Users/dylanruddy/Desktop" \
    "BinaryData" \
    --split
```

### Output

Given a `TEST/` folder containing `click.wav`, `logo.png`, and
`fonts/Roboto.ttf`, split mode produces:

```
BinaryData.h                        ← shared header (unchanged from standard mode)
BinaryData_click_wav.cpp            ← data for click.wav
BinaryData_logo_png.cpp             ← data for logo.png
BinaryData_fonts_Roboto_ttf.cpp     ← data for fonts/Roboto.ttf
```

Each `.cpp` file:
- is fully self-contained (includes its own file header and `#include "BinaryData.h"`)
- uses a safe name derived from the **relative path** of the source file
  (path separators and dots are replaced with `_`) so files with the same
  name in different sub-directories never collide

### Adding split files to your project

Add `BinaryData.h` and **all** of the generated `BinaryData_*.cpp` files to
your project.  The C++ API is identical to standard mode — consumers only ever
`#include "BinaryData.h"` and call `BinaryData::my_asset`.

> **Tip (CMake):** use a glob to pick up the generated files automatically:
> ```cmake
> file(GLOB BINARY_DATA_SOURCES "${OUTPUT_DIR}/BinaryData*.cpp")
> target_sources(MyPlugin PRIVATE ${BINARY_DATA_SOURCES})
> ```

---

## Wildcard Filtering

You can limit which files are embedded by passing a glob pattern as the last
argument (works in both standard and split mode):

```sh
# Embed only .wav files
BinaryBuilder "/path/to/assets" "/path/to/output" "BinaryData" "*.wav"

# Embed only .wav files, one .cpp per file
BinaryBuilder "/path/to/assets" "/path/to/output" "BinaryData" --split "*.wav"
```

---

## Sub-directory `#ifdef` Guards

Files found in **sub-directories** of the source folder are wrapped in
`#ifdef` guards named after the sub-directory (uppercased).  For example, a
file at `assets/Mac/icon.png` produces:

```cpp
#ifdef MAC
// ... icon_png data ...
#endif
```

This behaviour is the same in both standard and split mode.

---

## Notes

- Hidden files (names starting with `.`), empty files, and source-control
  metadata (`.svn`, `.scc`) are automatically skipped.
- Re-running BinaryBuilder overwrites any previously generated files in the
  target directory.
- The generated code has no JUCE runtime dependency — you can use it in any
  C++ project.
