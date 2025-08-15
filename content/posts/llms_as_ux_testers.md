
+++
title = "Fresh Eyes as a Service: Using LLMs to Test CLI Ergonomics"
date = "2025-08-15"
author = "Inanna Malick"
tags = ["llm", "ux", "cli", "testing", "detect", "tools"]
keywords = ["llm", "ux testing", "cli tools", "developer tools", "detect", "user experience"]
showFullContent = false
description = "How to use LLMs as UX testers to improve your CLI tool"
+++


Every tool has a complexity budget. Users will only devote so much time to understanding what a tool does and how before they give up. You want to use that budget on your innovative new features, not on arbitrary or idiosyncratic syntax choices.

Here's the problem: you can't look at it with fresh eyes and analyze it from that perspective. Finding users for a new CLI tool is hard, especially if the user experience isn't polished. But to polish the user experience, you need users, you need user feedback, and not just from a small group of power users. Many tools fail to grow past this stage.

What you really need is a vast farm of test users that don't retain memories between runs. Ideally, you would be able to tweak a dial and set their cognitive capacity, press a button and have them drop their short term memory. But you can't have this, because of 'The Geneva Convention' and 'ethics'. Fine.

LLMs provide exactly this. Fresh eyes every time (just clear their context window), adjustable cognitive capacity (just switch models), infinite patience, and no feelings to hurt. They get confused easily, but that's exactly what you want. That confusion is data.

<!--more-->

I'm not the only person noticing this emergent pattern. [For example](https://bsky.app/profile/steveklabnik.com/post/3lwfijlb76s27):

> Steve Klabnik @steveklabnik.com
> claude kept hallucinating CLI arguments to my compiler that every other compiler has but i hadn't implemented yet so i was like "fine, ill bump that up the roadmap"

## Why I wrote detect

I could never remember command line flags for `find`. I was constantly searching stack overflow for useful snippets, reading man pages, etc. It just didn't stick. I prefer expression languages, so I wrote a tool that uses them instead: [detect](https://github.com/inanna-malick/detect) 

```bash
detect 'filename(detect) && executable() || filename(.rs) && contains(map_frame)'
```

instead of

```bash
{
  find . -type f -perm -u=x -name '*detect*'
  find . -type f -name "*.rs" -exec grep --files-with-matches map_frame '{}' ';'
} |
  sort -u
```

You can find implementation details including some pretty neat tricks with recursion and multiphase expression evaluation [here](https://recursion.wtf/posts/recursion_lib_intro/). 

##  The single-user plateau

I quite happily used the tool for a year or two. In that time I rewrote the parser to use a selector/operator model: here it is detecting all files with size over 1kb _and_ names matching main with _either_ ts or js extensions:

```bash
detect '@size > 1024 && @name ~= main && (@extension == ts || @extension == js)'
```

It was handy, but the UX was clunky and I - having written it - was blind to that fact. And so it plateaued at N=1 users, with that user (me) quite happily using it for complex queries in my day to day work.

## LLMs as beta testers

While using Claude code to add features to `detect`, I discovered something: Claude is a great beta tester. I had it test the `detect` tool in different scenarios, gave it deliberately vague prompts like "look around this repo, see what you think", and just let it play. It did so, and over the course of hundreds of tool invocations it found real bugs: panics on SIGPIPE when the output of `detect` was piped to `head`, clunky regex parsing logic that required wrapping regexes in quotes that shouldn't have been necessary, missing features like kb/mb postfixes so it could write `1kb` instead of `1024`.

This gave me an idea: it wasn't just finding bugs, it was finding rough edges in the detect tool's UX. I started treating moments where it gave up and used `find` as useful signals, not just annoyances.

## LLM-driven UX improvements

Next I decided to systematize the approach. Here's how: first, I wrote a series of scenarios that required open-ended exploration of an open source repository. Then, I let a swarm of LLM agents (multiple Claude Code sessions running in parallel) loose to explore with the prompt

> run `detect --help` and run scenarios from @testplan.md

That's it. It's minimal, but that's exactly the point: there's no bias towards how to use the tool aside from the tool's own `--help` info. Also note that testplan.md doesn't say what tool to use, it just provides scenarios for which `detect` is a natural fit. Here's an example scenario:

> Situation: Code reviews are taking too long and developers report difficulty
> understanding certain parts of the system.
>
> Your Task:
> - Find modules that should be split into smaller, focused components
> - Look for files containing multiple distinct responsibilities
> - Flag overly complex files for refactoring consideration
> - Prioritize recently modified files that may have accumulated technical debt
>
> Deliverable: Refactoring candidates list with complexity assessment

After Claude finished running through the scenarios, I asked it to provide feedback, using this simple prompt:

> give me UX feedback on the detect tool. Tell me what you tried, what you expected, what worked, what didn't

Here's some feedback I received:

> ⏺ Pain Points 🔴
>
> 1. No Decimal Size Units
> ```bash
> # This fails - need decimal precision for common file sizes
> detect 'size > 0.5mb'
>
> # Forced workaround - unreadable byte calculations  
> detect 'size > 524288'
> ```
> Fix: Support decimal units like 0.5kb, 1.5mb, 2.3gb for intuitive size queries.

### What they found

Here's a highlight reel of issues they found:

- removing the `@` prefix on selectors, to go from `@size` to just `size` (this one was obvious in hindsight, but I had spent too long using it to notice the friction)
- 'in set' operators  - from `path.ext == js || path.ext == ts` to `path.ext in [js, ts]`
- glob support (it kept trying to use them and getting confused)
- parser flexibility:
	- '5Kb', '5 KB' instead of _just_ '5kb'
	- aliases like 'modified' for 'mtime', etc
	- support AND/OR/NOT in addition to '&&/||/!'
- hierarchical selectors (with alias support for shorthand)
	- path.full, path.name, path.ext, etc

### When to stop

Eventually, the UX bug reports started moving towards not-quite-feasible feature requests: 

- complex correlation between files in queries
- selectors specific to individual programming language features (eg imports)
- semantic features that would require the detect tool to embed LLM support

That's when I knew I was done.

## You should try this - here's how

Keep your prompts minimal, just the bare minimum required to use your tool + a scenario. Don't give it any hints or examples, just have it run `tool --help` and go from there. The whole point is seeing with fresh eyes, not pre-loading context window with hints. Let the LLM discover your tool, over and over again, and note how the LLM's experience changes as you improve your CLI ergonomics.

First start with Sonnet - it gets confused more easily, it'll try a bunch of things and give you good reports about the rough edges. Then switch to Opus. It'll be more creative, more likely to ask for new features, but also more likely to ask for over-complex features (correlation, context-specific selectors, etc). I used Anthropic's monthly subscription (their Max plan), and this testing strategy was surprisingly affordable: it barely made a dent in my usage limits. 
