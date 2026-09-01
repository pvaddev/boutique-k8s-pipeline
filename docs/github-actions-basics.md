# GitHub Actions — Basics

## Core concepts

- **Workflow**: a YAML file in `.github/workflows/`. One repo can have many.
- **Trigger (`on:`)**: what causes the workflow to run — a push, a pull
  request, a schedule (cron), or a manual click (`workflow_dispatch`).
- **Job**: a group of steps that runs on one machine (a "runner"). Jobs run
  in parallel by default; use `needs:` to make one wait for another.
- **Step**: one command or one reusable **Action** (a packaged step someone
  else wrote, e.g. `actions/checkout@v4` to pull your repo, or
  `docker/build-push-action@v6` to build/push an image).
- **Runner**: the VM that executes the job. `ubuntu-latest` is the free
  GitHub-hosted default; you can also register your own self-hosted runner
  (relevant later if you want a runner inside your kubeadm cluster).
- **Secrets**: `${{ secrets.MY_SECRET }}` — set in repo Settings → Secrets
  and variables → Actions. Never hardcode credentials in the YAML.

## Minimal working example

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Say hello
        run: echo "Hello from CI"
```

Put that in `.github/workflows/hello.yml`, push it, and watch it run under
the "Actions" tab of your repo on GitHub. That's the whole mental model —
everything else is layering on top of it.

## Path filtering (relevant to this project)

You have 12 services in one repo — you don't want every commit to rebuild
all 12. Filter by path:

```yaml
on:
  push:
    paths:
      - 'src/frontend/**'
```

Or use a matrix + `dorny/paths-filter` action to detect *which* services
changed in a single workflow run and only build those.

## A realistic build-and-push job

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # needed for OIDC auth to AWS, see below
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account-id>:role/github-actions-ecr
          aws-region: us-east-1

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: ./src/frontend
          push: true
          tags: <account-id>.dkr.ecr.us-east-1.amazonaws.com/frontend:${{ github.sha }}
```

Note the `role-to-assume` + `id-token: write` pattern: that's **OIDC
federation**, which lets GitHub Actions authenticate to AWS without you
ever storing a long-lived AWS access key as a secret. Worth learning early
since you're targeting EKS.

## Where this fits in the project

Phase 3 (`docs/phases.md`) is where you'll build the real pipeline:
lint → test → build → scan (Trivy) → push → sign (cosign). Start with the
minimal example above, get it green, then add one step at a time — don't
write the whole pipeline in one shot.

## Good next steps
- GitHub's own quickstart: https://docs.github.com/en/actions/quickstart
- `actions/checkout`, `docker/build-push-action`, `docker/setup-buildx-action`
  are the three you'll use immediately.
