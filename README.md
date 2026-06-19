# GigaOS

An AI-native OS personality running on a Linux base.

GigaOS is not a kernel project. It is an operating system defined by its **harness bus** — a typed,
capability-gated dispatch layer that sits between AI inference and the real system resources of a
Linux host. The experience is entirely ours; the substrate (network, filesystem, GPU) is Linux's.

## What GigaOS is

- A **harness bus** — typed message passing, capability enforcement, hot-swap backends
- A **typed AI service slot** — null / Claude API / oMLX / local model, swappable at boot
- A **natural-language terminal** — fuzzy intent in, deterministic harness dispatch out
- A **context-driven UI** — hallucinated layout over deterministic primitives
- Built in **Rust**, under **four lambda calculus guardrails**, on a **Linux base**

## What GigaOS is not

- A ground-up kernel (that is [gigaos-kernel](https://github.com/GigaAssistant/gigaos-kernel) — personal enrichment, not a product track)
- A demo that loses your files (durable primitives are always real, never hallucinated)
- A single-model system (two-tier: local model for fast paths, Claude API for reasoning)

## The Quality Rule

> Deterministic, real implementations for durable primitives.  
> Hallucinated layout only for bespoke/ephemeral surface.  
> Never hallucinate the filesystem. Hallucinate the file manager's layout.

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) — the four lambda guardrails and the harness layer model.

## Roadmap

See [ROADMAP.md](ROADMAP.md) — six milestones, inverted build order (experience first, appliance image last).

## Harness Spec

See [HARNESS_SPEC.md](HARNESS_SPEC.md) — the North Star: bus protocol, AI service slot, capabilities, AI reliability design, persistence model.

## Base Layer Decision

See [BASE_LAYER.md](BASE_LAYER.md) — open decision: which Linux base the appliance ships on.

## Related

- [gigaos-kernel](https://github.com/GigaAssistant/gigaos-kernel) — Philipp Oppermann Rust kernel tutorial skeleton. Personal enrichment and low-level reference. Not a product track, not a gate on anything here.
