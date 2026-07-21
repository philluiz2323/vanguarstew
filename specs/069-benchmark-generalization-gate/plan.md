# Plan 069 — generalization gate

- **Status:** draft (SDD Phase 2 — Plan)
- **Spec:** [`spec.md`](./spec.md) · **Issue:** #1910

Maps the [spec](./spec.md) onto `benchmark/generalization_gate.py` as-built. No product code.

## EARS → test mapping

| Spec section | Test group in `test_spec_069_generalization_gate.py` |
| ------------ | ---------------------------------------------------- |
| Constants | `test_constants_are_pinned` |
| Helpers | `test_is_number_semantics`, `test_is_number_rejects_oversized_int`, `test_num_formats_or_na`, `test_composite_masks_unscored_placeholder`, `test_scored_repos_prefers_count_then_per_repo` |
| Gate | `test_result_carries_all_keys`, `test_generalizes_passes_all_checks`, `test_overfit_gap_exceeds_tolerance`, `test_missing_partition_fails_has_partitions`, `test_partition_error_fails_no_partition_error`, `test_too_few_held_out_repos_fails`, `test_held_out_exceeding_tuned_is_within_tolerance` |
| Checks-row sanitation | `test_check_rows_list_skips_malformed_rows`, `test_check_rows_list_rejects_non_bool_passed`, `test_check_rows_list_warns_when_all_unusable` |
| Failed checks and headline | `test_failed_checks_names`, `test_headline_no_checks`, `test_headline_generalizes`, `test_headline_overfit_lists_failures` |
| Pure evaluation | `test_check_does_not_mutate_artifact` |

## Verification strategy

One contract-test group per EARS section; every missing-partition / placeholder / error / non-list
branch called out in the spec has an asserting test (lessons from the Spec 057 / 059 rejections).
Expectations are pinned as **literal** check names, `passed` booleans and detail strings — e.g. a
tuned `0.70` / held-out `0.40` run fixes `gap == 0.3` and `gap_within_tolerance` failing — rather
than re-deriving them from the module, so a silent contract change is caught here instead of masked.
Integration and CLI coverage stay in `tests/test_generalization_gate.py`.
