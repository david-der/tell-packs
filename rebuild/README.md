# Rebuild pack for TELL

This directory is enough to rebuild TELL, the detection engine behind the
rule packs in `../packs/`, without access to the original source. It is
written for an engineer working with Claude Code or a similar coding
assistant. Start a session in an empty repository, copy `CLAUDE.md` from this
directory to the repository root, and paste the kickoff prompt at the end of
this file.

The engine is small: about 2,300 lines of Python, five source packages, 58
tests. A careful assistant can rebuild it in a day or two of turns if it
follows `30_BUILD_PLAN.md` in order.

## File map

| File | What it holds | Requirement prefix |
|---|---|---|
| `README.md` | This orientation, conventions, ID index, kickoff prompt | |
| `CLAUDE.md` | Standing instructions for the receiving repository. Copy to its root. | |
| `00_PROGRAM.md` | Thesis, the design principles that are never relaxed, decisions already taken, glossary | |
| `10_PRD_TELL.md` | Product requirements: claims to prove, scope, stack reference, definition of done, demo script | |
| `11_SPEC_CORE.md` | Types, signal contract, segmenter, engine bus, profiles | `CORE`, `SEG`, `PROF` |
| `12_SPEC_STYLOMETRIC.md` | S3 stylometric signals: sentence shape, paragraph uniformity, punctuation, diversity, transitions | `STY` |
| `13_SPEC_STATISTICAL.md` | S2 statistical signals: perplexity, Binoculars, burstiness, the llama.cpp backend, model discovery | `STAT` |
| `14_SPEC_RULEPACKS.md` | S4 rule-pack loader, the four rule types, density scoring, co-occurrence discount, decay, structural detectors | `PACK` |
| `15_SPEC_FUSION_REPORT_CLI.md` | Fusion (groups, discounts, caps, bands), the evidence report in terminal and JSON, the CLI | `FUS`, `REP`, `CLI` |
| `16_SPEC_WEB.md` | The paste web UI and its JSON API, with screenshots | `WEB` |
| `30_BUILD_PLAN.md` | Milestones and half-day stories, stop points, requirement-to-story index, risks | |
| `seed/ai_slop.md` | Test fixture: a deliberately machine-flavored draft. Travels verbatim. | |
| `seed/human_essay.txt` | Test fixture: a human-written essay. Travels verbatim. | |
| `seed/sample-draft.md` | The bundled example the CLI checks by default and the web screenshots use | |
| `images/NN_*.png` | Screenshots of the web UI, numbered in tour order; captions in `images/MANIFEST.txt` | |
| `../packs/*.yaml` | The four shipped rule packs. The rebuilt engine loads them unchanged. | |

Read in numeric order. Every spec is self-contained except where it names a
dependency in its header.

## Conventions

**Normative language.** *Must* is a requirement a test can fail. *Should* is
the reference behavior; deviate only with a note in `docs/DECISIONS.md` of
the receiving repository. *May* is optional. Sections are marked
**[NORMATIVE]** when the exact algorithm, string, or number is the
requirement, and **[GUIDANCE]** when the section explains why.

**Requirement IDs.** `<PREFIX>-NN`, two digits, defined in exactly one spec's
requirements list. Stories in the build plan cite them. Prefixes are in the
file map above.

**Offsets.** Every span offset anywhere in the engine is a Python string
index (a Unicode code point index) into the normalized document text, never
into the raw input and never a byte or UTF-16 offset. The web front end
converts code points to UTF-16 before slicing. This rule appears in every
spec that touches spans because getting it wrong is the most common rebuild
failure.

**Units.** Log-likelihood ratios (LLR) are natural-log units. Densities are
hits per 1,000 words. Dates are ISO 8601. Word count is
`len(text.split())`.

**Figures have sources.** Every number in a spec says where it came from: a
constant in the reference code, a test assertion, or a computation run on a
seed file. Where two specs mention the same number, the spec whose prefix
owns the area is authoritative.

**Stack is a reference, not a mandate.** The reference build is Python 3.11,
`uv`, a `src/` layout with package `tell`, PyYAML, Rich, FastAPI, Uvicorn,
Jinja2, Tailwind for the web CSS, llama-cpp-python and numpy for the
optional S2 signals, pytest, ruff, Playwright for screenshots. Contracts,
constants, strings, and tests are the mandate. Rebuilding in another
language is allowed if every requirement still holds.

## Sanitization

The original lives in a private repository and deploys to a personal
server. Nothing about that deployment is in this pack. The following were
removed or replaced; the rebuilt project supplies its own values.

| Original | In this pack | Where it becomes configuration |
|---|---|---|
| Hosted URL of the paste UI | not mentioned | The receiving team's own host |
| Deploy recipes (tarball, object store, remote shell) | omitted | Out of scope; the web app is one `uvicorn` process |
| Owner's name and email | "the author" | `pyproject.toml` authors field |
| Absolute paths on the author's machine | relative paths | none |
| `models/` directory contents (two GGUF files, about 1.1 GB) | download commands only | `just models` |

## Never list

These hold in every file of this pack and in the rebuilt product. The
receiving `CLAUDE.md` repeats them.

1. No verdicts. No headline percentage, no pass or fail, no nonzero exit
   code for "AI detected".
2. Every heuristic LLR is uncalibrated and the report says so.
3. A single tell is worth almost nothing. The co-occurrence discount and the
   single-fire ceiling are policy, pinned by tests.
4. Absence of a signal is never evidence.
5. `context_required` rules surface spans and never score.
6. Packs carry `version`, `updated`, and `half_life_days`; a rule change
   bumps `version`.
7. A new signal must be assigned to a correlation group, or it escapes the
   discount and double-counts.

## Glossary

| Term | Meaning |
|---|---|
| Tell | A feature of text that occurs more often in unedited model output than in human writing |
| LLR | Log-likelihood ratio, natural log of P(observation given machine) over P(observation given human) |
| Signal | A plugin that scores a document and returns an LLR, confidence, spans, and a rationale |
| S2, S3, S4 | Signal classes from the product design: statistical, stylometric, lexical rule packs |
| Pack | One YAML file of rules, versioned and decaying |
| Profile | A detection context: prior, per-signal weight overrides, correlation groups, scopes |
| Correlation group | A set of signals that partly measure the same thing; fusion discounts all but the strongest |
| Band | One of `none`, `mild`, `elevated`, `substantial`, chosen by total LLR floor |
| Posterior | sigmoid(prior + total LLR), reported in JSON only and never as a verdict |
| Binoculars | Observer-model perplexity divided by observer-performer cross-perplexity; low means machine |
| Burstiness | Coefficient of variation of per-sentence perplexity; humans spike, models are smooth |
| Assist mode | The writer-facing mode; the only mode a signal with unmeasured false-positive rate may run in |

## Kickoff prompt

Paste this as the first message of a Claude Code session in an empty
repository that contains this `rebuild/` directory and the `packs/`
directory beside it:

```
Rebuild TELL from the specification pack in rebuild/. Read rebuild/README.md,
rebuild/00_PROGRAM.md, rebuild/10_PRD_TELL.md and rebuild/30_BUILD_PLAN.md
first, then the spec for each area before touching it. Copy
rebuild/CLAUDE.md to the repository root. Work the build plan in order, one
story per commit, and stop to report after each milestone marked as a stop
point. The packs in packs/ are data and load unchanged. Never add a verdict,
a percentage headline, or a nonzero exit code for a detection result.
```
