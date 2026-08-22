+++
title = 'GSoC 2026 Final Report: Modular Heap Allocators and TLSF Integration in RTEMS'
date = 2026-08-14T12:00:00+00:00
draft = false
tags = ['GSoC', 'RTEMS', 'TLSF', 'Memory Allocation', 'C', 'Real-Time Systems', 'Open Source']
categories = ['Development', 'Systems Programming']
+++

# GSoC 2026 Final Report: Modular Heap Allocators and TLSF Integration in RTEMS

## Project Summary

My Google Summer of Code 2026 project focused on making the RTEMS heap architecture modular and adding the [Two-Level Segregated Fit (TLSF)][tlsf] allocator as an alternative heap implementation. RTEMS (Real-Time Executive for Multiprocessor Systems) is a real-time operating system used in embedded and mission-critical systems.

RTEMS currently uses its first-fit Heap Allocator for dynamic memory allocation. The goal of this project is to let RTEMS use different heap implementations through a common allocator interface without requiring application-level code to depend on a specific backend. TLSF was selected as the first alternative backend because it organizes free blocks using two-level size classes and bitmaps, allowing suitable blocks to be located without scanning a linear free list.

The project is organized as four merge-request units:

1. Separate first-fit-specific helpers from the generic RTEMS heap header.
2. Add and test a TLSF implementation suitable for RTEMS targets.
3. Design an `rtems_allocator` abstraction that can dispatch a common set of operations to different heap backends.
4. Add a benchmark that compares first-fit and TLSF through the common allocator interface.

**Project Proposal:** [Proposal](https://docs.google.com/document/d/1Usw6m2bPfDzruv8gIzJPusDf5P4E9TAle3y3lxFMgi8/edit?tab=t.0#heading=h.qwofntymv246)  
**GitLab Activity:** [My contribution history](https://gitlab.rtems.org/users/mez3n/activity)  
**First-Fit Refactoring Issue:** [rtems/rtos/rtems#5620](https://gitlab.rtems.org/rtems/rtos/rtems/-/work_items/5620)  
**TLSF Tracking Issue:** [rtems/rtos/rtems#5633](https://gitlab.rtems.org/rtems/rtos/rtems/-/work_items/5633)  
**Allocator Integration Issue:** [rtems/rtos/rtems#5619](https://gitlab.rtems.org/rtems/rtos/rtems/-/work_items/5619)  
**Related Forum Discussions:** [Heap refactoring discussion](https://users.rtems.org/t/rtems-heap-refactoring-discussion/879), [alternative memory allocators](https://users.rtems.org/t/alternative-memory-allocators/597), and [allocator configuration](https://users.rtems.org/t/choosing-the-configuration-method-for-rtems-allocator/896)

## Work Completed

The work was organized into four separate merge requests so that each architectural step can be reviewed independently. The heap refactor has been merged, the TLSF implementation is currently under review, the allocator abstraction is nearly complete, and the benchmark is in progress.

## Refactoring the Existing Heap Implementation

The first part of the project prepared the existing RTEMS Heap Handler for multiple backends. Previously, `heapimpl.h` contained both heap helpers and details that belonged specifically to the first-fit implementation.

I split these responsibilities by moving first-fit-specific definitions and helpers into `heapfirstfitimpl.h`. The generic `heapimpl.h` now contains the common heap declarations, while first-fit source files include the new implementation-specific header where necessary.

**Merge Request:** [MR !1308 - Refactor heapimpl.h](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/1308)

This merge request has been merged. It provides a cleaner boundary between the public Heap Handler interface and its current first-fit backend, which is an important foundation for the rest of the project.

## TLSF Allocator Implementation

The second part of the project added a TLSF allocator to RTEMS. TLSF uses two levels of size classes. The first level identifies a broad block-size range, and the second level divides that range into smaller classes. Bitmaps indicate which classes contain free blocks, and each class has a free list. This structure makes allocation lookup more predictable than walking a linear first-fit free list.

I used the [mattconte/tlsf](https://github.com/mattconte/tlsf) implementation as a reference while developing the RTEMS TLSF allocator.

**Merge Request:** [Draft MR !1395](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/1395) - Add TLSF implementation with tests

The implementation includes:

- TLSF control data, first- and second-level bitmaps, and segregated free lists.
- Pool creation, addition, removal, and physical end sentinels.
- Allocation, aligned allocation, deallocation, block splitting, and coalescing.
- Reallocation and allocation-size queries.
- Heap and pool consistency checks.
- RTEMS-style initialization, allocation, free, and usable-size query interfaces.
- Configurable compile-time heap alignment based on `CPU_HEAP_ALIGNMENT`.
- Support for both 32-bit and 64-bit pointer widths.

The implementation exposes the classic TLSF-style interface for direct use and RTEMS-compatible entry points that can later be connected to the generic allocator layer. At present, TLSF is included explicitly by users; it has not yet replaced the allocator used implicitly by the RTEMS application allocation APIs.

### Alignment and Portability

Supporting different target architectures required more than changing integer types. Block-header layout, flag bits, size limits, bitmap operations, and alignment arithmetic all need to remain correct for both 32-bit and 64-bit systems.

The allocator now derives its fundamental alignment from `CPU_HEAP_ALIGNMENT` and supports the alignment values accepted by the implementation. Individual allocations may request a larger runtime alignment through the aligned-allocation interface.

I tested the implementation using QEMU on two RTEMS target configurations: `arm/xilinx_zynq_a9_qemu`, which is a 32-bit ARM target, and `riscv/rv64imafdc`, which is a 64-bit RISC-V target. This verifies the allocator on both 32-bit and 64-bit architectures.

### TLSF Tests

Two dedicated test suites were added:

- `heaptlsf01` exercises the lower-level TLSF interface, including heap creation, pool handling, basic and aligned allocation, free-block coalescing, exhaustion, reallocation, pool removal, and consistency checks.
- `heaptlsf02` exercises the RTEMS-compatible interface, including initialization, aligned allocation, boundary validation, freeing memory, exhaustion behavior, and usable-size queries.

## The `rtems_allocator` Abstraction

The third part of the project introduces an abstraction layer between callers and heap implementations. Its design is similar to other RTEMS subsystems that dispatch generic operations through a backend-specific operations structure.

The implementation is nearly complete on the `adding-rtems-allocator-structure` branch. A merge request has not yet been opened because the final startup, configuration, and interface design choices still need to be agreed upon.

The current structure contains:

- An `rtems_allocator` object containing an operations pointer and backend context.
- A common operations structure covering the available first-fit Heap Handler operations.
- Generic inline functions that dispatch operations to the selected backend.
- A common status type for translating backend-specific results.
- Correctly typed unsupported-operation stubs for allocators that cannot implement every operation.
- A first-fit adapter that connects the existing RTEMS Heap Handler to the new interface.
- Tests for the generic operations, the first-fit adapter, and unsupported-operation behavior.

This design keeps the caller independent of `Heap_Control` or `Heap_Tlsf_Control`. Each heap instance has its own backend context, while all instances using the same allocator implementation can share the same constant operations structure.

## RTEMS Heap Configuration

RTEMS normally has two important heap instances: `_Workspace_Area`, used for internal RTEMS objects, and `RTEMS_Malloc_Heap`, used by application allocation functions such as `malloc()`. With unified work areas enabled, both purposes share one memory area.

The remaining configuration question is how an application should select the allocator backend at startup and obtain the configured allocator object. The likely direction is to select one backend implementation for the application while allowing the workspace and malloc heaps to remain separate allocator instances when required. This avoids a runtime registry of allocator backends while preserving the existing two-heap organization.

After this design is finalized, the remaining integration work is to add a TLSF adapter, connect the configured allocators to the RTEMS workspace and application allocation paths, and verify both separate and unified work-area configurations.

## Allocator Benchmark

I am currently working on the fourth merge request, which will compare TLSF with the existing first-fit allocator in terms of allocation latency and fragmentation. I plan to adapt the workload structure of [mimalloc-bench's alloc-test](https://github.com/daanx/mimalloc-bench/tree/master/bench/alloc-test), particularly its allocation-size distribution and sequence of allocation and free operations.

The RTEMS benchmark will use the common `rtems_allocator` interface, deterministic workloads, RTEMS timing facilities, and fragmentation measurements that can be applied equally to both backends. Its development is in progress, but its merge request is delayed because it depends on the `rtems_allocator` branch and cannot be finalized until that interface is merged.

## Changes from the Original Proposal

The original proposal also included exploring and implementing RT-Mimalloc. This part was explicitly identified as an exploratory research effort because of its complexity and the uncertainty involved in adapting it to RTEMS.

During the coding period, the heap refactoring, TLSF implementation, portability work, testing, and allocator abstraction required more time than initially expected. I therefore prioritized establishing a correct and reviewable foundation for modular heap support instead of beginning an RT-Mimalloc implementation that could not be completed or properly tested within the available time.

The `rtems_allocator` abstraction is nearly finished, but completing it was delayed by the need to resolve architectural questions about startup configuration, backend selection, and how the workspace and malloc heap instances should obtain their allocator objects. The implementation, first-fit adapter, unsupported-operation handling, and tests are already present on the [`rtems_allocator` branch](https://gitlab.rtems.org/mez3n/rtems/-/tree/adding-rtems-allocator-structure?ref_type=heads). The remaining work is mainly to finalize these design decisions and make any resulting adjustments.

RT-Mimalloc remains a possible direction for future work after the allocator abstraction and TLSF integration are finalized. The abstraction developed during this project should make it easier to add and evaluate allocators such as RT-Mimalloc in the future.

## Current State

| Merge-request unit | Status | Reference |
|--------------------|--------|-----------|
| First-fit heap header refactor | Implemented and merged | [MR !1308](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/1308) |
| TLSF allocator and tests | Implemented; draft MR open with passing CI | [MR !1395](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/1395) |
| Generic `rtems_allocator` interface | Nearly complete; final configuration design pending | [`rtems_allocator` branch](https://gitlab.rtems.org/mez3n/rtems/-/tree/adding-rtems-allocator-structure?ref_type=heads) |
| First-fit versus TLSF benchmark | In progress; depends on the `rtems_allocator` branch | MR not opened yet |

## Challenges and Lessons Learned

This project required a large amount of low-level debugging. Allocator bugs can corrupt metadata and only become visible during a later allocation or free operation, so small mistakes in block sizes, flag handling, alignment, or sentinel initialization can be difficult to trace.

One important lesson was that modularity does not necessarily mean sharing every internal helper. Sharing algorithm-independent utilities is useful, but forcing implementations with different metadata and invariants to share internal block operations can make both implementations harder to understand and review. A small, stable interface between independent backends provides cleaner modularity.

I also gained a better understanding of portable systems code. Supporting both 32-bit and 64-bit targets required auditing arithmetic, bitmap widths, size limits, and memory layout rather than assuming that code working on one target would automatically work on another.

## Conclusion

The project established the main building blocks for modular heap allocation in RTEMS. The existing first-fit implementation now has a clearer implementation boundary, a working TLSF allocator and its tests are available for review, and the nearly complete `rtems_allocator` abstraction demonstrates how different heap backends can provide a common set of operations.

**Key Achievements:**

- Refactored the existing Heap Handler and merged the change into RTEMS.
- Implemented a TLSF allocator with pool management, aligned allocation, reallocation, consistency checks, and RTEMS-compatible interfaces.
- Added dedicated TLSF tests and verified the implementation on 32-bit and 64-bit targets.
- Nearly completed the generic allocator interface, including the first-fit adapter, status translation, unsupported-operation stubs, and tests.
- Defined the remaining path for startup configuration, TLSF integration, and comparative benchmarking.

I want to thank my mentors and the RTEMS community for their guidance, reviews, and discussions throughout the project. The work was more challenging than I initially expected, but it greatly improved my understanding of memory allocators, RTEMS internals, portable C, and the design of modular operating-system components.

I also want to thank Google for providing this opportunity. I plan to continue working with the RTEMS community after GSoC to review and integrate the TLSF implementation, finalize the allocator architecture, and complete the remaining integration and benchmarking work.

[tlsf]: http://www.gii.upv.es/tlsf/
