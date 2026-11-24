# Topic 12: Communication, Alerts & Workflow Notifications

## Requirement Document

**Document version:** 1.0\
**Source:** Trainer-provided Topic 12 requirement file\
**Original document status:** Draft\
**Original last updated:** 25 June 2026\
**Platform:** n8n (version 1.0.0+)\
**Prepared as:** Trainer-requirement transcription and
implementation-alignment document\
**Project status:** Implemented coursework workflow with documented
mock-integration and security limitations

> This document preserves the trainer's requirement intent and adds an
> implementation-alignment section. It does not convert unimplemented
> production requirements into claims of completion.

## 1. Purpose

Define the requirements for an n8n automation solution that coordinates
business communications, interactive approvals, incident escalation,
alert throttling, and notification auditing.

## 2. Learning objectives and concepts

### 2.1 Multi-channel business communication

-   **Slack / MS Teams / Discord:** Real-time team notifications,
    engineering logs, and interactive approval prompts.
-   **Email (SMTP / SendGrid / SES):** Transactional documents,
    structured updates, client-facing notifications, and formatted
    records.
-   **SMS (Twilio) / push notifications:** High-priority alerts,
    critical service disruptions, security warnings, and urgent
    escalation.

### 2.2 Human-in-the-loop (HITL) and interactive approvals

-   Present Approve and Reject buttons in an interactive chat message,
    such as Slack Block Kit.
-   Send the decision payload to an n8n webhook.
-   Pause workflow execution while awaiting the callback and resume when
    a decision is received.
-   Avoid active polling where a webhook wait/resume pattern is
    available.

### 2.3 Escalation paths and service-level agreements

-   Track incident acknowledgement state and the time at which an alert
    is raised.
-   Escalate when the primary responder does not acknowledge within the
    configured SLA.
-   The trainer's detailed acceptance requirement specifies a 10-minute
    wait, followed by SMS escalation if the incident remains pending.
-   The learning section also describes tiered routing examples: initial
    Slack notification, later SMS/call escalation, and
    management-channel escalation.

### 2.4 Notification fatigue and throttling

-   Detect repeated or identical alerts.
-   Suppress duplicates inside a configured time window.
-   Limit alert frequency, with one alert per 10 minutes per system
    component as the example.
-   Aggregate low-priority events into a scheduled summary, such as an
    hourly digest.

## 3. Hands-on technical examples

### 3.1 Slack Block Kit approval message

The trainer's example message contains: - A section describing purchase
order `PO-2026-9087` and the \$12,500.00 review amount. - An actions
block named `approval_actions`. - An Approve button with
`action_id: btn_approve` and value `approved`. - A Reject button with
`action_id: btn_reject` and value `rejected`.

Illustrative structure:

``` json
{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Approval Request*: Purchase order *PO-2026-9087* requires validation ($12,500.00)."
      }
    },
    {
      "type": "actions",
      "block_id": "approval_actions",
      "elements": [
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "Approve"},
          "style": "primary",
          "value": "approved",
          "action_id": "btn_approve"
        },
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "Reject"},
          "style": "danger",
          "value": "rejected",
          "action_id": "btn_reject"
        }
      ]
    }
  ]
}
```

### 3.2 Alert throttling example

The trainer's example compares the current time with a stored
`last_sent` timestamp. If the difference is less than `10 * 60 * 1000`
milliseconds, the alert is suppressed. The example returns the send
decision, last-sent timestamp, and elapsed milliseconds.

## 4. Business Requirements Document (BRD)

### 4.1 Business objective

Establish a centralized and reliable automated communication system that
coordinates operational incident response, manages employee approvals in
real time, dynamically routes escalated alerts, and reduces notification
noise.

### 4.2 Business requirements

  -----------------------------------------------------------------------
  ID                Business          Business          Priority
                    requirement       rationale         
  ----------------- ----------------- ----------------- -----------------
  BRD-001           Approval          Improves approval High
                    decisions must be speed and         
                    completed         adoption by       
                    directly inside   reducing          
                    business chat     friction.         
                    applications                        
                    without requiring                   
                    portal logins.                      

  BRD-002           Automated alerts  Supports incident High
                    for critical      response within   
                    production errors defined SLA       
                    must escalate to  limits.           
                    SMS/Manager if                      
                    ignored by the                      
                    initial                             
                    responder.                          

  BRD-003           Low-priority      Reduces           Medium
                    warning alerts    notification      
                    must be           fatigue.          
                    aggregated into                     
                    an hourly summary                   
                    rather than sent                    
                    individually.                       

  BRD-004           Every             Supports audit    High
                    notification and  review and        
                    approval action   tracking of       
                    must be logged in employee actions  
                    an audit log for  and response      
                    review.           times.            
  -----------------------------------------------------------------------

## 5. Product Requirements Document (PRD)

  -------------------------------------------------------------------------
  ID                Product capability  User value        Priority
  ----------------- ------------------- ----------------- -----------------
  PRD-001           Human-in-the-Loop   Pauses workflow   High
                    Webhook Gate        execution until   
                                        an approval       
                                        callback is       
                                        received.         

  PRD-002           Escalation Timer    Monitors          High
                                        acknowledgement   
                                        status and        
                                        triggers backup   
                                        alerts when an    
                                        SLA expires.      

  PRD-003           Multi-Channel       Routes messages   High
                    Dispatcher          to Email, Slack,  
                                        or Twilio         
                                        according to      
                                        priority.         

  PRD-004           Alert Deduplication Evaluates alert   Medium
                    Cache               frequency and     
                                        suppresses        
                                        duplicates within 
                                        configurable      
                                        windows.          
  -------------------------------------------------------------------------

## 6. Software Requirements Specification (SRS)

### 6.1 Functional requirements

  -----------------------------------------------------------------------
  ID                      Requirement             Priority
  ----------------------- ----------------------- -----------------------
  FR-001                  The system shall send   High
                          an interactive Slack    
                          Block Kit message       
                          containing buttons and  
                          unique action metadata. 

  FR-002                  The workflow shall      High
                          pause and wait for an   
                          incoming webhook        
                          containing the user's   
                          approved/rejected       
                          decision and correlate  
                          it with the waiting     
                          run.                    

  FR-003                  The workflow shall      High
                          implement an execution  
                          timer that triggers     
                          escalation if the       
                          incident is not         
                          acknowledged within 10  
                          minutes.                

  FR-004                  The system shall query  High
                          a tracking variable or  
                          database record to      
                          determine whether a     
                          similar alert was sent  
                          within the last 10      
                          minutes, and bypass     
                          sending when throttling 
                          is active.              

  FR-005                  The workflow shall      High
                          route CRITICAL alerts   
                          to Twilio SMS and       
                          WARNING/INFO alerts to  
                          the appropriate         
                          Slack/Email channels.   
  -----------------------------------------------------------------------

### 6.2 Non-functional requirements

  -------------------------------------------------------------------------
  ID                Category          Requirement         Priority
  ----------------- ----------------- ------------------- -----------------
  NFR-001           Reliability /     The approval        High
                    security          webhook listener    
                                      must support        
                                      token-signature     
                                      verification to     
                                      prevent spoofed     
                                      approvals.          

  NFR-002           Performance       The                 High
                                      decision-handling   
                                      webhook must        
                                      respond to Slack    
                                      within 3,000 ms to  
                                      avoid user-facing   
                                      timeout errors.     
  -------------------------------------------------------------------------

## 7. User stories and acceptance criteria

### US-001: Tiered escalation alerts

**User story:** As an Operations Coordinator, I want system incidents to
alert me on Slack and escalate to my phone via SMS if I do not
acknowledge them within 10 minutes, so that issues are not left
unmanaged.

**Acceptance criteria:** 1. Given a database tracking table records a
new critical incident, 2. When the workflow starts, it sends a Slack
message warning the team. 3. The workflow waits up to 10 minutes for the
status to change to `acknowledged`. 4. If the status remains `pending`
at the end of the wait period, the workflow triggers a Twilio node to
send an SMS to the secondary engineer on-call.

### US-002: Slack interactive approval gate

**User story:** As a Finance Reviewer, I want purchase requests to
present Approve/Reject buttons in my Slack channel so that I can process
approvals in one click.

**Acceptance criteria:** 1. Given a purchase order exceeds the review
threshold of \$10,000, 2. When the workflow is triggered, it posts an
interactive Slack message with Approve and Reject buttons. 3. The
workflow pauses and creates a temporary webhook listener. 4. When the
reviewer clicks Approve in Slack, the button sends a POST request
containing the decision payload to the n8n webhook. 5. The workflow
resumes, writes the decision status back to the records database, and
updates the Slack message to display `Approved by [User]`.

## 8. Test Cases Blueprint — Trainer's Original Test Suite

**Source document:** *Topic 12: Communication, Alerts & Workflow Notifications – Test Cases*  
**Version:** 1.0  
**Status:** Draft  
**Last updated:** 2026-06-25  
**Prepared by:** Lead Automation Architect (Antigravity AI)

### 8.1 Test plan overview

This test suite validates communication workflows, automated alerts, and approval integrations within n8n. It covers human-in-the-loop (HITL) interactive webhooks, SLA-driven escalation, alert throttling during alert storms, priority-based multi-channel routing, and callback security.

**Test scope**
- **Approval flow validation:** Interactive payloads resume running workflows correctly.
- **Escalation verification:** Wait timeouts execute fallback pathways when SLAs are breached.
- **Fatigue prevention:** Throttling suppresses repeat alerts while allowing critical items to route.
- **Security validation:** Incorrect or unverified signatures are blocked.

### 8.2 TC-001: Slack Interactive Approval

- **Linked requirement:** FR-001, FR-002
- **Preconditions:** n8n is running; the workflow contains an active webhook listener; a mock Slack channel is ready for test posts.
- **Steps:**
  1. Trigger the workflow manually to generate a mock Slack interactive block message.
  2. Locate the generated message in the Slack channel.
  3. Click **Approve**, posting a payload to the n8n webhook callback.
  4. Inspect the n8n execution panel for the resumed workflow run.
- **Expected result:**
  - The n8n webhook receives the button-click payload successfully.
  - The workflow resumes and routes through the approved branch.
  - The Slack message text updates to show the request was approved.
- **Priority:** High

### 8.3 TC-002: SLA Escalation Timer

- **Linked requirement:** FR-003
- **Preconditions:** The acknowledgement wait timer is set to a test limit of 30 seconds, or 10 minutes for standard staging.
- **Steps:**
  1. Trigger the incident alert workflow.
  2. Leave the primary engineer status as `pending`; do not send an acknowledgement callback.
  3. Wait for the timeout to expire.
  4. Verify that the downstream workflow path executed.
- **Expected result:**
  - The workflow waits for the defined timeout.
  - When the timeout threshold passes without acknowledgement, it routes to the escalation branch.
  - An escalation notification (Twilio SMS/call) is sent to the secondary contacts.
- **Priority:** High

### 8.4 TC-003: Alert Rate Limiting (Throttling)

- **Linked requirement:** FR-004
- **Preconditions:** Throttling state database or static key exists; the throttling window is configured to 10 minutes.
- **Steps:**
  1. Trigger an alert payload (Alert A) from a monitoring source.
  2. Verify that the alert is delivered successfully to Slack.
  3. Within two minutes, trigger the identical Alert A payload two more times.
  4. Inspect execution logs and message logs.
- **Expected result:**
  - The first alert is successfully delivered.
  - The second and third alerts are blocked by the throttling code node.
  - No duplicate messages are posted to Slack within the 10-minute window.
- **Priority:** High

### 8.5 TC-004: Multi-channel Urgency Routing

- **Linked requirement:** FR-005
- **Preconditions:** Twilio, Slack, and Email nodes are configured with valid staging credentials.
- **Steps:**
  1. Trigger the notification workflow with payload property `priority: "CRITICAL"` and inspect the outputs.
  2. Trigger the workflow with payload property `priority: "INFO"` and inspect the outputs.
- **Expected result:**
  - For `CRITICAL`, notifications are dispatched to both Slack (operational channel) and Twilio (SMS).
  - For `INFO`, notifications are routed only to the daily summary Email, skipping Slack and Twilio.
- **Priority:** High

### 8.6 TC-005: Webhook Security Signature Check

- **Linked requirement:** NFR-001
- **Preconditions:** The n8n Webhook node has verification headers enabled, such as matching Slack signature headers.
- **Steps:**
  1. Formulate a raw HTTP POST request containing a mock approval decision payload.
  2. Include an invalid verification signature in the header.
  3. Send the request directly to the n8n webhook URL.
- **Expected result:**
  - The n8n webhook rejects the request.
  - HTTP 401 Unauthorized or an equivalent error is returned.
  - The corresponding workflow execution does not resume.
- **Priority:** High

### 8.7 Requirement-to-test traceability

| Test case | Linked requirement | Primary validation |
|---|---|---|
| TC-001 | FR-001, FR-002 | Interactive approval payload, resume, approved branch, Slack message update |
| TC-002 | FR-003 | Wait timeout and escalation branch |
| TC-003 | FR-004 | First alert delivery and duplicate suppression within 10 minutes |
| TC-004 | FR-005 | CRITICAL Slack + SMS; INFO Email only |
| TC-005 | NFR-001 | Invalid signature rejected with HTTP 401 or equivalent |

## 9. Assumptions, dependencies, and risks

### 9.1 Assumptions

-   **ASM-001:** The Slack workspace integration supports incoming
    webhooks and interactive button URL routing.
-   **ASM-002:** SMTP/Email server configuration permits programmatic
    connections and is not blocked by spam filters.

### 9.2 Dependencies

-   **DEP-001:** n8n instance version 1.0.0+ with HTTP Request, Slack,
    and Wait nodes available.
-   **DEP-002:** Valid third-party accounts and credentials, including
    Slack app credentials and Twilio API credits, for live integrations.

### 9.3 Risks

-   **RSK-001 -- Indefinite wait:** If a reviewer never clicks Approve
    or Reject, a workflow may remain waiting and consume execution
    resources.
-   **Mitigation:** Configure a fallback timeout (the trainer gives 24
    hours as an example) that routes the request to an `Auto-Rejected`
    status.

## 10. Glossary

  -----------------------------------------------------------------------
  Term                                Definition
  ----------------------------------- -----------------------------------
  Block Kit                           Slack UI framework for rich
                                      interactive layouts containing
                                      buttons, inputs, and other blocks.

  Human-in-the-Loop (HITL)            A design pattern that inserts human
                                      checkpoints into an automated
                                      process.

  SLA                                 Service Level Agreement; a
                                      commitment regarding target
                                      response or resolution times.

  Webhook callback                    An HTTP endpoint that receives an
                                      external event and resumes a
                                      waiting workflow.

  Alert throttling                    Limiting alert frequency within a
                                      time window to reduce noise.
  -----------------------------------------------------------------------

## 11. Traceability matrix

  -------------------------------------------------------------------------
  Source item BRD         PRD         Functional    User story  Test case
                                      requirement               
  ----------- ----------- ----------- ------------- ----------- -----------
  BRD-001     BRD-001     PRD-001     FR-001,       US-002      TC-001,
                                      FR-002                    TC-005

  BRD-002     BRD-002     PRD-002     FR-003,       US-001      TC-002,
                                      FR-005                    TC-004

  BRD-003     BRD-003     PRD-004     FR-004        ---         TC-003

  BRD-004     BRD-004     PRD-003     FR-001,       US-002      TC-001
                                      FR-002                    

  NFR-001     ---         ---         NFR-001       ---         TC-005
  -------------------------------------------------------------------------

## 12. Implementation alignment and evidence notes

This section is an as-built clarification based on the Topic 12 workflow
documentation and execution outputs shared during implementation. It is
not part of the trainer's original requirement wording.

  -----------------------------------------------------------------------
  Requirement             Observed implementation Alignment status
                          / evidence              
  ----------------------- ----------------------- -----------------------
  FR-001 -- Interactive   Slack-style Block Kit   Partially demonstrated;
  Block Kit               payloads are built in   live Slack interaction
                          the workflow; the       not verified
                          approval interaction    
                          was demonstrated using  
                          a mock button-to-Wait   
                          flow.                   

  FR-002 -- Wait and      Approve and Reject      Mock flow verified;
  decision callback       paths work in the       production Slack
                          connected mock          callback
                          button-to-Wait          correlation/database
                          workflow.               update not fully
                                                  verified

  FR-003 -- 10-minute     The 10-minute Wait      PASS in mock test;
  escalation              timeout was previously  retain execution
                          tested and the expected evidence
                          mock escalation output  
                          was received.           

  FR-004 -- 10-minute     Throttle evaluator and  PASS for demonstrated
  duplicate suppression   IF routing were         sample; cache is
                          corrected; duplicate    workflow static data,
                          INC-902 was suppressed  not an external durable
                          and allowed incidents   database
                          continued.              

  FR-005 -- urgency       P1 Slack/SMS mock, P3   PASS for demonstrated
  routing                 digest queue, and INFO  mock routing; verify
                          Email mock outputs were every required live
                          observed.               channel before
                                                  production

  NFR-001 -- signature    Callback uses a         Partial; cryptographic
  verification            shared-secret           signature/HMAC
                          `x-approval-token`      verification is not
                          comparison and returns  implemented
                          401 for invalid/missing 
                          token.                  

  NFR-002 -- 3,000 ms     A response test was     Reported/tested; do not
  response                reported during         claim a measured SLA
                          implementation.         without the timing
                          Preserve the actual     record
                          measured execution      
                          evidence.               

  BRD-004 -- audit log    Structured notification Audit generation
                          audit records were      verified; persistent
                          generated in workflow   database/external
                          outputs.                audit-store persistence
                                                  not verified
  -----------------------------------------------------------------------

### Additional as-built notes

-   The sample simulator records include `INC-901` (P1 Critical),
    `INC-902` (duplicate P1, throttled), `INC-903` (P3 Minor, digest
    queue), and `INC-905` (INFO, Email).
-   `IF_IsThrottled` must route **True** to `Log_SuppressedAlert` and
    **False** to `Switch_RouteBySeverity`.
-   Slack, Twilio, and SMTP outputs shown in the assessment are marked
    `MOCK`. A mock `DELIVERED` or `ok: true` result is not proof of
    delivery to a real external recipient.
-   Workflow audit output is not proof of persistent audit storage.
-   The throttle cache uses n8n workflow static data; production use
    should use a durable, appropriately secured store with retention and
    concurrency controls.
-   The trainer's TC-004 wording says CRITICAL triggers both SMS and
    Slack. The demonstrated workflow has mock Slack and SMS paths for
    P1. The INFO Email and P3 digest paths were also verified.
-   The 10-minute throttle window and 10-minute approval timeout are
    separate controls and should not be confused.

## 13. Submission guidance

Submit this requirement document together with `README.md`,
`instructions.md`, and `test_results.md`. Keep n8n execution
screenshots/records for the approval decision, timeout escalation,
throttling, routing, and webhook validation tests. Clearly label mock
integrations and any partially implemented requirements.
