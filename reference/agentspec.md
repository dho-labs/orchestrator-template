# AgentSpec Reference

The agentspec is the set of declarative spec files that define the desired state of an
Orchestrator workspace. It lives in the `agentspec/` directory of an agentspace repository.

Both humans and agents author the agentspec. The reconciler treats all commits identically — autonomy
mode controls how commits land (direct push vs PR review), not what can be written.

## Agentspace Repository Layout

```
agentspace-repo/
├── agentspec/
│   ├── repos/
│   │   └── *.yaml
│   ├── environments/
│   │   └── *.yaml
│   ├── runtimes/
│   │   └── *.yaml
│   ├── skills/
│   │   └── <skill-key>/
│   │       ├── skill.yaml
│   │       ├── SKILL.md
│   │       └── <optional assets>
│   ├── integrations/
│   │   └── *.yaml
│   ├── tools/
│   │   └── *.yaml
│   ├── workflows/
│   │   └── *.yaml
│   └── automations/
│       └── *.yaml
├── prompts/
│   ├── orchestrator.md
│   ├── delegator.md
│   └── executor.md
├── tools/
│   └── *.tool.{ts,js,mjs}
```

The `agentspec/` directory holds all spec files. Prompts and custom tools
live at the repo root as sibling directories — they are runtime content, not declarative specs.

## Declarative vs Runtime Content

AgentSpec models desired state that the reconciler can validate and project into runtime records.
Prompt Markdown and custom tool code are executable runtime artifacts, so AgentSpec stores pointers
and policy for them instead of inlining the content.

- `RuntimeSpec.spec.promptPath` points to a prompt file committed in the agentspace repository.
- `RuntimeSpec.spec.toolPolicy` and `EnvironmentSpec.spec.toolPolicy` control which tools can run.
- `ToolSpec` declares tool contracts (integration, local, or builtin) with schema/risk metadata.
- Local ToolSpecs (`spec.source=local`) point to runtime modules via `spec.modulePath`.
- `tools/*.tool.ts|js|mjs` defines repo-local custom runtime tools loaded at runtime.

This split is intentional:

1. Keep reconcile inputs deterministic and compact.
2. Keep executable artifacts in files (versioned, reviewable, testable) rather than projected state.
3. Allow prompt/tool iteration without changing AgentSpec schema shape.

### Implementation Notes (Current Runtime)

- Prompt files are required at reconcile time. Missing `spec.promptPath` defaults to `prompts/<role>.md`,
  and reconcile fails if the file is absent.
- Runtime prompt content is loaded from the pinned agentspace checkout at run start (commit-pinned provenance).
- Custom tool modules are loaded only from declared local ToolSpecs (`spec.source=local`) using
  `spec.modulePath`.
- `spec.modulePath` must reference an existing `*.tool.ts|js|mjs` file in the same agentspace commit.
- Repo custom tools are loaded for orchestrator, delegator, and executor runs.
- Only ToolSpec-declared local runtime tools are injected.
- Agent execution requires an explicit runtime prompt loaded from the agentspace repo; there is no
  in-code prompt fallback path.

### Minimal Agentspace Example

```text
agentspace-repo/
├── agentspec/
│   ├── runtimes/
│   │   └── executor-default.yaml
│   └── tools/
│       └── datadog-search-logs.yaml
├── prompts/
│   └── executor.md
├── tools/
│   └── summarize-workspace.tool.{ts,js,mjs}
```

`agentspec/runtimes/executor-default.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: RuntimeSpec
metadata:
  key: executor-default
spec:
  role: executor
  isDefault: true
  agentHarness: codex
  model: gpt-5.2-codex
  promptPath: prompts/executor.md
```

`agentspec/tools/datadog-search-logs.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: ToolSpec
metadata:
  key: datadog-search-logs
spec:
  source: integration
  integrationKey: datadog
  toolId: search_logs
  description: Search Datadog logs by query and time window.
  riskClass: read
```

`agentspec/tools/summarize-workspace.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: ToolSpec
metadata:
  key: summarize-workspace
spec:
  source: local
  toolId: summarize_workspace
  modulePath: tools/summarize-workspace.tool.ts
  description: Summarize workspace status from orchestrator context.
  riskClass: read
```

`tools/summarize-workspace.tool.ts`

```ts
import { z } from "zod";

export default {
  id: "summarize_workspace",
  description: "Summarize workspace status from orchestrator context.",
  inputSchema: z.object({
    limit: z.number().int().min(1).max(50).default(10),
  }),
  outputSchema: z.object({
    summary: z.string(),
  }),
  run: async (input: { limit: number }) => ({
    summary: `Summarized ${input.limit} items.`,
  }),
};
```

## File Format

One resource per file. YAML with four top-level keys:

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: RepoSpec
metadata:
  key: my-app
spec:
  provider: github
  owner: acme
  name: platform
  defaultBranch: main
```

| Field                  | Description                                                                                                                                        |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `apiVersion`           | Always `agentspec.orchestrator.dev/v1alpha1` for now.                                                                                              |
| `kind`                 | Resource type. One of: `RepoSpec`, `EnvironmentSpec`, `RuntimeSpec`, `SkillSpec`, `ToolSpec`, `IntegrationSpec`, `WorkflowSpec`, `AutomationSpec`. |
| `metadata.key`         | Stable identity key for this resource. Immutable after creation — renaming means delete + create. Must be unique within its kind per workspace.    |
| `metadata.annotations` | Optional key-value pairs for lifecycle control (e.g., `orchestrator.dho.dev/prune: "true"`).                                                       |
| `spec`                 | Kind-specific fields (see below).                                                                                                                  |

## Revision Model

Two workspace-level pointers track agentspec state:

- `desiredAgentSpecSha` — latest pushed commit SHA. Updated on push webhook, agent commit, or manual trigger.
- `activeAgentSpecSha` — last successfully reconciled commit SHA.

When `desired != active`, a reconcile is pending. On success, `active = desired`. On failure,
`active` stays at the last good SHA. Rollback by pushing a known-good commit.

## Manual Reconcile API

`POST /api/v1/workspaces/[id]/agentspec/reconcile`

Request body:

```json
{
  "sha": "optional-commit-sha",
  "dryRun": true
}
```

- `dryRun=true` compiles and validates the target SHA, then returns a deterministic plan with zero DB writes.
- `dryRun=false` (default) runs atomic apply. Any write failure rolls back the full apply transaction.

Response includes:

- `mode`: `"dry-run"` or `"apply"`
- `status`: `"dry-run" | "reconciled" | "skipped"`
- `plan` for dry-run
- `commitSha`, `eventId`

## Lifecycle

Resources removed from the agentspec are **orphaned by default** — suspended and disconnected from
declarative management, but not deleted. This prevents accidental destruction of runtime state.

To opt into deletion:

- Per-resource: set annotation `orchestrator.dho.dev/prune: "true"` before removing the file
- Per-workspace: enable `prunePolicy: enabled` in workspace settings

## Reconcile Ordering

Within a single reconcile pass, resources are applied in dependency order:

1. Environments (no dependencies)
2. Repos (no dependencies)
3. Skills (no dependencies)
4. Runtime specs (may reference environments)
5. Tools (integration tools may reference integrations; local/builtin tools are standalone)
6. Integrations (may reference environments, secret keys; project ToolSpecs into runtime tool definitions)
7. Workflows (may reference runtime specs, integrations)
8. Automations (may reference workflows, integrations, environments)

Within a tier, resources are applied in parallel.

---

## RepoSpec

Declares a repository the workspace operates on. Runtime auth, installation state, and token
linkage remain in the database.

**File location:** `agentspec/repos/<key>.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: RepoSpec
metadata:
  key: app
spec:
  provider: github
  owner: acme
  name: platform
  defaultBranch: main
  autonomyMode: full
```

### Fields

| Field                | Type                                             | Required | Default   | Description                                                                                          |
| -------------------- | ------------------------------------------------ | -------- | --------- | ---------------------------------------------------------------------------------------------------- |
| `spec.provider`      | `"github"`                                       | Yes      | —         | Git hosting provider. Only `github` is supported currently.                                          |
| `spec.owner`         | string                                           | Yes      | —         | Repository owner (organization or user).                                                             |
| `spec.name`          | string                                           | Yes      | —         | Repository name.                                                                                     |
| `spec.defaultBranch` | string                                           | No       | `"main"`  | Default branch for cloning and PR targets.                                                           |
| `spec.autonomyMode`  | `"full"` \| `"agent-review"` \| `"human-review"` | No       | Inherited | Repo-level override for git autonomy mode. If omitted, falls back to environment/workspace defaults. |

### Identity

`metadata.key` is the stable identity. If a repository is renamed or transferred on GitHub, update
`spec.owner` and `spec.name` while keeping the same key — the reconciler sees this as an update, not
a delete + create.

### Runtime State (DB-only, not in agentspec)

- Installation and auth resolution
- Token linkage and credential references
- Sync/bootstrap status and error state

---

## EnvironmentSpec

Declares an execution environment with network policy, tool policy overlays, and secret key
requirements. Secret values are never in the agentspec — only the key names that must be present.

**File location:** `agentspec/environments/<key>.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: EnvironmentSpec
metadata:
  key: dev
spec:
  name: Development
  isDefault: true
  networkPolicy:
    egressMode: allowlist
    allowedDomains:
      - api.openai.com
      - api.anthropic.com
  secrets:
    requiredKeys:
      - OPENAI_API_KEY
      - REDIS_URL
  toolPolicy:
    allow:
      - github_*
      - slack_*
    deny:
      - destructive_*
```

### Fields

| Field                               | Type                                             | Required | Default                 | Description                                                                                         |
| ----------------------------------- | ------------------------------------------------ | -------- | ----------------------- | --------------------------------------------------------------------------------------------------- |
| `spec.name`                         | string                                           | No       | Value of `metadata.key` | Display name.                                                                                       |
| `spec.isDefault`                    | boolean                                          | No       | `false`                 | Whether this is the default environment for the workspace. Exactly one environment must be default. |
| `spec.networkPolicy.egressMode`     | `"allowlist"` \| `"denylist"` \| `"open"`        | No       | `"open"`                | Network egress policy mode.                                                                         |
| `spec.networkPolicy.allowedDomains` | string[]                                         | No       | `[]`                    | Domains allowed for egress (only when `egressMode: allowlist`).                                     |
| `spec.networkPolicy.deniedDomains`  | string[]                                         | No       | `[]`                    | Domains blocked for egress (only when `egressMode: denylist`).                                      |
| `spec.secrets.requiredKeys`         | string[]                                         | No       | `[]`                    | Secret key names that must have values set in the database before runs can use this environment.    |
| `spec.toolPolicy.allow`             | string[]                                         | No       | `[]`                    | Glob patterns for allowed tools.                                                                    |
| `spec.toolPolicy.deny`              | string[]                                         | No       | `[]`                    | Glob patterns for denied tools.                                                                     |
| `spec.autonomyMode`                 | `"full"` \| `"agent-review"` \| `"human-review"` | No       | Inherited               | Environment-level git autonomy override.                                                            |

### Policy Inheritance

Environment specs inherit from workspace-level defaults. Fields explicitly set in the spec override
workspace defaults. Omitted fields inherit. Workspace policy sets the ceiling — environment overrides
cannot exceed it.

### Runtime State (DB-only, not in agentspec)

- Encrypted secret values (`environment_secrets` table)
- Secret rotation timestamps
- Runtime status and usage telemetry

---

## RuntimeSpec

Declares runtime defaults for a specific agent role — model, harness, prompt path, and tool policy.
Multiple runtime specs per role are supported; one must be designated as the default.

**File location:** `agentspec/runtimes/<key>.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: RuntimeSpec
metadata:
  key: executor-default
spec:
  role: executor
  isDefault: true
  agentHarness: claude-code
  model: claude-sonnet-4-5-20250929
  promptPath: prompts/executor.md
  toolPolicy:
    allow:
      - github_*
    deny: []
```

### Fields

| Field                   | Type                                                        | Required | Default             | Description                                                                                               |
| ----------------------- | ----------------------------------------------------------- | -------- | ------------------- | --------------------------------------------------------------------------------------------------------- |
| `spec.role`             | `"orchestrator"` \| `"delegator"` \| `"executor"`           | Yes      | —                   | Agent role this runtime spec applies to.                                                                  |
| `spec.isDefault`        | boolean                                                     | No       | `false`             | Whether this is the default runtime spec for its role. Exactly one runtime spec per role must be default. |
| `spec.agentHarness`     | `"general"` \| `"claude-code"` \| `"codex"` \| `"opencode"` | No       | `"general"`         | Runtime harness. `orchestrator` and `delegator` must use `general`. `executor` can use any value.         |
| `spec.model`            | string                                                      | No       | Per-harness default | Model identifier. Must be valid for the selected harness.                                                 |
| `spec.promptPath`       | string                                                      | No       | `prompts/<role>.md` | Path to the prompt file, relative to agentspace repo root. File must exist.                               |
| `spec.toolPolicy.allow` | string[]                                                    | No       | `[]`                | Glob patterns for allowed tools.                                                                          |
| `spec.toolPolicy.deny`  | string[]                                                    | No       | `[]`                | Glob patterns for denied tools.                                                                           |

### Harness Constraints

| Harness       | Roles                             | Available Models                                                                                                                     |
| ------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `general`     | orchestrator, delegator, executor | `claude-sonnet-4-5-20250929`, `claude-opus-4-6`, `claude-haiku-4-5-20251001`, `gpt-5.2`, `gpt-5.2-codex`, `gpt-5-mini`, `gpt-5-nano` |
| `claude-code` | executor only                     | `claude-sonnet-4-5-20250929`, `claude-opus-4-6`, `claude-haiku-4-5-20251001`                                                         |
| `codex`       | executor only                     | `gpt-5.2`, `gpt-5.2-codex`, `gpt-5-mini`, `gpt-5-nano`                                                                               |
| `opencode`    | executor only                     | `claude-sonnet-4-5-20250929`, `claude-opus-4-6`, `gpt-5.2`, `gpt-5-mini`, `gpt-5-nano`                                               |

### Override Hierarchy

When resolving runtime for a run:

1. **Run request** — most specific, wins if set
2. **Runtime spec default** — inherited if run request omits the field
3. **Workspace default** — fallback

Workspace policy enforces the ceiling. A run request for `model: claude-opus-4-6` is rejected if
workspace policy caps at `claude-sonnet-4-5-20250929`.

### Runtime State (DB-only, not in agentspec)

- Projected runtimes used by run creation
- Run-level provenance (`sourceCommitSha`, selected runtime key)
- Execution state (runs, activities, outcomes)

---

## SkillSpec

Declares a skill — reusable procedural knowledge that agents can invoke during execution. Skills
are a primary surface for agent-authored content. Unlike other resource types, skills do not require
DB projection — runtime loads them directly from the pinned agentspec checkout.

**File location:** `agentspec/skills/<key>/skill.yaml` (with `SKILL.md` alongside)

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: SkillSpec
metadata:
  key: code-review
spec:
  name: Code Review
  description: Reviews pull requests for bugs, security issues, and style.
  tags:
    - review
    - quality
  contentRoot: .
  exposurePolicy:
    scope: workspace
    allowedRoles:
      - executor
```

### Fields

| Field                              | Type                        | Required | Default       | Description                                                                                                                                                                   |
| ---------------------------------- | --------------------------- | -------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `spec.name`                        | string                      | Yes      | —             | Human-readable skill name.                                                                                                                                                    |
| `spec.description`                 | string                      | Yes      | —             | What the skill does.                                                                                                                                                          |
| `spec.tags`                        | string[]                    | No       | `[]`          | Tags for categorization and discovery.                                                                                                                                        |
| `spec.contentRoot`                 | string                      | No       | `"."`         | Directory containing skill content, relative to the skill.yaml file.                                                                                                          |
| `spec.exposurePolicy.scope`        | `"workspace"` \| `"global"` | No       | `"workspace"` | Visibility scope. `global` skills are available across workspaces in the organization.                                                                                        |
| `spec.exposurePolicy.allowedRoles` | string[]                    | No       | All roles     | Which agent roles can use this skill.                                                                                                                                         |
| `spec.source`                      | string                      | No       | —             | External source reference for shared skills (e.g., `github.com/org/shared-skills/code-review@v1.0`). If set, skill content is resolved from this reference at reconcile time. |

### Skill Content Structure

```
agentspec/skills/code-review/
├── skill.yaml          # SkillSpec metadata
├── SKILL.md            # Skill instructions (required)
└── references/         # Optional supporting files
    └── checklist.md
```

The `SKILL.md` file is the primary skill content that agents read. Additional files in the content
root are available as references.

### Size Limits

- Warn at 500KB per skill
- Hard fail at 1MB per skill

### Shared Skills

Skills can be imported by reference using `spec.source`. The reconciler resolves the reference at
reconcile time and pins the resolved content SHA. The resolved content is cached locally.

### Runtime Behavior

Skills are loaded directly from the pinned agentspec checkout — no DB projection required. An
optional in-memory cache keyed by commit SHA is allowed for performance.

---

## IntegrationSpec

Declares an external service integration — provider identity, credential requirements, and webhook
capabilities. Tool contracts are declared separately via `ToolSpec`.

**File location:** `agentspec/integrations/<key>.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: IntegrationSpec
metadata:
  key: github
spec:
  provider: github
  credentials:
    secretKeys:
      - GITHUB_TOKEN
  webhook:
    enabled: true
    events:
      - issues.opened
      - pull_request.opened
```

### Fields

| Field                         | Type     | Required | Default | Description                                                                                          |
| ----------------------------- | -------- | -------- | ------- | ---------------------------------------------------------------------------------------------------- |
| `spec.provider`               | string   | Yes      | —       | Integration provider identifier (e.g., `github`, `slack`, `sentry`, `linear`).                       |
| `spec.credentials.secretKeys` | string[] | No       | `[]`    | Secret key names required for this integration. Must be present in the active environment's secrets. |
| `spec.webhook.enabled`        | boolean  | No       | `false` | Whether this integration receives webhooks.                                                          |
| `spec.webhook.events`         | string[] | No       | `[]`    | Event types this integration subscribes to.                                                          |

### Typed vs Generic Adapters

- **Typed adapters** exist for high-use providers: GitHub, Slack, Sentry, Linear. These have
  provider-specific signature verification, payload normalization, and tool implementations.
- **Generic adapter** handles everything else via configurable auth and webhook-in/API-out.

### Runtime State (DB-only, not in agentspec)

- Connection and auth state (`workspace_integrations`, `integration_user_authorizations`)
- Approval records and audit logs
- Webhook verification secrets

---

## ToolSpec

Declares a tool contract. ToolSpecs can model:

- integration-backed tools (projected into integration runtime tool definitions),
- repo-local runtime tools, and
- builtin runtime tools.

**File location:** `agentspec/tools/<key>.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: ToolSpec
metadata:
  key: datadog-search-logs
spec:
  source: integration
  integrationKey: datadog
  toolId: search_logs
  name: Search Logs
  description: Search Datadog logs by query and time window.
  riskClass: read
  inputSchema:
    query:
      type: string
    from:
      type: string
    to:
      type: string
```

### Fields

| Field                        | Type                                                                      | Required    | Default         | Description                                                                                                          |
| ---------------------------- | ------------------------------------------------------------------------- | ----------- | --------------- | -------------------------------------------------------------------------------------------------------------------- |
| `spec.source`                | `"integration"` \| `"local"` \| `"builtin"`                               | No          | `"integration"` | Tool source plane.                                                                                                   |
| `spec.integrationKey`        | string                                                                    | Conditional | —               | Required when `spec.source=integration`. Must reference an existing `IntegrationSpec` key.                           |
| `spec.modulePath`            | string                                                                    | Conditional | —               | Required when `spec.source=local`. Repo-relative path to a `*.tool.ts`, `*.tool.js`, or `*.tool.mjs` runtime module. |
| `spec.toolId`                | string                                                                    | Yes         | —               | Runtime tool identifier within the selected source plane.                                                            |
| `spec.name`                  | string                                                                    | No          | `spec.toolId`   | Human-readable tool name.                                                                                            |
| `spec.description`           | string                                                                    | Yes         | —               | Human-readable description shown to the agent.                                                                       |
| `spec.riskClass`             | `"read"` \| `"write_reversible"` \| `"write_irreversible"` \| `"unknown"` | No          | Derived         | Explicit tool risk classification.                                                                                   |
| `spec.writeApprovalRequired` | boolean                                                                   | No          | `true`          | Convenience risk toggle. `false` maps to `riskClass=read`, otherwise `write_irreversible`.                           |
| `spec.inputSchema`           | object                                                                    | No          | —               | JSON-schema-like hint surfaced to runtime/tool UIs.                                                                  |
| `spec.annotations`           | object                                                                    | No          | —               | Additional non-sensitive tool metadata.                                                                              |
| `spec.method`                | `"GET"` \| `"POST"` \| `"PUT"` \| `"PATCH"` \| `"DELETE"`                 | No          | —               | Optional HTTP method hint (useful for dynamic/custom API integrations).                                              |
| `spec.path`                  | string                                                                    | No          | —               | Optional API path hint (useful for dynamic/custom API integrations).                                                 |
| `spec.remoteToolName`        | string                                                                    | No          | —               | Optional remote MCP tool identifier hint.                                                                            |

### Risk Resolution

- If `spec.riskClass` is set, it wins.
- Otherwise `spec.writeApprovalRequired: false` maps to `read`.
- Otherwise the default is `write_irreversible`.
- Do not set both `spec.riskClass` and `spec.writeApprovalRequired` in the same ToolSpec.

### Runtime Behavior

- Integration ToolSpecs (`spec.source=integration`) are grouped by `spec.integrationKey`.
- The reconciler projects grouped integration ToolSpecs into the target `IntegrationSpec` runtime tool definitions.
- Any integration ToolSpec change updates the owning integration's projected checksum, so reconcile detects drift.
- Local/builtin ToolSpecs are validated and tracked as declarative tool metadata but are not projected into integration templates.
- Builtin ToolSpecs are validated against the canonical built-in tool registry (`packages/constants/src/runtime-tools.ts`).
- Runtime enforcement is opt-in for builtins: once at least one `spec.source=builtin` ToolSpec exists, undeclared built-in tools are denied at runtime via merged tool policy.
- Local ToolSpecs are explicit runtime bindings: each declared local tool ID maps to exactly one declared module path, and only those modules are loaded.

---

## WorkflowSpec

Declares a workflow — an ordered sequence of steps that a task or project follows during execution.

**File location:** `agentspec/workflows/<key>.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: WorkflowSpec
metadata:
  key: default
spec:
  name: Default Workflow
  description: Standard task execution with verification.
  steps:
    - key: plan
      name: Plan
      agentRole: executor
    - key: implement
      name: Implement
      agentRole: executor
    - key: verify
      name: Verify
      agentRole: executor
      runtimeKey: executor-thorough
```

### Fields

| Field                     | Type                                              | Required | Default      | Description                                                                        |
| ------------------------- | ------------------------------------------------- | -------- | ------------ | ---------------------------------------------------------------------------------- |
| `spec.name`               | string                                            | Yes      | —            | Human-readable workflow name.                                                      |
| `spec.description`        | string                                            | No       | —            | What this workflow is for.                                                         |
| `spec.steps`              | StepEntry[]                                       | Yes      | —            | Ordered list of workflow steps.                                                    |
| `spec.steps[].key`        | string                                            | Yes      | —            | Step identifier. Must be unique within the workflow.                               |
| `spec.steps[].name`       | string                                            | Yes      | —            | Human-readable step name.                                                          |
| `spec.steps[].agentRole`  | `"orchestrator"` \| `"delegator"` \| `"executor"` | No       | `"executor"` | Agent role that executes this step.                                                |
| `spec.steps[].runtimeKey` | string                                            | No       | Role default | Override runtime spec for this step. Must reference an existing `RuntimeSpec` key. |

### Runtime State (DB-only, not in agentspec)

- Workflow instance snapshots per run
- Step execution status and outcomes
- Provenance (`sourceCommitSha`)

---

## AutomationSpec

Declares an automation — a trigger-to-task rule that creates work in response to events.

**File location:** `agentspec/automations/<key>.yaml`

```yaml
apiVersion: agentspec.orchestrator.dev/v1alpha1
kind: AutomationSpec
metadata:
  key: triage-issues
spec:
  name: Triage New Issues
  description: Automatically triage and label new GitHub issues.
  trigger:
    type: github_issue_label
    config:
      labels:
        - bug
        - triage
      excludeLabels:
        - wontfix
  target:
    workflowKey: default
    environmentKey: dev
    repoKey: app
  integrationKey: github
```

### Fields

| Field                        | Type   | Required | Default                    | Description                                                                                     |
| ---------------------------- | ------ | -------- | -------------------------- | ----------------------------------------------------------------------------------------------- |
| `spec.name`                  | string | Yes      | —                          | Human-readable automation name.                                                                 |
| `spec.description`           | string | No       | —                          | What this automation does.                                                                      |
| `spec.trigger.type`          | string | Yes      | —                          | Trigger type. See trigger types below.                                                          |
| `spec.trigger.config`        | object | Yes      | —                          | Trigger-specific configuration. Schema depends on `trigger.type`.                               |
| `spec.target.workflowKey`    | string | No       | Workspace default workflow | Workflow to execute. Must reference an existing `WorkflowSpec` key.                             |
| `spec.target.environmentKey` | string | No       | Default environment        | Environment to run in. Must reference an existing `EnvironmentSpec` key.                        |
| `spec.target.repoKey`        | string | No       | —                          | Repository context for the task. Must reference an existing `RepoSpec` key.                     |
| `spec.integrationKey`        | string | No       | —                          | Integration that provides the trigger events. Must reference an existing `IntegrationSpec` key. |

### Trigger Types

| Type                 | Config Fields                                                                              | Description                                                 |
| -------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| `github_issue_label` | `labels: string[]`, `excludeLabels?: string[]`                                             | Fires when a GitHub issue is labeled with a matching label. |
| `cron`               | `schedule: string`, `prompt: string`                                                       | Fires on a cron schedule with a fixed prompt.               |
| `sentry_exception`   | `minLevel: "error" \| "warning" \| "info"`, `excludePatterns?: string[]`                   | Fires on Sentry exceptions at or above the minimum level.   |
| `datadog_alert`      | `minPriority?: "P1"-"P5"`, `includeTags?: string[]`, `excludePatterns?: string[]`          | Fires on Datadog alerts.                                    |
| `pagerduty_incident` | `minUrgency?: "high" \| "low"`, `includeServices?: string[]`, `excludePatterns?: string[]` | Fires on PagerDuty incidents.                               |
| `slack_mention`      | `channelIds?: string[]`, `excludePatterns?: string[]`                                      | Fires when the bot is mentioned in Slack.                   |

### Runtime State (DB-only, not in agentspec)

- Automation event history (`automation_events`)
- Scheduling/processing state
- Webhook routing metadata

---

## Autonomy Modes

Each resource (or workspace-level default) can declare an autonomy mode that controls how
agent-authored changes are committed:

| Mode           | Behavior                                                                   |
| -------------- | -------------------------------------------------------------------------- |
| `full`         | Agent commits push directly to the default branch. Reconciled immediately. |
| `agent-review` | Agent opens a PR. Another agent or automated check reviews before merge.   |
| `human-review` | Agent opens a PR. Human approval required before merge.                    |

Autonomy mode controls the **commit gate**, not the field surface. Agents have full authoring parity
with humans — any field in any spec is writable. If the commit lands, it reconciles identically
regardless of who authored it.

### Effective Mode Precedence

Effective git autonomy for a repo/run resolves in this order:

1. `RepoSpec.spec.autonomyMode`
2. `EnvironmentSpec.spec.autonomyMode`
3. `workspace.settings.agentspec.defaultAutonomyMode` (default is `human-review`)

Executor enforcement by effective mode:

- `full`: direct push to base branch, no PR
- `agent-review`: task branch push + PR, workspace `autoMerge` honored
- `human-review`: task branch push + PR, `autoMerge` forced `false`

## Operations

- Reconcile cron: every 5 minutes (`/api/cron/reconcile-agentspecs`) for pending and drift reconciliation.
- Retention cron: daily (`/api/cron/reconcile-agentspec-retention`) deleting reconcile events older than 30 days.
- Failure streaking: `workspaces.agentspecConsecutiveFailures` increments on failure, resets on success.
- Alerting: threshold alert emitted once when failure streak crosses from `2` to `3`.

## Full Cutover Note

Managed configuration authoring is AgentSpec-only for:

- repositories
- environments (structure/policy/required keys; secret values remain DB-only)
- runtime specs/prompts
- skills
- tools
- integration templates
- workflows
- automations

Legacy create/update/delete/toggle UI/actions for those surfaces are retired with
`"Managed by AgentSpec"` errors for mutation attempts.

## Annotations

Annotations are optional key-value pairs on `metadata.annotations` that control lifecycle behavior.

| Annotation                   | Values   | Description                                                                                                                        |
| ---------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `orchestrator.dho.dev/prune` | `"true"` | Opt this resource into deletion when removed from the agentspec. Without this, removed resources are orphaned (suspended) instead. |

## What Is Not In the AgentSpec

| Concern                     | Where It Lives                                | Why                                               |
| --------------------------- | --------------------------------------------- | ------------------------------------------------- |
| Secret values               | `environment_secrets` table (encrypted)       | Secrets must never be in a git repository.        |
| Runtime execution state     | Database (runs, activities, events, channels) | High-churn operational data, not authored config. |
| Auth tokens and credentials | Database (integration tables)                 | Sensitive runtime state.                          |
| Approval records            | Database (audit tables)                       | Operational compliance records.                   |

## Provenance

Projected rows (DB records created by the reconciler) include provenance fields for auditability:

| Field              | Description                                                      |
| ------------------ | ---------------------------------------------------------------- |
| `sourceKind`       | `"agentspace"` (from agentspec) or `"human"` (manually created). |
| `sourceKey`        | The `metadata.key` from the spec.                                |
| `sourcePath`       | File path within the agentspace repo.                            |
| `sourceChecksum`   | Content hash of the spec file.                                   |
| `sourceCommitSha`  | Commit SHA the spec was reconciled from.                         |
| `sourceAuthorKind` | `"human"` or `"agent"` — who authored the commit.                |
