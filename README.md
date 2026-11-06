# mlops-llmops-platform-local

End-to-end MLOps → LLMOps platform running locally on k3d + Podman: DVC/MinIO data
versioning, MLflow registry with champion/candidate gating, GitOps via ArgoCD, and
drift-triggered retraining. Later extended with vLLM serving and LLM eval pipelines.

> **Status:** 🚧 Work in progress. See [Roadmap](#roadmap).

## Goals

- Build a production-style ML lifecycle loop entirely on a laptop: data → train →
  gate → deploy → monitor → retrain.
- Keep the application layer (`ml/`, `serving/`, `monitoring/`) cloud-agnostic so it
  ports unchanged to the AWS version of this project (`mlops-llmops-platform-aws`).
- Extend the same platform to LLMOps: model serving, prompt/config versioning, and
  automated evaluation gates.

## Architecture

_Diagram coming with M1–M5._

| Concern             | Tool                                  |
|---------------------|---------------------------------------|
| Cluster             | k3d (k3s in Podman)                   |
| GitOps              | ArgoCD                                |
| Object storage      | MinIO (S3-compatible)                 |
| Data versioning     | DVC                                   |
| Data validation     | Pydantic, Great Expectations          |
| Experiment tracking | MLflow (Postgres + MinIO)             |
| CI/CD               | GitHub Actions (hosted + self-hosted) |
| Serving             | FastAPI, Argo Rollouts                |
| Monitoring          | Prometheus, Grafana, Evidently        |
| Orchestration       | Prefect                               |

## Repository layout

```
infra/        Environment-specific infrastructure (k3d config, Helm values)
gitops/       Manifests watched by ArgoCD (platform + apps)
ml/           Pipelines (dvc.yaml), training, validation, gating code
serving/      Model-serving API and container image
monitoring/   Drift jobs, alert rules, dashboards
.github/      CI workflows
```

## Prerequisites

- macOS (Apple Silicon / arm64), Podman Desktop or Podman CLI
- Podman machine: **rootful**, ≥ 6 CPU / 10 GiB RAM / 100 GB disk

| Tool    | Tested version |
|---------|----------------|
| podman  | 6.1.4          |
| k3d     | v5.9.0         |
| k3s     | v1.35.5-k3s1 (pinned in `infra/k3d/cluster.yaml`) |
| kubectl | v1.37.2        |
| helm    | v4.3.1         |
| argocd  | v3.6.0         |
| uv      | 0.13.7         |
| dvc     | 3.67.1         |
| gh      | 2.103.3        |

### Shell environment

k3d expects a Docker API socket. Point it at Podman (add to `~/.zshrc`):

```bash
# Podman API socket on macOS (used by the k3d CLI)
export DOCKER_HOST="unix://$(podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}' 2>/dev/null)"
# Socket path *inside* the rootful Podman VM (mounted by k3d into its helper containers)
export DOCKER_SOCK=/run/podman/podman.sock
```

## Quickstart

```bash
# 1. Size and start the Podman VM (first time only: init)
podman machine init --cpus 6 --memory 10240 --disk-size 100 --rootful
podman machine start

# 2. Create the local image registry (one-time; survives cluster rebuilds)
k3d registry create registry.localhost --port 0.0.0.0:5001 --default-network podman

# 3. Create the cluster
k3d cluster create --config infra/k3d/cluster.yaml

# 4. Verify
kubectl get nodes                                   # 3 nodes Ready
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/   # 404 from Traefik = ingress OK
```

Registry addressing:

| From                 | Address                                |
|----------------------|----------------------------------------|
| Mac (`podman push`)  | `localhost:5001/<image>` (add `--tls-verify=false`) |
| Pods in the cluster  | `k3d-registry.localhost:5001/<image>`  |

Tear down / rebuild:

```bash
k3d cluster delete mlops-llmops-platform-local      # registry and its images are kept
k3d cluster create --config infra/k3d/cluster.yaml
```

_More steps added as each milestone lands._

## Roadmap

- [x] Step 0 – Workstation, k3d cluster, local registry
- [ ] Step 1 – Platform bootstrap via ArgoCD (MinIO, Postgres, MLflow)
- [ ] M1 – Data versioning + validation gate
- [ ] M2 – Training + MLflow registry (`@candidate`)
- [ ] M3 – CI quality gate (`@champion` promotion)
- [ ] M4 – Serving via GitOps + canary rollout
- [ ] M5 – Drift detection → automated retraining
- [ ] L3 – LLM serving + evaluation harness

## Design decisions

- **Registry created outside the cluster config.** k3d's `registries.create` attaches
  the registry to Docker's default `bridge` network, which Podman doesn't have (its
  default is `podman`), and the config file can't override it. The registry is created
  once via CLI with `--default-network podman` and referenced with `registries.use`.
  Side benefit: pushed images survive `k3d cluster delete`.
- **`DOCKER_HOST` ≠ `DOCKER_SOCK` on Podman.** `DOCKER_HOST` is the macOS-side socket the
  k3d CLI talks to; `DOCKER_SOCK` is bind-mounted into k3d's tools container, so it must
  be the path inside the VM (`/run/podman/podman.sock` for a rootful machine).
- **Fixed API port (6550) and pinned k3s image.** Rebuilt clusters are identical and the
  kubeconfig stays stable.
- **Monorepo with a `gitops/` folder** instead of a separate GitOps repo; CI workflows use
  `paths:` filters to avoid commit loops.
- **Environment-specific code lives only in `infra/` and `gitops/`**, so `ml/`,
  `serving/`, and `monitoring/` port unchanged to the AWS version.

## License

MIT
