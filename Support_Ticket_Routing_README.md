# Support Ticket Priority Routing Workflow

**Topic 4 Assessment | Production-Ready n8n Workflow**  
**Document date:** 22 October 2026  
**Status:** Implementation blueprint; execution evidence and actual configuration require verification.

## 1. Purpose and scope
Receive support tickets through `POST /support-ticket`, validate required fields, normalize priority, route to the correct team, and generate ticket-specific email notifications. Invalid tickets must not reach notification nodes. Unsupported priorities must reach General Support.

## 2. Workflow architecture

```text
Webhook_POST_SupportTicket
  -> Validate_Required_Fields (IF or Filter + explicit reject branch)
       | invalid -> Rejected_Missing_Fields (log / controlled response)
       | valid -> Normalize_Priority (Edit Fields)
                   -> Build_Notification (Edit Fields)
                   -> Route_By_Priority (Switch)
                        | Low -> Notify_General_Support
                        | Medium -> Notify_Tier_1
                        | High -> Notify_Tier_2
                        | Urgent -> Notify_On_Call
                        | Fallback -> Notify_General_Support_Fallback
```

**Important response design:** An immediately responding Webhook acknowledges receipt, not successful validation or email delivery. If the assessor expects HTTP 400 for invalid tickets, use `Respond to Webhook` with response timing configured accordingly, rather than an immediate unconditional 200 response. Do not claim an invalid HTTP response has been implemented without testing it.

## 3. Node configuration blueprint

| Node | Type | Key configuration |
|---|---|---|
| Webhook_POST_SupportTicket | Webhook | POST; path `support-ticket`; test/production URL; JSON payload |
| Validate_Required_Fields | IF | name, email, subject, priority all nonempty after trim; optional email format check |
| Rejected_Missing_Fields | Edit Fields / log | record reason and rejected payload; no notification |
| Normalize_Priority | Edit Fields | `{{ String($json.body?.priority ?? $json.priority ?? '').trim().toLowerCase() }}` |
| Build_Notification | Edit Fields | extract name/email/subject/description; construct subject/body once |
| Route_By_Priority | Switch | exact matches `low`, `medium`, `high`, `urgent`; fallback output |
| Notify_* | Send Email | separate destination per branch, SMTP credential stored in n8n |

### Required field validation
For a Webhook node, incoming JSON is commonly nested under `$json.body`. Check the actual execution data. Example n8n expression for an IF boolean condition:

```javascript
{{ ['name','email','subject','priority'].every(k => String(($json.body ?? $json)[k] ?? '').trim().length > 0) }}
```

Use an IF node's boolean comparison (true) to route valid versus invalid items. If a Filter node is required by the rubric, use it on the valid path but retain an IF reject branch so rejected records are visible.

### Normalization and routing

```javascript
{{ String($json.body?.priority ?? $json.priority ?? '').trim().toLowerCase() }}
```

Route `low` to General Support; `medium` to Tier 1; `high` to Tier 2; `urgent` to on-call escalation; all other values (e.g., `Critical`) to General Support fallback. Configure Switch fallback output explicitly.

### Dynamic notification template

```text
Subject: [{{ $json.priorityNormalized.toUpperCase() }}] {{ $json.subject }}
Customer: {{ $json.name }}
Email: {{ $json.email }}
Priority: {{ $json.priorityNormalized }}
Subject: {{ $json.subject }}
Description: {{ $json.description || 'Not provided' }}
```

In `Build_Notification`, explicitly map `name`, `email`, `subject`, and `description` from the Webhook payload and retain `priorityNormalized`; use the output fields in downstream email nodes. Set team addresses using approved configuration or credentials; never invent real destination addresses.

## 4. Example input and expected routing

```json
{
  "name": "Alex Rivera",
  "email": "alex@example.com",
  "subject": "Cannot access account",
  "priority": " HIGH ",
  "description": "Login returns an error"
}
```

Expected: validation passes; priority becomes `high`; Tier 2 email branch selected; email contains all provided ticket details. This is an **illustrative expected result**, not evidence of execution.

## 5. Test execution matrix (fill actual results)

| ID | Scenario | Expected result | Actual result / evidence |
|---|---|---|---|
| TC-01 | Valid Low | General Support notification | Pending |
| TC-02 | Valid Medium | Tier 1 notification | Pending |
| TC-03 | Valid High | Tier 2 notification | Pending |
| TC-04 | Valid Urgent | On-call notification | Pending |
| TC-05 | Priority `Critical` | Fallback General Support | Pending |
| TC-06 | Priority ` HIGH ` | Normalized and routed to Tier 2 | Pending |
| TC-07 | Missing name | Rejected; no notification | Pending |
| TC-08 | Missing email | Rejected; no notification | Pending |
| TC-09 | Missing subject | Rejected; no notification | Pending |
| TC-10 | Missing priority | Rejected; no notification | Pending |
| TC-11 | SMTP failure | Error visible, no false delivery claim | Pending |
| TC-12 | Missing description | Safe placeholder, no expression crash | Pending |

## 6. Risks and mitigations

- **Priority case/whitespace:** trim and lowercase before Switch matching.
- **Unexpected priority:** enable Switch fallback branch.
- **Missing fields:** reject before building email; log rejection reason.
- **Email failures:** use n8n execution logs and error handling; verify actual delivery.
- **Credentials and personal data:** store SMTP secrets in n8n credentials; redact real customer information from public screenshots and GitHub.
- **Webhook exposure:** consider authentication, rate limiting and spam protection before production use.
- **Response semantics:** immediate acknowledgment cannot guarantee downstream processing success.

## 7. Naming and annotations

Use descriptive names matching node roles. Add sticky notes for (1) webhook input schema and owner, (2) required-field validation and rejection, (3) priority normalization and Switch rules, (4) team routing and fallback, (5) SMTP dependencies and failure monitoring. Replace `OWNER_TBD` and `TEAM_EMAIL_TBD` with authorized project values.

## 8. GitHub repository structure

```text
support-ticket-priority-routing/
  README.md
  docs/Support_Ticket_Routing_Assessment.pdf
  workflow/Support_Ticket_Routing.json   # export from n8n
  evidence/                            # add actual screenshots
  config/example.env                   # placeholders only, no secrets
```

### Reproduction instructions
1. Import the exported workflow JSON into n8n.
2. Configure SMTP credentials and approved support-team destinations.
3. Check Webhook path, request method, and response mode.
4. Start listening for a test event; POST example JSON using Postman.
5. Execute each test case and record actual output and notification evidence.
6. Activate the workflow only after security and production checks.

## 9. Evidence and submission checklist

- [ ] Actual full-workflow canvas screenshot with node labels
- [ ] Valid webhook input and successful execution screenshot
- [ ] Rejected required-field case screenshot
- [ ] Low / Medium / High / Urgent branch evidence
- [ ] Fallback `Critical` and mixed-case priority evidence
- [ ] Email body preview and delivery evidence (redacted)
- [ ] Exported workflow JSON in public repository
- [ ] PDF documentation and this README in repository
- [ ] Loom/YouTube demonstration link verified
- [ ] Unique public GitHub repository URL verified
- [ ] LMS description includes challenges, assumptions and limitations

## 10. Suggested demo sequence
Show the full canvas; describe webhook and validation; send valid High input; demonstrate Tier 2 output; test missing field; test `Critical` fallback; show other priority branches; open the dynamic email body; show workflow JSON export and GitHub files. Only describe behavior as working if the execution actually demonstrates it.

## 11. Limitations and completion status
This package documents a proposed implementation derived from the assessment. No live n8n execution, actual SMTP notification, GitHub publication, or video upload was verified for this document. Add actual test evidence and URLs before submitting.
