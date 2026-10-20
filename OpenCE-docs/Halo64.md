### 1. Does a 64-Bit Clean Port Benefit the Community? (Yes, Immensely!)

Even though Intel Macs are becoming rare, **a true 64-bit (LP64) conversion is the "Holy Grail" for the entire Halo decompilation community.**

Right now, every single port in this repository is built around elaborate workarounds to avoid touching the 32-bit assumptions:
* **Android:** Requires compiling guest code to Apple Watch's `arm64_32`, writing a custom assembly converter (`android_asm_convert.py`), building a custom `musl` libc, and generating trampoline stubs between a 32-bit guest and a 64-bit host.
* **macOS / iOS (OpenCE):** Requires a custom LLVM compiler pass (`guest_rebase.cpp`) using non-standard address spaces and virtual arena offsets.
* **Linux:** Requires users to maintain 32-bit multilib toolchains (`gcc-multilib`, `lib32-glibc`, `lib32-sdl3`), which modern distributions (Arch, Fedora, Ubuntu) are actively deprecating or removing.
* **Windows:** Stuck as a 32-bit PE binary (`i686-pc-windows-msvc`).

**If you make the codebase 64-bit clean:**
1. **Zero Hacks:** Android, iOS, macOS (Intel & Apple Silicon), Linux, and Windows all compile as standard native 64-bit executables using stock Clang/GCC with standard libraries.
2. **Huge Performance & Simplicity Gains:** Eliminates thousands of lines of trampoline code, syscall translation layers, custom compiler plugins, and ABI shims.
3. **Future-Proofing:** Guarantees Halo CE will run on any operating system and architecture (including future RISC-V or 64-bit-only platforms) for the next 20+ years.

---

### 2. Why Hasn’t It Been Done Yet? (The Technical Challenge)

In modern engines, data on disk is deserialized into memory objects. In Halo 1 (the 2001 Blam engine), **the disk layout IS the memory layout**:
1. **Zero-Copy Tag Loading:** When Halo loads a map (`.map`), it memory-maps blocks directly. Structures inside `.map` files contain raw 32-bit pointers that the original Xbox kernel fixed up in-place (`tag_memory_fixup`).
2. **Struct Layout Discrepancies:** In 64-bit (`LP64`), pointers are 8 bytes instead of 4 bytes, and `long` is 64-bit.
   ```c
   // On Xbox (32-bit): sizeof = 16 bytes
   // On x86_64 / arm64 (64-bit): sizeof = 24 or 32 bytes (with alignment padding)
   struct tag_reference {
       tag_group_tag group_tag; // 4 bytes
       char *name;              // 4 bytes -> expands to 8 bytes!
       long name_length;        // 4 bytes -> expands to 8 bytes!
       datum_index index;       // 4 bytes
   };
   ```
   If a struct's size changes in 64-bit code, **every single field offset in memory no longer lines up with the original Xbox disk data**.
3. **Save Games & Scripts:** Saved games (`.sav`) and HaloScript execution trees (`hs_runtime.c`) dump raw memory pools directly to disk.

---

### 3. Proven Strategies from Other Decompilations

Other legendary decompilations (such as *Super Mario 64 PC*, *Zelda: Ship of Harkinian*, and *DevilutionX*) faced this exact problem and solved it in one of two ways:

#### Strategy A: The "Pointer Handle / Relative Offset" Approach (Recommended)
Instead of letting raw pointers in tag structs expand to 64 bits:
* Replace struct pointers with explicit 32-bit types, such as `uint32_t` file offsets or a typed wrapper `tag_offset<T>`.
* When code accesses the data, a macro or helper function resolves the 32-bit offset against the map base address into a 64-bit memory pointer:
  ```c
  // Preserves exact 32-bit disk binary compatibility
  struct s_tag_block {
      int32_t count;
      uint32_t offset; // 32-bit offset relative to map base, not a 64-bit pointer
  };
  #define TAG_BLOCK_GET(block, type) ((type *)(map_base + (block)->offset))
  ```

#### Strategy B: Separation of "On-Disk" and "In-Memory" Definitions
* Keep packed on-disk struct definitions (`struct raw_tag_reference`) for reading maps.
* Inflate them into native in-memory structs (`struct tag_reference`) upon loading.
* This is how modern game engines work, though it requires writing loading code for each tag type.

---

### 4. What You Should Learn First

1. **C Memory Layout & Data Models:**
   * Understand **ILP32** (32-bit int, long, pointer) vs **LP64** (Linux/macOS 64-bit: 64-bit long & pointer) vs **LLP64** (Windows 64-bit: 32-bit long, 64-bit pointer).
   * Struct alignment rules, `#pragma pack(push, 1)`, and natural padding.
2. **Halo Blam Engine Architecture (Study these in the repo):**
   * **Tag Management:** [`source/tag_files/tag_groups.h`](file:///Users/ibm/projects/halo-ce-universal/source/tag_files/tag_groups.h), [`source/tag_files/tag_files.c`](file:///Users/ibm/projects/halo-ce-universal/source/tag_files/tag_files.c)
   * **Cache Loading:** [`source/cache/cache_files.c`](file:///Users/ibm/projects/halo-ce-universal/source/cache/cache_files.c)
   * **Memory Pools & Handles:** [`source/memory/memory_pool.c`](file:///Users/ibm/projects/halo-ce-universal/source/memory/memory_pool.c), `datum_index`
   * **HaloScript Engine:** [`source/hs/hs_runtime.c`](file:///Users/ibm/projects/halo-ce-universal/source/hs/hs_runtime.c)
3. **Pointer Swizzling:**
   * How relative pointers and fixup tables transform static file offsets into active pointers in RAM.

---

### 5. Essential Tools You Will Need

1. **Memory & Layout Diagnostic Tools:**
   * **`pahole` (Poke-a-Hole):** Analyzes compiled ELF/DWARF binaries and shows the exact padding, struct sizes, and field offsets.
   * **`clang -Xclang -fdump-record-layouts`:** Dumps the exact layout Clang generates for every struct in a C file.
2. **Compiler Sanitizers (Crucial!):**
   * Compile with `-fsanitize=address,undefined` (ASan/UBSan).
   * This immediately catches pointer truncation, unaligned memory access, and out-of-bounds reads the instant you test a level.
3. **Asset & Map Inspection:**
   * **[Invader](https://github.com/cpt-meh/Invader):** Inspect tag structures and verify byte-for-byte fidelity with retail Xbox maps.
4. **Binary Comparison & Hex Diffs:**
   * Tools like **ImHex** or `vbindiff` to compare in-memory dumps between a reference 32-bit build and your 64-bit test build.

---

### 6. Suggested Roadmap to Get Started

1. **Audit Global Types:** Replace generic `long` across the codebase with explicit `int32_t` or `intptr_t`. (On 64-bit macOS/Linux, `long` silently becomes 64-bit and breaks struct sizes).
2. **Pick One Subsystem (Start Small):**
   * Begin with standalone math, string handling, or compression (`source/math/`, `source/cseries/`).
   * Test them compiling natively as `x86_64` unit tests.
3. **Tackle the Tag Header & Index:**
   * Make `s_cache_file_header` and `s_tag_group` 64-bit compatible by isolating file offsets from pointer accesses.
4. **Verify Map Parsing:**
   * Write a small standalone CLI test tool that reads `ui.map` and parses all tag names without crashing on `x86_64`. Once you can parse every tag on a 64-bit build, you've conquered the hardest hurdle!