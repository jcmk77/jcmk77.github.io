---
title: "High-Performance Computing in CPU FPGA GPU"
date: 2026-05-01T11:35:00-06:00
Description: "A retrospective on the High-Performance Computing labs, from OpenMP GEMM to Vitis HLS on FPGA and CUDA image blur."
Tags: ["ENSC453", "OpenMP", "AVX2", "FPGA", "Vitis HLS", "CUDA", "Performance"]
Categories: ["Hardware Acceleration"]
DisableComments: false
draft: false
---

## Hardware Acceleration is ready
This is a brief learning log documenting my study of hardware acceleration. The topic appealed to me because it showed strong potential for improving computational performance and helped me better understand the importance of AI infrastructure. The project consists of fiver labs, primarily focused on matrix multiplication and optimization.

What ties all five labs together is one lesson: **raw parallelism is never enough by itself**. Every time the performance improved in a major way, it was because the design got better at moving data, reusing data, or avoiding unnecessary synchronization.

## Lab 1: Parallelism Helps, But Synchronization Still Matters

Lab 1 is the simplest report in the folder, but it already contains the core theme of the entire sequence. The baseline GEMM is parallelized with OpenMP in two different ways.

The first strategy is straightforward: parallelize the outer `i` loop with a single `#pragma omp parallel for`. That version scales reasonably well. The report shows runtime dropping from about **17.26 s** on one thread to **2.52 s** on 20 threads, for a speedup of about **6.86x**. It is not ideal scaling, but it is stable and easy to explain. Each thread gets full rows of `C`, so the work is coarse-grained and the synchronization overhead is low.

The second strategy parallelizes both `j` loops inside a surrounding OpenMP region. In theory, that sounds more aggressive. In practice, it is worse. The report shows that by 20 threads it only reaches about **5.77x** speedup, and the behavior becomes less stable. The explanation is good and very practical: every `omp for` ends with an implicit barrier, and with `NI = 2048`, the program pays that barrier cost again and again. Once thread counts climb and hybrid-core scheduling starts to matter, the slowest thread holds everybody back.

That is a good first lab lesson because it cuts against the naive idea that “more parallel directives means more performance.” It does not. Sometimes fewer synchronization points matter more.

## Lab 2: The Big CPU Speedups Came From Locality, Not Threads

Lab 2 is where the project becomes much more serious. The base runtime for the larger GEMM is listed as **506.74 s**, and the report does a good job identifying the real problem: the original loop order accesses the `B` matrix with a **16 KiB stride**, which is terrible for cache-line use.

The report walks through three tiling strategies:

- `TM1`: 1-D tiling
- `TM2`: 2-D tiling with loop permutation
- `TM3`: a more refined tiled structure with better reuse of `A`, `B`, and `C`

The important result is not just that tiling helps. It is that **different tile shapes change which memory bottleneck dominates**. By the time they get to `TM3`, the team is already thinking in the right way: not just “block it,” but “which loop should get the large tile, which loops should stay small, and how much reuse can I get before eviction risk takes over?”

Their best single-thread tile strategy reaches about **26x** speedup. Then explicit SIMD via `#pragma omp simd` pushes that to about **68x**, bringing runtime down to **7.45 s**. After adding OpenMP on top of the tiled SIMD kernel, runtime falls to **0.84 s**, roughly **603x** faster than the baseline. The final step is an AVX2 micro-kernel using intrinsics like `_mm256_loadu_ps`, `_mm256_mul_ps`, and `_mm256_add_ps`, which brings runtime down further to about **0.55 s**, or roughly **917x** speedup.

That jump is enormous, but it is not magic. The report makes it clear that the win came in layers:

- fix locality first
- vectorize the contiguous dimension
- parallelize after the working set makes sense
- only then drop into intrinsics for the last major gain

That ordering matters. The AVX2 code works because the data layout and loop order were already prepared for it.

## Lab 3: The Same Ideas Reappear On FPGA, Just More Explicitly

Lab 3 moves into Vitis HLS on an **Alveo U50 FPGA**, and this is where the course shifts from “optimize code” to “design a hardware-friendly data path.” The report frames the problem clearly: the matrices are **4096 x 4096**, so the data lives off-chip and must be pulled into manageable tiles. That immediately leads to a load/compute/store structure with on-chip buffers.

This part of the folder is useful because it shows how much more explicit optimization becomes in HLS. On CPU, we talk about cache locality. On FPGA, we declare the buffers ourselves. On CPU, the compiler decides a lot about memory behavior. On FPGA, we decide how tiles get loaded, when scaling by alpha and beta happens, what gets pipelined, and which arrays get partitioned.

The report’s progression is sharp:

- baseline estimated absolute latency: about **1960 s**
- load/compute/store tiled buffering: about **167.9 s**
- pipelined load/store: roughly the same latency, but a better structure for scaling
- optimized compute with a **16 x 32** parallel micro-kernel: about **0.91 s**

That last step is the real leap. The design uses unrolling, pipelining with `II=1`, array partitioning, and DSP-bound floating-point operations to create a hardware pipeline that updates a large output micro-tile in parallel while still staying inside the lab’s resource limits. The report lists the final resource use at about **48% BRAM**, **43% DSP**, **24% FF**, and **43% LUT**, which is a good reminder that FPGA optimization is always a negotiation between latency and silicon budget.

Lab 3 is probably the clearest place where the course stops feeling like software optimization and starts feeling like architecture.

## Lab 4: Performance Is Not Enough Until The Whole FPGA System Works

Lab 4 extends the Lab 3 kernel into a fuller acceleration pipeline. This is where the folder becomes especially interesting because it is no longer just about C-synthesis results. It is about **system integration**.

The first big idea is **memory coalescing through 512-bit AXI bus widening**. Instead of moving one 32-bit float at a time, the design packs **16 floats** into `ap_uint<512>` transfers and manually unpacks them into on-chip buffers. The report notes that this alone gives about **1.69x** additional speedup relative to Lab 3, with estimated absolute latency improving to about **0.537 s**.

The second big idea is **manual ping-pong buffering**. Two `A` buffers and two `B` buffers are alternated so that one pair can be loaded while the other pair is being consumed by `compute_gemm()`. The report compares the overlapped loop latency against the sum of the component latencies and shows that the overlap is real, not just something assumed from the code structure. That optimization yields another meaningful gain, landing around **0.559 s** on its own relative path.

Combined, the two ideas push the estimated latency to about **0.507 s**. That is only part of the story, though. Lab 4 also covers the full hardware acceleration development cycle:

- `sw-emu`: functional CPU-side emulation
- `hw-emu`: RTL-level hardware emulation
- real hardware execution on the **Alveo U50**

The actual hardware run reported in the document is about **1152.23 ms**, with post-place-and-route estimates around **0.517 s**. That gap is useful. It reminds me that end-to-end performance includes more than the kernel’s internal schedule. Tool flow, memory migration, runtime overhead, and system-level realities all matter.

Lab 4 is where the project stops being an HLS exercise and starts looking like actual accelerator deployment work.

## Lab 5: Different Algorithm, Same Core Lessons

Lab 5 looks like a detour at first because it leaves GEMM and switches to a **CUDA image blur** kernel on a **3840 x 2160** grayscale image with blur radius **21**. But conceptually it belongs in the same sequence.

The baseline runtime is about **0.364 s** including data transfer. The first optimization is not just “use shared memory.” The better idea is to reorganize the blur computation into phases:

1. load a halo-padded tile into shared memory
2. compute partial column sums
3. reuse those sums to form final pixel values

That arithmetic restructuring matters as much as the memory placement. The report explicitly says that the big speedup was not only from moving data from global to shared memory, but from **reusing partial sums** so the same work was not repeated unnecessarily.

The optimized kernel reaches about **0.139 s**, which is a **2.5x** speedup over the baseline. Then the host-device path is improved by registering existing host buffers with `cudaHostRegister()`, allowing the GPU DMA engine to move pinned memory more efficiently. That cuts runtime further to about **0.113 s**, giving roughly **1.23x** improvement over the already optimized kernel and about **3.22x** over baseline.

I also liked that the report spends real time on **control divergence** instead of pretending it does not matter. It estimates where divergence happens, why it is mostly confined to edge conditions, and how `__syncthreads()` protects the shared-memory phases from overlapping incorrectly. That is exactly the kind of detail that separates “I wrote a CUDA kernel” from “I understand why this CUDA kernel behaves the way it does.”

## What Five Labs Say

After reading all five reports, my main takeaway is that the labs are not really five separate stories. They are one story told across different platforms.

Lab 1 says: thread-level parallelism is useful, but synchronization overhead can kill it.  
Lab 2 says: locality and vectorization matter more than adding threads blindly.  
Lab 3 says: on FPGA, you must build locality and reuse explicitly with buffers and pragmas.  
Lab 4 says: kernel optimization is only half the job; the memory interface and deployment flow matter too.  
Lab 5 says: even on CUDA, the biggest wins still come from structuring data reuse and reducing movement.

So the real subject of `labs_GEMM` is not just GEMM. It is the discipline of asking the same question on every platform:

**Where is the data, how often am I moving it, and how much work do I get out of each movement?**

That is why the folder is worth reading as a whole. The syntax changes from OpenMP to AVX2 to HLS pragmas to CUDA. The hardware changes from CPU caches to FPGA BRAM/URAM to GPU shared memory. But the core reasoning stays surprisingly constant.

That consistency is probably the best thing ENSC 453 seems to teach.

**{Related Courses}**: ENSC 453 Parallel and Heterogeneous Computing, Computer Architecture, Embedded Systems, High Performance Computing
