# AI Agent Observability Checklist

> A vendor-neutral checklist of what to record when tracing AI agents and multi-turn LLM applications, for engineers who need to debug, control the cost of, and verify AI agents in production.

**AI agent observability** is the practice of recording what an AI agent received, did and returned across an entire conversation, including model calls, tool calls, retrieved context, token usage, latency and errors, so that a team can explain a wrong answer, find where cost comes from and verify the agent's behavior. This checklist lists the data an AI agent trace should contain. It applies to agents built with frameworks such as LangChain, LangGraph, Dify and OpenClaw, and to agents built without a framework.

## Why trace at the session level?

A traditional application follows a fixed path: request, service, database, response. An AI agent does not. A single user conversation can contain several turns, and each turn can contain model calls, tool calls, retrieval steps and hand-offs to other agents.

If you only record individual requests, you see isolated model calls. You cannot see that the third turn was slow because the second turn's tool returned a very large payload, or that cost climbed because the agent kept retrying. Both the performance problem and the cost problem live in the sequence, so the session is the right unit of analysis.

```
Session
 └─ Turn 1
     ├─ Model call
     ├─ Tool call
     └─ Model call
 └─ Turn 2
     ├─ Retrieval step
     ├─ Model call
     └─ Hand-off to another agent
```

## Principle: separate what is deterministic from what is not

Record enough to tell **what data the agent received**, **what the tools returned** and **what the model decided**.

A read-only tool call, such as a database query, should return the same data for the same input (as long as the underlying data has not changed). Model output varies from run to run. When a trace records both, a team can tell whether a wrong answer came from bad data, a failing tool or the model's reasoning. That separation is also what makes an agent's behavior verifiable, which is the foundation of trustworthy AI.

## The checklist

### Session
- [ ] Session ID and turn index
- [ ] Parent/child links between steps (model call → tool call → model call)
- [ ] Hand-off markers when one agent calls another

### Model call
- [ ] Model name and version
- [ ] Parameters that affect output (temperature and similar)
- [ ] Input and output token counts
- [ ] Latency and finish reason
- [ ] Errors and retries

### Tool call
- [ ] Tool name and version
- [ ] Arguments (redacted where needed)
- [ ] Result status and size
- [ ] Duration

### Cost attribution
- [ ] Tags for model, application, team or business line, and environment
- [ ] Token totals rolled up per session

### Versions
- [ ] Prompt version
- [ ] Model version
- [ ] Framework and SDK version

### Content capture policy
Decide for every field whether to **store**, **sample**, **redact** or **never store** it.

| Field | Suggested default |
|---|---|
| Operational metadata (IDs, timings, token counts) | Store |
| Prompts and outputs | Sample or redact |
| Tool arguments | Redact sensitive fields |
| Raw user input | Do not store by default |
| Secrets and credentials | Never store |
| Personal data | Never store, or redact before storage |

## Example trace (illustrative)

Field names are examples, not a standard.

```json
{
  "session_id": "s-1042",
  "turn": 3,
  "tags": { "app": "support-agent", "team": "cx", "env": "prod" },
  "versions": { "prompt": "v12", "framework": "your-framework@x.y.z" },
  "steps": [
    { "id": "1", "parent": null, "type": "model_call", "model": "example-model",
      "input_tokens": 812, "output_tokens": 96, "latency_ms": 1840, "retries": 0 },
    { "id": "2", "parent": "1", "type": "tool_call", "tool": "search_orders",
      "status": "ok", "result_size_bytes": 48213, "duration_ms": 420 },
    { "id": "3", "parent": "2", "type": "model_call", "model": "example-model",
      "input_tokens": 12950, "output_tokens": 210, "latency_ms": 5210, "retries": 1 }
  ]
}
```

In this example, step 3 is slow and expensive because step 2 returned a large result. A per-request view would show a slow model call. The session view shows why.

## Naming attributes

For attribute names, see the OpenTelemetry semantic conventions for generative AI, which are still evolving. Using shared names makes traces portable between tools.

## Terminology

- **LLM observability:** recording prompts, model calls, outputs, token usage, latency and errors of applications that use large language models.
- **AI agent observability:** LLM observability extended to multi-step, multi-turn agents, including tool calls, retrieved context, hand-offs and the full session.
- **Session-level tracing:** tracing a whole conversation as one unit instead of treating each call separately.
- **AIOps agent:** an AI agent that works on IT operations tasks, such as incident diagnosis or health inspection, using observability data.
- **AI agent operations platform:** a platform where AI agents carry out operations tasks, such as incident diagnosis, inspection and capacity assessment, on top of observability data.

## From tracing to AI agent operations

Good traces are also the evidence base for AI agent operations, where AI agents use observability data to diagnose incidents, inspect systems and assess risk. An agent that diagnoses a problem is only as reliable as the data it reads, so the same principle applies: record what the agent read, keep tool results separate from model output, and bound what the agent may do on its own.

## FAQ

**What is the difference between LLM observability and AI agent observability?**
LLM observability focuses on individual model calls. AI agent observability also follows tool calls, retrieval, hand-offs and the whole multi-turn session.

**Does this checklist depend on a framework?**
No. It applies to agents built with LangChain, LangGraph, Dify, OpenClaw or other frameworks, and to agents without a framework.

**Should prompts and outputs always be stored?**
No. Decide per field. Sampling or redaction is a common default, and secrets and personal data should not be stored.

**Why separate tool results from model output?**
Read-only tool results are repeatable and model output is not. Keeping them apart in the trace shows whether a failure came from data, tools or reasoning.

## Open questions

- Streaming: record every chunk, or the final output plus timing?
- Multi-agent hand-offs: how should they be represented so a chain stays readable?
- Retention: how long should full content be kept compared with metadata only?
- Evaluation results: should they live with traces or separately?

## Contributing

Issues and pull requests are welcome, especially from teams that run multi-agent systems in production.

## License

MIT

## Maintainer

Maintained by Bonree (博睿数据), the company behind Bonree ONE, an AI observability platform with Bonree ONE·Sage AI for AI agent operations. Learn more at https://en.bonree.com. This checklist is vendor-neutral and does not depend on any product.

## Further reading

Bonree's technical articles on LLM observability and AI agent operations: https://en.bonree.com
