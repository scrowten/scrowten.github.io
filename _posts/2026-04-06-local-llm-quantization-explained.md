---
layout: post
title: "🧮 Quantization Explained: How a 70B Model Fits on Your Laptop"
date: 2026-04-06 00:00:00
description: A beginner-friendly but technically deep guide to LLM quantization — covering precision formats, GGUF/GPTQ/AWQ, VRAM math, KV cache, and RAM offloading. Part 3 of the Self-Hosting LLMs series.
tags: llm quantization local-ai ollama gguf
categories: machine-learning llm self-hosting
toc:
  sidebar: left
---

*This is Part 3 of the **Self-Hosting LLMs** series. You can read this post standalone — no prior posts required.*

---

You've seen it on Reddit or Hugging Face: **"Llama 3.1 70B"**.

Seventy billion. That sounds enormous. Surely you need a server room or a $30,000 GPU cluster to run something like that, right?

Probably not.

With the right settings, a **70B model can run on a single consumer GPU** — or even mostly on your CPU if you're patient. And a **7B model runs beautifully on a laptop GPU** with 8GB of VRAM.

The technique that makes this possible is called **quantization**. It's the single most important concept to understand when you're running LLMs locally, and it's not as scary as it sounds.

By the end of this post you will:

* ✔️ Understand what a "parameter" actually is and why it takes memory
* ✔️ Know the difference between FP32, FP16, INT8, INT4 — and why it matters
* ✔️ Be able to calculate how much VRAM a model will need, yourself
* ✔️ Know when to use GGUF vs GPTQ vs AWQ
* ✔️ Understand what a KV cache is and why it silently eats your VRAM
* ✔️ Download and run a quantized model with a single command

Let's start from the beginning.

---

# 🧠 1. What Is a Parameter?

Before we talk about reducing model size, we need to understand what we're actually reducing.

A large language model is a neural network. At its core, it's a massive mathematical function — one that takes text as input and predicts the next word. This function has billions of "tunable knobs" called **parameters** (also called **weights**).

Think of it like a giant mixing board with 70 billion sliders. During training, the model learns to set each slider to the exact value that makes it good at language. Once training is done, those values are frozen — and when you run the model, the computer reads those slider values to produce its output.

Every single parameter is a **number** that has to be stored in memory. And the precision of that number — how many decimal places it has — determines how much space it takes.

That's where precision formats come in.

---

# 🎛️ 2. Precision Formats: FP32, FP16, BF16, INT8, INT4

Every number stored in a computer takes a fixed number of **bits** (the 0s and 1s that computers use). More bits = more precision = more memory used.

Here's how the common formats break down:

| Format | Bits per number | Bytes per number | Example: 7B model size |
|--------|----------------|-----------------|----------------------|
| FP32 (full precision) | 32 bits | 4 bytes | ~28 GB |
| FP16 (half precision) | 16 bits | 2 bytes | ~14 GB |
| BF16 (brain float 16) | 16 bits | 2 bytes | ~14 GB |
| INT8 (8-bit integer) | 8 bits | 1 byte | ~7 GB |
| INT4 (4-bit integer) | 4 bits | 0.5 bytes | ~3.5 GB |

### FP32 — The "Original" Format

32-bit floating point. This is the standard precision used in math and training. Each number can represent values with high accuracy across a huge range.

For a 70B parameter model: **70,000,000,000 × 4 bytes = 280 GB**.

That's more than three A100 GPUs (80 GB each) running simultaneously. Not practical for most of us.

### FP16 / BF16 — Half Precision

16-bit formats cut memory in half by being slightly less precise.

- **FP16**: 5 exponent bits, 10 mantissa bits. More precise in the middle range.
- **BF16**: 8 exponent bits, 7 mantissa bits. Same range as FP32, better for very large or very small values. Preferred for training, great for inference.

In practice, FP16 and BF16 produce nearly identical output quality for inference. A 70B model in FP16: **~140 GB**. Still requires two H100s.

### INT8 — 8-bit Integer

Instead of a floating point number, each weight becomes an integer between -128 and 127. The model is calibrated so these integers map closely to the original values.

Quality impact: Research shows **only a 0.04% drop** on standard benchmarks compared to FP16. Essentially free lunch.

A 7B model in INT8: **~7 GB** — fits on a single RTX 3080 or Mac with 8GB unified memory.

### INT4 — 4-bit Integer

Four bits means only **16 possible values** (-8 to 7). This is where you'd expect quality to fall off a cliff — but it doesn't, at least not dramatically.

Studies on Llama 3.1 8B showed that good INT4 quantization retains **98%+ accuracy** on MMLU-Pro (a standard reasoning benchmark). The quality loss is real but surprisingly small for most everyday tasks.

A 7B model in INT4: **~3.5 GB** — runs on anything with a halfway modern GPU, or even on CPU with patience.

---

# 🔢 3. The Math: Calculate VRAM Yourself

Here's the formula you can use for any model:

```
Model VRAM (GB) ≈ (Parameters in billions × Bits per weight) / 8 / 1024
```

Let's work through some examples:

**Llama 3.1 8B in FP16:**
```
8B params × 16 bits = 128 billion bits
128 billion bits ÷ 8 = 16 GB of bytes
≈ 16 GB VRAM
```

**Llama 3.1 8B in Q4 (INT4):**
```
8B params × 4 bits = 32 billion bits
32 billion bits ÷ 8 = 4 GB
≈ 4 GB VRAM
```

**Llama 3.1 70B in Q4 (INT4):**
```
70B params × 4 bits = 280 billion bits
280 billion bits ÷ 8 = 35 GB
≈ 35 GB VRAM
```

> ⚠️ **Important**: These numbers are for the **model weights only**. You'll need additional VRAM for the KV cache (more on this in Section 5) and runtime overhead (~20% buffer is a safe estimate). Always add ~20% on top of the weight-only estimate.

---

# 📦 4. Quantization Formats: GGUF, GPTQ, and AWQ

Now that you understand *why* we quantize, let's talk about *how*. There are several methods and file formats you'll encounter in the wild.

### GGUF — The One for Most People

**GGUF** (GGML Universal Format) is a file format that packages the quantized model and its metadata in a single `.gguf` file. It's the format used by **Ollama**, **llama.cpp**, and **LM Studio**.

What makes GGUF special:
- **CPU-friendly**: Unlike most GPU formats, GGUF works well on pure CPU
- **Mixed CPU+GPU**: You can load some layers to GPU and the rest to RAM — useful when the model almost fits on your GPU
- **Portable**: One file, no setup, no dependencies
- **Many quantization levels**: Q2, Q3, Q4, Q5, Q6, Q8 — pick your precision

GGUF is where most beginners should start.

### GPTQ — For Pure GPU Throughput

**GPTQ** uses calibration data (a sample of text) to find optimal 4-bit integer values for each weight. The calibration process minimizes the error introduced by quantization.

Key characteristics:
- **Requires GPU**: Not designed for CPU inference
- **Faster on GPU**: With optimized kernels (like Marlin), GPTQ can be ~5x faster than GGUF on the same GPU
- **Slightly better quality** than naive INT4 quantization at same bit count
- Used heavily with **vLLM** and **text-generation-inference**

Choose GPTQ if you have a dedicated GPU and care about maximum throughput (tokens per second).

### AWQ — The Smarter 4-bit

**AWQ** (Activation-Aware Weight Quantization) takes a clever approach: not all weights are equally important. AWQ identifies which weights have the most influence on outputs (by analyzing activation patterns) and protects those with higher precision.

Key characteristics:
- **No backpropagation** needed during quantization (faster to produce than GPTQ)
- **Retains ~95% quality** vs FP16
- Good balance of speed and quality
- Works well with **vLLM** and other GPU inference servers

### Quick Decision Guide

```
Just getting started / using Ollama / using LM Studio?
  → GGUF (Q4_K_M is the default recommendation)

Have a dedicated NVIDIA GPU, want maximum speed?
  → GPTQ or AWQ (use with vLLM)

Running on CPU or splitting across CPU + GPU?
  → GGUF only
```

---

# 🔤 5. Decoding the Naming Convention

When you browse Hugging Face or Ollama's model library, you'll see names like:

```
llama3.1:8b-instruct-q4_K_M
mistral:7b-instruct-v0.2-q5_K_S
deepseek-coder:6.7b-instruct-q8_0
```

Let's decode `q4_K_M`:

| Part | Meaning |
|------|---------|
| `q4` | 4-bit quantization |
| `K` | K-quants method (importance-weighted, smarter quantization) |
| `M` | Medium variant — balanced between size and quality |

The variants go: `S` (small) < `M` (medium) < `L` (large) — meaning `_S` is smaller/faster and `_L` is higher quality.

There's also the simpler `_0` suffix (like `q4_0` or `q8_0`) which uses the older, uniform quantization method. K-quants (with `_K_`) are generally better — they apply higher precision to the most important weights.

### The Community's Favorite: Q4_K_M

**Q4_K_M is the sweet spot** that the LocalLLaMA community has converged on for good reason:

- ~75% size reduction vs FP16
- Retains ~92% of original quality
- Runs fast on consumer hardware
- The K-quant method gives ~10-30% better perplexity vs naive Q4

When in doubt, start with Q4_K_M. If quality feels lacking, step up to Q5_K_M. If you're tight on VRAM, try Q3_K_M — though quality starts to noticeably dip below Q4.

### At-a-Glance Quality vs Size

| Quantization | Size (7B) | Quality (vs FP16) | Use When |
|---|---|---|---|
| Q8_0 | ~7 GB | ~99% | VRAM allows, maximum quality |
| Q6_K | ~5.5 GB | ~98% | Near-lossless, tighter VRAM |
| Q5_K_M | ~4.8 GB | ~96% | Great quality, slight size saving |
| **Q4_K_M** | **~4.1 GB** | **~92%** | **Default recommendation** |
| Q3_K_M | ~3.3 GB | ~85% | When you really need the space |
| Q2_K | ~2.7 GB | ~75% | Last resort, noticeable quality drop |

---

# 💾 6. The KV Cache: The Hidden VRAM Thief

Here's something most beginners don't know until they run out of memory: the model weights are only **part** of the VRAM you need. There's a silent consumer called the **KV cache**.

### What Is a KV Cache?

When an LLM generates text, it does so one token at a time. For each token it generates, the model performs an "attention" calculation that looks back at all previous tokens to understand context.

To avoid recomputing those attention values every single step, the model **caches them** in memory — this is the Key-Value (KV) cache.

The KV cache grows with:
- **Context length** — longer conversations = bigger cache
- **Batch size** — more simultaneous users = bigger cache

### How Much Does It Cost?

For a **Llama 3.1 8B** model with a 4,096 token context window:

```
KV cache ≈ 2 × context_length × num_layers × num_kv_heads × head_dim × 2 bytes (FP16)
         ≈ 2 × 4096 × 32 × 8 × 128 × 2 bytes
         ≈ ~0.5 GB
```

That's manageable for short conversations. But at **128K context** (which Llama 3.1 supports):

```
KV cache ≈ ~17 GB just for one long conversation
```

Suddenly the "4 GB model" needs 21 GB total. This surprises a lot of people.

### KV Cache Quantization

You can also quantize the KV cache itself:

| KV Cache Format | VRAM Reduction | Speed Impact |
|---|---|---|
| FP16 (standard) | Baseline | Fastest |
| FP8 | ~50% reduction | Minimal impact |
| INT8 (q8_0) | ~50% reduction | <5% slower — **best tradeoff** |
| INT4 | ~75% reduction | 90%+ slower at long contexts — avoid |

The community recommendation: **FP8 or q8_0 KV cache** if your tool supports it, otherwise leave it at FP16. INT4 KV cache is not worth the speed penalty.

### Practical Takeaway

When calculating "will this fit?", always account for context:

```
Total VRAM needed ≈ Model weights + KV cache + ~20% overhead

Example: Llama 3.1 8B Q4_K_M for a normal conversation (2K context)
  = 4.1 GB (weights) + 0.25 GB (KV cache) + ~1 GB (overhead)
  = ~5.4 GB total
  → Fits on a 6 GB GPU with room to spare ✓
```

---

# 🐢 7. RAM Offloading: Last Resort, Not a Strategy

What happens when a model is too large to fit in your GPU's VRAM? Some tools (especially Ollama and llama.cpp) offer **RAM offloading** — storing part of the model in your CPU's regular RAM and loading weights on demand.

It sounds like a lifesaver. In practice, it's a significant performance hit.

### How Much Slower?

Running a model with weights partially in RAM (via system memory) is roughly **20-30x slower** than running entirely in VRAM. A model that produces 40 tokens/second from VRAM might produce 1-2 tokens/second with heavy offloading.

With Ollama, you can see how layers are distributed:

```bash
ollama ps
```

Output will show something like:
```
NAME                    ID         SIZE      PROCESSOR    UNTIL
llama3.1:8b-q4_K_M     a6990ed6   5.2 GB    100% GPU     4 minutes from now
```

vs

```
NAME                    ID         SIZE      PROCESSOR    UNTIL
llama3.1:70b-q4_K_M    ...        43.0 GB   38% GPU      4 minutes from now
                                              62% CPU
```

That 62% CPU means 62% of the model's layers are offloaded to RAM — and your throughput will feel it.

### When Is Offloading OK?

- **Development/experimentation**: Acceptable to test a model you wouldn't deploy
- **Very short generations**: If you just need to generate 50 tokens to test something, it's tolerable
- **Night-time batch jobs**: If speed doesn't matter and you're fine waiting

For anything interactive (a chatbot, a coding assistant), offloading makes the experience frustrating. The solution is to use a smaller, more aggressively quantized model that fits fully in VRAM.

---

# 💻 8. What Can You Actually Run? (Hardware Cheat Sheet)

Here's a practical guide organized by how much VRAM (or unified memory) you have:

| VRAM | Hardware Examples | What Fits (Comfortably) |
|------|-------------------|------------------------|
| 4 GB | GTX 1650, older integrated | 3B models at Q4, 7B at Q2 (barely) |
| 6 GB | RTX 3060, GTX 1660 | 7B at Q4_K_M ✓, 13B at Q2 |
| 8 GB | RTX 3070, M1 MacBook (8GB) | 7B at Q5_K_M ✓, 13B at Q3-Q4 |
| 12 GB | RTX 3080, M2 Pro (12GB) | 13B at Q4-Q5 ✓, 30B at Q2-Q3 |
| 16 GB | RTX 4070 Ti, M2 Max (16GB) | 13B at Q6-Q8 ✓, 30B at Q4 ✓ |
| 24 GB | RTX 4090, RTX 3090, A5000 | 30B at FP16 ✓, 70B at Q3 ✓ |
| 32 GB | M3 Max (36GB), RTX 5090 | 70B at Q4 ✓ |
| 48 GB | A6000, dual RTX 3090 | 70B at Q5-Q6 ✓, small MoE models |
| 80 GB | A100, H100 | 70B at FP16 ✓, 405B at Q4 |

> **Apple Silicon note**: Mac unified memory is shared between CPU and GPU, so it's more efficient than discrete GPU+RAM setups. A MacBook Pro with 16GB unified memory can comfortably run 13B models and is one of the best consumer options for local LLMs today.

### General Rule of Thumb

**Find your VRAM, divide by model size in billions, look up the highest quality quant that fits:**

```
VRAM (GB) ÷ 0.5 ≈ max model size in billions at Q4
```

So 8 GB VRAM → up to ~16B params at Q4. 24 GB → up to ~48B params at Q4.

---

# 🚀 9. Hands-On: Download and Run a Quantized Model

Let's make this concrete. We'll use **Ollama** — the simplest way to get started.

### Install Ollama

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# Windows: download the installer from https://ollama.com
```

### Pull a Model With a Specific Quantization

```bash
# 7B model at Q4_K_M — the community sweet spot (~4 GB VRAM)
ollama pull llama3.1:8b-instruct-q4_K_M

# 7B model at Q5_K_M — one step higher quality (~5 GB VRAM)
ollama pull llama3.1:8b-instruct-q5_K_M

# If you have 24GB+ VRAM and want maximum quality
ollama pull llama3.1:8b-instruct-q8_0
```

If you just type the model name without a quantization tag, Ollama picks a sensible default (usually Q4_K_M):

```bash
ollama pull llama3.1:8b
```

### Check What's Loaded

```bash
# See which models are currently running and their VRAM usage
ollama ps

# List all models you've downloaded
ollama list
```

### Chat With It

```bash
ollama run llama3.1:8b-instruct-q4_K_M
```

Or use it via the API (which is OpenAI-compatible):

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama",  # required by the client, but ignored locally
)

response = client.chat.completions.create(
    model="llama3.1:8b-instruct-q4_K_M",
    messages=[
        {"role": "user", "content": "Explain quantization to a 10-year-old."}
    ],
)

print(response.choices[0].message.content)
```

This is the same OpenAI client you'd use with `api.openai.com` — just with a different `base_url`. That's the beauty of the OpenAI-compatible API standard that all local tools implement.

### Experiment: Feel the Quality Difference

Try the same prompt on different quants and see if you notice a difference:

```bash
# Lower quality
ollama pull llama3.1:8b-instruct-q2_K
ollama run llama3.1:8b-instruct-q2_K "Solve: if a train travels at 60 mph for 2.5 hours, how far does it go?"

# Standard quality
ollama run llama3.1:8b-instruct-q4_K_M "Solve: if a train travels at 60 mph for 2.5 hours, how far does it go?"
```

For simple questions, both will answer correctly. For complex reasoning, Q2 starts to struggle noticeably. This is a great way to build intuition for where the quality floor actually is.

---

# 🎯 Quick Reference Card

Print this out or keep it handy:

```
CHOOSING A QUANTIZATION
─────────────────────────────────────────────
Have lots of VRAM, want best quality?  → Q8_0
Standard use, best balance?            → Q4_K_M  ← Start here
Tight on VRAM?                         → Q3_K_M
Really tight?                          → Q2_K (quality noticeably degrades)

CHOOSING A FORMAT
─────────────────────────────────────────────
Ollama / LM Studio / llama.cpp?        → GGUF
vLLM / GPU server?                     → GPTQ or AWQ
CPU or mixed CPU+GPU?                  → GGUF only

VRAM ESTIMATE
─────────────────────────────────────────────
Rough formula: params_B × bits / 8 / 1024 = GB
Add 20% overhead + KV cache (~0.5 GB per 4K context)

7B Q4:  ~4 GB weights → ~5-6 GB total
13B Q4: ~7 GB weights → ~9 GB total
70B Q4: ~35 GB weights → ~40+ GB total
```

---

# 🎉 Wrapping Up

Quantization is what makes local LLMs actually practical. Instead of needing a $30,000 GPU cluster for a 70B model, you can run one on a gaming PC — with only a small, often barely-noticeable quality tradeoff.

The key ideas to remember:

- **Parameters × bit width ÷ 8 = model size in bytes** — the math is simple
- **Q4_K_M is the default recommendation** — best size/quality balance for most people
- **GGUF for local/CPU/Ollama, GPTQ/AWQ for GPU servers**
- **Don't forget the KV cache** — it's the hidden VRAM cost that surprises people
- **RAM offloading is a last resort**, not a feature — go smaller/more quantized instead

---

## What's Next in the Series

This series covers everything you need to go from "I've heard of LLMs" to running them confidently in your own environment:

- **Part 1**: [Why Self-Host? The Local LLM Primer](/blog/2026/local-llm-why-self-host/)
- **Part 2**: [Framework Showdown — Ollama, LM Studio, llama.cpp, vLLM](/blog/2026/local-llm-framework-showdown/)
- **Part 3**: Quantization Explained ← *you are here*
- **Part 4**: [Open Source Model Landscape — Benchmarks and Picking the Right One](/blog/2026/local-llm-model-landscape/)

---

*Have questions or corrections? Leave a comment below, or find me on GitHub at [@scrowten](https://github.com/scrowten).*
