---
layout: post
title: 'Inside Hob: Architecting an Offline Desktop Agent with Electron, llama.cpp,
  and MCP'
date: 2026-11-09 19:46:58 -0500
description: An architectural analysis of how Hob combines Electron, embedded llama.cpp
  bindings, and Model Context Protocol for privacy-first desktop workspace automation.
categories:
- AI/ML
- UI Engineering
tags:
- local-ai
- electron
- mcp
- llama-cpp
- open-source
author: BIITS LLC
---

*Published November 9, 2026 at 7:46 PM ET*

Cloud-hosted AI agents dominate software automation, but they force developers and users to trade data privacy for functionality. Managed platforms like [Omnara](https://github.com/omnara-ai/omnara) provide server-side sandboxes for running agents via remote APIs, while cloud infrastructures process sensitive enterprise context on remote hardware. That trade-off breaks down for privacy-focused engineers and organizations working under strict compliance bounds. [Hob](https://github.com/FrigadeHQ/hob), an open-source desktop application created by Frigade under the MIT license, presents a different path. Designed as a local alternative to cloud chat interfaces and workspace assistants, Hob runs open-weight models like Google Gemma and gpt-oss directly on macOS, Windows, and Linux host hardware without network egress or telemetry.

Building a local agentic application requires solving two hard problems simultaneously: managing native hardware constraints without crashing the user interface, and providing useful system-level capabilities without relying on remote sandboxes. Hob addresses both through a modular monorepo that glues native C++ model inference to a desktop user experience.

## The Monorepo Layout: Logic Isolation in TypeScript

Hob organizes its code as a TypeScript monorepo using `pnpm` workspaces. Structuring a desktop application into isolated workspaces prevents the UI layer from entangling itself with low-level model lifecycle management or hardware checks.

```
hob/
├── app/              # Core Electron runtime & UI
├── catalog/          # Model registry and metadata definitions
├── packages/picker/  # Dynamic hardware evaluation utilities
```

The `app` directory holds the main Electron host process and the user interface. It renders workspace views, tracks active document contexts, and handles windowing primitives across platform targets.

Separately, the `catalog` workspace manages definitions for supported open-weight models. It isolates model identifiers, context window specs, and target quantizations away from the main application views.

The brain behind execution compatibility is `packages/picker`. When a user launches Hob, `packages/picker` probes the underlying hardware. It checks available system memory, detects host GPU hardware, and inspects CPU architecture flags. Based on these measurements, it calculates an operational budget and selects a model quantization from `catalog` that fits comfortably within the system boundaries.

Decoupling hardware profiling from UI execution keeps the codebase maintainable. The UI component does not care whether a user is running a high-end desktop or an older laptop. It asks `packages/picker` for a recommended configuration, pulls the model manifest from `catalog`, and initialises the native execution environment.

## Native Model Inference via Embedded llama-cpp

Most web-first desktop tools rely on external REST APIs or local gateway daemons like Ollama to run models. Hob takes a direct path by embedding `llama-cpp` native bindings within its desktop runtime tree.

Running inference locally creates strict hardware limits. Specialized setups like Apple's MLX benchmark large models such as Muse Spark Open across Mac Studio clusters linked over RDMA ([Apple Machine Learning Research](https://machinelearning.apple.com/research/mlx-distributed-inference)). Desktop apps don't have that luxury. A local agent must run alongside web browsers, IDEs, and office suites on standard consumer hardware. Quantization is mandatory.

Hob targets quantized variants of open-weight models, including Google Gemma and gpt-oss formats. By executing 4-bit or 8-bit quantized parameters via `llama-cpp` C++ bindings, Hob achieves decent token generation rates on standard laptop hardware. The execution engine runs completely offline. You can process documents, generate text, and execute workflows in an airplane cabin with Wi-Fi disabled. Your data stays on your local storage drive.

The local execution path introduces trade-offs. Quantized open-weight models have lower parameter counts than massive cloud endpoints. They struggle more often on complex, multi-step logical reasoning chains. However, for context retrieval, structural parsing, and targeted document processing, the privacy guarantee and zero-latency local execution offset the reduction in raw model capacity.

## Local Workflows via Model Context Protocol (MCP)

An offline LLM that only outputs text in a chat window offers limited value for workspace automation. To automate tasks, the model needs to take actions. Hob integrates Model Context Protocol (MCP) support directly into its runtime loop.

MCP provides a uniform specification for connecting AI models to tools, file systems, and application APIs. In Hob, MCP interfaces allow local open-weight models to inspect directories, read workspace files, and perform structured actions like local PDF form filling.

```
+-------------------------------------------------------+
|                      Electron App                     |
|  +-------------------+        +--------------------+  |
|  |   UI Renderer     |        |   packages/picker  |  |
|  +---------+---------+        +---------+----------+  |
|            |                            |             |
|            v                            v             |
|  +-------------------------------------------------+  |
|  |               Main Process Runtime              |  |
|  |  +--------------------+   +------------------+  |  |
|  |  | embedded llama-cpp |   |   MCP Tooling    |  |  |
|  |  +--------------------+   +------------------+  |  |
|  +-------------------------------------------------+  |
+-------------------------------------------------------+
                           |
                           v
              Local OS / File System Access
```

Because MCP execution happens on host compute, context never leaves the machine. Processing an NDA or parsing financial forms does not involve transmitting unencrypted text to cloud endpoints.

This design introduces a critical security boundary question. Cloud platforms use strict runtime isolation primitives. NVIDIA's [OpenShell](https://github.com/NVIDIA/OpenShell) isolates autonomous agents by enforcing kernel-level system call policies and network tracing. AWS Lambda Web Functions rely on Firecracker microVMs ([AWS News Blog](https://aws.amazon.com/blogs/aws/announcing-aws-lambda-web-functions-for-fast-consistent-web-app-performance-anywhere/)) to isolate untrusted user code execution. Hob does not wrap its MCP tools in virtual machines or kernel sandbox policies. Instead, it runs tools directly inside the host user space under native OS permissions.

This is a real architectural trade-off. By relying on native host user execution, Hob keeps binary installation light and avoids virtual machine performance penalties. But it places the full burden of safety on local OS user permissions and tool verification. If an LLM misinterprets an instruction and calls an MCP file deletion tool, no virtualized container catches the call before it hits the host file system.

## Performance Bottlenecks in Desktop Agent Runtimes

Combining Electron UI processes with native C++ inference threads creates real resource management challenges. Chromium processes inside Electron consume significant system memory on their own. Placing a multi-gigabyte C++ inference process inside that same operational footprint tests system limits.

If `packages/picker` miscalculates system overhead and allocates a model quantization that consumes 90 percent of system RAM, the operating system will start swapping memory to disk. When system memory swaps aggressively, UI frame rates plunge and input latencies soar. Worse, an out-of-memory error inside the `llama-cpp` thread can crash the background worker process entirely.

UI engineering in this context requires strict process isolation. Hob isolates inference workers from the primary UI render thread so that long prefill phases or heavy token generation loops do not block the event loop. The UI renderer remains responsive, allowing users to scroll, read incoming tokens, or cancel execution threads without UI freezes.

## Balancing Local Control and Security

Hob proves that combining Electron, native `llama-cpp` bindings, and MCP tools produces a capable desktop agent without cloud dependencies. Its monorepo layout separates hardware profiling, catalog metadata, and user interactions into logical layers.

The broader challenge for on-device agent engines will be refining local security controls. As MCP tool ecosystems expand to support arbitrary shell execution and deep system modifications, running agents with plain host user privileges becomes riskier. The next step for local desktop agent architectures will be adopting lightweight OS isolation primitives, like macOS app sandboxes or Linux namespace boundaries, to constrain local tool execution without adding the footprint of full hypervisors.

## Further reading

- [https://github.com/FrigadeHQ/hob](https://github.com/FrigadeHQ/hob)
- [https://github.com/NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell)
- [https://labs.ycombinator.com/projects/games/index.html](https://labs.ycombinator.com/projects/games/index.html)
- [https://arstechnica.com/ai/2026/10/ai-coding-agents-generate-more-code-but-not-more-software/](https://arstechnica.com/ai/2026/10/ai-coding-agents-generate-more-code-but-not-more-software/)
- [https://machinelearning.apple.com/research/mlx-distributed-inference](https://machinelearning.apple.com/research/mlx-distributed-inference)
- [https://www.anthropic.com/research/provable-inference](https://www.anthropic.com/research/provable-inference)
- [https://aws.amazon.com/blogs/aws/announcing-aws-lambda-web-functions-for-fast-consistent-web-app-performance-anywhere/](https://aws.amazon.com/blogs/aws/announcing-aws-lambda-web-functions-for-fast-consistent-web-app-performance-anywhere/)
- [https://github.com/omnara-ai/omnara](https://github.com/omnara-ai/omnara)

