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

Think of Kubernaut like a hospital emergency room:

| Kubernaut Component | Hospital Analogy | What It Actually Does |
|---------------------|------------------|----------------------|
| **Gateway** | Front desk / Triage | Receives alerts, removes duplicates, creates a "case file" |
| **Signal Processing** | Lab work | Gathers context -- what namespace, what deployment, how critical |
| **Remediation Orchestrator** | Attending physician | Coordinates the whole process, decides what happens next |
| **AI Analysis (Kubernaut Agent)** | Diagnostic specialist | Investigates the problem -- reads logs, checks metrics, identifies root cause |
| **Workflow Execution** | Surgical team | Executes the approved fix (rollback, restart, scale up) |
| **Effectiveness Monitor** | Post-op check | Verifies the patient (service) is actually healthy after treatment |
| **Notification Controller** | Paging system | Sends approval requests and status updates to the team |
| **Data Storage** | Medical records | Keeps immutable records of everything that happened (audit trail) |

### How They Work Together (The Flow)

```
Alert fires (Prometheus/OpenShift Events)
    |
    v
Gateway receives it, checks "are we already working on this?"
    |
    v
Signal Processing enriches: "This is production, P0 priority, Deployment/payment-api"
    |
    v
Remediation Orchestrator creates an AI Analysis request
    |
    v
Kubernaut Agent (AI) investigates: reads logs, checks metrics, identifies root cause
    |
    v
AI selects a workflow: "rollback to revision 3" (confidence: 87%)
    |
    v
[If production] --> Slack notification: "Approve this rollback?" --> Operator approves
    |
    v
Workflow Execution runs the rollback via Tekton
    |
    v
Effectiveness Monitor checks: pods healthy? Alert gone? Metrics normal?
    |
    v
DONE - full audit trail recorded
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
