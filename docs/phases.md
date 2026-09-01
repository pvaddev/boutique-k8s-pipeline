# Roadmap

Work through these roughly in order. Each phase is independently useful to
learn even if you never finish the next one.

## Phase 0 — Repo hygiene (done)
Source code only, nothing else. `LICENSE-upstream` kept for attribution.

## Phase 1 — Containerize
- One Dockerfile per service, multi-stage, non-root user, pinned base image
  digest (not just `:tag`), minimal/distroless final stage where possible.
- `docker compose` file to run the whole app locally and sanity-check it
  before touching Kubernetes at all.
- `.dockerignore` per service.

## Phase 2 — Infra as Code
- `infra/kubeadm-dev`: scripts (or Ansible playbooks) to install
  containerd, kubeadm/kubelet/kubectl, init the control plane, install a
  CNI (Cilium or Calico), join workers. Treat this like a runbook you can
  re-run from a clean VM.
- `infra/eks-prod`: Terraform — VPC, EKS cluster, managed node group (or
  Fargate profile), IAM roles for service accounts (IRSA), ECR repos for
  images. Remote state in S3 + DynamoDB lock table.

## Phase 3 — CI (GitHub Actions)
- Trigger on PR + push to main, path-filtered per service so you're not
  rebuilding all 12 services on every commit.
- Steps: lint → unit test → build image → Trivy vulnerability scan → push
  to registry (ECR for prod, local registry or GHCR for dev) → sign with
  cosign.
- See `docs/github-actions-basics.md` first if you're new to Actions.

## Phase 4 — Kubernetes manifests
- `k8s/base`: Deployment, Service, ServiceAccount, resource
  requests/limits for every service, as Kustomize base resources.
- `k8s/overlays/dev`: single replica, no HPA, relaxed limits, `imagePullPolicy: Always`.
- `k8s/overlays/prod`: HPA, PodDisruptionBudget, NetworkPolicy, stricter
  limits, `imagePullPolicy: IfNotPresent`, pinned image digests.

## Phase 5 — GitOps CD (ArgoCD)
- Install ArgoCD on both clusters (or one ArgoCD in prod managing both via
  registered clusters).
- App-of-apps pattern: one root Application per environment, pointing at
  `k8s/overlays/dev` and `k8s/overlays/prod`.
- No manual `kubectl apply` from here on — merging to main is the only way
  to change cluster state.

## Phase 6 — Service mesh (Istio)
- mTLS between services, ingress Gateway, VirtualServices for traffic
  splitting — sets you up for canary releases in Phase 8.

## Phase 7 — Observability
- kube-prometheus-stack (Prometheus + Grafana + Alertmanager).
- Loki for logs, Tempo for traces — the app already emits OpenTelemetry
  spans, you just need a collector (otel-collector) wired up.

## Phase 8 — Secrets & policy
- External Secrets Operator pulling from AWS Secrets Manager (prod) / a
  local secrets store (dev).
- Kyverno or OPA Gatekeeper: block `:latest` tags, require resource
  limits, block root containers.

## Phase 9 — Progressive delivery
- Argo Rollouts for canary/blue-green, gated on Prometheus metrics
  (error rate, p99 latency) instead of a human clicking approve.
