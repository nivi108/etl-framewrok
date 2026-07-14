# Skill: Create Workflow Definition

**Version:** 1.0
**Last Modified:** 2026-07-13
**Invoked By:** `workflow_agent.md`, `orchestrator_agent.md`
**Category:** Skills

## Purpose
Step-by-step, mechanically repeatable recipe for generating one Databricks workflow/job definition from `Workflow_Config` entries sharing the same `workflow_name`.

## Trigger
Called once per distinct `workflow_name` in `Workflow_Config`. Receives all config rows for that workflow.

## Pre-Conditions
- All notebooks referenced by `Workflow_Config.notebook_name` have already been generated (by raw/silver/gold agents).
- `Workflow_Config` entries have valid `dependency_group`, `execution_order`, and `parallel_group` values.

## Step-by-Step Generation

### Step 1: Resolve Names
```
workflow_name = workflow_config_rows[0].workflow_name  # all rows share the same workflow_name
output_path = f"src/workflows/{workflow_name}.json"
```

### Step 2: Group and Order Tasks
```
groups = GROUP_BY(workflow_config_rows, dependency_group)
for each group:
    subgroups = GROUP_BY(group, execution_order)
    for each subgroup:
        parallel_batches = GROUP_BY(subgroup, parallel_group)
```

### Step 3: Validate Dimension-Before-Fact Ordering
```
for each group:
    dimension_tables = filter(group, fact_or_dimension == "Dimension")
    fact_tables = filter(group, fact_or_dimension == "Fact")
    for each fact in fact_tables:
        for each dim that fact references:
            if dim.execution_order >= fact.execution_order:
                RAISE ConfigError(f"dimension {dim.table_name} must have lower execution_order than fact {fact.table_name}")
```

### Step 4: Build Task Definitions
```
tasks = []
for each table in workflow_config_rows:
    task = {
        "task_key": f"task_{table.notebook_name}",
        "notebook_task": {
            "notebook_path": resolve_notebook_path(table.notebook_name),
            "base_parameters": {
                "environment": "{{environment}}",  # parameterized, not hardcoded
                "table_name": table.table_name,
                "pipeline_id": "{{pipeline_id}}"
            }
        },
        "timeout_seconds": table.timeout_minutes * 60,
        "max_retries": table.retry_count,
        "depends_on": resolve_dependencies(table, workflow_config_rows)
    }
    tasks.append(task)
```

### Step 5: Build Dependency Graph
```
for each task:
    predecessors = find_tasks_with(
        same dependency_group AND lower execution_order AND
        (different parallel_group OR lower parallel_group)
    )
    task.depends_on = [pred.task_key for pred in predecessors]
    # Tasks with same execution_order AND same parallel_group have no depends_on between them (parallel)
```

### Step 6: Write Workflow JSON
```
workflow_definition = {
    "name": workflow_name,
    "max_concurrent_runs": 1,
    "schedule": {
        "quartz_cron_expression": workflow_config_rows[0].schedule,
        "timezone_id": "UTC"
    },
    "tasks": tasks,
    "job_parameters": [
        {"name": "environment", "default": "prod"},
        {"name": "pipeline_id", "default": "auto_generated"}
    ]
}
WRITE output_path as JSON
```

## Post-Conditions
- One `.json` file exists at `output_path`.
- `max_concurrent_runs = 1` prevents overlapping runs.
- Dimension tasks precede fact tasks in dependency graph.

## Acceptance Criteria
- [ ] Dimension-before-fact ordering validated before output, not just assumed.
- [ ] Every task receives `environment` parameter, never hardcoded.
- [ ] Parallel tasks (same `execution_order` + same `parallel_group`) have no mutual dependency.
- [ ] `max_concurrent_runs = 1` is set.
- [ ] Retry count and timeout applied per task from config.