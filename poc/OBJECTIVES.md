# Broker Onboarding Workflow – POC Objectives

Status: draft, written before the discovery meeting with Broker Support. Update it after that meeting.

Related: [ARCHITECTURE.md](ARCHITECTURE.md) has the system design, diagrams and decision log.

## 1. Background

Broker Support says onboarding a new broker means getting approvals from several people. The work seems to happen mostly over email, and some onboardings take **up to 4 months**.

The idea is to model onboarding as a **human-in-the-loop workflow** built with the **Microsoft Agent Framework** (Python, `agent-framework`), shown on a page in the existing internal support portal.

## 2. Problem statement (hypotheses to validate)

We don't yet know where the 4 months go. Our working hypotheses are:

| Delays a workflow can fix | Delays it can't fix |
|---|---|
| Nobody knows whose turn it is, so items sit in inboxes | The broker is slow to supply documents |
| There are no chasers, SLAs or escalations | External checks are slow (FCA register, credit, legal) |
| Back-and-forth gets lost in email threads | An approver is deliberately slow, or the policy is contested |
| Nobody can see the status, so people keep asking "where is this at?" | |

**Objective 0:** Find out which column most of the delay falls into before we commit to a design. Visibility alone could be the headline benefit.

## 3. POC objectives

### O1 – Prove the workflow engine for long-running human approvals (no AI)
Build the workflow with plain-code executors only, so the engine is tested without model behaviour muddying the results.

- 3–4 fake approval stages. Each one pauses for a human response (`ctx.request_info` / `@response_handler`).
- Each stage accepts **Approve**, **Reject / Request changes** and **Comment** from the approver, and an **Override approval** from Broker Support (see O8).
- **Pushback loop:** a rejection or change request goes to a *Revise* step, then back to the same approver, so the flow isn't a straight line.
- **Durable state:** checkpoints are stored in a real database (Postgres), so a case survives process restarts and can resume later.
- Each broker onboarding is one workflow instance, identified by a **case ID**.

**Success:** we can start a case, pause it, kill the process, resume it, push back and resubmit, and finish all stages.

### O2 – Dashboard as the system of record
- A page (in the support portal or a standalone POC page) lists pending items per approver.
- The Approve, Reject and Comment buttons call an API, which sends the response into the paused workflow.
- The page shows the status and the **elapsed time per stage** for each case.

**Success:** an approver can act on a case without the console, and anyone can see where a case is and how long it's been there.

### O3 – Timers, chasers and escalation
- Per-stage SLA timers. For example, a nudge at *N* days and an escalation at *M* days.
- The POC uses **minutes instead of days**, so pause, chase, pushback and resume can all be demoed in one sitting.

**Success:** an overdue stage triggers a reminder, then an escalation, without anyone stepping in.

### O4 – Audit trail
- Our own event and audit tables record who decided what, when, through which channel, and with what comment. These are kept separately from the framework's checkpoints.
- Overrides are recorded as their own decision type, never as an ordinary approval (see O8).
- This is non-negotiable because we're a bank.

**Success:** the full decision history of any case can be produced from the audit tables alone.

### O5 – Workflow versioning
Cases can run for months, so the workflow design will change while cases are still in flight.

- Stamp every case with a `workflow_version`.
- Keep the workflow **thin**: it only decides "what's the next step". The real case state lives in our own tables.
- Default strategy: in-flight cases stay pinned to the version they started on. Make changes additive where possible. Write an explicit migration only for breaking changes.
- **Experiment:** change the graph while a case is paused, try to resume it, and record what breaks.

**Success:** we have a documented, tested answer to "what happens to in-flight cases when we change the workflow?"

### O6 – Email channel (later stage)
Email and the dashboard are two channels into the same workflow response.

1. **Email out:** notifications include a deep link to the dashboard item (via a shared mailbox or Microsoft Graph). This is low risk.
2. **Email in:** watch an inbox through Graph, match each reply to its case using the case ID in the subject, and post the response into the workflow.

**Success:** an approver can act from an email link, and later by replying to the email.

### O7 – Where AI adds value (later stage)
Approval routing is deterministic, so it stays as plain code. AI is added as extra nodes in the same graph:

- Classify inbound email replies as approve, reject or query, and pull out the comments.
- Summarise the broker pack for each approver and flag missing documents.
- Draft chasers and "please clarify" messages.

Guardrail: at first the agent only **suggests** a decision and a human confirms it. It never auto-approves.

**Success:** the AI steps measurably cut approver effort. They are also what justifies this framework over Logic Apps, Power Automate or a plain state machine.

### O8 – Broker Support approval override
Approvers sometimes give approval verbally, for example on a call or in a meeting. Broker Support must be able to record that approval on the approver's behalf so the case doesn't stall.

- Any pending approval stage can be completed by a Broker Support user with **Override approval**.
- The override must name the approver who gave the verbal approval.
- Justification is required: the user must enter a **comment** and/or attach **supporting files** (for example a call note, a meeting minute or a forwarded message). An override with neither is rejected.
- The override goes into the paused workflow as a normal response, so the case moves to the next stage exactly as it would after a direct approval.
- The audit trail records it as an **override**, with the Broker Support user, the named approver, the time, the comment and the attached files. It's shown distinctly on the dashboard.
- Supporting files are stored against the case (POC: local or blob storage, with metadata in the case tables).

**Success:** Broker Support can push a stalled stage forward with a recorded justification, and an auditor can tell overridden stages apart from directly approved ones.

## 4. Delivery stages

| Stage | Scope | Objectives |
|---|---|---|
| 1 | Workflow + console HITL (including override as a response type) + Postgres checkpoints + versioning experiment | O1, O4, O5, O8 (workflow side) |
| 2 | Dashboard/API + timers and escalation + override UI with file upload | O2, O3, O8 |
| 3 | Email out with a deep link | O6.1 |
| 4 | Email in + AI classification and summaries | O6.2, O7 |

## 5. Constraints and guardrails

- **No real broker data goes to a model** until the data and security teams have cleared it. The POC uses fake data.
- Use a personal or dev model key for the POC, not a work production key.
- The dashboard is the system of record. Parsing email replies is the fragile part.
- Evaluate the framework honestly: if the workflow engine feels heavy for what it does, compare it against Logic Apps or a plain state machine.

## 6. Open questions for the Broker Support meeting

1. What are the stages, and who approves each one? Are there delegates?
2. Which stages are conditional, for example by broker type or product?
3. What do they use today: email, a spreadsheet or a tracker?
4. Where do things stall or bounce back most often?
5. What documents and checks are involved (FCA register, credit checks, others)?
6. Can they walk through **one recent 4-month case** and mark where the time went, separating waiting on a person from waiting on the broker?
7. Overrides: who in Broker Support may override? Are there stages that must never be overridden (for example compliance or risk sign-off)? Should the approver be notified, or asked to confirm afterwards? Can Broker Support override a rejection, or only a pending stage?

## 7. Technology

- Python + `agent-framework` (versions pinned in the repo `pyproject.toml`), managed with `uv`.
- Postgres for checkpoints and the case and audit tables (already in the dev container).
- Later: Microsoft Graph for email, and Azure OpenAI for the AI steps.
- Reference examples in this repo: `examples/workflow_hitl_checkpoint.py`, `examples/workflow_hitl_checkpoint_pg.py` and `examples/workflow_hitl_requests.py`.

## 8. Path to production

The POC is built in Python so it can be run locally and changed quickly. These points should shape POC decisions now, even though production work comes later.

### 8.1 Language: Python POC, possibly .NET in production
- Agent Framework has the same building blocks in Python and .NET: workflows, executors, edges, human-input requests and checkpoints. The names differ (`ctx.request_info` / `@response_handler` in Python, `RequestPort` in .NET).
- There's no shared code, so moving to .NET means **rewriting the workflow layer**, not converting it. Check that every framework feature the POC relies on exists in the .NET SDK, because the two SDKs don't always ship features at the same time.
- **Make the design easy to port:**
  - Keep the workflow thin. Case state, audit, overrides and files live in our own tables (O5), which don't depend on the language.
  - The dashboard talks only to a plain REST API, so the backend language can change without touching the screens.
  - Keep business rules (stages, approvers, SLAs) in configuration or data, not scattered through workflow code.
- Checkpoints from Python **can't** be resumed in .NET. Cases still in progress at switchover finish on the Python version, or restart from the state in our tables.
- Decision point: once the stage-1 design has settled, choose between staying in Python and rewriting in .NET to sit beside the portal.

### 8.2 Hosting: local vs Foundry / Azure
This is a long-running backend process that waits weeks or months for people, not a chat agent answering a single request. Likely split: **Foundry for the AI parts** (models, possibly the classification and summary agents), and the **workflow, API and timers in our own Azure-hosted service**.

| Concern | Local POC | Production |
|---|---|---|
| Models | API key in `.env` | Foundry project endpoint, Entra ID / managed identity, no keys |
| Workflow engine and API | `uv run` locally | App Service, Container Apps or Functions. Assess Foundry hosted agents and durable hosting for Agent Framework against a workflow that waits for months |
| Checkpoints and case data | Postgres in the dev container | Managed Azure SQL or Postgres, with backups and retention |
| Timers and escalation | In-process, using minutes | Must survive restarts and scale-out, for example a scheduler or durable timers, never in-memory sleeps |
| Inbound email | Script polling Graph | Graph change notifications (webhooks) plus a queue |
| Override supporting files | Local folder | Blob storage with access control and retention |
| Observability | Console / Aspire | Application Insights over OpenTelemetry (see `examples/agent_otel_appinsights.py`) |
| Identity and access | None or fake users | Entra ID sign-in; roles for approver, Broker Support (override) and read-only |
| Security and data | Fake data only | Private networking, data residency, approval from the security team before real broker data reaches a model |

### 8.3 POC habits that keep production cheap
- Read all endpoints, connection strings and keys from configuration. Never hard-code them.
- Abstract storage for checkpoints, files and the clock (for timers) behind small interfaces, so local versions can be swapped for Azure ones.
- Stamp every case with `workflow_version` from day one (O5).
- Write audit events from the start (O4). Don't add them later.
- Use fake data only.

### 8.4 To verify before committing
Foundry hosting options and Agent Framework hosting have been changing quickly. Before choosing a production host, check the current Microsoft Learn docs and the Agent Framework changelog for:
- Foundry hosted agents: whether they suit long-running, human-in-the-loop workflows.
- Durable hosting options for Agent Framework workflows (for example the Azure Functions / Durable Task integrations).
- How the .NET SDK compares with Python for the workflow features we use.
