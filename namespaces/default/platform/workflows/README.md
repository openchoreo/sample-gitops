# GitOps Workflows

This directory contains OpenChoreo Workflow definitions that automate building, releasing, and promoting components through a GitOps-driven CI/CD pipeline.

## Available Workflows

| Workflow | Purpose | Build Method |
|---|---|---|
| [docker-gitops-release](#docker-gitops-release) | Build from Dockerfile and release via GitOps | Docker (Podman) |
| [google-cloud-buildpacks-gitops-release](#google-cloud-buildpacks-gitops-release) | Build without Dockerfile and release via GitOps | Google Cloud Buildpacks |
| [react-gitops-release](#react-gitops-release) | Build React/SPA app and release via GitOps | Node.js + nginx |
| [bulk-gitops-release](#bulk-gitops-release) | Promote existing releases to a target environment | N/A (no build) |

## How Build & Release Workflows Work

All three build-and-release workflows follow the same two-phase pattern:

**Build Phase:**
1. Clone source repository (supports private repos via git token)
2. Build container image (method varies by workflow)
3. Push image to container registry
4. Extract workload descriptor from source repo

**Release Phase:**
5. Clone GitOps repository
6. Create feature branch (`release/<component>-<timestamp>`)
7. Generate GitOps resources (Workload, ComponentRelease, ReleaseBinding) using `occ` CLI
8. Commit, push, and create a pull request

Merging the PR triggers Flux to sync the changes, deploying the component to the target environment.

## Configuration

Before using these workflows, update the `gitops-repo-url` parameter in each workflow file to point to your GitOps repository fork.

---

## docker-gitops-release

Use when your source repository has a **Dockerfile**.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `componentName` | string | yes | | Component name |
| `projectName` | string | yes | | Project name |
| `repository.url` | string | yes | | Source Git repository URL |
| `repository.revision.branch` | string | no | `main` | Git branch |
| `repository.revision.commit` | string | yes | | Git commit SHA |
| `repository.appPath` | string | no | `.` | Path to app directory in repo |
| `docker.context` | string | no | `.` | Docker build context path |
| `docker.filePath` | string | no | `./Dockerfile` | Dockerfile path |
| `workloadDescriptorPath` | string | no | `workload.yaml` | Path to workload descriptor |

**Example:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: greeter-build-001
  namespace: default
spec:
  workflow:
    name: docker-gitops-release
    kind: Workflow
    parameters:
      componentName: greeter-service
      projectName: demo-project
      repository:
        url: https://github.com/openchoreo/sample-workloads.git
        revision:
          branch: main
          commit: "abc1234"
        appPath: /service-go-greeter
      docker:
        context: /service-go-greeter
        filePath: /service-go-greeter/Dockerfile
      workloadDescriptorPath: workload.yaml
```

---

## google-cloud-buildpacks-gitops-release

Use when your source repository **does not have a Dockerfile**. Buildpacks auto-detect the language and build a container image.

Supported languages: Go, Java (Maven/Gradle), Node.js, Python, .NET Core, Ruby, PHP.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `componentName` | string | yes | | Component name |
| `projectName` | string | yes | | Project name |
| `repository.url` | string | yes | | Source Git repository URL |
| `repository.revision.branch` | string | no | `main` | Git branch |
| `repository.revision.commit` | string | yes | | Git commit SHA |
| `repository.appPath` | string | no | `.` | Path to app directory in repo |
| `buildpacks.builderImage` | string | no | `gcr.io/buildpacks/builder:v1` | Buildpacks builder image |
| `buildpacks.env` | string[] | no | `[]` | Build-time env vars (`KEY=VALUE`) |
| `workloadDescriptorPath` | string | no | `workload.yaml` | Path to workload descriptor |

**Example:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: reading-list-build-001
  namespace: default
spec:
  workflow:
    name: google-cloud-buildpacks-gitops-release
    kind: Workflow
    parameters:
      componentName: reading-list-service
      projectName: demo-project
      repository:
        url: https://github.com/openchoreo/sample-workloads.git
        revision:
          branch: main
          commit: "abc1234"
        appPath: /service-go-reading-list
      buildpacks:
        builderImage: gcr.io/buildpacks/builder:v1
        env: []
      workloadDescriptorPath: workload.yaml
```

---

## react-gitops-release

Use for **React and SPA** applications. Builds the app with Node.js, packages it into an nginx container with SPA routing configured.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `componentName` | string | yes | | Component name |
| `projectName` | string | yes | | Project name |
| `repository.url` | string | yes | | Source Git repository URL |
| `repository.revision.branch` | string | no | `main` | Git branch |
| `repository.revision.commit` | string | yes | | Git commit SHA |
| `repository.appPath` | string | no | `.` | Path to app directory in repo |
| `react.nodeVersion` | string | no | `18` | Node.js version (`16`, `18`, `20`, `22`) |
| `react.buildCommand` | string | no | `npm run build` | Build command |
| `react.outputDir` | string | no | `build` | Build output directory |
| `workloadDescriptorPath` | string | no | `workload.yaml` | Path to workload descriptor |

**Example:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: frontend-build-001
  namespace: default
spec:
  workflow:
    name: react-gitops-release
    kind: Workflow
    parameters:
      componentName: frontend
      projectName: demo-project
      repository:
        url: https://github.com/openchoreo/sample-workloads.git
        revision:
          branch: main
          commit: "abc1234"
        appPath: /webapp-react-frontend
      react:
        nodeVersion: "20"
        buildCommand: "npm run build"
        outputDir: build
      workloadDescriptorPath: workload.yaml
```

---

## bulk-gitops-release

Use to **promote existing releases** to a target environment. Does not build anything — generates ReleaseBindings for components that already have ComponentReleases.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `scope.all` | boolean | no | `false` | Promote all projects |
| `scope.projectName` | string | yes | | Target project (required even when `all: true`) |
| `gitops.repositoryUrl` | string | yes | | GitOps repository URL |
| `gitops.branch` | string | no | `main` | GitOps branch |
| `gitops.targetEnvironment` | string | no | `development` | Target environment |
| `gitops.deploymentPipeline` | string | yes | | Deployment pipeline name |

**Example — promote a specific project to staging:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: promote-doclet-staging-001
  namespace: default
spec:
  workflow:
    name: bulk-gitops-release
    kind: Workflow
    parameters:
      scope:
        all: false
        projectName: doclet
      gitops:
        repositoryUrl: "https://github.com/<your-org>/sample-gitops"
        branch: main
        targetEnvironment: staging
        deploymentPipeline: standard
```

**Example — promote all projects to production:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: WorkflowRun
metadata:
  name: promote-all-prod-001
  namespace: default
spec:
  workflow:
    name: bulk-gitops-release
    kind: Workflow
    parameters:
      scope:
        all: true
        projectName: placeholder
      gitops:
        repositoryUrl: "https://github.com/<your-org>/sample-gitops"
        branch: main
        targetEnvironment: production
        deploymentPipeline: standard
```

---

## Secrets

These workflows require git tokens stored in a `ClusterSecretStore`:

| Secret Key | Required By | Purpose |
|---|---|---|
| `git-token` | Build & release workflows | Clone private source repos |
| `gitops-token` | All workflows | Push branches and create PRs in GitOps repo |

See the [root README](../../../../README.md#create-git-secrets-in-the-openchoreo-key-vault) for setup instructions.