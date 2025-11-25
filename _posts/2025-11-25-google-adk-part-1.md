---
layout: post
title: 🚀 Build Your First AI Agent with Google’s ADK (Part 1)
date: 2025-11-24 14:25:00
description: Concise agentic tutorial using Google's ADK
tags: adk agents llm
categories: machine-learning llm agents
---

### From Prompt → Thought → Action → Result

AI is evolving fast—and it’s no longer limited to the classic *“you ask, it answers”* pattern.  
Today’s AI can do something far more powerful: **act like an intelligent problem-solver**.

Modern AI **agents** don’t just generate text.  

They can **think through a problem**, **decide what tools to use**, **take meaningful actions**, and **learn from the results** before giving you a final answer.  

It’s like upgrading from a calculator… to a smart assistant that can research, reason, and execute tasks on your behalf.

In this guide, you’ll get hands-on and build your **first real AI agent** using Google’s **Agent Development Kit (ADK)**—the same framework featured in the official Kaggle 5-Day Agents Course.  

No prior experience required. Just curiosity, a few lines of code, and you’re ready to create an AI that doesn’t just respond…  
**it gets things done.** 🚀


By the end, you will:

* ✔️ Install and configure ADK
* ✔️ Set up Gemini API authentication
* ✔️ Build your first agent with reasoning + tools
* ✔️ Watch your agent perform an action (Google Search!)
* ✔️ Explore the ADK Web Interface

Let’s begin. 🚀

---

# ⚙️ 1. Setup

## 1.1 Install Dependencies

If you're using Kaggle Notebooks, **google-adk** is already installed.
To install it locally:

```bash
pip install google-adk
```

---

## 1.2 Configure Your Gemini API Key

ADK uses **Gemini models**, which require authentication.

### 1) Create an API Key

Go to **Google AI Studio** → Generate an API key.

### 2) Add your key to your environment

Inside your project, create a `.env` file:

```bash
GOOGLE_API_KEY="YOUR_KEY_HERE"
```

---

## 1.3 Import ADK Components

These components give your agent LLM reasoning, tools, and execution runtime.

```python
from google.adk.agents import Agent
from google.adk.models.google_llm import Gemini
from google.adk.runners import InMemoryRunner
from google.adk.tools import google_search
from google.genai import types

print("✅ ADK components imported successfully.")
```

---

## 1.4 Configure Retry Behavior

LLMs sometimes hit rate limits or temporary service issues.
Retries help make your agent more reliable.

```python
retry_config = types.HttpRetryOptions(
    attempts=5,            # Max retries
    exp_base=7,            # Exponential backoff multiplier
    initial_delay=1,       # Delay before first retry
    http_status_codes=[429, 500, 503, 504]  # Retryable HTTP error codes
)
```

---

# 🤖 2. Build Your First AI Agent

## 2.1 What *Is* an AI Agent?

A normal LLM:

```
Prompt → LLM → Text
```

An AI agent:

```
Prompt → Agent → Thought → Action → Observation → Final Answer
```

Your agent will be able to:

* Interpret a question
* Decide whether to search the web
* Use Google Search
* Combine results into a final answer

---

## 2.2 Define Your Agent

Here we describe:

* **Name** & **description**
* **Model** (Gemini 2.5 Flash Lite)
* **Instructions** (how the agent should behave)
* **Tools** (the actions it can take — we give it Google Search)

```python
root_agent = Agent(
    name="helpful_assistant",
    model=Gemini(
        model="gemini-2.5-flash-lite",
        retry_options=retry_config
    ),
    description="A simple agent that can answer general questions.",
    instruction="You are a helpful assistant. Use Google Search for current info or if unsure.",
    tools=[google_search],
)

print("✅ Root Agent defined.")
```

---

## 2.3 Run Your Agent

To run an agent, you need a **Runner**.
This acts as the orchestrator that:

* Keeps track of the conversation
* Sends messages to the agent
* Handles tool calls & results

### a) Create the Runner

```python
runner = InMemoryRunner(agent=root_agent)
print("✅ Runner created.")
```

### b) Ask the Agent a Question

```python
response = await runner.run_debug(
    "What is Agent Development Kit from Google? What languages is the SDK available in?"
)
```

This method:

* Creates a temporary session
* Allows the agent to think
* Lets it call Google Search
* Returns the final answer

Perfect for prototyping!

---

## 2.4 What Just Happened?

Your agent:

1. Read your question
2. Realized it needed **current information**
3. Called the **Google Search tool**
4. Reviewed results
5. Generated the final answer

This is the foundation of AI agents:
**LLM → reasoning → tool choice → action → observation → improved answer**

To see a full trace (thoughts, actions, observations), use the ADK Web UI next.

---

# 💻 3. Explore the ADK Web Interface

ADK includes a browser-based UI for:

* Debugging agents
* Viewing traces
* Inspecting tool calls
* Exploring sessions

## 3.1 Create a Sample Agent

Run:

```bash
adk create sample-agent --model gemini-2.5-flash-lite --api_key $GOOGLE_API_KEY
```

This generates:

```
sample-agent/
├── agent.py
├── .env
└── __init__.py
```

## 3.2 Get Your Proxy URL (Kaggle only)

```python
url_prefix = get_adk_proxy_url()
```

## 3.3 Launch the Web UI

```bash
adk web --url_prefix {url_prefix}
```

📌 **Important:**
**Do NOT share your proxy link** — it contains an authentication token.

---

# 🎉 Congratulations!

You just created your **first AI agent** using Google’s ADK!

Key takeaways:

* Agents ≠ plain LLMs
* Agents **reason**, **use tools**, and **act**
* ADK makes building agents simple and structured
* You now have a working agent that can search the internet

Next steps:

* Try adding more tools
* Build multi-agent systems
* Explore advanced session management
* Deploy agents in real applications

---
