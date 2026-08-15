# AI Freelancing Business Automation Platform
### Summer School'26 — N8N Capstone Project

---

## 1. Problem Analysis

### 1.1 Business Context
Freelancers operate as one-person businesses, responsible for every stage of the client lifecycle — from finding work to getting paid — without the dedicated sales, admin, or accounting teams a traditional business would have. This creates a heavy administrative burden that competes directly with billable, income-generating work.

### 1.2 Stakeholders
- **Primary user**: The freelancer, who needs to run their business efficiently with minimal manual overhead.
- **Clients**: Expect fast, professional communication (proposals, scheduling, invoicing) despite dealing with a solo operator.
- **(Indirect) Future collaborators/employees**: As the freelancer scales, the same automated processes reduce onboarding friction.

### 1.3 Pain Points
- Manually writing personalized proposals for every lead is time-consuming and inconsistent in quality.
- Tracking leads, meetings, milestones, and invoices across scattered tools (email, spreadsheets, notes) leads to dropped follow-ups.
- Chasing overdue payments is uncomfortable and easy to forget.
- Requesting testimonials and maintaining a portfolio is rarely prioritized despite its long-term value for winning future work.
- No consolidated view of business performance (revenue, conversion rate, project throughput).

### 1.4 Objectives
Build an n8n-based automation platform that:
1. Captures and deduplicates incoming leads automatically.
2. Uses AI to draft personalized, professional proposals — with a human approval gate before anything reaches a client.
3. Logs client meetings and maintains a client record automatically.
4. Automatically invoices completed work and follows up on overdue payments.
5. Closes out finished projects, requests testimonials, and surfaces business analytics — with minimal manual intervention.

---

## 2. Workflow Architecture

### 2.1 System Overview
The platform is built as **5 independent, interconnected n8n workflows**, sharing a common data layer of **9 n8n Data Tables**. Workflows communicate either by direct HTTP calls (webhook-to-webhook) or by writing to shared tables that a later, schedule-triggered workflow reads.

### 2.2 Workflow Interaction & Event Flow

```
                         ┌──────────────────────┐
   Webhook (POST)   ───▶ │ WF1: Lead Intake &    │──▶ Leads table
   external lead form    │ Discovery             │    (dedupe by email+title)
                         └──────────┬────────────┘
                                    │ HTTP call (lead_id)
                                    ▼
                         ┌──────────────────────┐
                         │ WF2: AI Proposal Gen   │──▶ Gemini drafts proposal
                         │ & Human Approval       │──▶ Proposals table
                         │ (2 webhook triggers:   │──▶ Email w/ Approve/Reject
                         │  generate + approve)   │    links (human-in-loop)
                         └──────────┬────────────┘
                                    │ on approval → email sent to client
                                    ▼ (client replies, fills form)
                         ┌──────────────────────┐
   Form Trigger      ──▶ │ WF3: Meeting          │──▶ Clients table (dedupe)
                         │ Scheduling & Client    │──▶ Meetings table
                         │ Management             │
                         └──────────┬────────────┘
                                    │ (manual: project marked "won",
                                    │  milestones created/edited in table)
                                    ▼
                         ┌──────────────────────┐
   Schedule (daily)  ──▶ │ WF4: Milestone        │──▶ Invoices table
                         │ Tracking & Invoicing   │    (auto-generate + overdue
                         │                        │     reminders)
                         └──────────┬────────────┘
                                    ▼
                         ┌──────────────────────┐
   Schedule (weekly) ──▶ │ WF5: Project Closure,  │──▶ Testimonials table
                         │ Testimonials &         │──▶ Business analytics
                         │ Analytics              │    → Audit_Log
                         └──────────────────────┘

Cross-cutting: every workflow writes to Audit_Log on each meaningful
event (success, duplicate, error/reminder, approval/rejection, closure).
```

### 2.3 Design Decisions Worth Noting
- **5 workflows instead of the suggested 6**: Invoice Generation & Payment Follow-up was merged into the Milestone Tracking workflow, since milestone completion is what triggers invoicing — keeping the two tightly coupled processes in one place reduces cross-workflow calls.
- **"If Row Exists" is not a branching node.** A common early mistake in this build: it has a single output (passes the item through only if a match is found). True/false branching on existence requires **two parallel Data Table nodes** — one using "If Row Exists," one using "If Row Does Not Exist" — both fed from the same source.
- **Respond to Webhook nodes should avoid hand-typed JSON with embedded `{{ }}` expressions.** This proved unreliable in testing (validation errors on data that was actually correct). The fix used throughout: build the response shape in a Set node first, then use "Respond With: First Incoming Item."
- **`.item` vs `.first()` when referencing another node.** `.item` relies on n8n's paired-item tracking, which can break across Data Table node chains for single-item flows — `.first()` proved more reliable there. However, in true per-item loops (e.g., WF4/WF5 processing multiple milestones or invoices), `.item` is required to correctly pair each output with its corresponding input.

---

## 3. Data Model — n8n Data Tables

| Table | Key columns (beyond auto id/createdAt/updatedAt) | Purpose |
|---|---|---|
| `Leads` | client_name, client_email, project_title, project_description, budget, deadline, source, status, dedupe_hash | Incoming opportunities |
| `Clients` | name, email, company | Converted contacts |
| `Proposals` | lead_id, ai_draft_text, final_text, status, revision_notes | AI-drafted + approved proposals |
| `Meetings` | lead_id, client_id, scheduled_time, status | Scheduled client calls |
| `Projects` | lead_id, client_id, title, total_value, status | Active/closed engagements |
| `Milestones` | project_id, title, amount, status, invoiced, completed_at | Per-project deliverable tracking |
| `Invoices` | project_id, milestone_id, amount, due_date, status, reminder_count | Billing + payment tracking |
| `Testimonials` | project_id, client_id, text, status, requested_at | Client feedback requests |
| `Audit_Log` | workflow_name, event_type, related_id, message | Cross-workflow audit trail |

Dedupe strategy: `Leads` uses a computed hash of lowercased email + project title to prevent duplicate insertions; `Clients` dedupes by email using n8n's native "If Row Exists" / "If Row Does Not Exist" pattern.

---

## 4. Workflow Documentation

### WF1 — Lead Intake & Discovery
- **Trigger**: Webhook (POST `/lead-intake`)
- **Nodes**: Receive New Lead → Normalize Lead Data → Compute Dedupe Hash → [Check Duplicate Lead / Check New Lead] → Insert Lead / Log Duplicate Skipped → Log Lead Inserted → Build Success/Duplicate Response → Respond
- **Key feature**: Dedupe check prevents duplicate leads from the same client/project combination.

### WF2 — AI Proposal Generation & Approval
- **Triggers**: Two webhooks — `/generate-proposal` (POST, called with a lead_id) and `/approve-proposal` (GET, clicked from an email link)
- **Nodes**: Get Lead → Build Gemini Prompt → Generate Proposal (Google Gemini) → Insert Proposal → Send Approval Email (Gmail, with Approve/Reject links) → Acknowledge — then on the approval webhook: Update Proposal Status → branch on decision → (approved: Get Client Info → Send Proposal to Client → Update Lead Status) / (rejected: mark needs_revision) → Respond
- **Key feature**: This is the core AI feature — Gemini generates a personalized proposal considering client name, project details, budget, and deadline. A human approval gate (email-based) sits between AI generation and anything reaching the actual client.

### WF3 — Meeting Scheduling & Client Management
- **Trigger**: n8n Form Trigger (`Schedule a Meeting` form)
- **Nodes**: Get Lead for Meeting → [Check Client Exists / Client Does Not Exist → Insert New Client] → Insert Meeting → Send Meeting Confirmation → Update Lead Status
- **Key feature**: Client deduplication — reuses an existing client record rather than creating duplicates on repeat meeting requests.

### WF4 — Milestone Tracking & Invoicing
- **Trigger**: Schedule (daily, 9:00 AM)
- **Branch A**: Get Uninvoiced Completed Milestones → Generate Invoice → Mark Milestone Invoiced (loop, per milestone)
- **Branch B**: Get Overdue Invoices → Get Project/Client for Invoice → Send Payment Reminder → Increment Reminder Count (loop, per invoice)
- **Key feature**: Fully automated invoicing on milestone completion, with escalating overdue-payment reminders.

### WF5 — Project Closure, Testimonials & Analytics
- **Trigger**: Schedule (weekly, Monday 9:00 AM)
- **Branch A**: Get Active Projects → Get Project Milestones → Check All Milestones Complete (Code node) → Update Project Status → Get Project/Client Details → Insert Testimonial Request → Send Testimonial Request
- **Branch B**: Get All Leads/Projects/Invoices for Analytics → Compute Analytics (Code node) → Log Analytics Summary
- **Key feature**: Automatic project closure detection and computed business metrics (lead conversion rate, revenue collected, active/closed project counts).

---

## 5. Advanced Features Implemented

| Requirement | Where |
|---|---|
| AI-powered decision making | WF2 (Gemini proposal generation) |
| Human approval | WF2 (email-based approve/reject gate) |
| Error handling / retries | Retry On Fail enabled on all external API nodes (Gemini, Gmail) across WF2-WF5 |
| Logging / audit trail | Audit_Log table, written to by all 5 workflows |
| Scheduled workflows | WF4 (daily), WF5 (weekly) |
| Webhook-triggered workflows | WF1, WF2 (x2), WF3 (form) |
| Conditional branching | Dedupe checks (WF1, WF3), approval decision (WF2), milestone completeness (WF5) |
| Loops | Per-milestone invoicing (WF4), per-invoice reminders (WF4), per-project checks (WF5) |

---

## 6. Testing Documentation

All 5 workflows were tested end-to-end during development, including both success and edge-case paths:

| Workflow | Test | Result |
|---|---|---|
| WF1 | New lead submission | Lead inserted, audit logged |
| WF1 | Duplicate lead submission (same email+title) | Correctly skipped, no duplicate row, audit logged |
| WF2 | Proposal generation | Gemini produced a complete, well-structured proposal; stored with pending_approval status |
| WF2 | Approval via email link | Proposal status → sent, client email sent, lead status updated |
| WF2 | Rejection via email link | Proposal status → needs_revision, no client email sent |
| WF3 | Meeting request, new client | Client created, meeting logged, confirmation sent |
| WF3 | Meeting request, existing client | No duplicate client created, second meeting logged correctly |
| WF4 | Completed, uninvoiced milestone | Invoice auto-generated, milestone marked invoiced |
| WF4 | Already-invoiced milestone | Correctly excluded from re-processing |
| WF4 | Overdue unpaid invoice | Reminder email sent, reminder_count incremented |
| WF5 | Project with incomplete milestones | Correctly excluded from closure |
| WF5 | Project with all milestones complete | Project closed, testimonial requested |
| WF5 | Analytics computation | Correct lead count, conversion rate, and project counts across live data |

### Known Limitations
- Approval links rely on `localhost`, so they only work when opened on the same machine running n8n (no external tunnel configured).
- Meeting scheduling is manually initiated via form rather than parsing real client email replies.
- Marking a project "won" (creating its Projects/Milestones records) and marking invoices "paid" are currently manual data-table edits rather than automated steps.
