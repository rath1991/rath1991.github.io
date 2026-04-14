---
layout: post
title: "GPT-2 Small from Scratch: Three Pretraining Experiments"
description: "Baseline, then optimizer × warmup, then init × depth. What the training curves actually say."
tag: "Pretraining · Phase 1"
date: 2026-04-15
---

Three experiments, one model, one question: what does stability actually look like in a small language model, and what breaks it?

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

## What connects them

These three experiments test fundamentally different things — duration, optimizer choice, initialization — but they keep pointing at the same underlying structure: **the residual stream is the thing to watch**.

Warmup matters because cold `v_t` produces step sizes that hammer the residual stream before it has any structure. AdamW/no-warmup drift appears specifically at `ln_final` because unchecked weight asymmetry accumulates across 12 residual additions. Flat init's failure is that it lets each block add uncontrolled variance to the stream, making depth counterproductive.

Every signal in the training curves — gradient norm, activation std, param norm drift — is a different way of observing the same underlying quantity: whether the residual stream is in a regime where each layer can add structured, coherent signal, or whether it's fighting itself.

That framing is what Exp 04 tests directly. Pre-LN vs Post-LN changes where normalization sits relative to the residual addition — which means it changes how the stream variance behaves. The expectation, given what these three experiments show, is that Post-LN will be more fragile, and the fragility will interact with both warmup and init in predictable ways.
