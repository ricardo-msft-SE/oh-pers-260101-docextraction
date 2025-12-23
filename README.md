# OPERS Deterministic Agent Architecture (Single-Agent + Logic Apps)

This repository documents an architectural approach for **compliance- and correctness-critical** workflows (e.g., pension processing) where stakeholders require **deterministic outcomes** and want to minimize error amplification that can occur in loosely-coupled multi-agent chains.

The approach intentionally:

- Keeps **LLM usage narrow and controllable** (interpretation, extraction, classification, explanation)
- Uses **Azure Logic Apps** as the **deterministic orchestration backbone** (branching, retries, approvals, durable execution)
- Treats external actions (line-of-business updates, ticket creation, notifications) as **explicit, auditable workflow steps**

> Context: In the OPERS scoping discussion, the team highlighted a need for deterministic outcomes and raised concerns about multi-agent error propagation. The conversation also explicitly considered a solution that “might not be a multi-agent solution” and could rely more on Logic Apps for reliability.

---

## 1. When to use this pattern

Use **Single Agent + Logic Apps** when most of the process can be expressed as a **known set of steps** but you still need AI for:

- Extracting signals from **unstructured inputs** (documents, free-text notes, emails)
- Mapping ambiguous language to **known intents** or **policy categories**
- Producing **human-readable** explanations or draft summaries
- Assisting with **triage** (e.g., “which queue does this document go to?”)

Avoid pushing AI into the *core* of the decisioning when:

- Incorrect routing has **material impact** (financial, legal, customer harm)
- You must meet strict auditability requirements
- You need repeatable outcomes under the same inputs

---

## 2. High-level architecture

### Key idea

**One agent** acts as the “reasoning/interpretation front-end,” while **Logic Apps** provides a deterministic, testable, inspectable workflow engine.

- The agent can **call Logic Apps** as an action/tool.
- Logic Apps can call:
  - on-prem APIs (via gateway)
  - Azure Functions
  - enterprise systems (Dataverse/Dynamics/ServiceNow/etc.)
  - human approvals

### Diagram

![Deterministic Single-Agent + Logic Apps Architecture](arch.jpg)

---

## 3. Component responsibilities

### 3.1 Single Foundry Agent (narrow scope)

**Primary responsibilities**

- Normalize incoming inputs (document + metadata)
- Perform extraction/classification
- Decide *which* deterministic workflow to invoke (not *how* the workflow runs)
- Produce explanations for humans (optional)

**What it should NOT do**

- Execute critical business decisions without a deterministic guardrail
- Perform long, multi-step orchestration internally if those steps can be made deterministic

### 3.2 Azure Logic Apps (deterministic orchestration)

**Primary responsibilities**

- Implement the process as explicit steps:
  - validations
  - branching logic
  - retries + error handling
  - idempotency / deduping
  - approvals
  - audit trails
- Perform integrations to enterprise/on-prem systems

### 3.3 Shared services

- **Knowledge sources** (Azure AI Search / SharePoint / Fabric) for grounded retrieval (optional)
- **Observability** (Application Insights / Azure Monitor) for traces, logs, metrics
- **Policy & Safety** (content filters, guardrails) where appropriate

---

## 4. Example flow (OPERS-style “Document Actions”)

1. **Ingestion**: A document arrives (upload, email, scan queue)
2. **Agent extraction**: Agent extracts key fields (customer ID, doc type, dates)
3. **Workflow dispatch**: Agent calls Logic App `StartDocumentActionWorkflow` with structured payload
4. **Deterministic enrichment**: Logic App calls LOB API(s) to fetch customer/account details
5. **Rules & routing**: Logic App evaluates static rules (policy matrix / decision table)
6. **Human-in-the-loop (optional)**: If confidence below threshold or exceptions detected, request approval
7. **Execution**: Logic App performs the final action (route doc, create case, update system)
8. **Audit & notify**: Emit logs/traces and notify stakeholders

---

## 5. Determinism strategy

To make results as deterministic as possible:

- **Constrain agent scope** to interpretation and classification
- Use **explicit schemas** for all agent outputs (JSON) and validate them in Logic Apps
- Prefer **decision tables / rules engines** for final routing
- Add **confidence + fallback paths**:
  - If extraction fails → escalate
  - If ambiguity is high → human review
- Make workflows **idempotent** (safe to retry)

---

## 6. Why not multi-agent (at first)?

Multi-agent systems can be powerful, but in high-stakes domains they introduce extra failure modes:

- Cross-agent handoffs can **amplify** earlier mistakes
- Debugging “why did agent B do that?” becomes harder
- Determinism decreases when orchestration relies on probabilistic routing

This pattern starts with a **single agent + deterministic orchestration**, then graduates to multi-agent only if:

- You have clear modular sub-tasks that require separate specialists
- You can fully instrument, evaluate, and govern handoffs
- The latency and complexity trade-offs are acceptable

---

## 7. Implementation notes

### 7.1 Tooling integration

Azure AI Foundry supports adding **Logic Apps as tools/actions** for an agent. The Logic App should:

- Use an **HTTP Request trigger** with a documented JSON schema
- End with an HTTP response
- Live in the **same subscription + resource group** as the Foundry project if you want it discoverable in the Foundry portal

### 7.2 Payload contract (example)

```json
{
  "correlationId": "<guid>",
  "document": {
    "uri": "https://...",
    "type": "pension_form",
    "extracted": {
      "customerId": "12345",
      "receivedDate": "2025-12-23",
      "keyDates": ["2026-01-01"],
      "confidence": 0.91
    }
  },
  "requestedAction": "route_and_create_case",
  "agent": {
    "model": "<deployment>",
    "version": "1.0"
  }
}
```

### 7.3 Reliability checklist

- [ ] Schema validation in Logic App
- [ ] Durable retries with backoff
- [ ] Idempotency keys (`correlationId`)
- [ ] Human approval branch
- [ ] Audit log written for every branch
- [ ] Alerts on failure/latency

---

## 8. Repo layout (suggested)

```
.
├─ README.md
├─ docs/
│  ├─ architecture.png
│  └─ architecture.drawio (optional)
└─ logicapps/
   ├─ StartDocumentActionWorkflow.json
   └─ ...
```

---

## 9. Next steps

1. Define the **decision table** (rules) that drives routing
2. Decide thresholds for **human-in-the-loop**
3. Create the Logic App(s) and publish as agent tools
4. Add observability (traces + correlation IDs)
5. Build evaluation sets for extraction/classification quality

