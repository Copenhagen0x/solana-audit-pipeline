# Solana Audit Pipeline

A reusable, formal-verification-grade security audit pipeline for Solana programs. Five layers of rigor: multi-agent code review → empirical PoC → Kani formal verification → LiteSVM BPF-level reachability → cross-platform reproduction.

**Built and proven on**: [`Copenhagen0x/percolator-audit-2026-04`](https://github.com/Copenhagen0x/percolator-audit-2026-04) — independent security audit of Anatoly Yakovenko's Percolator perpetual DEX. Pipeline produced 1 active bug, 2 code-defect-class findings, 10 formal SAFE proofs (including 2 that machine-checked the maintainer's own G3 closure statement), and re-verified 305/305 of the maintainer's existing Kani proofs against current main with zero regressions.

> **Now operationalized as [SENTINEL](https://github.com/Copenhagen0x/audit-pipeline-cli)** — the autonomous audit platform built on this methodology. Continuous source-code monitoring, multi-agent recon, adversarial debate, empirical PoC validation, formal verification, and live mainnet shadow detection — orchestrated end-to-end without human intervention. Confirmed disclosure track record: [F7 against Percolator](https://github.com/aeyakovenko/percolator-prog/pull/39).

---

## Why this exists

Most Solana audits are "I read the code and I think this is a bug." This pipeline is "I formally proved it." The five layers each catch a different failure mode:

| Layer | What it catches |
|---|---|
| 1 — Multi-agent code review | Hypotheses you missed reading alone |
| 2 — Empirical PoC | False positives (if you can't write the test, it's not a bug) |
| 3 — Kani formal verification | Bugs that don't show up in random testing AND proves SAFE properties |
| 4 — LiteSVM BPF-level reachability | Whether the public API can actually drive state to the Kani-found witness |
| 5 — Cross-platform reproduction | Platform-specific artifacts vs real findings |

Every layer reduces a different uncertainty. Skipping a layer is fine if you accept that uncertainty; the pipeline is opinionated about which uncertainties matter for production-grade disclosure.

---

## What's in this repo

| Path | What it is |
|---|---|
| [`docs/`](./docs/) | Deep dives on each layer + lessons learned + reusability checklist |
| [`templates/`](./templates/) | Drop-in Rust file scaffolds for each layer (engine PoC, Kani CEX/SAFE, LiteSVM PoC + bound) |
| [`scripts/`](./scripts/) | VPS provisioning, Kani dispatch, LiteSVM dispatch, cross-platform compare |
| [`agent_prompts/`](./agent_prompts/) | A library of multi-agent prompts that produced the Percolator findings |
| [`examples/percolator-audit/`](./examples/percolator-audit/) | Pointer to the live audit that proved the pipeline |
| [`LICENSE`](./LICENSE) | CC BY 4.0 for docs, Apache-2.0 for templates / scripts |

---

## How to start a new audit

```bash
# 1. Clone the target Solana program at a pinned SHA (engine + wrapper if applicable)
git clone https://github.com/<org>/<engine-repo> /target/engine
git clone https://github.com/<org>/<wrapper-repo> /target/wrapper

# 2. Pin the SHAs you're auditing
cd /target/engine && git checkout <engine-sha>
cd /target/wrapper && git checkout <wrapper-sha>

# 3. Provision a VPS for heavy compute (Kani, LiteSVM, cross-platform)
bash scripts/provision_vps.sh <vps-host> <ssh-key-path>

# 4. Layer 1: spawn the multi-agent review
#    Use agent_prompts/01_codebase_orientation.md as the kickoff
#    Spawn 5-8 agents in parallel on disjoint hypotheses

# 5. Layer 2: for each candidate finding, write an empirical PoC
cp templates/engine_native_poc.rs.template /target/engine/tests/test_<finding>.rs
# edit, run: cargo test --test test_<finding> --features test

# 6. Layer 3: for each finding, write a Kani CEX harness; for each safety claim, write a SAFE harness
cp templates/kani_cex_panic_class.rs.template /target/engine/tests/proofs_<finding>.rs
cp templates/kani_safe_invariant.rs.template /target/engine/tests/proofs_<safety>.rs
bash scripts/dispatch_kani.sh <vps-host> <harness-name>

# 7. Layer 4: build LiteSVM PoC if exploit chain needed; build bound analysis if reachability question
cp templates/litesvm_exploit_chain.rs.template /target/wrapper/tests/test_<finding>_litesvm.rs
cp templates/litesvm_bound_analysis.rs.template /target/wrapper/tests/test_<finding>_bound.rs

# 8. Layer 5: cross-platform compare
bash scripts/cross_platform_compare.sh <vps-host>

# 9. Re-run the maintainer's existing Kani baseline against current main
bash scripts/dispatch_kani.sh <vps-host> --baseline
```

For a fully worked example, see [`examples/percolator-audit/`](./examples/percolator-audit/) and the live repo it points at.

---

## What this pipeline is NOT

- Not a vulnerability scanner. The pipeline produces high-confidence findings; it does not enumerate everything that *might* be wrong.
- Not a substitute for code review. Layer 1 augments human review; it does not replace the human doing the spec-vs-code gap analysis.
- Not a CI tool. Each layer is dispatched manually; integration into CI is possible but out of scope here (see the CLI version: `audit-pipeline-cli`).
- Not free to run end-to-end. Layers 3 (Kani) and 4 (LiteSVM) need real compute, and Layer 5 needs a dedicated VPS. Plan ~$50-200/month for the infrastructure if you're running >1 audit per quarter.

---

## Toolchain pinned (reference)

The Percolator audit ran on:

- Rust 1.95
- Solana 3.1.14 + cargo-build-sbf
- Kani 0.67.0 (CBMC backend, nightly-2025-11-21 toolchain)
- LiteSVM (embedded Solana VM library — runs the BPF program inside the test binary, no separate validator needed)
- proptest (property-based testing for Layer 2 supplements)
- gh CLI 2.x (issue + PR automation)

Other versions probably work; nothing in the pipeline depends on bleeding-edge features.

---

## Contributing

The pipeline is opinionated but not closed. If you find a layer that should be added (e.g., a Layer 6 for runtime instrumentation), open an issue. If a template should change, open a PR.

The prompts in `agent_prompts/` are a starting point. They're tuned to Solana programs with engine + BPF-wrapper architecture. For other architectures, the structure (hypothesis → enumeration → verification → reachability → reproduction) carries over but the specific prompts need adaptation.

---

## License

- **Documentation** (`docs/`, `README.md`, `agent_prompts/`): [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
- **Templates and scripts** (`templates/`, `scripts/`): [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)

See [`LICENSE`](./LICENSE) for full text.

---

## Contact

For questions, corrections, or collaboration: open an issue on this repo or contact the maintainer via [@Copenhagen0x](https://github.com/Copenhagen0x).
