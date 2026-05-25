# MCP ClickHouse Helm + Tasks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a minimal Helm chart and VS Code tasks to build/push Docker image tags (release + SHA) and deploy the server to Kubernetes in `localdev` using existing secret references.

**Architecture:** Introduce an independent chart under `helm/mcp-clickhouse/` with only chart metadata, values, deployment, service, and helpers. Configure runtime via values and secretKeyRef. Add task automation in `.vscode/tasks.json` for image build/push, chart validation, deployment, and smoke checks.

**Tech Stack:** Helm 3, Kubernetes manifests, Docker CLI, kubectl, VS Code tasks.

---

## File Structure

- Create: `helm/mcp-clickhouse/Chart.yaml`
- Create: `helm/mcp-clickhouse/values.yaml`
- Create: `helm/mcp-clickhouse/templates/_helpers.tpl`
- Create: `helm/mcp-clickhouse/templates/deployment.yaml`
- Create: `helm/mcp-clickhouse/templates/service.yaml`
- Create or Modify: `.vscode/tasks.json`

## Task 1: Create Helm Chart Skeleton

**Files:**

- Create: `helm/mcp-clickhouse/Chart.yaml`
- Create: `helm/mcp-clickhouse/values.yaml`

- [ ] **Step 1: Create chart metadata file**

```yaml
apiVersion: v2
name: mcp-clickhouse
description: Helm chart for MCP ClickHouse server
type: application
version: 0.1.0
appVersion: "1.0.0"
```

- [ ] **Step 2: Create chart values file**

```yaml
namespace: monitoring

image:
  repository: <container-registry>/<image-name>
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8000

container:
  port: 8000

server:
  transport: http
  authDisabled: true

clickhouse:
  host: <clickhouse-host>
  port: <clickhouse-port>
  secret:
    name: <existing-secret-name>
    userKey: <secret-user-key>
    passwordKey: <secret-password-key>
```

- [ ] **Step 3: Verify files exist**

Run: `Get-ChildItem helm/mcp-clickhouse`
Expected: `Chart.yaml` and `values.yaml` are listed.

- [ ] **Step 4: Commit**

```bash
git add helm/mcp-clickhouse/Chart.yaml helm/mcp-clickhouse/values.yaml
git commit -m "feat(helm): add chart metadata and values"
```

## Task 2: Add Helm Templates (Helpers, Deployment, Service)

**Files:**

- Create: `helm/mcp-clickhouse/templates/_helpers.tpl`
- Create: `helm/mcp-clickhouse/templates/deployment.yaml`
- Create: `helm/mcp-clickhouse/templates/service.yaml`

- [ ] **Step 1: Create helpers template**

```yaml
{{- define "mcp-clickhouse.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "mcp-clickhouse.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s" (include "mcp-clickhouse.name" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}

{{- define "mcp-clickhouse.labels" -}}
app.kubernetes.io/name: {{ include "mcp-clickhouse.name" . }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

- [ ] **Step 2: Create deployment template**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mcp-clickhouse.fullname" . }}
  labels:
    {{- include "mcp-clickhouse.labels" . | nindent 4 }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "mcp-clickhouse.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "mcp-clickhouse.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: mcp-clickhouse
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.container.port }}
              protocol: TCP
          env:
            - name: CLICKHOUSE_HOST
              value: "{{ .Values.clickhouse.host }}"
            - name: CLICKHOUSE_PORT
              value: "{{ .Values.clickhouse.port | toString }}"
            - name: CLICKHOUSE_USER
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.clickhouse.secret.name }}
                  key: {{ .Values.clickhouse.secret.userKey }}
            - name: CLICKHOUSE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.clickhouse.secret.name }}
                  key: {{ .Values.clickhouse.secret.passwordKey }}
            - name: CLICKHOUSE_MCP_AUTH_DISABLED
              value: "{{ .Values.server.authDisabled | toString }}"
          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
```

- [ ] **Step 3: Create service template**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mcp-clickhouse.fullname" . }}
  labels:
    {{- include "mcp-clickhouse.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app.kubernetes.io/name: {{ include "mcp-clickhouse.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
```

- [ ] **Step 4: Verify chart renders**

Run: `helm template mcp-clickhouse helm/mcp-clickhouse`
Expected: Deployment and Service render without errors.

- [ ] **Step 5: Commit**

```bash
git add helm/mcp-clickhouse/templates/_helpers.tpl helm/mcp-clickhouse/templates/deployment.yaml helm/mcp-clickhouse/templates/service.yaml
git commit -m "feat(helm): add deployment and service templates"
```

## Task 3: Add VS Code Tasks for Build, Push, Validate, Deploy

**Files:**

- Create or Modify: `.vscode/tasks.json`

- [ ] **Step 1: Add tasks.json with required tasks**

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "docker: build image (release + sha)",
      "type": "shell",
      "command": "$IMAGE_REPO = \"docker-private.repository.itools.radwarecloud.com/anatolyb/mcp-clickhouse\"; $IMAGE_TAG = if ($env:IMAGE_TAG) { $env:IMAGE_TAG } else { \"1.0.0\" }; $IMAGE_SHA = (git rev-parse --short HEAD).Trim(); docker build -t \"$IMAGE_REPO:$IMAGE_TAG\" -t \"$IMAGE_REPO:$IMAGE_SHA\" .",
      "problemMatcher": []
    },
    {
      "label": "docker: push image (release + sha)",
      "type": "shell",
      "command": "$IMAGE_REPO = \"docker-private.repository.itools.radwarecloud.com/anatolyb/mcp-clickhouse\"; $IMAGE_TAG = if ($env:IMAGE_TAG) { $env:IMAGE_TAG } else { \"1.0.0\" }; $IMAGE_SHA = (git rev-parse --short HEAD).Trim(); docker push \"$IMAGE_REPO:$IMAGE_TAG\"; docker push \"$IMAGE_REPO:$IMAGE_SHA\"",
      "dependsOn": ["docker: build image (release + sha)"],
      "problemMatcher": []
    },
    {
      "label": "helm: lint and template",
      "type": "shell",
      "command": "helm lint helm/mcp-clickhouse; helm template mcp-clickhouse helm/mcp-clickhouse",
      "problemMatcher": []
    },
    {
      "label": "helm: deploy localdev",
      "type": "shell",
      "command": "$IMAGE_REPO = \"docker-private.repository.itools.radwarecloud.com/anatolyb/mcp-clickhouse\"; $IMAGE_TAG = if ($env:IMAGE_TAG) { $env:IMAGE_TAG } else { \"1.0.0\" }; helm upgrade --install mcp-clickhouse helm/mcp-clickhouse --namespace localdev --create-namespace --set image.repository=$IMAGE_REPO --set image.tag=$IMAGE_TAG",
      "dependsOn": ["helm: lint and template"],
      "problemMatcher": []
    },
    {
      "label": "k8s: smoke check localdev",
      "type": "shell",
      "command": "kubectl -n localdev rollout status deployment/mcp-clickhouse; kubectl -n localdev get pods,svc",
      "dependsOn": ["helm: deploy localdev"],
      "problemMatcher": []
    }
  ]
}
```

- [ ] **Step 2: Verify tasks file is valid JSON**

Run: `Get-Content .vscode/tasks.json | ConvertFrom-Json | Out-Null; Write-Output "OK"`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add .vscode/tasks.json
git commit -m "chore(tasks): add docker and helm workflow tasks"
```

## Task 4: Verify Helm Output Contains SecretRefs and Probes

**Files:**

- No file changes (verification only)

- [ ] **Step 1: Render chart output to file**

Run: `helm template mcp-clickhouse helm/mcp-clickhouse > /tmp-mcp-clickhouse-render.yaml`
Expected: File created with rendered manifests.

- [ ] **Step 2: Verify secretKeyRef appears for user/password**

Run: `rg -n "secretKeyRef|<secret-user-key>|<secret-password-key>" /tmp-mcp-clickhouse-render.yaml`
Expected: Matches for both placeholder key names and secretKeyRef blocks.

- [ ] **Step 3: Verify health probes exist**

Run: `rg -n "readinessProbe|livenessProbe|/health" /tmp-mcp-clickhouse-render.yaml`
Expected: Matches for readiness/liveness and `/health` path.

- [ ] **Step 4: Cleanup render artifact**

Run: `Remove-Item /tmp-mcp-clickhouse-render.yaml`
Expected: File removed.

## Task 5: Final Verification and Summary Commit

**Files:**

- No new files required

- [ ] **Step 1: Run lint and template once more**

Run: `helm lint helm/mcp-clickhouse; helm template mcp-clickhouse helm/mcp-clickhouse | Out-Null; Write-Output "HELM_OK"`
Expected: `HELM_OK`

- [ ] **Step 2: Check git status for intended files only**

Run: `git status --short`
Expected: Only chart files and `.vscode/tasks.json` changes (plus any pre-existing unrelated untracked files).

- [ ] **Step 3: Commit any remaining intended changes**

```bash
git add helm/mcp-clickhouse .vscode/tasks.json
git commit -m "feat: add helm chart and deployment tasks"
```

## Spec Coverage Check

- Chart location and minimal structure: Covered by Task 1 + Task 2.
- Secret reference and key selection: Covered by Task 2 + Task 4.
- `/health` probes: Covered by Task 2 + Task 4.
- Build/push release + SHA without latest: Covered by Task 3.
- Deploy to `localdev` via Helm: Covered by Task 3.
- Validation commands (`helm lint`, `helm template`): Covered by Task 2, Task 3, Task 5.

## Placeholder Scan

- No `TBD`/`TODO` placeholders remain.
- Commands and code snippets are concrete.

## Type/Name Consistency

- Release name used as `mcp-clickhouse` consistently in template, deploy task, and smoke check.
- Secret name and key names match the approved design.
- Image repository and tag strategy match approved design (release + SHA, no latest).
