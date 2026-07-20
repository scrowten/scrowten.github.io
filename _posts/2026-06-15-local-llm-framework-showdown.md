---
layout: post
title: "🥊 Framework Showdown: Ollama vs LM Studio vs llama.cpp vs vLLM"
date: 2026-06-15 00:00:00
description: A hands-on comparison of the four main local LLM frameworks — what each does best, performance benchmarks, and which one to pick for your use case. Part 2 of the Self-Hosting LLMs series.
tags: llm local-ai self-hosting ollama vllm llama-cpp
categories: machine-learning llm self-hosting
toc:
  sidebar: left
---

*This is Part 2 of the **Self-Hosting LLMs** series. [Part 1](/blog/2026/local-llm-why-self-host/) covers why self-hosting is worth it.*

---

You've decided to run an LLM locally. You've read about the [cost savings and privacy benefits](/blog/2026/local-llm-why-self-host/). Now comes the practical question: **which tool do you actually use?**

The ecosystem has converged around four main options, and they're not interchangeable — each fills a different niche. Picking the wrong one won't break anything, but picking the right one saves you hours of frustration.

This post gives you a hands-on tour of all four, with real benchmarks, configuration examples, and a decision framework at the end.

---

# 🗺️ 1. The Landscape at a Glance

Before diving deep, here's how these four relate to each other:

```
┌─────────────────────────────────────────────────────┐
│                   Experience Layers                  │
│                                                     │
│     ┌──────────┐          ┌───────────┐             │
│     │  Ollama   │          │ LM Studio │             │
│     │  (CLI)    │          │   (GUI)   │             │
│     └────┬─────┘          └─────┬─────┘             │
│          │                      │                    │
│          ▼                      ▼                    │
│  ┌──────────────────────────────────────┐            │
│  │         llama.cpp / MLX              │  ◀─ Engine │
│  │     (C++ inference runtime)          │            │
│  └──────────────────────────────────────┘            │
│                                                     │
│  ┌──────────────────────────────────────┐            │
│  │              vLLM                    │  ◀─ Server │
│  │     (Python production server)       │            │
│  └──────────────────────────────────────┘            │
└─────────────────────────────────────────────────────┘
```

**Ollama** and **LM Studio** are experience layers — they wrap an engine (llama.cpp, and on Apple Silicon, MLX) and make it easy to use. **llama.cpp** is the engine itself — maximum control, maximum portability. **vLLM** is a different beast entirely — a production serving system designed for throughput, not convenience.

Understanding this hierarchy saves confusion: Ollama isn't competing with llama.cpp, it's built *on* llama.cpp.

---

# 🦙 2. Ollama — The Docker of LLMs

**What it is:** A single Go binary that wraps llama.cpp (and MLX on Apple Silicon) with a model registry, automatic GPU detection, and an OpenAI-compatible REST API.

**Best for:** Solo developers who want to get running in under 2 minutes.

### Install and Run

```bash
# Install (macOS / Linux)
curl -fsSL https://ollama.com/install.sh | sh

# Pull and run a model
ollama run qwen3:8b
```

That's it. Ollama detects your GPU (NVIDIA, AMD, or Apple Silicon), downloads the model, loads it, and gives you a chat prompt. No Python environment, no CUDA toolkit, no config files.

### The Model Registry

Ollama maintains a curated library of 500+ pre-quantized models at [ollama.com/library](https://ollama.com/library). Models follow a Docker-like naming convention:

```bash
ollama pull llama3.1:8b              # default quant (Q4_K_M)
ollama pull llama3.1:8b-q5_K_M       # specific quant
ollama pull qwen3:32b                # larger model
ollama pull deepseek-r1:14b          # reasoning model
```

This is one of Ollama's biggest advantages — you don't need to browse Hugging Face, figure out which GGUF file to download, or worry about tokenizer configs. It just works.

### OpenAI-Compatible API

Every Ollama install exposes a REST API on port 11434 that mirrors the OpenAI chat completions format:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama",
)

response = client.chat.completions.create(
    model="qwen3:8b",
    messages=[{"role": "user", "content": "Explain PagedAttention in 3 sentences."}],
)

print(response.choices[0].message.content)
```

This means **any tool that supports the OpenAI API works with Ollama** — LangChain, LlamaIndex, Continue.dev, Open WebUI, and hundreds more. Same code, different URL.

### Modelfiles — Custom Model Variants

Ollama's `Modelfile` system lets you create custom model configurations:

```dockerfile
FROM qwen3:8b

SYSTEM """You are a senior Python developer. Always include type hints,
write clean code, and explain your reasoning step by step."""

PARAMETER temperature 0.3
PARAMETER num_ctx 8192
```

```bash
ollama create my-coder -f Modelfile
ollama run my-coder
```

This bakes a system prompt, temperature, and context length into a named model variant you can share or reuse.

### GPU Management

Ollama handles GPU detection automatically:
- **NVIDIA**: Uses CUDA, detects multi-GPU setups
- **AMD**: ROCm support for RX 6000/7000 series
- **Apple Silicon**: Uses Metal (and MLX since Ollama 0.19+)
- **Fallback**: Automatically offloads to CPU if VRAM is insufficient

You can check how a model is loaded:

```bash
ollama ps
# NAME         ID         SIZE      PROCESSOR    UNTIL
# qwen3:8b     a6990ed6   5.2 GB    100% GPU     4 minutes from now
```

### Limitations

- **Single-user optimized**: Under concurrent load, throughput drops significantly (~41 tok/s vs vLLM's ~793 tok/s in benchmarks)
- **Limited quantization control**: You get what the registry offers; custom GGUF files require a Modelfile workaround
- **No tensor parallelism**: Can't split a model across multiple GPUs for inference (it can use multiple GPUs, but loads complete model copies)

---

# 🖥️ 3. LM Studio — The GUI-First Experience

**What it is:** A desktop application (Electron-based) with a built-in Hugging Face model browser, chat interface, and local API server.

**Best for:** People who prefer clicking over typing, or who want to visually browse and compare models.

### The Model Browser

LM Studio's standout feature is its **Discover** tab — a visual model browser connected directly to Hugging Face. You search for a model, see:
- Compatibility ratings based on your hardware
- File sizes for each quantization level
- Download counts and community ratings
- VRAM requirements with your specific GPU highlighted

Then you click **Download**. No command line, no manual file management.

### Chat Interface

The **Chat** tab feels like using ChatGPT, but everything runs locally. You can:
- Adjust temperature, top-p, and context length with sliders
- Compare outputs from different models side-by-side
- Save and organize conversation histories
- Upload documents for in-context processing (RAG)

### Developer Mode — Local API Server

LM Studio can start a local server on port 1234 that's OpenAI API-compatible:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio",
)

response = client.chat.completions.create(
    model="qwen3-8b",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

It also ships `lms`, a CLI tool, and `llmster`, a headless daemon for running models without the GUI.

### When LM Studio Shines

- **Model discovery**: The visual browser is genuinely useful when you're exploring options
- **Quick A/B testing**: Load two models, send the same prompt, compare results visually
- **Non-terminal users**: If your workflow is GUI-native, LM Studio fits naturally
- **Document chat**: Built-in RAG for chatting with local files

### Limitations

- **Electron overhead**: Heavier on system resources than Ollama's single binary
- **Smaller model registry**: Relies on Hugging Face directly — more models available, but less curation than Ollama's registry
- **macOS and Windows only**: No official Linux support (though community workarounds exist)
- **Closed source**: Unlike the other three tools, LM Studio itself is not open source

---

# ⚙️ 4. llama.cpp — The Engine Under the Hood

**What it is:** A C/C++ implementation of LLM inference, originally created by Georgi Gerganov. It's the engine that powers Ollama, LM Studio, and much of the local LLM ecosystem.

**Best for:** Power users who want maximum control, embedded deployments, or need to run on unusual hardware.

### Why llama.cpp Matters

llama.cpp is the reason local LLMs are practical. Before it existed (March 2023), running a language model required Python, PyTorch, and a CUDA-capable GPU. llama.cpp proved you could run Llama models in pure C++ on a MacBook CPU — and the local LLM movement exploded from there.

Today the project has 118K+ GitHub stars, 450+ contributors, and ships new builds almost daily.

### Direct Usage

```bash
# Build from source
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON  # or DGGML_METAL=ON for Mac
cmake --build build --config Release

# Run a model
./build/bin/llama-cli \
    -m models/qwen3-8b-q4_K_M.gguf \
    -p "Explain the difference between TCP and UDP" \
    -n 256
```

### Server Mode

llama.cpp includes a built-in HTTP server with an impressive feature set:

```bash
./build/bin/llama-server \
    -m models/qwen3-8b-q4_K_M.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    --n-gpu-layers 99 \
    --ctx-size 8192 \
    --parallel 4
```

Server mode features include:
- **OpenAI + Anthropic-compatible API routes**
- **Parallel decoding** for multiple concurrent requests
- **Function calling** support
- **Speculative decoding** for 2–3x faster inference with a draft model
- **Grammar-constrained generation** — force outputs to follow a JSON schema or other grammar
- **Embeddings and reranking** endpoints
- **Router mode** for loading multiple models with LRU eviction
- **Built-in web UI** with chat interface

### Speculative Decoding

One of llama.cpp's most powerful features — use a small "draft" model to propose tokens, then verify them with the main model:

```bash
./build/bin/llama-speculative \
    -m models/qwen3-8b-q4_K_M.gguf \
    -md models/qwen3-0.6b-q8_0.gguf \
    -p "Write a Python HTTP server" \
    --draft-max 8
```

This can achieve **2–3x speedup** on generation with minimal quality impact, because the small model's correct predictions are kept and only wrong ones are regenerated.

### Grammar-Constrained Generation

Force the model to output valid JSON, SQL, or any format defined by a GBNF grammar:

```bash
./build/bin/llama-cli \
    -m models/qwen3-8b-q4_K_M.gguf \
    --grammar-file grammars/json.gbnf \
    -p "List 3 programming languages with their year of creation"
```

This guarantees syntactically valid output — something you can't easily get from cloud APIs without post-processing.

### When to Use llama.cpp Directly

- **Embedded systems**: Runs on Raspberry Pi, Android, iOS, WebAssembly
- **Custom builds**: Need specific CUDA flags, quantization options, or experimental features
- **Speculative decoding**: Ollama doesn't expose this (yet)
- **Grammar-constrained output**: When you need guaranteed-valid structured output
- **Maximum performance**: Direct llama.cpp can squeeze out more tok/s than the Ollama wrapper

### Limitations

- **More setup**: You need to build from source (or download pre-built binaries), manage GGUF files yourself, and configure options manually
- **No model registry**: You download models from Hugging Face yourself
- **GGUF only**: Doesn't support GPTQ or AWQ formats (use vLLM for those)

---

# 🚀 5. vLLM — The Production Powerhouse

**What it is:** A Python-based LLM serving system built for maximum throughput. Uses PagedAttention, continuous batching, and tensor parallelism to serve many concurrent users efficiently.

**Best for:** Team deployments, production APIs, and any scenario where you need to serve multiple users simultaneously.

### The Throughput Gap

Here's why vLLM exists — single-user performance is roughly the same across all tools (~30 tok/s on an RTX 4090 with a 24B model). But under concurrent load:

| Tool | Single user | 10 concurrent users | 50 concurrent users |
|---|---|---|---|
| Ollama | ~30 tok/s | ~41 tok/s total | Queues requests |
| llama.cpp server | ~30 tok/s | ~120 tok/s total | ~200 tok/s total |
| vLLM | ~30 tok/s | ~400 tok/s total | ~793 tok/s total |

vLLM achieves **16–20x Ollama's concurrent throughput**. If you're building something that more than one person will use at the same time, this matters enormously.

### How It Achieves This

**PagedAttention** — Instead of pre-allocating contiguous memory for each request's KV cache, vLLM borrows the concept of virtual memory paging from operating systems. Memory is divided into fixed-size blocks allocated on-demand, reducing waste by up to 4x and enabling much larger batch sizes.

**Continuous batching** — Traditional serving waits for an entire batch to finish before starting a new one. vLLM uses rolling batches — as soon as one sequence finishes, a new request immediately takes its slot. GPU utilization stays high even with variable-length outputs.

**Tensor parallelism** — Split a model across 2, 4, or 8 GPUs transparently. A 70B model that doesn't fit on one GPU runs across two with near-linear scaling.

### Getting Started

```bash
pip install vllm

# Start the server
vllm serve meta-llama/Llama-3.1-8B-Instruct \
    --dtype auto \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.9
```

vLLM's API is also OpenAI-compatible:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="vllm",
)

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

### Multi-GPU Deployment

```bash
# Tensor parallel across 2 GPUs
vllm serve meta-llama/Llama-3.1-70B-Instruct \
    --tensor-parallel-size 2 \
    --dtype auto \
    --quantization awq
```

### Quantization Support

vLLM supports quantization formats optimized for GPU throughput:
- **GPTQ** — calibrated 4-bit quantization with Marlin kernels (~5x faster than naive INT4)
- **AWQ** — activation-aware quantization, good quality-speed balance
- **FP8** — 8-bit floating point, minimal quality loss, supported on H100/RTX 4090

Note: vLLM does **not** support GGUF format. If you need GGUF, use Ollama or llama.cpp.

### When to Use vLLM

- **Multi-user serving**: Internal team tools, API endpoints, chatbots with concurrent users
- **Maximum throughput**: Batch processing, document pipelines, high-volume inference
- **Multi-GPU setups**: When your model needs more than one GPU
- **Production deployments**: When uptime, monitoring, and scaling matter

### Limitations

- **Slow cold start**: Loading models takes minutes (compiles CUDA kernels on first run)
- **GPU required**: No CPU fallback — you need NVIDIA (or AMD ROCm) GPUs
- **Heavier setup**: Python environment, more dependencies, more configuration
- **Overkill for personal use**: If it's just you querying a model, Ollama is simpler and just as fast

---

# 📊 6. Head-to-Head Comparison

| Feature | Ollama | LM Studio | llama.cpp | vLLM |
|---|---|---|---|---|
| **Setup time** | 2 min | 5 min | 15-30 min | 10-20 min |
| **Interface** | CLI + API | GUI + API | CLI + API | API |
| **Model format** | GGUF (auto) | GGUF (auto) | GGUF | GPTQ, AWQ, FP8 |
| **Model registry** | Built-in (500+) | HuggingFace browser | Manual download | HuggingFace (auto) |
| **GPU support** | NVIDIA, AMD, Apple | NVIDIA, Apple | NVIDIA, AMD, Apple, Vulkan | NVIDIA, AMD |
| **CPU inference** | Yes (fallback) | Yes (fallback) | Yes (primary feature) | No |
| **Multi-GPU** | Limited | No | Layer splitting | Tensor parallelism |
| **Concurrent users** | Poor (~41 tok/s) | Poor | Moderate (~200 tok/s) | Excellent (~793 tok/s) |
| **Speculative decoding** | No | No | Yes (2-3x speedup) | Yes |
| **Grammar constraints** | No | No | Yes | Yes (guided decoding) |
| **OpenAI API** | Yes | Yes | Yes | Yes |
| **Open source** | Yes (MIT) | No | Yes (MIT) | Yes (Apache 2.0) |
| **Best for** | Getting started | Visual exploration | Power users, embedded | Production serving |

---

# 🎯 7. Decision Framework

### Start Here

```
Are you a solo developer getting started with local LLMs?
  └─→ Ollama. Done. You can always switch later.

Do you prefer a GUI over terminal?
  └─→ LM Studio for browsing and chatting.
      Still use Ollama for API integration.

Are you building something multiple people will use?
  └─→ vLLM. The throughput difference is massive.

Do you need structured output, speculative decoding,
or deployment on unusual hardware?
  └─→ llama.cpp directly.
```

### The Progression Most People Follow

1. **Start with Ollama** — get your first model running, experiment with different sizes
2. **Try LM Studio** — visually browse models, compare outputs side-by-side
3. **Hit a limit** — you need more speed, structured output, or multi-user support
4. **Graduate to llama.cpp or vLLM** — depending on whether you need control or throughput

Most developers stay at step 1 or 2 for a long time. That's fine. The experience layers exist precisely so you don't need to think about the engine.

### Can You Mix Them?

Absolutely. A common setup:
- **Ollama** on your laptop for development and quick queries
- **vLLM** on a GPU server for your team's shared inference endpoint
- **llama.cpp** in a CI pipeline for structured test output with grammar constraints

The OpenAI-compatible API standard means your application code doesn't care which backend is running. Change the URL, keep the code.

---

# 🔮 What's Next

This series covers everything from motivation to model selection:

- **Part 1**: [Why Self-Host? The Local LLM Primer](/blog/2026/local-llm-why-self-host/)
- **Part 2**: Framework Showdown ← *you are here*
- **Part 3**: [Quantization Explained — How a 70B Model Fits on Your Laptop](/blog/2026/local-llm-quantization-explained/)
- **Part 4**: [Open Source Model Landscape — Benchmarks and Picking the Right One](/blog/2026/local-llm-model-landscape/)

Next up: if you've picked your framework and want to understand *which model* to run on it — head to [Part 4](/blog/2026/local-llm-model-landscape/). If you want to understand the quantization settings you keep seeing (Q4_K_M, Q5_K_S, GPTQ), [Part 3](/blog/2026/local-llm-quantization-explained/) breaks it all down.

---

*Questions or corrections? Leave a comment below, or find me on GitHub at [@scrowten](https://github.com/scrowten).*
