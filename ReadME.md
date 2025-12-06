# virtual-memory-manager 📦

**User-Level Virtual Memory Allocator with 2-Level Paging & TLB Stats**

# PROJECT OVERVIEW  
This project implements a tiny virtual memory “subsystem” in C that sits on top of a 1 GB simulated physical memory. 

It exposes a malloc-style API (`n_malloc`, `n_free`) plus `put_data` / `get_data`, and backs them with:

1. A bitmap-based virtual/physical page allocator  
2. A 2-level page table (page directory + page table)  
3. A simple software TLB with compulsory-miss counting  
4. Basic matrix-multiply benchmark and multi-thread tests

The goal is to practice how real OSes wire together page tables, TLBs, and user-level allocators.

---

# KEY FEATURES 🔑

- **Bitmap-based page management:**  
  - One bitmap for physical frames, one for virtual pages  
  - First-fit scan for contiguous virtual pages in `get_next_avail`  
  - Never allocates VA `0x0` (starts from VPN 1)

- **Two-level page table:**  
  - Top-level page directory (`pde_t`) with 1024 entries  
  - Page tables stored in a reserved “PT region” of physical frames  
  - `map_page()` wires virtual pages → physical frames and sets `PTE_PRESENT | PTE_WRITABLE`

- **User-level allocation API:**  
  - `n_malloc(bytes)` → returns a virtual base address backed by mapped pages  
  - `n_free(va, size)` → clears virtual + physical bitmap bits and invalidates PTEs  
  - Rollback logic: if mapping fails halfway, already-allocated frames/pages are freed

- **Data movement helpers:**  
  - `put_data(va, buf, size)` splits writes across page boundaries using `translate()`  
  - `get_data(va, buf, size)` reads across pages, zero-fills on unmapped access  
  - Both functions work purely through the page tables + TLB, not raw indices

- **Matrix multiply sanity test:**  
  - `mat_mult(a, b, size, c)` treats `a`, `b`, `c` as matrices in VM space  
  - Uses `put_data`/`get_data`, so it stress-tests translation + page crossing

- **Software TLB with compulsory misses:**  
  - Direct-mapped TLB (`TLB_ENTRIES` slots) keyed by VPN  
  - `translate()` first probes the TLB, then walks page tables on a miss  
  - First access to a newly mapped VPN is counted as a compulsory miss  
  - `print_TLB_missrate()` prints `lookups`, `misses`, and miss rate

- **Thread-safe initialization & bitmaps:**  
  - `set_physical_mem()` protected by `init_lock`  
  - Physical and virtual bitmaps guarded by `phys_bitmap_lock` / `virt_bitmap_lock`  
  - TLB access guarded by `tlb_lock`

---

# TECHNICAL STACK 🧱

- **Language**  
  - C.

- **Memory model:**  
  - Simulated 1 GB physical memory (`g_physical_mem`)  
  - Page size: 4 KB (`PGSIZE`)  
  - Derived constants: number of physical frames, virtual pages, offsets, indices

- **Paging & address translation:**  
  - Macros from `my_vm.h`: `VA2U`, `U2VA`, `PDX`, `PTX`, `OFF`, `PFN_SHIFT`, flags like `PTE_PRESENT`  
  - 2-level page table stored inside simulated physical memory in a reserved frame region  
  - `translate(pgdir, va)` returns a `pte_t*` for the given virtual address

- **TLB design:**  
  - Struct `tlb` with arrays: `valid[]`, `vpn[]`, `pfn[]`, `last_used[]`  
  - Direct-mapped placement: `index = vpn % TLB_ENTRIES`  
  - Miss accounting:  
    - TLB miss when `translate()` doesn’t find a valid matching VPN  
    - Compulsory miss counted once per newly mapped VPN in `map_page()`  
  - `print_TLB_missrate()` used by benchmarks to report stats

- **Synchronization:**  
  - `pthread_mutex_t` locks for: init, bitmaps, and TLB  
  - Benchmarks (`multi_test.c`) use POSIX `pthread` threads to generate concurrent VM traffic

---

# HOW TO RUN THE PROGRAM 🚀

The instructions to run the project is located in the `HowToRun.md` file in the root directory.

---

# PERFORMANCE NOTES 📝

- **Unit tests (`tests/`):**  
  - `test_alloc.c` checks basic `n_malloc` / `n_free` with contiguous pages  
  - `test_putget.c` writes/reads across page boundaries to validate `put_data`/`get_data`  
  - `test_matmul.c` multiplies small matrices in VM space and checks the result

- **Benchmark programs (`benchmark/`):**  
  - `test.c`:  
    - Allocates three arrays, fills two as 5×5 matrices of `1`s, and multiplies them  
    - Frees and re-allocates to verify `n_free` resets bitmap state correctly  
    - Calls `print_TLB_missrate()` → ~3 compulsory misses (one per array)

  - `multi_test.c` (`mtest`):  
    - Spawns 15 threads that allocate, initialize, multiply, and free matrices  
    - Exercises concurrent calls to `n_malloc`, `put_data`, `get_data`, and `mat_mult`  
    - Expected compulsory misses around `3 arrays × 15 threads ≈ 45`  
    - Final `print_TLB_missrate()` shows lookups, misses, and overall miss rate

- **Key takeaways:**  
  - Once a VPN is in the TLB, repeated `put_data` / `get_data` calls mostly hit → lower miss rate.  
  - The cost of page table walks + TLB misses is visible when you scale up the thread count and matrix size.  
  - Even though everything is single-process / single-core, the design mirrors how an OS glues together paging, allocation, and TLB behavior.

---

# CONTRIBUTORS 👥

- **Kelvin Ihezue**, **Bryan Shangguan**
