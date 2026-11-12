---
name: clean-code
description: Judgment framework for writing, refactoring and reviewing functions: honest signatures, single-level bodies, types encoding invariants, clean-code-vs-performance tradeoffs. Use when writing or refactoring functions, doing code review, choosing polymorphism vs switch vs lookup table, or when the user mentions clean code, 重构, 代码评审, 函数设计, 这段代码干净吗, 怎么写函数 — even if they never say "clean code".
license: MIT
metadata: { author: L1rics, version: "1.0" }
---

Clean is not tidy. The goal is never "small functions" or "no switch" for their own sake — it is making code match the real structure of the problem it solves. Every rule below serves that match. When a rule and the structure disagree, the structure wins.

Run three tests on any function you write or review. They compress Logan Smith's "How to write the perfect function" talk, stress-tested against Casey Muratori's "Clean Code, Horrible Performance" critique. Examples are C++; the tests are language-agnostic.

**Test 1 — Honesty: does it touch the world only through its signature?**

An honest function reads and writes nothing that isn't reachable through its arguments, so the caller fully controls its behavior by constructing inputs. A dishonest function's signature lies about its inputs/outputs — usually a hidden global, singleton, or "who ran before me" dependency.

```cpp
// dishonest: same arguments, different results — secretly depends on global RNG state
void populate(int n) { std::srand(std::time(nullptr)); /* ... rand() ... */ }

// honest: randomness becomes the caller's decision (fixed seed = reproducible, testable, debuggable)
void populate(std::mt19937& rng, int n);
```

- Dishonesty is **infectious**: an honest function cannot call a dishonest one without becoming dishonest. So honest functions live at the leaves of the call tree. Push all impurity (IO, time, randomness, rendering) to the topmost level and inject it — pass the RNG, the clock, the writer, or a callback.
- In-place mutation is honest if the signature grants it: `sort(v)` mutating the vector you handed it is fine. Reading mutable global state is the classic lie.
- Framework hooks (`update()`, `main`) invert the direction: they hide the *call site*, not the implementation. Keep their bodies minimal glue that jumps into your own honest functions.
- If a function can't be unit-tested without booting the world, it is probably dishonest. That — not length — is the real review finding.

**Test 2 — Signature empathy: ask for exactly what you need, enforce what you require.**

Two failure modes, in opposite directions:

- Asking too much. A function that only loops should not demand a `vector<int>` — take a span/slice, or just "anything iterable". Passing a whole config class when two fields suffice is the "cashier demands your entire wallet" smell.
- Asking too little. Requirements that live only in comments ("must be normalized", "caller must hold the lock") are lies the type system can refute:

```cpp
// contract in a comment — forgettable
Vec3 reflect(Vec3 v, Vec3 normal);              // normal must be unit length!

// contract in a type — unforgettable
Vec3 reflect(Vec3 v, UnitVec3 normal);

auto receipt = lock(mutex);                     // lock() returns an empty receipt type;
Data d = read_guarded(store, receipt);          // read_guarded won't compile without it
```

Also: more than ~5 positional parameters, or a trailing mystery `bool` → take a params struct or strong types. And when part of the implementation is still unclear, call the function you wish existed and implement it later — a call site abstracts detail in time, not just in space.

**Test 3 — Abstraction level: every line of the body at the same altitude.**

Writing a function is stacking bricks — existing lower-level functions. Zooming into a brick to fiddle inside it must be a separate function call. Radar: **section comments inside a body ("// lowercase the name", "// binary search") mean mixed levels** — a to-do list of functions pretending to be one function.

```cpp
// mixed levels: character codes + hand-rolled binary search + business rule, plus exit(1)?!
bool asset_has_type(std::string_view name, AssetType type);

// single level: the problem was a data structure, not a procedure
bool asset_has_type(std::string_view name, AssetType type) {
    const Asset* a = assets.find_ci(name);      // case-insensitive map brick
    return a && a->type == type;
}
```

When extracting bricks, notice when several functions perform the same moves (lowercase + search here, lowercase + insert there): you are hand-maintaining an ad-hoc data structure — encapsulate it. This test subsumes the classics: single responsibility (one reason to change), produce-vs-consume separation, no non-trivial raw loops. A body may *do* things (call an algorithm, check its result); opening the algorithm's hood inline is where to draw the line.

**The performance counterweight — do not skip this test on hot paths.**

Muratori's shape-area benchmark: virtual dispatch ~35 cycles/shape, a switch ~24, noticing that every formula is just `coef × W × H` and using a lookup table ~3 (~10x; 20–25x once vectorizable). The "clean" class hierarchy fragmented one cross-type pattern across four files — humans *and* the compiler lost sight of it. Lessons:

- The reader you are optimizing for includes the optimizer. Indirection hides structure from both.
- Profiler-flagged hot paths earn the right to look "ugly": flat structs, switch or table dispatch, no virtual calls. Note the fast version above is still honest (all inputs/outputs in the signature) and still single-level — speed and the three tests usually align once the abstraction matches the problem.
- Cold business paths: readability compounds, cycles don't (Hovinen: at 1000 QPS, recovering 250 cycles takes over 1000 years to repay one hour of engineering).

**Applying it**

- Writing new code: decide dependencies first (what do I need from the world → make it a parameter), sketch the function you wish existed, then implement downward, keeping each body single-level.
- Reviewing code: run the tests in order — honesty (hidden state? testable without the world?), signature (over- or under-asking? comment-enforced contracts?), abstraction level (section comments? raw loops fiddling internals?) — then grant hot-path exceptions.

**Going deeper — read on demand, not upfront**

- Full talk notes, all code examples, the honest/dishonest function taxonomy: `references/perfect-function.md`
- The debate: benchmark data, rule-by-rule verdicts, Uncle Bob's rebuttal, all sources: `references/clean-code-debate.md`
