# Changelog

All notable changes to agent-proto-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

The schema-nv range moves to `^0.0.2`, the version whose `SchFault`
carries the `impl Error` that `Result<_, scherror.SchFault>` has
required since SPEC § 3.4.  Nothing in this package's own interface
changed.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `agpwire` — `AgpMessage` with four variants, `AgpId` with its two
  spellings kept distinct, batches with `batch_expects_response`, and
  `originator_of` for a bidirectional protocol.
- `agpsession` — `AgpSession`, `AgpState`, `AgpAllowed`, `may_send` and
  `may_receive`, capabilities as named fields, and the negotiation
  rule with `stdlib_version` asserted against what the toolchain
  sends.
- `agptool` — `AgpTool` with its description and schema as FIELDS,
  `AgpContent`, `check_arguments` over schema-nv, and `error_result`
  so `is_error` is not the field a caller forgot.
- `agpres` — resources, templates with a refusing `expand`,
  subscriptions, and roots with `is_within_roots`.
- `agpprompt` — prompts as user-facing commands, `AgpTurn`, and
  sampling with every field a preference and `requires_approval` as a
  value.
- `agpmeta` — the eight ordered log levels, progress tokens in
  `_meta`, cancellation with `response_may_still_arrive`, and ping.
- `agpcodec` — text with no framing, version-aware decode, session-
  gated encode, and the two facts about `std.json` as functions.
- `agperror` — `AgpRpcError` and `AgpFault` as two types, the standard
  codes as functions, and conversion in one direction only.

### Known

- **`AgpSession` is the load-bearing interface.** MCP's real failure
  mode is ordering and no message type shows it; `may_send` answers
  before a byte reaches a socket, and answers two different refusals
  because they lead a caller to opposite actions.
- **One state machine, three transports.** Keeping the framing out is
  what lets the same value serve stdio, SSE and streamable HTTP.
- **There is no typed MCP value in the project today.** `std.mcp`
  answers a tool catalogue as `[Str]` with tab-separated fields, and
  `orbit/novoagent` splits them by hand. `AgpTool` is what both sides
  would share.
- **The row this opens on `std.mcp`** is one call,
  `mcp.request(h, method, params_json) -> ?Str`; every typed surface
  here is reachable through it without the runtime learning an MCP
  type.
- **novoagent's `Action` is NOT replaced.** It is a text protocol for a
  model with no tool-calling API, which is a different problem.
- **A tool that failed is a result**, not an error, because the model
  is meant to see it.
- **Cancellation is advisory** and the answer may still arrive;
  `AgpUnknownId` is not fatal.
- **schema-nv's `{}`-versus-`null` note is stale**: `json.type_of` and
  `json.is_null` have landed, so
  `agpcodec.distinguishes_empty_object_from_null` answers true and the
  test compares the claim against the library's own behaviour.
- **`AgpTurn` rather than a second `Message`**, because `agpwire`
  already owns that word for a JSON-RPC envelope.
- **No device claim and no `@value` struct**: every type here is a
  document.
- The scaffold's `src/agent_proto.nv` was dropped for eight prefixed
  modules.
