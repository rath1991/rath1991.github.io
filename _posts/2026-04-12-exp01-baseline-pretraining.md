---
layout: post
title: "Exp 01: Baseline GPT-2 Small — 1B tokens, from scratch"
description: "The reference run. Everything else in this series compares against this."
tag: "Exp 01 · Pretraining"
date: 2026-04-12
---

Before you can understand what breaks, you need to know what healthy looks like. That's what this run is — 1B tokens, GPT-2 Small, standard config, nothing unusual. Every experiment after this one will be measured against it.

## The setup

GPT-2 Small: 12 layers, 12 heads, 768-dimensional embeddings. 85.7M parameters after accounting for weight tying. Trained on FineWeb-Edu — a filtered educational subset of Common Crawl. 1B tokens, which at ~30,500 tokens/sec on an RTX 5060 Ti with BF16 autocast comes out to about 9 hours.

| | |
|---|---|
| Effective batch | 524K tokens (8 micro × 64 accum × 1024 ctx) |
| LR | 6e-4 → 6e-5 cosine, warmup 50 steps |
| Optimizer | AdamW β1=0.9 β2=0.95 wd=0.1 clip=1.0 |
| Precision | BF16 autocast |

One thing worth noting on BF16: this isn't just a memory trick. On Blackwell tensor cores it gives a genuine 2.4× throughput improvement over FP32 (13k → 30.5k tok/s). The reason to prefer it over FP16 is dynamic range — BF16 has the same exponent width as FP32, so you don't need a GradScaler and overflow is essentially a non-issue.

## What the run looked like

![Exp 01 — full training overview](/assets/img/exp01_overview.png)
<p class="img-caption">8 metrics across 1907 steps. Each subplot is one signal worth watching.</p>

## Observations

**Loss** started near 11, which is exactly what you'd expect — random initialization over a vocabulary of 50,257 gives an expected loss of ln(50257) ≈ 10.82. It dropped to around 4 by the end of 1B tokens. Train and val tracked closely throughout, no sign of overfitting, which makes sense — you can't meaningfully overfit a 124M parameter model on 1B tokens from a large varied corpus. The final ~4 is slightly above the ~3.0–3.2 target for this setup, worth keeping in mind as a reference point for the variants.

**Learning rate** warmed up steeply from ~0 to 6e-4 in 50 steps, then gradually underwent cosine decay to 6e-5 by the end of training. Shape is exactly as designed.

**Gradient norm** showed spikes during warmup. This is expected and worth understanding correctly: random initialization puts the model in a high-curvature region of the loss landscape where gradient magnitudes are large and directions are incoherent across batches — initialization state, erratic early updates, input data distribution seen by a model that hasn't yet built any meaningful internal representations. Warmup is the mitigation here, not the cause — it keeps the step size small while those large incoherent gradients exist. Post-warmup the norm stabilises, implying stable training and healthy gradient flow through the rest of the run.

## Why this matters

This run establishes the baseline everything else gets compared to. The next experiment (Exp 02) tests what happens when you take away weight decay (Adam vs AdamW) and skip the warmup entirely — both are conditions where these same metrics should look noticeably different. The whole point is that once you know what healthy looks like, the deviations become readable.

That's the goal of this series. Not just to train models — to learn to read them.
