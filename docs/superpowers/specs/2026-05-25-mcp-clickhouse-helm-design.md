# MCP ClickHouse Helm Deployment Design

## Context

This design defines a minimal, explainable deployment flow for running the MCP ClickHouse server in Kubernetes using Helm, with image publication to a private Docker registry as the first step.

The design intentionally avoids application code or Dockerfile behavior changes.

## Goals

- Deploy MCP ClickHouse server to Kubernetes using an independent Helm chart.
- Reuse existing Kubernetes secrets by reference (no secret values in repo).
- Publish container image first, then deploy via Helm.
- Keep chart and tasks minimal and environment-aligned.

## Non-Goals

- No Dockerfile behavior changes.
- No Helm chart push to chart registry.
- No secret creation or rotation from this repo.
- No source-code changes in the Python server implementation.

## Confirmed Decisions

- Transport/Auth mode: HTTP with auth disabled.
- ClickHouse endpoint: `<clickhouse-host>:<clickhouse-port>`.
- Existing secret: `<existing-secret-name>`.
- Secret keys for credentials:
  - `<secret-user-key>`
  - `<secret-password-key>`
- Docker repository:
  - `<container-registry>/<image-name>`
- Tag policy:
  - Use configurable release tag (default `1.0.0`)
  - Use git SHA tag
  - Do not use `latest`
- Runtime deployment target namespace: `localdev`.
- Chart values namespace remains configurable with default `monitoring`.

## Architecture

### Artifact Flow

1. Build image from existing project Dockerfile.
2. Tag image with:
   - release tag (`1.0.0` by default, configurable)
   - git SHA
3. Push both tags to the configured private registry repository.
4. Validate Helm chart (`lint` and `template`).
5. Deploy or upgrade release in Kubernetes using `helm upgrade --install` to `localdev`.

### Chart Layout

Chart location:

- `helm/mcp-clickhouse/`

Minimal files:

- `helm/mcp-clickhouse/Chart.yaml`
- `helm/mcp-clickhouse/values.yaml`
- `helm/mcp-clickhouse/templates/_helpers.tpl`
- `helm/mcp-clickhouse/templates/deployment.yaml`
- `helm/mcp-clickhouse/templates/service.yaml`

## Configuration Contract

### Image Values

- `image.repository` default:
  - `<container-registry>/<image-name>`
- `image.tag` default:
  - `1.0.0`
- `image.pullPolicy` default:
  - `IfNotPresent`

### Runtime Values

- `namespace` default:
  - `monitoring`
- `service.type` default:
  - `ClusterIP`
- `service.port` default:
  - `8000`
- `container.port` default:
  - `8000`

### ClickHouse Values

- `clickhouse.host` default:
  - `<clickhouse-host>`
- `clickhouse.port` default:
  - `<clickhouse-port>`
- `clickhouse.secret.name` default:
  - `<existing-secret-name>`
- `clickhouse.secret.userKey` default:
  - `<secret-user-key>`
- `clickhouse.secret.passwordKey` default:
  - `<secret-password-key>`

### Auth/Transport Values

- `server.transport` default:
  - `http`
- `server.authDisabled` default:
  - `true`

## Kubernetes Behavior

### Deployment

Deployment injects environment variables required by the MCP server:

- `CLICKHOUSE_HOST` from values
- `CLICKHOUSE_PORT` from values
- `CLICKHOUSE_USER` from `valueFrom.secretKeyRef`
- `CLICKHOUSE_PASSWORD` from `valueFrom.secretKeyRef`
- `CLICKHOUSE_MCP_AUTH_DISABLED=true`

### Health Probes

Because transport is HTTP, use `/health` for probes:

- readinessProbe: HTTP GET `/health` on container port
- livenessProbe: HTTP GET `/health` on container port

### Service

Expose HTTP port internally via ClusterIP service.

## VS Code Tasks Design

Task file:

- `.vscode/tasks.json`

### Required Tasks

1. Build image
- Build using project Dockerfile.
- Compute git SHA.
- Tag local image with release tag and SHA.

2. Push image
- Push release tag.
- Push SHA tag.

3. Helm lint/template
- `helm lint helm/mcp-clickhouse`
- `helm template ...` with local values

4. Helm deploy
- `helm upgrade --install`
- namespace `localdev`
- chart path `helm/mcp-clickhouse`

5. Optional smoke check
- `kubectl -n localdev rollout status deployment/$RELEASE_NAME`
- `kubectl -n localdev get pods,svc`

### Task Variables

Use centralized variables in task commands:

- `IMAGE_REPO`
- `IMAGE_TAG` (default `1.0.0`)
- `IMAGE_SHA` (from `git rev-parse --short HEAD`)
- `RELEASE_NAME`
- `NAMESPACE` (deploy task default `localdev`)

## Error Handling and Validation

- If Docker login is missing, push task should fail with clear registry auth error.
- If secret is not present in target namespace (`localdev`), deployment will fail at runtime; task output must make namespace explicit.
- If secret key names are wrong, pod startup fails; document keys in values and README notes for chart.
- If cluster connectivity is missing, deploy task fails early from Helm/Kubectl.

## Testing Strategy

### Local/CI-Style Checks

- `helm lint` must pass.
- `helm template` must render without errors.
- Deployment manifest must include secretKeyRef for credentials.
- Deployment manifest must include `/health` probes.

### Cluster Checks

- Deployment rollout succeeds in `localdev`.
- Service is created.
- Pod reaches Ready state.
- `/health` endpoint responds OK inside cluster routing path.

## Security and Compliance

- Never commit secret values.
- Only reference existing secret names and keys.
- Keep auth-disabled mode explicit and environment-scoped.
- Keep deployment changes minimal and auditable.

## Implementation Boundaries

- This design authorizes creation of Helm chart files under `helm/mcp-clickhouse/`.
- This design authorizes creation/update of `.vscode/tasks.json` for the specified workflow.
- This design does not authorize source-code behavior changes.

## Risks and Mitigations

- Risk: values default namespace (`monitoring`) differs from deploy task namespace (`localdev`).
  - Mitigation: deploy task sets namespace explicitly to `localdev`; document that values default is generic.
- Risk: image tag drift between pushed tags and helm deploy value.
  - Mitigation: deploy task injects selected `IMAGE_TAG` explicitly.
- Risk: secret key mismatch in future.
  - Mitigation: keys are configurable values with environment defaults.

## Acceptance Criteria

- Chart exists at `helm/mcp-clickhouse/` with minimal templates.
- Tasks exist for build, push, lint/template, deploy, and optional smoke check.
- Build/push uses release tag + git SHA and never `latest`.
- Helm deploy targets `localdev` and consumes existing secret reference.
- No Dockerfile or app code changes are required.
