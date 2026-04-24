---
layout: post
title: "Reviewing Massive Skia Bumps with an AI Skill"
author: "Jonathan Peppers"
excerpt: "A SkiaSharp upstream bump can be 100K–500K lines of C++, headers, generated bindings, and dependency churn — far too large for any human to read line-by-line. We built a skill that turns that wall of diff into a focused, auditable review."
---

# Reviewing Massive Skia Bumps with an AI Skill

*April 2026 — Jonathan Peppers*

[SkiaSharp](https://github.com/mono/SkiaSharp) is the .NET binding for [Skia](https://skia.org/), Google's 2D graphics library that powers Chrome, Android, Flutter, and a long list of cross-platform UI frameworks — including .NET MAUI. Every few months we pull in a new Skia milestone. That single "bump" PR routinely contains **100,000 to 500,000 lines of changes**: a fresh upstream merge, regenerated C headers, updated `DEPS`, and re-generated P/Invoke bindings on the C# side.

Reviewing that by hand is hopeless. You can scroll for an hour and still not know whether a single struct field shifted, whether a patch was silently dropped, or whether a third-party dependency moved to an unvetted commit. So we wrote an AI skill that does the mechanical work for us and produces a small, auditable report a human can actually read.

The skill lives in the SkiaSharp repo: [**mono/SkiaSharp — `.agents/skills/review-skia-update`**](https://github.com/mono/SkiaSharp/tree/main/.agents/skills/review-skia-update).

## The Problem with a Skia Bump PR

A Skia update is never one PR — it's two PRs that have to be reviewed together:

- A **mono/skia** PR that bumps the Skia submodule (C++ sources, headers, `DEPS`, upstream merge commits, our local patches)
- A companion **SkiaSharp** PR with the C# side (regenerated `*Api.generated.cs` bindings and any hand-written wrapper changes)

The questions a reviewer actually needs to answer are small and specific:

- Did the upstream merge bring in exactly the commits it claims to, and nothing else?
- Are any of our local patches missing — and if so, were they upstreamed or silently dropped?
- Did the regenerated P/Invoke bindings match what the new C headers describe, byte-for-byte?
- Did `DEPS` move any third-party dependency to a commit we haven't audited?
- On the C# side, are null handling, disposal, and ABI compatibility still correct?

Those are good questions. They're also buried under hundreds of thousands of lines of noise.

## What the Skill Does

The [`SKILL.md`](https://github.com/mono/SkiaSharp/blob/main/.agents/skills/review-skia-update/SKILL.md) is short and opinionated. It defines a four-phase pipeline:

```
Phase 1: Run orchestrator  →  Phase 2: Write summaries & build report
Phase 3: Review C# PR      →  Phase 4: Validate & persist
```

### Phase 1 — One Script Does the Mechanical Work

A single Python orchestrator handles every check that doesn't require judgment:

```bash
python3 .claude/skills/review-skia-update/scripts/run_review.py \
    --skia-pr {skia_pr_number} \
    --skiasharp-pr {skiasharp_pr_number} \
    --milestone {milestone_number}
```

It fetches PR metadata for both PRs, checks out the branches, **regenerates the P/Invoke bindings from scratch**, diffs them against what's checked in, audits `DEPS`, walks the upstream merge to figure out which commits and patches landed, and writes everything to a single `raw-results.json`.

Two rules in the skill matter a lot here:

> **NO-RETRY POLICY:** Run the orchestrator exactly ONCE. If generation reports FAIL, that is the result. Do NOT re-run to get a different outcome.

> **Never trust generated files** — The orchestrator regenerates independently.

That second one is the whole point. The agent never reads the bindings the PR author committed and asks "do these look right?" — it regenerates them itself and diffs. If they don't match, that's a real signal, not a vibe check.

### Phase 2 — Turn the Diff Into Prose

With `raw-results.json` in hand, the agent walks every added, removed, or changed item across upstream commits, interop files, `DEPS` entries, and companion-PR files, and writes a factual one-paragraph summary for each. It includes the actual diff in the JSON so reviewers can see exactly what the summary refers to.

The most interesting rule lives here: **removed upstream patches must be verified, not guessed at.** If one of our local patches no longer applies, the skill requires the agent to look at the new upstream code and determine *why* it was dropped — was it upstreamed, was it obsoleted by a refactor, or did someone just delete it? Speculation is forbidden.

### Phase 3 — Review the C# Side

The orchestrator already filtered out `*Api.generated.cs` (those were validated mechanically in Phase 1), leaving only the hand-written C# changes. The agent reviews those for null handling, disposal patterns, and ABI compatibility, then cross-links each interop change to the C# code that calls it.

### Phase 4 — Validate, Then Persist

Before the report is allowed to be saved, it has to pass a JSON-schema validator:

```bash
pwsh .claude/skills/review-skia-update/scripts/validate-skia-review.ps1 \
    {output_dir}/{pr_number}.json \
  || python3 .claude/skills/review-skia-update/scripts/validate-skia-review.py \
        {output_dir}/{pr_number}.json
```

The skill is blunt about this:

> 🛑 **PHASE GATE: You CANNOT proceed to persist without passing validation.**
> **NEVER hand-roll your own validation. NEVER assume it passes. RUN THE SCRIPT.**

Once it passes, a persist script copies the JSON into `output/ai/repos/mono-skia/ai-review/` and renders a self-contained HTML report (Bootstrap 5 + diff2html) that can be attached to the PR or shared as a gist.

The reviewer ends up with a one-screen summary like:

```
✅ Review: ai-review/{pr_number}.json

Generated Files:    PASS/FAIL
Upstream Integrity: PASS/REVIEW_REQUIRED (Na/Nr/Nc)
Interop Integrity:  PASS/REVIEW_REQUIRED (Na/Nc)
DEPS Audit:         PASS/REVIEW_REQUIRED (Na/Nc)
Companion PR:       PASS/REVIEW_REQUIRED (Na/Nc)
Risk:               HIGH/MEDIUM/LOW
```

…and a clickable HTML report that drills into every diff behind those rows.

## Why It Works

A few design choices are doing the heavy lifting:

**1. Mechanical work is mechanical.** The orchestrator is a normal Python script. The agent doesn't "decide" whether the bindings match — `diff` does. The agent only narrates and judges. This keeps the audit trail deterministic and reproducible.

**2. The schema is the contract.** Every report has to validate against [`skia-review-schema.json`](https://github.com/mono/SkiaSharp/tree/main/.agents/skills/review-skia-update). The agent can't quietly skip a section, omit a diff, or invent a field — the validator rejects it and the phase gate stops the run. That's how you get a 500K-line PR reduced to a small artifact you can actually trust.

**3. No retries, no per-file PASS.** Two rules that sound restrictive but are doing real work:

> **No retries** — Run once, report what happened.
> **No per-file PASS/FAIL** — AI provides factual summaries; all items need human review.

The skill never declares "this file is fine." It surfaces *what changed*, ranks risk, and puts a human in the loop on every item that matters. The job of the AI is to make the human's review tractable, not to replace it.

**4. The structure forces the right questions.** The four-phase split — mechanical checks → factual summaries → C# review → validate — mirrors how an experienced SkiaSharp maintainer would actually approach the PR. Encoding that workflow as a skill means even a fresh reviewer (or a fresh AI session) starts in the right place and ends with the same artifact.

## What It Feels Like in Practice

Before this skill, reviewing a Skia bump meant either rubber-stamping it (bad) or spending most of a day scrolling diffs (also bad, just slower). Now I run one command, wait a few minutes, and open an HTML report that tells me exactly:

- which upstream commits landed,
- which of our patches were dropped and why,
- whether a single P/Invoke signature drifted,
- which dependencies in `DEPS` moved and to what commit,
- and what's worth my time on the C# side.

The PR is still big. The review isn't.

If you maintain something with a similarly painful "upstream merge" PR — a vendored library, a generated SDK, a regenerated gRPC surface — this is a pattern worth stealing. The orchestrator is just a script. The skill is just a markdown file. The magic is in deciding, up front, which questions a reviewer actually needs answered, and refusing to ship a report that doesn't answer them.

## Links

- [`mono/SkiaSharp` — `review-skia-update` skill](https://github.com/mono/SkiaSharp/tree/main/.agents/skills/review-skia-update)
- [`SKILL.md`](https://github.com/mono/SkiaSharp/blob/main/.agents/skills/review-skia-update/SKILL.md) — the full pipeline definition
- [SkiaSharp](https://github.com/mono/SkiaSharp) — .NET bindings for [Skia](https://skia.org/)
