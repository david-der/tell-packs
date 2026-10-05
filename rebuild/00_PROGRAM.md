# 00 · Program — what TELL is and the rules that never relax

## 1. Thesis

Every existing AI-text detector ships the same product: paste text, get a
percentage. TELL (as in a poker tell, a signal the writer did not mean to
give) treats detection as a portfolio problem instead. Many cheap,
independent, individually fallible signals each return a log-likelihood
ratio (LLR) with the spans that drove it. A fusion layer that knows which
signals are correlated adds them up. The output is an evidence report a
person can argue with: contributions, flagged spans, a stated and currently
unmeasured false-positive rate, and a "what would change this" block.

The product being rebuilt is **Phase 1, Assist mode**, plus the optional
**Phase 2 statistical signals**. Assist mode answers one question for a
writer: does my draft read like a machine wrote it, and where? It is not,
and must never become, an adjudication tool.

## 2. What is in the build

One product, three surfaces over one engine:

| Surface | Entry | Spec |
|---|---|---|
| Engine | `tell.engine.check(raw_text, profile_name, packs_dir, source)` | 11, 12, 13, 14, 15 |
| CLI | `tell check <file|->`, `tell packs`, `tell profiles` | 15 |
| Web | FastAPI app, `GET /`, `POST /api/check`, `GET /api/profiles`, `GET /healthz` | 16 |

The engine runs three signal classes:

- **S3 stylometric** (spec 12): five cheap features over sentence and
  paragraph spans. Always on. Skip below 120 words.
- **S2 statistical** (spec 13): perplexity, Binoculars ratio, burstiness
  under a local pair of small reference models through llama.cpp. Optional.
  Self-skip with a stated reason when the models are absent.
- **S4 rule packs** (spec 14): the four YAML packs in `../packs/`. Always
  on. Load unchanged.

Fusion (spec 15) assigns every signal to a correlation group, discounts all
but the strongest in each group, caps any single signal, and forbids a
document where only one signal fired from reaching the `elevated` band.

## 3. Rules that never relax [NORMATIVE]

1. **No verdicts.** No headline percentage. No pass or fail. The CLI exits 0
   on every completed check regardless of band. The web UI never shows the
   posterior as a headline.
2. **Uncalibrated means Assist-only.** Every heuristic `SignalResult`
   carries `calibrated=False`. Every report carries the sentence that the
   false-positive rate is not yet measured for this profile. Remove the
   need for the caveat with calibration corpora; never remove the caveat.
3. **A single tell is worth almost nothing.** The co-occurrence discount in
   rule packs (spec 14) and the single-fire ceiling in fusion (spec 15) are
   policy, not tuning. Tests pin both.
4. **Absence of a signal is never evidence.** A skipped signal contributes
   zero and is listed as skipped. No signal may return a negative LLR for
   "nothing found".
5. **`context_required` rules never score.** They surface spans unscored
   until a judge exists. The engine does not guess whether "spine" is
   literal.
6. **Tells decay.** Every pack carries `version`, `updated`, and
   `half_life_days`. Weight halves per half-life past `updated`, floored at
   0.25. Any rule change bumps `version`.
7. **Every signal belongs to a correlation group.** A signal whose id
   matches no group glob lands in `ungrouped` and escapes the discount. Add
   the glob in the profile when adding a signal.
8. **Offsets are code points into the normalized text.** Never raw-input
   offsets, never bytes, never UTF-16. The web front end converts.
9. **Tests never load model weights.** S2 math is tested through a stub
   backend. The real models are a developer convenience, not a test
   dependency.
10. **No humanizer.** The product never ships the countermeasure.

## 4. Decisions already taken [GUIDANCE — do not relitigate]

| Decision | Branch taken | Why |
|---|---|---|
| Signal output unit | Natural-log LLR, not a 0 to 1 score | Heterogeneous signals become addable; the arithmetic is auditable |
| Fusion in the absence of a calibration corpus | Hand-set correlation groups with harmonic discounts 1/(1+0.6k), cap ±3.0 per signal, single-fire ceiling 2.9 | The honest fallback the design prescribes; learned weights wait for labeled data |
| Band floors | 1.0 mild, 3.0 elevated, 6.0 substantial | 3.0 is about 20:1 odds; the ceiling of 2.9 sits just under it so one signal cannot cross |
| Profiles | Three built in: `default` (prior -2.0), `blog` (-1.8, lexical ×1.15), `technical` (-2.5, dampens diversity, sentence shape, paragraph uniformity, perplexity) | The technical profile exists because careful technical writing and non-native English false-fire on exactly those features |
| Normalization | Line endings, non-breaking space, zero-width space only | Curly quotes, em-dashes, and emoji are signal, not noise |
| Segmenter | Hand-written regex splitter, Markdown-aware for headings, bullets, fences | No NLP dependency; offsets must be exact and reproducible |
| S2 reference models | Qwen2.5-0.5B base (observer) and instruct (performer), Q8_0 GGUF, through llama-cpp-python | Runs on CPU in seconds; Binoculars needs a base/instruct pair |
| S2 when models absent | Signals self-skip with the reason in the report; nothing else changes | Zero-config install; the engine must work without 1.1 GB of weights |
| Raw perplexity weight | Low, and ×0.3 under the technical profile | It is the ESL and technical-writing false-positive signal |
| Rule packs | YAML, four rule types, loaded from a directory, hot-reloadable by re-reading | A house style guide is a rule pack; non-programmers can write one |
| Structural detectors | Six named detectors in code, referenced by name from YAML | Parsed-document features cannot be expressed as regex |
| Web app state | Stateless, no database, no auth | Nothing to protect until author baselines arrive; the check spends no API credit |
| Web CSS | Tailwind standalone binary, built to a committed CSS file | No Node toolchain in the repository |
| Terminal renderer | Rich | Tables and panels without a custom layout engine |
| Exit codes | 0 on any completed check; 2 on usage errors | A nonzero exit for "AI detected" would be a verdict |
| Posterior | Present in JSON as `assessment.posterior`, absent from the terminal headline | The design calls for it; the headline rule forbids showing it as a percentage |

## 5. Out of scope, and why

| Not built | Why |
|---|---|
| S1 provenance (C2PA, watermarks) | Phase 4. Also the easiest place to turn absence into an accusation. |
| S5 LLM judge | Phase 3. Until it exists, `context_required` rules surface unscored. |
| S6 vendor adapters | Phase 3. |
| S7 behavioral and process evidence | Phase 4. |
| Fast-DetectGPT curvature | Listed in the design; not implemented. Binoculars is the statistical workhorse. |
| Author baselines, calibration corpora, per-cohort FPR | Phase 2 work that has not landed. Every report says so. |
| Authentication, persistence, rate limiting on the web app | Arrive with author baselines. |
| VS Code extension | Listed in the design; not built. |
| Deployment | A single `uvicorn` process. How it is hosted is the receiving team's choice. |

## 6. Glossary

See `README.md`. The receiving repository should copy that glossary into
its own README so a reader never has to open the pack to decode a term.

## 7. Data disclaimer

`seed/ai_slop.md` and `seed/sample-draft.md` are deliberately bad text
written to trip the packs. `seed/human_essay.txt` is a short human-written
essay used only as the quiet fixture. Neither is a calibration corpus.
Numbers derived from them (the figures tests assert) are regression pins,
not measurements of anything about the world.
