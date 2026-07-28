# Backstage Software Templates

Golden-path software templates for [Backstage](https://backstage.io), enabling developers to self-service the creation of production-ready microservices — complete with source code scaffolding, containerization, Helm charts, CI/CD pipelines, and GitOps-based deployment via ArgoCD.

## Overview

This repository provides a **Backstage Scaffolder template** that automates the entire lifecycle of a new microservice, from initial code generation to a running workload on Kubernetes:

1. A developer fills in a short form in the Backstage **Create Component** UI.
2. Backstage scaffolds a new repository from the template, pushes it to GitHub, and registers it in the **Software Catalog**.
3. A GitHub Actions pipeline builds and publishes a container image.
4. The pipeline updates the corresponding Helm values file and commits the change back to the repository (GitOps).
5. **ArgoCD** detects the change and syncs the application to the target Kubernetes namespace.

The result: a new service goes from "idea" to "running in a cluster" with a single self-service request — no manual repo setup, pipeline wiring, or Kubernetes manifests required.

## Available Templates

| Template | Description | Stack |
|---|---|---|
| `python-app` | Minimalistic Flask microservice template | Python, Flask, Docker, Helm, ArgoCD |

## Repository Structure

```
python-app/
├── template.yaml              # Backstage Scaffolder template definition
└── template/                  # Skeleton content copied into every new component
    ├── catalog-info.yaml      # Backstage catalog entity definition
    ├── Dockerfile              # Container image build definition
    ├── requirements.txt        # Python dependencies
    ├── src/
    │   └── app.py               # Flask application entrypoint
    ├── mkdocs.yml               # TechDocs configuration
    ├── docs/
    │   └── index.md             # Component documentation (rendered via TechDocs)
    ├── k8s/
    │   ├── deploy.yaml           # Raw Kubernetes deployment manifest (reference)
    │   ├── service.yaml
    │   └── ingress.yaml
    └── charts/
        ├── ${{ values.app_name }}/       # Per-app Helm chart, name templated per instance
        │   ├── Chart.yaml
        │   ├── values-${{ values.app_env }}.yaml   # Environment-specific values (dev/prod)
        │   └── templates/
        │       ├── deployment.yaml
        │       ├── service.yaml
        │       ├── serviceaccount.yaml
        │       ├── ingress.yaml
        │       ├── hpa.yaml
        │       ├── _helpers.tpl
        │       ├── NOTES.txt
        │       └── tests/
        │           └── test-connection.yaml
        └── argocd/
            └── values-argo.yaml            # ArgoCD Application bootstrap values
```

## How It Works

### 1. Scaffolding (Backstage)

The `template.yaml` defines a Backstage Scaffolder template with two required inputs:

| Parameter | Description |
|---|---|
| `component_id` | Name of the new microservice (validated against a DNS/repo-safe naming pattern) |
| `environment` | Target deployment environment — `dev` or `prod` |

The template runs three scaffolder steps:

1. **`fetch:template`** — Copies the `template/` skeleton into a new working directory, substituting `${{ values.app_name }}` and `${{ values.app_env }}` placeholders with the values supplied by the user.
2. **`publish:github`** — Creates a new GitHub repository under the target organization and pushes the generated content.
3. **`catalog:register`** — Registers the new repository's `catalog-info.yaml` in the Backstage Software Catalog, making the component instantly discoverable.

### 2. Continuous Integration (GitHub Actions — `ci` job)

Triggered on every push to `main` that touches `src/**`:

- Checks out the repository.
- Provisions any required TLS/CA certificates for internal registries.
- Builds the Docker image using Buildx.
- Pushes the image to Docker Hub, tagged with the short Git commit SHA.

### 3. Continuous Deployment (GitHub Actions — `cd` job)

Runs on a self-hosted runner after `ci` succeeds:

- Updates `charts/<app_name>/values-<app_env>.yaml` with the newly built image tag (using `yq`).
- Commits and pushes the updated values file back to the repository — the GitOps source of truth.
- Installs the ArgoCD CLI and authenticates against the in-cluster ArgoCD server.
- Ensures the ArgoCD **repository** connection and **application** exist (creates them if missing, targeting the correct namespace and values file).
- Triggers a manual **sync** and waits for the application to become healthy.

### 4. GitOps Delivery (ArgoCD)

ArgoCD continuously reconciles the live cluster state with the Helm chart in Git. Each environment (`dev`, `prod`) maps to its own Kubernetes namespace and its own values file, so the same chart can be promoted across environments simply by updating the relevant `values-<env>.yaml`.

## Platform Architecture

The reference platform used to develop and validate these templates runs entirely on a local **kind** (Kubernetes-in-Docker) cluster:

```
kind-control-plane (Docker)
├── backstage/            → Backstage instance + PostgreSQL
├── argocd/               → ArgoCD (application-controller, server, repo-server, dex, redis)
├── ingress-nginx/        → Ingress controller
├── github-runner/        → Self-hosted GitHub Actions runner (executes the `cd` job)
└── <dev|prod>/           → Namespaces hosting deployed application workloads
```

- **Backstage** hosts the Scaffolder UI and Software Catalog.
- **ArgoCD** manages GitOps-based delivery to `dev` and `prod` namespaces.
- **Ingress NGINX** exposes services outside the cluster.
- A **self-hosted GitHub Actions runner** executes the deployment (`cd`) job with in-cluster network access to ArgoCD.

## Prerequisites

To use these templates in your own environment, you will need:

- A running **Backstage** instance with the Scaffolder plugin enabled.
- A **Kubernetes** cluster (e.g., kind, k3s, or a managed cluster) with:
  - **ArgoCD** installed and accessible from your CI runner.
  - **Ingress NGINX** (or equivalent) for external access.
- A **GitHub organization** with permissions for Backstage to create and push repositories (`publish:github` action).
- A **self-hosted GitHub Actions runner** with network access to the ArgoCD server.
- A **Docker Hub** account (or another container registry) for publishing images.

### Required Secrets & Variables

Configure the following at the organization or repository level in GitHub:

| Name | Type | Purpose |
|---|---|---|
| `DOCKERHUB_USERNAME` | Variable | Docker Hub login |
| `DOCKERHUB_TOKEN` | Secret | Docker Hub access token |
| `SOPHOS_CA_CRT` | Secret | Internal CA certificate for trusted registry/network access |
| `ARGOCD_PASSWORD` | Secret | ArgoCD admin password used by the self-hosted runner |

## Usage

1. Open the Backstage **Create Component** page.
2. Select **Python Flask Template**.
3. Provide a `component_id` (service name) and choose an `environment` (`dev` or `prod`).
4. Submit — Backstage will scaffold, publish, and register the new service.
5. Watch the pipeline run in the new repository's **Actions** tab.
6. Once the `cd` job completes, the service will be live in the corresponding Kubernetes namespace, managed by ArgoCD.

## Adding a New Template

To add support for a new stack (e.g., Node.js, Go):

1. Create a new top-level folder (e.g., `nodejs-app/`) following the same structure as `python-app/`.
2. Define a `template.yaml` describing the Scaffolder parameters and steps.
3. Provide a `template/` skeleton with source code, `Dockerfile`, Helm chart, and `catalog-info.yaml`.
4. Register the new template's `template.yaml` location in your Backstage `app-config.yaml` under `catalog.locations`.

## Roadmap

- [ ] Parameterize CI/CD workflow generation directly from the Scaffolder template (avoid duplicating pipeline logic per template).
- [ ] Add automated sync-wave / rollback strategy in ArgoCD for failed deployments.
- [ ] Stabilize the self-hosted GitHub Actions runner (resolve current `CrashLoopBackOff`).
- [ ] Add additional stack templates (Node.js, Go, Java/Spring Boot).
- [ ] Add automated tests and template validation in CI.
- [ ] Support multi-environment promotion (`dev` → `staging` → `prod`) with approval gates.

## License

Specify your license here (e.g., MIT, Apache-2.0).

## Maintainer

**Zahra Ehghaghi** — [github.com/zahra-ehghaghi](https://github.com/zahra-ehghaghi)
