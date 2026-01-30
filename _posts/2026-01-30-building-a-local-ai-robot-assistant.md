---
layout: post
title: "Building a Local AI Robot Assistant with Reachy Mini and Dell Pro Max GB10"
date: 2026-01-30
tags: [ai, robotics, langgraph, vllm, nvidia, edge-ai]
description: "How I took the NVIDIA/Hugging Face Reachy Mini demo from CES 2026 and made it run entirely on local hardware — no cloud APIs required."
---

At CES 2026, NVIDIA and Hugging Face unveiled an incredible demo: a personal AI assistant running on a [Reachy Mini](https://www.pollen-robotics.com/) robot powered by DGX Spark. Jensen Huang showed how you could have your own "office R2D2" — an AI buddy that can see, speak, and act.

I wanted to take it further: **what if everything ran locally?**

This is the story of how I forked the [original demo](https://huggingface.co/blog/nvidia-reachy-mini) and modified it to run 30B+ parameter models entirely on a Dell Pro Max with GB10 — no cloud APIs, no rate limits, complete privacy.

![Reachy Demo](/assets/images/reachy-demo.png)

## The Original Demo

The [NVIDIA/Hugging Face CES demo](https://huggingface.co/blog/nvidia-reachy-mini) is an excellent starting point. It uses:
- **NeMo Agent Toolkit** for agent orchestration and ReAct-style tool calling
- **Pipecat** for real-time voice and vision streaming
- **NVIDIA Nemotron models** via cloud APIs (build.nvidia.com)
- **Reachy Mini** as the physical embodiment

The architecture routes queries intelligently: text questions go to a fast model, visual queries hit a VLM, and action requests use a ReAct agent with tools. It's well-designed and works great.

But I wanted to cut the cord to the cloud.

## Why Local?

The obvious answer is privacy and latency — but honestly, I wanted to see what's possible when you have ~128GB of GPU memory sitting on your desk. Cloud APIs are great for prototyping, but there's a ceiling to what you can do when every inference costs money and adds 200ms of round-trip latency.

With local inference:
- **Zero latency** for routing decisions (Phi-3 responds in <50ms)
- **No API costs** after hardware investment
- **Full control** over model behavior and fine-tuning
- **Offline operation** — it works on an airplane

## The Hardware

### Reachy Mini
[Reachy Mini](https://www.pollen-robotics.com/) is a desktop robot from Pollen Robotics with:
- Expressive head with pan/tilt control
- Animated antenna for emotional expression
- Built-in camera for vision
- Smooth servo movement

It's designed for human-robot interaction research, but it makes a fantastic desk companion.

### Dell Pro Max with GB10

The Dell Pro Max workstation with NVIDIA GB10 Grace Blackwell is a beast:
- **GB10 Superchip** — Grace CPU + Blackwell GPU on unified memory
- **128GB unified memory** — enough for 70B+ models in FP8
- **Compact form factor** — sits under your desk, not in a server room

This is the "edge AI" dream realized: datacenter-class inference in a desktop form factor.

## What I Changed

### Swapped NeMo Agent Toolkit for LangGraph

The original demo uses [NeMo Agent Toolkit](https://github.com/NVIDIA/NeMo-Agent-Toolkit) for orchestration. It's solid, but I wanted more control over state management and memory persistence. I rebuilt the agent using [LangGraph](https://python.langchain.com/docs/langgraph/) because:

1. **Stateful by design** — conversation history, user preferences, and spatial memory persist across sessions
2. **Graph-based routing** — intent classification determines which node handles each request
3. **MCP integration** — Model Context Protocol for connecting to calendars, files, and more

The architecture:

![Architecture](/assets/images/reachy-arch.png)

```
┌─────────────────────────────────────────┐
│           Pipecat Pipeline              │
│  STT → LangGraph Agent → TTS            │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│           LangGraph Agent               │
│  Router → Vision/Conversation/Tools     │
└─────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
   [Reachy]    [Memory]     [MCP]
   Movement    Spatial      Calendar
   Emotions    Long-term    Files
   Tracking    History      GitHub
```

### Local Models via vLLM

Instead of hitting NVIDIA's cloud endpoints, I run models locally with [vLLM](https://github.com/vllm-project/vllm) in Docker containers:

```yaml
services:
  agent-llm:
    image: nvcr.io/nvidia/vllm:25.12-py3
    command: >
      vllm serve Qwen/Qwen3-VL-30B-A3B-Instruct-FP8
      --port 8000
      --gpu-memory-utilization 0.6
      --max-model-len 64000
      --enable-auto-tool-choice
      --tool-call-parser hermes
    ports:
      - "8002:8000"

  routing-llm:
    command: >
      vllm serve microsoft/Phi-3-mini-4k-instruct
      --gpu-memory-utilization 0.15
      --max-model-len 4096
    ports:
      - "8003:8000"
```

**Model choices:**
- **Qwen3-VL-30B** (FP8) — Main agent with vision and tool calling. Handles complex requests, describes what it sees, and controls the robot. (Replaces Nemotron VLM)
- **Phi-3 Mini** — Fast router for intent classification. Same as the original demo's router.

The key insight from the original demo still applies: use a tiny model for routing (~50ms) and only invoke the big model when needed.

### Added Memory Systems

The original demo is stateless — each conversation starts fresh. I added:

**Spatial Memory** — The robot remembers where things are:

> "I put my keys on the shelf by the door."
> 
> *[Later...]*
> 
> "Where are my keys?"
> 
> "Your keys are on the shelf by the door — you mentioned putting them there earlier."

**Long-term Memory** — User preferences persist across sessions.

**Emotional State** — The robot tracks and expresses emotions through antenna movements and head tilts.

## What I Learned

1. **The original demo's routing architecture is solid.** The three-way split (text/vision/tools) is the right abstraction. I kept it.

2. **Memory is harder than it looks.** Deciding what to remember, how to index it, and when to retrieve it is its own research problem.

3. **Unified memory changes the game.** GB10's unified CPU/GPU memory means no more juggling model weights between devices. Just load and run.

4. **Robots are fun.** There's something magical about AI that can move and react physically. It changes how you interact with it.

## What's Next

- **Fine-tuning** Qwen3 on robot-specific interactions
- **Better MCP integrations** — home automation, calendar, music
- **Multi-robot coordination** — I may have ordered another Reachy...
- **Voice cloning** — custom TTS voice for more personality

## Try It Yourself

**Original demo:** [huggingface.co/blog/nvidia-reachy-mini](https://huggingface.co/blog/nvidia-reachy-mini)

**My local fork:** [github.com/spencerbull/reachy-personal-assistant](https://github.com/spencerbull/reachy-personal-assistant/tree/spark-local-langgraph)

You'll need:
- Reachy Mini (or run in simulation mode)
- NVIDIA GPU with ~60GB+ VRAM for the full stack (or use cloud APIs as fallback)
- Python 3.11+, Docker, and patience

The `spark-local-langgraph` branch has the local model setup. Check the README for detailed instructions.

---

*Thanks to the teams at NVIDIA, Hugging Face, and Pollen Robotics for the original demo and open-sourcing the code. Standing on the shoulders of giants.*

*Got questions or want to share your own robot projects? Find me on [GitHub](https://github.com/spencerbull).*
