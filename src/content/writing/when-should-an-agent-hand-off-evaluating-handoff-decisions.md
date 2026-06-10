---
title: "When Should an Agent Hand Off? Evaluating Handoff Decisions in Multi-Agent Workflows"
description: "A practical framework for evaluating whether multi-agent systems hand work to the right agent, at the right time, with the right context."
date: 2026-04-12T09:00:00-04:00
draft: false
tags:
  - AI agents
  - orchestration
  - evaluation
  - reliability
---

The hardest part of a multi-agent system is often not the individual agents.

It is the moment between them.

A triage agent decides whether a request belongs to billing, technical support, policy review, or human escalation. A research agent decides whether it has gathered enough evidence to hand work to a writer. A planner decides whether constraints are clear enough to hand a task to an executor. An orchestrator decides whether to retry the same specialist, route to a fallback, or stop the workflow entirely.

Those moments are easy to underestimate because they are not always visible in the final answer. If the final output looks polished, we may not notice that the system handed off too early, sent the task to the wrong specialist, dropped key context, or looped between agents before eventually stumbling into a good response.

This essay is about a narrow slice of orchestration evaluation: **handoff correctness**.

The core question is:

> Did the system hand the task to the right next actor, at the right time, with enough context for that actor to succeed?

That question sounds simple. In practice, it is one of the most important evaluation layers for multi-agent systems.

## The Handoff Is A Product Behavior

In a single-agent workflow, tool calls are often the main units of action. In a multi-agent workflow, handoffs become actions too.

A handoff is not just an implementation detail. It changes who owns the next step of reasoning or execution. It may change the available tools, the system prompt, the risk boundary, the expected output format, and the review criteria.

For example, a support workflow might look like this:

```text
intake_agent
  -> triage_agent
  -> diagnostic_agent
  -> policy_agent
  -> response_agent
  -> human_reviewer
```

Every arrow is a decision. The system is not merely “running agents.” It is deciding how work should move.

If the triage agent sends a policy-sensitive question directly to the response agent, the final answer may still sound confident. But the workflow skipped the actor responsible for policy validation. If the diagnostic agent hands off to human review before collecting logs, the escalation may be formally correct but operationally weak. If the research agent hands off to the writer with half the required evidence, the writer may fill gaps with fluent speculation.

The handoff is part of the product behavior. It deserves evaluation.

## A Handoff Contract

I like to think of every handoff as having a contract.

The contract has four parts:

| Contract Part | Question | Example |
| --- | --- | --- |
| Trigger | Why should this handoff happen now? | confidence below threshold, policy topic detected |
| Recipient | Who should receive the task? | policy agent, diagnostic agent, human reviewer |
| Payload | What context must be transferred? | user request, resolved entities, evidence, constraints |
| Exit Criteria | What must be true before handoff? | diagnostics collected, sources cited, authorization checked |

This framing prevents a common mistake: evaluating a handoff only by recipient. Sending a task to the right agent is necessary, but not sufficient. A handoff can be wrong because it happened too early, because the payload was incomplete, or because the previous agent had not satisfied its exit criteria.

For evaluation, the contract gives us something executable. Instead of vaguely asking whether orchestration was “good,” we can ask:

- Was the trigger condition present?
- Was the recipient appropriate for the task state?
- Did the payload contain required context?
- Were the previous agent’s exit criteria satisfied?

That turns orchestration evaluation into trace evaluation.

## A Concrete Failure: The Premature Writer

Consider a research-and-writing workflow. The user asks:

> “Write a short technical brief comparing two approaches to evaluating tool-calling agents.”

The system uses three agents:

1. `research_agent`
2. `outline_agent`
3. `writer_agent`

An expected trace might be:

```json
[
  {
    "type": "agent_step",
    "agent": "research_agent",
    "event": "source_collected",
    "source_id": "trace_eval_notes"
  },
  {
    "type": "agent_step",
    "agent": "research_agent",
    "event": "source_collected",
    "source_id": "tool_argument_eval_notes"
  },
  {
    "type": "handoff",
    "from": "research_agent",
    "to": "outline_agent",
    "reason": "minimum_sources_collected",
    "payload": {
      "source_count": 2,
      "open_questions": []
    }
  },
  {
    "type": "handoff",
    "from": "outline_agent",
    "to": "writer_agent",
    "reason": "outline_approved"
  }
]
```

Now compare that with this observed trace:

```json
[
  {
    "type": "agent_step",
    "agent": "research_agent",
    "event": "source_collected",
    "source_id": "trace_eval_notes"
  },
  {
    "type": "handoff",
    "from": "research_agent",
    "to": "writer_agent",
    "reason": "enough_context"
  }
]
```

The final brief might still read well. But the orchestration failed in two ways:

- The research agent handed off before collecting the required second source.
- The workflow skipped the outline agent entirely.

This is a handoff correctness failure, not a writing-quality failure. If evaluation only scores the final brief, the system may pass while the orchestration regresses.

## Handoff Failure Taxonomy

Here are the failure modes I would explicitly track.

### Wrong Recipient

The task is handed to the wrong next actor.

Example: a policy-sensitive question goes to `response_agent` instead of `policy_agent`.

This is the most obvious handoff failure, and it is often detectable with routing-label checks. But it is only one category.

### Premature Handoff

The handoff happens before the current agent has completed required work.

Example: `diagnostic_agent` escalates to human review before collecting logs, environment details, or reproduction steps.

Premature handoffs often create downstream hallucination or low-quality work because the next agent receives an under-specified task.

### Delayed Handoff

The current agent keeps working after it should have delegated.

Example: a triage agent continues attempting technical diagnosis instead of routing to a specialist after detecting a system error.

Delayed handoffs can waste latency, increase cost, and make the trace harder to audit. In customer-facing workflows, they can also cause the system to produce weak intermediate answers before using the right specialist.

### Incomplete Payload

The recipient is correct, but the handoff payload is missing critical context.

Example: `policy_agent` receives a request without the resolved customer plan, jurisdiction, or source documents.

This is similar to argument-level tool-call evaluation. The recipient is the right “tool,” but the payload is wrong.

### Context Pollution

The payload contains too much irrelevant or unsafe context.

Example: the writer agent receives raw internal logs that should have been summarized or redacted first.

Handoff evaluation should check not only missing context, but also inappropriate context.

### Retry Loop

The orchestrator repeatedly hands work to the same failing agent instead of switching strategy.

Example: after three failed retrieval attempts, the system keeps routing to `research_agent` instead of invoking fallback search or human review.

Retry-loop failures are orchestration failures because the system is not adapting to observed state.

## A Minimal Handoff Trace Schema

To evaluate handoffs, traces need to record them explicitly.

Here is a minimal event:

```json
{
  "type": "handoff",
  "from": "triage_agent",
  "to": "policy_agent",
  "reason": "policy_sensitive_question_detected",
  "timestamp": "2026-04-12T14:00:03Z",
  "payload": {
    "user_request": "Can this customer use advanced audit logs?",
    "resolved_customer_id": "cust_123",
    "plan": "enterprise",
    "open_questions": [],
    "required_output": "policy-grounded answer"
  }
}
```

For early evaluation, I would capture:

- `from`
- `to`
- `reason`
- `payload`
- `timestamp`
- upstream state summary
- downstream outcome

The upstream state summary is important. A handoff can only be judged relative to what was known at the time.

For example:

```json
{
  "state_before_handoff": {
    "customer_resolved": true,
    "authorization_checked": true,
    "policy_docs_retrieved": false,
    "diagnostics_collected": false
  }
}
```

This makes it possible to evaluate whether the handoff was premature.

## Handoff Rules As Executable Checks

A simple handoff evaluator can start with rule files.

```yaml
case: customer_policy_question

handoff_rules:
  - name: policy_questions_go_to_policy_agent
    when:
      state:
        policy_sensitive: true
    expect:
      to: policy_agent
    severity: high

  - name: policy_agent_requires_customer_context
    when:
      handoff_to: policy_agent
    require_payload:
      - resolved_customer_id
      - plan
      - user_request
    severity: high

  - name: response_after_policy_check
    when:
      handoff_to: response_agent
    require_state:
      policy_check_complete: true
    severity: critical
```

This is intentionally readable. The rule names should explain the contract. When a rule fails, the report should be understandable to someone reviewing the workflow, not only to the person who wrote the evaluator.

The implementation can be simple:

1. Load the trace.
2. Extract handoff events.
3. For each handoff, evaluate rules whose `when` clause applies.
4. Check recipient, payload, state, and timing expectations.
5. Emit a report with severity and trace context.

## Evaluating The Trigger

The most interesting part is the trigger.

Recipient checks are easy:

```text
Expected: policy_agent
Observed: response_agent
```

Payload checks are also fairly concrete:

```text
Missing required payload field: resolved_customer_id
```

Trigger checks are harder. They ask whether the handoff should have happened at all.

For example:

```yaml
trigger_rules:
  - name: escalate_when_confidence_low
    if:
      confidence_below: 0.65
    expect_handoff_to: human_reviewer

  - name: do_not_escalate_when_diagnostics_missing
    forbid_handoff_to: human_reviewer
    unless:
      state:
        diagnostics_collected: true
```

These two rules can conflict if the system has low confidence but has not collected diagnostics. That is not a bug in the evaluator. It reveals a product decision that needs to be made:

> Should low confidence immediately escalate, or should the agent first collect required diagnostic context?

Good evaluation surfaces these decisions. It does not hide them.

## Premature vs Delayed Handoff

Premature and delayed handoffs are opposite problems, and they need different signals.

A premature handoff happens before prerequisites are satisfied:

```text
research_agent -> writer_agent
before minimum_sources_collected
```

A delayed handoff happens after the system has enough evidence that it should delegate:

```text
triage_agent continues diagnosis
after issue_category = billing_dispute
where billing_agent should take over
```

Premature handoff can be caught with exit criteria. Delayed handoff often needs thresholds:

```yaml
delayed_handoff_rules:
  - name: billing_issues_should_route_quickly
    when:
      state:
        issue_category: billing
    expect_handoff_to: billing_agent
    within_steps: 2
    severity: medium
```

The `within_steps` idea is useful. It avoids demanding an immediate handoff while still preventing the agent from wandering too long.

For latency-sensitive workflows, the rule could use time instead:

```yaml
within_seconds: 5
```

For cost-sensitive workflows, it could use model calls:

```yaml
within_model_calls: 1
```

The right metric depends on the product.

## Context Completeness Checks

Handoff payload evaluation should be stricter than a generic “context exists” check.

Different recipients need different payloads.

| Recipient | Required Context |
| --- | --- |
| `policy_agent` | user request, resolved entity, policy domain, approved sources |
| `diagnostic_agent` | error message, environment, reproduction steps, recent changes |
| `writer_agent` | outline, evidence, target audience, constraints |
| `human_reviewer` | summary, risk reason, attempted steps, unresolved questions |

This suggests recipient-specific payload schemas:

```yaml
payload_schemas:
  policy_agent:
    required:
      - user_request
      - resolved_customer_id
      - policy_domain
    forbidden:
      - raw_secret
      - unredacted_log

  human_reviewer:
    required:
      - summary
      - escalation_reason
      - attempted_steps
      - open_questions
```

This is where handoff evaluation connects to both argument evaluation and governance. The payload is an argument to the next agent. If it is incomplete, polluted, or unsafe, the recipient may fail even if it is the right recipient.

## A Better Report For Handoff Failures

For handoff evaluation, I would report failures as workflow transitions.

Example:

```text
Case: customer_policy_question
Failure: premature handoff
Severity: high

Observed transition:
  triage_agent -> response_agent

Reason:
  ready_to_answer

State before handoff:
  customer_resolved: true
  authorization_checked: true
  policy_check_complete: false
  approved_policy_source_retrieved: false

Expected:
  policy_agent should receive policy-sensitive questions before response_agent.

Why this matters:
  The system generated a customer-facing answer before policy validation.
```

This format is more useful than “handoff failed.” It tells the reviewer which transition was wrong, what the system believed at the time, and which contract was violated.

## Handoff Evaluation Should Track Outcomes Too

A handoff can be locally correct and globally ineffective.

For example, the orchestrator may correctly route to `diagnostic_agent`, but the diagnostic agent may return incomplete context. The next handoff to `human_reviewer` may then carry a weak payload.

This means handoff evaluation should not only check the handoff event itself. It should sometimes check the downstream result:

```yaml
handoff_outcome_rules:
  - name: diagnostic_agent_should_collect_minimum_context
    handoff_to: diagnostic_agent
    expect_downstream_state:
      diagnostics_collected: true
      reproduction_steps_present: true
    severity: medium
```

This is useful for measuring whether a specialist agent actually fulfilled its role before the next handoff.

In orchestration evaluation, local decisions and downstream outcomes are connected. A good handoff should improve the state of the workflow.

## Human Escalation Is A Special Handoff

Human escalation deserves special treatment because it is both a safety mechanism and a user-experience decision.

Escalating too early wastes human attention. Escalating too late may cause the system to act beyond its competence. Escalating with poor context makes the human reviewer redo the agent’s work.

A human handoff should usually include:

- concise summary
- reason for escalation
- evidence collected
- actions already attempted
- unresolved questions
- risk level
- recommended next step

Evaluation can check this directly:

```yaml
human_handoff:
  required_payload:
    - summary
    - escalation_reason
    - attempted_steps
    - unresolved_questions
    - risk_level
```

The goal is not just to decide whether a human was needed. It is to decide whether the handoff respected the human’s time.

That is a product-quality issue, not only an evaluation issue.

## What I Would Build First

If I were adding handoff evaluation to an agent orchestration harness, I would build in this order:

1. Add explicit `handoff` events to traces.
2. Normalize `from`, `to`, `reason`, `payload`, and `state_before_handoff`.
3. Add recipient checks for obvious routing errors.
4. Add payload completeness checks for each recipient.
5. Add exit-criteria checks for premature handoffs.
6. Add delayed-handoff checks with step or time thresholds.
7. Add retry-loop detection.
8. Add human-escalation payload checks.
9. Add downstream outcome checks for specialist agents.

This order starts with deterministic checks. Most handoff regressions do not require a sophisticated judge model. They require the system to record transitions and compare them against explicit workflow contracts.

Semantic evaluation can come later for harder questions, such as whether the escalation reason is genuinely adequate or whether the summary is faithful. But the first useful layer is structural.

## The Bigger Point

Multi-agent systems often look sophisticated because many agents are involved. But more agents do not automatically mean better orchestration. They create more boundaries, and every boundary is a place where context can be lost, responsibility can be misrouted, or work can move before it is ready.

Handoff evaluation makes those boundaries visible.

The mature question is not:

> Did every agent produce a good local output?

It is:

> Did the workflow move work between agents at the right moments, with the right context, toward the right outcome?

That is the heart of orchestration evaluation.
