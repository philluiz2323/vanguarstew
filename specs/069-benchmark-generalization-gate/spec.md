# Spec 069 — generalization gate

- **Status:** draft (SDD Phase 1 — Specify)
- **Owner:** benchmark
- **Issue:** #1910
- **Constitution:** [`AGENTS.md`](../../AGENTS.md) → *Benchmark integrity (M1–M3)*
- **Methodology:** [`blog/spec-driven-development.md`](../../blog/spec-driven-development.md)
- **Related:** [`benchmark/generalization_gate.py`](../../benchmark/generalization_gate.py) (the gate
  under test), [`benchmark/gap_integrity.py`](../../benchmark/gap_integrity.py) (the reported gap's
  integrity, Spec 027), [`benchmark/promotion.py`](../../benchmark/promotion.py) (the promotion
  decision), [`benchmark/acceptance.py`](../../benchmark/acceptance.py) (`_partition_error`),
  [`scripts/generalization_gate.py`](../../scripts/generalization_gate.py) (the CI entry point)

This spec makes the **existing, implicit** generalization contract explicit. It describes the
as-built behavior of `benchmark/generalization_gate.py`; it introduces **no behavior change**.

## Why

M3/M4 ask the agent to hold up on *diverse, unseen* repos, not just the ones it was tuned on.
`run_multi_replay --generalization` reports a `tuned` partition, a `held_out` partition, and a
`generalization_gap` — but nothing turns that into a pass/fail decision. A run that tuned to 0.70
and collapsed to 0.40 on held-out repos has a 0.30 gap that should **block** promotion; today it
flows through unflagged. `check_generalization` gates it: both partitions scored, enough distinct
held-out repos, no partition error, and a tuned-minus-held-out drop within tolerance — recomputing
the gap from the two composites rather than trusting a possibly-stale `generalization_gap` field.

## User stories

1. **As a CI maintainer**, I can gate a `--generalization` run on `scripts/generalization_gate.py`
   and log a stable `generalization_headline()` GENERALIZES/OVERFIT line.
2. **As a benchmark operator**, I can trust GENERALIZES means both partitions really scored (an
   unscored placeholder `0.0` can't masquerade as a real score), enough held-out repos, no error,
   and the drop is within tolerance.
3. **As a reviewer**, every missing-partition, placeholder, error and headline branch is written
   down (addressing the incompleteness class of rejection seen on Specs 057/059).

## Constants

- `DEFAULT_MAX_GAP` SHALL be `0.1`, `DEFAULT_MIN_HELD_OUT_REPOS` SHALL be `3`.
- `_CHECK_ROW_KEYS` SHALL be `("name", "passed")`.

## Acceptance criteria (EARS)

### Helpers

- `_is_number(value)` SHALL be true only for a non-boolean, finite `int`/`float`; a `NaN`/`inf`
  SHALL be false, and an oversized `int` (`math.isfinite` raising `OverflowError`) SHALL be false.
- `_dict(value)` SHALL return `value` when it is a `dict`, otherwise `{}`.
- `_num(value)` SHALL be `f"{value:.3f}"` when `_is_number(value)`, otherwise `"n/a"`.
- `_composite(partition)` SHALL return `None` when the partition's `scored_repos` is `_is_number`
  and falsy (an unscored placeholder), else the `composite_mean` when `_is_number`, else `None`.
  A partition with **no** `scored_repos` key SHALL keep a genuine `composite_mean` of `0.0`.
- `_scored_repos(partition)` SHALL return `scored_repos` when `_is_number`; else, when `per_repo` is
  a list, `len(per_repo)` minus the entries that are dicts with a `_is_number` `tasks == 0`; else
  `None`.

### Gate (`check_generalization`)

- `tuned` / `held` SHALL be `_composite` of the `tuned` / `held_out` partitions; `held_repos` SHALL
  be `_scored_repos(held_out)`; `both` SHALL be `tuned is not None and held is not None`.
- `gap` SHALL be `round(tuned - held, 3)` when `both`, otherwise `None`.
- The result SHALL always carry `passed`, `checks`, `tuned_composite`, `held_out_composite`, `gap`,
  `held_out_repos`, `max_gap`, `min_held_out_repos`; `held_out_repos` SHALL be `held_repos` when
  `_is_number` else `None`; `passed` SHALL be `all(c["passed"] for c in checks)`.
- Four checks SHALL be added in order: `has_partitions`, `no_partition_error`,
  `enough_held_out_repos`, `gap_within_tolerance`.
- `has_partitions` SHALL pass when `both`; detail SHALL be
  `"tuned composite {t}, held-out composite {h}"` when `both`, else
  `"a composite is missing from the tuned or held-out partition"`.
- `no_partition_error` SHALL pass when neither partition's `_partition_error` is set; detail SHALL be
  `"both partitions completed without error"`, else
  `"partition error(s): tuned={tuned_err!r}, held_out={held_err!r}"`.
- `enough_held_out_repos` SHALL pass when `_is_number(held_repos)` and `held_repos >=
  min_held_out_repos`; detail SHALL be `"{held_repos} held-out repo(s) >= {min}"` when numeric, else
  `"held-out repo count unavailable"`.
- `gap_within_tolerance` SHALL pass when `gap is not None and gap <= max_gap` (a held-out score that
  exceeds tuned is a non-positive gap and always within tolerance); detail SHALL be
  `"tuned - held-out = {gap} <= {max_gap}"` when `gap is not None`, else
  `"cannot compare the partitions"`.

### Checks-row sanitation (`_check_rows_list`)

- `None` / non-list `checks` SHALL yield `[]` (with a warning for the non-list case).
- A row SHALL be skipped (with a warning) when it is not a dict, is missing `name`/`passed`, has a
  non-`str` `name`, or a non-`bool` `passed`.
- WHEN `checks` is non-empty but no row survives THEN a warning SHALL be logged.

### Failed checks and headline

- `failed_checks(result)` SHALL return the `name` of every sanitized check whose `passed` is falsy.
- WHEN no sanitized checks exist THEN `generalization_headline` SHALL be exactly
  `generalization: no checks evaluated`.
- WHEN `result.passed` is truthy THEN it SHALL be
  `generalization: GENERALIZES (tuned {t} -> held-out {h}, gap {g})` (each rendered with `_num`).
- OTHERWISE it SHALL be `generalization: OVERFIT ({f}/{n} checks failed: {names})`.

### Pure evaluation

- The module SHALL perform no I/O.
- `check_generalization()` SHALL NOT mutate its input.

## Out of scope

- The reported gap's integrity (`gap_integrity`, Spec 027) and the promotion decision
  (`promotion`).
- Tuning the default thresholds.

## Verification

- `tests/test_spec_069_generalization_gate.py` exercises each EARS block above, pinning **literal**
  expected check names, `passed` values and detail strings.
- Broader coverage (including the CLI) remains in `tests/test_generalization_gate.py`.
