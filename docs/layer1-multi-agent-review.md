# Layer 1 — Multi-agent code review

**Goal**: Reduce the "did I miss a hypothesis" uncertainty.

**Output**: A prioritized list of candidate findings, each with a short evidence trail (file:line citations + a one-sentence claim).

## The model

A single human reviewer is rate-limited by attention. You can read 7,000 lines of Rust carefully, but you cannot simultaneously hold 12 disjoint hypotheses about that code in your head and trace each through the call graph.

Multi-agent review fans out the hypothesis space. Each agent owns ONE hypothesis and goes deep on it. The orchestrator (you) collects the agent outputs and decides which candidates earn a Layer-2 PoC attempt.

## The shape of a good agent prompt

A good agent prompt does four things:

1. **States the hypothesis precisely**, framed as a question (not an instruction)
   - Bad: "Find all integer overflow bugs"
   - Good: "Does any code path in `src/percolator.rs` write to `position_basis_q` without enforcing `MAX_POSITION_ABS_Q`?"

2. **Names the files to read** with absolute paths

3. **Tells the agent what to deliver** (table format, line citations, verdict)

4. **Caps the response length** (typically 600-800 words)

Examples of agent prompts that worked well during the Percolator audit are in `agent_prompts/` in this repo.

## The orchestration pattern

```
Phase 1: HYPOTHESIS GENERATION
  - You write 5-10 hypotheses based on:
    - Spec-vs-code gaps (English claims in spec/comments that aren't asserted in code)
    - Dangerous primitives (panic-class functions, saturating arithmetic in safety-critical paths)
    - State transitions (atomic blocks, lazy invariants, cross-tx state)
    - Authorization chains (admin-gated, permissionless, signer-required)

Phase 2: PARALLEL DISPATCH
  - Spawn N agents in parallel, one per hypothesis
  - Each agent gets its own focused prompt
  - Wait for all to return

Phase 3: SYNTHESIS
  - You read the agent outputs
  - Categorize: real candidate / refuted / inconclusive
  - For real candidates, queue Layer-2 PoC work
```

## Failure modes to guard against

| Failure mode | Mitigation |
|---|---|
| Agent over-confidence — claims "VERIFICATION FAILED with CEX" when the harness didn't compile | Always cross-check claimed verdicts against raw artifacts (log files, source lines). Don't promote an agent verdict into a finding without verifying. |
| Hypothesis bias — "find this bug" produces false positives | Frame hypotheses as "is this invariant true?" — produces clean negatives that strengthen the disclosure |
| Agent stops at the first plausible answer instead of exhaustively scanning | Add to the prompt: "List EVERY instance, even minor. Final tally must include at least N items." |
| Agents converge on the same candidate (because the prompt over-anchored) | Disjoint hypotheses; spread the surface so each agent owns a non-overlapping slice |

## How many agents in parallel?

Empirical observation from the Percolator audit:

| Audit phase | Agents | Notes |
|---|---|---|
| First-pass orientation | 3-5 | Coverage of distinct code regions |
| Hypothesis-driven recon (e.g., insurance-balance hunt) | 5-10 | Each on a specific candidate path |
| Self-audit / quality check | 30+ | One per file + cross-cutting checks |

Beyond ~10 in parallel you start to lose orchestration coherence. Use sequential batches if you need more breadth.

## What a good Layer 1 output looks like

For each candidate finding, the agent should return roughly:

```
Hypothesis: <one sentence, framed as a question>
Verdict: TRUE / FALSE / NEEDS_LAYER_2_TO_DECIDE
Evidence:
  - file:line (claim)
  - file:line (counter-claim if any)
  - call chain (entrypoint → ... → site)
Confidence: HIGH / MED / LOW
Suggested next step: Layer 2 PoC name, OR refute via Layer 3 SAFE proof, OR drop
```

You then aggregate these into a candidate-findings spreadsheet (or markdown table) and prioritize which earn Layer 2 work.

## Time budget

For a first-pass on a ~7000-line Rust engine + ~5000-line wrapper:
- Hypothesis generation (you): 1-2 hours
- Parallel agent dispatch + waiting: 30-60 min
- Synthesis (you): 1 hour
- Total: 3-4 hours

Repeat audits on similar codebases: 1-2 hours total because the hypothesis library is reusable.

## See also

- [`agent_prompts/`](../agent_prompts/) — actual prompts that produced findings on Percolator
- [Layer 2 — Empirical PoC](./layer2-empirical-poc.md) — what to do with the candidates
