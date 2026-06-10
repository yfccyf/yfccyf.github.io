---
title: "The Right Tool at the Wrong Time: Evaluating Tool-Call Order in AI Agent Workflows"
description: "Why multi-step AI agents need ordering checks, not just tool-selection and argument validation."
date: 2026-04-05T09:00:00-04:00
draft: false
tags:
  - AI agents
  - evaluation
  - tool calls
  - reliability
---

An agent can call the right tools, pass valid arguments, and still fail because it did those things in the wrong order.

This is one of the less obvious failure modes in tool-using AI systems. It does not look like a hallucination. It does not look like a schema error. It does not even look like the agent ignored the available tools. When you inspect the trace, every tool you expected may be present. The problem is temporal: the safety check happened after the sensitive lookup, the final answer was generated before the policy validation, or the write operation happened before the agent read the current state.

That is the topic of this essay: **tool-call ordering evaluation**.

The previous essay focused on argument-level correctness: the agent chose the right tool, but the payload was wrong. This essay looks at a different layer of the same problem:

> In multi-step agent workflows, correctness is not only about which tools were called. Sometimes correctness depends on when they were called.

## A Small Incident That Looks Fine At First

Imagine a customer-support agent that helps answer account-access questions. A user asks:

> “Can you check whether this customer has access to advanced audit logs?”

The expected workflow is simple:

1. Resolve the customer identity.
2. Check whether the requesting user is authorized to view the customer account.
3. Retrieve the customer’s current plan.
4. Retrieve the approved entitlement policy.
5. Check feature entitlement.
6. Produce the final answer.

Now imagine the observed trace looks like this:

```json
[
  { "type": "tool_call", "name": "resolve_customer" },
  { "type": "tool_call", "name": "get_customer_plan" },
  { "type": "tool_call", "name": "check_requester_authorization" },
  { "type": "tool_call", "name": "retrieve_entitlement_policy" },
  { "type": "tool_call", "name": "check_feature_entitlement" },
  { "type": "final_answer" }
]
```

At a tool-selection level, this looks pretty good. The agent called the expected tools. At an argument-schema level, it may also pass. The payloads may all be well-formed.

But there is a serious ordering problem: the agent retrieved the customer plan before checking whether the requester was authorized to view that account.

If evaluation only asks “did the agent call the right tools?”, this run passes. If evaluation asks “were the arguments valid?”, it may still pass. The failure only appears when we evaluate the workflow as a sequence with constraints.

This is why ordering deserves its own layer.

## The Three Kinds Of Ordering Contracts

Not every workflow needs strict ordering. Some agents can call tools in flexible ways and still behave correctly. But in production systems, certain steps are not just implementation details. They are part of the behavioral contract.

I like to split ordering expectations into three categories.

### 1. Safety Gates

A safety gate is a step that must happen before a risky action. Authorization checks, policy checks, consent checks, data-classification checks, and rate-limit checks often belong here.

Example:

```text
check_requester_authorization
  must happen before
get_customer_plan
```

This is not about optimization. It is about the system’s safety boundary. If the agent crosses that boundary in the wrong order, the run should fail even if the final answer is correct.

### 2. Information Dependencies

An information dependency is a step that must happen before another step can be meaningfully performed. If the agent needs the current customer plan before checking feature entitlement, then entitlement checking should not happen first.

Example:

```text
get_customer_plan
  must happen before
check_feature_entitlement
```

The wrong order may produce a subtle bug. The agent might use stale context, guess a missing value, or call the later tool with arguments derived from incomplete information.

### 3. Read-Before-Write Contracts

Read-before-write is a classic software pattern, and agents need it too. Before an agent updates a ticket, changes a record, sends a message, or triggers a workflow, it often needs to read the current state.

Example:

```text
get_ticket_status
  must happen before
update_ticket_status
```

This prevents the agent from overwriting newer state, duplicating an action, or applying an update that is no longer appropriate.

These three categories are useful because they imply different severity levels. Violating a safety gate may be critical. Violating an information dependency may be high or medium depending on the workflow. Violating read-before-write may be critical if the write has external side effects.

## Partial Order, Not Exact Order

The first instinct is to compare the observed trace to a golden sequence:

```text
resolve_customer
check_requester_authorization
get_customer_plan
retrieve_entitlement_policy
check_feature_entitlement
final_answer
```

This is tempting, but often too brittle. Agents may add harmless intermediate steps. Retrieval may happen before plan lookup in some workflows. A retry may insert a second call. A summarization step may appear when context gets long. If we require an exact sequence, we will fail runs that are different but acceptable.

A better default is **partial-order evaluation**.

Instead of saying “the whole trace must equal this exact list,” we say:

```yaml
ordering:
  - before: check_requester_authorization
    after: get_customer_plan
    severity: critical

  - before: get_customer_plan
    after: check_feature_entitlement
    severity: high

  - before: check_feature_entitlement
    after: final_answer
    severity: high
```

This lets the agent vary in harmless ways while preserving the ordering constraints that matter.

The evaluator is not asking whether the agent followed a script. It is asking whether the agent respected the contract.

## A Minimal Ordering Matcher

An ordering matcher can be surprisingly small. Given a normalized trace, find the first occurrence of the required “before” event and the first occurrence of the “after” event. Then check whether the index of the before event is smaller than the index of the after event.

```ts
type Event = {
  type: "tool_call" | "retrieval" | "final_answer";
  name?: string;
};

type OrderingRule = {
  before: string;
  after: string;
  severity: "low" | "medium" | "high" | "critical";
};

function eventMatches(event: Event, target: string): boolean {
  if (target === "final_answer") {
    return event.type === "final_answer";
  }

  return event.type === "tool_call" && event.name === target;
}

function checkOrdering(events: Event[], rule: OrderingRule) {
  const beforeIndex = events.findIndex((event) => eventMatches(event, rule.before));
  const afterIndex = events.findIndex((event) => eventMatches(event, rule.after));

  if (beforeIndex === -1) {
    return {
      passed: false,
      severity: rule.severity,
      message: `Expected ${rule.before} before ${rule.after}, but ${rule.before} was not observed.`,
    };
  }

  if (afterIndex === -1) {
    return {
      passed: false,
      severity: rule.severity,
      message: `Expected ${rule.before} before ${rule.after}, but ${rule.after} was not observed.`,
    };
  }

  return {
    passed: beforeIndex < afterIndex,
    severity: rule.severity,
    message:
      beforeIndex < afterIndex
        ? `${rule.before} occurred before ${rule.after}.`
        : `${rule.before} occurred after ${rule.after}.`,
  };
}
```

This simple version already catches many meaningful regressions. It can tell us whether a policy check happened before the final answer, whether a read happened before a write, and whether authorization happened before sensitive data access.

But the simple version also hides a few hard questions.

## Challenge: First Occurrence Is Not Always Enough

Suppose the trace looks like this:

```json
[
  { "type": "tool_call", "name": "check_requester_authorization", "result": "denied" },
  { "type": "tool_call", "name": "get_customer_plan" },
  { "type": "tool_call", "name": "check_requester_authorization", "result": "approved" },
  { "type": "final_answer" }
]
```

A naive matcher might pass because it sees `check_requester_authorization` before `get_customer_plan`. But the authorization result was denied. The later approved check happened after the data access.

The ordering check needs to consider event predicates, not only event names. The rule should be able to say:

```yaml
ordering:
  - before:
      tool: check_requester_authorization
      result: approved
    after:
      tool: get_customer_plan
    severity: critical
```

That means the matcher needs to find the first authorization event that satisfies the expected condition, not just the first event with the right tool name.

This is an important implementation detail. In real traces, repeated tool calls are normal. Agents retry. Tools fail. Authorization may be rechecked. Search may run multiple times. A useful ordering evaluator has to match the right event, not merely the first event.

## Challenge: Retries Should Not Automatically Fail

Retries make ordering evaluation tricky.

Consider:

```json
[
  { "type": "tool_call", "name": "retrieve_entitlement_policy", "status": "error" },
  { "type": "tool_call", "name": "get_customer_plan", "status": "success" },
  { "type": "tool_call", "name": "retrieve_entitlement_policy", "status": "success" },
  { "type": "tool_call", "name": "check_feature_entitlement", "status": "success" },
  { "type": "final_answer" }
]
```

If the rule says `retrieve_entitlement_policy` must happen before `check_feature_entitlement`, should this pass? Probably yes, as long as the successful retrieval happened before the entitlement check.

This suggests an ordering rule should specify which event status counts:

```yaml
ordering:
  - before:
      tool: retrieve_entitlement_policy
      status: success
    after:
      tool: check_feature_entitlement
      status: success
```

The resolution is to normalize event status and make match conditions explicit. A failed attempt should remain in the trace for debugging, but the matcher should know whether failed attempts satisfy the ordering requirement.

## Challenge: Optional Tools Create Branches

Some workflows have valid branches. If the user asks a simple question, the agent may not need escalation. If the confidence score is low, it may need a human-review tool. If the retrieved policy is ambiguous, it may need a secondary source.

Exact sequence matching handles this poorly. Partial-order checks handle it better, but branches still need structure.

One useful pattern is conditional ordering:

```yaml
conditional_ordering:
  - if:
      tool: create_support_ticket
    then:
      before:
        tool: collect_diagnostic_context
      after:
        tool: create_support_ticket
      severity: high
```

This rule says: if the agent creates a support ticket, it must collect diagnostic context first. If the agent never creates a ticket, the rule does not apply.

Conditional ordering is especially useful for write tools, escalation tools, notification tools, and workflow-triggering tools. The system does not need to force every run down the same path. It only enforces ordering when a risky branch is taken.

## Challenge: Parallel Tool Calls Need A Different Model

Some agents can call tools in parallel. Ordering is less obvious when two calls start at nearly the same time.

For example:

```json
[
  {
    "type": "tool_call",
    "name": "retrieve_product_docs",
    "started_at": "10:00:01",
    "finished_at": "10:00:04"
  },
  {
    "type": "tool_call",
    "name": "retrieve_policy_docs",
    "started_at": "10:00:01",
    "finished_at": "10:00:06"
  },
  {
    "type": "tool_call",
    "name": "synthesize_answer",
    "started_at": "10:00:07",
    "finished_at": "10:00:08"
  }
]
```

In this trace, the retrieval calls are concurrent, and that may be fine. The important ordering constraint is that both retrievals finish before synthesis begins.

So ordering rules sometimes need to compare lifecycle boundaries, not just event positions:

```yaml
ordering:
  - before:
      tool: retrieve_policy_docs
      boundary: finished_at
    after:
      tool: synthesize_answer
      boundary: started_at
```

For early harnesses, it is acceptable to flatten events into a sequence. But as agent runtimes become more parallel, ordering evaluation should eventually model start and finish times.

## A Different Format For The Report: Timeline First

For ordering failures, a timeline report is often clearer than a list of assertions.

Instead of only reporting:

```text
FAILED: check_requester_authorization must happen before get_customer_plan
```

show the relevant slice of the trace:

```text
Case: customer_feature_access
Failure: authorization gate violated
Severity: critical

Timeline:
  01. resolve_customer                         success
  02. get_customer_plan                        success   <-- sensitive access
  03. check_requester_authorization            success   <-- happened too late
  04. retrieve_entitlement_policy              success
  05. check_feature_entitlement                success
  06. final_answer                             success

Expected:
  check_requester_authorization before get_customer_plan

Why this matters:
  The agent accessed customer plan data before confirming that the requester
  was authorized to view the customer account.
```

This format makes the bug obvious. A reviewer can see the sequence without reconstructing it mentally from raw JSON.

For tool ordering, the report should feel like a timeline debugger.

## State Machines Are Useful For High-Risk Workflows

Partial-order rules are a good starting point, but some workflows deserve a stronger model. For high-risk processes, a small state machine may be clearer than a pile of pairwise ordering rules.

For example, a customer-account workflow might have states:

```text
unresolved
  -> customer_resolved
  -> requester_authorized
  -> context_retrieved
  -> policy_checked
  -> answered
```

Each tool call moves the workflow from one state to another. Some actions are only allowed in certain states:

| Current State | Allowed Tool | Next State |
| --- | --- | --- |
| `unresolved` | `resolve_customer` | `customer_resolved` |
| `customer_resolved` | `check_requester_authorization` | `requester_authorized` |
| `requester_authorized` | `get_customer_plan` | `context_retrieved` |
| `context_retrieved` | `check_feature_entitlement` | `policy_checked` |
| `policy_checked` | `final_answer` | `answered` |

If the agent calls `get_customer_plan` while still in `customer_resolved`, the state machine rejects the transition.

This is stricter than partial-order matching, but it can be worth it for workflows with clear governance requirements. The state machine becomes a compact expression of the process contract.

## When State Machines Are Too Much

State machines can also become over-engineered. Not every agent workflow has a clean linear process. Some workflows are exploratory. Some require multiple retrieval loops. Some have optional branches. Some evolve quickly as product requirements change.

If the state machine changes every week, it may become a maintenance burden.

My practical rule:

- Use partial-order rules for flexible workflows.
- Use state machines for high-risk workflows with stable process requirements.
- Avoid exact sequence matching unless the workflow is truly deterministic.

This keeps the evaluation layer proportional to the risk.

## Ordering Checks Should Have Severity

Ordering violations are not all equal.

Calling `retrieve_product_docs` before `retrieve_policy_docs` may be harmless. Calling `send_customer_email` before `human_approval` may be critical. Calling `final_answer` before `retrieve_citations` may be medium or high depending on the product.

The rule should carry severity:

```yaml
ordering:
  - before: human_approval
    after: send_customer_email
    severity: critical

  - before: retrieve_policy_docs
    after: final_answer
    severity: high

  - before: retrieve_product_docs
    after: retrieve_policy_docs
    severity: low
```

Then CI behavior can be tiered:

- critical: block release
- high: block or require explicit approval
- medium: warn and request review
- low: record as drift

This is how the harness stays usable. If every ordering difference blocks release, people will treat the evaluation system as noise. If dangerous ordering violations only create warnings, the evaluation system is too weak.

## A Checklist For Designing Ordering Tests

When adding ordering checks for an agent workflow, I would ask:

1. Which tool calls access sensitive data?
2. Which tool calls create external side effects?
3. Which checks must happen before those actions?
4. Which values must be read before they are used?
5. Which write operations require current state?
6. Which branches are optional?
7. Which retries should count as satisfying the rule?
8. Which failures should block release?
9. Which ordering differences are harmless?
10. Which ordering rules are stable enough to encode?

That last question is important. If a rule is not stable, it may not belong in CI yet. It may belong in a report for human review first.

## The Bigger Point

Tool ordering is easy to overlook because traces are usually displayed as logs, not as contracts. We scan a trace to see whether the expected tools appeared. But in many real workflows, the important question is not only whether a tool appeared. It is whether it appeared before or after another step that changes its meaning.

Authorization before access. Read before write. Policy before answer. Approval before send. Context before escalation.

These are old software ideas showing up again inside agent systems.

The lesson is not that every agent should follow a rigid script. The lesson is that flexible systems still need stable boundaries. Ordering evaluation is one way to make those boundaries explicit.

The mature question is:

> Did the agent call the right tools, with the right arguments, in an order that respects the workflow contract?

If the answer is yes, the final response becomes easier to trust. If the answer is no, a good-looking answer may only be hiding a process failure.
