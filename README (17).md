# Topic 12 -- Communication, Alerts & Workflow Notifications

An n8n coursework project demonstrating multi-channel incident
notifications, severity routing, duplicate-alert throttling, interactive
approval handling, timeout escalation, webhook token validation, and
workflow-generated audit records.

## Project status

**Assessment implementation:** Executed and published workflows with
mock notification integrations.

> Important: Slack, Twilio SMS, and SMTP Email outputs in this
> assessment are mock/test outputs. They do not prove delivery to real
> Slack channels, phone numbers, or email inboxes. Audit records are
> generated in workflow execution output; persistent database or
> external audit-store storage has not been demonstrated.

## Workflows

### 1. Main notification workflow

**Workflow:**
`Topic 12 - Communication, Alerts & Workflow Notifications`

Main processing sequence:

`Manual_Trigger` → `Code_SimulateIncidents` →
`Code_EvaluateSeverityAndThrottle` → `IF_IsThrottled` → suppression log
or severity switch → channel formatter/queue → mock notification → audit
output.

Key nodes: - `Code_SimulateIncidents` -- generates sample incident
data. - `Code_EvaluateSeverityAndThrottle` -- assigns severity and
checks the service/error-signature throttle key. - `IF_IsThrottled` --
separates duplicates from alerts allowed to continue. -
`Log_SuppressedAlert` -- records suppressed duplicate alerts. -
`Switch_RouteBySeverity` -- routes allowed incidents by severity. -
`Code_BuildP1BlockKit` -- builds a P1 Critical Slack Block Kit
payload. - `Code_BuildP2SlackCard` -- builds a P2 Major Slack alert
payload. - `Code_StageInDigestQueue` -- stages P3 Minor incidents for a
digest. - `HTTP_PostToMockSlack` -- sends the Slack payload to the
configured mock receiver. - `Code_MockTwilioSMS` -- generates mock SMS
output. - `Code_MockEmailNotification` -- generates mock Email output. -
`Code_AuditNotificationDelivery`, `Code_AuditSMSNotification`, and
`Code_AuditEmailNotification` -- generate structured audit records in
workflow output.

**Critical IF wiring:** `IF_IsThrottled` True → `Log_SuppressedAlert`;
False → `Switch_RouteBySeverity`.

Current severity rules documented for the implementation: - Explicit
`INFO` priority → `INFO`. - Explicit `WARNING` priority → `P2_MAJOR`. -
Production incident with more than 100 impacted users → `P1_CRITICAL`. -
Other production incidents or incidents with impacted users →
`P2_MAJOR`. - Otherwise → `P3_MINOR`.

The throttle key combines service and error signature. The suppression
window is 10 minutes. The cache is n8n workflow static data, not an
external database.

### 2. Interactive approval and timeout workflow

**Workflow:** `Topic 12 - Slack Interactive Approval Gate`

The workflow creates a Slack-style approval message with Approve and
Reject actions, pauses at `Wait_ForApprovalDecision`, and continues
through the selected decision path or the timeout path. The assessment
used a connected mock button-to-Wait flow.

The Wait node is configured for 10 minutes. If no decision is received,
the workflow executes the mock escalation path, including: -
`Code_CreateTimeoutEscalation` - `Code_BuildEscalationNotification` -
`Code_MockEscalationDelivery`

The 10-minute timeout was previously tested and the expected mock
escalation output was reported.

### 3. Approval callback and token-validation workflow

**Workflow / entry point:** `Webhook_ApprovalCallback`

The callback receives a POST request, checks the `x-approval-token`
header, validates the decision, returns HTTP 401 for a missing/invalid
token and HTTP 200 for a valid token, and generates an approval audit
record.

The current implementation uses a shared-secret token comparison. It is
not cryptographic HMAC/signature verification. Never place the actual
token value in this README, screenshots, or a public repository.

## Sample incident data

  -----------------------------------------------------------------------
  Incident                Service                 Purpose
  ----------------------- ----------------------- -----------------------
  `INC-901`               `payment-gateway`       P1 Critical incident

  `INC-902`               `payment-gateway`       Duplicate error
                                                  signature; should be
                                                  throttled after INC-901

  `INC-903`               `inventory-service`     P3 Minor incident;
                                                  digest queue

  `INC-905`               `reporting-service`     INFO incident; mock
                                                  Email route
  -----------------------------------------------------------------------

These are test records, not live production incidents.

## Requirements and test documentation

-   [`requirement_document.md`](requirement_document.md) -- trainer's
    BRD, PRD, SRS, user stories, acceptance criteria, test cases, and
    implementation alignment.
-   [`instructions.md`](instructions.md) -- workflow architecture, node
    explanations, execution guidance, and trainer demonstration notes.
-   [`test_results.md`](test_results.md) -- observed test outputs and
    PASS/limitations summary. Update it only when new execution evidence
    is available.

## Setup and execution

### Prerequisites

-   Access to an n8n workspace (trainer requirement: n8n 1.0.0+).
-   The Topic 12 workflows imported or created in the workspace.
-   Any required mock receiver endpoints configured for the assessment.
-   Test credentials/secrets configured inside n8n, not hardcoded in
    documentation or shared workflow exports.

### Run the main notification workflow

1.  Open `Topic 12 - Communication, Alerts & Workflow Notifications`.
2.  Start the workflow using `Manual_Trigger`.
3.  Inspect the output of `Code_SimulateIncidents`.
4.  Inspect `Code_EvaluateSeverityAndThrottle`.
5.  Confirm `IF_IsThrottled` routes True to `Log_SuppressedAlert` and
    False to `Switch_RouteBySeverity`.
6.  Inspect each severity output and its downstream mock notification
    node.
7.  Inspect the corresponding audit output nodes.
8.  Record screenshots or execution IDs for the submission.

### Run the approval workflow

1.  Open `Topic 12 - Slack Interactive Approval Gate`.
2.  Start a new test execution.
3.  Confirm the workflow reaches `Wait_ForApprovalDecision`.
4.  Test the Approve and Reject mock decision paths separately.
5.  For timeout validation, start a fresh run and do not submit a
    decision; allow the configured 10-minute timeout to expire.
6.  Inspect `Code_CreateTimeoutEscalation`,
    `Code_BuildEscalationNotification`, and
    `Code_MockEscalationDelivery`.
7.  Preserve the execution evidence.

### Run the callback security test

1.  Open the callback webhook workflow and copy its test/production
    webhook URL from n8n.
2.  Send a POST request with the expected decision JSON and the
    configured `x-approval-token` header for the positive test.
3.  Repeat with an invalid or missing token.
4.  Confirm the invalid/missing-token request returns HTTP 401 and the
    valid-token request follows the accepted decision path.
5.  Do not disclose the actual shared secret in screenshots or source
    control.

## Demonstrated test outcomes

Based on the execution outputs shared during implementation: - Duplicate
suppression: `INC-902` was marked throttled and logged as suppressed. -
Allowed incidents: `INC-901`, `INC-903`, and `INC-905` continued to
severity routing after the IF wiring was corrected. - P1 Slack mock:
mock receiver returned `ok: true` for `#critical-incidents`. - P1 SMS
mock: `INC-901` produced `SMS_NOTIFICATION_DISPATCHED` with
`deliveryStatus: DELIVERED` in MOCK mode. - INFO Email mock: `INC-905`
produced `EMAIL_NOTIFICATION_DISPATCHED` with
`deliveryStatus: DELIVERED` in MOCK mode. - P3 digest queue: `INC-903`
produced a `QUEUED` digest record for `HOURLY_MINOR_INCIDENT_DIGEST`. -
Corresponding Slack, SMS, and Email audit records were generated in
workflow outputs. - The 10-minute approval timeout test was previously
run and the expected mock escalation output was reported. - The
invalid-token callback test returned HTTP 401 in the observed test. -
Approve and Reject were reported as working in the connected mock
button-to-Wait workflow.

See `test_results.md` for the test-by-test record. Keep each result tied
to its actual n8n execution evidence.

## Known limitations and production hardening

1.  **Mock delivery:** Configure real Slack app credentials, Twilio
    credentials, and SMTP/provider credentials before claiming live
    delivery.
2.  **Webhook security:** Replace shared-secret equality checking with
    the trainer-required cryptographic signature/HMAC verification,
    including timestamp/replay protections where appropriate.
3.  **Audit persistence:** Send audit events to a durable database or
    approved audit store and verify writes/retention.
4.  **Throttle persistence:** Workflow static data is not a distributed,
    durable rate-limit store. Use a persistent store with expiry,
    concurrency-safe updates, and retention controls for production.
5.  **Performance evidence:** Keep measured response-time output if
    claiming the NFR-002 target of less than 3,000 ms.
6.  **Approval persistence and Slack update:** Verify live callback
    correlation, database status update, and Slack message update before
    claiming the full production acceptance criteria.
7.  **Timeout fallback:** The trainer's risk section suggests an
    additional fallback such as 24-hour auto-rejection for approvals
    that remain unanswered. Treat this as a production-hardening item
    unless separately implemented and tested.

## Trainer demonstration checklist

-   Explain the incident → severity → throttle → route flow.
-   Show the duplicate incident being suppressed.
-   Show the P1 Slack/SMS mock outputs.
-   Show the INFO Email mock output and P3 digest queue output.
-   Show the corresponding audit records.
-   Demonstrate Approve and Reject in the mock approval workflow.
-   Show the previously tested 10-minute timeout escalation output.
-   Show HTTP 401 for invalid/missing callback token and the valid-token
    path.
-   State clearly which integrations are mocked and which
    security/storage enhancements remain outstanding.

## License and data note

This is a training/assessment project. Sample incident IDs, services,
recipients, and messages are demonstration data. Add a license only if
the project owner or trainer specifies one.
