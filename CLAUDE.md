# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a GitOps demo repository showcasing Red Hat Advanced Cluster Management for Kubernetes (RHACM) with ArgoCD. It demonstrates multi-cluster deployment patterns across hub and managed clusters in different environments (dev, qa, uat, prod).

## Architecture

### Two-Tier GitOps Structure

The repository uses a hierarchical "app-of-apps" pattern with two main layers:

1. **Platform Layer** (`gitops/platform/`): Infrastructure and platform components
   - Hub cluster configuration (ArgoCD, ACM policies, RBAC)
   - Managed cluster configuration (deployed via ApplicationSets)
   - Uses `openshift-gitops` namespace

2. **Shared Applications Layer** (`gitops/shared/`): Application deployments
   - Demo applications (e.g., rollouts-demo)
   - Uses `shared-gitops` namespace

### ACM Cluster Organization

**ManagedClusterSets**: Clusters are organized into sets based on environment:
- `hub-clusters`: Hub cluster itself
- `dev-clusters`, `qa-clusters`, `uat-clusters`, `prod-clusters`: Managed clusters by environment

**Placements**: Define which clusters receive which manifests:
- `hub-only-placement`: Targets hub-clusters set
- `{env}-only-placement`: Targets specific environment cluster sets
- `all-managed-placement`: Targets all managed clusters (dev, qa, uat, prod)

**Label Requirements**: When importing managed clusters into ACM, they must have:
- `env` label: One of `dev`, `qa`, `uat`, or `prod`
- Cluster must be added to corresponding ClusterSet

### ApplicationSet Patterns

**Push Model (dev)**: ArgoCD on hub pushes directly to managed clusters
- Standard ApplicationSet without ACM annotations
- Example: `demo-rollouts-dev`

**Pull Model (qa, uat, prod)**: Applications are pulled to managed cluster's local ArgoCD
- Requires ACM annotations:
  ```yaml
  annotations:
    apps.open-cluster-management.io/ocm-managed-cluster: "{{name}}"
    apps.open-cluster-management.io/ocm-managed-cluster-app-namespace: shared-gitops
    argocd.argoproj.io/skip-reconcile: "true"
  labels:
    apps.open-cluster-management.io/pull-to-ocm-managed-cluster: "true"
  ```
- Example: `demo-rollouts-qa`, `demo-rollouts-uat`, `demo-rollouts-prod`

### Environment-Specific Configuration

Each environment uses different banner colors (configured via Helm parameters):
- dev: blue
- qa: purple
- uat: orange
- prod: red

## Common Commands

### Initial Setup

```bash
# Deploy platform layer (must be logged in as cluster admin)
oc apply -f gitops/platform/app-of-apps/applications.yaml

# Deploy shared applications layer (can be logged in as developer)
oc apply -f gitops/shared/app-of-apps/applications.yaml
```

### ArgoCD Access

```bash
# Get hub cluster ArgoCD URL (openshift-gitops namespace)
echo "https://$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')"

# Get shared ArgoCD URL (shared-gitops namespace)
echo "https://$(oc get route shared-gitops-server -n shared-gitops -o jsonpath='{.spec.host}')"
```

### Argo Rollouts

```bash
# Access Argo Rollouts dashboard (must be on target cluster)
oc argo rollouts dashboard
```

### ACM and Cluster Management

```bash
# Give ArgoCD cluster-admin access (required during setup)
oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller --rolebinding-name gitops-role-binding
```

## Key Implementation Details

### Progressive Delivery (Production)

Production environment uses Argo Rollouts for canary deployments:
- Canary strategy with traffic splitting: 20% → 40% → 60% → 80% → 100%
- Manual approval required at 20% (pause step)
- Automatic progression with 10s pauses between subsequent steps
- Uses OpenShift Route plugin for traffic routing

### Sync Waves

ArgoCD sync waves control deployment order:
- Wave 0: Hub platform manifests
- Wave 1: Managed cluster ApplicationSets
- Wave 3: Application deployments

### Helm Usage

Platform manifests for managed clusters use Helm charts with parameters:
- `managedservername`: Cluster name (from ACM placement)
- `bannercolor`: Environment-specific color
- `serverurl`: Kubernetes API server URL

### Known Issues

On UAT cluster, Gatekeeper instance must be manually created after GitOps deployment. Navigate to "Ecosystem → Installed operators → Gatekeeper operator" and create instance with default values.

## Repository Structure

```
gitops/
├── platform/              # Platform-level configuration
│   ├── app-of-apps/       # Root Application for platform layer
│   ├── argo-apps/         # Platform Applications and ApplicationSets
│   └── platform-manifests/
│       ├── hub/           # Hub cluster Helm chart
│       └── managed/       # Managed cluster Helm chart
└── shared/                # Application-level configuration
    ├── app-of-apps/       # Root Application for shared layer
    ├── argo-apps/         # Application ApplicationSets
    └── demo-apps/         # Demo application manifests
        └── rollouts-demo/ # Kustomize overlays per environment
```

## Typical Workflows

### Adding a New Managed Cluster

1. Import cluster via ACM console ("Infrastructure → Clusters → Import Cluster")
2. Add cluster to appropriate ClusterSet (dev, qa, uat, or prod)
3. Apply `env` label matching the ClusterSet
4. Wait 5-10 minutes for ACM policies and GitOps manifests to sync

### Deploying a New Application

Create ApplicationSet in `gitops/shared/argo-apps/` following the pattern:
- Use `clusterDecisionResource` generator with `acm-placement` configMapRef
- Match on appropriate placement label
- Use pull model annotations for qa/uat/prod environments
- Use push model (no annotations) for dev environment

### Modifying Platform Configuration

Platform changes go through the hub Application:
- Hub-specific: Modify `gitops/platform/platform-manifests/hub/`
- Managed clusters: Modify `gitops/platform/platform-manifests/managed/`
- Changes sync automatically via ArgoCD
