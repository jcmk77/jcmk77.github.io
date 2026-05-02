---
title: "Hardware Acceleration: Likelihood Kernel" 
date: 2026-05-02T11:05:00-06:00
Description: "A retrospective on an ENSC 453 likelihood-kernel project, and how the real breakthrough came from redesigning the FPGA pipeline around data movement instead of forcing the original irregular access pattern onto the hardware."
Tags: ["ENSC453", "FPGA", "GPU", "OpenMP", "HLS", "CUDA", "Particle Filter", "Performance Engineering"]
Categories: ["Hardware Acceleration"]
DisableComments: false
draft: false
---

This project changed the way I think about optimization because it forced me to admit that my original acceleration plan was wrong.

**Likelihood Kernel can be used for AI, Robotics and Automation**

On paper, the project sounded straightforward enough: take the likelihood kernel from a particle-filter workload, accelerate it on FPGA and GPU, and compare both against a CPU baseline. In practice, the kernel was dominated by exactly the kind of memory behavior that makes clean-looking acceleration plans fall apart. Every particle needed to inspect a circular mask of pixels around its location, and those locations were scattered across a **4000 x 4000** image. That meant the arithmetic itself was not the real problem. The real problem was that the data access pattern was irregular, random, and hostile to the memory system.

That ended up becoming the real story of the project.

## I Started By Underestimating The Memory Problem

The kernel itself was conceptually simple. For each particle, I needed to evaluate a radius-5 circular mask, which gave **69 valid offsets** per particle, gather those pixel values, sum them, and compute a likelihood value. Because every particle was independent, it was tempting to think the hardware story would be mostly about exposing more parallelism.

That assumption did not survive contact with the FPGA.

Our initial idea was to replicate the original algorithm on the device as directly as possible. We built logic to calculate each particle's mapped indices and then gather the masked pixels from the full image in DDR. The problem was that these indices were scattered unpredictably. Two nearby particles in the particle array were not necessarily nearby in the image, and the FPGA kept paying the price for that mismatch.

The result was frustratingly clear: the design spent its time waiting on memory. The FPGA was slower than the CPU and GPU, and several "optimizations" did almost nothing because they were buried behind the same bottleneck. In some cases, adding more structure only made routing and scheduling harder without actually improving throughput.

That was the most important moment in the project for me. I had to stop treating the original plan as sacred. The design was not underperforming because the pragmas were wrong. It was underperforming because I was asking the FPGA to solve the wrong version of the problem.

## The Real Breakthrough Was Changing Where The Work Happened

Once I accepted that, the direction of the project changed.

Instead of forcing the device to perform the irregular image gather itself, I moved that part of the pipeline to the host side. The host and testbench preprocessed every particle's masked pixels and packed them into a clean stream where each particle became one row of contiguous values. After that, the FPGA kernel no longer had to chase random image addresses at all. It only had to consume a prepacked pixel stream and compute the likelihood values efficiently.

That decision felt like a deviation from the original spirit of the problem at first. Looking back, I think it was the most honest engineering choice I made in the project.

It also made the later optimizations meaningful. Before that pivot, most changes were hidden behind the cost of irregular memory access. After the pivot, I finally had a kernel whose structure matched the strengths of the hardware.

## The CPU Baseline Was Useful Because It Was Honest

On the CPU side, I refactored the baseline so it matched the FPGA and GPU math more closely, then parallelized across particles with OpenMP. That part was relatively clean because the particles were independent. I used `#pragma omp parallel for` with static scheduling on both the precomputation stage and the likelihood stage.

The optimized CPU version reached about **237 ms** with **6 threads** when measuring the likelihood computation phase. That was a reasonable baseline, but it also highlighted the limits of the CPU for this workload. Even though CPUs handle irregular access better than accelerators in many cases, they still do not have the same degree of parallel throughput. For this project, the CPU became less interesting as an endpoint and more useful as a sanity check: if my accelerator design could not beat this version, something fundamental was probably wrong.

## The FPGA Story Was Really Four Different Designs

What I liked about the FPGA side of the project was that it documented the optimization path clearly instead of pretending the final result appeared all at once.

The first step was basic **buffering**. I split the kernel into load, compute, and store stages, moved data into on-chip buffers, and processed particles in tiles of **128**. That was necessary, but it was nowhere near enough. The estimated latency at that stage was about **1.503 s**, which was only **0.16x** of the CPU baseline. The design was structurally cleaner, but the compute engine still was not doing enough useful work per cycle.

The second step was **compute optimization**. I simplified the likelihood equation algebraically so I could replace much of the double-precision-heavy work with integer accumulation plus a final linear transform using precomputed `scale` and `bias`. Then I split the compute stage into initialization, accumulation, and finalize phases. I completely partitioned the per-particle sum array, unrolled across particles, and pipelined across mask offsets. That cut the estimated latency to about **0.828 s**, but it still was not a compelling accelerator. What it did teach me was that the compute stage itself was not the full story. I had reduced compute latency sharply, but memory movement was still dominating too much of the design.

The third step was where the project really turned: **memory optimization**.

I widened the AXI interfaces to **512 bits**, which let me move **16 integers** or **8 doubles** per transfer. That forced me to reorganize the data layout and add explicit packing and unpacking logic with `ap_uint<512>`. It also exposed a small but important hardware-design detail that I would have missed earlier in the term: **69** does not divide evenly into 16-lane words. If I packed the data naively, the end of one particle's mask would overlap with the start of the next particle and create awkward multiplexing logic. The fix was to pad each particle's 69 values out to **80**, which made every particle exactly five 512-bit words wide.

That padding step felt trivial when I first read it in the code. It was not trivial. It was one of the cleanest examples in the project of a software-minded layout being reshaped into a hardware-friendly one.

Once the packed layout, partitioning, and widened interfaces were in place, the estimated FPGA latency dropped to about **0.061 s**, which corresponded to **3.89x** speedup over the CPU baseline. At that point, the design finally felt like it was exploiting the FPGA rather than merely existing on it.

The last FPGA step was **ping-pong buffering** using `#pragma HLS DATAFLOW`. Because the kernel stages passed whole arrays rather than streams, I used task-level overlap through inferred PIPO buffers instead of FIFO-style streaming. That allowed one tile to load while another computed and a third stored. The estimated latency dropped again to about **0.051 s**, or **4.65x** over the CPU baseline.

What I found especially useful there was that the dataflow version did not just improve latency. It also changed the way I thought about HLS. Instead of writing one fast loop nest, I was now trying to build a macro-pipeline with overlapping stages and resource reuse.

## The Actual Hardware Run Was A Good Reality Check

I appreciated that the project did not stop at C-synthesis numbers.

The full hardware flow included hardware emulation and an actual run on the **Alveo U50**. For emulation, the test cases were reduced in size because RTL-backed emulation is slow, and the host side also chunked particle batches to keep the workflow practical. That part of the project was less glamorous than kernel optimization, but it mattered. It forced me to think about host-device interaction, correctness across full and partial tiles, and whether the kernel design survived real execution rather than just optimistic synthesis assumptions.

On real hardware, the kernel execution time for the full **3 million-particle** case came out to about **138 ms**, compared with an estimated kernel latency of **71 ms** from post-place-and-route reporting. I thought that gap was instructive. It reminded me that FPGA timing reports are useful design tools, but they are not the same thing as observed runtime. Memory behavior, system overhead, and real platform effects still matter.

Even so, the hardware result was still meaningful: the FPGA reached about **1.72x** speedup over the CPU baseline for kernel-focused timing.

I also think it is important to state the caveat clearly. This was **not** an end-to-end apples-to-apples pipeline comparison. The FPGA kernel operated on a prepacked pixel stream, not on raw irregular image gathers. I do not see that as a flaw in the project, but I do think it matters for interpreting the result honestly.

## The GPU Won On Performance, But Not By Accident

The GPU implementation performed best overall, and the report makes it clear why.

The work was split into two kernels. The first kernel gathered each particle's masked pixels from the image into an intermediate pixel buffer. The second kernel performed the likelihood computation from that regularized buffer. That separation was smart because it kept the messy irregular gather stage isolated while allowing the actual math kernel to read contiguous data. The offsets were stored in constant memory, the design used pinned host memory and asynchronous transfers, and the extraction path included a fast in-bounds case so the common path could skip unnecessary boundary checks.

The final GPU execution time was about **7.08 ms**, which translated to roughly **33.47x** speedup over the CPU baseline.

I do not think the interesting lesson there is simply that "GPUs are fast." The more interesting lesson is that the GPU could tolerate the workload better because it had exactly the advantages this kernel wanted: massive thread-level parallelism, high memory bandwidth, and enough scheduling flexibility to hide latency that would cripple a more rigid design.

In other words, the GPU won because the problem matched the machine.

## What I Actually Learned From This Project

If I had to summarize the project honestly, I would not say that I learned how to optimize a likelihood kernel. I would say that I learned how important it is to reshape a problem before trying to accelerate it.

The biggest lesson for me was that hardware acceleration is not just about mapping code to a faster platform. Sometimes the real work is deciding which parts of the original algorithm should stay where they are, which parts should move, and how the data should be reorganized so the accelerator is solving a version of the problem it is actually good at.

That is why the failed first plan mattered so much. It taught me something better than a smooth success would have. It showed me that performance engineering is not a sequence of isolated tricks. It is a willingness to challenge the original decomposition of the problem.

By the end of this project, the platform comparison was still useful:

- **CPU:** 237 ms
- **FPGA:** 138 ms
- **GPU:** 7.08 ms

But those numbers were not the part I valued most. What stayed with me was the design pivot behind them.

The FPGA did not become competitive because I added one clever pragma. It became competitive because I stopped forcing irregular memory access into the wrong place, rebuilt the kernel around packed data, and then optimized the compute and transfer structure around that new representation.

That is the part of the project I still think about now. The best optimization I made was not unrolling a loop or widening a bus. It was admitting that the original plan was not good enough and changing the pipeline before I wasted more time polishing the wrong design.
