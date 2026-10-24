# GKE + Observe demo deployment

`.github/workflows/deploy-gke.yml` deploys this fork to GKE on every push to
`main` that touches:

- `src/flagd/demo.flagd.json`: syncs the `demo-flagd-config` ConfigMap and
  restarts flagd (about 1 minute).
- `src/payment/**` or `deploy/gke/**`: builds the payment image, pushes it to
  Artifact Registry tagged with the short commit SHA, and runs `helm upgrade`.
  The payment service reports that SHA as `service.version`.

Run it manually (Actions → Deploy to GKE → Run workflow) for the first install;
a manual run does both.

Telemetry goes from the demo's collector to the Observe Agent forwarder in the
`observe` namespace, which sends it to Observe. The bundled Jaeger, Prometheus,
Grafana and OpenSearch are disabled.

## One-time setup

```sh
export PROJECT_ID=<gcp-project>
export REGION=us-central1
export CLUSTER=otel-demo
export REPO=mharner33/opentelemetry-demo
export PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')
export SA=github-deployer@$PROJECT_ID.iam.gserviceaccount.com
```

### 1. Cluster and registry

A Standard cluster is recommended. The Observe Agent reads node logs through
hostPath mounts, which Autopilot restricts.

```sh
gcloud container clusters create "$CLUSTER" --project "$PROJECT_ID" \
  --region "$REGION" --num-nodes 1 --machine-type e2-standard-4 \
  --workload-pool "$PROJECT_ID.svc.id.goog"

gcloud artifacts repositories create otel-demo --project "$PROJECT_ID" \
  --location "$REGION" --repository-format docker
```

### 2. Workload Identity Federation for GitHub Actions

```sh
gcloud iam workload-identity-pools create github --project "$PROJECT_ID" --location global

gcloud iam workload-identity-pools providers create-oidc github-repo \
  --project "$PROJECT_ID" --location global --workload-identity-pool github \
  --issuer-uri https://token.actions.githubusercontent.com \
  --attribute-mapping google.subject=assertion.sub,attribute.repository=assertion.repository \
  --attribute-condition "assertion.repository=='$REPO'"

gcloud iam service-accounts create github-deployer --project "$PROJECT_ID"

gcloud iam service-accounts add-iam-policy-binding "$SA" --project "$PROJECT_ID" \
  --role roles/iam.workloadIdentityUser \
  --member "principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github/attribute.repository/$REPO"

# container.admin rather than container.developer: the chart creates a
# ClusterRole and ClusterRoleBinding for the collector.
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:$SA" --role roles/container.admin

gcloud artifacts repositories add-iam-policy-binding otel-demo --project "$PROJECT_ID" \
  --location "$REGION" --member "serviceAccount:$SA" --role roles/artifactregistry.writer
```

### 3. Observe Agent

Create a Kubernetes ingest token on your Observe tenant's **Add Data** page,
then:

```sh
gcloud container clusters get-credentials "$CLUSTER" --region "$REGION" --project "$PROJECT_ID"

kubectl create namespace observe
kubectl -n observe create secret generic agent-credentials \
  --from-literal=OBSERVE_TOKEN=<ingest-token>

helm repo add observe https://observeinc.github.io/helm-charts
helm install observe-agent observe/agent --namespace observe --version 0.96.2 \
  --values deploy/gke/observe-agent-values.yaml \
  --set observe.collectionEndpoint.value=https://<CUSTOMER_ID>.collect.observeinc.com/ \
  --set cluster.name="$CLUSTER"
```

Keep the release name `observe-agent`: `values.yaml` sends to the
`observe-agent-forwarder` Service it creates.

### 4. GitHub repository variables

Settings → Secrets and variables → Actions → Variables:

| Variable | Value |
| --- | --- |
| `GCP_PROJECT_ID` | `$PROJECT_ID` |
| `GCP_WIF_PROVIDER` | `projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github/providers/github-repo` |
| `GCP_SA_EMAIL` | `$SA` |
| `GKE_CLUSTER` | `$CLUSTER` |
| `GKE_LOCATION` | `$REGION` |
| `GAR_LOCATION` | `$REGION` |

No secrets are needed in GitHub: authentication uses Workload Identity
Federation, and the Observe token stays in the cluster. The workflow runs in
the `gke-demo` environment, where you can add a required reviewer if you want
an approval step.

### 5. Disable the upstream workflows

They run on every push to `main` in a fork and build or test the whole demo:

```sh
for wf in checks.yml build-images.yml run-telemetry-tests.yml \
  run-agentic-telemetry-tests.yml fossa.yml ossf-scorecard.yml zizmor.yml \
  gradle-wrapper-validation.yml nightly-release.yml codeql-analysis.yml stale.yml; do
  gh workflow disable "$wf" --repo "$REPO"
done
```

## Notes

- The flagd UI (`/feature`) edits a copy of the flags inside the flagd pod. The
  next flag deploy restarts flagd and resets it to the file in git.
- The chart version is pinned in the workflow (`CHART_VERSION`). Its
  `appVersion` should match `IMAGE_VERSION` in `.env`, so the payment image
  built from this repo matches the other services.
- To roll back a bad payment change, revert the commit and push.
