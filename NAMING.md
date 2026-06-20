# GigaOS — Axiomatic Naming Scheme
Version: 0.1
Date: 2026-06-20

---

## The Problem This Solves

Hallucinated names are ambiguous by nature. An AI that generates names like `notes`, `view1`,
or `my-template` will produce collisions, context loss, and silent overwrites. The persistence
model (think / remember / forget) depends on names being unambiguous, stable across sessions,
and machine-verifiable — not human-memorable guesses.

The axiomatic naming scheme makes invalid names unrepresentable. A bare-word name cannot be
issued to the bus. The type system rejects it.

---

## The Scheme

Every named resource in GigaOS uses the form:

```
<owner>.<scope>.<purpose>.<identifier>
```

All four segments are required. A name missing any segment is invalid and rejected at parse time.

### Segments

| Segment | Meaning | Examples |
|---|---|---|
| `owner` | Who or what created this resource | `user`, `system`, `ai`, `james` |
| `scope` | The domain or harness context | `storage`, `settings`, `ui`, `session` |
| `purpose` | What the resource is for | `notes`, `dashboard`, `config`, `layout` |
| `identifier` | Unique discriminator within the above | `home`, `2026q2`, `main`, `v3` |

### Examples

```
user.storage.notes.home           # the user's home notes folder template
user.ui.dashboard.project-giga   # a saved project dashboard for GigaOS work
system.settings.config.defaults  # system default configuration
ai.session.context.summary       # AI-generated session context summary
james.ui.layout.file-browser     # James's saved file browser layout
```

### Invalid (rejected by the bus)

```
notes          # bare word — no owner, no scope, no purpose, no identifier
my-template    # no segments
view1          # no segments
user.notes     # only two segments
```

---

## Canonical IDs

Human-readable names are for user display only. The canonical identity of a resource is its
content-addressed hash:

```rust
pub struct TemplateId(blake3::Hash);
```

The `TemplateId` is derived from the content of the template at time of creation. Identical
content produces the same `TemplateId`. This is the "real" name — the four-segment name is
a display alias that maps to a `TemplateId` in the naming registry.

Benefits:
- Rename is free: change the display alias, the `TemplateId` is unchanged.
- Deduplication is automatic: saving the same layout twice produces the same `TemplateId`.
- Content integrity: loading by `TemplateId` is content-verifiable.

---

## The Naming Registry

The registry maps four-segment names to `TemplateId`s. It lives in the storage harness.

```rust
pub struct NamingRegistry {
    entries: HashMap<FourSegmentName, TemplateId>,
}

pub struct FourSegmentName {
    owner: String,
    scope: String,
    purpose: String,
    identifier: String,
}

impl FourSegmentName {
    pub fn parse(s: &str) -> Result<Self, NamingError>;
    pub fn canonical(&self) -> String;   // "owner.scope.purpose.identifier"
}
```

The registry enforces:
- Four-segment structure at parse time.
- No bare words.
- Collision detection: registering a name that maps to a different `TemplateId` requires
  explicit overwrite confirmation (a separate `RememberWithOverwrite` bus op).

---

## NL Terminal Integration

The NL terminal parser maps natural language to four-segment names. The AI generates the
name proposal; the bus validates it.

```
User: "save this as my notes layout"
AI generates: "user.ui.layout.notes"
Bus validates: FourSegmentName::parse("user.ui.layout.notes") → Ok
RememberRequest { name: "user.ui.layout.notes", view: ... } → dispatched
```

If the AI generates an invalid name (bare word, wrong segment count), the bus returns
`NamingError::InvalidFormat` and the terminal asks the user to clarify or confirm a
generated valid name.

The AI never writes to the registry directly. The name goes through the bus.

---

## Lambda Guardrail Alignment

| Guardrail | Application |
|---|---|
| Minimality | Four segments is the irreducible form — no segment can be omitted without losing information |
| Confluence | Two concurrent `remember` calls with different names commute; same name requires serialization |
| Substitution Semantics | Renaming is a registry substitution (old name → new mapping); content unchanged |
| Type Discipline | `FourSegmentName` is a distinct type; `&str` cannot be passed where a name is expected |
