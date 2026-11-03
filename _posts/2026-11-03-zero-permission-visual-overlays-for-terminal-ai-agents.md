---
layout: post
title: Zero-Permission Visual Overlays for Terminal AI Agents
date: 2026-11-03 17:25:47 -0500
description: How click-through spatial cues like big-arrow-on-the-screen solve human-in-the-loop
  attention bottlenecks in macOS terminal agent workflows.
categories:
- UI Engineering
- AI/ML
tags:
- macos
- ai agents
- swift
- ui engineering
- hitl
author: BIITS LLC
---

*Published November 3, 2026 at 5:25 PM ET*

Terminal-bound AI agents like Claude Code and Codex can inspect complex codebases, execute refactoring tasks, and run multi-step terminal commands autonomously. Yet, these CLI workers frequently stall when faced with desktop interaction barriers. When an agent triggers an operating system permission prompt or requires a manual confirmation click in a native GUI window, it typically prints a plain text instruction like "please click Allow in the dialog" into an active shell. If your terminal window is unfocused or buried beneath overlapping IDEs, that prompt sits unnoticed. The agent stalls waiting for input, and your workflow grinds to a halt. The interaction gap between terminal-bound execution and graphical user interfaces creates a persistent Human-in-the-Loop (HITL) bottleneck.

An open-source macOS utility named [big-arrow-on-the-screen](https://github.com/franzenzenhofer/big-arrow-on-the-screen), executable as the CLI binary `bigarrow`, addresses this attention gap by turning spatial cues into an agent primitive. Rather than attempting to take over mouse cursors or send simulated input events, `bigarrow` draws self-dismissing visual annotations directly over target UI elements on screen. It paints bright arrows, colored target rings, and callout text directly across windows without taking focus away from your active application. Mouse clicks pass straight through the drawn graphic to the underlying button, allowing you to complete necessary human approvals without managing background helper daemons or adjusting window layering manually.

## Engineering Zero-Permission Window Overlays

Building floating desktop UI utilities on macOS normally involves elevated system privileges. Tools that attempt to monitor UI element trees or draw interactive overlays often demand Accessibility authorization under macOS privacy settings or require Screen Recording permissions to parse display contents. Requesting these privileges creates setup friction for developers and opens potential security exposure when running agent tools locally.

The design of `bigarrow` bypasses this permission barrier entirely. Built as a single, zero-dependency Swift binary, the application operates without a background daemon, menu bar icon, telemetry suite, or user account. It achieves overlay rendering by instantiating a transparent, borderless window layer that spans all active displays and macOS Spaces using native windowing primitives. Because it does not record screen pixels or intercept system-wide user events, macOS allows the binary to paint graphical overlays across the screen without requiring Accessibility or Screen Recording grants.

Click-through behavior is fundamental to keeping the developer in control. When `bigarrow` renders an arrow or target banner over a button, it sets window interaction flags that let hardware pointer events pass directly to the application beneath it. Your keyboard focus stays locked to whichever editor or terminal window you were actively typing in. You can click the targeted UI element directly through the visual cue, after which the rendered graphic automatically cleans itself up and exits.

## Programmatic Spatial Cues as Agent Skills

Autonomous agents interact with `bigarrow` by invoking simple command-line arguments exposed as an extensible tool skill. When Claude Code or Codex encounters an OS security prompt during local execution, the agent issues a structured command specifying the application name, targeted element, and descriptive text. For example, if an agent triggers a system privilege check while running commands, it can call:

```bash
bigarrow point --element "Allow" --app "System Settings" --text "Franz, click Allow: Ghostty may control your Mac"
```

The binary translates these parameters into immediate spatial feedback. Beyond directional point commands, the tool supports target rings, custom color palettes, and positional anchors across window boundaries. When navigating permission sub-menus, an agent can guide multi-step interactions through sequential commands:

```bash
bigarrow start --element Terminal_Toggle --app "System Settings" \
 --text "Franz, switch this on: Terminal may control your Mac" \
 --from right --color green --close-button

bigarrow start --element Add --role button --app "System Settings" \
 --text "Not in the list? Plus. Then find it." \
 --from bottom-right --style ring --color orange
```

Because commands run statelessly, agents can issue spatial cues during automated shell scripts without maintaining persistent socket connections. The visual primitives serve strictly to direct human gaze and intent. This operational model contrasts sharply with fully autonomous sandboxes like NVIDIA's [OpenShell](https://github.com/NVIDIA/OpenShell), which enforces strict kernel-level system call policies, or desktop wrappers like [Hob](https://github.com/FrigadeHQ/hob), which package local models into self-contained graphical applications. `bigarrow` operates purely at the visual boundary between existing terminal agents and native host operating systems.

## Spatial Positioning Tradeoffs and Limitations

While zero-permission overlays remove authorization friction, they introduce design tradeoffs regarding layout accuracy. Because `bigarrow` intentionally avoids requesting Accessibility permissions, it cannot query the macOS Accessibility API (`AXUIElement`) to discover the exact pixel coordinates of a target button inside another process. Instead, it relies on application-level target heuristics, relative layout hints, or positional anchors supplied by the calling process.

This architectural choice creates clear operational failure modes:

* Dynamic UI Layout Shifts: If a target desktop application resizes, restructures its layout, or opens on an unexpected monitor offset, an estimated coordinate might render an arrow pointing to empty space or an incorrect button.
* Multi-Display Boundary Conflicts: While `bigarrow` renders across multiple displays and Spaces, spatial coordinate calculations across mixed-DPI display setups require explicit spatial positioning flags (`--from right`, `--from bottom-right`) to prevent visual clipping.
* Overuse and Screen Clutter: If an agent repeatedly paints large visual arrows during high-frequency command execution, visual distraction quickly replaces helpful guidance.

These constraints highlight the boundary of passive spatial cues. The tool works best for infrequent, high-friction human intervention points, such as granting OAuth permissions, approving hardware access, or confirming system installation prompts. Evaluating agent efficiency involves measuring not just solve rates, but also how clean and minimal agent modifications remain. Code modification benchmarks like [Parsimony](https://parsimony.lasu.dev/) measure net syntax units added by AI agents to penalize unnecessary code bloat. A similar minimalism principle applies to GUI intervention tools, where less visual intrusion yields higher developer focus.

## Spatial Attention Patterns in Agent UX

As autonomous CLI tools take on longer execution paths, bridging the gap between headless terminal processes and native user interfaces becomes a core UX challenge. The zero-permission overlay pattern demonstrated by `bigarrow` proves that effective agent-human handoffs do not require complex background daemons or invasive accessibility hooks. Simple, stateless visual cues generated directly on top of native OS windows provide sufficient spatial context for developers to intervene without breaking their operational flow.

Future agent runtimes will likely need standardized layout primitives that resolve spatial coordinates more reliably across multi-monitor setups without sacrificing low-permission security models. How future iterations of macOS handle cross-window drawing boundaries without tighter privilege gating remains an open question for utility developers building lightweight HITL agent tools.

## Further reading

- [https://github.com/franzenzenhofer/big-arrow-on-the-screen](https://github.com/franzenzenhofer/big-arrow-on-the-screen)
- [https://github.com/NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell)
- [https://labs.ycombinator.com/projects/games/index.html](https://labs.ycombinator.com/projects/games/index.html)
- [https://machinelearning.apple.com/research/mlx-distributed-inference](https://machinelearning.apple.com/research/mlx-distributed-inference)
- [https://github.com/FrigadeHQ/hob](https://github.com/FrigadeHQ/hob)
- [https://aws.amazon.com/blogs/aws/announcing-aws-lambda-web-functions-for-fast-consistent-web-app-performance-anywhere/](https://aws.amazon.com/blogs/aws/announcing-aws-lambda-web-functions-for-fast-consistent-web-app-performance-anywhere/)
- [https://github.com/omnara-ai/omnara](https://github.com/omnara-ai/omnara)
- [https://parsimony.lasu.dev/](https://parsimony.lasu.dev/)

