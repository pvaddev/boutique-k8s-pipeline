# dev cluster — kubeadm

Scripts/playbooks to bootstrap a self-managed Kubernetes control plane +
worker(s) for the `dev` environment.

Planned contents:
- `scripts/00-prereqs.sh`   — disable swap, load kernel modules, install containerd
- `scripts/01-control-plane.sh` — kubeadm init, install CNI (Cilium/Calico)
- `scripts/02-join-worker.sh`   — kubeadm join for additional nodes
- `ansible/` — same steps as an idempotent playbook, once the shell scripts
  work, so you can re-provision from scratch reliably

Treat this directory as a runbook: it should take a bare VM (or a few) to a
working cluster with zero manual steps outside these scripts.
