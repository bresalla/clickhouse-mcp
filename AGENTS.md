# Agent Guidance

Use this file for repo-wide behavior that is not obvious from the file tree.

## Scope

- For Helm packaging work, start with [README.md](README.md) and [Dockerfile](Dockerfile).
- If those files are sufficient, do not inspect Python source just to create or update Kubernetes packaging.
- Keep deployment changes minimal and explainable.

## Container Assumptions

- Preserve the Docker image entrypoint: `python -m mcp_clickhouse.main`.
- Preserve the existing Dockerfile behavior unless the user explicitly approves a change.
- Treat ClickHouse credentials and MCP auth settings as runtime configuration, not build-time defaults.

## Helm Packaging Rules

- Put the chart in `helm/mcp-clickhouse/` unless the user asks for a different location.
- Reuse existing Kubernetes Secrets by reference. Never commit secret values.
- Prefer `valueFrom.secretKeyRef` over inline environment values for sensitive settings.
- Assume `/health` is the probe target when HTTP or SSE transport is enabled.
- If transport, port, image registry, or secret names are not documented, ask before inventing defaults.

## VS Code Tasks

- Keep tasks small wrappers around existing CLI tools such as `docker`, `uv`, `helm`, and `kubectl`.
- Add only the tasks needed to build, run, test, lint/template, and deploy the chart.
- If a run or test command is not documented, call that out and ask before hard-coding it.

## Verification Scope

- Do not run full `pytest` for Helm packaging or deployment-task work.
- Verify only the code and artifacts related to the approved spec (for example Helm templates, chart lint/template output, and task configuration).

## Change Control

- Ask for approval before each file change in Helm packaging work and explain why the change is needed.
- Avoid opportunistic refactors and do not modify application behavior as part of chart setup.
- NEVER run `git push` from this workspace.
