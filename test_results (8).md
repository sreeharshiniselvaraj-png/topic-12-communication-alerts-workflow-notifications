# Topic 12 – Communication, Alerts & Workflow Notifications
## Test Results

**Test suite source:** Trainer-provided *Topic 12: Communication, Alerts & Workflow Notifications – Test Cases*  
**Test suite version:** 1.0  
**Source status:** Draft  
**Source last updated:** 2026-06-25  
**Execution environment:** n8n coursework workflows; mock integrations used where stated  
**Result basis:** Execution outputs and test outcomes shared during the project walkthrough

> PASS means the described behavior was observed in the available test evidence. It does not imply production integration, durable storage, or full compliance with a different acceptance criterion that was not demonstrated.

## 1. Test summary

| Test ID | Test case | Result | Evidence qualification |
|---|---|---|---|
| TC-001 | Slack Interactive Approval | PASS – mock flow | Approve and Reject were reported working in the connected mock button-to-Wait workflow. Live Slack message update and production callback correlation were not verified. |
| TC-002 | SLA Escalation Timer | PASS – mock timeout | The 10-minute timeout had already been tested and the expected mock escalation output was received. |
| TC-003 | Alert Rate Limiting (Throttling) | PASS – demonstrated sample | Duplicate incident was marked throttled and suppression audit events were generated. The exact three-trigger Alert A sequence should only be claimed if its separate execution evidence is retained. |
| TC-004 | Multi-channel Urgency Routing | PARTIAL PASS – mock routing | P1 Slack and SMS mock outputs and INFO Email mock output were observed. The exact negative assertion that INFO skips Slack and SMS, and that all live staging credentials are configured, was not independently established. |
| TC-005 | Webhook Security Signature Check | PARTIAL PASS – shared-token validation | Invalid/missing `x-approval-token` returned HTTP 401. Cryptographic Slack signature verification was not implemented or demonstrated. |

## 2. Detailed test results

### TC-001: Slack Interactive Approval

**Linked requirements:** FR-001, FR-002  
**Trainer expected result:** Webhook receives the button payload; workflow resumes through the approved branch; Slack message updates to show approval.

**Observed evidence**
- Approve and Reject were reported as working in the connected mock button-to-Wait workflow.
- The workflow reached the approval wait step and the decision paths were exercised.

**Result:** **PASS – mock interaction demonstrated; full trainer acceptance not completely verified.**

**Qualification**
- Live Slack interactive callback delivery was not verified.
- Updating the original Slack message to display the approver and final decision was not independently evidenced.
- Do not describe this as a fully production-integrated Slack approval.

### TC-002: SLA Escalation Timer

**Linked requirement:** FR-003  
**Trainer expected result:** Workflow waits for the configured period; if no acknowledgement arrives, the escalation branch sends an SMS/call to secondary contacts.

**Observed evidence**
- The 10-minute timeout test was previously executed.
- The expected mock escalation output was received.
- The user confirmed that the 10-minute test had already completed successfully.

**Result:** **PASS – 10-minute mock timeout path verified.**

**Qualification**
- The notification output was mock-mode evidence; it does not prove an actual SMS or phone call reached a secondary contact.

### TC-003: Alert Rate Limiting (Throttling)

**Linked requirement:** FR-004  
**Trainer expected result:** First identical alert is delivered; two repeats within two minutes are blocked; no duplicate Slack messages are posted within the 10-minute window.

**Observed evidence**
- `INC-902` (`payment-gateway:AUTH_TIMEOUT_504`) was evaluated with `isThrottled: true` and `shouldAlert: false`.
- Suppression audit events were generated, including `ALERT_SUPPRESSED_BY_THROTTLE_POLICY`.
- `IF_IsThrottled` routing was corrected and verified: True → `Log_SuppressedAlert`; False → `Switch_RouteBySeverity`.
- The allowed incidents continued to routing while the duplicate was suppressed.

**Result:** **PASS – demonstrated duplicate-suppression behavior.**

**Qualification**
- The shared evidence confirms a duplicate was suppressed. Retain the original execution history for the exact trainer sequence of one initial Alert A plus two repeated Alert A triggers within two minutes.
- The throttle state uses workflow static data in this implementation; this is not equivalent to a durable external database.

### TC-004: Multi-channel Urgency Routing

**Linked requirement:** FR-005  
**Trainer expected result:**
- `CRITICAL` → Slack operational channel and Twilio SMS.
- `INFO` → daily summary Email only, skipping Slack and Twilio.

**Observed evidence**
- P1 Critical Slack mock receiver accepted the payload: `ok: true`, channel `#critical-incidents`.
- P1 Critical SMS mock output for `INC-901` reported `deliveryStatus: DELIVERED`, `deliveryMode: MOCK`.
- INFO Email mock output for `INC-905` reported `deliveryStatus: DELIVERED`, `deliveryMode: MOCK`.
- Corresponding Slack, SMS, and Email audit records were generated.
- P3 Minor `INC-903` was queued in `digest_queue` as `HOURLY_MINOR_INCIDENT_DIGEST`.

**Result:** **PARTIAL PASS – mock channel outputs observed.**

**Qualification**
- The observed P1 Slack and SMS outputs support the CRITICAL routing requirement.
- The INFO Email output supports the Email route, but the evidence shown does not by itself prove that Slack and SMS were skipped for INFO. Verify the INFO execution branch outputs and absence of Slack/SMS dispatch before marking the full TC-004 acceptance criteria PASS.
- The additional P3 digest behavior is useful, but it is not a substitute for the exact INFO-only Email expectation.
- Mock delivery is not live provider delivery.

### TC-005: Webhook Security Signature Check

**Linked requirement:** NFR-001  
**Trainer expected result:** Invalid signature is rejected with HTTP 401 or equivalent, and the workflow execution does not resume.

**Observed evidence**
- Invalid or missing `x-approval-token` was rejected.
- The observed response returned HTTP `401` with an unauthorized message.
- The callback implementation compares a supplied shared token with an expected shared token.

**Result:** **PARTIAL PASS – invalid shared-token rejection verified; signature verification not implemented.**

**Qualification**
- A shared-secret header comparison is not the same as cryptographic Slack request-signature verification (for example, HMAC verification over the raw request body with timestamp validation).
- To claim full NFR-001 compliance, implement and test the trainer-required signature verification and confirm that invalid signatures do not resume the waiting approval execution.

## 3. Additional notification and audit checks

These checks supplement the five trainer test cases; they should not be presented as additional trainer TC IDs unless the trainer assigns them.

| Additional check | Incident | Observed result | Status |
|---|---|---|---|
| Critical SMS mock dispatch | INC-901 | TWILIO provider label, MOCK mode, DELIVERED status | PASS – mock |
| Critical SMS audit record | INC-901 | `NOTIFICATION_DISPATCHED`, `notificationType: SMS` | PASS – generated record |
| Critical Slack mock receiver | INC-901 | `ok: true`, `#critical-incidents` | PASS – mock |
| Critical Slack audit record | INC-901 | `NOTIFICATION_DISPATCHED`, `deliveryStatus: DELIVERED` | PASS – generated record |
| INFO Email mock dispatch | INC-905 | SMTP provider label, MOCK mode, DELIVERED status | PASS – mock |
| INFO Email audit record | INC-905 | `NOTIFICATION_DISPATCHED`, `notificationType: EMAIL` | PASS – generated record |
| P3 digest queue | INC-903 | `digestStatus: QUEUED`, `HOURLY_MINOR_INCIDENT_DIGEST` | PASS – queue output |
| Duplicate suppression audit | INC-901 / INC-903 / INC-905 sample events | `ALERT_SUPPRESSED_BY_THROTTLE_POLICY` events were generated | PASS – generated records |

## 4. Overall assessment

**The core workflow behaviors have been demonstrated in mock/test mode.** TC-002 and the demonstrated duplicate-suppression behavior have clear positive evidence. TC-001 is demonstrated through the mock interaction flow. TC-004 has positive mock outputs for the expected channels but still needs confirmation of the INFO exclusion paths. TC-005 verifies shared-token rejection but does not satisfy cryptographic signature verification.

### Remaining work before claiming full trainer acceptance

1. **TC-001:** Capture evidence that the Slack message itself is updated after approval, if the trainer requires that exact step.
2. **TC-003:** Preserve execution evidence for the complete first-send plus two-repeat sequence inside two minutes.
3. **TC-004:** Run the exact INFO test and verify no Slack or Twilio node dispatches; verify the CRITICAL run dispatches to both Slack and SMS.
4. **TC-005:** Implement actual request-signature verification, then retest invalid signature rejection and ensure the waiting workflow does not resume.
5. **Production readiness:** Replace mock providers with real credentials only when authorized; use durable storage for audit records and throttle state if production persistence is required.

## 5. Evidence handling

For trainer submission, attach or retain n8n execution screenshots/IDs showing:
- Approval message, button decision, and resumed execution.
- 10-minute Wait timeout and escalation branch output.
- First alert delivery and repeat-alert suppression.
- CRITICAL Slack and SMS routing; INFO Email-only routing.
- Invalid callback token/signature response and the non-resumed execution.

Do not include actual API keys, webhook secrets, approval tokens, private phone numbers, or credentials in the submitted Markdown files or screenshots.
