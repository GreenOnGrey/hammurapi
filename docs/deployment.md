# Deployment

**To install Hammurapi with Nabu**, use the [installer](https://github.com/GreenOnGrey/installer). It installs on one machine with minikube or into an existing Kubernetes cluster, creates the client of Hammurapi in Nabu, and handles updates. Start with [its quick start](https://github.com/GreenOnGrey/installer/blob/main/docs/quickstart.md); [products.md](https://github.com/GreenOnGrey/installer/blob/main/docs/products.md) covers the GitHub App and the webhook. This page describes the images and the chart of this repository, for a deployment of your own.

Hammurapi runs on Kubernetes with the Helm chart of this repository. It needs **Nabu** — the
agent of Hammurapi; without `NABU_URL`, `NABU_CLIENT_ID` and `NABU_CLIENT_SECRET` the chart does
not render and the services do not start.

## Images

| Image | Built from | Contents |
| --- | --- | --- |
| `hammurapi-core` | `hammurapi-core/Dockerfile`, target `release` | Static binary on distroless: `api`, `worker`, `cleaner`, `migrate` |
| `hammurapi-web` | `hammurapi-web/Dockerfile` | SPA on unprivileged nginx (port 8080) proxying `/api`, `/admin/api`, `/hooks` to `API_UPSTREAM` |

```sh
docker build --target release --build-arg VERSION=1.2.0 \
  -t registry.example.com/hammurapi-core:1.2.0 ../hammurapi-core
docker build -t registry.example.com/hammurapi-web:1.2.0 ../hammurapi-web
```

## Kubernetes (Helm)

The chart in `helm/hammurapi` deploys:

| Resource | Purpose |
| --- | --- |
| `Deployment` api | Core image, probes on `:9100`; the chat proxied to Nabu, `/mcp/nabu` for the personal agents of Nabu |
| `Deployment` worker | Kafka consumers, workflow engine, runs of the service agents of Nabu and code tasks; the MCP of runs on `:8083` (the service `<release>-worker`), which Nabu calls |
| `Deployment` web | SPA (optional, `web.enabled`) |
| `CronJob` cleaner | Daily maintenance, `cleaner.schedule` |
| `Job` migrate | `helm.sh/hook: pre-install,pre-upgrade` — migrations before the rollout |
| `Service` ×3, `Ingress` | `/api`, `/admin/api`, `/hooks` → api; `/` → web |
| `ConfigMap`, `Secret` | Configuration; or reference an existing Secret |
| `ServiceMonitor` | Optional Prometheus Operator scraping |

Postgres, Kafka, S3, Whisper and Nabu are not part of the chart — use managed services or their
own charts.

```sh
kubectl create secret generic hammurapi-secrets \
  --from-literal=DATABASE_URL='postgres://…' \
  --from-literal=TOKEN_ENCRYPTION_KEY="$(openssl rand -base64 32)" \
  --from-literal=WEBHOOK_SECRET="$(openssl rand -hex 24)" \
  --from-literal=GITLAB_CLIENT_ID=… --from-literal=GITLAB_CLIENT_SECRET=… \
  --from-literal=S3_ACCESS_KEY=… --from-literal=S3_SECRET_KEY=… \
  --from-literal=NABU_CLIENT_SECRET=… \
  --from-literal=GITLAB_BOT_TOKEN=…   --from-literal=CI_RESULTS_SECRET="$(openssl rand -hex 24)"

helm upgrade --install hammurapi ./helm/hammurapi \
  --set image.repository=registry.example.com/hammurapi-core \
  --set coreImage.repository=registry.example.com/hammurapi \
  --set web.image.repository=registry.example.com/hammurapi-web \
  --set ingress.host=hammurapi.example.com \
  --set config.PUBLIC_URL=https://hammurapi.example.com \
  --set config.GIT_REPO=product/specs \
  --set nabu.url=https://nabu-api.example.com \
  --set nabu.clientId=…
```

`image` is the core release image used by `api` and `worker`; `coreImage` overrides the image for
`cleaner` and `migrate`. If Nabu runs in another cluster, set `nabu.workerMcpUrl` to an address of
the worker's MCP that Nabu reaches. Models, skills, MCP servers and sandboxes of the agents are
configured in Nabu; bind the scenarios to its service agents in **Administration → Nabu** (see
[agent.md](agent.md)).

### Sticky sessions

A user's SSE stream (the streamed chat answer) lives in one `api` pod. `ingress.stickySessions: true` (default)
adds cookie affinity annotations for ingress-nginx; with another ingress controller, configure the
equivalent. Without affinity the answer may stream to another pod's connection and appear only after a reload.

### Scaling

- `api`: stateless; scale horizontally. The agent lives in Nabu.
- `worker`: one consumer group; Kafka partitions (6 per topic by default) bound the parallelism,
  and events of one feature are always processed in order. Workflow runs are leased with
  `SELECT … FOR UPDATE SKIP LOCKED`, so several workers share them safely.
- Code tasks: `CODE_MAX_PARALLEL` per instance and one per service repository; they run in the
  sandboxes of Nabu, so their resources are sized in Nabu.
- Kafka topics `hammurapi.git` and `hammurapi.imports` are created on start if missing.

### Validating the chart

```sh
helm lint helm/hammurapi
helm template hammurapi helm/hammurapi --set nabu.url=https://nabu-api.example.com --set nabu.clientId=x \
  | kubeconform -summary -ignore-missing-schemas
```
