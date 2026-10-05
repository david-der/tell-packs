# CLAUDE.md — TELL rebuild

This repository rebuilds TELL, a composable AI-text detection engine, from
the specification pack in `rebuild/`. Read `rebuild/README.md`,
`rebuild/00_PROGRAM.md`, `rebuild/10_PRD_TELL.md` and
`rebuild/30_BUILD_PLAN.md` before writing any code, and the spec for an
area before touching that area. The pack is the source of truth; when this
file and the pack disagree, the pack wins.

## What is being built

- **Engine** (`src/tell/`): segmenter, signal contract, S3 stylometric
  signals, optional S2 statistical signals, S4 rule packs, fusion, report.
  Specs 11 to 15.
- **CLI** (`tell check | packs | profiles`): spec 15.
- **Web paste UI** (FastAPI on port 8001): spec 16.
- **Rule packs** (`packs/*.yaml`): data. They load unchanged. Do not edit
  them to make a test pass.

Build order and stop points: `rebuild/30_BUILD_PLAN.md`. Stop and report
to a human after milestones M1, M4, and M7.

## How work is done

One story per commit. For each story: restate it in one paragraph with the
requirement IDs it satisfies, implement, run the whole suite with
`just test`, run `just lint`, check each requirement ID against its
spec sentence, commit. Never report a story done with a failing test.

Task runner: `just`. Recipes: `setup`, `test`, `lint`, `fix`, `dev`,
`packs`, `setup-statistical`, `models`, `css`, `css-watch`, `web`,
`screenshot`. Runtime: Python 3.11+, `uv`.

## Rules that are never relaxed

1. **No verdicts.** No headline percentage, no pass or fail, no nonzero
   exit code for a detection result. The CLI exits 0 on every completed
   check.
2. **Uncalibrated means Assist-only.** Every heuristic `SignalResult` has
   `calibrated=False`; every report carries the FPR-not-measured sentence
   verbatim from spec 15.
3. **A single tell is worth almost nothing.** The co-occurrence table in
   spec 14 and the single-fire ceiling in spec 15 are policy. Tests pin
   them; do not loosen a test to make a document score higher.
4. **Absence of a signal is never evidence.** Skipped signals contribute 0
   and are listed.
5. **`context_required` rules never score.** Surface the spans.
6. **Tells decay.** Packs carry `version`, `updated`, `half_life_days`.
7. **Every signal has a correlation group.** When you add a signal, add its
   id glob to the profile groups, or it double-counts.
8. **Offsets are code points into the normalized text.** The web front end
   maps to UTF-16 with `codePointMap`; do not remove it.
9. **Tests never load model weights.** The `no_real_models` autouse
   fixture stays. S2 is tested through a stub backend.
10. **No network in the default path.** No CDN at runtime, no API call, no
    model download except through `just models`.

## Evidence and tests

- Tests: `tests/test_{segmenter,stylometric,statistical,rulepacks,fusion,cli,web}.py`, 58 in the reference build.
- Web evidence: `just screenshot <label>` writes five PNGs to
  `screenshots/` (gitignored). Spec 16 names the states.

## Commit messages

One commit per story. Lead with the story id and what it delivered; end
with the test count. Record any deviation from a spec in
`docs/DECISIONS.md` in the same commit.
