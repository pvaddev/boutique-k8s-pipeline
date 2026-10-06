---
name: containerize
description: Write the Dockerfile and .dockerignore for one Online Boutique service under src/, then build the image and smoke-test it. Use when asked to containerize, dockerize, or write the Dockerfile for a service.
argument-hint: <service-name>
---

Containerize `src/$ARGUMENTS`. If no service name was given, list the services under `src/` that have no Dockerfile yet and ask which one.

`shoppingassistantservice` is out of scope (it needs AlloyDB, Secret Manager, and Gemini on GCP) — confirm before containerizing it.

## 1. Inspect the service

- Read its build file (`go.mod`, `package.json`, `requirements.txt`, `build.gradle`, `*.csproj`) for the language and runtime version. The base image version must match it.
- Find the entrypoint, the listening port, and which env vars are required, from the source and the README "Configuration reference" table.
- Note which non-code files the process reads at runtime (`products.json`, `data/`, `templates/`, `static/`, `proto/`) — they must be copied into the final stage.
- Read `src/adservice/Dockerfile` and `src/cartservice/src/Dockerfile` so the new file matches their style.

## 2. Write the Dockerfile

Follow the "Dockerfile conventions" section of `CLAUDE.md`. In particular:

- The build context is the service directory, so nothing outside it (including the top-level `protos/`) can be copied. Use the proto copy or generated stubs already inside the service.
- Pin every base image as `image:tag@sha256:...`. Look the digest up for the tag you chose (`docker buildx imagetools inspect <image>:<tag>` or `skopeo inspect docker://<image>:<tag>`) — never invent or reuse one from memory.
- Copy the dependency manifest and install dependencies before copying the source.
- Use the smallest final stage the language allows (static Go binary on distroless/scratch; slim runtime images for Node and Python).
- Run as a non-root numeric UID.
- Set the port env var and `EXPOSE` the same port. `currencyservice` and `paymentservice` have no default port, so the image must set `PORT`.

## 3. Write the .dockerignore

Place it next to the Dockerfile. Exclude build output, dependency directories (`node_modules/`, `__pycache__/`, `.gradle/`, `bin/`, `obj/`), editor files, and the `Dockerfile` and `.dockerignore` themselves.

## 4. Build and smoke-test

```sh
docker build -t $ARGUMENTS:dev src/$ARGUMENTS
docker run --rm -d --name $ARGUMENTS-smoke -p <port>:<port> <env flags> $ARGUMENTS:dev
```

Pass `-e DISABLE_PROFILER=1` (plus `DISABLE_TRACING=1` / `DISABLE_STATS=1` where the service uses those) so it does not try to reach Google Cloud. Services with required `*_SERVICE_ADDR` variables exit without them; a placeholder such as `localhost:1` is enough to get the process to start for a health check.

Then verify:

- gRPC services — standard health protocol:
  ```sh
  grpcurl -plaintext -import-path protos -proto grpc/health/v1/health.proto \
    localhost:<port> grpc.health.v1.Health/Check
  ```
  Expect `"status": "SERVING"`. Where the service has no upstream dependencies, also call one real method using `-proto demo.proto`.
- `frontend` — `curl -fsS localhost:8080/_healthz`.
- `loadgenerator` — no server; confirm `locust --version` runs inside the image.

Also confirm the container is not running as root (`docker exec $ARGUMENTS-smoke id -u`, or `docker inspect` the image's `User` if the image has no shell), check `docker logs` for startup errors, then stop the container.

If the build or the health check fails, fix the Dockerfile and repeat this step. Do not report the service as done on an unverified image.

## 5. Update the docs

In `README.md`: bump the Dockerfile count and service list in the Phase 1 row of "Current status", and add the Dockerfile to the "Repo layout" block. Update the "Current state" paragraph in `CLAUDE.md` to match.

## 6. Report

State the final image size (`docker images $ARGUMENTS:dev`), the base images and digests used, the smoke-test result, and anything about this service that deviated from the conventions and why. Do not commit unless asked.
