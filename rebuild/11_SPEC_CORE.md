# 11 · Core: types, signal contract, segmenter, engine bus, profiles

**Purpose.** The data types every other module shares, the plugin contract a
signal implements, the normalizer and segmenter that turn raw text into a
`Document`, the bus that runs every signal and hands results to fusion, and
the built-in detection profiles.

**Marker.** [NORMATIVE] unless a section says [GUIDANCE].
**Requirement prefixes.** `CORE` (types and engine), `SEG` (segmenter), `PROF` (profiles).
**Tests that hold it to account.** `tests/test_segmenter.py` (6 tests), `tests/conftest.py` (fixtures). Engine behavior is proven indirectly by `tests/test_fusion.py::test_end_to_end_orders_fixtures` and `tests/test_cli.py` (see `15_SPEC_FUSION_REPORT_CLI.md`).
**Depends on.** Nothing. **Feeds.** Every other spec: 12 (stylometric), 13 (statistical), 14 (rule packs), 15 (fusion, report, CLI), 16 (web).

Package layout in the reference build: `src/tell/types.py`, `src/tell/segmenter.py`, `src/tell/engine.py`, `src/tell/profiles.py`, `src/tell/__init__.py` (holds `__version__ = "0.1.0"` and the one-line package docstring), `src/tell/signals/__init__.py` (re-exports `RulePackSignal`, `load_pack`, `load_packs`, `STYLOMETRIC_SIGNALS`).

## 1. Principles

1. **One contract.** A regex pack, a stylometric feature, and a model-backed scorer all return the same `SignalResult`. Fusion cannot tell them apart.
2. **LLRs, not scores.** A signal returns a log-likelihood ratio in natural-log units: how much more probable the observation is under "machine" than under "human" for this profile. `+1.0` is roughly e:1 toward machine.
3. **Uncalibrated by default.** Every `SignalResult` carries `calibrated=False` until measured against a corpus. The report states this (see spec 15).
4. **Offsets are code points into normalized text.** Every `Span.start` and `Span.end` is a Python string index into `Document.text`, never into the raw input. Nothing downstream re-normalizes.
5. **Normalization removes noise, never signal.** Line endings and two exotic whitespace characters are normalized. Curly quotes, em-dashes, and emoji are left untouched because rule packs score them.
6. **A broken signal cannot break a check.** A signal that raises is recorded as skipped with the error in its rationale; the check completes.
7. **Profiles over global thresholds.** The same text scores differently under `default`, `blog`, and `technical`, by design.

## 2. The contract: `types.py`

All dataclasses use `from __future__ import annotations`. Frozen means `@dataclass(frozen=True)`.

### 2.1 `Cost` (enum)

| Member | Value |
|---|---|
| `FREE` | `"free"` |
| `CPU` | `"cpu"` |
| `GPU` | `"gpu"` |
| `API_CHEAP` | `"api_cheap"` |
| `API_EXPENSIVE` | `"api_expensive"` |

### 2.2 `Span` (frozen)

A region of the normalized document text that drove a signal.

| Field | Type | Default | Meaning |
|---|---|---|---|
| `start` | `int` | required | Code-point index into `Document.text`, inclusive |
| `end` | `int` | required | Code-point index, exclusive |
| `note` | `str` | `""` | Short explanation shown next to the span |
| `rule_id` | `str \| None` | `None` | The pack rule that produced it, when any |

Method `excerpt(text: str, context: int = 0) -> str`:

```python
lo = max(0, self.start - context)
hi = min(len(text), self.end + context)
prefix = "…" if lo > 0 else ""
suffix = "…" if hi < len(text) else ""
return prefix + text[lo:hi].replace("\n", " ") + suffix
```

The ellipsis is the single code point U+2026. Newlines inside the excerpt become spaces.

### 2.3 `SignalResult` (mutable dataclass)

| Field | Type | Default | Meaning |
|---|---|---|---|
| `signal_id` | `str` | required | e.g. `"lexical.claudeisms.v9"` |
| `llr` | `float` | required | Log-likelihood ratio, natural-log units |
| `confidence` | `float` | required | The signal's own confidence in its LLR, 0..1. Fusion multiplies the LLR by it. |
| `spans` | `list[Span]` | `[]` | What in the text drove the LLR |
| `rationale` | `str` | `""` | Human-readable, shown verbatim in the report |
| `version` | `str` | `"0"` | Signal version string |
| `calibrated` | `bool` | `False` | True only once measured against a corpus |
| `skipped` | `bool` | `False` | True when the signal did not score |
| `meta` | `dict[str, Any]` | `{}` | Free-form extras (unused by fusion) |

Class method `skip(signal_id: str, reason: str, version: str = "0") -> SignalResult` returns a result with `llr=0.0`, `confidence=0.0`, `rationale=reason`, `version=version`, `skipped=True`, and empty spans.

### 2.4 `Sentence` (frozen)

| Field | Type | Meaning |
|---|---|---|
| `start` | `int` | Offset into `Document.text` |
| `end` | `int` | Exclusive offset |
| `text` | `str` | Exactly `Document.text[start:end]` (test `test_sentence_offsets_are_exact`) |

Property `word_count` is `len(self.text.split())`.

### 2.5 `Paragraph` (frozen)

| Field | Type | Default | Meaning |
|---|---|---|---|
| `start` | `int` | required | Offset of the block |
| `end` | `int` | required | Exclusive offset |
| `text` | `str` | required | `Document.text[start:end]` |
| `kind` | `str` | required | One of `"prose"`, `"heading"`, `"bullet_list"`, `"code"` |
| `bullets` | `tuple[Span, ...]` | `()` | One span per bullet line, only when `kind == "bullet_list"`; each span's `note` is `"bullet"` |

### 2.6 `Document` (mutable dataclass)

| Field | Type | Default | Meaning |
|---|---|---|---|
| `text` | `str` | required | The normalized text |
| `paragraphs` | `list[Paragraph]` | required | Every blank-line-separated block, in order |
| `sentences` | `list[Sentence]` | required | Sentences from prose blocks and bullet items, in order |
| `source` | `str` | `"<stdin>"` | File name or `"<stdin>"`, shown in the report |

Properties:

- `word_count`: `len(self.text.split())`. This is the word count used everywhere (densities, minimum-length gates, report).
- `prose_sentences`: the sentences whose `start` falls inside a paragraph with `kind == "prose"`. Bullet-item sentences are excluded. Implemented as: collect `(p.start, p.end)` for prose paragraphs; keep `s` where any `lo <= s.start < hi`.

### 2.7 `Profile` (mutable dataclass)

| Field | Type | Default | Meaning |
|---|---|---|---|
| `id` | `str` | required | Profile name |
| `description` | `str` | required | Shown by `tell profiles` and the web profile list |
| `prior_log_odds` | `float` | `-2.0` | Skeptical prior added to the fused total before the sigmoid |
| `weight_overrides` | `dict[str, float]` | `{}` | Signal-id glob → multiplier |
| `groups` | `dict[str, list[str]]` | `{}` | Correlation group name → list of signal-id globs |
| `scopes` | `tuple[str, ...]` | `("prose", "longform")` | Scopes a pack must intersect to run (see spec 14) |

Method `weight_for(signal_id: str) -> float`: start at `1.0`; for every `(pattern, m)` in `weight_overrides`, if `fnmatch.fnmatch(signal_id, pattern)` then multiply by `m`. All matching overrides multiply together.

### 2.8 `Signal` (runtime-checkable Protocol)

```python
class Signal(Protocol):
    id: str
    cost: Cost
    latency_budget_ms: int
    requires: set[str]
    half_life_days: int | None

    def score(self, doc: Document, profile: Profile) -> SignalResult: ...
```

`requires` is informational in the reference build (`{"text"}` everywhere); nothing enforces it. `half_life_days` is `None` for non-pack signals.

### 2.9 `ramp(x, lo, hi) -> float`

```python
if hi <= lo:
    return 1.0 if x >= hi else 0.0
return min(1.0, max(0.0, (x - lo) / (hi - lo)))
```

0 at or below `lo`, 1 at or above `hi`, linear between. Every heuristic signal turns "how far above the human baseline" into evidence with this function.

## 3. Segmenter: `segmenter.py`

Entry point `segment(raw: str, source: str = "<stdin>") -> Document`. Also exported: `normalize(raw) -> str` and `split_sentences(text, start, end) -> list[Sentence]`.

### 3.1 Regexes and constants

```python
_HEADING_RE = re.compile(r"^\s{0,3}#{1,6}\s")
_BULLET_RE = re.compile(r"^\s*(?:[-*+]|\d{1,3}[.)])\s+")
_FENCE_RE = re.compile(r"^\s*(```|~~~)")
_SENT_BOUNDARY = re.compile(r"[.!?]+[\"'')\]]*\s+")
```

The abbreviation set (lower case, no trailing period):

```python
_ABBREV = {"e.g", "i.e", "etc", "vs", "cf", "dr", "mr", "mrs", "ms", "prof",
           "st", "no", "fig", "et al", "approx", "dept", "inc", "jr", "sr"}
```

Note `"et al"` contains a space and can never match a single token from `_is_boundary`; it is in the set as written and is inert. Keep it for fidelity or drop it; no test depends on it.

### 3.2 `normalize(raw)`

Exactly these replacements, in this order, and nothing else:

| Step | From | To |
|---|---|---|
| 1 | `"\r\n"` | `"\n"` |
| 2 | `"\r"` | `"\n"` |
| 3 | `" "` (no-break space) | `" "` (U+0020) |
| 4 | `"​"` (zero-width space) | `""` (removed) |

Curly quotes, em-dashes, emoji, tabs, and all other whitespace are untouched. Step 4 shifts offsets relative to the raw input; that is why every offset is into the normalized text.

### 3.3 Paragraph blocks

`_paragraph_blocks(text) -> list[(start, end)]`:

```python
blocks = []; pos = 0
for chunk in re.split(r"\n\s*\n", text):
    if not chunk.strip():
        pos += len(chunk) + 2
        continue
    start = text.index(chunk, pos)
    blocks.append((start, start + len(chunk)))
    pos = start + len(chunk)
```

Blocks are separated by a newline, optional whitespace, newline. A block keeps its leading and trailing non-blank-line whitespace (a leading space on the first line stays inside the block). Source: `test_crlf_normalized` gives two blocks at `(0, 31)` and `(33, 47)` for `"One line.\r\nStill same paragraph.\r\n\r\nNew paragraph."` after normalization.

### 3.4 Classification

`_classify(block) -> str`, evaluated on `lines = [ln for ln in block.split("\n") if ln.strip()]`:

1. No lines → `"prose"`.
2. First line matches `_FENCE_RE` → `"code"`.
3. Exactly one line and it matches `_HEADING_RE` → `"heading"`. A multi-line block whose first line is a heading is **not** a heading; it is prose (or a bullet list by rule 4).
4. Let `bullet_lines` be the count of lines matching `_BULLET_RE`. If `bullet_lines >= 1` and `bullet_lines >= max(1, len(lines) // 2)` → `"bullet_list"`. Integer division: a 3-line block with 1 bullet line qualifies (`3 // 2 == 1`), a 4-line block needs 2.
5. Otherwise `"prose"`.

### 3.5 Multi-block fenced code

While iterating blocks in order, keep `in_code = False`.

- If `in_code` is true: force `kind = "code"`; if `_FENCE_RE.search(block)` finds a fence anywhere in the block, set `in_code = False` (this block closes the fence).
- Else if `kind == "code"` and `block.count("```") % 2 == 1`: set `in_code = True` (the fence opened here and did not close in this block). Only triple backticks are counted for the open/close parity; a `~~~` fence that spans blocks is not tracked.

### 3.6 Bullet spans

For a `bullet_list` block, `_bullet_spans(text, start, end)` walks `text[start:end].split("\n")`, advancing `pos` by `len(line) + 1` per line, and emits `Span(pos, pos + len(line), note="bullet")` for every non-blank line matching `_BULLET_RE`. Non-bullet lines inside the block (continuation lines) get no span. Source: the `ai_slop.md` fixture has one bullet list with exactly 4 bullet spans (`test_bullets_extracted`).

### 3.7 Sentence splitting

`split_sentences(text, start, end)` works on `region = text[start:end]`:

1. For each match `m` of `_SENT_BOUNDARY` in `region`, skip it unless `_is_boundary(region, m.start())` is true.
2. Otherwise take `raw = region[last:m.end()]`, strip it to `seg`; if non-empty, emit `Sentence(start + last + lead, start + m.end() - trail, seg)` where `lead` and `trail` are the counts of leading and trailing whitespace stripped from `raw`. Set `last = m.end()`.
3. After the loop, the tail `region[last:]`, stripped, becomes a final sentence if non-empty, with `start + last + lead` as its start and `len(tail)` as its length.

Sentence offsets therefore exclude the whitespace that followed the terminal punctuation but include the punctuation and any closing quote or bracket.

`_is_boundary(text, match_start)`:

```python
before = text[: match_start + 1]
last_word = re.findall(r"[\w.]+$", before[:-1])
if last_word:
    w = last_word[0].rstrip(".").lower()
    if w in _ABBREV or (len(w) == 1 and w.isalpha()):
        return False
    if re.search(r"\d$", w) and re.match(r"\d", text[match_start + 1 : match_start + 3].strip() or " "):
        return False
return True
```

Three rejections: the token before the punctuation is a listed abbreviation; it is a single letter (an initial such as `A.`); it ends in a digit and a digit follows the punctuation within two characters (a decimal such as `2.5`).

Worked examples, computed on the reference build:

| Input | Sentences |
|---|---|
| `We tried it, e.g. the blue one, and it worked. Then we left.` | `We tried it, e.g. the blue one, and it worked.` / `Then we left.` |
| `Version 2.5 shipped. Dr. Smith agreed. A. Lincoln spoke.` | `Version 2.5 shipped.` / `Dr. Smith agreed.` / `A. Lincoln spoke.` |

### 3.8 `segment(raw, source)`

1. `text = normalize(raw)`.
2. For each block from `_paragraph_blocks(text)`: classify, apply the fence tracking of 3.5, compute bullets when `kind == "bullet_list"`, append `Paragraph(start, end, block, kind, bullets)`.
3. If `kind == "prose"`: extend sentences with `split_sentences(text, start, end)`.
4. If `kind == "bullet_list"`: for each bullet span `b`, match `_BULLET_RE` against `text[b.start:b.end]` and split sentences from `b.start + m.end()` to `b.end`, so the bullet marker is never part of a sentence.
5. Headings and code blocks produce no sentences.
6. Return `Document(text, paragraphs, sentences, source)`.

Fixture figures (computed): `seed/human_essay.txt` segments to 288 words, 23 sentences, 6 paragraphs all `prose`. `seed/ai_slop.md` segments to 290 words, 19 sentences, 10 paragraphs with kinds `heading, prose, heading, prose, prose, heading, bullet_list, prose, heading, prose`.

## 4. Engine bus: `engine.py`

```python
DEFAULT_PACKS_DIR = Path(__file__).resolve().parents[2] / "packs"
```

That is the `packs/` directory at the repository root when the module lives at `src/tell/engine.py` (two parents above `src/tell/`). The rebuilt project must resolve to the same place.

### 4.1 `build_signals(packs_dir=None) -> list[Signal]`

Roster order is fixed:

1. `STYLOMETRIC_SIGNALS` from `signals/stylometric.py` (five signals, spec 12).
2. `statistical_signals()` from `signals/statistical.py` (three signals, spec 13; they self-skip when no model is configured).
3. One `RulePackSignal` per pack returned by `load_packs(directory)`, where `directory` is `Path(packs_dir)` if given else `DEFAULT_PACKS_DIR`, and only if `directory.is_dir()`. `load_packs` sorts `*.yaml` then `*.yml` by filename (spec 14).

Resulting signal ids on the reference build, in order:

```
stylometric.sentence_shape, stylometric.paragraph_uniformity,
stylometric.punctuation, stylometric.diversity, stylometric.transitions,
statistical.binoculars, statistical.perplexity, statistical.burstiness,
lexical.chatbot-voice.v1, lexical.claudeisms.v9, lexical.gpt-register.v4,
lexical.seo-slop.v3
```

### 4.2 `run_signals(doc, profile, signals) -> list[SignalResult]`

Calls `sig.score(doc, profile)` for each signal in roster order. Any exception is caught and replaced by `SignalResult.skip(sig.id, f"signal error: {e!r}")`. The rationale string is therefore `signal error: ` followed by the Python repr of the exception, for example `signal error: ValueError('bad pattern')`.

### 4.3 `check(raw_text, profile_name="default", packs_dir=None, source="<stdin>")`

Returns the 4-tuple `(doc, profile, results, fused)`:

1. `doc = segment(raw_text, source=source)`
2. `profile = get_profile(profile_name)` (raises `KeyError` for an unknown name; see 5.3)
3. `signals = build_signals(packs_dir)`
4. `results = run_signals(doc, profile, signals)`
5. `fused = fuse(doc, profile, results)` (spec 15)

The signal roster is rebuilt on every call. There is no caching of packs or models at this layer (the statistical backend caches itself, spec 13).

## 5. Profiles: `profiles.py`

### 5.1 Correlation groups (shared by all three profiles)

```python
_BASE_GROUPS = {
    "regularity": [
        "stylometric.sentence_shape",
        "stylometric.paragraph_uniformity",
        "stylometric.punctuation",
        "stylometric.diversity",
        "statistical.burstiness",
    ],
    "register": [
        "stylometric.transitions",
        "lexical.*",
    ],
    "statistical": [
        "statistical.binoculars",
        "statistical.perplexity",
    ],
}
```

Each profile receives `dict(_BASE_GROUPS)` (a shallow copy). A signal id is matched against the globs in dictionary order; first group wins. An id matching no glob lands in a group literally named `"ungrouped"` (spec 15), where it receives no discount. **Every new signal must be added to a group here.**

### 5.2 The profiles

| id | `description` (verbatim) | `prior_log_odds` | `weight_overrides` | `scopes` |
|---|---|---|---|---|
| `default` | `General prose. Skeptical prior; balanced weights.` | `-2.0` | `{}` | `("prose", "longform")` |
| `blog` | `Longform blog/newsletter. Structural tells weigh more (human blogs rarely emoji-heading).` | `-1.8` | `{"lexical.*": 1.15}` | `("prose", "longform", "blog")` |
| `technical` | `Specs, RFCs, docs. Dampens the features that false-fire on careful technical writing and non-native English (low diversity, uniform sentences).` | `-2.5` | `{"stylometric.diversity": 0.4, "stylometric.sentence_shape": 0.5, "stylometric.paragraph_uniformity": 0.6, "statistical.perplexity": 0.3}` | `("prose", "longform", "technical")` |

`PROFILES` is a dict keyed by id in that order. The CLI's `--profile` choices are `sorted(PROFILES)`, i.e. `blog, default, technical`.

### 5.3 `get_profile(name) -> Profile`

Returns `PROFILES[name]`. For an unknown name raises `KeyError` with message exactly:

```
unknown profile '<name>' (available: blog, default, technical)
```

(`', '.join(sorted(PROFILES))` produces the available list; `from None` suppresses the chained exception.)

## 6. Constants

| Constant | Value | Where | Source |
|---|---|---|---|
| `__version__` | `"0.1.0"` | `tell/__init__.py` | code |
| Default `source` | `"<stdin>"` | `Document`, `segment`, `check` | code |
| Paragraph kinds | `prose`, `heading`, `bullet_list`, `code` | `Paragraph.kind` | code |
| Bullet span note | `"bullet"` | `_bullet_spans` | code |
| Default `prior_log_odds` | `-2.0` | `Profile` | code |
| Profile priors | default `-2.0`, blog `-1.8`, technical `-2.5` | `profiles.py` | code |
| Default scopes | `("prose", "longform")` | `Profile` | code |
| `DEFAULT_PACKS_DIR` | `<repo>/packs` | `engine.py` | code |
| Skip rationale on exception | `"signal error: {e!r}"` | `run_signals` | code |
| Ungrouped group name | `"ungrouped"` | fusion (spec 15) | code |

## 7. Decisions already taken [GUIDANCE — do not relitigate]

1. **Normalization is minimal.** Only CRLF, CR, NBSP, and zero-width space are touched. Curly quotes, em-dashes, and emoji are signal for the rule packs and must survive. Unicode NFC normalization was considered and rejected for the same reason.
2. **Offsets are code points into normalized text.** The web front end maps them to UTF-16 (spec 16). No module ever reports raw-input offsets.
3. **A throwing signal becomes a skip.** One broken pack must not take down a check. The error repr is surfaced in the report's skipped list so it is not silent.
4. **Ungrouped signals are allowed but dangerous.** The engine does not refuse them; it names the group `"ungrouped"` and they get no discount. The never list makes grouping a review requirement rather than a runtime check.
5. **No author baselines yet.** `Profile` has no author-corpus field. `requires` on the protocol exists for that future and is informational now.
6. **Headings never yield sentences.** A heading block produces a paragraph and nothing else, so sentence-shape statistics are not polluted by titles.
7. **Bullet items are sentences but not prose sentences.** They feed `Document.sentences` (used by sentence-shape and burstiness) but not `prose_sentences` (used by transitions). See specs 12 and 13 for which list each signal reads.
8. **`technical` halves the regularity features and cuts perplexity to 0.3.** This is the deliberate answer to the non-native-English and spec-writer false-positive problem described in the product design; it is not tuning to be revisited without a cohort corpus.
9. **The roster is rebuilt per check.** Simplicity over speed; a check is fast enough that caching packs was not worth staleness bugs during pack editing.

## 8. Requirements

### CORE (types and engine)

- **CORE-01** `Cost` is an enum with exactly the five members and string values in §2.1.
- **CORE-02** `Span` is frozen with fields `start`, `end`, `note=""`, `rule_id=None`, and `excerpt()` behaves as in §2.2 including the U+2026 prefix and suffix rules and newline-to-space replacement.
- **CORE-03** `SignalResult` has the nine fields and defaults in §2.3, and `SignalResult.skip(id, reason)` yields `llr=0.0`, `confidence=0.0`, `skipped=True`, `rationale=reason`.
- **CORE-04** `Document.word_count` equals `len(text.split())`.
- **CORE-05** `Document.prose_sentences` returns exactly the sentences whose `start` lies inside a `prose` paragraph.
- **CORE-06** `Profile.weight_for(id)` multiplies together every override whose glob matches `id` and returns `1.0` when none match.
- **CORE-07** `ramp(x, lo, hi)` returns 0 at or below `lo`, 1 at or above `hi`, linear between, and when `hi <= lo` returns 1.0 for `x >= hi` else 0.0.
- **CORE-08** `build_signals()` returns signals in the order stylometric, statistical, rule packs, and the rule packs are in filename order.
- **CORE-09** `build_signals(packs_dir)` with a non-directory path returns only the stylometric and statistical signals.
- **CORE-10** `run_signals` converts an exception from `score()` into a skipped result whose rationale is `"signal error: " + repr(exception)` and continues with the next signal.
- **CORE-11** `check()` returns `(Document, Profile, list[SignalResult], FusedResult)` in that order.
- **CORE-12** `DEFAULT_PACKS_DIR` resolves to the `packs/` directory at the repository root.
- **CORE-13** `__version__` is `"0.1.0"`.

### SEG (segmenter)

- **SEG-01** `normalize` performs exactly the four replacements in §3.2 and no others; the output contains no `"\r"`.
- **SEG-02** Blocks are split on `\n\s*\n`; blank chunks are skipped; block offsets index the normalized text.
- **SEG-03** A block whose first non-blank line matches `_FENCE_RE` is `code`.
- **SEG-04** A single-line block matching `_HEADING_RE` is `heading`; a multi-line block is never `heading`.
- **SEG-05** A block is `bullet_list` when at least one line and at least `len(lines) // 2` lines match `_BULLET_RE`.
- **SEG-06** A `bullet_list` paragraph carries one `Span(note="bullet")` per matching line; the `ai_slop.md` seed yields one list with 4 bullets.
- **SEG-07** An unclosed triple-backtick fence marks every following block `code` until a block containing a fence.
- **SEG-08** Sentence boundaries are matches of `_SENT_BOUNDARY` not rejected by `_is_boundary`; abbreviations in `_ABBREV`, single-letter tokens, and digit-period-digit are rejected.
- **SEG-09** For every sentence, `doc.text[s.start:s.end] == s.text`.
- **SEG-10** Sentences are produced only from `prose` blocks and from bullet items with the marker removed; headings and code produce none.
- **SEG-11** `segment("We tried it, e.g. the blue one, and it worked. Then we left.")` yields exactly 2 sentences.
- **SEG-12** `segment("One line.\r\nStill same paragraph.\r\n\r\nNew paragraph.")` yields exactly 2 paragraphs and a text with no `"\r"`.

### PROF (profiles)

- **PROF-01** Exactly three profiles exist with the ids, descriptions, priors, overrides, and scopes in §5.2.
- **PROF-02** Every profile's `groups` equals `_BASE_GROUPS` as listed in §5.1.
- **PROF-03** `get_profile` with an unknown name raises `KeyError` whose message is `unknown profile '<name>' (available: blog, default, technical)`.
- **PROF-04** `weight_for("stylometric.diversity")` is `0.4` under `technical` and `1.0` under `default`; `weight_for("lexical.claudeisms.v9")` is `1.15` under `blog`.

## 9. Test mapping

Fixtures from `tests/conftest.py`: `ai_text` reads `seed/ai_slop.md` (in the reference build `tests/fixtures/ai_slop.md`), `human_text` reads `seed/human_essay.txt`, `packs_dir` is the repository `packs/`. An autouse fixture `no_real_models` removes `TELL_OBSERVER_MODEL` and `TELL_PERFORMER_MODEL` from the environment and points the statistical module's default models directory at a nonexistent path so tests never load weights (spec 13).

| Test | Setup | Assertions | Proves |
|---|---|---|---|
| `test_sentence_offsets_are_exact` | `segment(human_text)` | `doc.sentences` non-empty; for every `s`, `doc.text[s.start:s.end] == s.text` | SEG-09 |
| `test_paragraph_kinds` | `segment(ai_text)` | kinds set contains `"heading"`, `"bullet_list"`, `"prose"` | SEG-03..05 |
| `test_bullets_extracted` | `segment(ai_text)` | at least one `bullet_list` paragraph; `len(lists[0].bullets) == 4` | SEG-06 |
| `test_abbreviations_do_not_split` | `segment("We tried it, e.g. the blue one, and it worked. Then we left.")` | `len(doc.sentences) == 2` | SEG-08, SEG-11 |
| `test_crlf_normalized` | `segment("One line.\r\nStill same paragraph.\r\n\r\nNew paragraph.")` | `"\r" not in doc.text`; `len(doc.paragraphs) == 2` | SEG-01, SEG-02, SEG-12 |
| `test_word_count` | `segment(human_text)` | `doc.word_count > 250` (actual: 288) | CORE-04 |

Requirements without a direct test in the reference suite (CORE-01..03, CORE-05..13, SEG-07, SEG-10, PROF-01..04) are proven by the downstream suites in specs 12 to 16 and by the end-to-end ordering test in spec 15. The rebuilt project should add direct unit tests for CORE-06, CORE-07, CORE-10, and PROF-03; each is a three-line test from the statements above.
