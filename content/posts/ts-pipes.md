+++ 
draft = false
date = 2024-04-09T22:45:32+05:00
title = "Pipes and Composition in TypeScript"
description = "How pipe and functional composition turn nested transformations into readable, testable data flow—and what breaks when you do not use them."
slug = "ts-pipes"
authors = []
tags = ["typescript", "functional-programming"]
categories = []
externalLink = ""
series = []
+++

TypeScript does not have a built-in pipe operator (unlike some other languages), but the *idea* is everywhere: take a value, run it through a sequence of small functions, and get a result. When you structure code that way, you are doing Functional Composition: building a larger transformation out of smaller, reusable pieces.

## Composition: `compose` vs `pipe`

**Composition** ties functions together so the output of one feeds the next.

- **`compose(f, g, h)(x)`** usually means `f(g(h(x)))`: evaluation flows **right to left** (you build the pipeline from the innermost step outward). This matches mathematical composition \(f \circ g\).
- **`pipe(x, h, g, f)`** (or a left-to-right chain) means `f(g(h(x)))` when read in call order: you start with `x`, then `h`, then `g`, then `f`. It matches **reading order** in English source code.

Both express the same underlying idea; **pipe** is often ergonomically nicer in application code because the steps appear in the order they run.

## A minimal `pipe` in TypeScript

You can implement a small `pipe` with `reduce` (many libraries—lodash/fp, Ramda, fp-ts—offer richer, fully-typed variants):

```typescript
function pipe<V>(value: V): V;
function pipe<V, A>(value: V, f1: (x: V) => A): A;
function pipe<V, A, B>(value: V, f1: (x: V) => A, f2: (x: A) => B): B;
function pipe<V, A, B, C>(
  value: V,
  f1: (x: V) => A,
  f2: (x: A) => B,
  f3: (x: B) => C
): C;
function pipe(value: unknown, ...fns: Array<(x: unknown) => unknown>): unknown {
  return fns.reduce((acc, fn) => fn(acc), value);
}
```

Example:

```typescript
const trimmed = (s: string) => s.trim();
const upper = (s: string) => s.toUpperCase();

const label = pipe("  hello  ", trimmed, upper); // "HELLO"
```

For larger applications, reaching for a well-tested utility (or code-generated overloads) avoids manually extending overload tuples forever.

## Pitfalls when you skip composition

**1. Nested calls are read inside-out**

```typescript
const result = finalize(normalize(parseRaw(input)));
```

You must start in the middle or the right and work outward. That is fine for two steps; it does not scale when five or six transformations pile up.

**2. “Temp variable staircase”**

```typescript
const a = parseRaw(input);
const b = normalize(a);
const c = finalize(b);
```

This restores readability but adds noise and reassignable names. Every new step needs another name unless you adopt a pipe style (or adopt block scoping discipline rigorously).

**3. Mixed concerns in one function**

Without small steps, teams often cram validation, mapping, I/O side effects, and formatting into one large function. That makes unit testing and reuse painful: you cannot test “normalize” without dragging in everything else.

**4. Hidden mutation and order bugs**

When logic is a tangle of statements, it is easier to mutate shared objects between steps or to call helpers in the wrong order. Pure, single-purpose functions composed in a pipe make ordering explicit and reduce accidental cross-step mutation.

**5. Harder refactors**

Inserting “log metrics here” or “cap values here” in a nested expression is awkward. In a pipe, it is often one more function in the list.

## Why this pattern is useful

- **Readability**: Data flows in one direction, matching how you describe the feature in words.
- **Testability**: Each function is a candidate for a small unit test; the pipeline is a thin orchestration layer.
- **Reuse**: The same `normalize` can appear in multiple pipelines.
- **Change control**: Swapping or inserting steps is localized; diffs stay small and reviews stay focused.

Pipes and composition do not replace good domain modeling or error handling. You still need a story for failures (e.g. `Result` types, exceptions, or early returns) and for async work (async pipelines or `async`/`await` chains). They are a **structural** tool: they keep transformations honest, linear, and easy to reason about—like an assembly line you can actually read from left to right.
