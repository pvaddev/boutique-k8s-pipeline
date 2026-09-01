# ArgoCD

`projects/` — AppProject definitions (RBAC boundaries per environment).
`apps/` — Application manifests, one pointing at k8s/overlays/dev
(dev cluster) and one at k8s/overlays/prod (EKS cluster). App-of-apps root
Application goes here too once you have more than a couple of Applications.
