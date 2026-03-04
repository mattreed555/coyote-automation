# Declarative Automation Specification v0

Draft date: March 3, 2026

Status: executable subset for the initial build

Purpose:
- Define the smallest useful subset of the automation format to implement first.
- Optimize for a single-file authoring experience.
- Keep the underlying model aligned with the fuller declarative spec.

This document is intentionally narrower than [DeclarativeAutomationSpec.md](/Users/matthewreed/Projects/Personal/AgentReference/DeclarativeAutomationSpec.md).

## v0 Goal

`v0` exists to prove the runtime model, not the editor model.

The runtime should be able to:

- load one Python automation file
- parse YAML front matter from the top of that file
- validate the declared contract
- execute the script inside a bounded host-managed runtime
- produce a run record

The goal is not feature breadth. The goal is to prove that a useful automation can be authored and run with very low friction.

## Core v0 Decision

`v0` supports exactly one executable unit per automation:

- one Python script file

That file contains:

- YAML front matter at the top
- Python code below

The YAML front matter is the declarative contract.
The Python code is the task logic.

The runtime should compile the front matter into the same internal automation object model the app will later use.

This means the single-file format is an authoring convenience for `v0`, not the long-term canonical storage format.

## Design Principle

For `v0`, prefer:

- one file over multiple files
- one script over a step graph
- explicit declarations over hidden access
- host-managed capabilities over raw credentials
- simple validation over broad expressiveness

`v0` should prove the model with constrained automations such as:

- Organizer
- eBay Searcher
- a simple notes/report summarizer

It is not intended to fully support Deep Researcher or eBay Selling Assistant yet.

## File Format

A `v0` automation is a Python file with YAML front matter encoded as line comments at the top.

Suggested shape:

```python
# ---
# version: "0.0"
# kind: automation
# metadata:
#   id: example-automation
#   name: Example Automation
# triggers:
#   - type: schedule
#     schedule:
#       mode: interval
#       every_minutes: 60
# host_capabilities:
#   log:
#     enabled: true
# ---

from automation_runtime import Runtime

def main():
    rt = Runtime.from_env()
    rt.log.info("Hello from v0")
    return {"ok": True}
```

Parsing rules:

- the file must begin with `# ---`
- front matter ends at the next `# ---`
- each front matter line must begin with `#`
- the host strips the leading comment marker and parses the remaining YAML
- the rest of the file is the Python program

The validator should reject files that do not follow this structure.

## Conceptual Model

Even though `v0` uses a single-file authoring format, the underlying model is still:

`Schedule + Resources + State + Host Capabilities + Outputs + Policy`

There is no multi-step workflow in `v0`.

The single Python script is the only executable step.

Future versions may add plugins as host-loaded capability extensions, but that is not part of `v0`.

## Supported Front Matter Fields

### `version`

Required.

For `v0`:

```yaml
version: "0.0"
```

### `kind`

Required.

For `v0`:

```yaml
kind: automation
```

### `metadata`

Required fields:

- `id`
- `name`

Optional fields:

- `description`
- `tags`

Example:

```yaml
metadata:
  id: summarize-notes
  name: Summarize Notes
  description: Read notes and produce a short summary.
  tags: [notes, summary]
```

### `triggers`

Required.

`v0` supports exactly one trigger, and it must be `schedule`.

Example:

```yaml
triggers:
  - type: schedule
    schedule:
      mode: weekly
      days: [MON]
      at: "08:00"
      timezone: America/Los_Angeles
```

Supported schedule modes:

- `interval`
- `weekly`
- `cron`

Unsupported in `v0`:

- multiple triggers
- non-schedule triggers
- `daily`
- `monthly`
- `once`

### `resources`

Optional, but usually present.

Supported resource categories:

- `files`
- `apis`
- `secrets`
- `models`

Resource declarations are capability declarations. They define what the script may access through the host runtime.

Examples:

```yaml
resources:
  files:
    - id: notes_folder
      path: ~/Notes/Weekly
      access: read_only
```

```yaml
resources:
  apis:
    - id: ebay_search
      provider: ebay
      scopes: [search_listings]
```

```yaml
resources:
  models:
    - id: default_model
      class: local
      capability: standard
```

Unsupported in `v0`:

- `tools`
- provider-specific runtime binding

### `state`

Optional.

`state` is supported in `v0`, but only as operational memory for correctness.

Supported state store kinds:

- `key_value`
- `set`

Example:

```yaml
state:
  stores:
    - id: seen_listing_ids
      kind: set
      scope: automation
      retention: durable
```

Use cases:

- deduplication
- checkpoints
- last processed markers

### `host_capabilities`

Optional, but central to the `v0` model.

This declares which host-managed services the script may use.

Supported host capabilities in `v0`:

- `llm`
- `state`
- `log`
- `html`
- `artifacts`

Example:

```yaml
host_capabilities:
  llm:
    enabled: true
    capability_level: standard
    reasoning_style: analytical
  state:
    read: [seen_listing_ids]
    write: [seen_listing_ids]
  log:
    enabled: true
  html:
    enabled: true
    allowed_sources: [web_search]
  artifacts:
    allowed_outputs: [summary_report]
```

Capability meanings:

- `llm`: host-routed model access without exposing raw provider credentials
- `state`: host-managed operational state reads and writes
- `log`: structured run logging
- `html`: host-mediated remote HTML retrieval and normalized content extraction, allowing sanitization and prompt-injection checks before content reaches the script
- `artifacts`: host-managed output writing to declared artifacts

Strong rule:

- the script should use only declared host capabilities
- host capabilities should be the preferred way to access models, storage, and remote HTML

Future note:

- later versions may allow plugin-provided capabilities under a host-managed `plugins` namespace
- plugins may extend capabilities, but they may not bypass the host

### `plugins`

Reserved for future versions.

The long-term direction is that plugins should be host-loaded capability extensions declared in front matter and then exposed through the runtime, for example under `rt.plugins.*`.

`plugins` is intentionally unsupported in `v0`.

### `inputs`

Optional.

This is the simplest way to pass static parameters into the script without introducing a step graph.

Inputs declared here are made available through `rt.input(...)`.

Example:

```yaml
inputs:
  query: vintage stereo receiver
  max_price: 400
```

### `outputs`

Optional.

Useful as a declaration of expected artifacts.

If the script uses `host_capabilities.artifacts`, the referenced artifacts should be declared here.

Example:

```yaml
outputs:
  artifacts:
    - id: summary_report
      type: file
      path: ./outputs/notes-summary.json
```

### `policy`

Optional, minimal support only.

Supported fields:

- `retry.max_attempts`
- `failure_handling.mode`

Supported values:

- `failure_handling.mode: mark_failed`

Example:

```yaml
policy:
  retry:
    max_attempts: 1
  failure_handling:
    mode: mark_failed
```

`v0` should be fail-fast by default.

Unsupported in `v0`:

- approvals
- budgets
- advanced retry policies

### `observability`

Optional, minimal support only.

Supported fields:

- `log_level`
- `persist_step_outputs`

Example:

```yaml
observability:
  log_level: standard
  persist_step_outputs: true
```

## Explicitly Unsupported in v0

The validator should reject or refuse to execute these features:

- `memory`
- `code`
- `plugins`
- `workflow`
- `fetch`
- `llm_task`
- `write`
- `agent_task`
- `action`
- `notify`
- `approve`
- `transform`
- `for_each`
- `merge`
- `when`
- multiple triggers
- non-schedule triggers
- self-modifying automations
- hidden side effects outside declared host capabilities

This is deliberate. `v0` is meant to prove the core runtime, not the full concept.

## Runtime Contract

The Python program should be executed inside a bounded host-managed runtime.

Recommended runtime behavior:

- the host parses and validates front matter before running the script
- the host exposes declared resources through a narrow runtime API
- the host exposes only declared host capabilities through a narrow runtime API
- the host captures structured script output
- the host captures logs
- the host enforces timeout and basic process limits

Strong rule:

- no ambient filesystem access beyond declared file resources
- no ambient network access beyond declared API resources and host-managed HTML retrieval
- no raw model provider credentials when `llm` is available through the host

## Script Interface

The script should expose a `main()` function.

Suggested shape:

```python
from automation_runtime import Runtime

def main():
    rt = Runtime.from_env()
    return {"ok": True}
```

The runtime is responsible for:

- providing access to declared inputs
- providing access to declared resources
- providing access to declared host capabilities

### Suggested Runtime API

The exact implementation can evolve, but the conceptual API should look like:

- `rt.input(name)`
- `rt.resource(id)`
- `rt.llm.complete(...)`
- `rt.state.get(id, default=...)`
- `rt.state.set(id, value)`
- `rt.log.info(...)`
- `rt.html.fetch(...)`
- `rt.artifacts.write(id, value)`

Examples:

- `rt.resource("notes_folder").read()`
- `rt.resource("ebay_search").call(operation="search_listings", params={...})`
- `rt.artifacts.write("summary_report", result)`

## Input Model

`v0` supports explicit static inputs only.

Optional `inputs` may be declared in front matter and made available through `rt.input(...)`.

Example:

```yaml
inputs:
  query: vintage stereo receiver
  max_price: 400
```

This is useful for small parameterized automations without inventing a step graph.

## Validation Rules

The `v0` library should validate at least the following:

### Structural Validation

- `version` must be `0.0`
- `kind` must be `automation`
- exactly one schedule trigger must exist
- only supported front matter fields may be present
- the file must contain valid front matter and Python code

### Reference Validation

- every resource reference in front matter must be internally valid
- every `host_capabilities.state` reference must point to declared state stores
- every `host_capabilities.artifacts` reference must point to declared output artifacts

### Capability Validation

- the script may only access declared resources through the runtime
- the script may only use declared host capabilities
- `host_capabilities.llm` must route through declared model policy or default model policy

### Simplicity Validation

- reject unsupported fields rather than silently ignoring them
- reject unsupported schedule modes
- reject unsupported capability names

## Minimal Run Model

Even in `v0`, the runtime should produce a basic run record.

Recommended fields:

```yaml
run:
  id: run_123
  automation_id: summarize-notes
  started_at: 2026-03-03T10:00:00Z
  ended_at: 2026-03-03T10:00:04Z
  status: succeeded
  script:
    path: ./automations/summarize_notes.py
    status: succeeded
  artifacts:
    - type: file
      path: ./outputs/notes-summary.json
```df

This does not need to be stored as YAML at runtime; it is just the conceptual shape.

## Recommended First Implementation Order

Build the runtime in this sequence:

1. Front matter parser and schema validator
2. Reference validator and default normalization
3. Python script execution
4. host-managed `log`
5. host-managed `artifacts`
6. host-managed `state`
7. host-managed `llm`
8. host-managed `html`
9. basic run recording

This order lets you test useful workflows early without implementing the broader system.

## First Good Proving-Ground Automations

The best first proving-ground automations are:

1. Organizer
   Simple file reads, one summary, one output artifact.

2. eBay Searcher
   Uses an API resource, host-managed LLM access, and simple dedupe state.

These are enough to validate the runtime model without needing multi-step orchestration.

## Example v0 Automation

```python
# ---
# version: "0.0"
# kind: automation
# metadata:
#   id: summarize-notes
#   name: Summarize Notes
#   description: Read notes and produce a short summary.
# triggers:
#   - type: schedule
#     schedule:
#       mode: weekly
#       days: [MON]
#       at: "08:00"
#       timezone: America/Los_Angeles
# resources:
#   files:
#     - id: notes_folder
#       path: ~/Notes/Weekly
#       access: read_only
#   models:
#     - id: default_model
#       class: local
#       capability: standard
# host_capabilities:
#   llm:
#     enabled: true
#     capability_level: standard
#     reasoning_style: analytical
#   artifacts:
#     allowed_outputs: [summary_report]
#   log:
#     enabled: true
# outputs:
#   artifacts:
#     - id: summary_report
#       type: file
#       path: ./outputs/notes-summary.json
# policy:
#   retry:
#     max_attempts: 1
#   failure_handling:
#     mode: mark_failed
# observability:
#   log_level: standard
#   persist_step_outputs: true
# ---

from automation_runtime import Runtime

def main():
    rt = Runtime.from_env()

    notes = rt.resource("notes_folder").read()

    result = rt.llm.complete(
        task_type="summarization",
        objective="Summarize the main themes and action items from these notes.",
        data={"notes": notes},
        output_schema={
            "summary": "string",
            "action_items": ["string"],
        },
    )

    rt.artifacts.write("summary_report", result)
    rt.log.info("Summary written")

    return {"ok": True}
```

## Bottom Line

`v0` should be a strict, boring, single-file executable subset.

If it can reliably run one or two useful constrained automations from a Python file with YAML front matter, it has done its job.

That is enough to validate the abstraction before you invest in the editor, the app shell, or the broader agentic features.
