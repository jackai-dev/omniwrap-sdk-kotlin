# Native Kotlin SDK sessions

Acquire the exact JAR and independently verify its SHA-256. Import
`dev.omniwrap.omniwrap.sdk.AgentSession` and its nested `Provider`, `Proposal`, `Call`, `Usage`, `Options`
and `Checkpoint` types. Both renderer editions retain the same package; model
collisions may change the session class name recorded in `agent-session.json`.

Implement `Provider.identity`, `version` (`omniwrap.agent-provider/v1`), suspend
`propose(context, remaining)` and suspend `close()`. Use `Proposal.calls(usage,
calls)` or `Proposal.done(usage, result)`; the latter accepts explicit JSON null.
`discovery()` and `agent-session.json` name real methods, parameter classes,
schemas and typed `propose*` helpers. Sessions invoke native suspend SDK methods.

Supply the configured client, provider, verified archive hash, explicit operation
allowlist, external checkpoint key of at least 32 bytes, result projection and
suspend write approval callback. With the installed nested types imported:

```kotlin
val limits = mapOf(
    "turns" to 8.0, "model_calls" to 8.0, "tool_calls" to 20.0,
    "tokens" to 8000.0, "cost" to 1.0, "elapsed_seconds" to 60.0,
    "concurrency" to 2.0, "input_bytes" to 1048576.0, "output_bytes" to 1048576.0)
val options = Options(archiveSha256, allowedOperationIds, limits,
    Checkpoint(checkpointPath, externalKey), projectResult, approveWrite)
val session = AgentSession(configuredClient, provider, options)
val result = session.run(jsonInput) // from your coroutine
```

Use the actual session class name above. `projectResult` returns plain JSON with
application-known secrets removed; `approveWrite(operationId, argumentsSha256)`
must approve each specifically authorized write, including declared writes on a
GET operation. Automatic universal secret detection is not provided.

All nine budgets are mandatory and positive. Whole batches validate before effects
or approval. Plain JSON uses finite numbers, safe integers, valid Unicode and at
most 64 levels. Unsupported constraints/formats, regex, model-owned header/cookie
arguments and stream/upload operations are refused. Every referenced/optional/
composition schema branch is qualified before instance matching.

SDK retries are disabled. The native JDK transport buffers each session response
within the smaller of the client limit and remaining output budget, at most 64 MiB;
the caller's shorter timeout is retained. Cancelling the coroutine cancels the
in-flight JDK future. Custom transports, providers and callbacks remain trusted
application code and must cooperate with cancellation; they are not sandboxed.
Provider usage is self-reported. Concurrent response storage scales with the
concurrency and per-call bounds. Cleanup timeout is surfaced.

Checkpoints use HMAC, revision checks, exclusive locks, atomic replacement and
hash-chained projected receipts. They bind source, archive, registry, provider,
allowlist and budgets, and preserve the wall deadline across process restarts.
Keep the permanent `.started` guard with the checkpoint. Missing state after
initialization fails closed. Power loss, hostile filesystems and entire-directory
loss are not covered. Recover an abandoned `.lock` only after its writer stopped.

Completed sessions resume without HTTP. Unresolved calls require explicit
reconciliation. Uncertain mutations raise `SessionException` with
`code=agent_session_in_doubt` and `retryable=false`, retaining the native cause.
Other refusals use `agent_session_refused`. Inspect the external system, then
`session.reconcile(callId, knownResult)` records the verified projected result
without calling the SDK. A failed checkpoint write poisons the instance; reload
after inspecting storage, then reconcile explicitly.

Verify real calls, native errors, budgets, malformed batches, storage failures,
cancellation and forced-process-kill recovery before connecting a live provider.
