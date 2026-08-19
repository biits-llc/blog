---
layout: post
title: 'Streaming MoE Experts from SSD: Running Gemma 4 26B in 2 GB RAM'
date: 2026-08-18 13:35:47 -0400
description: Discover how TurboFieldfare uses Swift 6.2 and Metal 4 to stream Gemma
  4 26B experts from SSD, enabling 26B model execution in 2 GB of RAM on Apple Silicon.
categories:
- AI/ML
- UI Engineering
tags:
- macOS
- metal
- swift
- moe
- llm
- turbofieldfare
author: BIITS LLC
---

*Published August 18, 2026 at 1:35 PM ET*

Local large language model execution has long been constrained by available unified memory. When a model requires 14 GB on disk, traditional runtime engines expect to allocate at least that much RAM to hold every weight layer before evaluating a single prompt. However, [TurboFieldfare](https://github.com/drumih/turbo-fieldfare), an open-source engine built for Apple Silicon Macs, demonstrates that ultra-selective parameter streaming from SSD can fundamentally alter memory requirements. By taking advantage of the sparse architecture of Google's instruction-tuned Gemma 4 26B-A4B model, TurboFieldfare runs a 26-billion-parameter model using roughly 2 GB of system memory on an 8 GB Mac.

Rather than relying on existing frameworks, this custom runtime handles offloading on a per-token basis. It maintains a light core memory footprint in RAM and streams missing parameters directly from storage during every token generation pass.

## Rethinking Memory Budgets for Mixture-of-Experts Models

Mixture-of-Experts (MoE) architectures differ fundamentally from dense Transformer models. In a dense network, every parameter participates in evaluating every single token. In a sparse MoE model like Gemma 4 26B-A4B, total parameter count does not dictate per-token compute demands. Gemma 4 contains 26 billion total parameters, yet its routing layers direct each token to a fraction of the available sub-networks, requiring only about 3.88 billion active parameters per generation step.

Conventional local inference engines like llama.cpp or MLX treat model files as monolithic allocations. They load all expert layers into system memory during initialization. On machines with limited RAM, such as an entry-level 8 GB MacBook Air, loading a 14 GB model is impossible without severe OS-level memory swapping that stalls the system.

TurboFieldfare solves this by setting a strict ~2 GB memory budget. The runtime retains only two primary components inside system RAM:
1. A shared 1.35 GB model core containing base layer weights and routing logic.
2. An FP16 key-value (KV) cache configured for up to 4,000 context tokens.

The large, conditionally activated expert weights remain on disk. During generation, as execution reaches a routed expert layer, the system reads only the dynamic parameters necessary for those active experts from storage, processes the layer on the GPU, and discards or overwrites those buffers for the next step.

## Quantization, Disk Footprint, and SSD Bandwidth

For per-token disk streaming to function effectively, storage layout and quantization must minimize file I/O overhead. TurboFieldfare uses MLX affine 4-bit quantization with a group size of 64 for both shared and routed experts. The router layers themselves use an 8-bit quantization scheme to maintain routing accuracy without adding substantial weight to the permanent RAM core.

This quantization scheme brings the total installed text-only model size down to approximately 14.3 GB on disk. 

Modern Apple Silicon hardware equips Mac computers with fast NVMe drives capable of multi-gigabyte-per-second sequential and random reads. Because only ~3.88B active parameters are evaluated per token out of the 26B total, the runtime reads only a fraction of the 14.3 GB archive during any single decoding pass. Instead of requiring unified memory to store inactive expert sub-networks, the runtime converts the SSD into a secondary tier of high-bandwidth weight storage.

## Bespoke Swift 6.2 and Metal 4 Architecture

A common approach in open-source AI utilities is to write Python or Swift bindings around C++ backends. TurboFieldfare diverges from this pattern by functioning as a bespoke native application written directly in Swift 6.2 and Metal 4 for macOS 26.

Building the engine natively for Apple Silicon allows fine-grained control over GPU memory buffers and asynchronous disk reads. The codebase provides a complete system package: a native foreground Mac app, a command-line interface, a streaming model installer, and a sibling `decode-service` executable.

The architecture bypasses generic execution graphs in favor of custom Metal compute kernels tuned specifically for Gemma 4 26B-A4B tensor shapes. By pairing Swift 6.2 concurrency with Metal 4 memory pipelines, the system schedules file reads and GPU kernel dispatches concurrently. The background decode service streams expert buffers into mapped Metal memory without causing frame drops or UI freezing in the main Mac desktop interface.

The project is backed by a structured experimental record consisting of 103 measured benchmark runs. These trials systematically evaluated Metal compute kernel performance, prefill times, decode speeds, KV cache memory footprint, and storage bus throughput under varying system loads.

## Measured Performance Across Apple Silicon Generations

The performance output of TurboFieldfare highlights how SSD bandwidth scales across different generations of Apple Silicon hardware.

On an 8 GB M2 MacBook Air, TurboFieldfare delivers decode speeds between 5.1 and 6.3 tokens per second. On hardware where loading a 26B model into unified memory would normally trigger instant out-of-memory errors, achieving over 5 tokens per second offers practical usability for local interactions and background text generation.

On higher-tier hardware with wider storage buses, throughput increases substantially. When tested on a 24 GB M5 Pro Mac, the engine achieves decode speeds ranging from 31 to 35 tokens per second. The increased read performance of the M5 Pro storage controller, combined with higher GPU core counts, allows Metal compute kernels to operate near peak efficiency while streaming weight data continuously.

## Trade-Offs in Ultra-Selective Expert Offloading

Streaming weights from disk introduces explicit system trade-offs that engineers must consider:

- **Prefill Latency:** Processing long initial prompt contexts (prefill) requires evaluating prompt tokens across experts repeatedly. Because prefill involves large batches of tokens that may activate a wider spread of expert sub-networks, initial prompt processing incurs higher latency compared to fully in-memory execution.
- **Drive Write & Read cycles:** Continuous parameter streaming relies heavily on read operations across the NVMe drive. While read operations do not wear out flash storage cells like writes do, sustained throughput keeps the storage controller active, impacting thermal headroom and battery consumption on portable devices.
- **Model Specificity:** Because TurboFieldfare is optimized specifically around the layer counts, expert routing layouts, and tensor dimensions of Gemma 4 26B-A4B, it functions as a highly tuned engine for a single model family rather than a general-purpose runner like llama.cpp.

## Implications for UI Engineering and Edge AI Runtimes

The design of TurboFieldfare points toward a shift in how software engineers can think about local AI deployment. As open-weights models increasingly shift toward sparse MoE designs, total parameter count will continue to grow while per-token active parameters remain small.

By designing runtimes around native OS storage APIs, Metal compute kernels, and selective weight fetching, developers can execute large-scale models on standard consumer hardware. TurboFieldfare proves that a 26B parameter model does not inherently require 32 GB or 64 GB of unified RAM. With explicit streaming design, an 8 GB Mac can become a viable target for sparse, frontier-class language models.

## Further reading

- [https://github.com/drumih/turbo-fieldfare](https://github.com/drumih/turbo-fieldfare)
- [https://www.docker.com/products/docker-sandboxes/](https://www.docker.com/products/docker-sandboxes/)
- [https://runtimewire.com/article/jack-dorsey-block-buzz-team-chat-ai-agents-git](https://runtimewire.com/article/jack-dorsey-block-buzz-team-chat-ai-agents-git)
- [https://scalex.dev/blog/ai-agent-permissions-stats/](https://scalex.dev/blog/ai-agent-permissions-stats/)
- [https://github.com/palmier-io/palmier-pro](https://github.com/palmier-io/palmier-pro)
- [https://www.codewithbullet.com](https://www.codewithbullet.com)
- [https://speko.ai/](https://speko.ai/)
- [https://arxiv.org/abs/2608.02412](https://arxiv.org/abs/2608.02412)

