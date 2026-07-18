# Presenter Preparation Guide

**Time to read: 15 minutes**
**Presentation duration: 30 minutes**
**Audience: Mixed ops team + engineering leadership**

---

## Before You Start: What You Need to Know

You don't need to be a Kubernetes expert to deliver this presentation. You need to understand:
1. What problem Kubernaut solves (alert fatigue, slow remediation)
2. How it works at a high level (detect, investigate, fix, verify)
3. Why it's safe (approval gates, audit trail, scope limits)
4. What we're proposing (a low-risk pilot)

This guide gives you that knowledge in plain language.

---

## The Basics: Concepts You'll Reference

### What is OpenShift?
Red Hat's enterprise Kubernetes platform. Think of it as a managed container orchestration system that runs our applications. When something goes wrong (a service crashes, runs out of memory, or misbehaves), operators get alerted and have to fix it manually.

### What is a "pod"?
The smallest deployable unit in Kubernetes/OpenShift. It's a running instance of your application. When people say "the pod crashed," they mean one instance of an app stopped working.

### What is "CrashLoopBackOff"?
A common error where a pod keeps crashing, restarting, crashing again, in a loop. It's one of the most frequent alerts operators deal with.

### What is MTTR?
Mean Time to Remediate -- how long it takes from "alert fires" to "problem is fixed." This is the key metric we're trying to improve.

### What is a CRD?
Custom Resource Definition -- a way to extend Kubernetes with new object types. Kubernaut uses CRDs to track remediation progress. The key point for the audience: "It's native to OpenShift, not a bolted-on tool."

### What is Tekton?
A Kubernetes-native CI/CD pipeline tool. Kubernaut uses it to execute remediation workflows (the actual fix steps). It's like a recipe executor for infrastructure changes.

### What is RBAC?
Role-Based Access Control -- security rules that control who (or what) can do what in the cluster. Key point: "Kubernaut respects the same security rules as everything else."

---

## The Architecture: What Each Piece Does

### The Operator Layer (Kubernetes-Native Controllers)

Kubernaut is a **Kubernetes operator** -- the same pattern Red Hat uses to build OpenShift itself. It extends OpenShift with Custom Resource Definitions (CRDs) that represent remediation state.

**What "operator" means in practice:**
- It watches for changes to custom resources (like RemediationRequest)
- When a resource changes, the operator "reconciles" it -- makes the actual state match the desired state
- This is the same pattern that makes a Deployment controller restart pods when they crash
- Kubernaut has 5 separate operator controllers, each watching its own CRD

**The 5 CRD Controllers:**

| Controller | CRD it watches | What it does on reconcile |
|------------|---------------|--------------------------|
| Signal Processing | `SignalProcessing` | Enriches alert with K8s context (what namespace, what deployment, owner chain). Runs Rego policies to classify environment (prod/staging) and priority (P0-P3). |
| AI Analysis | `AIAnalysis` | Calls the Kubernaut Agent HTTP API to investigate. Polls for results. Records selected workflow. |
| Workflow Execution | `WorkflowExecution` | Creates a Tekton PipelineRun (or K8s Job or Ansible playbook) from the workflow catalog. Watches for completion/failure. |
| Effectiveness Monitor | `EffectivenessAssessment` | After fix executes, checks: is alert resolved? Are pods healthy? Are Prometheus metrics back to normal? Scores 0-100. |
| Notification | `NotificationRequest` | Sends messages to Slack, PagerDuty, Teams, email. Handles approval request delivery and retry. |

**The Orchestrator (the "conductor"):**
- Watches the main `RemediationRequest` CRD
- Creates child CRDs in sequence: SP -> AIA -> WFE -> EA
- Implements a phase state machine: Pending -> Processing -> Analyzing -> Executing -> Verifying -> Completed
- Handles timeouts (global: 1 hour, per-phase: 5-30 min configurable)
- Handles blocking conditions (resource already being remediated, backoff from failures)
- If any child fails, orchestrator decides: retry? escalate? timeout?

### The Agent Layer (LLM-Powered Investigation)

The Kubernaut Agent is the AI brain. It's a stateless HTTP service that receives investigation requests and runs a multi-turn LLM conversation.

**How the Agent investigates (like your best SRE):**

```
REQUEST ARRIVES: "CrashLoopBackOff on payment-api pod in production"
    |
    v
PRE-ENRICHMENT: Agent gathers context BEFORE calling the LLM
  - Resolves owner chain: Pod -> ReplicaSet -> Deployment
  - Checks for GitOps labels (ArgoCD, Flux)
  - Queries DataStorage for remediation history on this target
  - Detects: HPA present? PDB present? StatefulSet?
    |
    v
PHASE 1 - ROOT CAUSE ANALYSIS (multi-turn LLM loop):
  Turn 1: LLM reads alert context, decides: "I need to check pod logs"
    -> Tool call: kubectl logs payment-api-xyz
    -> Result: "OOMKilled, memory usage spiking at startup"
  Turn 2: LLM decides: "Was there a recent deployment?"
    -> Tool call: kubectl rollout history deployment/payment-api
    -> Result: "Revision 5 deployed 30 min ago by user@company.com"
  Turn 3: LLM decides: "Let me check memory metrics"
    -> Tool call: prometheus query (container_memory_usage)
    -> Result: "Memory 2x higher since revision 5"
  Turn 4: LLM concludes: "OOM caused by memory leak in rev 5"
    -> Submits RCA: root_cause="memory leak in revision 5"
    |
    v
PHASE 2 - WORKFLOW SELECTION:
  Turn 1: LLM searches workflow catalog
    -> Tool call: list_available_actions (filter: Deployment, rollback)
    -> Result: "rollback-deployment-v2 (confidence: 92%)"
  Turn 2: LLM validates parameters
    -> Tool call: get_workflow (rollback-deployment-v2)
    -> Result: schema requires TARGET_NAMESPACE, TARGET_NAME, REVISION
  Turn 3: LLM selects and submits
    -> submit_result_with_workflow: rollback-deployment-v2, params filled
    |
    v
RESULT: workflow_id=rollback-deployment-v2, confidence=0.92, revision=4
```

**Key Agent features to explain:**
- **Tool registry:** The LLM doesn't have free access to the cluster. It can only use registered tools (kubectl get/describe/logs, Prometheus queries, AlertManager silence). Each tool call is logged.
- **Shadow agent:** A SECOND LLM watches every step of the investigation for suspicious behavior (prompt injection defense). If it flags something, the whole investigation stops (fail-closed).
- **MCP Interactive mode:** An operator can "take over" an investigation mid-flight, ask the AI additional questions, steer the investigation, then hand back for workflow selection. Like pair-programming with the AI.
- **Multi-provider LLM:** Supports OpenAI, Anthropic Claude, Google Gemini, or locally-hosted models (Ollama). Air-gap compatible.

### How They Work Together (The Full Flow)

```
Alert fires (Prometheus AlertManager / OpenShift Events)
    |
    v
GATEWAY: Receives webhook, computes fingerprint (SHA256 of owner+namespace+kind)
  - Checks: is there already an active RR for this fingerprint?
  - If yes: increment occurrence counter, return 202 (deduplicated)
  - If no: create RemediationRequest CRD in kubernaut-system namespace
    |
    v
ORCHESTRATOR: Reconciles new RR, checks routing conditions
  - Is this namespace managed? (label check, deny-by-default)
  - Is there a backoff window from previous failures?
  - Is another WFE running on same target? (resource lock)
  - All clear: creates SignalProcessing CRD -> phase = Processing
    |
    v
SIGNAL PROCESSING: Reconciles SP CRD
  - Enriches: fetches namespace labels, deployment spec, owner chain
  - Classifies: runs Rego policy -> environment=Production, priority=P0
  - Categorizes: signal mode (reactive/proactive), normalized severity
  - Completes: updates SP status -> Orchestrator watches, sees completion
    |
    v
ORCHESTRATOR: SP complete, creates AIAnalysis CRD -> phase = Analyzing
    |
    v
AI ANALYSIS CONTROLLER: Reconciles AIA CRD
  - Submits investigation to Kubernaut Agent HTTP API (async: 202 + session_id)
  - Polls for result (15s interval)
  - Agent runs RCA + workflow selection (10-30 seconds)
  - Result arrives: workflow_id, confidence, parameters
  - Updates AIA status with selectedWorkflow
    |
    v
ORCHESTRATOR: AIA complete, checks confidence
  - Confidence >= 80% (configurable): creates WorkflowExecution CRD
  - Confidence < 80%: creates RemediationApprovalRequest + Notification
    -> Slack: "Approve rollback of payment-api? Confidence: 75%"
    -> Operator clicks approve
    -> THEN creates WorkflowExecution CRD -> phase = Executing
    |
    v
WORKFLOW EXECUTION: Reconciles WFE CRD
  - Looks up workflow in DataStorage catalog (OCI bundle reference)
  - Creates Tekton PipelineRun (or K8s Job) with parameters
  - Watches PipelineRun status until completion/failure
  - Records execution time, exit codes, output
    |
    v
ORCHESTRATOR: WFE complete -> phase = Verifying
  - Creates EffectivenessAssessment CRD
    |
    v
EFFECTIVENESS MONITOR: Reconciles EA CRD
  - Waits stabilization window (30s configurable)
  - Checks: alert still firing? -> AlertManager query
  - Checks: pods healthy? -> K8s pod status
  - Checks: metrics improved? -> Prometheus pre/post comparison
  - Checks: spec hash changed? -> drift detection
  - Produces effectiveness score (0-100)
    |
    v
ORCHESTRATOR: EA terminal -> phase = Completed
  - Creates completion Notification CRD
  - Records final audit events
  - Sets retention expiry (24h default, then CRD garbage collected)
```

---

## Key Talking Points Cheat Sheet

### When Someone Asks "How is this different from X?"

| They mention... | Your answer |
|-----------------|-------------|
| **PagerDuty** | "PagerDuty alerts humans. Kubernaut investigates AND fixes. They complement each other -- PagerDuty fires the alert, Kubernaut handles it." |
| **Rundeck / Ansible Tower** | "Those execute pre-defined scripts. Kubernaut investigates first to determine WHICH script to run. It's the difference between a button and a decision-maker." |
| **Dynatrace / DataDog auto-remediation** | "Those are tied to their monitoring platform. Kubernaut is open -- works with any alert source and any execution backend." |
| **Just writing better runbooks** | "Runbooks require a human to follow them at 3am. Kubernaut IS the runbook -- but it executes itself and verifies the outcome." |

### When Someone Asks About Safety

**Your core message:** "Kubernaut is as conservative or aggressive as you configure it to be. Out of the box, it requires human approval for everything. You gradually allow auto-execution for low-risk actions you trust."

Key safety facts:
- Only operates on namespaces you explicitly label (`kubernaut.ai/managed=true`)
- Approval required for production workloads (configurable threshold)
- After 3 consecutive failures, it stops and pages a human
- Complete audit trail -- every action traceable
- SOC2 and FedRAMP audit control mappings built-in

### When Someone Asks About Cost / Resources

- Runs as 12 lightweight containers (similar to running a monitoring stack)
- PostgreSQL database for audit/state (can share existing cluster)
- LLM is the variable cost:
  - OpenAI/Anthropic: ~$0.01-0.10 per investigation (depends on complexity)
  - Local (Ollama): infrastructure cost only, no per-call fees
  - Estimated: $50-200/month for moderate alert volume (100 alerts/day)

### When Someone Asks "What Happens If the AI Gets It Wrong?"

"Great question. Three layers of protection:
1. For critical workloads, a human must approve before anything executes
2. After execution, it verifies the fix actually worked -- if not, it escalates
3. If the same issue fails 3 times, it permanently blocks and pages a human

The worst case for an auto-approved action is that a low-risk operation (like restarting a non-critical pod) doesn't fix the problem -- at which point Kubernaut escalates. It never silently fails."

---

## New Slides: What You Need to Know

### Slide 4: Kubernaut Agent Deep-Dive (the "wow" slide)

This is the most technically impressive slide. The key concept: Kubernaut's AI is an **autonomous agent**, not just "calling ChatGPT."

**What "agentic" means (explain it like this):**
- A regular AI chatbot answers questions you ask it
- An AI agent DECIDES what questions to ask, then goes and finds the answers itself
- Kubernaut's agent is like an SRE that thinks out loud: "OOMKilled? Let me check logs. Memory spiking? Was there a deploy? Yes, 30 min ago. That's the cause."
- It makes 3-8 tool calls per investigation, each one a deliberate decision

**The ReAct loop (Reason-Act-Observe):**
1. Agent REASONS: "Given this OOMKilled alert, I should check pod logs"
2. Agent ACTS: calls the kubectl tool to get logs
3. Agent OBSERVES: reads the result, identifies memory spike at startup
4. Agent REASONS again: "Memory spike at startup -- was there a recent deploy?"
5. Repeat until confident enough to submit a recommendation

**Shadow Agent (the safety differentiator):**
- A completely separate AI watches every step of the investigation
- It's looking for: prompt injection, suspicious tool calls, anomalous reasoning
- If it detects something wrong, it kills the entire investigation immediately
- This is "fail-closed" -- errors are treated as suspicious (safe default)

**The line to memorize:** "This is an AGENT, not a prompt. It has agency -- it decides what to investigate, how deep to go, and when it has enough confidence to recommend action."

### Slide 6: Day in the Life (the "make it real" slide)

Three panels showing what operators actually see. Walk through left to right:

**Left - Slack notification:**
- This arrives in your team's channel (or DM to on-call)
- Shows: what's wrong, what the AI found, what it wants to do, confidence score
- One-click Approve or Reject buttons
- Key point: "No investigation needed -- the AI already did it"

**Middle - Terminal (oc get rr --watch):**
- For operators who prefer the CLI
- Phases tick through in real-time: Pending -> Processing -> Analyzing -> Executing -> Verified
- "62 seconds end-to-end" -- that's the number to emphasize

**Right - Metrics graph:**
- Memory climbing (the leak), then blue dashed line (rollback executes), then drops to stable
- Visual proof it worked
- "The Effectiveness Monitor sees this and confirms: problem resolved"

**The punchline:** "62 seconds from alert to verified fix. One Slack button click. Full audit trail. Compare that to the 47-minute manual workflow on slide 1."

---

## Presentation Tips

### Opening (first 60 seconds)
Start with empathy, not technology. "We've all been paged at 3am..." gets immediate buy-in.

### The Architecture Slide
Don't explain every service. Walk through ONE scenario end-to-end. If someone wants detail, press the down arrow for the full architecture grid.

### The Safety Slide
This is where you win over skeptics. Slow down here. Make eye contact. The message is: "Control remains with your team."

### The Close
End with a specific, low-risk ask: "2-3 non-production namespaces, 30 days, then we review data together." This removes the fear of commitment.

### If You Get Stuck on a Technical Question
"That's a great technical question -- let me connect you with the engineering team who can give you the full detail on that. For today, the key point is [restate the business value]."

---

## Using the Speaker Notes

1. Open the presentation in Chrome: https://rrbanda.github.io/knaut/
2. Press `S` on your keyboard -- this opens a separate "Speaker View" window
3. The speaker view shows:
   - Current slide (what the audience sees)
   - Your notes (only you see these)
   - Next slide preview
   - Elapsed time
4. Put the speaker view on your laptop screen, the main presentation on the projector
5. Use arrow keys to advance slides

### Keyboard Shortcuts
- `Right arrow` or `Space`: Next slide
- `Left arrow`: Previous slide
- `Down arrow`: Sub-slide (architecture detail on Slide 3)
- `S`: Open speaker notes
- `F`: Full screen
- `O` or `Esc`: Slide overview (bird's eye view)

---

## Day-Of Checklist

- [ ] Test the presentation URL works: https://rrbanda.github.io/knaut/
- [ ] Open in Chrome (best reveal.js support)
- [ ] Press `S` to confirm speaker notes window opens
- [ ] Test projector/screen sharing shows the main window (not the notes)
- [ ] Have a glass of water ready
- [ ] Arrive 5 minutes early to set up screen sharing

---

## After the Presentation

If they say "let's do it" -- the next step is:
1. Pick 2-3 non-critical namespaces (dev/staging)
2. Identify top 5 most common alerts (CrashLoop, OOM, HPA ceiling, failed deploy, disk pressure)
3. Schedule a 1-hour technical deep-dive with the engineering team
4. Target: Helm install in 2 weeks, 30-day measurement period

If they say "let me think about it" -- send a follow-up email with:
- Link to this presentation
- The proposed pilot scope (2-3 namespaces, zero production risk)
- An offer to do a live demo in a test cluster
