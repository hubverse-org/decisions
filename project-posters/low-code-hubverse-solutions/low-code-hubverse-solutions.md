# Project Poster: Low code hubverse solutions

- Date: 2026-09-25

- Owner: Li

- Team: Anna, Li, ... (?)

- Status: Draft

## ❓ Problem space

We are interested in knowing the specific needs of our users, and we identified private hubs as a subset to prioritize. A distinct group within these are hubs where the admins are also the modellers: national public health teams who want to run established models on their own surveillance data and get hub-formatted output, without forecasting or coding expertise. The [MicroHub project](https://github.com/sjfox/microhub) (NAU, Spencer Fox) serves exactly this group. It started with the Paraguay SARI hub and has since been rolled out through PAHO and CDC workshops in Chile and Argentina, both of which plan national forecasting hubs. MicroHub produces output in the hubverse column layout, but the hub side is still manual: users create a repo from `hubTemplate`, clone it with GitHub Desktop, drop a single CSV containing all models into `model-output/microhub/`, and are told to leave `hub-config` and `model-metadata` untouched. The MicroHub team would like tighter integration so their users get validation, dashboards and the rest of the ecosystem.

### What are we doing?

Create R functions that MicroHub can call under the hood, and that any hub admin can also use directly, to stand up and write to a hub without editing files by hand. Ordered by priority:
- **`create_tasks_json()`**: builds a valid `tasks.json` from plain inputs, through the `hubAdmin` `create_*()` functions and `write_config()`. Initial scope: one round using `round_id_from_variable`, multiple modeling tasks, standard task IDs, output types `mean`, `median` and `quantile`, and target metadata with `target_id` and `target_keys` filled from the target values and the rest (name, units, type) supplied by the caller. MicroHub passes the values it already holds (forecast date, horizons, target groups, quantile levels).
- **`create_hub()`**: creates the hub file structure locally, including `admin.json`, a `tasks.json` from `create_tasks_json()`, and model metadata for the fixed MicroHub model set, so the result validates out of the box.
- **`save_model_out_tbl()`**: saves a `model_out_tbl` with the correct formatting, splitting outputs containing multiple models into individual files within directories named by model ID, and validating the result against the hub config.
- Later: flows to add a round to, or otherwise edit, an existing config, building on the `hubAdmin` config class methods (hubverse-org/hubAdmin#158).

`create_tasks_json()` takes arguments and does not prompt; interactive input is a widget's job. It composes lower-level builders, so that a widget can later sit on the same pieces:
- one entry point per task ID, called by name (e.g. `create_task_id("horizon", range = 0:3)`): the tasks schema gives the task ID's type, which selects how its values are described (a range, a weekly date sequence, a list of codes). `required` and `optional` gain defaults filled from those values, so existing calls that pass them are unchanged. Custom task IDs supply their type and reuse the same builders. This complements `hubAdmin`'s existing per-output-type constructors, which can gain presets such as standard quantile level sets
- types, constraints and descriptions come from the tasks schema, which the `create_*()` functions already validate against and a widget would read for its inputs and help text
- the inputs can be passed as a single list, as MicroHub would from its app state
- input checks report problems per argument, so a widget can show each next to its input
- building the config is separate from writing it, so a caller can validate and preview first

A config builder widget over these functions, as a Shiny module in its own repo, would make hub setup genuinely low-code for admins beyond MicroHub. That is a separate piece of work, for its own poster if and when we take it on.


### Why are we doing this?

We would like to support more users, particularly those with less GitHub or coding experience, and make strides to (at least partially) decouple from total GitHub dependency. A function that writes a valid `tasks.json` from plain inputs also simplifies the hardest step in standing up any hub, and is the base a config builder for other hub admins would build on.

### What are we _not_ trying to do?

We will not be developing any code directly for the MicroHub project, just hubverse (R) functions that it can call (and that can also be used directly by hub admins). Passing values from MicroHub's settings is MicroHub's side, so coordination with the NAU team is a dependency. Multi-round configs, `sample`, `pmf` and `cdf` output types, typed validation of custom task IDs, and a config builder widget are out of scope for this poster.

### How do we judge success?

- A MicroHub user can create a hub, including a valid `tasks.json` and model metadata, from the MicroHub UI without editing files or using GitHub.
- Forecasts saved from MicroHub into that hub pass `hubValidations::validate_submission()` with no manual intervention.
- A hub admin outside MicroHub can create a valid `tasks.json` for the initial scope with one call to `create_tasks_json()`.

### What are possible solutions?

We can use [`hubTemplate`](https://github.com/hubverse-org/hubTemplate) as a guide for `create_hub()` and [`trendsEnsemble::save_model_out_tbl()`](https://github.com/reichlab/trendsEnsemble/blob/main/R/save_model_out_tbl.R) as an example for `save_model_out_tbl()`. For `create_tasks_json()`, `hubAdmin`'s `create_task_id()`, `create_output_type_*()`, `create_target_metadata_item()`, `create_round()`, `create_config()` and `write_config()` do the construction and schema validation, so the function only maps its inputs onto them, with defaults from the [tasks schema](https://github.com/hubverse-org/schemas). `hubValidations` provides validation on save. The [Hubverse Support Toolkit](https://dma-prime.github.io/Hubverse-Support-Tools/) from DMA-PRIME (Clemson) already generates model metadata files from a hub's `model-metadata-schema.json` with a browser form; worth coordinating with them, and a useful precedent for a later config builder.

## ✅ Validation

This section is for listing assumptions that should be validated before
kicking off a project. The answers to the prompts below may change over
time, as the team gets more information.

### What do we already know?

- MicroHub uses R under the hood, so we only need to code R versions of the functions
- MicroHub currently saves all the forecasts in a single model output file, which means a `save_model_out_tbl()` function would be useful
- MicroHub does not yet use the hubverse modeling hub file structure, only the data standards, which `save_model_out_tbl()` would also need
- MicroHub output uses a `model` column rather than `model_id`, and its `reference_date` and `target_group` columns are not standard task IDs in schema v6.0.0 (standard: `origin_date`, `forecast_date`, `scenario_id`, `location`, `target`, `target_date`, `target_end_date`, `horizon`, `age_group`)
- MicroHub validates user-uploaded outside forecasts with its own ad hoc checks, which `hubValidations` could replace once a hub config exists
- We have some existing templates/code that can help us write some of the desired functions (see "What are possible solutions?" section)

### What do we need to answer?

- What package out of `hubUtils`, `hubData`, or `hubAdmin` is the best place for these functions to live?
- Do we want to write more than just the `create_hub()` and `save_model_out_tbl()` functions in this initial push?
- How important is it that hubs using MicroHub have the typical config and metadata files, given that they generally don't have outside modeling team submitting forecasts?

## 👍 Ready to make it

If, after defining the problem space and validating assumptions, the team
decides to move forward with the project, this section is to document
a proposed solution.

Keep the answers to the prompts below brief--this document isn't
inteded to be a detailed project plan.

### Proposed solution

`create_tasks_json()`, `create_hub()` and `save_model_out_tbl()` in the hubverse R packages, called by MicroHub and usable standalone, so that a MicroHub run writes validated output into a hubverse hub without the user touching files or GitHub.

### Visualize the solution

Optional. If there is a mockup, data dictionary, or other artifact that clarifies
the project's implementation, you can put it here (or link to it).

### Scale and scope

A place to define items like:

- Required team size
- Required delivery date (if applicable)