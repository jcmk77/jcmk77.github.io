---
title: "Hardware Acceleration: GEMM Performance Engineering"
date: 2026-04-02T10:15:00-06:00
Description: "A retrospective on five ENSC 453 labs, from OpenMP GEMM on CPU to HLS on FPGA and CUDA on GPU, and how each one changed how I think about performance."
Tags: ["ENSC453", "GEMM", "OpenMP", "AVX2", "FPGA", "HLS", "CUDA", "Performance Engineering"]
Categories: ["Hardware Acceleration"]
DisableComments: false
draft: false
---

When I started these HPC labs, I thought performance work would mostly mean "adding more parallelism." By the end, I had a very different view. What the course really taught me was that performance comes from controlling **data movement, synchronization, and reuse** far more carefully than I first expected.

The sequence was also well designed for that lesson. It started with OpenMP on CPU, moved into cache tiling and vectorization, then forced me to think like a hardware designer on FPGA, and finally ended with CUDA optimization on GPU. Even though the implementation platforms kept changing, the core question stayed the same:

**Where is my data, and how often am I forcing the machine to move it?**

Looking back, that question explains almost every good result I got.

## Lab 1 Made Me Respect Synchronization Costs

The first lab looked simple on the surface: parallelize GEMM with OpenMP and study the scaling. What made it useful was that it showed me very early that two implementations can do the same arithmetic and still behave very differently because of synchronization structure.

For the first version, I parallelized the outer `i` loop directly with `#pragma omp parallel for`. That gave each thread a coarse chunk of work and kept the synchronization overhead low. On one thread, the kernel took about **17.26 s**. On 20 threads, it dropped to about **2.52 s**, which gave me a speedup of roughly **6.86x**.

The second version was more complicated. I used a surrounding `#pragma omp parallel` region and then placed `#pragma omp for` directives inside the outer loop. On paper, it still looked parallel. In practice, it inserted repeated implicit barriers into a kernel with `NI = 2048`, so the threads kept stopping to synchronize thousands of times. That version only reached about **5.77x** speedup on 20 threads.

That gap changed how I thought about OpenMP. Before this lab, I would have described the two versions as minor variations. After measuring them, I stopped treating synchronization as a small implementation detail. It was part of the algorithm.

My main takeaway from Lab 1 was that coarse-grained work distribution usually wins if it avoids repeated coordination. The threads were not struggling because GEMM was hard to parallelize. They were struggling because I had given them too many chances to wait for each other.

## Lab 2 Taught Me That Locality Comes Before Heroics

Lab 2 was the point where performance optimization started to feel less like API usage and more like systems work.

The starting point was a serial GEMM that took about **506.74 s**. The code was doing the right arithmetic, but the memory behavior was bad. The biggest issue was the access pattern for matrix `B`: the loop order was forcing a stride of **4096 floats**, which is **16 KiB** between adjacent accesses. That is about the fastest possible way to remind myself that caches are not magic.

I approached the optimization in stages.

First, I used 1-D tiling to improve reuse along the `j` dimension. That helped, but it was still only part of the story. Then I moved to 2-D tiling and loop permutation so that the kernel could reuse `A`, `B`, and `C` more intelligently. The most useful version for me was the third tiled implementation, where I stopped insisting on one uniform tile size and instead tuned each loop dimension separately. The best setting I found was:

- `Ti = 32`
- `Tj = 1024`
- `Tk = 32`

That detail mattered to me because it was one of the first times I felt I was optimizing for the machine I actually had rather than for symmetry on paper. The best answer was not the prettiest answer.

Once the memory behavior was under control, vectorization started to pay off. Adding `#pragma omp simd` to the beta-scaling and inner update loops brought the runtime down to about **7.45 s**, which was already a **68x** speedup over the original baseline. Then I combined the tiled structure with OpenMP across the outer loop and got the runtime down to about **0.84 s**, or roughly **603x** speedup.

The final step was replacing the compiler-directed SIMD path with explicit **AVX2 intrinsics**. I used 256-bit vectors, broadcasted a single `A` value across a register, loaded chunks of `B`, accumulated into vectorized `C`, and unrolled across `k`. That pushed the runtime down again to about **0.55 s**, which corresponded to roughly **917x** speedup over where I started.

What I liked about Lab 2 was that it gave me a disciplined order of operations:

1. fix locality
2. exploit SIMD
3. add threading
4. then consider hand-tuned intrinsics

That order has stayed with me. If I had started with AVX2 before fixing the access pattern, I would have been vectorizing a bad idea faster.

## Lab 3 Forced Me To Think Like A Hardware Designer

By Lab 3, the problem stopped being "how do I help the CPU cache?" and became "how do I build the memory hierarchy myself?"

On FPGA, a `4096 x 4096` matrix is far too large to treat casually. Each matrix is about **64 MB**, so I could not just assume the data would sit near the compute units. I had to explicitly tile the problem, move tiles into local buffers, compute on them, and write them back out. That led me to a load-compute-store design with on-chip `buffer_A`, `buffer_B`, and `buffer_C` arrays.

That first transformation already changed the scale of the result. The baseline HLS estimate was on the order of **1960 s** absolute latency. After reorganizing the kernel around tiled buffering, the estimate dropped to about **167.9 s**.

The real jump came when I optimized the compute engine instead of only the memory staging. I built a parallel micro-kernel around **16-way row parallelism** and **32-way column unrolling**, partitioned the arrays so the hardware could actually feed that parallelism, and bound the floating-point operations onto DSP resources with explicit latencies. At that point, the kernel stopped looking like a C loop nest and started looking like a hardware schedule.

That version brought the estimated absolute latency down to about **0.91 s** while still staying within the resource budget:

- **BRAM:** 48%
- **DSP:** 43%
- **FF:** 24%
- **LUT:** 43%

This lab was where I stopped thinking of "optimization" as purely shrinking runtime. On FPGA, optimization is a balance between latency, throughput, and resource pressure. A fast design that cannot fit is not a good design. I had to care about whether I was spending BRAM or DSPs responsibly, not just whether a loop could be unrolled further.

## Lab 4 Taught Me That A Fast Kernel Is Not Enough

Lab 4 extended the FPGA work in the direction that mattered most: making the design behave better as a system, not just as a compute core.

The first major improvement was widening the memory interface to **512 bits** over AXI4. I originally tried using helper infrastructure for wide buses, but the limitations were not worth fighting. The better route was to pack and unpack the data manually with `ap_uint<512>` and move **16 floats per memory chunk**. That reduced the number of memory transactions and improved effective bandwidth much more directly.

That alone brought the latency estimate down substantially relative to the Lab 3 design. Then I added **ping-pong buffering** so one tile could be loading while another was computing. What made that step satisfying was that I could actually verify the overlap instead of just claiming it. The measured per-tile latency lined up closely with the compute stage rather than the serial sum of load-plus-compute, which showed that the overlap was real.

With the combined optimizations, the design reached an estimated latency of about **0.507 s** to **0.694 s** depending on which estimate path I looked at, for roughly **1.8x** speedup over the Lab 3 reference design. Resource use stayed controlled, though URAM and BRAM usage became much more meaningful as the buffering strategy got more ambitious.

What mattered even more to me was that this lab forced me to go beyond HLS estimates and work through the deployment pipeline:

- software emulation
- hardware emulation
- real hardware execution on the **Alveo U50**
- OpenCL host orchestration
- correctness checking against a CPU reference

On real hardware, the full run reported about **1152.23 ms** execution time. That number was not identical to the synthesis estimates, and I thought that was useful rather than disappointing. It reminded me that the hardware development cycle is not just about the kernel body. Host setup, memory movement, tool behavior, and post-route timing all matter once the design leaves the notebook.

Lab 4 made the project feel more honest. I was no longer just optimizing a function. I was building an accelerator pipeline that had to survive contact with an actual board.

## Lab 5 Closed The Loop On Data Reuse

The last lab changed applications, but not really the lesson.

Instead of GEMM, I optimized a CUDA image blur on a **3840 x 2160** image with blur radius **21**. The baseline implementation, including transfers, took about **0.364 s**. The goal was to improve kernel performance first and then improve host-device transfer behavior.

The biggest step was redesigning the computation around shared memory. I used a **16 x 16** thread block to cooperatively load data for a **32 x 32** output tile with halo coverage, which gave a shared tile of **74 x 74**. But the part I found most interesting was not just the shared-memory cache itself. The bigger gain came from reusing **partial column sums** so the blur did not recompute the same work repeatedly.

That distinction mattered to me. It would have been easy to say "shared memory made it faster" and stop there. The more accurate explanation was that shared memory enabled a better reuse strategy.

With the optimized kernel, the runtime dropped to about **0.139 s**, which was roughly **2.5x** faster than the baseline. I also analyzed control divergence instead of assuming it away. Most of the divergence was limited to the uneven work assignment during tile loading and to edge conditions near the image boundaries. The percentages were small enough that they did not dominate the final result, which helped confirm that the design direction was sound.

After that, I optimized the transfer path using **pinned host memory** with `cudaHostRegister()`. I kept the kernel launch repeated across multiple runs so I could isolate the effect better, then compared the transfer-aware version against the already optimized kernel. That brought the runtime down again to about **0.113 s**, which was another **1.23x** speedup over the kernel-only optimized version and about **3.22x** faster than the original baseline.

I liked this lab because it ended on a very complete point: performance work is not finished when the kernel gets fast. If the transfer path is still careless, the application is still leaving time on the table.

## What Stayed True Across All Five Labs

The hardware kept changing across these labs, but the engineering logic became more consistent for me, not less.

Across CPU, FPGA, and GPU, the same ideas kept reappearing:

- reduce unnecessary synchronization
- improve locality before chasing raw parallelism
- tile around the memory hierarchy you actually have
- create reuse explicitly instead of hoping the hardware will infer it
- overlap communication and computation when possible
- measure the full system, not just the inner loop

That is what I now think of as the real value of this sequence. It was not just a set of disconnected optimizations. It trained me to look at performance problems structurally.

When I look back at the numbers, the large speedups are satisfying. But the more important part is that I now have a much better instinct for why those speedups happened. In Lab 1, it was synchronization structure. In Lab 2, it was locality and vectorization. In Labs 3 and 4, it was explicit buffering and bandwidth-aware hardware design. In Lab 5, it was staged reuse plus better transfer behavior.

So if I had to summarize what these taught me in one sentence, it would be this:

**High performance is rarely about making arithmetic cheaper. It is about making data less expensive to move, wait on, and recompute.**
