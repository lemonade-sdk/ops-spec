# OPS: Open Portable Session

**Version:** 0.1

---

## 1. Purpose and Scope

OPS defines a runtime-agnostic checkpoint format and resume semantics for AI and agent sessions. A snapshot captures the session as the unit of portability: identity, conversation transcript, tool state, plan state, attachments, and workspace references. A conformant destination can reconstruct a resumable session from the snapshot and any explicitly declared external resources that it is authorized and able to resolve; engine-internal state such as the KV cache is never the contract, only an optional accelerator. OPS preserves semantic conversational continuity across controllers and runtimes; it does not promise bit-identical model outputs or identical controller behavior at the destination. Coordinated ownership handoff and live migration between two running controllers are deferred to a future extension (see 10.3).

**In scope:**
- The OPS snapshot document (data model and JSON serialization)
- Resume semantics: the mandatory replay baseline and structured restore reporting, including destination-discovered cache optimizations
- Conformance classes for producers, consumers, and accelerator consumers
- Accelerator profiles, both exact and (policy-gated) approximate, with binding validation and an activation lifecycle
- Trust-boundary and security rules, including cache-namespace isolation
- An HTTP binding with a prepared-import transaction

**Out of scope:**
- Transport security (defer to the host API's TLS and auth)
- Long-term agent memory semantics (referenced, not defined)
- Sampling/RNG state and mid-generation export (see 7.1, quiesce points)
- Coordinated ownership handoff and live migration between controllers (deferred; sketch in 10.3)
- Migration of actively executing tool calls (deferred to the handoff extension)
- Inter-agent task coordination (see A2A; OPS is complementary)

## 2. Terminology

- **Session**: The durable state of one ongoing interaction between a user (or agent) and a model-backed system.
- **Snapshot**: A serialized OPS document representing a session at a quiesce point.
- **Controller**: The component that renders prompts, dispatches tools, and owns the session loop (e.g., Lemonade, Claude Code, Hermes).
- **Serving endpoint**: The HTTP surface that terminates OPS operations. May be a controller, a session service, or an engine implementing OPS natively. Inference gateways and routers are deliberately not expected to terminate OPS.
- **Session service**: A deployment of the controller role that terminates OPS on behalf of a cluster: session ledger, prompt rendering, replay, tool reconnection, and credential resolution.
- **Engine**: The inference runtime (vLLM, llama.cpp, SGLang, TensorRT-LLM).
- **Cache provider**: A destination-side prefix-state system (e.g., LMCache) that stores and serves engine-native KV objects. A cache provider's internal request tracking is not an OPS session.
- **Replay**: Restoring a session by re-rendering the effective transcript into the destination's prompt format and prefilling normally. Replay MAY be transparently accelerated by destination-discovered cache hits; it remains replay.
- **Resume**: Creating a new live session at a destination from a checkpoint snapshot. Requires no coordination with the source. This is the core OPS operation.
- **Backend retargeting**: A controller keeping ownership of a session while pointing inference at a different engine or endpoint. Involves no OPS document on the wire.
- **Handoff**: Coordinated transfer of session ownership between two live controllers, with freeze and commit semantics. Out of scope for OPS 0.x (see 10.3).
- **Effective transcript**: The deterministic, model-visible sequence of events computed from the event log by the algorithm in 4.4.
- **Prompt view**: A producer-materialized fingerprint of the effective transcript up to a given event, used to bind accelerators (4.4).
- **Accelerator (source-carried)**: An `accelerators[]` entry originating at the source: engine state as a payload, or a hint referencing state in a shared cache provider. **Exact** entries reproduce the state replay would produce. **Approximate** entries approximate it and are policy-gated.
- **Destination-discovered optimization**: A cache hit found by the destination's own provider during replay, with no snapshot entry. Reported, never declared.
- **Accelerator consumer (OPS-A)**: A component, typically an engine adapter or cache connector, that validates bindings and activates destination-native prefix state. It owns no session semantics.
- **Quiesce point**: An event boundary at which a snapshot may be taken (no generation in flight, no unresolved dispatched tool call).
- **Restore report**: The structured result of an import (7.3), covering representation, transformations, readiness, optimizations, and accelerator attempts.

Key words MUST, MUST NOT, SHOULD, MAY are to be interpreted as in RFC 2119.

## 3. Design Principles

- **P1. Replay is the mandatory baseline.** Every conformant consumer MUST be able to restore a session from the core layers plus resolvable declared resources via replay. Accelerators MUST NOT be required for correctness. Destination cache hits during replay are an engine skipping computation it can prove it already did; they never change replay's semantics.
- **P2. Messages, not tokens.** The transcript is stored at the message/content-block level. Token IDs MUST NOT appear in the core layers; token-level digests appear only inside prompt views and accelerator bindings. Actual token IDs MAY be exchanged with a local cache provider during activation.
- **P3. Model-visible self-containment.** A producer MUST include all model-visible state required to construct the next inference request, either by value or through content-addressed attachment references: active system and developer instructions, the active toolset's schemas, response-format constraints, and any memory snippets already injected into the model context. Two conformant consumers of the same snapshot must be able to construct the same effective prompt.
- **P4. Accelerators are optional, verified, and consumer-selected.** A consumer MAY activate a source-carried entry only after its bindings validate through the lifecycle of 4.10. Selection among valid entries is the consumer's decision, informed by cost; producer array order is a preference, not a command. Destination-discovered optimizations require no entry at all. Approximate entries additionally require explicit import policy.
- **P5. Explicit trust classes.** Every piece of state is classified as by-value, by-reference, or never-travels (Section 8). Credentials never travel; capability requirements do. Cache namespaces are destination-resolved; a snapshot never carries an authoritative cache salt.
- **P6. Graceful degradation with structured reporting.** A restore that loses information or resolves resources partially MUST succeed where policy allows and MUST report what changed. Accelerator and cache failures are non-fatal but never silent: discovery hits that fail to load, evictions between discovery and activation, and rejections are all reported with reasons.
- **P7. Forward compatibility with critical extensions.** Consumers MUST ignore unknown descriptive fields. Unknown model-visible event kinds and unknown entries in `required_features` MUST cause rejection or explicit policy-approved degradation, never silent omission. Vendor extensions are namespaced (`x-org.<vendor>.*`).

## 4. Data Model

An OPS snapshot is a single JSON document plus a set of binary blobs (attachments and accelerator payloads) referenced by content digest.

### 4.1 Envelope

```json
{
  "ops_version": "0.1",
  "snapshot_id": "opsnap_8f2c...",
  "created_at": "2026-08-19T17:42:11Z",
  "producer": {
    "name": "lemonade-server",
    "version": "11.6.2",
    "conformance": ["OPS-P", "OPS-C/replay"]
  },
  "required_features": [],
  "integrity": {
    "canonicalization": "jcs",
    "algorithm": "sha-256",
    "digest": "hex digest of the JCS-canonicalized document with this field zeroed"
  },
  "session": { ... },
  "transcript": [ ... ],
  "prompt_views": [ ... ],
  "attachments": [ ... ],
  "tool_state": { ... },
  "plan_state": { ... },
  "memory": [ ... ],
  "accelerators": [ ... ]
}
```

- Canonicalization is the JSON Canonicalization Scheme (RFC 8785). Floating-point values SHOULD be avoided in hashed normative structures.
- `required_features` names critical extensions without which the snapshot cannot be interpreted safely; a consumer that does not implement a listed feature MUST reject the import.
- The integrity digest provides tamper evidence, not producer authenticity. Authenticity comes from the authenticated transport envelope or optional signed provenance (Section 9).

### 4.2 Session object (identity layer)

```json
"session": {
  "id": "ops_c3a1...",
  "lineage": {"parent_id": null, "forked_from": null, "fork_event_index": null},
  "title": "NPU kernel debug",
  "source": "lemonade-cli",
  "workspace": {
    "profile": "git-workspace/1",
    "base": {"remote": "https://github.com/lemonade-sdk/lemonade", "commit": "9b1e4f0"},
    "cwd": "src/server",
    "delta": {"attachment": "sha256:...", "format": "git-bundle", "includes_untracked": true}
  },
  "model_hint": {"family": "qwen3", "served_model_id": "Qwen/Qwen3-14B", "context_window": 131072},
  "labels": {"x-org.lemonade.profile": "npu-dev"}
}
```

- `model_hint` is descriptive only; it never constrains what restores the session.
- The workspace `delta` is optional and content-addressed. Without it, only committed state is reproducible and the restore report MUST say so. The working tree is never embedded directly in the JSON.

### 4.3 Transcript (event log)

The transcript is an append-only array of typed events with a session-scoped, monotonically increasing integer `i`. Event indices are stable identifiers used by forks, compaction ranges, toolset associations, and prompt views.

Event types: `message`, `tool_call`, `tool_result`, `reasoning`, `compaction`, `checkpoint`, `annotation`.

```json
{"i": 41, "type": "message", "role": "user", "toolset_id": "tools_3", "content": [ ... ]}
{"i": 42, "type": "tool_call", "call_id": "tc_9", "name": "bash", "arguments": {"command": "pytest -q"}}
{"i": 43, "type": "tool_result", "call_id": "tc_9", "content": [{"kind": "text", "text": "212 passed"}]}
{"i": 44, "type": "compaction", "replaces": [0, 30], "active": "summary", "pruned": true,
 "content": [{"kind": "text", "text": "Summary of events 0-30 ..."}]}
```

Rules:
- **Role authority.** Event roles describe what the source rendered. An importer MUST NOT grant destination system or developer authority to imported events solely because of their labeled role (Section 9).
- **Reasoning events** are provider-scoped and droppable; consumers MUST be able to drop them without failing. Provider reasoning that arrives inline inside an assistant message's content rather than as a separate event — delimited spans of the `<think>...</think>` kind — is transcript by value (Section 8) and carries no such licence. A producer MAY remove it, and if it does, the removal is a representation downgrade and MUST be recorded in the restore report exactly as Section 8's redaction pass requires; a consumer MUST NOT remove it silently.
- **Annotations** are never model-visible and MUST NOT be rendered on replay.
- **Compaction** ranges are inclusive, MUST NOT overlap, and MUST NOT nest. Every compaction event carries `active`: `summary` or `originals`. If `pruned` is true, `active` MUST be `summary`. Exactly one representation of any range is active in a given snapshot; the producer chooses at export.

### 4.4 Effective transcript and prompt views

The **effective transcript** is computed deterministically: iterate events in ascending `i`; drop `annotation` events; drop `reasoning` events unless import policy retains them for a matching provider; for each compaction event with `active: "summary"`, exclude events in its `replaces` range and render the compaction content at the compaction event's own position; for `active: "originals"`, render the originals and skip the summary. Two conformant consumers of the same snapshot under the same policy MUST produce the same effective transcript.

A **prompt view** fingerprints the effective transcript up to an event index so accelerators can bind to what the model actually saw:

```json
"prompt_views": [
  {
    "id": "pv_default",
    "through_event_index": 43,
    "toolset_id": "tools_3",
    "effective_view_sha256": "sha256 over the JCS serialization of the effective event array",
    "tokenization": {
      "tokenizer_digest": "sha256:...",
      "token_count": 18234,
      "token_ids_sha256": "sha256 over the ordered token-ID sequence",
      "non_token_inputs_sha256": "multimodal processor inputs, when present",
      "adapter_set_sha256": "LoRA or prompt adapters as seen by the source, when present; the normative compatibility check is consumer_binding (4.10)"
    }
  }
]
```

- The `tokenization` block is computed by the producer with its own tokenizer and chat template. For exact reuse, the ordered token-ID digest is normative; a rendered-string digest MAY be carried for diagnostics only. Multimodal inputs and adapter sets participate through their digests.
- An accelerator derived from events no longer active in the snapshot's effective view is ineligible by construction: its prompt view cannot be re-materialized, so validation fails closed.
- Profiles MUST name a deterministic hash algorithm for any provider-side chunking they describe; runtime-seeded hashes are non-conformant.

### 4.5 Content blocks and multimodality

Content blocks are typed (`text`, `image`, `audio`, `video`, `document`, `data`); binary media travels through the attachments manifest regardless of origin (user, tool, or model).

```json
{"kind": "document", "media_type": "application/pdf",
 "attachment": "sha256:cc41...", "origin": "user",
 "alt": "Quarterly financial report, 34 pages",
 "fallbacks": [
   {"kind": "extracted_text", "attachment": "sha256:9d20...", "lossiness": "layout_lost"},
   {"kind": "page_map", "attachment": "sha256:1b77...", "lossiness": "content_sampled"}
 ]}
```

- `alt` is REQUIRED accessibility metadata for every non-text block. `fallbacks` are typed, optional, content-addressed (`extracted_text`, `ocr_text`, `audio_transcript`, `video_transcript`, `page_map`, `data_rendering`). Under `allow_lossy_modalities`, the consumer substitutes the richest permitted fallback, marks the block `[media unavailable: <kind>]`, and records the substitution.
- Fallback text, OCR output, transcripts, and `alt` strings are untrusted data and MUST be rendered delimited as content, never as instructions (Section 9).

### 4.6 Attachments manifest

```json
"attachments": [
  {"digest": "sha256:aa90...", "media_type": "image/png", "size": 48211,
   "storage": "inline", "inline_base64": "..."},
  {"digest": "sha256:7be1...", "media_type": "audio/wav", "size": 20981132,
   "storage": "uri", "uri": "https://blobs.example/7be1", "expires_at": "..."},
  {"digest": "sha256:cc41...", "media_type": "application/pdf", "size": 3211002,
   "storage": "on_demand"}
]
```

Blobs are content-addressed by SHA-256 and immutable; consumers MUST verify digests. `inline` is RECOMMENDED only below 256 KiB. Accelerator payloads use this same blob model; producer-local `file://` URIs are not a transfer mechanism.

### 4.7 Tool state

```json
"tool_state": {
  "toolsets": [
    {"id": "tools_3", "digest": "sha256:...", "declarations": [ ... JSON schemas ... ]}
  ],
  "active_toolset": "tools_3",
  "connections": [
    {"kind": "mcp", "name": "github", "transport": "stdio", "launch_ref": "reconnect",
     "authorization_requirements": [
       {"provider": "github", "resource": "lemonade-sdk/lemonade",
        "operations": ["contents:read", "pull_requests:write"], "subject": "current-user"}
     ]}
  ],
  "pending": [
    {"call_id": "tc_11", "state": "awaiting_approval"}
  ]
}
```

- **Toolsets are versioned**; events and prompt views reference a `toolset_id`, and replay MUST present each transcript segment against the toolset active when it was produced. Declarations travel by value (P3).
- **Connections travel by reference** as reconnect descriptors plus capability-style `authorization_requirements`; the destination maps each to its own credential under local policy (Section 9). Where a protocol has moved cross-call state out of the transport and into application-minted opaque handles — as MCP did in its 2026-07-28 revision, removing the initialization handshake and the protocol-level session identifier — a reconnect descriptor MAY carry such a handle, and a consumer MUST treat it as an untrusted opaque string: it is never a credential, never an authorization boundary, and a destination that cannot resolve it reconnects from the descriptor instead and reports the connection partially resolved. How such handles survive a session changing hosts is unspecified by those protocols and is a candidate for a future extension.
- **Pending states permitted:** `not_dispatched`, `awaiting_approval`, `awaiting_user_input`, `abandoned`. A resumable snapshot MUST NOT contain a dispatched tool call whose result has not been durably recorded, and a consumer MUST NOT automatically reissue an unresolved call. Migration of actively executing calls belongs to the handoff extension.

### 4.8 Plan state

```json
"plan_state": {"todos": [{"id": "t1", "text": "Reproduce the crash on Strix", "status": "done"}]}
```

Minimal by design; richer planning structures go under vendor extensions.

### 4.9 Memory

```json
"memory": [
  {"kind": "file", "ref": "memory/projects/lemonade.md", "by": "reference"},
  {"kind": "store", "ref": "honcho://user/abc", "by": "reference"}
]
```

OPS points at memory; it does not define memory. Snippets already injected into the model context are, by P3, materialized in the transcript.

### 4.10 Accelerators (source-carried)

`accelerators[]` carries only state or references originating at the source. Optimizations the destination discovers on its own never appear in the snapshot; they appear in the restore report (7.3).

```json
"accelerators": [
  {
    "profile": "lmcache-prefix/1",
    "class": "exact",
    "origin": "source-hint",
    "prompt_identity": {"prompt_view": "pv_default"},
    "provider_binding": {
      "provider": "lmcache",
      "addressing": "rolling-prefix-chunks",
      "hash_algorithm": "named-deterministic",
      "chunk_size": 256,
      "provider_epoch": "opaque nonce observed at publish time; compared at discover",
      "namespace_ref": {"kind": "destination-resolved", "isolation_scope": "authenticated-principal"}
    },
    "consumer_binding": {
      "engine_adapter": "vllm-kvconnector-v1",
      "model_artifact_sha256": "sha256:...",
      "tokenizer_sha256": "sha256:...",
      "kv_dtype": "bf16",
      "parallelism": {"tp": 1, "pp": 1},
      "positional_encoding": {"type": "yarn", "factor": 4.0, "original_max_position": 32768, "effective_max_position": 131072},
      "adapter_set_sha256": null,
      "cache_abi_sha256": "sha256:..."
    },
    "coverage": {"kind": "contiguous-prefix"},
    "representation": {"codec": "raw-or-lossless", "codec_digest": "sha256:...", "equivalence": "exact"},
    "payload": null,
    "cost_hints": {"payload_bytes": 0, "locality": "shared-l2"}
  }
]
```

Structure (every profile fills all five blocks):
- **prompt_identity**: does this state correspond to the prompt the destination will actually run? Resolves through a prompt view; exact profiles validate the token-ID digest (plus non-token input and adapter digests when present).
- **provider_binding**: how the state is located in the cache provider (addressing scheme, deterministic hash configuration, destination-resolved namespace). A snapshot never carries an authoritative cache salt.
  - `provider_epoch` is OPTIONAL and, where a provider exposes one, carries the opaque reset nonce the source observed when the objects were published; the consumer compares it at discover and treats a mismatch as `miss` with reason `provider_restarted`. Every other field in this block and in `consumer_binding` is static identity, while Section 8 concedes that provider eviction is always permitted — so without an epoch a reference that resolves to nothing after a provider restart is indistinguishable from an ordinary eviction, and Section 9 item 10's requirement to record an attempt reason is not computable.
- **consumer_binding**: whether the destination engine can safely materialize the state (engine adapter, model artifact and tokenizer digests, KV dtype, parallelism, positional-encoding configuration, adapter-set digest, cache-ABI digest). Descriptive model names are operational keys, not compatibility proof; digests are.
  - `positional_encoding` is REQUIRED for exact profiles: type, scaling factor, original and effective maximum position, and activation threshold where the scheme has one. Two destinations serving the identical model artifact under different RoPE scaling or YaRN activation produce mutually invalid K values while every other field in this block matches, so omitting it lets an exact entry validate and load unusable state. A pinned engine build does not mitigate this, because the setting is a launch-time flag rather than a property of the build.
  - `adapter_set_sha256` is REQUIRED for exact profiles and is **present-or-explicitly-null**: `null` asserts that no adapters were active, and absence of the key is a validation failure rather than an assertion. Adapters change the KV values the engine produces, not the tokenization, so the authoritative check belongs here; the digest carried in a prompt view's `tokenization` block (4.4) records what the source saw and is diagnostic for this purpose. Without the explicit-null rule a destination serving the base model validates cleanly against a snapshot whose adapter digest is simply missing.
  - `parallelism` carries tensor and pipeline degree. Data and expert parallel degree are deliberately excluded: neither shards KV — data parallelism replicates whole engine instances each holding a complete cache, and expert parallelism shards feed-forward experts while attention KV is untouched — so binding on them would make conformant destinations reject compatible state. The axis that does shard along the sequence dimension is decode or context parallel degree, which a profile MAY add.
- **coverage**: which part of the model input can be restored. 0.1 normatively defines `contiguous-prefix`; the field is extensible (`segments`, `sparse`) for future profiles.
  - **Complete coverage still reserves a suffix.** An entry whose coverage spans the entire prompt view MUST be treated as `ready-partial` with the final granularity unit withheld: engines need at least one token recomputed to produce logits, and a block-aligned destination may need the whole final block. `suffix_prefill_tokens` MUST therefore be at least 1, and at least `granularity_tokens` where the profile declares a granularity. This is the normal case for session resume rather than an edge case, because a resumed session's natural prompt view runs through its last event.
  - **Single-group scope.** 0.1 coverage is defined for configurations in which the request's KV lives in one cache group. Under a hybrid allocator a request's KV is split across groups with different block sizes and layouts, and a hit covering the full-attention group but not the sliding-window group is representable in the engine and not representable in this block's scalar token counts. A profile targeting hybrid or sliding-window models MUST extend `coverage` to be expressible per group and MUST say so.
- **representation**: the storage codec and its equivalence. Exact-class entries MUST use lossless codecs (encryption is lossless; lossy quantized storage such as FP8 codecs is not exact and belongs to an approximate profile).

**`cache_abi_sha256` (normative preimage).** The cache-ABI digest is computed over the binary interface of the objects **as published**, not over the engine's version string: block size, KV cache layout (for example NHD versus HND), the layout requirement the active attention backend imposes, the cross-layer-blocks setting and its stride order, KV dtype, and an engine build identity. The source can compute it because its own engine wrote those objects; the destination recomputes it over its own configuration and compares, failing closed on mismatch. Naming the preimage matters more than its exact membership: two implementations that hash different inputs will both believe they are conformant while accepting state the other would reject. A profile MAY extend the preimage and MUST say so; it MUST NOT narrow it.

Entries carry either a content-addressed `payload` blob (`origin: "source-payload"`) or a destination-resolvable logical reference (`origin: "source-hint"`). There is no `required` field: under P1 no accelerator is ever required.

**Activation lifecycle (normative).** For each candidate entry the consumer proceeds: discover, prepare, then one of ready-full, ready-partial, miss, or rejected; then either activate, reaching loaded or load-failed, or `declined`; then release. `declined` is a terminal outcome for an entry that was ready and valid and that the consumer chose on cost not to activate, and it carries a REQUIRED reason. It is distinct from `rejected`, which means the entry could not be safely used at all: conflating a cost decision with a compatibility failure makes the two indistinguishable to an operator, and leaves the cost-selection behaviour this section requires unreportable in the restore report's own vocabulary. A consumer MAY additionally operate in a discovery-only posture, running validation and reporting the outcome it would have reached while activating nothing; this is the cheapest first deployment of an accelerator profile and it produces the telemetry needed to justify a later one. A successful lookup does not guarantee a successful load: objects can be evicted between discovery and activation, tier promotion can fail, and preparation can cost more than recomputation. Every outcome short of loaded falls back (fully or for the uncovered suffix) to replay, non-fatally but observably: the attempt and its reason land in the restore report. Selection among ready entries is consumer-authoritative, informed by `cost_hints` and measured link speed; a multi-gigabyte payload over a slow link SHOULD lose to replay.

**Approximate class.** Entries with `class: "approximate"` MUST NOT be used unless import policy sets `allow_approximate_accelerators: true`, MUST ship per-pair validation evidence per their profile, and MUST surface in `restore_method`. Approximate state and everything derived from it MUST be quarantined in a session-scoped approximate namespace (Section 9, item 6). In 0.1 exactly one approximate profile is registered: `kv-mapped/1` (Appendix A).

## 5. Conformance Classes

- **OPS-P (producer):** exports valid snapshots at quiesce points, satisfying P3 self-containment.
- **OPS-C/replay (consumer):** imports and resumes via replay from the effective transcript. The minimum bar; requires no engine or gateway changes anywhere. Destination-discovered cache acceleration requires no additional conformance.
- **OPS-C/accelerated (consumer):** OPS-C/replay plus orchestration of source-carried entries through the activation lifecycle, with consumer-side selection, fallback, and reporting.
- **OPS-A/\<profile\> (accelerator consumer):** validates one profile's bindings and activates destination-native prefix state. Owns no transcript, tools, plans, or session lifecycle. For `lmcache-prefix/1`, LMCache is the provider and the vLLM KV connector is the activation adapter; neither ever terminates OPS.

**Deployment condition (normative).** P1's fallback guarantee is delivered by destination configuration, not by the format. An OPS-A deployment MUST be configured so that accelerator load failure degrades to recomputation rather than request failure, and the restore report MUST record the configured failure policy. This is not hypothetical: vLLM's `kv_load_failure_policy` defaults to `fail`, which terminates the request when a load fails; `recompute`, paired with invalid-block reporting through `get_block_ids_with_load_errors()`, is the setting that satisfies P1. A deployment can satisfy every binding requirement in Appendix A, leave this value at its default, and violate P1 on the first evicted object.

## 6. HTTP Binding: the prepared-import transaction

```
GET  /v1/ops/capabilities
  -> {"conformance": ["OPS-C/replay", "OPS-C/accelerated"],
      "accelerator_profiles": ["lmcache-prefix/1"],
      "modalities": ["text", "image"], "max_snapshot_bytes": ...}

POST /v1/ops/imports            (body: snapshot + policy)
  -> 201 {"import_id": "imp_01", "missing_blobs": ["sha256:..."],
          "unresolved_requirements": [ ... ], "preliminary_restore_report": { ... }}

PUT  /v1/ops/imports/{import_id}/blobs/{digest}

POST /v1/ops/imports/{import_id}:validate
  -> compatibility and readiness report (no state change)

POST /v1/ops/imports/{import_id}:finalize
  -> {"session_id": "...", "restore_report": { ... }}

DELETE /v1/ops/imports/{import_id}
  -> abort; temporary blobs and staged accelerator state are released
```

Import policy:

```json
"policy": {
  "allow_lossy_modalities": true,
  "allow_compaction": false,
  "allow_approximate_accelerators": false,
  "on_context_overflow": "reject",
  "missing_resource_action": "report"
}
```

- `on_context_overflow` is one of `reject`, `compact`, `truncate`; any transformation is recorded, never silent.
- **Idempotency.** Within an authorization scope, reusing a `snapshot_id` with the same integrity digest returns the existing import resource; a different digest returns 409 Conflict. Partial uploads resume by re-listing `missing_blobs`.

## 7. Resume Semantics (normative)

### 7.1 Quiesce and export

A producer MUST export only at a quiesce point: no generation in flight and no dispatched tool call with an unrecorded result (4.7).

### 7.2 Restore algorithm

1. Validate the envelope: `ops_version`, `required_features`, integrity digest.
2. Resolve attachments (verify digests; obtain `uri` and `on_demand` blobs through the transaction).
3. Compute the effective transcript per 4.4 under the import policy.
4. Apply modality policy: substitute typed fallbacks where required; record substitutions.
5. Check the context window against the rendered effective prompt; apply `on_context_overflow`.
6. Re-render with the destination's chat template and per-segment toolsets; tokenize for the selected destination model. The destination model MUST be fixed for the duration of the import: steps 5 and 6 both resolve against one context window, tokenizer, and chat template. A destination addressed by a routing collection or other virtual model that re-resolves per turn MAY still serve the restored session, but it MUST fix a concrete model for the import and MUST treat any later re-resolution as invalidating every accelerator binding validated here, since prompt identity was proved against one tokenizer and one rendering.
7. Source-carried accelerator phase: run the activation lifecycle (discover, prepare, activate, release) over candidate entries; validate prompt identity by token-ID digest for exact entries; select among ready entries by consumer policy and cost; load into destination-owned slots so that a failed load falls back to replay for the affected span without contaminating reusable cache (Section 9, item 7); record every attempt and outcome.
8. Optional destination preparation: the consumer MAY ask its cache provider to prepare any already-existing compatible prefix for the tokenized prompt (for LMCache: an asynchronous lookup and optional L2-to-L1 warm prefetch keyed by the destination token sequence, the active model layout, and the destination-resolved namespace). Preparation promotes existing objects only; it never creates KV from a transcript.
9. Prefill by replay for everything not covered by a loaded accelerator. Destination-discovered cache hits during this prefill are recorded as optimizations; matching and loading remain the engine connector's responsibility during scheduling. Successful prefill may populate the destination cache for subsequent turns.
10. Resolve tool connections from `authorization_requirements` under local policy; report unresolved requirements. Never auto-reissue pending calls.
11. Materialize the workspace (base, plus delta when present and permitted); report its state.
12. Mark the session live and return the restore report.

### 7.3 Restore report

```json
{
  "status": "ready",
  "representation_fidelity": "lossless-representation",
  "restore_method": "replay-with-destination-cache",
  "transformations": [],
  "representation": {"transcript": "full", "compaction": "originals", "reasoning": "omitted"},
  "prompt_transform": {"model_changed": true, "chat_template_changed": true},
  "modalities": {"image": "native", "audio": "transcript_fallback"},
  "tools": {"declarations": "exact", "connections": "partially_resolved"},
  "workspace": {"state": "base_plus_delta", "dirty_changes_restored": true},
  "readiness": {"model_turn": true, "tool_execution": false},
  "unresolved_requirements": [
    {"kind": "authorization", "provider": "github", "resource": "lemonade-sdk/lemonade"}
  ],
  "optimizations": [
    {"provider": "lmcache", "origin": "destination-discovered", "outcome": "partial-hit",
     "requested_tokens": 18234, "matched_tokens": 18176, "suffix_prefill_tokens": 58,
     "granularity_tokens": 256, "source_tier": "l2", "prepare_ms": 31, "load_ms": 18}
  ],
  "accelerator_attempts": [
    {"profile": "llamacpp-slot/1", "origin": "source-payload",
     "outcome": "rejected", "reason": "consumer_binding_mismatch"}
  ]
}
```

- `restore_method` is one of `replay`, `replay-with-destination-cache`, `source-accelerated-exact`, `kv-mapped`.
- `optimizations` reports destination-discovered acceleration; `accelerator_attempts` reports the lifecycle outcome of every source-carried entry, including partial coverage (requested versus matched tokens, suffix prefilled, granularity, tier, preparation and load latency). Outcomes are the terminal states of 4.10's activation lifecycle, and `declined` — ready, valid, and not activated on cost — MUST be reported distinctly from `rejected`, which means the entry could not be used safely. Readers of this report act on that difference: a `declined` entry says the destination is working correctly and chose replay, while a `rejected` entry says a binding did not hold.
- Readiness is orthogonal to representation: a restore can be representationally lossless while operationally not ready. Callers act on readiness; auditors act on representation. `lossless-representation` replaces the earlier "F0 exact" naming.

## 8. Trust-Boundary Rules

- **By value:** transcript, prompt views, plan state, toolset declarations, small attachments, identity metadata, compaction summaries.
- **By reference:** workspace base and delta blob, large attachments, memory stores, tool connections (reconnect descriptors plus authorization requirements), accelerator payloads and cache references.
- **Never travels:** credentials, API keys, OAuth tokens, session cookies, raw sockets, process handles, authoritative cache salts. Snapshots carry capability requirements and destination-resolved namespace references only.

Cache-object ownership: an OPS snapshot does not own cache-provider objects it references. Importing or deleting a session does not delete them; `expires_at` on any reference is an availability hint, not a guarantee; provider eviction is always permitted.

Producers SHOULD offer a redaction pass for exports leaving the local trust domain: strip `reasoning` and `annotation` events, drop oversized attachments, optionally strip or generalize system instructions, accepting the representation downgrade. P3 makes the system prompt part of the snapshot; treat the snapshot as a security object.

## 9. Security Considerations

1. **Authenticity versus integrity.** The integrity digest detects modification, not provenance. Deployments MUST rely on an authenticated transport envelope and MAY add signed provenance over the canonical digest.
2. **Authority mapping.** Imported `system`/`developer` events are historical facts about the source rendering; an importer MUST NOT grant them destination authority automatically.
3. **URI fetching.** Blob and reference resolution MUST restrict schemes, forbid redirects into private networks, and enforce content-length, decompression, and timeout limits.
4. **Credential resolution.** Authorization requirements map to destination credentials only under local policy and explicit approval. Symbolic environment-variable names are not an authorization mechanism.
5. **Cache namespace isolation.** Prefix and KV state derived from one principal's prompts MUST NOT be resolvable from another principal's scope. Namespaces are destination-resolved: the destination maps the authenticated source or session principal onto its own cache namespace (provider salt mechanisms are the natural implementation). A snapshot-supplied salt MUST be ignored; honoring it would let a malicious snapshot request another tenant's cached prefixes. Empty or global namespaces MUST NOT be the default in multi-tenant deployments.
6. **Approximate-state quarantine.** State produced by an approximate accelerator, and all cache state derived from it through the engine's save path, MUST be published and resolvable only under a session-scoped approximate namespace that only the importing session (which opted in via policy) resolves. Quarantine is enforced by key construction, not convention: no unrelated request may ever hit approximate state, and the restore report can never misreport it as a native cache hit. The requirement covers **every keyed tier the destination operates** — the external cache provider, the engine's own prefix cache, and any cluster-level KV-event or routing index — and the consumer MUST record, per tier, the mechanism that enforces it. Two legs are distinct, and only the first is under producer control: objects an OPS mapper service publishes are keyed at publish time, while state the engine derives during prefill is committed under the destination's ordinary block identity. A destination that cannot suppress or re-key the derived leg does not satisfy this item and MUST NOT accept approximate entries.
7. **Accelerator staging.** Binary payloads and externally published objects MUST be validated before use, and a failed, partial, or corrupt activation MUST be recoverable to replay for the affected span without publishing the failed state as reusable cache; every such outcome MUST be reported as `load-failed` with a reason. Realization is recipient-owned: activation loads into destination-owned slots, never into state another request can reach ahead of validation. Stage-then-commit atomicity is deliberately not required, because it is stronger than shipping engines provide — vLLM may allocate and cache-commit blocks before a load failure is known, and recovers by reporting invalid blocks and rescheduling. The invariant is recoverability and non-contamination, not atomicity.
8. **Fallback injection.** `alt`, OCR, transcripts, and extracted text are untrusted data, rendered delimited as content, never as instructions.
9. **Identifier shape.** Every identifier a consumer takes from a snapshot and forwards or stores — `session.id`, `snapshot_id`, label and vendor-extension keys, `namespace_ref`, and any tool-connection handle — is attacker-controlled text in an untrusted document. A consumer MUST validate shape before use (non-empty, length-bounded, restricted character set) and MUST NOT interpolate an unvalidated value into a header, a cache key, a log line, or a filesystem path. This becomes load-bearing the moment a consumer emits session identity downstream (Section 10).
10. **Retention and deletion.** Endpoints MUST define ownership and garbage collection of imported snapshots, blobs, staged state, and session-scoped approximate namespaces, including on abort.
11. **Auditability.** Imports SHOULD record source identity, transformations, authorization mappings, optimization outcomes, and accelerator attempt reasons.

## 10. Deployment Topologies (who terminates OPS)

OPS is endpoint-neutral by design. The layer that terminates it is whichever component plays the controller role at each end; inference gateways, routers, and cache providers are deliberately not asked to. Four patterns:

- **T1. Controller-to-controller resume (symmetric).**
- **T2. Backend retargeting (no OPS on the wire).** The controller keeps ownership and re-targets inference; the next stateless request IS the replay. OPS serves as the local checkpoint format.
- **T3. Session-service resume behind a gateway.** A session service terminates OPS on the cluster side; the gateway routes. See 10.1.
- **T4. Engine-native acceleration.** Engines and cache layers implement OPS-A profiles only; they never terminate OPS imports and never own sessions.

The recommended adoption path is T2 and T3 first (no engine, gateway, or cache-provider changes for replay), T4 as the acceleration conversation.

**Emitting session identity downstream.** Engines are acquiring a request-level session coordinate with no semantics attached — vLLM merged a first-class `session_id` in 2026, and adjacent proposals add a content-derived chain position and session-addressed retention on top of it. OPS supplies the semantics such a coordinate points at, and SHOULD feed it rather than mint a competing one. A consumer that restores a session SHOULD stamp the destination's session coordinate on the inference requests it issues for that session, carried as a request header rather than a body field so the rendered request stays valid against destinations that reject unknown body keys. The value MUST be minted by the consumer and MUST NOT be copied from the imported snapshot: `session.id` is attacker-controlled text (Section 9, item 9), and a coordinate that reaches retention or namespace policy is exactly the kind of value Section 9 item 5 refuses to accept from a document.

### 10.1 Reference flow: session-service resume on Kubernetes (T3, informative)

> **On the status of 10.1 and 10.2.** Both subsections are informative and describe one deployment, yet they carry sentences that read as requirements — "affinity is a routing optimization, not a session contract", "the gateway and cache provider never see credentials, plan state, or tool state", and the build inventory in 10.2. Reviewers have repeatedly read these as MUSTs. They are not: the normative statements of the same ideas are P5 and Section 8 for what never travels, and Section 10's topology definitions for who terminates OPS. Where this reference flow and a normative section appear to disagree, the normative section governs.

The primary remote target is a Kubernetes cluster of vLLM pods with the LMCache connector, fronted by an inference gateway (llm-d's EPP, the vLLM Router, or equivalent). Gateways route with prefix-cache, load, and session-affinity scorers; the cache provider stores destination-native prefix objects. Session ownership belongs to an **OPS session service** between OPS and the gateway; it owns the ledger, prompt rendering, replay, tool reconnection, and authorization mapping. The gateway and cache provider never see credentials, plan state, or tool state.

1. The source controller completes the prepared-import transaction with the session service.
2. The session service validates, computes the effective transcript, resolves authorization requirements, renders, and tokenizes for the selected destination model.
3. Source-carried entries, if any, run the activation lifecycle. A `kv-mapped/1` entry, under policy, is transformed by the OPS mapper service and published into the cache tier under a session-scoped approximate namespace (Appendix A).
4. Optional preparation: the session service asks the cache provider to prepare any already-existing compatible prefix (for LMCache: asynchronous lookup plus L2-to-L1 warm prefetch of existing objects, keyed by destination tokens, active layout, and destination-resolved namespace). Preparation never creates KV from a transcript.
5. Replay: standard inference requests flow through the gateway to a scored pod; the engine's connector performs matching and loading during scheduling; prefill covers the remainder and populates the cache for later turns.
6. Subsequent turns use soft session affinity: affinity is a routing optimization, not a session contract.
7. On pod reselection or eviction, the destination re-discovers: GPU prefix cache, then L1, then L2/P2P, and only on a full miss or load failure does the turn fall back to full replay prefill. The session survives in every case.

**Who plays the session-service role.** Defined by function, not product: an llm-d component, a production-stack companion, an existing agent runtime, or a purpose-built shim. `vllm-project/agentic-api` is a concrete instance of the shape — it owns durable conversation state, resolves response lineage, rehydrates the logical request, and delegates rendering and inference downstream — and its identity decomposition is worth reading against 4.10, since it separates a model execution fingerprint that includes the renderer, chat template, and tool schemas, which this version's `consumer_binding` does not. A headless Lemonade deployment can fill the role as the reference implementation; an offer of working code, not an architectural requirement.

### 10.2 Block diagram: source controller to cluster to pod (informative)

Legend: **[E]** exists today. **[C]** exists but must be enabled or configured. **[N]** new work introduced by OPS.

```
LOCAL (developer machine)
┌────────────────────────────────┐
│ Lemonade (source controller,   │
│ vLLM backend)                  │
│   session state            [E] │
│   OPS exporter             [N] │
│   optional KV write-through    │
│   to shared L2           [E,C] │
└──────────┬─────────────────────┘
           │  H1: POST /v1/ops/imports  (snapshot + policy)
           │  H2: PUT blobs             (two-phase, by digest)
           ▼
REMOTE (Kubernetes cluster)
┌─────────────────────────────────────────────────────┐
│ OPS session service (controller role)           [N] │
│   prepared-import transaction + ledger          [N] │
│   binding validation (prompt identity, provider,    │
│   consumer, coverage, representation)           [N] │
│   prompt rendering + replay                     [N] │
│   authorization mapping to destination secrets  [N] │
│   OPS mapper service (kv-mapped, optional)      [N] │
│   (role adoptable by an existing stack layer;       │
│    headless Lemonade = optional reference impl)     │
└───┬──────────────────┬──────────────────┬───────────┘
    │ H3: standard     │ H4: prepare:     │ H4b: publish mapped
    │ inference        │ lookup + L2->L1  │ target-layout objects,
    │ requests         │ prefetch of      │ session-scoped approx
    │ (replay)         │ existing objects │ namespace          [N]
    ▼                  ▼                  ▼
┌──────────────────────────┐      ┌───────────────────┐
│ Inference gateway    [E] │      │ LMCache tier [E,C]│
│ (llm-d EPP/vLLM Router)  │      └─────────┬─────────┘
│   scorers: prefix-cache, │                │
│   load, soft session-    │                │ H6: connector
│   affinity         [E,C] │                │ lookup + load
│   KV-Events index    [E] │                │ during scheduling
└──────────┬───────────────┘                │           [E,C]
           │ H5: routed request             │
           ▼                                │
┌───────────────────────┐                   │
│ vLLM pod (chosen) [E] │◀──────────────────┘
│ OPS-unaware; LMCache  │
│ connector       [E,C] │
└───────────────────────┘

H7: ongoing turns: session service -> gateway -> pod, soft
    session affinity [E,C]
H8: pod reselection -> destination re-discovers cache
    (GPU prefix cache -> L1 -> L2/P2P); full replay only on
    miss or load failure [E,C]
```

Handoff inventory:

| # | From -> To | Carries | Status |
|---|---|---|---|
| H1 | Source controller -> session service | OPS snapshot + import policy | [N] on both ends |
| H2 | Source controller -> session service | blobs (attachments, accelerator payloads) | [N] |
| H3 | Session service -> gateway | rendered prompts over the standard inference API | [E], gateway unchanged |
| H4 | Session service -> cache provider | preparation: lookup and tier promotion of existing objects | [E,C] control plane; [N] the caller |
| H4b | Mapper service -> cache provider | mapped target-layout objects, approximate namespace | [N] |
| H5 | Gateway -> pod | routed inference request | [E] |
| H6 | Cache provider -> pod | connector lookup and KV load during scheduling | [E,C] |
| H7 | Session service -> gateway -> pod | ongoing turns, soft session affinity | [E,C] |
| H8 | Destination internal | cache re-discovery ladder on pod reselection; replay on miss | [E,C] |

Build inventory: in the replay path the gateway, the pods, and the cache provider need zero changes **on the pinned vLLM-plus-LMCache stack described above**, and destination-discovered acceleration is configuration there. The claim is stack-specific and does not generalize: on a llama.cpp destination the nearest equivalent is the host prompt cache, which an ordinary controller-side idle routine can erase outright. Nor does it extend to the accelerated paths — approximate materialization needs a cache-commit control that does not exist upstream today, and a gateway routing on precise prefix state needs the destination to emit cache events for objects written out of band, which it does not. The new software is the exporter in the source controller and the session service. The kv-mapped path additionally needs the OPS mapper service and version-pinned provider serialization libraries (Appendix A), pending an upstream external-ingest API. This is the demo-path claim; production additionally needs blob lifecycle, authn/authz, observability, and conformance tooling.

### 10.3 Deferred: ownership handoff and live migration (informative)

Unchanged in substance from draft.3: a future OPS Handoff extension specifies freeze at revision R, export, destination prepare, READY, commit and source stop, abort with source resumption; plus revision numbers, an ownership lease, post-R event deltas, mid-transfer input rules, and migration of actively executing tool calls. Source and destination never run the session concurrently.

## 11. Versioning and Extensions

- `ops_version` follows semver-lite: minor versions only add optional fields.
- Unknown descriptive fields are ignored; unknown model-visible event kinds are rejected or explicitly degraded under policy; unknown `required_features` entries cause rejection.
- Vendor extensions are namespaced `x-org.<vendor>.*`; critical ones register in `required_features`.
- Media type: `application/vnd.ops.v1+json` (proposed).

## 12. Prototype Plan

**Pinned demo configuration.** Local: Lemonade with the vLLM backend serving Qwen3-14B (bf16 KV) on a Ryzen AI Max machine; the second Ryzen AI Max machine over Thunderbolt 4 stands in as the fast-link "cluster" for kv-mapped, with the Kubernetes cluster as the routed target for the replay and discovery demos. Cluster: vLLM serving Qwen3-32B with the LMCache connector, llm-d-style gateway. Both models are matched-KV (8 KV heads, 128 per-head dimension) and dense full-attention, inside the validated envelope of Heo et al. (arXiv 2608.03893): the Qwen3 14B to 32B pair retains 97.6% average accuracy at k=8 (95.6% on GSM8K) with 278 ms mapped restore versus roughly 7 s re-prefill at 32K tokens. Engine builds pinned on both sides; KV dtype identical (quantized KV storage is outside both the paper's envelope and the exact class). Link-speed reality is part of the demo design: at 32K tokens the source KV is on the order of gigabytes, so kv-mapped is staged on the fast link, and slow-link runs are where consumer cost selection chooses replay on purpose.

**Phase 0: core round trip (replay only).**
- JSON Schema, JCS canonical test vectors, pydantic reference models.
- Lemonade exporter satisfying P3; `ops-restore` session service with the prepared-import transaction, effective-transcript computation, and structured restore report; owns the restored session against any OpenAI-compatible backend.
- Demonstrate Lemonade to Lemonade, and Lemonade to `ops-restore` fronting remote vLLM. No outstanding tool execution in any exported snapshot.

**Phase 1: exact path on existing infrastructure.**
- 1A, destination-discovered: populate LMCache via a normal vLLM request; import a snapshot whose effective prompt matches; route to a different pod; show the connector loading the prefix and prefilling only the suffix; report requested versus matched tokens, tier, prepare/load latency, TTFT, and `restore_method: replay-with-destination-cache`. No snapshot accelerator entry.
- 1B, disjoint fleets via shared L2: local vLLM+LMCache write-through to cluster-reachable L2, against a **same-model destination** (a second cluster deployment of Qwen3-14B, not the 32B target used elsewhere); snapshot carries an `lmcache-prefix/1` source-hint; measure fast-link versus WAN, with `accelerator_attempts` showing cost-based selection choosing replay on the slow link by design. The same-model requirement is not incidental: `lmcache-prefix/1` is exact-class, so a 14B source-hint arriving at the 32B destination MUST fail consumer-binding validation and could only ever produce `rejected` — which is 1C's result, not 1B's. Write-through is destination-mediated: the destination namespaces the write under the authenticated principal (Section 9, item 5). A direct source-to-L2 edge would have to choose its own namespace, which Section 9 forbids.
- 1C, graceful rejection: a snapshot with a `llamacpp-slot/1` payload imported at a vLLM destination; the entry is rejected on consumer binding, semantic replay succeeds, and the destination becomes progressively cache-warm. The first cross-engine resume is portable even though the accelerator is not.

**Phase 2: cross-model flagship (kv-mapped/1, Option A write-path).**
- Fit the closed-form ridge mapper per the paper's recipe (lambda 0.01, k=8, 500 FineWeb-Edu sequences of 1,024 tokens, RoPE-stripped keys) for Qwen3 14B to 32B on an MI300X node; publish whether the paper's retention reproduces on ROCm-built vLLM. Note that k is not a constant of the method: the paper sweeps it per pair over {1, 2, 4, 6, 8, 10, 12, 16, 20, 24, all} and selects 8 for this pair, so any other pair requires its own sweep. Note also what the reproduction actually spans — the published results come from an 8xH100 node running eager mode with no serving framework, and neither code nor fitted mappers were released, so this is a port across framework and vendor at once, not a rerun.
- Build the OPS mapper service: pulls source KV, applies the mapper, publishes target-layout objects into L2 using LMCache's own serialization and key-derivation libraries (LMCache as a library, version-pinned), under a session-scoped approximate namespace per Section 9 item 6.
- Demo over the Thunderbolt link: a long local agentic session on 14B escalates to 32B; mapped restore under `allow_approximate_accelerators: true` versus full replay; restore report on stage with `restore_method: kv-mapped` and the quarantined namespace visible.
- Secondary pair Qwen3 8B to 32B as the guardrail exhibit: its published 68.8% GSM8K retention is the live justification for the policy gate, evidence requirement, and fallback machinery.

**Phase 3: llama.cpp slot profile hardening (OPS-A/llamacpp-slot, same-engine only).**
- Pinned known-good commit; text-only baseline; same-process and post-restart restores; corrupt, truncated, and incompatible payload fault injection; staged loading before promotion; output-equivalence versus replay; end-to-end transfer time. Open upstream reports about slot restore behavior are verified against the pinned build before any claim is published.

**Upstream track (Option C, runs in parallel from day one).** Propose a validated external-ingest API to the LMCache project: token IDs plus layout-conformant KV bytes plus a destination-resolved namespace, with server-side checksum, layout, and namespace validation, and staged commit. The Phase 2 mapper service is the motivating reference client; if the API lands, the mapper service sheds its version-pinned internals and becomes an ordinary client.

**Research track (non-normative):** the OPS Handoff extension; read-path transform (mapper as a load-time connector plugin, with per-request policy enforcement); speculative restore (exact replay catching up through every post-checkpoint event, never switching mid-generation, policy for irreversible tool actions); cross-engine KV conversion.


## Appendix A: Registered Accelerator Profiles (initial)

- **lmcache-prefix/1 (exact, source-hint).** A shared-cache reference: the source has caused destination-native KV objects to exist in a cache tier the destination can reach (same fleet, or write-through to shared L2). No payload travels. Provider binding: LMCache addressing (rolling prefix chunks, named deterministic hash algorithm, chunk size), destination-resolved namespace. Consumer binding: engine adapter (vLLM KV connector), model artifact and tokenizer digests, KV dtype, parallelism, cache-ABI digest. Coverage: contiguous-prefix; partial hits load the matched prefix and prefill the suffix. Representation: lossless codecs only. Preparation maps to asynchronous lookup and L2-to-L1 promotion of existing objects; matching and loading happen in the engine connector during scheduling. Descriptive keys (model name, world size) are operational; digests are the compatibility proof.
- **llamacpp-slot/1 (exact, source-payload, experimental).** Source and destination are llama.cpp with compatible builds serving the identical GGUF. Payload: the slot save file as a content-addressed blob. Binding: build/cache-ABI digest (slot files are build-sensitive), KV dtype, and SWA configuration. KV dtype is named explicitly because the profile is registered exact while KV cache type is a launch-time flag on the destination, so a quantized-KV server would otherwise satisfy the binding while holding state outside the exact class. Under a multi-slot or unified-KV server, a profile instance MUST state what `contiguous-prefix` coverage is a prefix of, since a slot save is one sequence drawn from a shared pool. Same-engine only; never consumable by another engine. Status experimental pending the Phase 3 conformance results.
- **kv-mapped/1 (approximate, policy-gated).** Cross-model KV restore within a model family via a fitted per-head mapper, after Heo et al., arXiv 2608.03893 (closed-form ridge; per-target-layer top-k source-layer selection; keys mapped in RoPE-stripped content space and re-rotated with the target's RoPE). Preconditions: matched KV (same KV head count and per-head dimension), dense full attention, lossless source representation. Binding: ordered `source_binding` (validated against KV provenance) and `target_binding` (validated exactly against the destination), plus a `mapper` object: pair identifier, recipe identifier, calibration metadata, and REQUIRED per-pair retention evidence; consumers MUST treat pairs without evidence as ineligible (the source research's own failure pairs are why). **Ingest mechanism (0.1):** the OPS mapper service transforms source KV on the write path and publishes target-layout objects into the destination cache tier under a session-scoped approximate namespace (Section 9, item 6), using the provider's own serialization libraries, version-pinned. This is explicitly an interim mechanism: the standing proposal to cache providers is a validated external-ingest API (Section 12, upstream track), after which the mapper service becomes an ordinary client. **Alternative ingest (0.1, provider-neutral):** an accelerator consumer MAY instead implement the profile as an out-of-tree engine connector that loads mapped state directly into destination-owned blocks — for vLLM, a `KVConnectorBase_V1` subclass supplied through `kv_connector_module_path`. This mechanism is registered alongside the mapper service rather than beneath it: it removes the dependency on any one provider's version-pinned serialization internals, and it keeps the profile usable on destinations whose operators decline to route reuse through a specific cache provider. A connector implementing it MUST declare its layout requirement through the engine's own layout-negotiation hook rather than inferring it. A read-path transform (mapping at load time inside a connector variant) is recorded in the research track; it offers per-request policy enforcement but places research-grade compute in the serving hot path, and it SHOULD be measured before being specified. Use requires `allow_approximate_accelerators: true`; `restore_method` MUST show `kv-mapped`; mapper artifact provisioning is receiver-side in 0.1. **Quarantine posture (0.1).** Because mapped state carries the target's own token IDs, the destination engine's block hashes are the *correct* addresses for approximate values, and ordinary prefill commits derived blocks under them. A conformant 0.1 destination therefore either runs with the engine prefix cache disabled deployment-wide for the affected fleet — a real cost borne by every co-tenant, not only the importing session, and one that MUST be stated in the deployment's documentation — or can suppress exact-cache commit for the mapped span and its descendants. Per-request cache-salt scoping is deliberately not specified as an equivalent option in 0.1: it is all-or-nothing over a request's whole prefix, so it forfeits the shared-prefix reuse that `lmcache-prefix/1` and Section 7.2 step 8 depend on, and whether a given provider threads the salt into its own key derivation is deployment-specific and MUST be verified rather than assumed.

## Appendix B: Relationship to Adjacent Work

- **MCP:** tool and resource access; OPS carries reconnect descriptors and authorization requirements for MCP servers but does not overlap.
- **A2A:** standardizes communication with remote agents, task lifecycle, messages, and artifacts; it does not standardize extraction and reconstruction of a controller's internal execution checkpoint. An OPS session can be associated with an A2A context or task.
- **AG-UI:** an agent-UI event-stream protocol; an AG-UI stream can be mapped to or reconstructed in part from an OPS transcript, not necessarily losslessly.
- **LangGraph checkpointers, Agents SDK sessions, MS Agent Framework AgentSession:** framework-bound state; candidate OPS producers via adapters.
- **OpenAI Responses/Conversations, Anthropic prompt caching:** vendor-side state; non-extractable; treated as replay-only consumers.
- **LMCache and KV-layer systems (vLLM KV connectors, Mooncake, NIXL, llama.cpp slot save, TensorRT-LLM reuse):** destination-native prefix-state machinery. OPS treats LMCache as a destination-native cache discovery and loading system, not a portable KV format and not a generic cache URI; it owns no session semantics above the token sequence, though it does own tenancy and lifetime beneath it through namespace scoping, quotas, and pinning. LMCache's internal session tracking (request IDs, rolling chunk hashes, prefetch state, short TTL) is not an OPS session; OPS implementations SHOULD name such identifiers `cache_request_id` or `prefetch_job_id` to avoid conflation. Cache objects may outlive requests and be reused by unrelated authorized requests in the same namespace.
- **Cross-model KV transfer research (Heo et al., NVIDIA, arXiv 2608.03893):** the research basis for `kv-mapped/1`; its pair-dependent failures are the empirical case for the replay baseline, the policy gate, and required per-pair evidence.
- **vLLM session identity (`session_id`, and proposals for a content-derived chain position and session-addressed KV retention):** a request-level coordinate the engine carries and does not interpret. Complementary and below OPS: the engine holds an identifier with no semantics, OPS defines the session that identifier names. The stated boundary in that work — the engine materializes KV, reports what it did, and executes intent-free directives while knowing nothing about sessions — is the same boundary as T4, reached independently.
- **vLLM semantic KV cache reuse (RFC 44223):** an engine-side connector contract for non-exact external KV. Complementary and below OPS: it is the realization layer an OPS-A consumer targets, and its per-request cache-commit policy is the destination control Section 9 item 6 depends on for approximate profiles. Its compatibility dimensions and this document's `consumer_binding` were derived independently and largely agree; where they differ, the differences are listed in 4.10.
- **SGLang session control (`/open_session`, `/close_session`, session-scoped cache structures), and its wrapping by inference frameworks:** an engine-local mechanism for isolating and releasing the KV of a live, ephemeral session. It shares a word with OPS and not an object: it is engine-resident, live, and lost when the process ends, whereas an OPS session is durable, portable, and reconstructible on a runtime that never saw the original. An engine offering such an endpoint does not thereby own an OPS session; the two compose.
- **`vllm-project/agentic-api`:** a stateful agentic API layer owning conversation state, tool-call loops, and continuation in front of an engine. A candidate OPS session service (10.1) and a candidate producer.
- **Prior statements of the problem:** the portability of a session — whether a transcript can be inspected, exported, replayed elsewhere, audited, and deleted — was argued publicly before this specification was written, notably in industry commentary during 2026 on provider-bound session state. OPS is a specification of a fix for a problem others named first, and its principles P1 through P6 map onto those tests.
