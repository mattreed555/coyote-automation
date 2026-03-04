# v0 Build Handoff Prompt

Build the first vertical slice of the `v0` automation runtime described in the local project documents.

Use these documents as the source of truth:

- Vision.md - The overall direction and aim of the project
- DeclarativeAutomationSpecV0.md - The Spec for the first version in that direction
- V0ImplementationDesign.md - 

Your job is **not** to build the full product. Your job is to build the smallest real implementation that proves the architecture.

## Goal

Implement a CLI-based `v0` runtime that:

- loads one Python automation file with YAML front matter
- parses and validates the front matter
- compiles it into a normalized in-memory object
- runs the automation immediately in test mode
- executes the Python worker inside an Apple Container
- exposes only two host-managed capabilities in the first slice:
  - `log`
  - `artifacts`
- records a minimal run record

The schedule must be:

- required
- parsed
- validated

But it must **not** be used for scheduling yet. The CLI should run the automation immediately regardless of the declared schedule.

## Scope Boundary

Build only the first vertical slice.

### Required in this slice

- single-file automation source format: Python file with YAML front matter in comment lines
- front matter parser
- schema validator for the supported `v0` subset
- compiler/normalizer into an internal automation object
- CLI command to run an automation immediately
- Apple Container worker execution
- host/worker communication channel
- host-managed `log` capability
- host-managed `artifacts` capability
- minimal run store / run record
- one sample automation file that demonstrates the flow
- tests for parsing and validation

### Explicitly out of scope for this slice

Do not implement these yet:

- scheduler daemon
- background runs
- UI
- `state`
- `llm`
- `html`
- `memory`
- plugins
- approvals
- multi-step workflows
- `workflow` support from the fuller spec
- direct host command execution
- broad network access

You may leave stubs or TODOs for later capabilities, but do not build them now.

## Architecture Requirements

Follow the architectural intent from the docs:

- the automation file is parsed into front matter + Python body
- the front matter is validated before any execution
- the validated front matter is compiled into an internal object
- the Python worker runs in an Apple Container
- the worker does not receive raw service credentials
- the worker communicates with the host through a narrow runtime interface
- the host owns capability enforcement

Treat the host capability layer as the main trust boundary.

Even in this first slice, do not let the worker bypass the host for logging or artifact writing.

## Suggested Components

Implement the first slice with explicit components, even if lightweight:

- `AutomationLoader`
- `Validator`
- `Compiler`
- `RunManager`
- `ContainerRuntime`
- `CapabilityHost`
- `ArtifactStore`
- `RunStore`
- `CLI`

You do not need to fully implement `ConfigStore` yet if the first slice can run without model providers or secrets, but leave a clean place for it.

## Input Format to Support

Support the `v0` single-file Python format with front matter:

```python
# ---
# version: "0.0"
# kind: automation
# metadata:
#   id: hello-automation
#   name: Hello Automation
# triggers:
#   - type: schedule
#     schedule:
#       mode: interval
#       every_minutes: 60
# host_capabilities:
#   log:
#     enabled: true
#   artifacts:
#     allowed_outputs: [hello_output]
# outputs:
#   artifacts:
#     - id: hello_output
#       type: file
#       path: ./outputs/hello.json
# policy:
#   retry:
#     max_attempts: 1
#   failure_handling:
#     mode: mark_failed
# ---

from automation_runtime import Runtime

def main():
    rt = Runtime.from_env()
    rt.log.info("Running automation")
    rt.artifacts.write("hello_output", {"ok": True})
    return {"ok": True}
```

## Validation Requirements

The validator must fail closed.

At minimum validate:

- file starts with `# ---`
- closing front matter delimiter exists
- front matter parses as YAML
- `version == "0.0"`
- `kind == "automation"`
- exactly one trigger exists
- trigger type is `schedule`
- schedule mode is one of `interval`, `weekly`, `cron`
- only supported front matter fields are present
- if `host_capabilities.artifacts` is declared, referenced artifact ids exist in `outputs.artifacts`
- unsupported fields such as `plugins`, `memory`, `workflow`, `llm_task`, etc. are rejected

Do not silently ignore unsupported fields.

## Runtime Requirements

The first slice runtime should:

- create a run id
- record a run start
- launch the worker in an Apple Container
- provide the worker with a minimal `automation_runtime` helper
- expose `log` and `artifacts` via a host-mediated interface
- capture success/failure
- persist a minimal run record

The worker should not directly write arbitrary files.

It should write only through `rt.artifacts.write(id, value)`, and the host must map `id` to a declared artifact path.

## Apple Container Requirement

Do not simulate the container boundary with a plain subprocess for this task.

Use Apple Containers as the actual worker execution path for the first slice.

If there is setup friction, work through it. That friction is part of the point of the exercise.

If you need to simplify, simplify inside the containerized flow, not by removing the container boundary.

## CLI Requirements

It is acceptable to implement only one command first:

- `run <path-to-automation.py>`

Behavior:

- parse + validate
- print validation errors if invalid
- if valid, run immediately
- print a concise run summary
- print artifact output paths

Optional:

- add `validate <path>` if it is straightforward

## Storage Requirements

Keep storage simple, but real.

At minimum persist:

- run records
- artifact files

If you can do this cleanly:

- use SQLite for run records

If SQLite slows the slice too much:

- a temporary JSON-on-disk run store is acceptable for the first cut, but keep the interface swappable

## Tests

Include tests for:

- front matter extraction
- invalid front matter shape
- schema validation failures
- artifact capability validation
- compiling a valid file into a normalized internal object

Make sure:

- the container launch path is real
- the non-container path is not substituted in as the default implementation]

Try to do a full container test if you can. 

## Deliverables

When finished, provide:

1. the implemented code
2. a short summary of the architecture you built
3. any deviations from the docs
4. known gaps / TODOs
5. exact instructions for how to run the sample automation

## Most Important Constraint

Do not overbuild.

This is a vertical slice to prove:

- the front matter contract
- the container boundary
- the host capability model
- and the basic run record path

It is not an excuse to implement the rest of the roadmap.
