# v0 Implementation Design

Draft date: March 4, 2026

Purpose:
- Define the concrete design for building `v0`.
- Translate [DeclarativeAutomationSpecV0.md](/Users/matthewreed/Projects/Personal/AgentReference/DeclarativeAutomationSpecV0.md) into implementable subsystems.
- Give a developer a clear build target for the first working runtime.

This is a developer-facing implementation memo, not a product vision document.

## v0 Outcome

`v0` should be a CLI-driven test mode that:

- loads one Python automation file with YAML front matter
- parses and validates the front matter
- ignores the schedule for execution timing, but still validates and normalizes it
- runs the script immediately on demand
- runs the script inside an Apple Container from the beginning
- exposes only host-managed capabilities to the script
- records the run, outputs, and state changes

This is enough to prove the runtime model before building a UI or a scheduler daemon.

## Fixed Decisions

These are intentionally fixed for `v0`:

### 1. Single-file automation source

The input unit is one Python file with YAML front matter at the top.

There is no separate YAML file in `v0`.

### 2. One executable script per automation

`v0` has no workflow graph.

There is one script, one `main()` entrypoint, and one bounded runtime session.

### 3. Apple Containers from the start

The worker executes inside an Apple Container in `v0`.

This is a deliberate design choice:

- container friction is part of the real product
- the runtime boundary is core to the architecture
- container assumptions should be tested early, not retrofitted later

### 4. CLI test mode only

`v0` is not a scheduler.

The CLI runs the automation immediately, regardless of what the parsed schedule says.

The schedule is still:

- required in the file
- parsed
- validated
- normalized

But it is not yet used to decide when to run.

### 5. Host-managed capabilities are the trust boundary

Scripts do not receive raw service credentials or ambient access by default.

The script interacts with:

- declared resources
- declared host capabilities

All sensitive or external operations go through the host.

## Non-Goals

Explicitly out of scope for `v0`:

- background scheduling
- a persistent daemon
- a GUI
- multi-step workflows
- plugin loading
- memory stores
- approvals and human-in-the-loop flows
- arbitrary host command execution
- broad integration catalogs

## High-Level Architecture

`v0` should be built as a small set of cooperating components:

1. `CLI`
2. `AutomationLoader`
3. `Validator`
4. `Compiler`
5. `RunManager`
6. `ContainerRuntime`
7. `CapabilityHost`
8. `StateStore`
9. `ArtifactStore`
10. `RunStore`
11. `ConfigStore`

The runtime should be designed so these parts are explicit, even if some are lightweight at first.

## Component Responsibilities

### 1. `CLI`

User-facing command for `v0`.

Responsibilities:

- accept a path to a Python automation file
- run validation
- optionally print normalized metadata
- launch a run immediately
- print a concise run summary

Suggested first commands:

- `validate <file>`
- `run <file>`

Optional later:

- `inspect <file>`
- `show-run <run-id>`

### 2. `AutomationLoader`

Loads and splits the source file into:

- front matter
- Python source body

Responsibilities:

- verify the file begins with `# ---`
- locate the closing `# ---`
- strip comment prefixes
- return raw YAML text plus Python code

This component should do no semantic validation beyond file-shape parsing.

### 3. `Validator`

Validates the front matter against the `v0` contract.

Validation layers:

- file structure validation
- YAML syntax validation
- schema validation
- reference validation
- capability validation
- safety validation

Key checks:

- `version == "0.0"`
- `kind == automation`
- exactly one schedule trigger exists
- only supported front matter fields are present
- only supported schedule modes are used
- referenced state stores exist
- referenced output artifacts exist
- declared host capabilities are valid
- `plugins`, `memory`, `workflow`, and other unsupported fields are rejected

The validator should fail closed.

Unsupported fields should cause an error, not a warning.

### 4. `Compiler`

Converts validated front matter into an internal normalized automation object.

Responsibilities:

- apply defaults
- normalize schedule shape
- normalize resource descriptors
- normalize host capability grants
- normalize output artifacts
- attach source metadata such as file path and file hash

The compiled object is what the runtime executes.

The source file is not executed directly without first compiling the contract.

### 5. `RunManager`

Coordinates one immediate run.

Responsibilities:

- create a run id
- create a run record in `starting` state
- launch the worker in a container
- attach capability host session state
- collect logs, outputs, and final status
- persist the completed run record

This is the orchestration layer for a single execution.

### 6. `ContainerRuntime`

Runs the Python worker inside an Apple Container.

This is a required boundary in `v0`.

Responsibilities:

- prepare a container image or runtime base
- mount only the minimal required files/directories
- inject the script source
- inject connection/session metadata
- establish the host communication channel
- enforce timeout and termination policy

The container should not receive:

- raw provider credentials
- broad host filesystem access
- ambient network access not explicitly required by the host design

### 7. `CapabilityHost`

Host-side implementation of the runtime services the script is allowed to call.

In `v0`, this is the heart of the trust model.

Responsibilities:

- expose only declared capabilities
- validate each call against the compiled automation contract
- route to the appropriate underlying subsystem
- log usage per run
- deny undeclared access

Initial `v0` capabilities:

- `log`
- `artifacts`
- `state`
- `llm`
- `html`

### 8. `StateStore`

Persistent store for operational state only.

Supported `v0` kinds:

- `key_value`
- `set`

Responsibilities:

- resolve declared stores
- enforce read/write access
- perform atomic reads and writes
- record state changes in the run record

This store is for execution correctness, not broader memory.

### 9. `ArtifactStore`

Host-managed output storage.

Responsibilities:

- resolve declared artifact ids to actual paths
- enforce that only declared artifacts may be written
- write files safely
- record artifact writes in the run record

Scripts should write artifacts through the host, not by opening arbitrary file paths.

### 10. `RunStore`

Persistent storage for run records.

Responsibilities:

- create run entries
- append logs and events
- finalize status
- support later inspection

Even if this starts as a local JSON or SQLite store, it should be treated as a first-class subsystem.

### 11. `ConfigStore`

Global host configuration, separate from the automation file.

Responsibilities:

- load model provider configuration
- load secret references
- load runtime profiles
- load storage roots
- load policy defaults

Automations should reference profiles and declared capabilities, not raw credentials.

## Execution Topology

The runtime has two processes:

1. Host process
   The CLI plus all host-owned systems.

2. Worker process
   The Python automation script running inside an Apple Container.

The worker should be treated as untrusted relative to host secrets and host policy, even if the user authored it.

That is the safest stance for architecture.

## Container Design

### Why Apple Containers in v0

This is not an optional hardening pass later.

Containerization is part of the actual product architecture, so `v0` should pay that cost now.

The goal is to validate:

- filesystem mounting assumptions
- runtime packaging assumptions
- host/worker communication
- debug ergonomics
- performance overhead

### Container contents

The container needs:

- Python runtime
- the small `automation_runtime` helper library
- the user script source
- temporary working area if required

It should not contain:

- host secret stores
- raw model provider credentials
- broad host source trees unless explicitly mounted

### Mounting model

Recommended minimal mounts:

- read-only mount for the single automation script
- read-only mounts for declared file resources only
- a narrow writable workspace for temp files if needed
- a narrow IPC mount or socket path for host communication

This should be explicit and inspectable.

### Network model

For `v0`, prefer:

- the worker does not make direct external calls by default
- remote work happens through host capabilities like `llm` and `html`

That keeps the container simple and pushes policy to the host.

If the worker is later allowed direct network access, that should be an explicit future design change.

## Host/Worker Communication

The worker needs a narrow way to call host-managed capabilities.

Recommended direction:

- use a local RPC mechanism over a Unix domain socket shared into the container

Why this is the right default:

- local only
- explicit session boundary
- easy to scope to one run
- works well with host-managed capabilities

The host should create:

- a per-run socket
- a per-run session token

The worker should receive only:

- socket location
- session token
- minimal run metadata

The `automation_runtime` Python helper library should hide the transport details from user code.

## Global Configuration

You need a host-level config file or config store separate from automation files.

This config should define:

- model profiles
- provider accounts
- secret references
- runtime profiles
- storage roots
- policy defaults

### Model configuration

The host should own model routing.

The config should support:

- local model endpoints
- cloud provider accounts
- named profiles like `fast_local`, `balanced_local`, `deep_reasoning`

Automation files should declare:

- that they want `llm`
- desired capability level / reasoning style

They should not embed:

- raw API keys
- provider-specific credentials

### Secrets

For macOS-first `v0`, the design should assume:

- secrets are stored in the macOS Keychain, or
- a local encrypted store protected by a Keychain-held master secret

The worker container should not receive raw secret values by default.

The host resolves secrets internally when serving a capability request.

## Parsing and Validation Pipeline

The pipeline should be:

1. load source file
2. extract front matter
3. parse YAML
4. validate schema
5. validate references
6. validate capability declarations
7. normalize defaults
8. compute source hash
9. compile execution object

This pipeline should be deterministic and testable.

### Source hashing

Each run should record the hash of the source file used.

This gives you:

- reproducibility
- change detection
- later auditability

## Runtime API Surface

The Python helper library should present a small, stable interface:

- `rt.input(name)`
- `rt.resource(id)`
- `rt.llm.complete(...)`
- `rt.state.get(id, default=...)`
- `rt.state.set(id, value)`
- `rt.state.merge_unique(id, values)`
- `rt.log.info(...)`
- `rt.log.warn(...)`
- `rt.html.fetch(...)`
- `rt.artifacts.write(id, value)`

This API should be thin. Most behavior should remain in the host.

## Capability Security Model

The host capability layer is the primary trust boundary.

Every capability call should be:

- declared in front matter
- authorized against the compiled contract
- logged into the run record
- denied by default if undeclared

### `log`

Host responsibilities:

- structured run-associated logging
- severity tagging
- persistence

### `artifacts`

Host responsibilities:

- map artifact ids to declared output paths
- enforce allowed artifact ids
- write content safely
- record artifact writes

### `state`

Host responsibilities:

- enforce allowed reads/writes
- ensure store existence
- apply updates atomically
- record state changes

### `llm`

Host responsibilities:

- select provider/profile
- use host-owned credentials
- enforce capability settings
- log usage
- enforce timeouts and retries

The worker should not receive raw model credentials.

### `html`

This is especially important.

Host responsibilities:

- fetch remote content
- normalize HTML
- extract useful structured text/content
- sanitize dangerous or noisy content
- perform prompt-injection scanning / content filtering before returning content to the worker

This is where IronClaw-style defensive content handling belongs in `v0`.

The worker should consume host-sanitized content, not raw uncontrolled HTML by default.

## Scheduler Handling in v0

`v0` is intentionally not a scheduler.

However:

- `triggers.schedule` is still required in the file
- the host must parse and validate it
- the compiled object should still contain normalized schedule data

But:

- the CLI ignores the schedule at execution time
- `run` means run now

This keeps the automation contract future-compatible without requiring a daemon in `v0`.

## CLI Behavior

### `validate`

Should:

- parse front matter
- validate it
- report any schema/reference/capability errors
- optionally print normalized summary

### `run`

Should:

- do everything `validate` does
- create a run record
- launch the container worker
- stream or summarize logs
- print final status and artifact locations

## Persistence

`v0` should persist at least:

- run records
- state stores
- artifacts

The simplest acceptable `v0` storage choices are:

- SQLite for run records and state, or
- SQLite for run records + JSON-on-disk for state

Artifacts should remain ordinary files at declared paths, but they should be written through the host.

## Error Model

At minimum, classify failures as:

- source load error
- front matter parse error
- schema validation error
- reference validation error
- container launch error
- worker runtime error
- capability denial
- capability backend error
- artifact write failure
- state write failure

The run record should capture which failure class occurred.

## Recommended Implementation Order

Build `v0` in this order:

1. `AutomationLoader`
2. `Validator`
3. `Compiler`
4. `ConfigStore`
5. `RunStore`
6. `ArtifactStore`
7. `StateStore`
8. Apple `ContainerRuntime`
9. `CapabilityHost` with `log` and `artifacts`
10. `CapabilityHost` with `state`
11. `CapabilityHost` with `llm`
12. `CapabilityHost` with `html`
13. `CLI run`

This sequence gives you a working vertical slice as early as possible while preserving the real architecture.

## First Vertical Slice

The first vertical slice should be:

- one automation file
- one container run
- one declared file resource
- host-managed `log`
- host-managed `artifacts`

No `llm` yet.

That proves:

- front matter parsing
- container launch
- host/worker communication
- runtime API basics
- artifact writing

Then add `state`, then `llm`, then `html`.

## Open Questions

These are important, but they should not block the first build:

- What exact Apple Containers integration API/library will the host use?
- How will the Python runtime be packaged into the container image?
- Should the source file be bind-mounted or copied into a temporary container workspace?
- What concrete RPC transport format should be used over the Unix socket?
- Should `html` return raw sanitized text, structured blocks, or both?

## Bottom Line

`v0` should be a CLI-based immediate runner for a single-file Python automation, executed inside an Apple Container, with all meaningful power flowing through host-managed capabilities.

That is the right build target because it forces you to prove the hardest parts early:

- the container boundary
- the host capability model
- secrets and provider isolation
- safe HTML handling
- and run-time trustworthiness
