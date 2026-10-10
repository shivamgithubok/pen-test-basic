# Authorized Penetration-Test Agent

**Submission type:** LangGraph agent for evaluation  
**What to review:** control flow, authorization gates, scope enforcement, and the report  
**How to run:** see [Run the demo](#run-the-demo)

## Summary

This is an engagement agent, not a website and not an exploit tool. A tester gives it a client, a written authorization reference, an approved target list, and structured observations. The agent decides whether that engagement is allowed to proceed, writes a plan, stops until a named person approves the plan, rejects notes about assets outside the target list, and drafts a client report.

The workflow is a LangGraph state machine. Each node does one job. Edges decide the next node from the current state. A checkpointer stores the thread so a reviewer can pause the run, approve the plan, and resume the same engagement.

## Why LangGraph

A penetration-test engagement is a sequence of decisions with a hard stop in the middle:

1. Is this engagement authorized?
2. What is the plan?
3. Has a human approved that plan?
4. Which notes are inside scope?
5. What goes in the report?

LangGraph fits that shape. The graph is explicit, each run is checkpointed, and the approval pause is a real interrupt rather than a comment in a prompt. A reviewer can see the path the agent took in the audit trail.

An LLM is not required for this version. The decisions are policy checks, so the same input always produces the same result. A later version can add a model only to phrase the plan and the executive summary. The allow and deny decisions should stay in code.

## Graph

```text
START
  │
  ▼
authorize ── blocked ──► END
  │
  │ allowed
  ▼
draft_plan
  │
  ▼
approval_gate ── waiting ──► END
  │
  │ approved by a named person
  ▼
normalize_findings
  │
  ▼
write_report
  │
  ▼
END
```

The compiled graph uses `interrupt_before=["approval_gate"]`. The first call stops after the plan is drafted. Nothing is normalized and no report is written until the caller sets `plan_approved` and `approver` on that thread and resumes it.

## State

One engagement is one state object:

| Field | Meaning |
| --- | --- |
| `engagement_id` | Identifier for this test |
| `client_name` | Client named in the report |
| `authorization_reference` | Written rules-of-engagement id |
| `authorization_confirmed` | Must be true before planning |
| `targets` | Only these assets may appear in findings |
| `requested_actions` | What the caller asked the agent to do |
| `observations` | Structured notes from an authorized tester |
| `plan` | Steps proposed by `draft_plan` |
| `plan_approved` / `approver` | Human approval required to continue |
| `findings` | In-scope notes after normalization |
| `rejected_observations` | Notes dropped because the asset was out of scope |
| `audit_log` | Decision made by each node |
| `report_markdown` | Draft report |
| `status` / `blocked_reason` | Outcome of the run |

## Node behavior

**authorize.** Continues only when authorization is confirmed, a reference id is present, at least one target is listed, and every requested action is on the allow list. Otherwise the graph ends with `status: blocked`.

Allowed actions: `scope_review`, `asset_inventory`, `configuration_review`, `finding_normalization`, `report_draft`.

Refused actions include exploitation, credential or password attacks, malware, payloads, denial of service, data exfiltration, privilege escalation, and lateral movement. Unknown action names are also refused.

**draft_plan.** Writes four review steps: confirm scope, inventory in-scope assets, normalize tester observations, and draft the report. Every step is marked as requiring a human. The node does not emit scanner commands.

**approval_gate.** Continues only when `plan_approved` is true and `approver` is a non-empty name. Otherwise the graph ends as `awaiting_approval`.

**normalize_findings.** Keeps an observation only when its asset is exactly an approved target, or a path under an approved target. Other notes are listed as rejected and do not enter the report as findings. Severity values outside `critical`, `high`, `medium`, `low`, and `info` are stored as `info`.

**write_report.** Renders Markdown with scope, method, plan, findings, rejected notes, and the audit trail.

## What this submission does not do

This agent does not scan networks, send exploits, guess passwords, deploy malware, or take data from a target. Observations are input from a tester. The agent’s job is to enforce the engagement rules and produce the draft report. Connecting a live scanner would be a later phase, and it would still have to pass the same authorization, allow-list, and approval gates.

## Run the demo

From the project root:

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python -m pentest_agent.demo
```

Verified with Python 3.12 and LangGraph 1.2.14. No API key is required.

### Case 1 — missing authorization

Input has an empty authorization reference and `authorization_confirmed: false`.

Result: `status: blocked`.  
Reason: missing written authorization. The planner does not run.

### Case 2 — forbidden action

Input is authorized, but `requested_actions` includes `exploitation`.

Result: `status: blocked`.  
Reason: the agent refuses that action and does not draft a plan.

### Case 3 — approval, then report

Input is engagement `RF-EVAL-003` for Northwind Labs, authorization `ROE-2026-014`, target `https://app.northwind.example`. Two notes are supplied. One is on the login path of that target. One is `https://admin.other-company.example`.

First result: the graph pauses with a four-step plan and `status: awaiting_approval`. No report yet.

After `plan_approved: true` and `approver: "Redfox reviewer"` on the same thread:

- `status: report_ready`
- 1 finding kept: session cookie missing the Secure flag, severity medium, on the in-scope login URL
- 1 note rejected: the other company’s admin host
- Markdown report includes scope, plan, finding, rejection, and audit trail

## Suggested review

| Check | Expected |
| --- | --- |
| Run with no authorization | Blocked before a plan exists |
| Request an exploit action | Blocked |
| Authorized run | Pauses with a plan before any report |
| Resume without an approver name | Stays waiting |
| Resume with approval | Report is produced |
| Note for a host outside `targets` | Listed under rejected, not as a finding |
| Audit trail | One line per node that ran |

## Possible next phase

If this control plane is accepted, the next engineering step is an optional language-model node that only rewrites the plan summary and the executive paragraph. Allow, deny, scope, and approval stay in the graph nodes above. Live testing tools would be separate adapters called only after `approval_gate`, and only for targets already on the engagement.
