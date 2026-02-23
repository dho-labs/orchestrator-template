# orchestrator-template

Starter agentspace repository that follows the current AgentSpec layout used by Orchestrator.

## What this template includes

- `agentspec/` with starter specs for repos, environments, runtimes, skills, tools, integrations, workflows, and automations
- `prompts/` with role prompts (`orchestrator`, `delegator`, `executor`)
- `tools/` for custom tool implementations
- `reference/agentspec.md` with the full AgentSpec reference

## Quick start

1. Update `agentspec/repos/app.yaml` with your real GitHub owner/repo/branch.
2. Set required secret keys in `agentspec/environments/dev.yaml` to match your workspace secrets.
3. Adjust runtime defaults in `agentspec/runtimes/*.yaml` (harness, model, prompt paths).
4. Define tool contracts in `agentspec/tools/*.yaml` (integration/local/builtin) and align them with runtime tools.
5. For local tools, set `spec.modulePath` to an existing `tools/*.tool.ts|js|mjs` module.
6. Update `agentspec/workflows/default.yaml` to match your execution flow.
7. Remove or edit starter integration/tool/automation examples before production use.
8. Commit and push. Reconcile applies this desired state in your workspace.

## Rules to keep valid

- Keep `apiVersion` as `agentspec.orchestrator.dev/v1alpha1`.
- Keep `metadata.key` stable after creation. Renames should be delete + create.
- Exactly one default environment should be marked with `spec.isDefault: true`.
- Exactly one default runtime per role should be marked with `spec.isDefault: true`.
- Prompt paths in runtime specs must exist in `prompts/`.

## Starter keys and cross-references

- Repo key: `app`
- Environment key: `dev`
- Runtime keys: `orchestrator-default`, `delegator-default`, `executor-default`
- Workflow key: `default`
- Tool keys: `github-list-pull-requests`, `github-create-issue`
- Integration key: `github`
- Automation key: `triage-issues`
- Skill key: `code-review`

If you change keys, update all references across files.
