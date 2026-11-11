# Design: Hub-wide unique sample IDs

Companion to the [project poster](sample-ids.md). The poster sets out the problem, the ID format and the decisions. This document holds the function prototypes and their callers, and the detail behind mixed coarseness, the hash settings, recasting integer IDs and samples shared across modeling tasks.

Terms are used as the poster defines them in [Key terms](sample-ids.md#key-terms).

- [Functions](#functions)
  - [`hub_hash()`](#hub_hash)
  - [`create_hub_spl_ids()`](#create_hub_spl_ids)
  - [`submission_tmpl_samples()`](#submission_tmpl_samples)
- [Callers](#callers)
- [Mixed coarseness within a modeling task](#mixed-coarseness-within-a-modeling-task)
- [Hash settings](#hash-settings)
- [Integer-ID hubs](#integer-id-hubs)
- [Samples shared across modeling tasks](#samples-shared-across-modeling-tasks)

## Functions

### `hub_hash()`

In hubUtils.

```r
hub_hash(..., hash_length = 12L, hash_fn = NULL)
```

- **`...`**: named components. Each is an atomic vector, or a list of character vectors for components that are sets. Components are recycled, so each has length 1 or n. Names are required and unique, and are part of the hashed input.
- **`hash_length`**: the number of leading hex characters kept, from 1 to 64.
- **`hash_fn`**: `NULL` for SHA-256, or a function that takes the input strings and returns hashes. A custom function is not reproducible by other tools.

The input string for each element is `name=value|name=value`, in argument order. Atomic values are converted with `as.character()`, and `NA` becomes `"NA"`. Sets are sorted and joined with `,`.

`hub_hash()` knows nothing about hubs. Callers hash each distinct context once and map the result back to rows, never hashing per row. They always pass the model ID, the round ID and the compound unit's values, named by task ID, then, for a shared sample, the group's modeling task indexes, in that order. It uses `openssl::sha256()`, which is vectorised: a million inputs take about 1.5 s. `rlang::hash()` is not used, because its output cannot be reproduced outside R.

### `create_hub_spl_ids()`

In hubUtils, re-exported by hubValidations ([#409](https://github.com/hubverse-org/hubValidations/issues/409)). Modeling task matching needs nothing hubUtils does not already import. The only new dependency is the hashing package `hub_hash()` needs.

```r
create_hub_spl_ids(
  tbl, hub_path = NULL,
  model_id = NULL, round_id_col = NULL,
  compound_taskid_set = NULL, subset = NULL,
  hash_length = NULL,
  show_config = FALSE
)
```

- **`tbl`**: a submission file or a `model_out_tbl`, of any number of models and rounds. Rows of other output types are returned unchanged. Integer sample IDs are recast to character IDs in the new format.
- **`hub_path`**: a path or a `SubTreeFileSystem`. May be `NULL` when `tbl` carries the stamp added by hubUtils [#310](https://github.com/hubverse-org/hubUtils/issues/310), which includes the compound task ID set and the hash setting.
- **`model_id`**: required when `tbl` has no `model_id` column. An error if it disagrees with the column.
- **`round_id_col`**: `NULL` for the hub's round ID column, following the rule in hubUtils #310.
- **`compound_taskid_set`**: `NULL`, or a list with one element per modeling task. A `NULL` element means the configured set. An override must match the configured set or be coarser.
- **`subset`**: `NULL`, or a named list of task ID values. Rows outside the subset are returned unchanged, so calls can be chained. Passed to modeling task matching, which applies it.
- **`hash_length`**: `NULL` for the length from `get_spl_hash_settings()`, which reads the hub's `sample_id_hash` setting from the config or the stamp, or 12 where the hub has none. A value overrides it.
- **`show_config`**: `TRUE` prints the `sample_id_hash` setting for the top level of `tasks.json` that matches the generated IDs, ready to paste in.

### `submission_tmpl_samples()`

In hubValidations. Argument names follow `submission_tmpl()`, which already has `compound_taskid_set`.

```r
submission_tmpl_samples(
  path, round_id,
  compound_taskid_set = NULL, subset = NULL, n_samples = NULL,
  model_id = NULL, hash_length = NULL,
  required_vals_only = FALSE, complete_cases_only = TRUE,
  derived_task_ids = NULL
)
```

- **`subset`**: as in `create_hub_spl_ids()`. Passed to grid expansion, which applies it and errors on values the config does not allow.
- **`n_samples`**: `NULL`, an integer, or a list with one element per modeling task. The number of samples per compound unit. The default is `min_samples_per_task`, and the value must be within the configured minimum and maximum. A coarser sample counts once in each compound unit it covers, as in hubValidations [#403](https://github.com/hubverse-org/hubValidations/pull/403).
- **`model_id`**: `NULL` for plain indexes of the configured sample ID `type`. A string for hashed IDs, which need the character type. The hash arguments are ignored when `model_id` is `NULL`.

Returns sample rows only.

## Callers

| Caller | Has | Needs |
|---|---|---|
| A team before submitting, calling `create_hub_spl_ids()` on a file | one file, one round, `hub_path`, no `model_id` column | the `model_id` argument; the configured or a declared coarser set; IDs that match the hub's `sample_id_hash`, or fit its `max_length` |
| Collected data (hubData) | many models and rounds; the hubUtils #310 stamp | the set from the stamp; opt-in, because `collect_hub()` does not rewrite submitted IDs by default |
| hubEnsembles, hubEvals | a `model_out_tbl`, sometimes without a config | the set as an argument; samples split to the set before ensembling |
| `submission_tmpl_samples()` | the config only | samples generated from the declared structure |

## Mixed coarseness within a modeling task

A modeling task whose samples mix coarseness (allowed by hubValidations [#404](https://github.com/hubverse-org/hubValidations/pull/404)) gets its IDs from chained calls. Each call is limited by a `subset` of task ID values and given its own set, and rows outside its subset are returned unchanged (hubValidations [#422](https://github.com/hubverse-org/hubValidations/issues/422)).

## Hash settings

The hub-level `sample_id_hash` object is introduced in the poster's [hash settings](sample-ids.md#hash-settings-in-the-config).

**`length`** is the number of hex characters in the hash, 12 by default. The hash is computed for every compound unit, so a large hub holds up to about a million distinct hashes, and every unit uses positions from 1. A hash collision is then an ID collision. With a million hashes, the chance of any collision is almost certain with 8 characters, about 0.2% with 12 and negligible with 16.

The functions read this setting, and fall back to 12 characters when there is none.

It is either one or the other. When a hub sets `sample_id_hash`, the ID format is fixed, and the expected length and pattern follow from it. The sample's `max_length` and `pattern` in `output_type_id_params` then carry no information of their own, and could only contradict it:

- `validate_config()` errors when a hub sets `sample_id_hash` and a sample's `output_type_id_params` also sets `max_length` or `pattern`;
- validation derives the length and pattern from `sample_id_hash`, and hubAdmin's `create_output_type_sample()` leaves them out.

Hubs with free-form character IDs keep setting `max_length` and `pattern` as now, and the schema requires `max_length` only for them.

A round-level override can be added later if a hub ever needs a longer hash for new rounds. The parts of the hash that are the same in every hub (the model ID, the round ID, the named compound unit values, the input string and the hash function) are documented once in hubDocs.

### A default `max_length` of 25 until then

Until a hub sets `sample_id_hash`, teams that generate IDs need the hub's `max_length` to allow them. There is no default today: the tasks schema requires `max_length` for character IDs and gives 6 as an example, and hubAdmin's `create_output_type_sample()` defaults it to `NULL`. A 16-character hash, a 2-digit modeling task segment, two separators and a 4-digit index make 24 characters, which 25 allows. The default goes into hubAdmin's `create_output_type_sample()`, hubTemplate and the hubDocs examples (schemas #138).

## Integer-ID hubs

Integer sample IDs are recast to character IDs in the new format by `create_hub_spl_ids()`, in `collect_hub()` when asked to or as a separate step on a `model_out_tbl`. Recasting uses the hub's hash setting where there is one, and the function defaults otherwise, unless the caller overrides them. IDs recast with the defaults are reproducible; IDs recast with overrides are not, unless the overrides are shared.

## Samples shared across modeling tasks

An optional add-on, motivated in the poster's [section](sample-ids.md#samples-shared-across-modeling-tasks-optional-add-on).

**The setting.** An opt-in config setting, at hub or round level, lists the groups of modeling tasks in a round that may share samples. A modeling task is in at most one group. Without the setting, every sample belongs to one modeling task. It is opt-in because otherwise numbering samples per modeling task would silently make them joint across modeling tasks. Where it lives in `tasks.json` is open.

**Conditions.** The modeling tasks in a group must have identical sample `output_type_id_params`, which hubAdmin checks at config validation, and the task ID that separates them must be left out of `compound_taskid_set`. Each modeling task's part of a shared sample is validated against the shared params, as hubValidations #403 does for coarser samples.

**IDs.** A group allows sharing but does not require it, so a round can hold three kinds of sample. Take modeling tasks 1 and 2 as a group, and 3 outside any group:

| Sample | ID shape | Example ID |
|---|---|---|
| In modeling task 3 | `<hash>_<mt>_<index>` | `3fa9c2e1b07d_3_042` |
| In modeling task 1 or 2, not shared | `<hash>_<mt>_<index>` | `3fa9c2e1b07d_1_042` |
| Shared by modeling tasks 1 and 2 | `<hash>_<index>` | `7b04d9a25e1c_042`, in the rows of both |

A shared sample has no single modeling task for the `<mt>` segment, so the segment is dropped and the group's modeling task indexes go into the hash instead. This keeps two groups in one round apart even when they have the same model, round and compound unit values. The shape declares whether a sample is shared, which validation can check once sharing relaxes `check_tbl_spl_mt_unique()`:

- an ID with an `<mt>` segment stays within that modeling task;
- an ID without one stays within one group, and its hash matches that group. It may cover only part of the group.

Without the shape, independent samples numbered the same way in each modeling task of a group would be read as shared. The pattern derived from `sample_id_hash` accepts both shapes in a hub that allows sharing, for example `^[0-9a-f]{12}(_[0-9]+)?_[0-9]+$`. See the [comment on hubValidations #409](https://github.com/hubverse-org/hubValidations/issues/409#issuecomment-6662695134).
