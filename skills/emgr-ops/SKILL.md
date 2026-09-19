---
name: emgr-ops
description: "Running EmgR — the four Docker image flavours and the digest-pinned base images that must not be bumped casually, the two Helm charts and the two different library charts they depend on, the compose stack, the healthcheck binary, /health versus /metrics, and OpenTelemetry wiring. Load before changing a Dockerfile, a chart, a probe, the compose file, or anything about how the service is deployed or observed."
---

# emgr-ops

> **Verified against emgr `446c4df6` (2026-09-19).** Version-sensitive claims
> below carry the date they became true. On an older or newer emgr, trust the
> repository over this page. See
> [VERSIONING.md](https://github.com/vaam-apps/emgr-skills/blob/main/VERSIONING.md).

## Four image flavours, because storage is a compile-time feature

| Flavour   | Docker target    | Cargo features  |
| --------- | ---------------- | --------------- |
| `fs`      | `fs_deploy`      | `local_fs`      |
| `fs_otel` | `fs_otel_deploy` | `local_fs otel` |
| `s3`      | `s3_deploy`      | `s3`            |
| `s3_otel` | `s3_otel_deploy` | `s3 otel`       |

`build.yml`'s matrix names those four targets exactly. Each deploy stage is
`FROM base_deploy` plus one `COPY` of its builder's binary; the app binary is
built `--profile perf --locked`, the healthcheck binary `--profile prod`.

## The base-image pins are digest-pinned, and one of them is load-bearing

Builder: `rust@sha256:…` — **Debian bookworm (12)**. Runtime:
`gcr.io/distroless/cc-debian12@sha256:…`.

> **Do not bump the builder to a newer Debian casually.** A previous pin was
> a trixie image (glibc 2.41) against a bookworm runtime (2.36). The images
> built cleanly and died instantly at startup with `version 'GLIBC_2.38' not
found` — and **CI stayed green throughout**, because a build-only pipeline
> structurally cannot catch a dynamic-linking failure that surfaces at process
> start ([#62](https://github.com/vaam-apps/image-resizer/issues/62)).

That is why `build.yml` now **runs each image before pushing it**:

1. Builds host-arch-only with `load: true`, not pushed.
2. For `otel` flavours, starts the container with only
   `ALLOW_UNSIGNED_REQUESTS=true` and **asserts it exits non-zero** with
   `METRICS_AUTH_TOKEN must be set` in the log — a positive test that the
   fail-closed check still fails closed.
3. Starts it properly, polls `/health` for a `200` up to 30 times, then execs
   `/app/healthcheck` inside the container and requires exit 0.
4. Only then builds and pushes `linux/amd64,linux/arm64`.

Note step 2 is a **guard-failure proof baked into CI**: it would catch the
check being accidentally removed.

The runtime image runs `USER nonroot:nonroot`. That line is technically
redundant — the `:nonroot`-derived digest already runs uid 65532 — and is kept
explicitly because Trivy's `DS-0002` greps the Dockerfile text for a literal
`USER` instruction.

## `healthcheck`, and why it is not just a TCP connect

`src/bin/healthcheck.rs` is the container `HEALTHCHECK`
(`CMD ["/app/healthcheck"]`, 30s interval, 5s timeout, 3 retries) and the
Kubernetes exec probe.

> The previous implementation only called `connect()`. A wedged process that
> completes the TCP handshake and then never writes a response — exactly the
> case [#44](https://github.com/vaam-apps/image-resizer/issues/44)/[#48](https://github.com/vaam-apps/image-resizer/issues/48)
> flag — **reported healthy**. It performs a real HTTP request with a read
> timeout now.

## `/health` vs `/metrics`

- `GET /health` — unauthenticated, returns the literal `OK`, checks nothing.
  Deliberate: k8s probes cannot carry a bearer token, and the handler leaks no
  telemetry. Protecting it would mean breaking probes or running a second
  unauthenticated listener — a materially bigger change than this endpoint's
  sensitivity justifies. `GET /` temporary-redirects here.
- `GET /metrics` — **mounted only under `--features otel`**, behind bearer-token
  middleware, fail-closed at startup. See `emgr-security`.

## Two charts, two _different_ library charts

| Chart             | Kind                   | `version` | `appVersion` | Depends on                                                         |
| ----------------- | ---------------------- | --------: | -----------: | ------------------------------------------------------------------ |
| `helm/emgr`       | Deployment             |   `0.1.9` |      `0.2.1` | `common` **4.0.1** from `bjw-s-labs.github.io/helm-charts`         |
| `helm/serverless` | Knative, scale-to-zero |   `0.1.3` |      `0.2.1` | `common` **`"*"`** from `oci://registry-1.docker.io/bitnamicharts` |

Three things to notice before editing either:

1. **They are not the same `common`.** One is the bjw-s library chart; the
   other is Bitnami's. Do not copy a values block between them.
2. **`helm/serverless` pins its dependency as `"*"`** — an unpinned floating
   major. `helm/emgr` pins `4.0.1`.
3. **Only `appVersion` is release-please-managed** (it carries the
   `# x-release-please-version` marker). The `version:` fields are
   human-bumped and have already drifted independently of the app version —
   which is exactly why coupling them to the release number would be a
   silent _downgrade_ the moment the numbers diverge.

`helm/emgr` also carries a `PodDisruptionBudget` and liveness/readiness/startup
probes; before [#48](https://github.com/vaam-apps/image-resizer/issues/48) a
voluntary disruption could evict every replica at once and a wedged container
was never restarted.

> **Nothing in CI rendered either chart for a long time**, which is how
> `helm/emgr` shipped without the `SIGNING_KEY`/`SIGNING_SALT`/`METRICS_AUTH_TOKEN`
> wiring ([#84](https://github.com/vaam-apps/image-resizer/issues/84)) and
> `helm/serverless` shipped a doubled `ghcr.io/ghcr.io/…` image reference, a
> literal unexpanded `${LOG_LEVEL:-info}`, and an unpublished `latest` tag
> ([#85](https://github.com/vaam-apps/image-resizer/issues/85)). CI's
> `helm-verify` job now runs `dependency build`, `lint` and `template` on
> both.

Charts publish to the `gh-pages` branch via `chart-releaser`, in the same
workflow that publishes the mkdocs site.

## Compose

`make up` → `docker compose -p emgr -f compose.yaml up -d --remove-orphans
--build`. The stack is the `app` service built at the `fs_otel_deploy` target
plus `tracking` (Jaeger, OTLP collector and UI on `16686`). The app listens on
**`13001`**.

A `volume_init` busybox one-shot `chown`s the image volume to `65532:65532`
before the app starts — the runtime image is non-root and cannot create its
own storage directory in a fresh named volume.

`make help` lists the rest (`down`, `destroy`, `logs`, `ps`, `stats`, …). Note
**no `make` target runs tests, clippy or benches** — those live only in CI and
in `docs/development/`.

> `make init` is gone. It was the OpenAPI-codegen bootstrap a fresh clone used
> to require; [#53](https://github.com/vaam-apps/image-resizer/issues/53)
> replaced the generated router with a hand-written one, so `cargo build` now
> works from a clean checkout with no Docker step first.

## Observability

Under `--features otel`: OTLP traces and metrics
(`OTLP_SPAN_ENDPOINT`, `OTLP_METRIC_ENDPOINT`, `OTLP_SERVICE_NAME`), a
Prometheus `/metrics` endpoint, and `tracing` logging at `LOG_LEVEL`.

**Providers flush on exit.** Before
[#42](https://github.com/vaam-apps/image-resizer/issues/42), buffered
telemetry was lost on every restart. The same change added SIGTERM/SIGINT
graceful drain, bounded by `SHUTDOWN_TIMEOUT_SECS` (default 20) — chosen to
sit comfortably inside Kubernetes' own 30s
`terminationGracePeriodSeconds` default, so the process always finishes
draining before the orchestrator escalates to `SIGKILL`.

One implementation note worth not undoing: the drain deadline is **not** a
`tokio::time::timeout` wrapped around `axum::serve(..).with_graceful_shutdown(..)`.
That would bound the server's entire pre-shutdown runtime, which is wrong.
