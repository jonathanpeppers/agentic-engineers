---
layout: post
title: "Sorting Networks: From Paper to Performance with AI"
author: "Jonathan Peppers"
excerpt: "Implementing a new CS paper algorithm in C# using GitHub Copilot, then benchmarking against .NET's built-in sort — achieving up to 41x faster performance with SIMD vectorization. 59 PRs in 3 days, from research paper to optimized library."
---

# Sorting Networks: From Paper to Performance with AI

*April 2026 — Jonathan Peppers*

<iframe width="560" height="315" src="https://www.youtube.com/embed/FoRRWCkMsFc" title="Sorting Networks: From Paper to Performance with AI" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

Can AI help a developer move faster from research paper to working, optimized code? I set out to answer that question by finding a recent CS paper, implementing its algorithm in C# with GitHub Copilot, and benchmarking it against .NET's built-in sort. The result: **up to 41x faster** for specialized sizes — built in 3 days with 59 PRs.

**GitHub repo:** [jonathanpeppers/SortingNetworks](https://github.com/jonathanpeppers/SortingNetworks)

## Why This Talk

Most developers haven't seen what AI can do today. The tooling has moved fast — it's worth showing real examples, not just hype. I have access to a lot of tokens, so let me show you what's possible with a real project, start to finish. My entire workflow has changed — I don't use an IDE anymore. This project was built entirely with AI-assisted tools.

## The Premise

1. **Find a Paper** — Discover a recent CS paper with a new algorithm (November 2025)
2. **Use AI to Implement** — Use GitHub Copilot to implement the algorithm in C# / .NET
3. **Benchmark It** — Compare against the existing Base Class Library (BCL) sort

## Prior Art: Paper → Implementation

This pattern isn't new. John Carmack did it in 1993 when building the Doom engine — he found an academic paper on Binary Space Partitioning (BSP) trees by Fuchs, Kedem & Naylor (SIGGRAPH 1980) and implemented it to subdivide game levels so the engine only rendered visible surfaces. That breakthrough made Doom's sprawling levels run in real-time on early '90s PCs.

**The pattern:** Find a Paper → Implement the Algorithm → Ship Something Faster

- **Carmack (1993):** BSP paper (1980) → Doom engine → real-time 3D on consumer PCs
- **This project (2026):** Sorting network paper (2025) → SIMD sort → up to 41x faster

## GitHub Copilot Products Used

- **Copilot in VS Code** — AI-powered code completion and chat directly in the editor
- **Copilot CLI** — AI assistant in the terminal for code generation and refactoring
- **Copilot Code Review** — AI-powered pull request reviews that catch bugs and suggest improvements
- **Copilot Coding Agent** — Autonomous agent that implements features from GitHub Issues

## The Paper

> **"Depth-13 Sorting Networks for 28 Channels"**
> Chengu Wang — [arXiv:2511.04107](https://arxiv.org/abs/2511.04107) — Published November 6, 2025

**Key Finding:** Improved depth upper bounds for 27 and 28 channel sorting networks from 14 to 13 layers, using reflectional symmetry + SAT solver.

**Timeline:**
- **Nov 2025** — Paper published
- **Apr 17, 2026** — Project started
- **Apr 18, 2026** — SIMD optimized

## How Sorting Networks Work

A sorting network is a fixed sequence of compare-and-swap operations. Unlike quicksort or mergesort, the sequence of comparisons is determined entirely by the input length — not by the data values.

Key properties:

- **Data-oblivious** — Comparisons depend on size, not values
- **Branch-free** — Eliminates input-dependent control flow
- **Parallelizable** — Independent pairs in the same layer run at once
- **Fixed depth** — Fewer layers = lower latency

Each compare-and-swap is simple:

```
if (a > b) {
    swap(a, b)
}
```

This paper: **28 channels, 13 layers deep, ~185 comparators** (previous best: 14 layers).

### A Simple Example: Sort 3 Elements

A depth-3 network with 3 comparators — the full 27-element version uses the same pattern at scale:

```csharp
// Sort 3 elements with a sorting network
static void Sort3(ref int e0, ref int e1, ref int e2)
{
    // Layer 1
    if (e0 > e1) { int t = e0; e0 = e1; e1 = t; }

    // Layer 2
    if (e1 > e2) { int t = e1; e1 = e2; e2 = t; }

    // Layer 3
    if (e0 > e1) { int t = e0; e0 = e1; e1 = t; }
}
```

**Walkthrough with `[5, 1, 3]`:**

| Step | Action | Result |
|------|--------|--------|
| Layer 1 | Compare e0, e1 → swap! | `[1, 5, 3]` |
| Layer 2 | Compare e1, e2 → swap! | `[1, 3, 5]` |
| Layer 3 | Compare e0, e1 → no swap | `[1, 3, 5]` ✓ Sorted! |

Same pattern at scale: **27 elements × 13 layers × ~185 comparators** — all code-generated.

## C# Scalar Implementation

Every compare-and-swap is unrolled into straight-line code — no loops, no branches:

```csharp
// Sort 27 elements with a depth-13 sorting network
int e0 = first;
int e1 = Unsafe.Add(ref first, 1);
// ... load e2 through e26 ...

// Layer 1 comparators:
if (e1  > e26) { int t = e1;  e1  = e26; e26 = t; }
if (e2  > e25) { int t = e2;  e2  = e25; e25 = t; }
if (e3  > e24) { int t = e3;  e3  = e24; e24 = t; }
// ... 13 layers, ~185 total comparators ...
```

Key optimizations:
- **No bounds checks** — `Unsafe.Add` skips array bounds validation
- **Type-specific `>`** — No `IComparable` virtual dispatch
- **Code-generated** — `generate_unrolled.cs` produces all the methods

## First Attempt: The Generic Approach

The first implementation (PR #1) used a generic loop with `IComparable<T>.CompareTo()` — **it was slower!**

```csharp
// Generic loop — overhead of CompareTo + virtual dispatch
private static void ApplyNetwork<T>(Span<T> span, int[] network)
    where T : IComparable<T> {
  for (int i = 0; i < network.Length; i += 2) {
    ref T a = ref Unsafe.Add(ref first, network[i]);
    if (a.CompareTo(b) > 0) { /* swap */ }
  }
}
```

**What went wrong:**
- `IComparable<T>.CompareTo()` — virtual dispatch on every comparison
- Loop + array indexing — `network[i]` indirection on every step
- Generic constraints — prevents JIT from using native `>` operator

## SIMD: Hardware Intrinsics

The real performance breakthrough came from replacing scalar `if`/`swap` with vectorized min/max/blend — processing an entire layer at once:

1. **Shuffle** — Rearrange elements to pair up comparators for this layer
2. **Min / Max** — Compare ALL pairs in the layer simultaneously
3. **Select** — Pick min or max for each position using a bitmask

```csharp
// Simplified AVX2 example for byte (all 27 elements in one Vector256)
var shuffled = Vector256.Shuffle(vec, layerPermutation);
var mins = Vector256.Min(vec, shuffled);
var maxs = Vector256.Max(vec, shuffled);
vec = Vector256.ConditionalSelect(layerMask, maxs, mins);
```

Register packing by type:
- **byte:** All 27 in one `Vector256`
- **int:** 4× `Vector256` (8 each)
- **double:** 7× `Vector256` (4 each)

## Trust, but Verify

AI can write code that *looks* fast. Only rigorous benchmarking tells you if it actually is.

```csharp
[MemoryDiagnoser]
[SimpleJob(warmupCount: 5, iterationCount: 15)]
public class IntSortingBenchmarks {
  [Params(27, 28)]
  public int Length { get; set; }

  [Benchmark(Baseline = true)]
  public void ArraySort() => ...;

  [Benchmark]
  public void NetworkSort() => ...;
}
```

BenchmarkDotNet provided:
- **Statistical rigor** — Mean, error, StdDev, ratios with confidence
- **Memory diagnostics** — `[MemoryDiagnoser]` confirms zero allocations
- **Batched ops** — 1000 ops/invoke to reduce timer overhead

Remember PR #1? The generic `IComparable` loop looked correct but was slower than `Array.Sort`. BenchmarkDotNet caught this immediately. Without measurement, we would have shipped slower code.

## CI: GitHub Actions

Every push and PR runs build, test, and benchmarks across 4 platforms — covering x86, ARM64, and all SIMD paths:

| Platform | Architecture | SIMD |
|----------|-------------|------|
| ubuntu-latest | x64 | AVX2 + AVX-512 |
| ubuntu-24.04-arm | ARM64 | AdvSimd / NEON |
| windows-latest | x64 | AVX2 |
| macos-latest | ARM64 | AdvSimd / NEON |

A `summarize` job merges all benchmark results into the `$GITHUB_STEP_SUMMARY` — viewable directly in the Actions UI. Copilot Code Review runs on every PR.

## Results

### x86 (Intel Core i9-9900K, AVX2)

27 elements, .NET 10:

| Type | Array.Sort | NetworkSort | Speedup |
|------|-----------|-------------|---------|
| sbyte | 1,452 ns | 41 ns | **35x** |
| byte | 1,301 ns | 39 ns | **33x** |
| float | 1,636 ns | 76 ns | **22x** |
| double | 1,648 ns | 93 ns | **18x** |
| short | 1,402 ns | 101 ns | **14x** |
| long | 1,404 ns | 104 ns | **14x** |
| uint | 107 ns | 54 ns | **2.0x** |
| string | 979 ns | 527 ns | **1.9x** |
| int | 101 ns | 57 ns | **1.8x** |
| ulong | 118 ns | 100 ns | **~1.2x** |
| char | 93 ns | 97 ns | **~1x** |

> `char` and `ulong`: .NET 10 already has SIMD-optimized `Array.Sort` for these types.

### ARM64 (Apple M1, AdvSimd / NEON)

27 elements, .NET 10:

| Type | Array.Sort | NetworkSort | Speedup |
|------|-----------|-------------|---------|
| byte | 1,200 ns | 29 ns | **41x** |
| sbyte | 1,230 ns | 30 ns | **41x** |
| short | 1,264 ns | 54 ns | **23x** |
| ushort | 1,254 ns | 54 ns | **23x** |
| float | 1,363 ns | 78 ns | **17x** |
| double | 1,523 ns | 111 ns | **14x** |
| long | 1,254 ns | 106 ns | **12x** |
| char | 98 ns | 50 ns | **2.0x** |
| uint | 108 ns | 75 ns | **1.4x** |
| int | 98 ns | 78 ns | **1.3x** |

**41x faster** for `byte` on Apple M1 with NEON intrinsics.

## The Journey: 59 PRs in 3 Days

### Day 1 — Foundation

- **PR #1:** Generic sorting network (slower than BCL!)
- **PR #2:** Code-gen unrolled methods
- **PR #3:** Value locals for register allocation
- **PR #5:** Int-specific, focus on 27/28 only

### Day 1 — Optimization

- **PR #8:** Specialize for all 13 primitive types
- **PR #12:** String overload with ordinal comparison
- **PR #13:** Port generator from Python to C#
- **PR #14:** AVX2 SIMD for byte/sbyte (33x!)

### Day 2+ — SIMD Everywhere

- **PR #17:** ARM64 AdvSimd/NEON
- **PR #20:** AVX-512 for 16-bit types
- **PR #35-39:** AVX2/AVX-512 for int, float, double
- **PR #58:** ARM64 TBL1 + early-exit optimizations

All code-generated by `scripts/generate_unrolled.cs` — change the generator, regenerate everything.

## Conclusions

- **AI accelerates the research-to-code pipeline** — From reading a paper to a working, optimized library in days, not weeks
- **Copilot handled the tedious parts** — Code generation scripts, test coverage, SIMD boilerplate, CI/CD setup
- **Domain knowledge still matters** — Understanding SIMD intrinsics, .NET JIT behavior, and benchmark methodology was essential
- **The results speak for themselves** — Up to 41x faster than `Array.Sort` for specialized sizes

### What's Next?

Could this go in the BCL? Sorting networks need a custom network per array length — you'd need networks for every size, not just 27/28. A C# Source Generator inside a NuGet package may be a better fit for this narrower use case.

## Links

- **Video:** [Sorting Networks: From Paper to Performance with AI](https://youtu.be/FoRRWCkMsFc)
- **Repo:** [github.com/jonathanpeppers/SortingNetworks](https://github.com/jonathanpeppers/SortingNetworks)
- **Paper:** [arXiv:2511.04107 — "Depth-13 Sorting Networks for 28 Channels"](https://arxiv.org/abs/2511.04107)
