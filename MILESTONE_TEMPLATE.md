# GigaOS — Milestone Template
Version: 0.1
Date: 2026-06-19

One pull request per milestone. One milestone per scope boundary.
The milestone is complete when all acceptance criteria pass. Nothing else ships.

---

## Template

Copy this block at the start of every milestone. Fill it in before writing any code.

```markdown
# Milestone N — [Topic]

## Scope
What this milestone adds (one sentence):

What this milestone does NOT touch:
- [ ] item
- [ ] item

## Source
Reference: ROADMAP.md § Milestone N
Architecture reference: ARCHITECTURE.md § [section]

## Acceptance Criteria
The milestone is DONE when ALL of these pass:
- [ ] `cargo build` — zero errors, zero warnings (warnings treated as errors)
- [ ] `cargo clippy -- -D warnings` — clean
- [ ] `cargo test` — all harness integration tests pass
- [ ] [specific functional criterion, e.g. "read_dir via bus returns real directory listing"]
- [ ] [bus-path criterion: test instruments the bus, not std::fs directly]

## Lambda Check
Before merging, verify each guardrail:
- [ ] Minimality: No new primitive that can be composed from existing ones
- [ ] Confluence: Concurrent/independent operations commute; shared state is serialized
- [ ] Substitution: State transitions produce new state; side effects at harness boundary only
- [ ] Type discipline: New invariants encoded at type level where possible

## Known Breakage
Things intentionally left incomplete (each item becomes a future milestone's acceptance criterion):
- item: why it's deferred

## Unsafe Inventory
New `unsafe` blocks introduced this milestone:
| Location | SAFETY rationale |
|---|---|
| file.rs:line | reason |

## Capabilities Added
New harness capability tokens introduced:
| Token | Harness | Operations permitted |
|---|---|---|
| `StorageReadCapability` | storage-harness | read_dir, read_file |

## Dependencies Added
```toml
[dependencies]
new-crate = "x.y.z"
```

## Commit Message
```
milestone N: [topic] — [one-line description]

- bullet 1
- bullet 2
- bullet 3
```

## Next Milestone Seed
What milestone N+1 starts with:
- Problem it solves:
- First thing to implement:
- ROADMAP reference:
```

---

## Process Rules

### Before writing any code
1. Copy the template above and fill it in completely.
2. Define acceptance criteria. If you can't define them, the scope is unclear — stop and clarify.
3. Identify lambda guardrail tension points.
4. Verify the bus-path integration test is specced — if the test can pass by calling `std::fs` directly, the test is wrong.

### During the milestone
- One PR. One branch. `git checkout -b milestone-N-[slug]`.
- Commit frequently. Small commits with clear messages.
- If scope expands, stop. Either update the template or defer to next milestone.
- Every new `unsafe` block gets a `// SAFETY:` comment before the PR opens.
- Capability tokens go in the template before the harness code that uses them.

### Acceptance criteria (non-negotiable)
- `cargo build` — zero errors, zero warnings (treat warnings as errors: `#![deny(warnings)]`)
- `cargo clippy -- -D warnings` — clean
- `cargo test` — integration tests pass, including bus-path verification

The bus-path criterion is not optional: every milestone that adds a harness must include an integration
test that asserts the dispatch went `terminal → bus → harness → result`, not `terminal → std::fs → result`.
Mock the bus in unit tests, instrument it in integration tests.

If any criterion fails, the milestone is not done. No exceptions.

### Known breakage log
This is a debt ledger, not a shame list. Every item must become a future milestone's acceptance criterion.
An item without a target milestone is just deferred indefinitely — that is not acceptable. Assign it.

### Merging
Squash merge to main. Commit message: `milestone N: [topic]`.
Tag the commit: `git tag milestone-N`.

---

## Example: Milestone 1 — Prove the Spine

```markdown
# Milestone 1 — Prove the Spine

## Scope
What this milestone adds: An NL terminal that takes "open my notes folder" and dispatches a
real read_dir through the typed harness bus using Claude API as the AI backend.

What this milestone does NOT touch:
- [ ] Local inference / oMLX (Milestone 4)
- [ ] Capability enforcement (Milestone 2)
- [ ] Settings harness (Milestone 2)
- [ ] UI layer (Milestone 3)
- [ ] Second harness (Milestone 2)

## Source
Reference: ROADMAP.md § Milestone 1 — Prove the Spine
Architecture reference: ARCHITECTURE.md § Architecture Layers

## Acceptance Criteria
- [ ] `cargo build` — zero errors, zero warnings
- [ ] `cargo clippy -- -D warnings` — clean
- [ ] `cargo test` — integration test passes
- [ ] Integration test: asserts dispatch path was `terminal → bus → storage-harness → result`,
      NOT `terminal → std::fs → result` (bus must be instrumented, not bypassed)
- [ ] Live demo: `"open my notes folder"` → real directory listing printed to stdout

## Lambda Check
- [x] Minimality: storage-harness and ai-service are irreducible — neither composes from the other
- [x] Confluence: concurrent read_dir on independent paths commute; no shared mutable state
- [x] Substitution: AI result flows into bus Request; harness returns new state to terminal
- [x] Type discipline: `Request<StorageOp>` — typed end to end; wrong request type fails at compile time

## Known Breakage
- Capability enforcement not live (Milestone 2): any caller can dispatch any request
- AiService backend hardcoded to Claude API (Milestone 4): no abstraction yet
- No settings persistence (Milestone 2): state lost between sessions

## Unsafe Inventory
(none expected — userspace Rust with std should require no unsafe in M1)

## Capabilities Added
(none formal — capability enforcement lands in Milestone 2)

## Dependencies Added
```toml
[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anthropic = "0.1"   # or reqwest-based Claude API client
```

## Commit Message
```
milestone 1: prove the spine — NL terminal dispatching through typed harness bus

- harness-bus crate: StorageRequest routing, typed Request<T>/Response<T>
- storage-harness crate: read_dir and read_file against real filesystem
- ai-service crate: AiService trait + ClaudeApiService impl (hardcoded)
- nl-terminal crate: input loop, Claude API intent parse, bus dispatch
- integration test: verifies bus path, not direct std::fs
```

## Next Milestone Seed
- Problem: Bus handles only one harness type; capability enforcement not live
- First thing: Add harness registry, abstract AiService slot, add capability tokens
- ROADMAP reference: ROADMAP.md § Milestone 2 — Bus Generality + Harness Registry
```
