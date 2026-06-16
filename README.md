<div align="center">

# 🧩 DesignDecode

### *Decoding systems engineering from silicon to shell — a CPU simulator, the Linux scheduler, high-frequency ISR patterns, and a 30-day kernel built from scratch.*

[![C](https://img.shields.io/badge/Language-C-blue?logo=c&logoColor=white)](https://en.wikipedia.org/wiki/C_(programming_language))
[![Python](https://img.shields.io/badge/Assembler-Python-yellow?logo=python&logoColor=white)](https://www.python.org/)
[![AArch64](https://img.shields.io/badge/Target-ARMv8--A%20%2F%20AArch64-red?logo=arm&logoColor=white)](https://www.arm.com/)
[![QEMU](https://img.shields.io/badge/Emulator-QEMU-orange?logo=qemu&logoColor=white)](https://www.qemu.org/)
[![Linux](https://img.shields.io/badge/Platform-Linux-black?logo=linux&logoColor=white)](https://kernel.org/)
[![Docs](https://img.shields.io/badge/Docs-Markdown-success?logo=markdown&logoColor=white)](#-repository-map)

*From building a CPU in C, to reading the real Linux scheduler, to shipping a bootable AArch64 kernel in 30 days.*

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Repository Map](#-repository-map)
- [Module 1 — Single-Threaded CPU Simulator](#-module-1--single-threaded-cpu-simulator)
- [Module 2 — Linux Scheduler Internals](#-module-2--linux-scheduler-internals)
- [Module 3 — High-Frequency ISR Design Guide](#-module-3--high-frequency-isr-design-guide)
- [Module 4 — nkernel: A 30-Day Kernel From Scratch](#-module-4--nkernel-a-30-day-kernel-from-scratch)
- [Interview Question Bank](#-interview-question-bank)
- [Quick Start](#-quick-start)
- [Who Is This For?](#-who-is-this-for)

---

## 🎯 Overview

**DesignDecode** is a structured, hands-on exploration of **low-level systems engineering**. It is built bottom-up — you start by building hardware behaviour in C, then study how a production operating system schedules work, then learn the patterns that keep interrupt handlers alive under extreme load, and finally assemble all of it into a real, bootable kernel.

The content targets engineers preparing for **kernel, driver, platform, and performance roles** (NVIDIA / Google / Meta-style systems interviews) who want genuine hardware-aware intuition rather than surface-level OS trivia.

```
Four pillars, built from the metal up:

  🖥️  CPU Simulator        →   🗓️  Linux Scheduler        →   ⚡  High-Frequency ISRs   →   🐧  nkernel (30 days)
  build the hardware            read the real scheduler        survive 100 kHz IRQs          ship a bootable kernel
  C · custom ISA · QEMU         CFS/EEVDF · RT · DEADLINE       ring buffers · DMA · C11      AArch64 · MMU · VFS · SMP

                              ➕  Interview Question Bank — 120+ OS / kernel / driver questions
```

---

## 🗂️ Repository Map

```
01_DesignDecode/
│
├── 01_SingleThreaded_CPU/                 ← CPU simulator built from scratch in C
│   ├── Architecture/                      ← 10 design documents (ISA → toolchain → walkthrough)
│   ├── Implementation/
│   │   ├── src/                           ← cpu.c, alu.c, decoder.c, memory.c, main.c (+ headers)
│   │   ├── programs/                      ← .asm sources + assembled .bin programs
│   │   ├── tools/                         ← assembler.py (two-pass assembler)
│   │   └── qemu/                          ← full-system QEMU boot scripts + initramfs
│   └── Log_Analysis/                      ← 10 real execution scenarios, analysed instruction by instruction
│
├── 02_Scheduler/                          ← Linux scheduler internals + a working context switcher
│   ├── 01_Sample/                         ← reference scheduler skeleton
│   ├── 02_Scheduler_Implementation/       ← 14 deep-dives: CFS/EEVDF, RT, DEADLINE, PELT, EAS, sched_ext…
│   └── 03_context_switchig_scheduler/     ← C demo: TCB, ready queue, context switch + ftrace analysis
│
├── 03_High_Frequency_ISR_Design_Guide/    ← ISR design patterns for 100 kHz+ interrupts
│   ├── 00_High_Frequency_ISR_Design_Guide.Md
│   ├── 00_High_Frequency_ISR_Design/      ← rules, sync primitives, overflow detection
│   ├── 01_LockFree_RingBuffer_SPSC/       ← single-producer / single-consumer queue
│   ├── 02_DoubleBuffer_PingPong/          ← ping-pong buffers, no copy in the ISR
│   ├── 03_DMA_PingPong_HardwareOffload/   ← let the DMA engine do the producing
│   ├── 04_Linux_Kernel_BottomHalf/        ← softirqs, tasklets, workqueues, threaded IRQs
│   └── 05_Atomics_MemoryOrdering_C11/     ← memory_order_*, acquire/release, fences
│
├── 04_Kernel_Devlopment/                  ← nkernel: a Linux-like AArch64 kernel in 30 days
│   └── docs/                              ← architecture.md + day01…day30 build journal
│
├── Questions.Md                           ← 120+ OS / kernel / driver interview questions
└── README.md                              ← this file
```

---

## 🖥️ Module 1 — Single-Threaded CPU Simulator

> **Goal:** Understand exactly how a CPU works by building one.

A **fully self-contained 32-bit RISC CPU simulator** written in C — a custom ISA, a Python assembler, and full-system QEMU support. No libraries. No shortcuts.

### Architecture Pipeline

```
┌──────────────┐     ┌───────────────┐     ┌──────────────────┐
│  .asm Source │────▶│ Python        │────▶│  .bin Binary     │
│ (fibonacci)  │     │ Assembler     │     │ (loaded into     │
└──────────────┘     │ (two-pass)    │     │  flat memory)    │
                     └───────────────┘     └────────┬─────────┘
                                                    │
                              ┌─────────────────────▼──────────────────────┐
                              │              CPU Core (cpu.c)               │
                              │                                             │
                              │  ┌─────────┐  ┌─────────┐  ┌───────────┐   │
                              │  │  FETCH  │─▶│ DECODE  │─▶│  EXECUTE  │   │
                              │  │ mem[PC] │  │ decoder │  │   ALU /   │   │
                              │  └─────────┘  └─────────┘  │  handler  │   │
                              │                            └─────┬─────┘   │
                              │                            ┌─────▼─────┐   │
                              │                            │ WRITEBACK │   │
                              │                            │  Rd / PC  │   │
                              │                            └───────────┘   │
                              └─────────────────────────────────────────────┘
```

### Key Components

| Component | File | Description |
|-----------|------|-------------|
| **CPU Core** | `Implementation/src/cpu.c` | Fetch → decode → execute → writeback loop |
| **ALU** | `Implementation/src/alu.c` | ADD, SUB, MUL, DIV, AND, OR, XOR, SHL, SHR + flags (Z, N, C, O) |
| **Decoder** | `Implementation/src/decoder.c` | Splits 32-bit words into typed `Instruction` structs |
| **Memory** | `Implementation/src/memory.c` | Flat address space, little-endian, R/W accessors |
| **ISA** | `Implementation/src/instructions.h` | Fixed 32-bit instruction encoding |
| **Assembler** | `Implementation/tools/assembler.py` | Two-pass Python assembler with label resolution |
| **QEMU Setup** | `Implementation/qemu/` | Boot scripts + initramfs for full-system runs |

### Architecture Documents

| # | Document | What It Covers |
|---|----------|----------------|
| 00 | [Overview](./01_SingleThreaded_CPU/Architecture/00_Overview.md) | System diagram, design philosophy, data flow |
| 01 | [ISA](./01_SingleThreaded_CPU/Architecture/01_ISA.md) | Instruction encoding, opcode table, register aliases |
| 02 | [Memory Model](./01_SingleThreaded_CPU/Architecture/02_Memory_Model.md) | Flat memory, address map |
| 03 | [ALU](./01_SingleThreaded_CPU/Architecture/03_ALU.md) | Arithmetic/logic operations, flag semantics |
| 04 | [Decoder](./01_SingleThreaded_CPU/Architecture/04_Decoder.md) | Bit extraction, sign extension, disassembler |
| 05 | [CPU Core](./01_SingleThreaded_CPU/Architecture/05_CPU_Core.md) | State machine, instruction cycle, step/run |
| 06 | [Assembler](./01_SingleThreaded_CPU/Architecture/06_Assembler.md) | Two-pass assembler, label resolution, syntax |
| 07 | [Toolchain & Build](./01_SingleThreaded_CPU/Architecture/07_Toolchain_Build.md) | Makefile, native vs ARM cross-compilation |
| 08 | [Fibonacci Walkthrough](./01_SingleThreaded_CPU/Architecture/08_Fibonacci_Walkthrough.md) | Step-by-step trace of Fibonacci execution |
| 09 | [Interview: CPU Design](./01_SingleThreaded_CPU/Architecture/09_Interview_CPU_Design.md) | CPU architecture interview Q&A |

### Log Analysis Scenarios

Ten real execution traces analysed at the instruction level:

| # | Scenario | What It Demonstrates |
|---|----------|----------------------|
| 01 | [Normal Fibonacci Run](./01_SingleThreaded_CPU/Log_Analysis/01_Normal_Fibonacci_Run/) | Correct execution trace, register state |
| 02 | [Infinite Loop Detection](./01_SingleThreaded_CPU/Log_Analysis/02_Infinite_Loop_Detection/) | PC cycle detection, cycle-limit enforcement |
| 03 | [Division By Zero](./01_SingleThreaded_CPU/Log_Analysis/03_Division_By_Zero/) | Fault handling, trap mechanism |
| 04 | [Stack Overflow](./01_SingleThreaded_CPU/Log_Analysis/04_Stack_Overflow/) | SP tracking, memory boundary violation |
| 05 | [Illegal Opcode Fault](./01_SingleThreaded_CPU/Log_Analysis/05_Illegal_Opcode_Fault/) | Undefined-instruction exception |
| 06 | [Memory Out Of Bounds](./01_SingleThreaded_CPU/Log_Analysis/06_Memory_Out_Of_Bounds/) | Address-space violation |
| 07 | [Function Call & Return](./01_SingleThreaded_CPU/Log_Analysis/07_Function_Call_Return/) | Call-stack discipline, link register |
| 08 | [Large Immediate Encoding](./01_SingleThreaded_CPU/Log_Analysis/08_Large_Immediate_Encoding/) | Sign extension, immediate limits |
| 09 | [Off-By-One Loop Bug](./01_SingleThreaded_CPU/Log_Analysis/09_Off_By_One_Loop_Bug/) | Flag-based branch condition errors |
| 10 | [Flag Dependency Chain](./01_SingleThreaded_CPU/Log_Analysis/10_Flag_Dependency_Chain/) | Multi-instruction flag propagation |

---

## 🗓️ Module 2 — Linux Scheduler Internals

> **Goal:** Move from "the OS schedules tasks" to understanding *exactly how* the Linux scheduler makes every decision — then prove it with a working context switcher.

The [`02_Scheduler/`](./02_Scheduler) module pairs **14 deep-dive documents on the real Linux scheduler** with a hands-on C implementation of thread control blocks, ready queues, and context switching, analysed with ftrace.

### Scheduler Deep-Dives

| # | Document | Focus |
|---|----------|-------|
| 00 | [Scheduler Architecture](./02_Scheduler/02_Scheduler_Implementation/00_Scheduler_Architecture_Main.md) | The big picture: runqueues, classes, the main `schedule()` path |
| 01 | [CFS / EEVDF Internals](./02_Scheduler/02_Scheduler_Implementation/01_CFS_EEVDF_Internals.md) | Fair scheduling, `vruntime`, the EEVDF transition |
| 02 | [RT Scheduler](./02_Scheduler/02_Scheduler_Implementation/02_RT_Scheduler_Internals.md) | `SCHED_FIFO` / `SCHED_RR`, priority arrays |
| 03 | [DEADLINE Scheduler](./02_Scheduler/02_Scheduler_Implementation/03_Deadline_Scheduler_Internals.md) | `SCHED_DEADLINE`, CBS, EDF |
| 04 | [Load Balancing & NOHZ](./02_Scheduler/02_Scheduler_Implementation/04_Load_Balancing_and_NOHZ.md) | Migration, sched domains, tickless idle |
| 05 | [CPU Placement, EAS & Capacity](./02_Scheduler/02_Scheduler_Implementation/05_CPU_Placement_EAS_and_Capacity.md) | Energy-aware scheduling, capacity-aware placement |
| 06 | [PELT & Utilization Signals](./02_Scheduler/02_Scheduler_Implementation/06_PELT_and_Utilization_Signals.md) | Per-entity load tracking |
| 07 | [Schedutil & DVFS](./02_Scheduler/02_Scheduler_Implementation/07_Schedutil_DVFS_Integration.md) | Frequency selection from utilization |
| 08 | [Topology & Sched Domains](./02_Scheduler/02_Scheduler_Implementation/08_Topology_and_Sched_Domains.md) | NUMA / SMT topology modelling |
| 09 | [sched_ext (BPF)](./02_Scheduler/02_Scheduler_Implementation/09_Sched_Ext_BPF_Design.md) | Pluggable BPF schedulers |
| 10 | [Core Scheduling & SMT](./02_Scheduler/02_Scheduler_Implementation/10_Core_Scheduling_SMT.md) | SMT security, core scheduling |
| 11 | [Concurrency, Locking & Migration](./02_Scheduler/02_Scheduler_Implementation/11_Concurrency_Locking_and_Migration.md) | rq locks, migration safety |
| 12 | [uclamp & CPU Controller](./02_Scheduler/02_Scheduler_Implementation/12_uclamp_and_CPU_Controller.md) | Utilization clamping, cgroup CPU control |
| 13 | [Interview Terminology](./02_Scheduler/02_Scheduler_Implementation/13_Scheduler_Interview_Terminology_Deep_Explanation.md) | Every term, explained for interviews |

### Context-Switching Demo

The [`03_context_switchig_scheduler/`](./02_Scheduler/03_context_switchig_scheduler) folder contains a runnable C scheduler plus annotated output:

| File | What It Covers |
|------|----------------|
| [`01_tcb_scheduler.c`](./02_Scheduler/03_context_switchig_scheduler/01_tcb_scheduler.c) | TCB, ready queue, and a real context switch |
| [`02_design_doc.md`](./02_Scheduler/03_context_switchig_scheduler/02_design_doc.md) | Design rationale for the implementation |
| [`06_ftrace_logs_explained.md`](./02_Scheduler/03_context_switchig_scheduler/06_ftrace_logs_explained.md) | ftrace logs of multithreaded behaviour, explained |
| [`ftrace_multithread.sh`](./02_Scheduler/03_context_switchig_scheduler/ftrace_multithread.sh) | Script to capture the traces yourself |

---

## ⚡ Module 3 — High-Frequency ISR Design Guide

> **Goal:** Design interrupt handlers that survive 100 kHz interrupt rates — only ~10 µs per interrupt.

At **100,000 interrupts/sec you get ~10 µs per ISR**. The [`03_High_Frequency_ISR_Design_Guide/`](./03_High_Frequency_ISR_Design_Guide) module catalogs the canonical patterns that keep ISRs minimal and offload heavy work safely. The golden rule: **the ISR pushes, the worker pulls.**

```
[ Hardware ] --IRQ--> [ ISR: push to ring buffer ] --> [ Worker Task: process data ]
        minimal, bounded work in here ───┘                  ───┘ heavy lifting out here
```

| # | Pattern | Why It Matters |
|---|---------|----------------|
| 00 | [Design Guide Overview](./03_High_Frequency_ISR_Design_Guide/00_High_Frequency_ISR_Design_Guide.Md) | The golden rule, design rules, overflow detection |
| 01 | [Lock-Free Ring Buffer (SPSC)](./03_High_Frequency_ISR_Design_Guide/01_LockFree_RingBuffer_SPSC) | Single-producer / single-consumer queue between ISR and worker |
| 02 | [Double Buffer (Ping-Pong)](./03_High_Frequency_ISR_Design_Guide/02_DoubleBuffer_PingPong) | Two buffers swapped under a flag — no copy in the ISR |
| 03 | [DMA Ping-Pong (Hardware Offload)](./03_High_Frequency_ISR_Design_Guide/03_DMA_PingPong_HardwareOffload) | Let the DMA engine do the producing |
| 04 | [Linux Kernel Bottom Halves](./03_High_Frequency_ISR_Design_Guide/04_Linux_Kernel_BottomHalf) | Softirqs, tasklets, workqueues, threaded IRQs |
| 05 | [Atomics & Memory Ordering (C11)](./03_High_Frequency_ISR_Design_Guide/05_Atomics_MemoryOrdering_C11) | `memory_order_*`, acquire/release, fences |

---

## 🐧 Module 4 — nkernel: A 30-Day Kernel From Scratch

> **Goal:** Take everything above and ship a real, bootable kernel.

[`04_Kernel_Devlopment/`](./04_Kernel_Devlopment) documents **nkernel** — a Linux-like, monolithic, preemptive, SMP-ready educational kernel for **ARMv8-A / AArch64** running on **QEMU `virt` (Cortex-A72)** — built and documented over 30 days, from first boot to a tagged `v1.0` with an interactive shell.

📐 Start with the [**Architecture Overview**](./04_Kernel_Devlopment/docs/architecture.md) for goals, non-goals, the memory map, and the full phase plan.

### Phase Map

| Phase | Days | Theme | Gate |
|:---:|:---:|---|---|
| 1 | 01–05 | Boot, UART, exceptions, GIC, timer | Periodic tick in QEMU |
| 2 | 06–11 | MMU + physical/virtual memory | `kmalloc`/`vmalloc` + page-fault recovery |
| 3 | 12–16 | Tasking, scheduler, locks, EL0 drop | Two kthreads + first userspace `hello` |
| 4 | 17–19 | Syscalls, ELF loader, signals stub | User ELF runs via `exec`, returns via `exit` |
| 5 | 20–24 | VFS, initramfs, VirtIO-blk, ext2 | `cat /mnt/file` persists across reboot |
| 6 | 25–27 | libc, init, shell, devfs | Interactive shell on serial |
| 7 | 28–30 | SMP, hardening, release | 2-CPU boot, CI green, tag `v1.0` |

### Daily Build Journal

| Days | Topics |
|---|---|
| [01](./04_Kernel_Devlopment/docs/day01-boot.md) · [02](./04_Kernel_Devlopment/docs/day02-uart-el1.md) · [03](./04_Kernel_Devlopment/docs/day03-fdt-panic.md) · [04](./04_Kernel_Devlopment/docs/day04-exceptions.md) · [05](./04_Kernel_Devlopment/docs/day05-gic-timer.md) | Boot, UART @ EL1, FDT & panic, exceptions, GIC + timer |
| [06](./04_Kernel_Devlopment/docs/day06-mmu-theory.md) · [07](./04_Kernel_Devlopment/docs/day07-mmu-enable.md) · [08](./04_Kernel_Devlopment/docs/day08-bootmem.md) · [09](./04_Kernel_Devlopment/docs/day09-buddy.md) · [10](./04_Kernel_Devlopment/docs/day10-slab-vmalloc.md) · [11](./04_Kernel_Devlopment/docs/day11-page-fault.md) | MMU theory & enable, boot memory, buddy, slab/vmalloc, page faults |
| [12](./04_Kernel_Devlopment/docs/day12-task-context.md) · [13](./04_Kernel_Devlopment/docs/day13-scheduler.md) · [14](./04_Kernel_Devlopment/docs/day14-locks-wait.md) · [15](./04_Kernel_Devlopment/docs/day15-fork-mm.md) · [16](./04_Kernel_Devlopment/docs/day16-eret-el0.md) | Task context, scheduler, locks & wait, fork/mm, drop to EL0 |
| [17](./04_Kernel_Devlopment/docs/day17-syscalls.md) · [18](./04_Kernel_Devlopment/docs/day18-elf-loader.md) · [19](./04_Kernel_Devlopment/docs/day19-exit-signals.md) | Syscalls, ELF loader, exit & signals |
| [20](./04_Kernel_Devlopment/docs/day20-vfs.md) · [21](./04_Kernel_Devlopment/docs/day21-initramfs.md) · [22](./04_Kernel_Devlopment/docs/day22-virtio-blk.md) · [23](./04_Kernel_Devlopment/docs/day23-ext2-ro.md) · [24](./04_Kernel_Devlopment/docs/day24-pagecache-ext2-rw.md) | VFS, initramfs, VirtIO-blk, ext2 (RO → RW), page cache |
| [25](./04_Kernel_Devlopment/docs/day25-nlibc.md) · [26](./04_Kernel_Devlopment/docs/day26-init-ksh.md) · [27](./04_Kernel_Devlopment/docs/day27-devfs.md) | `nlibc`, `init` + `ksh` shell, devfs |
| [28](./04_Kernel_Devlopment/docs/day28-smp-psci.md) · [29](./04_Kernel_Devlopment/docs/day29-hardening.md) · [30](./04_Kernel_Devlopment/docs/day30-release.md) | SMP via PSCI, hardening, release `v1.0` |

---

## 🎓 Interview Question Bank

[`Questions.Md`](./Questions.Md) is a high-coverage **OS / kernel / driver interview question bank** aimed at NVIDIA-style systems-software, kernel/driver, platform, and performance roles — 120+ questions grouped by topic:

- **Core OS Fundamentals** — boot path, user vs kernel mode, syscalls, traps/faults/aborts, monolithic vs microkernel
- **Processes, Threads & Scheduling** — CFS, priority inversion, RT classes, affinity, NUMA, `fork`/`vfork`/`clone`
- **Memory, Concurrency, Interrupts, Drivers & Performance** — and more

---

## 🚀 Quick Start

```bash
# Clone
git clone https://github.com/nilprajapati97/01_DesignDecode.git
cd 01_DesignDecode

# 1) Build & run the CPU simulator
cd 01_SingleThreaded_CPU/Implementation
#   see Architecture/07_Toolchain_Build.md for the Makefile + QEMU instructions

# 2) Read the Linux scheduler deep-dives
#   open 02_Scheduler/02_Scheduler_Implementation/00_Scheduler_Architecture_Main.md

# 3) Study the high-frequency ISR patterns
#   open 03_High_Frequency_ISR_Design_Guide/00_High_Frequency_ISR_Design_Guide.Md

# 4) Follow the 30-day kernel build
#   start at 04_Kernel_Devlopment/docs/architecture.md, then day01-boot.md
```

> Most modules are documentation-first: read the design docs alongside the code. The CPU simulator and the context-switching scheduler demo are runnable; build instructions live in their respective folders.

---

## 👥 Who Is This For?

- Engineers preparing for **kernel / driver / platform / performance** interviews who want depth, not flashcards.
- Developers who learn best **bottom-up** — by building the hardware, then the OS on top of it.
- Anyone who wants a single place that connects **CPU architecture → scheduling → interrupt handling → a working kernel**.

---

<div align="center">

*Built to decode systems design — one layer of the stack at a time.*

</div>
