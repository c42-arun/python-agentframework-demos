# Broker Onboarding Workflow – POC Objectives

Status: draft. First written before discovery with Broker Support, then updated on 2026-11-24 with the business process in [BusinessWorkflow.md](BusinessWorkflow.md). The open questions in §6 still need answers.

Related: [ARCHITECTURE.md](ARCHITECTURE.md) has the system design, diagrams and decision log. [BusinessWorkflow.md](BusinessWorkflow.md) has the current business process as a sequence diagram.

## 1. Background

Broker Support says onboarding a new broker means getting approvals from several people. The work happens mostly over email, and some onboardings take **up to 4 months**.

The idea is to model onboarding as a **human-in-the-loop workflow** built with the **Microsoft Agent Framework** (Python, `agent-framework`), shown on a page in the existing internal support portal.

### 1.1 The current process (from BusinessWorkflow.md)

**Broker Support is the hub.** Every handoff goes through them. The steps run in order:

| # | Step | Who acts | Who we're waiting on |
|---|---|---|---|
| 1 | Gather Companies House data, accounts and the broker's website, and send them to Broker Managers | Broker Support | Broker Support |
| 2 | First approval | Broker Managers | Internal approver |
| 3 | Preliminary sign-off: "any concerns?" | AFG Execs | Internal approver |
| 4 | Application pack: Broker Support asks a Broker Manager to send the pack to the broker. The broker returns the introducer pack, IDs, CV and business accounts to the Broker Manager, who passes them back. | Broker Manager ↔ Broker | **Broker** (through a Broker Manager) |
| 5 | Run credit searches | Broker Support | Broker Support |
| 6 | Fraud check on the application pack and credit search results | AML | Internal team |
| 7 | Final approval: the full documentation goes to every exec in parallel. Each one approves or declines. **2 of N approvals** are needed. | AFG Execs | Internal approvers |
| 8 | Trading agreement sent to the broker for signing. The broker returns a scanned signed copy by email. | Broker | **Broker** |
| 9 | Set the broker up in LeaseSoft/DataHub | Broker Support | Broker Support |
| – | The physical signed copy arrives later by post. It doesn't block set-up. | Broker | Broker |

What this changes compared with the original assumptions:
- Not every step is an approval. There are Broker Support **tasks** (1, 5, 9), a **consultation** (3), **requests** to outside parties (4, 6, 8) and **approvals** (2, 7).
- One step needs a **quorum** (2 of N execs in parallel), not a single approver.
- Execs can **decline**, which ends the onboarding. The original design only had "request changes".
- The broker is contacted through Broker Managers for the application pack, but directly by Broker Support for the trading agreement.
- The process involves outside systems that the POC treats as manual steps: Companies House, credit search, and LeaseSoft/DataHub.

## 2. Problem statement (hypotheses to validate)

We still don't know where the 4 months go. The process in §1.1 now shows which steps depend on whom:

| Delays a workflow can fix | Delays it can't fix |
|---|---|
| Nobody knows whose turn it is, so items sit in inboxes. Broker Support is the hub, so every handoff is a chance for this. | The broker is slow to supply the application pack (step 4) or return the signed agreement (step 8) |
| There are no chasers, SLAs or escalations | External checks are slow (credit searches, AML fraud checks) |
| Gathering exec responses: chasing N execs until 2 approve | An approver is deliberately slow, or the policy is contested |
| Back-and-forth gets lost in email threads, especially the Broker Manager ↔ broker relay | |
| Nobody can see the status, so people keep asking "where is this at?" | |

**Objective 0:** Find out which column most of the delay falls into before we commit to a design. Every step now has a **waiting on** category (Broker Support, internal approver, internal team, broker), so elapsed time can be totalled for each category. Visibility alone could be the headline benefit.

## 3. POC objectives

### O1 – Prove the workflow engine for long-running human steps (no AI)
Build the workflow with plain-code executors only, so the engine is tested without model behaviour muddying the results.

- Model the **real steps from §1.1** using fake data, not made-up stages.
- Support the four **step kinds**:
  - **task**: Broker Support does it and marks it done, attaching evidence.
  - **approval**: one or more approvers, with a quorum (default 1).
  - **consultation**: "any concerns?"
  - **request**: wait for documents or results from the broker or AML.
- Each step pauses for a human response (`ctx.request_info` / `@response_handler`).
- Approvers can **Approve**, **Request changes**, **Decline** and **Comment**. Broker Support can **Override approval** (see O8).
- **Quorum:** the final exec approval completes when 2 of N execs approve. Individual votes are recorded, and the workflow resumes only once the quorum outcome is known.
- **Decline** ends the case as `Declined`.
- **Pushback loop:** a change request goes to a *Revise* step, then back to the same approver, so the flow isn't a straight line.
- **Durable state:** checkpoints are stored in a real database (Postgres), so a case survives process restarts and can resume later.
- Each broker onboarding is one workflow instance, identified by a **case ID**.

**Success:** we can run a case through all nine steps, including the 2-of-N vote. We can pause it, kill the process, resume it, push back and resubmit, decline a case, and finish one.

### O2 – Dashboard as the system of record
- A page (in the support portal or a standalone POC page) lists pending items per person, and lists what Broker Support is waiting on from the broker or AML.
- The Approve, Decline, Request changes and Comment buttons call an API, which sends the response into the paused workflow.
- Each case shows its status, the **elapsed time per step** and **who we're waiting on**. For the exec vote it shows the votes so far ("1 of 2 approvals; waiting on Exec B, Exec C").
- The physical agreement copy is shown as an **open follow-up** on completed cases.
- **Documents:** users can upload documents against a case or a specific stage, and view or download them from the dashboard. Examples are the Companies House extract, the application pack, credit search reports, AML results, the signed agreement and override evidence. Files are stored through the **existing documents service** (Azure Blob), which only needs a parent identifier and a document name. Our case tables record which case, stage and decision each document belongs to.

**Success:** anyone can act on a case without the console, and anyone can see where a case is, who it's waiting on and how long it's been there. Every document for a case can be found, viewed and downloaded from the case page, grouped by stage.

### O3 – Timers, chasers and escalation
- Per-step SLA timers. For example, a nudge at *N* days and an escalation at *M* days.
- Who gets chased depends on who we're waiting on:
  - Exec vote: only the execs who haven't voted.
  - Application pack: the Broker Manager, who relays to the broker.
  - Trading agreement: the broker directly.
- The POC uses **minutes instead of days**, so pause, chase, pushback and resume can all be demoed in one sitting.

**Success:** an overdue step triggers a reminder, then an escalation, to the right person, without anyone stepping in.

### O4 – Audit trail
- Our own event and audit tables record who decided what, when, through which channel, and with what comment. These are kept separately from the framework's checkpoints.
- Every exec vote is audited individually, including votes that arrive after the quorum is met.
- Document uploads and downloads are audited (who, which document, when), because application packs contain personal data.
- Progress milestones within a step are audited, for example "pack sent to broker" and "pack received from broker". This gives Objective 0 its data.
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
Email and the dashboard are two channels into the same workflow response. Today, execs and AML work by email, so this matters for adoption.

1. **Email out:** notifications include a deep link to the dashboard item (via a shared mailbox or Microsoft Graph). This is low risk.
2. **Email in:** watch an inbox through Graph, match each reply to its case using the case ID in the subject, and post the response into the workflow. This includes exec votes and the broker's scanned signed agreement.

**Success:** an approver can act from an email link, and later by replying to the email.

### O7 – Where AI adds value (later stage)
Routing is deterministic, so it stays as plain code. AI is added as extra nodes in the same graph. The process in §1.1 suggests these uses:

- **Due diligence summary (step 1):** summarise the Companies House data, the accounts and the broker's website for the Broker Managers.
- **Application pack completeness (step 4):** check that the introducer pack, IDs, CV and business accounts are all present and readable, and flag gaps before the pack goes to AML.
- **Exec pack (step 7):** turn the "full documentation so far" into one summary for each exec, with the AML and credit findings highlighted.
- Classify inbound email replies as approve, decline, request changes, "no concerns" or query, and pull out the comments.
- Draft chasers and "please clarify" messages.

Guardrail: at first the agent only **suggests** a decision and a human confirms it. It never auto-approves.

**Success:** the AI steps measurably cut approver effort. They are also what justifies this framework over Logic Apps, Power Automate or a plain state machine.

### O8 – Broker Support approval override
Approvers sometimes give approval verbally, for example on a call or in a meeting. Broker Support must be able to record that approval on the approver's behalf so the case doesn't stall.

- Override applies to the Broker Manager approval, the exec preliminary sign-off, and **an individual exec's vote** in the 2-of-N approval.
- The override must name the approver who gave the verbal approval.
- Justification is required: the user must enter a **comment** and/or attach **supporting files** (for example a call note, a meeting minute or a forwarded message). An override with neither is rejected.
- The override goes into the workflow (or the quorum count) as a normal response, so the case moves on exactly as it would after a direct approval.
- The audit trail records it as an **override**, with the Broker Support user, the named approver, the time, the comment and the attached files. It's shown distinctly on the dashboard.
- Supporting files are uploaded through the same document upload as every other case document (O2). They are stored against the case and linked to the override decision.

**Success:** Broker Support can push a stalled step forward with a recorded justification, and an auditor can tell overridden steps or votes apart from direct ones.

### O9 – System touchpoints (later, not committed)
Today these steps are manual. The POC records them as Broker Support tasks with evidence attached:
- **Companies House** (step 1): the public API could pre-fill company data.
- **Credit searches** (step 5): the provider is to be confirmed. Results are attached to the case.
- **LeaseSoft/DataHub** (step 9): set-up could be triggered from the case once it completes.

**Success (if pursued):** at least one manual lookup or re-keying step is removed, without changing the workflow.

### O10 – Status chat in the dashboard (later; groundwork in stage 1)
A chat panel where users ask status questions in plain English. For example:
- "Where are we with Prism Brokers?"
- "Has Acme Brokerage sent us back the application pack yet?"
- "Which cases have been waiting on the broker for more than two weeks?"

- **Read-only.** The chat answers questions. It never approves, overrides or changes anything. "Approve it for me" gets a link to the dashboard.
- **Grounded.** Answers come only from the same read-only case queries the dashboard uses, never from text-to-SQL or workflow checkpoints. Every answer names the case ref, links to the case page and says what time its data is from.
- **Permission-scoped.** The chat runs as the asking user and sees only the cases they can see.
- **Metadata only.** A model sees stage, status, dates, who we're waiting on, and document names and types. It never sees document contents, ID or CV data, or (by default) free-text comments. This is enforced in the query layer, not in the prompt.
- Questions and the tools called to answer them are logged, recording who asked about which case.

**Groundwork in stage 1** (cheap now, and useful without AI):
- a shared read-only query layer
- searchable broker identity (legal name, trading names, Companies House number, fuzzy match)
- a plain-English status summary built in code (`describe_case`)
- a fixed list of audit event types

**Success:** users get correct answers to common status questions, with a link to the case page that confirms them. Testing uses a fixed set of sample questions with expected answers.

This is the natural **first AI feature**: it's read-only, uses metadata only, and every answer can be checked against the dashboard. It still needs data and security clearance before real case data reaches a model (§5).

## 4. Delivery stages

| Stage | Scope | Objectives |
|---|---|---|
| 1 | Workflow with the real steps and step kinds, 2-of-N quorum, decline, console HITL (including override as a response type), Postgres checkpoints, versioning experiment. Status chat groundwork: read-only query layer, broker search, `describe_case`, event-type list. | O1, O4, O5, O8 (workflow side), O10 (groundwork) |
| 2 | Dashboard/API (including "waiting on" and elapsed-time views, and document upload, view and download through the documents service) + timers and escalation + override UI | O2, O3, O8 |
| 2b | Status chat panel on the dashboard (read-only, metadata only). Needs data/security clearance first. | O10 |
| 3 | Email out with a deep link | O6.1 |
| 4 | Email in + AI classification, summaries and completeness checks | O6.2, O7 |
| – | System touchpoints, if pursued | O9 |

## 5. Constraints and guardrails

- **No real broker data goes to a model** until the data and security teams have cleared it. The POC uses fake data.
- The application pack contains **personal data** (IDs, CVs). It never goes into the POC, not even as test files. Use obviously fake documents only.
- The status chat (O10) gets **read-only** tools only. The fields a model may see are set in the query layer, not in prompts.
- Use a personal or dev model key for the POC, not a work production key.
- The dashboard is the system of record. Parsing email replies is the fragile part.
- Evaluate the framework honestly: if the workflow engine feels heavy for what it does, compare it against Logic Apps or a plain state machine.

## 6. Open questions for Broker Support

Answered by BusinessWorkflow.md: what the stages are and who acts on each (Q1, partly), and which documents and checks are involved (Q5, partly). The rest are still open.

**Original questions**
1. ~~What are the stages, and who approves each one?~~ Answered in §1.1. Still open: are there delegates? Which Broker Manager approves: the one assigned to the broker, or any of them?
2. Which steps are conditional, for example by broker type or product? Does every broker go through all nine steps?
3. What do they use today: email, a spreadsheet or a tracker?
4. Where do things stall or bounce back most often?
5. ~~What documents and checks are involved?~~ Answered: Companies House, accounts, the broker's website, the introducer pack, IDs, CV, business accounts, credit searches and the AML fraud check. Still open: is the FCA register checked, and if so, where?
6. Can they walk through **one recent 4-month case** and mark where the time went, against the nine steps in §1.1?
7. Overrides: who in Broker Support may override? Are there steps that must never be overridden (for example the AML check)? Should the approver be notified, or asked to confirm afterwards? Can Broker Support override a decline, or only a pending step? Does an overridden exec vote count towards the 2-of-N quorum?

**New, from the business process**

8. **Preliminary sign-off (step 3):** does each exec reply explicitly, or does silence by a deadline count as "no concerns"? Does it go to all execs or one? What happens if someone raises a concern?
9. **Exec quorum (step 7):** how many execs are there (N), and are they named people or a pool? Does a single decline veto the broker, or is it only declined once 2 approvals become impossible? Once 2 have approved, are the remaining execs told they don't need to respond?
10. **Can the Broker Manager decline** at step 2, or only approve or ask for more?
11. **AML (step 6):** is the fraud check a pass/fail decision, or results passed on for the execs to weigh? Can AML stop a case on its own?
12. **Credit searches (step 5):** which provider? Is there a pass/fail threshold, or are the results just passed on?
13. **Application pack (step 4):** why does it go through a Broker Manager, while the trading agreement goes directly to the broker? What happens when the pack comes back incomplete?
14. **Trading agreement (step 8):** is it a standard document or negotiated? Can set-up start on the scanned copy alone, as the diagram suggests? Is the physical copy ever chased if it doesn't arrive?
15. Do any steps overlap in practice, for example credit searches starting while waiting for the pack?

**Existing documents service** (for its owners rather than Broker Support)

16. ~~What happens when the same name is uploaded twice?~~ Answered: the service assigns a unique GUID per upload, so duplicates are safe. To confirm: is the GUID returned to the caller on upload, and do downloads use it?
17. How do downloads work: does the service stream the file, or hand out a short-lived SAS URL? How does it authenticate callers (managed identity, API key, user token)?
18. Are there size or type limits, virus scanning, or support for metadata (stage, document type) and listing by parent? Can documents be deleted, and is there a retention policy?
19. Is there a dev or test instance the POC can use, and is it cleared for fake test documents?

**Status chat** (for data and security teams)

20. Which case fields may be sent to a model for the status chat (O10)? The proposed allow-list is stage, status, dates, who we're waiting on, document names and types, and broker and approver names. Proposed exclusions are free-text comments and all document contents.
21. How long must chat questions and answers be kept, and who may read them?

## 7. Technology

- Python + `agent-framework` (versions pinned in the repo `pyproject.toml`), managed with `uv`.
- Postgres for checkpoints and the case and audit tables (already in the dev container), with `pg_trgm` for fuzzy broker-name search.
- The existing internal **documents service** (Azure Blob) for case documents, from stage 2. Stage 1 uses a local stand-in with the same contract.
- Later: Microsoft Graph for email, and Azure OpenAI for the AI steps and the status chat (an Agent Framework agent with `@tool` functions over the read-only query layer).
- Later and not committed (O9): the Companies House API, a credit search provider, and LeaseSoft/DataHub.
- Reference examples in this repo: `examples/workflow_hitl_checkpoint.py`, `examples/workflow_hitl_checkpoint_pg.py` and `examples/workflow_hitl_requests.py`.

## 8. Path to production

The POC is built in Python so it can be run locally and changed quickly. These points should shape POC decisions now, even though production work comes later.

### 8.1 Language: Python POC, possibly .NET in production
- Agent Framework has the same building blocks in Python and .NET: workflows, executors, edges, human-input requests and checkpoints. The names differ (`ctx.request_info` / `@response_handler` in Python, `RequestPort` in .NET).
- There's no shared code, so moving to .NET means **rewriting the workflow layer**, not converting it. Check that every framework feature the POC relies on exists in the .NET SDK, because the two SDKs don't always ship features at the same time.
- **Make the design easy to port:**
  - Keep the workflow thin. Case state, audit, overrides, votes and files live in our own tables (O5), which don't depend on the language.
  - The dashboard talks only to a plain REST API, so the backend language can change without touching the screens.
  - Keep business rules (steps, step kinds, approvers, quorum, SLAs) in configuration or data, not scattered through workflow code.
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
| Case documents and override files | Local stand-in for the documents service (stage 1), then its dev instance (stage 2). Fake documents only. | The existing documents service (Azure Blob). Our API checks case access before every download. Application packs contain IDs and CVs, so retention and access rules need sign-off. |
| System touchpoints (O9) | Manual tasks with evidence | Companies House API, credit provider and LeaseSoft/DataHub integrations, if pursued |
| Observability | Console / Aspire | Application Insights over OpenTelemetry (see `examples/agent_otel_appinsights.py`) |
| Identity and access | None or fake users | Entra ID sign-in; roles for Broker Support (including override), Broker Manager, AFG Exec, AML and read-only |
| Security and data | Fake data only | Private networking, data residency, approval from the security team before real broker data reaches a model |

### 8.3 POC habits that keep production cheap
- Read all endpoints, connection strings and keys from configuration. Never hard-code them.
- Abstract storage for checkpoints, documents (local stand-in vs the documents service) and the clock (for timers) behind small interfaces, so local versions can be swapped for Azure ones.
- Stamp every case with `workflow_version` from day one (O5).
- Write audit events from the start (O4). Don't add them later.
- Use fake data only.

### 8.4 To verify before committing
Foundry hosting options and Agent Framework hosting have been changing quickly. Before choosing a production host, check the current Microsoft Learn docs and the Agent Framework changelog for:
- Foundry hosted agents: whether they suit long-running, human-in-the-loop workflows.
- Durable hosting options for Agent Framework workflows (for example the Azure Functions / Durable Task integrations).
- How the .NET SDK compares with Python for the workflow features we use.
