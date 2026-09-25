---
author: Sato Seinosuke
title: "From Append-Only AI Sessions to a Structured Research Workflow"
subtitle: Rethink how I use Claude Code for research and design sessions.
date: 2026-05-17T15:11:01-04:00
draft: true
layout: docs
tags:
- AI
- Claude
- workflow
---

## Motivation

My default habit was to keep appending to an existing Claude session as long as the topic felt related. The reasoning felt sound: Claude already has the context, so why throw it away?

After thinking it through, I'm convinced this is an anti-pattern — for two compounding reasons.

## Problem 1: focus degradation

The "Lost in the Middle" effect is well-documented:
[LLMs exhibit a U-shaped attention curve (Stanford, 2023)](https://cs.stanford.edu/~nfliu/papers/lost-in-the-middle.arxiv2023.pdf).
Information at the start or end of the context window is retrieved reliably; information in the middle degrades.

Real-world data from extended sessions:

| Context Usage | Observed Behaviour |
|---|---|
| ~20–40% | Losing track of earlier decisions, circular reasoning |
| 40–50% | Noticeably degraded; auto-compaction kicks in |
| ~60% | Repetition, forgetting earlier decisions, thrashing |
| 80%+ | Gets rough |

Sonnet 4.6 has two mitigations:

- **Context Awareness**: the model knows its remaining token budget and adjusts. Not a fix — knowing the window is 70% full doesn't restore attention to the middle.
- **Context Compaction (beta)**: auto-summarises older turns as limits approach. Extends session length but is lossy — subtle design constraints and the *why* behind decisions are what gets compressed away first.

For design and architecture work, this is dangerous. Early design decisions, rejected alternatives, and upfront constraints all drift toward the middle as the session grows. Claude won't say "I've lost track of that." It will answer confidently from a degraded representation.

## Problem 2: cost

Without optimisation, the entire conversation history is sent as input tokens on every round trip. Sonnet 4.6 pricing:

| Token type | Rate |
|---|---|
| Input (standard) | $3.00 / 1M tokens |
| Cache write | $3.75 / 1M tokens |
| **Cache read** | **$0.30 / 1M tokens** |
| Output | $15.00 / 1M tokens |

The 90% prompt caching discount softens the cost curve significantly. Cost alone isn't a decisive argument for fresh sessions — but it compounds with the quality problem: you're paying more for worse attention.

## The MCP fetch trap

My second habit was to treat MCP-fetched source code and documentation as a reason to continue a session. *The source is already in context, no need to re-fetch.*

This inverts the attention quality problem. Fetched source is pulled early in a session. By turn 10 or 15, it has drifted to the middle of the context — exactly where attention is worst. You avoided a re-fetch, but traded it for Claude reasoning over a poorly-attended copy of the source.

Fetched code and documentation are the worst candidates for long-term context residency:

- They're large (2,000–10,000+ tokens each)
- They're static — they never change, yet consume token budget every turn
- They drift toward the middle positionally as the session grows
- They're dense — fuzzy attention to code produces silent errors, not obvious ones

The failure mode is insidious: Claude doesn't say "I've lost track of that file." It confidently suggests a fix that contradicts a constraint in the source it already read.

Re-fetching in a fresh session isn't a cost — it's a quality gain. A fresh fetch positions the source at the end of a short context, in the highest-attention zone. Unlike human memory, LLMs have no unconscious background processing. Information doesn't consolidate between turns — it either survives in the active context or it doesn't exist.

## Long context windows are not memory systems

The marketing pitch for large context windows focuses on the ceiling: *"you can fit 1M tokens."* The honest framing is about **the floor**: the only hard guarantee a context window provides is at its boundary — things outside it are gone. Everything inside is subject to attention quality, which is a gradient, not a binary.

A larger window doesn't flatten that gradient. It stretches it. A large window also removes the forcing function of a tight one. **Sloppiness scales**.

Long context windows are a genuine advantage for one-shot processing of large static documents. The problem is specifically **multi-turn, accumulating conversations**.

The better question is never *"how much can I fit?"* but:

> **What is the minimum viable context that contains everything Claude needs and nothing it doesn't?**

This reframes the context window from a storage bucket into a workbench.

## The workflow pattern

### Design principles

1. **Sessions are topics, not projects.** One focused research question per session.
2. **Externalise knowledge immediately.** Claude's working memory is unreliable over long sessions; a well-written spec file is permanent and precise.
3. **Spec quality comes from your review, not Claude's attention.** Read and edit the spec yourself before treating it as canonical.
4. **Re-fetch rather than rely on buried context.** The marginal cost of an MCP call is negligible compared to the quality of a fresh, high-attention read.
5. **Session boundaries are structural decisions, not judgement calls.** Define criteria upfront and apply them mechanically.

### The workflow

---

#### Phase 0 — session setup

**0a.** Create a session manifest file (see details below). List only the prior spec files directly relevant to today's research question in the `load:` front matter. If unsure whether a spec is relevant, it probably isn't.

**0b.** Open a fresh session. Load the session manifest file (details below).

---

#### Phase 1 — focused fetch and initial analysis

**1.** State your narrowly scoped research question. One topic per session.

**2.** Claude fetches source/doc via MCP.

**3.** Claude answers with initial analysis. Read it fully before replying.

---

#### Phase 2 — back-and-forth until genuine understanding

**4.** Go back and forth with Claude until you genuinely understand the *why* and each proposed step. No skipping.

---

#### Phase 3 — spec writing and human review

**5.** Ask Claude to write the spec/note file.

**6.** Read the spec fully. Fix anything wrong or incomplete yourself, then ask Claude to incorporate your edits.

---

#### Phase 4 — session boundary

**7.** Ask Claude: *"Does anything in this session's findings contradict or modify the specs you loaded at the start? If so, update them now."*

**8.** Once Phase 3 is complete, `/clear` unconditionally. If the urge to continue without clearing feels strong, treat that as a signal Phase 2 is genuinely not done — return there instead.

**9.** After `/clear`: go back to Phase 0 — write a new session manifest with the spec just written and any other relevant prior specs in `load:`.

---

#### Phase 5 — implementation (separate session)

**10.** Only begin implementation after all relevant specs are written, reviewed, and consistent.

**11.** Fresh session. Write a session manifest with only the specs the specific task directly touches in `load:`.

**12.** Treat specs as ground truth. If implementation reveals something a spec got wrong, fix the spec first.

## The session manifest file

To make loading the right files a deliberate, low-friction act, use a **session manifest format** as a single entry point for each Claude Code CLI session.

### Format

```markdown
---
load:
  - specs/A1.md
  - specs/B2.md
  - specs/C3.md
goal: "Understand how B2's session storage integrates with A1's auth token lifecycle"
---

## Session notes

Prior session conclusions, open questions, constraints, what not to re-cover.
This file is committed to git — write it as a future-you artifact.
```

**`load:`** — imperative verb, unambiguous instruction. Paths relative to CWD, not to the session file itself.

**`goal:`** — frames Claude's attention before it reads anything.

### Repo structure

Git track your spec files and session files.

```
project-docs/
├── specs/
│   ├── A1.md
│   ├── B2.md
│   └── C3.md
└── sessions/
    ├── session-1.md
    └── session-2.md
```

## Make it a skill

This is better implemented as a Claude Code skill — a slash command that handles both scaffolding new session manifests and loading existing ones. Feed the URL of this blog post to Claude and ask it to generate the skill. For an example, see [this skill definition](https://github.com/satoseino/substance/blob/main/.claude/skills/satoseino-session-file/SKILL.md).

## Key takeaways

- **Append-only sessions are not free.** Every turn that pushes fetched source toward the middle is a quality regression, not a continuity gain.
- **Externalise ruthlessly.** A spec file with your review applied is more reliable than any amount of context window.
- **Session boundaries are structural decisions.** Treat them like commit boundaries — deliberate, frequent, and rules-based.
- **Re-fetching is almost always worth it.** The cost of an MCP call is negligible; the quality of a freshly positioned source in a short context is not.
