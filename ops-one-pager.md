# OPS: Open Portable Session

## Executive thesis

**The Agentic session, not just the model, endpoint, or KV-cache should be the portable unit of execution to enable efficient inference routing across tiers of compute (client -> on-prem server -> frontier cloud).**

Today, the state that makes an AI agent useful is fragmented across a specific harness and runtime: conversation history, system instructions, tool definitions, plans, artifacts, workspace context, and external-resource references. 

Moving from a laptop to an on-prem cluster, escalating inference to a larger model, typically requires bespoke reconstruction of controller state and replaying the full context, adding integration complexity, inference latency, and duplicate compute cost.

**OPS proposes an open, runtime-agnostic checkpoint and resume standard for agent sessions.** An OPS snapshot captures the semantic state required to reconstruct a session at another compatible compute tier. Engine-internal state such as the KV cache is explicitly not the portability contract, it is an optional accelerator that may reduce restore latency when compatibility can be verified.

## The core idea

OPS separates **continuity** from **performance**:

1. **Semantic continuity is guaranteed through replay.** Every Every conformant OPS destination must be able to reconstruct the session from its messages, tool state, attachments, plans, workspace references, and other declared resources. This is the universal fallback and works across different models and inference engines (llamacpp, vLLM etc.).

2. **Acceleration is optional and verified.** A destination may reuse compatible KV state, discover an existing cached prefix through systems such as LMCache, or use a future cross-model mapping mechanism. If validation fails, the session still resumes through replay.

3. **Security boundaries are explicit.** Session content may move by value or content-addressed reference. Credentials, API keys, tokens, live sockets, and authoritative cache namespaces never travel. The destination reconnects resources and maps authorization under its own policies.

4. **Every degradation is visible.** Missing media, summarized history, unavailable tools, cache misses, and rejected accelerators are reported in a structured restore report rather than being silently ignored.

The architectural model:

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/6dfea5a4-3483-4244-9f2e-37e022549ebc" />


The controller or session service owns the session. The gateway continues to route. The inference engine continues to serve models. Cache systems such as LMCache continue to manage engine-native prefix state. None of those infrastructure layers is asked to understand agent plans, tools, credentials, or workspace semantics.

## Why this matters

OPS enables a unified execution fabric across device, edge, enterprise infrastructure, and frontier cloud without exposing those placement decisions to the user.

A session could begin privately on a local model, escalate to a larger cloud model for a difficult task, move to an enterprise cluster for tool access, and later return to the device while preserving the interaction’s semantic continuity. This creates several strategic benefits:

* **Infrastructure independence:** Models, serving engines, and deployment targets can evolve without trapping the user’s session in one runtime.
* **Cost, performance, and privacy optimization:** Work can move to the most appropriate execution tier while retaining context.
* **Ecosystem leverage:** Existing standards and systems remain complementary. MCP for tools, A2A for agent communication, gateways for routing, and LMCache for KV reuse while OPS fills the missing checkpoint and reconstruction layer for agent sessions.
* **A path to faster model escalation:** Exact cache reuse works where engine compatibility exists; approximate cross-model KV transfer can be added behind explicit quality and security controls.

## Adoption strategy

OPS is designed for incremental adoption.

The initial implementation requires only:

* An **exporter** in a source controller such as Lemonade.
* A lightweight **session service** that imports the checkpoint, reconstructs the session, reconnects authorized resources, and sends ordinary requests to any OpenAI-compatible backend.

The replay path requires no modifications to inference gateways, vLLM pods, or cache providers. Destination-native cache reuse can be enabled through configuration. Engine-specific acceleration profiles can then be added independently, without changing the core session format.
