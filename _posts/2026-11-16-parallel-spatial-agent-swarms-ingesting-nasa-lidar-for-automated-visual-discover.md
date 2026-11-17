---
layout: post
title: 'Parallel Spatial Agent Swarms: Ingesting NASA LiDAR for Automated Visual Discovery'
date: 2026-11-16 19:56:07 -0500
description: How 48 parallel AI agents scanned 1,200 square kilometers of NASA LiDAR
  tiles to locate Maya architectural anomalies under dense jungle canopy.
categories:
- AI/ML
- UI Engineering
tags:
- spatial ai
- remote sensing
- lidar
- multi-agent systems
- computer vision
author: BIITS LLC
---

*Published November 16, 2026 at 7:56 PM ET*

When raw spatial remote sensing data meets distributed agent fleets, visual processing tasks that once demanded months of manual labor collapse into a single weekend. A personal research project by software builder [Pietro Miotti](https://pietromiotti.com/latest/maya-site) demonstrated this approach by organizing 48 parallel AI agents to analyze bare-ground elevation maps generated from public NASA LiDAR data. The dataset spanned approximately 1,200 square kilometers of dense jungle in Mexico. Operating concurrently, the swarm scanned tiled elevation images for synthetic geometric shapes typical of Maya architecture, completing the initial identification of structural candidates in under 72 hours.

## Stripping the Canopy with Bare-Ground LiDAR

Optical satellite photography faces severe limitations when mapping regions covered by dense tropical rainforests. High-resolution satellite cameras capture only the top layer of vegetation, presenting an unbroken expanse of green canopy that conceals underlying terrain features. Human settlements, earthworks, and stone structures built centuries ago remain completely invisible to conventional optical sensors.

Light Detection and Ranging (LiDAR) changes this equation by firing high-frequency laser pulses from an aircraft down toward the earth. While many pulses reflect off leaves and branches, a fraction of the laser light penetrates small openings in the tree canopy to strike the terrain beneath. Sensor receivers record the return timing of these ground pulses, enabling software to separate vegetation height from actual ground surface elevation.

The resulting dataset produces a bare-ground digital elevation surface where trees and shrubs are computationally stripped away. When rendered with elevation-based color gradients or hillshade lighting, forgotten topography becomes visible. Hilltops covered in jungle suddenly reveal distinct terraces, raised causeways, rectangular enclosures, and artificial mounds.

Miotti initiated the project on October 18 while traveling in Peru. Inspired by historical field exploration methods like Hiram Bingham's 1911 expedition to Machu Picchu, he investigated whether autonomous software agents could accelerate the manual work of scanning elevation maps. Because remote sensing coverage in Peru involved high commercial survey costs and complex permitting processes, the project shifted focus toward publicly accessible NASA LiDAR data covering ancient Maya territories across Mexico.

## Orchestrating a 48-Agent Vision Swarm

Processing 1,200 square kilometers of high-resolution LiDAR imagery sequentially poses a heavy computational bottleneck for a single model or human reviewer. High-resolution raster strips must be evaluated at a scale detailed enough to spot house platforms measuring just a few meters across, without losing structural context across multi-kilometer site layouts.

To handle this scale, the pipeline sliced the NASA elevation datasets into a grid of discrete image tiles. Rather than feeding the entire geographical region into a single processing pipeline, the system deployed a swarm of 48 AI agents running in parallel.

Each agent received a systematic assignment of elevation tiles to evaluate. The prompt instructions directed the visual agents to look for synthetic geometric signatures associated with Maya urban settlement patterns. Key structural targets included:

* Step pyramids and conical mounds featuring sharp elevation changes.
* Elevated masonry platforms designed to raise residential structures above swampy ground.
* Rectangular or square house foundations arranged in clusters.
* Cleared, flattened central plazas surrounded by aligned earthworks.

Operating overnight on personal computational resources, the 48 agents processed the full 1,200 square kilometer tile set. By October 19, the swarm generated a consolidated report detailing flagged coordinates across the terrain, complete with cropped image segments showing regular geometric outlines rising above bare ground.

## The Literature Retrieval Bottleneck: Discovery vs. Rediscovery

While the parallel execution of visual agents proved effective for raw spatial anomaly detection, the experiment exposed a major weakness in agentic literature verification workflows.

Upon reviewing the flagged locations, the project initially categorized one prominent cluster of structures as an unmapped Maya site. An initial lookup against official public resources, including the register maintained by Mexico's National Institute of Anthropology and History (INAH), confirmed that no official site record existed for those specific coordinates. This prompted a direct inquiry to the Mexican government to share the findings.

A subsequent correction updated the story on November 11. The site was not entirely unknown to academic science. It had already been mapped and documented in a scholarly publication by Estrada-Belli et al. (2025). The agent pipeline had missed this study when performing automated literature retrieval.

This outcome highlights an important architectural boundary for domain-specific agent swarms. AI vision agents possess strong capabilities for local pattern recognition, rapidly isolating geometric anomalies from matrix raster data. However, secondary validation agents often struggle when cross-referencing spatial anomalies against messy, non-centralized academic literature. Academic publishing formats, paywalled PDFs, varied naming conventions, and recent preprints create significant friction for automated search tools.

Relying on agents to claim a new discovery based on a failed search query introduces high false-positive rates for novelty. An anomaly found on a LiDAR map simply means the software detected a geometric structure. Proving that no human researcher has ever documented that structure requires robust, verified indexing across specialized academic databases, historical mapping projects, and local administrative records.

## System Design Lessons for Spatial AI Engineering

The 72-hour Maya survey experiment provides clear design patterns for software engineers building spatial analysis and remote sensing pipelines.

First, tiled parallelization works exceptionally well for spatial computer vision. Dividing massive geographical raster grids into uniform tiles lets developers scale agent processing linearly across compute nodes. Because each elevation tile can be evaluated independently, adding more concurrent agents directly reduces job completion time without introducing complex shared-state synchronization issues.

Second, prompt engineering for spatial agents must account for geographic false positives. Natural geological formations often mimic synthetic structures. Erosion ridges, limestone sinkholes (cenotes), and natural hill terraces can trigger high confidence scores from vision models unless prompts explicitly instruct models to look for right-angle alignments, symmetrical bounding walls, and repeating architectural groupings.

Third, automated verification requires a dedicated multi-stage pipeline rather than a single literature search step. Future spatial research systems should separate the detection phase from the validation phase:

1. Detection Swarm: Scans bare-ground LiDAR tiles for visual geometric signatures and outputs coordinate candidate lists with spatial metadata.
2. Spatial Index Matching: Cross-references candidate coordinates against geographic spatial databases, official monument registers (such as INAH), and open-access spatial vector layers.
3. Deep Literature Retrieval: Executes targeted semantic searches over specialized academic text repositories, pulling full-text papers that mention nearby geographic features, water bodies, or historical region names.
4. Human-in-the-Loop Triage: Presents matched visual renders, coordinate overlays, and relevant literature snippets to domain experts for final verification.

Automating spatial discovery relies on more than fine-tuned vision models processing high-resolution pixels. True automated research systems require coordination between parallel image processing swarms and high-precision document retrieval infrastructure. Until spatial AI agents can reliably query unindexed academic literature alongside bare-ground elevation maps, human archaeologists will remain essential for validating what the swarm uncovers under the forest canopy.

## Further reading

- [https://pietromiotti.com/latest/maya-site](https://pietromiotti.com/latest/maya-site)
- [https://www.cbsnews.com/news/united-airlines-bans-ai-agents-booking-flights/](https://www.cbsnews.com/news/united-airlines-bans-ai-agents-booking-flights/)
- [https://github.com/NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell)
- [https://github.com/GoogleChrome/modern-web-guidance](https://github.com/GoogleChrome/modern-web-guidance)
- [https://github.com/FrigadeHQ/yogurt](https://github.com/FrigadeHQ/yogurt)
- [https://www.anthropic.com/research/provable-inference](https://www.anthropic.com/research/provable-inference)
- [https://github.com/FrigadeHQ/hob](https://github.com/FrigadeHQ/hob)
- [https://github.com/omnara-ai/omnara](https://github.com/omnara-ai/omnara)

