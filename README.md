# E-Commerce DevOps Learning Project

This repo starts from **only the source code** of Google's
[Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo)
(12 polyglot microservices, Apache 2.0 licensed — see `LICENSE-upstream`).
Everything else — Dockerfiles, Kubernetes manifests, Helm chart, Terraform,
Skaffold/Cloud Build config, Istio manifests — has been deliberately removed.
The goal of this project is to rebuild all of that DevOps tooling from
scratch as a hands-on learning exercise.

## Target architecture

| Environment | Cluster | Purpose |
|---|---|---|
| `dev` | Self-managed `kubeadm` cluster (local/on-prem) | Fast iteration, cheap, full control over the control plane |
| `prod` | AWS EKS | Managed control plane, IAM integration, realistic cloud patterns |

Both environments are driven by the same GitOps source of truth (`gitops/`),
just pointed at different clusters/contexts.

## Repo layout

```
src/                  Pure application source (untouched, from upstream)
protos/               gRPC service definitions (needed to build src/*)
infra/
  kubeadm-dev/         Scripts/Ansible to stand up the dev cluster with kubeadm
  eks-prod/            Terraform for the AWS EKS prod cluster
k8s/
  base/                Kustomize base manifests (shared across envs)
  overlays/dev/        Dev-specific patches (1 replica, relaxed limits, etc.)
  overlays/prod/       Prod-specific patches (HPA, PodDisruptionBudgets, etc.)
gitops/
  argocd/              ArgoCD Application / AppProject manifests
.github/workflows/     CI pipelines (see docs/github-actions-basics.md)
docs/                  Learning notes and the phase-by-phase roadmap
```

## Build order (see `docs/phases.md` for detail)

1. **Containerize** — write a Dockerfile per service in `src/<service>/`
2. **Infra as Code** — `infra/kubeadm-dev` (dev) and `infra/eks-prod` (prod)
3. **CI** — `.github/workflows/` builds, scans, and pushes images
4. **K8s manifests** — `k8s/base` + per-env overlays
5. **GitOps CD** — ArgoCD watches this repo and syncs each cluster
6. **Service mesh, observability, policy, progressive delivery** — layered
   on top once the core loop works

## Services in `src/`

adservice, cartservice, checkoutservice, currencyservice, emailservice,
frontend, loadgenerator, paymentservice, productcatalogservice,
recommendationservice, shippingservice, shoppingassistantservice.

Check each service's own README/build file (`go.mod`, `pom.xml`,
`package.json`, `*.csproj`, `requirements.txt`) to see what language/runtime
it needs before writing its Dockerfile.
