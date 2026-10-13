# Healthcare Customer Service Automation (Kore.ai XO 11)

A virtual assistant built on **Kore.ai XO 11** that handles routine healthcare admin requests. It asks the patient for an ID, looks the record up through a REST API, and uses the real response to decide what to say next. When a case needs a person, it hands over to a human agent with the relevant details.

It stays on the administrative side. It never diagnoses, recommends treatment or interprets medical results.

## Use cases

| Use case | What the patient can do | Status |
|---|---|---|
| 1. Appointment Management | Check, reschedule or cancel an appointment | Built |
| 2. Insurance Eligibility and Coverage Status | Check whether insurance is active and verified | Built |
| 3. Lab Report Status | Check report status | Not started |

## How it works

```
Patient message -> Intent -> Entity (ID) -> Service node (GET to mock API)
   -> Script node (exact-ID match, sets status) -> Condition per status
   -> Message reply, or Agent Transfer
```

1. The dialog is triggered by an intent (for example "Reschedule my appointment").
2. An entity node collects the ID (`appointmentId` or `memberId`).
3. A service node calls the mock API with the ID in the URL.
4. A script node finds the exact record and sets context variables, ending with one status value.
5. Conditions on the script node send the conversation to one message per status.
6. Statuses that need manual handling go through a confirmation, a context script and an Agent Transfer node.

## Use Case 1: Appointment Management

API: `GET /appointments?appointmentId=<ID>`

| Status | Bot behaviour |
|---|---|
| CONFIRMED | Shows date, time and provider; asks whether to keep, reschedule or cancel |
| RESCHEDULED | Shows the updated details and asks if more help is needed |
| CANCELLED | Confirms the cancellation and gives the reference |
| RESCHEDULE_NOT_ALLOWED | Explains it cannot be rescheduled here; offers a representative |
| NOT_FOUND | Asks the patient to verify the appointment ID |

![Use Case 1 canvas](screenshots/uc1-dialog-canvas.png)
![Use Case 1 confirmed test](screenshots/uc1-confirmed-test.png)

## Use Case 2: Insurance Eligibility

API: `GET /insurance?memberId=<ID>`

The script combines `eligibilityStatus` and `verificationStatus` into a single `insuranceStatus`.

| Status | Bot behaviour |
|---|---|
| ACTIVE_VERIFIED | Confirms active coverage with payer, plan and dates |
| INACTIVE | Explains coverage is inactive and offers help contacting support |
| VERIFICATION_REQUIRED | Explains it could not be fully verified |
| PAYER_MISMATCH | Explains the details do not match provider records; offers a transfer |
| NOT_FOUND | Asks the patient to verify the member ID |

![Use Case 2 canvas](screenshots/uc2-dialog-canvas.png)
![ACTIVE_VERIFIED test](screenshots/uc2-active-verified-test.png)
![PAYER_MISMATCH test](screenshots/uc2-payer-mismatch-test.png)

## Agent transfer

The bot transfers when a case cannot be handled automatically: rescheduling not allowed, a payer mismatch, or a patient who agrees to speak to a representative. Only administrative details are passed, never medical information.

- Use Case 1 passes: appointmentId, status, date, time, provider, eligibility flags, transfer reason.
- Use Case 2 passes: memberId, eligibilityStatus, payerName, planName, verificationStatus, transfer reason, and any issue the patient reported.

![Transfer debug log](screenshots/uc2-agent-transfer-debug.png)

**Trial limitation:** no live agent system was connected, so after the transfer starts the bot says it cannot complete the request. The proof is in the debug log and the passed context data.

## Repository structure

```
mock-api/       Sample data for the two mock endpoints
scripts/        Script node code used in the Kore.ai dialogs
docs/           Test cases and the project presentation (.pptx)
screenshots/    Dialog canvases and test results
kore-export/    Place the exported Kore.ai app here
```

## Setup

### 1. Mock API (mockAPI.io)
1. Create a project, then two resources: `appointments` and `insurance`.
2. Match the field names in `mock-api/*.json`.
3. Open each resource's **Data** view, replace the generated records with the JSON from this repo, and click **Update**.
4. Test in a browser: `https://<your-project-id>.mockapi.io/api/v1/appointments?appointmentId=APT45021`

### 2. Kore.ai XO 11
1. Create an app named `HealthcareCustomerServiceBot`.
2. Create dialogs `AppointmentManagement` and `InsuranceEligibility` with their training utterances.
3. Add the entity, service and script nodes. Service type: Custom Service, Web Service, Rest, method GET. Put the ID in the URL, for example `.../appointments?appointmentId={{context.entities.appointmentId}}`.
4. Paste the matching file from `scripts/` into each script node.
5. On each script node, add one condition per status (`context.appointmentStatus` or `context.insuranceStatus` Equals To the status text) pointing to its message node.
6. Add the confirmation, transfer script and Agent Transfer nodes.
7. Click **Sync Updates**, then try it in the Test panel.

Or import the app from `kore-export/` if an export has been added.

## Testing

Test IDs and expected results are in [docs/test-cases.md](docs/test-cases.md).

Sample IDs: appointments `APT45021` to `APT45024`; insurance `INS784512` to `INS784515`; any other ID returns not found.

## Problems found and fixed

| Problem | Cause | Fix |
|---|---|---|
| "I could not understand that" | New intent not trained or synced | Synced updates and trained utterances |
| RESCHEDULED returned a not-found reply | Condition value held a node name | Corrected the condition values |
| Empty "If context exists" rows | Leftover conditions | Deleted them and set fallback paths |
| Unknown or partial IDs | Loose API search and error text | Exact-ID match; non-list results mean NOT_FOUND |

## Current status and roadmap

Done: both use cases, mock APIs, status-based replies, Use Case 2 transfer tests.

Still to do:
- [ ] Retry limits and fallback for wrong IDs
- [ ] Friendly handling of backend errors
- [ ] Transfer when the patient asks for a representative
- [ ] VERIFICATION_REQUIRED transfer path (Use Case 2)
- [ ] Test and screenshot the Use Case 1 transfer and remaining statuses
- [ ] Use Case 3: Lab Report Status

## Tech stack

Kore.ai XO 11 (Dialog Builder), mockAPI.io, JavaScript script nodes.

## Author

Your name here. Add a license of your choice.

