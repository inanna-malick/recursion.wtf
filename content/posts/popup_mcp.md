
## I. The Latency Problem

- There's a classic CS diagram that shows the order of magnitude differences between operations: nanoseconds for RAM access, microseconds for SSD reads, milliseconds for network round trips. LLMs are another order of magnitude slower: a single conversational turn will often take tens of seconds to complete.
- This is a problem: traditional chat and agent interfaces interleave LLM invocations between every chat message or tool invocation. Each LLM invocation has to step through the full context window including previous conversational turns, burning nontrivially expensive tokens as it does so. 
- Instead, I suggest a different model, where each LLM invocation generates a structured dialogue tree (think Disco Elysium or Baldur's Gate) (show screenshot of Unity/other gamedev tool interaction tree editor). This dialogue tree is then executed locally, with each user interaction revealing further structure without requiring one LLM invocation per step.
	- (GIF showing popup-mcp dialogue tree interaction pattern goes here)
## II. The Popup Tool
### What It Is
- That's why I created popup (link). It's an MCP tool that lets LLMs spawn freeform GUI popups, inverting the traditional control flow and allowing LLMs to 'push' questions to the user mid-turn.
- Popup supports GUI elements including sliders, checkboxes, text input windows, and dropdowns, with the ability to nest conditional branches under each to create dialogue trees that are only displayed if a specific element or set of elements is selected. This allows for nested branching multiselect/dropdown choice elements, or text box escape hatches so users can provide clarification if the LLM's missing something important.
	-  (GIF showing nested conditionals with escape hatch)
- Instead of invoking an LLM for each conversational step, this allows LLMs to build entire dialogue trees in single call. These are then rendered for the user to interact with locally, with milliseconds of latency instead of seconds.
### Why it's useful

**User Experience:**
- Sliders, dropdowns, checkboxes vs. typing responses
- Constrained inputs vs. free text parsing
- Defaults and _shape_ of questions asked are themself a mode of communication
	- (GIF)
- Only relevant branches are rendered, so user attention is conserved.

**LLM Affordances**
- LLM asks multiple rounds of questions without yielding conversational turn
- Maintains context and momentum, instead of asking a question, then writing an essay
- Enables rapid-fire interaction style vs. stop-start text exchanges
	-  (GIF showing this pattern?)

### How it's implemented

Local MCP tool: describe tool lifecycle, what MCP is, etc briefly
-  (GIF showing invocation via claude desktop, focus on JSON representation)
Remote-capable tool: discuss cloudflare MCP service bridge - janky but viable, show letta screenshot
-  (GIF showing this working via letta)
### Especially Valuable For

**MemGPT-style systems** (e.g., Letta): These systems frequently modify system prompt contents (memory blocks), preventing prompt caching. Each LLM invocation hits full latency. Popup pattern becomes critical—compress multiple rounds into one to minimize uncached calls.

## What next

This is implemented as a standalone popup tool, but if you're designing a GUI system for LLM interaction consider integrating this capability directly. Give your LLM a tool interface that allows it to display checkboxes, sliders, dropdowns, etc. as first-class interaction primitives in your GUI. Let your LLMs render dialogue trees upfront instead of falling back to the LLM at each step.

### Why aren't people already doing this?

**It breaks the "chatting with a human" facade.** Most LLM interfaces prioritize mimicking human conversation. Popup makes the tool-ness explicit. 

**This is a feature**: LLMs aren't people, and it's worth remembering that. What works for humans isn't necessarily what's best for LLMs.
