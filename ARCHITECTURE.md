# GigaOS — Architecture
Version: 0.2 (userspace harness on Linux base)
Date: 2026-06-19

---

## Premise

GigaOS is an AI-native OS personality running on a Linux base. It is defined by its harness bus —
a typed, capability-gated dispatch layer written in Rust that sits between AI inference and real
system resources. The substrate (network stack, filesystem, GPU drivers, device support) is Linux's.
The OS's identity — the harness model, the AI service layer, the NL terminal, the lambda discipline
— is entirely ours.

This is the same shape as ChromeOS, SteamOS, and Android: a recognizable personality on top of a
Linux base, where the personality is the product and the base is infrastructure.

---

## The Four Guardrails

Every architectural decision is evaluated against these four tests.
If a design cannot be justified against all four, it does not ship.
These guardrails were designed for a kernel. They are equally valid in userspace.

### 1. Minimality
> Every harness primitive must be irreducible — it cannot be expressed in terms of primitives already present.

Lambda: in λ-calculus, there are no redundant terms. Every abstraction does work the rest of the
calculus cannot. Apply the same test to harness components.

**In practice:**
- Before adding a new harness, prove it cannot be composed from existing ones.
- No convenience wrappers at the bus level. Wrappers belong above the harness.
- The set of harness primitives forms a minimal basis.

**Examples:**
- StorageHarness: minimal. There is no lower-level file access in the harness model.
- A "readFileAsString" shortcut: not minimal. Composes StorageHarness + encoding.
- AiService: minimal. There is no lower-level intent-to-action path.

---

### 2. Confluence (Church-Rosser Property)
> Independent harness operations must converge to the same state regardless of execution order.

Lambda: if term M can reduce to both N₁ and N₂, there exists N₃ reachable from both.
GigaOS state must have the same property: independent bus operations commute.

**In practice:**
- No shared mutable state without synchronization.
- Two concurrent storage reads on independent files: independent, commute, no lock needed.
- Two concurrent writes to the same file: NOT independent. Exclusive access required.
- AI inference requests: stateless — two concurrent inference calls commute.

**Design test:** For any two bus operations O₁ and O₂: "does order matter?"
- No → independent state → no synchronization needed.
- Yes → shared state → serialize or lock.

---

### 3. Substitution Semantics
> Harness state transitions are substitutions, not side-effectful mutations.

Lambda: β-reduction replaces a bound variable with a value — the term is substituted, not mutated.
GigaOS treats state transitions the same way: produce new state, don't silently mutate old state.

**In practice:**
- Prefer returning new state over mutating state in place.
- Use type-state pattern: old state is consumed by the transition.
- Side effects (actual FS writes, network calls) happen at the harness boundary, not scattered through logic.

**Example: wrong (mutation scattered)**
```rust
self.settings.theme = new_theme;          // side effect
self.settings.dirty = true;               // side effect
if auto_save { self.write_to_disk(); }    // side effect
```

**Example: right (substitution-style)**
```rust
let new_settings = self.settings.with_theme(new_theme); // compute new state
new_settings.flush(storage_harness)?;                    // one side effect at boundary
self.settings = new_settings;                            // one substitution
```

---

### 4. Type Discipline (System F → Rust)
> GigaOS invariants are encoded at the type level, not enforced at runtime.

Lambda: System F prevents ill-typed terms from existing. Use Rust's type system — derived from
System F — to make invalid states unrepresentable.

**In practice:**
- Capability tokens: only a process holding `StorageCapability` can call the storage harness.
- AI service slot: `AiService` is a trait with typed backends — null/Claude/oMLX/local.
  Swapping backends is a substitution, not a config flag.
- Harness request/response: `Request<T>` → `Response<T>` — type-checked end to end.
- Ephemeral vs durable UI: separate types, not a runtime flag.

**The goal:** If it compiles, invariants hold. No `if debug { assert_invariant(); }`.

---

## Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│                  NL Terminal                             │  ← User-facing: fuzzy intent in
│  Intent parser → AI Service → Bus dispatch               │    deterministic action out
├─────────────────────────────────────────────────────────┤
│                  UI Layer                                │  ← Hallucinated layout on top of
│  EphemeralView | DurableTemplate (saved)                 │    deterministic primitives
├─────────────────────────────────────────────────────────┤
│                  Harness Bus                             │  ← Typed message passing
│  Request<T> → capability check → route → Response<T>    │    Capability enforcement
├─────────────────────────────────────────────────────────┤
│    Storage  │   AI Service  │  Search  │  Network  │ …  │  ← Harnesses: deterministic,
│   Harness   │     Slot      │ Harness  │  Harness  │    │    real implementations
├─────────────────────────────────────────────────────────┤
│                  Linux Base                              │  ← Substrate: network, filesystem,
│  Filesystem │ Network stack │ GPU drivers │ Drivers      │    GPU, hardware — not hand-written
└─────────────────────────────────────────────────────────┘
```

Each layer exposes a typed API. No layer reaches through its neighbor to touch a lower layer directly.
The AI service handles fuzzy intent. Harnesses handle deterministic execution. Linux handles the substrate.

---

## The Deterministic / Hallucinated Boundary

This is the most important design principle in GigaOS and must be decided at the type level, not
enforced at runtime.

**Durable primitives — always real, always deterministic:**
- Files, directories, file content
- User settings, configuration
- Service state, logs
- Harness implementations

**Bespoke / ephemeral surface — may be hallucinated:**
- UI layout: the arrangement of elements in a generated file manager
- Generated "apps": a task board, a budget view, a project overview
- Visual scaffolding: the chrome around deterministic content

**The rule:** AI interprets fuzzy intent and selects a harness call. The harness executes
deterministically against real Linux resources. The result flows back through the bus.
AI never executes a filesystem call directly. AI never writes to storage without going through
the storage harness. The harness is the boundary where fuzziness ends and correctness begins.

**Persistence model:**
Hallucinated UI starts ephemeral — regenerated fresh each session, no state saved.
A user can promote a generated surface to durable (`remember`), or remove it from the active
generation (`forget`). `gc` is the only truly irreversible operation.

State transitions:
```
                            [think]
                               ↓
                      session buffer only
                      (no footprint, stateless)

Ephemeral UI  →  [remember]  →  DurableTemplate (stored in StorageHarness)
                                      ↓
                                 [load]  →  Deterministic render (not re-hallucinated)
                                      ↓
                               [forget]  →  New generation (template excluded)
                                                  ↓
                                             [gc]  →  Permanent deletion (irreversible)
```

- `think` — stateless. Output lives in the session buffer only. No template created, no generation produced.
- `remember` — promotes EphemeralView to DurableTemplate. Stored in the storage harness. Versioned.
- `forget` — creates a new NixOS-style generation that excludes the named template. The template still
  exists in prior generations. Rollback is free until `gc` runs.
- `gc` — garbage-collects unreferenced generations. The only operation that permanently deletes templates.

Once saved via `remember`, the template is a real durable primitive. It obeys the same rules as any
other durable resource: content-addressed, versioned, deterministic on load.

---

## The AI Service Slot

The AI service is a typed, swappable slot. A bare install boots with `NullAiService` — the harness
bus works, capability enforcement works, all deterministic operations work. Nothing depends on
inference being available.

```rust
pub trait AiService: Send + Sync {
    fn infer(&self, prompt: Prompt, budget: TokenBudget) -> AiResult;
}

pub enum AiBackend {
    Null,
    ClaudeApi(ClaudeApiService),
    Omlx(OmlxService),
    Local(LocalInferenceService),
}
```

**Two-tier inference model:**
- Base tier: Claude API + a small local model for fast-path operations (intent classification,
  short completions, routine dispatch)
- Supercharged tier (Giga²OS): large local model on a GPU box — same harness interface, different
  backend substituted at boot

Swapping tiers is a substitution at the AiService slot. No harness code changes.

---

## Capabilities

The harness bus is the security boundary. Capability enforcement is type-level, not runtime policy.

Each harness call requires a typed capability token. The bus only grants the call if the
caller holds the required capability. The AI service can only dispatch to harnesses within
its granted capability set. Capabilities are granted at session start from a policy; they
cannot be escalated at runtime.

Principle: least-privilege per harness. A "read notes" operation holds `StorageReadCapability`.
It cannot write, cannot spawn processes, cannot make network calls — not because of runtime
checks but because it does not hold those capability types.

This is the type-discipline guardrail applied to security. Invalid capability combinations
are unrepresentable, not just forbidden.

---

## NL Terminal Modes

The NL terminal has three primitive interaction modes. These are irreducible (Minimality guardrail):
none can be expressed in terms of the others.

| Mode | Primitive | Footprint | Reversible |
|---|---|---|---|
| `think` | Stateless reasoning, exploration, hallucination | Session buffer only; no template, no generation | N/A — nothing persisted |
| `remember` | Promote EphemeralView → DurableTemplate | Stored in storage harness; new generation | Yes — `forget` reverses it |
| `forget` | Exclude DurableTemplate from active generation | New generation produced; prior generations intact | Yes — roll back to prior generation |

**`gc`** is not a terminal mode — it is a maintenance operation that permanently deletes unreferenced
generations. It is the only irreversible act in the persistence model.

The terminal presents these as natural language operations, not explicit commands. The NL parser
maps user intent to one of the three modes plus optional target context.

---

## Deployment Modes

GigaOS ships in four hardware tiers. All tiers share the same harness bus, same NL terminal,
same capability model, same NixOS base. The only difference is which AiService backend is wired
into the slot at boot.

| Tier | Name | AI Backend | Hardware |
|---|---|---|---|
| 0 | GigaOS | Claude API (remote) + CPU fast-path | Any machine; no GPU required |
| 1 | GigaOS+ | Ollama 14B local | Mid-range machine; 16GB+ RAM |
| 2 | Giga²OS | vLLM 70B | Dedicated GPU box; NVIDIA A100/3090 class |
| 3 | Giga³OS | Multi-GPU tensor-parallel | Multi-GPU rack; enterprise inference |

Upgrading from Tier 0 to Tier 1+ is a backend substitution at the `AiService` slot.
No harness code changes. No capability model changes. No NL terminal changes.
The appliance config (`services.gigaos.aiBackend`) selects the tier.

---

## Language and Stack

| Choice | Rationale |
|---|---|
| Rust (stable) | Memory safety, ownership, trait-based polymorphism for harness interfaces |
| `std` | We are in userspace on Linux — no `no_std` constraints here |
| Linux base | Driver coverage = portability; network/FS/GPU already exist |
| Cargo workspace | One workspace, multiple crates (`harness-bus`, `ai-service`, `storage-harness`, etc.) |
| Claude API (M1) | Fastest path to a working AI backend — no local inference setup needed |
| Ollama / llama.cpp (M4+) | Local inference backend for oMLX slot and Supercharged tier |

---

## The Unsafe Contract

All `unsafe` blocks must:
1. Have a `// SAFETY:` comment explaining the invariant being upheld.
2. Be as small as possible.
3. Be justified by a spec or Rust reference.
4. Be reviewed on every PR.

In userspace Rust, `unsafe` should be rare. The goal is for most harness code to contain none.
