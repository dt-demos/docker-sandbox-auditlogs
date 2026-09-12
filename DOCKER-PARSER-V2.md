# Docker Sandbox Audit Log — OpenPipeline Parser V2 

**Approach:** 
* Custom `docker.audit.*` attributes for key fields — direct DQL filtering without parsing.
    * All source Docker fields preserved in `content` as a JSON string.
    * Use `parse content, "JSON:json"` in DQL to access any field not promoted to a named attribute.
* Global log fields (`log.source`, `host.name`, `loglevel`) follow the [Dynatrace Log and Audit Semantic Dictionary](https://docs.dynatrace.com/docs/semantic-dictionary/model/log).

**Pipeline file:** `pipeline-docker-v2.json`  

---

## Design

Five processors run in order on every ingest record:

| # | Processor | Matcher |
|---|---|---|
| 1 | Build `content` JSON | always |
| 2 | Map `log.source`, `host.name`, `docker.audit.*` | always |
| 3 | Pull the risk band classification out of the `c_score_report` payload and promote it to the named attribute `docker.audit.cscore_band` so it can be filtered directly without parsing `content` | `c_score_report` records only |
| 4 | Set `loglevel` | always |
| 5 | Remove all original Docker fields | always |

---

## Named attributes after processing

| Model | Attribute | Value | Source |
|---|---|---|---|
| Log | `log.source` | `"docker-sandbox-auditlogs"` | static |
| Log | `host.name` | machine hostname | `hostname` |
| Log | `loglevel` | `ERROR` / `WARN` / `INFO` | calculated (see below) |
| Docker | `docker.audit.org_id` | Docker org UUID | `org_id` |
| Docker | `docker.audit.session_id` | governance session UUID | `audit_session_id` |
| Docker | `docker.audit.category` | `AUDIT_CATEGORY_*` | `category` |
| Docker | `docker.audit.action_type` | `network_egress`, `tool_invocation`, … | `action_type` |
| Docker | `docker.audit.decision` | `AUDIT_DECISION_*` | `decision` |
| Docker | `docker.audit.enforcement_mode` | `audit` / `warn` / `enforce` / `""` | `enforcement_mode` |
| Docker | `docker.audit.cscore_band` | `C_SCORE_BAND_*` | `c_score_report` payload `band` field |
| Log | `content` | full record JSON (see below) | all Docker fields |

Everything else is accessed via `parse content, "JSON:json"` in DQL.

### How `loglevel` is calculated

| `decision` | `enforcement_mode` | `loglevel` |
|---|---|---|
| `AUDIT_DECISION_DENY` | `enforce` | `ERROR` |
| `AUDIT_DECISION_DENY` | any other / absent | `WARN` |
| anything else | any | `INFO` |

---

## `content` field structure

Processor 1 runs first and serializes all incoming Docker record fields into a single JSON string stored in `content`. This happens before any fields are removed, so nothing is lost. Processor 5 then drops all the original top-level source fields, leaving `content` as the only place the full record data lives.

Each Docker audit record carries exactly one action payload (e.g. `network_egress`, `tool_invocation`). Rather than keeping whichever payload field arrived, P1 merges all possible payload field names into a single `payload` key so the structure of `content` is consistent regardless of action type.

| Key | Docker source field |
|---|---|
| `audit_event_id` | `audit_event_id` |
| `timestamp` | `timestamp` |
| `schema_version` | `schema_version` |
| `category` | `category` |
| `decision` | `decision` |
| `action_type` | `action_type` |
| `enforcement_mode` | `enforcement_mode` |
| `username` | `username` |
| `user_email` | `user_email` |
| `auth_method` | `auth_method` |
| `oauth_client_id` | `oauth_client_id` |
| `agent` | `agent` |
| `org_id` | `org_id` |
| `org_name` | `org_name` |
| `audit_session_id` | `audit_session_id` |
| `sandbox_id` | `sandbox_id` |
| `resource_id` | `resource_id` |
| `policy_id` | `policy_id` |
| `policy_rule` | `policy_rule` |
| `policy_name` | `policy_name` |
| `policy_type` | `policy_type` |
| `policy_source` | `policy_source` |
| `policy_version` | `policy_version` |
| `os` | `os` |
| `app_version` | `app_version` |
| `client_name` | `client_name` |
| `hostname` | `hostname` |
| `payload` | first non-null of all action payload fields (`tool_invocation`, `network_egress`, `filesystem_mount`, `session`, `c_score_report`, … `policy_action`); `null` if none present |

> **Note:** `payload` is embedded as a JSON object (not a string), so DQL accesses sub-fields as `json[payload][field]` — not via a second `parse`.

---

## DQL query patterns

```dql
// Base filter — all Docker governance events
fetch logs
| filter log.source == "docker-sandbox-auditlogs"

// Filter without parsing content (use named attributes)
fetch logs
| filter log.source == "docker-sandbox-auditlogs"
| filter docker.audit.decision == "AUDIT_DECISION_DENY"
| summarize count(), by:{docker.audit.action_type}

// Access fields inside content
fetch logs
| filter log.source == "docker-sandbox-auditlogs"
| parse content, "JSON:json"
| filter json[org_name] == "aigovdemo"
| fields timestamp, json[username], json[agent], docker.audit.action_type, docker.audit.decision

// Access nested payload field (e.g. network egress port)
fetch logs
| filter log.source == "docker-sandbox-auditlogs"
| filter docker.audit.action_type == "network_egress"
| parse content, "JSON:json"
| fields json[resource_id], json[payload][destination_port], json[payload][protocol]

// C-Score risk band — direct attribute, no parse needed
fetch logs
| filter log.source == "docker-sandbox-auditlogs"
| filter docker.audit.cscore_band == "C_SCORE_BAND_RED" or docker.audit.cscore_band == "C_SCORE_BAND_ORANGE"
| parse content, "JSON:json"
| fields timestamp, json[username], docker.audit.cscore_band, json[payload][c_score]
```

---

## Dashboards

Two dashboards ship with this repo, both written for the V2 parser's `log.source` and field layout:

| File | Description |
|---|---|
| `docker-dashboard-policy-enforcement-v2.yaml` | Decision rates, enforcement mode, denied actions/users, policy rule frequency |
| `docker-dashboard-risk-monitor-v2.yaml` | C-Score risk bands, PII detection, high-risk session detail |

See [README.md](README.md) for install instructions.
