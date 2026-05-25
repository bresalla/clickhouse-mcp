---
description: "Use when creating or updating a Helm chart and VS Code tasks for this MCP server with minimal changes and existing Kubernetes Secrets."
---

Create or update the deployment packaging for this repository under these constraints:

1. Start from [README.md](../../README.md) and [Dockerfile](../../Dockerfile).
2. Do not inspect Python source unless those files are insufficient to answer a required deployment question.
3. Before each proposed file change, ask for approval and explain why that specific change is needed.
4. Keep every change minimal and easy to justify.
5. Create the chart in `helm/mcp-clickhouse/` as an independent directory.
6. Reuse existing Kubernetes Secrets by reference. Do not add secret values to the repo.
7. Do not change Dockerfile behavior unless the user explicitly approves it and the change is required.
8. Add or update `.vscode/tasks.json` only for the smallest useful set of tasks:
   - build image
   - run locally
   - run tests
   - helm lint or template
   - helm upgrade or install
9. If a command, port, transport mode, secret name, or deployment assumption is missing from docs, stop and ask instead of inventing it.

When you finish, summarize the approved changes and any assumptions that still need confirmation.