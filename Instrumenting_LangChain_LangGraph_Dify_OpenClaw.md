---
title: "Instrumenting LangChain, LangGraph, Dify, and OpenClaw: What Actually Differs at the Tracing Layer"
published: false
description: "A look at how LangChain, LangGraph, Dify, and OpenClaw expose tool calls, token usage, and multi-step execution differently, and why that makes out-of-the-box instrumentation coverage worth more than a framework checklist."
tags: observability, opentelemetry, ai, devops
---

The previous post walked through instrumenting an LLM call by hand: a span per model call, a histogram for token usage, a small set of fixed-cardinality labels. That works when there's one call site to wrap. It gets harder once a platform team is supporting several product teams that each picked a different agent framework, because **LangChain, LangGraph, Dify, and OpenClaw don't expose the same thing to instrument in the same place.**

*Disclosure: I work at Bonree, an observability vendor. Most of this post is vendor-neutral; Bonree ONE appears near the end.*

## Four frameworks, four different instrumentation surfaces

- **LangChain** runs in-process as a Python or JS library, and exposes a callback interface (handlers for on_llm_start, on_llm_end, on_tool_start, and similar events). Instrumentation attaches to these callbacks directly in the calling application's process.

- **LangGraph** adds an explicit state machine on top: nodes, edges, and checkpoints. The natural place to attach a span is per node execution and per state transition, not per model call, since a single node can make zero, one, or several model calls.

- **Dify** is mostly consumed as a hosted, low-code platform: a large part of the pipeline runs inside Dify's own engine rather than in the calling application's process, so in-process callbacks aren't available for most of it. Visibility typically comes from Dify's own execution logs or webhooks instead.

- **OpenClaw** is a standalone, always-on agent runtime rather than a library you call, so instrumentation has to happen at the boundary between it and everything else, its API or webhook surface, instead of inline in application code.

In practice, this means "add tracing" means four different things depending on the framework: wrap a callback, wrap a node function, consume an execution log, or hook a webhook. None of that is difficult in isolation. It becomes real, ongoing work when a platform team is the one expected to keep all four producing comparable data.

## Why comparable matters more than complete

It's not enough for each framework to be traced somehow. If LangChain traces carry gen_ai.agent.name and LangGraph traces carry graph.node.name with no equivalent, per-agent cost and error-rate comparisons across frameworks silently break. The fix is the same one from the previous post: standardize on a shared attribute schema, such as OpenTelemetry's GenAI semantic conventions, and map every framework's own event model onto it, rather than letting each framework's instrumentation invent its own field names.

## What automatic instrumentation is actually buying you

Given that mapping work, "out-of-the-box support for framework X" in an observability tool is worth more than it sounds like on a feature list. It's not just a connector, it's someone else having already done the work of mapping LangChain's callbacks, LangGraph's node/edge model, Dify's execution logs, and OpenClaw's API surface onto one consistent schema, so a platform team isn't building and maintaining four separate adapters by hand.

## Limitations

- All four frameworks change quickly. Automatic instrumentation still needs to be re-validated against new framework versions, the same way hand-written instrumentation does.

- A hosted platform like Dify may not expose every internal step to any external collector, in-process or not, so some signals may only ever be as detailed as what the platform's own logs provide.

- Consistent collection solves comparability, not governance. Turning collected data into per-team cost and usage rollups is the labeling and aggregation problem covered in the previous post, not something instrumentation coverage does by itself.

## Where Bonree ONE fits

Bonree ONE 4.0's AI Observability capability instruments Python, Node.js, and Java applications, with automatic adaptation for common model-native APIs and out-of-the-box support for LangChain, LangGraph, Dify, and OpenClaw, among other frameworks. See Bonree | AI Observability and the post Building AI Observability for the Native Stack: Architecture Design and Engineering Practice from Bonree ONE 4.0 - DEV Community for the underlying architecture.

## A question for readers

If you're running agents on more than one of these frameworks, how are you keeping their telemetry comparable today: a shared schema you enforce yourselves, a vendor's auto-instrumentation, or have you not tried to reconcile them yet?

#observability #opentelemetry #devops #Bonree
