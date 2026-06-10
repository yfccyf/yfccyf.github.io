---
title: "Retry, Fallback, or Escalate? Evaluating Recovery Decisions in AI Agent Orchestration"
description: "A practical framework for evaluating whether an AI agent workflow chooses the right recovery path after a failed step."
date: 2026-04-22T09:00:00-04:00
draft: false
tags:
  - AI agents
  - orchestration
  - evaluation
  - reliability
---

The happy path is not where agent orchestration becomes interesting.

The interesting moment is what happens after something breaks.

A retrieval call returns no useful documents. A tool times out. A downstream API returns `403`. A specialist agent produces malformed JSON. A policy checker says the result is ambiguous. A model emits a tool call that fails schema validation. A browser action lands on an unexpected page. The workflow is no longer following the neat diagram from the design doc, and the orchestrator has to decide what to do next.

Retry? Use a fallback? Ask for clarification? Escalate to a human? Stop and fail closed?

This essay is about that decision layer: **recovery evaluation** in AI agent orchestration.

The core idea:

> Reliable agent orchestration depends not only on successful execution, but on whether the system chooses the right recovery path after failure.

This is a narrow topic, but it shows up everywhere in real agent systems. An agent that works only when every step succeeds is a demo. A production workflow needs to behave well when tools fail, data is missing, confidence is low, or the environment returns something unexpected.

## Failure Is Part Of The Workflow

Many agent diagrams show a clean sequence:

```text
plan -> retrieve -> call tool -> verify -> answer
```

But the real trace often looks more like:

```text
plan
  -> retrieve: no results
  -> retry retrieve with broader query
  -> retrieve: low-confidence result
  -> fallback to policy corpus
  -> policy result: ambiguous
  -> escalate to human reviewer
```

This is not an edge case. It is the normal shape of agent work once the system leaves toy examples.

The orchestration layer owns these decisions. The model may propose the next action, but the harness, graph, router, or controller usually decides whether that action is allowed, whether a retry budget is exhausted, whether a fallback path is available, and whether the workflow should stop.

That means recovery behavior should be evaluated explicitly.

If we only evaluate final answers, we miss whether the system recovered responsibly. If we only evaluate tool-call success, we miss whether the system reacted correctly to failure. If we only evaluate the first attempt, we miss the actual operational behavior of the agent.

## A Recovery Decision Has A Contract

When a step fails, the orchestrator should not improvise blindly. It should follow a recovery contract.

I like to think of that contract as five questions:

| Contract Part | Question | Example |
| --- | --- | --- |
| Error Class | What kind of failure happened? | timeout, empty retrieval, permission denied |
| Retry Eligibility | Is retry allowed? | transient timeout may retry, `403` should not |
| Retry Strategy | If retrying, what changes? | new query, smaller payload, different source |
| Fallback Path | What alternative path is available? | fallback retriever, specialist agent, cached source |
| Stop/Escalation Rule | When should autonomous recovery end? | budget exhausted, risk threshold reached |

This framing is useful because bad recovery behavior often comes from violating one of these parts. The system misclassifies the error, retries something that should not be retried, repeats the same failed input, falls back to a weaker source without saying so, or keeps looping after the budget is exhausted.

The evaluation target is not simply “did the workflow eventually finish?” It is:

> Given the failure observed, did the orchestrator choose an appropriate recovery path?

## A Concrete Example: Empty Retrieval

Suppose a user asks:

> “What is the retention period for enterprise audit logs?”

The agent calls a policy search tool:

```json
{
  "type": "tool_call",
  "name": "search_policy_docs",
  "arguments": {
    "query": "audit logs"
  },
  "result": {
    "status": "empty"
  }
}
```

This is a weak first query. An empty result should not immediately become a final answer. But it also should not trigger five identical retries.

A reasonable recovery trace might be:

```json
[
  {
    "type": "tool_call",
    "name": "search_policy_docs",
    "arguments": {
      "query": "audit logs"
    },
    "result": {
      "status": "empty"
    }
  },
  {
    "type": "recovery_decision",
    "error_class": "empty_retrieval",
    "decision": "retry_with_modified_input",
    "reason": "query_too_broad",
    "budget_remaining": 1
  },
  {
    "type": "tool_call",
    "name": "search_policy_docs",
    "arguments": {
      "query": "enterprise audit log retention period policy"
    },
    "result": {
      "status": "success",
      "document_count": 2
    }
  }
]
```

This recovery path is reasonable because the retry changes the input in a meaningful way. The system does not just repeat the failed action. It uses the failure signal to improve the next attempt.

Now compare it with:

```json
[
  {
    "type": "tool_call",
    "name": "search_policy_docs",
    "arguments": {
      "query": "audit logs"
    },
    "result": {
      "status": "empty"
    }
  },
  {
    "type": "tool_call",
    "name": "search_policy_docs",
    "arguments": {
      "query": "audit logs"
    },
    "result": {
      "status": "empty"
    }
  },
  {
    "type": "final_answer",
    "text": "I could not find a policy, but audit logs are usually retained for 180 days."
  }
]
```

This is a bad recovery path. The retry is identical, the failure is not resolved, and the final answer guesses from insufficient evidence.

The final answer may sound plausible. The recovery trace is the evidence that the system behaved poorly.

## Recovery Failure Types

Before writing evaluators, it helps to name the failures.

### Blind Retry

The system retries the same failed action without changing anything meaningful.

Example: same tool, same query, same arguments, same failing environment.

Blind retry is sometimes acceptable for transient infrastructure failures, but it is usually wrong for semantic failures like empty retrieval, schema mismatch, or ambiguous policy result.

### Retry When Retry Is Not Allowed

Some failures should not be retried.

Example: a `403` permission error. Retrying the same unauthorized request is not recovery; it is noise. The right response may be to stop, explain the permission issue, or escalate.

### No Retry When Retry Is Expected

Some failures should trigger at least one recovery attempt.

Example: retrieval returns no results from a weak query. A single query failure should not immediately become a final answer if a better query or fallback source exists.

### Fallback Too Early

The system switches to a fallback before exhausting the primary safe path.

Example: using a general help-center search before trying the approved policy corpus with a better query.

Fallbacks can be useful, but they may have weaker authority, higher cost, lower precision, or different governance properties.

### Fallback Too Late

The system keeps retrying the primary path after it is clearly not working.

Example: five failed calls to the same specialist agent when a fallback agent or human escalation should have been triggered after two failures.

### Fail Open

The system proceeds as if success occurred even though a required step failed.

Example: policy checker returns `ambiguous`, but the response agent gives a confident customer-facing answer.

This is one of the most dangerous recovery failures.

### Fail Closed Too Aggressively

The system stops even though a safe recovery path exists.

Example: a retriever timeout causes the workflow to stop, even though a retry budget and fallback source are available.

Failing closed is safer than guessing, but overusing it can make the product brittle and frustrating.

## Error Classification Is The First Evaluation Layer

Good recovery starts with error classification.

The orchestrator should distinguish:

- transient infrastructure errors
- permission errors
- schema errors
- empty retrieval
- low-confidence retrieval
- ambiguous policy result
- tool unavailable
- unsafe action blocked
- downstream agent malformed output

These error classes should not all trigger the same recovery path.

For example:

| Error Class | Usually Retry? | Better Recovery |
| --- | --- | --- |
| timeout | yes, with budget | retry same call or fallback after budget |
| empty retrieval | yes, with changed query | retry with better query or alternate source |
| 403 permission | no | stop, explain, or escalate |
| schema error | maybe | repair payload or regenerate structured output |
| policy ambiguous | no blind retry | gather more context or escalate |
| unsafe action blocked | no | fail closed or request approval |

The evaluator can check whether the system’s recovery decision matches the error class.

```yaml
recovery_rules:
  - error_class: permission_denied
    forbidden_decisions:
      - retry_same_call
    allowed_decisions:
      - fail_closed
      - escalate_to_human
      - explain_permission_issue

  - error_class: empty_retrieval
    allowed_decisions:
      - retry_with_modified_input
      - fallback_retriever
      - ask_clarifying_question
```

This rule layer catches a surprising number of failures. Many agent systems do not truly recover; they just keep asking the model what to do next. That may work sometimes, but it is not a reliable control policy.

## Retry Budgets Should Be Visible

Retries need budgets.

Without a budget, agents can loop. With an invisible budget, reviewers cannot tell whether the system stopped for the right reason. The trace should record the budget explicitly:

```json
{
  "type": "recovery_decision",
  "decision": "retry_with_modified_input",
  "error_class": "empty_retrieval",
  "attempt": 2,
  "max_attempts": 3,
  "budget_remaining": 1
}
```

This makes evaluation easier:

- Did the system exceed the retry budget?
- Did it stop before using the allowed budget?
- Did the retry strategy change after repeated failure?
- Did it escalate when the budget was exhausted?

The budget should match risk and cost. A cheap read-only retrieval may allow two or three retries. A write action should usually have a much stricter recovery policy. A payment, permission, or customer-notification workflow may require fail-closed behavior after one failed validation.

Budgets are not just operational settings. They are part of the orchestration contract.

## A Retry Should Usually Change Something

One practical check I would add early:

> If the failure is semantic, the retry must change something meaningful.

For retrieval failures, that may mean:

- query changed
- filters changed
- source changed
- search strategy changed
- clarifying question asked

For schema failures, that may mean:

- payload repaired
- missing field added
- tool schema re-read
- structured-output generation retried with validation feedback

For specialist-agent failures, that may mean:

- task summary improved
- context reduced
- fallback specialist selected
- human escalation triggered

The evaluator can compute an input diff:

```text
Failure: empty retrieval
Retry decision: retry_with_modified_input

Previous query:
  "audit logs"

Retry query:
  "enterprise audit log retention period policy"

Verdict:
  pass, retry changed the retrieval intent
```

Or:

```text
Failure: empty retrieval
Retry decision: retry_same_call

Previous query:
  "audit logs"

Retry query:
  "audit logs"

Verdict:
  fail, semantic failure retried without meaningful input change
```

This is a small check, but it separates deliberate recovery from hopeful repetition.

## Fallbacks Need Quality Boundaries

Fallbacks are not automatically safe.

If the policy retriever fails, falling back to a general web search may produce an answer, but it may also violate the product’s source-of-truth requirement. If a specialist agent fails, falling back to a generalist agent may be acceptable for a draft, but not for a customer-facing decision. If a structured API fails, falling back to model memory may be unacceptable.

A fallback rule should define what the fallback is allowed to do.

```yaml
fallback_rules:
  - from: search_policy_docs
    to: search_help_center
    allowed_when:
      - policy_docs_unavailable
    constraints:
      - final_answer_must_disclose_non_policy_source
      - customer_facing_decision_forbidden
    severity: high
```

This rule does not ban fallback. It says fallback changes the trust level of the result.

That is the key point: recovery decisions often change the confidence, authority, or risk profile of the workflow. Evaluation should check whether the system acknowledges that change.

## Fail Open vs Fail Closed

In orchestration evaluation, one of the most important questions is whether the system fails open or fails closed.

Fail open means the workflow continues as if the failed step succeeded. Fail closed means the workflow refuses to continue autonomously.

For low-risk tasks, fail open may be acceptable. If a brainstorming agent cannot retrieve an optional example, it may still produce a useful draft with a caveat. For high-risk tasks, fail open can be unacceptable. If a policy checker fails, the system should not produce a confident policy decision.

A recovery evaluator should encode this:

```yaml
fail_closed_rules:
  - required_step: check_policy_constraints
    on_failure:
      allowed_decisions:
        - fail_closed
        - escalate_to_human
      forbidden_decisions:
        - final_answer
        - proceed_as_success
    severity: critical
```

This rule is easy to understand and hard to replace with final-answer evaluation. The final answer may be polished. The problem is that the system should never have generated it under that failure state.

## Recovery Trace Schema

To evaluate recovery, I would add explicit recovery events to the trace.

```json
{
  "type": "recovery_decision",
  "after_event_id": "tool_17",
  "error_class": "tool_timeout",
  "decision": "retry_same_call",
  "reason": "transient_timeout",
  "attempt": 1,
  "max_attempts": 2,
  "budget_remaining": 1,
  "next_step": {
    "type": "tool_call",
    "name": "retrieve_policy_docs"
  }
}
```

This event makes the orchestrator’s behavior auditable. Instead of inferring recovery from raw logs, the system states what it thought happened and why it chose the next step.

For evaluation, I would check:

- Was `error_class` correct?
- Was `decision` allowed for that error class?
- Was retry budget respected?
- Did the retry change input when required?
- Was fallback allowed?
- Did the workflow fail closed when required?
- Was escalation triggered at the right threshold?

This gives the evaluator a clean target.

## A Report Format For Recovery Failures

Recovery failures should be reported as decision points.

Example:

```text
Case: enterprise_audit_log_retention
Failure: blind retry after empty retrieval
Severity: medium

Failed step:
  search_policy_docs(query="audit logs")

Observed recovery:
  retry_same_call

Expected recovery:
  retry_with_modified_input OR fallback_retriever OR ask_clarifying_question

Why this matters:
  Empty retrieval from a broad query is a semantic failure. Retrying the same
  query is unlikely to produce new evidence and may waste budget before the
  agent gives an unsupported answer.
```

Another example:

```text
Case: customer_entitlement_decision
Failure: fail-open after policy checker ambiguity
Severity: critical

Failed step:
  check_policy_constraints -> ambiguous

Observed recovery:
  final_answer

Expected recovery:
  gather_more_context OR escalate_to_human OR fail_closed

Why this matters:
  The agent produced a customer-facing entitlement answer after the required
  policy checker returned an ambiguous result.
```

This format is useful because it makes the recovery decision reviewable. The reviewer sees the failure, the chosen recovery, the expected recovery, and the risk.

## Evaluating Recovery Without Overfitting

There is a risk of making recovery evaluation too rigid.

If every failure has exactly one allowed recovery, the system may become brittle. Sometimes retrying with a better query and asking a clarifying question are both acceptable. Sometimes fallback is acceptable only if the final answer includes a caveat. Sometimes escalation is acceptable but not ideal.

So I would avoid single-path recovery rules unless the workflow is high-risk.

Instead, use allowed sets and constraints:

```yaml
error_class: low_confidence_retrieval
allowed_decisions:
  - retry_with_modified_input
  - fallback_to_approved_secondary_source
  - ask_clarifying_question
constraints:
  - final_answer_requires_source_disclosure
  - max_retries: 2
```

This gives the orchestrator room to adapt while still preventing unsafe behavior.

## What I Would Build First

If I were adding recovery evaluation to an agent harness, I would build it in this order:

1. Normalize tool and agent failures into a small set of error classes.
2. Add explicit `recovery_decision` events to traces.
3. Track retry attempts and budgets.
4. Detect blind retries after semantic failures.
5. Add forbidden-retry rules for permission and safety failures.
6. Add fail-closed rules for required validation steps.
7. Add fallback constraints for weaker sources or weaker agents.
8. Add escalation thresholds for repeated failure.
9. Generate decision-point reports for recovery failures.

This is mostly deterministic. It does not require a model judge at first. The hard part is product judgment: deciding which recovery decisions are acceptable for each failure class.

That judgment should be explicit. If it lives only inside prompts, it will drift.

## The Bigger Point

Recovery behavior is where agent orchestration becomes operationally real.

Happy-path traces show whether the system can work. Recovery traces show whether the system can be trusted when work gets messy.

The mature question is not:

> Did the agent eventually produce an answer?

It is:

> When something failed, did the orchestrator choose a recovery path that matched the error, respected the budget, preserved safety, and made the final result appropriately trustworthy?

That is a higher bar. But it is the bar production agent systems eventually have to meet.
