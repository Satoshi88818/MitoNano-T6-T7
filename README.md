**MitoNano**  
*Exact-mass-conserving, distributed agent-based simulation of mitochondrial dynamics*

MitoNano is a high-integrity multi-physics simulation framework for modeling populations of agents (mitochondria) that move, exchange mass with a diffusible field, interact with one another, divide, fuse, switch mobility states, and carry heterogeneous phenotypes — all while preserving exact global mass bookkeeping even under full domain decomposition, live migration, and rebalancing.

---

### Highlights

- **Exact global mass conservation** to relative error \( < 10^{-9} \) under simultaneous activation of every mechanism
- Full cross-node physics: agent–agent exchange, field diffusion, **and** fusion across node seams
- Stochastic fission / fusion with optional phenotype mutation on division
- Two-state (mobile / bound) kinetics producing anomalous transport
- Heterogeneous agent phenotypes and per-agent uptake kinetics
- Spatially heterogeneous diffusion (harmonic-mean interfaces)
- Source / sink boundary conditions with strict global-boundary vs. internal-seam distinction
- Dynamic load balancing (median-cut rebalancing) that also conservatively remaps field mass
- 194 / 194 tests passing across three crates, zero warnings

---

### Architecture

The project is deliberately split into three crates:

| Crate | Responsibility | Status |
|-------|----------------|--------|
| **mitonano_core** | Single-node physics engine (`UnifiedEngine`), concentration field, agents, interaction, kinetics, mobility, population primitives | 93 tests |
| **mitonano_orchestrator** | Domain decomposition, migration, rebalancing (validated against a simpler agent type) | 61 tests (untouched) |
| **mitonano_federated** | Integration layer that runs real `UnifiedEngine` instances on each node, adds cross-node couplings, fission/fusion, phenotypes, heterogeneous diffusion, boundary conditions, etc. | 40 tests |

`unified_engine.rs` inside `mitonano_core` is kept **byte-identical** to the original reference implementation. All behavioral extensions are performed at the federated call site by re-implementing the per-tick sequence, preserving the verified core.

---

### Core Invariant

In the absence of sources and sinks:

\[
\text{total_combined_mass}(t) = C_0 - \text{cumulative_eliminated}(t)
\]

When injection and drainage boundary conditions are active the identity generalizes to:

\[
\text{total_combined_mass}(t) = C_0 + \text{injected}(t) - \text{drained}(t) - \text{eliminated}(t)
\]

Both forms are verified to relative tolerance \( 10^{-9} \) at every tick, including under maximal complexity (see `global_bookkeeping_identity_survives_every_gap_closed_alongside_everything_else`).

---

### Physics & Biological Features

**Motion & Transport**
- Bounded SDE motion (drift + diffusion) with reflecting boundaries
- Optional chemotaxis (gradient-biased drift)
- Two-state mobile/bound kinetics (anomalous transport via trapping)

**Field Coupling**
- Diffusible concentration field with release / uptake
- Linear or saturable (Michaelis–Menten) uptake
- Per-agent uptake overrides
- Spatially heterogeneous diffusivity with correct harmonic-mean interfaces
- Cross-node field diffusion using the identical stencil

**Agent–Agent Interaction**
- Radius-limited antisymmetric payload exchange
- Cross-node interaction with boundary-candidate filtering

**Population Dynamics**
- Stochastic fission (division) and fusion (merging)
- Exact payload conservation by construction (`split_payload` / `merge_payload`)
- Optional mutation on fission (daughter may receive a divergent phenotype)
- Cross-node fusion

**Heterogeneity**
- Per-agent phenotype overrides (`AgentParams`)
- Independent control of drift, diffusion, elimination, and release rates

**Domain Decomposition**
- Equal-width or rebalanced x-slabs
- Live agent migration
- Median-cut rebalancing that also remaps field mass conservatively
- Strict distinction between global outer boundaries and internal seams for source/sink conditions

---

### Getting Started

```bash
# Layout (sibling directories)
mitonano_core/
mitonano_orchestrator/
mitonano_federated/

# Build & test (Rust 1.75.0)
cd mitonano_core && cargo test
cd ../mitonano_orchestrator && cargo test
cd ../mitonano_federated && cargo test
```

**Pinned dependencies** (required for the tested toolchain):
- `rand = "=0.8.5"`
- `rayon = "=1.7.0"` / `rayon-core = "=1.11.0"` (core only)
- `nalgebra = "0.32"` (orchestrator only)

The closed-form Monte-Carlo suite in `mitonano_core` takes approximately 50 seconds.

---

### Test Summary (T7)

```
mitonano_core          93 / 93
mitonano_orchestrator  61 / 61
mitonano_federated     40 / 40
─────────────────────────────
Total                 194 / 194
```

Key integration tests live in `mitonano_federated/tests/conservation.rs` and progressively enable more mechanisms while continuously checking the global bookkeeping identity.

---

### Design Principles

1. **Exact conservation is non-negotiable** — approximate conservation is treated as a bug.
2. **Protect the verified core** — `unified_engine.rs` remains byte-identical; extensions live outside it.
3. **Uniform local / cross-node treatment** — the same numerical schemes are used on both sides of a seam.
4. **Pure primitives first** — decision and bookkeeping functions (`event_probability`, `split_payload`, `mutation_fires`, `uptake_amount`, `harmonic_mean`, …) are independently testable.
5. **Isolate the variable** — integration tests disable confounding mechanisms when validating a single pathway.
6. **Honest scoping** — remaining architectural work (shared `Agent` type) is documented rather than forced.

---

### Current Limitations / Open Items

- Agent type reconciliation between `mitonano_core` and `mitonano_orchestrator` is deliberately left open (would require adapter layer).
- Fusion, while now cross-node capable, still uses a sequential local-then-cross design (sufficient for correctness and “at most one fusion per agent per tick”).
- Phenotype and uptake heterogeneity are currently parameter overrides; richer internal agent state is future work.

---

### Project Status

**T7 — All remaining physical gaps closed.**  
Cross-node fusion, per-agent uptake heterogeneity, spatially heterogeneous diffusion, and source/sink boundary conditions are implemented and verified. The single remaining open item is architectural rather than physical or numerical.

---

*MitoNano prioritizes correctness and verifiable invariants over feature velocity. The books must balance.*
