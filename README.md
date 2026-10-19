# OS Assignment 8: Memory Management in xv6

**Course:** Operating Systems Lab  
**Roll Number:** 2401MC53  
**Topic:** xv6 (x86 Architecture, 32-bit Two-Level Paging)

---

## Table of Contents
1. [Overview](#overview)
2. [Repository Structure](#repository-structure)
3. [Question 1: Virtual to Physical Address Translation](#question-1-virtual-to-physical-address-translation)
4. [Question 2: Physical Memory Statistics](#question-2-physical-memory-statistics)
5. [Question 3: Process Page Table Dumping](#question-3-process-page-table-dumping)
6. [Question 4: Lazy Memory Allocation (Demand Paging)](#question-4-lazy-memory-allocation-demand-paging)
7. [Summary of Memory Usage Metrics](#summary-of-memory-usage-metrics)
8. [Compilation & Execution Instructions](#compilation--execution-instructions)
9. [Problems Faced and Resolutions](#problems-faced-and-resolutions)

---

## Overview

This repository contains the full implementation and verification suite for **Assignment 8: Memory Management in xv6** (Questions 1 through 4). xv6 utilizes a $32$-bit virtual address space managed via two-level paging with $4\text{ KB}$ pages ($10$-bit Page Directory Index, $10$-bit Page Table Index, and $12$-bit Offset).

Key features implemented in this assignment:
* **Question 1:** Custom virtual-to-physical address translation (`va2pa`).
* **Question 2:** Real-time physical memory instrumentation and tracking (`getmeminfo`).
* **Question 3:** Process page table inspector and hierarchy dumper (`vmprint`).
* **Question 4:** Demand paging / Lazy heap memory allocation via page fault handling (`T_PGFLT`).

---

## Repository Structure

```text
.
├── defs.h            # Header file with kernel function prototypes & walkpgdir declaration
├── exec.c            # Process loader (stack allocation & guard page creation)
├── kalloc.c          # Physical memory frame allocator & free page tracking (freepages)
├── mmu.h             # Memory Management Unit definitions & flags (PTE_P, PTE_W, PTE_U)
├── proc.c            # Process management & memory allocation routines
├── proc.h            # Process control block (struct proc) definitions
├── sysproc.c         # System call handlers (sys_va2pa, sys_getmeminfo, sys_sbrk)
├── syscall.c         # System call dispatch table
├── syscall.h         # System call numbers
├── trap.c            # Hardware interrupt and page fault (T_PGFLT) trap handler
├── vm.c              # Page table manipulation routines (walkpgdir, mappages, vmprint)
├── Makefile          # Kernel build rules and user program inclusion
│
├── vatest.c          # User test program for Question 1 (va2pa)
├── memtest.c         # User test program for Question 2 (getmeminfo & leak check)
├── vmprinttest.c     # User test program for Question 3 (vmprint page table hierarchy)
└── lazytest.c        # User test program for Question 4 (demand paging for heap)
```

---

## Question 1: Virtual to Physical Address Translation

### Overview & Implementation
* **System Call:** `int va2pa(int va)`
* Walks the calling process's page table starting from `myproc()->pgdir` using `walkpgdir()` (exposed in `defs.h`).
* Validates present bit (`PTE_P`) and user-mode accessibility (`PTE_U`). Returns `-1` if the page is unmapped or inaccessible from user mode.
* Kernel-side logs the decomposition using `cprintf`
* **User Program (`vatest.c`):** Prints virtual and physical addresses for global, stack, and dynamic (`malloc`) variables. Verifies unmapped addresses (e.g., $0x7FFFFFFF$) return `-1`. Demonstrates that after `fork()`, parent and child share virtual addresses for global variables but map to distinct physical frames.

---

## Question 2: Physical Memory Statistics

### Overview & Implementation
* **System Call:** `int getmeminfo(struct meminfo *mi)`
* Maintains a `freepages` counter in `kalloc.c`, atomically updated during `kalloc()` and `kfree()` while holding `kmem.lock`.
* Returns total physical pages ($\frac{\text{PHYSTOP}}{\text{PGSIZE}}$), current free pages, process user pages ($\frac{sz}{\text{PGSIZE}}$), and page table pages.
* **User Program (`memtest.c`):** Prints baseline memory info, expands memory with `sbrk()`, forks a child process, and verifies that `free_pages` returns exactly to its pre-fork baseline after child exit and reaping (`wait()`).

---

## Question 3: Process Page Table Dumping

### Overview & Implementation
* **System Call:** `int vmprint(void)`
* Traverses `pgdir` entries below `KERNBASE` ($0x80000000$).
* Displays hierarchical page table layout using formatted indentation:
  * Level 1: Page Directory Index, physical table address, flags.
  * Level 2: Page Table Index, mapped virtual address, physical frame address, flags (`P`, `W`, `U`).
* Identifies the `exec()` stack guard page (`PTE_U` cleared).
* **User Program (`vmprinttest.c`):** Calls `vmprint()` at startup, after `sbrk(8 * 4096)`, and after shrinking heap with `sbrk(-4096)`.

---

## Question 4: Lazy Memory Allocation (Demand Paging)

### Overview & Implementation
* Defer physical memory allocation in `sys_sbrk()` when $n > 0$ by expanding `myproc()->sz` without calling `growproc()`. Memory shrinking ($n < 0$) continues to deallocate physical pages immediately.
* Intercept `T_PGFLT` traps in `trap.c`. Obtains faulting virtual address via `rcr2()`.
* Ensures kernel system calls (`read`, `write`, `copyin`, `copyout`) handle lazy buffers safely without crashing.
* **User Program (`lazytest.c`):** Calls `sbrk(32 MB)`, touches widely spaced pages, checks `getmeminfo()`, tests `fork()`, and verifies system calls using untouched buffers.

---

## Summary of Memory Usage Metrics

| Execution Scenario | Total Pages | Free Pages | Process User Pages ($\frac{sz}{\text{PGSIZE}}$) | Page Table Pages |
| :--- | :--- | :--- | :--- | :--- |
| **Initial Boot Baseline (Q2)** | 32,768 | 32,210 | 3 | 3 |
| **After `sbrk(+10 Pages)` (Q2)** | 32,768 | 32,200 | 13 | 3 |
| **After `fork()` Child Creation (Q2)** | 32,768 | 32,187 | 13 | 4 |
| **After Child Exit & Reap (Q2)** | 32,768 | 32,200 | 13 | 3 |
| **After `sbrk(32 MB)` Lazy Call (Q4)** | 32,768 | 32,200 | 8,205 | 3 |
| **After Touching 3 Lazy Pages (Q4)** | 32,768 | 32,197 | 8,205 | 3 |

---

## Compilation & Execution Instructions

### 1. Prerequisites
Ensure the x86 QEMU emulator and GNU multilib build tools are installed:
```bash
sudo apt-get update
sudo apt-get install git build-essential qemu-system-x86 gcc-multilib
```

### 2. Building and Launching xv6
Build the kernel and launch QEMU in non-graphic mode:
```bash
make clean
make qemu-nox
```

### 3. Running User Test Programs
Execute user programs inside the xv6 shell:
```text
$ vatest
$ memtest
$ vmprinttest
$ lazytest
```

---

## Problems Faced and Resolutions

1. **Header Inclusion Conflicts (`struct meminfo`):**
   * *Issue:* Redefining `struct meminfo` in both `user.h` and test files caused compilation errors.
   * *Resolution:* Consolidated `struct meminfo` strictly in `user.h`.

2. **Kernel Faults on Untouched Lazy Buffers:**
   * *Issue:* Passing untouched lazy heap memory to system calls like `read()` caused kernel faults because system calls operate in kernel mode.
   * *Resolution:* Updated kernel memory copy routines (`argptr`, `copyin`) to inspect unmapped addresses within heap bounds ($\text{Guard} < va < sz$) and trigger physical frame allocation on demand.

3. **Compiler Strictness (`-Werror`):**
   * *Issue:* GCC treated unused prototypes and minor formatting inconsistencies as fatal errors.
   * *Resolution:* Wrapped single-statement conditionals in explicit curly braces `{}` and removed unused prototypes.
