# Topic 13 - Error Handling, Retries & Recovery Strategies

## Overview

This n8n assessment demonstrates a primary payment workflow that intentionally fails, and a separate error recovery workflow that captures diagnostic information, classifies failures, creates a simulated dead-letter queue (DLQ) record, routes recovery actions, and writes audit outputs.

## Workflows

### A. Primary payment workflow

`Schedule Trigger (temporary) / Manual Trigger -> Code_SimulateFailingOrder -> HTTP_FlakyPaymentGateway`

- `Code_SimulateFailingOrder`: creates a sample transaction `TX-CRASH-801`, email, amount `1450`, and `simulateFault: GATEWAY_TIMEOUT_504`.
- `HTTP_FlakyPaymentGateway`: POSTs JSON to `https://httpstat.us/504`, with Retry On Fail, max 3 tries, 2000 ms between tries, and Stop Workflow on failure.
- The observed failure was **ECONNRESET / connection closed unexpectedly**, not a confirmed HTTP 504 response.
- Workflow Settings -> **Error Workflow** selects `Topic 13 - Error Recovery Handler deb`.

### B. Error recovery handler

`Error_Trigger -> Code_ExtractDiagnostics -> Code_PersistToDLQ -> Switch_ClassifyRecovery`

- `TRANSIENT` -> `Code_ScheduleReplay` -> `Code_RecoveryAudit`
- `PERMANENT` -> `Code_EscalateIncident` -> `Code_RecoveryAudit`

**Node responsibilities**

| Node | Purpose |
|---|---|
| `Error_Trigger` | Receives failed production execution details from the primary workflow. |
| `Code_ExtractDiagnostics` | Extracts workflow ID/name, execution ID, failed node, error message/type, and classifies recovery type. |
| `Code_PersistToDLQ` | Produces a structured DLQ record with ID, retry count, and PENDING status. |
| `Switch_ClassifyRecovery` | Routes TRANSIENT and PERMANENT failures to separate branches. |
| `Code_ScheduleReplay` | Sets RETRY_SCHEDULED, PENDING_RETRY, retryCount +1, and nextRetryAt +5 minutes. |
| `Code_EscalateIncident` | Sets ESCALATED_TO_ENGINEERING, ESCALATED, HIGH priority, and Engineering Support assignment. |
| `Code_RecoveryAudit` | Emits RECOVERY_AUDIT with the selected recovery action and status. |

## Test results

| Test | Evidence | Result |
|---|---|---|
| Primary gateway failure | Main workflow screenshot, failed HTTP node | Passed: failure generated |
| Real Error Trigger | Diagnostics shows workflow name, execution 516452, failed HTTP node | Passed |
| Transient classification | `errorType: ECONNRESET`, `recoveryType: TRANSIENT` | Passed |
| Transient scheduling record | `RETRY_SCHEDULED`, `PENDING_RETRY`, `retryCount: 1` | Passed |
| Transient audit | `RECOVERY_AUDIT`, execution 516452 | Passed |
| Permanent classification and route | Pinned test data, execution `TEST-PERM-001` | Passed (simulated) |
| Escalation record | `ESCALATED_TO_ENGINEERING`, `ESCALATED`, `Engineering Support` | Passed (simulated) |
| Permanent audit | `RECOVERY_AUDIT`, `TEST-PERM-001` | Passed (simulated) |

## How to demonstrate

1. Open the primary workflow and explain the sample order and HTTP retry settings.
2. Show the production gateway failure and the selected Error Workflow in Settings.
3. Open the error handler and show real diagnostic output for `ECONNRESET`.
4. Follow the transient branch and show the scheduled retry record and final audit.
5. Show the pinned permanent-error test and engineering escalation audit.
6. Remove pinned test data; disable the temporary one-minute Schedule Trigger and publish the main workflow change.

## Implementation boundaries

- `Code_PersistToDLQ` generates a DLQ **record in workflow output**; it does not store it durably in a database or external queue.
- `Code_ScheduleReplay` calculates `nextRetryAt` but **does not perform an automatic replay** after five minutes.
- `Code_EscalateIncident` creates an engineering escalation **record**, not an actual email, Slack message, or ticket.
- The permanent-error branch was tested with **simulated/pinned output**, while the transient branch used an observed production workflow failure.
- Public test endpoint behavior can vary; this test observed `ECONNRESET`, not a confirmed HTTP 504 response.

## Submitted evidence

See `Topic13_Evidence_Report.pdf` for 10 annotated screenshots covering the workflows, diagnostics, transient replay scheduling, permanent escalation, and final audit results.
