# RISC-V Edge-Container OS Reading List & Architecture Roadmap

A curated roadmap covering every phase of building an edge-focused, container-capable operating system for 64-bit RISC-V (`rv64gc`).

---

## Stage 1: Hardware Specs & Toolchain Setup
- **RISC-V Unprivileged & Privileged Specs (v20191213 / Latest):** Focus on Machine Mode (M-Mode), Supervisor Mode (S-Mode), Control and Status Registers (CSRs), and SV39 virtual memory addressing.
  - *Reference:* [riscv.org/technical/specifications](https://riscv.org/technical/specifications/)
- **Linker Scripts - GNU ld Manual:** Read Section 3 ("Linker Scripts") focusing on `SECTIONS`, `MEMORY`, `.`, and symbol definitions.
  - *Reference:* [Sourceware GNU LD Documentation](https://sourceware.org/binutils/docs/ld/Scripts.html)
- **QEMU RISC-V `virt` Board Specification:** Read memory map details (DRAM starts at `0x80000000`, 16550A UART sits at `0x10000000`).
  - *Reference:* [QEMU Target RISC-V Documentation](https://www.qemu.org/docs/master/system/target-riscv.html)

---

## Stage 2: Core Kernel Reference Textbooks & Tutorials
- **Operating Systems: Three Easy Pieces (OSTEP) by Remzi & Andrea Arpaci-Dusseau:**
  - *Must-read chapters:* Virtualization (CPU scheduling, direct execution), Memory (Paging, TLB, multi-level tables), Concurrency, and Persistence. Free online: [ostep.org](https://pages.cs.wisc.edu/~remzi/OSTEP/).
- **MIT 6.S081 / 6.828 (xv6-riscv):**
  - Read the source and accompanying book for **xv6-riscv** (modern standard teaching OS targeting QEMU virt RISC-V).
  - *Reference:* [MIT xv6-riscv Git & Book](https://pdos.csail.mit.edu/6.828/2023/xv6.html)
- **Stephen Marz's Adventures in RISC-V Operating Systems:**
  - Hands-on guide for writing bootloaders, MMU pagers, UART drivers, and scheduler in Rust/C.
  - *Reference:* [osblog.stephenmarz.com](https://osblog.stephenmarz.com/)

---

## Stage 3: Memory Subsystems
1. **Physical Allocator (Page Frames):**
   - Implement a simple **Bitmap Allocator** or **Buddy Allocator** that marks physical 4 KB frames between `_kernel_end` and available RAM bounds as free or used.
2. **Virtual Memory (SV39 Paging):**
   - Three-level page tables (VPN[2], VPN[1], VPN[0]).
   - Map kernel code/data identity-mapped (`VA == PA`) or higher-half.
   - User-space addresses mapped from `0x0` up to user boundaries.
   - Setting `satp` register and flushing via `sfence.vma`.
3. **Kernel Heap (Byte Allocation):**
   - Implement a **SLAB** or simple **Freelist/First-Fit** allocator on top of allocated pages to back dynamic arrays and objects.

---

## Stage 4: Traps, Context Switching, & Scheduling
- **Interrupts and Exceptions:** Configure `stvec` (Supervisor Trap Vector Base Address) and handle `ecall` (system calls), timer interrupts (`sie`, `sip`), and page faults (`stval`, `scause`).
- **Context Switching:** Write assembly trampolines saving register state (`x1-x31`) onto a process control block (PCB) trapframe and restoring user context with `sret`.
- **Preemptive Scheduler:** Configure the RISC-V timer (via SBI `sbi_set_timer` or CLINT) to fire regular ticks for a round-robin scheduler.

---

## Stage 5: Containerization Primitives for Edge
To isolate lightweight edge workloads without a heavy hypervisor, your kernel needs:
1. **Process Namespaces:**
   - **PID Namespace:** Virtualized process IDs where container init sees itself as PID 1.
   - **Mount Namespace:** Each container has an isolated view of the VFS mount table.
   - **Network/IPC Namespace:** Separate loopback and socket interfaces.
2. **Resource Boundaries (Mini-cgroups):**
   - Track ticks consumed per process group in the scheduler. Throttle processes exceeding assigned CPU quota or kill those breaching max frame allocation.
3. **Copy-on-Write (CoW) Memory & RootFS:**
   - Container image layering: Mount a base read-only file system tree and attach an ephemeral writable scratch space.

