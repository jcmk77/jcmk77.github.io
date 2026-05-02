---
title: "Hardware Acceleration: Likelihood Kernel"
date: 2026-05-01T19:10:00-06:00
Description: "A retrospective on the likelihood-kernel project, and how an irregular particle-filter workload had to be reshaped before FPGA and GPU acceleration really made sense."
Tags: ["ENSC453", "FPGA", "GPU", "Vitis HLS", "CUDA", "OpenMP", "Performance"]
Categories: ["Hardware Acceleration"]
DisableComments: false
draft: false
---

## The Hard Part Wasn't the Math, It Was the Memory 
TThis project focused on hardware acceleration for a likelihood kernel, a type of computation commonly used in AI, robotics, and automation workloads. What stayed with me was not simply that the project accelerated a kernel, but that it documented a more important realization: **the likelihood math was never the real problem. The memory pattern was.**

On paper, the kernel looks manageable. For each particle, drop a radius-5 circular mask over the image, inspect the **69** pixels under that mask, and compute a likelihood value. The project fixes the image at **4000 x 4000**, pushes the particle count up to **3 million**, and compares three implementations: a CPU baseline, a GPU version, and an FPGA design. But the hard part is that particles are scattered randomly, so the pixel accesses are scattered too. That means the kernel is not naturally a clean streaming workload. It is an irregular gather problem pretending to be a compute problem.

That distinction ends up explaining almost everything in the folder.

## The Most Important Part Of The Project Is The Deviation From Plan

The strongest section in the report is not the optimization summary. It is the part where the team admits the original plan did not work.

The initial FPGA direction tried to preserve the original Rodinia-style behavior directly on the device: calculate particle indices, map the mask offsets into the image, and gather the required pixels from scattered locations in DDR. In theory, that sounds faithful to the algorithm. In practice, it was a bad fit for the FPGA. The report is blunt about it: the random accesses created a massive bottleneck, the design was slower than the CPU and GPU, and later optimizations like ping-pong buffering could even make things worse.

That is the turning point of the whole project.

Instead of forcing the FPGA to solve the worst possible version of the problem, the team changed the shape of the problem itself. The host side began **pre-extracting each particle's masked pixels** into a clean sequential layout before the kernel ran. In the code, that shows up as a packed stream where each particle contributes **80 integers**: the real **69** masked pixels plus **11 zeros** of padding so the data aligns cleanly to **512-bit** memory words. Once that transformation happens, the FPGA kernel no longer performs irregular image gathers at all. It just consumes a regular particle-major stream and computes likelihoods.

That move is the deepest idea in the folder. The project stopped asking, "How do we accelerate the original access pattern?" and started asking, "How do we reorganize the data so acceleration becomes possible?"

## The Math Gets Simpler Once The Data Is In The Right Shape

Another good design choice appears in both the report and the source: the likelihood expression is algebraically simplified.

The original log-likelihood computation is framed in terms of squared differences from background and foreground means. But once the constants are expanded, the inner loop no longer needs full quadratic arithmetic per pixel. It can be reduced to a linear form:

`likelihood = scale * pixelSum - bias`

That is exactly what the FPGA kernels do. They accumulate integer pixel values first, then apply the scale and bias afterward. In the code, the constants collapse to the same values used across the different optimization stages:

- `scale = (256 / 50) * (1 / 69)`
- `bias = 41984 / 50`

This matters because it strips the kernel down to what hardware actually wants: a regular reduction followed by a simple affine transform. The reports describe this as a compute optimization, but it is really part of the same broader theme. The project keeps removing irregularity, overhead, and unnecessary structure until the problem finally looks like hardware.

## The Repository Preserves The FPGA Story In Stages

One thing I liked about this folder is that the FPGA path is preserved as a sequence of concrete directories rather than a single final kernel. The names tell the story directly:

- `buffer`
- `comp-opt`
- `mem-opt`
- `ping-pong`
- `hw-emu`
- `hw-run`

That sequence makes the optimization staircase unusually easy to read.

The `buffer` version is the first honest hardware decomposition. It splits the kernel into **load, compute, and store** stages, tiles work into blocks of **128 particles**, and introduces on-chip `buffer_pixels` and `buffer_likelihood` arrays. Load and store are pipelined with `II=1`, but the compute stage is still too weak to matter much. The report gives this version an estimated latency of **1.503 s**, only **0.16x** the CPU benchmark. In other words: buffering alone does not save a fundamentally memory-bound design.

The `comp-opt` version attacks the math stage directly. It fully partitions the particle-sum array, completely partitions the particle dimension of `buffer_pixels`, unrolls across particles, and pipelines across mask offsets. That cuts compute latency sharply, but it still only reaches **0.828 s** and **0.29x** speedup. The report notes a **97%** drop in compute-step latency, which is real progress, but the whole kernel is still dragged down by moving data.

The biggest jump comes in `mem-opt`. This is where the project starts treating the external interface as part of the kernel design instead of a fixed cost. The arrays are widened to **512-bit AXI ports**, the **69** pixels are padded to **80** so each particle fits into exactly **5** wide words, and the load/store logic packs and unpacks data with `ap_uint<512>`. The report lists this version at **0.061 s** estimated latency and **3.89x** speedup. That is the first point where the FPGA actually beats the CPU benchmark on paper.

Then `ping-pong` adds task-level overlap through `#pragma HLS DATAFLOW`, letting the tool infer PIPO buffers between load, compute, and store. This improves the estimated latency again to **0.051 s** and **4.65x** speedup. What is especially interesting is that the report is careful not to romanticize this step. The slide deck later admits the benefit from ping-pong was limited because the stages were imbalanced and the design still waited on the slowest part of the pipeline.

That honesty makes the project more useful. It shows not just what optimization was added, but whether it actually changed the critical path.

## The Host Code Tells The Truth About The Accelerator Boundary

The final `hw-run/host.cpp` is one of the most revealing files in the repo.

It does not just launch a kernel. It builds the real contract between host and FPGA. The host allocates the global arrays, initializes the synthetic image and particle locations, gathers each particle's masked pixels on the CPU, packs them into wide words, and then feeds the FPGA in chunks of **65,536 particles**. On the way back, it unpacks the results and accumulates the total likelihood.

That chunking code is not incidental. It is the practical expression of the project's new architecture: the CPU handles the irregular gather, the FPGA handles the regular math stream.

The `hw-emu` side reinforces this. Its host uses a smaller **512 x 512** image and an emulation chunk size of **512** particles so the team could test several cases without waiting forever for RTL co-simulation. The test scenarios are not decorative either. They check exact tile alignment, an unaligned tail with boundary conditions, and a multi-chunk stress case. That tells me the team was not satisfied with getting one happy-path demo. They were explicitly validating that the stream-oriented redesign behaved correctly at tile boundaries and across repeated launches.

This is also where the folder becomes very credible. It does not hide the preprocessing behind vague language. It makes the boundary between "host work" and "device work" explicit in the code and in the report.

## The Performance Numbers Are Good, But The Caveat Is More Important

The final comparison in the report is simple:

- **CPU:** 237 ms
- **FPGA:** 138 ms
- **GPU:** 7.08 ms

That gives speedups of:

- **FPGA:** **1.72x** over CPU
- **GPU:** **33.47x** over CPU

Those numbers are worth reporting, but the project is more interesting because it also explains what they do **not** mean.

The timings are kernel-focused, not end-to-end. The CPU timing excludes index precomputation. The GPU timing measures kernel execution, not the whole pipeline. The FPGA timing measures only execution on the **prepacked pixel stream**, not the host-side gather and packing that made the design feasible in the first place.

The report is also careful about synthesis optimism. The final FPGA hardware run landed around **138 ms**, while the post-place-and-route estimate was **71 ms** and the slide deck reports roughly **134 ms** in presentation form. That gap matters. High-level synthesis assumes friendlier memory behavior than real hardware delivers, especially once DDR and routing delays become real.

So the result is not "the FPGA is disappointing." The result is that once the workload is restructured into something streamable, the FPGA does accelerate it, but the GPU still dominates because this problem remains massively parallel and the GPU is simply better at hiding memory latency at scale.

## The GPU Side Is Mostly In The Reports, And That Says Something Too

One subtle detail of the folder is that the source tree is overwhelmingly about the FPGA path. The GPU strategy is preserved mainly in the written report and the slide deck rather than in a parallel CUDA source hierarchy.

Even so, the GPU story is clear. The report describes a design where one CUDA thread handles one particle, the mask offsets live in constant memory, and the work is split into two kernels: one for extraction and one for the final likelihood math. That second stage is the interesting part because it mirrors the FPGA insight. The GPU also performs better once it stops mixing irregular gathering with arithmetic and instead computes on a more regular intermediate buffer.

That parallel is worth noticing. The FPGA and GPU ended up converging on the same high-level answer:

1. separate the irregular gather from the arithmetic
2. regularize the data layout
3. let the accelerator consume a cleaner stream

The GPU just happens to be much better suited to the scale of the workload once that restructuring is done. With thousands of threads, strong memory bandwidth, cached irregular loads, constant-memory offsets, and pinned-memory support on the host side, it reaches **7.08 ms** where the FPGA finishes at **138 ms**.

## What I Took Away From The Folder

The obvious takeaway is that the GPU wins, the FPGA helps, and the CPU is the baseline. But that is the shallow version.

The deeper takeaway is that this project is really about **data-layout transformation as optimization**.

The project began with a naive idea: port the original likelihood kernel directly to hardware. It improved only after the team accepted that the algorithm, as originally expressed, was hostile to the FPGA. From that point forward, every important improvement followed the same pattern:

- simplify the likelihood math
- isolate the compute kernel from irregular access
- pack the data into a transport-friendly shape
- widen the memory interface
- overlap stages only after the stage boundaries are well defined

That is why I found this folder more compelling than a normal benchmark report. It preserves the part that many retrospectives skip: the moment where the original plan had to be abandoned because the machine was teaching a different lesson.

In the end, the likelihood kernel was not mainly a story about clever arithmetic or one magic pragma. It was a story about learning that the real accelerator design problem was upstream of the kernel itself.

The hard part was not the math. It was making the memory behavior look regular enough that the math could finally matter.

**{Related Courses}**: Parallel and Heterogeneous Computing, Computer Architecture, Embedded Systems, High Performance Computing
