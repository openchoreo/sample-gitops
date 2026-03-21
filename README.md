# Sample GitOps for OpenChoreo

This repository demonstrates how to use [OpenChoreo](https://openchoreo.dev/) in a GitOps-driven workflow. It includes workflow definitions, platform resources, and CD tool configurations to build and deploy sample components using OpenChoreo's CI/CD capabilities.

## Table of Contents

- [Repository Structure](#repository-structure)
- [Available Workflows for Automation](#available-workflows-for-automation)
- [Key Concepts](#key-concepts)
- [Tutorials](#tutorials)

---

## Repository Structure

The repository separates platform-level resources from application resources:

```
.
├── flux/                                    # Flux CD configuration
│   ├── gitrepository.yaml                  # Repository pointer
│   ├── namespaces-kustomization.yaml       # Namespace syncing
│   ├── platform-shared-kustomization.yaml  # Cluster resources
│   ├── oc-demo-platform-kustomization.yaml # Platform resources
│   └── oc-demo-projects-kustomization.yaml # Application resources
├── platform-shared/                        # Cluster-scoped resources
│   └── cluster-workflow-templates/
│       └── argo/
│           ├── docker-with-gitops-release.yaml
│           ├── google-cloud-buildpacks-gitops-release-template.yaml
│           ├── react-gitops-release-template.yaml
│           └── bulk-gitops-release-template.yaml
└── namespaces/                             # Namespace-scoped resources
    └── <namespace>/
        ├── namespace.yaml
        ├── platform/                       # Platform team resources
        │   ├── infra/
        │   │   ├── deployment-pipelines/
        │   │   │   └── standard.yaml
        │   │   └── environments/
        │   │       ├── development.yaml
        │   │       ├── staging.yaml
        │   │       └── production.yaml
        │   ├── component-types/
        │   │   ├── service.yaml
        │   │   ├── webapp.yaml
        │   │   ├── database.yaml
        │   │   └── message-broker.yaml
        │   ├── traits/
        │   │   ├── persistent-volume.yaml
        │   │   ├── api-management.yaml
        │   │   └── observability-alert-rule.yaml
        │   └── workflows/
        │       ├── bulk-gitops-release.yaml
        │       ├── docker-with-gitops.yaml
        │       ├── google-cloud-buildpacks-gitops-release.yaml
        │       └── react-gitops-release.yaml
        └── projects/                       # Development team resources
            └── <project-name>/
                ├── project.yaml
                └── components/
                    └── <component-name>/
                        ├── component.yaml
                        ├── workload.yaml
                        ├── releases/
                        │   └── <component>-<date>-<revision>.yaml
                        └── release-bindings/
                            ├── <component>-development.yaml
                            └── <component>-staging.yaml
```

**Key notes:**

- The `platform/` directory syncs first, ensuring Environments, DataPlanes, and ComponentTypes exist before Components. The CD tool enforces ordering through dependency configuration.
- The `platform-shared/` directory contains cluster-scoped resources such as Argo ClusterWorkflowTemplates. In production setups, this would also include ClusterComponentTypes, ClusterTraits, ClusterDataPlanes, and ClusterAuthzRoles.

---

## Available Workflows for Automation

This repository includes OpenChoreo Workflow definitions that automate the end-to-end GitOps lifecycle — building container images, releasing components, and promoting them across environments — all driven through pull requests:

| Workflow | When to Use |
|---|---|
| **docker-gitops-release** | Source repo has a Dockerfile — works with any language |
| **google-cloud-buildpacks-gitops-release** | Source repo has no Dockerfile — auto-detects Go, Java, Node.js, Python, .NET, Ruby, PHP |
| **react-gitops-release** | React or SPA apps — builds with Node.js and packages into nginx |
| **bulk-gitops-release** | Promote existing releases to a target environment (no build) |

The three build-and-release workflows (docker, buildpacks, react) clone your source code, build a container image, push it to the registry, and create a pull request in this GitOps repository with the generated Workload, ComponentRelease, and ReleaseBinding manifests.

The bulk release workflow generates ReleaseBindings to promote already-built components across environments (e.g., development to staging).

For detailed parameters and usage examples, see the [workflows README](./namespaces/default/platform/workflows/README.md).

---

## Key Concepts

Deploying a component to an environment in OpenChoreo requires the following resources:

1. **ComponentRelease** — an immutable snapshot capturing the exact Component state, ComponentType, and Workload at a specific point in time.
2. **ReleaseBinding** — binds a ComponentRelease to a specific Environment, triggering OpenChoreo to render and deploy the actual Kubernetes resources (Deployment, Service, etc.).

In a GitOps workflow, all resources are committed as YAML manifests. The build-and-release workflows generate these manifests and create pull requests. Once merged, the CD tool automatically synchronizes them to the cluster.

To deploy the same release to additional environments, create new ReleaseBinding manifests referencing the same ComponentRelease but targeting different environments. This demonstrates the ReleaseBinding model: **one immutable release, multiple environments**.

---

## Tutorials

Choose a CD tool to get started with a complete end-to-end tutorial:

| CD Tool     | Tutorial                                | Status    |
|-------------|-----------------------------------------|-----------|
| **Flux CD** | [GitOps with Flux CD](./flux/README.md) | Available |
| **Argo CD** | Coming soon                             | Planned   |

Each tutorial walks you through forking this repository, configuring the CD tool, building and deploying the sample Doclet application, and promoting components across environments.
