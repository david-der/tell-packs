# 30 · Build plan — milestones, stories, stop points

Each story cites the requirement IDs it satisfies and the tests it must add
or pass. Stories are sized to half a day of assisted work. One story is one
commit. Test names are the reference build's; the rebuild keeps them so the
test mapping in each spec stays usable.

## 1. Order of work

| Milestone | What is green at the end | Stop and report to a human? |
|---|---|---|
| M0 · Scaffold | package imports, `just test` runs zero tests, packs and fixtures in place | No |
| M1 · Core and segmenter | `test_segmenter.py` (6) | **Yes** |
| M2 · Stylometric signals | `test_stylometric.py` (5) | No |
| M3 · Rule packs | `test_rulepacks.py` (14) | No |
| M4 · Fusion, report, CLI | `test_fusion.py` (7), `test_cli.py` (7); the CLI demo steps 1 to 5 | **Yes** |
| M5 · Statistical signals | `test_statistical.py` (9); demo steps 6 and 7 | No |
| M6 · Web UI | `test_web.py` (10); five screenshots; demo step 8 | No |
| M7 · Finish | all 58 tests, lint clean, README, decisions log | **Yes** |

Definition of done: `10_PRD_TELL.md` §5. M2, M3 and M5 depend only on M1
and can be built in any order after it; M4 needs M2 and M3; M6 needs M4.

## 2. Repository layout

See `10_PRD_TELL.md` §4.

## 3. M0 · Scaffold

**M0.S1 Project skeleton.** `pyproject.toml` with the dependencies, dev
and `statistical` groups, console script, ruff and pytest config from
`10_PRD_TELL.md` §3. `justfile` with every recipe name from §3 (recipes
may be stubs that print "not yet"). `src/tell/__init__.py` with
`__version__ = "0.1.0"`. Empty `signals/` and `web/` packages. `.gitignore`
covering `.venv/`, `__pycache__/`, `.pytest_cache/`, `.ruff_cache/`,
`dist/`, `screenshots/`, `tools/tailwindcss`, `models/`, `.DS_Store`.
Refs CORE-13. Checks `uv sync`, `uv run tell --version` once CLI exists.

**M0.S2 Data in place.** Copy `../packs/*.yaml` to `packs/`,
`seed/ai_slop.md` and `seed/human_essay.txt` to `tests/fixtures/`,
`seed/sample-draft.md` to `examples/`. Write `tests/conftest.py` exactly as
spec 13 §9 gives it (the `no_real_models` autouse fixture, `ai_text`,
`human_text`, `packs_dir`). Refs STAT-01. Checks `uv run pytest -q` collects
with no import errors.

Exit: `just test` runs and reports zero tests.

## 4. M1 · Core and segmenter — stop and report after

**M1.S1 Types.** `types.py`: `Cost`, `Span` with `excerpt()`,
`SignalResult` with `skip()`, `Sentence`, `Paragraph`, `Document` with
`word_count` and `prose_sentences`, `Profile` with `weight_for()`, the
`Signal` Protocol, `ramp()`. Refs CORE-01 to CORE-07. Checks: write
`tests/test_types.py` with one assertion per CORE ID (the reference build
has none; add them).

**M1.S2 Segmenter.** `segmenter.py`: the four regexes and the abbreviation
set verbatim from spec 11 §3.1, `normalize`, block splitting,
classification, fence tracking, bullet spans, `split_sentences` with
`_is_boundary`, `segment`. Refs SEG-01 to SEG-12. Checks
`test_sentence_offsets_are_exact`, `test_paragraph_kinds`,
`test_bullets_extracted`, `test_abbreviations_do_not_split`,
`test_crlf_normalized`, `test_word_count`.

**M1.S3 Profiles.** `profiles.py`: `_BASE_GROUPS`, the three profiles with
exact description strings, priors, overrides, scopes, `get_profile` with
its KeyError message. Refs PROF-01 to PROF-04. Checks: add
`tests/test_profiles.py` asserting PROF-03 and PROF-04.

Exit: `test_segmenter.py` green; `segment(ai_text)` reports 10 paragraphs,
19 sentences, one bullet list with 4 bullets (figures from spec 11 §9).

## 5. M2 · Stylometric signals

**M2.S1 Signal class and the short-document gate.** `signals/stylometric.py`:
`StylometricSignal` with id, cost, latency, requires, half_life_days,
version, the `MIN_WORDS` gate and its exact skip string, confidence
plumbing. Refs STY-01 to STY-04. Checks `test_short_document_skips`.

**M2.S2 Sentence shape and paragraph uniformity.** `_excess_kurtosis`,
`_sentence_shape`, `_paragraph_uniformity` with the thresholds and rationale
format strings from spec 12 §4 and §5. Refs STY-05 to STY-11. Checks
`test_uniform_sentences_score_higher_than_bursty`.

**M2.S3 Punctuation, diversity, transitions.** `_punctuation_profile`,
`_lexical_diversity` (window, 200-token floor and its message),
`_transition_density` with the `TRANSITIONS` tuple and regex verbatim. The
`STYLOMETRIC_SIGNALS` roster in order. Refs STY-12 to STY-22. Checks
`test_transition_density_flags_spans`,
`test_technical_profile_dampens_diversity`, `test_human_text_stays_low`.

Exit: `test_stylometric.py` green; on `ai_slop.md` the three nonzero
features are sentence shape +1.138, paragraph uniformity +0.640,
transitions +1.400 (spec 12 §9), and every feature is 0.0 on
`human_essay.txt`.

## 6. M3 · Rule packs

**M3.S1 Loader and validation.** `signals/rulepacks.py`: `Rule`, `Pack`
with `staleness`, `PackError`, `load_pack` with the eight error messages in
order, `load_packs` ordering, `_compile` with the emoji class expansion.
Refs PACK-01 to PACK-07. Checks `test_load_bundled_packs`,
`test_validation_unknown_type`, `test_validation_missing_pattern`,
`test_validation_bad_regex`, `test_emoji_shorthand_matches`,
`test_staleness_decay`.

**M3.S2 Density scoring and the pack signal.** `_DEFAULT_BASELINES`,
`_density_llr`, lexicon, regex and rate evaluators producing spans, the
co-occurrence table, `context_required` surfacing, scope mismatch skip,
the rationale format, `RulePackSignal`. Refs PACK-08 to PACK-14. Checks
`test_single_rule_fire_is_discounted`, `test_context_required_is_unscored`,
`test_surface_as_verb_scored_without_judge`, `test_claudeisms_fire_on_slop`,
`test_gpt_register_fires_on_slop`, `test_chatbot_voice_on_transcript`,
`test_claudeisms_quiet_on_human`.

**M3.S3 Structural detectors.** The six detectors from spec 14 §8 with
their regexes, floors, ramps and strings, registered in
`_STRUCTURAL_DETECTORS` under the exact names the packs reference. Refs
PACK-15 to PACK-20. Checks `test_anaphora_detector`; add one test per
remaining detector (spec 14 §12 lists the IDs with no reference test).

Exit: `test_rulepacks.py` green; `tell packs` (once M4 lands) lists
`claudeisms v9`, `gpt-register v4`, `chatbot-voice v1`, `seo-slop v3`.

## 7. M4 · Fusion, report, CLI — stop and report after

**M4.S1 Engine bus.** `engine.py`: `DEFAULT_PACKS_DIR`, `build_signals`
roster order, `run_signals` error isolation, `check`. Until M5 lands,
`statistical_signals()` may be a stub returning three skipped signals with
the spec 13 skip reason. Refs CORE-08 to CORE-12. Checks: add
`tests/test_engine.py` asserting CORE-08 and CORE-10 with a signal that
raises.

**M4.S2 Fusion.** `fusion.py`: `_group_for`, `_discounts`, the fused value
formula, the 0.25 fired threshold and `SINGLE_FIRE_CEILING`, posterior,
`BANDS` with exact strings, `calibrated`, `_what_would_change` strings and
conditions. Refs FUS-01 to FUS-14. Checks all seven tests in
`test_fusion.py`.

**M4.S3 Report.** `report.py`: `report_dict` with every JSON key from spec
15 §4, `FPR_LINE`, `_ACTIONS`, `_BAND_STYLE`, `render_json`,
`render_terminal` with Rich (header, assessment panel, contributions table,
spans table honoring `max_spans`, the what-would-change panel, the
recommended action line). Refs REP-01 to REP-13. Checks `test_check_json`,
`test_check_terminal_renders`.

**M4.S4 CLI.** `cli.py`: the three subcommands, every flag, default, help
string, exit code and error message from spec 15 §5; `packs` listing with
the stale marker rule. Refs CLI-01 to CLI-12. Checks
`test_check_missing_file`, `test_check_empty_stdin`, `test_packs_listing`,
`test_profiles_listing`, `test_unknown_profile_rejected`.

Exit: demo steps 1 to 5 in `10_PRD_TELL.md` §6 behave as written;
`ai_slop.md` scores at least 2.0 LLR above `human_essay.txt`.

## 8. M5 · Statistical signals

**M5.S1 Backend protocol and discovery.** `signals/statistical.py`:
`StatAnalysis`, `StatisticalBackend`, env var names, default models dir
and filenames, `_resolve`, `statistical_status`, `get_default_backend`
caching, `backend_skip_reason` strings, `MIN_TOKENS`. Refs STAT-01 to
STAT-08. Checks `test_signals_skip_without_backend`,
`test_end_to_end_reports_skipped_statistical`.

**M5.S2 The three features and the signal class.** `_sentence_log_ppls`,
`perplexity_feature`, `binoculars_feature`, `burstiness_feature` with
formulas, thresholds and rationale strings verbatim; `StatisticalSignal`
sharing one analysis across the three ids; `statistical_signals(backend)`.
Refs STAT-09 to STAT-15, STAT-21 to STAT-23. Checks
`test_low_perplexity_scores_higher`,
`test_binoculars_low_ratio_reads_machine`,
`test_binoculars_skips_without_performer`,
`test_burstiness_smooth_vs_spiky`, `test_short_text_skips`,
`test_technical_profile_dampens_perplexity`, `test_binoculars_math_ratio`.

**M5.S3 llama.cpp backend and model download.** `LlamaBackend` per spec
13 §6 (constructor arguments, tokenization, log-softmax logprobs,
cross-entropy, character offsets). The `setup-statistical` and `models`
recipes with the two download URLs. Refs STAT-16 to STAT-20. Checks: the
manual verification run in spec 13 §10 (Binoculars about 0.72 on
instruct-generated text, about 1.0 on `human_essay.txt`); no automated
test loads weights.

Fallback: if llama-cpp-python fails to build on the target machine after
one focused attempt, ship M5.S1 and M5.S2 with the backend absent. The
engine is complete without it; the signals skip with the stated reason.

Exit: with models present, `tell check examples/sample-draft.md --json`
has three `statistical.*` contributions and an empty `skipped` list.

## 9. M6 · Web UI

**M6.S1 API.** `web/__init__.py`: the FastAPI app, `CheckRequest`,
`MAX_BYTES` from `TELL_MAX_BYTES`, `POST /api/check` with the exact
400/413/422 bodies, `GET /api/profiles`, `GET /healthz`, `_instruments()`
with every instrument description string verbatim, static mounting. Refs
WEB-01 to WEB-09. Checks `test_healthz`, `test_profiles_endpoint`,
`test_check_slop`, `test_check_human`, `test_check_rejects_unknown_profile`,
`test_check_rejects_oversize`, `test_check_rejects_empty`.

**M6.S2 Page and styles.** `templates/index.html` with every DOM id and
copy string from spec 16 §3 and the appendix; `static/src/input.css`;
download the Tailwind standalone CLI to `tools/tailwindcss` and build
`static/css/tailwind.css` with `just css`; commit the built CSS. Refs
WEB-10 to WEB-18. Checks `test_index_renders`.

**M6.S3 Front end.** `static/js/app.js`: request flow, `codePointMap`
verbatim, highlight rendering, report cards, profile population, error and
loading states, the How it works dialog. Refs WEB-20 to WEB-25, WEB-30 to
WEB-33, WEB-40 to WEB-45, WEB-50 to WEB-56. Checks
`test_how_it_works_documents_live_instruments`,
`test_glossary_defines_terms_for_novices`; manual: paste a line with an
emoji before a flagged phrase and confirm the highlight lands on the
phrase.

**M6.S4 Screenshot tour.** `scripts/screenshot.py` per spec 16 §7: side
port 8129, the five states, viewports 1440×1000 and 390×844, file naming.
Refs WEB-60. Checks: `just screenshot rebuild` writes five PNGs that match
`images/01` to `images/05` in layout.

Exit: `test_web.py` green; the five screenshots exist.

## 10. M7 · Finish — stop and report after

**M7.S1 Lint and suite.** `just lint` clean, all 58 reference tests plus
the added ones green in under five seconds without model weights.

**M7.S2 README and decisions.** Project README with quick start, the
glossary from `rebuild/README.md`, the load-bearing rules, and the S2
section. `docs/DECISIONS.md` listing every place the build deviated from a
spec, with one line of why.

**M7.S3 Demo.** Run all eight demo steps from `10_PRD_TELL.md` §6 and
record the output in `docs/DEMO.md`.

## 11. Requirement → story index

| Prefix | IDs | Stories |
|---|---|---|
| CORE | 01–07 | M1.S1 |
| CORE | 08–12 | M4.S1 |
| CORE | 13 | M0.S1 |
| SEG | 01–12 | M1.S2 |
| PROF | 01–04 | M1.S3 |
| STY | 01–04 | M2.S1 |
| STY | 05–11 | M2.S2 |
| STY | 12–22 | M2.S3 |
| PACK | 01–07 | M3.S1 |
| PACK | 08–14 | M3.S2 |
| PACK | 15–20 | M3.S3 |
| FUS | 01–14 | M4.S2 |
| REP | 01–13 | M4.S3 |
| CLI | 01–12 | M4.S4 |
| STAT | 01–08 | M5.S1 |
| STAT | 09–15, 21–23 | M5.S2 |
| STAT | 16–20 | M5.S3 |
| WEB | 01–09 | M6.S1 |
| WEB | 10–18 | M6.S2 |
| WEB | 20–25, 30–33, 40–45, 50–56 | M6.S3 |
| WEB | 60 | M6.S4 |

Where a spec's own section numbering groups IDs differently from the
ranges above, the spec wins; adjust the story, not the spec.

## 12. Risks and mitigations

- **Offsets drift between segmenter and renderer.** Mitigation: SEG-09 is
  tested on the human fixture at M1, and M6.S3 includes the emoji manual
  check before the milestone closes.
- **A test is loosened to make the slop fixture score higher.** Mitigation:
  the fusion ceilings are policy; `CLAUDE.md` rule 3 forbids it, and
  `docs/DECISIONS.md` must record any threshold change.
- **llama-cpp-python does not build.** Mitigation: the M5.S3 fallback; the
  engine is complete without the backend.
- **Pack version bump breaks the web suite.** Spec 16 decision 6 notes
  that one web test hard-codes `lexical.claudeisms.v9`. Mitigation: the
  rebuild may assert on the pack name prefix instead and record the
  deviation.
- **Tailwind binary unavailable offline.** Mitigation: the built
  `tailwind.css` is committed; `just css` is only needed when
  `input.css` changes.
