---
name: rust-engineering
description: Design, implement, refactor, and review production Rust codebases. Prioritizes maintainable codebase architecture, domain modeling, deep modules, clear seams, testability, and simple dependency structure, while using Rust's type system, ownership model, and tooling to reinforce those goals.
---

# Rust Engineering

Write Rust that is easy to understand, change, test, and operate.

Correct ownership and lifetimes are necessary, but they are not the primary goal.
Prefer a well-designed codebase expressed idiomatically in Rust over code that merely demonstrates sophisticated Rust techniques.

The priority order is:

1. Clear domain model
2. Good module boundaries
3. Locality of behavior and change
4. Small, stable interfaces
5. Explicit dependencies and effects
6. Testable behavior
7. Correctness and failure handling
8. Idiomatic Rust
9. Performance, when evidence requires it

Do not sacrifice higher priorities for lower ones without a concrete reason.

---

# 1. Understand Before Editing

Before making substantial changes:

- Read the relevant module, its callers, and its tests.
- Inspect `Cargo.toml`, workspace layout, features, and toolchain/MSRV constraints.
- Look for existing project conventions before introducing new ones.
- Read domain documentation, glossary, ADRs, or equivalent files when present.
- Determine whether the requested change belongs inside an existing module or reveals a missing abstraction.

Do not start by creating new traits, modules, wrappers, or generic abstractions.

First understand where the behavior naturally belongs.

When modifying an existing codebase, preserve its established architecture unless there is a concrete reason to change it.

---

# 2. Model the Domain Explicitly

Code should use the language of the problem domain.

Prefer domain concepts over implementation concepts.

Bad:

```rust
struct DataManager;
struct RequestHandler;
struct ItemProcessor;
```

Better:

```rust
struct AlertRule;
struct DetectionResult;
struct WorkflowExecution;
```

When two concepts have different rules or lifecycles, model them as different types even if their underlying representation is identical.

Prefer:

```rust
struct UserId(Uuid);
struct SessionId(Uuid);
```

over passing unrelated values as generic `Uuid`, `String`, or integers everywhere.

Use types to make invalid states difficult to construct, but do not create ceremonial wrapper types that add no useful invariant or meaning.

Ask:

- What concepts exist in this domain?
- Which states are valid?
- Which transitions are allowed?
- Which invariants can be represented structurally?
- Which distinctions matter to callers?

Use "parse, don't validate" where appropriate: convert loosely structured input into a validated domain representation at the system boundary.

---

# 3. Design Deep Modules

Prefer modules that hide substantial behavior behind small interfaces.

A good module provides leverage:

- callers learn little,
- callers can accomplish a lot,
- implementation details remain local,
- changes remain concentrated.

Avoid modules that merely rename or forward calls.

Before creating an abstraction, apply the deletion test:

> If this abstraction disappeared, would meaningful complexity spread across several callers?

If not, the abstraction may not be earning its existence.

Prefer:

```text
small interface
      ↓
substantial cohesive behavior
```

over:

```text
large interface
      ↓
thin forwarding layer
      ↓
another thin forwarding layer
```

A function, type, module, crate, or subsystem can all be a module in this sense.

---

# 4. Put Seams Where Behavior Actually Varies

A seam is a place where one implementation can be changed without rewriting its callers.

Introduce seams for real variation, not imagined future variation.

Avoid creating traits merely because dependency inversion is theoretically possible.

One implementation usually does not require a trait.

Prefer:

```rust
struct SqliteEventStore {
    // ...
}
```

until there is a real second implementation, testing requirement, plugin boundary, or architectural reason for polymorphism.

Add a trait when the seam has demonstrated value.

Good reasons include:

- multiple meaningful implementations,
- external infrastructure boundary,
- test substitute at a stable architectural seam,
- plugin or extension point,
- independent ownership of implementations.

Poor reasons include:

- "we may need another implementation someday",
- "traits are cleaner",
- "dependency injection requires interfaces".

Avoid Java-style interface proliferation.

---

# 5. Keep Dependencies Pointing Toward Stable Concepts

Domain logic should not depend unnecessarily on frameworks, databases, HTTP libraries, serialization formats, or runtime details.

Prefer:

```text
transport / persistence / framework
              ↓
         application logic
              ↓
           domain
```

Infrastructure should adapt to the domain, rather than defining it.

Do not leak infrastructure-specific types deeply into the codebase unless they are genuinely part of the domain.

For example, prefer converting an HTTP request into a domain command near the HTTP boundary rather than passing framework request objects through multiple layers.

Likewise, database row types should usually remain near persistence code.

---

# 6. Optimize for Locality

Behavior that changes together should usually live together.

Avoid spreading one feature across many tiny modules merely to achieve superficial separation.

Signs of poor locality:

- understanding one operation requires jumping through many files,
- changing one rule requires editing many unrelated modules,
- validation is duplicated across callers,
- orchestration logic is repeated,
- important invariants exist only in call-site conventions.

Prefer concentrating related behavior behind one coherent interface.

Small files are not automatically good design.
Large files are not automatically bad design.

Optimize for conceptual locality, not line count.

---

# 7. Prefer Explicit Data Flow

Make dependencies, inputs, outputs, and side effects visible.

Prefer functions that accept what they need and return meaningful results.

Prefer:

```rust
fn evaluate(rule: &Rule, event: &Event) -> DetectionResult
```

over hidden access to global state or implicit service locators.

Separate computation from effects where it improves clarity.

A useful shape is:

```text
input
  ↓
domain computation
  ↓
decision/result
  ↓
effectful adapter
```

Do not force pure-functional architecture where it makes the code less clear, but keep important business logic independently understandable whenever practical.

---

# 8. Use Rust Ownership to Express Architecture

Ownership should reflect responsibility.

Ask:

- Who owns this value?
- Who is allowed to mutate it?
- How long must it live?
- Is sharing actually necessary?

Do not use `clone()` merely to silence the borrow checker.

A borrow-checker conflict often indicates one of:

- ownership is assigned to the wrong module,
- responsibilities are mixed,
- a function is doing too much,
- data lives too long,
- mutation is occurring at the wrong level.

Reconsider the design before adding clones, `Arc`, `Mutex`, or lifetime complexity.

Prefer borrowed inputs where ownership is unnecessary:

```rust
fn parse(input: &str)
fn process(items: &[Item])
```

rather than:

```rust
fn parse(input: String)
fn process(items: Vec<Item>)
```

when the function does not need ownership.

But do not contort APIs to avoid small, intentional clones.
Clarity beats theoretical allocation purity.

---

# 9. Minimize Shared Mutable State

Shared mutable state is an architectural decision, not a convenience.

Do not introduce:

```rust
Arc<Mutex<T>>
```

as a default solution to ownership problems.

First consider:

- transferring ownership,
- message passing,
- immutable shared state,
- actor/task ownership,
- splitting read and write responsibilities,
- shortening lifetimes,
- restructuring the operation.

Use shared synchronization when the domain actually requires shared mutable access.

Keep locking regions small and obvious.

Never hold blocking locks across `.await`.

Treat cancellation, shutdown, backpressure, and task ownership as part of async API design.

---

# 10. Use Types for Important Invariants

Use Rust's type system where doing so removes ambiguity or invalid states.

Useful tools include:

- enums for closed state sets,
- newtypes for semantically distinct identifiers,
- constructors for validated values,
- non-empty or constrained domain types where meaningful,
- `Option` when absence is expected,
- `Result` when failure is part of the operation.

Prefer:

```rust
enum JobState {
    Pending,
    Running,
    Completed,
    Failed,
}
```

over loosely coordinated booleans such as:

```rust
is_started: bool,
is_finished: bool,
has_failed: bool,
```

Avoid type-level cleverness that makes normal code difficult to read.

The type system should reduce cognitive load, not move runtime complexity into compile-time puzzles.

---

# 11. Error Handling Is Interface Design

An error is part of a module's interface.

Do not expose internal implementation errors directly when callers should reason about a smaller set of domain failures.

Libraries should generally define meaningful error types.

Applications may add contextual information when propagating failures.

Avoid routine `unwrap()` and `expect()` in production paths.

They are acceptable when an invariant is genuinely guaranteed and locally obvious.

Do not collapse meaningful failures into strings too early.

Prefer errors that help callers decide what to do next.

Ask:

- Can the caller recover?
- Should this failure cross the module seam?
- Is this an invariant violation or an expected failure?
- Does the caller need implementation details?

---

# 12. Avoid Premature Abstraction

Duplication is often cheaper than the wrong abstraction.

Do not immediately generalize two similar pieces of code.

Wait until the shared concept becomes clear.

Prefer an abstraction based on shared semantics, not merely shared syntax.

Bad abstraction:

```text
two functions look similar
→ generic helper
→ complicated configuration flags
```

Better:

```text
two operations evolve
→ common domain concept emerges
→ abstraction represents that concept
```

Avoid generic frameworks inside application code unless repeated evidence justifies them.

Do not build mini-frameworks for hypothetical future requirements.

---

# 13. Prefer Composition Over Configuration Explosion

Avoid functions, structs, or modules whose behavior is controlled by many booleans or unrelated options.

Bad:

```rust
process(
    event,
    true,
    false,
    true,
    RetryMode::Aggressive,
)
```

Prefer explicit domain operations or configuration objects with coherent meaning.

If many flags change the semantics of an operation, consider whether multiple concepts have been compressed into one module.

---

# 14. Keep Public Interfaces Small

Every public item creates maintenance cost.

Default to private visibility.

Use `pub(crate)` when workspace-internal visibility is enough.

Expose only what callers genuinely need.

Avoid leaking internal helper types.

Do not make fields public merely for convenience.

A smaller interface:

- is easier to understand,
- is easier to test,
- allows more implementation freedom,
- reduces accidental coupling.

---

# 15. Tests Verify Behavior Through Seams

Tests should primarily exercise stable behavior through meaningful module interfaces.

Avoid tests coupled to internal implementation structure.

Do not mock every internal collaborator.

Prefer realistic tests at meaningful seams.

Use unit tests for dense deterministic logic.
Use integration tests for important interactions between modules.
Use end-to-end tests sparingly for critical system behavior.

A refactor that preserves behavior should not require rewriting most tests.

If tests break whenever internal structure changes, the test seam is probably wrong.

Test names should describe domain behavior, not implementation mechanics.

---

# 16. Use Specialized Testing When It Buys Confidence

Choose testing techniques based on risk.

Consider:

- property testing for parsers, transforms, invariants, and algorithms,
- fuzzing for hostile or structurally complex inputs,
- `loom` for subtle concurrency behavior,
- Miri for unsafe or memory-sensitive code,
- snapshot tests for stable structured output where reviewing diffs is valuable,
- benchmarks for performance-sensitive paths.

Do not add specialized tools ceremonially.

Use them where they address a concrete failure mode.

---

# 17. Unsafe Requires a Proof Obligation

Do not use `unsafe` to bypass architectural or borrow-checker problems.

Before adding unsafe code, determine whether a safe design is practical.

Every unsafe block must have an explicit safety argument describing the invariant that makes the operation valid.

Keep unsafe code:

- small,
- isolated,
- covered by safe interfaces,
- tested independently where practical.

Use Miri when relevant.

Unsafe code should reduce complexity at its callers, not export additional obligations to them.

---

# 18. Performance Requires Evidence

Do not optimize code based only on intuition.

Before complicating a design for performance:

1. identify the relevant workload,
2. measure it,
3. locate the actual bottleneck,
4. change the smallest useful area,
5. measure again.

Do not replace clear architecture with complex zero-copy, pooling, unsafe code, custom allocation, lock-free structures, or advanced generics without evidence.

Prefer algorithmic and architectural improvements before micro-optimizations.

Performance claims require measurements.

---

# 19. Dependencies Have Architectural Cost

Before adding a crate, ask:

- Does it eliminate meaningful complexity?
- Is the dependency actively maintained?
- Is its scope appropriate?
- Does it dominate the architecture?
- Does it introduce unnecessary runtime, build, or supply-chain cost?

Do not reimplement mature functionality merely to avoid dependencies.

But do not introduce large frameworks to solve tiny problems.

Prefer dependencies that remain behind local seams so they can be replaced without contaminating the domain model.

---

# 20. Refactoring Rules

Refactor toward:

- fewer concepts callers must understand,
- smaller interfaces,
- stronger locality,
- clearer ownership,
- fewer duplicated invariants,
- simpler dependencies,
- better domain language.

Do not refactor solely to:

- reduce line count,
- increase abstraction,
- introduce design patterns,
- split files,
- eliminate every duplicate,
- maximize trait usage.

A refactor should make future changes easier.

Prefer small structural improvements over broad rewrites.

Preserve behavior unless changing behavior is part of the task.

---

# 21. Implementation Workflow

For substantial changes, follow this sequence.

## Explore

Understand:

- domain concepts,
- existing architecture,
- callers,
- tests,
- dependencies,
- toolchain constraints.

## Design

Before coding, identify:

- which module owns the behavior,
- the smallest useful interface,
- important invariants,
- where side effects occur,
- whether a real seam exists,
- the intended ownership model.

For non-trivial new modules, briefly consider at least two plausible designs before committing to one.

## Implement

Work vertically.

Prefer completing one useful behavior end-to-end rather than creating many speculative layers first.

Keep changes scoped.

Avoid unrelated cleanup unless it directly enables the task.

## Verify

Run appropriate checks regularly.

Typical baseline:

```bash
cargo check
cargo test
cargo clippy --all-targets --all-features -- -D warnings
cargo fmt --check
```

Use the project's existing commands when they differ.

When available and appropriate:

```bash
cargo nextest run
cargo test --doc
cargo +nightly miri test
```

Do not claim completion if relevant checks were not run or failed.

## Review

Before finishing, inspect the change again from an architectural perspective.

Ask:

- Did this change introduce unnecessary concepts?
- Is the behavior located where it belongs?
- Did the interface become larger than necessary?
- Did infrastructure leak inward?
- Did ownership become clearer or more complicated?
- Are clones hiding a design problem?
- Are traits representing real seams?
- Are errors meaningful at the caller level?
- Do tests verify behavior rather than implementation?
- Is there speculative abstraction that can be removed?

Simplify before finishing.

---

# 22. Common Rust Agent Failure Modes

Actively avoid these patterns.

### Clone-driven development

Repeatedly cloning values to make ownership errors disappear.

Treat repeated cloning as a signal to inspect ownership and module responsibilities.

### Arc-Mutex-driven architecture

Wrapping shared application state in `Arc<Mutex<_>>` before designing who should own it.

### Trait-first design

Creating traits before real variability exists.

### Generic overengineering

Introducing lifetimes, generic parameters, associated types, macros, or typestate where ordinary structs and enums would communicate the design better.

### Layer proliferation

Creating:

```text
handler
→ service
→ manager
→ repository
→ adapter
```

where most layers merely forward calls.

### Primitive obsession

Passing strings, UUIDs, maps, or booleans through the entire system when domain distinctions matter.

### Infrastructure-shaped domain

Letting database schemas, JSON payloads, HTTP libraries, or framework conventions define core business types.

### Internal-test obsession

Testing private helpers extensively while missing the externally meaningful behavior.

### Compiler-satisfaction refactors

Restructuring code solely to appease the borrow checker without considering whether the new design communicates responsibility clearly.

The compiler is a design constraint, not the architect.

---

# Final Standard

Good Rust code is not code that uses the most Rust features.

Good Rust code makes:

- ownership obvious,
- domain concepts explicit,
- invalid states difficult,
- behavior local,
- interfaces small,
- dependencies visible,
- failures understandable,
- tests durable,
- changes predictable.

Prefer boring code with strong structure over clever code with sophisticated types.

Use Rust's strictness to reinforce good architecture, not as a substitute for it.
