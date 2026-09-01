# Running a 27B on a mini-PC: the complete measured story

Notes for people serving local models on their own hardware: a 27B-class
model on a unified-memory mini-PC, at full native context, with every dial
pinned by a paired measurement instead of a forum hunch. Everything below is
an adaptation of KyaniteLabs' published Strix Halo measurements — [the blog
article](https://kyanitelabs.tech/blog/qwen-27b-strix-halo-complete) and the
canonical repo
[KyaniteLabs/qwen38-27b-strix-halo](https://github.com/KyaniteLabs/qwen38-27b-strix-halo)
(MIT), which holds the configs, the instruments, the methodology, and the raw
logs behind every number. These notes restate the journey for mini-PC
operators — the context ladders, the KV verdict, the frozen config, the
measured knees, the bugs — with every number's original validity label
intact. Nothing here re-measures or re-brands anything; if a number here ever
disagrees with the canonical repo, the canonical repo wins.

Labels · The rig · The frozen config · The four speeds · The context ladders
· The KV verdict · The measured knees · Two bugs, on the record · The chassis
· Method rules worth stealing · The honest gap · Reproduce it · Provenance

---

## Read the labels first

The canonical repo's methodology defines three evidence classes, and every
number below keeps the one it was published with:

- **CLEAN** — paired design, n satisfies the rules, no known confound.
  Quotable as a headline.
- **DIRECTIONAL** — real signal but underpowered, or measured with a custom
  instrument. Quote only with the label attached.
- **PILOT** — n=1 or uncontrolled. It can justify running the real
  experiment; it never gates a decision. Several maps below are PILOT-class
  on purpose: the canonical repo published them as *maps, not laws*.

Arithmetic on measured inputs is labeled **ARITHMETIC** where it appears.

`Provenance: KyaniteLabs/qwen38-27b-strix-halo METHODOLOGY.md, "Evidence classes".`

## The rig

One box, owned outright, no cloud. Every headline number below was measured
on it, and a number quoted without these conditions is a misquote:

| Part | Value |
|---|---|
| Machine | GMKtec EVO-X2 mini-PC, $1,400 |
| APU | AMD Ryzen AI Max+ 395 (Strix Halo), Radeon 8060S, gfx1151 |
| Memory | 96 GB unified LPDDR5X; 64 GB GTT visible to the GPU |
| Backend | ROCm 7.2.4 (HIP build) |
| Engine | llama.cpp, b10435-era dflash branch build (9d57ce4) |
| Model | Qwen3.8-27B (Apache-2.0), Unsloth UD-Q4_K_XL dynamic quant |
| Serving | single host, temperature 0 for every graded run |

`Provenance: KyaniteLabs/qwen38-27b-strix-halo METHODOLOGY.md "Conditions of every headline number" + README; blog "the complete measured story" (2026-08-21). VALIDITY: the conditions line every number below inherits.`

## The frozen config

The arc closed on 2026-08-21 with every dial pinned by a pre-registered
measurement. This is the machine as it ships:

| Dial | Final value | Evidence behind it |
|---|---|---|
| Weights | UD-Q4_K_XL | quant ladder; paired gates |
| KV cache | q4_0 (K+V) | 08-18 paired gate (below); 08-21 paired trade vs q8_0 |
| Context | 262,144 (native max) | retrieval proven AT the ceiling: 261,130/262,144 tok |
| Speculation | draft-mtp + ngram-mod, n-max 12 | 4-way paired walls (table below) |
| Thinking | off by default; hard lane thinks | 3-band knee maps (below) |
| Needle retrieval | exact, all depths, both seeds, at the ceiling | 13/13 post-fix + format sweep |
| Vision | fixed + upstream-validated | 6/6 real-UI screenshots; fix validated 9/9 paired |

`Provenance: KyaniteLabs/qwen38-27b-strix-halo README, "ARC CLOSED 2026-08-21" table; results/config-27b-2026-08-21/.`

The last open dial was speculation, measured with paired walls on three
byte-identical 200-token prose problems. The serving config held:

| Spec state | Mean wall per 200-token answer |
|---|---|
| mirror (= serving): draft-mtp + ngram-mod, n-max 12 | **15.1 s** |
| spec off | 17.8 s |
| ngram only | 17.7 s |
| mtp only | 15.1 s |

MTP does the work; ngram adds nothing on prose but is free when it misses,
and a five-arm sweep had already shown the n-max 12 cap costs nothing
(c30 median 1.5 s uncapped vs 1.5 s capped; spec off 7.5 s — spec decoding
is worth 5x on repetition-heavy tasks, labeled as such).

`Provenance: KyaniteLabs/qwen38-27b-strix-halo results/config-27b-2026-08-21/README.md (pre-registered mirror verdict) + results/spec-sweep-2026-08-19/. VALIDITY: PILOT (n=1 per cell, paired rows, counting-class rows diagnostic-only).`

The second-to-last dial was the KV trade, measured the same night at 131k
context: q8_0 was faster on decode and every warm query, and cost real
memory. The decision kept the light cache:

| quant | needle @52% of 130,719 tok | warm quote/exist/sum | GTT loaded |
|---|---|---|---|
| q8_0 | HIT exact, clean stop | 8.0 / 1.3 / 9.1 s | 21,592 MB |
| q4_0 | HIT exact, clean stop | 8.4 / 1.5 / 11.9 s | 19,589 MB |

One honest discrepancy, carried rather than smoothed: the canonical
close-out states the full-window trade as "8 GB vs 1-3 s on warm
follow-ups," while the repo's own per-config arithmetic (9.1 GB q8_0 vs
4.8 GB q4_0 at 262k, below) puts the delta at ~4.3 GB, and the blog wrote
"~4 GB". Both readings point at the same decision — keep the light cache —
but the two published numbers do not agree, so both are stated here.

`Provenance: KyaniteLabs/qwen38-27b-strix-halo results/config-27b-2026-08-21/README.md (dial 1 + close-out); README "ARC CLOSED" table; blog "the complete measured story". VALIDITY: PILOT (n=1 per cell; fill walls 930/915 s parity; window cut after the decision cells completed).`

## The four speeds

One headline tok/s is the wrong mental model for this box. The canonical
speed sheet carries four regimes, each its own number:

| Regime | Rate | What it is |
|---|---|---|
| Reading (prefill) | ~260-390 tok/s | ingesting your prompt (live traffic 262/305; f16-era ladder 390) |
| Writing, short answers | ~20-26 tok/s | quick replies, math answers |
| Writing, long sustained answers | ~12-13 tok/s | 200-700-token prose generations |
| Speed-trick bursts | 150-312 tok/s | **ARTIFACT** — speculative decode replaying repetitive text; never real work |

Product translation: a 200-word answer lands in ~15 seconds, a long
700-word answer in ~60, and prompts are read at hundreds of tokens per
second. Cold one-shot throughput measured 59.7 tok/s (count-to-30); warm
back-to-back repeats hit 148-163 only because ngram speculation replays the
bench's own repetition — real conversational traffic is **11-24 tok/s**
(creative long-form ~11-14, structured/code ~29-40), and that is
memory-bandwidth physics for a 27B dense model on LPDDR5X, not a tunable.

`Provenance: KyaniteLabs/qwen38-27b-strix-halo README "SPEED SHEET" + "Read the warm numbers honestly". VALIDITY: CLEAN band structure; individual rates DIRECTIONAL (custom c30 instrument, labeled per regime).`

## The context ladders

Context became nearly free in three measured steps, each rung gated before
promotion:

1. **96k for free (08-15).** The GQA architecture keeps the KV cache small
   — ~2.1 GB per 32k tokens — so the ladder 32k → 64k → 96k ran at literally
   zero speed cost; 128k probed fine at 6.1 GB.
2. **262k for the price of a flag (08-16).** Quantizing the cache to q8_0
   (K+V) halved it, so the full native 262,144 window fit in *less* memory
   than 96k of f16. Each rung (96k → 160k → 192k → 262k) was A/B-gated on
   the real-task battery; the warm band held (150-158 on the 262k row); the
   one labeled cost was
   prefill, ~299 tok/s vs ~390 on f16 (q8 dequant on cache reads).
3. **q4_0 KV (08-18).** The paired gate below halved the cache again —
   9.1 → 4.8 GB at the full window — with no measured accuracy cost.

`Provenance: KyaniteLabs/qwen38-27b-strix-halo README updates 2026-08-15/16/18; blog "one week local" + "the native context ceiling, for free". VALIDITY: ladder structure CLEAN (each rung gated); per-rung rates DIRECTIONAL.`

Why this works at all is architecture, not luck: Qwen3.8-27B is a hybrid —
64 layers, of which only 16 do full attention; the other 48 are Gated
DeltaNet (linear attention with a fixed-size running state). The KV cache
exists only in the 16 attention layers: 4 KB per token per layer, 64 KB per
token total — 17.2 GB at f16 across a full 262k window, ~9.1 GB at q8_0,
~4.8 GB at q4_0. ARITHMETIC on the published per-token sizes.

`Provenance: blog "Lab Notes: the basin was a bug" (2026-08-20) + "the model that can't forget but can't remember" (2026-08-19). VALIDITY: ARITHMETIC (deterministic on the 16-layer attention structure).`

**The ceiling itself was measured, not assumed.** Exact needle retrieval at
every tested depth, two seeds, up to **261,130 of 262,144 tokens — 99.6% of
the window**, the literal ceiling; 13/13 cells post-fix across both seeds.
Format did not matter (prose and code exact; described-in-words prompts
returned every part in order).

`Provenance: KyaniteLabs/qwen38-27b-strix-halo results/needle-format-2026-08-19/nativemax-results.log + README arc table; blog "the complete measured story". VALIDITY: PILOT per cell (n=1 depth cells, two seeds), stated by the canonical repo as a map, not a measurement.`

**Load once, query many.** A 198k-token prefix cost **1,818 s (~30 min)**
to prefill cold, then four warm follow-ups against the cached prefix all
stopped clean: planted-code retrieval, quote (16 s), yes/no (9 s), summary
(27 s). The format-sweep re-run measured quote 23.8 s, yes/no 10.0 s,
summary 17.5 s. The product shape: **loading is expensive; maintaining is
cheap** — which is the whole economic argument for keeping a document
resident instead of re-reading it.

`Provenance: blog "the model that can't forget" (2026-08-19, quote-probe) + "the complete measured story" (2026-08-20 quote-rerun). VALIDITY: PILOT (single-run cells, single haystack). Cold prefill rate through the needle wall: 109.5 tok/s at ~198k, needle-class number (full prompt + answer, not a bench harness).`

## The KV verdict

The question: every 262k row in the community hardware map ran q4-KV; the
rig served q8. Does dropping KV to 4 bits cost accuracy? The answer came
from a pre-registered, self-executing promotion gate — if the q4 arm matched
needle hits and posted zero corruption tripwires, the unit file flips;
anything less, the old champion restores itself. It flipped.

| Layer | Design | Result |
|---|---|---|
| Task accuracy | GSM8K, n=60 stratified, paired | both arms 96.67%, exact McNemar p=1.0, zero discordants; q4 slightly *faster* (18.78 vs 19.47 s mean) |
| Deep-context integrity | needles at 50%/90% depth, identical ~198k haystack, same-session control | 0/2 hits on **both** arms (pre-bug instrument era — see bugs below); zero tripwires on q4, one on q8 |
| Retrieval boundary | needle at 25% depth (where q8 historically retrieves) | FOUND, clean stop — the boundary did not move |

Net: **~47% smaller KV cache (4.3 GB saved, 9.1 → 4.8 GB) at the same
262,144-token context, zero measured accuracy cost.** The bits arithmetic:
q8_0 encodes ~8.5 bits per KV value, q4_0 ~4.5. The q5_0 and q5_1 arms
matched the same accuracy; q4_0 won on kept memory. The tripwire is a
finish_reason anomaly counter kept armed on every serving hour; its one trip
that night fired on the *q8* arm (a 40-token answer that never stopped).

`Provenance: KyaniteLabs/qwen38-27b-strix-halo results/kv-curve-2026-08-19/ + README "UPDATE 2026-08-18"; blog "Lab Notes: the KV verdict" (2026-08-18) + "the model that can't forget" (2026-08-19). VALIDITY: paired cells CLEAN for the parity claim (p=1.0, zero discordants); needle cells PILOT (n=2 stations + one boundary probe).`

The gate also caught its own instrument, and the correction is a method law
now: the haystack was built from an estimate (5,800 paragraphs × ~45 tokens
= "262k"), but the server's own counter said **198,227 tokens**. The paired
comparison was untouched — both arms prefilled the identical haystack — but
every "needles at 262k" claim was overstating the instrument. Rule: **label
instruments with the counter at the source, not the generator's estimate.**

`Provenance: blog "Lab Notes: the KV verdict", "The miss we caught in ourselves"; KyaniteLabs/qwen38-27b-strix-halo README (server prompt_n 198,227).`

## The measured knees

Two results transfer to almost any rig, and they are a pair of knees: where
thinking stops paying, and where capping it stops costing.

**Verdict night (2026-08-17/18).** Fifty Omni-MATH problems
(25 hard / 15 mid / 10 easy), every problem asked under all three thinking
budgets, cell order rotated, exact McNemar on the discordant pairs:

| Thinking budget | Accuracy (n=50) | Median wall per problem |
|---|---|---|
| Uncapped (explicit 10^6 override) | 44% | 367 s |
| 1024 tokens | 40% | — |
| 512 tokens | 40% | 49 s |

Caps were statistically free (uncapped vs 512: p=0.73; 0.63 and 1.0 on the
other contrasts) and **protective**: left uncapped, the model thought itself
to death on **26 of 50** problems — reasoning past the output ceiling and
emitting no answer, which scores as wrong. On the hardest band
(difficulty >= 5.0) every cell scored a flat 16%: capability-bound, not
budget-bound. A cap cannot take what the model never had. One caveat the
canonical repo carries: a capped cell measures accuracy under *forced early
termination*, so the observed knee sits at or above the model's true knee —
read it conservatively.

`Provenance: KyaniteLabs/qwen38-27b-strix-halo results/omnimath-paired-2026-08-18.jsonl + METHODOLOGY.md; blog "Lab Notes: verdict night" (2026-08-18). VALIDITY: paired n=50, one benchmark, one night — a datapoint, not a law (the canonical repo's own label).`

The shipped default that earned its number that night: **1024-token budget
for margin, 512 where turn speed matters**, thinking off by default.

**Where thinking *does* pay.** On a 40-problem hard/medium set at
temperature 0, thinking-on rescued **15/40** hard problems vs **4/40**
thinking-off; on HumanEval-30 the same budget bought *nothing* (28/30
identical both ways). A wider GSM8K sweep (150 paired problems, six budget
cells) landed within one problem across every budget — 147 / 147 / 147 /
146 / 146 / 146. And on LiveCodeBench-30 (public subset, Wilson 95% CI
49-81%), thinking rescued 5 of the 15 problems where no-think produced no
code at 4096 tokens. The rule the canonical repo runs: reasoning pays
exactly where the task is hard; publish the knee, not the hype. The knee is
hardware-specific — measure it on your box before shipping a default.

`Provenance: blog "Lab Notes: the measured-knees method" (2026-08-20) + "67% LiveCodeBench-30" (2026-08-20) + "93% HumanEval" (2026-08-19); KyaniteLabs/qwen38-27b-strix-halo results/README.md (gsm8k-budget correct counts) + results/deep-context-2026-08-18/kneemap-gsm8k-hard.log (think_med=241ch, caps >=512 non-binding). VALIDITY: DIRECTIONAL (subsets, n=30-40 per arm class).`

## Two bugs, on the record

Both bugs are part of the machine's story, and both changed how the rig is
operated. They are preserved here because they are the two failure shapes
every fork-runner will eventually meet.

### The upstream bug that blamed the model

Mid-arc, long context and vision silently broke on this GPU class. The
first read blamed the model — wrong. The real cause was one upstream
llama.cpp change (`c7d8722`, host buffers on HIP integrated GPUs): prompts
over ~2k tokens split across decode calls, produced NaN logits, and the
model sampled `/` forever. A six-prompt vision battery read **0/6 before a
revert, 5/6 after** (five hits sub-second; all six stopped clean after);
text HumanEval on the same binary stayed 28/30. The revert was bisected on
this hardware, reported upstream
([issue 26209](https://github.com/ggml-org/llama.cpp/issues/26209)), and the
upstream fix was later validated on this silicon with **9/9 identical paired
answers**. Post-fix, the same needle instrument retrieved 13/13 exact across
both seeds at the ceiling; every pre-fix long-context row in the canonical
repo is quarantined as an instrument-era artifact. The canonical repo's own
verdict: *"Degeneration was the instrument, not the model."* — and a wrong
early number stayed published with its correction attached, not quietly
deleted.

`Provenance: blog "Lab Notes: we reverted a llama.cpp regression" (2026-08-19) + "the basin was a bug" (2026-08-20) + "the complete measured story"; KyaniteLabs/qwen38-27b-strix-halo results/vision-2026-08-19/, results/needle-format-2026-08-19/, results/pr25863-validation-2026-08-21/. VALIDITY: vision battery DIRECTIONAL (n=6); 13/13 remap PILOT per cell, two seeds.`

### The one-line bug (fork drift)

On the box's second lane — LFM2.5-2.6B (Q4_K_M), a small fast model with a
DSpark draft model for speculative decoding — the server hard-aborted on the
first decode step:
`GGML_ASSERT(t_layer_inp[il] != nullptr)`, in the draft graph reserve. Stock
upstream llama.cpp with the same model pair ran clean, which cleared the
draft model and upstream, and left one suspect: the production fork. It had
drifted — upstream's support had gained three pieces the fork lacked, down
to a single line in the layer loop:

```c
res->t_layer_inp[il] = cur;
```

Without it, the draft graph reserved memory for layer inputs that stayed
null; the first null check aborted the server. The ported fix was a 2-file
diff (+18/−8), and the paired re-measurement (n=3 arms, temperature 0,
byte-identical outputs on a greedy spot check):

| Arm | Runs (tok/s) | Median | vs baseline |
|---|---|---|---|
| no draft | 93.5 / 94.1 / 93.9 | 93.9 | — |
| F16 draft | 102.7 / 110.5 / 130.0 | **110.5** | **+17.7%** |
| Q8_0 draft | 65.6 / 74.4 / 80.3 | 74.4 | **−20.8%** |
| F16 draft, 1,200-tok generation | 141.4 | **141.4** | +50.6%* |

*Cross-horizon comparison, labeled: the baseline was not run at 1,200
tokens. Acceptance ran 0.64-0.80 on short prose and 0.89 on the sustained
generation — the speedup grows with horizon. Two receipts that drafting
changed the clock, not the answers: a full GSM8K first-100 strict run scored
57.0% with the draft vs 53.3% on a separate spec-off control (unpaired,
statistically indistinguishable, p≈0.65).

The counter-intuitive keeper: **quantizing the draft made it slower than no
draft at all** — the Q8_0 draft landed 21% below baseline with essentially
the same acceptance (~0.80), because dequantizing the small draft network
every drafting step costs more than the memory savings buys back on this
iGPU. The transferable lesson is bigger than either number: fork drift dies
quietly, and the fix for "our fork crashes on upstream features" is to diff
against upstream's version of the same feature, not to debug the model.

`Provenance: blog "The One-Line Bug That Crashed Our Fast Lane" (2026-08-25); upstream llama.cpp PR #27383 (ported as branch dspark-lfm2-fix). VALIDITY: speed arms PILOT (n=3, one prose prompt; long-horizon n=1); correctness receipts DIRECTIONAL (unpaired GSM8K arms, n=1 byte check).`

## The chassis

The thermal envelope is measured doctrine, not folklore:

- **Sustained serving edge:** 90-92 °C package at 120 W, fans at 100%, for
  hours of generation with zero throttling. Steady-warm beats heat-cycling;
  no ritual cooldown breaks.
- **The 39-second crash (08-15):** stacking a `git clone` plus a 14-job HIP
  compile on top of live 27B serving killed the box in 39 seconds (journal
  ended mid-write; no panic, no OOM, no thermal trip; only a button drain
  revived it). The 724,067-line post-mortem ranked power-delivery on the
  stock 230 W brick as the leading hypothesis. Doctrine: **build first,
  serve second, never stacked** — this chassis is not a workstation, but a
  legit servant. The redemption: a 41-round agentic soak ran 41/41 clean
  riding the 92 °C boost-throttle edge with 45-second recovery.
- **The one hard trip:** back-to-back half-hour prefills (~6 °C above the
  envelope) tripped the EC at `temp=98C -> fan=100%` — the load was the bug,
  and the crash log stands next to the corrected science in the canonical
  repo.

`Provenance: blog "one week local" ("39 seconds") + "the basin was a bug" ("then the box cut power") + "verdict night" ("the envelope it rode in on"). VALIDITY: post-mortem rankings DIRECTIONAL (falsifiable-hypothesis ordering, n=1 event).`

The fan story is its own companion repo. The firmware exposes no standard
Linux fan control, and the stock auto curve saturates around ~60% duty /
~3260 RPM at 90 °C and above — leaving headroom unused exactly where it is
needed. The fix chain, all documented in
[KyaniteLabs/evo-x2-ec](https://github.com/KyaniteLabs/evo-x2-ec): the
kernel `ec_sys` module loads read-only by default and *silently discards*
writes while looking writable (the trap that cost two debugging sessions);
the register map came from the ACPI DSDT work in
nathanmarlor/strix-halo-fan-control (reverse-engineered on the Bosgame M5 —
the same Sixunited AXB35-02 board — and validated byte-for-byte on the
EVO-X2); and the firmware reverts manual duty writes within seconds, so only
a daemon re-asserting every ~2 s holds. Tuned-curve measurements: peak
96.5 °C vs 97.5-97.8 °C stock on a 105 s probe, fan2 at 4,539 RPM vs the
3,260 stock ceiling, 100% duty by 82 °C, quiet idle preserved; re-baselined
at **−3.5 to −5.8 °C peak** (n=3 probes, with the honest caveat that the
stock arm was not re-run). The register map itself (P-MODE `0x31`, fan duty
`0x33`/`0x34`, tachometer pairs, package temp `0x70`, the fan3 tach
extension, and the tach-tearing read rule) lives in the companion repo —
this doc does not duplicate it.

`Provenance: KyaniteLabs/evo-x2-ec README (register map, traps, measurements); KyaniteLabs/qwen38-27b-strix-halo re-baseline update 2026-08-16 (fan daemon delta); blog "one week local" (EC detective arc). VALIDITY: DIRECTIONAL (n=1-3 probes, stock arm not re-run — labeled in the canonical repo).`

## Method rules worth stealing

Each rule below was learned in the canonical repo by publishing a number
that later had to be corrected. The full list lives in the canonical
`METHODOLOGY.md`; these are the ones that transfer to any local rig:

1. **Paired cells, exact McNemar.** Every problem sees every arm; compare
   within-problem, on the discordant pairs — not averages across sets.
2. **n>=3 per arm in one thermal window** before any speed number is called
   CLEAN; cold and warm are separate bands, never averaged. n=1 is
   PILOT-class: a map, never a law.
3. **Audit the server's defaults before trusting a control arm.** The
   invalid first cap experiment had a "no cap" cell silently inheriting the
   server's 2048 reasoning budget — the tell was six byte-identical thinking
   lengths. Control arms send explicit overrides, never omissions.
4. **Output caps must not bind mid-think**, or length-death masquerades as a
   quality regression; keep `finish_reason` in every row and a tripwire
   armed on every serving hour.
5. **Label instruments with the counter at the source** (server
   `prompt_n`), not the generator's estimate.
6. **Time-per-task beats decode tok/s.** The quant ladder closed on it:
   Q3@128k decodes faster than Q4 (63-64 vs 59.7 cold) yet finishes
   identical correct tasks 35-50% slower because its thinking is ~2x more
   verbose (7.6-7.7 s vs 10.6-16.1 s per task); Q2_K_XL lost on both axes
   (54.7 cold; dequant cost exceeds the bandwidth saved). A faster pipe
   feeding dumber tokens loses to a slower pipe feeding sharper ones.
7. **Publish the negative results and the corrections.** The dead ends, the
   quarantined instrument-era rows, and the walked-back numbers are part of
   the canonical record, not embarrassing footnotes.

`Provenance: KyaniteLabs/qwen38-27b-strix-halo METHODOLOGY.md + README quant-ladder rows + docs/findings.md; blog "verdict night" ("the twist") + "one week local" ("faster tokens != better"). VALIDITY: the rules are method commitments; the quant-ladder numbers quoted within are DIRECTIONAL (tpt n=2-3).`

## The honest gap

- **One box.** Every number comes from one $1,400 mini-PC. No cross-machine
  replication yet — the canonical methodology says to treat absolute values
  as this-box measurements and compare only within-condition.
- **Small n in the maps.** The needle maps are n=1 per depth cell (two
  seeds); the spec walls and the 08-21 KV trade are n=1 paired cells; the
  vision battery is n=6; the time-per-task arms run n=2-3. The CLEAN bar is
  only met where it is claimed.
- **Pre-fix long-context rows are quarantined**, and the 50k-token cells
  were never re-run on the fixed build — the canonical repo declines to
  claim retrieval at those lengths rather than guess.
- **The 8 GB / ~4 GB KV-trade discrepancy** (frozen-config section) is
  unresolved in the published sources; both numbers are carried above.
- **No power/energy claims.** The canonical methodology does not publish
  measured power numbers for this box yet.
- **One model, one quant family.** Qwen3.8-27B, Unsloth dynamic quants,
  llama.cpp at one build era. Build-era matters: spec-decode behavior moved
  between llama.cpp builds during the arc.

## Reproduce it

The canonical repo is the instrument — clone it and point the bench at your
running server. Zero hand-typed numbers: the bench prints a filled NUMBERS
block for the canonical README's community table, which takes rig rows by
PR, inside or outside the published bands.

```sh
git clone https://github.com/KyaniteLabs/qwen38-27b-strix-halo
cd qwen38-27b-strix-halo

# one-command bench: cold/warm c30, time-per-task battery, quality suite
bash components/bench/bench.sh
```

- **Serve it:** `components/strix-halo-env.sh` + `components/qwen38-27b-halo-mtp.ini`
  on llama.cpp b10435+ — the frozen config is the README quickstart verbatim
  (UD-Q4_K_XL, `--spec-type draft-mtp,ngram-mod`, n-max 12, n-min 24,
  thinking off by default).
- **Measure it:** `components/bench/tpt-battery.py` (5 auto-graded tasks —
  the instrument behind the knees) and `tpt-style.py`; methodology and
  evidence-class rules in `METHODOLOGY.md`; raw paired JSONL under `results/`.
- **The measurement kit add-ons:** the same two instruments this doc's
  sibling gifts ship as stand-alone repos —
  [resonant-stack-bench](https://github.com/simongonzalezdc/resonant-stack-bench)
  (serving throughput + battery) and
  [resonant-delegation-bench](https://github.com/simongonzalezdc/resonant-delegation-bench)
  (certified walk-away floors for job classes), both built to make these
  measurements one command on your box.
- **The chassis:** EC register map, `ec_sys` trap, and fan daemon recipe in
  [KyaniteLabs/evo-x2-ec](https://github.com/KyaniteLabs/evo-x2-ec).
- **The full narrative:** the [complete-story
  article](https://kyanitelabs.tech/blog/qwen-27b-strix-halo-complete) and
  the [one-week
  prequel](https://kyanitelabs.tech/blog/qwen-27b-strix-halo-one-week-local),
  plus the lab notes linked per section above.

Label rules the canonical repo asks of any posted row: warm counts under
ngram are repetition-assisted — the label is part of the number; n>=3 per
arm in one thermal window; cold = first exposure.

## Provenance, credits, license

All measurements, instruments, and raw logs are KyaniteLabs' published work:
the [qwen38-27b-strix-halo](https://github.com/KyaniteLabs/qwen38-27b-strix-halo)
repo (MIT — README, METHODOLOGY.md, results/), the
[evo-x2-ec](https://github.com/KyaniteLabs/evo-x2-ec) companion repo, and
the [kyanitelabs.tech blog](https://kyanitelabs.tech/blog) articles cited in
the stamps above. Nothing here re-measures or re-brands them; these notes
are an adaptation written for people building on local hardware, published
under the same MIT license (see `LICENSE`).

The stack stands on upstream work the canonical repo credits in full:
Qwen3.8-27B (Qwen team, Apache-2.0); Unsloth dynamic quants; llama.cpp (MIT)
— the engine, the bug reports (#26209, #27154 throughput discussion), and
the fixes (#25863, #27383); nathanmarlor/strix-halo-fan-control (MIT) for
the EC daemon and DSDT-derived register map. If a number here ever
disagrees with the canonical repo, the canonical repo wins — flag an issue
there or here.
