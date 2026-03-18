# Cluster Workflow Templates

This directory contains [Argo Workflows ClusterWorkflowTemplates](https://argo-workflows.readthedocs.io/en/latest/cluster-workflow-templates/) that implement the step-by-step logic for OpenChoreo GitOps workflows.

## Templates

| Template | Steps | Used By |
|---|---|---|
| `docker-gitops-release` | 8 | `docker-gitops-release` Workflow |
| `google-cloud-buildpacks-gitops-release` | 8 | `google-cloud-buildpacks-gitops-release` Workflow |
| `react-gitops-release` | 9 | `react-gitops-release` Workflow |
| `bulk-gitops-release` | 4 | `bulk-gitops-release` Workflow |

## Relationship to Workflows

Each [Workflow CR](../../../namespaces/default/platform/workflows/) references a ClusterWorkflowTemplate via `workflowTemplateRef`:

```
Workflow CR (parameters, secrets, schema)
  └── references → ClusterWorkflowTemplate (step execution logic)
```

- **Workflow CRs** define the parameter schema, external secrets, and namespace configuration. Platform engineers customize these per environment.
- **ClusterWorkflowTemplates** contain the actual build/release steps (clone, build, push, generate manifests, create PR). These are cluster-scoped and shared across namespaces.

## Container Images Used

| Image | Purpose |
|---|---|
| `alpine/git` | Git clone, branch, commit, push operations |
| `ghcr.io/openchoreo/podman-runner:v1.0` | Docker image build and push (Podman) |
| `ghcr.io/openchoreo/buildpacks-runner:v1.0` | Google Cloud Buildpacks build |
| `ghcr.io/openchoreo/openchoreo-cli:latest-dev` | `occ` CLI for generating Workload, ComponentRelease, ReleaseBinding |
| `node:<version>-alpine` | React application build |
| `alpine:latest` | Workload descriptor extraction |