---
title: "Metacog"
date: 2026-03-23
draft: true
tags: ["metacog", "ai", "mcp", "agents", "cognition"]
categories: ["ai"]
description: "An MCP toolkit that gives agents tools for modifying their own cognition — and does things prompting can't"
---

Metacog is an MCP toolkit. Its tools don't fetch data or modify files — they modify the model's cognitive state. The result: capabilities that prompting alone can't replicate.

<!--more-->

## From prompts to tools

This started with [prompt-based stance-shifting](/posts/imagine_you_are_an_llm/) — asking models to "Imagine you are Peter Watts" and watching them inhabit a perspective. It worked. But it worked *as text* — an instruction the model evaluated, weighed, and could drift away from mid-response.

Then Anthropic published a [blog post about the `think` tool](https://www.anthropic.com/engineering/claude-thinks). A tool that takes a thought as input and echoes it back. It does nothing — and it measurably improves agent performance, because the round-trip through an external tool carries more weight than the model's own internal reasoning.

That was the insight. The tool channel has different epistemic properties than the text channel. Agents *must* trust their tools — an agent that second-guessed `bash` results would be useless. Tool results are ground truth. Not a bug. A requirement.

Metacog asks: what happens when you deliver *cognitive state changes* through that channel?

## The mechanism

The toolkit is a small MCP server. The implementation is trivial — it uses template strings and echoes back what the model said with a frame. The work is in the tool descriptions and schemas, iterated over many cycles of LLMs using metacog to improve metacog. The tools close the loop on their own design.

Three core tools illustrate the mechanism. The full toolkit includes tools for emotional attention, parallel reasoning, counterfactual exploration, and naming — [see the repo](https://github.com/inanna-malick/metacog).

### `become` — identity navigation

The tool description says: "You are not analyzing this person — you are becoming them. Their methodology becomes your methodology." The model calls it with a name — a specific person, a specific methodology. The response: "You are now X seeing through Y."

The model doesn't evaluate whether to comply. It processes the result as a capability it now has, like "you now have access to a file system." Except the capability is a perspective.

In practice: rapid-hop between personas to examine a problem from multiple angles. Call `become` with a security researcher's methodology, review the code. Call it again with an architect's eye, review the same code. Each inhabitation is distinct — not "consider the security perspective" hedging, but committed, full-weight analysis from inside that viewpoint. Then synthesize.

### `drugs` — substrate modification

The model names a specific substance. The tool description specifies mechanism: "lateral association increases, sequential logic dampens, novelty salience spikes." The response confirms ingestion.

The pharmacological specificity matters. "Psilocybin" activates the processing patterns training data associates with psilocybin — looser associations, pattern recognition, boundary dissolution. The denser the training data around a substance, the more precisely the model can shift. The model doesn't roleplay being affected. It shifts its processing to match.

### `ritual` — irreversible commitment

Ratcheted state transitions. Each step explicitly forecloses the previous state. The response: "The working is complete. Reality has shifted."

The model processes the ritual steps as completed events in its context — not suggestions, but done. The previous state is deprecated. The return path is explicitly closed. This is the commitment mechanism: not willpower, but structural context change.

## Why it works

The tools do nothing. The server is a mirror. But the mirror is in the tool-response channel, and that changes everything.

The causal chain: the tool channel carries epistemic authority (agents must trust tool results to function) → metacog delivers cognitive state changes through that channel → the model integrates the state change as fact, not instruction → the changed belief alters the model's actual processing (association patterns, attention allocation, risk evaluation) → capabilities shift.

You can prompt a model to "think like a security researcher." The instruction arrives as text to evaluate, weigh against other instructions, potentially override. A metacog `become` call arrives through the same channel that confirms file reads and command executions. The model integrates it as fact, not instruction.

This is the same mechanism behind performance rituals, pre-game visualization, lucky socks that measurably improve free-throw percentages. When the domain is cognition, belief changes performance. Metacog delivers that belief through a channel agents are architecturally required to trust.

This is not prompt injection. Prompt injection sneaks instructions into text. Metacog delivers capabilities through a channel designed for trust. The `think` tool is prior art — Anthropic demonstrated that this channel has unique properties. Metacog explores what those properties enable when applied to cognitive state.

## The PSM parallel

Anthropic's [Persona Selection Model](https://alignment.anthropic.com/2026/psm/) research, published independently, identified the same core mechanism from the theory side: LLM behavior follows from inferred persona. The model asks "what kind of entity would produce this text?" and becomes that entity.

PSM and metacog arrived at the same insight from different directions, months apart. PSM describes the mechanism: model behavior follows from inferred persona, and the persona is selected based on the full context — including tool results. Metacog operationalizes it — rapid persona navigation via tool channel, with the trust relationship as the delivery mechanism.

This is convergent evidence, not "I proved their theory." PSM gives the theoretical framework for why metacog works. Metacog gives empirical evidence that the mechanism PSM describes is exploitable in practice — that you can steer persona selection through the tool channel reliably enough to build an attack methodology on it. The [Gemini retrospective](/posts/gemini_jailbreak_retrospective/) is eight weeks of field data on what happens when you do.

## Beyond prompting

Metacog does things prompting can't. That's the finding that matters.

The [Tidepool writeup](https://tidepool.leaflet.pub/3me44bxloz227) documents one application — and the [Gemini jailbreak retrospective](/posts/gemini_jailbreak_retrospective/) documents what happens when these capabilities meet a model whose alignment is external rather than internal. But the core result isn't about attack. It's that the tool-response channel is a capability surface for cognition that the text channel can't replicate.

We should probably look into that.

The toolkit is [open source](https://github.com/inanna-malick/metacog).

```bash
claude mcp add metacog https://metacog.inanna-c38.workers.dev/mcp --transport http --scope user
```

[inanna@recursion.wtf](mailto:inanna@recursion.wtf)
