# Declarative Automation Specification v0.1

Draft date: March 5, 2026

Status: executable subset for the next build phase

Purpose:
- Extend `v0` without losing its script-first simplicity.
- Add host-managed runtime profiles.
- Add simple function plugins for advanced domain behavior (for example Deep Research).

This document is intentionally narrower than [DeclarativeAutomationSpec.md](/Users/matthewreed/Projects/Personal/AgentReference/DeclarativeAutomationSpec.md).

## v0.1 Goal

`v0.1` keeps the `v0` model:

- one automation file
- one Python script entrypoint
- one declarative front matter contract

And adds two capabilities:

- runtime profile selection
- plugin function calls through host mediation

## Core Design Rules

- The YAML front matter is the contract.
- The Python body is the logic.
- Plugins may extend capabilities, but they may not bypass the host.
- Dependencies come from host-managed runtime profiles, not runtime `pip install`.
- `v0.1` remains CLI test mode: schedule is parsed and validated but execution runs immediately.

## File Format

A `v0.1` automation is one Python file with YAML front matter encoded as line comments at the top.

```python
# ---
# version: "0.1"
# kind: automation
# metadata:
#   id: example
#   name: Example
# triggers:
#   - type: schedule
#     schedule:
#       mode: interval
#       every_minutes: 60
# runtime_profile: python-core@1.0.0
# host_capabilities:
#   log:
#     enabled: true
# ---

from automation_runtime import Runtime

def main():
    rt = Runtime.from_env()
    rt.log.info("hello")
    return {"ok": True}
```

Front matter parsing rules:

- file must begin with `# ---`
- front matter ends at the next `# ---`
- each front matter line must begin with `#`
- host strips comment prefix and parses YAML
- remaining file is Python source

## Supported Front Matter Fields

### `version`

Required.

For `v0.1`:

```yaml
version: "0.1"
```

### `kind`

Required.

For `v0.1`:

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

### `triggers`

Required.

`v0.1` supports exactly one trigger and it must be `schedule`.

Supported schedule modes:

- `interval`
- `weekly`
- `cron`

### `resources`

Optional.

Supported resource categories:

- `files`
- `apis`
- `secrets`
- `models`

Resources define allowed access through host APIs. They do not imply automatic execution.

### `state`

Optional.

Supported store kinds:

- `key_value`
- `set`

`state` is operational memory for correctness (cursors, dedupe ids, checkpoints).

### `runtime_profile`

Required.

Declares the runtime image/profile used for the main script.

Example:

```yaml
runtime_profile: python-core@1.0.0
```

### `runtime_profiles`

Optional override of allowed profiles for this automation.

If omitted, host global policy applies.

Example:

```yaml
runtime_profiles:
  allowed:
    - python-core@1.0.0
    - python-research@1.0.0
```

### `plugins`

Optional.

Declares host-loaded plugin providers and the functions they expose.

Simple plugin model in `v0.1`:

- each plugin exposes named functions
- each function has an input/output contract
- script calls plugin functions through host runtime API

Example:

```yaml
plugins:
  providers:
    - id: deep_research
      runtime_profile: python-research@1.0.0
      version: "1"
      functions:
        - id: run
          input_schema:
            topic: string
            context_sources: [string]
          output_schema:
            report_markdown: string
            citations: [string]
          timeout_seconds: 600
      config:
        max_subtasks: 6
```

### `host_capabilities`

Required.

Declares host-managed capabilities available to the script.

Supported built-ins in `v0.1`:

- `log`
- `artifacts`
- `state`
- `llm`
- `html`
- `plugins`

Example:

```yaml
host_capabilities:
  log:
    enabled: true
  artifacts:
    allowed_outputs: [research_report]
  plugins:
    deep_research:
      enabled: true
      functions: [run]
```

`plugins` permission is two-layered:

- plugin and function are declared in `plugins.providers[*].functions`
- script is granted usage in `host_capabilities.plugins`

### `inputs`

Optional static inputs passed to script.

### `outputs`

Optional but recommended.

If `host_capabilities.artifacts` is enabled, expected artifacts should be declared here.

### `policy`

Optional.

Suggested `v0.1` fields:

- `retry.max_attempts`
- `failure_handling.mode`

### `observability`

Optional.

Suggested fields:

- `log_level`
- `persist_step_outputs` (script result payload)

## Runtime Profiles

Runtime profiles are host-managed and immutable.

Recommended baseline profiles:

1. `python-core@1.0.0`
- Python stdlib + automation runtime SDK only
- default profile for most automations

2. `python-research@1.0.0`
- `python-core` plus document/web parsing tooling
- for deep research and synthesis pipelines

3. `python-data@1.0.0`
- `python-core` plus `numpy`, `pandas`, `pyarrow`, `openpyxl`
- for spreadsheet and tabular automations

4. `python-ml-classic@1.0.0`
- `python-data` plus `scikit-learn`, `scipy`, `joblib`
- for classical ML scoring/forecasting

Rules:

- no runtime `pip install`
- profile id must resolve to a host-approved immutable image
- profile versions should be pinned and explicit

## Plugin Function Contract

Plugin functions are simple operation calls.

The canonical script API for `v0.1`:

```python
result = rt.plugins.call("deep_research", "run", {
    "topic": "v0.1 plugin architecture",
    "context_sources": ["notes_vault"]
})
```

Contract expectations:

- `provider_id` and `function_id` must be declared
- input payload should validate against function `input_schema`
- output payload should validate against function `output_schema`
- host enforces `timeout_seconds`
- host logs invocation metadata and result status

`v0.1` keeps plugin calls synchronous for simplicity.

## Security Model

Strong rules:

- scripts and plugins never receive broad ambient credentials by default
- host owns secret resolution, provider access, and policy checks
- plugins may extend capabilities, but may not bypass host
- html retrieval should remain host-mediated so sanitization and prompt-injection checks can run before script/plugin consumption

Dependency and supply-chain rules:

- plugin dependencies are determined by `runtime_profile`
- plugin manifests should be versioned
- host should record plugin id/version/profile in run records

## Validation Rules

Schema rules:

- required fields must exist (`version`, `kind`, `metadata`, `triggers`, `runtime_profile`, `host_capabilities`)
- `version` must be `"0.1"`
- trigger list length must be 1 and type must be `schedule`

Profile and plugin rules:

- `runtime_profile` must be in host-approved profile allowlist
- each plugin provider id must be unique
- each plugin function id must be unique within provider
- each granted function in `host_capabilities.plugins` must be declared by that provider

Capability rules:

- script may only access declared host capabilities
- state read/write references must map to declared `state.stores`
- artifacts writes must map to declared `outputs.artifacts`

## Runtime API (v0.1 minimum)

Expected script runtime surface:

- `rt.input(name)`
- `rt.resource(id)`
- `rt.log.info/warn/error(...)`
- `rt.artifacts.write(id, payload)`
- `rt.state.get/set/add/remove(...)` (if enabled)
- `rt.llm.complete(...)` (if enabled)
- `rt.html.fetch(...)` (if enabled)
- `rt.plugins.call(provider_id, function_id, payload)` (if enabled)

## Out Of Scope For v0.1

- multi-step workflow graph
- asynchronous plugin jobs
- plugin hooks and lifecycle callbacks
- plugin-driven schedule mutation
- runtime dependency installation from automation files
- non-schedule triggers

## Example: Deep Research Automation (`v0.1`)

```python
# ---
# version: "0.1"
# kind: automation
# metadata:
#   id: weekly-research-brief
#   name: Weekly Research Brief
#   description: Run a bounded deep research function and save a brief.
# triggers:
#   - type: schedule
#     schedule:
#       mode: weekly
#       days: [MON]
#       at: "09:00"
#       timezone: America/Los_Angeles
# runtime_profile: python-core@1.0.0
# resources:
#   files:
#     - id: notes_vault
#       path: ~/Notes/Research
#       access: read_only
# plugins:
#   providers:
#     - id: deep_research
#       runtime_profile: python-research@1.0.0
#       version: "1"
#       functions:
#         - id: run
#           input_schema:
#             topic: string
#             context_sources: [string]
#           output_schema:
#             report_markdown: string
#             citations: [string]
#           timeout_seconds: 600
#       config:
#         max_subtasks: 6
# host_capabilities:
#   log:
#     enabled: true
#   artifacts:
#     allowed_outputs: [research_report]
#   plugins:
#     deep_research:
#       enabled: true
#       functions: [run]
# outputs:
#   artifacts:
#     - id: research_report
#       type: file
#       path: ./outputs/weekly-research-brief.md
# policy:
#   retry:
#     max_attempts: 1
#   failure_handling:
#     mode: mark_failed
# observability:
#   log_level: standard
# ---

from automation_runtime import Runtime

def main():
    rt = Runtime.from_env()

    result = rt.plugins.call("deep_research", "run", {
        "topic": "AI automation tools for local-first workflows",
        "context_sources": ["notes_vault"]
    })

    rt.artifacts.write("research_report", result["report_markdown"])
    rt.log.info("Research brief written", citations=len(result.get("citations", [])))

    return {"ok": True}
```

