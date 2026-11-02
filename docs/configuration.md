# Configuration

Every setting of a Hammurapi instance is an environment variable. Secrets come from a Kubernetes
Secret (Helm). Empty values mean "use the default".

`api`, `worker` and `cleaner` read the same variables; `migrate` only needs `DATABASE_URL`.
`api` and `worker` do not start without Nabu (see [Nabu](#nabu)).

## URLs and listeners

| Variable | Purpose | Default |
| --- | --- | --- |
| `PUBLIC_URL` | URL people open. Used for the OAuth callback (`<PUBLIC_URL>/api/v1/auth/callback`); `https://` turns on `Secure` cookies | `http://localhost:8080` |
| `HTTP_ADDR` | User API, admin API and webhooks | `:8080` |
| `SERVICE_ADDR` | `/healthz`, `/readyz`, `/metrics` (no authentication — keep it internal) | `:9100` |
| `WORKER_MCP_ADDR` | MCP endpoint of the worker for the runs of service agents of Nabu (Discovery, gate generation, checks, code tasks) | `:8083` |
| `WORKER_MCP_URL` | Address of that endpoint inside the network; the default of `NABU_WORKER_MCP_URL` | `http://localhost:8083` |
| `HOOKS_URL` | Base URL of `/hooks/v1/*` given to deploy systems as the callback | `PUBLIC_URL` |

## Git provider

One provider and one repository per instance.

| Variable | Purpose | Default |
| --- | --- | --- |
| `GIT_PROVIDER` | `github` or `gitlab` | — (required) |
| `GIT_BASE_URL` | Provider URL as seen from the server (self-hosted GitLab, GitHub Enterprise) | `https://github.com` / `https://gitlab.com` |
| `GIT_OAUTH_URL` | Provider URL as seen from browsers, if different (split DNS) | `GIT_BASE_URL` |
| `GIT_REPO` | Specifications repository, `owner/name` | — (required) |
| `GIT_DEFAULT_BRANCH` | Main branch that features are merged into | `main` |
| `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY` | GitHub App identity | — |
| `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` | GitHub App OAuth credentials (user-to-server tokens) | — (required for GitHub) |
| `GITLAB_CLIENT_ID`, `GITLAB_CLIENT_SECRET` | GitLab OAuth application | — (required for GitLab) |
| `WEBHOOK_SECRET` | Shared with the webhooks of the spec, service and catalog repositories. GitLab sends it as `X-Gitlab-Token`; GitHub signs with it (`X-Hub-Signature-256`) | — (required for `api`) |
| `GITLAB_BOT_TOKEN` | GitLab: token of the bot account that commits, merges and triggers pipelines (GitHub uses the App) | — |
| `HAMMURAPI_BOT_LOGIN` | Login of the bot (`<app-slug>[bot]` on GitHub); its commits and reviews are not counted as human | — |

See [git-providers.md](git-providers.md) and [cycle.md](cycle.md).

## Development cycle

| Variable | Purpose | Default |
| --- | --- | --- |
| `CI_RESULTS_SECRET` | HMAC secret of `/hooks/v1/ci-results`; two comma-separated values while rotating | — (CI results are refused) |
| `CODE_TIMEOUT` | Maximum duration of a code task in the sandbox of Nabu (`RUNNER_TIMEOUT` is read for one release) | `2h` |
| `CODE_MAX_PARALLEL` | Code tasks running at once per instance; one per service repository (`RUNNER_MAX_PARALLEL` is read for one release) | `10` |
| `WORKFLOW_MAX_ATTEMPTS` | Attempts of an effect (deploy, merge, agent call…) before the run is blocked | `8` |
| `WORKFLOW_LEASE` | How long a worker holds a run before another may take it | `2m` |
| `WORKFLOW_DISPATCHERS` | How many effects a worker runs at once; an effect that runs an agent holds one for minutes, the runs are processed by a loop of their own | `4` |
| `DISCOVERY_TIMEOUT` | Maximum duration of one Discovery session | `20m` |
| `WRITER_TIMEOUT` | Maximum duration of one run of the writer of specifications | `30m` |

Deploy settings, the feature-flag webhook, the Backstage catalog, stage and metric sources are not
environment variables: administrators set them in the web app (see [cycle.md](cycle.md)).

## Specification index

The worker checks the default branch of the specification repository and indexes specifications
added past Hammurapi as implemented; the index serves the **Specification** section, its search and
the agent's `spec_*` tools. The period of the check is not an environment variable: a global
administrator sets it in **Administration → Settings** (default every hour).

| Variable | Purpose | Default |
| --- | --- | --- |
| `SPEC_SCAN_PUSH_DEBOUNCE` | Delay of the check after a push to the default branch (a series of pushes gives one check) | `30s` |
| `SPEC_SCAN_PUSH_DEBOUNCE_MAX` | Pushes never delay the check longer than this after the first one | `5m` |
| `SPEC_SCAN_MAX_FILE_BYTES` | A larger `spec.md` is searchable by its title and headings only | `2MB` |
| `SPEC_SCAN_TIMEOUT` | Limit of one check | `15m` |
| `SPEC_FILE_PREVIEW_MAX_BYTES` | Larger files of an area are offered for download only | `5MB` |
| `SPEC_SEARCH_MAX_LIMIT` | Results per page of `/api/v1/spec/search` at most | `50` |
| `SPEC_AGENT_READ_MAX_CHARS` | `spec_read` returns longer documents by sections | `40000` |
| `SPEC_AGENT_PAGE_SIZE` | Page of `spec_search` and `spec_references` | `20` |

## Users

| Variable | Purpose | Default |
| --- | --- | --- |
| `BOOTSTRAP_ADMINS` | Emails (or, transitionally, git logins), comma-separated, that become global administrators on their **first** sign-in | — |
| `DEFAULT_LANGUAGE` | Interface language before sign-in when the browser language is not supported, and for new users: `en`, `ru`, `de`, `es`, `zh-CN` | `en` |

## Sign-in

Sign-in goes through one provider of the deployment (FTR.HMR.CMN-0006). Edits of specifications
still commit on behalf of the user, so a user who signs in through OIDC links a git account in the
profile menu; the interface offers it when a change needs it.

| Variable | Purpose | Default |
| --- | --- | --- |
| `AUTH_PROVIDER` | Empty — the OAuth of the git provider, as before; `github` — GitHub with the email and an optional organization; `oidc` — a corporate provider (Keycloak, ADFS…) | — |
| `GITHUB_ALLOWED_ORG` | `github`: only members of this organization may sign in | — |
| `GITHUB_LOGIN_CLIENT_ID`, `GITHUB_LOGIN_CLIENT_SECRET` | `github`: a separate OAuth App for sign-in (callback `<PUBLIC_API_URL>/api/v1/auth/callback`, scopes `read:user user:email read:org`); required when the git provider is GitLab. Empty — sign-in through the app of the GitHub git provider | — |
| `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` | `oidc`: the issuer and the client (PKCE, verified email required) | — |
| `OIDC_SCOPES`, `OIDC_PROVIDER_NAME` | `oidc`: scopes and the name on the sign-in button | `openid email profile`, `Keycloak` |

With `github` and no separate app, sign-in uses the GitHub App of the git provider and links the git
account at once; give the App the permissions **Email addresses: read** (account) and **Members:
read** (organization) so that emails and the organization can be checked. With a separate sign-in
app or OIDC the user links the git account in the profile menu (and may unlink it).

A user who signs in through a new provider is matched with the earlier account by a verified email
(collected from the git provider in the background after the upgrade). Probable matches without an
email wait for a global administrator in **Admin → Users**.

## Nabu

Nabu is the agent of Hammurapi and is **required**: `api` and `worker` do not start without
`NABU_URL`, `NABU_CLIENT_ID` and `NABU_CLIENT_SECRET`. The chat is the user's personal agent in
Nabu, and the scenarios of the cycle run on service agents of Nabu bound in
**Administration → Nabu**. Register Hammurapi in Nabu (**Nabu → Administration → Clients**, right
`delegate`, service agents allowed), then set:

| Variable | Purpose | Default |
| --- | --- | --- |
| `NABU_URL` | Address of the Nabu API, e.g. `https://nabu-api.example.com` | — (required) |
| `NABU_CLIENT_ID`, `NABU_CLIENT_SECRET` | The service client of Hammurapi in Nabu | — (required) |
| `NABU_CLIENT_NAME` | The name of that service client in Nabu: Nabu names the channel of messages written in Hammurapi `client:<name>`, and only this channel may give instructions to the researcher of Discovery (FTR.HMR.DISC-0001 R6) | `hammurapi` |
| `NABU_JWKS_CACHE` | How long the signing keys of Nabu are cached | `10m` |
| `NABU_WORKER_MCP_URL` | The MCP of the worker as the service agents of Nabu reach it | `WORKER_MCP_URL/mcp` |
| `NABU_WEB_URL` | The site of Nabu: global administrators get a link from a code task to the runs of Nabu | — (no link) |

The personal agents of Nabu call `POST <PUBLIC_API_URL>/mcp/nabu` with a JWT of Nabu (audience
`hammurapi`) and `Nabu-On-Behalf-Of: <email>`; Hammurapi applies the rights of that user.

Models, skills, MCP servers, sandboxes and proxies of the agents are configured in Nabu. Code
agents need the workspace "Nabu sandbox", the profile `code` and the proxy "Package registries".
See [agent.md](agent.md).

## Infrastructure

| Variable | Purpose | Default |
| --- | --- | --- |
| `DATABASE_URL` | Postgres 16 connection string | — (required) |
| `KAFKA_BROKERS` | Comma-separated brokers | — (required for `api`, `worker`) |
| `S3_ENDPOINT` | S3/MinIO host:port | — (required) |
| `S3_BUCKET` | Bucket for attachments and import archives (created if missing) | `hammurapi` |
| `S3_ACCESS_KEY`, `S3_SECRET_KEY` | S3 credentials | — |
| `S3_USE_SSL` | Use HTTPS for S3 | `false` |
| `WHISPER_URL` | [whisper-asr-webservice](https://github.com/ahmetoner/whisper-asr-webservice) base URL (model `small`) | — (voice input fails without it) |
| `TOKEN_ENCRYPTION_KEY` | 32 bytes, base64 or hex; encrypts users' provider tokens (AES-GCM). Changing it forces everyone to sign in again | — (required) |

## Limits

| Variable | Purpose | Default |
| --- | --- | --- |
| `UPLOAD_MAX_BYTES` | Chat attachment size limit | `20971520` (20 MB) |
| `UPLOAD_ALLOWED_TYPES` | Comma-separated MIME types, detected from content, not extension | images (JPEG, PNG, GIF, WebP, HEIC), Office (DOC/DOCX/XLS/XLSX/PPT/PPTX), TXT, PDF |
| `IMPORT_MAX_BYTES` | Import archive size | `52428800` (50 MB) |
| `IMPORT_MAX_UNCOMPRESSED_BYTES` | Unpacked size of an archive (zip-bomb protection) | `209715200` (200 MB) |
| `IMPORT_MAX_FILES` | Files in an archive | `500` |
| `IMPORT_ALLOWED_ASSET_TYPES` | Types allowed next to `spec.md` in an archive | PNG, JPEG, GIF, WebP, SVG, PDF, HTML |

The retention period of chat attachments is not an environment variable: a global administrator
sets it in **Administration → Settings** (default 90 days).

## Observability

| Variable | Purpose | Default |
| --- | --- | --- |
| `LOG_LEVEL` | `debug`, `info`, `warn`, `error` (JSON logs on stdout) | `info` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP/HTTP endpoint for traces; the standard `OTEL_*` variables apply | — (no export) |
