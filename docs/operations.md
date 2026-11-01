# Operations

## Health checks

Service port `:9100` of `api`, `worker` and `agent` (no authentication, keep it off the public ingress):

| Path | Meaning |
| --- | --- |
| `/healthz` | Liveness: the process is up |
| `/readyz` | Readiness: Postgres and Kafka reachable. Nabu, the LLM and the git provider do **not** affect readiness of `api` and `worker` — an agent failure shows up as an error in the chat and a stopped run, not as a pod failure |
| `/metrics` | Prometheus metrics |

## Metrics

| Metric | What to watch |
| --- | --- |
| `hammurapi_http_requests_total{route,method,status}`, `hammurapi_http_request_duration_seconds` | Error rate and latency per route |
| `hammurapi_git_api_errors_total{provider,op}` | Provider failures (rate limits, expired tokens, outages) |
| `hammurapi_git_provider_up` | 1 when the last provider call succeeded |
| `hammurapi_webhook_events_total{result}` | `accepted`, `processed`, `unauthorized` (wrong secret), `error` |
| `hammurapi_kafka_consumer_lag{topic}` | Worker falling behind |
| `hammurapi_llm_requests_total`, `hammurapi_llm_errors_total{class,connection}` | LLM runs and their errors: balance, authorization, rate limits, outages |
| `hammurapi_llm_tokens_total{direction}` | Token use of runs of Nabu (also in Administration → Usage; cost is counted in Nabu) |
| `hammurapi_gate_transitions_total{area,to}` | Gate throughput, resets to draft |
| `hammurapi_workflow_runs{kind,state}` | Active runs of Discovery, gate generation, codegen, validation, release, rollback |
| `hammurapi_workflow_blocked_total{kind,reason}` | Runs that exhausted their attempts — each one waits for a human on the General page |
| `hammurapi_workflow_transition_duration_seconds{kind,step}` | Slow steps |
| `hammurapi_code_tasks_total{type,result,error_class}`, `hammurapi_code_start_seconds`, `hammurapi_code_apply_seconds` | Code tasks in the sandbox of Nabu, preparing the working copy and committing the changes |
| `hammurapi_code_task_states_total{type,status}`, `hammurapi_code_task_duration_seconds{type}`, `hammurapi_code_tokens_total{type}` | Code tasks entering each status, their duration and the tokens of their agents |
| `hammurapi_deploy_runs_total{environment,status}`, `hammurapi_release_rollbacks_total` | Deploys and rollbacks |

## Logs and traces

Logs are JSON on stdout with `trace_id`, `span_id`, `request_id`, `user_id` and `mode`. Traces go to
`OTEL_EXPORTER_OTLP_ENDPOINT` (OTLP/HTTP): spans for HTTP requests, provider calls, Nabu, Whisper,
Kafka produce/consume and SQL.

## Maintenance (cleaner)

`hammurapi cleaner` (CronJob, daily by default) removes:

- chat attachments older than the retention set in **Administration → Settings** (object and row);
- expired sessions and edit locks;
- import jobs not confirmed within 24 hours, with their archives;
- branches of deleted features whose deletion failed at the provider (retried with the deleting
  user's token).

## Backups

- **Postgres** holds statuses, history, roles, chat and settings — back it up.
- **The git repository** holds all document content and rules — it is backed up by your provider.
- **S3** holds chat attachments (temporary by design) and import archives (deleted after import).

Hammurapi can be restored from a Postgres backup plus the repository; statuses are not stored in
git.

## Security notes

- Provider tokens are encrypted with AES-GCM (`TOKEN_ENCRYPTION_KEY`). Rotating the key signs
  everyone out of the provider (they sign in again).
- Session cookie: `HttpOnly`, `SameSite=Lax`, `Secure` over HTTPS. Every state-changing request
  needs the CSRF double-submit header `X-CSRF-Token`.
- Webhooks without a valid secret are rejected with 401 and never reach Kafka.
- Attachments and archive files are checked by content, not extension. Archives are read in memory
  with limits on size, unpacked size and file count; `..`, absolute paths and symlinks are rejected.
- The MCP of runs (`:8083` of the worker) is never routed through the ingress; its NetworkPolicy
  admits the namespace of Nabu. MCP grants are per run; task tokens (`hmt_…`, stored as SHA-256)
  are valid only while their task runs.
- Git credentials never leave Hammurapi: Nabu gets a working copy of the branch and gives back
  the changed files; the worker commits them as the bot. Paths outside the copy and `.git` in the
  changes are dropped. The pods of Hammurapi need no Kubernetes permissions.
- CI results, deploy and feature-flag webhooks need an HMAC signature with a timestamp at most five
  minutes old; secrets are shown once and rotated with two active at a time.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| "The document changed, reload the page" on every approval | Webhooks do not arrive: check the provider's delivery log, `WEBHOOK_SECRET`, and `hammurapi_webhook_events_total` |
| Edits do not reset statuses | Same as above, or the worker is down / lagging |
| "Git provider session expired, sign in again" | The refresh token was revoked or `TOKEN_ENCRYPTION_KEY` changed |
| `api` or `worker` do not start: "NABU_URL … are required" | Nabu is required: set `NABU_URL`, `NABU_CLIENT_ID`, `NABU_CLIENT_SECRET` |
| Chat shows "The balance of the connection … ran out" / "is not authorized" | The model connection in Nabu: top up or replace its key in the administration of Nabu |
| Chat says the agent is unavailable | Nabu does not answer or the client of Hammurapi is refused; see Administration → Nabu and the logs of Nabu |
| Analysis or gate generation stops with an LLM error | The reason is on the issue or feature (class and connection); fix the connection and press "Retry" |
| Analysis or gate generation stays "the agent is working" | The service agent of Nabu is slow or `DISCOVERY_TIMEOUT` is too short; see the run's last error on the issue and the run in Nabu |
| A code task fails at once | The scenario is not bound (Administration → Nabu), the agent has no "Nabu sandbox" workspace, the working copy is over 5000 files / 200 MB, or Nabu cannot reach the worker's MCP (`NABU_WORKER_MCP_URL`, NetworkPolicy) |
| Validation waits for CI forever | The CI step does not post to `/hooks/v1/ci-results`, the signature is wrong (401 in the CI log), or test names do not contain the QA test case IDs |
| A release is blocked on merge | Branch protection, required checks or merge conflicts — the reason is on the release; fix it and retry, or roll back |
| A release waits for a deploy | The deploy job does not call `/hooks/v1/deploy` with the `runId`; mark the deploy by hand on the release |
| Voice input says recognition is unavailable | `WHISPER_URL` not set or the Whisper service is not running |
