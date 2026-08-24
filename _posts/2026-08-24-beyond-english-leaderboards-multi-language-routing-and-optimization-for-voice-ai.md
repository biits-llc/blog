---
layout: post
title: 'Beyond English Leaderboards: Multi-Language Routing and Optimization for Voice
  AI Pipelines'
date: 2026-08-24 13:56:28 -0400
description: English-centric speech leaderboards fail in production. Learn how multi-language
  STT routing optimizes WER, latency, and cost for voice agents.
categories:
- AI/ML
- UI Engineering
tags:
- voice ai
- speech to text
- llm routing
- livekit
- pipecat
author: BIITS LLC
---

*Published August 24, 2026 at 1:56 PM ET*

Voice AI engineering teams often make a fatal mistake when picking speech models: they select whichever provider tops the latest English evaluation table and deploy it globally. That strategy breaks down the moment real users start speaking German, Arabic, or Tamil. Evaluation suites for speech recognition and voice synthesis are heavily skewed toward high-resource English audio. When you measure cross-lingual performance across real voice production pipelines, standard marketing claims disintegrate.

## The Blind Spot of Universal Speech Benchmarks

Data published by [Speko](https://speko.ai/) highlights a glaring gap in modern speech evaluation. Out of 23 evaluated speech models on their benchmarking suite, 11 are measured exclusively in English. The model sitting at the absolute top of an English transcription leaderboard might rank near the bottom or fail entirely when processed against conversational German or Norwegian.

There is no single Speech-to-Text (STT) model that dominates across all global locales. Across 9 evaluated languages (English, Arabic, French, German, Hindi, Norwegian, Spanish, Tamil, and Telugu), four separate models claim top accuracy depending entirely on the target language. Selecting a single vendor for a global deployment guarantees a degraded experience for non-English users.

## Empirical Variance: Accuracy vs. Cost in Multilingual STT

Evaluating voice processing hardware and model architectures requires weighing Word Error Rate (WER) against cost per minute. The financial spread across speech providers is massive, and paying a premium price does not reliably yield low transcription error rates.

Universal-3.5 Pro from AssemblyAI achieves a top-tier 2.0% WER at $0.0075 per minute. OpenAI's GPT-4o Transcribe registers 2.3% WER at $0.0060 per minute, while GPT-4o-mini Transcribe sits closely behind at 2.7% WER for half the cost ($0.0030 per minute). Alibaba's Qwen3-ASR achieves 2.8% WER at $0.0054 per minute. On the budget end, Modulate's Velma 2 reaches 4.4% WER at $0.0010 per minute. That offers a 7x cost reduction compared to high-end engines while preserving reasonable transcript quality for standard customer intake workflows.

```
Model                     WER     Cost / min
--------------------------------------------
Universal-3.5 Pro         2.0%    $0.0075
GPT-4o Transcribe         2.3%    $0.0060
GPT-4o-mini Transcribe    2.7%    $0.0030
Qwen3-ASR                 2.8%    $0.0054
Realtime STT-1            3.3%    $0.0025
Chirp 3                   3.9%    $0.0160
Velma 2                   4.4%    $0.0010
Grok STT                  4.8%    $0.0033
Solaria-1                 5.0%    $0.0125
Pulse                     5.1%    ~$0.0050
stt-rt-v5                 7.5%    $0.0020
Gradium ASR               8.4%    $0.0104
Nova-3                    9.8%    $0.0048
Ink-2                    11.0%    $0.0090
```

Higher prices do not guarantee better accuracy. Google's Chirp 3 costs $0.0160 per minute yet yields a 3.9% WER. Deepgram's Nova-3 posts a surprisingly poor 9.8% WER at $0.0048 per minute, while Cartesia's Ink-2 falls to an 11.0% WER at $0.0090 per minute. Soniox's stt-rt-v5 hits 7.5% WER at $0.0020 per minute, and Inworld's Realtime STT-1 delivers 3.3% WER for $0.0025 per minute. Other specialized options like xAI's Grok STT (4.8% WER at $0.0033/min), Gladia's Solaria-1 (5.0% WER at $0.0125/min), and Smallest's Pulse (5.1% WER at ~$0.0050/min) land in distinct efficiency pockets.

If your system hardcodes Deepgram Nova-3 or Cartesia Ink-2 across all regional audio streams, you are overpaying for elevated error rates in non-English contexts.

## Architectural Tradeoffs: Monolithic S2S vs. Modular Pipelines

Voice agent architectures generally split into two design approaches: monolithic Speech-to-Speech (S2S) real-time models, or modular pipelines chaining STT, LLM, and Text-to-Speech (TTS) services.

Monolithic S2S engines like OpenAI's GPT Live Transcribe ($0.0170 per minute) minimize audio processing latency by removing explicit intermediate text generation steps. The model ingests real-time audio streams directly and emits synthesized voice tokens. This structural directness comes at a high price tag and removes granular control over prompts, explicit guardrails, and middle-stage inspection.

Modular pipelines remain the practical choice for enterprise voice applications. Chaining discrete STT, LLM, and TTS services allows engineering teams to enforce safety filters, scrub sensitive data, and inject dynamic database context before calling the core language model. Crucially, a discrete pipeline enables stage-by-stage dynamic routing. An incoming call from a user speaking Spanish can route to a cost-effective Spanish STT model, pass text to an efficient reasoning LLM, and stream output through a localized TTS engine.

## Implementing Dynamic Routing in Real-Time Voice Frameworks

Hardcoding vendor SDKs and managing API credentials for dozens of providers produces unmaintainable glue code. To streamline infrastructure, teams are turning toward provider-neutral voice gateways. [Speko](https://speko.ai/) hosts a managed data plane at `relay.speko.dev` with typed contracts for STT, LLM, and TTS routing.

Instead of writing custom WebSocket glue code for every speech vendor, open-source agent frameworks like LiveKit and Pipecat plug directly into gateway abstraction layers. For Python developers using LiveKit, replacing vendor-specific code with multi-provider dynamic routing requires changing only a few object instantiations:

```python
from livekit.agents import AgentSession
from speko_gateway.livekit import LLM, STT, TTS

session = AgentSession(
    stt=STT(credential_source="auto"),
    llm=LLM(model="auto", objective="balanced"),
    tts=TTS(credential_source="auto")
)
```

Setting `model="auto"` with an explicit objective like `"balanced"` delegates provider selection to the gateway runtime. The system evaluates language metadata or early stream audio detection to select the best provider. If a caller switches from English to Hindi mid-conversation, the gateway re-routes subsequent audio frames to the optimal Hindi engine without disrupting the front-end agent session.

## Critical Realities: Proxy Latency and Benchmark Decay

Dynamic speech routing solves cross-lingual accuracy degradation, but it introduces operational trade-offs that engineers cannot ignore.

Adding an intermediate proxy gateway between client workers and downstream speech vendors adds network hops. If your LiveKit or Pipecat workers run in AWS `us-east-1` and your routing gateway resides elsewhere, every audio chunk incurs extra round-trip latency. For real-time voice applications aiming for under 500 milliseconds of total turnaround time, a 40-millisecond proxy delay represents a significant fraction of your latency budget. You must verify whether your routing setup supports direct provider connections or region-matched gateway relays.

Provider models also drift over time. Vendors frequently deploy unannounced updates to signal processing, noise suppression, and tokenization layers. An STT model that wins top accuracy for French in January might suffer regressions by spring. Routing proxies must run continuous, automated evaluations against benchmark audio datasets to ensure routing tables reflect true live performance rather than outdated static charts. Without automated real-time verification, dynamic routing risks steering traffic into broken or degraded vendor endpoints.

## Further reading

- [https://speko.ai/](https://speko.ai/)
- [https://github.com/drumih/turbo-fieldfare](https://github.com/drumih/turbo-fieldfare)
- [https://www.docker.com/products/docker-sandboxes/](https://www.docker.com/products/docker-sandboxes/)
- [https://scalex.dev/blog/ai-agent-permissions-stats/](https://scalex.dev/blog/ai-agent-permissions-stats/)
- [https://www.codewithbullet.com](https://www.codewithbullet.com)
- [https://arxiv.org/abs/2608.02412](https://arxiv.org/abs/2608.02412)
- [https://line9.ai/diagram](https://line9.ai/diagram)

