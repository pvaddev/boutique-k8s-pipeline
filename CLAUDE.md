# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A DevOps learning project. `src/` and `protos/` are the unmodified application source of Google's Online Boutique (12 polyglot gRPC microservices, Apache 2.0 — `LICENSE-upstream`). All upstream DevOps tooling (Dockerfiles, k8s manifests, Helm, Terraform, Skaffold, Istio) was deliberately deleted; the point of the project is to author that tooling from scratch, phase by phase.

Consequences for how to work here:

- The deliverable is the tooling (Dockerfiles, compose, Terraform, workflows, Kustomize, ArgoCD), not application changes. Avoid editing the upstream source under `src/` unless a build genuinely requires it.
- Do not copy or restore the upstream Dockerfiles/manifests — they are written here by hand on purpose.
- `docs/phases.md` is the roadmap and the spec for each phase's requirements. The README's "Current status" table tracks progress; update it when a phase item lands (e.g. the Dockerfile count).
- `infra/`, `k8s/`, and `gitops/` currently hold only placeholder READMEs describing what is planned there.

## Current state

Phase 1 (Containerize). Dockerfiles exist for `cartservice`, `adservice`, `checkoutservice`, `currencyservice`, and `emailservice`; the other seven services and the compose file are still to do. The whole shop cannot be run yet.

`.github/workflows/hello-ci.yml` is a starter that only lists `src/`. It runs on pushes to `master`, on PRs, and on manual dispatch.

## Commands

Build and run the existing images:

```sh
# cartservice — build context is src/cartservice/src (where the .csproj is), not the service root
docker build -t cartservice:dev src/cartservice/src
docker run --rm -p 7070:7070 cartservice:dev            # add -e REDIS_ADDR=<host>:6379 for Redis

docker build -t adservice:dev src/adservice
docker run --rm -p 9555:9555 -e DISABLE_STATS=1 -e DISABLE_TRACING=1 adservice:dev

# checkoutservice — needs all six *_SERVICE_ADDR vars; placeholders (localhost:1) are enough to start it
docker build -t checkoutservice:dev src/checkoutservice

# currencyservice — image sets PORT=7000 and DISABLE_PROFILER=1
docker build -t currencyservice:dev src/currencyservice
docker run --rm -p 7000:7000 currencyservice:dev

# emailservice — image sets PORT=8080 and DISABLE_PROFILER=1; dummy mode, requests are only logged
docker build -t emailservice:dev src/emailservice
docker run --rm -p 8080:8080 emailservice:dev
```

Smoke-test a gRPC service (no browser UI) against the shared proto:

```sh
grpcurl -plaintext -import-path protos -proto grpc/health/v1/health.proto \
  localhost:<port> grpc.health.v1.Health/Check        # any gRPC service
grpcurl -plaintext -import-path protos -proto demo.proto \
  -d '{"context_keys": ["clothing"]}' localhost:9555 hipstershop.AdService/GetAds
```

Unit tests — only the Go services and cartservice have any; each Go service is its own module, so run from its directory:

```sh
(cd src/frontend && go test ./...)                 # also: checkoutservice, productcatalogservice, shippingservice
(cd src/shippingservice && go test -run TestName ./...)    # single Go test
dotnet test src/cartservice/tests/cartservice.tests.csproj
dotnet test src/cartservice/tests/cartservice.tests.csproj --filter "FullyQualifiedName~TestName"
```

The Java, Node.js, and Python services have no tests (`npm test` is a stub that exits 1), and there is no lint configuration anywhere yet.

## Dockerfile conventions

The five existing Dockerfiles set the pattern; follow it for the remaining services, starting from the one in the same language where there is one:

- Multi-stage: build stage with the SDK/compiler, minimal runtime stage carrying only the output. Go uses `gcr.io/distroless/static-debian12:nonroot` (static binary, `CGO_ENABLED=0`); Node.js uses `node:24-bookworm-slim` to install and `gcr.io/distroless/nodejs24-debian12:nonroot` to run (same libc in both stages; `CMD ["server.js"]` because the image's entrypoint is already `node`). Python uses `python:3.11-slim-bookworm` to install and `gcr.io/distroless/python3-debian12:nonroot` to run (`CMD ["email_server.py"]`, entrypoint is already `python3`).
- Base images pinned by digest (`image:tag@sha256:...`), not tag alone. Use the index digest — the top `Digest:` line of `docker buildx imagetools inspect <image>:<tag>`. The entries under `Manifests:` with platform `unknown/unknown` are attestations and fail the build.
- Dependency manifest copied and restored before the source, so the dependency layer stays cached (`cartservice.csproj` → `dotnet restore`; `build.gradle` → `./gradlew downloadRepos`, a task that exists in `build.gradle` for exactly this; `go.mod` + `go.sum` → `go mod download`; `package.json` + `package-lock.json` → `npm ci --omit=dev`; `requirements.txt` → `pip install --no-cache-dir --target=/app/deps -r requirements.txt`).
- Runtime runs as a non-root **numeric** UID (`USER 10001`, `USER 65532` on distroless, or `$APP_UID` on the .NET image) so Kubernetes `runAsNonRoot` can verify it.
- `EXPOSE` plus the matching port env var (`PORT`, `ASPNETCORE_URLS`) set in the image.
- A `.dockerignore` next to each Dockerfile, excluding build output, `node_modules/`, editor files, and the Dockerfile itself. It must not exclude files the service reads at runtime (`proto/`, `data/`).
- Distroless images have no shell: use exec-form `ENTRYPOINT`/`CMD`, and check the user with `docker inspect -f '{{.Config.User}}'` rather than `docker exec`.

Node-specific: `currencyservice` installs with `npm ci --omit=dev --ignore-scripts` and sets `ENV DISABLE_PROFILER=1` in the runtime stage. Its locked `pprof` 4.0.0 (a native module used only by Cloud Profiler) has no prebuilt binary for Node 24 and does not compile against it, so the install script is skipped; without `DISABLE_PROFILER` the container crashes at startup. `paymentservice` also depends on `pprof` (5.0.0) — check whether it builds before reusing this workaround.

Python-specific: `emailservice` installs with `pip install --target=/app/deps` and sets `ENV PYTHONPATH=/app/deps` in the runtime stage — a virtualenv breaks when copied, because its interpreter symlink points at `/usr/local/bin/python` and distroless has Python at `/usr/bin/python3`. Both stages must be the same Python minor (3.11, what distroless Debian 12 ships): `grpcio` and `markupsafe` are compiled per version, and a mismatch shows up as an `ImportError` at startup. Also set `PYTHONUNBUFFERED=1` so logs reach `docker logs` immediately. `WORKDIR` must be the directory holding `templates/` — the template path is relative to the working directory and is loaded at startup. `recommendationservice` should follow the same layout.

Before writing a service's Dockerfile, check its build file (`go.mod`, `package.json`, `requirements.txt`, `*.csproj`, `build.gradle`) for the runtime version.

## Application architecture

`frontend` (Go, HTTP :8080, health at `/_healthz`) is the only externally reachable service and calls nearly every backend. `checkoutservice` is the second orchestrator: cart → catalog → currency → shipping → payment → email. `recommendationservice` calls `productcatalogservice`; `cartservice` optionally talks to Redis. Everything else is a leaf. The README has the full call graph, per-service ports, and the complete env-var reference table — consult it when wiring compose or manifests.

Things that span multiple services and are easy to get wrong:

- **Proto contract.** `protos/demo.proto` is the source of truth, but each service carries its own copy or generated stubs, refreshed by its `genproto.sh` (run from the service directory; paths are relative `../../protos`). Go services commit generated code in `genproto/`; Python services commit `demo_pb2*.py`; Node services and adservice hold a copied `.proto` (without the `go_package` option) and compile/load it at build or run time; cartservice has its own `src/protos/Cart.proto`. Docker build contexts are per-service, so they cannot reach `protos/` — the in-service copy is what gets built.
- **Service wiring is env-var driven and strict.** Missing `*_SERVICE_ADDR` variables make `frontend` and `checkoutservice` exit at startup. `frontend` requires `SHOPPING_ASSISTANT_SERVICE_ADDR` to be set to *some* value even though the assistant is only used when `ENABLE_ASSISTANT=true`.
- **No default port on the Node services.** `currencyservice` and `paymentservice` require `PORT`. `emailservice`, `recommendationservice`, and `frontend` all default to 8080, which collides outside per-container network namespaces.
- **GCP defaults must be switched off.** The code assumes Google Cloud. Set `DISABLE_PROFILER=1` generally, plus `DISABLE_TRACING=1` / `DISABLE_STATS=1` for `shippingservice` and `adservice`, or services try to reach Cloud Profiler/Trace.
- **Health checks.** All gRPC services implement the standard gRPC health protocol (use gRPC probes); only `frontend` has an HTTP health endpoint.
- **Cart store selection.** `cartservice` chooses at startup: Redis if `REDIS_ADDR` is set, else Spanner/AlloyDB if configured, else in-memory (lost on restart, not shared across replicas).
- **`shoppingassistantservice` is out of scope.** It needs AlloyDB, Secret Manager, and Gemini on GCP; the target platforms are kubeadm (dev) and AWS EKS (prod).
- **Fault injection.** `productcatalogservice` honours `EXTRA_LATENCY` (e.g. `5.5s`) and toggles a slow reload-per-request mode on `SIGUSR1`/`SIGUSR2` — intended for later observability and canary phases.

## Target design (not built yet)

Two environments from one GitOps source: `dev` on a self-managed kubeadm cluster, `prod` on AWS EKS. Planned flow: path-filtered GitHub Actions per service (lint → test → build → Trivy → push → cosign), images to GHCR/local registry for dev and ECR for prod, Kustomize `k8s/base` patched by `k8s/overlays/{dev,prod}`, ArgoCD app-of-apps syncing each cluster to its overlay.
