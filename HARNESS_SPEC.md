# GigaOS — Harness Spec
Version: 0.1 (North Star)
Date: 2026-06-19

This document is the authoritative specification for the GigaOS harness bus.
It describes what the bus must be — not what it currently is.
Implementation catches up to this spec milestone by milestone.

---

## What the Harness Bus Is

The harness bus is the single dispatch layer between AI inference and real system resources.
Every operation that touches a durable resource — filesystem, settings, network, search —
goes through the bus. Nothing reaches through it.

The bus enforces three properties:
1. **Type safety** — requests and responses are typed end to end. Wrong types fail at compile time.
2. **Capability gating** — callers hold typed capability tokens. No token, no dispatch.
3. **Determinism** — harness implementations are always real, never hallucinated.
   The AI interprets fuzzy intent and selects a harness call. The harness executes deterministically.

---

## Bus Protocol

### Request and Response

```rust
pub struct Request<T: HarnessOp> {
    pub op: T,
    pub capability: T::Capability,
    pub context: RequestContext,
}

pub struct Response<T: HarnessOp> {
    pub result: T::Output,
    pub metadata: ResponseMetadata,
}

pub trait HarnessOp: Send + Sync {
    type Output: Send + Sync;
    type Capability: Capability;
}
```

Every harness operation is a type that implements `HarnessOp`.
The bus routes on the type. Dispatching an unregistered type is a compile-time error.

### Dispatch

```rust
pub trait HarnessBus: Send + Sync {
    fn dispatch<T: HarnessOp>(
        &self,
        request: Request<T>,
    ) -> impl Future<Output = Result<Response<T>, BusError>> + Send;
}
```

The AI service calls `dispatch`. The bus checks the capability token, routes to the registered
harness, and returns the response. The AI service never calls harness methods directly.

### Error Model

```rust
pub enum BusError {
    CapabilityDenied { op: &'static str },
    HarnessNotRegistered { op: &'static str },
    HarnessError(Box<dyn std::error::Error + Send + Sync>),
    Timeout,
}
```

`CapabilityDenied` is a type-level error in the ideal model. In M1/M2, it surfaces at runtime
as the capability enforcement is implemented. The goal (Type Discipline guardrail) is to make
`CapabilityDenied` unreachable at runtime by making it a compile-time constraint.

---

## AI Service Slot

The AI service is a typed, swappable slot. The bus is not bound to any specific AI backend.
A bare install boots with `NullAiService` — all deterministic harness operations work, the
NL terminal degrades gracefully, nothing depends on inference being available.

### Trait

```rust
pub trait AiService: Send + Sync {
    fn infer(
        &self,
        prompt: Prompt,
        budget: TokenBudget,
    ) -> impl Future<Output = AiResult> + Send;
}

pub struct Prompt {
    pub system: Option<String>,
    pub messages: Vec<Message>,
    pub tools: Vec<ToolSpec>,
}

pub struct TokenBudget {
    pub max_input: u32,
    pub max_output: u32,
}

pub enum AiResult {
    Message(AiMessage),
    ToolCall(ToolCall),
    Error(AiError),
}
```

### Backends

```rust
pub enum AiBackend {
    Null,
    ClaudeApi(ClaudeApiService),
    Omlx(OmlxService),
    Local(LocalInferenceService),
}
```

Swapping backends is a substitution at the slot. No harness code changes. No config flag.
`NullAiService` responds to every inference request with a graceful degradation message.
`ClaudeApiService` is the M1 backend. `OmlxService` and `LocalInferenceService` are M4.

### Two-Tier Routing (M4+)

Fast-path operations (intent classification, short completions, routine dispatch) route to
the local model. Complex operations (multi-step reasoning, ambiguous intent, planning) route
to Claude API.

The routing decision is made by the AI service tier, not by the caller.
Callers see a single `AiService` trait; routing is internal to the backend.

---

## Capabilities

Capabilities are typed tokens. A caller that does not hold the required capability cannot
dispatch the corresponding request — not because of a runtime check, but because the type
system rejects the call.

### Capability Trait

```rust
pub trait Capability: Send + Sync + 'static {
    fn name() -> &'static str;
}
```

### Defined Capabilities (M1/M2 scope)

```rust
pub struct StorageReadCapability;
pub struct StorageWriteCapability;
pub struct SettingsReadCapability;
pub struct SettingsWriteCapability;
```

Capability tokens are granted at session start from a policy. They cannot be escalated
at runtime. The AI service holds only the capabilities granted to it for the current session.

**Principle: least privilege per harness.**
A "read notes" operation holds `StorageReadCapability`. It cannot write, cannot spawn
processes, cannot make network calls — not because of runtime guards, but because it does
not hold those capability types.

### Capability Grant (M2)

```rust
pub struct SessionPolicy {
    pub granted: Vec<Box<dyn Capability>>,
}
```

The boot manifest declares the capabilities granted at startup. The AI service cannot
expand its granted set. Capability grant is immutable once the session starts.

---

## Deterministic / Hallucinated Boundary

This is the most important invariant in GigaOS and is enforced at the type level.

### Durable Primitives — always deterministic

These are never hallucinated, never generated, always real:
- File contents, directory listings, file metadata
- User settings and configuration
- Service state, logs
- Harness implementations themselves

```rust
pub struct DurableFile {
    path: PathBuf,        // real path on the real filesystem
    content: Vec<u8>,     // real content from storage harness
}
```

### Ephemeral Surface — may be generated

These are AI-generated and live only in the session:
- UI layout: the arrangement of elements in a file manager view
- Generated "apps": task boards, budget views, project overviews
- Visual scaffolding: the chrome around deterministic content

```rust
pub struct EphemeralView {
    layout: GeneratedLayout,   // AI-generated, session-only
    data: Vec<DurableRef>,     // references into durable primitives
}
```

The `EphemeralView` wraps durable references. The layout is generated; the data is real.
When a view is rendered, data comes from the harness; layout comes from inference.

### Persistence Model

A user can save a generated surface. This promotes it from ephemeral to durable.

```
EphemeralView  →  [save command]  →  DurableTemplate (stored in StorageHarness)
                                           ↓
                                      [load]  →  Deterministic render (no re-generation)
```

```rust
pub struct DurableTemplate {
    id: TemplateId,
    version: u32,
    layout: SerializedLayout,   // frozen, stored in storage harness
    bindings: Vec<HarnessBinding>,  // which harness ops populate this template
}
```

Once saved, the template is a real durable resource. It is loaded deterministically.
It is not re-hallucinated on next session. It obeys versioning and can be updated explicitly.

**The rule:** AI interprets fuzzy intent → selects a harness call → harness executes
deterministically → result flows back. AI never calls a harness directly.
AI never writes to storage without a `Request<StorageOp>` going through the bus.

---

## Harness Registry

At boot, the bus reads the boot manifest and registers the declared harnesses.
Harnesses not in the manifest are unavailable for dispatch.

```rust
pub trait Harness: Send + Sync {
    fn name(&self) -> &'static str;
    fn dispatch_raw(
        &self,
        op: &dyn Any,
        ctx: &RequestContext,
    ) -> BoxFuture<Result<Box<dyn Any + Send>, HarnessError>>;
}
```

The typed `dispatch<T>` method on the bus downcasts to the concrete harness and op type.
The `dispatch_raw` method is the harness's internal contract with the registry.

### Boot Manifest (M2)

```toml
[harnesses]
storage = { crate = "storage-harness", capabilities = ["StorageRead", "StorageWrite"] }
settings = { crate = "settings-harness", capabilities = ["SettingsRead", "SettingsWrite"] }
```

---

## Defined Harnesses

### StorageHarness (M1)

Minimal ops for M1:

```rust
pub enum StorageOp {
    ReadDir { path: PathBuf },
    ReadFile { path: PathBuf },
}

impl HarnessOp for StorageOp {
    type Output = StorageResult;
    type Capability = StorageReadCapability;
}

pub enum StorageResult {
    DirListing(Vec<DirEntry>),
    FileContent(Vec<u8>),
    Error(StorageError),
}
```

Full ops (M3):

```
read_dir, read_file, write_file, create_file, delete_file, move_file, list, stat, versioning
```

### SettingsHarness (M2)

```
read_setting, write_setting, list_settings, reset_to_default
```

Settings are stored as structured key-value pairs, versioned, persisted to disk.

### SearchHarness (M5)

```
full_text_search, search_by_tag, search_by_date_range
```

Search operates over the contents of the storage harness. It does not reach outside it.

### NetworkHarness (M5+)

```
http_get, http_post, dns_resolve
```

Network calls go through the bus. The AI service cannot make network calls directly.

---

## NL Terminal Modes (M1)

The NL terminal has two primitive modes. These are irreducible (Minimality guardrail):
neither can be expressed in terms of the other.

| Mode | What it does | Harness call | Footprint |
|---|---|---|---|
| `think` | Stateless reasoning / hallucination | None | Session buffer only |
| `dispatch` | Deterministic harness execution | Required | Depends on harness op |

`remember` and `forget` are **not** independent primitives — they are `dispatch` operations
against the ContextService harness (`ContextOp::Save` and `ContextOp::Exclude`). Elevating
them to primitive status would privilege memory effects over filesystem effects without
justification, violating Minimality. User-facing `remember`/`forget` keywords in the NL
terminal are syntactic sugar that the parser resolves to the appropriate `dispatch` call.

### `think` — Stateless Inference

```rust
pub struct ThinkRequest {
    pub prompt: String,
    pub session_context: SessionContext,
}

pub struct ThinkResponse {
    pub content: String,
    // No TemplateId — nothing persisted, no harness call made
}
```

Stateless. No harness call, no template, no generation. Output lives in the session buffer only.
The session buffer is NOT written to the storage harness. When the session ends, `think` output
is gone — full hallucination, zero footprint.

**Critical boundary:** deterministic lookup (`recall`) must NOT go through `think`.
`"What did I save as X?"` is a `dispatch(ContextOp::Get { name })` — a harness call returning
real stored content. Routing it through `think` would put a hallucinating model in front of a
deterministic lookup, violating the quality rule. The NL parser must route intent-to-recall
directly to dispatch, never through think.

### `dispatch` — Deterministic Harness Execution

All world-effect operations go through `dispatch`: filesystem, settings, services, and
ContextService operations (save/exclude/get). The same bus protocol applies to all:

```rust
pub trait HarnessBus: Send + Sync {
    fn dispatch<T: HarnessOp>(
        &self,
        request: Request<T>,
    ) -> impl Future<Output = Result<Response<T>, BusError>> + Send;
}
```

The NL terminal calls `think` to interpret fuzzy intent into a typed `HarnessOp`, then calls
`dispatch` to execute it deterministically. These two steps are always sequential: generate
intent, then execute. Never hallucinate the execution itself.

### Full Persistence State Machine

The persistence operations (`remember`, `forget`, `gc`) are `dispatch` calls to the ContextService harness.

```
                         [think]
                            ↓
                   session buffer only
                   (no harness call, no generation)

EphemeralView  →  dispatch(ContextOp::Save)   →  DurableTemplate (stored, new generation)
                                                        ↓
                                               dispatch(ContextOp::Exclude)
                                                        ↓
                                               New generation (template excluded)
                                               Prior generations intact → rollback free
                                                        ↓
                                               dispatch(ContextOp::Gc)
                                                        ↓
                                               Permanent deletion (irreversible)
```

**Rollback addressing rule:** Rollback is always name-addressed (`restore template X v3`),
never position-addressed (`roll back to generation 47`). Exposing generation indices as a
user-facing primitive would surface the non-confluent history ordering — a Confluence violation
at the interface level.

---

## Context Service Slot (M2)

The ContextService harness handles all persistent context: session memory, semantic recall,
template storage. It is a typed slot — the NL terminal sees a single `ContextService` trait.

**Design constraint (Minimality + Confluence):** A single SQLite backend is required.
Two stores (Qdrant + SQLite) would fail both tests: (1) Qdrant's vector capability is
composable from SQLite + sqlite-vec, so it fails Minimality; (2) dual-write for every
`remember`/`forget` creates a distributed consistency problem — atomic in one store, orphaned
in the other, a latent Confluence violation inside the terminal primitives. A single SQLite
file makes `remember`/`forget` single atomic transactions.

```rust
pub trait ContextService: Send + Sync {
    fn load_context(
        &self,
        query: &str,
        budget: ContextBudget,
    ) -> impl Future<Output = ContextResult> + Send;

    fn save(&self, name: FourSegmentName, view: EphemeralView) -> impl Future<Output = TemplateId> + Send;
    fn exclude(&self, name: FourSegmentName) -> impl Future<Output = GenerationId> + Send;
    fn gc(&self, keep_generations: u32) -> impl Future<Output = ()> + Send;
    fn get(&self, name: FourSegmentName) -> impl Future<Output = Option<DurableTemplate>> + Send;
}

pub struct ContextBudget {
    pub max_tokens: u32,
    pub max_results: u32,
}

pub struct ContextResult {
    pub entries: Vec<ContextEntry>,
    pub total_tokens: u32,
}

pub enum ContextBackend {
    Null,
    Sqlite(SqliteContextService),  // FTS5 (lexical) + sqlite-vec (vector) — single file, no daemon
    Custom(Box<dyn ContextService>),
}
```

**SQLite backend:** FTS5 provides lexical full-text search; sqlite-vec provides dense vector
search (semantic recall). Both run in the same file, same transaction, no daemon. Single
portable file. `Custom` is the extension point for future backends (e.g., remote sync).
Qdrant is not in scope — re-evaluate only with a measured QPS argument at appliance scale.

The backend is configured in the NixOS appliance config:
```nix
services.gigaos.contextBackend = "sqlite";
```

---

## Observability: Langfuse Tracing (M1)

Every inference request passing through the AI service slot is traced to Langfuse.
This is an M1 deliverable — not optional, not M4. The bus must be observable from day one.

```rust
pub struct LangfuseTracer {
    client: LangfuseClient,
    project_id: String,
}

impl LangfuseTracer {
    pub fn trace_inference(&self, req: &Prompt, result: &AiResult, latency_ms: u64) -> TraceId;
    pub fn trace_bus_dispatch(&self, op: &str, capability: &str, latency_ms: u64) -> SpanId;
}
```

The tracer is injected into the `AiService` slot at construction. Every `infer()` call emits:
- Input prompt (system + messages)
- Output (message or tool call)
- Token counts (input / output)
- Latency

Every bus dispatch emits:
- Harness op type
- Capability token used
- Result (success / error)
- Latency

This gives an **observable reduction history**: every intent → dispatch → result chain is
recorded. Debugging a misbehaving session means reading the Langfuse trace, not adding print
statements. The trace is the source of truth for what the system actually did.

---

## AI Reliability Design (M4)

### Job Queue

All inference requests are enqueued. No fire-and-forget. The caller gets a `JobHandle`
and polls or awaits the result.

```rust
pub struct JobHandle {
    id: JobId,
    rx: oneshot::Receiver<AiResult>,
}
```

Two concurrent inference requests are queued and resolved in order.
The queue is bounded; backpressure propagates to the caller.

### Prompt-Cache Breakpoints

Long prompts are segmented at stable prefix boundaries — system prompt, context preamble,
prior turn summary. Each segment is a cache point. Token cost for repeated prefixes collapses.

Cache prefix structure:
```
[SYSTEM PROMPT]       ← never changes within a session; cache forever
[SESSION CONTEXT]     ← changes per session; cache at session start
[TURN CONTEXT]        ← changes per turn; cache at turn boundary
[USER MESSAGE]        ← always fresh; no cache
```

### Streaming with Reconnect

Long inference ops stream tokens to the caller. The stream is checkpointed every N tokens.
If the inference connection drops, the caller can resume from the last checkpoint.

```rust
pub struct StreamCheckpoint {
    job_id: JobId,
    tokens_received: u32,
    partial_content: String,
}
```

### Checkpointing

Long operations (multi-step reasoning, document generation) write intermediate state to
the storage harness at defined checkpoints. If the process restarts, the operation can
resume from the last checkpoint rather than starting over.

---

## Open Questions (tracked here until resolved)

| Question | Blocking | Target |
|---|---|---|
| Bus implementation: actor model (tokio/actix) vs typed function dispatch? | M1 | Before M1 code starts |
| Capability token: zero-cost at compile time or runtime check in M1/M2? | M2 | Before M2 code starts |
| `AiService::infer` return type: `Future` vs `Stream` for streaming? | M4 | Before M4 code starts |
| DurableTemplate serialization format: JSON, bincode, RON? | M3 | Before M3 code starts |
