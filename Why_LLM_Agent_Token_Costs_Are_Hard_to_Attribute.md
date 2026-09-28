---
title: "Why LLM Agent Token Costs Are Hard to Attribute, and What Telemetry Design Can Do About It"
published: false
description: "Design principles for attributing token cost in LLM agent applications: per-call measurement, cardinality-aware metrics, team-level rollups, and vendor-neutral OpenTelemetry pipelines."
tags: observability, opentelemetry, ai, devops
---

**AI observability** is the practice of collecting traces, metrics, and logs from LLM-backed applications so that latency, quality, and token cost can be explained per model call, per agent, and per conversation, not only per HTTP request.

*Disclosure: I work at Bonree, an observability vendor. Most of this post is vendor-neutral; Bonree ONE appears near the end.*

## The invoice tells you the total, not the cause

One user message to an agent can trigger several model calls, tool calls, and retries. Provider dashboards often aggregate by API key or project, which does not show which agent, which model choice, which team, or which conversation drove the spend. That gap is what telemetry design has to close.

## Design principles

- **Measure per model call, not per request.** A request-level number hides which step in the chain was expensive.
- **Split by cardinality.** Use metrics for totals, with a small fixed set of labels such as model, agent, team, and token type. Keep unbounded identifiers, such as conversation or user IDs, in traces, where per-conversation detail belongs. An ID used as a metric label creates a new time series for every ID.
- **Store token counts, convert to currency later.** Prices change. Raw counts stay valid, and a price table applied at analysis time keeps history correct.
- **Treat prompt and completion capture as a separate decision.** Content often contains customer data and needs its own review.
- **Know how your provider counts tokens.** Cached and reasoning tokens may be reported differently across providers, so check what each field includes before comparing numbers.
- **Make spend attributable to a team, not only a model.** Once agent, model, and token-type labels exist on the same metrics, a team-level rollup is a query, not a separate reporting pipeline, and it turns a cost question into a governance one: which teams are running agents, at what volume, and with what effect.

## Use a standard path

OpenTelemetry's GenAI semantic conventions define shared names for LLM and agent telemetry. They are still evolving, so pin versions and re-check on upgrade. Keeping collection vendor-neutral also turns the choice of backend into a configuration decision rather than an instrumentation rewrite, which matters to any DevOps or SRE team that expects to change tools someday.

## A minimal example

The snippet below is vendor-neutral OpenTelemetry code that illustrates the metrics-versus-traces split. It is **not a Bonree ONE configuration**. Tested with opentelemetry-sdk 1.44.0 and opentelemetry-semantic-conventions 0.65b0; the GenAI conventions are still evolving, so re-check attribute names when you upgrade.

```python
from opentelemetry import trace, metrics

# Assumes a TracerProvider and MeterProvider are configured elsewhere.
tracer = trace.get_tracer("agent.instrumentation")
meter = metrics.get_meter("agent.instrumentation")
token_usage = meter.create_histogram(
    "gen_ai.client.token.usage", unit="{token}",
    description="tokens per call",
)

def call_llm(client, *, model, messages, agent, team, session_id):
    with tracer.start_as_current_span(f"chat {model}") as span:
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.request.model", model)
        span.set_attribute("gen_ai.agent.name", agent)
        span.set_attribute("gen_ai.conversation.id", session_id)  # span only, never a metric label
        resp = client.chat(model=model, messages=messages)  # stand-in for your SDK call
        span.set_attribute("gen_ai.usage.input_tokens", resp.usage.input_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", resp.usage.output_tokens)
        labels = {"gen_ai.request.model": model, "gen_ai.agent.name": agent, "team": team}
        token_usage.record(resp.usage.input_tokens, {**labels, "gen_ai.token.type": "input"})
        token_usage.record(resp.usage.output_tokens, {**labels, "gen_ai.token.type": "output"})
        return resp
```

Note that the conversation ID lives on the span only, while team is a small, fixed-cardinality label, so it is safe to put on the metric. In a quick local test with a stub client and two conversations across two teams, the metric produced four series (two teams × input/output), not one per conversation.

## Where Bonree ONE fits

Bonree ONE 4.0 includes an AI Observability capability that Bonree describes as providing end-to-end tracing, span-level analysis, token usage monitoring, performance metrics, and session-level context tracking for large-model and agent environments. Bonree ONE also supports standard OpenTelemetry ingestion of traces, metrics, and logs. Whether a team-level rollup like the one above is useful on top of that depends on whether agent and team labels are already being recorded consistently, which is the instrumentation-design question this post is about, not something any specific platform does automatically. See Bonree | AI Observability and Bonree | OpenTelemetry Standard Integration, or the post Bonree | AI Observability in Practice: Instrumenting Agent Chains, Not Just API Calls. More at Bonree | Make IT Operations More Intelligent.

## A question for readers

How does your team attribute agent token spend today: per API key, per agent, per team, or per conversation?

#observability #opentelemetry #devops #Bonree
