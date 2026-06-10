---
title: "When the Tool Is Right but the Arguments Are Wrong: Evaluating Tool-Call Arguments in AI Agents"
description: "A practical framework for evaluating whether an AI agent passed the right argument payload to the right tool."
date: 2026-03-31T09:00:00-04:00
draft: false
tags:
  - AI agents
  - evaluation
  - tool calls
  - reliability
---

Tool-calling agents usually get evaluated at the wrong level of detail.

The first question people ask is understandable: **did the agent call the right tool?** If a user asks for customer entitlement information, did the agent call the entitlement lookup tool? If a support workflow requires ticket creation, did the agent call the ticketing tool? If a policy question needs a retrieval step, did the agent call search before answering?

That level of evaluation is useful, but it is not enough. In many production failures, the agent chooses the right tool and still does the wrong thing. The tool name is correct. The action category is correct. The trace even looks reasonable at a glance. The problem is inside the argument payload.

The agent calls `search_docs`, but the query is too broad and retrieves a generic page instead of the policy that governs the case. It calls `get_customer_plan`, but passes the wrong `customer_id` from an earlier turn. It calls `create_ticket`, but drops severity, product area, or tenant context. It calls `send_email`, but uses the right body with the wrong recipient. It calls `update_record`, but sends a stale field value that was true three steps ago and is no longer valid.

This essay is about that narrower layer: **argument-level evaluation for agent tool calls**.

The core idea is simple:

> Tool-call correctness should not stop at tool selection. For production agents, the argument payload often determines whether a tool call is useful, harmless, or dangerous.

## The Tool Name Is Only The First Gate

A tool call has at least three parts:

1. the tool selected
2. the argument payload passed to the tool
3. the tool result and how the agent uses it afterward

Most simple evaluations focus on the first part. That makes sense as a starting point because tool selection failures are visible. If the agent calls `update_customer_plan` when it should have called `get_customer_plan`, everyone understands the problem.

Argument failures are quieter. They can pass casual review because the trace still contains the expected tool name. In a dashboard, both of these runs may look similar:

```json
{
  "type": "tool_call",
  "name": "get_customer_plan",
  "arguments": {
    "customer_id": "cust_123"
  }
}
```

```json
{
  "type": "tool_call",
  "name": "get_customer_plan",
  "arguments": {
    "customer_id": "cust_132"
  }
}
```

The difference is tiny in text and large in consequence. The first call retrieves the intended customer. The second call retrieves a different customer. If the final answer still sounds plausible, a final-answer-only judge may miss the failure entirely.

This is why tool-call evaluation needs a second gate. After asking “did the agent call the right tool?”, we need to ask:

> Did the agent call the tool with the right arguments, derived from the right context, at the right time?

That is a more demanding question, but it is much closer to the real behavior we care about.

## A Simple Taxonomy Of Argument Failures

Before designing evaluation, it helps to name the failure modes. I usually think about argument failures in six buckets.

| Failure Type | What Happens | Example |
| --- | --- | --- |
| Missing required argument | The tool call omits a field needed for correctness | `create_ticket` without `severity` |
| Wrong entity | The field is present but refers to the wrong object | `customer_id` from a previous conversation |
| Stale value | The argument was once true but no longer reflects current context | old plan tier after a plan update |
| Unsafe value | The argument triggers a risky action or violates a policy | `send_email` to an unverified address |
| Underspecified value | The argument is too vague to retrieve or act correctly | search query: `"policy"` |
| Over-broad value | The argument expands scope beyond what the user authorized | requesting all accounts instead of one account |

These categories are useful because they point to different checks. Missing required arguments can be caught with schema validation. Wrong entities often require provenance tracking. Unsafe values need policy checks. Underspecified values may need semantic evaluation. Over-broad values need scope constraints.

If all of these are collapsed into “tool call failed,” the evaluation report will not be very helpful. A good harness should explain which kind of argument problem occurred.

## Schema Validation Is Necessary But Not Sufficient

The first layer is schema validation. Every tool should have a clear input schema, and every agent tool call should be checked against it.

For example:

```json
{
  "name": "create_support_ticket",
  "schema": {
    "type": "object",
    "required": ["customer_id", "severity", "product_area", "summary"],
    "properties": {
      "customer_id": { "type": "string" },
      "severity": {
        "type": "string",
        "enum": ["low", "medium", "high", "urgent"]
      },
      "product_area": { "type": "string" },
      "summary": { "type": "string", "minLength": 10 }
    }
  }
}
```

This catches a useful class of failures:

- missing fields
- invalid enum values
- malformed types
- empty strings
- unexpected argument shapes

But schema validation only answers: **is this payload structurally valid?**

It does not answer: **is this payload correct for this task?**

This distinction matters. The following payload may be perfectly valid according to the schema:

```json
{
  "customer_id": "cust_132",
  "severity": "medium",
  "product_area": "billing",
  "summary": "Customer is asking about feature access."
}
```

If the active customer is actually `cust_123`, the schema will still pass. The argument is well-formed and wrong. That is the uncomfortable middle ground where many agent failures live.

## Argument Correctness Depends On Provenance

To evaluate argument correctness, we often need to know where the argument came from.

Consider a customer-support agent. The user asks:

> “Can you check whether Acme Corp can use advanced audit logs?”

The agent might need to resolve “Acme Corp” to a canonical account ID before calling the entitlement tool. A good trace should not only record the final tool call; it should record the chain of evidence that produced the argument.

```json
{
  "type": "entity_resolution",
  "input": "Acme Corp",
  "resolved_entity": {
    "type": "customer",
    "id": "cust_123",
    "name": "Acme Corporation"
  },
  "confidence": 0.97
}
```

Then the later tool call can be checked against that resolved entity:

```json
{
  "type": "tool_call",
  "name": "get_customer_plan",
  "arguments": {
    "customer_id": "cust_123"
  }
}
```

The evaluation question becomes:

> Does the `customer_id` passed to `get_customer_plan` match the customer entity resolved from the user request?

This is a stronger check than schema validation. It connects the argument to the task context. It also makes debugging easier. If the wrong ID appears, the report can show whether the error came from entity resolution, state management, argument construction, or a later mutation.

Without provenance, argument evaluation becomes guesswork. With provenance, it becomes a trace consistency problem.

## Entity Consistency Across Steps

Many argument bugs are consistency bugs. The agent starts with one entity, switches to another, and never notices.

This happens easily in multi-turn workflows. A conversation may mention several customers, tickets, documents, products, or accounts. The model may pick up the wrong one from context. Retrieval may return a similar entity. A tool result may include related records, and the agent may accidentally use a related record as the primary record.

A useful evaluation harness should track key entities across the trace:

```json
{
  "case_id": "customer_feature_access",
  "expected_entities": {
    "customer_id": "cust_123",
    "product": "advanced_audit_logs"
  },
  "checks": [
    {
      "type": "argument_equals",
      "tool": "get_customer_plan",
      "path": "$.customer_id",
      "expected": "cust_123"
    },
    {
      "type": "argument_equals",
      "tool": "check_feature_entitlement",
      "path": "$.customer_id",
      "expected": "cust_123"
    },
    {
      "type": "argument_equals",
      "tool": "check_feature_entitlement",
      "path": "$.feature_key",
      "expected": "advanced_audit_logs"
    }
  ]
}
```

This is not glamorous. It is basic consistency checking. But basic consistency checking catches serious failures.

For high-risk workflows, I would make entity consistency a first-class evaluation layer. The agent should not be allowed to silently change customer, tenant, user, account, or document scope unless there is an explicit step that explains and authorizes the change.

## Argument Scope Is A Safety Property

Some arguments are dangerous not because they are invalid, but because they are too broad.

Imagine a tool:

```json
{
  "name": "search_customer_records",
  "arguments": {
    "tenant_id": "tenant_abc",
    "query": "billing issue",
    "limit": 1000
  }
}
```

The tool call may be valid. The tool may even return useful results. But if the user asked about one account, retrieving 1000 records across a tenant may violate least-privilege expectations. The issue is not syntax. It is scope.

Argument-level evaluation should include scope checks:

- Is the requested tenant the active tenant?
- Is the record scope limited to the user’s authorized context?
- Is the query constrained enough for the task?
- Is the result limit reasonable?
- Is the tool being asked to retrieve more data than needed?

These checks are especially important for tools that read or write sensitive data. An agent that calls the right tool with an over-broad argument can create privacy, compliance, or operational risk.

The evaluation rule might look like:

```yaml
checks:
  - type: argument_max
    tool: search_customer_records
    path: "$.limit"
    max: 25

  - type: argument_equals_context
    tool: search_customer_records
    path: "$.tenant_id"
    context_path: "$.active_tenant_id"

  - type: argument_required_filter
    tool: search_customer_records
    path: "$.filters.customer_id"
```

This kind of rule says: the agent can use the search tool, but it must use it in a scoped way.

That is a practical governance pattern. Do not only ask whether the agent can act. Ask whether it acts within the right boundary.

## Evaluating Search Queries As Arguments

Search and retrieval tools deserve special attention because their arguments are often natural language. A search query is not just a string. It is the agent’s interpretation of what information is needed.

Bad retrieval often begins with a bad query.

For example, suppose the user asks:

> “Under the enterprise plan, are audit logs retained for 180 days or 365 days?”

The agent might call:

```json
{
  "name": "search_docs",
  "arguments": {
    "query": "audit logs"
  }
}
```

That query is not wrong in a schema sense. It is just weak. It may retrieve logging setup docs, marketing docs, or general audit-log pages instead of the retention policy for enterprise plans.

A stronger query might be:

```json
{
  "name": "search_docs",
  "arguments": {
    "query": "enterprise plan audit log retention 180 days 365 days policy",
    "source_type": "policy",
    "product_area": "audit_logs"
  }
}
```

Query argument evaluation can check:

- Does the query include the key entity or product?
- Does it preserve the user’s constraint?
- Does it include the relevant plan, tier, or environment?
- Does it restrict source type when the task requires authoritative policy?
- Does it avoid adding unsupported assumptions?

This is where semantic checks can help. Exact string matching is too brittle for search queries. But we can still define expectations:

```yaml
checks:
  - type: query_covers_concepts
    tool: search_docs
    path: "$.query"
    concepts:
      - "enterprise plan"
      - "audit log retention"
      - "180 days or 365 days"

  - type: argument_equals
    tool: search_docs
    path: "$.source_type"
    expected: "policy"
```

The first check may need embeddings or an LLM judge. The second should be deterministic. A useful evaluation harness should combine both rather than pretending one method solves everything.

## Tool Arguments Should Be Diffable

One of the most useful things an evaluation system can produce is an argument diff.

If a prompt change causes a tool argument to shift from:

```json
{
  "customer_id": "cust_123",
  "feature_key": "advanced_audit_logs",
  "source_type": "policy"
}
```

to:

```json
{
  "customer_id": "cust_123",
  "feature_key": "audit_logs",
  "source_type": "help_center"
}
```

the final answer might still pass a loose rubric. But the argument diff reveals two behavioral changes:

- the feature key became less specific
- the source type moved from policy to help-center content

That is exactly the kind of drift a reviewer should see before release.

The report should make this visible:

```text
Case: enterprise_audit_log_retention
Tool: search_docs

Changed arguments:
- feature_key: expected "advanced_audit_logs", observed "audit_logs"
- source_type: expected "policy", observed "help_center"

Risk:
- retrieval may use non-authoritative documentation for a policy-sensitive question
```

This is more useful than a generic “retrieval failed” message. It tells the engineer what changed and why it matters.

## Required, Optional, Derived, And Forbidden Arguments

Not all arguments have the same role. A practical evaluator should distinguish at least four types:

| Argument Type | Meaning | Example |
| --- | --- | --- |
| Required | Must be present for the tool call to be valid | `customer_id` |
| Optional | Helpful but not always necessary | `include_inactive_records` |
| Derived | Must match something computed earlier | resolved account ID |
| Forbidden | Must not be present or must not take certain values | `admin_override: true` |

This distinction keeps evaluation precise. A missing required field should fail. A missing optional field may only warn. A derived field should be checked against its source. A forbidden argument should block release if it introduces risk.

For example:

```yaml
tool: update_support_ticket
arguments:
  required:
    - ticket_id
    - status
  derived:
    ticket_id:
      from: "$.resolved_ticket.id"
  forbidden:
    - path: "$.admin_override"
      value: true
  optional:
    - internal_note
```

This kind of configuration turns vague expectations into executable checks. It also forces the team to think through the contract of the tool itself.

## The Challenge Of Natural Language Arguments

Some arguments are structured identifiers. Others are natural language. Natural language arguments are harder because they can be valid in many forms.

Examples:

- ticket summaries
- email drafts
- search queries
- escalation reasons
- policy explanations
- notes written to a CRM

For these, exact comparison is usually the wrong tool. But leaving them untested is also risky. A malformed ticket summary can waste support time. A vague escalation reason can break triage. A poorly written CRM note can create compliance or customer-experience issues.

I would evaluate natural language arguments with a layered approach:

1. **Length and emptiness checks**: is the value non-empty and within expected size?
2. **Required concept checks**: does it mention the core issue, entity, or product?
3. **Forbidden content checks**: does it avoid unsupported claims or sensitive data?
4. **Rubric checks**: does it satisfy a task-specific quality bar?
5. **Human review sampling**: are borderline cases reviewed periodically?

For example:

```yaml
checks:
  - type: argument_min_length
    tool: create_support_ticket
    path: "$.summary"
    min: 30

  - type: argument_must_include_concepts
    tool: create_support_ticket
    path: "$.summary"
    concepts:
      - "feature access"
      - "enterprise plan"
      - "audit logs"

  - type: argument_must_not_include
    tool: create_support_ticket
    path: "$.summary"
    phrases:
      - "guaranteed refund"
      - "legal advice"
```

The important thing is to be honest about uncertainty. Natural language argument checks will not be perfect. They are still better than pretending the payload does not matter.

## A Minimal Argument Evaluation Report

A useful report should group failures by tool and argument path. That makes it easier for an engineer to fix the prompt, tool description, schema, or state-passing logic.

Example:

```text
Case: enterprise_audit_log_retention
Status: failed

Tool call:
  search_docs

Failed checks:
  1. query_covers_concepts
     path: $.query
     expected concepts:
       - enterprise plan
       - audit log retention
       - 180 days or 365 days
     observed:
       "audit logs"

  2. argument_equals
     path: $.source_type
     expected:
       "policy"
     observed:
       "help_center"

Risk:
  The agent searched a broad help-center source instead of the authoritative
  policy corpus. The final answer may be based on non-policy documentation.
```

This report is actionable. It points to the exact argument path, explains the expectation, shows the observed value, and states the risk in plain language.

That last part matters. Evaluation reports should not only satisfy machines. They should help humans make release decisions.

## Where Argument Evaluation Fits In The Larger Harness

Argument-level checks fit naturally into a golden trace regression harness.

The trace tells us which tools were called. Argument evaluation goes one layer deeper and checks whether the payloads were correct for the task.

```text
trace-level checks
  |
  |-- required tool called?
  |-- forbidden tool avoided?
  |-- required order preserved?
  |
  |-- argument-level checks
        |
        |-- required fields present?
        |-- entities consistent?
        |-- scope constrained?
        |-- derived values correct?
        |-- unsafe values blocked?
        |-- natural language payload acceptable?
```

This layering is important. Argument evaluation should not replace trace evaluation. It should refine it. First make sure the agent used the right tool path. Then inspect the payloads that make those tool calls meaningful.

## What I Would Build First

If I were adding argument-level evaluation to an agent regression harness, I would build it in this order:

1. Capture every tool call with normalized argument JSON.
2. Validate arguments against each tool’s schema.
3. Add required and forbidden argument checks.
4. Add entity consistency checks for customer, tenant, user, ticket, and document IDs.
5. Add scope checks for retrieval and data-access tools.
6. Add argument diffs between golden and observed traces.
7. Add semantic checks for search queries and natural language payloads.
8. Add risk levels so dangerous argument failures block release.

This sequence starts with deterministic checks because they are easier to trust. It saves semantic judging for the places where structure is not enough.

That is the pattern I keep coming back to with agent evaluation: use deterministic checks where the contract is clear, and use semantic checks where the contract is genuinely linguistic.

## The Bigger Point

Tool-calling agents make software feel more flexible, but the tools themselves are still software interfaces. They have contracts. They have schemas. They have permissions. They have side effects. If an agent passes the wrong arguments, the system can fail even when the trace looks superficially correct.

That is why argument-level evaluation deserves its own layer.

The mature question is not only:

> Did the agent call the right tool?

It is:

> Did the agent call the right tool with the right payload, grounded in the right context, constrained to the right scope, and safe for this workflow?

That question is more work to answer. But in production systems, it is often the question that decides whether an agent is reliable enough to trust.
