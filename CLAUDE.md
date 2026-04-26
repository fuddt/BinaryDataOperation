# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Run

```bash
# Configure (first time or after CMakeLists.txt changes)
cmake -S . -B cmake-build-debug

# Build
cmake --build cmake-build-debug

# Run
./cmake-build-debug/BinaryDataOperation
```

The project uses C++20 and CMake 4.1+. There is no test suite — verification is done by running `main.cpp` and observing stdout.

## Project Purpose

This is a **learning and design repository** for understanding binary data manipulation in C++. It is not a production library. The goal is to design and prototype a "Partial Binary Patch" system — a mechanism for safely updating specific fields within a raw binary buffer without touching the rest.

The learning is structured across phased Markdown documents (Phase 1–5). Code experiments go in `main.cpp`.

## Core Architecture (Phase 5 target design)

The central design uses three components working together:

**`LayoutTable`** — a runtime map from field ID to byte-range descriptor, replacing compile-time structs:
```cpp
using LayoutTable = std::unordered_map<FieldID, FieldDescriptor>;
// FieldDescriptor = { size_t offset; size_t size; }
```

**`PatchRequest`** — describes what to overwrite and with what data:
```cpp
struct PatchRequest { FieldID field; const uint8_t* newData; size_t newDataSize; };
```

**`applyPatch()`** — performs `std::memcpy` at `buffer + desc.offset` after validating bounds, field existence, and size match. Returns `bool` indicating success.

Key design principle: **one patch function, multiple layouts**. Format differences (e.g., `owner` at offset 4 vs offset 8) are handled by swapping the `LayoutTable`, not branching the patch logic.

## Safety Rules for `memcpy`-based patches

Every patch function must validate before writing:
1. `buffer != nullptr` and `request.newData != nullptr`
2. Field exists in the layout table
3. `desc.offset <= bufferSize` (overflow-safe: check offset first, then size)
4. `desc.size <= bufferSize - desc.offset`
5. `request.newDataSize == desc.size` (fixed-length fields require exact size match)
