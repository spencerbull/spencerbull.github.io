---
layout: post
title: "Building a Local AI Robot Assistant with Reachy Mini and Dell Pro Max GB10"
date: 2026-01-30
tags: [ai, robotics, langgraph, vllm, nvidia, edge-ai]
description: "How I built a personal AI assistant that runs entirely on local hardware — no cloud APIs required."
---

There's something deeply satisfying about AI that runs on hardware you can touch. No API calls disappearing into the cloud. No rate limits. No "sorry, the service is unavailable." Just silicon and software doing exactly what you tell it to.

This is the story of how I built a personal AI assistant using a [Reachy Mini](https://www.pollen-robotics.com/) robot and a Dell Pro Max with NVIDIA GB10, running 30B+ parameter models entirely locally.

![Reachy Demo](/assets/images/reachy-demo.png)

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

## The Software Stack

### LangGraph for Agent Orchestration

I chose [LangGraph](https://python.langchain.com/docs/langgraph/) over raw prompting or simpler agent frameworks because:

1. **Stateful by design** — conversation history, user preferences, and spatial memory persist across sessions
2. **Graph-based routing** — intent classification determines which node handles each request
3. **Tool calling** — structured interaction with the robot hardware and external services
4. **MCP integration** — Model Context Protocol for connecting to calendars, files, and more

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

### vLLM for Local Inference

Running models locally with [vLLM](https://github.com/vllm-project/vllm) in Docker containers:

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
- **Qwen3-VL-30B** (FP8) — Main agent with vision and tool calling. Handles complex requests, describes what it sees, and controls the robot.
- **Phi-3 Mini** — Fast router for intent classification. Determines whether a request needs vision, tools, or simple conversation.

The key insight: use a tiny model for routing (~50ms) and only invoke the big model when needed. Most chitchat doesn't need 30B parameters.

## Key Features

### Spatial Memory

The robot remembers where things are:

> "I put my keys on the shelf by the door."
> 
> *[Later...]*
> 
> "Where are my keys?"
> 
> "Your keys are on the shelf by the door — you mentioned putting them there earlier."

This uses a simple but effective spatial memory system that organizes objects by room/location and persists across sessions.

### Face Tracking

When you say "look at me," the robot uses MediaPipe for real-time face detection and smoothly tracks your face with its head servos. It maintains eye contact naturally during conversation.

### Emotional Expression

The robot expresses emotions through:
- Antenna position (perked up = curious, drooped = sad)
- Head tilt patterns
- Movement speed and smoothness

These are inferred from conversation context and explicitly called via tools when appropriate.

### Vision Understanding

With Qwen3-VL, the robot can actually see:

> "What's on my desk?"
> 
> "I can see a coffee mug, a notebook, your keyboard, and what looks like a small potted plant. The mug appears to be empty — need a refill?"

## What I Learned

1. **Routing is everything.** A fast, accurate router makes the system feel responsive. Don't send everything to your biggest model.

2. **Memory is harder than it looks.** Deciding what to remember, how to index it, and when to retrieve it is its own research problem.

3. **Unified memory changes the game.** GB10's unified CPU/GPU memory means no more juggling model weights between devices. Just load and run.

4. **Robots are fun.** There's something magical about AI that can move and react physically. It changes how you interact with it.

## What's Next

- **Fine-tuning** Qwen3 on robot-specific interactions
- **Better MCP integrations** — home automation, calendar, music
- **Multi-robot coordination** — I may have ordered another Reachy...
- **Voice cloning** — custom TTS voice for more personality

## Try It Yourself

The code is open source: [github.com/spencerbull/reachy-personal-assistant](https://github.com/spencerbull/reachy-personal-assistant/tree/spark-local-langgraph)

You'll need:
- Reachy Mini (or run in simulation mode)
- NVIDIA GPU with ~60GB+ VRAM for the full stack (or use cloud APIs)
- Python 3.11+, Docker, and patience

The `spark-local-langgraph` branch has the local model setup. Check the README for detailed instructions.

---

*Got questions or want to share your own robot projects? Find me on [GitHub](https://github.com/spencerbull).*
