# Recipe: benchmark models on your own data

Goal: compare LLM models on a **saved, reusable enrichment test** — same schema, same entity,
same settings — with every result auto-scored against a gold reference.

Requires the **owner** role — your own role when connected via OAuth, or the key's role when
using an API key — and a plan that includes Model Benchmarks (otherwise tools return
`benchmarks_not_in_plan`).

## 1. Create the scenario

> Create a benchmark scenario "French pharma quarterly" using the pharma company schema,
> entity "Sanofi", strategy auto, English + French, judged by Claude Sonnet.

`create_benchmark_scenario` — a **scoring judge is mandatory** at creation. The scenario
freezes the whole test definition (schema, entity, strategy, languages, toggles, attachments,
scoring config) so runs stay comparable over time.

## 2. Set the gold reference

> Enrich Sanofi against that schema with my best model, and use the result as the benchmark
> reference.

Two-step: an ordinary `enrich_entity`, then `set_benchmark_reference` with the (reviewed!)
output. The reference must be marked **verified** — a scenario without a verified reference
refuses to run. Review the reference values before verifying; every future score is measured
against them.

## 3. Run it

> Run the benchmark on GPT-5, Gemini and Mistral Large.

`run_benchmark` accepts an explicit model list, a provider filter, or nothing (= all active
models with a usable key). It's async — poll with `get_job_status`. Each model's enrichment is
scored at the tail of the run: completeness, correctness, hallucination rate, and a weighted
overall score. Runs deduct credits like normal enrichments; repetitions multiply the cost.

Results are **idempotent per (scenario, model)** — re-running a model overwrites its previous
result, and results from an older scenario config are flagged stale.

## 4. Read the scoreboard

> Which model wins on quality per dollar?

`get_benchmark_scenario` returns per-model quality / cost / speed. Add
`include_reference=true` to also see the gold reference for spot-checking.

## 5. Iterate

`update_benchmark_scenario` changes the test definition (existing results get flagged stale);
`delete_benchmark_scenario` removes the scenario and its results.
