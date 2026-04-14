---
layout: post
title: "Exp 03: Initialization + Depth — Init Quality Is a Precondition, Not a Detail"
description: "3 depths × 2 init schemes. A 4x shallower model with flat init loses to full depth with scaled init."
tag: "Exp 03 · Pretraining"
date: 2026-04-14
---

Exp 02 showed that warmup dominates over optimizer choice in the first 50 steps. Exp 03 goes further back — to initialization itself. The question: does the init scheme matter, and does it interact with depth?

The experiment is a 3×2 factorial: three model depths (3L, 6L, 12L) crossed with two initialization schemes (flat std=0.02 everywhere, and GPT-2's scaled init where `c_proj` weights are divided by √(2×N)). Six variants, 300 steps each, everything else identical to the Exp 01 baseline.

## The setup

| Variant | Layers | Init |
|---------|--------|------|
| A | 3L | Flat (std=0.02) |
| B | 3L | Scaled (c_proj × 1/√(2N)) |
| C | 6L | Flat |
| D | 6L | Scaled |
| E | 12L | Flat |
| F | 12L | Scaled ← closest to baseline |

## What the run looked like

![Exp 03 — initialization and depth comparison](/assets/img/exp03_overview.png)
<p class="img-caption">6 variants across 8 metrics. The scaled/flat split is visible from the first few steps and widens throughout.</p>

## Observations

**Scaled init converges faster and lower at every depth.** The separation is visible from the first few steps and widens throughout the run. The mechanism is direct: flat init lets each residual block contribute O(1) variance to the residual stream, so variance accumulates as O(N) across N layers. Gradients flowing back through this amplified stream are noisier, weight updates less directional, and the early phase of training is partially wasted fighting the init rather than building structure. Scaled init bounds residual stream variance to O(1), so every gradient step from the start is doing useful work.

The more telling comparison: **12L/scaled ends lower than 3L/flat.** A 4× shallower model with messy init falls behind a full-depth model with clean init. Init quality isn't a minor hyperparameter at depth — it's a precondition for depth to be useful at all.

**Gradient norm shows the same cold-start pattern as Exp 02, but driven by init rather than warmup.** Early spikes are sharper and less coherent for flat variants, especially at 12L. Flat init places the model in a high-curvature region of the loss landscape — the same condition that no-warmup creates by applying full LR before the moment estimates are reliable. Scaled init enters training quietly and stabilises fast.

**Parameter norm is cleanly stratified by depth** — 12L highest, 3L lowest, 6L in between. Within each depth pair, flat sits above scaled. This follows from init scale: flat residual projections start larger and get pushed further by noisier early gradients. Unlike the parameter norm anomaly in Exp 02 (where no-warmup variants ended unexpectedly smaller), the flat-vs-scaled gap here has a clean first-principles explanation.

**Activation std is the key result.** Scaled init produces *higher* final-layer activation std than flat init at every depth — the gap widens with depth. At 3L the lines are close; by 12L there's clear separation. This is counterintuitive: scaled init uses smaller weights (lower parameter norm), yet produces more expressive final representations.

The explanation is that flat init forces layers to compete. Early noisy residual contributions cause later layers to learn defensive near-zero outputs to prevent blow-up, and some of those contributions cancel through the residual accumulation — the network wastes depth. Scaled init lets layers compose: each block adds structured, coherent signal to the residual stream, and the full depth actually contributes. Higher activation std at the final layer is the geometric signature of that — the representations are spread out and discriminative rather than partially self-cancelled.

**The activation std gap grows with depth.** Small at 3L, large at 12L. Each additional block under flat init adds more noise to the residual stream; each additional block under scaled init adds more coherent signal. At 12L the divergence is the dominant signal in the panel — larger than the depth effect itself within the flat group. This has a direct implication for Phase 2: at 24L or 48L, flat init would produce residual stream variance that LayerNorm can no longer reliably suppress. What we see at 12L is a small preview of that failure mode.

**Activation mean** stays near zero for most variants, as expected from Pre-LN with no additive bias in the residual stream. Two scaled init variants show slight upward drift by step 200 — small magnitude, opposite direction to the Exp 02 C drift. Too early to call at 300 steps, but worth watching at full run length: if it persists, it would suggest clean gradient signal is compound-biasing the residual stream in a consistent direction across blocks.

## Takeaway

Scaled init is unambiguously better — faster convergence, cleaner gradients, lower parameter norm, higher final activation std. The result holds at all three depths and the benefit compounds with depth. The mechanism is variance control in the residual stream, not a subtle optimizer interaction.

For all remaining Phase 1 experiments, the baseline should use scaled init by default.

The connection to Exp 04 is direct: Post-LN normalises after the residual addition, which means it sees the full accumulated residual stream variance before normalising — exactly the quantity that scaled init is controlling. The expectation going in: Post-LN will be more sensitive to init scheme than Pre-LN, and scaled init will be proportionally more important for Post-LN stability.
