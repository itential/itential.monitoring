# Kibana Users Guide — Itential Log Observability

## Table of Contents

- [Overview](#overview)
- [Setting Up Index Patterns](#setting-up-index-patterns)
- [Searching Logs in Discover](#searching-logs-in-discover)
- [Tracing an Automation Workflow End-to-End](#tracing-an-automation-workflow-end-to-end)
- [Key Fields Reference](#key-fields-reference)
- [Recommended Alerting Rules](#recommended-alerting-rules)
- [Related Documents](#related-documents)

---

## Overview

Once the ELK stack is deployed and Filebeat is shipping logs, Kibana provides the
operational interface for log search, dashboards, and alerting.

Access Kibana at `https://<kibana-host>:5601`.

---

## Setting Up Index Patterns

Before searching, create an index pattern that covers all Itential log indices:

1. Navigate to **Stack Management → Kibana → Data Views**
2. Click **Create data view**
3. Set the index pattern to `itential-logs-*`
4. Set the time field to `@timestamp`
5. Click **Save data view to Kibana**

Optionally create a second data view for `itential-failures-*` to focus searches on
automation failures specifically.

---

## Searching Logs in Discover

Navigate to **Discover** and select the `itential-logs-*` data view.

The search bar accepts KQL (Kibana Query Language). Common queries for Itential
operations:

**Find all automation failures in the last 24 hours:**
```
tags: "automation_failure"
```

**Find all events for a specific workflow by name:**
```
itential.job.name: "Provision Device"
```

**Find all failed jobs for a specific workflow:**
```
itential.job.name: "Provision Device" and itential.job.status: (error or failed)
```

**Find all IAP webserver API calls by a specific user:**
```
app: "itential-platform-web" and itential.user: "jsmith"
```

**Find all gateway connectivity failures:**
```
tags: "gateway_connectivity_failure"
```

**Find all slow MongoDB queries:**
```
tags: "slow_query"
```

**Find all events for a specific IAP sub-process:**
```
itential.process: "WorkFlowEngine"
```

---

## Tracing an Automation Workflow End-to-End

When a workflow failure needs to be investigated across IAP and IAG5, use the following
sequence:

**Step 1 — Find the job in the webserver log.**

Filter on the IAP webserver index and search for the job submission event. The IAP
Job ID appears in the URL of `POST /jobs/start` response and subsequent `GET /jobs/{id}`
polling requests.

```
app: "itential-platform-web" and itential.http.url: "/jobs/start"
```

Note the IAP Job ID from the response URL.

**Step 2 — Locate the gateway dispatch.**

Search the GatewayManager logs for the dispatch event using the time window of the job:

```
itential.process: "GatewayManager"
```

The GatewayManager log contains the **Gateway Dispatch UUID** — the cross-boundary
correlation key between IAP and IAG5. This UUID appears in both `GatewayManager.log`
and IAG5's `gateway.log`.

**Step 3 — Trace the IAG5 execution.**

Search across all indices using the Gateway Dispatch UUID:

```
itential.job.id: "<gateway-dispatch-uuid>"
```

This surfaces all IAG5 events associated with the dispatch — device connection,
command execution, and the final outcome — providing the complete execution trail
in a single result set.

---

## Key Fields Reference

The Logstash pipeline normalizes log data into a consistent set of fields. The most
useful fields for searching and filtering are:

| Field | Description | Source |
|---|---|---|
| `app` | Application identifier — set by Filebeat input | All sources |
| `itential.process` | IAP sub-process name (e.g. `WorkFlowEngine`, `GatewayManager`) | IAP |
| `itential.job.id` | Workflow job ID or gateway dispatch UUID | IAP, IAG5 |
| `itential.job.name` | Workflow name | IAP WorkFlowEngine |
| `itential.job.status` | Job execution status | IAP WorkFlowEngine |
| `itential.job.duration_ms` | Job execution time in milliseconds | IAP WorkFlowEngine |
| `itential.task.id` | Task ID within a workflow | IAP WorkFlowEngine |
| `itential.task.name` | Task name within a workflow | IAP WorkFlowEngine |
| `itential.gateway.name` | IAG5 gateway instance name | IAP GatewayManager |
| `itential.user` | User who made an API call | IAP webserver |
| `itential.http.method` | HTTP method of the API call | IAP webserver |
| `itential.http.url` | API endpoint URL | IAP webserver |
| `itential.http.status` | HTTP response status code | IAP webserver |
| `itential.http.duration_ms` | API response time in milliseconds | IAP webserver |
| `itential.client_ip` | Client IP address of the API caller | IAP webserver |
| `tags` | Failure tags: `automation_failure`, `gateway_connectivity_failure`, `slow_query` | Logstash |
| `environment` | Environment label set in Filebeat (e.g. `production`, `staging`) | All sources |
| `@timestamp` | Normalized event timestamp | All sources |

---

## Recommended Alerting Rules

Configure alerting rules under **Observability → Alerts** to detect critical failure
conditions proactively:

| Rule | Condition | Suggested Severity |
|---|---|---|
| Automation failure spike | `automation_failure` tag count > 5 in 5 min | Critical |
| Gateway connectivity loss | `gateway_connectivity_failure` tag count > 3 in 5 min | Critical |
| Platform auth failures | Failed auth events > 10 in 5 min | High |
| MongoDB slow query surge | `slow_query` tag count > 20 in 10 min | Warning |
| Job queue depth rising | `itential.job.status: queued` count > 50 | Warning |
| Filebeat shipping lag | No new documents in `itential-logs-*` for 5 min | High |

> **Note:** Email and webhook alert connectors are included in the free Basic tier.
> PagerDuty and Slack connectors may require a paid Standard subscription for
> self-managed deployments.

---

## Related Documents

- [ELK Installation Guide](elk_guide.md)
- [High Level Design — Itential + Elastic Stack](itential_elk_hld_v7.md)
- [Low Level Design — Itential + Elastic Stack](itential_elk_lld.md)
