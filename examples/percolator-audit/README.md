# Worked example — Percolator audit

This is the live audit that the pipeline was developed against. It produced
the methodology, the templates, the scripts, and the agent prompts in this
repository.

**Live disclosure**: https://github.com/Copenhagen0x/percolator-audit-2026-04

**Filed issue**: https://github.com/aeyakovenko/percolator-prog/issues/55

**Maintainer's public acknowledgment**: posted on Twitter by Anatoly Yakovenko (@toly), who attributed multiple findings to @Copenhagen0x and shipped one of the recommended fixes (h_max cap) ~10 hours after Issue #55 was filed.

## What the audit produced

| Artifact | Count | Pipeline layer |
|---|---|---|
| Active bug | 1 (cursor-wrap consumption-budget reset) | Layers 1-5 |
| Code-defect findings | 2 (mul_div_floor_u128 panic class — engine math unsafe but not exploitable at default caps per Layer-4 bound analysis) | Layers 1-5 |
| Disclosed-but-not-a-bug | 1 (Sibling B — defense-in-depth recommendation only) | Layers 1-3 |
| Formal SAFE proofs | 10 (including 2 that machine-checked the maintainer's own G3 closure statement) | Layer 3 |
| Re-run of maintainer's existing Kani baseline | 305/305 PASS, 0 regressions | Layer 5 |
| Native PoC tests | 4 (engine-side) | Layer 2 |
| LiteSVM PoC tests | 8 (wrapper-side) | Layer 4 |
| Cross-platform reproduction | All 12 PoCs bit-identical on Windows + Linux VPS | Layer 5 |

## How each layer was applied

### Layer 1 — Multi-agent code review

Used 3 parallel "v11" agents on disjoint hypotheses:
- v11-L1: pattern-hunt for Bug #2 sibling overflow sites
- v11-L2: end-to-end verification of CatchupAccrue (G3 closure)
- v11-L3: implicit invariant hunt + Kani harness authoring

Each agent's output fed into Layers 2-3.

### Layer 2 — Empirical PoC

Wrote engine-native panic tests for each candidate finding:
- `test_v9_warmup_overflow.rs` (Bug #2)
- `test_v11_l1_trade_open_overflow.rs` (Bug #3)

Plus regression-guard tests proving Bug #1's downstream effects.

### Layer 3 — Kani formal verification

5 Kani files, 17 harnesses total:
- 4 CEX harnesses (formally proved bugs reachable)
- 10 SAFE harnesses (formally proved invariants hold)
- 3 unwind-noise FAILs (later confirmed as Kani-tooling artifacts, not real bugs)

### Layer 4 — LiteSVM bound + reachability

5 LiteSVM test files for the BPF wrapper:
- 2 cursor-wrap exploit chain tests (Bug #1)
- 1 cursor-wrap regression-guard test (Bug #1)
- 1 trade-open call-chain reachability + bound analysis (Bug #3)
- 1 warmup-overflow bound analysis (Bug #2)

Bound analyses produced the "~167M years" and "~18 years" wall-clock reachability numbers that downgraded Bugs #2/#3 from "active exploit" to "code defect."

### Layer 5 — Cross-platform reproduction

Provisioned a Hetzner VPS (6-core, Ubuntu 22.04). Same toolchain as local
(Rust 1.95 + Solana 3.1.14 + Kani 0.67.0). Ran every PoC test on both
hosts; bit-identical results.

Capstone Layer-5 deliverable: re-ran maintainer's full 305-harness Kani
baseline against current main. Result: 305/305 PASS, 0 regressions.

## Timeline

| Phase | Duration | What happened |
|---|---|---|
| Initial recon (Layer 1) | ~3 hours | Multi-agent hypothesis generation + verification |
| PoC + Kani (Layers 2-3) | ~6 hours | Wrote and verified the 4 CEX + 10 SAFE harnesses |
| LiteSVM (Layer 4) | ~4 hours | Bound analyses + call-chain reachability tests |
| VPS + cross-platform (Layer 5) | ~6 hours including baseline re-run | Provisioning + execution |
| Disclosure documentation | ~3 hours | DISCLOSURE.md + EXEC_BRIEF.md + RECOMMENDED_PATCH.md + METHODOLOGY.md |
| Self-audit (5 rounds, 30 agents) | ~3 hours | Caught 13 substantive errors before publication |
| Total | ~25 hours over 3 days | |

## What this proves about the pipeline

The audit demonstrates that the pipeline can produce production-grade disclosure with the following properties:

1. **Formal-verification-backed findings** — each bug has a Kani CEX, not just an empirical test
2. **Honest reachability framing** — Layer-4 bound analyses downgraded findings BEFORE the maintainer could push back, preserving credibility
3. **Maintainer-respectful** — the disclosure reads as collaborative, not adversarial; the maintainer publicly attributed and acted on it
4. **Reproducible cross-platform** — the maintainer can run every PoC on their own infrastructure and get the same results
5. **Self-audited** — 13 errors caught in self-audit before publication; zero errors caught BY the maintainer post-publication

## Files in the live disclosure

The disclosure repo at https://github.com/Copenhagen0x/percolator-audit-2026-04 contains:

```
percolator-audit-2026-04/
├── README.md                       (front door, TL;DR table)
├── DISCLOSURE.md                   (canonical disclosure document)
├── EXEC_BRIEF.md                   (one-page reference)
├── METHODOLOGY.md                  (the 5-layer pipeline doc)
├── RAW_KANI_RESULTS.md             (per-harness verification times)
├── RECOMMENDED_PATCH.md            (exact 3-site git diff for the fix)
├── LICENSE
├── baseline/                       (305-harness re-run artifacts)
│   ├── README.md
│   ├── kani_baseline_summary.txt
│   ├── kani_baseline_timings.tsv
│   └── kani_baseline_full.log.gz
└── tests/
    ├── engine/  (7 files)          (Layer 2 PoCs + Layer 3 Kani harnesses)
    └── wrapper/ (5 files)          (Layer 4 LiteSVM tests)
```

Each file in `tests/` corresponds to a template in this pipeline's `templates/` directory.

## Lessons specific to Percolator

These lessons live in `docs/lessons-learned.md`. Some highlights:

- Engine constants matter: knowing `MAX_VAULT_TVL = 1e16` and `MAX_ACCOUNT_POSITIVE_PNL = 1e32` was essential for Layer-4 bound analysis
- The `test_visible!` macro pattern is what made Layer 2 + 3 possible against private engine state
- Solana mainnet target slot time is ~400 ms; use 500 ms (2 slots/sec) for conservative bounds
- The `wide_mul_div_floor_u128` U256-intermediate helper was already in the engine; the recommended fix was a one-token swap, not a new helper

## See also

- [`../../docs/`](../../docs/) for the conceptual layer documents
- [`../../templates/`](../../templates/) for the test-pattern templates
- [`../../agent_prompts/`](../../agent_prompts/) for the prompts used in the recon
