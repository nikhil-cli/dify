# Feature: Per-request LLM model override for Service API workflow runs

## Status
Implemented (backend core path). Tests not yet written. Not yet reviewed/merged.

## Problem
A client application calling Dify's Service API (`POST /v1/workflows/run`, using an API key) had
no way to choose or change the LLM model (provider + model name) used by a workflow's LLM node(s)
on a per-request basis. The model was fixed on the workflow, and changing it required editing the
workflow via the console (web UI or console API) and republishing.

## Prerequisite (already supported by Dify, no code change)
Dify already has a mechanism for making a workflow's LLM node model configurable at the
workflow-authoring level:

1. Add an environment variable of `value_type: "llm"` to the workflow (e.g. named
   `MODEL_CHOICE`), with a value like `{"provider": "openai", "name": "gpt-4o-mini", "mode": "chat"}`.
   See [api/core/workflow/llm_environment_variable.py](../../../api/core/workflow/llm_environment_variable.py)
   (`LLMEnvironmentVariable`, `LLMModelSelection`).
2. Set the target LLM node's `model_selector` field to `["env", "MODEL_CHOICE"]` instead of a
   static model.

At runtime, [api/core/workflow/node_factory.py](../../../api/core/workflow/node_factory.py)
(`_resolve_llm_model_reference`) resolves the selector against the workflow's persisted
environment variables and swaps in the referenced model. This part required no changes.

## What was missing
The Service API run payload only accepted `inputs` (workflow start-node variables), `files`, and
`response_mode`. There was no field to override an environment variable's value for a single run,
and env vars can otherwise only be changed via the console API (persisted, workflow-wide).

## Solution implemented
Added an optional `env_var_overrides` field to the workflow run request. It lets a caller override
the `provider`/`name` of an already-configured `llm`-type environment variable, for that single run
only — never persisted to the `Workflow` database row.

### Request shape
```json
POST /v1/workflows/run
{
  "inputs": {},
  "response_mode": "blocking",
  "env_var_overrides": {
    "MODEL_CHOICE": { "provider": "openai", "name": "gpt-4o-mini" }
  }
}
```

### Behavior
- Only `provider` and `name` are overridable (no `mode` or `completion_params`) — scope decision
  to keep this simple and avoid mode-mismatch validation complexity.
- If `MODEL_CHOICE` doesn't exist, isn't an `llm`-type environment variable, or the override value
  fails validation, the override is **silently ignored** and the workflow's originally configured
  model is used. No error is raised (explicit product decision — favor availability over strictness
  for this per-request convenience feature).
- Requires the prerequisite one-time workflow setup above (env var + `model_selector`). There is no
  way to override a node that hasn't been wired to an env var — this is a limitation of the
  existing model-resolution architecture, not something this change attempts to bypass.

### Code changes
1. [api/controllers/common/controller_schemas.py](../../../api/controllers/common/controller_schemas.py)
   — added `LLMModelOverride` (`provider`, `name`) and `env_var_overrides: dict[str, LLMModelOverride] | None`
   on the base `WorkflowRunPayload`. The Service API's `WorkflowRunPayload` subclasses this, so the
   field is available on `POST /v1/workflows/run` automatically.
2. [api/core/app/apps/workflow/app_generator.py](../../../api/core/app/apps/workflow/app_generator.py)
   — threads `args.get("env_var_overrides")` into the `extras` dict on `WorkflowAppGenerateEntity`,
   following the existing pattern used for trace IDs (request-scoped metadata, not persisted).
3. [api/core/app/apps/workflow/app_runner.py](../../../api/core/app/apps/workflow/app_runner.py)
   — added `WorkflowAppRunner._apply_env_var_overrides()`, called just before
   `add_variables_to_pool(...)` in `run()`. It copies `self._workflow.environment_variables`, and
   for each override entry whose name matches an existing `LLMEnvironmentVariable`, replaces
   `provider`/`name` on a copy of that variable's value (preserving `mode`/`completion_params`).
   Invalid or unknown entries are logged and skipped. The persisted `Workflow.environment_variables`
   column is never modified — only the in-memory list fed into the `VariablePool` for this run.

No changes were needed to `node_factory._resolve_llm_model_reference`,
`LLMEnvironmentVariable`/`LLMModelSelection`, `build_bootstrap_variables`, or
`add_variables_to_pool` — they resolve purely from the `VariablePool`, so the override is
transparent to them.

### Verification done so far
- `ruff check` passes on all three changed files.
- No Pylance/type errors reported.

## What's next / not yet done

1. **Unit tests** (none written yet):
   - Valid override: workflow has an `llm` env var + node `model_selector` pointing at it; run via
     service API with a matching `env_var_overrides` entry; assert the LLM node resolves to the
     overridden provider/model (mock the model invocation and inspect the resolved `ModelConfig`).
   - Unknown override name: falls back to the workflow's configured model, no error raised.
   - Override targets a variable that exists but isn't an `LLMEnvironmentVariable` (e.g. a plain
     string env var): ignored silently, falls back.
   - Confirm `Workflow.environment_variables` in the DB is byte-for-byte unchanged after a run with
     overrides (no accidental write-back).
   - Existing tests for `resolve_llm_model_config_from_environment` /
     `validate_llm_environment_model_references` still pass unmodified.
2. **Confirm all workflow-run entrypoints are covered.** `WorkflowAppRunner` was assumed to be the
   single runner for all workflow-type app executions (workflow app + workflow-as-tool paths).
   This should be verified so `env_var_overrides` behaves consistently everywhere a workflow can be
   invoked via the Service API, not just the primary `/workflows/run` endpoint.
3. **Chat-based apps.** `WorkflowRunPayloadBase` may be shared by other run endpoints (e.g.
   chatflow-style apps that internally run a workflow). Decide whether `env_var_overrides` should
   be exposed there too or scoped to workflow-app runs only, and adjust the base/subclass schemas
   accordingly.
4. **OpenAPI/docs.** The new field has a `description` set on the pydantic model so it should
   surface in the generated API schema, but the public API reference docs (if maintained
   separately, e.g. under `docs/`) may need a manual update with an example.
5. **Code review.** This has not yet been reviewed. Recommended before merging:
   `backend-code-review` skill (scoped to `api/`) or a manual PR review pass, focused on:
   - Correctness of `_apply_env_var_overrides` (e.g. `model_copy` semantics, that `id`/`selector`
     are preserved).
   - Whether silent-ignore-on-invalid-override is still the right behavior for production, or
     whether a warning surfaced to the caller (e.g. in the response) would be more useful.
   - Whether `LLMModelOverride` should reject unknown/disabled providers eagerly, or leave that to
     the normal model-invocation error path (current design chooses the latter, to avoid
     duplicating provider-validation logic here).
6. **Decide on node-ID-based override (future).** Current design only supports overriding by
   env-var name (requires the one-time workflow setup). If that setup step proves to be too much
   friction for API clients, a follow-up could support overriding an LLM node directly by node ID,
   bypassing the need for a pre-wired `model_selector`. Not attempted in this change; flagged as a
   possible follow-up.
