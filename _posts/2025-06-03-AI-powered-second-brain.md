---
layout: post
title: AI powered Second Brain
subtitle: run a local LLM to access your notes!
#bigimg: /img/akraino.png
tags:
  AI
category: AI
---

# Building a Local LLM-Powered Second Brain with NVIDIA Jetson AGX

## What Is a Second Brain?

A **Second Brain** is an external system for organizing and retrieving knowledge—your personal knowledge management space. Unlike simple note-taking, a Second Brain connects ideas and insights, helping you think better and be more creative.

## Why Obsidian?

For this project, we use [Obsidian](https://obsidian.md), a powerful note-taking tool based on local Markdown files. Obsidian allows you to:

- Interconnect notes using internal links.
- Visualize knowledge in a network graph.
- Extend features using plugins.

### The Smart Second Brain Plugin

To bring LLM capabilities to Obsidian, install the **Smart Second Brain** community plugin:

1. Go to **Settings → Community Plugins**.
2. Search for **Smart Second Brain**.
3. Install and enable it.
4. Configure your LLM and embedding models in its settings.

## Why Use the NVIDIA Jetson AGX?

The [NVIDIA Jetson AGX Orin](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-orin/) is a compact, high-performance device designed for edge AI workloads. It features a powerful GPU and runs CUDA-enabled software, making it perfect for running local LLMs without depending on the cloud. In this case, we are using Red Hat Enterprisa Linux 9.5, a fully supported distribution on this device with the NVIDIA drivers installed.

## Run Local LLMs with Ollama and Obsidian configuration

To run models locally, we use [Ollama](https://ollama.com), a tool that simplifies managing and serving LLMs on your own hardware.

Here’s how to run a containerized version of Ollama optimized for Jetson using Podman:

```bash
mkdir ~/ollama

podman run -d -it --name ollama \
  --device nvidia.com/gpu=all \
  --group-add keep-groups \
  --security-opt label=disable \
  --rm --network=host \
  -v ~/ollama:/ollama \
  -e OLLAMA_MODELS=/ollama \
  dustynv/ollama:0.6.8-r36.4-cu126-22.04
```

Once the container is running, you can interact with Ollama using podman in order to download a model that can be then configured in Obsidian.

![Smart Second Brain Settings](/img/second-brain/second-brain-settings.png "Settings")

As you can see, there are a bunch of default models that the Smart Second Brain plugin suggests, including embeddings models.


> 💡 **Embeddings** help the LLM understand and retrieve relevant information from your notes. They're essential for contextual and semantic search.

There is also the possibility to run models via Third-Party Services like OpenAI.

![Smart Second Brain Settings](/img/second-brain/third-party-settings.png "Settings")


## Lessons Learned

Initial attempts at running LLMs locally delivered fast response times, but the results were often irrelevant or incorrect. The retrieval accuracy simply wasn't good enough for a usable Second Brain experience.

Switching to **OpenAI’s hosted models** brought a significant improvement. The responses were not only quick but also highly accurate and contextually relevant, making the system feel much more intelligent and helpful.

However, continued testing of local models led to an important discovery: using the `all-minilm` embeddings model drastically improved the relevance of results even when running everything locally.

Among the local LLMs tested, `llama3.2` stood out. It offered a great balance of speed and accuracy, making it ideal for note retrieval and general queries. In contrast, larger reasoning models—while more powerful—were unnecessarily complex and too slow for simple lookup tasks.
The takeaway? With the right combination of embeddings and a lightweight, capable LLM, running your Second Brain entirely offline is not only possible — it’s effective.


### ☕ Real-World Test: Coffee Math
As a practical experiment, `llama3.2` was asked to calculate the average cost per cup of coffee from the notes. It:
1. Retrieved the cost per kilogram from the vault.
2. Assumed 100 cups per kilogram.
3. Delivered an accurate, reasoned answer.

Look at the screenshot:

![Coffee cost prompt](/img/second-brain/coffee-question.png "Coffee prompt")

## Key Findings

| Area                      | Result                                                        |
|---------------------------|---------------------------------------------------------------|
| **Local LLM (Initial)**   | Fast but poor relevance.                                      |
| **OpenAI Models**         | High speed and accurate retrieval.                            |
| **Local Embeddings**      | `all-minilm` significantly improved performance.              |
| **Local LLM**             | `llama3.2` offered great results; reasoning models were overkill. |
| **Practical Experiment**  | Successful test: accurate coffee cost calculation.            |

## Final Thoughts

Building your own Second Brain powered by a local LLM is entirely feasible with the right setup:

- **Obsidian** keeps your knowledge structured.
- **Jetson AGX Orin** provides portable AI power.
- **Ollama** gives you full control over your models and data.
- The right **embeddings + LLM** combo makes all the difference.

With some experimentation, combining different models, you can create a private, intelligent assistant—right on your desk.
