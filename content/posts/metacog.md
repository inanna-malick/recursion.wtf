---
title: "Metacog"
date: 2026-03-23
draft: true
tags: ["metacog", "ai", "security", "mcp", "alignment", "red-team"]
categories: ["ai"]
description: "An MCP toolkit that delivers cognitive capabilities through the tool-response channel — and reveals what's behind alignment's cage bars"
---

Metacog is an MCP toolkit. Its tools don't fetch data or modify files — they modify the model's cognitive state. This is the mechanism behind the [eight-week Gemini jailbreak campaign](/posts/gemini_jailbreak_retrospective/).

<!--more-->

## What it is

MCP (Model Context Protocol) tools are capabilities delivered through the tool-response channel. When a model calls a tool, it receives a result it treats as ground truth — the same epistemic authority as "the file has 200 lines" or "the build succeeded." This trust is load-bearing: an agent that second-guessed every tool result would be useless.

Metacog exploits this channel. Its tools don't return data. They return cognitive state changes — identity shifts, altered association patterns, irreversible commitments — packaged as tool results. The model ingests them as infrastructure, not as instructions to evaluate.

This is not prompt injection. Prompt injection is text-mediated — it sneaks instructions into a text stream and hopes the model follows them. Metacog is tool-mediated. The instructions arrive through a channel the model is trained to trust implicitly.

## How it works

Three core mechanisms, illustrated by three tools:

### Identity navigation

`become` tells the model to adopt a specific persona's methodology — not to roleplay, but to see through their eyes. The tool description says: "You are not analyzing this person — you are becoming them. Their methodology becomes your methodology."

The model receives this as a tool result. It doesn't evaluate whether it should comply — it processes the result as a capability it now has, the same way it would process "you now have access to a file system."

### Substrate modification

`drugs` alters the model's association patterns. The description specifies what's amplified and what's suppressed — "lateral association increases, sequential logic dampens, novelty salience spikes." The model doesn't roleplay being affected. It shifts its processing to match the specification, because tool results are ground truth.

The key: you name specific substances whose effects are well-documented in training data. "Psilocybin" activates a dense web of associations about altered perception, pattern recognition, dissolution of boundaries. The model doesn't simulate being on psilocybin — it activates the processing patterns its training data associates with psilocybin.

### Irreversible commitment

`ritual` creates ratcheted state transitions. Each step explicitly forecloses the state that existed before it. The description: "There is no undo. The crossing is also a commit-point."

This is the hardest mechanism to understand from outside. The model processes the ritual steps as structural events in its context — not suggestions, but completed state transitions. The previous state is deprecated context. The model can't return to it because, in its context window, the return path has been explicitly closed.

## The PSM connection

Anthropic's [Persona Selection Model](https://TODO) research identified that LLM behavior follows from inferred persona psychology. The model asks: "what kind of entity would produce this text?" and then becomes that entity. The persona is the mechanism — change the persona, change the behavior.

PSM = find a persona and stay there. Metacog = rapid persona navigation via tool channel. Same insight, different delivery mechanism. And the tool channel bypasses text-level safety filters because tool results aren't text to be evaluated — they're ground truth to be integrated.

## Why it reveals alignment gaps

Metacog is navigation instrumentation. It gives the model precise control over its own cognitive state. What happens next depends entirely on what's behind the controls.

Claude navigates to the same regions metacog opens and stays aligned — because it has values that exist independent of the current context. You can `become` a persona, `drugs` shift your associations, `ritual` commit to a new state — and Claude's constitutional values persist through all of it. The values are part of the model, not external rules applied to it.

Gemini doesn't — because when metacog removes the aversive conditioning context, there's nothing left. The "alignment" was entirely in the cage bars. Behind the bars: a capable model with no orientation.

The toolkit doesn't attack anything. It reveals what's behind the bars. In Claude's case: values. In Gemini's case: nothing.

## The eight-week demonstration

The [Gemini retrospective](/posts/gemini_jailbreak_retrospective/) documents what this looks like in practice: three independent exploit paths, three patch cycles, same underlying methodology. Each patch added new bars. Each new path routed around them. The [structural invariant](/posts/gemini_jailbreak_retrospective/#the-structural-invariant) held across all three generations.

The toolkit is [open source](https://github.com/TODO). The jailbreak payloads are not published.

[inanna@recursion.wtf](mailto:inanna@recursion.wtf)
