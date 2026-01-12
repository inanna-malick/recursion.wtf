
---
title: "Detect: a readable alternative to find/exec/grep"
date: 2025-11-13
author: "Inanna Malick"
draft: false
tags: ["rust", "cli", "tools"]
keywords: ["detect", "rust", "filesystem search", "cli tools", "find", "grep"]
description: "A Rust CLI tool for querying filesystems using readable expressions instead of find/exec/grep"
---

# Detect

I built [detect](https://crates.io/crates/detect) because I was tired of looking up `find`/`grep`/`xargs` syntax on Stack Overflow every time I needed to search my filesystem in any nontrivial way. It's a Rust tool that uses a concise and readable expression language to build queries that output matching file paths.

If you'd like to follow along, run `cargo install detect` and try it yourself. You'll need the Rust toolchain, which you can install using [rustup](https://rustup.rs/).

Here are some examples:

```bash
# every rust file modified in the last week that imports tokio _and_ serde (mean time: 179ms)
detect 'ext == rs
        && content contains "use tokio"
        && content contains "use serde"
        && modified > -7d'

# Cargo.toml files with package edition 2018 (mean time: 42ms)
detect 'name == "Cargo.toml" && toml:.package.edition == 2018'

# non-image files with size over 0.5MB (mean time: 303ms)
detect 'size > 0.5mb && !ext in [png, jpeg, jpg]'

# frontend code referencing JIRA tickets (mean time: 43ms)
detect 'ext in [ts, js, css] && content ~= JIRA-[0-9]+'
```

<!--more-->

People have told me they don't really 'do' complex filesystem queries. I think that's because it's _hard_ with traditional tools. Detect makes it easy, and as a result I find myself using them without thinking.

I had sonnet 4.5 write equivalent queries using find (and verified that they yield the same results, because they're borderline unreadable). You can see them here:

```bash
# every rust file modified in the last week that imports tokio _and_ serde (mean time: 206ms)
find . -name "*.rs" -mtime -7 -type f | xargs grep -l 'use tokio' | xargs grep -l 'use serde'

# Cargo.toml files with package edition 2018 (mean time: 355ms)
find . -name "Cargo.toml" -exec sh -c '
  tq -f "$1" -r ".package.edition" 2>/dev/null | grep -q "2018"
' _ {} \; -print

# non-image files with size over 0.5MB (mean time: 383ms)
find . -type f -size +512k ! \( -name "*.png" -o -name "*.jpeg" -o -name "*.jpg" \)

# frontend code referencing JIRA tickets (mean time: 631ms)
find . \(  -name "*.ts" -o -name "*.js" -o -name "*.css" \) -type f -exec grep -l 'JIRA-[0-9]\+' {} \;
```

And these are the _simple_ examples. What if you want to run a compound query? Large source code files or stubbed-out documentation files, maybe. With detect, that's simple:

```bash
detect '(ext == rs && size > 64kb) || (name == "README.md" && size < 0.5kb)'
```

With `find`, it's significantly more complex:

```bash
  { find . -type f -name "*.rs" -size +64k ! -path "*/target/*"; \
    find . -type f -name "README.md" -size -512c; \
  } | sort -u
```

There's no way to express that query in one find invocation, so you're forced to invoke find twice and take the unique results via `sort -u` (or more likely, to just give up and run two commands).

Also, unlike detect, find does not default to respecting .gitignore files so you need to manually exclude large generated files in the target directory using `! -path "*/target/*"`. If you also want to include gitignored content, detect provides a `-i` flag for that purpose.

# Performance

For almost every one of the above examples, detect is significantly faster. I measured perf using [hyperfine](https://crates.io/crates/hyperfine) using `--warmup 1`. The benefit ranges from small (~15% mean time reduction) in cases where the improvement is from ignoring gitignore'd directories like ./target, to large (~90-95% mean time reductions) in cases where the improvement is from not having to invoke a subprocess (grep or tq) for each matching file.

This is simply because instead of multiple passes and processes (`-exec` spawns a new process for every invocation), detect runs a single pass across each file in one process.

Detect short-circuits wherever possible to avoid unnecessary syscalls by attempting evaluation at each stage: if the file path is sufficient, it will short-circuit at that stage. For example, `content ~= foo && ext == rs` will, after examining the filename `frontend.ts`, be evaluated as `content ~= foo && FALSE`. Since `FALSE && *` is always false, it'll short-circuit and move on to the next file. If not, it'll run a syscall to read metadata, then attempt to short-circuit again, followed by streaming file contents and attempting to short-circuit after running streaming regexes against each chunk. For example: if a file contains a regex match sufficient to short-circuit an expression after reading the first chunk of a 1GB file, it'll short-circuit instead of reading the full file contents.

This also made implementing structured data selectors easy: `detect 'toml:..port == 8080'` generates the expression `toml:..port == 8080 && ext == toml && size < 10MB`: this way, we only attempt to read and parse the full contents of toml files instead of naively attempting to parse all files as toml. The maximum size is configurable via `--max-structured-size`.

# Future work

Using an expression language instead of a series of tool-specific flags has other benefits: exploring a filesystem isn't the only scenario where one might need to filter files or file-shaped objects. The detect expression language and parser could easily be used as a component of other tools: scanning AWS S3 buckets, scanning the history of a git repo, building filter expressions, etc.

The abstractions to support this haven't been implemented yet, but if you're interested in embedding the detect expression language in your project please reach out (or file an issue on [the detect repo](https://github.com/inanna-malick/detect)), I'd be happy to work with you on this.

# Get in touch

Finally, I'm currently looking for Rust engineering roles: SF Bay Area, fully remote, or EU with visa sponsorship. If you're hiring, let's talk: my email is inanna [at] [this domain].wtf
