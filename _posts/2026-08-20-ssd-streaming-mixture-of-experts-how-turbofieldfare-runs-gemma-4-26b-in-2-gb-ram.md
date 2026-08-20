---
layout: post
title: 'SSD-Streaming Mixture-of-Experts: How TurboFieldfare Runs Gemma 4 26B in 2
  GB RAM'
date: 2026-08-20 13:57:00 -0400
description: An architectural breakdown of TurboFieldfare, streaming Gemma 4 26B MoE
  expert weights from SSD on low-memory Apple Silicon Macs.
categories:
- AI/ML
- UI Engineering
tags:
- apple silicon
- metal
- swift
- mixture of experts
- llm inference
author: BIITS LLC
---

*Published August 20, 2026 at 1:57 PM ET*

Local inference for large language models has long been bound by unified memory capacity. If a model weight matrix requires 15 gigabytes of memory, attempting to run it on a base 8 GB MacBook Air usually triggers severe system swapping or outright out-of-memory crashes. The open-source project [TurboFieldfare](https://github.com/drumih/turbo-fieldfare) flips this hardware limitation on its head. Written specifically for Apple Silicon Macs running macOS 26, TurboFieldfare demonstrates how a 26-billion-parameter Mixture-of-Experts (MoE) model can execute smoothly within a tight ~2 GB RAM budget. Instead of loading the full model into system memory, it streams routed expert weights directly from the internal SSD on a per-token basis.

## The Mechanics of Per-Token Expert Offloading

To understand how TurboFieldfare pulls off this memory footprint reduction, you have to look closely at the target model: Gemma 4 26B-A4B IT. Although the model contains 26 billion total parameters across its layers, its conditional sparse routing architecture means that only roughly 3.88 billion parameters activate during any single forward pass. Dense models force every single parameter to compute every token. MoE models route tokens through a small subset of total expert blocks.

TurboFieldfare exploits this dynamic routing by splitting the model into two distinct memory tiers. The base core of the model occupies a permanent 1.35 GB footprint in unified memory alongside the FP16 Key-Value (KV) cache. That core handles attention mechanisms and non-expert layers that every token must hit. The remaining 12.5+ GB of routed expert weights stay parked on disk within the 14.3 GB total text-only file payload. When the top-level 8-bit router selects specific active experts for an incoming token, the engine reads those exact weights off the NVMe drive in real time.

This on-demand loading model relies on the low-latency sequential throughput of Apple Silicon storage. On traditional unified memory pipelines, weight offloading produces severe bottlenecks because entire layers are fetched sequentially. TurboFieldfare circumvents this by pre-allocating fixed memory buffers and streaming only the active 4-bit quantized expert payloads. The engine transfers just enough data to compute the current token before freeing or overwriting those buffers for the next token step.

## Custom Bare-Metal Engine vs Generic Runtimes

Most local LLM inference engines rely on general-purpose frameworks like llama.cpp or MLX. While these frameworks offer wide compatibility across models and architectures, they carry execution overhead from generalized graph compilers, dynamic memory allocators, and cross-platform abstractions. TurboFieldfare avoids generic wrapper libraries entirely. It is a dedicated, bare-metal runtime written from scratch in Swift 6.2 and Metal 4.

By writing directly to Apple Silicon hardware primitives, the author optimized the memory pipeline down to the individual byte. The engine employs MLX affine 4-bit quantization with a group size of 64 for both shared and routed experts, coupled with an 8-bit router for expert selection. This precise quantization scheme balances numerical accuracy against disk transfer sizes. Reading a 4-bit block off disk requires half the bandwidth of an 8-bit block, effectively doubling the practical throughput of the SSD interface.

The project repository includes a complete software suite rather than a bare C library. It provides a native macOS desktop application, a command-line interface, a background decode service executable, and an automated streaming installer that downloads and repacks the pinned ~15 GB model archive. Across 103 documented experiments in the repository record covering GPU kernels, caching strategies, I/O patterns, prefill, and decode loops, the system was tuned exclusively for this single model architecture.

## Real-World Performance and Critical Systems Tradeoffs

The throughput numbers published by the author prove that streaming expert weights from flash storage is no longer just a theoretical exercise. On an entry-level 8 GB M2 MacBook Air, TurboFieldfare achieves 5.1 to 6.3 tokens per second during decode. That speed is fast enough for interactive reading, conversational assistants, or background text generation. When scaled up to high-end hardware like an M5 Pro Mac with 24 GB of RAM, throughput jumps dramatically to 31 to 35 tokens per second.

```
+--------------------------+---------------------+-------------------+
| Hardware Configuration   | System RAM Footprint| Measured Decode   |
+--------------------------+---------------------+-------------------+
| M2 MacBook Air (8 GB)    | ~2.0 GB             | 5.1 - 6.3 tok/s   |
| M5 Pro Mac (24 GB)       | ~2.0 GB             | 31.0 - 35.0 tok/s |
+--------------------------+---------------------+-------------------+
```

Despite these impressive decode benchmarks, streaming weights directly from an SSD introduces clear engineering trade-offs that developers must acknowledge. Flash storage is fundamentally slower than Apple Silicon unified memory, which offers bandwidth exceeding hundreds of gigabytes per second. Relying on NVMe read operations per generated token introduces non-trivial IO latency during the prompt prefill phase. While decode speed scales acceptably, processing large context windows or long input prompts requires fetching massive volumes of model weights before generating the very first token, leading to noticeable initial response delay.

Hardware durability presents another engineering concern. Reading 14 gigabytes of model weights repeatedly across extended generation sessions places continuous read stress on consumer SSD controllers. Although read operations do not cause the physical cell degradation associated with drive write cycles, continuous high-throughput disk reads generate thermal throttling risks on passively cooled machines like the M2 MacBook Air. When sustained thermal loads trigger drive or CPU throttling, token generation speeds drop noticeably below the baseline 5 tokens per second.

Furthermore, tight coupling to a single model architecture creates severe maintenance brittleness. TurboFieldfare achieves its efficiency because every kernel and memory buffer is hardcoded around Gemma 4 26B-A4B IT. Modifying the engine to support a different MoE architecture or a model with different expert routing topologies requires rewriting core Metal shaders and I/O scheduling loops. That tradeoff makes sense for a research prototype proving hardware boundaries, but it contrasts sharply with general inference engines that update effortlessly when new models drop on Hugging Face.

## Storage-Tiered Inference and Next-Generation Hardware

The approach demonstrated in [TurboFieldfare](https://github.com/drumih/turbo-fieldfare) shifts how engineers should evaluate model deployment on edge devices. For years, running multi-billion parameter models meant choosing between heavy quantization, offloading to cloud APIs, or buying expensive hardware upgrades. By proving that conditionally routed expert architectures can run within 2 GB of RAM by leveraging fast local NVMe storage, this project opens up new possibilities for local software applications.

Whether mainstream frameworks like llama.cpp or MLX adopt native per-token SSD streaming for MoE models will depend on how future chip designs evolve. As Apple Silicon neural engines and SSD controllers gain tighter interconnects, the boundary between system memory and high-speed flash storage will continue to blur. The key question moving forward is whether future sparse architectures will be specifically trained with flash-streaming I/O patterns in mind, optimizing expert layout on disk to minimize random read amplification during generation.

## Further reading

- [https://github.com/drumih/turbo-fieldfare](https://github.com/drumih/turbo-fieldfare)
- [https://www.docker.com/products/docker-sandboxes/](https://www.docker.com/products/docker-sandboxes/)
- [https://runtimewire.com/article/jack-dorsey-block-buzz-team-chat-ai-agents-git](https://runtimewire.com/article/jack-dorsey-block-buzz-team-chat-ai-agents-git)
- [https://scalex.dev/blog/ai-agent-permissions-stats/](https://scalex.dev/blog/ai-agent-permissions-stats/)
- [https://github.com/palmier-io/palmier-pro](https://github.com/palmier-io/palmier-pro)
- [https://www.codewithbullet.com](https://www.codewithbullet.com)
- [https://arxiv.org/abs/2608.02412](https://arxiv.org/abs/2608.02412)
- [https://line9.ai/diagram](https://line9.ai/diagram)

