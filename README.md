# Sample GitOps for OpenChoreo

This repository demonstrates how to use OpenChoreo in a GitOps-driven workflow. It includes Flux configurations, workflow definitions, and platform resources to build and deploy sample components using OpenChoreo's CI/CD capabilities.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setting Up the GitOps Repository](#setting-up-the-gitops-repository)
- [Create Git Secrets](#create-git-secrets-in-the-openchoreo-key-vault)
- [Deploy the GitOps Sample](#deploy-the-gitops-sample)
- [Build and Deploy Components](#build-and-deploy-components)
- [Try Out the Sample](#try-out-the-sample)
- [Promote to Staging](#promote-components-to-staging)

---

## Prerequisites

### 1. Install OpenChoreo

Follow the official documentation: [Try it out on k3d locally](https://openchoreo.dev/docs/next/getting-started/try-it-out/on-k3d-locally/)

> [!WARNING]
> Do **not** install the OpenChoreo default resources. Only create the **default dataplane** and **build plane**.

### 2. Install Flux

Follow the [official Flux installation guide](https://fluxcd.io/flux/installation/#dev-install), or run:

```bash
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml
```

---

## Setting Up the GitOps Repository

1. **Fork this repository.**

2. **Update the GitOps repository URL** in the following files to point to your fork, then commit and push the changes to your forked repository:
   - [`flux/gitrepository.yaml`](./flux/gitrepository.yaml) — update the `spec.url` field
   - [`namespaces/default/platform/workflows/docker-with-gitops.yaml`](./namespaces/default/platform/workflows/docker-with-gitops.yaml) — update the `gitops-repo-url` parameter

3. **Generate a GitHub Personal Access Token (PAT)** with read/write access to your forked repository.

---

## Create Git Secrets in the OpenChoreo Key Vault

Store your GitHub PAT in the OpenBao secret store so OpenChoreo workflows can access your repositories:

```bash
# Secret for cloning source repositories
kubectl exec -n openbao openbao-0 -- bao kv put secret/git-token git-token=<your_github_pat>

# Secret for pushing to and creating PRs in the GitOps repository
kubectl exec -n openbao openbao-0 -- bao kv put secret/gitops-token git-token=<your_github_pat>
```

Replace `<your_github_pat>` with your actual token.

---

## Deploy the GitOps Sample

Apply the Flux resources to start syncing this repository with your cluster:

```bash
kubectl apply -f flux/
```

Flux will now watch this repository and apply any changes to your cluster automatically.

---

## Build and Deploy Components

Trigger the build and release workflows for each component in the **Doclet** sample application. Each `WorkflowRun` builds a container image and creates a pull request in your GitOps repository targeting the **development** environment.

### Document Service

```bash
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: document-svc-manual-01
  namespace: default
  labels:
    openchoreo.dev/project: "doclet"
    openchoreo.dev/component: "document-svc"
spec:
  workflow:
    name: docker-gitops-release
    parameters:
      componentName: document-svc
      projectName: doclet
      docker:
        context: /project-doclet-app/service-go-document
        filePath: /project-doclet-app/service-go-document/Dockerfile
      repository:
        appPath: /project-doclet-app/service-go-document
        revision:
          branch: main
          commit: ""
        url: https://github.com/openchoreo/sample-workloads.git
      workloadDescriptorPath: workload.yaml
EOF
```

### Collaboration Service

```bash
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: collab-svc-manual-01
  namespace: default
  labels:
    openchoreo.dev/project: "doclet"
    openchoreo.dev/component: "collab-svc"
spec:
  workflow:
    name: docker-gitops-release
    parameters:
      componentName: collab-svc
      projectName: doclet
      docker:
        context: /project-doclet-app/service-go-collab
        filePath: /project-doclet-app/service-go-collab/Dockerfile
      repository:
        appPath: /project-doclet-app/service-go-collab
        revision:
          branch: main
          commit: ""
        url: https://github.com/openchoreo/sample-workloads.git
      workloadDescriptorPath: workload.yaml
EOF
```

### Frontend

```bash
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: frontend-workflow-manual-01
  namespace: default
  labels:
    openchoreo.dev/project: "doclet"
    openchoreo.dev/component: "frontend"
spec:
  workflow:
    name: docker-gitops-release
    parameters:
      componentName: frontend
      projectName: doclet
      docker:
        context: /project-doclet-app/webapp-react-frontend
        filePath: /project-doclet-app/webapp-react-frontend/Dockerfile
      repository:
        appPath: /project-doclet-app/webapp-react-frontend
        revision:
          branch: main
          commit: ""
        url: https://github.com/openchoreo/sample-workloads.git
      workloadDescriptorPath: workload.yaml
EOF
```


> [!NOTE]
> The source code for the Doclet sample application is available at [openchoreo/sample-workloads](https://github.com/openchoreo/sample-workloads/tree/main/project-doclet-app).

### Merge the Pull Requests

Once all three workflows complete, **3 pull requests** will be created in your forked GitOps repository — one for each component. Review and merge them, then wait for Flux to sync and deploy the components to your cluster.

---

## Try Out the Sample

Once Flux has synced the merged changes, the Doclet application components will be running in your cluster. You can explore the deployed services and frontend through the OpenChoreo platform.

---

## Promote Components to Staging

After validating in the development environment, promote the entire **Doclet** project to staging using the bulk release workflow:

```bash
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: bulk-release-manual-01
  namespace: default
spec:
  workflow:
    name: bulk-gitops-release
    parameters:
      scope:
        all: false
        projectName: "doclet"
      gitops:
        repositoryUrl: "https://github.com/<your-github-username>/sample-gitops"
        branch: "main"
        targetEnvironment: "staging"
        deploymentPipeline: "standard"
EOF
```

Replace `<your-github-username>` with your GitHub username. Once the workflow completes, a pull request will be created in your forked GitOps repository to promote all Doclet components from development to staging in a single operation. Merge the PR and wait for Flux to sync the changes to your cluster.

