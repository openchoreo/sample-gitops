# GitOps Workflows

This directory contains OpenChoreo Workflow definitions that automate building, releasing, and promoting components through a GitOps-driven CI/CD pipeline.

## Available Workflows overview

| Workflow | Purpose | Build Method |
|---|---|---|
| [docker-gitops-release](#docker-gitops-release) | Build from Dockerfile and release via GitOps | Docker (Podman) |
| [google-cloud-buildpacks-gitops-release](#google-cloud-buildpacks-gitops-release) | Build without Dockerfile and release via GitOps | Google Cloud Buildpacks |
| [react-gitops-release](#react-gitops-release) | Build React/SPA app and release via GitOps | Node.js + nginx |
| [bulk-gitops-release](#bulk-gitops-release) | Promote existing releases to a target environment | N/A (no build) |

> [!NOTE]  
> To learn more about how OpenChoreo workflows work, please refer to the [documentation](https://openchoreo.dev/).

## Build and Release Workflows

The following build-and-release workflows automate the build and deployment of OpenChoreo components in a GitOps setup:

1. [docker-gitops-release](#docker-gitops-release)
2. [google-cloud-buildpacks-gitops-release](#google-cloud-buildpacks-gitops-release)
3. [react-gitops-release](#react-gitops-release)

All three workflows follow the same high-level pattern. The main difference is how the container image is built.

**Build phase**
1. Clone the source repository (private repositories are supported through a git token).
2. Build the container image using the workflow-specific method.
3. Push the image to the container registry.
4. Extract the workload descriptor from the source repository.

**Release phase**
1. Clone the GitOps repository.
2. Create a feature branch (`release/<component>-<timestamp>`).
3. Generate the GitOps resources (`Workload`, `ComponentRelease`, and `ReleaseBinding`) using the `occ` CLI.
4. Commit the changes, push the branch, and create a pull request.

Merging the PR triggers the CD tool to sync the changes, deploying the component to the target environment.

> [!TIP]
> These workflows require git tokens stored in a `ClusterSecretStore`:
>
> | Secret Key | Required By | Purpose |
> |---|---|---|
> | `git-token` | Build & release workflows | Clone private source repos |
> | `gitops-token` | All workflows | Push branches and create PRs in GitOps repo |

### Configuration

Before using the build-and-release workflows, update the hardcoded `gitops-repo-url` value in each workflow definition so it points to your GitOps repository fork.

### Running the Workflows

The YAML snippets below are runnable `WorkflowRun` manifests, not just reference examples. Save a manifest to a file, update the parameter values for your component, and apply it with `kubectl apply -f <file>`. 

---

#### docker-gitops-release

Workflow definition: [`docker-with-gitops-release.yaml`](./docker-with-gitops-release.yaml)

Use when your source repository has a **Dockerfile**.

**Parameters:**

| Parameter                    | Type   | Required | Default         | Description                                                      |
|------------------------------|--------|----------|-----------------|------------------------------------------------------------------|
| `componentName`              | string | yes      |                 | Component name                                                   |
| `projectName`                | string | yes      |                 | Project name                                                     |
| `repository.url`             | string | yes      |                 | Source repository URL                                            |
| `repository.revision.branch` | string | no       | `main`          | Source repository branch to check out                            |
| `repository.revision.commit` | string | yes      |                 | Source repository Git commit SHA or reference                    |
| `repository.appPath`         | string | no       | `.`             | Application path within the source repository                    |
| `docker.context`             | string | no       | `.`             | Docker build context relative to the source repository root      |
| `docker.filePath`            | string | no       | `./Dockerfile`  | Dockerfile path relative to the source repository root           |
| `workloadDescriptorPath`     | string | no       | `workload.yaml` | Path to the workload descriptor relative to `repository.appPath` |

**WorkflowRun manifest:**

Save the following manifest as `docker-gitops-release-run.yaml`, update the repository, commit, and path values for your component, and run `kubectl apply -f docker-gitops-release-run.yaml`.

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

#### google-cloud-buildpacks-gitops-release

Workflow definition: [`google-cloud-buildpacks-gitops-release.yaml`](./google-cloud-buildpacks-gitops-release.yaml)

Use when your source repository **does not have a Dockerfile**. Buildpacks auto-detect the language and build a container image.

Supported languages: Go, Java (Maven/Gradle), Node.js, Python, .NET Core, Ruby, PHP.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `componentName` | string | yes | | Component name |
| `projectName` | string | yes | | Project name |
| `repository.url` | string | yes | | Source repository URL |
| `repository.revision.branch` | string | no | `main` | Branch to check out |
| `repository.revision.commit` | string | yes | | Git commit SHA or reference |
| `repository.appPath` | string | no | `.` | Application path within the repository |
| `buildpacks.builderImage` | string | no | `gcr.io/buildpacks/builder:v1` | Buildpacks builder image |
| `buildpacks.env` | string[] | no | `[]` | Build-time environment variables in `KEY=VALUE` format |
| `workloadDescriptorPath` | string | no | `workload.yaml` | Path to the workload descriptor relative to `repository.appPath` |

**Example WorkflowRun manifest:**

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

#### react-gitops-release

Workflow definition: [`react-gitops-release.yaml`](./react-gitops-release.yaml)

Use for **React and SPA** applications. Builds the app with Node.js, packages it into an nginx container with SPA routing configured.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `componentName` | string | yes | | Component name |
| `projectName` | string | yes | | Project name |
| `repository.url` | string | yes | | Source repository URL |
| `repository.revision.branch` | string | no | `main` | Branch to check out |
| `repository.revision.commit` | string | yes | | Git commit SHA or reference |
| `repository.appPath` | string | no | `.` | Application path within the repository |
| `react.nodeVersion` | string | no | `18` | Node.js version (`16`, `18`, `20`, `22`) |
| `react.buildCommand` | string | no | `npm run build` | Build command |
| `react.outputDir` | string | no | `build` | Frontend build output directory |
| `workloadDescriptorPath` | string | no | `workload.yaml` | Path to the workload descriptor relative to `repository.appPath` |

**Example WorkflowRun manifest:**

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

## Promotion Workflow

This sample workflow automates the promotion of components from one environment to another in an OpenChoreo GitOps setup.

> [!TIP]
> This workflow requires a `gitops-token` stored in a `ClusterSecretStore` to push branches and create PRs in the GitOps repository.

### bulk-gitops-release

Workflow definition: [`bulk-gitops-release.yaml`](./bulk-gitops-release.yaml)

Use to **promote existing releases** to a target environment. Does not build anything — generates ReleaseBindings for components that already have ComponentReleases.
This workflow can be used to promote all components in a namespace or all components in a specific project.

**Parameters:**

| Parameter                   | Type    | Required | Default       | Description                                                               |
|-----------------------------|---------|----------|---------------|---------------------------------------------------------------------------|
| `scope.all`                 | boolean | no       | `false`       | Promote all projects                                                      |
| `scope.projectName`         | string  | yes      |               | Project name to promote; still required as a placeholder when `all: true` |
| `gitops.repositoryUrl`      | string  | yes      |               | GitOps repository URL                                                     |
| `gitops.branch`             | string  | no       | `main`        | GitOps repository branch                                                  |
| `gitops.targetEnvironment`  | string  | no       | `development` | Target environment name                                                   |
| `gitops.deploymentPipeline` | string  | yes      |               | Deployment pipeline name                                                  |

**Example WorkflowRun manifest for promoting a single project to staging:**

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

**Example WorkflowRun manifest for promoting all components**

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
