# Topic 12 – Communication, Alerts & Workflow Notifications

## 1. Project overview

This n8n project demonstrates incident classification, duplicate-alert suppression, severity-based notification routing, interactive approval handling, timeout escalation, webhook token validation, and workflow-generated audit records.

The project follows the Topic 12 requirement file and implementation plan. The notification integrations in the current assessment build use mock delivery outputs; they do not send real Slack messages, SMS messages, or SMTP email.

## 2. Workflows

### A. Main notification workflow

**Workflow name:** `Topic 12 - Communication, Alerts & Workflow Notifications`

Main path:

1. `Manual_Trigger` – starts a test run manually.
2. `Code_SimulateIncidents` – emits sample incident records.
3. `Code_EvaluateSeverityAndThrottle` – classifies each incident and checks the in-memory alert cache.
4. `IF_IsThrottled` – separates suppressed incidents from alerts allowed to continue.
5. `Log_SuppressedAlert` – creates a suppression record for duplicates.
6. `Switch_RouteBySeverity` – routes allowed incidents by severity.
7. Severity-specific formatter / queue nodes:
   - `Code_BuildP1BlockKit` – builds a P1 Critical Slack Block Kit payload.
   - `Code_BuildP2SlackCard` – builds a P2 Major Slack alert.
   - `Code_StageInDigestQueue` – queues P3 Minor incidents for the digest.
   - INFO route – sends the informational incident to the mock Email node.
8. `HTTP_PostToMockSlack` – posts Slack payloads to the configured mock receiver.
9. `Code_MockTwilioSMS` and `Code_MockEmailNotification` – generate mock SMS and Email notification outputs.
10. `Code_AuditNotificationDelivery`, `Code_AuditSMSNotification`, and `Code_AuditEmailNotification` – generate workflow-output audit records for their respective paths.

**Important routing condition:** `IF_IsThrottled` must route `true` to `Log_SuppressedAlert` and `false` to `Switch_RouteBySeverity`. The evaluator's `isThrottled` value is true for duplicates; only non-throttled alerts should continue to severity routing.

**Severity rules currently used:**

| Input / condition | Severity |
|---|---|
| `priority` is `INFO` | `INFO` |
| `priority` is `WARNING` | `P2_MAJOR` |
| Production incident with more than 100 impacted users | `P1_CRITICAL` |
| Other production incidents or incidents with impacted users | `P2_MAJOR` |
| Otherwise | `P3_MINOR` |

The throttle key is the combination of service and error signature. The configured suppression window is 10 minutes. The cache uses n8n workflow static data; it is an in-memory/stateful workflow cache, not an external database.

### B. Interactive approval and timeout workflow

**Workflow name:** `Topic 12 - Slack Interactive Approval Gate`

The workflow builds a Slack-style Block Kit approval message with Approve and Reject buttons, pauses at `Wait_ForApprovalDecision`, and resumes with the decision or follows the timeout branch. The current assessment implementation uses a mock button-to-Wait flow.

The Wait node is configured for a 10-minute limit. If no decision arrives before timeout, the workflow finalizes the timeout state and runs the mock escalation path, including `Code_CreateTimeoutEscalation`, `Code_BuildEscalationNotification`, and `Code_MockEscalationDelivery`.

### C. Approval callback and token-validation workflow

**Workflow name:** `Webhook_ApprovalCallback`

This workflow receives a POST callback, checks the `x-approval-token` header, validates the decision, returns HTTP 401 for missing/invalid tokens and HTTP 200 for a valid decision, and generates an approval audit record.

The current implementation performs a shared-secret token comparison. It is not a cryptographic HMAC/signature verification implementation. Do not publish the secret in project documentation or screenshots.

## 3. Running and testing

1. Open the relevant workflow in n8n.
2. Use the Manual Trigger for the main notification or approval workflow tests.
3. For the callback test, send a POST request to the published callback webhook with the expected JSON decision body and the configured token header. Use a deliberately invalid or missing token for the negative test.
4. Inspect each node's execution output, not only the canvas connection.
5. For the main notification workflow, confirm that the IF node sends throttled incidents to the suppression logger and allowed incidents to the severity Switch.
6. Inspect the mock notification node and its corresponding audit node.
7. Keep mock delivery outputs clearly identified as `MOCK`.

The main workflow also has a published webhook trigger for the throttle test (`tc003-throttle`).

## 4. Current sample incident set

The restored simulator emits four records:

| Incident | Service | Test purpose |
|---|---|---|
| `INC-901` | `payment-gateway` | P1 Critical incident |
| `INC-902` | `payment-gateway` | Duplicate signature; should be throttled after INC-901 |
| `INC-903` | `inventory-service` | P3 Minor incident; digest queue |
| `INC-905` | `reporting-service` | INFO incident; mock Email route |

The sample set is assessment data and should not be interpreted as live production incidents.

## 5. Audit and integration limitations

- Audit nodes generate structured records in n8n execution output. Persistent database or external audit-store persistence has not been demonstrated.
- Slack, Twilio, and SMTP outputs are mocks. A mock response such as `ok: true` or `deliveryStatus: DELIVERED` confirms acceptance by the mock/test node, not delivery to a real recipient.
- The callback token check is a shared-secret comparison. The trainer requirement describes token signature verification; cryptographic signature/HMAC verification remains a security enhancement.
- The in-memory throttle cache is not a durable distributed cache. For production use, use an appropriately secured persistent store with expiry/retention controls.
- The 10-minute timeout was previously tested and produced the expected mock escalation output. Retain the actual n8n execution evidence with the submission.

## 6. Trainer requirement mapping

| Requirement | Implementation area |
|---|---|
| FR-001 – Interactive Slack Block Kit | Approval message builder and P1 Block Kit formatter |
| FR-002 – Pause and resume on decision webhook | Wait node / approval callback flow |
| FR-003 – Escalate after 10 minutes | Wait timeout and mock escalation branch |
| FR-004 – Suppress similar alerts within 10 minutes | Severity/throttle Code node and IF branch |
| FR-005 – Route by urgency | P1 Slack + SMS mock; INFO Email mock; P2 Slack path |
| NFR-001 – Approval token validation | Callback webhook shared-secret header check; cryptographic signature verification is not implemented |
| NFR-002 – Respond within 3000 ms | Callback webhook response test; preserve measured execution evidence |
| BRD-004 – Audit log | Workflow-generated notification and approval audit records; persistent storage not verified |

## 7. Before presenting

- Show the main workflow canvas and explain the incident → classification → throttle → route flow.
- Demonstrate the duplicate incident being suppressed.
- Show P1 Slack/SMS mock outputs, INFO Email mock output, and P3 digest queue output.
- Show the approval Approve/Reject flow and timeout escalation.
- Show the invalid-token HTTP 401 and valid-token HTTP 200 callback tests.
- Explain honestly that notification delivery is mocked and audit records are generated in workflow output rather than persisted to a database.
