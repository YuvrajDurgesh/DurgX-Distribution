# DurgX Operating System

<div align="center">

**A High-Reliability, Capability-Based Asynchronous Microkernel for x86_64**

[![Status](https://img.shields.io/badge/Status-Architecture_v0.1_(Pre--Alpha)-informational?style=flat-square)](#status)
[![License](https://img.shields.io/badge/License-Proprietary_Core-blue?style=flat-square)](#license)
[![Boot Protocol](https://img.shields.io/badge/Boot-Limine_(UEFI%2FBIOS)-success?style=flat-square)](#boot-sequence)
[![Architecture](https://img.shields.io/badge/Target-x86__64_(ARM64_portable)-orange?style=flat-square)](#architecture-highlights)
[![Website](https://img.shields.io/badge/Docs_%26_Sandbox-galaxyium.space-purple?style=flat-square)](https://galaxyium.space)

</div>

---

## 1. Overview

**DurgX** is an experimental, modern microkernel engineered around pure capability-based security, rigorous hardware fault isolation, and zero-copy asynchronous messaging. 

Unlike monolithic kernels where a single faulty device driver or memory corruption can bring down the entire system, DurgX confines all device drivers, network stacks, filesystems, and POSIX layers to **Ring 3 user-space servers**. Faults within user-space drivers trigger structured supervision events rather than kernel panics.

### Primary Use Cases
- **Edge Autonomy & Robotics:** Hard real-time guarantees, bounded fault domains, and non-blocking sensor loops.
- **Aerospace & CubeSats:** In-flight component recovery where kernel panics are catastrophic.
- **Confined Secure Enclaves:** Strict principle of least authority with zero implicit `root` privileges.

---

## 2. Architectural Invariants & Guarantees

```
┌───────────────────────────────── Ring 3 (User Space) ─────────────────────────────────┐
│                                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  ┌────────────────────────┐    │
│  │  NVMe Driver │  │  GPU Driver  │  │  Net (e1000e) │  │  POSIX / Linux Shim    │    │
│  │  (Cap-scoped)│  │  (Cap-scoped)│  │  Driver Task  │  │  (vDSO, glibc / musl)  │    │
│  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘  └────────────┬───────────┘    │
│         │                  │                   │                       │              │
│  ┌──────┴──────────────────┴───────────────────┴───────────────────────┴───────────┐ │
│  │     VFS Server  │  Network Stack Server  │  Device Manager  │  Root Scheduler   │ │
│  └──────────────────────────────────┬──────────────────────────────────────────────┘ │
│                                     │ IPC (Shared-Memory Rings, Capability-Gated)    │
└─────────────────────────────────────┼────────────────────────────────────────────────┘
                                      │ SYSCALL / SYSRET
┌─────────────────────────────────────┼───────────────────────── Ring 0 (Nucleus) ─────┐
│  ┌─────────────┐  ┌────────────────┐│ ┌───────────────┐  ┌─────────────────────────┐  │
│  │ Capability   │  │ VMM / Paging   ││ │ Scheduler     │  │ IPC Transport           │  │
│  │ Invocation   │  │ (HHDM, PCID)   ││ │ (Chase-Lev    │  │ (Ring Registration,     │  │
│  │ Dispatcher   │  │                ││ │  Work-Steal)  │  │  Futex Wake)            │  │
│  └─────────────┘  └────────────────┘│ └───────────────┘  └─────────────────────────┘  │
│  ┌─────────────┐  ┌────────────────┐│ ┌───────────────┐                               │
│  │ PMM (Buddy)  │  │ IRQ Vectoring  ││ │ Timekeeping   │   Exhaustive Ring 0 logic.   │
│  │ (NUMA-Aware) │  │ (MSI-X to Cap) ││ │ (TSC-Deadline)│   No filesystems or drivers. │
│  └─────────────┘  └────────────────┘│ └───────────────┘                              │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

### 1. No Implicit Authority
There is no `root` account or ambient authority. Access to physical memory frames, I/O ports, interrupt lines, or peer services requires possessing an unforgeable 128-bit **Capability handle** (`object_id`, `rights`, `generation`).

### 2. Policy-Free Ring 0 Nucleus
Ring 0 contains only primitives that strictly mandate hardware privilege (`CR3`, `IDT`, `IA32_TSC_DEADLINE`, capability resolution, and runqueue primitives). Device enumeration, scheduling heuristics, filesystem arbitration, and networking are unprivileged servers.

### 3. Structural Fault Isolation (Erlang-Style Supervision)
If a driver faults (e.g., page fault, GPF), Ring 0 isolates the offending Thread Control Block (TCB) and transmits a capability fault packet to its registered supervisor task (`Device Manager`). The supervisor decides whether to restart the driver, replay in-flight block transactions, or quarantine the device—**without halting the OS**.

### 4. Zero-Copy IPC Fast Path
DurgX provides two communication models:
* **Synchronous Rendezvous:** Small control messages ($\le 4$ registers passed directly via CPU registers with zero memory access).
* **Asynchronous Rings (SPSC/MPSC):** Lock-free submission and completion rings backed by granted physical frames for high-throughput block and network I/O.

### 5. Deterministic Memory Accounting
The kernel contains no runtime dynamic memory allocator. All kernel-managed objects (`TCB`, `PageTable`, `Endpoint`) are explicitly retyped from user-supplied `Untyped` memory capabilities, eliminating out-of-memory kernel panics.

---

## 3. Quick Start: Running DurgX

We distribute pre-compiled, bootable `.iso` images powered by the **Limine Boot Protocol** for evaluation in virtualized environments.

### Step 1: Download & Verify

Download the pre-alpha ISO from our [Releases](https://github.com/galaxyium/durgx/releases) section.

Verify image integrity using SHA-256:

```bash
# Linux / WSL
sha256sum durgx.iso

# macOS
shasum -a 256 durgx.iso

# Windows (PowerShell)
Get-FileHash .\durgx.iso -Algorithm SHA256
```

Compare the resulting hash against the value published on the release tag.

---

### Step 2: Booting with QEMU

DurgX outputs diagnostics directly over the serial interface. Run the image with serial redirection enabled:

#### Standard x86_64 Emulation (BIOS / Direct Boot)
```bash
qemu-system-x86_64 \
  -cdrom durgx.iso \
  -m 512M \
  -smp 4 \
  -serial stdio \
  -display none
```

#### UEFI Boot (Requires OVMF)
```bash
qemu-system-x86_64 \
  -bios /usr/share/ovmf/OVMF.fd \
  -cdrom durgx.iso \
  -m 1G \
  -smp 4 \
  -serial stdio
```

#### Hardware Acceleration (KVM / Hypervisor)
To run with native host performance on Linux:
```bash
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -cdrom durgx.iso \
  -m 1G \
  -smp cores=4 \
  -serial stdio
```

---

## 4. Subsystem Specifications

| Subsystem | Specification Details |
| :--- | :--- |
| **Boot Protocol** | Limine Request Protocol (MemMap, HHDM, RSDP, Framebuffer) |
| **Physical Memory (PMM)** | NUMA-aware Buddy Allocator ($4\text{ KiB}$ to $4\text{ MiB}$ blocks) |
| **Virtual Memory (VMM)** | 4-Level / 5-Level Paging, Higher-Half Direct Map (HHDM), PCID/ASID support |
| **SMP Bring-up** | ACPI MADT parsing $\rightarrow$ LAPIC/IOAPIC configuration $\rightarrow$ INIT-SIPI-SIPI sequence |
| **Scheduler** | Cache-aware, lock-free work-stealing (Chase-Lev deques) with strict RT-EDF queues |
| **Binary Compatibility** | Ring 3 POSIX/Linux translation shim with user-space vDSO acceleration |

---

## 5. Interactive Portal & Documentation

For interactive visual sandboxes, live crash recovery demonstrations, and detailed architectural write-ups:

👉 **Visit the Official DurgX Hub:** [galaxyium.space](https://galaxyium.space)

---

## 6. Status & Roadmap

DurgX is currently at **Architecture Specification v0.1 (Pre-Alpha)**. 

- [x] **Phase 0:** Microkernel Specifications, Capability Model Formalization
- [x] **Phase 1:** Limine Boot, HHDM Setup, Serial Debugging, Trap Frame ABI
- [ ] **Phase 2:** Buddy Allocator (PMM) & Permanent Virtual Memory Maps (VMM)
- [ ] **Phase 3:** Capability Radix Tree & Retype Invocations
- [ ] **Phase 4:** Lock-Free Ring IPC & Synchronous Rendezvous
- [ ] **Phase 5:** SMP Bring-up & Work-Stealing Scheduling
- [ ] **Phase 6:** Ring 3 Driver Supervisor Subsystem & Fault Recovery
- [ ] **Phase 7:** POSIX Shim Server & Dynamic Linker Integration

---

## 7. License & Disclaimers

The pre-compiled distributions, technical documentation, and interface specifications are published for research, benchmarking, and security evaluation under the stewardship of **[Galaxyium](https://galaxyium.space)**.

*Notice: Interfaces, capability structures, and system call ABI definitions are subject to change during pre-alpha development.*