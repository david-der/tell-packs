# 12 · S3 stylometric signals

**Purpose.** Five cheap, explainable authorship features computed from the
segmented document with no language model: sentence-length shape, paragraph
uniformity, punctuation profile, windowed lexical diversity, and
paragraph-initial transition density. Each returns a heuristic
log-likelihood ratio (LLR, natural-log units) with a rationale shown
verbatim in the report.

**Marker.** [NORMATIVE] unless a section says [GUIDANCE].
**Requirement prefix.** `STY`.
**Tests that hold it to account.** `tests/test_stylometric.py` (5 tests), plus
`test_end_to_end_orders_fixtures` in `tests/test_fusion.py`.
**Depends on.** `11_SPEC_CORE.md`: `Document`, `Sentence`, `Paragraph`,
`Span`, `SignalResult`, `Profile.weight_for`, `ramp`, the `Signal` protocol,
and the `prose_sentences` property. **Feeds.** `15_SPEC_FUSION_REPORT_CLI.md`
through the `regularity` and `register` correlation groups defined in
`11_SPEC_CORE.md` (`PROF` requirements).

## 1. Principles

1. Every feature here is weak alone. Fusion places four of the five in the
   `regularity` correlation group and the fifth (`transitions`) in
   `register`, so they cannot triple-count. A rebuild that adds a sixth
   stylometric signal must assign it a group (never-list item 7).
2. Below `MIN_WORDS` (120) every stylometric signal skips with a stated
   reason. Stylometry has no power on short text.
3. All thresholds are hand-set provisional baselines, not measured cohort
   distributions. Every result carries `calibrated=False` (the default on
   `SignalResult`), so these run in Assist mode only.
4. The `technical` profile multiplies the LLR of `diversity`,
   `sentence_shape`, and `paragraph_uniformity` by factors below 1 because
   those three are the known false-positive features on non-native English
   and careful technical prose. The multipliers are defined in
   `11_SPEC_CORE.md` (`PROF`), not here; this spec only requires that the
   signal applies `profile.weight_for(self.id)`.
5. Offsets in every `Span` are code-point indices into the normalized
   document text (`Document.text`). Never raw-input, byte, or UTF-16 offsets.

## 2. Out of scope (not built) [GUIDANCE]

The product design lists more S3 features than the reference build
implements. The following are **not built** and the rebuild must not add
them to pass this spec: function-word distribution against a baseline,
syntactic tree-depth variance, Oxford-comma consistency, colon-before-list
rate, em-dash rate (this lives in the `claudeisms` rule pack as a `rate`
rule, same correlation group, so it is not double-counted), heading
density, bullet-list ratio, bolded lead-in bullets, and tricolon frequency
(the last four are S4 structural detectors, see `14_SPEC_RULEPACKS.md`).

## 3. The `StylometricSignal` shell

One dataclass wraps one feature function. There are no subclasses.

```python
@dataclass
class StylometricSignal:
    id: str                       # "stylometric.<feature>"
    version: str                  # "1" for every shipped feature
    compute: Callable[[Document], tuple[float, float, list[Span], str]]
    cost: Cost = Cost.FREE
    latency_budget_ms: int = 50
    requires: set[str] = {"text"}
    half_life_days: int | None = None
```

`compute(doc)` returns `(llr, confidence, spans, rationale)`.

### 3.1 `score(doc, profile)` [NORMATIVE]

1. If `doc.word_count < MIN_WORDS`, return
   `SignalResult.skip(self.id, reason, self.version)` where `reason` is
   exactly:

   ```
   document too short for stylometry ({doc.word_count} words < {MIN_WORDS})
   ```

   Example for a five-word input: `document too short for stylometry (5 words < 120)`.
2. Otherwise call `compute(doc)`, multiply the returned LLR by
   `profile.weight_for(self.id)`, and return
   `SignalResult(signal_id=self.id, llr=llr, confidence=confidence, spans=spans, rationale=rationale, version=self.version)`.
   `calibrated` and `skipped` keep their defaults (`False`).
3. The profile multiplier applies to the LLR only. Confidence, spans, and
   rationale are unchanged. A negative LLR is multiplied too.
4. A feature that has too little material for its own analysis (sections
   4 to 7 give the floors) returns LLR `0.0` with a low confidence and a
   rationale saying why. It is **not** marked skipped. Only the
   `MIN_WORDS` check produces a skipped result.

## 4. `stylometric.sentence_shape` [NORMATIVE]

Models are smooth; humans spike. Low coefficient of variation (CV) of
sentence length and negative excess kurtosis both point machine-ward.

**Input.** `lengths = [float(s.word_count) for s in doc.prose_sentences if s.word_count > 0]`.
Only sentences inside paragraphs of kind `prose` count. Bullet items,
headings, and code are excluded.

**Floor.** If `len(lengths) < 8`, return
`(0.0, 0.2, [], f"only {len(lengths)} prose sentences — not enough for shape analysis")`.

**Statistics.**

```python
mean = statistics.fmean(lengths)
cv = statistics.pstdev(lengths) / mean if mean else 0.0    # population stdev
kurt = _excess_kurtosis(lengths)
```

`_excess_kurtosis` is verbatim:

```python
def _excess_kurtosis(values: list[float]) -> float:
    n = len(values)
    if n < 4:
        return 0.0
    mean = statistics.fmean(values)
    var = statistics.pvariance(values, mean)
    if var == 0:
        return -3.0
    m4 = sum((v - mean) ** 4 for v in values) / n
    return m4 / (var**2) - 3.0
```

Population moments throughout. Zero variance returns `-3.0` (the minimum
excess kurtosis), which is what a document of identical-length sentences
produces.

**LLR.**

```python
llr = 1.2 * ramp(0.50 - cv, 0.0, 0.25) + 0.6 * ramp(-kurt, 0.5, 2.0)
```

The first term is 0 at CV 0.50 or above and reaches 1.2 at CV 0.25 or
below. The second term is 0 at excess kurtosis -0.5 or above and reaches
0.6 at -2.0 or below. Maximum LLR is 1.8.

**Confidence.** `min(0.9, 0.3 + 0.02 * len(lengths))`.

**Spans.** Always empty.

**Rationale.** Exactly:

```
sentence-length CV {cv:.2f} (human prose typically ≥ 0.50), excess kurtosis {kurt:+.1f} over {len(lengths)} sentences
```

The kurtosis is printed with an explicit sign (`+0.8`, `-0.6`).

**Worked values** (computed by running the reference signal on the seed
files with the `default` profile):

| Input | sentences | CV | kurtosis | LLR | confidence |
|---|---|---|---|---|---|
| `seed/ai_slop.md` | 15 | 0.27 | -0.6 | +1.138 | 0.60 |
| `seed/human_essay.txt` | 23 | 0.77 | +0.8 | 0.000 | 0.76 |
| the `uniform` string of `test_uniform_sentences_score_higher_than_bursty` | 20 | 0.00 | -3.0 | +1.800 | 0.70 |
| the `bursty` string of the same test | 12 | 0.98 | -1.2 | +0.281 | 0.54 |

## 5. `stylometric.paragraph_uniformity` [NORMATIVE]

**Input.** `lengths = [float(len(p.text.split())) for p in doc.paragraphs if p.kind == "prose"]`.

**Floor.** If `len(lengths) < 4`, return
`(0.0, 0.2, [], f"only {len(lengths)} prose paragraphs — not enough")`.

**Statistics.** `mean = fmean(lengths)`; `cv = pstdev(lengths) / mean if mean else 0.0`.

**LLR.** `0.9 * ramp(0.45 - cv, 0.0, 0.30)`. Zero at CV 0.45 or above,
0.9 at CV 0.15 or below.

**Confidence.** `min(0.8, 0.3 + 0.05 * len(lengths))`.

**Spans.** Always empty.

**Rationale.** Exactly:

```
paragraph-length CV {cv:.2f} over {len(lengths)} paragraphs (uniform ≈ machine)
```

**Worked values.** `seed/ai_slop.md`: 5 paragraphs, CV 0.24, LLR +0.640,
confidence 0.55. `seed/human_essay.txt`: 6 paragraphs, CV 0.50, LLR 0.000,
confidence 0.60. Source: running the reference signal.

## 6. `stylometric.punctuation` [NORMATIVE]

Consistency tells. Perfectly typographic quotes read model-consistent;
mixed curly and straight quotes read as a paste-and-edit history and push
the LLR negative (human-ward). Em-dash rate is deliberately absent here
(see section 2).

**Counts** over `doc.text`, with `words = max(1, doc.word_count)`:

| Name | Pattern or expression |
|---|---|
| `curly` | count of `re.findall(r"[‘’“”]", text)` (U+2018, U+2019, U+201C, U+201D) |
| `straight` | count of `re.findall(r"[\"']", text)` |
| `semis` | `text.count(";") / words * 1000` |

**LLR and rationale parts**, starting from `llr = 0.0` and an empty
`parts` list, applied in this order:

1. If `curly + straight >= 6`, let `curly_frac = curly / (curly + straight)`.
   - If `curly_frac >= 0.95`: `llr += 0.35`; append
     `f"quotes 100% typographic ({curly}/{curly + straight}) — model-consistent"`.
   - Else if `0.15 < curly_frac < 0.85`: `llr -= 0.4`; append
     `"mixed curly/straight quotes — edit-history artifact, reads human"`.
   - Otherwise nothing.
2. If `semis > 4.0`: `llr += 0.3 * ramp(semis, 4.0, 10.0)`; append
   `f"semicolon rate {semis:.1f}/1k words (human p95 ≈ 4)"`.

**Rationale.** `"; ".join(parts)` if `parts` is non-empty, else exactly
`punctuation profile unremarkable`.

**Confidence.** Always `0.5`. **Spans.** Always empty.

Range: LLR is between -0.4 and +0.65.

**Worked values.** Both seed files: LLR 0.000, rationale
`punctuation profile unremarkable`. Source: running the reference signal.

## 7. `stylometric.diversity` [NORMATIVE]

Windowed type-token ratio plus hapax rate. Known to false-fire on
non-native and plain technical prose, hence the low weights and the
`technical` profile multiplier.

**Tokens.** `tokens = [w.lower() for w in re.findall(r"[a-zA-Z']+", doc.text)]`.
ASCII letters and apostrophes only; digits and non-ASCII letters are not
tokens.

**Floor.** If `len(tokens) < 200`, return
`(0.0, 0.2, [], "under 200 alphabetic tokens — diversity not meaningful")`.
Note this floor is on alphabetic tokens, not on `word_count`; a 150-word
document passes the `MIN_WORDS` gate and still lands here.

**Statistics.**

```python
window = 100
ttrs = [len(set(tokens[i : i + window])) / window
        for i in range(0, len(tokens) - window + 1, window)]   # non-overlapping windows
mattr = statistics.fmean(ttrs)
counts = Counter(tokens)
hapax = sum(1 for c in counts.values() if c == 1) / len(counts)   # share of types seen once
```

Windows are non-overlapping and a trailing partial window is dropped.

**LLR.** `0.5 * ramp(0.66 - mattr, 0.0, 0.12) + 0.3 * ramp(0.45 - hapax, 0.0, 0.20)`.
Zero when `mattr >= 0.66` and `hapax >= 0.45`; maximum 0.8.

**Confidence.** Always `0.5`. **Spans.** Always empty.

**Rationale.** Exactly:

```
windowed TTR {mattr:.2f} (human longform ≈ 0.70), hapax rate {hapax:.2f}
```

**Worked values.** `seed/ai_slop.md`: TTR 0.82, hapax 0.84, LLR 0.000.
`seed/human_essay.txt`: TTR 0.78, hapax 0.75, LLR 0.000. Source: running
the reference signal. Both seed files are too varied to trip this feature;
it exists for long, flat documents.

## 8. `stylometric.transitions` [NORMATIVE]

Stock connective openers ("However," "Moreover," "Additionally,") at the
start of sentences and paragraphs.

**The list**, verbatim and in this order:

```python
TRANSITIONS = (
    "however", "moreover", "additionally", "furthermore", "in conclusion",
    "overall", "ultimately", "in summary", "that said", "notably",
    "importantly", "consequently", "therefore", "in addition",
)
```

**The regex**, built from the list:

```python
_TRANSITION_RE = re.compile(
    r"^(?:" + "|".join(re.escape(t) for t in TRANSITIONS) + r")\b[,:]?",
    re.IGNORECASE,
)
```

Anchored at the start of the string it is matched against. An optional
trailing comma or colon is included in the match, so the span covers
`However,` not `However`.

**Counting.**

1. Sentence hits: for each `s` in `doc.prose_sentences`, if
   `_TRANSITION_RE.match(s.text)` matches, increment `sent_hits` and append
   `Span(s.start, s.start + m.end(), note="transition opener")`. The span
   is the opener only, offset into `Document.text`.
2. Paragraph hits: `prose = [p for p in doc.paragraphs if p.kind == "prose"]`;
   `para_hits` is the count of `p` where
   `_TRANSITION_RE.match(p.text.lstrip("#> ").lstrip())` matches.
3. `n_sent = max(1, len(doc.prose_sentences))`; `density = sent_hits / n_sent`.

**LLR.** `1.0 * ramp(density, 0.08, 0.25)`, plus `0.4` if `prose` is
non-empty and `para_hits / len(prose) > 0.3`. Maximum 1.4.

**Confidence.** `min(0.85, 0.3 + 0.03 * n_sent)`.

**Spans.** The first 12 sentence-opener spans (`spans[:12]`).

**Rationale.** Exactly:

```
{sent_hits}/{n_sent} sentences open with a stock transition ({para_hits}/{max(1, len(prose))} paragraphs)
```

**Worked values** (source: running the reference signal):

| Input | sent_hits / n_sent | para_hits / paragraphs | LLR | confidence | spans |
|---|---|---|---|---|---|
| `seed/ai_slop.md` | 9/15 | 3/5 | +1.400 | 0.75 | 9 |
| `seed/human_essay.txt` | 0/23 | 0/6 | 0.000 | 0.85 | 0 |

The nine spans on `seed/ai_slop.md` slice to exactly, in document order:
`Moreover,` `Additionally,` `Furthermore,` `However,` `Ultimately,`
`That said,` `In conclusion,` `Overall,` `Notably,`.

## 9. Roster [NORMATIVE]

`STYLOMETRIC_SIGNALS` is a list in this order, every entry with
`version="1"`:

| Order | id | compute |
|---|---|---|
| 1 | `stylometric.sentence_shape` | section 4 |
| 2 | `stylometric.paragraph_uniformity` | section 5 |
| 3 | `stylometric.punctuation` | section 6 |
| 4 | `stylometric.diversity` | section 7 |
| 5 | `stylometric.transitions` | section 8 |

The engine (`11_SPEC_CORE.md`) places this list first in the signal roster.

## 10. Constants

Source for every row: the reference `stylometric.py`.

| Constant | Value | Used by |
|---|---|---|
| `MIN_WORDS` | 120 | skip gate in `score` |
| default `latency_budget_ms` | 50 | every signal |
| default `cost` | `Cost.FREE` | every signal |
| sentence floor | 8 prose sentences | sentence_shape |
| CV weight, CV ramp | 1.2; `ramp(0.50 - cv, 0.0, 0.25)` | sentence_shape |
| kurtosis weight, ramp | 0.6; `ramp(-kurt, 0.5, 2.0)` | sentence_shape |
| sentence_shape confidence | `min(0.9, 0.3 + 0.02 * n)` | sentence_shape |
| paragraph floor | 4 prose paragraphs | paragraph_uniformity |
| paragraph weight, ramp | 0.9; `ramp(0.45 - cv, 0.0, 0.30)` | paragraph_uniformity |
| paragraph confidence | `min(0.8, 0.3 + 0.05 * n)` | paragraph_uniformity |
| quote-count floor | 6 | punctuation |
| typographic threshold, bonus | `>= 0.95`; +0.35 | punctuation |
| mixed-quote band, penalty | `0.15 < frac < 0.85`; -0.4 | punctuation |
| semicolon floor, ramp, weight | 4.0 per 1k; `ramp(semis, 4.0, 10.0)`; 0.3 | punctuation |
| punctuation confidence | 0.5 | punctuation |
| token floor | 200 alphabetic tokens | diversity |
| window | 100 tokens, non-overlapping | diversity |
| TTR weight, ramp | 0.5; `ramp(0.66 - mattr, 0.0, 0.12)` | diversity |
| hapax weight, ramp | 0.3; `ramp(0.45 - hapax, 0.0, 0.20)` | diversity |
| diversity confidence | 0.5 | diversity |
| density ramp, weight | `ramp(density, 0.08, 0.25)`; 1.0 | transitions |
| paragraph bonus | +0.4 when `para_hits / len(prose) > 0.3` | transitions |
| transitions confidence | `min(0.85, 0.3 + 0.03 * n_sent)` | transitions |
| span limit | 12 | transitions |

## 11. Decisions already taken [GUIDANCE]

- **Kurtosis alongside CV.** CV alone misses a document whose sentences
  vary moderately but never go very short or very long. Negative excess
  kurtosis captures the missing tails, which is the documented
  model behavior. Population moments were chosen so a constant-length
  document yields exactly -3.0 rather than a division error.
- **Windowed TTR, not global TTR.** Global type-token ratio falls with
  document length by construction; a fixed 100-token window compares
  documents of different lengths fairly. Non-overlapping windows keep it
  simple; a trailing partial window is dropped rather than scaled.
- **Transitions count sentence-initial, with a paragraph-initial bonus.**
  Sentence-level density is the measure; the paragraph-initial position is
  where models place these words most, so crossing 30% of paragraphs adds
  a flat 0.4 rather than a second density ramp.
- **Em-dash rate lives in the rule pack, not here.** It is a `rate` rule in
  `claudeisms` so the pack's half-life decay applies to it. Both sit in the
  same correlation group, so no double count.
- **No spans from four of the five features.** Shape, uniformity,
  punctuation, and diversity are document-level statistics; a span would
  point at nothing a writer could edit. Only transitions produce spans.
- **Features that lack material return 0, not skipped.** A skip is reserved
  for "the whole class has no power here" (`MIN_WORDS`). A feature-level
  floor says so in its rationale and contributes nothing.
- **The `technical` profile dampens diversity, sentence_shape, and
  paragraph_uniformity.** These are the features that penalize restrained
  vocabulary and uniform sentences, which is what careful technical
  writing and non-native English look like. The multipliers belong to the
  profile (`PROF` requirements), so a rebuild changes them there.

## 12. Requirements

| ID | Requirement |
|---|---|
| STY-01 | Every stylometric signal must return a skipped `SignalResult` with rationale `document too short for stylometry ({n} words < 120)` when `doc.word_count < 120`. |
| STY-02 | Every stylometric signal must have `cost=Cost.FREE`, `latency_budget_ms=50`, `requires={"text"}`, `half_life_days=None`, `version="1"`, and an id of the form `stylometric.<feature>`. |
| STY-03 | `score` must multiply the feature LLR by `profile.weight_for(self.id)` and leave confidence, spans, and rationale unchanged. |
| STY-04 | Every result must have `calibrated=False`. |
| STY-05 | `sentence_shape` must use only sentences from `doc.prose_sentences` with `word_count > 0`, and return LLR 0.0, confidence 0.2, rationale `only {n} prose sentences — not enough for shape analysis` when fewer than 8 qualify. |
| STY-06 | `sentence_shape` must compute LLR as `1.2 * ramp(0.50 - cv, 0.0, 0.25) + 0.6 * ramp(-kurt, 0.5, 2.0)` with population stdev and the `_excess_kurtosis` function in section 4. |
| STY-07 | `sentence_shape` must produce the rationale format in section 4, with the kurtosis printed with an explicit sign. |
| STY-08 | A document of 20 identical-length sentences must score `sentence_shape` LLR 1.8 (CV 0.00, kurtosis -3.0). |
| STY-09 | `paragraph_uniformity` must use only paragraphs of kind `prose`, return LLR 0.0 and rationale `only {n} prose paragraphs — not enough` below 4 of them, and otherwise compute `0.9 * ramp(0.45 - cv, 0.0, 0.30)`. |
| STY-10 | `punctuation` must count curly quotes (U+2018, U+2019, U+201C, U+201D), straight quotes (`"` and `'`), and semicolons per 1,000 words, and apply the rules in section 6 in that order. |
| STY-11 | `punctuation` must return rationale `punctuation profile unremarkable` and LLR 0.0 when no rule applies, and confidence 0.5 always. |
| STY-12 | `punctuation` must return a negative LLR (-0.4) when at least 6 quotes are present and between 15% and 85% of them are curly. |
| STY-13 | `diversity` must tokenize with `[a-zA-Z']+` lower-cased and return LLR 0.0, confidence 0.2, rationale `under 200 alphabetic tokens — diversity not meaningful` below 200 tokens. |
| STY-14 | `diversity` must compute mean TTR over non-overlapping 100-token windows and hapax as the share of types with count 1, and LLR as `0.5 * ramp(0.66 - mattr, 0.0, 0.12) + 0.3 * ramp(0.45 - hapax, 0.0, 0.20)`. |
| STY-15 | `transitions` must use the `TRANSITIONS` tuple and regex in section 8, case-insensitive, anchored at the start, including an optional trailing comma or colon in the match. |
| STY-16 | `transitions` must emit one `Span(start, start + match_end, note="transition opener")` per matching prose sentence, offset into `Document.text`, limited to the first 12. |
| STY-17 | `transitions` must compute LLR as `ramp(sent_hits / n_sent, 0.08, 0.25)` plus 0.4 when more than 30% of prose paragraphs open with a transition. |
| STY-18 | `transitions` on `seed/ai_slop.md` must report `9/15 sentences open with a stock transition (3/5 paragraphs)`, LLR 1.4, and spans that slice to the nine openers listed in section 8. |
| STY-19 | `STYLOMETRIC_SIGNALS` must contain exactly the five signals of section 9, in that order. |
| STY-20 | The sum of the five default-profile LLRs on `seed/human_essay.txt` must be 0.0 (test bound: below 1.5). |
| STY-21 | The `technical` profile must give `diversity` an LLR less than or equal to the `default` profile's on the same document. |
| STY-22 | No stylometric signal may emit spans except `transitions`. |

## 13. Test mapping

All tests use `segment` from the segmenter and `get_profile` from
profiles. `ai_text` and `human_text` are the seed fixtures loaded by
`conftest.py`.

| Test | Setup | Exact assertions | Proves |
|---|---|---|---|
| `test_short_document_skips` | `segment("Just a few words here.")`, every signal in `STYLOMETRIC_SIGNALS`, `default` profile | `result.skipped` is true; `"too short" in result.rationale` | STY-01, STY-19 |
| `test_uniform_sentences_score_higher_than_bursty` | `uniform` = 20 copies of `"The system processes the incoming data and writes the result number {i} today."` joined by spaces (260 words); `bursty` = the 135-word grief paragraph in the test file; `sentence_shape` only, `default` profile | `u.llr > b.llr`; `u.llr > 0.5`. Reference values: u.llr = 1.8, b.llr = 0.281 (source: running the signal) | STY-05, STY-06, STY-08 |
| `test_transition_density_flags_spans` | `transitions` on `segment(ai_text)`, `default` profile; `flagged = {doc.text[s.start:s.end].strip(",: ").lower() for s in result.spans}` | `result.llr > 0.3` (reference 1.4); `result.spans` non-empty (reference 9); `flagged & {"however", "moreover", "additionally", "furthermore"}` non-empty | STY-15, STY-16, STY-17, STY-18 |
| `test_technical_profile_dampens_diversity` | `diversity` on `segment(ai_text)` under `default` and `technical` | `technical.llr <= default.llr` (reference: both 0.0, since the fixture has TTR 0.82) | STY-03, STY-21 |
| `test_human_text_stays_low` | all five signals on `segment(human_text)`, `default`, LLRs summed | `total < 1.5` (reference total 0.0) | STY-20 |
| `test_end_to_end_orders_fixtures` (in `test_fusion.py`) | full `check()` on both fixtures | `ai.total_llr > human.total_llr + 2.0`; bands. The stylometric contribution to the AI fixture is 3.178 before fusion (sentence_shape 1.138, paragraph_uniformity 0.640, transitions 1.400) | STY-18, STY-20 |

Requirements without a direct test (STY-02, STY-04, STY-07, STY-09 to
STY-14, STY-22) are verified by inspection or by tests the rebuild should
add; a rebuild that adds one test per untested row is encouraged and
should keep the reference values above as its expected outputs.
