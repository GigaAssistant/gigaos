# GigaOS — Roadmap
Version: 0.2
Date: 2026-06-19

Build order: **top-down, not bottom-up.**
Prove the experience first. Appliance image is the final packaging step.

---

## Build Order Principle

The old plan built bottom-up: boot → kernel → memory → network → AI → UI. That front-loads
all the invisible infrastructure and puts the thing you care about last.

GigaOS builds top-down: prove the NL terminal dispatching through the harness on week one.
Every subsequent milestone extends downward (more harnesses, more capabilities) and outward
(more surfaces, richer UI). The appliance image is last — once the experience works on a
Linux VM, bootable packaging is a known problem (immutable image + installer), not a blocker.

---

## Milestone 1 — Prove the Spine
**Estimate:** 2-4 weekends  
**Target platform:** Mac mini or Linux VM (GigaNixOS at 192.168.1.148 works)

**Goal:** An NL terminal that takes `"open my notes folder"` and dispatches a real filesystem
call through the typed harness. Not a mock. Not `std::fs` called directly. The path must go
through the bus.

**What this proves:**
- The harness bus routes correctly
- The AI service (Claude API, hardcoded) interprets fuzzy intent
- The storage harness makes a deterministic `read_dir` against the real filesystem
- The quality rule holds: AI handles the fuzzy part, harness handles the deterministic part

**Deliverables:**
- `crates/harness-bus` — bus with a single message type: `StorageRequest`
- `crates/storage-harness` — storage harness with `read_dir` and `read_file` operations
- `crates/ai-service` — `AiService` trait + `ClaudeApiService` impl (hardcoded in M1)
- `crates/nl-terminal` — minimal NL input loop: read line → Claude API → bus dispatch → print result
- Integration test: assert dispatch path went `terminal → bus → storage-harness → result`
  (NOT `terminal → std::fs → result` — the test must instrument the bus)

**Acceptance criteria:**
- [ ] `cargo build` — zero errors, zero warnings
- [ ] `cargo test` — integration test passes, bus path verified end-to-end
- [ ] Live demo: `"open my notes folder"` → real directory listing printed
- [ ] Clippy passes

**M1 AI backend: Claude API hardcoded.** Local inference setup is M4. If "AI service" means
"stand up local Qwen," the estimate explodes. Pin the backend to Claude API until the bus
abstraction is proven.

**Lambda check:**
- Minimality: storage harness and AI service are irreducible — neither can be composed from the other
- Confluence: concurrent read operations on independent paths commute
- Substitution: AI service result flows into bus request; harness returns new state
- Type discipline: `Request<StorageOp>` cannot be dispatched without `StorageCapability` token

---

## Milestone 2 — Bus Generality + Harness Registry
**Estimate:** 2-4 weekends

**Goal:** The bus handles multiple harness types. The AI service backend is abstracted behind
the typed slot (not hardcoded). Capability enforcement is live.

**Deliverables:**
- Harness registry: dynamic harness registration at boot
- `AiService` trait abstracted — `ClaudeApiService` is one impl, not the only path
- Typed capability tokens per harness (see ARCHITECTURE.md)
- Second harness: settings (`crates/settings-harness`) — read/write user config
- Boot manifest: declares which harnesses load at startup
- Hot-swap stub: AiService can be replaced without restart (stub OK, full impl is M4)

**Acceptance criteria:**
- [ ] Two harnesses running and routed correctly by bus
- [ ] Capability enforcement: test that a request without the required token is rejected at compile time
- [ ] Settings harness: `"set my theme to dark"` → settings write → persisted to disk
- [ ] `cargo test` — capability tests + integration tests pass

---

## Milestone 3 — Storage Harness Full + UI Foundation
**Estimate:** 4-6 weekends

**Goal:** Storage harness is production-grade (files, settings persistence, error handling).
First hallucinated UI surface built on top of deterministic FS operations.

**Deliverables:**
- Storage harness: create, read, write, delete, list, move, versioning stub
- Settings persistence: user config stored and loaded across sessions
- First generated surface: file manager layout hallucinated over `read_dir` results
- Persistence model implemented: ephemeral UI → save → DurableTemplate
  (saved template stored in storage harness, loaded deterministically on next session)
- UI framework decision: native (egui/iced/tauri) or rendered text (TUI)

**Acceptance criteria:**
- [ ] File operations work and survive process restart (durability test)
- [ ] Generated file manager renders directory contents correctly
- [ ] Saving a generated surface: it loads identically on next session (deterministic, not re-hallucinated)
- [ ] `cargo test` — storage harness has full unit test coverage

---

## Milestone 4 — Local Inference + Two-Tier Model
**Estimate:** 4-6 weekends

**Goal:** oMLX / Ollama wired into the `AiService` slot. Fast-path operations use local model.
Claude API handles complex reasoning. Two-tier model operational.

**Deliverables:**
- `OmlxService` and/or `OllamaService` impl of `AiService` trait
- Two-tier routing: local model for intent classification + short completions;
  Claude API for multi-step reasoning
- AI reliability design (full):
  - Job queue: all inference requests queued, not fire-and-forget
  - Prompt-cache breakpoints: cache prefix at segment boundaries to reduce token burn
  - Streaming with reconnect: inference streams checkpointed every N tokens
  - Checkpointing: long operations save intermediate state to storage harness
- Hot-swap: live backend swap without restart (AiService slot swapped at runtime)
- `NullAiService` validated: bare install boots and deterministic operations work without any AI backend

**Acceptance criteria:**
- [ ] Swap backend from Claude API to local model at runtime, NL terminal continues working
- [ ] Job queue: two concurrent inference requests are queued and resolved in order
- [ ] Streaming reconnect: simulate inference interruption, verify resume from checkpoint
- [ ] Bare boot with `NullAiService`: all harness operations work, NL terminal gracefully degrades

---

## Milestone 5 — NL Terminal Full Build
**Estimate:** 4-6 weekends

**Goal:** Full NL terminal experience — natural language to any wired harness, with
context-driven UI that regenerates based on what the user is doing.

**Deliverables:**
- NL terminal: full intent parser wired to all available harnesses
- Context-driven UI: display adapts to current operation (file browsing vs settings vs search)
- Search harness: full-text search over storage harness contents
- Service spawner harness: launch/stop background services through the bus
- Session context: AI maintains short-term context across commands in a session
- "Type what you want and it wires up": user can request a new capability and the terminal
  scaffolds it if it can be composed from existing harnesses

**Acceptance criteria:**
- [ ] End-to-end: `"find all notes about GigaOS and show them as a list"` — works via search + storage + generated UI
- [ ] Service spawner: `"start the backup service"` → service spawned through harness
- [ ] Context continuity: follow-up command references context from prior command in session
- [ ] Generated capability: user requests something composable → terminal assembles it

---

## Milestone 6 — Appliance Image
**Estimate:** 4-8 weekends  
**Prerequisite:** BASE_LAYER.md decision made.

**Goal:** GigaOS is a bootable image. Install it on a machine, it replaces the existing OS,
boots directly into the GigaOS experience.

**Deliverables:**
- Bootable image built from the chosen base layer (see BASE_LAYER.md)
- Immutable appliance pattern: read-only system layer, user data in a separate partition
  (Silverblue/ostree or Nix profile model)
- Atomic updates: system can update itself without breaking user data
- Install script: wipe-and-install path for the Marketplace box target
- First-boot wizard: minimal config (AI backend selection, storage path)

**Acceptance criteria:**
- [ ] Fresh machine boots into GigaOS NL terminal from cold start
- [ ] All M5 capabilities work on the appliance image
- [ ] Update: apply a new image without touching user data
- [ ] First-boot wizard completes in under 5 minutes

---

## GPU Tier (Giga²OS) — Parallel Track from M4

The Supercharged tier — a large local model on a GPU box — is enabled by the Linux base
(CUDA drivers already exist) and wires in at the AiService slot.

This track can run in parallel with M5/M6:
- Large local model running in Ollama / vLLM on the GPU machine
- `LocalGpuService` impl of `AiService` trait
- Same harness bus, same capability model, same NL terminal — different backend
- Base tier (Mac mini / no GPU) degrades gracefully to Claude API

The CUDA/ROCm story depends on the BASE_LAYER.md choice (see that doc — GPU inference is
the deciding variable for base layer selection).

---

## What is not in this roadmap

- Hand-written TCP/IP or network stack (Linux's stack)
- Custom NIC drivers (Linux's drivers)
- From-scratch compositor or bootloader (Linux base)
- In-kernel inference (userspace; the soft-float contradiction disappears entirely here)
- AArch64 port (M6+ scope, possible once the appliance image exists)

These are not compromises. They are the correct scope boundary for a project whose identity
is the harness model, not the substrate.

---

## The Oppermann Kernel Track

`gigaos-kernel` (separate repo) is the Philipp Oppermann "Writing an OS in Rust" skeleton.
It is personal enrichment and a low-level reference — not a product gate, not a milestone here.
Pull from that code only if you ever need a specific low-level reference during the userspace build.
Do not plan around it or score its sessions as progress toward GigaOS.
