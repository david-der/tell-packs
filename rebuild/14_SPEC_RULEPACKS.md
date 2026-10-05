# 14 · S4 rule packs — loader, scoring, decay, structural detectors

**Purpose.** The S4 signal class: declarative YAML packs of lexical and structural tells, loaded at check time, each pack presented to the engine as one signal. This is where "I know it when I see it" knowledge lives.

**Normative marker.** Sections marked **[NORMATIVE]** carry the exact algorithm, string, or number. Sections marked **[GUIDANCE]** explain why.

**Requirement prefix.** `PACK`.

**Tests that hold it to account.** `tests/test_rulepacks.py` (14 tests, listed in §12), plus the end-to-end ordering test in `15_SPEC_FUSION_REPORT_CLI.md`.

**Depends on.** `11_SPEC_CORE.md` for `Document`, `Paragraph`, `Sentence`, `Span`, `SignalResult`, `Profile`, `Cost`, and `ramp`. **Feeds** `15_SPEC_FUSION_REPORT_CLI.md` (one `SignalResult` per pack) and `16_SPEC_WEB.md` (span notes and rule ids shown in the UI).

**Data that travels verbatim.** The four packs in `../packs/` load unchanged. Nothing in this spec is needed to edit them; everything in this spec is needed to run them.

---

## 1. Principles

1. **A single rule firing is worth almost nothing.** Real humans use em-dashes. The signal is co-occurrence density across independent rules. The pack's total log-likelihood ratio (LLR) is scaled down hard when few distinct rules fired (§6.4), and fusion caps any one signal on top of that.
2. **Tells decay.** Every pack carries `half_life_days` and `updated`. Effective weight halves per half-life past `updated`, floored at 0.25, and the rationale says when a pack is stale (§5).
3. **`context_required` rules never score.** Deciding whether "spine" is literal needs judgment. The engine surfaces the spans with an "unscored" note and excludes the rule from the LLR (§6.6).
4. **A broken pack never takes down a check.** Validation errors raise `PackError` at load; the engine bus (see `11_SPEC_CORE.md`) isolates a signal that throws at score time.
5. **Packs are data.** The rebuilt engine must load the shipped packs byte-for-byte and reproduce the assertions in §12.

---

## 2. YAML schema

### 2.1 Top-level keys

| Key | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `pack` | string | yes | | Pack name. Appears in the signal id `lexical.<pack>.v<version>`. |
| `version` | integer | no | `1` | Bumped on any rule change. Appears in the signal id. |
| `half_life_days` | integer or absent | no | absent (`None`) | Decay half-life. Absent means no decay. |
| `updated` | ISO date | no | absent (`None`) | Date of last rule change. Absent means no decay. |
| `scope` | list of strings | no | `[prose]` | A pack runs only when its scope intersects the profile's scopes (§6.1). |
| `rules` | list of rule mappings | no | `[]` | An empty or missing list is valid; the pack then never fires. |

`updated` may be written unquoted (`updated: 2026-09-02`, which the YAML parser yields as a date) or quoted (`"2026-09-02"`, which the loader converts with `date.fromisoformat`). Both must work.

### 2.2 Rule keys

| Key | Type | Default | Required by | Meaning |
|---|---|---|---|---|
| `id` | string | | all types | Rule identifier, unique within the pack. Written into every span as `rule_id`. |
| `type` | one of `lexicon`, `regex`, `rate`, `structural` | | all types | Evaluator selector. |
| `weight` | float | `1.0` | | Multiplier on the rule's bounded evidence unit (§6.3). |
| `pattern` | string (Python `re` syntax, with `\p{Emoji}` shorthand) | `None` | `regex`, `rate` | Pattern to count. |
| `terms` | list of strings | `()` | `lexicon` | Words or phrases to count. |
| `unit` | `per_1k_words` or `per_list` | `per_1k_words` | | `rate` only: denominator for the frequency. |
| `baseline` | mapping `{human_p50: float, human_p95: float}` | `{}` | | Overrides the type default baseline (§6.2). Partial overrides merge key by key. |
| `detect` | string, one of the six detector names in §8 | `None` | `structural` | Which structural detector runs. |
| `threshold` | float | `0.6` | | Used by `bullet_starts_with_bold_phrase_then_colon_or_dash` only (§8.1). |
| `note` | string | `""` | | Shown next to every span the rule produces. Falls back to a per-type default when empty (§6.5). |
| `context_required` | boolean | `false` | | When true, the rule's spans are surfaced and its LLR is discarded (§6.6). |

The loader coerces `weight` and `threshold` with `float()`, `version` with `int()`, `context_required` with `bool()`, and `terms` with `tuple()`. A `pattern` on a `lexicon` or `structural` rule is permitted and compiled for validation but never used.

---

## 3. Loader [NORMATIVE]

### 3.1 `load_pack(path) -> Pack`

Validation runs in this order and stops at the first failure. Each failure raises `PackError`, a subclass of `ValueError`, with the message shown. `{path}` is the path as given.

| Step | Condition | Message |
|---|---|---|
| 1 | File read as UTF-8 and parsed with YAML safe loading; the parser raises | `{path}: invalid YAML: {parser error}` |
| 2 | Parsed value is not a mapping, or has no `pack` key | `{path}: missing top-level 'pack' name` |
| 3 | For each rule, index `i` from 0: missing `id` or `type` | `{path}: rule #{i} missing 'id' or 'type'` |
| 4 | `type` not in `lexicon`, `regex`, `rate`, `structural` | `{path}: rule '{id}' has unknown type '{type}'` |
| 5 | `type` is `regex` or `rate` and `pattern` is missing or empty | `{path}: rule '{id}' ({type}) requires 'pattern'` |
| 6 | `type` is `lexicon` and `terms` is missing or empty | `{path}: rule '{id}' (lexicon) requires 'terms'` |
| 7 | `type` is `structural` and `detect` is missing or empty | `{path}: rule '{id}' (structural) requires 'detect'` |
| 8 | `pattern` is present and does not compile (§3.3) | `{path}: rule '{id}' pattern does not compile: {re error}` |

Steps 3 through 8 repeat per rule, in file order. An unknown `detect` name is **not** a load error; it evaluates to a zero hit with the detail `unknown structural detector '{detect}'` (§6.5).

The result is a `Pack` with fields `name` (string), `version` (int), `half_life_days` (int or None), `updated` (date or None), `scope` (tuple of strings), `rules` (tuple of `Rule`), `path`.

### 3.2 `load_packs(directory) -> list[Pack]`

Loads every `*.yaml` in the directory in sorted filename order, then every `*.yml` in sorted filename order, and returns the packs in that order. The first invalid file aborts the whole load with its `PackError`.

### 3.3 Pattern compilation

Every pattern is compiled as:

```python
re.compile(pattern.replace(r"\p{Emoji}", _EMOJI_CLASS), re.MULTILINE)
```

`re.MULTILINE` is always on, so `^` and `$` match at line boundaries. Case-insensitivity is not added; packs write `(?i)` inline where they want it.

The stdlib `re` module has no Unicode property classes, so the literal text `\p{Emoji}` is replaced before compiling with this exact character class (written with escapes; it is one string of 46 code points):

```
[☀-➿⬀-⯿️←-⇿\U0001f000-\U0001faff\U0001fb00-\U0001fbff]
```

The ranges are: Miscellaneous Symbols and Dingbats (U+2600 to U+27BF), Miscellaneous Symbols and Arrows (U+2B00 to U+2BFF), the variation selector U+FE0F, Arrows (U+2190 to U+21FF), and the supplementary symbol planes U+1F000 to U+1FAFF and U+1FB00 to U+1FBFF. Digits, `#`, and `*` carry the formal Emoji property and are deliberately excluded.

---

## 4. Rule and Pack data types

```python
@dataclass(frozen=True)
class Rule:
    id: str
    type: str                 # lexicon | regex | rate | structural
    weight: float
    pattern: str | None = None
    terms: tuple[str, ...] = ()
    unit: str = "per_1k_words"
    baseline: dict[str, float] = {}      # empty dict default
    detect: str | None = None
    threshold: float = 0.6
    note: str = ""
    context_required: bool = False

@dataclass
class Pack:
    name: str
    version: int
    half_life_days: int | None
    updated: date | None
    scope: tuple[str, ...]
    rules: tuple[Rule, ...]
    path: Path | None = None
    # property: staleness (§5)

@dataclass
class RuleHit:                # internal: one rule's evaluation
    rule: Rule
    llr: float
    spans: list[Span]
    detail: str
```

Tests construct `Pack` and `Rule` directly (§12, `test_context_required_is_unscored`, `test_staleness_decay`), so these constructors with these field names and defaults are part of the contract.

---

## 5. Decay [NORMATIVE]

`Pack.staleness` is a property:

```python
if not half_life_days or not updated:
    return 1.0
age = (date.today() - updated).days
if age <= 0:
    return 1.0
return max(0.25, 0.5 ** (age / half_life_days))
```

| age in half-lives | staleness |
|---|---|
| 0 | 1.00 |
| 1 | 0.50 |
| 2 | 0.25 |
| 3 or more | 0.25 (floor) |

The floor exists so a stale pack still whispers instead of vanishing silently. Staleness multiplies the pack LLR (§6.4) and, when below 0.9, adds `pack staleness decay ×{staleness:.2f} — refresh due` to the rationale and marks the pack `stale ×{staleness:.2f}` in the `tell packs` listing (see `15_SPEC_FUSION_REPORT_CLI.md`). Source: `test_staleness_decay` asserts 0.25 at 240 days with a 120-day half-life, and 1.0 at age 0.

---

## 6. Scoring [NORMATIVE]

### 6.1 The signal contract

`RulePackSignal(pack)` implements the `Signal` protocol from `11_SPEC_CORE.md`:

| Attribute | Value |
|---|---|
| `id` | `f"lexical.{pack.name}.v{pack.version}"`, for example `lexical.claudeisms.v9` |
| `cost` | `Cost.FREE` |
| `latency_budget_ms` | `100` |
| `requires` | `{"text"}` |
| `half_life_days` | `pack.half_life_days` |
| `version` | `str(pack.version)` |

`score(doc, profile)` first checks scope. If the profile has scopes, the pack has a scope, and the two sets do not intersect, it returns a skipped result with reason:

```
pack scope {list(pack.scope)} outside profile scopes
```

For example `pack scope ['prose', 'longform', 'blog'] outside profile scopes`. All four shipped packs include `prose`, and every built-in profile includes `prose`, so none skip on scope today.

### 6.2 Baselines

Density rules compare a per-1,000-word (or per-list) value against a human baseline. The type defaults, in hits per 1,000 words:

| rule type | `human_p50` | `human_p95` |
|---|---|---|
| `lexicon` | 0.5 | 3.0 |
| `regex` | 0.2 | 2.0 |
| `rate` | 0.5 | 3.0 |

A rule's `baseline` mapping overrides these key by key. A type not in the table (only `structural`, which does not use density scoring) falls back to the `regex` row.

Source: the `_DEFAULT_BASELINES` constant in the reference build. These numbers are hand-set, not measured; see "Decisions already taken".

### 6.3 Density to LLR

With `lo = human_p50`, `hi = human_p95`, `x` the measured value and `count` the number of matches:

```python
r = ramp(x, lo, hi) + 0.5 * ramp(x, hi, 2 * hi)
llr = rule.weight * r if count else 0.0
```

`ramp(x, lo, hi)` is 0 at or below `lo`, 1 at or above `hi`, linear between (defined in `11_SPEC_CORE.md`). So a rule contributes nothing below the human median, its full `weight` at the human 95th percentile, and up to 1.5 times `weight` at twice the 95th percentile. Heavy repetition is stronger evidence than one nudge past the baseline.

### 6.4 Pack total

After evaluating every rule (§6.5):

```python
fired = [h for h in hits if h.llr > 0]          # scored rules with positive LLR
raw = sum(h.llr for h in fired)
co = _COOCCURRENCE.get(len(fired), 1.0)
llr = raw * co * pack.staleness * profile.weight_for(signal.id)
```

The co-occurrence multiplier by number of distinct rules that fired:

| distinct rules fired | multiplier |
|---|---|
| 0 | 0.00 |
| 1 | 0.35 |
| 2 | 0.65 |
| 3 | 0.85 |
| 4 or more | 1.00 |

`profile.weight_for` applies the profile's glob overrides (for example the `blog` profile's `lexical.*: 1.15`); see `11_SPEC_CORE.md`.

The returned `SignalResult` has:

| field | value |
|---|---|
| `llr` | as above |
| `confidence` | `min(0.85, 0.35 + 0.1 * len(fired))` |
| `spans` | every span of every fired rule, in rule order, followed by the deferred spans of §6.6 |
| `rationale` | §6.7 |
| `version` | `str(pack.version)` |
| `calibrated` | `False` (the dataclass default) |
| `meta` | `{"rules_fired": len(fired), "hit_density_per_1k": round(density, 2), "cooccurrence_scale": co, "staleness": round(staleness, 2)}` where `density` is the count of spans across fired rules per 1,000 words |

A rule whose evaluation produced no spans and an LLR of exactly 0.0 is dropped before anything else. A rule with spans but LLR 0.0 (below the median, or a structural detector under its floor) is kept in `hits` but is not in `fired`; its spans are **not** returned.

### 6.5 Rule evaluators

All matching runs over `doc.text`, the normalized document text; span offsets are code-point indices into it.

**`lexicon`.** Build one pattern from the terms:

```python
re.compile(r"(?i)\b(?:" + "|".join(re.escape(t) for t in rule.terms) + r")\b")
```

Case-insensitive, whole-word, alternation in list order. Multi-word terms match as literal phrases with single spaces; apostrophes and hyphens inside a term are escaped and match literally (`it's important to note`, `load-bearing`). A term beginning or ending with a non-word character would defeat `\b`; no shipped term does. Each match is a span with `note = rule.note or "lexicon term"`. The density is `matches / max(1, word_count) * 1000`. Detail string: `{n} hits ({t1, t2, …})` listing up to six distinct matched strings, lowercased and sorted.

**`regex`.** Compile per §3.3, find all matches in `doc.text`, one span per match with `note = rule.note or "pattern match"`. Density per 1,000 words as above. Detail: `{n} matches, {density:.1f}/1k words`.

**`rate`.** Same matching as `regex`, `note = rule.note or "rate pattern"`. The value is per 1,000 words unless `unit` is `per_list`, in which case it is `matches / max(1, number of bullet_list paragraphs)`. Detail: `rate {value:.1f} {unit} ({n} occurrences)`.

**`structural`.** Look up `rule.detect` in the detector table (§8). Unknown name: `RuleHit(rule, 0.0, [], f"unknown structural detector '{rule.detect}'")`, which the §6.4 filter then drops.

### 6.6 `context_required`

After evaluation, a rule with `context_required: true` that produced spans or a nonzero LLR goes to a `deferred` list instead of `hits`. Its LLR never enters `raw`. Its spans are appended to the result after the fired spans, each with the note rewritten as:

```
{original note} — context required, unscored (S5 judge not in Phase 1)
```

and the rationale gains `{n} context-required rule(s) surfaced unscored`. Source: `test_context_required_is_unscored` asserts `llr == 0.0`, non-empty spans, and `"context required"` in every span note. The web UI and terminal report show these spans like any other; the reader decides.

### 6.7 Rationale

Parts joined with `"; "`, in this order:

1. `{n} rules fired`, or `no rules fired` when none did.
2. For each fired rule in descending LLR, at most five: `{rule.id}: {detail}`.
3. If any deferred: `{n} context-required rule(s) surfaced unscored`.
4. If exactly one rule fired: `single-rule fire — heavily discounted (co-occurrence is the signal)`.
5. If staleness is below 0.9: `pack staleness decay ×{staleness:.2f} — refresh due`.

Example computed by running the reference build on `seed/sample-draft.md` with the default profile, `lexical.claudeisms.v9` (the fired-rule list stops at five although nine fired):

```
9 rules fired; quietly_doing_y: 1 matches, 3.4/1k words; antithesis_family: 2 matches, 6.9/1k words; colon_reveal: 1 matches, 3.4/1k words; sincerity_hedges: 3 hits (genuinely, it's important to note, that said); bold_leadin_bullets: 4/4 bullets use **Bold phrase:** lead-ins; 1 context-required rule(s) surfaced unscored
```

---

## 7. Constants

| Constant | Value | Where used |
|---|---|---|
| Co-occurrence multipliers | `{0: 0.0, 1: 0.35, 2: 0.65, 3: 0.85}`, else 1.0 | §6.4 |
| Staleness floor | 0.25 | §5 |
| Staleness rationale threshold | below 0.9 | §6.7, `tell packs` listing |
| Confidence | `min(0.85, 0.35 + 0.1 × rules fired)` | §6.4 |
| Rationale rule cap | 5 fired rules listed | §6.7 |
| Lexicon detail term cap | 6 distinct terms listed | §6.5 |
| Default baselines | §6.2 table | §6.3 |
| Over-p95 bonus | +0.5 × ramp(x, p95, 2 × p95) | §6.3 |
| Default `threshold` | 0.6 | §8.1 |
| Default `weight` | 1.0 | §6.3 |
| `latency_budget_ms` | 100 | §6.1 |
| Regex flags | `re.MULTILINE` | §3.3 |
| Bold lead-in minimum | 2 matching bullets | §8.1 |
| Tricolon ramp | 0.10 to 0.30 per prose sentence | §8.2 |
| Anaphora run length | 3 or more sentences; `min(1.5, 0.75 × runs)` | §8.3 |
| Fragment punch | ≥ 8 prose sentences; fragment ≤ 3 words after ≥ 10 words; ≥ 2 fragments; ramp 0.05 to 0.18 | §8.4 |
| Uniform bullets | lists of ≥ 4 bullets; CV < 0.20; `min(1.5, 0.8 × lists)` | §8.5 |
| Heading density | ≥ 3 headings; ramp 5.0 to 15.0 per 1,000 words | §8.6 |

---

## 8. Structural detectors [NORMATIVE]

Each detector has the signature `(rule, doc) -> RuleHit`. The table of names is exact; a pack's `detect` value must equal one of these strings.

| `detect` value | section |
|---|---|
| `bullet_starts_with_bold_phrase_then_colon_or_dash` | 8.1 |
| `tricolon_density` | 8.2 |
| `anaphora_triple` | 8.3 |
| `fragment_punch` | 8.4 |
| `uniform_bullet_length` | 8.5 |
| `heading_density` | 8.6 |

`doc.prose_sentences` means sentences whose start offset lies inside a paragraph of kind `prose` (defined in `11_SPEC_CORE.md`). `doc.paragraphs[i].bullets` are the per-line bullet spans of a `bullet_list` paragraph.

### 8.1 `bullet_starts_with_bold_phrase_then_colon_or_dash`

Pattern, matched with `.match()` against the text of each bullet span:

```python
r"^\s*(?:[-*+]|\d{1,3}[.)])\s+\*\*[^*\n]{2,80}?(?:[:—–-]\s*\*\*|\*\*\s*[:—–-])"
```

It accepts the separator inside the bold (`**Phrase:** text`) or just after it (`**Phrase**: text`), with colon, em-dash, en-dash, or hyphen as separator.

```
total  = number of bullets across all bullet_list paragraphs
spans  = one span per matching bullet (the whole bullet line), note "bold lead-in bullet"
if total == 0: RuleHit(rule, 0.0, [], "no bullet lists")
frac   = len(spans) / total
llr    = rule.weight * ramp(frac, rule.threshold * 0.7, rule.threshold)  if len(spans) >= 2 else 0.0
detail = f"{len(spans)}/{total} bullets use **Bold phrase:** lead-ins"
```

With the shipped `threshold: 0.6`, the ramp runs from 42% to 60% of bullets.

### 8.2 `tricolon_density`

Pattern, searched with `finditer` inside each prose sentence's text:

```python
r"\b[\w'-]+, [\w'-]+, and [\w'-]+\b"
```

```
spans   = one per match, offset by the sentence start, note "rule-of-three"
n       = max(1, number of prose sentences)
density = len(spans) / n
llr     = rule.weight * ramp(density, 0.10, 0.30)
detail  = f"tricolon in {len(spans)}/{n} sentences"
```

### 8.3 `anaphora_triple`

Opener of a sentence: the first two tokens of `re.findall(r"[\w']+", text.lower())`, as a tuple (one token if the sentence has only one; empty if none).

Walk the prose sentences in order. From sentence `i`, extend `j` while `j + 1` exists, the opener of `i` is non-empty, and the opener of `j + 1` equals the opener of `i`. If the run `i..j` has 3 or more sentences, count one run and add one span from `sents[i].start` to `sents[j].end` with note `f"{run length} sentences open with '{opener words joined by a space}'"`. Continue from `j + 1`.

```
llr    = rule.weight * min(1.5, 0.75 * runs)
detail = f"{runs} anaphoric run(s) of 3+ sentences"
```

Note that any repeated two-word opener counts, not only pronoun openers. In `test_anaphora_detector` the filler sentence repeated three times is itself a run.

### 8.4 `fragment_punch`

```
sents = prose sentences
if len(sents) < 8: RuleHit(rule, 0.0, [], f"only {len(sents)} prose sentences")
spans = for each consecutive pair (prev, s):
          if s.word_count <= 3 and prev.word_count >= 10 and s.text.endswith("."):
              span over s, note "punch fragment"
if len(spans) < 2: RuleHit(rule, 0.0, [], f"{len(spans)} punch fragment(s) — below floor of 2")
density = len(spans) / len(sents)
llr     = rule.weight * ramp(density, 0.05, 0.18)
detail  = f"{len(spans)} punch fragments in {len(sents)} sentences"
```

Known false positive, recorded for the builder: a numbered list marker that the segmenter splits into its own one-word sentence can count as a fragment.

### 8.5 `uniform_bullet_length`

For each `bullet_list` paragraph with at least 4 bullets: lengths are word counts (`len(text.split())`) of each bullet line; `cv = pstdev(lengths) / mean` (population standard deviation; `cv = 0.0` if the mean is 0). A list with `cv < 0.20` is uniform: add a span over the whole paragraph with note `f"{len(bullets)} bullets, length CV {cv:.2f}"`.

```
llr    = rule.weight * min(1.5, 0.8 * uniform_lists)
detail = f"{uniform_lists} list(s) with near-uniform bullet length"
```

### 8.6 `heading_density`

```
headings = paragraphs of kind "heading"
if len(headings) < 3: RuleHit(rule, 0.0, [], f"{len(headings)} headings — below floor of 3")
per_1k = len(headings) / max(1, word_count) * 1000
llr    = rule.weight * ramp(per_1k, 5.0, 15.0)
spans  = one span per heading paragraph, note "heading"   — returned only if llr > 0, else []
detail = f"{len(headings)} headings, {per_1k:.1f}/1k words (human longform ≈ 3/1k)"
```

---

## 9. The shipped packs

Computed from the YAML files in `../packs/` on 2026-10-05.

| file | `pack` | `version` | `updated` | `half_life_days` | `scope` | rules | types used | `context_required` rules |
|---|---|---|---|---|---|---|---|---|
| `claudeisms.yaml` | `claudeisms` | 9 | 2026-10-04 | 120 | prose, longform | 19 | regex 10, structural 5, lexicon 3, rate 1 | `metaphor_vocabulary` |
| `gpt-register.yaml` | `gpt-register` | 4 | 2026-09-02 | 120 | prose, longform | 14 | regex 10, lexicon 2, rate 2 | none |
| `chatbot-voice.yaml` | `chatbot-voice` | 1 | 2026-09-02 | 240 | prose, longform | 3 | regex 3 | none |
| `seo-slop.yaml` | `seo-slop` | 3 | 2026-09-02 | 180 | prose, longform, blog | 6 | regex 4, lexicon 1, structural 1 | none |

Features the shipped packs exercise, which the rebuilt loader must therefore support: unquoted ISO dates; inline `(?i)`, `(?im)`, `(?m)` flags; the `\p{Emoji}` shorthand (`emoji_in_headings`, `emoji_bullet_leads`); `unit: per_list` with an explicit baseline whose `human_p50` is 0.0 (`emoji_bullet_leads`); partial and full `baseline` overrides (`em_dash_rate`, `aggressive_bolding`); lookbehind and lookahead (`em_dash_rate`); doubled single quotes inside single-quoted YAML scalars (`antithesis_family`); multi-word and apostrophe-bearing lexicon terms (`sincerity_hedges`, `reformulation`); `threshold` on a structural rule (`bold_leadin_bullets`); `context_required: true` (`metaphor_vocabulary`); all six structural detectors (five in `claudeisms`, `heading_density` in `seo-slop`).

The packs load unchanged. Do not reformat, re-quote, or reorder them; the rule counts above are asserted by `test_load_bundled_packs`.

---

## 10. Decisions already taken [GUIDANCE — do not relitigate]

- **The co-occurrence multipliers are policy, not tuning.** 0.35 / 0.65 / 0.85 encode "one tell is noise" and are pinned by `test_single_rule_fire_is_discounted`. Calibration corpora may replace them later; a rebuild does not adjust them.
- **Default baselines are hand-set.** No corpus measurement backs the p50/p95 defaults in §6.2. They stay until a calibration corpus exists, and every result carries `calibrated=False`.
- **`backtick_emphasis` has weight 0.4** because it false-fires on real inline code in technical writing. Do not raise it; the `technical` profile exists for the same reason.
- **`em_dash_rate` excludes `---`** (the pattern `(?<!-)--(?!-)`) so Markdown table rules and horizontal rules are not counted as dashes.
- **Shared structural tells live in `claudeisms.yaml`.** Bold lead-in bullets, tricolons, anaphora, punch fragments, and uniform bullets appear in both Claude and GPT output; they sit in the Claude pack by naming accident and move only when they earn a pack of their own.
- **`surface_as_verb` scores without a judge; `metaphor_vocabulary` does not.** The verb form is almost always metaphorical, so it is a plain `regex` rule; the noun list needs context and is `context_required`. Pinned by `test_surface_as_verb_scored_without_judge` and `test_context_required_is_unscored`.
- **Staleness floors at 0.25 rather than 0.** A decayed pack still contributes and is flagged, so stale packs are noticed rather than silently gone.
- **Packs reload on every check.** The engine calls `load_packs` each time it builds the signal roster, which makes edits to a pack file take effect on the next check with no restart. There is no per-tenant override layer.
- **Unknown structural detector names are a runtime no-op, not a load error.** A pack written for a newer engine degrades to silence for that rule instead of refusing to load.

---

## 11. Requirements

| ID | Requirement |
|---|---|
| PACK-01 | `load_pack` must raise `PackError` with the exact messages and in the order of §3.1. |
| PACK-02 | `load_packs` must return packs for `*.yaml` in sorted filename order followed by `*.yml` in sorted filename order. |
| PACK-03 | Every pattern must compile with `re.MULTILINE` after replacing the literal `\p{Emoji}` with the character class in §3.3. |
| PACK-04 | `Pack.staleness` must follow §5, including the 0.25 floor and the 1.0 result when `half_life_days` or `updated` is absent or the age is zero or negative. |
| PACK-05 | `RulePackSignal.id` must be `lexical.<pack>.v<version>`, with `cost` FREE, `latency_budget_ms` 100, `requires` `{"text"}`, and `half_life_days` from the pack. |
| PACK-06 | `score` must return a skipped result with reason `pack scope [...] outside profile scopes` when the pack's scope and the profile's scopes do not intersect. |
| PACK-07 | Density rules must score by §6.3 against the §6.2 baselines, merged key by key with the rule's `baseline`. |
| PACK-08 | A `lexicon` rule must match its terms case-insensitively at word boundaries, including multi-word phrases, over the normalized text. |
| PACK-09 | A `rate` rule with `unit: per_list` must divide matches by the number of `bullet_list` paragraphs (minimum 1). |
| PACK-10 | The pack LLR must be `sum of fired rule LLRs × co-occurrence multiplier × staleness × profile weight`, with the multipliers of §6.4. |
| PACK-11 | `meta` must contain `rules_fired`, `hit_density_per_1k`, `cooccurrence_scale`, and `staleness` as defined in §6.4. |
| PACK-12 | `confidence` must be `min(0.85, 0.35 + 0.1 × rules fired)`. |
| PACK-13 | A `context_required` rule must contribute 0 to the LLR, must not count as fired, and must surface its spans with the note suffix ` — context required, unscored (S5 judge not in Phase 1)`. |
| PACK-14 | The rationale must be assembled as in §6.7, including the `single-rule fire — heavily discounted (co-occurrence is the signal)` part when exactly one rule fired and the `{n} context-required rule(s) surfaced unscored` part when any were deferred. |
| PACK-15 | Every span must carry `rule_id` equal to the rule's `id` and a `note` that is the rule's `note` or the per-type default of §6.5. |
| PACK-16 | Spans of rules that evaluated with LLR 0 must not be returned. |
| PACK-17 | Each of the six structural detectors must implement §8.1 through §8.6 exactly, including floors, ramps, detail strings, and span notes. |
| PACK-18 | An unknown `detect` name must evaluate to a zero hit with detail `unknown structural detector '<name>'` and must not raise. |
| PACK-19 | The four packs in `../packs/` must load unchanged with the versions and rule counts of §9. |
| PACK-20 | `Pack` and `Rule` must be constructible directly with the field names and defaults of §4. |

---

## 12. Test mapping

All tests live in `tests/test_rulepacks.py`. Fixtures (`ai_text`, `human_text`, `packs_dir`) come from `tests/conftest.py`: `ai_text` is `seed/ai_slop.md`, `human_text` is `seed/human_essay.txt`, `packs_dir` is the packs directory. `_write_pack(tmp_path, body)` writes a dedented YAML string to `tmp_path/test.yaml` and returns the path.

| Test | Setup | Assertions | Proves |
|---|---|---|---|
| `test_load_bundled_packs` | `load_packs(packs_dir)` | names ⊇ `{claudeisms, gpt-register, seo-slop, chatbot-voice}`; `claudeisms` has `version == 9` and `len(rules) == 19` | PACK-02, PACK-19 |
| `test_validation_unknown_type` | pack `bad`, one rule `id: x`, `type: telepathy` | `PackError` matching `unknown type` | PACK-01 |
| `test_validation_missing_pattern` | pack `bad`, rule `id: x`, `type: regex`, no pattern | `PackError` matching `requires 'pattern'` | PACK-01 |
| `test_validation_bad_regex` | rule `type: regex`, `pattern: '(unclosed'` | `PackError` matching `does not compile` | PACK-01, PACK-03 |
| `test_emoji_shorthand_matches` | pack `emoji`, rule `heading_emoji`, `type: regex`, `pattern: '^#{1,6}[^\n]*\p{Emoji}'`, weight 1.6; document `"## 🚀 Launch plan\n\nSome body text here."`; default profile | some span has `rule_id == "heading_emoji"` | PACK-03, PACK-15 |
| `test_single_rule_fire_is_discounted` | pack `one` with lexicon rules `pickaxe` (terms `[pickaxe]`) and `never_fires` (terms `[zyzzyva]`), both weight 2.0; scored on `human_text` (which contains "pickaxe") | `meta["rules_fired"] == 1`; `meta["cooccurrence_scale"] == 0.35`; `"single-rule fire"` in rationale | PACK-08, PACK-10, PACK-11, PACK-14 |
| `test_context_required_is_unscored` | `Pack(name="ctx", version=1, half_life_days=None, updated=today, scope=("prose",), rules=(Rule(id="metaphor", type="lexicon", weight=5.0, terms=("fabric","landscape"), context_required=True),))` scored on `ai_text` | `llr == 0.0`; spans non-empty; every span note contains `"context required"` | PACK-13, PACK-20 |
| `test_staleness_decay` | `Pack` with `half_life_days=120`, `updated = today − 240 days`, no rules; and one with `updated = today` | first `staleness ≈ 0.25` (abs 0.01); second `== 1.0` | PACK-04, PACK-20 |
| `test_claudeisms_fire_on_slop` | `claudeisms` from `packs_dir` on `ai_text`, default profile | `meta["rules_fired"] >= 5`; `llr > 2.0`; span rule ids ⊇ `{antithesis_family, sincerity_hedges, bold_leadin_bullets}` | PACK-07, PACK-10, PACK-17, PACK-19 |
| `test_gpt_register_fires_on_slop` | `gpt-register` on `ai_text` | `meta["rules_fired"] >= 5`; rule ids ⊇ `{register_words, emoji_in_headings, aggressive_bolding}` | PACK-03, PACK-07, PACK-19 |
| `test_chatbot_voice_on_transcript` | `chatbot-voice` on the three-paragraph transcript in the test ("Great question! You're absolutely right … I appreciate you sharing … I hope this helps! Let me know if you'd like … feel free to …") | `meta["rules_fired"] == 3`; `llr > 2.0` | PACK-07, PACK-10 |
| `test_anaphora_detector` | `claudeisms` on `"The deploy finished around noon and nobody noticed anything odd. " × 3` followed by `"It didn't refuse. It didn't hedge. It didn't throw. The whole suite just passed on the first run, which surprised everyone."` | some span has `rule_id == "anaphora_triple"` | PACK-17 (§8.3) |
| `test_surface_as_verb_scored_without_judge` | `claudeisms` on two sentences using "surface the errors" and "surface the slow queries" | exactly 2 spans with `rule_id == "surface_as_verb"`; none of their notes contain `"context required"` | PACK-13, PACK-15 |
| `test_claudeisms_quiet_on_human` | `claudeisms` on `human_text` | `llr < 1.0` | PACK-10 |

The end-to-end test `test_end_to_end_orders_fixtures` in `tests/test_fusion.py` (specified in `15_SPEC_FUSION_REPORT_CLI.md`) additionally requires the full roster, packs included, to score `ai_text` at least 2.0 LLR above `human_text`.

Requirements PACK-05, PACK-06, PACK-09, PACK-12, PACK-16, and PACK-18 have no dedicated test in the reference build. A rebuild should add one each; the acceptance line is the requirement sentence.
