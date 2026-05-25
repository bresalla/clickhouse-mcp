---
description: "Use when editing Helm packaging or deployment-task files for this repo. Keeps Helm work separate, minimal, and secret-safe."
applyTo: "helm/**"
---

- Keep the Helm chart isolated under `helm/mcp-clickhouse/`.
- Preserve the runtime behavior defined by [README.md](../../README.md) and [Dockerfile](../../Dockerfile).
- Prefer chart values for non-sensitive configuration and existing Secret references for sensitive configuration.
- Do not inline credentials, auth tokens, or cluster-specific secret values.
- Default to minimal Kubernetes objects: `Chart.yaml`, `values.yaml`, `templates/deployment.yaml`, `templates/service.yaml` only if required, and probe configuration when HTTP or SSE transport is used.
- Treat `/health` as the probe endpoint only when the selected transport exposes HTTP or SSE.
- If the docs do not establish a port, transport, or secret contract, ask the user before adding template defaults.
- Do not modify application code or Dockerfile behavior from Helm work unless explicitly approved.