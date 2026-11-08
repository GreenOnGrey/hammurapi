# The agent

Every user has a personal agent in a chat available on every screen, and agents do the background
work of the cycle: Analysis of issues, generation of the tech and QA gates, checks of CI results,
and code in the service repositories. The agent is **Nabu**, the corporate agent platform:
Hammurapi has no agent of its own and **does not start without Nabu** (`NABU_URL`,
`NABU_CLIENT_ID`, `NABU_CLIENT_SECRET`). Models, skills, MCP servers, sandboxes, limits, usage and
audit of the agent are configured in Nabu.

## How Hammurapi uses Nabu

- **The chat** is a window to the personal agent of the user in Nabu — the same conversation,
  memory, name and tone as in the messenger or on the site of Nabu. `api` proxies it on behalf of
  the user (delegation by email) and adds the open issue, feature or release as the context.
- **Personal agents call Hammurapi** through `/mcp/nabu` with a JWT of Nabu and the email of the
  user; the tools act with the user's roles. A user that does not exist yet is created without
  roles.
- **Background scenarios are runs of service agents of Nabu.** In **Administration → Nabu** a
  global administrator binds each scenario — issue analysis, gate generation, conformance check,
  code generation, review update, rollback revert — to a service agent the client of Hammurapi may
  use. An unbound scenario does not run: its task fails with "the scenario is not bound to an
  agent of Nabu", and the page shows a warning. Runs reach the MCP of the worker
  (`NABU_WORKER_MCP_URL`, `:8083`) with a grant for their one job.
- **Code runs in the sandbox of Nabu.** For a code task the worker downloads the branch through
  the API of the git provider (at most 5000 files, 200 MB) and uploads it to Nabu; the service
  agent changes the files in a temporary sandbox with the profile `code` (Go, Python, JavaScript)
  and the proxy of package registries, without any git credentials; the worker takes the changed
  files, commits them as the bot with trailers and opens or updates the PR. A revert and an update
  of a PR that merges cleanly need no agent. The agent of code scenarios needs the workspace
  "Nabu sandbox", the profile `code` and the proxy "Package registries" — Administration → Nabu
  warns otherwise.
- **Errors** of runs are classified and shown in plain words: the sandbox did not start, the
  working copy is too large, the time or token limit, no changes, a git error, and the classes of
  LLM errors (balance, authorization, rate limit, unavailable, bad request, context overflow,
  crash).
- **Usage** of runs stays in Hammurapi for reports: **Administration → Usage** (global
  administrators). Cost and models are counted in Nabu.

## Tools (MCP)

Each run gets a grant that decides which tools it sees.

In every scenario the agent also reads the specification of the default branch through read-only
tools over the same index as the **Specification** section (FTR.HMR.CMN-0005): `spec_tree` (domains,
systems, features), `spec_search` (full text and ID prefix, 20 per page), `spec_read` (a document
or one section; long documents come with the table of contents and the sections left out),
`spec_requirements` (R1… with acceptance criteria), `spec_references` (where a feature or a
requirement is mentioned) and `spec_links` (FTR.HMR.DEV-0002: what a feature builds on and which
requirements of other features it changes, and who builds on it or changes it; without a feature —
the graph of these links by domain, system and kind). The agent of Nabu reads them through the same
MCP server: a tool added by an upgrade shows up as new after the check of the connection `hammurapi`
in the catalog of Nabu and is used once allowed there.

The reading tools of the table below (`list_issues`, `read_issue`, `list_features`, `search_specs`,
`read_spec`, `read_rules`, `list_services`, `read_service_file`, `test_metric_query`) and `spec_*` carry the MCP annotation
`readOnlyHint`. A client that offers only read-only tools — a catalog item of Nabu with **Read only**
on — shows them and hides the tools that change data, `create_issue` among them.

| Tool | Personal agent | Discovery | Gate generation | Check | Code task |
| --- | --- | --- | --- | --- | --- |
| `list_issues`, `read_issue`, `list_features`, `search_specs` | yes | yes | yes | yes | no |
| `read_spec`, `read_rules`, `list_services` | yes | yes | yes | yes | its feature |
| `read_service_file` | no | yes | yes | yes | no (it has the working copy) |
| `test_metric_query` | with an open context | yes | no | no | no |
| `request_discovery_revision` | experts of the domain, from the chat of the issue | no | no | no | no |
| `request_spec_revision`, `read_spec_questions` | experts of the domain, from the Specification tab of the feature (`read_spec_questions`: anyone) | no | no | no | no |
| `save_discovery` | no | its issue | no | no | no |
| `submit_gate`, `submit_gate_file` | no | no | the areas of its run | no | no |
| `report_discrepancy` | no | no | no | its feature | no |
| `report_progress` | no | no | no | no | its task |

Edits made for a user commit with the user's token and the trailer `Hammurapi-Agent`, so history
shows "Agent on behalf of <name>"; background work commits as the bot with `Hammurapi-Initiator`.
There is no delete tool — irreversible actions are for people only.
