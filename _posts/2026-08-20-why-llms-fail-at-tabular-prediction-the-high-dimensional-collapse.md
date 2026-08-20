---
layout: post
title: 'Why LLMs Fail at Tabular Prediction: The High-Dimensional Collapse'
date: 2026-08-20 14:41:51 -0400
description: Research reveals why generic LLMs collapse on high-dimensional tables,
  disproving common tokenization and CSV formatting myths.
categories:
- AI/ML
tags:
- llms
- tabular-data
- machine-learning
- transformers
- benchmarks
author: BIITS LLC
---

*Published August 20, 2026 at 2:41 PM ET*

Despite their capabilities across natural language processing and software development, generic large language models consistently struggle when applied to predictive tasks on tabular data. For years, practitioners assumed that structural friction was the primary bottleneck. Common theories blamed tokenization artifacts, poor handling of CSV structures, or an inability to process numeric noise. Recent research by Marta Garnelo and Wojciech M. Czarnecki, titled [Why Large Language Models Fail at Tabular Prediction](https://arxiv.org/abs/2608.02412), systematically tests these assumptions and disproves most of them. By evaluating a frontier LLM in a pure inference regime, the researchers isolated the actual trigger for performance degradation: input dimensionality. While classical algorithms maintain or improve their predictions as feature count rises, transformers experience a distinct breakdown in accuracy once tables expand beyond two dimensions.

## Isolate the Model: The Pure Inference Setup

To uncover why transformers fail on structured tables, Garnelo and Czarnecki eliminated external confounding variables. They evaluated a frontier LLM under a strict pure inference regime. The model received a single prompt containing both training samples and target test queries in a single generation pass. No external tools, fine-tuning, or agentic feedback loops were permitted. This setup forced the network to perform in-context learning directly from the raw tabular data present in the prompt.

By isolating the transformer inside this minimal loop, the authors evaluated five widespread hypotheses regarding tabular performance drops. The first theory suggested that LLMs fail on non-linearly separable or noisy distributions. The second blamed the flattened CSV representation for obscuring column interactions. The third targeted numeric tokenization, arguing that subword tokenizers mangle multi-digit numbers. The fourth posited that evaluating multiple test queries within a single prompt degraded generation quality. The fifth focused on feature space dimensionality.

## Falsifying the Four Standard Scapegoats

Controlled experiments systematically dismantled four of the five theories. Noise handling and non-linear boundaries turned out not to be the cause. When presented with low-dimensional noisy datasets, the model handled non-linear decisions effectively. CSV format manipulation also failed to explain the performance gap. Re-formatting the data layout or adjusting column representations did not resolve the core accuracy drops observed in benchmark tasks.

Numeric tokenization was similarly vindicated. While subword tokenizers do split multi-digit integers awkwardly, controlling for token boundaries did not alter the fundamental degradation curve. Query batch size was the fourth hypothesis disproven. Classifying one test sample versus dozens of test points in a single context window did not cause the underlying failure. The research showed that even when given pristine, single-query prompts with normalized numeric values, the transformer struggled specifically as dataset properties shifted along one particular axis.

## Dimensionality as the Decisive Failure Vector

The decisive factor emerged when the researchers altered feature dimensions across thirty-one benchmark datasets using random linear projections. The study evaluated nine different machine learning methods on these projected spaces. Across all thirty-one datasets, the frontier LLM was the only system out of nine whose predictive accuracy consistently decreased as input dimensionality grew. Every classical baseline, ranging from tree ensembles to linear models, either maintained its performance or improved as feature space expanded.

To understand the nature of this collapse, the researchers conducted a behavioral comparison against 252 distinct classical model configurations. In two-dimensional feature space, the transformer demonstrated surprising consistency with traditional local predictors. Its spatial decision boundaries matched local, distance-based classical algorithms with up to 91.6% grid agreement. When mapping two-dimensional decision surfaces, the LLM effectively acted like a nearest-neighbor or distance-weighted local estimator.

That spatial competence vanishes as soon as columns multiply. When feature dimensions increase, the transformer's decision boundary degrades into erratic outputs. To test whether this degradation was simply random noise injection, the researchers added tuned, dimension-dependent noise to classical estimators. None of the noise-corrupted classical models reproduced the specific error patterns generated by the high-dimensional LLM. The transformer does not merely become imprecise in high dimensions. Its spatial reasoning mechanism completely breaks down in a manner distinct from traditional statistical estimators.

## Engineering Implications and Tabular Architectures

These findings highlight a clear engineering boundary for applied machine learning teams. Prompting generic LLMs with raw, high-dimensional CSV files in an in-context setup is fundamentally flawed. If a dataset contains dozens or hundreds of columns, an unassisted language model will routinely yield inferior results compared to decision trees or gradient-boosted models. The standard impulse to throw larger context windows or cleaner token formatting at tabular tasks misses the mathematical bottleneck entirely.

Architecturally, this reinforces why specialized tabular foundation models and hybrid workflows remain necessary. If in-context inference is required for low-dimensional tabular tasks, pre-processing workflows should explicitly include aggressive dimensionality reduction, feature selection, or local manifold projection before assembling the prompt context. Without feature compression, high-dimensional token sequences degrade the transformer's implicit distance metrics.

The open question left by Garnelo and Czarnecki's research is the precise internal mechanism driving this dimensional sensitivity. We know that transformer attention layers struggle to maintain relative distance representations across many orthogonal feature axes without explicit spatial inductive biases. Until future architectural revisions solve high-dimensional distance modeling inside standard self-attention, traditional tree-based algorithms and specialized tabular architectures will continue to dominate high-dimensional predictive analytics on structured data.

## Further reading

- [https://arxiv.org/abs/2608.02412](https://arxiv.org/abs/2608.02412)
- [https://github.com/drumih/turbo-fieldfare](https://github.com/drumih/turbo-fieldfare)
- [https://www.docker.com/products/docker-sandboxes/](https://www.docker.com/products/docker-sandboxes/)
- [https://scalex.dev/blog/ai-agent-permissions-stats/](https://scalex.dev/blog/ai-agent-permissions-stats/)
- [https://github.com/palmier-io/palmier-pro](https://github.com/palmier-io/palmier-pro)
- [https://www.codewithbullet.com](https://www.codewithbullet.com)
- [https://line9.ai/diagram](https://line9.ai/diagram)

