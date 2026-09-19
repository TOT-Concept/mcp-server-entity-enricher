<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Model benchmarks

Compare models on enrichment, sample generation or schema generation, using the correct reference and score interpretation.

## Define the test

Creating, editing, deleting or running benchmarks requires owner and a plan with Model Benchmarks. `list_benchmark_scenarios` discovers scenarios; `get_benchmark_scenario` reads one. Create with a mandatory `scoring_judge_model_key` and choose the task:

| `scenario_type` | Required task input | Reference before running |
|---|---|---|
| `enrichment` | `schema_id` and fixed `entity_data` — the entity to enrich, the same JSON `enrich_entity` takes (read `get_schema.input_contract`); refused when it carries no value | Verified expected entity output. |
| `sample_generation` | `sample_request` | None: rubric-scored by the judge. |
| `schema_generation` | `entity_samples` (1..20 samples of one entity type) | Verified generated schema document, drafted from the same samples. |

`description` is a note shown in the Benchmarks tab that no model reads — an entity written there is never enriched. Use sample naming/language/search options where exposed; enrichment output languages are separate. Schema generation can opt into semantic IDs. `scenario_type` is immutable. On update, `sample_params` and `schema_gen_params` replace the whole parameter object; preserve settings you intend to retain. The judge can be changed but never cleared. Changing the test definition marks previous results stale.

## Verify references

For enrichment, a strong model and source documents can help draft expected output, but model output alone is not a verified reference. Check values against trusted evidence or obtain human sign-off before `set_benchmark_reference(reference_verified=true)`. Schema-generation references must parse as the supported schema document; inspect structure and annotations as well as JSON syntax. `get_benchmark_scenario(include_reference=true)` returns the stored reference and fixed inputs (`entity_data`, or the schema-generation `entity_samples`) for review. Schema-generation scoring reads those samples as evidence — types, nullability and identity key sets the samples prove are settled without the judge. For both reference-scored types, every judge question is answered blind (the two answers as A and B), and each result reports `findings`: where the candidate showed the reference should change (a verdict in its favour, a rule the samples prove, a value the reference lacked that the judge confirmed). At the end of every scoring pass those findings are folded across the scored models and **applied to the reference automatically**; an edit a pass already made is only replaced by stronger evidence (the samples, a `wrong` verdict, or more agreeing models — accumulated across passes), never by one more model's `better` verdict, so incremental passes cannot make the reference drift toward the last model scored. The log is `reference_meta.auto_applied` on `get_benchmark_scenario` (`skipped` entries say why: no mechanical patch, refused by the save gate, or kept the standing automatic edit), and `revert_benchmark_reference_updates` undoes entries and pins their paths. Automatic edits never stale scores; a manual `set_benchmark_reference` does.

Do not call `set_benchmark_reference` for sample-generation scenarios: the tool rejects them. They are runnable without a gold reference, but still need a scoring judge.

## Run and interpret

`run_benchmark` accepts an explicit `model_keys` list or a `providers` filter. Omitting both runs **all active models with usable provider keys**, which can be expensive. Model executions, repetitions and judging are billed; select a scope consistent with the user's budget. Creating a scenario does not execute its models.

The start returns `job_id` and `total_models`. Poll `get_job_status`; retrieve the scores with `get_benchmark_scenario_results`. Results are upserted per scenario/model: re-running replaces that model's stored result rather than creating a historical series.

An organization's benchmark runs and scoring passes execute one at a time so timings stay comparable. A launch while another is in flight is accepted and queued: the response carries `queue_position`, and `get_job_status` reports status `pending` with that position until the lane admits the job — keep polling rather than relaunching. A launch on a scenario whose run is still queued is folded into that run (`merged: true`; `job_id` names the queued run and `total_models` counts the union). Cancelling a queued job releases its place without spending anything.

Filter by provider/model/status and sort by `overall`, `quality`, `cost`, `speed` or `last_run`. Overall uses the organization's configured per-task quality/speed/cost weights and is null if a component is missing. Missing sort metrics always sort last. Status tags are independent: `success` can coexist with `stale` or `stale_score`. Selecting successful rows does not establish that their scores are current; inspect the returned stale flags before recommending a winner.

Quality scores depend on the reference, rubric and judge; they are not independent proof of truth. Report those conditions with the actual tested models. The organization's task defaults can use configured benchmark scoring sources, but the MCP catalogue does not expose every model-management or scoring administration operation. Full dashboards and benchmark result import/export remain in the web app.
