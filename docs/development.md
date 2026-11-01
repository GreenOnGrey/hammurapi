# Development

Check out the repositories side by side:

```text
hammurapi/        docs, Helm chart, rules
hammurapi-core/   Go backend
hammurapi-web/    React SPA
nabu-core/        the agent platform Hammurapi requires
```

Tools: Go 1.27, Node 24, Docker (integration tests).

There is no local demo stack: Hammurapi needs Nabu, a git provider, Postgres, Kafka and S3. Run
`api` and `worker` against your own infrastructure and a Nabu instance (`NABU_URL`,
`NABU_CLIENT_ID`, `NABU_CLIENT_SECRET`, see [configuration.md](configuration.md)).

## Backend

```sh
cd hammurapi-core
make build              # bin/hammurapi
make test               # unit tests
make test-integration   # + Postgres via dockertest v4 (needs Docker)
make generate           # mocks via `go tool mockgen`
```

Run the api from your IDE: `hammurapi migrate`, then `hammurapi api` / `hammurapi worker` with
the variables of [configuration.md](configuration.md). `cmd/fakegitlab` (an in-memory imitation of
the GitLab API) is a test helper of `hammurapi-core`, not a stand.

Layout: vertical slices in `internal/features/*` (handlers, service, repository); adapters in
`internal/platform/*`; the shared projection of features and gates in `internal/specdata`, and of
issues, services, tasks and releases in `internal/cycledata`. Cycle steps are state machines on the
workflow engine (`internal/features/workflows`): a slice registers its machine and its effects, the worker
runs them. External dependencies sit behind interfaces with generated mocks.

## Frontend

```sh
cd hammurapi-web
npm ci
npm run dev             # http://localhost:5173, proxies /api and /admin/api to localhost:8080
npm test                # vitest: locales (key parity, ICU), formatting, markdown helpers
npm run build           # typecheck + production build
```

Point the dev server at another api with `HAMMURAPI_API=http://host:port npm run dev`. Sign-in
redirects to the provider and back to `PUBLIC_URL`, so for local UI work set
`PUBLIC_URL=http://localhost:5173` for the api.

## Tests

| Level | Where | What |
| --- | --- | --- |
| Unit | `hammurapi-core` `go test ./...` | Services on mockgen mocks: roles per area, sequential approval, stale approval, locks, deletion rules, webhook projection and idempotency; runs of Nabu and LLM error classes; code tasks (working copy, changes, error classes); MCP permissions; providers against `httptest`; archive parsing (zip bombs, traversal) |
| Integration | `go test -tags integration ./...` | Every repository query on Postgres 16 in Docker; migrations; concurrent numbering; locks; the workflow engine (leases, retries, blocking); the whole cycle from an issue to a release and a rollback with stubbed effects; Discovery and the chat with a fake Nabu; the usage report |
| Frontend | `hammurapi-web` `npm test` | All five locales have the same keys and valid ICU; fallback to English; Russian plurals; `Intl` formatting |

## Translations

Interface strings live in `hammurapi-web/locales/<lang>.json` (ICU MessageFormat). English is the
fallback. Add a key to every file — `npm test` fails when locales diverge. Error texts are keyed by
the API error code under `errors.*`.
