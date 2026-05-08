---
layout: post
title: "Sorting Networks Part 2: From 27/28 to a Source Generator for Sizes 2–64"
author: "Jonathan Peppers"
excerpt: "Taking a hardcoded SIMD sort for two sizes and turning it into a C# source generator that handles sizes 2–64, any type, with zero allocations — shipped as a NuGet package. 80 PRs total, all built with AI."
tags:
  - copilot
  - csharp
  - performance
  - source-generators
  - simd
  - nuget
---

# Sorting Networks Part 2: From 27/28 to a Source Generator for Sizes 2–64

*May 2026 — Jonathan Peppers*

In [Part 1](/2026/04/22/sorting-networks-from-paper-to-performance-with-ai.html), I took a CS paper about depth-13 sorting networks, implemented it in C# with GitHub Copilot, added SIMD vectorization, and got up to **41x faster** than `Array.Sort` — all in 3 days with 59 PRs. Part 1 ended with an open question:

> *Could this go in the BCL? Sorting networks need a custom network per array length — you'd need networks for every size, not just 27/28. A C# Source Generator inside a NuGet package may be a better fit for this narrower use case.*

So that's exactly what I did. The 27/28-only hardcoded sorter is now a **Roslyn source generator** that emits optimized sort methods for **any size from 2 to 64**, for **any type**. It's published as a NuGet package — [`SortingNetworks.SourceGen`](https://www.nuget.org/packages/SortingNetworks.SourceGen) — and the initial release is [0.1.0 on GitHub](https://github.com/jonathanpeppers/SortingNetworks/releases/tag/0.1.0).

**GitHub repo:** [jonathanpeppers/SortingNetworks](https://github.com/jonathanpeppers/SortingNetworks)

## What Changed Since Part 1

Part 1's implementation had a `scripts/generate_unrolled.cs` script that produced `.cs` files checked into the repo — a static code generator. It only worked for sizes 27 and 28, using the networks from the paper.

The new approach is a **Roslyn incremental source generator**. You add one NuGet package, decorate a `partial class` with attributes, and the generator produces optimized sort methods at compile time — no checked-in generated code, no scripts to run.

From PR #63 to PR #119, here's what was built:

- **Source generator** — Roslyn `IIncrementalGenerator` that reads `[SortingNetwork]` attributes and emits sort methods
- **Sizes 2–64** — Using Batcher's odd-even merge networks (2–22), best-known optimal networks (23–32), and [Dobbelaere's SorterHunter](https://github.com/bertdobbelaere/SorterHunter) networks (33–64)
- **All types** — Every primitive, `decimal`, `string`, custom `IComparable<T>` value types, and arbitrary types via `IComparer<T>`
- **Full SIMD coverage** — x86 AVX2, AVX-512 (including VBMI), ARM64 AdvSimd/NEON — all generated per-type, per-size
- **NuGet package** — Published as `SortingNetworks.SourceGen` with public API analyzers and proper versioning

## Usage

```
dotnet add package SortingNetworks.SourceGen
```

```csharp
using SortingNetworks;

[SortingNetwork(27, typeof(int))]
[SortingNetwork(28, typeof(int))]
[SortingNetwork(10, typeof(double))]
partial class MySorter { }

// Sort a span of exactly 27 ints — uses sorting network
Span<int> data = [27, 26, 25, /* ... */ 3, 2, 1];
MySorter.Sort(data);

// Array overload works too
int[] array = [3, 1, 4, 1, 5, 9, 2, 6, /* ... 27 elements */];
MySorter.Sort(array);

// IComparer<T> for custom ordering
MySorter.Sort(data, Comparer<int>.Create((a, b) => b.CompareTo(a)));
```

That's it. The source generator does the rest at compile time: picks the optimal sorting network for each declared size, emits unrolled scalar compare-and-swap code, and — where profitable — emits SIMD-vectorized paths with runtime hardware detection.

### Custom Types

Any value type implementing `IComparable<T>` gets unrolled `.CompareTo()` methods — the JIT devirtualizes and inlines these on value types:

```csharp
public record struct Point(int X, int Y) : IComparable<Point>
{
    public int CompareTo(Point other)
    {
        int cmp = X.CompareTo(other.X);
        return cmp != 0 ? cmp : Y.CompareTo(other.Y);
    }
}

[SortingNetwork(27, typeof(Point))]
partial class MySorter { }

Span<Point> points = /* ... */;
MySorter.Sort(points); // unrolled CompareTo
```

For types that don't implement `IComparable<T>`, the `IComparer<T>` overload is generated — you pass a comparer at runtime.

### Handling Unsupported Sizes

Define an `OnFallback` method to handle sizes outside your declared networks:

```csharp
[SortingNetwork(27, typeof(int))]
partial class MySorter
{
    static void OnFallback(Span<int> span)
    {
        span.Sort(); // delegate to BCL for other sizes
    }
}

MySorter.Sort(data);       // size 27 → sorting network
MySorter.Sort(otherData);  // any other size → OnFallback
```

Without `OnFallback`, unsupported sizes throw `ArgumentException`.

## The Architecture Challenge

Part 1's code generator was a script that wrote `.cs` files to disk. That works fine for a single repo with two sizes, but it doesn't scale:

- Users would need to run a script and check in generated code
- Adding a new size means re-running the script
- No IDE integration — no IntelliSense on generated methods until you build

A Roslyn source generator solves all of this. The generator runs inside the compiler, reads your `[SortingNetwork]` attributes, and emits code as part of the build. IntelliSense works immediately. No scripts. No generated files to check in.

The generator has these main components:

| Component | Purpose |
|-----------|---------|
| `SortingNetworkGenerator.cs` | Roslyn `IIncrementalGenerator` entry point — reads attributes, dispatches to emitters |
| `NetworkDatabase.cs` | Database of best-known sorting networks for sizes 2–64 |
| `BatcherNetworkBuilder.cs` | Builds Batcher's odd-even merge sort networks algorithmically |
| `ScalarEmitter.cs` | Emits unrolled compare-and-swap code for all types |
| `SimdX86Emitter.cs` | Emits AVX2 and AVX-512 SIMD paths |
| `SimdArmEmitter.cs` | Emits ARM64 AdvSimd/NEON paths |

The `NetworkDatabase` holds the actual comparator sequences. For sizes 2–22, networks are generated at compile time via Batcher's algorithm. For sizes 23–32, optimal/best-known networks are embedded (including the depth-13 networks from the paper for 27/28). For sizes 33–64, best-known networks from Dobbelaere's SorterHunter project are used.

## Expanding to All Sizes

Part 1 only supported sizes 27 and 28. Getting to 2–64 required several things:

**Sizes 2–22 — Batcher's odd-even merge sort:** These small sizes use a well-known constructive algorithm. Batcher's method isn't optimal (it uses more comparators than necessary), but it produces valid networks with reasonable depth for small inputs. The `BatcherNetworkBuilder` generates these programmatically — no hardcoded data needed.

**Sizes 23–26 and 29–32 — Best-known optimal networks (PR #74):** These fill the gap between the Batcher range and the paper's sizes. They come from research into optimal sorting networks and are embedded directly in `NetworkDatabase.cs`.

**Sizes 33–64 — SorterHunter (PR #100):** Bert Dobbelaere's [SorterHunter](https://github.com/bertdobbelaere/SorterHunter) project maintains a database of best-known sorting networks. These are the current best results from the research community.

## SIMD for Every Size

The Part 1 SIMD paths were hardcoded for 27/28 elements. Making the SIMD emitters work for arbitrary sizes 2–64 was a significant effort.

### x86: AVX2 and AVX-512

The `SimdX86Emitter` generates hardware-specific SIMD code based on element type and size:

- **byte/sbyte:** AVX-512 VBMI uses a single `Vector512<byte>` with `PermuteVar64x8` for all sizes up to 64. On CPUs without VBMI, AVX2 falls back to `Vector256<byte>` for sizes up to 32.
- **short/ushort/char:** AVX-512BW uses `PermuteVar32x16` (single register for sizes ≤32) or `PermuteVar32x16x2` (two registers for sizes 33–64).
- **int/uint/float:** AVX2 uses four `Vector256` registers with `PermuteVar8x32` cross-vector shuffles. AVX-512F uses two `Vector512` registers with `PermuteVar16x32x2`.
- **long/ulong/double:** AVX-512F uses four `Vector512` registers. AVX2 fallback uses seven `Vector256<double>` with `Permute4x64`.

### ARM64: AdvSimd/NEON

The `SimdArmEmitter` generates NEON paths using `Vector128` registers:

- **byte/sbyte:** Up to four `Vector128` registers with TBL4 lookups (sizes up to 64)
- **short/ushort/char:** Up to eight registers with multi-stage TBL/TBX chains (sizes up to 64)
- **int/uint/float:** Up to eight registers with two-stage TBL/TBX (sizes up to 32; scalar fallback above 32)

A critical optimization on ARM64 was **TBL1 vs TBL4** (PR #101). When all elements of a shuffled vector come from a single source register, the generator emits `Vector128.Shuffle` (TBL1) instead of `VectorTableLookup` (TBL4). On Ampere Altra/Neoverse cores, TBL4 has significantly higher latency than TBL1 — this optimization was the difference between being **slower** and **1.6x faster** than `Array.Sort` for `char` on those cores.

## Benchmark Results

All benchmarks run on .NET 10 with BenchmarkDotNet, zero allocations confirmed via `[MemoryDiagnoser]`.

### x86 AVX2 (Intel Core i9-9900K)

| Type | Array.Sort (27) | GeneratedSort (27) | Speedup |
|------|----------------|-------------------|---------|
| byte | 1,303 ns | 38 ns | **34x** |
| sbyte | 1,479 ns | 39 ns | **38x** |
| float | 1,598 ns | 66 ns | **24x** |
| double | 1,639 ns | 100 ns | **16x** |
| short | 1,407 ns | 100 ns | **14x** |
| long | 1,465 ns | 100 ns | **15x** |
| int | 105 ns | 54 ns | **1.9x** |
| string | 1,076 ns | 514 ns | **2.1x** |
| decimal | 2,376 ns | 794 ns | **3.0x** |

### x86 AVX-512 VBMI (AMD EPYC 9V74)

With AVX-512 VBMI, **all** byte/sbyte sizes fit in a single `Vector512`:

| Type | Size | Array.Sort | GeneratedSort | Speedup |
|------|------|-----------|--------------|---------|
| byte | 27 | 1,250 ns | 53 ns | **24x** |
| byte | 32 | 1,516 ns | 54 ns | **28x** |
| byte | 34 | 1,759 ns | 64 ns | **27x** |
| sbyte | 38 | 2,160 ns | 68 ns | **32x** |

### ARM64 (Apple M1, AdvSimd/NEON)

| Type | Array.Sort (27) | GeneratedSort (27) | Speedup |
|------|----------------|-------------------|---------|
| byte | 1,132 ns | 31 ns | **37x** |
| sbyte | 1,206 ns | 32 ns | **38x** |
| short | 1,124 ns | 48 ns | **23x** |
| float | 1,363 ns | 74 ns | **18x** |
| double | 1,461 ns | 110 ns | **13x** |

### Sizes 33–64 (ARM64, SIMD-accelerated)

| Type | Size | Array.Sort | GeneratedSort | Speedup |
|------|------|-----------|--------------|---------|
| byte | 34 | 2,831 ns | 85 ns | **33x** |
| sbyte | 38 | 3,340 ns | 94 ns | **36x** |
| short | 40 | 3,439 ns | 164 ns | **21x** |
| float | 36 | 3,269 ns | 220 ns | **15x** |

### Custom value types

| Type | Array.Sort (27) | GeneratedSort (27) | Speedup |
|------|----------------|-------------------|---------|
| Record (2× int) | 3,839 ns | 1,129 ns | **3.4x** |

## The Journey: PRs #63–119

Part 1 ended at PR #59 with 59 PRs in 3 days. The source generator work added another 21 PRs over the following two weeks, bringing the total to **80 merged PRs**.

### Phase 1 — Source Generator Foundation (Apr 25)

- **PR #63:** Initial Roslyn source generator — `[SortingNetwork]` attribute, scalar emitter for all primitives
- **PR #70:** Array overloads (`T[]` in addition to `Span<T>`)
- **PR #71:** ARM SIMD emitter for the source generator
- **PR #73:** AVX2 fallback for 16-bit types
- **PR #75:** AVX2 path for `double`

### Phase 2 — Expand to All Sizes (Apr 28 – May 2)

- **PR #74:** Add optimal networks for sizes 23–26 and 29–32
- **PR #100:** Add best-known networks for sizes 33–64 from SorterHunter
- **PR #88:** Expand SIMD emitters to support sizes 33–64
- **PR #89:** String type support in the source generator
- **PR #81:** `IComparer<T>` overloads for custom ordering

### Phase 3 — Type System & API (May 2–3)

- **PR #98:** Support arbitrary types via `IComparer<T>` (not just `IComparable<T>`)
- **PR #99:** `OnFallback` API for configurable behavior on unsupported sizes
- **PR #103:** Fix `nint`/`nuint` scalar perf by delegating to fixed-size types
- **PR #113:** Add `decimal` as a first-class type
- **PR #107:** AVX-512 VBMI as primary path for all byte/sbyte sizes

### Phase 4 — Ship It (May 3–5)

- **PR #90:** Remove the old non-generator sorter, restructure for NuGet
- **PR #97:** Add sample project to verify end-to-end
- **PR #109:** PublicApi analyzer for API surface tracking
- **PR #114:** Fix incremental generator caching for IDE performance
- **PR #116:** Rename NuGet package to `SortingNetworks.SourceGen`
- **PR #118:** Target `net8.0` as minimum TFM

## Lessons Learned

### Source generators have sharp edges

Incremental source generators need careful equality implementations. PR #114 fixed a bug where the generator wasn't caching properly — the `Equatable` model types needed correct `Equals`/`GetHashCode` implementations or the generator would re-run on every keystroke, killing IDE performance.

### Network selection matters more than you'd think

Batcher's odd-even merge sort is elegant and easy to generate algorithmically, but it's not optimal — it uses more comparators than necessary. For sizes 23+, switching to best-known networks from research databases (the arXiv paper, SorterHunter) saved layers and comparators, which translates directly to fewer SIMD instructions or fewer branches in the scalar path.

### SIMD register strategy varies by type width

The generator can't use a one-size-fits-all SIMD approach. `byte` fits 64 elements in one `Vector512`, but `double` fits only 8 in a `Vector512` — meaning 27 doubles need four registers and cross-vector shuffles. The emitters have per-type logic to pick the right register count and shuffle strategy.

### ARM64 is not one architecture

The TBL1 optimization story (PR #101) was a reminder that ARM64 performance varies wildly between implementations. Apple Silicon handles TBL4 (4-register table lookup) efficiently, but Ampere Neoverse does not. Without the TBL1 optimization for intra-register shuffles, the SIMD path was actually **slower** than `Array.Sort` on Ampere for `char`.

## What's Next

The source generator is published and usable today. Areas for future work:

- **More sizes** — The current limit is 64. Larger networks exist but may not be practical for sorting network approaches.
- **Descending sort** — Currently requires `IComparer<T>` overload; a dedicated attribute parameter could emit more optimal code.
- **Runtime BCL integration** — If .NET ever wants built-in sorting network support, the generator's architecture could inform that design.

## Links

- **NuGet:** [`SortingNetworks.SourceGen`](https://www.nuget.org/packages/SortingNetworks.SourceGen)
- **Release:** [v0.1.0 on GitHub](https://github.com/jonathanpeppers/SortingNetworks/releases/tag/0.1.0)
- **Repo:** [github.com/jonathanpeppers/SortingNetworks](https://github.com/jonathanpeppers/SortingNetworks)
- **Part 1:** [Sorting Networks: From Paper to Performance with AI](/2026/04/22/sorting-networks-from-paper-to-performance-with-ai.html)
- **Paper:** [arXiv:2511.04107 — "Depth-13 Sorting Networks for 28 Channels"](https://arxiv.org/abs/2511.04107)
