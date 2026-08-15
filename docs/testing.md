# Testing Documentation

All 5 workflows were tested end-to-end during development, covering both success paths and edge cases. Screenshots for key tests are in [`screenshots/`](../screenshots/).

---

## WF1 — Lead Intake & Discovery

| Test ID | Purpose | Input | Steps | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|---|
| WF1-T1 | New lead submission | New lead (unique email + title) | POST to `/lead-intake` webhook | Lead inserted, audit logged | Lead inserted, audit logged | ✅ Pass |
| WF1-T2 | Duplicate lead submission | Same email + title as an existing lead | POST to `/lead-intake` webhook | Correctly skipped, no duplicate row, audit logged | Correctly skipped, no duplicate row, audit logged | ✅ Pass |

**Evidence:** `screenshots/wf1-lead-inserted.png`

---

## WF2 — AI Proposal Generation & Approval

| Test ID | Purpose | Input | Steps | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|---|
| WF2-T1 | Proposal generation | Valid `lead_id` | POST to `/generate-proposal` | Gemini produces a complete proposal; stored with `pending_approval` status | Gemini produced a complete, well-structured proposal; stored with pending_approval status | ✅ Pass |
| WF2-T2 | Approval via email link | Click "Approve" link | GET `/approve-proposal` (approve) | Proposal status → `sent`, client email sent, lead status updated | Proposal status → sent, client email sent, lead status updated | ✅ Pass |
| WF2-T3 | Rejection via email link | Click "Reject" link | GET `/approve-proposal` (reject) | Proposal status → `needs_revision`, no client email sent | Proposal status → needs_revision, no client email sent | ✅ Pass |

**Evidence:** `screenshots/wf2-proposal-approved.png`

---

## WF3 — Meeting Scheduling & Client Management

| Test ID | Purpose | Input | Steps | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|---|
| WF3-T1 | Meeting request, new client | New client's meeting details | Submit "Schedule a Meeting" form | Client created, meeting logged, confirmation sent | Client created, meeting logged, confirmation sent | ✅ Pass |
| WF3-T2 | Meeting request, existing client | Existing client's meeting details | Submit form again | No duplicate client created, second meeting logged correctly | No duplicate client created, second meeting logged correctly | ✅ Pass |

**Evidence:** `screenshots/wf3-client-inserted.png`

---

## WF4 — Milestone Tracking & Invoicing

| Test ID | Purpose | Input | Steps | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|---|
| WF4-T1 | Completed, uninvoiced milestone | Milestone marked complete, not yet invoiced | Daily schedule run | Invoice auto-generated, milestone marked invoiced | Invoice auto-generated, milestone marked invoiced | ✅ Pass |
| WF4-T2 | Already-invoiced milestone | Milestone already invoiced | Daily schedule run | Correctly excluded from re-processing | Correctly excluded from re-processing | ✅ Pass |
| WF4-T3 | Overdue unpaid invoice | Invoice past due date, unpaid | Daily schedule run | Reminder email sent, `reminder_count` incremented | Reminder email sent, reminder_count incremented | ✅ Pass |

**Evidence:** `screenshots/wf4-invoice-generated.png`

---

## WF5 — Project Closure, Testimonials & Analytics

| Test ID | Purpose | Input | Steps | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|---|
| WF5-T1 | Project with incomplete milestones | Project with open milestones | Weekly schedule run | Correctly excluded from closure | Correctly excluded from closure | ✅ Pass |
| WF5-T2 | Project with all milestones complete | Project, all milestones done | Weekly schedule run | Project closed, testimonial requested | Project closed, testimonial requested | ✅ Pass |
| WF5-T3 | Analytics computation | Live lead/project/invoice data | Weekly schedule run | Correct lead count, conversion rate, and project counts | Correct lead count, conversion rate, and project counts across live data | ✅ Pass |

**Evidence:** `screenshots/wf5-project-closed.png`

---

## Summary

- **13 test cases** across 5 workflows — 13 passed, 0 failed.
- Coverage includes both success paths (new record creation) and edge cases (duplicates, already-processed records, overdue states, incomplete conditions).
- Screenshots show both the n8n execution result (green checkmarks on the canvas) and the resulting row written to the relevant Data Table, confirming each workflow does real, verifiable work — not just runs without erroring.

## Known Limitations

- Approval links in WF2 rely on `localhost`, so they only work when opened on the same machine running n8n (no external tunnel configured).
- Meeting scheduling (WF3) is manually initiated via form rather than parsing real client email replies.
- Marking a project "won" (creating its Projects/Milestones records) and marking invoices "paid" are currently manual Data Table edits rather than automated steps.
