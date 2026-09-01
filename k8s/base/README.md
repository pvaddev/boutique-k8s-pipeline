# Kustomize base

Shared manifests for all 12 services: Deployment, Service,
ServiceAccount, and default resource requests/limits. Environment-specific
values live in `../overlays/dev` and `../overlays/prod` as Kustomize
patches, not here.
