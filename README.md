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

- macOS with Podman machine (rootful, ≥ 6 CPU / 12 GB RAM)
- `podman`, `k3d`, `kubectl`, `helm`, `argocd`, `uv`, `dvc`, `gh`

<!-- Add exact versions once installed: podman --version, k3d version, etc. -->

## Quickstart

```bash
# 1. Start Podman VM
podman machine start

# 2. Create cluster + local registry
k3d cluster create --config infra/k3d/cluster.yaml

# 3. Verify
kubectl get nodes
```

_More steps added as each milestone lands._

## Roadmap

- [ ] Step 0 – Workstation, k3d cluster, local registry
- [ ] Step 1 – Platform bootstrap via ArgoCD (MinIO, Postgres, MLflow)
- [ ] M1 – Data versioning + validation gate
- [ ] M2 – Training + MLflow registry (`@candidate`)
- [ ] M3 – CI quality gate (`@champion` promotion)
- [ ] M4 – Serving via GitOps + canary rollout
- [ ] M5 – Drift detection → automated retraining
- [ ] L3 – LLM serving + evaluation harness

## Design decisions

_Short notes on key choices and trade-offs, added as they're made._

## License

MIT
