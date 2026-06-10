---
title: "AI Language Model from Scratch"
date: 2026-05-01T20:05:00-06:00
Description: "A retrospective on an ENSC 413 edge-LLM project that trained a compact model from scratch, fine-tuned Gemma and Qwen with LoRA, and benchmarked everything on a Raspberry Pi 5."
Tags: ["ENSC413", "LLM", "Raspberry Pi", "LoRA", "Quantization", "llama.cpp", "Edge AI", "Transformers"]
Categories: ["Edge AI"]
DisableComments: false
draft: false
---

The most interesting part was simply "getting an LLM to run on a Raspberry Pi." What I actually learned was how training, deployment, quantization, and evaluation all change once I stop treating language models as abstract software and start treating them as systems with real hardware limits.

The main question I kept coming back to was this:

**What are LLMs I need and how do I handle them?**

That question shaped the whole project. We trained a compact model from scratch, fine-tuned open-source models with LoRA, converted everything into GGUF for `llama.cpp`, benchmarked on a **Raspberry Pi 5 with 16 GB RAM**, and compared speed, memory use, and output quality across both structured and open-ended tasks. Looking back, the most important part was not the benchmarking itself. It was learning how to define tasks that match what different model sizes can realistically do.

## Splitting The Problem In Two Was The Best Decision

One of the best decisions I made in this project was to stop treating all language tasks as the same.

Instead of asking whether a model was simply "good" or "bad," we split evaluation into two benchmarks:

- **Benchmark A**: natural-language robot commands mapped into a strict JSON schema
- **Benchmark B**: short economics question answering, evaluated semantically rather than by exact formatting

That split ended up explaining almost every result.

For Benchmark A, I mainly cared about things like valid JSON rate, schema compliance, and whether the output was actually usable for a robot control pipeline. That is a bounded output space. The model does not need broad world knowledge there. It needs reliability, consistency, and fast response under a narrow contract.

For Benchmark B, the situation is completely different. The output is freer, multiple phrasings can be correct, and the model needs actual semantic coverage rather than just structural discipline.

This feels obvious to me now, but we did not start there. Earlier in the project, we tried a mixed Linux CLI task where the model sometimes had to return an exact command and sometimes explain it. That turned out to be a bad fit for a small model. Once I stopped pushing the model toward a vague general assistant and instead designed tasks that matched what a compact edge model could realistically do, the whole project became much clearer.

## Training Edges From Scratch Taught Me The Most

The self-trained model, which we called **Edges**, was the part of the project that taught me the most because it forced me to think through the entire engineering pipeline instead of just downloading a checkpoint.

From the report and the `Edges` code, the final design was compact but not trivial. The model uses:

- a **Mistral tokenizer** with vocabulary size **32,000**
- **16 transformer layers**
- hidden size **768**
- **8 query heads** and **2 key-value heads**
- grouped-query attention to reduce KV-cache size
- RMSNorm and RoPE-based positional encoding

The exported GGUF metadata later shows about **148 million parameters**, which was exactly the kind of scale we were aiming for: much smaller than the open-source baselines, but still large enough to learn useful patterns.

The data pipeline mattered just as much as the architecture. For pretraining, we used **FineWeb**, **FineWeb_Edu**, and **Wikipedia**, packed into **2048-token** sequences for next-token prediction. After that, we ran supervised fine-tuning on two separate instruction datasets:

- **30k robot-command examples** plus **1k validation**
- **30k economics QA examples** plus **1k validation**

The training artifacts in `Edges/runs` still make the project feel very real to me. The run info records a **4090 D** as the training GPU, uses **bfloat16**, and shows full supervised fine-tuning from the pretrained checkpoint. The saved validation samples also show exactly what I cared about during development: whether the model could emit a valid `robot_cmd_v1` object without inventing fields or drifting into explanation.

That was the whole point of Edges. I was not trying to build a miniature ChatGPT. I was trying to build a narrow edge model for narrow edge jobs.

## Edges Worked Best When I Constrained The Output

The results made that specialization very obvious.

On the structured robot-command benchmark, **Edges-SFT** performed surprisingly well for its size:

- **100% valid JSON**
- **70% task score**
- about **0.116 s** first-token latency
- about **43 tokens/sec**
- about **363 MB** peak RAM on the Raspberry Pi

That is a very good edge profile. It is fast, light, stable, and good enough to be useful when the output contract is strict.

But the same model dropped hard on the economics QA benchmark:

- **43% semantic score**
- about **0.14 s** first-token latency
- about **45.6 tokens/sec**
- about **335 MB** peak RAM

That gap captures the whole project for me. The custom model was not failing because it was badly engineered. It was failing because open-ended semantic knowledge is a much broader target than structured command emission. I came away from this project with a much stronger belief that **small models become far more viable when the task is sharply bounded**.

## The Gemma And Qwen Pipelines Were About Practical Adaptation

The `gemma-ft` and `qwen_ft` branches are not about inventing a new architecture. They are about building a practical adaptation-and-deployment workflow under limited resources.

Both follow the same general pattern:

1. preprocess chat-style JSONL into prompt/completion form
2. mask prompt tokens with `-100`
3. train only on the final assistant output
4. use **LoRA** instead of full fine-tuning
5. merge adapters back into the base model
6. export to **GGUF**
7. benchmark under `llama.cpp`

The training configs are also consistent:

- **2 epochs**
- max sequence length **512**
- per-device batch size **1**
- gradient accumulation **16**
- learning rate **1e-4**
- LoRA rank **16**
- LoRA alpha **32**
- dropout **0.05**
- adapters attached to `q_proj`, `k_proj`, `v_proj`, and `o_proj`

What I liked here was that the code stayed explicit. The training scripts make completion-only supervision, truncation policy, and adapter placement very clear. This was not just "we fine-tuned Qwen." It was a low-resource fine-tuning pipeline that I could actually reason about.

The saved eval metrics also support what we wrote in the report. On the robot JSON task, Qwen's validation numbers are strong:

- **1.0 valid JSON rate**
- **1.0 schema pass rate**
- **0.912 exact match rate**

Gemma is still decent, but clearly weaker:

- **0.948 valid JSON rate**
- **0.948 schema pass rate**
- **0.745 exact match rate**

So even before Raspberry Pi deployment, I could already see that the higher-capacity model would probably become the stronger semantic system later.

## Quantization Was More Subtle Than I Expected

At first, quantization felt like it should be simple: compress the model, reduce memory, and move on. In practice, it was more subtle than that.

We converted everything to GGUF and used `llama.cpp` quantization targets like **Q4_K_M** and **Q2_K_S**. The disk-size reductions were substantial:

- **Gemma3-1B**: **1.9 GB** to **769 MB** at Q4, **641 MB** at Q2
- **Qwen3-4B**: **7.5 GB** to **2.4 GB** at Q4, **1.6 GB** at Q2
- **Qwen3-14B**: **29.6 GB** to **9.0 GB** at Q4

But one of the most useful things I learned was that **smaller does not automatically mean better**. In the report, the 2-bit models were not necessarily faster to start generating than the 4-bit ones. The likely cause is CPU-side decompression overhead, which is exactly the kind of deployment detail I would have missed if I had only looked at file size.

I also saw that aggressive quantization can damage structured reliability very badly. On Benchmark A, **Gemma3-1B Q2_K_S** collapsed to **4% valid JSON** and **0% task score**. **Qwen3-4B Q2_K_S** held up better semantically than Gemma's 2-bit variant, but it still degraded too much for dependable structured control.

So the practical lesson for me was not "quantize as much as possible." It was:

**Quantize enough to make deployment feasible, but not so far that the task itself breaks.**

## The Benchmark Harness Was One Of The Strongest Parts Of The Project

The `llama/benchmark_llamacpp.py` script is easy to overlook, but I think it is one of the strongest engineering artifacts in the entire project.

It starts a local `llama-server`, sends prompts from JSONL files, records streamed answers, logs per-case outputs, and samples process-level resource usage with `psutil`. The output directories include:

- `answers.jsonl`
- per-case CSV and JSONL results
- category summaries
- overall summaries
- run configs
- server logs

That mattered because I did not want to rely on vague impressions like "this felt fast." I wanted repeatable measurements for:

- first-token latency
- decode speed
- peak and steady RAM
- CPU utilization
- stability across runs

The saved run summaries also confirm the real deployment target: a **4-core Raspberry Pi 5** running `llama.cpp`, usually with **4 inference threads**, `ctx-size` **1024**, and one warmup request before benchmarking. That gave the report's latency and RAM tables much more credibility.

## The Winner Depends On What I Need The Model To Do

If the task is robot command parsing, then a small specialized model is not only viable. In my opinion, it may actually be the right choice.

Edges-SFT is dramatically lighter than the larger open models and responds almost instantly. For deterministic edge control, that mattered more to me than broad semantic elegance.

If the task is semantic QA, the picture changes completely. The strongest result came from the larger Qwen family:

- **Qwen3-4B** reaches **100% semantic score** on Benchmark B, but uses about **8.0 GB** peak RAM and has about **4.16 s** first-token latency
- **Qwen3-4B Q4_K_M** keeps **96% semantic score** while reducing peak RAM to about **5.1 GB** and first-token latency to about **2.97 s**
- **Qwen3-14B Q4_K_M** also reaches **100% semantic score**, but at about **14.9 GB** peak RAM and roughly **13 s** first-token latency, which already feels close to impractical on the Raspberry Pi

For me, that makes **Qwen3-4B Q4_K_M** the most convincing general-purpose edge compromise in the whole report. It is still heavy, but it preserves enough capability to justify its cost.

At the same time, Edges stays compelling because it is the right size and shape for a different job.

## The Final Recommendation Felt More Like System Design Than Model Comparison

By the end of the report, I no longer thought of this project as a simple model bake-off. It started to feel much more like system design.

Our final recommendation was a **tiered architecture**:

- use **Edges-SFT** for instant, deterministic, schema-bound actions
- invoke **Qwen3-4B Q4_K_M** only when more open semantic reasoning is required

That is a much stronger conclusion than just saying that Qwen scored higher. It treats edge AI as orchestration, not just model selection.

It also matches the lessons from our own failures. The earliest version of the custom model was too shallow, with a weak tokenizer and low semantic accuracy. The earlier mixed-task Linux CLI idea was also too ambiguous for a small model. The project improved once I stopped expecting one small model to do everything and instead aligned model capacity with task structure.

That is probably the part I find most convincing now.

## What I Took Away From The Project

This project made me think less about "can a Raspberry Pi run an LLM?" and more about **how much task structure I can trade for speed, memory efficiency, and deployment simplicity**.

The answer I came away with is pretty clear:

- if the task is constrained, repetitive, and schema-based, a compact custom model can be surprisingly effective
- if the task needs broader semantic knowledge, larger open-source models still dominate, even after quantization
- 4-bit quantization can be a very practical middle ground
- 2-bit compression is not a free lunch and can actively damage structured reliability

So the main lesson I took from `413FinalProject` is not that small models replace large ones. It is that edge AI works best when I stop thinking in terms of one universal model and instead design a system where each model has a well-defined job.

That was a much more useful lesson for me than just proving that a language model can run locally.

**{Related Courses}**: ENSC 413 Deep Learning, Natural Language Processing, Embedded Systems, Edge AI
