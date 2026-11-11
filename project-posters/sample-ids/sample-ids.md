# Project Poster: Hub-wide unique sample IDs

- Date: 2026-11-09

- Owner: Anna Krystalli

- Team: TBD

- Status: draft

Design detail (function prototypes and their callers, mixed coarseness, the hash settings, recasting integer IDs, and samples shared across modeling tasks) is in the companion [design document](design.md).

## ❓ Problem space

### What are we doing?

Defining a hubverse-wide format for sample `output_type_id`s, and the tooling that generates IDs in that format. A sample ID in the new format is:

- **unique** across models, rounds and modeling tasks;
- **deterministic**: the same sample in the same context always gets the same ID, whoever generates it, whenever, and whatever else is in the table;
- **parsable, for verification**: its segments split on `_`, so a tool can recompute each segment from the row and the config and check that it matches. The data already says which model, round and modeling task a row belongs to, so parsing is for checking that an ID was generated from the declared structure, not for reading that information out of it.

It also records the dependence structure the sample was drawn under:

```
3fa9c2e1b07d_2_042
<hash>_<mt>_<index>
```

The hash identifies the context the sample was drawn in, down to its compound unit, and the modeling task segment and the index locate the sample within it. One generator serves every point where sample IDs are made or combined:

- **Preparing a submission:** teams generate IDs for their samples instead of numbering them by hand, or start from a template of correctly structured samples generated from the hub config.
- **Collecting data:** collected model output can be given IDs in the new format, whatever IDs were submitted, including integer IDs. This standardises the samples to one compound task ID set per modeling task across the dataset. The set is the configured one by default, read from the hub config or from the values stamped on collected data (see below), and can be overridden with a coarser one. Samples coarser than the set are split to it, and dependence beyond the set is dropped.
- **Ensembling and scoring:** combined model output gets IDs that are unique across models and rounds, split to the compound task ID set the method needs.
- **Casting to the sample subclass:** possibly, the sample subclass of `model_out_tbl` ([subclasses poster](../model-out-subclasses/model-out-subclasses.md)) gets IDs in the new format when it is constructed. This likewise standardises the samples to one compound task ID set per modeling task, by default the configured one stamped on collected data ([stamping](../model-out-subclasses/model-out-subclasses.md#populating-config-derived-values), hubUtils [#310](https://github.com/hubverse-org/hubUtils/issues/310)), or a coarser one given to the constructor.

### Key terms

- **Sample ID**: the `output_type_id` of a row of output type `sample`. All rows that share a sample ID form one joint draw.
- **Modeling task**: one entry of a round's `model_tasks` in `tasks.json`, a combination of task ID values with its own output types.
- **Compound task ID set**: the task IDs whose values are fixed within one sample (`compound_taskid_set` in the config). The task IDs left out are the ones a sample is joint across.
- **Compound unit**: one combination of values of the compound task ID set. A sample covers one compound unit, or several when it is coarser than the declared set.
- **Coarser and finer samples**: a sample is coarser than the declared set when it is joint across more task IDs than the set leaves out, and finer when it is joint across fewer.
- **Context hash**: the leading segment of a sample ID, a fixed-length hex hash of the model ID, the round ID and the values of the sample's compound unit, with their task ID names. The names record the declared compound task ID set.

### Why are we doing this?

- **Sample IDs collide once data is combined.** A sample ID identifies a joint draw only within one submission file. Teams reuse the same IDs across models and rounds, so combined data holds the same ID for unrelated draws. Hashing the model, round and compound unit into the ID makes it unique across the hub.
- **Reindexing after collection is not deterministic.** Today, code that combines samples has to reindex them, or group by model, round and compound task ID set alongside the ID. A reindexed ID depends on what else is in the table, so outputs reindexed separately can still collide when combined (hubEnsembles [#205](https://github.com/hubverse-org/hubEnsembles/issues/205), hubEvals [#101](https://github.com/hubverse-org/hubEvals/issues/101)). A hashed ID depends only on the sample's own context, so it is the same however the data is split.
- **There is no convention for sample IDs.** A hub can describe the scheme it wants only in documentation ([reichlab/flu-metrocast#570](https://github.com/reichlab/flu-metrocast/issues/570)), and nothing checks that teams follow it. A shared generator, and later schema settings, give every team in a hub the same IDs for the same structure.
- **Misindexing within a file is not caught.** Rows that share an ID are read as one joint draw, so IDs reused across locations by mistake declare independent draws joint across locations. Samples coarser than the compound task ID set are valid, so validation passes the file ([hubValidations #409](https://github.com/hubverse-org/hubValidations/issues/409#issuecomment-6627635951)). IDs generated from the declared structure prevent this.

### What are we _not_ trying to do?

- Not changing submitted data. `collect_hub()` and other read paths do not rewrite submitted sample IDs unless asked to.
- Not detecting compound task ID sets from data. The set is always declared in the config, or given explicitly as a coarser override.
- Not requiring the new format in submissions. Hubs with integer sample IDs keep them, and their data can be recast to the new format, or reindexed, after it is read.

### How do we judge success?

- A team can generate valid, hub-wide unique sample IDs for a submission file with one call, without choosing a numbering scheme.
- Model output combined across models and rounds, with IDs from the generator, can be ensembled and scored without reindexing to make IDs unique.
- Collected model output can be given IDs in the new format, in `collect_hub()` or as a separate step before ensembling or scoring, whatever IDs were submitted.
- IDs do not change when values are added to a round's config.
- Generated IDs validate in every listed hub with character sample IDs, once the hub sets `sample_id_hash` or raises its `max_length`.
- Generating IDs for the same samples again gives the same IDs, whether before submission, after collection or before ensembling. The exception is samples submitted coarser than the set used later, which are split to that set and get new IDs.
- Parts of a submission generated in separate calls (by modeling task, by subset, in chunks or with different compound task ID sets, which hubValidations [#400](https://github.com/hubverse-org/hubValidations/issues/400) allows) bind together without ID collisions.
- A team can generate a template of correctly structured samples from the config alone.

### What are possible solutions?

A hash of the context a sample was drawn in, joined to an index that locates the sample within it. See "Ready to make it" for the design and the alternatives considered.

## ✅ Validation

### What do we already know?

**What hubs submit today.** Of the 28 hubs listed on hubverse.io (configs fetched 2026-11-05), 15 accept samples and 11 have received them. A scan of their submitted sample IDs (all rounds for most hubs, recent rounds for the largest):

| Hub | Type, `max_length` | Typical IDs | Samples per ID string | Models with an ID in several compound units |
|---|---|---|---|---|
| CDCgov covid19-forecast-hub | character, 15 | `AK_100` | 9.4 | 3 of 9 (across targets) |
| CDCgov rsv-forecast-hub | character, 15 | `AK_100` | 5.0 | 3 of 8 (across targets) |
| FluSight | character, 15 | `AK_1`, `AK00`, `US800` | 6.8 | 1 of 13 |
| flu-metrocast | character, 20 | `S1`, `s001`, `colorado-springs100` | 2.1 | 12 of 15 |
| variant-nowcast-hub | character, 15 | `AZ1`, `CA00`; the hub ensemble uses `UGA-multicastCA100` (over `max_length`) | 7.2 | none since a set was configured |
| variant-nowcast-model-dev-retro | character, 15 | `UT20` | 12.9 | no set configured |
| BDBV | integer | `1` to `100` | 22 | 10 of 11 |
| four scenario modeling hubs | integer | none: `output_type_id` is `NA`, and samples are identified by `run_grouping` and `stochastic_run` | 1.6 to 2.3 | not comparable |

"Samples per ID string" is the number of distinct (model, round, ID) combinations per distinct ID, so each ID string stands for that many unrelated samples once data is combined. An ID in several compound units is either a joint sample or a reused label, and the data cannot tell which. Many teams already put the compound unit into the ID by hand, as in `AK_100`.

**The length the format needs.** With a 12-character hash, every hub with samples would need 18 characters: `max_samples_per_task` is between 100 and 600 everywhere, so the index takes 3 digits, and no round has more than 9 modeling tasks. Every character-ID hub except flu-metrocast would need to raise its `max_length`.

## 👍 Ready to make it

### Proposed solution

#### The ID format

`<hash>_<mt>_<index>`, for example `3fa9c2e1b07d_2_042`.

- **`<hash>`**: the context hash.
- **`<mt>`**: the index of the modeling task within the round's `model_tasks`, counting from 1, zero-padded to the number of modeling tasks in the round.
- **`<index>`**: the sample's position among the samples in its compound unit, counting from 1, zero-padded to the width of `max_samples_per_task`.
- **`_` separates every segment**, so an ID parses without knowing the hash length, even though hex hashes contain digits. `-` is not used, because it already joins team and model in `model_id` and appears in date round IDs.

What each part keeps apart:

| Part | Keeps apart | Why it is needed |
|---|---|---|
| #️⃣ Model ID (hashed) | samples from different models | teams reuse the same IDs across models |
| #️⃣ Round ID (hashed) | samples from different rounds | teams reuse the same IDs across rounds |
| #️⃣ Compound unit values, with their task ID names (hashed) | samples in different compound units, and samples generated with different sets | lets parts generated separately combine without collisions; the names record the compound task ID set, so calls with different sets (mixed coarseness) cannot collide |
| `<mt>` | samples from different modeling tasks in one round | modeling tasks in a round can differ only on task IDs outside the compound task ID set, such as `target`, so their compound unit values can be the same |
| `<index>` | samples in the same compound unit | the samples drawn for one compound unit |

Including the unit values costs nothing in length, because the hash has a fixed length.

#### One declared set per modeling task

Each call generates IDs for one compound task ID set per modeling task. By default this is the set in the config. A caller can override it with an explicit coarser set. Every sample is made to conform to the set used:

- a sample that matches the set keeps its rows together;
- a coarser sample is split to the set, with a message that dependence beyond the set was dropped;
- a finer sample is an error.

The hash always records the declared set. A modeling task that mixes coarseness (allowed by hubValidations #404) gets its IDs from separate calls, one per set ([design](design.md#mixed-coarseness-within-a-modeling-task)).

#### Compound units in the hash

The hash covers the sample's model, round and compound unit values, so every compound unit has its own hash. Within a unit, samples are numbered from 1 in the order of their submitted IDs. For example, with a compound task ID set of `origin_date` and `location`, location `01`'s samples are `3fa9c2e1b07d_1_001` to `_100`, and location `02`'s start again at `9c04e1d2a7f3_1_001`.

An ID therefore depends only on the sample's own values. Parts of a submission generated separately combine without collisions, and adding a value to a round's config does not change existing IDs.

#### Samples shared across modeling tasks (optional add-on)

This add-on can be delivered after the core ID format. It is in this poster because its config setting is related to the hash setting and worth designing alongside it.

**Why.** In flu-scenario-modeling-hub, each target with samples is its own modeling task, and the hub [asks teams](https://github.com/midas-network/flu-scenario-modeling-hub/blob/main/model-output/README.md) to give the same sample tag to every target that comes from the same trajectory. Those samples span modeling tasks, which hubValidations rejects (`check_tbl_spl_mt_unique()`, [#336](https://github.com/hubverse-org/hubValidations/pull/336)), so the hub relies on custom validation. Allowing shared samples would let hubs like this express the structure they want and use the standard tools.

**What it adds.** An opt-in config setting lists groups of modeling tasks in a round that may share samples. A shared sample carries one ID in the rows of every modeling task it covers. Its ID drops the `<mt>` segment and hashes the group instead, so the shape of an ID declares whether a sample is shared, and validation can tell intended sharing from IDs reused by mistake. The setting, its conditions and the IDs are set out in the [design document](design.md#samples-shared-across-modeling-tasks).

#### Hash settings in the config

The first release takes the hash length as a function argument, with a default. Longer term, it becomes a hub-level setting in the schema, so that every team in a hub generates IDs the same way. It is an optional `sample_id_hash` object at the top level of `tasks.json`, next to `output_type_id_datatype`, and applies to every modeling task with character sample IDs:

```json
{
  "schema_version": "...",
  "output_type_id_datatype": "auto",
  "sample_id_hash": { "length": 12 },
  "rounds": [...]
}
```

The length is 12 characters by default, which keeps the chance of any ID collision small even in a hub with a million compound units. It should not change once a hub has data, because regenerating IDs with a different length changes them.

It is either one or the other: a hub sets `sample_id_hash`, and the expected length and pattern of sample IDs follow from it, or it sets `max_length` and `pattern` per modeling task, as hubs with free-form character IDs do now. Config validation rejects both. The rules, the collision odds and the `max_length` to use before a hub sets `sample_id_hash` are in the [design document](design.md#hash-settings).

#### Integer-ID hubs

A hash cannot be stored in an integer ID, so hubs with integer sample IDs keep them in submissions. After the data is read, its IDs can be recast to character IDs in the new format, in `collect_hub()` or before ensembling or scoring. Where the IDs must stay integers, reindexing (proposed in hubUtils #328) gives consecutive integers that are unique within the `model_out_tbl`.

#### Functions

| Function | Package | Does |
|---|---|---|
| `hub_hash()` | hubUtils | Hashes named components to fixed-length hex strings. Knows nothing about hubs. |
| `create_hub_spl_ids()` | hubUtils, re-exported by hubValidations | Replaces the sample IDs in a submission file or `model_out_tbl` with IDs in the new format ([#409](https://github.com/hubverse-org/hubValidations/issues/409)). Optionally prints the `tasks.json` settings that validate them. |
| `submission_tmpl_samples()` | hubValidations | Generates sample rows from the declared structure in the config, with plain indexes or hashed IDs. |

`create_hub_spl_ids()` lives in hubUtils so that hubData, the subclasses and the downstream packages can all call it. Prototypes and arguments are in the [design document](design.md#functions).

#### Alternatives considered

- **Reindexing only** (hubUtils #328): reindexing is a valid conversion whenever the model ID, round ID and compound task ID set are known, either from the attributes `collect_hub()` is planned to stamp on collected data ([`model_out_tbl` subclasses poster](../model-out-subclasses/model-out-subclasses.md), hubUtils #310) or supplied by hand. Given a set, it also splits coarser samples to it, which ensembling needs (hubEnsembles #205). A reindexed ID depends on what else is in the table, so it does not replace a hub-wide format.
- **Detected rather than declared sets**: rejected. One declared set per modeling task, with coarser sets by explicit override.
- **Indexing compound units by their position in the config**: rejected. Positions shift when values are added to or missing from the config, which changes IDs for the same samples, as in [example-complex-forecast-hub#42](https://github.com/hubverse-org/example-complex-forecast-hub/issues/42).

### Visualize the solution

```
hubUtils
  ├── hub_hash()                        named components -> fixed-length hex
  ├── get_spl_hash_settings()           the hub's hash setting, from the config or the stamp
  └── create_hub_spl_ids()              submission file or model_out_tbl -> new IDs
        uses hub_hash() once per (model, round, compound unit)

hubValidations
  ├── create_hub_spl_ids()              re-exported from hubUtils
  └── submission_tmpl_samples()         config -> sample rows, plain or hashed IDs

schemas / hubAdmin
  ├── sample_id_hash at the top level of tasks.json
  ├── validate_config(): sample_id_hash, or max_length and pattern, not both
  └── default max_length 25 until a hub sets sample_id_hash

consumers
  ├── hubData: opt-in recasting in collect_hub(), using the set stamped by hubUtils #310
  ├── hubEnsembles, hubEvals: recast, or set as argument, before ensembling or scoring
  └── hubUtils subclasses: possibly recast in as_model_out_sample()
```

### Scale and scope

- **Team:** Anna Krystalli (owner); others TBD.
- **Repos touched:** hubUtils, hubValidations, schemas, hubAdmin, hubTemplate, hubDocs; consumers in hubData, hubEnsembles and hubEvals.
- **Phasing:** (1) `hub_hash()` and `create_hub_spl_ids()` in hubUtils, with the hash length as an argument with a default; (2) recasting in `collect_hub()` and the downstream packages, and the template function; (3) the hub-level hash setting in the schema, read by the functions as defaults.
- **Dependencies:** the schema change for the hub-level hash setting; hubValidations #403 and #404 for validating coarser and mixed samples; hubUtils #310 for the set, and the hash setting, stamped on collected data; task ID subsets in grid expansion and modeling task matching (hubValidations [#422](https://github.com/hubverse-org/hubValidations/issues/422)).
- **Related work, all open unless noted:**
  - hubValidations [#403](https://github.com/hubverse-org/hubValidations/pull/403) (PR): validate samples per compound unit and accept coarser samples.
  - hubValidations [#404](https://github.com/hubverse-org/hubValidations/pull/404) (PR): allow samples of different coarseness within one modeling task ([#400](https://github.com/hubverse-org/hubValidations/issues/400)).
  - hubValidations [#410](https://github.com/hubverse-org/hubValidations/pull/410) (PR): validate sample IDs against the configured `type` and `max_length`.
  - hubValidations [#413](https://github.com/hubverse-org/hubValidations/pull/413) (PR): build the sample index once for all sample checks.
  - schemas [#138](https://github.com/hubverse-org/schemas/issues/138): restrict the characters allowed in character sample IDs, with an optional `pattern`.
  - schemas [#139](https://github.com/hubverse-org/schemas/issues/139): an `allow_overlapping_samples` config property.
  - hubUtils [#310](https://github.com/hubverse-org/hubUtils/issues/310): config-derived values per modeling task, stamped on collected data.
  - hubUtils [#328](https://github.com/hubverse-org/hubUtils/issues/328): reindex sample IDs in a `model_out_tbl`.
  - hubAdmin [#168](https://github.com/hubverse-org/hubAdmin/pull/168) (merged): modeling tasks in a round must be distinguishable.
