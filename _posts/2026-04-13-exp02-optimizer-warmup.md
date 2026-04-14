---
layout: post
title: "Exp 02: Optimizer + Warmup — What Actually Matters in the First 50 Steps"
description: "Adam vs AdamW, warmup on/off. Four variants. The dominant variable isn't the optimizer."
tag: "Exp 02 · Pretraining"
date: 2026-04-13
---

Exp 01 established the baseline. This one asks a more pointed question: how sensitive is training to the choice of optimizer, and how much does the warmup schedule actually matter?

The experiment is a 2×2 factorial: Adam vs AdamW, warmup vs no warmup. Variant D (AdamW + warmup) is the baseline from Exp 01. Everything else in the model config is unchanged — same 12-layer GPT-2 Small, same 300-step run.

## The setup

| Variant | Optimizer | Warmup |
|---------|-----------|--------|
| A | Adam | None |
| B | Adam | 50 steps |
| C | AdamW | None |
| D | AdamW | 50 steps ← baseline |

All other hyperparameters identical to Exp 01.

## What the run looked like

![Exp 02 — optimizer and warmup comparison](/assets/img/exp02_overview.png)
<p class="img-caption">4 variants across 8 metrics. The warmup split is the dominant visual divide in every panel.</p>

## Observations

**The dominant variable is warmup, not the optimizer.** Both no-warmup cases (A and C) track together strikingly higher on loss throughout — and they track *each other* more closely than either tracks its optimizer counterpart. The mechanism is the cold-start problem in moment estimates. Adam-family optimizers initialise `v_t` near zero, so the effective step size `lr / sqrt(v_t)` is enormous in the first ~20 steps. With `β2=0.95` the moment memory is roughly 20 steps, which means 50 warmup steps is well-calibrated — it buys exactly enough time for `v_t` to converge before full LR is applied. Weight decay produces no distinguishable difference in the loss curves.

**Gradient norm shows the clearest optimizer-level signal.** AdamW (C, D) damps grad_norm fluctuations more than Adam (A, B), and the separation becomes pronounced after ~750 steps. The timing is meaningful: weight decay is a slow-burn effect. Early in training the decay term is small relative to the gradient signal. By ~750 steps the regularization pressure has accumulated enough to visibly constrain parameter growth, which in turn stabilises the gradient. Adam no-warmup (A) is the most erratic throughout — it combines the worst of both variables.

**Parameter norm shows something counterintuitive.** No-warmup variants (A, C) end up with *smaller* L2 parameter norms than their warmup counterparts. This is surprising — no-warmup applies larger step sizes early, which you'd expect to push weights further. The data doesn't support a clean first-principles explanation here. It's worth flagging rather than rationalising: per-layer norm tracking in a later experiment would give a cleaner read on whether this is a depth effect or something else.

**Update-to-weight ratio** converges to the healthy ~1e-3 zone for all four variants. Adam no-warmup (A) gets there fastest — but this is a scale artifact, not an optimizer health signal. At early steps weight norms are small (near init), so the ratio is naturally high. The faster "convergence" reflects large step magnitudes against a small denominator.

**Activation mean revealed the most interesting signal.** The `ln_final` activation mean in AdamW no-warmup (C) drifts upward after ~1000 steps while all other variants stay near zero. The mechanism: without warmup, the first ~20 steps take enormous effective steps, pushing weight matrices into asymmetric configurations before the optimizer has reliable curvature estimates. AdamW's decoupled weight decay then shrinks all weights proportionally — but if certain embedding directions were pushed systematically positive by those early noisy steps, decay shrinks them slower than new gradient signal pushes them further. Adam no-warmup (A) doesn't have decoupled weight decay, so the same reinforcement mechanism doesn't operate.

Why does the drift appear late (~1000 steps) rather than early? Two reasons. First, LayerNorm suppresses early-stage drift — the bias per block is small and normalisation is strong. Second, the effect compounds: each block contributes a small mean bias to the residual stream, and that only becomes visible at `ln_final` once enough steps have elapsed for the per-block biases to structurally solidify across all 12 layers. In a 24L or 48L model the same mechanism would become visible much sooner.

## Takeaway

Warmup is load-bearing. Optimizer choice is secondary at this scale — AdamW provides better grad_norm stability but this doesn't translate to better loss on 300 steps with a data-rich corpus. Weight decay is a slow-burn effect: invisible in loss dynamics, visible only in gradient stability, and only after several hundred steps.

The activation mean drift in AdamW no-warmup is the experiment's most interesting output. It raises a question Exp 02 can't answer: is it concentrated in early blocks (suggesting early chaotic steps as the origin) or spread across depth (suggesting compound accumulation through the residual stream)? That's exactly the kind of signal that calls for per-block activation tracking — which is what Exp 04 is designed to capture.
