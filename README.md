# Azure SRE Agent Canvas Runbook

Use Azure SRE Agent's diagnosis as the starting point, then inspect the evidence independently in your developer workflow: reconstruct the failure check, follow request/dependency/exception records, widen the scope, and test competing explanations. This runbook includes complete Log Analytics queries, Copilot prompts, execution approvals, and interpretation guidance for the [Azure SRE Agent canvas demo](https://aka.ms/sreagent/canvas/demo).

The example follows a September 22 demo incident in which gateway health passed while Orders and payments failed. Adapt the queries to your own authorized workspace, incident window, and observed dependency targets. **Private identifiers and endpoints have been replaced with placeholders; this repository does not provide access to the demonstration environment.**

See the [canvas guide](https://aka.ms/sreagent/canvas/doc) for setup and the [companion blog](https://aka.ms/sreagent/canvas/blog) for the investigation story.

**Case:** `demo1-dt-snow` / **INC0922001**, "Trading order and payment requests failing while gateway health remains available."

**Correct investigation:** connect to the thread returned for **INC0922001** in your authorized demo environment. **INC0922002** is a later recurrence; its 20:23 stop and recovery evidence must not be used for this first-incident window. When using your own incident, select its actual returned thread rather than copying a demonstration identifier.

**Source and window:** your authorized Log Analytics workspace, identified below as `YOUR_WORKSPACE_ID`; **2026-09-22T17:30:00Z <= TimeGenerated < 2026-09-22T19:45:00Z** for the saved baseline. The recorded demo also explores a narrower 18:33-19:03 UTC window. Do not compare their counts as if they were the same query or population.

The linked investigation reports a stopped database. The saved logs independently show connectivity timeouts, not historical database state. **Q1 is a reconstructed query with a saved result, not the exposed original SRE execution. Q2, Q3, Q3C, Q4, and Q5 are complete follow-up drafts, not yet run.** Their columns are grounded in the table references below; populated values and live execution remain unverified.

### What the first SRE record says

The first incident records its PostgreSQL database as **Stopped**, and all three Container Apps as provisioning **Succeeded**. Its recorded probes were `/health` 200 at 0.39 seconds, `/api/orders` 500 with a database connection timeout at 15.2 seconds, and `/api/payments` 502 wrapping a downstream 500 at 15.1 seconds. These are **SRE-recorded state and probe observations**, not newly repeated resource or latency checks. Provisioning success is not business-API health.

That incident's Activity Log check found **no stop event in the past week**, with possible scope limitations. It does not contain the later recurrence's manual-stop proof. Its recovery command required approval; the prior read did not confirm completed recovery. Keep both the stop actor/cause and recovery outcome open.

The SRE snapshot's **111 Orders / 108 payments** counts differ from the fixed-window saved replay's **120 / 115 gateway records**. This reconstruction is not an exact original-query replay, and the discrepancy is not itself a product-health change. Preserve query/window/population differences before comparing counts.

SRE already identified both Orders and payments impact, the early failure timestamps, and roughly 15-second failed probes. Widening our own Orders-focused query independently checks that scope; it is not a claim that SRE overlooked payments or latency. New success/latency/capacity measurements require their own actual execution.

## Run route and approvals

Use the **Azure CLI / Log Analytics API** route under your existing, explicitly chosen signed-in identity. You need Azure CLI, Node.js, and permission to query the selected workspace. Do not silently switch accounts/profiles or register/replace a connector. A Log Analytics API endpoint is not interchangeable with a Kusto cluster URL.

Before execution, replace `YOUR_WORKSPACE_ID` with your approved workspace ID. Q3 and Q4 also require the exact `YOUR_ORDERS_DEPENDENCY_TARGET` and `YOUR_DATABASE_HOST` observed in your reviewed results. Adapt the historical interval when investigating a different incident, and confirm that the tables and retained data exist. Do not run unchanged placeholders and interpret an empty result as evidence of health.

For each query, copy its entire KQL block into `la-query.kql` in a dedicated local directory. The following commands create the JSON body with **only `query`**, then execute the reviewed request. They are instructions for a separately approved run, not operations performed by this repository.

```bash
WORKSPACE_ID='YOUR_WORKSPACE_ID'
node -e 'const fs = require("node:fs"); fs.writeFileSync("la-query.json", JSON.stringify({query: fs.readFileSync("la-query.kql", "utf8")}, null, 2) + "\n");'
az rest \
  --method POST \
  --url "https://api.loganalytics.io/v1/workspaces/${WORKSPACE_ID}/query" \
  --resource https://api.loganalytics.io \
  --headers Content-Type=application/json \
  --body @la-query.json \
  > la-results.json
```

Keep a distinct copy of each reviewed query and its actual result before replacing the files for the next stage. A failed request, partial/error response, or empty result is not a successful replay. Preserve the error; do not change endpoint or broaden scope silently. If using Portal or an API `timespan` separately, set it to this same historical interval, not the default last 24 hours. Each KQL block also fixes the interval explicitly.

**Preparation prompt**

> Prepare Q1 from this runbook for my approved workspace, September 22 from 17:30 to 19:45 UTC. Show the complete KQL, workspace ID, exact az rest command, and output file. This is a read-only Log Analytics query through my existing CLI identity. Do not execute, change profiles, or change the provider.

**Execution approval template**, reused with the reviewed query label:

> Run only the Q1 query and command you just showed, against that workspace and historical window. Keep the response in the stated file. Stop on an error or partial result; do not retry with another source or wider scope.

This approval does not cover Q2 or another changed query, metadata reads, historical configuration reads, source activation, sharing with SRE, or Teams publishing. Review and approve each separately. Local CLI execution uses the developer's identity and network; a cloud connector uses its provider's path.

## Q1: reconstructed baseline, saved result

This is the **complete KQL** used for the saved baseline. It has **no app-name, Orders-operation, or dependency-target filter**. Every saved `AppRoleName` is blank; adding an app-name filter would discard the observed rows.

```kusto
let startTime = datetime(2026-09-22T17:30:00Z);
let endTime = datetime(2026-09-22T19:45:00Z);
union
    (AppRequests
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where Success == false or toint(ResultCode) >= 500
     | project TimeGenerated, Table="AppRequests", AppRoleName, Detail=Name, ResultCode=tostring(ResultCode)),
    (AppExceptions
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | project TimeGenerated, Table="AppExceptions", AppRoleName, Detail=strcat(ExceptionType, ": ", OuterMessage), ResultCode=""),
    (AppDependencies
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where Success == false or toint(ResultCode) >= 500
     | project TimeGenerated, Table="AppDependencies", AppRoleName, Detail=strcat(Name, " -> ", Target), ResultCode=tostring(ResultCode))
| summarize Records=count(), FirstSeen=min(TimeGenerated), LastSeen=max(TimeGenerated) by Table, AppRoleName, Detail, ResultCode
| order by Table asc, Records desc, AppRoleName asc, Detail asc, ResultCode asc
| take 100
```

**Interpretation prompt**

> Explain this actual result by signature. Separate Orders/payments failures and database timeouts from exporter 401s and later 404s. These are telemetry record counts, not unique user requests. Distinguish the reported stopped-database diagnosis from what the logs independently establish.

### Saved observations, not new runs

The saved result has **14 grouped rows**. Selected observations:

| Stream/signature | Records | First seen UTC |
| --- | ---: | --- |
| Request `GET /api/orders`, 500 | 120 | 17:40:54.8075374 |
| Request `GET /orders`, 500 | 120 | 17:40:54.807648 |
| Dependency `GET /orders` to the Orders service, 500 | 120 | 17:40:54.8077237 |
| Exception `System.TimeoutException: Failed to connect to YOUR_DATABASE_HOST:5432` | 119 | 17:41:09.8219856 |
| Request `GET /api/payments`, 502 | 115 | 17:41:10.665301 |
| Request `GET /payments`, 500 | 116 | 17:41:10.666113 |
| Dependency `GET /payments` to the payments service, 500 | 116 | 17:41:10.6654765 |
| Later request `GET /orders`, 404 | 56 | 18:11:45.8555904 |
| Exporter dependencies, traces/logs/metrics, 401 | 7,916 / 951 / 674 | Present from about 17:30 |

The Orders and payments dependency targets are their respective Container Apps service hosts in the saved result; those hosts and the database address are omitted from this public copy. These records establish failures on those paths. They do not establish many/all services failed, prove the database was stopped, or identify unique end-user requests. Gateway and backend telemetry may describe the same call at different layers; don't add their counts as separate users.

The later Orders 404s and exporter 401s are separate signatures, not interchangeable evidence for the database-connectivity failure. The baseline also contains root-route 404s and one payments 499. First-seen timestamps locate observed records; they are not guaranteed incident onset.

## Q2: narrow route/status breakdown

**Not yet run.** This isolates Orders and payments request names but retains all statuses, including successes if present. `Records` is the observed row count; `RepresentedItems` sums the documented sampling weights. Inspect `MissingWeightRows` before treating that sum as complete.

```kusto
let startTime = datetime(2026-09-22T17:30:00Z);
let endTime = datetime(2026-09-22T19:45:00Z);
AppRequests
| where TimeGenerated >= startTime and TimeGenerated < endTime
| where Name in ("GET /api/orders", "GET /orders", "GET /api/payments", "GET /payments")
| summarize Records=count(),
            RepresentedItems=sum(ItemCount),
            MissingWeightRows=countif(isnull(ItemCount) or ItemCount < 1),
            FirstSeen=min(TimeGenerated),
            LastSeen=max(TimeGenerated)
    by Name, ResultCode, Success
| order by Name asc, ResultCode asc, Success asc
| take 100
```

**Run/interpretation prompt**

> Prepare Q2 for the same workspace and window; wait for approval before running. Then explain each route/status separately. Do not combine gateway and backend rows into unique request counts or infer success from health-probe traffic.

**Verify:** the exact request names, result codes, whether successes exist, and sampling-weight completeness. Q1's failure-only output does not tell us success volume. An absent route means no matching records in this query, not proof the service was healthy or unused.

## Q3: bounded underlying Orders and database evidence

**Not yet run.** Up to 25 records per stream, at most 75 total. The sample selects failed Orders requests, failed Orders dependencies, and a database timeout signature. Replace the dependency target and database host placeholders with the exact values from your reviewed baseline before execution. Exception rows are initially selected by signature/time, **not** assumed to belong to an Orders operation.

```kusto
let startTime = datetime(2026-09-22T17:30:00Z);
let endTime = datetime(2026-09-22T19:45:00Z);
union
    (AppRequests
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where Name in ("GET /api/orders", "GET /orders")
     | where Success == false or toint(ResultCode) >= 500
     | order by TimeGenerated asc, Id asc
     | take 25
     | project EventTime=TimeGenerated, Stream="Request", OperationId, Id, ParentId,
               OperationName, Detail=Name, Target="", ResultCode, Success,
               DurationMs, ItemCount),
    (AppDependencies
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where Name == "GET /orders"
     | where Target == "YOUR_ORDERS_DEPENDENCY_TARGET"
     | where Success == false or toint(ResultCode) >= 500
     | order by TimeGenerated asc, Id asc
     | take 25
     | project EventTime=TimeGenerated, Stream="Dependency", OperationId, Id, ParentId,
               OperationName, Detail=Name, Target, ResultCode, Success,
               DurationMs, ItemCount),
    (AppExceptions
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where ExceptionType == "System.TimeoutException"
     | where OuterMessage == "Failed to connect to YOUR_DATABASE_HOST:5432"
     | order by TimeGenerated asc, OperationId asc
     | take 25
     | project EventTime=TimeGenerated, Stream="Exception", OperationId, Id="",
               ParentId, OperationName, Detail=strcat(ExceptionType, ": ", OuterMessage),
               Target="", ResultCode="", Success=bool(null),
               DurationMs=real(null), ItemCount)
| order by EventTime asc, Stream asc, Id asc
| take 75
```

**Interpretation prompt**

> Inspect these raw records. What do the failed request, downstream dependency, and timeout actually say? Are operation IDs populated, and which match? A nearby timestamp alone is not a trace link or proof the database was stopped. Identify the limits of this capped sample.

### Q3C: follow populated operation IDs

**Not yet run.** This follows up to ten nonempty operation IDs from failed Orders requests, using the same workspace/window. It retains up to 200 related records. Empty IDs are never joined together. If the seed has no IDs, an empty result is a correlation limit, not "no exceptions."

```kusto
let startTime = datetime(2026-09-22T17:30:00Z);
let endTime = datetime(2026-09-22T19:45:00Z);
let sampleOperations = materialize(
    AppRequests
    | where TimeGenerated >= startTime and TimeGenerated < endTime
    | where Name in ("GET /api/orders", "GET /orders")
    | where Success == false or toint(ResultCode) >= 500
    | where isnotempty(OperationId)
    | summarize FirstRequest=min(TimeGenerated) by OperationId
    | order by FirstRequest asc, OperationId asc
    | take 10
    | project OperationId
);
union
    (AppRequests
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where isnotempty(OperationId) and OperationId in (sampleOperations)
     | project EventTime=TimeGenerated, Stream="Request", OperationId, Id, ParentId,
               OperationName, Detail=Name, Target="", ResultCode, Success,
               DurationMs, ItemCount),
    (AppDependencies
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where isnotempty(OperationId) and OperationId in (sampleOperations)
     | project EventTime=TimeGenerated, Stream="Dependency", OperationId, Id, ParentId,
               OperationName, Detail=Name, Target, ResultCode, Success,
               DurationMs, ItemCount),
    (AppExceptions
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where isnotempty(OperationId) and OperationId in (sampleOperations)
     | project EventTime=TimeGenerated, Stream="Exception", OperationId, Id="",
               ParentId, OperationName, Detail=strcat(ExceptionType, ": ", OuterMessage),
               Target="", ResultCode="", Success=bool(null),
               DurationMs=real(null), ItemCount)
| order by OperationId asc, EventTime asc, Stream asc, Id asc
| take 200
```

**Interpretation prompt**

> Follow the populated operation IDs and parent/span identifiers. Which request, dependency, and exception records belong to the same trace? Distinguish operation membership from a verified parent-child edge. Do not generalize this ten-operation, 200-row sample to every failure.

`AppRequests.TimeGenerated` is request-processing **start**, and `DurationMs` is its duration in milliseconds; exception/dependency timestamps have their own recording semantics. Don't narrate sorted timestamps as a proven causal order. Sampling, missing instrumentation, and the row cap can omit related records.

## Q4: broader 5xx and database-connectivity signatures

**Not yet run.** Remove Q3's Orders request-name and dependency-target restrictions, not an app-name filter that never existed. Inspect all request/dependency 5xx rows and the reviewed database timeout signature across available operations. Replace `YOUR_DATABASE_HOST` before execution. This intentionally excludes exporter 401s and later 404s; Q1 retains those separate signatures.

```kusto
let startTime = datetime(2026-09-22T17:30:00Z);
let endTime = datetime(2026-09-22T19:45:00Z);
union
    (AppRequests
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where toint(ResultCode) between (500 .. 599)
     | project TimeGenerated, Stream="Request 5xx", OperationName, Detail=Name,
               Target="", ResultCode, ItemCount),
    (AppDependencies
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where toint(ResultCode) between (500 .. 599)
     | project TimeGenerated, Stream="Dependency 5xx", OperationName, Detail=Name,
               Target, ResultCode, ItemCount),
    (AppExceptions
     | where TimeGenerated >= startTime and TimeGenerated < endTime
     | where ExceptionType == "System.TimeoutException"
     | where OuterMessage == "Failed to connect to YOUR_DATABASE_HOST:5432"
     | project TimeGenerated, Stream="Database connection timeout", OperationName,
               Detail=strcat(ExceptionType, ": ", OuterMessage), Target="",
               ResultCode="", ItemCount)
| summarize Records=count(),
            RepresentedItems=sum(ItemCount),
            MissingWeightRows=countif(isnull(ItemCount) or ItemCount < 1),
            FirstSeen=min(TimeGenerated),
            LastSeen=max(TimeGenerated)
    by Stream, OperationName, Detail, Target, ResultCode
| order by Stream asc, Records desc, OperationName asc, Detail asc, Target asc
| take 100
```

**Run/interpretation prompt**

> Prepare and review Q4 with the same source/window, then wait for execution approval. Which operations and targets appear beyond Orders? Separate observed payments impact from any newly returned scope. Do not say many or all services failed, and don't infer a shared cause solely from contemporaneous errors.

**Verify:** actual operation names, dependency targets, first/last observations, signatures, and whether the 100-group cap is reached. Blank operation names limit attribution. Match traces separately before asserting that another route shares the same database cause. Gateway health does not establish successful business requests.

## H: test historical deployment and database state

This is a required hypothesis check, **not a runnable query with guessed resources**. Identify the actual database and resource group from your approved incident source; do not copy a demonstration resource name. A verified historical configuration/state table, its retained coverage, and independent endpoint/resource mapping are still needed. Verify the exact resource type/ID before constructing a resource command. Do not invent an `AzureActivity`, resource-changes, or database-metrics query and call it qualified.

**Preparation prompt**

> For September 22, 17:30 to 19:45 UTC, test the competing gateway deployment or configuration explanation and seek independent database state or availability evidence. First list the actual retained historical sources, verified resource identities, schemas, and retention that would support these checks. Preview each read-only command or complete query and its scope; do not execute yet. Do not substitute current configuration or current database health for historical evidence. If a source is unavailable, mark the check open.

| Check | Inputs required before a separately approved read | Interpretation limit |
| --- | --- | --- |
| Gateway deployment/config | Verified gateway resource; retained incident-window revision/deployment records; timestamped image/digest and relevant configuration snapshots/change details | A change near the failures is a competing explanation, not causality. No returned change is not proof of no change unless coverage/retention are established. |
| Database state/availability | Verified mapping from the saved endpoint to the actual database resource; retained state transitions, service/audit events or availability evidence covering the incident | Timeouts alone cannot distinguish stopped state from network/firewall/other reachability problems. A current running/stopped value does not prove September state. |
| Cross-check | Independent timestamps in UTC, covered interval, and connection-path/config evidence | Compare against observed failure onset; gaps stay gaps. Don't infer historical snapshots from today's configuration. |

Azure resource-change history has limited retention. For this older interval, first establish what historical evidence still exists and whether collection actually covered it. A successful query against an empty or expired source is not a ruled-out hypothesis.

If a retained `AzureActivity` source is verified, project only the needed timestamp, resource, operation/status, and approved change details. Omit `Claims`/`Claims_d`, caller/user identity, authorization, and full `Properties` payloads. The first incident's missing stop event is a gap to assess, not permission to borrow the later recurrence's event.

**After actual evidence is returned**

> Compare only the returned historical records with the request/timeout timeline. Which deployment/config and database explanations are supported, contradicted, or still open? The June 17 bad-image incident explains why we tested deployment; it is not September 22 evidence. Do not declare "configuration ruled out" or "database proved stopped" without the relevant independent records.

**Unavailable-stage wording:** "The logs support database connectivity trouble. I still need retained incident-window configuration and database-state evidence to distinguish the explanations."

## Q5: same-route successes, failures, and request timing

**Optional; not yet run.** This stays entirely in **AppRequests**, separated by exact route and five-minute request-start bucket. It does not join dependency/exception duration or compare a business route with a health probe. `Outcome` keeps successful HTTP responses, failures, and unknowns separate.

```kusto
let startTime = datetime(2026-09-22T17:30:00Z);
let endTime = datetime(2026-09-22T19:45:00Z);
AppRequests
| where TimeGenerated >= startTime and TimeGenerated < endTime
| where Name in ("GET /api/orders", "GET /orders", "GET /api/payments", "GET /payments")
| extend Outcome=case(
    Success == false or toint(ResultCode) between (400 .. 599), "Failure",
    Success == true and toint(ResultCode) between (200 .. 399), "Success",
    "Unknown")
| summarize Records=count(),
            RepresentedItems=sum(ItemCount),
            MissingWeightRows=countif(isnull(ItemCount) or ItemCount < 1),
            SampledRows=countif(ItemCount > 1),
            DurationRows=countif(isnotnull(DurationMs) and DurationMs >= 0),
            MeanDurationMs=avgif(DurationMs, isnotnull(DurationMs) and DurationMs >= 0),
            MinDurationMs=minif(DurationMs, isnotnull(DurationMs) and DurationMs >= 0),
            MaxDurationMs=maxif(DurationMs, isnotnull(DurationMs) and DurationMs >= 0)
    by RequestStart=bin(TimeGenerated, 5m), Name, Outcome, ResultCode
| order by RequestStart asc, Name asc, Outcome asc, ResultCode asc
| take 500
```

**Run/interpretation prompt**

> Prepare Q5 and wait for approval. In the actual result, compare successes and failures for the same exact AppRequests route and time bucket. Explain milliseconds, row counts, ItemCount sampling weights, and missing duration/weight values. Do not claim recovery from absent rows, use healthy probes as successful business requests, or infer reliable p95 from sparse sampled data.

`DurationMs` is real request duration in **milliseconds**, not the query's execution time. `MeanDurationMs`, minimum, and maximum here describe observed duration rows, **not a sampling-weighted population**. `ItemCount` expresses represented telemetry items; even weighted route totals must not combine gateway and backend layers into unique user requests. Empty buckets/successes may reflect missing collection, sampling, or no matching traffic.

P95 is deliberately omitted from this first timing take. Before adding it, establish adequate duration sample size, route/layer consistency, sampling policy, and how the percentile should be weighted. A handful of sampled records is not a defensible tail-latency comparison.

### Private timing/capacity pickup

No complete private-source query or result is supplied for this pickup. The visible private Kusto connection was marked **Target not tested**; a connected provider is not successful target access or an executed query. Don't replace the working Log Analytics route with it.

> Preview a comparison using my approved private performance source. First show the actual source/schema, complete bounded query, time window, timing or capacity fields and units, and how they align with these request routes. Wait for separate source/metadata/query approvals. If those inputs aren't available, keep this as a proposed check, not a result or capacity diagnosis.

## End with a reviewed update, not an invented resolution

> Draft a September 22 INC0922001 update with: the SRE-reported diagnosis; independently observed Orders/payments failures and connectivity timeouts; any actual correlation or wider-impact findings; deployment/config and historical database checks still open; and the approved investigation link. Distinguish fresh query execution over historical data from current production health. Show the exact body and destination; do not send.

Only an approved frozen preview and confirmed delivery support a claim that a Teams update was sent. Use plain text/paragraphs/bullets; don't claim HTML styling or an @Everyone notification. Keep the incident, time window, evidence, and actual delivery receipt aligned. A saved query result is not a posting receipt or proof of completed recovery.

A terminal continuation must connect **INC0922001 in that CLI conversation** and return an actual read/summary. The App connection doesn't transfer. Canvas actions in CLI are not proof of standalone renderer-free Azure MCP support.

## References and saved-evidence provenance

- The baseline query and 14 grouped observations were recorded for the demonstration. Public copies omit private resource/thread/workspace identifiers and replace dependency hosts and the database address with explicit placeholders. The sample counts are saved observations, not results returned by running this repository.
- First-incident thread identity, state/probe observations, Activity Log limitation, and unconfirmed recovery were supplied from the saved event history. They remain attributed SRE-recorded evidence, separate from the local query output.
- [AppRequests schema](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/apprequests): `DurationMs`, `ItemCount`, `OperationId`, request-start `TimeGenerated`, and the other request columns used above.
- [AppDependencies schema](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/appdependencies): target, operation/parent/span identifiers, duration, result, and sampling fields.
- [AppExceptions schema](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/appexceptions): exception signature and operation/parent identifiers.
- [Canvas guide](https://aka.ms/sreagent/canvas/doc), [demo](https://aka.ms/sreagent/canvas/demo), and [companion blog](https://aka.ms/sreagent/canvas/blog).
