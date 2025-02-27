
- Supports defining multiple ResourceQuota objects via a single "quotas" array in the values file.
- Enforces that each quota must specify its target namespace.
- Strictly auto‑generates the quota name as “<namespace>-quota” (i.e. no manual override for the name is allowed).
- Validates the values using a JSON Schema.
- Organizes environment‑specific values (one values file per environment, for example, rnd, qa, uat, prod) so that you can create quotas in different namespaces.
- Includes an example Argo CD Application manifest that demonstrates how the chart is deployed using the environment values file.

---

# Guide: Reusable Helm Chart for ResourceQuotas  
**(Strict Namespace Enforcement, Auto‑Generated Names, JSON Schema Validation, and Argo CD Support)**

This guide explains how to create a Helm chart that:
  
1. Defines multiple ResourceQuota objects via a `quotas` array.
2. Renders a single YAML file with all ResourceQuotas separated by document delimiters (`---`).
3. Enforces that each quota entry must include a non‑empty target namespace, and auto‑generates its name strictly as `<namespace>-quota` (i.e. no override allowed).
4. Validates your values using a JSON Schema.
5. Uses distinct values files for each environment (rnd, qa, uat, prod) to support quotas in different namespaces.
6. Supports deployment via Argo CD (with an explicit Argo CD path) while understanding that explicit namespaces in the rendered manifest are authoritative.

---

## 1. Chart Structure

Your chart repository (named “quota‑chart”) should be organized as follows:

```
quota-chart/
├── Chart.yaml
├── values.yaml               # Default values (fallback)
├── values.schema.json        # JSON Schema for validating values.yaml
├── env/
│   ├── rnd/
│   │   └── values.yaml       # Overrides for RND
│   ├── qa/
│   │   └── values.yaml       # Overrides for QA
│   ├── uat/
│   │   └── values.yaml       # Overrides for UAT
│   └── prod/
│       └── values.yaml       # Overrides for Production
└── templates/
    └── resourcequota.yaml    # Template that creates ResourceQuota objects
```

*Note:* In this design, there is one distinct values file per environment that can define multiple quota stanzas (each with its own target namespace).

---

## 2. Define the Chart Files

### A. Chart.yaml

```yaml
apiVersion: v2
name: quota-chart
description: A reusable Helm chart to create one or more Kubernetes ResourceQuotas with strict auto‑generated names.
version: 0.1.0
appVersion: "1.0.0"
```

### B. Default values.yaml

Since quota names are auto‑generated, each quota object only needs to specify its target namespace and its "hard" resource limits:

```yaml
quotas:
  - namespace: default
    hard:
      pods: "10"
      cpu: "4"
      memory: "16Gi"
```

### C. JSON Schema: values.schema.json

Create a file named `values.schema.json` to validate that your values file contains a required `quotas` array. Each quota object must include a non‑empty `namespace` and a required `hard` map. (No "name" is allowed.)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Quota Chart Values Schema",
  "type": "object",
  "properties": {
    "quotas": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "namespace": {
            "type": "string",
            "minLength": 1,
            "description": "Target namespace for this quota. Must not be empty."
          },
          "hard": {
            "type": "object",
            "description": "Map of resource names (e.g., pods, cpu, memory) to quota limits as strings.",
            "patternProperties": {
              "^[a-zA-Z0-9./_-]+$": { "type": "string", "minLength": 1 }
            },
            "minProperties": 1,
            "additionalProperties": false
          }
        },
        "required": ["namespace", "hard"],
        "additionalProperties": false
      }
    }
  },
  "required": ["quotas"],
  "additionalProperties": false
}
```

### D. Template: resourcequota.yaml

Create or update `quota-chart/templates/resourcequota.yaml`. This template iterates over the `quotas` array and creates one ResourceQuota per entry. It uses the quota’s specified namespace (or defaults to `.Release.Namespace`) for both the `metadata.namespace` and auto‑generates the quota name as `<target-namespace>-quota`.

```yaml
{{- /*
  This template creates multiple ResourceQuota objects from the "quotas" array.
  Each quota object must include:
    - "namespace": the target namespace (required)
    - "hard": a map of resource limits (required)
  The quota name is strictly auto-generated as "<target-namespace>-quota".
*/ -}}
{{- range $index, $quota := .Values.quotas }}
  {{- $targetNamespace := default .Release.Namespace $quota.namespace }}
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {{ printf "%s-quota" $targetNamespace }}
  namespace: {{ $targetNamespace }}
spec:
  hard:
{{ toYaml $quota.hard | nindent 4 }}
---
{{- end }}
```

*Note:* This template produces a single rendered YAML file containing multiple documents separated by `---`.

---

## 3. Environment-Specific Values Files

Each environment’s values file can define one or more quota stanzas to support creating quotas in different namespaces. For example:

### **env/rnd/values.yaml:**

```yaml
quotas:
  - namespace: rnd
    hard:
      pods: "12"
      cpu: "5"
      memory: "20Gi"
  - namespace: logging
    hard:
      pods: "8"
      cpu: "4"
      memory: "16Gi"
```

### **env/qa/values.yaml:**

```yaml
quotas:
  - namespace: qa
    hard:
      pods: "15"
      cpu: "6"
      memory: "24Gi"
  - namespace: staging
    hard:
      pods: "10"
      cpu: "5"
      memory: "18Gi"
```

### **env/uat/values.yaml:**

```yaml
quotas:
  - namespace: uat
    hard:
      pods: "20"
      cpu: "8"
      memory: "32Gi"
  - namespace: preprod
    hard:
      pods: "12"
      cpu: "6"
      memory: "24Gi"
```

### **env/prod/values.yaml:**

```yaml
quotas:
  - namespace: prod
    hard:
      pods: "30"
      cpu: "12"
      memory: "64Gi"
  - namespace: critical
    hard:
      pods: "12"
      cpu: "6"
      memory: "24Gi"
  - namespace: monitoring
    hard:
      pods: "8"
      cpu: "4"
      memory: "16Gi"
```

In each file, the quota’s name will be auto‑generated strictly as `<namespace>-quota`. For example, in the prod file the quotas will be named “prod-quota”, “critical-quota”, and “monitoring-quota”.

---

## 4. Deploying via Helm CLI

To deploy the chart manually, use the `-f` flag to specify the appropriate environment-specific values file. For example, to deploy the Production environment configuration:

```bash
helm install my-release ./quota-chart -f env/prod/values.yaml --namespace argocd
```

*Note:* Even though you pass `--namespace argocd` (which is the Argo CD Application’s destination), the rendered YAML explicitly sets `metadata.namespace` (e.g., "prod", "critical", "monitoring") based on your values file. Kubernetes creates the ResourceQuota objects in those namespaces.

---

## 5. Argo CD Support for Multiple Values Files

Below is an example Argo CD Application manifest for the Production environment. In this example, the Application’s destination namespace is set to “argocd” (for Argo CD’s own objects), but the chart’s rendered YAML specifies the target namespaces (from the values file) for each ResourceQuota.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: quota-chart-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-org/quota-chart.git'
    targetRevision: HEAD
    path: quota-chart
    helm:
      valueFiles:
        - env/prod/values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated: {}
```

**Reminder:**  
Since the rendered YAML explicitly sets each ResourceQuota’s namespace (e.g., "prod", "critical", "monitoring"), those objects are created in their specified namespaces regardless of the Argo CD destination namespace.

---

## 6. Verify the Deployment

After deployment (via Helm CLI or Argo CD), verify that the ResourceQuota objects are created in the correct target namespaces. For example, for Production:

```bash
kubectl get resourcequotas -n prod
kubectl describe resourcequota prod-quota -n prod

kubectl get resourcequotas -n critical
kubectl describe resourcequota critical-quota -n critical

kubectl get resourcequotas -n monitoring
kubectl describe resourcequota monitoring-quota -n monitoring
```

The output should confirm that each quota is strictly named as `<namespace>-quota` and created in its respective namespace with the defined resource limits.

---

## 7. Summary

This guide demonstrated how to:

1. Create a reusable Helm chart that defines a `quotas` array—each element representing a ResourceQuota for a specific namespace.
2. Enforce configuration using a JSON Schema that requires each quota to include a non‑empty `namespace` and a `hard` resource limit map, with no manual name override.
3. Use distinct values files per environment (e.g., rnd, qa, uat, prod) that can include multiple quota stanzas, supporting quotas in different namespaces.
4. Strictly auto‑generate each ResourceQuota’s name as “<namespace>-quota.”
5. Deploy the chart via Helm CLI or via Argo CD using the appropriate environment-specific values file.
6. Understand that explicit namespaces in the rendered YAML take precedence over the Argo CD destination namespace.
# Addition to the guide section 4

## 8. Updating Existing Quotas

When decreasing quota limits in environments with running workloads:

- Kubernetes won't evict or terminate existing resources that exceed the new quota
- New resources that would exceed the quota will be rejected
- Before reducing quotas, check current usage with:
  kubectl get resourcequota <namespace>-quota -n <namespace> --output=yaml

To gracefully reduce quotas:
1. Check current usage across all affected namespaces
2. If needed, scale down workloads first
3. Then apply the reduced quota
4. Use a gradual approach for production environments
