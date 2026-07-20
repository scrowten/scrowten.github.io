---
layout: post
title: "🗺️ Open Source Model Landscape: Benchmarks and Picking the Right One"
date: 2026-07-20 00:00:00
description: A practical guide to the open-source LLM landscape in mid-2026 — model families, benchmark comparisons, size-tier recommendations, and how to pick the right model for your hardware and use case. Part 4 of the Self-Hosting LLMs series.
tags: llm local-ai self-hosting ollama models benchmarks
categories: machine-learning llm self-hosting
toc:
  sidebar: left
---

*This is Part 4 of the **Self-Hosting LLMs** series — the final installment. Read this standalone or start from [Part 1](/blog/2026/local-llm-why-self-host/).*

---

You've got your [framework picked out](/blog/2026/local-llm-framework-showdown/). You understand [how quantization works](/blog/2026/local-llm-quantization-explained/). Now comes the question that actually determines your experience: **which model do you run?**

A year ago, the answer was simple — Llama or Mistral, pick a size that fits. Today the landscape is crowded, competitive, and genuinely confusing. There are half a dozen model families, each with multiple sizes, specialized variants, and overlapping claims about being "best."

This post cuts through the noise. I'll map out the major model families, show you how they compare on real benchmarks, and give you concrete recommendations by hardware tier and use case.

---

# 🏔️ 1. The Major Model Families

As of mid-2026, six model families dominate the open-source landscape. Each has a different philosophy, licensing approach, and sweet spot.

### Meta — Llama 4

The model family that started the open-source LLM revolution. Llama 4 introduced a **Mixture of Experts (MoE)** architecture — a major shift from Llama 3's dense models.

| Model | Total Params | Active Params | Architecture | Context |
|---|---|---|---|---|
| Llama 4 Scout | 109B | 17B | 16 experts, 1 active | 10M tokens |
| Llama 4 Maverick | 400B | 17B | 128 experts, 1 active | 1M tokens |

The key insight: **active parameters determine speed, total parameters determine quality.** Scout runs at 17B-model speeds but has 109B of learned knowledge to draw from. Maverick's 10M token context is the longest of any open model.

**License:** Llama Community License (free for most commercial use, restrictions above 700M MAU)

**Best for:** Long-context tasks, general-purpose use, established ecosystem support

### Alibaba — Qwen 3

The most versatile model family in 2026. Qwen 3 spans **eight dense sizes** (0.6B to 32B) plus **two MoE models** (30B-A3B and 235B-A22B), all under Apache 2.0.

| Model | Params | Active | VRAM (Q4) | Notable |
|---|---|---|---|---|
| Qwen3-0.6B | 0.6B | 0.6B | <1 GB | Edge/mobile |
| Qwen3-4B | 4B | 4B | ~2.5 GB | Rivals Qwen 2.5-72B on benchmarks |
| Qwen3-8B | 8B | 8B | ~5 GB | Best all-rounder at this size |
| Qwen3-14B | 14B | 14B | ~9 GB | Sweet spot for 12-16GB GPUs |
| Qwen3-32B | 32B | 32B | ~20 GB | Strong reasoning, fits on 24GB |
| Qwen3-30B-A3B | 30B | 3B | ~5 GB | MoE, runs on 8GB VRAM, outperforms QwQ-32B |
| Qwen3-235B-A22B | 235B | 22B | ~15 GB | Flagship, only 22B active |

The standout: **Qwen3-4B matches Qwen 2.5-72B on benchmarks** — a model needing 3GB of VRAM competing with one needing 43GB. That's how fast the field is moving.

All Qwen 3 models feature a **`/think` toggle** that switches between chain-of-thought reasoning and fast chat mode.

**License:** Apache 2.0 (no restrictions)

**Best for:** The "default recommendation" for most local users — strong across coding, reasoning, and general tasks

### DeepSeek — V3 and R1

DeepSeek runs two parallel model lines: **V3** for general use and **R1** for reasoning.

| Model | Params | Active | Focus |
|---|---|---|---|
| DeepSeek V3 | 671B | 37B | General-purpose MoE |
| DeepSeek R1 | 671B | 37B | Reasoning-focused (chain-of-thought) |
| R1 Distills | 1.5B–70B | Dense | R1 reasoning distilled into smaller models |

**DeepSeek R1** dominates math and logic benchmarks — **97.3% on MATH-500**, with a Codeforces rating in the 96th percentile. The distilled variants (14B and 32B) bring that reasoning capability to consumer hardware.

**License:** MIT (no restrictions)

**Best for:** Math, science, complex reasoning, competitive programming

### Google — Gemma

Google's open model family, optimized for efficiency.

| Model | Params | Active | Notable |
|---|---|---|---|
| Gemma 4 12B-A2B | 12B | 2B | Ultra-efficient MoE |
| Gemma 4 26B-A4B | 26B | 4B | Best practical local recommendation for many |

**Gemma 4 26B-A4B** is interesting — 26B total parameters but only 3.8B active, meaning it runs at small-model speeds with large-model knowledge.

**License:** Gemma Terms of Use (permissive, some restrictions)

**Best for:** Efficiency-focused deployments, when you need maximum quality per VRAM dollar

### Mistral — Devstral and Mistral

Mistral continues to punch above its weight with coding-focused models.

| Model | Params | Focus |
|---|---|---|
| Devstral Small 2 | 24B | Software engineering, coding |
| Mistral Large | 123B | General-purpose |
| Mistral Nemo | 12B | Lightweight, multilingual |

**Devstral Small 2** is purpose-built for software tasks — coding, debugging, code review. It fits on a single 16GB+ GPU and ships under Apache 2.0.

**License:** Apache 2.0 (Devstral, Nemo), Mistral Research License (Large)

**Best for:** Coding-specific tasks, European language support

### Zhipu AI — GLM-5

The newest contender, and currently the strongest all-round open model.

| Model | Params | Architecture | Notable |
|---|---|---|---|
| GLM-5.2 | 744B | MoE | 91.2% GPQA Diamond, 62.1% SWE-bench Pro |
| GLM-5 | Various | Dense | 77.8% SWE-bench Verified (strongest coding) |

GLM-5.2 requires serious hardware (multi-GPU setups), but the smaller GLM-5 variants are more accessible.

**License:** Apache 2.0

**Best for:** Maximum benchmark performance, research-grade tasks

---

# 📊 2. Benchmark Comparison

Benchmarks aren't everything, but they're a useful starting point. Here's how the major models compare across key tasks:

### General Reasoning

| Model | MMLU | GPQA Diamond | ARC-C |
|---|---|---|---|
| GLM-5.2 (744B MoE) | 90.1% | **91.2%** | — |
| Qwen3-235B (22B active) | 88.5% | 77.2% | — |
| Llama 4 Maverick (17B active) | **85.5%** | — | — |
| DeepSeek V3 (37B active) | 88.0% | — | — |
| Qwen3-32B | 84.2% | 68.4% | — |
| Llama 3.1 70B | 86.0% | — | — |

### Math and Reasoning

| Model | MATH-500 | AIME '24 | Codeforces |
|---|---|---|---|
| DeepSeek R1 | **97.3%** | 79.8% | 96th percentile |
| Qwen3-235B | 90.1% | **85.7%** | — |
| GLM-5.2 | 92.0% | 81.5% | — |
| DeepSeek R1 32B distill | 94.3% | — | — |
| Qwen3-32B | 87.5% | — | — |

### Coding

| Model | SWE-bench Verified | HumanEval | LiveCodeBench |
|---|---|---|---|
| GLM-5 | **77.8%** | — | — |
| Qwen3-Coder-480B | 69.6% | — | — |
| DeepSeek V3 | 42.0% | 82.6% | — |
| Devstral Small 2 (24B) | — | 80.1% | — |
| Qwen3-32B | 38.5% | 79.2% | — |

### The Takeaway

No single model wins everything. **DeepSeek R1** dominates math. **GLM-5** leads coding. **Qwen 3** has the broadest size range and best efficiency. **Llama 4** has the strongest ecosystem and longest context.

For *local* use, the models that run on consumer hardware matter more than the 200B+ flagships. That narrows the practical field significantly.

---

# 🎯 3. Recommendations by Size Tier

This is the section you'll actually use. Here's what to run based on how much VRAM (or unified memory) you have:

### Tier 1: 4–8 GB VRAM

*Hardware: RTX 3060 (8GB), M1/M2 MacBook (8GB), GTX 1650*

| Model | VRAM (Q4) | Best For |
|---|---|---|
| **Qwen3-8B** | ~5 GB | General-purpose, all-rounder |
| **Qwen3-30B-A3B** | ~5 GB | MoE, punches way above its weight |
| Gemma 4 12B-A2B | ~4 GB | Ultra-efficient MoE |
| DeepSeek R1 7B distill | ~4.5 GB | Reasoning on a budget |
| Llama 3.2 3B | ~2 GB | Fastest, lightweight tasks |

**My pick:** Start with **Qwen3-8B** for general use. If your tasks are reasoning-heavy, try **DeepSeek R1 7B distill**. The MoE models (**Qwen3-30B-A3B**, **Gemma 4 12B-A2B**) offer remarkable quality for their VRAM footprint.

```bash
ollama run qwen3:8b
ollama run qwen3:30b-a3b
```

### Tier 2: 12–16 GB VRAM

*Hardware: RTX 4070 Ti (16GB), RTX 3080 (12GB), M2 Pro (16GB)*

| Model | VRAM (Q4) | Best For |
|---|---|---|
| **Qwen3-14B** | ~9 GB | Best all-rounder at this tier |
| **DeepSeek R1 14B distill** | ~9 GB | Math, reasoning, debugging |
| Gemma 4 26B-A4B | ~8 GB | Efficient MoE, excellent quality |
| Mistral Nemo 12B | ~8 GB | Multilingual, lightweight |
| Qwen3-32B | ~20 GB | Tight fit at Q3, excellent quality |

**My pick:** **Qwen3-14B** as the daily driver. Switch to **DeepSeek R1 14B** when you need step-by-step reasoning for math or debugging. If you can squeeze it, **Qwen3-32B at Q3_K_M** (~17 GB) is a significant jump in capability.

```bash
ollama run qwen3:14b
ollama run deepseek-r1:14b
```

### Tier 3: 24 GB VRAM

*Hardware: RTX 4090, RTX 3090, M3 Max (36GB)*

| Model | VRAM (Q4) | Best For |
|---|---|---|
| **Qwen3-32B** | ~20 GB | Top recommendation — strong everything |
| **DeepSeek R1 32B distill** | ~20 GB | Exceptional reasoning |
| Devstral Small 2 (24B) | ~15 GB | Purpose-built for coding |
| Llama 4 Scout (17B active) | ~22 GB | Long context champion |

**My pick:** **Qwen3-32B** is the best general-purpose model you can run on a single consumer GPU. For coding specifically, **Devstral Small 2** is purpose-built and excellent. **DeepSeek R1 32B** is the reasoning king at this size.

```bash
ollama run qwen3:32b
ollama run devstral:24b
ollama run deepseek-r1:32b
```

### Tier 4: 48+ GB VRAM (Multi-GPU or Datacenter)

*Hardware: Dual RTX 4090, A6000, A100, H100*

| Model | VRAM (Q4) | Best For |
|---|---|---|
| **Llama 3.3 70B** | ~40 GB | The established benchmark |
| **DeepSeek R1 70B distill** | ~40 GB | Near-frontier reasoning |
| **Qwen3-235B-A22B** | ~15 GB per GPU | Flagship MoE |
| Llama 4 Maverick (17B active) | ~45 GB | 1M token context |

At this tier, you're running models that genuinely compete with cloud APIs on quality. **vLLM with tensor parallelism** becomes the right serving choice.

```bash
vllm serve Qwen/Qwen3-235B-A22B-AWQ \
    --tensor-parallel-size 2 \
    --quantization awq
```

---

# 🧪 4. Recommendations by Use Case

### Coding Assistant

Your local model needs to understand code structure, follow instructions precisely, and generate syntactically correct output.

| Priority | Model | Why |
|---|---|---|
| 1st | **Qwen3-32B** | Strong coding + general ability |
| 2nd | **Devstral Small 2** | Purpose-built for software |
| 3rd | **Qwen3-14B** | Great if 32B doesn't fit |
| Budget | **Qwen3-8B** | Surprisingly capable for its size |

### Math and Reasoning

When you need step-by-step problem solving, logical deduction, or scientific reasoning.

| Priority | Model | Why |
|---|---|---|
| 1st | **DeepSeek R1 32B** | 94.3% MATH-500 at consumer hardware |
| 2nd | **DeepSeek R1 14B** | Strong reasoning, less VRAM |
| 3rd | **Qwen3-32B** (thinking mode) | Good reasoning with `/think` toggle |

### Document Summarization and RAG

For processing long documents, answering questions over knowledge bases, or building RAG pipelines.

| Priority | Model | Why |
|---|---|---|
| 1st | **Llama 4 Scout** | 10M token context window |
| 2nd | **Qwen3-32B** | 32K context, strong comprehension |
| 3rd | **Qwen3-14B** | Good quality at lower VRAM |

### General Chat / Personal Assistant

For everyday questions, drafting emails, brainstorming, and general knowledge.

| Priority | Model | Why |
|---|---|---|
| 1st | **Qwen3-14B** or **Qwen3-32B** | Best all-rounders |
| 2nd | **Qwen3-30B-A3B** | MoE efficiency, runs on 8GB |
| 3rd | **Llama 3.2 3B** | Ultra-fast, good for simple tasks |

---

# ⚖️ 5. How to Actually Evaluate

Benchmarks tell you what a model *can* do in ideal conditions. They don't tell you how it *feels* to use for your specific work. Here's a practical evaluation approach:

### The 10-Prompt Test

Pick 10 real prompts from your actual work — not toy examples. Run each through 2–3 candidate models and rate:

1. **Accuracy** — Is the output correct?
2. **Instruction following** — Did it do what you asked, or what it wanted?
3. **Speed** — Is it fast enough for interactive use? (Under 1 second first-token latency, 20+ tok/s generation)
4. **Format** — Does it produce clean output, or wrap everything in unnecessary markdown?

### Speed Matters More Than You Think

A model that's 5% better on benchmarks but half as fast will frustrate you in practice. For interactive use (coding, chat, quick questions), these are rough minimums:

- **Time to first token**: Under 1 second
- **Generation speed**: 20+ tok/s for comfortable reading, 10+ tok/s for usable
- **Context window**: 8K minimum for coding (to fit surrounding code), 32K+ for documents

### The MoE Advantage

Mixture of Experts models (Qwen3-30B-A3B, Gemma 4 26B-A4B, Llama 4 Scout) deserve special attention. They route each token through a subset of parameters, giving you:
- **Large-model knowledge** at small-model speeds
- **Lower VRAM usage** than their total parameter count suggests
- **Better quality per compute** than equivalent dense models

The trade-off: MoE models use more disk space (full model weights are stored) and can have slightly less consistent output quality across domains.

---

# 🔄 6. The Landscape Will Keep Moving

A year ago, running a competitive local model meant Llama 2 70B at Q4 on a beefy GPU. Today, **Qwen3-4B at 3GB VRAM matches what Qwen 2.5-72B at 43GB could do** — a 14x efficiency gain in one generation.

This pace will continue. By the time you read this, there may be newer models worth trying. Here's how to stay current:

- **[Ollama library](https://ollama.com/library)** — check for new model additions weekly
- **[Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard)** — HuggingFace's benchmark tracker
- **r/LocalLLaMA** — the Reddit community that tests everything first
- **[Chatbot Arena](https://lmarena.ai/)** — blind human preference rankings

The good news: the tools ([Ollama](/blog/2026/local-llm-framework-showdown/), [quantization](/blog/2026/local-llm-quantization-explained/)) and concepts you've learned in this series stay relevant even as the models change. `ollama pull` works the same whether the model behind it is Llama 4 or Llama 7.

---

# 📚 Series Recap

You made it through the whole series. Here's what we covered:

- **Part 1**: [Why Self-Host?](/blog/2026/local-llm-why-self-host/) — The cost math, privacy case, and your first 5-minute setup
- **Part 2**: [Framework Showdown](/blog/2026/local-llm-framework-showdown/) — Ollama, LM Studio, llama.cpp, vLLM — which to use when
- **Part 3**: [Quantization Explained](/blog/2026/local-llm-quantization-explained/) — How a 70B model fits on a 24GB GPU
- **Part 4**: Open Source Model Landscape ← *you are here*

If you've followed along, you now know:
- ✔️ **Why** to self-host (cost, privacy, control)
- ✔️ **How** to set up your tools (Ollama for most, vLLM for production)
- ✔️ **What** quantization settings to use (Q4_K_M default, GGUF format)
- ✔️ **Which** model to run (Qwen 3 for most, DeepSeek R1 for reasoning, Devstral for coding)

The local LLM ecosystem is no longer a hobbyist curiosity — it's a practical, competitive alternative to cloud APIs for a wide range of tasks. And it runs on hardware you probably already own.

---

*Questions, corrections, or model recommendations I missed? Leave a comment below, or find me on GitHub at [@scrowten](https://github.com/scrowten).*
