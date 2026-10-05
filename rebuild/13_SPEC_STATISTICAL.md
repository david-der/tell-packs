# 13 · S2 statistical signals: perplexity, Binoculars, burstiness

**Purpose.** Three signals that score a document with a local language model
and report how unsurprising the text is to that model, in absolute terms
(perplexity), relative to what a paired model would have predicted
(Binoculars), and sentence by sentence (burstiness). They are the only
signals in the engine that need model weights, and they are optional: with
no model configured they skip and say why, and every other signal is
unaffected.

**Normative marker.** Sections marked **[NORMATIVE]** carry the exact
formulas, strings, and constants. Other sections are **[GUIDANCE]**.

**Requirement prefix.** `STAT`.

**Tests that hold it to account.** `tests/test_statistical.py` (9 tests,
all against a stub backend) and the `no_real_models` autouse fixture in
`tests/conftest.py`. The test mapping is in §12.

**Depends on.** `11_SPEC_CORE.md` for `Document`, `Profile`,
`SignalResult`, `Cost`, and `ramp`. **Feeds** `15_SPEC_FUSION_REPORT_CLI.md`
(correlation groups `statistical` and `regularity`) and `16_SPEC_WEB.md`
(the How it works modal reads `statistical_status()`).

## 1. Principles

1. **Zero configuration.** If the two default model files exist in the
   `models/` directory at the project root, the signals run. Two environment
   variables override the paths. If neither is present, each signal returns
   a skipped result whose rationale names the environment variable to set.
2. **Tests never load weights.** The math is separated from the model
   backend behind a one-method protocol so a stub can drive every feature to
   its extremes. The autouse fixture in `conftest.py` makes the default
   backend unreachable in every test.
3. **Binoculars is the workhorse.** It has the highest weight (up to +1.6)
   and the highest confidence (0.55) of the three. It is the only one of the
   three that can produce a negative (human-ward) LLR.
4. **Raw perplexity is deliberately weak.** Low perplexity also describes
   careful, non-native, and plain technical prose. Its maximum LLR is +0.5,
   its confidence is 0.4, its rationale carries a caution sentence, and the
   `technical` profile multiplies it by 0.3 (see `11_SPEC_CORE.md`).
5. **Thresholds are provisional.** None of the thresholds below were
   calibrated against a corpus for this model pair. Every `SignalResult`
   from these signals has `calibrated=False` (the dataclass default), and
   the Binoculars rationale says "uncalibrated for this model pair".
6. **One analysis per document.** The backend computes per-token scores
   once per text and the three signals share it through the backend's
   cache. The three signals never load a model more than once per process.

## 2. What the product design asked for and what was built [GUIDANCE]

The product design (S2 in the original PRD) listed four statistical
methods. Three are built. The fourth is not.

| Method | Status |
|---|---|
| Perplexity under a reference model | Built: `statistical.perplexity` |
| Binoculars cross-perplexity ratio | Built: `statistical.binoculars` |
| Burstiness (variance of per-sentence perplexity) | Built: `statistical.burstiness` |
| Fast-DetectGPT conditional probability curvature | **Not built.** No code, no signal id, no test. Do not add it as part of the rebuild; it is a later story. |

The design said "Requires open-weight reference models we host. Budget for
GPU." The build runs a 0.5-billion-parameter pair quantized to 8 bits on
CPU; GPU offload is requested from llama.cpp but is not required.

## 3. Data contract [NORMATIVE]

### 3.1 `StatAnalysis`

Per-token scores for one text, aligned to the normalized document text by
character offset.

| Field | Type | Meaning |
|---|---|---|
| `offsets` | `list[int]` | Character offset into the text of each scored token. Code point index, same convention as every span in the engine. |
| `obs_logprobs` | `list[float]` | Observer log P(token given prefix), natural log, one per scored token. Same length as `offsets`. |
| `xents` | `list[float] \| None` | Per-position cross-entropy between the performer's next-token distribution and the observer's. `None` when no performer model is loaded. Length is `min(observer positions, performer positions)`. |
| `observer_name` | `str` | Display name of the observer model, default `"observer"`. The llama.cpp backend uses the file stem. |
| `performer_name` | `str \| None` | Display name of the performer model, default `None`. |

### 3.2 `StatisticalBackend`

A structural protocol with one method:

```python
class StatisticalBackend(Protocol):
    def analyze(self, text: str) -> StatAnalysis: ...
```

Anything with that method is a backend. The tests supply a stub (§12). The
shipped implementation is `LlamaBackend` (§5).

## 4. Model discovery [NORMATIVE]

### 4.1 Names

| Constant | Value |
|---|---|
| `OBSERVER_ENV` | `"TELL_OBSERVER_MODEL"` |
| `PERFORMER_ENV` | `"TELL_PERFORMER_MODEL"` |
| `_DEFAULT_MODELS_DIR` | `<project root>/models`, computed as the module file's path, three parents up, joined with `"models"`. With the `src/tell/signals/statistical.py` layout, parents[3] is the project root. |
| `_DEFAULT_OBSERVER` | `"qwen2.5-0.5b-base-q8_0.gguf"` |
| `_DEFAULT_PERFORMER` | `"qwen2.5-0.5b-instruct-q8_0.gguf"` |

### 4.2 Resolution order

For each of observer and performer, `_resolve(env, default_name)` returns:

1. The environment variable's value, if the variable is set and non-empty.
   The value is returned as given, without checking that the file exists.
2. Otherwise `str(_DEFAULT_MODELS_DIR / default_name)` if that file exists.
3. Otherwise the empty string `""`.

### 4.3 `statistical_status()`

A cheap probe that never loads a model. It must return exactly these keys:

```python
{
    "llama_cpp_installed": bool,   # `import llama_cpp` succeeded
    "observer": str | None,        # resolved path, or None when "" resolved
    "observer_exists": bool,       # observer is non-empty and Path(observer).is_file()
    "performer": str | None,
    "performer_exists": bool,
}
```

The web UI's How it works modal calls this to tell the reader whether the
S2 instruments are on (`16_SPEC_WEB.md`).

### 4.4 `get_default_backend()`

A lazy module-level singleton over two module globals,
`_default_backend: StatisticalBackend | None` and
`_backend_error: str | None`, both initially `None`.

1. If `_default_backend` is not `None` or `_backend_error` is not `None`,
   return `_default_backend` without re-probing. A failure is therefore
   remembered for the life of the process.
2. Call `statistical_status()`.
3. If `observer` is falsy, set `_backend_error` to
   ``"no reference model — set TELL_OBSERVER_MODEL to a GGUF path (see justfile `models`)"``
   and return `None`. (The em-dash and the backticks around `models` are part
   of the string.)
4. If `observer_exists` is false, set `_backend_error` to
   `f"{OBSERVER_ENV}={status['observer']} does not exist"` and return `None`.
5. If `llama_cpp_installed` is false, set `_backend_error` to
   ``"llama-cpp-python not installed — `uv sync --group statistical`"``
   and return `None`.
6. Let `perf` be `status["performer"]` if `performer_exists` else `None`. A
   configured-but-missing performer is silently dropped; Binoculars then
   skips with its own reason (§7).
7. Construct `LlamaBackend(status["observer"], perf)`. On any exception, set
   `_backend_error = f"model load failed: {e}"`. Return `_default_backend`
   (still `None` on failure).

The check order is observer-unset, observer-missing, library-missing. The
test `test_signals_skip_without_backend` asserts that the first message
contains `TELL_OBSERVER_MODEL`.

### 4.5 `backend_skip_reason()`

Returns `_backend_error` if set, else `"statistical backend unavailable"`.

## 5. The llama.cpp backend [NORMATIVE]

`LlamaBackend(observer_path: str, performer_path: str | None)`.

### 5.1 Construction

Both models are constructed with `llama_cpp.Llama(model_path=..., **kw)`
where

```python
kw = dict(n_ctx=4096, logits_all=True, verbose=False, n_gpu_layers=-1)
```

`logits_all=True` is required: the per-token method reads the logits of
every position, not just the last. `n_gpu_layers=-1` asks llama.cpp to
offload every layer when a GPU backend is available and is a no-op on CPU
builds. `observer_name` and `performer_name` are the file stems
(`Path(path).stem`), so the default pair reports as
`qwen2.5-0.5b-base-q8_0` and `qwen2.5-0.5b-instruct-q8_0`. The performer is
`None` when `performer_path` is `None`. The constructor also creates an
empty dict `_cache: dict[int, StatAnalysis]`.

The observer and performer must share a tokenizer. The base and instruct
checkpoints of one model family do. Two unrelated models do not, and the
cross-entropy in §5.3 would then compare distributions over different
vocabularies, which is meaningless. The rebuild must document this
constraint next to the download recipe.

### 5.2 Per-token log-probabilities

`_token_logprobs(llm, text)` returns `(offsets, logprobs, dists)`:

1. `tokens = llm.tokenize(text.encode("utf-8"), add_bos=True, special=False)`,
   then truncated to the first `llm.n_ctx()` tokens (4096). Text beyond
   that is not scored.
2. `llm.reset()`, then `llm.eval(tokens)`, then
   `logits = np.array(llm.eval_logits)` with shape `(len(tokens), vocab)`.
3. For `i` from 1 to `len(tokens) - 1` (the BOS token at index 0 is never
   scored):
   - `piece = llm.detokenize([tokens[i]]).decode("utf-8", errors="replace")`
   - `ls = _log_softmax(logits[i - 1])`, because the logits at position
     `i - 1` predict token `i`
   - append `float(ls[tokens[i]])` to `logprobs`, `ls` to `dists`, and the
     running character position `pos` to `offsets`, then `pos += len(piece)`.

   The first scored token gets offset 0. Offsets are therefore the
   cumulative decoded length of the preceding scored tokens, in code points.

`_log_softmax(row)` is `row - row.max()` then minus `log(sum(exp(...)))`,
over the full vocabulary row, in float64 via numpy.

### 5.3 Cross-entropy for Binoculars

In `analyze(text)`, after the observer pass, if a performer is loaded:

```python
_, _, perf_dists = self._token_logprobs(self._performer, text)
n = min(len(obs_dists), len(perf_dists))
xents = [float(-(np.exp(perf_dists[i]) * obs_dists[i]).sum()) for i in range(n)]
```

That is, for each position, the cross-entropy H(performer, observer) =
−Σ_v P_performer(v) · log P_observer(v), with the performer's probabilities
as the weights and the observer's log-probabilities as the values. The
direction matters and is the one in the Binoculars paper: the ratio in §7.2
divides observer log-perplexity by this quantity.

### 5.4 Cache

`analyze` keys its cache on `hash(text)`. On a hit it returns the stored
`StatAnalysis`. Before storing a new entry, if the cache already holds more
than 4 entries it is cleared entirely. The cache is what lets the three
signals share one model pass: the engine calls `score` on each of the three
signals in turn with the same `doc.text`.

### 5.5 Minimum size

`MIN_TOKENS = 80`. A signal skips when `len(an.obs_logprobs) < 80` with
rationale `f"only {n} scored tokens < 80"` (see §8).

## 6. Sentence alignment for burstiness [NORMATIVE]

`_sentence_log_ppls(doc, an) -> list[float]`:

1. One empty bucket per `doc.sentences` entry (all sentences, prose and
   bullet items alike; the segmenter decides what is a sentence).
2. Walk `zip(an.offsets, an.obs_logprobs, strict=True)` with a cursor `si`
   over the sentence ranges `(start, end)`. Advance `si` while
   `off >= ranges[si][1]`. If `ranges[si][0] <= off < ranges[si][1]`, append
   the logprob to bucket `si`. Tokens that fall between sentences (headings,
   code, whitespace) are dropped.
3. Return `[-mean(bucket) for bucket in buckets if len(bucket) >= 4]`: the
   per-sentence log-perplexity, only for sentences with at least 4 scored
   tokens.

## 7. The three features [NORMATIVE]

Each feature returns `(llr, confidence, rationale)`. `ramp(x, lo, hi)` is
the engine's clamp: 0 at or below `lo`, 1 at or above `hi`, linear between.
`fmean` is `statistics.fmean`; `pstdev` is `statistics.pstdev` (population
standard deviation).

### 7.1 Perplexity: `perplexity_feature(doc, an)`

```python
log_ppl = -fmean(an.obs_logprobs)
ppl = exp(log_ppl)
llr = 0.5 * ramp(2.1 - log_ppl, 0.0, 0.6)
confidence = 0.4
rationale = (
    f"perplexity {ppl:.1f} under {an.observer_name} (provisional human floor ≈ 8). "
    "Caution: low perplexity also describes careful/ESL/technical prose"
)
```

The LLR is 0 at log-perplexity 2.1 or above (perplexity ≈ 8.17, the
"human floor ≈ 8" in the rationale), rises linearly, and reaches its
maximum +0.5 at log-perplexity 1.5 or below (perplexity ≈ 4.48). It is
never negative: high perplexity is not evidence of a human.

### 7.2 Binoculars: `binoculars_feature(an)`

```python
if an.xents is None:
    raise ValueError("binoculars needs a performer model")
log_ppl = -fmean(an.obs_logprobs)
mean_xent = fmean(an.xents)
if mean_xent <= 0:
    return 0.0, 0.0, "degenerate cross-entropy"
b = log_ppl / mean_xent
llr = 1.6 * ramp(0.95 - b, 0.0, 0.20) - 0.5 * ramp(b - 1.05, 0.0, 0.25)
confidence = 0.55
rationale = (
    f"Binoculars ratio {b:.3f} ({an.observer_name} / {an.performer_name}); "
    "below ≈0.9 reads machine, above ≈1.05 reads human — provisional thresholds, "
    "uncalibrated for this model pair"
)
```

The `ValueError` is never reached through `StatisticalSignal.score`, which
skips first (§8); it protects direct callers.

Shape of the LLR as a function of the ratio `b`:

| Ratio `b` | LLR |
|---|---|
| ≤ 0.75 | +1.6 (maximum) |
| 0.75 to 0.95 | linear from +1.6 down to 0 |
| 0.95 to 1.05 | 0 |
| 1.05 to 1.30 | linear from 0 down to −0.5 |
| ≥ 1.30 | −0.5 (minimum) |

Note the asymmetry: a low ratio is strong machine evidence; a high ratio is
weak human evidence. Source: the two `ramp` terms above.

### 7.3 Burstiness: `burstiness_feature(doc, an)`

```python
per_sent = _sentence_log_ppls(doc, an)
if len(per_sent) < 8:
    return 0.0, 0.2, f"only {len(per_sent)} scoreable sentences — burstiness needs 8+"
mean = fmean(per_sent)
cv = pstdev(per_sent) / mean if mean > 0 else 0.0
llr = 0.7 * ramp(0.22 - cv, 0.0, 0.12)
confidence = 0.5
rationale = (
    f"per-sentence perplexity CV {cv:.2f} over {len(per_sent)} sentences "
    "(humans spike ≥ ~0.25; models are smooth)"
)
```

The under-8 case is a zero LLR with confidence 0.2, not a skip: it still
appears in the contributions list with that rationale. The LLR is +0.7 at
coefficient of variation 0.10 or below, 0 at 0.22 or above, and never
negative.

## 8. The signal class [NORMATIVE]

`StatisticalSignal` is a dataclass implementing the `Signal` protocol:

| Field | Default | Note |
|---|---|---|
| `id` | required | `"statistical.binoculars"`, `"statistical.perplexity"`, `"statistical.burstiness"` |
| `version` | required | `"1"` for all three |
| `compute` | required | `Callable[[Document, StatAnalysis], tuple[float, float, str]]` |
| `needs_performer` | `False` | `True` for Binoculars only |
| `backend` | `None` | `None` means use `get_default_backend()` lazily at score time |
| `cost` | `Cost.GPU` | |
| `latency_budget_ms` | `10_000` | |
| `requires` | `{"text", "logprobs"}` | |
| `half_life_days` | `None` | statistical signals do not decay |

`score(doc, profile)` in this order:

1. `backend = self.backend or get_default_backend()`. If `None`, return
   `SignalResult.skip(self.id, backend_skip_reason(), self.version)`.
2. `an = backend.analyze(doc.text)`.
3. If `self.needs_performer and an.xents is None`, skip with
   `f"needs a performer model — set TELL_PERFORMER_MODEL (same tokenizer family)"`.
4. If `len(an.obs_logprobs) < MIN_TOKENS`, skip with
   `f"only {len(an.obs_logprobs)} scored tokens < {MIN_TOKENS}"`.
5. `llr, confidence, rationale = self.compute(doc, an)`, then
   `llr *= profile.weight_for(self.id)`.
6. Return `SignalResult(signal_id=self.id, llr=llr, confidence=confidence,
   rationale=rationale, version=self.version)`. No spans: these signals do
   not localize. `calibrated` stays `False`.

`statistical_signals(backend=None)` returns the three signals in the order
Binoculars, perplexity, burstiness, each with the given backend. The
Binoculars `compute` is `lambda doc, an: binoculars_feature(an)`. The engine
calls `statistical_signals()` with no argument (`11_SPEC_CORE.md`).

The profile weight for `statistical.perplexity` is 0.3 in the `technical`
profile and 1.0 elsewhere; the three ids are assigned to correlation groups
in the profiles (`binoculars` and `perplexity` to `statistical`,
`burstiness` to `regularity`). An id left out of every group escapes the
fusion discount; the rebuild must keep those assignments.

## 9. Installing the backend and the models [NORMATIVE]

Optional dependency group in `pyproject.toml`:

```toml
[dependency-groups]
statistical = [
    "llama-cpp-python>=0.3",
    "numpy>=1.26",
]
```

Task-runner recipes:

```just
# install the optional llama.cpp backend
setup-statistical:
    uv sync --group statistical

# download the reference model pair (Qwen2.5-0.5B base+instruct GGUF, ~1GB)
# into models/ — Binoculars needs base as observer, instruct as performer
models:
    mkdir -p models
    [ -f models/qwen2.5-0.5b-base-q8_0.gguf ] || curl -fL -o models/qwen2.5-0.5b-base-q8_0.gguf \
        "https://huggingface.co/QuantFactory/Qwen2.5-0.5B-GGUF/resolve/main/Qwen2.5-0.5B.Q8_0.gguf"
    [ -f models/qwen2.5-0.5b-instruct-q8_0.gguf ] || curl -fL -o models/qwen2.5-0.5b-instruct-q8_0.gguf \
        "https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q8_0.gguf"
    @echo "models in place — found automatically; TELL_OBSERVER_MODEL/TELL_PERFORMER_MODEL only needed to override"
```

The two files total about 1.1 GB. The recipe is idempotent: an existing
file is not re-downloaded. `models/` must be in `.gitignore` and must be
excluded from any packaging or deployment archive. Once the files are in
`models/`, no environment variable is needed; the variables exist to point
at a different pair.

Without the group installed or the files present, `tell check` still runs
and the report lists the three signals under skipped with the reason from
§4.4.

## 10. Constants

| Name | Value | Where | Source |
|---|---|---|---|
| `MIN_TOKENS` | 80 | skip threshold on scored tokens | `statistical.py` |
| `n_ctx` | 4096 | llama.cpp context; tokens beyond it are not scored | `LlamaBackend.__init__` |
| Cache size | 4 | `_cache` cleared when it exceeds 4 entries | `LlamaBackend.analyze` |
| Min tokens per sentence | 4 | bucket included in burstiness | `_sentence_log_ppls` |
| Min sentences for burstiness | 8 | below: LLR 0, confidence 0.2 | `burstiness_feature` |
| Perplexity LLR | `0.5 * ramp(2.1 - log_ppl, 0, 0.6)` | max +0.5 | `perplexity_feature` |
| Perplexity confidence | 0.4 | | `perplexity_feature` |
| Binoculars LLR | `1.6 * ramp(0.95 - b, 0, 0.20) - 0.5 * ramp(b - 1.05, 0, 0.25)` | range −0.5 to +1.6 | `binoculars_feature` |
| Binoculars confidence | 0.55 (0.0 when cross-entropy ≤ 0) | | `binoculars_feature` |
| Burstiness LLR | `0.7 * ramp(0.22 - cv, 0, 0.12)` | max +0.7 | `burstiness_feature` |
| Burstiness confidence | 0.5 | | `burstiness_feature` |
| `latency_budget_ms` | 10 000 | | `StatisticalSignal` |
| `cost` | `Cost.GPU` | | `StatisticalSignal` |
| Default observer | `qwen2.5-0.5b-base-q8_0.gguf` | | `_DEFAULT_OBSERVER` |
| Default performer | `qwen2.5-0.5b-instruct-q8_0.gguf` | | `_DEFAULT_PERFORMER` |
| `technical` weight on perplexity | 0.3 | | profiles, `11_SPEC_CORE.md` |

Ground-truth figures recorded by the original author after a manual run
with the default pair (source: the original README; not a test assertion):

| Text | Binoculars ratio | Reading |
|---|---|---|
| Text generated by the instruct model itself | ≈ 0.72 | strong machine signal, LLR +1.6 by §7.2 |
| `seed/human_essay.txt` | ≈ 1.0 | neutral, LLR 0 |

A rebuild should reproduce the order of these two readings with the same
pair; the exact values depend on the llama.cpp build and need not match.

## 11. Decisions already taken [GUIDANCE — do not relitigate]

- **Model pair: Qwen2.5-0.5B base as observer, Qwen2.5-0.5B-Instruct as
  performer, both Q8_0 GGUF.** Small enough to run on CPU in a few seconds
  per document, and base plus instruct of one family guarantees a shared
  tokenizer, which Binoculars requires.
- **llama.cpp through `llama-cpp-python`, not PyTorch.** No CUDA
  dependency, one wheel, runs on a laptop and on a small server.
- **Weights are discovered, not configured.** `models/` beside the project
  is checked before any environment variable is required, so a fresh
  checkout plus `just models` is the whole setup.
- **`models/` is gitignored and excluded from deployment archives.** The
  files are about 1.1 GB and are re-downloadable.
- **Tests stub the backend and never touch weights.** Loading is slow and
  llama.cpp's Metal teardown can abort the interpreter at exit. The autouse
  fixture in `conftest.py` clears both environment variables, points
  `_DEFAULT_MODELS_DIR` at `/nonexistent-tell-models`, and resets
  `_default_backend` and `_backend_error` to `None` before every test.
- **Perplexity never goes negative, Binoculars can.** High perplexity is
  not treated as human evidence because it also describes noise; a high
  Binoculars ratio is treated as weak human evidence because the ratio
  normalizes away prompt and domain effects.
- **Fast-DetectGPT curvature is deferred.** The design listed it as a
  second opinion; the build shipped three methods and left it as a later
  story.
- **Burstiness lives in the `regularity` correlation group with the
  stylometric sentence-shape signal**, not in `statistical`, because both
  measure sentence-level smoothness and would double-count.

## 12. Requirements

| ID | Requirement |
|---|---|
| STAT-01 | `statistical_signals()` must return exactly three signals with ids `statistical.binoculars`, `statistical.perplexity`, `statistical.burstiness`, in that order, each with `version` `"1"`. |
| STAT-02 | When no backend is available, each signal's `score` must return a skipped `SignalResult` whose rationale is `backend_skip_reason()`. |
| STAT-03 | With both environment variables unset and no files in the default models directory, the skip rationale must contain the string `TELL_OBSERVER_MODEL`. |
| STAT-04 | `_resolve` must prefer a set environment variable over the default file, and must return `""` when neither applies. |
| STAT-05 | `statistical_status()` must return the five keys in §4.3 and must not load a model. |
| STAT-06 | `get_default_backend()` must remember a failure in `_backend_error` and must not re-probe on later calls. |
| STAT-07 | The Binoculars signal must skip with a rationale containing `performer` when the analysis has `xents is None`. |
| STAT-08 | Any of the three signals must skip with rationale `only {n} scored tokens < 80` when fewer than 80 tokens were scored. |
| STAT-09 | `perplexity_feature` must return LLR `0.5 * ramp(2.1 - log_ppl, 0.0, 0.6)` with confidence 0.4, where `log_ppl` is the negated mean observer log-probability. |
| STAT-10 | `binoculars_feature` must compute `b = log_ppl / mean(xents)` and return LLR `1.6 * ramp(0.95 - b, 0, 0.20) - 0.5 * ramp(b - 1.05, 0, 0.25)` with confidence 0.55. |
| STAT-11 | `binoculars_feature` must format the ratio to three decimals in its rationale, prefixed `Binoculars ratio `. |
| STAT-12 | `binoculars_feature` must return `(0.0, 0.0, "degenerate cross-entropy")` when the mean cross-entropy is not positive. |
| STAT-13 | `burstiness_feature` must group token log-probabilities by containing sentence, keep sentences with at least 4 scored tokens, and return LLR 0 with confidence 0.2 when fewer than 8 remain. |
| STAT-14 | `burstiness_feature` must return LLR `0.7 * ramp(0.22 - cv, 0, 0.12)` with confidence 0.5, where `cv` is population standard deviation over mean of per-sentence log-perplexity. |
| STAT-15 | `score` must multiply the feature LLR by `profile.weight_for(signal_id)` before returning. |
| STAT-16 | The llama.cpp backend must construct both models with `n_ctx=4096, logits_all=True, verbose=False, n_gpu_layers=-1`. |
| STAT-17 | The backend must score tokens 1..n−1 using the logits at position i−1 for token i, with a full-vocabulary log-softmax, and must never score the BOS token. |
| STAT-18 | Token offsets must be the cumulative code-point length of the decoded pieces of preceding scored tokens, starting at 0. |
| STAT-19 | Cross-entropy at each position must be `-(exp(performer_logprobs) * observer_logprobs).sum()`, truncated to the shorter of the two sequences. |
| STAT-20 | The backend must cache analyses by `hash(text)` and clear the cache when it would exceed 4 entries. |
| STAT-21 | The results of these signals must carry no spans and `calibrated=False`. |
| STAT-22 | The test suite must never load model weights; an autouse fixture must disable discovery before every test. |
| STAT-23 | A `models` task-runner recipe must download the two default files into `models/` idempotently, and `models/` must be gitignored. |

## 13. Test mapping

All tests use `StubBackend(logprob_fn, xent_fn=None)`: it tokenizes on
single spaces, assigns offsets as the cumulative `len(token) + 1`, and
fills `obs_logprobs[i] = logprob_fn(i)` and, when `xent_fn` is given,
`xents[i] = xent_fn(i)`. Names are `stub-obs` and `stub-perf`.
`LONG_TEXT` is `"The quick brown fox jumps over the lazy dog again today. "`
repeated 12 times and stripped: 132 whitespace tokens, 12 sentences of 11
tokens. `_score(sig_id, backend, text=LONG_TEXT, profile="default")` picks
the signal by id from `statistical_signals(backend)` and scores
`segment(text)` under `get_profile(profile)`.

| Test | Setup | Assertions | Proves |
|---|---|---|---|
| `test_signals_skip_without_backend` | `_disable_backend` (same steps as the autouse fixture, dir `/nonexistent`); default backend | For each of the three signals on `LONG_TEXT`: `result.skipped` is true and `"TELL_OBSERVER_MODEL" in result.rationale` | STAT-02, STAT-03, STAT-06 |
| `test_low_perplexity_scores_higher` | `smooth`: every logprob −1.2 (perplexity ≈ 3.3). `surprising`: every logprob −3.0 (perplexity ≈ 20) | perplexity LLR for `smooth` `> 0.3` (it is exactly 0.5); for `surprising` `== 0.0` | STAT-09 |
| `test_binoculars_low_ratio_reads_machine` | `machine`: logprob −1.5, xent 2.5 (ratio 0.6). `human`: logprob −2.5, xent 2.0 (ratio 1.25) | `m.llr > 1.0` (it is 1.6); `h.llr < 0.0` (it is −0.4); `"Binoculars ratio" in m.rationale` | STAT-10, STAT-11 |
| `test_binoculars_skips_without_performer` | stub with no `xent_fn` | `result.skipped`; `"performer" in result.rationale` | STAT-07 |
| `test_burstiness_smooth_vs_spiky` | text: 15 sentences `"Sentence number {i} has exactly six words."` joined by spaces (7 tokens each, 105 tokens). `smooth`: every logprob −1.8. `spiky`: logprob −0.8 when `(i // 7) % 2 == 0`, else −3.5, so sentences alternate | `s.llr > 0.3` (CV 0 gives 0.7); `b.llr < s.llr` (large CV gives 0) | STAT-13, STAT-14 |
| `test_short_text_skips` | perplexity on `"too few words here"` (4 tokens) with logprob −1.0 | `result.skipped` | STAT-08 |
| `test_technical_profile_dampens_perplexity` | `smooth` stub, perplexity under `default` then `technical` | `technical.llr < default.llr` (0.15 vs 0.5) | STAT-15 |
| `test_binoculars_math_ratio` | `StatAnalysis(offsets=[0, 5], obs_logprobs=[-2.0, -2.0], xents=[2.0, 2.0])` passed straight to `binoculars_feature` | `"1.000" in rationale`; `llr < 0.1` (it is 0.0) | STAT-10, STAT-11 |
| `test_end_to_end_reports_skipped_statistical` | `_disable_backend`; `check(ai_text)` on `seed/ai_slop.md` | the set of skipped signal ids includes all three statistical ids | STAT-01, STAT-02 |

Requirements STAT-16 through STAT-20 (the llama.cpp backend) have no
automated test, by the decision in §11. The rebuild verifies them by a
manual run with the default pair: `tell check seed/human_essay.txt --json`
must list the three statistical signals under contributions rather than
skipped, with the observer name `qwen2.5-0.5b-base-q8_0` in the perplexity
rationale.

## 14. Out of scope

- Fast-DetectGPT curvature (§2).
- Any hosted or GPU deployment of the models. The web app runs the same
  in-process backend; whether a server has the weights is an operational
  choice outside this pack.
- Calibrating the thresholds. The numbers in §7 are provisional and the
  rationales say so; replacing them is a later phase that needs labeled
  corpora.
