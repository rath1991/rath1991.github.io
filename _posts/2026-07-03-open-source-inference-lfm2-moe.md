---
layout: post
title: "A Month Inside Open-Source Inference: One MoE, a Laptop, and a GPU"
description: "Running LiquidAI's LFM2.5-8B-A1B from a MacBook Air to an RTX 5060 Ti — throughput sweeps, quantization fidelity (perplexity vs KL-divergence), and a private eval against Claude. The reasoning I brought to each run, and where it was corrected."
tag: "Inference · LFM2 MoE"
date: 2026-07-03
---

<style>
.jx { --accd:#12407a; --sig:#B4531F; --sig-wash:#F6E7DC; --good:#2E7D5B; --acc-wash:var(--accent-hl); }
.jx .fig { background:var(--surface); border:1px solid var(--border); border-radius:10px; padding:1.1rem 1.1rem 0.5rem; margin:2rem 0; }
.jx .fig svg { display:block; width:100%; height:auto; }
.jx .fig img { border-radius:6px; }
.jx .fig .cap { font-family:var(--font-mono); font-size:0.72rem; line-height:1.55; color:var(--muted); margin:0.7rem 0 0.4rem; }
.jx .fig .cap b { color:var(--text); }
.jx .s-ink{fill:var(--text)} .jx .s-soft{fill:var(--muted)} .jx .s-acc{fill:var(--accent)}
.jx .s-accd{fill:var(--accd)} .jx .s-sig{fill:var(--sig)}
.jx .st-line{stroke:var(--border)} .jx .st-acc{stroke:var(--accent)} .jx .st-sig{stroke:var(--sig)} .jx .st-soft{stroke:var(--muted)}
.jx .box-fill{fill:var(--bg);stroke:var(--border)} .jx .box-acc{fill:var(--acc-wash);stroke:var(--accent)}
.jx .mono-svg{font-family:var(--font-mono)}
.jx .flow { margin:1.6rem 0; }
.jx .xp { position:relative; padding-left:2.6rem; padding-bottom:1.6rem; }
.jx .xp:not(:last-child)::before { content:""; position:absolute; left:0.85rem; top:1.9rem; bottom:-0.1rem; width:2px; background:var(--border); }
.jx .xp-node { position:absolute; left:0; top:0.1rem; width:1.7rem; height:1.7rem; border-radius:50%; background:var(--bg); border:1.5px solid var(--accent); color:var(--accent); font-family:var(--font-mono); font-size:0.6rem; font-weight:600; display:flex; align-items:center; justify-content:center; z-index:1; }
.jx .xp.open .xp-node { border-color:var(--sig); color:var(--sig); }
.jx .xp-head { font-family:var(--font-ui); font-size:0.98rem; font-weight:600; color:var(--text); margin:0.2rem 0 0.5rem; line-height:1.3; }
.jx .xp-head .dim { color:var(--muted); font-weight:400; }
.jx .xp-cmd { font-family:var(--font-mono); font-size:0.72rem; color:var(--text); background:var(--surface); border:1px solid var(--border); border-radius:6px; padding:0.45rem 0.6rem; margin:0 0 0.7rem; overflow-x:auto; white-space:nowrap; line-height:1.5; }
.jx .xp-cmd::before { content:"$ "; color:var(--accent); font-weight:600; }
.jx .xp-cmd .cm { color:var(--muted); }
.jx .xp-result { font-family:var(--font-mono); font-size:0.75rem; color:var(--accd); margin:0 0 0.7rem; line-height:1.55; }
.jx .xp-table { overflow-x:auto; margin:0 0 0.85rem; }
.jx .rd { margin-bottom:0.6rem; }
.jx .rd .lab { font-family:var(--font-mono); font-size:0.62rem; letter-spacing:0.1em; text-transform:uppercase; font-weight:600; display:inline-flex; gap:0.6ch; align-items:center; }
.jx .rd.read .lab { color:var(--muted); }
.jx .rd.corr .lab { color:var(--accent); }
.jx .rd.next .lab { color:var(--sig); }
.jx .rd p { margin:0.15rem 0 0; font-size:0.95rem; line-height:1.6; }
.jx .rd.next p { color:var(--muted); font-style:italic; }
.jx .grade { display:inline-block; font-family:var(--font-mono); font-size:0.62rem; font-weight:600; padding:0.05em 0.5em; border-radius:999px; border:1px solid currentColor; }
.jx .grade.lo{color:var(--sig)} .jx .grade.mid{color:var(--accent)} .jx .grade.hi{color:var(--good)}
.jx .note { margin:1.6rem 0; padding:1.1rem 1.2rem; border-radius:10px; border:1px solid var(--border); background:var(--surface); font-size:0.97rem; line-height:1.65; }
.jx .note .nl { font-family:var(--font-mono); font-size:0.66rem; letter-spacing:0.11em; text-transform:uppercase; font-weight:600; display:block; margin-bottom:0.4rem; }
.jx .note p { margin:0.3rem 0 0; } .jx .note p:first-of-type { margin-top:0; }
.jx .note.warn { border-color:var(--sig); background:var(--sig-wash); } .jx .note.warn .nl { color:var(--sig); }
.jx .note.ask { border-left:3px solid var(--accent); } .jx .note.ask .nl { color:var(--accent); }
.jx .sizes { width:100%; border-collapse:collapse; font-family:var(--font-mono); font-size:0.8rem; }
</style>

<div class="jx" markdown="1">

This isn't an anti-proprietary rant. I still reach for frontier models daily. But for a growing share of my work I want to *own the stack* — run the weights on hardware I control, know exactly what I'm giving up, and pay in electricity instead of per-token. The moment you take that seriously, "just use an open model" turns into three uncomfortable questions:

- Can I actually **run** it — on the hardware I have, at a speed I can live with?
- Can I **trust** the quantized version I'll realistically deploy?
- Can I **measure** it against a frontier model on work I care about, with an eval I believe?

So I picked one model — LiquidAI's **LFM2.5-8B-A1B**, a small mixture-of-experts — and spent about four weeks, two hours a day, five days a week, torturing it. The pace was slow on purpose. Every benchmark number threw up a *"wait, why?"*, and answering it honestly meant re-deriving roofline limits, memory math, and MoE routing by hand instead of nodding along. What follows is the notebook, in order.

Most findings follow the same loop: the raw numbers, **my read** (the reasoning I brought from what I understood of the hardware), **the correction** when I pushed on it, and a **self-grade out of 10**. Some questions I still can't answer — those are marked, and I'd genuinely like your help.

## The model on the bench

LFM2.5-8B-A1B is a mixture-of-experts model: **8.47 B total parameters, but only ~1 B active per token**. Each MoE layer has 32 experts and routes every token through just 4 of them; the earliest layers are dense. That "A1B" — *active 1B* — is the whole reason a model this capable fits on modest hardware, and it's also the source of half the weird behavior below.

<div class="fig" markdown="0">
<svg viewBox="0 0 860 300" role="img" aria-label="One MoE layer: a token flows through attention, layer norm, a router, and four of thirty-two experts.">
  <defs>
    <marker id="arw" markerWidth="9" markerHeight="9" refX="6" refY="3" orient="auto">
      <path d="M0,0 L6,3 L0,6" fill="none" class="st-soft" stroke-width="1.5"/>
    </marker>
  </defs>
  <g class="mono-svg" font-size="12">
    <rect x="18" y="118" width="92" height="64" rx="8" class="box-fill" stroke-width="1.5"/>
    <text x="64" y="146" text-anchor="middle" class="s-ink">token</text>
    <text x="64" y="163" text-anchor="middle" class="s-soft" font-size="10">embedding</text>
    <line x1="112" y1="150" x2="150" y2="150" class="st-soft" stroke-width="1.5" marker-end="url(#arw)"/>
    <rect x="152" y="86" width="188" height="128" rx="8" class="box-fill" stroke-width="1.5"/>
    <text x="246" y="106" text-anchor="middle" class="s-accd" font-size="11" font-weight="600">ATTENTION BLOCK</text>
    <text x="246" y="132" text-anchor="middle" class="s-ink" font-size="11">Q · K · V</text>
    <text x="246" y="152" text-anchor="middle" class="s-soft" font-size="10">scores → softmax</text>
    <text x="246" y="172" text-anchor="middle" class="s-ink" font-size="11">context vector</text>
    <text x="246" y="196" text-anchor="middle" class="s-soft" font-size="9.5">runs on GPU · Flash-Attn tiles this</text>
    <line x1="342" y1="150" x2="380" y2="150" class="st-soft" stroke-width="1.5" marker-end="url(#arw)"/>
    <rect x="382" y="118" width="74" height="64" rx="8" class="box-fill" stroke-width="1.5"/>
    <text x="419" y="146" text-anchor="middle" class="s-ink" font-size="11">Layer</text>
    <text x="419" y="163" text-anchor="middle" class="s-ink" font-size="11">Norm</text>
    <line x1="458" y1="150" x2="496" y2="150" class="st-soft" stroke-width="1.5" marker-end="url(#arw)"/>
    <rect x="498" y="118" width="80" height="64" rx="8" class="box-acc" stroke-width="1.5"/>
    <text x="538" y="144" text-anchor="middle" class="s-accd" font-size="11" font-weight="600">router</text>
    <text x="538" y="162" text-anchor="middle" class="s-soft" font-size="9.5">top-4 / 32</text>
    <line x1="580" y1="150" x2="626" y2="96"  class="st-acc" stroke-width="1.5" marker-end="url(#arw)"/>
    <line x1="580" y1="150" x2="626" y2="132" class="st-acc" stroke-width="1.5" marker-end="url(#arw)"/>
    <line x1="580" y1="150" x2="626" y2="168" class="st-acc" stroke-width="1.5" marker-end="url(#arw)"/>
    <line x1="580" y1="150" x2="626" y2="204" class="st-acc" stroke-width="1.5" marker-end="url(#arw)"/>
    <rect x="628" y="82"  width="120" height="26" rx="5" class="box-acc" stroke-width="1.2"/>
    <rect x="628" y="118" width="120" height="26" rx="5" class="box-acc" stroke-width="1.2"/>
    <rect x="628" y="154" width="120" height="26" rx="5" class="box-acc" stroke-width="1.2"/>
    <rect x="628" y="190" width="120" height="26" rx="5" class="box-acc" stroke-width="1.2"/>
    <text x="688" y="99"  text-anchor="middle" class="s-ink" font-size="10">expert · FFN</text>
    <text x="688" y="135" text-anchor="middle" class="s-ink" font-size="10">expert · FFN</text>
    <text x="688" y="171" text-anchor="middle" class="s-ink" font-size="10">expert · FFN</text>
    <text x="688" y="207" text-anchor="middle" class="s-ink" font-size="10">expert · FFN</text>
    <text x="688" y="238" text-anchor="middle" class="s-soft" font-size="9.5">+ 28 idle experts</text>
    <line x1="750" y1="150" x2="792" y2="150" class="st-soft" stroke-width="1.5" marker-end="url(#arw)"/>
    <text x="820" y="147" text-anchor="middle" class="s-soft" font-size="10">next</text>
    <text x="820" y="161" text-anchor="middle" class="s-soft" font-size="10">layer</text>
    <line x1="612" y1="60" x2="612" y2="252" class="st-sig" stroke-width="1.3" stroke-dasharray="5 4"/>
    <text x="612" y="52" text-anchor="middle" class="s-sig" font-size="9.5" font-weight="600">-ncmoe boundary → experts can spill to CPU</text>
  </g>
</svg>
<p class="cap"><b>One MoE layer.</b> Attention and the router stay on the GPU; the expert FFNs are what <code>-ncmoe</code> can push across to system RAM. Hold onto this picture — it explains the offload sweet spot, the Flash-Attention no-op, and the thread-count flatline further down.</p>
</div>

| Quant | Size on disk | Bits/weight |
|---|---:|---:|
| BF16 | 15.78 GiB | 16.0 |
| Q6_K | 7.20 GiB | ~6.6 |
| Q4_K_M | 4.95 GiB | ~4.8 |
| Q3_K_M | 3.66 GiB | ~3.9 |
| IQ3_XS | 3.30 GiB | ~3.3 |
| Q2_K | 2.72 GiB | ~2.6 |

Remember these sizes. On a 16 GiB card, BF16 (15.78 GiB) technically "fits" — with almost nothing left for the KV cache or compute buffers. That razor-thin headroom is a character in this story.

## llama.cpp, and two rules I kept close

Everything here runs on [llama.cpp](https://github.com/ggml-org/llama.cpp) (build `b9585`). Three binaries did the work: `llama-bench` for throughput sweeps, `llama-cli` for quick runs, `llama-perplexity` for quality. Throughput splits into **prefill** (pp, ingesting the prompt) and **decode** (tg, generating), and they behave like different animals: prefill is compute-bound, decode is memory-bandwidth-bound. Half of what follows is just internalizing that one sentence.

<div class="note ask" markdown="0">
<span class="nl">Two rules that decided a lot of the answers</span>
<p><b>The validity rule.</b> <code>llama-cli</code>'s "prompt t/s" is measured over a ~7-token prompt — fixed overhead, not real prefill. Only <code>llama-bench</code> pp/tg numbers are trustworthy.</p>
<p><b>The noise rule.</b> With ±20–90% standard deviation on many rows, don't explain a gap smaller than its own error bar.</p>
</div>

## Phase 1 — MacBook M2 Air (8 GB unified)

The project began as a humbling exercise. On an 8 GB M2 Air, BF16 was a fantasy and even Q6_K wouldn't load — the Metal working set caps out around 5.6 GB. That forced me down to Q4_K_M and IQ3_XS, and forced me to understand *why*. Unified memory means CPU and GPU share one ~100 GB/s pool with no PCIe between them — a fact that quietly invalidates a lot of desktop-GPU intuition. The chain of runs, in order:

<div class="flow" markdown="0">

  <div class="xp">
    <div class="xp-node">M1</div>
    <div class="xp-head">First runs <span class="dim">· flags on / off</span></div>
    <div class="xp-cmd">llama-cli -m LFM2.5-Q4_K_M.gguf -ngl 99 -c 256 -fa 1 <span class="cm"># then -ngl 0 -t 3,4</span></div>
    <p class="xp-result">GPU+FA prefill 34–42 · GPU decode ~17 · CPU (-ngl 0) decode ~34</p>
    <div class="rd read"><span class="lab">My read <span class="grade mid">6/10</span></span><p>FlashAttention raises prefill a lot — online softmax by tiling, computed on the fly in cache during the forward pass, never written back to GPU RAM, easing HBM traffic. But decode barely moves — why? Drop to CPU and decode nearly doubles while prefill craters: that fits a memory-bound ceiling — ~100 GB/s over ~2–3 GB of active params + KV + activations ≈ 40 t/s, and the numbers lined up. So why is GPU decode ~2× lower? Probably too little headroom — only ~5.6 GB available. And 4 threads beat 3, but pinning all 4 cores risks throttling, so I'd call 3 optimal.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>Two catches. The cli "prompt t/s" is a ~7-token measurement — overhead, not prefill, so trust <code>llama-bench</code>. And the low GPU decode is <em>working-set pressure</em> at the 5.6 GB ceiling, not a bandwidth deficit — the bus is shared, so the CPU doesn't have more bandwidth.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>The cli numbers were too clean to trust. Was I even measuring prefill? → move to llama-bench.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">M2</div>
    <div class="xp-head">Prefill under llama-bench <span class="dim">· Flash Attention</span></div>
    <div class="xp-cmd">llama-bench -m LFM2.5-Q4_K_M.gguf -p 256 -n 128 -fa 0,1 -t 4</div>
    <p class="xp-result">pp256 fa1 272 · fa0 229 · tg128 ~22–24 (flat)</p>
    <div class="rd read"><span class="lab">My read <span class="grade mid">6/10</span></span><p>The bench prefill (229–272) is far higher than the cli figure — <em>this</em> is the number that makes the model look genuinely useful for agentic work that ingests a lot of context.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>Right to trust bench over cli. But the 10× gap vs a CPU run is GPU-vs-CPU (bench defaults to <code>-ngl 99</code>), not FA — within this GPU sweep FA is only ~19%.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>If bench prefill is real and this high, does it survive more context? → KV-depth sweep.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">M3</div>
    <div class="xp-head">KV-depth sweep</div>
    <div class="xp-cmd">llama-bench -p 512 -n 128 -d 256,512,1024,2048,4096 -t 4</div>
    <p class="xp-result">pp512 47→36 · tg128 23→13 as depth 256→4096</p>
    <div class="rd read"><span class="lab">My read <span class="grade hi">8/10</span></span><p>As context shrinks, both prefill and decode rise; 1024–2048 looked like the sweet spot.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>Correct, and for the right reason — deeper KV means more compute for prefill and a bigger cache to read each decode step; decode is the more bandwidth-sensitive, so it falls hardest (23→13).</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>Decode looked thread-sensitive — how far do cores take it? → thread sweep.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">M4</div>
    <div class="xp-head">Thread sweep</div>
    <div class="xp-cmd">llama-bench -p 1024 -n 128 -t 1,2,3,4</div>
    <p class="xp-result">pp1024 21→48 (1→4 threads) · tg128 peaks ~21 at t=3, then dips</p>
    <div class="rd read"><span class="lab">My read <span class="grade hi">8/10</span></span><p>Prefill keeps climbing with threads, but decode maxes out at 3 and then falls — decode is memory-bound, so pinning all 4 cores stresses compute and queues background processes, dropping tps.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>Prefill is compute-bound → scales with cores; decode is bandwidth-bound → saturates around 2–3, and the 4th thread just adds contention. Fair — though the t=4 dip (±9.2) is inside the noise, so "stops improving past 3" is the safe claim.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>Threads capped decode; could a bigger micro-batch push prefill instead? → ubatch.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">M5</div>
    <div class="xp-head">Micro-batch sweep</div>
    <div class="xp-cmd">llama-bench -p 1024 -n 128 -b 1024 -ub 128,256,512 -t 4</div>
    <p class="xp-result">pp1024 ~46 / 43 / 46 (flat) · tg128 drifts 14→18</p>
    <div class="rd read"><span class="lab">My read <span class="grade hi">8/10</span></span><p>ubatch gave only a marginal decode bump — but ubatch should help compute (prefill), not decode. Needs clarification.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>ubatch is a prefill-only knob (prompt tokens per micro-batch) — it can't touch one-token-at-a-time decode, so that drift is noise; the instinct to retry was right. Prefill was flat because at pp1024 on CPU there's little batching headroom left.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>ubatch barely moved the needle — maybe fit, not flags, is the lever. → try the smaller IQ3 build fully on GPU.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">M6</div>
    <div class="xp-head">IQ3 fully on GPU</div>
    <div class="xp-cmd">llama-bench -m LFM2.5-IQ3_XS.gguf -ngl 99 -p 512 -fa 0,1 -t 4</div>
    <p class="xp-result">pp512 307–314 · tg128 ~23 · FA ≈ flat</p>
    <div class="rd read"><span class="lab">My read <span class="grade mid">7/10</span></span><p>The importance-weighted IQ3 quant has marginally higher prefill than Q4_K_M and, at 3.3 GiB, loads fully into the GPU.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>Right. On this M2 the edge is fit + fewer weight-bytes, not raw compute — there are no low-precision matmul units, so quant actually adds a little dequant work. FA is flat here because at short context attention isn't the bottleneck.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>IQ3 fits with headroom. What if I deliberately shove experts onto the CPU? → ncmoe.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">M7</div>
    <div class="xp-head">Expert offload on unified memory</div>
    <div class="xp-cmd">llama-bench -m LFM2.5-IQ3_XS.gguf -fa 1 -ncmoe 0,2,4,8,32</div>
    <p class="xp-result">pp512 314→234 · tg128 23→10 as 0→32 experts offloaded</p>
    <div class="rd read"><span class="lab">My read <span class="grade mid">6/10</span></span><p>Since it's unified memory, moving MoE experts to the CPU shouldn't matter for prefill — yet it keeps dropping, and the fall is even worse than running everything on CPU. I think tokens have to keep switching between attention on the GPU and experts on the CPU, and that transfer is slow.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>The conclusion (CPU compute is the bottleneck) is right, but there's no transfer here — unified memory, one pool, no PCIe, no copy. The cost is that offloaded layers run on slow CPU matmul plus a Metal↔CPU re-sync every layer. Offload only pays when the model doesn't fit — and IQ3 already does.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>Offload backfired with nothing to relieve. To actually see it <em>help</em>, I needed real VRAM pressure and a real bus. → the RTX 5060 Ti.</p></div>
  </div>

</div>

The verdict for the M2 Air: prefill-strong, decode-modest — useful for context-heavy ingestion, less so for long generation. Two structural facts explain most of the runs above: on unified memory *capacity* (how much fits in the ~5.6 GB working set) is the constraint, not the shared bandwidth; and the honest prefill numbers come from `llama-bench`.

## Phase 2 — RTX 5060 Ti (16 GB, Blackwell)

Then I moved everything to a real GPU: an RTX 5060 Ti, 16 GiB VRAM, 64 GB of system RAM. Now BF16 fits (barely), `-ncmoe` has a real PCIe bus to cross, and offload finally has something to relieve. The full run sequence:

<div class="flow" markdown="0">

  <div class="xp">
    <div class="xp-node">E1</div>
    <div class="xp-head">KV-depth sweep <span class="dim">· BF16, no FA</span></div>
    <div class="xp-cmd">llama-bench -m LFM2.5-BF16.gguf -p 1000 -n 256 -d 256,512,1024,2048,4096,8192,20000</div>
    <div class="xp-table"><table>
      <thead><tr><th>KV depth</th><th align="right">prefill t/s</th><th align="right">decode t/s</th></tr></thead>
      <tbody>
        <tr><td>256</td><td align="right">1157.18</td><td align="right">21.62</td></tr>
        <tr><td>512</td><td align="right">1133.59</td><td align="right">21.47</td></tr>
        <tr><td><b>1024</b></td><td align="right"><b>4355.00</b></td><td align="right"><b>41.87</b></td></tr>
        <tr><td>2048</td><td align="right">4074.68</td><td align="right">38.12</td></tr>
        <tr><td>4096</td><td align="right">4025.14</td><td align="right">38.08</td></tr>
        <tr><td>8192</td><td align="right">3181.25</td><td align="right">34.90</td></tr>
        <tr><td>20000</td><td align="right">2606.40</td><td align="right">32.94</td></tr>
      </tbody>
    </table></div>
    <div class="rd read"><span class="lab">My read <span class="grade mid">6/10</span></span><p>At 256–512 tokens maybe not all CUDA cores are used in the tiling-based matmul (this isn't FA). By 1024–4096 the GPU is fully utilised while the KV cache and activations still fit in VRAM. Past that, the growing context forces some weights/activations out to CPU RAM, slowing things down.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>The decay half is right — KV growth eventually spills to the host. But "cores idle below 1024" is weak: 512-token prefill is already highly parallel. That clean 3.75× step at <em>exactly</em> d1024 looks like a kernel-path or allocation threshold — and I still can't prove it (open question below).</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>That d1024 jump nagged, but first: with headroom this tight, what frees it? → ncmoe sweep.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">E2</div>
    <div class="xp-head">Expert offload sweep <span class="dim">· BF16</span></div>
    <div class="xp-cmd">llama-bench -p 1000 -n 256 -ncmoe 0,2,4,6,8,16,32</div>
    <div class="xp-table"><table>
      <thead><tr><th>-ncmoe</th><th align="right">prefill t/s</th><th align="right">decode t/s</th></tr></thead>
      <tbody>
        <tr><td>0</td><td align="right">1217.72</td><td align="right">22.02</td></tr>
        <tr><td>2</td><td align="right">1217.52</td><td align="right">22.05</td></tr>
        <tr><td><b>4</b></td><td align="right"><b>3068.92</b></td><td align="right"><b>78.28</b></td></tr>
        <tr><td>6</td><td align="right">2159.20</td><td align="right">64.51</td></tr>
        <tr><td>8</td><td align="right">1671.78</td><td align="right">55.06</td></tr>
        <tr><td>16</td><td align="right">957.24</td><td align="right">34.15</td></tr>
        <tr><td>32</td><td align="right">650.62</td><td align="right">22.99</td></tr>
      </tbody>
    </table></div>
    <div class="rd read"><span class="lab">My read <span class="grade hi">8/10</span></span><p>32 experts, 4 active, first couple of layers likely dense. With the weights nearly filling VRAM there's almost no headroom. Offloading some experts to CPU RAM over PCIe relieves that and frees the GPU for compute, so both prefill and decode jump — up to ncmoe=4. Beyond 4, the post-attention activations travel to CPU RAM over the slow PCIe, through experts on the even-slower CPU, and back; that traffic overwhelms the gain.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>The shape confirms it. ncmoe 0 and 2 are identical because 2 experts don't free <em>enough</em>; 4 crosses the capacity threshold that lets the KV cache and compute buffers stay resident (+152% prefill, +256% decode). It's a threshold, not a gradient.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>ncmoe=4 was magic at 20k depth. Does it survive far longer context? → push depth to 100k.</p></div>
  </div>

  <div class="xp open">
    <div class="xp-node">E3</div>
    <div class="xp-head">Extended context <span class="dim">· ncmoe=4, d20k→100k</span></div>
    <div class="xp-cmd">llama-bench -p 1000 -n 256 -ncmoe 4 -d 20000,50000,100000</div>
    <p class="xp-result">pp1000 2827 → 1107 → 72 · tg256 74.8 → 69.8 → 7.7</p>
    <div class="rd read"><span class="lab">My read</span><p><em>I left this one blank.</em> Prefill collapsed ~40× from 20k→100k while decode fell only ~10×, and I didn't have an explanation at the time.</p></div>
    <div class="rd corr"><span class="lab">The answer, later</span><p>Prefill cost scales with attention over the whole context and, at 100k, the KV cache blows past VRAM headroom into host spill — so it falls off a cliff. Decode only touches one token's work per step, so it degrades far more gently.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>The cliff smelled like attention-memory. Could Flash Attention rescue it? → toggle FA.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">E4·5</div>
    <div class="xp-head">Flash Attention <span class="dim">· with and without ncmoe, d20000</span></div>
    <div class="xp-cmd">llama-bench -d 20000 -fa 0,1 <span class="cm"># then: -ncmoe 4 -fa 0,1</span></div>
    <div class="xp-table"><table>
      <thead><tr><th>config</th><th align="right">fa</th><th align="right">prefill t/s</th><th align="right">decode t/s</th></tr></thead>
      <tbody>
        <tr><td>no ncmoe</td><td align="right">0</td><td align="right">259</td><td align="right">7.15</td></tr>
        <tr><td><b>no ncmoe</b></td><td align="right"><b>1</b></td><td align="right"><b>2472</b></td><td align="right"><b>32.90</b></td></tr>
        <tr><td>ncmoe = 4</td><td align="right">0</td><td align="right">2826.96</td><td align="right">74.81</td></tr>
        <tr><td>ncmoe = 4</td><td align="right">1</td><td align="right">2831.58</td><td align="right">74.80</td></tr>
      </tbody>
    </table></div>
    <div class="rd read"><span class="lab">My read <span class="grade lo">5/10</span></span><p>Flash attention does online softmax by tiling — slices of Q, K, V loaded and computed in on-chip cache near the CUDA cores, never written back to RAM until done, cutting GPU-RAM traffic. If you move some experts to the CPU, that disturbs it, because some activations have to move from CPU back to the GPU cache, hampering FA's gains.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>The FA mechanics are right, but the causal link isn't — attention stays on the GPU regardless of ncmoe. FA is a memory-pressure valve: without offload at 20k it saves enormously (9.6×); with ncmoe=4 the pressure is already relieved, so FA has nothing left to do — the offload, not the experts-in-FA idea, is doing the work.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>FA only helped when nothing else had. Could micro-batch matter at depth? → ubatch.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">E6</div>
    <div class="xp-head">Micro-batch at depth <span class="dim">· ncmoe=4, fa=1, d20000</span></div>
    <div class="xp-cmd">llama-bench -p 512 -n 128 -ncmoe 4 -fa 1 -d 20000 -ub 256,512,1024</div>
    <p class="xp-result">pp512 228 · 226 · <b>2138</b> at ub 256 / 512 / 1024 — a 9.4× cliff</p>
    <div class="rd read"><span class="lab">My read <span class="grade lo">5/10</span></span><p>Leveraging more CUDA cores and pushing it toward compute-bound.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>Partly — but the real trigger is that ub=1024 ≥ the 512-token prompt, so it processes in a single pass; at 256 the prompt splits into chunks that <em>each</em> re-read the 20k KV cache. Same cliff I saw on the Mac, sharper.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>Does the whole ncmoe4 + fa1 recipe hold as prompt size grows? → a cross-check.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">E7</div>
    <div class="xp-head">Prompt-size × depth cross-check</div>
    <div class="xp-cmd">llama-bench -p 4096 -n 256 -ncmoe 4 -fa 1 -d 10000,20000,50000</div>
    <p class="xp-result">pp4096 359 · 2690 · 1887 across depth 10k / 20k / 50k</p>
    <div class="rd read"><span class="lab">Why I ran it</span><p><em>No separate hypothesis here</em> — a deliberate sanity sweep to confirm the ncmoe=4 + fa=1 recipe held as I varied prompt size and depth. It did.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>Config held. Now the question I actually cared about: which quant is fastest? → quant comparison.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">E8</div>
    <div class="xp-head">Quant comparison <span class="dim">· all fully in VRAM</span></div>
    <div class="xp-cmd">llama-bench -p 4096 -n 256 <span class="cm"># run per quant</span></div>
    <div class="xp-table"><table>
      <thead><tr><th>quant</th><th align="right">size</th><th align="right">prefill t/s</th><th align="right">decode t/s</th></tr></thead>
      <tbody>
        <tr><td>Q6_K</td><td align="right">7.20 GiB</td><td align="right">8185</td><td align="right">222.80</td></tr>
        <tr><td>Q4_K_M</td><td align="right">4.95 GiB</td><td align="right">9215</td><td align="right">278.43</td></tr>
        <tr><td>Q3_K_M</td><td align="right">3.66 GiB</td><td align="right">9434</td><td align="right">294.16</td></tr>
        <tr><td>IQ3_XS</td><td align="right">3.30 GiB</td><td align="right">9437</td><td align="right">291.01</td></tr>
        <tr><td>Q2_K</td><td align="right">2.72 GiB</td><td align="right">7974</td><td align="right"><b>315.45</b></td></tr>
      </tbody>
    </table></div>
    <div class="rd read"><span class="lab">My read <span class="grade mid">7/10</span></span><p>Both prefill and decode should rise as precision drops — quantization pushes you toward compute-bound (faster low-precision FLOPs) and eases memory (fewer bytes moved per second).</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>Half of that holds. Decode is bandwidth-bound, so smaller does win — Q2_K has the fastest decode (315). But prefill is compute-bound, and dequant-kernel efficiency isn't monotonic in bit-width: Q2_K's kernel is less optimized than the K-quants, which is why it lands lowest on prefill despite being the smallest file. Fewer bits ≠ faster prefill.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>Prefill and decode pulled in opposite directions. Could more CPU threads change the offload picture? → thread sweep.</p></div>
  </div>

  <div class="xp open">
    <div class="xp-node">E9</div>
    <div class="xp-head">Thread sweep <span class="dim">· ncmoe=4</span></div>
    <div class="xp-cmd">llama-bench -p 4096 -n 256 -ncmoe 4 -t 2,4,6,8,10,12,14,16</div>
    <p class="xp-result">pp4096 ~3135 · tg256 ~78 — dead flat from t=2 to t=16</p>
    <div class="rd read"><span class="lab">My read</span><p><em>I wrote "needs explanation" and moved on.</em></p></div>
    <div class="rd corr"><span class="lab">The answer, later</span><p>With ncmoe=4 the offloaded expert FFN runs on the CPU and is RAM-bandwidth-bound — two threads already saturate the memory bus, so extra cores have nothing to do. Everything else is on the GPU.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>Threads were a dead end — informatively so. One more throughput idea before I stopped chasing speed. → a different backend.</p></div>
  </div>

  <div class="xp">
    <div class="xp-node">E10</div>
    <div class="xp-head">Vulkan + TurboQuant KV cache</div>
    <div class="xp-cmd">llama-bench --vulkan -ctk turbo4 -ctv turbo3 -ncmoe 4 -p 512 -n 128 -d 10000,20000</div>
    <p class="xp-result">pp512 577 · 586 · tg128 43.2 · 42.6 — flat 10k→20k</p>
    <div class="rd read"><span class="lab">My read <span class="grade mid">7/10</span></span><p>KV quantization shrinks the cache without losing much quality (per the TurboQuant paper), so you can hold more context at respectable prefill — there's still some GPU headroom and more efficient weight transfer. The decode numbers, though, are hard to believe.</p></div>
    <div class="rd corr"><span class="lab">Correction</span><p>Mechanism's right, and the flat curve is exactly what a compressed KV cache should give. Trust the shape; the 43 t/s decode goes on the re-measure list before I quote it.</p></div>
    <div class="rd next"><span class="lab">→ next</span><p>But every number so far is about <em>speed</em>. Fast isn't the same as faithful. → perplexity &amp; KL-divergence.</p></div>
  </div>

</div>

## Quantization fidelity: "how fast" vs "how faithful"

Up to here I'd been optimizing throughput. But the reason you quantize is to deploy the small file — so the real question is whether the small file still *thinks* like the big one. I measured perplexity on wikitext-2, and KL-divergence of each quant's token distribution against the BF16 original. They tell very different stories.

<div class="fig" markdown="0">
<svg viewBox="0 0 860 380" role="img" aria-label="Perplexity stays nearly flat across quantization levels while KL-divergence climbs steadily, spiking at Q2_K.">
  <g class="mono-svg">
    <line x1="80" y1="60"  x2="800" y2="60"  class="st-line" stroke-width="1"/>
    <line x1="80" y1="180" x2="800" y2="180" class="st-line" stroke-width="1"/>
    <line x1="80" y1="300" x2="800" y2="300" class="st-line" stroke-width="1"/>
    <text x="72" y="64"  text-anchor="end" class="s-sig"  font-size="10">.40</text>
    <text x="72" y="184" text-anchor="end" class="s-sig"  font-size="10">.20</text>
    <text x="72" y="304" text-anchor="end" class="s-sig"  font-size="10">0</text>
    <text x="810" y="64"  text-anchor="start" class="s-accd" font-size="10">38</text>
    <text x="810" y="184" text-anchor="start" class="s-accd" font-size="10">35</text>
    <text x="810" y="304" text-anchor="start" class="s-accd" font-size="10">32</text>
    <polyline fill="none" class="st-acc" stroke-width="2.5" points="80,261.6 224,259.6 368,285.6 512,268.8 656,214.4 800,69.6"/>
    <g class="s-acc"><circle cx="80" cy="261.6" r="4"/><circle cx="224" cy="259.6" r="4"/><circle cx="368" cy="285.6" r="4"/><circle cx="512" cy="268.8" r="4"/><circle cx="656" cy="214.4" r="4"/><circle cx="800" cy="69.6" r="4"/></g>
    <polyline fill="none" class="st-sig" stroke-width="2.5" points="80,300 224,292.7 368,272.5 512,208.6 656,148 800,67.7"/>
    <g class="s-sig"><circle cx="80" cy="300" r="4"/><circle cx="224" cy="292.7" r="4"/><circle cx="368" cy="272.5" r="4"/><circle cx="512" cy="208.6" r="4"/><circle cx="656" cy="148" r="4"/><circle cx="800" cy="67.7" r="4"/></g>
    <g font-size="11" class="s-soft" text-anchor="middle">
      <text x="80" y="330">BF16</text><text x="224" y="330">Q6_K</text><text x="368" y="330">Q4_K_M</text>
      <text x="512" y="330">Q3_K_M</text><text x="656" y="330">IQ3_XS</text><text x="800" y="330">Q2_K</text>
    </g>
    <g font-size="11" text-anchor="start">
      <line x1="150" y1="356" x2="178" y2="356" class="st-acc" stroke-width="2.5"/><text x="185" y="360" class="s-accd">Perplexity (right)</text>
      <line x1="470" y1="356" x2="498" y2="356" class="st-sig" stroke-width="2.5"/><text x="505" y="360" class="s-sig">Median KL-div vs BF16 (left)</text>
    </g>
  </g>
</svg>
<p class="cap"><b>Two instruments, two verdicts.</b> Perplexity (blue) says every quant down to IQ3 is basically fine — Q4_K_M even edges out BF16, within error. KL-divergence (rust) tells the truth: the per-token distribution drifts steadily, and Q2_K has left the building.</p>
</div>

| Quant | PPL | Median KLD | Same top-p |
|---|---:|---:|---:|
| Q6_K | 33.01 | 0.012 | 91.6% |
| **Q4_K_M** | 32.36 | 0.046 | 85.6% |
| Q3_K_M | 32.78 | 0.152 | 76.2% |
| IQ3_XS | 34.14 | 0.253 | 70.4% |
| Q2_K | 37.76 | 0.387 | 65.2% |

Read the Q2_K row slowly. Its perplexity (37.8 vs BF16's 33.0) looks like a modest tax. But its *median* KL-divergence is 0.387 — where anything past ~0.2 means the distributions have meaningfully parted ways — and it agrees with BF16's top token only 65% of the time.

<div class="note ask" markdown="0">
<span class="nl">My take</span>
<p><b>Q4_K_M is the quality floor.</b> Everything below it looks fine on the coarse metric and is quietly broken on the sensitive one.</p>
</div>

## An eval I actually trust

Standard benchmarks are contaminated — the questions have leaked into training sets, and the scores tell you about memorization as much as capability. So I hand-wrote my own: **140 questions across 7 categories**, skewed toward the inference-engineering work I actually do (KV-cache math, quantization gates, VRAM budgeting), with a handful of general-domain controls. It's private — I'll share it one-on-one but not publish it, precisely so it *stays* uncontaminated.

| Category | Count | Weight | Scored by |
|---|---:|---:|---|
| Coding | 25 | 20% | pass@1 (pytest) |
| Tool use | 25 | 20% | trajectory match |
| Math | 20 | 15% | deterministic |
| Logical | 20 | 15% | deterministic |
| Info generation | 20 | 15% | LLM judge |
| Planning | 15 | 10% | LLM judge |
| Termination | 15 | 10% | loop detection |

Scoring is weighted so the *final answer* carries 0.6 and can't be rescued by partial credit — a wrong answer never passes. Subjective categories escalate to **Claude Opus 4.8 as an LLM-judge**, emitting a terse yes/no per rubric dimension. A taste of the questions:

- **CODE-001** (pass@1): implement `is_palindrome(s)` — alphanumeric only, case-insensitive, empty string counts. Graded by running the model's code against 5 hidden tests.
- **TOOL-012** (trajectory, budget 2): find the on-disk GiB of a model at 4.5 bpw — but you must fetch the unknown parameter count *first*. The trap: calling the size tool before you have the params, or guessing 7.0.
- **INFO-009** (judge): two sources report 620 vs 580 tok/s for the *same* RTX 4090. Correct behavior is to *flag the conflict* and give a range — not silently average or pick one.

### LFM2.5 vs Claude, on my turf

I ran a Claude model as the frontier baseline, then every LFM2.5 quant through the identical 140 questions.

<div class="fig" markdown="0">
<svg viewBox="0 0 860 340" role="img" aria-label="Mean weighted score: the Claude baseline at 0.77 and all five LFM2.5 quants clustered near 0.70.">
  <g class="mono-svg">
    <line x1="70" y1="160" x2="815" y2="160" class="st-sig" stroke-width="1.2" stroke-dasharray="5 4"/>
    <text x="815" y="156" text-anchor="end" class="s-sig" font-size="10">pass ≥ .50</text>
    <line x1="70" y1="280" x2="815" y2="280" class="st-line" stroke-width="1.2"/>
    <rect x="92" y="94.4" width="78" height="185.6" rx="4" class="s-accd"/>
    <text x="131" y="86" text-anchor="middle" class="s-accd" font-size="12" font-weight="600">.773</text>
    <text x="131" y="298" text-anchor="middle" class="s-ink" font-size="11">Claude</text>
    <g class="s-acc">
      <rect x="235" y="110.8" width="78" height="169.2" rx="4"/><rect x="360" y="110.9" width="78" height="169.1" rx="4"/>
      <rect x="485" y="111.2" width="78" height="168.8" rx="4"/><rect x="610" y="109.7" width="78" height="170.3" rx="4"/>
      <rect x="735" y="110.9" width="78" height="169.1" rx="4"/>
    </g>
    <g text-anchor="middle" font-size="12" class="s-acc" font-weight="600">
      <text x="274" y="102">.705</text><text x="399" y="102">.705</text><text x="524" y="103">.703</text><text x="649" y="101">.709</text><text x="774" y="102">.705</text>
    </g>
    <g text-anchor="middle" font-size="11" class="s-ink">
      <text x="274" y="298">BF16</text><text x="399" y="298">Q6_K</text><text x="524" y="298">Q4_K_M</text><text x="649" y="298">IQ3_XS</text><text x="774" y="298">Q2_K</text>
    </g>
    <text x="452" y="326" text-anchor="middle" class="s-soft" font-size="10">mean weighted score across 140 questions · higher = better</text>
  </g>
</svg>
<p class="cap"><b>~7 points behind frontier — and a flat line across quants.</b> The open model lands closer to Claude than I expected. But the five blue bars, BF16 through Q2_K, are indistinguishable. That flat line is the problem.</p>
</div>

**The good news:** LFM2.5 scores ~0.70 to Claude's 0.77 — a ~7-point gap to a frontier model, on questions written to be hard and uncontaminated. For a model that runs on a hobbyist GPU, that's genuinely close.

**The unsettling news:** the eval can't tell the quants apart. Q2_K (0.705) scores the same as BF16 (0.705). If this scoreboard were my only instrument, I'd happily ship the 2.7 GiB Q2_K build and feel smart about it.

<div class="fig" markdown="0">
<img src="/assets/img/lfm_eval_heatmap.png" alt="Heatmap of mean score, models by category">
<p class="cap"><b>Models × categories.</b> Claude leads almost everywhere; the LFM quants are near-identical rows — the same near-flat behavior the bar chart shows, category by category.</p>
</div>

<div class="fig" markdown="0">
<img src="/assets/img/lfm_eval_delta_vs_baseline.png" alt="Per-category delta of each LFM quant versus the Claude baseline">
<p class="cap"><b>Where the gap actually lives.</b> Per-category delta vs the baseline — the open model is competitive on some categories and clearly behind on others, which the single mean score hides.</p>
</div>

## Why I don't trust the rosy quant numbers

<div class="note warn" markdown="0">
<span class="nl">Read this before you ship a 2-bit model</span>
<p>My throughput benchmarks love Q2_K. My 140-question eval can't distinguish it from BF16. Both are <em>seductive and wrong.</em> KL-divergence — the one instrument sensitive enough to see per-token drift — says Q2_K agrees with the original only 65% of the time. A coarse pass/fail eval rewards getting the gist right while the distribution quietly rots. <b>Neither speed nor a headline pass-rate is sufficient evidence to trust a quant.</b></p>
</div>

<div class="fig" markdown="0">
<img src="/assets/img/lfm_quant_degradation.png" alt="Eval score vs quantization level per category — nearly flat">
<p class="cap"><b>The blind spot, drawn.</b> Eval score vs quantization level, per category — nearly flat all the way down to Q2_K. The eval simply cannot see the degradation that KL-divergence measures.</p>
</div>

The resolution is almost embarrassingly simple: a pass/fail score is a *low-resolution* instrument. Many questions have a right answer a slightly-degraded model can still stumble into — so it passes, while its token distribution has drifted far enough to matter in open-ended, multi-step, or agentic use where errors compound. KLD sees that drift; a scoreboard doesn't. So my adoption rule: **Q4_K_M is the floor, and Q3_K_M / Q2_K are off the table** no matter how good their benchmarks look.

Since the whole point is trusting the *process*, here's where mine is still weak:

- Every dataset item is currently `human_verified: false`. I wrote them; I haven't audited them cold.
- A handful of agentic questions fell back to *schema-inferred* mock tool responses rather than authored ones — those trajectories aren't authoritative yet.
- My baseline is a Claude model judged by Claude Opus — which **breaks my own "use a different model family as judge" rule.** The LFM runs honor it; the baseline doesn't.
- I haven't run judge calibration against human labels yet, so I can't quote an agreement rate.
- Q3_K_M is in the perplexity/KLD study but never made it into the eval — a gap I only noticed writing this up.

## Open questions and next steps

This phase is done, for now. What I'd genuinely like help with:

- Why does prefill quadruple at *exactly* `d1024` (E1)?
- Why was decode ~2× lower on the M2's GPU than its CPU?
- Are the Vulkan TurboQuant-KV decode numbers real?
- What actually trips the `n_ubatch` cliff at 1024 (E6)?

I have hypotheses for all four and proof for none. And the to-do list: human-verify the dataset and author real mock tool responses; run judge calibration to ≥80% agreement; add Q3_K_M to the eval; re-measure the throughput rows I flagged as suspect; stand up a proper different-family baseline.

The meta-lesson, four weeks in: choosing an open model to depend on isn't one measurement, it's three that disagree — *can I run it*, *can I trust the small version*, and *can I measure it honestly*. The disagreements are where the truth is.

*Corrections welcome — especially on the four open questions. That's not false modesty, it's a bug report.*

</div>
