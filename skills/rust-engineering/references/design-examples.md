# Design Examples

Use these examples when implementing or reviewing domain types, module boundaries, seams, ownership, or configuration APIs. Adapt them to existing repository conventions.

## Domain Types

| Situation                                   | Preferred shape                                   | Reason                                        |
| ------------------------------------------- | ------------------------------------------------- | --------------------------------------------- |
| Concepts with different rules or lifecycles | Separate domain types                             | Prevent callers from mixing distinct concepts |
| Closed states                               | `JobState::{Pending, Running, Completed, Failed}` | Avoid contradictory combinations of booleans  |
| Constrained values                          | Validated constructors and controlled mutation    | Keep invariants valid after construction      |
| Expected absence or failure                 | `Option` or `Result`                              | Make caller decisions explicit                |

Prefer names such as `AlertRule`, `DetectionResult`, and `WorkflowExecution` over generic `DataManager` or `ItemProcessor`.

```rust
// Keep user and session identifiers distinct so callers cannot swap them.
struct UserId(Uuid);
struct SessionId(Uuid);
```

A newtype is useful when the distinction matters to callers; a wrapper without added meaning or invariants is ceremony. Validate loosely structured input into domain types at boundaries, and verify allowed state transitions. Use typestate, generics, lifetimes, or macros only when they remove concrete complexity.

## Deep Modules and Seams

An abstraction should hide substantial cohesive behavior behind a small interface:

```text
small interface → substantial cohesive behavior
```

A function, type, module, crate, or subsystem can be a module in this sense. Ask whether deleting the abstraction would spread meaningful complexity across several callers. A forwarding layer that merely renames calls usually fails that test.

| Demonstrated need                                      | Seam decision                                                  |
| ------------------------------------------------------ | -------------------------------------------------------------- |
| One concrete implementation                            | Keep it concrete unless another boundary requires polymorphism |
| Multiple meaningful implementations                    | A trait can express their shared semantics                     |
| External infrastructure or test substitute             | Use a stable seam when it improves isolation                   |
| Plugin contract or independently owned implementations | Expose the contract required by its consumers                  |

```rust
struct SqliteEventStore {
    // Keep persistence concrete until a real seam requires polymorphism.
}
```

Generalize common semantics rather than superficially similar syntax. Duplication can be cheaper than an abstraction requiring unrelated configuration flags.

## Dependency Direction and Locality

```text
transport / persistence / framework
              ↓
         application logic
              ↓
           domain
```

For example, convert an HTTP request into a domain command near the HTTP boundary. Keep database row types near persistence unless the representation is genuinely part of the domain.

| Symptom                                                 | Design question                                      |
| ------------------------------------------------------- | ---------------------------------------------------- |
| Understanding an operation requires visiting many files | Can its cohesive behavior live behind one interface? |
| One rule changes in unrelated modules                   | Where should the shared invariant be owned?          |
| Validation or orchestration repeats at call sites       | Can the responsible module enforce it?               |
| Important invariants exist only in caller conventions   | Can the interface make those obligations explicit?   |

File size alone is not a design verdict.

## Explicit Data Flow

```rust
// Make the rule and event inputs explicit so evaluation is independently testable.
fn evaluate(rule: &Rule, event: &Event) -> DetectionResult
```

A useful shape is input → domain computation → decision/result → effectful adapter. Separate computation from effects where it improves clarity; retain direct effects when forcing separation would obscure the operation.

## Ownership and Shared Mutation

Before adding a clone, `Arc`, `Mutex`, or lifetime machinery for a borrow conflict, identify who owns the value, who mutates it, and how long it needs to live. Mixed responsibilities or mutation at the wrong level may be the underlying problem.

| Ownership need                         | Input shape                                                      |
| -------------------------------------- | ---------------------------------------------------------------- |
| Read text without retaining ownership  | `&str`                                                           |
| Read a collection without consuming it | `&[Item]`                                                        |
| Store or consume text or a collection  | `String` or `Vec<Item>` when that ownership serves the operation |

Small intentional clones are acceptable when they keep the design clear.

For shared mutable access, compare transferring ownership, immutable sharing, message passing, task ownership, split read/write responsibilities, and shorter lifetimes. Choose the simplest correct option for actual access requirements.

## Configuration and Public Interfaces

A call such as `process(event, true, false, true, RetryMode::Aggressive)` hides the meaning of its choices. Prefer explicit domain operations or a coherent configuration object.

Keep fields and helper types private unless callers need them. Expose the invariants, ordering, errors, and configuration callers must understand alongside type signatures.
