# Test-time curriculum adaptation: a negative pilot with auditable course progress

Can verified, student-generated proofs of related courses help a small Lean prover solve a target it has not solved under a fixed sampling budget? This pilot used **AI-MO/Kimina-Prover-RL-1.7B** on `aime_1997_p9`. **No strict original-target success was observed.** This small evidence pack supports planning a controlled follow-up study; it is not a complete reproduction release.

Two separate adaptation paths each completed eight LoRA updates, totaling 512 optimizer steps per path. Courses addressed inverse inequalities, floor identities, algebraic relations, and composition into longer proof chains. Training examples were selected from recorded strict student-generated course successes, with current and historical replay. Although the base model name contains “RL,” these updates used supervised cross-entropy, not a new policy-gradient RL algorithm. Frozen training manifests confirm LoRA rank 16, alpha 32, and learning rate 0.00005. A failed reference cross-entropy check on node2 produced zero optimizer steps and is excluded from completed-update counts.

The frozen generation settings were temperature 0.6, top-p 0.95, top-k 20, a 24,576-token output limit, a 32,768-token total sequence limit, and four-request waves. The actual experimental verifier used **Lean 4.24.0**, with Mathlib commit `f897ebcf72cd16f89ab4577d0c826cd14afaafc7`, as recorded in both node setup receipts. The initial model identity file also records upstream release-era Lean 4.15 compatibility context; that is not this experiment's runtime. These identities and the pinned model revision are separated in [aggregate_metrics.json](aggregate_metrics.json). “Strict” refers to the recorded verifier accepting the exact target declaration, with no `sorry` or unresolved metavariables and only allowed transitive axioms. This release did not rerun Lean.

| Measurement | Baseline | Path node1 | Path node2 |
|---|---:|---:|---:|
| Original-target strict successes / attempts | 0/128 | 0/64 | 0/76 |
| Baseline output truncations | 95/128 | — | — |
| Completed LoRA updates / optimizer steps | 0/0 | 8/512 | 8/512 |
| Course requests started / completed | — | 800/800 | 608/606 |
| Course strict / truncated / naturally unproved | — | 188/334/278 | 150/269/187 |
| Durable course + target output tokens | — | 11,038,486 | 9,279,739 |

The path target denominators are cumulative probes across evolving checkpoints and supplementary searches, **not evaluations of one final checkpoint**. Course successes include repeated samples and related tasks; they are not counts of distinct learned capabilities. Two interrupted course requests on node2 remain in the started denominator; their actual output is unknown, with a retained maximum output budget of 49,152 tokens. Baseline output was 2,675,213 tokens and is excluded from the path totals. Generated tokens, training token exposures, maximum budgets, and overlapping wall clocks must not be conflated with billed GPU hours.

One descriptive milestone is useful for designing the next experiment. On node1, round R8 solved a local inverse-floor course (`fl58`) in 1/16 attempts while a longer connected course (`fl61`) remained 0/16. After an initial training dependency failure, recovery completed the intended 64-step update, including 32 exposures to that successful `fl58` response. The next round produced 1/16 strict proofs of `fl61`: deriving a cubic relation from the full premises in a unified inverse representation. It did not prove the original target's final conclusion. Sampling randomness, changing course selection, and the absence of matched controls prevent attributing this sequence causally to curriculum learning.

The snapshot ends after a further search-only phase: node1 obtained 0/32 strict course responses and node2 obtained 2/32. Those searches caused **zero additional updates and zero additional original-target probes**. Both paths remained at eight completed updates. The work was paused after these searches.

The largest baseline limitation is truncation: 95/128 responses hit the output limit. Consequently, 0/128 describes this exact policy and budget, not mathematical inability. The pilot also lacks randomized curriculum controls, replicated independent seeds, and held-out target evaluation. It establishes neither end-to-end success nor a causal curriculum benefit.

A proposed funded study would first resolve termination and throughput in a small real-model preflight, then freeze prompts, backend, seeds, batching, lengths, and stopping rules. A matched-budget comparison would test direct target sampling, verified self-training without an adaptive curriculum, and adaptive related-course training across multiple targets and independent seeds. The primary endpoint would be strict original-target success; secondary measures would separately track truncation, extractable complete proofs, course outcomes, all started-request costs, and GPU time. These are proposed experiments, not completed results.

[aggregate_metrics.json](aggregate_metrics.json) contains the curated measurements. [provenance.json](provenance.json) identifies exact local source files by relative path and SHA-256, and describes what was checked. Source hashes alone cannot validate omitted raw evidence. This pack includes no raw model responses, adapters, target proof text, private reference proofs, personal information, credentials, or remote machine details.
