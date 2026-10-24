# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## What this repo is

A polyglot microservice "astronomy shop" webstore (Go, .NET, Java/Kotlin, C++,
Ruby, PHP, Rust, TypeScript/Node, Python, Elixir) that exists to demonstrate
OpenTelemetry instrumentation. Every service runs as a container; there is no
top-level native build. Docker (or Podman via `DOCKER_CMD=podman
DOCKER_COMPOSE_CMD="podman compose"`) and Make are the only hard prerequisites.

## Commands

```sh
make start                         # full demo + observability stack + extras
make start-minimal                 # core services only (no Kafka/accounting/fraud-detection)
make start-agentic                 # adds agent, mcp, chatbot (compose.agent.yaml)
make start-profiling               # adds eBPF profiler + Firepit
make stop                          # tears down every compose combination, removes volumes

make build service=<name>          # build one service image
make restart service=<name>        # recreate one running container
make redeploy service=<name>       # build + recreate one service (normal dev loop)

make run-telemetry-tests           # start demo, run end-to-end telemetry pytest, stop
make run-telemetry-tests-minimal   # same, minimal scope
make run-telemetry-tests-agentic   # same, agentic scope
make run-frontend-tests            # Cypress tests (compose.tests.yaml)

make docker-generate-protobuf      # regenerate gRPC stubs from pb/demo.proto (in Docker)
make generate-protobuf             # same, using local toolchains (ide-gen-proto.sh)

make check                         # misspell, markdownlint, license headers, links, changelog
make yamllint                      # CI runs: yamllint . -f github --strict
python3 ./internal/tools/sanitycheck.py   # EOL/indentation/trailing-newline checks run in CI
make chlog-new / make chlog-validate      # changelog fragment in .chloggen/
```

UI entry points once running (all through the Envoy frontend-proxy on
`localhost:8080`): `/` shop, `/jaeger/ui`, `/grafana/`, `/loadgen/`,
`/feature/` (flagd UI), `/telemetry/` (Weaver docs), `/opamp/`, `/chatbot/`
(agentic only).

Single telemetry test: the tests run inside a container on the
`opentelemetry-demo` network, so with the demo up, run pytest from inside the
`opentelemetry-demo-telemetry-tests` image (built from `test/telemetry/`), e.g.
`pytest test_traces.py -k "checkout" -v` or
`pytest test_traces_edges.py -k "frontend->cart" -v`.

Weaver schema check (as CI does it):

```sh
docker run --rm --mount "type=bind,source=$PWD/telemetry-schema,target=/home/weaver/source,readonly" \
  "$(grep -oP '(?<=^FROM )\S*otel/weaver\S*' src/telemetry-docs/Dockerfile)" registry check -r source
```

## Architecture

**Layered compose files.** The `Makefile` composes the stack from
`-f` layers: `compose.yaml` (core services, flagd, valkey, postgres, collector),
`compose.full.yaml` (Kafka, accounting, fraud-detection, and overrides to
checkout/collector), `compose.observability.yaml` (Jaeger, Prometheus,
OpenSearch, Grafana, OpAMP), `compose.extras.yaml` (vendor customizations),
plus optional `compose.agent.yaml`, `compose.profiling.yaml`,
`compose.tests.yaml`. Later layers override earlier ones for the same service,
so a service's config may be split across files. Always invoke compose with
`--env-file .env --env-file .env.override` (the Makefile does this).

**Configuration.** `.env` holds image versions, ports, hostnames and build args
for every service. `.env.override` is for local overrides (e.g. LLM keys,
Podman socket) and must not be committed upstream.

**Inter-service contracts.** All gRPC services share `pb/demo.proto`. Generated
code is committed for checkout and product-catalog (`genproto/`) and
frontend (`protos/demo.ts`); other languages copy the proto at build time.
After editing the proto, run `make docker-generate-protobuf` — CI fails on a
dirty tree. Checkout publishes orders to Kafka, consumed by accounting and
fraud-detection (full mode only).

**Telemetry pipeline.** Services export OTLP to `otel-collector`. Collector
config is also layered: `src/otel-collector/otelcol-config.yml` plus
`-full`, `-observability`, `-extras` variants matching the compose layers.
The collector fans out to Jaeger (traces), Prometheus (metrics) and OpenSearch
(logs); Grafana dashboards/datasources are provisioned from `src/grafana/`.

**Feature flags.** Failure scenarios are driven by flagd using
`src/flagd/demo.flagd.json`, edited live via the flagd-ui service.

**Ingress.** `src/frontend-proxy/envoy.tmpl.yaml` is the single routing table
for every path under `:8080`; a new UI or endpoint needs a route there.

**Telemetry schema.** `telemetry-schema/` is the Weaver registry (see
AGENTS.md); `src/telemetry-docs` renders it.

**Telemetry tests** (`test/telemetry/`) check at runtime that each service
emits traces/metrics/logs to the right backend and that expected service-to-
service edges appear. `services.py` holds the source of truth:
`SIGNAL_MATRIX` (service → signals + scope) and `SERVICE_EDGES`. A new service
or edge is added there.

## Contribution requirements

- Every source file needs the Apache-2.0 license header (`make checklicense`).
- User-visible, telemetry, config or workflow changes need a `.chloggen/`
  fragment (`make chlog-new`), otherwise the PR needs the `Skip Changelog` label.
- Most user docs live on opentelemetry.io (Demo docs), not in this repo —
  update them alongside behavior changes.
