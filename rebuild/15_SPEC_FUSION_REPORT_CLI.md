# 15 · Fusion, evidence report, and CLI

**Purpose.** Combine per-signal log-likelihood ratios (LLRs) into one total
with correlation awareness, render that total as an evidence report in the
terminal and as JSON, and expose both through the `tell` command.

**Marker.** Sections 3, 4 and 5 are **[NORMATIVE]**. Sections 1, 2 and 7 are
**[GUIDANCE]**.

**Requirement prefixes.** `FUS` (fusion), `REP` (report), `CLI` (command
line).

**Tests that hold this spec to account.** `tests/test_fusion.py` (7 tests)
and `tests/test_cli.py` (7 tests). Section 9 lists each.

**Depends on.** `11_SPEC_CORE.md` for `Document`, `Profile`, `SignalResult`,
`Span`, the segmenter, the engine's `check()` function, and the profile
table. `14_SPEC_RULEPACKS.md` for `load_packs` and `PackError`.

**Feeds.** `16_SPEC_WEB.md`, which calls `report_dict` for its JSON API.

---

## 1. Principles [GUIDANCE]

1. **No verdicts.** The report carries evidence and stated uncertainty. It
   never prints a headline percentage, a pass or fail word, or a nonzero
   exit code because of what it found.
2. **One lone tell is worth almost nothing.** A document where only one
   signal fired can never reach the `elevated` band. This is policy, pinned
   by a test, not a tuning parameter.
3. **Signals are correlated.** Sentence uniformity, paragraph uniformity,
   and register words partly measure the same thing. Within a correlation
   group the strongest signal counts fully and the rest are discounted
   harmonically.
4. **Every number is traceable.** The report shows raw LLR, discount, and
   fused LLR per signal, and the rationale string the signal wrote.
5. **The "what would change this" block is mandatory.** It is what turns a
   score into something a reader can act on.
6. **Uncalibrated means Assist-only.** Phase 1 LLRs are heuristic. The
   report says the false-positive rate (FPR) is unmeasured, every time.

## 2. Data flow [GUIDANCE]

```
engine.check(raw, profile_name, packs_dir, source)
  -> (Document, Profile, list[SignalResult], FusedResult)
       |                                        |
       +-- report.render_terminal(...)          +-- report.render_json(...)
       +-- report.report_dict(...)  (shared by JSON and the web API)
```

`fuse()` takes the document, the profile, and every `SignalResult` the
engine collected. It never reads the text itself except for the word count
used by one advisory line.

---

## 3. Fusion [NORMATIVE]

Module: `tell/fusion.py`.

### 3.1 Constants

| Name | Value | Meaning | Source |
|---|---|---|---|
| `CAP_SINGLE_SIGNAL` | `3.0` | Absolute clamp on any one signal's fused contribution, both directions | `fusion.py` |
| `SINGLE_FIRE_CEILING` | `2.9` | Maximum total when exactly one signal fired; just under the `elevated` floor | `fusion.py` |
| fired threshold | `0.25` | A contribution counts as "fired" when its fused LLR exceeds this | `fusion.py` |
| discount slope | `0.6` | The `k`-th strongest member of a group is multiplied by `1 / (1 + 0.6 k)` | `fusion.py` |
| negative threshold | `-0.2` | A contribution below this is listed as pointing human-ward | `fusion.py` |
| short-document limit | `300` words | Below this the advisory block adds the low-power line | `fusion.py` |

### 3.2 Dataclasses

```python
@dataclass
class Contribution:
    signal_id: str
    raw_llr: float
    fused_llr: float      # after group discount, confidence, and cap
    group: str
    discount: float
    rationale: str

@dataclass
class FusedResult:
    total_llr: float
    posterior: float      # sigmoid(prior + total); NOT a verdict
    band: str             # "none" | "mild" | "elevated" | "substantial"
    assessment: str       # the band's sentence, see 3.6
    contributions: list[Contribution]
    skipped: list[SignalResult]
    what_would_change: list[str] = field(default_factory=list)
    calibrated: bool = False
```

### 3.3 Group assignment

A signal belongs to the first group in `profile.groups` (insertion order)
that has a glob matching its `signal_id`, using `fnmatch.fnmatch`. A signal
matched by no glob belongs to the group named `"ungrouped"`.

```python
def _group_for(signal_id: str, profile: Profile) -> str:
    for group, patterns in profile.groups.items():
        if any(fnmatch.fnmatch(signal_id, p) for p in patterns):
            return group
    return "ungrouped"
```

An ungrouped signal receives no discount. That is why the never list
requires every new signal to be assigned a group.

### 3.4 Discount and per-signal fused value

Within each group, active members are sorted by descending `abs(llr)`. The
member at position `k` (0-based) receives discount `d_k`:

```python
def _discounts(n: int) -> list[float]:
    return [1.0 / (1.0 + 0.6 * k) for k in range(n)]
```

The first six values, computed from the formula: `1.000, 0.625, 0.455,
0.357, 0.294, 0.250`.

Each member's fused value is:

```python
fused = max(-CAP_SINGLE_SIGNAL, min(CAP_SINGLE_SIGNAL, r.llr * d * r.confidence))
```

Skipped results (`SignalResult.skipped is True`) are excluded from grouping
and from the total and are carried in `FusedResult.skipped` unchanged.

### 3.5 Total, single-fire ceiling, posterior

```python
total = sum(c.fused_llr for c in contributions)
fired = [c for c in contributions if c.fused_llr > 0.25]
if len(fired) == 1:
    total = min(total, SINGLE_FIRE_CEILING)
posterior = 1.0 / (1.0 + math.exp(-(profile.prior_log_odds + total)))
```

The ceiling applies only when exactly one contribution exceeds the fired
threshold. Zero fired or two or more fired leaves the total untouched.

### 3.6 Bands

The band is the first row whose floor the total meets or exceeds, checked
top to bottom. The assessment strings are exact; they appear in the terminal
header, the JSON, and the web UI.

| Floor (total LLR ≥) | `band` | `assessment` |
|---|---|---|
| `6.0` | `substantial` | `Substantial tell density — flagged spans read like unedited model output` |
| `3.0` | `elevated` | `Elevated tell density — worth a close read of the flagged spans` |
| `1.0` | `mild` | `Mild stylistic overlap with machine-generated text` |
| `-inf` | `none` | `No elevated signal — reads as authored` |

The dash in the first, second and fourth strings is U+2014 (em dash).

### 3.7 Ordering and the calibrated flag

After the total is computed, `contributions` is sorted by descending
`fused_llr`. `calibrated` is `True` only when there is at least one active
result and every active result has `calibrated=True`. In the reference build
no signal sets `calibrated=True`, so the flag is always `False`.

### 3.8 The advisory block

`what_would_change` is a list of strings built in this order. Lines 1 and 4
are always present.

1. Always:
   `An author baseline (3+ prior documents by the same writer) would replace population baselines with the strongest signal we have (P5). Not yet ingested.`
2. When `doc.word_count < 300`:
   `Short document ({n} words): every signal is low-power here; treat the assessment as weak either way.`
   where `{n}` is the word count.
3. When `skipped` is non-empty:
   `{k} signal(s) skipped ({ids}) — more text or a different profile would activate them.`
   where `{k}` is the count and `{ids}` is the comma-space-joined `signal_id`
   of at most the first four skipped results. The dash is U+2014.
4. When any contribution has `fused_llr < -0.2`:
   `Some signals point human-ward ({ids}) — mixed evidence, not a clean read.`
   where `{ids}` is the comma-space-joined `signal_id` of every such
   contribution in contributions order. The dash is U+2014.
5. Always:
   `Statistical signals (S2 Binoculars/Fast-DetectGPT) land in Phase 2 and fail independently of these surface features.`

Line 5 is carried unchanged from the reference build even though the S2
signals now exist; see Decisions already taken.

### 3.9 Worked example

Source: computed by running `tell check examples/sample-draft.md --json`
(the file shipped here as `seed/sample-draft.md`) on the reference build
with the S2 models present, default profile, prior `-2.0`.

| signal | group | position in group | discount | raw LLR | fused LLR |
|---|---|---|---|---|---|
| `lexical.gpt-register.v4` | register | 0 | 1.000 | +10.881 | +3.000 (capped) |
| `lexical.claudeisms.v9` | register | 1 | 0.625 | +9.475 | +3.000 (capped) |
| `lexical.seo-slop.v3` | register | 2 | 0.455 | +5.784 | +2.235 |
| `stylometric.sentence_shape` | regularity | 0 | 1.000 | +1.138 | +0.683 |
| `stylometric.transitions` | register | 3 | 0.357 | +1.400 | +0.375 |
| `stylometric.paragraph_uniformity` | regularity | 1 | 0.625 | +0.640 | +0.220 |
| `statistical.binoculars` | statistical | 0 | 1.000 | -0.164 | -0.090 |

Fused values include each signal's own `confidence` multiplier, which is why
`sentence_shape` at discount 1.0 shows 0.683 from a raw 1.138. Total LLR
`+9.422`, band `substantial`, posterior `0.999`. Seven contributions exceed
the fired threshold, so the single-fire ceiling does not apply.

---

## 4. The evidence report [NORMATIVE]

Module: `tell/report.py`.

### 4.1 Fixed strings

`FPR_LINE`, used verbatim in both renderers:

```
FPR not yet measured for this profile — Assist mode only (ship gate P3). Nothing here is sufficient for adverse action.
```

Recommended action per band (`_ACTIONS`):

| band | string |
|---|---|
| `none` | `Nothing to do.` |
| `mild` | `Optional: skim the flagged spans; a light rewrite pass clears most of them.` |
| `elevated` | `Read the flagged spans and rewrite the ones that aren't in your voice.` |
| `substantial` | `Rework the flagged sections in your own words before shipping; the density, not any single phrase, is the finding.` |

Band colors for the terminal (`_BAND_STYLE`, Rich style names):

| band | style |
|---|---|
| `none` | `green` |
| `mild` | `yellow` |
| `elevated` | `dark_orange` |
| `substantial` | `red` |

### 4.2 `report_dict` JSON shape

`report_dict(doc, profile, results, fused) -> dict` is the single source
for the JSON renderer and the web API. Keys and nesting are exact. Floats
are rounded to 3 decimals.

```python
{
  "source": doc.source,
  "profile": profile.id,
  "word_count": doc.word_count,
  "assessment": {
    "band": fused.band,
    "text": fused.assessment,
    "total_llr": round(fused.total_llr, 3),
    "posterior": round(fused.posterior, 3),
    "calibrated": fused.calibrated,
    "fpr_note": FPR_LINE,
  },
  "contributions": [
    {"signal": c.signal_id, "raw_llr": round(c.raw_llr, 3),
     "fused_llr": round(c.fused_llr, 3), "group": c.group,
     "group_discount": round(c.discount, 3), "rationale": c.rationale}
    for c in fused.contributions          # already sorted by fused_llr desc
  ],
  "spans": [
    {"signal": r.signal_id, "start": s.start, "end": s.end,
     "rule_id": s.rule_id, "note": s.note,
     "excerpt": s.excerpt(doc.text, context=20)}
    for r in results if not r.skipped for s in r.spans   # results order, then span order
  ],
  "skipped": [{"signal": r.signal_id, "reason": r.rationale} for r in fused.skipped],
  "what_would_change": fused.what_would_change,
  "recommended_action": _ACTIONS[fused.band],
}
```

Span rules:

- `start` and `end` are code-point indices into the normalized document
  text (see `11_SPEC_CORE.md`). Never bytes, never UTF-16 units.
- `excerpt` is `Span.excerpt(text, context=20)`: 20 characters of context
  on each side, newlines replaced by spaces, `…` (U+2026) prefixed when the
  window does not start at 0 and suffixed when it does not reach the end.
- `rule_id` is `null` for stylometric and statistical spans and the rule's
  id for pack spans.
- Spans are in `results` order, not sorted by position. The terminal
  renderer sorts them; the JSON does not.

### 4.3 JSON example

Source: `tell check examples/sample-draft.md --json` on the reference build,
S2 models present. The `spans` array is trimmed to its first 3 of 76 entries
and `contributions` to its first 3 of 12; everything else is verbatim.

```json
{
  "source": "sample-draft.md",
  "profile": "default",
  "word_count": 290,
  "assessment": {
    "band": "substantial",
    "text": "Substantial tell density — flagged spans read like unedited model output",
    "total_llr": 9.422,
    "posterior": 0.999,
    "calibrated": false,
    "fpr_note": "FPR not yet measured for this profile — Assist mode only (ship gate P3). Nothing here is sufficient for adverse action."
  },
  "contributions": [
    {
      "signal": "lexical.gpt-register.v4",
      "raw_llr": 10.881,
      "fused_llr": 3.0,
      "group": "register",
      "group_discount": 1.0,
      "rationale": "9 rules fired; emoji_in_headings: 4 matches, 13.8/1k words; vibrant_tapestry: 2 matches, 6.9/1k words; whether_youre: 1 matches, 3.4/1k words; crucial_role: 1 matches, 3.4/1k words; grand_closers: 1 matches, 3.4/1k words; pack staleness decay ×0.83 — refresh due"
    },
    {
      "signal": "lexical.claudeisms.v9",
      "raw_llr": 9.475,
      "fused_llr": 3.0,
      "group": "register",
      "group_discount": 0.625,
      "rationale": "9 rules fired; quietly_doing_y: 1 matches, 3.4/1k words; antithesis_family: 2 matches, 6.9/1k words; colon_reveal: 1 matches, 3.4/1k words; sincerity_hedges: 3 hits (genuinely, it's important to note, that said); bold_leadin_bullets: 4/4 bullets use **Bold phrase:** lead-ins; 1 context-required rule(s) surfaced unscored"
    },
    {
      "signal": "statistical.binoculars",
      "raw_llr": -0.164,
      "fused_llr": -0.09,
      "group": "statistical",
      "group_discount": 1.0,
      "rationale": "Binoculars ratio 1.132 (qwen2.5-0.5b-base-q8_0 / qwen2.5-0.5b-instruct-q8_0); below ≈0.9 reads machine, above ≈1.05 reads human — provisional thresholds, uncalibrated for this model pair"
    }
  ],
  "spans": [
    {
      "signal": "stylometric.transitions",
      "start": 489,
      "end": 498,
      "rule_id": null,
      "note": "transition opener",
      "excerpt": "…our daily routines. Moreover, the seamless integr…"
    },
    {
      "signal": "stylometric.transitions",
      "start": 609,
      "end": 622,
      "rule_id": null,
      "note": "transition opener",
      "excerpt": "…llective potential. Additionally, a robust workflow u…"
    },
    {
      "signal": "stylometric.transitions",
      "start": 712,
      "end": 724,
      "rule_id": null,
      "note": "transition opener",
      "excerpt": "…y, and consistency. Furthermore, meticulous planning…"
    }
  ],
  "skipped": [],
  "what_would_change": [
    "An author baseline (3+ prior documents by the same writer) would replace population baselines with the strongest signal we have (P5). Not yet ingested.",
    "Short document (290 words): every signal is low-power here; treat the assessment as weak either way.",
    "Statistical signals (S2 Binoculars/Fast-DetectGPT) land in Phase 2 and fail independently of these surface features."
  ],
  "recommended_action": "Rework the flagged sections in your own words before shipping; the density, not any single phrase, is the finding."
}
```

The three `statistical.*` contributions appear in `contributions` only when
the S2 reference models are present (see `13_SPEC_STATISTICAL.md`). Without
them, the same three signal ids appear under `skipped`, each with a
`reason`, and `what_would_change` gains the line from 3.8 item 3. The
stylometric signals likewise move to `skipped` for documents under 120
words (see `12_SPEC_STYLOMETRIC.md`).

`render_json` returns `json.dumps(report_dict(...), indent=2,
ensure_ascii=False)`.

### 4.4 Terminal layout

`render_terminal(doc, profile, results, fused, console=None, max_spans=14)`
prints to a Rich `Console` in this order. Every element is required; styles
are the reference.

1. **Header panel.** Title `TELL — evidence report`, border in the band
   style. Four lines:
   - `DOCUMENT  {source}   PROFILE  {profile.id}   WORDS  {word_count}`
     (first segment bold, remainder dim), then a blank line.
   - `ASSESSMENT  {assessment}` with the assessment in bold band style.
   - `EVIDENCE    total LLR {total:+.2f}  ·  posterior {posterior:.2f} (prior {prior:+.1f})`
   - `⚠ {FPR_LINE}` in italic dim.
2. **Signal contributions table.** Title `Signal contributions`,
   left-justified. Columns `signal` (cyan, no wrap), `LLR` (right-justified,
   formatted `{fused:+.2f}`; red when > 1.0, yellow when > 0.3, else green),
   `grp` (dim), `rationale` (max width 70). One row per contribution in
   `fused.contributions` order; when `discount < 1` the rationale is suffixed
   with `  (group discount ×{discount:.2f})` in dim. After the contributions,
   one row per skipped signal with `—` in the LLR column and the skip reason
   in dim as the rationale.
3. **Flagged spans table**, only when at least one span exists. Spans are
   gathered from every non-skipped result and sorted by `start`. Title
   `Flagged spans ({shown} of {total})` where `shown = min(total,
   max_spans)`. Columns `where` (right-justified, dim; the `start` offset),
   `excerpt` (max width 58; `Span.excerpt(text, context=24)`), `why` (max
   width 40, dim; `{rule_id or signal_id without the "stylometric." prefix}:
   {note}`). At most `max_spans` rows.
4. **What would change this panel.** Blue border, title
   `What would change this`, one line per entry prefixed `· ` (U+00B7, space).
5. **Recommended action line.** Bold: `RECOMMENDED ACTION  {_ACTIONS[band]}`.

Fixed-width companion of the header, one contributions row, the spans table
title, and the closing elements. Source: the reference build on
`seed/sample-draft.md` at 100 columns with `--max-spans 4`. Table borders
omitted; strings exact.

```
TELL — evidence report
DOCUMENT  sample-draft.md   PROFILE  default   WORDS  290

ASSESSMENT  Substantial tell density — flagged spans read like unedited model output
EVIDENCE    total LLR +9.42  ·  posterior 1.00 (prior -2.0)
⚠ FPR not yet measured for this profile — Assist mode only (ship gate P3). Nothing here is sufficient for adverse action.

Signal contributions
signal                             LLR   grp          rationale
lexical.gpt-register.v4          +3.00   register     9 rules fired; emoji_in_headings: 4 matches, 13.8/1k words; …
lexical.claudeisms.v9            +3.00   register     9 rules fired; quietly_doing_y: 1 matches, … (group discount ×0.62)

Flagged spans (4 of 76)
where  excerpt                                   why
    0  # 🚀 The Ultimate Guide to M…             emoji_in_headings: near-zero human rate in professional longform
    8  # 🚀 The Ultimate Guide to Modern Pro…    ultimate_guide: pattern match

What would change this
· An author baseline (3+ prior documents by the same writer) would replace population baselines with the strongest signal we have (P5). Not yet ingested.
· Short document (290 words): every signal is low-power here; treat the assessment as weak either way.
· Statistical signals (S2 Binoculars/Fast-DetectGPT) land in Phase 2 and fail independently of these surface features.
RECOMMENDED ACTION  Rework the flagged sections in your own words before shipping; the density, not any single phrase, is the finding.
```

---

## 5. The command line [NORMATIVE]

Module: `tell/cli.py`. Entry point `tell = "tell.cli:main"` in
`pyproject.toml`. `main(argv: list[str] | None = None) -> int` returns the
exit code; `if __name__ == "__main__": raise SystemExit(main())`.

Package version string `__version__ = "0.1.0"` lives in `tell/__init__.py`.

### 5.1 Parser

Program name `tell`. Description:
`TELL — composable AI-text detection. Evidence, not verdicts.`

Top-level option `--version` prints `tell 0.1.0` and exits 0. Subcommands
are required (`dest="command"`).

`tell --help` output, verbatim from the reference build:

```
usage: tell [-h] [--version] {check,packs,profiles} ...

TELL — composable AI-text detection. Evidence, not verdicts.

positional arguments:
  {check,packs,profiles}
    check               check a document (file path or - for stdin)
    packs               list loaded rule packs and their freshness
    profiles            list detection profiles

options:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
```

### 5.2 `tell check`

| Argument | Type | Default | Help string | Notes |
|---|---|---|---|---|
| `file` | positional | required | (none) | A path, or `-` for stdin |
| `--profile` | choice | `default` | (none) | Choices are `sorted(PROFILES)`: `blog`, `default`, `technical` |
| `--packs` | string | `None` | `directory of rule packs (default: bundled packs/)` | Passed to `engine.check(packs_dir=...)` |
| `--json` | flag | off | `machine-readable report` | |
| `--max-spans` | int | `14` | `span rows in terminal report` | Terminal only |

Behavior, in order:

1. If `file == "-"`, read all of stdin; `source` is `<stdin>`. Otherwise
   the path must be a regular file, else print `tell: no such file: {path}`
   to stderr and return `2`. Read it as UTF-8 with `errors="replace"`;
   `source` is the file's basename (`Path.name`).
2. If the text is empty or whitespace only, print `tell: empty input` to
   stderr and return `2`.
3. Call `engine.check(raw, profile_name=args.profile, packs_dir=args.packs,
   source=source)`.
4. With `--json`, print `render_json(...)` to stdout. Otherwise call
   `render_terminal(..., max_spans=args.max_spans)`.
5. Return `0`. A completed check always exits 0, whatever the band.

An invalid `--profile` is rejected by argparse: message
`tell check: error: argument --profile: invalid choice: 'nonsense' (choose from blog, default, technical)`
on stderr and exit `2` (argparse raises `SystemExit`).

`tell check --help`, verbatim:

```
usage: tell check [-h] [--profile {blog,default,technical}] [--packs PACKS]
                  [--json] [--max-spans MAX_SPANS]
                  file

positional arguments:
  file

options:
  -h, --help            show this help message and exit
  --profile {blog,default,technical}
  --packs PACKS         directory of rule packs (default: bundled packs/)
  --json                machine-readable report
  --max-spans MAX_SPANS
                        span rows in terminal report
```

### 5.3 `tell packs`

Option `--packs` (help `directory of rule packs`), default the bundled
`packs/` directory (`engine.DEFAULT_PACKS_DIR`, which is `<repo root>/packs`
resolved relative to the package).

- If loading raises `PackError`, print `tell: {error}` to stderr and return
  `2`. Example from the reference build with a regex rule lacking a pattern:
  `tell: /path/badpack.yaml: rule 'a' (regex) requires 'pattern'`.
- If the directory yields no packs (including a nonexistent directory),
  print `no packs found in {directory}` and return `0`.
- Otherwise print one line per pack, in load order (sorted by filename):

  ```
  {name} v{version} — {n} rules, half-life {half_life_days}d, updated {updated}{stale}
  ```

  where `{stale}` is the empty string when `pack.staleness >= 0.9` and
  otherwise `  stale ×{staleness:.2f}` (two leading spaces, rendered red).
  The name is rendered cyan. Return `0`.

Output on the reference build on 2026-10-05, source: `tell packs`:

```
chatbot-voice v1 — 3 rules, half-life 240d, updated 2026-09-02
claudeisms v9 — 19 rules, half-life 120d, updated 2026-10-04
gpt-register v4 — 14 rules, half-life 120d, updated 2026-09-02  stale ×0.83
seo-slop v3 — 6 rules, half-life 180d, updated 2026-09-02  stale ×0.88
```

The staleness numbers are date-dependent (see `14_SPEC_RULEPACKS.md` for
the formula); the strings around them are not.

### 5.4 `tell profiles`

No options. Print one line per profile in `PROFILES` insertion order
(`default`, `blog`, `technical`):

```
{id} — {description}
```

with the id in cyan. Return `0`. Output on the reference build:

```
default — General prose. Skeptical prior; balanced weights.
blog — Longform blog/newsletter. Structural tells weigh more (human blogs rarely emoji-heading).
technical — Specs, RFCs, docs. Dampens the features that false-fire on careful technical writing and non-native English (low diversity, uniform sentences).
```

(Rich wraps long lines at the console width; the wrapping is not part of
the contract.)

### 5.5 Exit codes

| Code | When |
|---|---|
| `0` | A check completed (any band); `packs` or `profiles` listed, including "no packs found" |
| `2` | Missing file; empty input; `PackError` from `packs`; argparse rejection (bad choice, missing argument) |

There is no exit code that means "AI detected". Tests pin this.

---

## 6. Constants summary

| Constant | Value | Owner |
|---|---|---|
| `CAP_SINGLE_SIGNAL` | 3.0 | this spec |
| `SINGLE_FIRE_CEILING` | 2.9 | this spec |
| fired threshold | 0.25 | this spec |
| discount slope | 0.6 | this spec |
| band floors | 6.0 / 3.0 / 1.0 / -inf | this spec |
| short-document limit | 300 words | this spec |
| human-ward threshold | -0.2 | this spec |
| JSON excerpt context | 20 chars | this spec |
| terminal excerpt context | 24 chars | this spec |
| `--max-spans` default | 14 | this spec |
| staleness display threshold | 0.9 | this spec (formula owned by `14_SPEC_RULEPACKS.md`) |
| profile priors | -2.0 / -1.8 / -2.5 | `11_SPEC_CORE.md` |
| version string | `0.1.0` | this spec |

---

## 7. Decisions already taken [GUIDANCE — do not relitigate]

- **`SINGLE_FIRE_CEILING` is 2.9, not 3.0.** The `elevated` floor is 3.0;
  the ceiling sits just under it so a lone signal can land in `mild` but
  never `elevated`. The cap of 3.0 on a single signal is a separate rule;
  together they mean one signal alone maxes out at 2.9 total.
- **Discounts are harmonic with slope 0.6**, giving 1, 0.625, 0.455, ...
  A learned correlation penalty is a Phase 2 item that needs a calibration
  corpus. Hand-set structure with wide stated uncertainty is the fallback
  the product design prescribes.
- **Confidence multiplies into the fused value.** A signal's own
  `confidence` scales its contribution before the cap. The alternative
  (confidence as a report-only annotation) was rejected because it would let
  a low-confidence signal count fully.
- **The posterior exists in JSON only.** It is reported so the arithmetic is
  auditable, never as a headline. The terminal shows it inline next to the
  prior for the same reason. It is not a probability that the text is
  machine-written; the LLRs are uncalibrated.
- **Exit code is always 0 on a completed check.** A detector that exits
  nonzero on "detected" becomes a gate in CI within a week.
- **Rich renders the terminal report.** Any renderer that produces the same
  strings in the same order satisfies the requirements; the box-drawing is
  not part of the contract.
- **`what_would_change` line 5 still says S2 "lands in Phase 2".** The S2
  signals were built after this string was written and the string was not
  updated. A rebuild may keep it verbatim (the reference does) or replace it
  with a line that states the S2 thresholds are provisional; either
  satisfies FUS-14 as long as the block has an always-present closing line.
- **Spans in JSON are unsorted; spans in the terminal are sorted by
  offset.** The web front end sorts for itself.

---

## 8. Requirements

### Fusion

| ID | Requirement |
|---|---|
| FUS-01 | `fuse(doc, profile, results)` must exclude results with `skipped=True` from grouping and from the total, and must return them unchanged in `FusedResult.skipped`. |
| FUS-02 | Each active result must be assigned to the first profile group whose glob list matches its `signal_id` by `fnmatch`, or to `"ungrouped"` when none matches. |
| FUS-03 | Within a group, members must be sorted by descending absolute LLR, and the member at 0-based position `k` must receive discount `1 / (1 + 0.6 k)`. |
| FUS-04 | A member's fused value must equal `llr × discount × confidence` clamped to `[-3.0, 3.0]`. |
| FUS-05 | `total_llr` must be the sum of fused values, and when exactly one contribution has fused value > 0.25 the total must be reduced to at most 2.9. |
| FUS-06 | Two signals in different groups with LLR 1.0 and confidence 1.0 must produce `total_llr == 2.0`. |
| FUS-07 | `posterior` must equal `1 / (1 + exp(-(profile.prior_log_odds + total_llr)))`. |
| FUS-08 | `band` must be `substantial` when total ≥ 6.0, `elevated` when ≥ 3.0, `mild` when ≥ 1.0, else `none`, and `assessment` must be the exact string in 3.6 for that band. |
| FUS-09 | A single result with LLR 50.0 and confidence 1.0 must yield `total_llr ≤ 2.9` and a band other than `substantial`. |
| FUS-10 | `contributions` must be sorted by descending `fused_llr`, and each must carry `signal_id`, `raw_llr`, `fused_llr`, `group`, `discount`, `rationale`. |
| FUS-11 | `calibrated` must be `True` only when at least one active result exists and all active results have `calibrated=True`; with the reference signals it must be `False`. |
| FUS-12 | `what_would_change` must begin with the author-baseline line in 3.8 item 1, verbatim. |
| FUS-13 | `what_would_change` must add the short-document line when `doc.word_count < 300`, the skipped line when any result is skipped (listing at most four ids), and the human-ward line when any contribution has fused value < -0.2, in that order. |
| FUS-14 | `what_would_change` must end with an always-present closing line (the reference string in 3.8 item 5). |

### Report

| ID | Requirement |
|---|---|
| REP-01 | `report_dict` must return exactly the top-level keys `source`, `profile`, `word_count`, `assessment`, `contributions`, `spans`, `skipped`, `what_would_change`, `recommended_action`. |
| REP-02 | `assessment` must contain `band`, `text`, `total_llr`, `posterior`, `calibrated`, `fpr_note`, with `fpr_note` equal to `FPR_LINE` verbatim and floats rounded to 3 decimals. |
| REP-03 | Each `contributions` entry must contain `signal`, `raw_llr`, `fused_llr`, `group`, `group_discount`, `rationale`, in fused order. |
| REP-04 | Each `spans` entry must contain `signal`, `start`, `end`, `rule_id`, `note`, `excerpt`, where `start`/`end` are code-point offsets into the normalized text and `excerpt` uses 20 characters of context with `…` markers. |
| REP-05 | `skipped` must list `{signal, reason}` for every skipped result, and `recommended_action` must be the `_ACTIONS` string for the band. |
| REP-06 | `render_json` must emit `report_dict` as JSON with `indent=2` and `ensure_ascii=False`. |
| REP-07 | The terminal report must print a header panel titled `TELL — evidence report` containing the `DOCUMENT`/`PROFILE`/`WORDS` line, the `ASSESSMENT` line, the `EVIDENCE` line with total LLR, posterior and prior, and `FPR_LINE`. |
| REP-08 | The terminal report must print a table titled `Signal contributions` with columns `signal`, `LLR`, `grp`, `rationale`, one row per contribution followed by one row per skipped signal showing `—`. |
| REP-09 | When `discount < 1`, the terminal rationale must be suffixed with `(group discount ×{discount:.2f})`. |
| REP-10 | When any span exists, the terminal report must print a table titled `Flagged spans ({shown} of {total})` with columns `where`, `excerpt`, `why`, sorted by start offset, limited to `max_spans` rows (default 14). |
| REP-11 | The `why` cell must be `{rule_id}: {note}` for pack spans and `{signal id minus "stylometric." prefix}: {note}` otherwise. |
| REP-12 | The terminal report must print a panel titled `What would change this` with one `· `-prefixed line per entry, then the line `RECOMMENDED ACTION  {action}`. |
| REP-13 | No renderer may print a percentage headline, the words "pass" or "fail" as a result, or any verdict beyond the band assessment string. |

### CLI

| ID | Requirement |
|---|---|
| CLI-01 | The `tell` entry point must expose subcommands `check`, `packs`, `profiles` with the help strings in 5.1, and `--version` must print `tell 0.1.0`. |
| CLI-02 | `tell check <file> --json` must print the `report_dict` JSON to stdout and return 0. |
| CLI-03 | `tell check <file>` without `--json` must render the terminal report and return 0; the output must contain `TELL` and `Signal contributions`. |
| CLI-04 | `tell check <missing path>` must print `tell: no such file: {path}` to stderr and return 2. |
| CLI-05 | `tell check -` must read stdin, and whitespace-only input must print `tell: empty input` to stderr and return 2. |
| CLI-06 | `--profile` must accept exactly `blog`, `default`, `technical` and reject any other value with an argparse error and exit 2. |
| CLI-07 | `--packs <dir>` must be forwarded to the engine as the pack directory for `check`, and used as the directory for `packs`. |
| CLI-08 | `--max-spans N` must limit the terminal spans table to N rows and must default to 14. |
| CLI-09 | `tell packs` must print one line per pack in the format of 5.3, append `  stale ×{s:.2f}` when staleness < 0.9, and return 0. |
| CLI-10 | `tell packs` must print `no packs found in {dir}` and return 0 for an empty or nonexistent directory, and must print `tell: {error}` and return 2 on `PackError`. |
| CLI-11 | `tell profiles` must print `{id} — {description}` for every profile and return 0; the output must contain `default`, `technical`, and `blog`. |
| CLI-12 | A completed check must return 0 regardless of band. |

---

## 9. Test mapping

All tests run under the autouse fixture in `tests/conftest.py` that unsets
`TELL_OBSERVER_MODEL` and `TELL_PERFORMER_MODEL` and points the models
directory at a nonexistent path, so S2 signals are skipped in every test.
Fixtures `ai_text` and `human_text` read `seed/ai_slop.md` and
`seed/human_essay.txt`; `packs_dir` is the bundled `packs/` directory.

`_doc()` in `test_fusion.py` is `segment("Some words. " * 60)` (120 words).

### `tests/test_fusion.py`

| Test | Setup | Assertions | Proves |
|---|---|---|---|
| `test_single_signal_capped` | one result `("lexical.x.v1", llr=50.0, confidence=1.0)`, default profile | `total_llr <= min(3.0, 2.9)`; `band != "substantial"` | FUS-04, FUS-05, FUS-09 |
| `test_single_fire_cannot_reach_elevated` | one result `("stylometric.sentence_shape", llr=10.0, confidence=1.0)` | `total_llr <= 2.9` | FUS-05 |
| `test_correlated_group_discounted` | two results in group `regularity`: `sentence_shape` 1.0 and `paragraph_uniformity` 1.0, confidence 1.0 | `total_llr < 2.0`; `min(discount over contributions) < 1.0` | FUS-02, FUS-03 |
| `test_independent_groups_not_discounted` | `stylometric.sentence_shape` 1.0 and `lexical.claudeisms.v7` 1.0, confidence 1.0 | `total_llr == 2.0` | FUS-02, FUS-06 |
| `test_skipped_signals_excluded` | one active result and one `SignalResult.skip("lexical.claudeisms.v7", "scope mismatch")` | `len(skipped) == 1`; `len(contributions) == 1` | FUS-01 |
| `test_uncalibrated_flag_and_wwct` | one result `sentence_shape` 1.0 | `calibrated is False`; some `what_would_change` line contains `"author baseline"` (case-insensitive) | FUS-11, FUS-12 |
| `test_end_to_end_orders_fixtures` | `check(ai_text, packs_dir=packs_dir)` and `check(human_text, packs_dir=packs_dir)` | `ai.total_llr > human.total_llr + 2.0`; `ai.band in ("elevated", "substantial")`; `human.band in ("none", "mild")` | FUS-05, FUS-08 |

### `tests/test_cli.py`

| Test | Setup | Assertions | Proves |
|---|---|---|---|
| `test_check_json` | write `ai_text` to `tmp_path/draft.md`; `main(["check", path, "--json"])` | returns 0; parsed JSON has `assessment.band in ("elevated", "substantial")`, `assessment.calibrated is False`, `"FPR not yet measured" in assessment.fpr_note`, non-empty `contributions`, `spans`, `what_would_change` | CLI-02, CLI-12, REP-01, REP-02 |
| `test_check_terminal_renders` | write `human_text` to `tmp_path/essay.txt`; `main(["check", path])` | returns 0; stdout contains `"TELL"` and `"Signal contributions"` | CLI-03, REP-07, REP-08 |
| `test_check_missing_file` | `main(["check", "/nope/definitely-not-here.md"])` | returns 2 | CLI-04 |
| `test_check_empty_stdin` | `sys.stdin` patched to `io.StringIO("   ")`; `main(["check", "-"])` | returns 2 | CLI-05 |
| `test_packs_listing` | `main(["packs"])` | returns 0; stdout contains `"claudeisms"` | CLI-09 |
| `test_profiles_listing` | `main(["profiles"])` | returns 0; stdout contains `"default"`, `"technical"`, `"blog"` | CLI-11 |
| `test_unknown_profile_rejected` | `main(["check", path, "--profile", "nonsense"])` | raises `SystemExit` | CLI-06 |

Requirements with no direct test in the reference suite (a rebuild should
add one each): FUS-07, FUS-10, FUS-13, FUS-14, REP-03 through REP-06,
REP-09 through REP-13, CLI-01, CLI-07, CLI-08, CLI-10. The values to assert
are in sections 3 to 5.
