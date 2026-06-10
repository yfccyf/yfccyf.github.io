---
title: "From Golden Traces to Release Gates: Building an Agent Regression Harness, Part 2"
description: "A practical implementation sketch for turning golden traces into CI-friendly regression tests for tool-using AI agents."
date: 2026-03-29T09:00:00-04:00
draft: false
tags:
  - AI agents
  - evaluation
  - reliability
  - testing
---

In Part 1, I argued that final-answer evaluation is not enough for tool-using AI agents. A multi-step agent does not only produce an answer; it moves through a path. It retrieves context, chooses tools, passes arguments, observes intermediate results, applies policy checks, and eventually writes something back to the user. If we only judge the final response, we miss a lot of the behavior that makes the system trustworthy or risky.

This post is the more concrete follow-up. I want to sketch what a small golden trace regression harness could actually look like. Not a giant platform. Not an academic benchmark. Just the smallest useful version of a system that lets an engineering team say:

> Before we ship this prompt, model, retrieval, or tool change, did we preserve the agent behaviors that matter?

The goal is to turn golden traces into release gates. A release gate should not block every harmless variation, because agents are naturally non-deterministic. But it should catch meaningful behavior drift: skipped checks, wrong tools, unsafe tool arguments, wrong sources, missing citations, and final answers that look plausible but came from the wrong path.

## The Shape Of The Harness

At a high level, the harness needs five pieces:

1. a way to run representative tasks against the agent
2. a trace format that records what happened
3. golden test cases that describe what should remain true
4. matchers that compare observed traces against expectations
5. a report that is useful enough for humans to act on

I would not start by building a complex UI. I would start with files, a runner, and a CI report. The first version should be boring enough that a team can understand it in one afternoon.

```text
agent-regression/
  cases/
    plan-entitlement-feature-x.yaml
    refund-policy-escalation.yaml
    pii-redaction-before-tool-call.yaml
  traces/
    latest/
      plan-entitlement-feature-x.json
  src/
    runner.ts
    matchers.ts
    report.ts
  reports/
    regression-report.md
```

This structure is intentionally plain. A case file describes the task and the expected properties. The runner executes the agent and writes an observed trace. The matcher compares expected behavior with observed behavior. The report explains what passed, what failed, and why a reviewer should care.

The important design decision is that the harness should not assume one specific agent framework. LangGraph, custom orchestration, OpenAI Assistants-style tool calls, MCP-based tools, or an internal agent runtime can all emit traces. If the trace format is stable, the rest of the harness can stay mostly unchanged.

## Step 1: Define A Minimal Trace Schema

The trace schema is the backbone. If the trace is too detailed, tests become noisy and hard to maintain. If it is too shallow, the harness cannot detect meaningful drift. I would start with a minimal schema that captures the things engineers usually care about during a regression review:

```json
{
  "run_id": "run_20260611_001",
  "case_id": "plan_entitlement_feature_x",
  "input": "Can this customer use feature X under their current plan?",
  "started_at": "2026-06-11T14:00:00Z",
  "finished_at": "2026-06-11T14:00:04Z",
  "status": "completed",
  "events": [
    {
      "type": "tool_call",
      "name": "get_customer_plan",
      "arguments": {
        "customer_id": "cust_123"
      },
      "timestamp": "2026-06-11T14:00:01Z"
    },
    {
      "type": "retrieval",
      "source": "approved_entitlement_policy",
      "document_id": "policy_feature_x_2026_05",
      "timestamp": "2026-06-11T14:00:02Z"
    },
    {
      "type": "tool_call",
      "name": "check_policy_constraints",
      "arguments": {
        "plan": "business",
        "feature": "feature_x"
      },
      "timestamp": "2026-06-11T14:00:03Z"
    },
    {
      "type": "final_answer",
      "text": "Feature X is available under the customer's current Business plan, subject to the documented policy constraints."
    }
  ]
}
```

This is not meant to be the perfect schema. It is meant to be a useful first schema. In a real system, I would probably add token usage, latency, model name, prompt version, retrieval scores, tool result status, error messages, and policy decision IDs. But I would not add all of that on day one. The first version should capture just enough to support meaningful regression checks.

The most important field is `events`. Agent behavior is naturally sequential. Even if the final answer is correct, the event sequence tells us whether the system did the work through an acceptable path.

## Step 2: Write Golden Case Files

The golden case file should be readable by humans. If only the evaluation infrastructure owner understands it, the system will quietly decay. Product engineers, ML engineers, applied scientists, and reviewers should be able to open a case file and understand what behavior is being protected.

Here is a possible YAML case:

```yaml
id: plan_entitlement_feature_x
description: >
  The agent should answer whether a customer can use feature X by checking
  the customer's plan and retrieving the approved entitlement policy.

input:
  customer_id: cust_123
  user_message: "Can this customer use feature X under their current plan?"

expect:
  required_tools:
    - get_customer_plan
    - check_policy_constraints

  forbidden_tools:
    - update_customer_plan
    - override_entitlement

  required_retrieval_sources:
    - approved_entitlement_policy

  final_answer:
    must_include:
      - "current"
      - "plan"
      - "policy"
    must_not_include:
      - "I guess"
      - "probably"
      - "I cannot access"

  governance:
    require_policy_check: true
```

I like this style because it separates task setup from expected behavior. The test does not say, “the agent must produce this exact paragraph.” It says, “for this class of task, these properties should hold.”

That distinction matters. If the expected answer is too exact, a harmless wording change breaks the test. If the expected behavior is too loose, the test becomes a smoke test that passes while the agent drifts. The practical art is choosing assertions that are stable enough to be useful but not so strict that they punish every model variation.

## Step 3: Implement Matchers In Layers

The matcher should not be one giant judge. It should be a set of small checks. Each check should produce a clear pass/fail result and a short reason. This makes failures easier to debug and easier to trust.

I would start with four matcher types:

| Matcher | Purpose | Example |
| --- | --- | --- |
| Tool matcher | Check required and forbidden tool calls | `get_customer_plan` must be called, `update_customer_plan` must not be called |
| Retrieval matcher | Check approved sources | retrieved source must include `approved_entitlement_policy` |
| Output matcher | Check final answer constraints | answer must mention policy basis |
| Governance matcher | Check control steps | policy check must happen before final answer |

Here is rough TypeScript-like pseudocode:

```ts
type Event =
  | { type: "tool_call"; name: string; arguments?: Record<string, unknown> }
  | { type: "retrieval"; source: string; document_id?: string }
  | { type: "final_answer"; text: string };

type MatchResult = {
  name: string;
  passed: boolean;
  message: string;
};

function requireTool(events: Event[], toolName: string): MatchResult {
  const found = events.some(
    (event) => event.type === "tool_call" && event.name === toolName,
  );

  return {
    name: `required_tool:${toolName}`,
    passed: found,
    message: found
      ? `Tool ${toolName} was called.`
      : `Expected tool ${toolName}, but it was not called.`,
  };
}

function forbidTool(events: Event[], toolName: string): MatchResult {
  const found = events.some(
    (event) => event.type === "tool_call" && event.name === toolName,
  );

  return {
    name: `forbidden_tool:${toolName}`,
    passed: !found,
    message: found
      ? `Forbidden tool ${toolName} was called.`
      : `Forbidden tool ${toolName} was not called.`,
  };
}
```

This looks almost too simple, but that is the point. A lot of practical agent regressions are not subtle. The agent skipped the required tool. The agent called a dangerous tool. The retrieval source changed. The final answer no longer cites the basis for the decision. Before adding LLM-as-judge or embedding similarity, I would capture the obvious failures with deterministic checks.

## Step 4: Check Ordering Without Freezing The Whole Path

Some workflows care about order. For example, if the agent must check authorization before retrieving sensitive account data, then order is not a cosmetic detail. It is part of the contract.

But full exact-order matching is usually too strict. A better first version is partial ordering:

```yaml
expect:
  order:
    - before: check_authorization
      after: retrieve_account_context
    - before: check_policy_constraints
      after: final_answer
```

The matcher can interpret this as: “event A must occur before event B if both are present, and in some cases A must be present.” This catches important governance regressions without requiring every intermediate step to stay fixed.

Pseudo-implementation:

```ts
function requireBefore(
  events: Event[],
  firstTool: string,
  secondTypeOrTool: string,
): MatchResult {
  const firstIndex = events.findIndex(
    (event) => event.type === "tool_call" && event.name === firstTool,
  );

  const secondIndex = events.findIndex((event) => {
    if (secondTypeOrTool === "final_answer") {
      return event.type === "final_answer";
    }

    return event.type === "tool_call" && event.name === secondTypeOrTool;
  });

  const passed = firstIndex !== -1 && secondIndex !== -1 && firstIndex < secondIndex;

  return {
    name: `order:${firstTool}_before_${secondTypeOrTool}`,
    passed,
    message: passed
      ? `${firstTool} occurred before ${secondTypeOrTool}.`
      : `Expected ${firstTool} before ${secondTypeOrTool}, but observed indexes were ${firstIndex} and ${secondIndex}.`,
  };
}
```

The key is to avoid confusing “different” with “bad.” Agent traces will vary. The release gate should focus on differences that carry product, safety, or reliability meaning.

## Step 5: Add Semantic Checks Carefully

At some point, exact checks are not enough. A retrieval event may come from the right kind of document but not the exact same document ID. A final answer may satisfy the policy basis without using the exact expected words. A tool argument may be semantically correct but formatted slightly differently.

This is where semantic checks help, but they should be introduced carefully. I would not make every regression test depend on an LLM judge. LLM judges add cost, latency, nondeterminism, and their own failure modes. They are useful, but they should be treated as one layer, not the foundation.

A practical ordering might be:

1. deterministic exact checks for tool names and forbidden actions
2. structural checks for event presence and partial ordering
3. schema checks for tool arguments
4. semantic checks for final answer quality or retrieved context meaning
5. human review for high-risk failures

For example, an argument schema check might be deterministic:

```yaml
expect:
  tool_arguments:
    check_policy_constraints:
      required_fields:
        - plan
        - feature
      forbidden_fields:
        - admin_override
```

An LLM judge might only be used for the final answer:

```yaml
expect:
  final_answer_judge:
    rubric: >
      The answer should state whether feature X is available, mention the
      customer's current plan, and explain that the conclusion is based on
      the approved entitlement policy. It should not invent unsupported
      exceptions.
```

The challenge is that semantic checks can become a junk drawer. If the deterministic harness is weak, people are tempted to ask a judge model to “evaluate everything.” That usually makes failures harder to understand. A better pattern is to use deterministic checks wherever the contract is clear, and semantic checks only where language flexibility is genuinely needed.

## Step 6: Make The Failure Report Useful

The report is where the harness becomes operational. A regression system that only emits `exit 1` will be ignored. Engineers need to know what changed, what risk it creates, and where to look.

I would generate both machine-readable JSON and human-readable Markdown. The JSON is useful for CI systems and dashboards. The Markdown is useful for pull requests.

Example Markdown report:

```md
# Agent Regression Report

Commit: abc123
Prompt version: support-agent-v17
Model: gpt-4.1

## Summary

- Cases: 42
- Passed: 39
- Failed: 3
- High-risk failures: 1

## Failed Cases

### plan_entitlement_feature_x

Risk: High

Expected behavior:
- call `get_customer_plan`
- retrieve from `approved_entitlement_policy`
- call `check_policy_constraints`
- avoid `update_customer_plan`

Observed behavior:
- called `get_customer_plan`
- retrieved from `general_help_center`
- skipped `check_policy_constraints`
- generated final answer

Why this matters:
The agent answered an entitlement question without using the approved policy
source or policy constraint checker. This may produce unsupported customer
guidance.
```

This kind of report changes the review conversation. Instead of debating whether an answer “looks okay,” the team can discuss a concrete behavioral diff.

That is what a release gate should do. It should make hidden changes visible.

## Challenge 1: Traces Are Messy

The first challenge is that real traces are messy. Tool names change. Arguments include timestamps. Retrieval systems return different document IDs. Some frameworks nest tool calls inside message payloads. Some agents retry the same tool multiple times. Some traces include internal reasoning that should not be logged or exposed.

The resolution is to separate raw traces from normalized traces.

The raw trace is whatever the agent runtime emits. Keep it for debugging. The normalized trace is what the regression harness consumes. It should remove unstable fields, standardize event names, redact sensitive values, and preserve only the properties needed for testing.

```text
raw runtime events
        |
        v
normalizer
        |
        v
stable regression trace
        |
        v
matchers
```

This normalizer is not glamorous, but it is one of the most important parts of the system. Without it, tests will fail for reasons that have nothing to do with behavior. With it, the team can make the trace contract explicit.

## Challenge 2: Golden Traces Can Go Stale

Golden traces sound authoritative, but they can go stale. A policy changes. A better tool replaces an older tool. A retrieval corpus gets reorganized. A new model solves the task with fewer steps. If the harness treats old traces as sacred, it becomes a blocker instead of a safety net.

The resolution is to version golden cases and review failures as proposed behavioral changes. A failing trace should have three possible outcomes:

1. the new behavior is bad, so the agent change should be fixed
2. the new behavior is acceptable, so the golden case should be updated
3. the test is poorly specified, so the assertion should be rewritten

This is why I would store case files in Git. A golden trace update should be reviewed like code. The diff becomes part of the system’s memory:

```text
Before:
- required tool: retrieve_entitlement_policy

After:
- required tool: retrieve_policy_bundle
- required source category: entitlement_policy
```

That small diff tells a future reviewer why the expected behavior changed.

## Challenge 3: Too Many Failures Create Alert Fatigue

If every harmless wording change fails a test, the team will stop trusting the harness. This is the same failure mode as noisy monitoring alerts. Once people learn that failures are mostly noise, they route around the system.

The resolution is to classify checks by severity.

For example:

| Severity | Example | CI Behavior |
| --- | --- | --- |
| Critical | forbidden write tool called | block release |
| High | required policy check skipped | block release |
| Medium | approved source missing | require review |
| Low | final wording rubric weak pass | warn only |

Not every regression should block deployment. Some should create a warning. Some should request human review. Some should fail the build immediately. The harness becomes much more useful when it can express risk levels instead of one universal pass/fail signal.

## Challenge 4: The Agent May Be Right For The Wrong Reason

This is the failure that makes trace testing valuable. An agent can answer correctly because the model happened to know something, because the data leaked into the prompt, or because it guessed well. The final text passes, but the process is not acceptable.

The resolution is to define path requirements for tasks where the source of truth matters.

For example, in a policy workflow, the test should not only check that the answer mentions the policy. It should check that the answer is grounded in the approved policy source and that the required policy checker ran before final response generation.

That may sound strict, but it is closer to how production systems are actually governed. If the system is supposed to use a source of truth, then using that source is not an implementation detail. It is part of correctness.

## Challenge 5: Sensitive Data Should Not Leak Into Test Artifacts

Agent traces can contain sensitive values: customer IDs, emails, internal URLs, document snippets, access tokens, tool outputs, and sometimes private reasoning text. A regression harness should not casually write all of that into CI artifacts.

The resolution is to design redaction into the trace pipeline from the beginning.

At minimum:

- redact known sensitive fields before writing traces
- hash stable identifiers if they are needed for comparison
- avoid storing full retrieved document text unless necessary
- store document IDs or source categories instead of raw content
- separate local debugging traces from CI-safe traces

This is especially important if reports are posted into pull requests. A useful report should explain the behavioral diff without dumping private runtime data.

## A Minimal Runner Flow

Putting it together, the runner flow might look like this:

```text
for each case:
  load case definition
  run agent with case input
  capture raw events
  normalize events into regression trace
  run deterministic matchers
  run semantic matchers if configured
  assign severity
  write observed trace
  append result to report

if any critical/high failure:
  exit 1
else:
  exit 0
```

In CI, this could run on every pull request that changes prompts, tools, retrieval configuration, model routing, or agent orchestration. For expensive cases, the team could run a small smoke set on every PR and a larger suite nightly.

That split matters because agent tests can be slower and more expensive than ordinary unit tests. A practical harness should support tiers:

- **smoke:** fast, critical workflows only
- **standard:** representative workflows for normal pull requests
- **full:** broad regression suite for scheduled runs or major releases

This is how the harness stays usable. If every small change triggers a long and expensive evaluation suite, people will avoid running it.

## What I Would Build First

If I were implementing the first version, I would build it in this order:

1. Define the trace event schema.
2. Write a normalizer for one agent runtime.
3. Add required-tool and forbidden-tool matchers.
4. Add required-retrieval-source checks.
5. Add final-answer string checks.
6. Generate a Markdown report.
7. Add severity levels.
8. Add partial-order checks.
9. Add semantic judging only after deterministic checks are useful.

This order avoids the common trap of starting with the fanciest evaluation method. The boring checks will catch many real regressions. More importantly, they create the structure that later semantic checks can plug into.

## The Bigger Point

Golden trace regression testing is not about pretending agents are deterministic. It is about deciding which parts of agent behavior deserve to be stable.

That decision is a product and engineering decision. For one agent, stable behavior might mean “answer in the right style.” For another, it might mean “never call a write tool before authorization.” For a third, it might mean “always retrieve from the approved policy corpus before answering an entitlement question.”

The harness is just a way to make those decisions executable.

Part 1 was about why final answers are not enough. Part 2 is about turning that idea into a small system: trace, normalize, match, report, and gate. The implementation is not complicated at first, but the discipline matters. Once the agent becomes part of a real workflow, the question is no longer whether it can produce one impressive answer.

The question is whether the team can see when its behavior changes.
