---
layout: post
title: "From Three Days to Three Minutes: Reviewing a Half-Million-Line Skia Bump"
author: "Matthew Leibowitz"
excerpt: "Bumping SkiaSharp to a new Skia release used to mean staring at a 500,000-line diff for the better part of a week. Now I open one HTML report, click around for a few minutes, and I'm done. Here's how a small AI skill changed my life."
---

# From Three Days to Three Minutes: Reviewing a Half-Million-Line Skia Bump

*April 2026 — Matthew Leibowitz ([@mattleibow](https://github.com/mattleibow))*

Let me tell you about my least favourite week of every quarter.

[SkiaSharp](https://github.com/mono/SkiaSharp) is the .NET binding for [Skia](https://skia.org/) — Google's 2D graphics engine that powers Chrome, Android, Flutter, and a long list of UI frameworks including .NET MAUI. Every few months, Google ships a new Skia milestone, and somebody (usually me) has to pull it in. That "somebody" then opens a pull request that looks something like this:

> **changed 4,217 files with 487,902 additions and 312,884 deletions**

Yes, really. Half a million lines. C++, headers, build files, vendored third-party code, regenerated P/Invoke bindings, an upstream merge with hundreds of commits, and a pile of our own carry-patches that may or may not still apply. And it's not even *one* PR — it's two: one in `mono/skia`, and a companion in `mono/SkiaSharp` for the C# side. They have to be reviewed together or not at all.

For years, my "review" looked like this:

1. Open the PR.
2. Scroll.
3. Keep scrolling.
4. Go make coffee.
5. Quietly approve and hope for the best.

That is not a review. That's a prayer.

## The Real Questions Hiding in That Diff

Here's the thing — buried under all those generated files and submodule churn, the questions a reviewer actually needs to answer are tiny:

- Did the upstream merge bring in the commits we *think* it brought in?
- Are any of our local patches missing — and if they're gone, were they upstreamed, or did somebody just delete them?
- Do the regenerated P/Invoke bindings actually match the new C headers, byte-for-byte?
- Did `DEPS` quietly move a third-party library (freetype, libjpeg-turbo, …) to a commit nobody audited?
- On the C# side, are null handling, disposal, and ABI compatibility still correct?

Five questions. Half a million lines. You do the math.

## Enter the Skill

So we built a [`review-skia-update` skill](https://github.com/mono/SkiaSharp/tree/main/.agents/skills/review-skia-update) that lives in the SkiaSharp repo. I run one command, the AI does the boring mechanical work (regenerating bindings, walking the upstream merge, diffing `DEPS`), and out the other end pops a single self-contained HTML file.

That HTML file is the whole point of this post. Let me show you.

## The Report

When the run finishes, I open the report in my browser and see this:

![SkiaSharp review report — executive summary and recommendations](https://github.com/user-attachments/assets/b2e99283-b413-4e85-a174-3637a6b14928)

In the time it takes to read one paragraph, I already know:

- **What this PR is.** *"Skia update from chrome/m133 to chrome/m147 — a 14-milestone jump."* Fourteen milestones! That used to be a "block off the calendar" kind of PR.
- **What actually changed at the architectural level.** *"SKPath becoming immutable in upstream Skia… The C# companion PR preserves backward compatibility by marking mutation methods `[Obsolete]` and delegating to a lazy SKPathBuilder internally."* That is the single most important sentence in the entire 500K-line diff, and it's right at the top.
- **What I should personally double-check.** A short, ranked list of recommendations: verify the `SkPixmap.getColor()` fork patch, check that `freetype` and `libjpeg-turbo` bumps don't introduce CVE regressions, confirm the D3D `operator==` ungate patches are still needed, test backward compatibility of the now-deprecated `SKPath` mutation methods.
- **The risk level.** A big red **HIGH** badge in the corner. No ambiguity.
- **A receipt.** PR HEAD, BASE, and upstream SHAs printed right there, so I can reproduce the analysis or argue with it later.

That's the executive summary. Then come the tabs.

## The "Upstream" Tab — Where the Magic Lives

The tabs along the bottom are the real superpower. Generated Files (0 issues — bindings regenerated and matched). Interop (26 things to look at). DEPS (6 third-party bumps). Companion PR (27 C# items). And the one I always click first: **Upstream — 29 items**.

![SkiaSharp review report — Upstream tab showing added/removed/changed patches](https://github.com/user-attachments/assets/dbb6a3c0-18c7-4f31-90b9-d4dd9e509eee)

This tab answers the question that used to keep me up at night: *what actually happened to our carry-patches?* For every patch on top of upstream Skia, the report tells me whether it was **added, removed, changed, or unchanged** — and for each one, a one-paragraph summary of *why*. Was it upstreamed? Obsoleted by a refactor? Quietly dropped? The skill is required to look at the new upstream code and *explain*; it's not allowed to shrug.

Click any item and the actual diff expands inline (it's powered by [diff2html](https://diff2html.xyz/)), so I can verify the AI's summary against the real code without leaving the page. No git checkouts. No `gh pr view`. No twelve browser tabs.

Same story for the other tabs:

- **Generated Files** — the skill *regenerates* the P/Invoke bindings from scratch and diffs them against what's in the PR. If the author's bindings don't match, that's a real signal, not a vibe check. In this PR: a clean PASS, 0 mismatches.
- **DEPS** — every third-party submodule bump, with the old SHA, the new SHA, and a note about whether it matches the upstream Chrome revision.
- **Companion PR** — only the *hand-written* C# changes, because the auto-generated `*Api.generated.cs` files were already validated mechanically.

Every section has a count badge. Every item has a diff. Nothing requires me to remember anything.

## Why This Changes Everything

I want to be clear about what the AI is and isn't doing here, because that's the part I think people get wrong.

**The AI is not approving my PR.** It never says "this file is fine, ship it." There's an explicit rule in the skill: no per-file PASS/FAIL. The job of the AI is to make my review *tractable*, not to replace it.

**The AI is not guessing.** The mechanical checks are mechanical — a Python script regenerates bindings, runs `diff`, walks the merge graph. The AI only narrates the results and ranks the risk. If a binding doesn't match, that's `diff` saying so, not a language model having a feeling.

**The AI is doing the part I was bad at.** I am a human. I cannot read 4,000 files. I cannot remember which of our 80-odd carry-patches were added in 2019 and why. I cannot eyeball whether a regenerated P/Invoke signature drifted by one parameter. The computer is *great* at all of those things, and now it does them in a few minutes instead of me doing them badly over three days.

The before-and-after is genuinely silly:

| | **Before** | **After** |
|---|---|---|
| Time to first useful review | ~2–3 days | ~3 minutes |
| Files I personally scroll through | thousands | dozens |
| "Did I miss a dropped patch?" anxiety | constant | zero |
| Reviewers willing to look at the PR | 1 (me, reluctantly) | anyone with a browser |

That last row matters more than the others. The HTML report is a single self-contained file. I can drop it in a gist, paste the link in a PR comment, and now *anybody* on the team can meaningfully review a Skia bump — not just the one person who has memorised the layout of the repo. The knowledge isn't in my head anymore; it's in the report.

## Steal This Pattern

If you maintain anything with a "huge regenerated PR" problem — a vendored library, a generated SDK surface, a regenerated gRPC or OpenAPI client, a giant codegen artifact — this pattern works. The recipe is short:

1. Write down the five questions a reviewer actually needs to answer.
2. Make a script that mechanically answers the ones a script *can* answer.
3. Have the AI narrate the results and flag the items a human still needs to look at.
4. Render it as one HTML file you can share.

That's it. The skill is a markdown file. The orchestrator is a Python script. The "magic" is just deciding, up front, what a good review looks like — and then refusing to ship a report that doesn't deliver one.

What used to be a painful, day-eating slog is now something I genuinely look forward to. Skia bumped to a new milestone? Cool. Let me grab a coffee and click through the report.

This time, the coffee is for fun.

## Links

- [`mono/SkiaSharp` — `review-skia-update` skill](https://github.com/mono/SkiaSharp/tree/main/.agents/skills/review-skia-update)
- [Sample report (HTML, open in a browser)](https://gist.githubusercontent.com/mattleibow/63a533bd398e35efab3fe7f9e4031400/raw/c5c07be9123ffdf35ff52faab86264863edb2194/184.html)
- [SkiaSharp](https://github.com/mono/SkiaSharp) — .NET bindings for [Skia](https://skia.org/)
