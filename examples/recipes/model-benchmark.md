<!-- Generated from the packaged MCP guide; do not edit this copy. -->

# Model benchmarks

Compare models on enrichment, sample generation or schema generation, using the correct reference and score interpretation.

## Define the test

Creating, editing, deleting or running benchmarks requires owner and a plan with Model Benchmarks. `list_benchmark_scenarios` discovers scenarios; `get_benchmark_scenario` reads one. Create with a mandatory `scoring_judge_model_key` and choose the task:

| `scenario_type` | Required task input | Reference before running |
|---|---|---|
| `enrichment` | `schema_id` and fixed `entity_data` | Verified expected entity output. |
| `sample_generation` | `sample_request` | None: rubric-scored by the judge. |
| `schema_generation` | `sample_json` | Verified generated schema document. |

Use sample naming/language/search options where exposed; enrichment output languages are separate. Schema generation can opt into semantic IDs. `scenario_type` is immutable. On update, `sample_params` and `schema_gen_params` replace the whole parameter object; preserve settings you intend to retain. The judge can be changed but never cleared. Changing the test definition marks previous results stale.

## Verify references

For enrichment, a strong model and source documents can help draft expected output, but model output alone is not a verified reference. Check values against trusted evidence or obtain human sign-off before `set_benchmark_reference(reference_verified=true)`. Schema-generation references must parse as the supported schema document; inspect structure and annotations as well as JSON syntax. `get_benchmark_scenario(include_reference=true)` returns the stored reference and fixed entity input for review.

Do not call `set_benchmark_reference` for sample-generation scenarios: the tool rejects them. They are runnable without a gold reference, but still need a scoring judge.

## Run and interpret

`run_benchmark` accepts an explicit `model_keys` list or a `providers` filter. Omitting both runs **all active models with usable provider keys**, which can be expensive. Model executions, repetitions and judging are billed; select a scope consistent with the user's budget. Creating a scenario does not execute its models.

The start returns `job_id` and `total_models`. Poll `get_job_status`; retrieve the scores with `get_benchmark_scenario_results`. Results are upserted per scenario/model: re-running replaces that model's stored result rather than creating a historical series.

Filter by provider/model/status and sort by `overall`, `quality`, `cost`, `speed` or `last_run`. Overall uses the organization's configured per-task quality/speed/cost weights and is null if a component is missing. Missing sort metrics always sort last. Status tags are independent: `success` can coexist with `stale` or `stale_score`. Selecting successful rows does not establish that their scores are current; inspect the returned stale flags before recommending a winner.

Quality scores depend on the reference, rubric and judge; they are not independent proof of truth. Report those conditions with the actual tested models. The organization's task defaults can use configured benchmark scoring sources, but the MCP catalogue does not expose every model-management or scoring administration operation. Full dashboards and benchmark result import/export remain in the web app.
