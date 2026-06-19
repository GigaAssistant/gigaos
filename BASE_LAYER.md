# GigaOS — Base Layer Decision
Version: 0.1 (Open Decision)
Date: 2026-06-19

This document is an open decision. It must be resolved before Milestone 6 (Appliance Image).
It does not block M1 through M5. Track progress: fill in the "Decision" field below when made.

**Decision:** PENDING  
**Blocking:** M6 (Appliance Image)  
**Deciding axis:** GPU/CUDA inference support (see below)

---

## What We're Deciding

Which Linux base the GigaOS appliance image ships on. This is the layer beneath the harness bus —
the substrate that provides the filesystem, network stack, GPU drivers, and package ecosystem.

The harness bus, lambda guardrails, NL terminal, and AI service slot are all substrate-independent.
They run on any Linux base. This decision is about packaging, distribution, and GPU support —
not about the OS personality, which is entirely ours regardless of the base choice.

---

## Why This Decision Is Hard to Reverse

Once the appliance image build is wired to a base layer:
- Package declarations, atomic update scripts, and installer tooling all couple to the base.
- Moving bases mid-M6 is a rewrite of the packaging layer.
- The CUDA/driver story is entirely different on musl vs glibc vs NixOS.

Make this decision once, with clear criteria. Don't revisit it during M6.

---

## Deciding Axis: GPU/CUDA Inference

The Supercharged tier (Giga²OS) requires local GPU inference — large model, CUDA, NVIDIA drivers.
This is not M1 scope. It is a parallel track from M4 (ROADMAP.md § GPU Tier).

**The CUDA constraint eliminates Alpine/musl-based distributions outright.**
CUDA assumes glibc. musl compatibility requires a compatibility shim that is finicky and
unsupported by NVIDIA. Do not build the appliance on Alpine or any musl-based base.

**Remaining candidates are all glibc-based.** The GPU/CUDA axis then distinguishes:
- Which bases have a proven, well-supported CUDA path.
- Which bases require significant manual work to get CUDA working.
- Which bases get CUDA support from the distribution vs. requiring upstream sources.

---

## Candidate Matrix

| Axis | Fedora Silverblue | NixOS | Ubuntu (Noble/LTS) | Debian (Stable) |
|---|---|---|---|---|
| **GPU/CUDA inference** | ✅ RPM Fusion CUDA; rpm-ostree overlay; proven path | ⚠️ Powerful but finicky; nixpkgs CUDA support exists, requires config; unstable channel for latest drivers | ✅ Official CUDA repos; easiest install path; best ecosystem support | ✅ Official CUDA packages; slightly behind Ubuntu; solid |
| **Atomic updates** | ✅ ostree/rpm-ostree native; immutable root | ✅ NixOS generations are atomic by definition | ⚠️ Requires Snap or manual overlay; not native immutable | ⚠️ Requires additional tooling |
| **Immutable appliance pattern** | ✅ Silverblue is designed for this; known tools (Flatpak, toolbox, bootc) | ✅ Declarative config IS the immutable pattern; NixOS generations replace image layers | ⚠️ Possible but not native | ⚠️ Possible but not native |
| **Reproducibility** | ⚠️ rpm-ostree composefs; good but not hermetic | ✅ Nix closures are hermetic; system = pure function of config | ❌ Not reproducible by default | ❌ Not reproducible by default |
| **Lambda guardrail alignment** | ✅ Good | ✅✅ Best — system as pure function of its config IS the substitution guardrail made literal | Neutral | Neutral |
| **Existing Giga experience** | None | ✅ GigaNixOS VM at 192.168.1.148 running; familiarity with config | Some | None |
| **Appliance tooling maturity** | ✅ Bootc/image-mode actively developed; first-class tooling | ⚠️ NixOS ISO generation is mature; image-mode tooling less standardized | ✅ Mature; Ubuntu Core exists | ✅ Mature |
| **First-boot wizard** | ✅ Anaconda handles it | ⚠️ Requires custom NixOS installer or Calamares | ✅ Ubiquity/Subiquity | ✅ Debian installer |
| **Community/ecosystem** | Large Fedora community; GNOME-first | Dedicated but smaller; high-quality docs | Largest Linux community | Stable, conservative |

---

## The Core Tension

**Fedora Silverblue** is the pragmatic path:
- GPU/CUDA: RPM Fusion provides CUDA packages; rpm-ostree overlay is the standard approach.
- Atomic appliance: ostree/rpm-ostree is designed exactly for this use case.
- Installer tooling: Anaconda, bootc, image-mode are all production-ready.
- Risk: Less philosophically aligned. "Just works" rather than "proven by construction."

**NixOS** is the philosophically aligned path:
- Lambda guardrail alignment: a NixOS system is defined as a pure function of its config.
  This is the Substitution guardrail and Minimality guardrail made literal at the OS level.
- Reproducibility: Nix closures are hermetic. A NixOS config IS the system.
- Existing experience: GigaNixOS VM is already running. Config patterns are known.
- Risk: CUDA path is "powerful but finicky" — requires stable channel management, NIXPKGS_ALLOW_UNFREE,
  and driver version pinning. Not the easy path for a GPU-forward appliance.

**Ubuntu/Debian** are viable but score poorly on two critical axes:
- Not natively immutable/atomic. Requires additional tooling to get there.
- No philosophical alignment — they're general-purpose distros, not appliance-oriented.

---

## Recommendation (to be confirmed by James)

The decision hinges on how heavily the GPU tier is weighted in M6 vs later.

**If GPU tier is a day-1 appliance requirement:**
→ Fedora Silverblue. The CUDA path is proven and maintained. Atomic appliance tooling is first-class.
  Pay the philosophical cost now; the system works reliably on GPU hardware.

**If GPU tier is a post-M6 "Supercharged" upgrade:**
→ NixOS. The base-layer choice becomes the most architecturally coherent option.
  CUDA finickiness is manageable when it's not on the critical path for the first appliance image.
  The existing GigaNixOS experience shortens setup time significantly.

The Silverblue vs NixOS tension is genuinely a good problem to have — both are defensible.
Pick based on when you need GPU support to be production-ready, not on philosophical preference alone.

---

## What to Decide

Answer these questions to resolve the decision:

1. **GPU tier timing:** Does M6 ship with GPU inference support, or is that a post-M6 upgrade?
2. **Installer investment:** Are you willing to write a custom NixOS installer, or does Anaconda handle it?
3. **Team familiarity:** Is the GigaNixOS config experience sufficient to build an appliance config from scratch?

When answered, fill in:
- **Decision:** [Silverblue | NixOS]
- **Rationale:** [one paragraph]
- **CUDA plan:** [specific packages, overlay approach, driver pinning strategy]
- **Atomic update plan:** [ostree layering | NixOS generations]
- **First-boot wizard plan:** [Anaconda | Calamares + custom | NixOS custom]
