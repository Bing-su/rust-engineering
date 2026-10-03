---
name: rust-engineering
description: Design, implement, refactor, and review production Rust codebases. Prioritizes maintainable codebase architecture, domain modeling, deep modules, clear seams, testability, and simple dependency structure, while using Rust's type system, ownership model, and tooling to reinforce those goals.
---

# Rust Engineering

Write Rust that is easy to understand, change, test, and operate.

Correctness, memory safety, security, required compatibility, and the user's requirements are mandatory constraints. Architectural preferences never justify violating them.
Prefer a well-designed codebase expressed idiomatically in Rust over code that merely demonstrates sophisticated Rust techniques.

Within those constraints, the priority order is:

1. Clear domain model
2. Good module boundaries
3. Locality of behavior and change
4. Small, stable interfaces
5. Explicit dependencies and effects
6. Testable behavior
7. Meaningful failures and idiomatic Rust
8. Performance, when evidence requires it

Do not sacrifice higher priorities for lower ones without a concrete reason.

---

# Workflow

Choose **Implement** for requested changes and **Review** for an assessment. Review alone does not authorize edits. Both start with Explore and use the relevant principles below.

## Explore

Apply Understand Before Editing. Establish the requested behavior or review scope and read repository instructions. Also inspect the lockfile, supported targets, and CI commands.

Finish exploration when you can identify the behavior's owner, affected callers and contracts, and checks needed to verify it.

## Implement

1. Identify the smallest useful interface, important invariants, side effects, and ownership model. For a non-trivial new module, briefly consider two plausible designs.
2. Complete one useful behavior end-to-end. Reuse existing code and suitable dependencies; apply Prefer Crates That Remove Maintained Code before hand-writing supporting functionality. Keep unrelated cleanup out of scope.
3. Add or update the smallest meaningful regression check for changed behavior. For a bug, establish a check exposing the failure before fixing its shared cause.
4. Run relevant verification below and inspect the final diff for correctness, compatibility, and unnecessary concepts. Preserve behavior during refactoring unless a behavior change was requested.

Finish when the requested behavior is implemented, relevant checks pass, and the diff contains only justified changes. If verification is blocked, report the remaining gap explicitly rather than claiming verified completion.

## Review

1. **Establish scope.** Use the requested commit, branch, diff, files, or whole-codebase scope. For a working-tree review, include staged, unstaged, and relevant untracked files. Identify affected public APIs, persistent data, wire formats, and security boundaries.
2. **Trace behavior.** Follow the scoped code through callers, state changes, effects, failure paths, cleanup, and observable output. Compare it with the requested specification and repository standards.
3. **Validate findings.** Apply the relevant risks below. Search and lint hits are leads, not findings: confirm the triggering condition and consequence in surrounding code.
4. **Verify and report.** Run relevant checks where practical. Report actionable findings by severity; separate unconfirmed risks and verification gaps from confirmed defects. Apply fixes only when requested.

| Changed area              | Review risks                                                                                                                               |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Inputs and failures       | Boundary validation, reachable panics, arithmetic on external sizes, partial updates, rollback and cleanup failures                        |
| Async and concurrency     | Lock ordering, guards across await, blocking executor work, cancellation data loss, bounded queues, overload, task ownership and shutdown  |
| Unsafe and FFI            | Safety contracts, validity, alignment, initialization, aliasing, layout, ownership transfer, deallocation, unwinding, manual `Send`/`Sync` |
| Public contracts          | Visibility, trait and auto-trait changes, error variants, serialized shapes, features, targets, MSRV and semver                            |
| Security and dependencies | Attacker-controlled paths, secret exposure in logs/errors/Debug output, resource limits, advisories and repository supply-chain policy     |
| Performance               | Material costs on actual workloads, quadratic work, unbounded growth and evidence supporting claims                                        |
| Tests and documentation   | Observable regressions, relevant failures, API contracts, safety requirements and executable public examples                               |

| Finding field | Required content                                                                                                                            |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Severity      | P1: urgent correctness, safety or security defect; P2: actionable defect under a concrete condition; P3: lower-impact maintainability issue |
| Location      | File and tight line range                                                                                                                   |
| Condition     | Concrete input or caller path triggering the issue                                                                                          |
| Impact        | Observable behavior, safety, compatibility or maintenance consequence                                                                       |
| Correction    | Smallest useful fix or regression check                                                                                                     |

Example: `[P2] Preserve the queued job on cancellation — src/worker.rs:42; cancelling after dequeue loses the job before persistence; acknowledge only after the write succeeds.`

Style preferences are not correctness findings. If no actionable findings remain, say so. Finish with checks performed and any unreviewed scope or verification gaps.

## Verify

Use the project's existing build and CI commands first, including required generation or fixture steps. Select packages, Cargo targets, platform targets, and feature configurations from the affected code and supported CI matrix. Keep check, test, and Clippy configurations aligned.

For a single crate with default features, a baseline is:

```bash
cargo fmt --check
cargo check --all-targets
cargo test
cargo clippy --all-targets -- -D warnings
```

Follow repository lint policy when it differs. Use `-p` or `--workspace` for the intended package scope and `--target` for relevant platform checks. Test affected optional features and supported configurations with or without default features. Use `--all-features` only when that combination is valid; mutually exclusive features need separate runs.

For example, for a crate supporting an optional `json` feature, repeat applicable check, test, and Clippy commands with `--features json`. Include `--no-default-features` when supported and affected.

| Additional check           | Use when                                                         |
| -------------------------- | ---------------------------------------------------------------- |
| `cargo nextest run`        | The project uses nextest; run doctests separately                |
| `cargo test --doc`         | Public examples changed or the chosen test runner omits doctests |
| `cargo +nightly miri test` | Relevant unsafe or memory-sensitive paths can run under Miri     |
| Project benchmarks         | A performance improvement is claimed                             |
| MSRV toolchain checks      | Changed code or dependencies may raise the declared MSRV         |

Report what ran, passed, failed, or could not run. Compilation and lint success do not prove behavioral correctness.

---

# Understand Before Editing

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

# Model the Domain Explicitly

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
// Keep user and session identifiers distinct so callers cannot swap them.
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

| Domain need          | Prefer                                                                                               |
| -------------------- | ---------------------------------------------------------------------------------------------------- |
| Closed states        | An enum, such as `JobState::{Pending, Running, Completed, Failed}`, rather than coordinated booleans |
| Distinct identifiers | Newtypes when the distinction matters to callers                                                     |
| Constrained values   | Validated constructors and controlled mutation                                                       |
| Expected absence     | `Option`                                                                                             |
| Expected failure     | `Result`                                                                                             |

Check allowed transitions as well as representable states. Prefer ordinary structs and enums to generic, lifetime, macro, or typestate machinery unless the extra machinery removes concrete complexity. Types should reduce cognitive load.

---

# Design Deep Modules

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

An interface includes the invariants, ordering, error modes, and configuration callers must understand, not only type signatures.

---

# Put Seams Where Behavior Actually Varies

A seam is a place where one implementation can be changed without rewriting its callers.

Introduce seams for real variation, not imagined future variation.

Avoid creating traits merely because dependency inversion is theoretically possible.

One implementation usually does not require a trait.

Prefer:

```rust
struct SqliteEventStore {
    // Keep persistence concrete until a real seam requires polymorphism.
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

Duplication is often cheaper than the wrong abstraction. Generalize shared semantics once they are clear, rather than similar syntax that needs configuration flags to fit.

Avoid generic frameworks for hypothetical future requirements.

---

# Keep Dependencies Pointing Toward Stable Concepts

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

# Optimize for Locality

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

# Prefer Explicit Data Flow

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

# Use Rust Ownership to Express Architecture

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

The compiler is a design constraint, not the architect; a borrow-checker fix should still communicate responsibility clearly.

---

# Minimize Shared Mutable State

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

Choose the simplest correct option; an actor is not automatically simpler than a lock.

Keep locking regions small and obvious.

Never hold blocking locks across `.await`.

Treat cancellation, shutdown, backpressure, and task ownership as part of async API design.

---

# Error Handling Is Interface Design

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

# Prefer Composition Over Configuration Explosion

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

# Keep Public Interfaces Small

Every public item creates maintenance cost.

Default to private visibility.

Use `pub(crate)` for visibility within the current crate. Other crates, including workspace siblings, require an appropriate `pub` interface; Rust has no workspace-only visibility modifier.

Expose only what callers genuinely need.

Avoid leaking internal helper types.

Do not make fields public merely for convenience.

A smaller interface:

- is easier to understand,
- is easier to test,
- allows more implementation freedom,
- reduces accidental coupling.

---

# Tests Verify Behavior Through Seams

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

# Use Specialized Testing When It Buys Confidence

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

# Unsafe Requires a Proof Obligation

Do not use `unsafe` to bypass architectural or borrow-checker problems.

Before adding unsafe code, determine whether a safe design is practical.

| Unsafe construct                       | Required argument                                                                                             |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `unsafe` block                         | A `SAFETY` comment explaining why the operation's preconditions hold here                                     |
| `unsafe impl`, including `Send`/`Sync` | An explicit proof of the trait's safety contract, including relevant ownership and synchronization invariants |
| Public `unsafe fn` or unsafe trait     | A `# Safety` contract documenting caller or implementer obligations                                           |

For example, before constructing a slice from a raw pointer, explain its validity, alignment, initialization, length, lifetime, and aliasing guarantees. Non-nullness alone is not a complete proof.

At FFI boundaries, verify layout, ownership transfer, matching deallocation, callback lifetimes, and permitted unwinding.

Keep unsafe code:

- small,
- isolated,
- covered by safe interfaces,
- tested independently where practical.

Use Miri when relevant and supported. Passing Miri on exercised paths supplements the safety argument; it does not prove soundness for every caller.

Safe wrappers must uphold hidden safety obligations for every permitted caller. Unsafe APIs must document any obligations left to their callers.

---

# Performance Requires Evidence

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

# Prefer Crates That Remove Maintained Code

Minimal implementation means minimizing code and behavior we must maintain, not minimizing the dependency count. Prefer a suitable established crate over hand-written parsing, protocol handling, repetitive trait implementations, or custom infrastructure.

Reuse the project's existing suitable crate first. Otherwise, when a trigger below applies, **add the recommended crate by default** instead of reimplementing its functionality. An absent dependency is not a reason to choose a manual implementation. These are task-specific defaults, not a starter dependency bundle.

Use the standard library when it already covers the required semantics directly. Depart from a default for a concrete reason, such as repository policy, MSRV, `no_std`, target support, license restrictions, or disproportionate build/binary cost. Explain that reason briefly; dependency count alone is insufficient. Keep compatible existing alternatives rather than migrating them just to match this list.

## Common Defaults

| Task trigger                                                           | Default crate                                                                                                                                               | Replace or avoid                                                                                                                     |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Define custom errors that callers distinguish                          | [thiserror](https://docs.rs/thiserror/latest/thiserror/)                                                                                                    | Repetitive manual `Display`, `Error`, source chaining, and `From` implementations                                                    |
| Propagate varied errors with context at application/CLI boundaries     | [anyhow](https://docs.rs/anyhow/latest/anyhow/)                                                                                                             | Ad-hoc boxed errors and string conversion; retain typed errors where callers need recovery decisions                                 |
| Serialize/deserialize structured data                                  | [serde](https://docs.rs/serde/latest/serde/), plus [serde_json](https://docs.rs/serde_json/latest/serde_json/) for JSON                                     | Manual codecs and JSON string assembly; prefer typed payloads for known schemas                                                      |
| CLI options, flags, subcommands, help, or argument validation          | [clap](https://docs.rs/clap/latest/clap/)                                                                                                                   | A growing `std::env::args` parser; a single positional argument can stay in std                                                      |
| Operational logging, especially async or request-scoped diagnostics    | [tracing](https://docs.rs/tracing/latest/tracing/) and application-side [tracing-subscriber](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/) | Custom logging/context plumbing; libraries emit events and leave subscriber setup to applications                                    |
| Named combinable bit masks                                             | [bitflags](https://docs.rs/bitflags/latest/bitflags/)                                                                                                       | Manual flag wrappers, bit operators, and membership helpers; preserve required representation and unknown-bit behavior               |
| Parse, resolve, or modify URLs and query parameters                    | [url](https://docs.rs/url/latest/url/)                                                                                                                      | Splitting URLs or concatenating unescaped query strings; URL parsing does not establish destination authorization                    |
| Generate, parse, or format UUIDs required by the domain/protocol       | [uuid](https://docs.rs/uuid/latest/uuid/)                                                                                                                   | Custom UUID/random identifier implementations; retain ordinary numeric IDs where those are the actual contract                       |
| Regular-expression matching                                            | [regex](https://docs.rs/regex/latest/regex/)                                                                                                                | A homegrown pattern matcher; compile reusable patterns once, use string methods for literal matching                                 |
| Temporary files/directories with lifecycle cleanup                     | [tempfile](https://docs.rs/tempfile/latest/tempfile/)                                                                                                       | Invented temporary names and scattered cleanup; use as a dev dependency when only tests need it                                      |
| Recursive directory traversal                                          | [walkdir](https://docs.rs/walkdir/latest/walkdir/)                                                                                                          | Hand-written recursive traversal; preserve explicit error and symlink policy                                                         |
| Iterator operations missing from std that remove custom loops/helpers  | [itertools](https://docs.rs/itertools/latest/itertools/)                                                                                                    | Custom grouping, combinations, or join utilities; use std iterator methods when equivalent                                           |
| Calendar dates/times, timestamps, parsing, or formatting               | [chrono](https://docs.rs/chrono/latest/chrono/)                                                                                                             | Hand-written calendar arithmetic; use `std::time` for elapsed time and simple durations                                              |
| Exact decimal arithmetic with variable scale, such as prices and rates | [rust_decimal](https://docs.rs/rust_decimal/latest/rust_decimal/)                                                                                           | Floating-point money and a custom decimal type; fixed-scale integer minor units remain valid, with explicit rounding/overflow policy |

For example, a permission mask should usually become a `bitflags!` type rather than a newtype with manual operators. A library's typed error can use `thiserror`, while its CLI adds `anyhow::Context`; these choices serve different callers and can coexist.

## Development Defaults When the Technique Applies

| Concrete testing need                                       | Default dev dependency                                   | Use it for                                                                                                |
| ----------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Multiple named input/output cases or reusable test setup | [rstest](https://docs.rs/rstest/latest/rstest/) | Parameterized `#[case]` tests and `#[fixture]` setup instead of duplicated tests or a custom case runner |
| Properties over many parser, transform, or invariant inputs | [proptest](https://docs.rs/proptest/latest/proptest/)    | Input generation and shrinking rather than a custom randomized test harness                               |
| Large stable output best verified by reviewing diffs        | [insta](https://docs.rs/insta/latest/insta/)             | Snapshots with intentional review; normalize nondeterminism and keep focused assertions for small results |
| Repeated performance comparisons                            | [criterion](https://docs.rs/criterion/latest/criterion/) | Benchmark sampling and comparisons rather than a homegrown timing harness                                 |

Ordinary unit and integration tests use Rust's built-in test harness; `rstest` augments it when parameterization or fixtures reduce repetition. For example, use named cases for valid, empty, and malformed parser input so failures identify the specific case. Specialized tools still require the concrete risks described in Use Specialized Testing When It Buys Confidence.

## Add Only the Relevant Dependency

Before adding, check the workspace manifest and existing versions, current crate documentation and maintenance/advisories, required features, MSRV, targets, and repository dependency policy. Select a compatible version at implementation time rather than copying a version from this skill. Test/benchmark-only crates belong in dev dependencies.

Keep dependencies behind meaningful interfaces where it reduces coupling. Call them directly when a forwarding wrapper would add no value. Report the chosen crate and the implementation it replaced in one sentence; do not build a comparison framework or add unrelated dependencies.

---

# Refactoring Rules

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
