---
layout: post
title: "GPT-2 Small from Scratch: Four Pretraining Experiments"
description: "Baseline, optimizer × warmup, init × depth, normalization × LR. What the training curves actually say."
tag: "Pretraining · Phase 1"
date: 2026-04-15
---

Four experiments, one model, one question: what does stability actually look like in a small language model, and what breaks it?

The setup is GPT-2 Small — 12 layers, 12 heads, 768-dimensional embeddings, 85.7M parameters after weight tying. Trained on FineWeb-Edu, a filtered educational subset of Common Crawl. Each run tracks 8 signals simultaneously: loss, learning rate, gradient norm, parameter norm, update-to-weight ratio, and activation statistics at the final layer. That's the vocabulary for everything below.

---

## Exp 01 — Baseline

Before you can read what breaks, you need to know what healthy looks like.

**Config.** All defaults. 1B tokens, 1907 steps at a 524K token effective batch size (8 micro × 64 grad accumulation × 1024 context). AdamW with β1=0.9, β2=0.95, weight decay 0.1, gradient clip 1.0. LR cosine schedule from 6e-4 to 6e-5 with a 50-step warmup. BF16 autocast throughout — on Blackwell tensor cores this gives a genuine 2.4× throughput improvement over FP32 (13k → 30.5k tok/s), with no GradScaler needed because BF16 preserves FP32's exponent width.

![Exp 01 — full training overview](/assets/img/exp01_overview.png)
<p class="img-caption">8 metrics across 1907 steps. The baseline everything else is compared against.</p>

**Loss** started near 11 — random initialization over a 50,257-token vocabulary gives an expected loss of ln(50257) ≈ 10.82. It dropped to around 4 by end of training. Train and val tracked closely throughout; you can't meaningfully overfit a 124M parameter model on 1B tokens from a large varied corpus. The final ~4 is slightly above the ~3.0–3.2 target, which becomes the reference point for all variants.

**Gradient norm** showed spikes during warmup. This is worth understanding correctly. Random initialization places the model in a high-curvature region — gradients are large in magnitude and incoherent in direction before the model has built any structure. Warmup keeps the step size small while this is happening, so it's the mitigation, not the cause. Post-warmup the norm stabilises.

**Parameter norm** climbed slowly and monotonically. **Update-to-weight ratio** sat near the target 1e-3 at peak LR, drifting down with cosine decay as expected. Activations at `ln_final` stayed near zero mean, 0.8–1.2 std range. Nothing unusual. This is what healthy looks like.

---

## Exp 02 — Optimizer × Warmup

Two binary switches: Adam vs AdamW, warmup on vs off. Four variants, 1907 steps each, everything else from Exp 01 unchanged.

| Variant | `optimizer` | `warmup_steps` |
|---------|-------------|----------------|
| A | `"adam"` | `0` |
| B | `"adam"` | `50` |
| C | `"adamw"` | `0` |
| D | `"adamw"` | `50` ← baseline |

![Exp 02 — optimizer × warmup comparison](/assets/img/exp02_overview.png)
<p class="img-caption">4 variants overlaid. Warmup is the dominant axis. Optimizer choice is secondary at this scale.</p>

**The warmup result is unambiguous.** Variants A and C — no warmup, different optimizers — track together at a higher loss throughout the entire run. The gap opens in the first 50 steps and never closes. The mechanism is the cold-start problem in Adam's second moment estimate: `v_t` initializes near zero, so the effective step size `lr / sqrt(v_t)` is enormous in the first ~20 steps. With β2=0.95 the memory is ~20 steps — so 50 warmup steps is precisely calibrated to let `v_t` converge before full LR is applied.

**The optimizer result is more subtle.** Adam vs AdamW produces no distinguishable difference in loss curves. The separation appears only in gradient norm, and only after ~750 steps — AdamW shows noticeably better damping of fluctuations. The timing is meaningful: weight decay needs ~750 steps to accumulate enough regularization pressure to visibly constrain parameter growth. Early in training the decay term is small relative to the gradient signal. It's a slow-burn effect, irrelevant to loss dynamics on a data-rich corpus, visible only in gradient stability over the long run.

**One anomaly worth flagging.** The no-warmup variants end with *smaller* L2 parameter norms than their warmup counterparts. Counterintuitive, since no-warmup applied larger steps early. This doesn't have a clean first-principles explanation from the current data — worth revisiting with per-layer norm tracking rather than rationalising it.

**Activation mean drift.** Variant C — AdamW without warmup — shows `ln_final` activation mean drifting upward after ~1000 steps while all other variants stay near zero. The combination matters: without warmup, the first ~20 steps take enormous effective steps, pushing weight matrices into asymmetric configurations before the optimizer has reliable curvature estimates. Weight decay then shrinks all weights proportionally — but if certain embedding directions were pushed systematically positive by those early noisy steps, decay shrinks them slower than new gradient signal pushes them further. Adam without warmup (A) doesn't trigger the same mechanism because there's no decoupled weight decay to reinforce the asymmetry.

The magnitude is small at 12 layers. The concern is the trend — in a deeper or longer run, this kind of drift shifts the effective operating point of every LayerNorm downstream.

**Takeaway:** warmup is load-bearing, optimizer choice is secondary at this scale and data regime.

---

## Exp 03 — Initialization × Depth

Six variants: three depths (3L, 6L, 12L) crossed with two init schemes (flat std=0.02 everywhere vs GPT-2 scaled init where c_proj weights are divided by √(2 × n_layer)). 300 steps each — a diagnostic run, not a full training run.

| Variant | `n_layer` | `scaled_init` | `max_steps` |
|---------|-----------|---------------|-------------|
| A | `3` | `False` | `300` |
| B | `3` | `True` | `300` |
| C | `6` | `False` | `300` |
| D | `6` | `True` | `300` |
| E | `12` | `False` | `300` |
| F | `12` | `True` | `300` |

All other fields unchanged from Exp 01.

![Exp 03 — init × depth comparison](/assets/img/exp03_overview.png)
<p class="img-caption">6 variants. Dashed = flat init, solid = scaled. The gap between them grows with depth.</p>

**Scaled init converges faster and lower at every depth.** The separation is visible from the first few steps. The mechanism is direct: flat init lets each residual block contribute O(1) variance to the residual stream, so variance accumulates as O(N) across N layers. Gradients flowing back through this amplified stream are noisier, weight updates less directional, and early training is partially wasted fighting the initialization. Scaled init bounds residual stream variance to O(1) — every gradient step from the start is doing useful work.

The most telling comparison: **12L/scaled ends lower than 3L/flat.** A model four times shallower with flat init falls behind a full-depth model with clean init. Init quality isn't a minor hyperparameter at depth — it's a precondition for depth to be useful at all.

**Gradient norm** follows the same pattern. Early spikes in flat variants are sharper and less coherent, especially at 12L. The flat/12L spike pattern is the same family as the cold-start behaviour seen in Exp 02 with no warmup — a high-curvature initialization state where gradient signal is large and incoherent until the optimizer accumulates reliable curvature estimates.

**The key result is in activation std.** Scaled init produces *higher* final-layer activation std than flat init at every depth, and the gap widens with depth. This is counterintuitive — scaled init uses smaller weights (lower parameter norm), yet the representations at the final layer are more spread out and discriminative.

The explanation is that flat init forces layers to compete. Early noisy residual contributions cause later layers to learn defensive near-zero outputs to prevent blow-up, and some contributions cancel through residual accumulation. The network wastes depth. Scaled init lets layers compose — each block adds coherent signal to the residual stream, and the full depth contributes. Higher activation std at the final layer is the geometric signature of that.

**The gap scales with depth.** At 3L the dashed and solid lines in activation std are close. By 12L there is clear separation — larger than the depth effect itself within the flat group. For Phase 2 (24L, 48L DeepSeek-style models) this isn't optional — flat init at those depths would produce residual stream variance that LayerNorm can no longer reliably suppress.

**Takeaway:** scaled init is unambiguously better, the benefit compounds with depth, and the mechanism is variance control in the residual stream. All remaining Phase 1 experiments use scaled init as the default.

---

## Exp 04 — Normalization × Learning Rate

Exp 03 ended with a prediction: Post-LN would be more sensitive to LR because the LayerNorm Jacobian `J_LN` sits across the full residual gradient path rather than only the sublayer branch. Exp 04 tests that directly. Six variants: Pre-LN and Post-LN each crossed with three learning rates. 1000 steps each, scaled init and AdamW with warmup=50 unchanged.

| Variant | `norm_position` | `max_lr` | `min_lr` |
|---------|-----------------|----------|----------|
| A | `"pre"` | `3e-4` | `3e-5` |
| B | `"pre"` | `6e-4` | `6e-5` |
| C | `"pre"` | `1e-3` | `1e-4` |
| D | `"post"` | `3e-4` | `3e-5` |
| E | `"post"` | `6e-4` | `6e-5` |
| F | `"post"` | `1e-3` | `1e-4` |

![Exp 04 — normalization × learning rate](/assets/img/exp04_overview.png)
<p class="img-caption">6 variants, Pre-LN solid, Post-LN dashed. The LR sensitivity reversal is the main result.</p>

**Pre-LN converged faster and lower at every LR.** Best Pre-LN (C, 1e-3) reached val loss 4.35. Best Post-LN (D, 3e-4) reached 5.85 — a gap of ~1.5 nats after 1000 steps.

**The LR sensitivity reversed between the two norm variants.** In Pre-LN, higher LR produced better final loss: C > B > A. The model was not destabilised at 1e-3 — the clean gradient path kept training healthy. In Post-LN the pattern flipped: lower LR produced better final loss. D (3e-4) was the only variant that learned meaningfully. E (6e-4) and F (1e-3) essentially stalled after step 100, oscillating around 7.5–7.6 for the remaining 900 steps. That's not slow convergence — it's a training failure.

The reversal is the sharpest result in this experiment. LR tolerance is not a function of model size or dataset alone — it's directly determined by where normalization sits relative to the residual connection.

**The mechanism is in the gradient structure.** In Pre-LN (`x' = x + F(LN(x))`), the residual identity path bypasses LayerNorm entirely:

```
∂L/∂x = ∂L/∂x' · (I + J_F · J_LN)
```

The `I` term means the upstream gradient passes through unchanged regardless of `J_LN`. In Post-LN (`x' = LN(x + F(x))`), LayerNorm sits after the residual add — there is no bypass:

```
∂L/∂x = ∂L/∂x' · J_LN(h) · (I + J_F)
```

`J_LN` gates everything. At the start of training, activations have high variance and `J_LN` scales gradients unpredictably — suppressing some components, amplifying others, zeroing out the mean and variance directions entirely. Across 12 stacked blocks this compounds multiplicatively. High LR amplifies the instability: at 1e-3 Post-LN fails immediately; at 3e-4 it survives but converges slowly.

**Gradient norm** was much more erratic in Post-LN throughout training. Early spikes were sharper and more sustained — `J_LN` gating in the residual path produces unstable gradient scaling in the first ~100 steps that Pre-LN avoids entirely.

**Parameter norm** was consistently larger in Pre-LN. Post-LN normalizes the residual stream at every block output, implicitly constraining activation magnitudes — downstream weights don't need to grow large to handle large activations because the block output is always re-normalised before being passed on. In Pre-LN the residual stream accumulates freely across all 12 blocks and weights adapt by growing. This is not pathological; it reflects that Pre-LN weights are doing more work in absolute terms because the residual stream is richer.

**Residual stream depth profile** at step 900 shows the contrast cleanly. Pre-LN: mean near zero across all layers, std increasing steadily with depth as each block adds coherent signal. Post-LN E and F: irregular oscillating profiles — the normalised block outputs are incoherent, the network is not building structured representations across depth.

**Takeaway:** Pre-LN is unambiguously better across all tested LRs. The mechanism is precise: the residual identity bypass in Pre-LN allows higher LRs without instability, while Post-LN's `J_LN` gating makes it LR-sensitive and prone to early training stalls. Pre-LN remains the baseline for all remaining Phase 1 experiments.

---

## What connects them

These four experiments test fundamentally different things — duration, optimizer choice, initialization, normalization placement — but they keep pointing at the same underlying structure: **the residual stream is the thing to watch**.

Warmup matters because cold `v_t` produces step sizes that hammer the residual stream before it has any structure. AdamW/no-warmup drift appears specifically at `ln_final` because unchecked weight asymmetry accumulates across 12 residual additions. Flat init's failure is that it lets each block add uncontrolled variance to the stream, making depth counterproductive. Post-LN's fragility is that `J_LN` gates the residual gradient path, making the entire system sensitive to anything that inflates activation variance — high LR, flat init, no warmup — and any such perturbation compounds across all 12 blocks.

Every signal in the training curves — gradient norm, activation std, parameter norm drift, residual depth profile — is a different way of observing the same underlying quantity: whether the residual stream is in a regime where each layer can add structured, coherent signal, or whether it's fighting itself.

The practical result of all four: **scaled init, AdamW with warmup, Pre-LN, LR around 1e-3** — that's what a stable small transformer pretraining run looks like. Every deviation from that cluster produces a readable signature in the curves.
