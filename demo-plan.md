# Plan: push-to-GKE demo pipeline sending telemetry to Observe

The idea is that a commit to `main` changes the running app, and Observe shows
the effect within a few minutes. There are two kinds of change, and they work
differently:

| Change | What the pipeline does | Time to show up |
| --- | --- | --- |
| `src/flagd/demo.flagd.json` (e.g. `paymentFailure` → `"50%"`) | Update the flagd ConfigMap, then restart flagd. No image build. | About 1 minute |
| `src/payment/charge.js` (a "bad deploy") | Build the payment image, push it to Artifact Registry, roll out with `helm upgrade`. | About 3–5 minutes |

Both are worth showing. The flag change is quick and easy to undo. The code
change makes the better story: error rates rise right after a new
`service.version` appears, and Observe can tie the regression to a specific
commit.

## 1. Deploy with the official Helm chart

This repo has no Kubernetes manifests, but the upstream
`open-telemetry/opentelemetry-demo` Helm chart deploys the same images. Keep a
values file in this fork, e.g. `deploy/gke/values.yaml`, containing:

- **Turn off the bundled backends** (`jaeger`, `prometheus`, `grafana`,
  `opensearch`: `enabled: false`) so Observe is the only place to look. This
  also frees up a lot of cluster memory.
- **Point the collector at Observe** (see section 2).
- **Set the payment image** to your own registry:

  ```yaml
  components:
    payment:
      imageOverride:
        repository: us-docker.pkg.dev/<PROJECT>/otel-demo/payment
        tag: <git-sha>        # set by CI with --set
      envOverrides:
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: service.version=<git-sha>   # links errors to a deploy in Observe
  ```

- **Leave the load generator on.** It produces a steady stream of checkouts,
  so payment errors actually show up.

On cluster size: the full demo with the backends turned off needs about 6–8 GB
of memory. Use either GKE Autopilot or a Standard cluster with 3×
`e2-standard-4` nodes.

## 2. Send telemetry to Observe

According to Observe's docs, the endpoint accepts **OTLP over HTTP only (no
gRPC)**: `https://<CUSTOMER_ID>.collect.observeinc.com/v2/otel`, with
`Authorization: Bearer <token>`. There's one catch: traces sent with a generic
Datastream token arrive as raw data and don't land in the Tracing/APM datasets.
For traces, Observe recommends a Kubernetes Explorer integration token plus the
header `x-observe-target-package: Tracing`. Check the exact token type and
endpoint on your tenant's **Add Data** page.

There are two ways to set this up:

- **A (recommended): install the Observe Agent Helm chart in the cluster**,
  then point the demo's collector at it with a plain `otlp` exporter. You get
  Kubernetes Explorer data (pods, nodes, events) for free, which helps the demo
  ("payment pods are fine, the code is broken"). Observe's docs also position
  this as the supported path for Kubernetes plus APM.
- **B (simpler): have the demo's collector export directly to Observe:**

  ```yaml
  opentelemetry-collector:
    extraEnvs:
      - name: OBSERVE_TOKEN
        valueFrom: { secretKeyRef: { name: observe-credentials, key: token } }
    config:
      exporters:
        otlphttp/observe:
          endpoint: https://<CUSTOMER_ID>.collect.observeinc.com/v2/otel
          headers:
            authorization: "Bearer ${env:OBSERVE_TOKEN}"
            x-observe-target-package: Tracing
      service:
        pipelines:
          traces:  { exporters: [otlphttp/observe, span_metrics] }
          metrics: { exporters: [otlphttp/observe] }
          logs:    { exporters: [otlphttp/observe] }
  ```

  **Pitfall:** Helm merges maps but replaces lists. Any pipeline you override
  needs its *complete* `exporters` list. Otherwise the chart's defaults, which
  point at the Jaeger/OpenSearch/Firepit services you turned off, will throw
  errors. Also drop the `profiles` pipeline unless you plan to show profiling.

Either way, keep the token in a Kubernetes Secret (`observe-credentials`),
created once by hand or by CI from a GitHub secret. Don't commit it to the
values file.

## 3. One-time GCP setup

1. Create the GKE cluster and a namespace (`otel-demo`).
2. Create an Artifact Registry Docker repository (`otel-demo`).
3. Set up **Workload Identity Federation** for GitHub Actions instead of a
   service-account JSON key. Create a pool and provider limited to the
   `mharner33/opentelemetry-demo` repo, and a service account with:
   - `roles/container.developer`
   - `roles/artifactregistry.writer`
4. Run the first install by hand:
   `helm install otel-demo open-telemetry/opentelemetry-demo -n otel-demo -f deploy/gke/values.yaml`.
5. Pin the chart version so CI upgrades can't pull in an unrelated chart
   release.

## 4. GitHub configuration in this fork

- **Secrets and variables:** `GCP_PROJECT_ID`, `GCP_WIF_PROVIDER`,
  `GCP_SA_EMAIL`, `GKE_CLUSTER`, `GKE_LOCATION`, `OBSERVE_TOKEN`,
  `OBSERVE_CUSTOMER_ID`.
- **Disable the upstream workflows you don't need** (`nightly-release`,
  `release`, `build-images`, `stale`, `fossa`, `ossf-scorecard`, the telemetry
  tests, and so on). Otherwise every demo push starts a lot of upstream CI, and
  some of those workflows try to publish images.
- Optionally create a `gke-demo` GitHub environment with protection rules, if
  you want an approval step before deploys.

## 5. The workflow (`.github/workflows/deploy-gke.yml`)

```yaml
on:
  push:
    branches: [main]
    paths: ['src/flagd/demo.flagd.json', 'src/payment/**', 'deploy/gke/**']
  workflow_dispatch:

permissions: { contents: read, id-token: write }

jobs:
  changes:            # dorny/paths-filter -> outputs: flags, payment, values
  flags:              # if: flags changed
    steps:
      - google-github-actions/auth (WIF)
      - google-github-actions/get-gke-credentials
      - run: |
          kubectl -n otel-demo create configmap flagd-config \
            --from-file=demo.flagd.json=src/flagd/demo.flagd.json \
            --dry-run=client -o yaml | kubectl apply -f -
          kubectl -n otel-demo rollout restart deploy/flagd
          kubectl -n otel-demo rollout status deploy/flagd
  payment:            # if: payment or values changed
    steps:
      - auth + gcloud auth configure-docker us-docker.pkg.dev
      - docker/build-push-action:
          context: .                       # repo root, same as compose.yaml
          file: src/payment/Dockerfile
          tags: us-docker.pkg.dev/$PROJECT/otel-demo/payment:${{ github.sha }}
      - get-gke-credentials
      - run: |
          helm upgrade otel-demo open-telemetry/opentelemetry-demo -n otel-demo \
            --version <pinned> -f deploy/gke/values.yaml \
            --set components.payment.imageOverride.tag=${{ github.sha }} ...
```

**Why flagd needs a restart:** the chart's flagd pod copies the ConfigMap into
a writable volume at startup, via an init container. Changing the ConfigMap
alone does nothing until the pod restarts. One side effect: the restart wipes
any changes made earlier in the flagd UI (`/feature`), so treat git as the only
source of flag state during the demo.

Optional: add a final step that posts a deployment marker to Observe (an event
with the commit SHA and message), so the deploy appears on your charts.

## 6. Demo script

1. Show a healthy baseline in Observe (the payment service map and error rate).
2. **Change the flag:** set `paymentFailure` to `"50%"`, commit, and push.
   Errors carrying `demo.user_context.loyalty_level=gold` rise within about a
   minute.
3. Revert the flag and errors drop again.
4. **Bad deploy:** edit `charge.js` (e.g. reject a card type, or throw on some
   percentage of requests without checking the flag), then push. Errors rise
   and line up with the new `service.version`.
5. Revert the commit and the pipeline rolls back automatically.

Keep these demo changes on this fork's `main` and never open PRs upstream. The
repo's `AGENTS.md` is strict about AI-generated content going into upstream
PRs.

## Open decisions

1. **Observe Agent (A) or exporting directly from the collector (B)?**
   Recommendation: A.
2. **Push-based CI (above), or GitOps with Argo CD watching the repo?**
   Push-based is simpler, and seeing the GitHub Action run makes a good live
   visual.
3. **Is a cluster and an Observe token already available?** If not, start
   with section 3.

## Sources

- [Observe: Send data from an existing OpenTelemetry collector](https://docs.observeinc.com/en/latest/content/send-data/otel-collector.html)
- [Observe: OTel endpoint](https://docs.observeinc.com/en/aws-surgery/content/data-ingestion/endpoints/otel.html)
- [Observe: Configure application instrumentation](https://docs.observeinc.com/en/latest/content/observe-agent/ConfigureApplicationInstrumentation.html)
- [Observe: Configure your own OTel collector without Kubernetes](https://docs.observeinc.com/docs/configure-your-own-otel-collector-in-a-non-kubernetes-environment)
- [opentelemetry-demo Helm chart values.yaml](https://raw.githubusercontent.com/open-telemetry/opentelemetry-helm-charts/main/charts/opentelemetry-demo/values.yaml)
