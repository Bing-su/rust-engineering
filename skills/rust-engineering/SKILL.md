---
name: rust-engineering
description: General Rust engineering guidance for implementation, refactoring, reviews, domain modeling, and module boundaries. Prefer an available skill dedicated to the requested task; use this skill as Rust-specific supporting guidance or as a fallback when none applies.
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

Honor explicit user choices and repository instructions. When an available dedicated skill applies to the requested task, follow it for that task's scope, workflow, and reporting. Use this skill's Rust principles and verification guidance as supporting reference. For example, prefer a diagnosis skill for debugging, a TDD skill for requested test-first development, or a review skill for code review.

For parts of the task without an applicable dedicated skill, choose **Implement** for requested changes and **Review** for an assessment. These fallback workflows start with Explore and use the relevant principles below. Review alone does not authorize edits.

## Explore

Establish the requested behavior or review scope and read repository instructions. Start with the affected module, callers, tests, observable contracts, and relevant project checks. Expand into domain documentation, Cargo manifests, lockfile, supported targets, toolchain/MSRV, and CI configuration when the change or review scope reaches those concerns. Preserve established architecture unless a concrete problem justifies changing it.

Finish exploration when you can identify the behavior's owner, affected callers and contracts, and checks needed to verify it.

## Implement

1. Identify the smallest useful interface, important invariants, side effects, and ownership model. For a non-trivial new module, briefly consider two plausible designs.
2. Complete one useful behavior end-to-end. Reuse existing code and suitable dependencies; apply Dependency Selection before hand-writing supporting functionality. Keep unrelated cleanup out of scope.
3. Add or update the smallest meaningful regression check for changed behavior. For a bug, establish a check exposing the failure before fixing its shared cause. For a behavior-preserving refactor, reuse existing checks when they cover the affected contracts; add a check only for a concrete coverage gap.
4. Run relevant verification below and inspect the final diff for correctness, compatibility, and unnecessary concepts. Preserve behavior during refactoring unless a behavior change was requested.

Finish when the requested behavior is implemented, relevant checks pass, and the diff contains only justified changes. If verification is blocked, report the remaining gap explicitly rather than claiming verified completion.

## Review

This workflow and finding format apply only to fallback reviews.

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

Use the project's existing build and CI commands first, including required generation or fixture steps. Scale verification to the affected behavior and risk while completing required repository checks. Select packages, Cargo targets, platform targets, and feature configurations from the affected code and supported CI matrix. Keep check, test, and Clippy configurations aligned.

For a single crate with default features, a baseline is:

```bash
cargo fmt --check
cargo check --all-targets
cargo test
cargo clippy --all-targets -- -D warnings
```

Follow repository lint policy when it differs. Use `-p` or `--workspace` for the intended package scope and `--target` for relevant platform checks. Test affected optional features and supported configurations with or without default features. Use `--all-features` only when that combination is valid; mutually exclusive features need separate runs.

For example, for a crate supporting an optional `json` feature, repeat applicable check, test, and Clippy commands with `--features json`. Include `--no-default-features` when supported and affected.

| Additional check                      | Use when                                                                                  |
| ------------------------------------- | ----------------------------------------------------------------------------------------- |
| `cargo nextest run`                   | The project uses nextest; run doctests separately                                         |
| `cargo test --doc`                    | Public examples changed or the chosen test runner omits doctests                          |
| Selected ignored or hardware tests    | Affected behavior is covered by tests excluded from ordinary runs; execute when available |
| Reference or differential comparisons | Numerical or format compatibility needs a trusted implementation or pre-change baseline   |
| `cargo +nightly miri test`            | Relevant unsafe or memory-sensitive paths can run under Miri                              |
| Project benchmarks                    | A performance improvement is claimed                                                      |
| MSRV toolchain checks                 | Changed code or dependencies may raise the declared MSRV                                  |

Compare structured output by semantic values unless textual formatting is contractual. For floating-point comparisons, use tolerances justified by the operation, numeric precision, and execution environment; retain relevant length, range, and finite-value checks.

Report what ran, passed, failed, or could not run, including relevant excluded tests. Bound claims to the tested inputs, configurations, and workloads: compilation and lint success do not prove behavioral correctness, and passing small test cases does not establish memory use or performance under representative workloads.

---

## Load References When Needed

| Task condition                                                                                                                                                 | Required reference                                                                        |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Designing or changing domain types, module boundaries, polymorphic seams, ownership, or configuration APIs; reviewing those design choices                     | Read [Design examples](references/design-examples.md) for concrete tradeoffs and examples |
| Implementing functionality that may replace hand-written support code with a crate, selecting a dependency, or reviewing custom helpers and dependency choices | Read [Crate defaults](references/crate-defaults.md) before choosing an implementation     |

Load only the references relevant to the current task. The decision criteria below apply throughout.

## Design Decisions

| Concern       | Decision criterion                                                                                                                                                                                                                                                                       |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Domain model  | Use domain vocabulary. Distinguish concepts with different rules or lifecycles, validate boundary input into domain types, and check allowed transitions as well as representable states. Prefer ordinary structs and enums; wrappers must add a useful invariant or caller distinction. |
| Deep modules  | Hide cohesive behavior behind a small interface. Count invariants, ordering, failure modes, and configuration as interface cost. An abstraction earns its place when removing it would spread meaningful complexity across callers.                                                      |
| Seams         | Add traits for demonstrated variation, a useful test substitute at a stable seam, an infrastructure boundary, a plugin contract, or independently owned implementations. Keep a single implementation concrete when polymorphism adds no value.                                          |
| Dependencies  | Keep framework, transport, serialization, and database details near their boundaries unless they belong to the domain. Convert infrastructure inputs into domain commands there.                                                                                                         |
| Locality      | Keep behavior that changes together together. Concentrate shared invariants and orchestration; file size and similar syntax alone do not justify splitting or generalizing.                                                                                                              |
| Data flow     | Make inputs, outputs, dependencies, and effects explicit. Separate computation from effects where this improves independent understanding and testability.                                                                                                                               |
| Ownership     | Assign ownership to the module responsible for the value. Reconsider responsibility, lifetimes, and mutation before solving borrow conflicts with clones, sharing, or lifetime machinery. Borrow when ownership is unnecessary; small intentional clones are acceptable.                 |
| Configuration | Prefer explicit domain operations or coherent configuration objects. Many unrelated booleans can indicate several concepts compressed into one API.                                                                                                                                      |
| Public APIs   | Default to private visibility and expose only what callers need. `pub(crate)` is crate-local; workspace siblings need an appropriate `pub` interface. Keep internal helper types and mutable fields private unless they serve a caller contract.                                         |

## Shared State and Async

Choose synchronization from actual access requirements. Consider ownership transfer, immutable sharing, message passing, shorter lifetimes, or split responsibilities before adding shared mutation. An actor is not automatically simpler than a lock.

Keep locking regions small and visible. Never hold blocking locks across `.await`. Treat cancellation, shutdown, backpressure, and task ownership as part of the API contract.

## Errors

Keep failures meaningful at module boundaries: typed errors for library callers that need recovery decisions, and contextual propagation at application boundaries. Expose implementation details only when callers need them.

Use `unwrap()` or `expect()` in production only for a guaranteed, locally obvious invariant. Retain structured failures until callers no longer need to distinguish them.

## Tests

Exercise observable behavior through stable module interfaces. Use unit tests for deterministic logic, integration tests for important interactions, and end-to-end tests for critical system behavior. Mock at meaningful seams; a behavior-preserving refactor should rarely require rewriting tests.

| Concrete risk or verification need                                   | Technique to consider                  |
| -------------------------------------------------------------------- | -------------------------------------- |
| Parser, transform, algorithm, or domain invariant across many inputs | Property testing                       |
| Hostile or structurally complex input                                | Fuzzing                                |
| Subtle concurrency interleavings                                     | `loom`                                 |
| Unsafe or memory-sensitive behavior                                  | Miri where supported                   |
| Large stable structured output suited to diff review                 | Snapshot tests with intentional review |
| Performance-sensitive paths or claimed improvements                  | Benchmarks on the relevant workload    |

Use specialized tools only when they address a concrete failure mode.

## Unsafe and FFI

Prefer a safe design when practical. Unsafe code must remain small, isolated behind appropriate interfaces, and supported by an explicit safety argument.

| Construct                              | Required argument                                                               |
| -------------------------------------- | ------------------------------------------------------------------------------- |
| `unsafe` block                         | A `SAFETY` comment explaining why the operation's preconditions hold here       |
| `unsafe impl`, including `Send`/`Sync` | Proof of the trait contract, including ownership and synchronization invariants |
| Public `unsafe fn` or unsafe trait     | A `# Safety` contract describing caller or implementer obligations              |

For example, constructing a slice from a raw pointer requires validity, alignment, initialization, length, lifetime, and aliasing guarantees; non-nullness alone is insufficient.

At FFI boundaries, verify layout, ownership transfer, matching deallocation, callback lifetimes, and permitted unwinding. A safe wrapper must uphold hidden obligations for every permitted caller. Miri results supplement the safety argument rather than proving soundness for every caller.

## Performance

Before complicating a design, identify a relevant workload, measure it, locate the bottleneck, make the smallest useful change, and measure again. Prefer algorithmic and architectural improvements before micro-optimizations.

Zero-copy, pooling, unsafe code, custom allocation, lock-free structures, and advanced generics need evidence supporting their added complexity. Report measurements for performance claims.

## Dependency Selection

Minimize code and behavior to maintain, rather than dependency count alone. Apply these choices in order:

| Priority | Choice                              | Decision                                                                                                               |
| -------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 1        | Explicit user or repository choices | Honor the chosen technology and constraints within correctness, safety, and compatibility requirements                 |
| 2        | Existing implementations or crates  | Reuse suitable code at reasonable maintenance cost; preserve compatible alternatives to this skill's defaults          |
| 3        | Standard library                    | Use std when it directly covers the required semantics without substantial custom support code                         |
| 4        | Task-specific crate defaults        | Use [Crate defaults](references/crate-defaults.md) for otherwise unconstrained choices instead of hand-writing support |

Concrete constraints favoring another choice include MSRV, `no_std`, target support, license restrictions, or disproportionate build/binary cost; explain the reason briefly. A difference from the default alone is not a reason to migrate an existing dependency or a review finding.

Before adding a dependency, check workspace versions, current crate documentation and maintenance/advisories, required features, MSRV, supported targets, and repository policy. Select a compatible version at implementation time; test/benchmark-only crates belong in dev dependencies.

Keep dependencies behind meaningful interfaces when that reduces coupling; call them directly when a forwarding wrapper adds no value. Report the chosen crate and the maintained implementation it replaces in one sentence.

## Refactoring

Preserve behavior unless the requested task changes it. Include external contracts beyond Rust public APIs: serialized fields, stored record keys, wire formats, and required missing/unknown-field diagnostics. Retain adapters or mappings that express genuine format differences; removing them is useful only when their contracts remain enforced. Favor small structural improvements that reduce caller concepts, strengthen locality and ownership, or remove duplicated invariants.

When an external contract requires lint-disfavored names or structure, follow repository lint policy and prefer the narrowest justified exception with a reason. Read [Compatibility examples](references/design-examples.md#compatibility-and-lint-exceptions) when refactoring externally named fields, adapters, or lint exceptions.

Judge a refactor by easier future changes, rather than line count, file count, pattern usage, or trait usage. Generalize shared semantics once they are clear; similar syntax alone may be cheaper to duplicate.
