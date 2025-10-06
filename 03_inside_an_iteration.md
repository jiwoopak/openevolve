# Inside an Iteration

The real drama unfolds in `openevolve/iteration.py`. Each iteration pulls from evolutionary memory, negotiates with the LLM, and tests the result under the evaluator's gaze:

![Iteration visualizer](../openevolve-visualizer.png)

```python
parent, inspirations = database.sample(num_inspirations=config.prompt.num_top_programs)
prompt = prompt_sampler.build_prompt(
    current_program=parent.code,
    parent_program=parent.code,
    program_metrics=parent.metrics,
    previous_programs=[p.to_dict() for p in island_previous_programs],
    top_programs=[p.to_dict() for p in island_top_programs],
    inspirations=[p.to_dict() for p in inspirations],
    language=config.language,
    evolution_round=iteration,
    diff_based_evolution=config.diff_based_evolution,
    program_artifacts=parent_artifacts if parent_artifacts else None,
    feature_dimensions=database.config.feature_dimensions,
)

llm_response = await llm_ensemble.generate_with_context(
    system_message=prompt["system"],
    messages=[{"role": "user", "content": prompt["user"]}],
)

if config.diff_based_evolution:
    diff_blocks = extract_diffs(llm_response)
    child_code = apply_diff(parent.code, llm_response)
else:
    child_code = parse_full_rewrite(llm_response, config.language)

child_metrics = await evaluator.evaluate_program(child_code, child_id)
```

Every line shows careful guardrails: prompt construction respects islands and artifacts, the LLM output is parsed differently for diffs versus rewrites, and the evaluator runs asynchronously with retries. Metadata about prompts and changes are stored alongside the program so the database can replay the reasoning or trace improvements.

When scalability becomes a concern, `process_parallel.py` mirrors this logic inside worker processes. It serializes minimal snapshots of the database, lazily initializes LLM clients, and keeps islands isolated so parallel evolution does not corrupt shared state. The result is a system that can explore many mutations at once without losing the narrative thread of how each idea was born.
