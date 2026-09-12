# agent-proto-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The Model Context Protocol as novo-lang types, over the standard
library's JSON value. The protocol types `novoagent` and `std.mcp`
would share — this row exists so the two stop declaring it twice.

- `agpwire` — JSON-RPC 2.0: four message shapes, ids, batches;
- `agpsession` — the state machine as a value; **the load-bearing one**;
- `agptool` — tools, with schema-nv compiled input schemas;
- `agpres` — resources, templates, subscriptions, roots;
- `agpprompt` — prompts and sampling;
- `agpmeta` — logging, progress, cancellation, ping;
- `agpcodec` — text, and version-aware decoding;
- `agperror` — the peer's errors and this package's, kept apart.

```
novo pkg add agent-proto-nv
novo pkg build
novo test
```

## The one example that will work

A client that cannot send a request at the wrong moment, because the
session refuses it before the transport sees it.

```novo ignore
use agpsession
use agptool
use agpcodec

fn list_tools(s: AgpSession, conn: McpConn) -> Result<[AgpTool], AgpFault> [io, mcp]
    match agpsession.may_send(s, "tools/list")
        AgpNotYet(state)   => Err(AgpWrongState("tools/list", state))
        AgpNotOffered(cap) => Err(AgpWrongState("tools/list", cap))
        AgpAllow =>
            let req = agptool.list_request(AgpIdNum(next_id()), "")
            let text = agpcodec.encode_in(s, req)!
            match agpcodec.of_text(send_and_wait(conn, text))!
                AgpResult(_, r) => agptool.tools_of(r)
                AgpError(_, e)  => Err(AgpBadField("tools/list", "", e.message))
                _               => Err(AgpAmbiguousMessage)
```

## The load-bearing interface: `AgpSession`

```novo ignore
pub enum AgpState
    AgpFresh | AgpInitializing | AgpInitialized | AgpReady | AgpClosing | AgpClosed

pub enum AgpAllowed
    AgpAllow
    AgpNotYet(state: Str)
    AgpNotOffered(capability: Str)

pub fn may_send(s: AgpSession, method: Str) -> AgpAllowed
```

**MCP's real failure mode is ordering, and no message type shows it.**
A `tools/list` sent before `initialize` has been answered is a
well-formed JSON-RPC request with a valid method and valid params, and
it is wrong. A server that sends a request before the client's
`notifications/initialized` has arrived is wrong the same way. So is
anything at all after a shutdown. None of that is visible in the
message — it is visible only in what has already happened — so a
library whose whole surface is message types leaves its callers to
rediscover the sequence one hung connection at a time.

`may_send` answers before a byte reaches a socket, and it gives **two
different refusals** because they lead a caller to opposite actions:
`AgpNotYet` means finish the handshake and try again; `AgpNotOffered`
means this peer will never do it, so stop asking.

**This is why it is transport-independent.** MCP runs over stdio, over
SSE and over streamable HTTP: three framings, three sets of connection
lifecycle problems, and *one* state machine identical in all of them.
Putting it in a `core` package with no transport is what lets the same
value be checked in a unit test, driven by a subprocess client, and
reused by a server.

The negotiated version is part of the state rather than a field beside
it, because what may be sent depends on what was agreed — a capability
the peer did not declare is not available, and a method a later
revision introduced is not available at an earlier one.

**What it is not: a scheduler.** The session tracks what is outstanding
so a response can be matched and a duplicate id refused. It does not
decide what to send next, time anything out, or retry. A `core`
package has no clock, and the caller that has one is better placed.

## What novoagent and std.mcp would share, and what each keeps

This is the row's whole purpose, so it is worth being exact.

### What the standard library has today

`compiler/stdlib/mcp.nv` is `McpClient`, a struct around a runtime
handle. Its surface is:

```novo ignore
McpClient.connect(cmd, args) -> ?McpClient          [io, mcp]
McpClient.list_tools(self) -> [Str]                 [io, mcp]
McpClient.list_tools_described(self) -> [Str]       [io, mcp]
McpClient.call(self, tool, args_json) -> ?Str       [io, mcp]
McpClient.capabilities(self) -> [Str]               [io, mcp]
```

**There is no typed MCP value anywhere in the project.** A tool
catalogue comes back as `[Str]` in which each entry is
`"name \t description \t arg[*], arg"` — a serialisation the runtime
invented because there was no type to answer with — and
`orbit/novoagent`'s `render_catalogue` and `catalogue_names` recover the
fields with `str.split(entry, "\t")`. Capabilities come back as `[Str]`
too, so a caller that wrote `"tool"` for `"tools"` gets `false` with no
complaint. Arguments go out as a JSON string a caller formatted by
hand, and the result comes back as `?Str`.

### What each keeps

**`std.mcp` keeps the handle and the transport.** Subprocess lifecycle,
the wire, the `[io, mcp]` effect row — that is host work a `core`
package cannot do and should not want to. Its header already argues
that `McpClient` is deliberately not a `Connection`, because it speaks
JSON-RPC *objects* rather than a byte stream; this package is the
declaration of what those objects are.

**`novoagent` keeps its `Action`**, and that is not a concession. Its
`proto.nv` is not MCP at all: `mlserve` implements the OpenAI request
shape without `tools`, `tool_calls` or `tool_choice`, so the model
cannot be handed a schema and cannot emit a structured call, and the
whole module exists to get a reliable action out of a *plain text*
completion — balanced-brace extraction, comment stripping, fence
tolerance. None of that is replaced by a protocol type.

**What novoagent takes is the catalogue.** `render_catalogue` walks
`[AgpTool]` with `.name`, `.description` and `.required_arguments`
instead of splitting tabs; `catalogue_names` becomes
`agptool.names_of`; and `visible_tools`'s filter operates on a field
rather than on a substring of a serialised line.

**And what both take is the check.** `agptool.check_arguments` answers
schema-nv's own `SchOutput` — the instance location, the schema
location, the keyword — which is exactly the correction an agent needs
to fix its own call, and which `-32602` costs a round trip and a model
turn to learn.

### The row this opens on `std.mcp`

One call, and everything else follows from it:

```novo ignore
mcp.request(h: McpClientH, method: Str, params_json: Str) -> ?Str   [io, mcp]
```

A raw request/response over the handle the runtime already owns. With
it, `agpcodec.of_text` decodes whatever comes back and every typed
surface above is reachable without the runtime learning a single MCP
type. Without it, each new method needs a new `mcp.*` entry point and a
new hand-rolled serialisation — which is how the tab-separated
catalogue happened.

The narrower alternative — `list_tools_typed() -> [AgpTool]` — would
put a package type in the standard library, which is backwards: the
stdlib does not depend on the registry.

### The version, asserted rather than assumed

`compiler/stdlib/mcp.nv` and `compiler/mcp/novo_mcp.ml` both send
`protocolVersion: "2024-11-05"`.
`agpsession.stdlib_version()` answers that string and the test suite
asserts it, so the day one of them moves, the test fails until the
other does. `agpsession.versions_supported()` is the list this package
implements and `negotiate` is the rule between them.

## Three more decisions worth arguing

### A notification is a different value from a request

Sending a notification where a request belongs is a wait for an answer
that was never coming; sending a request where a notification belongs
leaves an id outstanding the peer will never close. `AgpMessage` makes
them different variants, so the mistake is a type error rather than a
hang, and `agpwire.expects_response` is the predicate a sender checks
before it decides to wait.

The same care goes to ids: `AgpIdNum(3)` and `AgpIdStr("3")` are
**different ids**, because an answer to one is not an answer to the
other and a library that normalised both to a string would match a
response to the wrong request in a session that used both.

And a batch of only notifications expects **no response at all** — not
an empty array. `agpwire.batch_expects_response` says so before a
caller starts waiting.

### A tool that failed is a result, not an error

A tool call that fails *inside* the tool — a command that exited
non-zero, a file that was not there — answers a normal JSON-RPC
**result** whose `isError` is true. That is deliberate in the protocol:
the model is supposed to *see* the failure and react to it, which it
cannot do if the failure travels as a protocol error. So
`AgpToolResult.is_error` carries it and `agperror` has no variant for
it, and `agptool.error_result` exists so `is_error` cannot be the field
a caller forgot.

`agperror` keeps the two vocabularies apart for the same reason: the
peer's `AgpRpcError` is part of the conversation, and this package's
`AgpFault` never reaches the peer. A library that folded them would
make "the server said your argument was invalid" and "you called a
method before initialize finished" the same shape, and a caller
retrying on both would retry the one that can never succeed.

### Cancellation is advisory, and the answer may still arrive

MCP's cancellation is a **notification**: unacknowledged, with no answer
coming. So the response to a cancelled request may still arrive,
because the server may have sent it before the notification got there
— and a client that removed the id from its outstanding set the moment
it cancelled then meets an answer it cannot attribute.
`agpsession.received` calls that `AgpUnknownId`, `agperror.is_fatal`
says false, and the right move is to drop it.
`agpmeta.response_may_still_arrive()` is the rule as a value.

`initialize` is the one request that may **not** be cancelled, and
`agpmeta.may_cancel` says so — a general cancellation path gets that
wrong and leaves a session in a state neither side can name.

## A note that turned out to be stale

schema-nv 0.0.1's manifest records, as its known limitation, that
`std.json` cannot tell `{}` from `null` — which is the `type` keyword's
whole job in a schema. This package expected to inherit it.

It does not: `json.type_of` and `json.is_null` have since landed and
answer exactly that question, and `docs/stdlib/json.md` has a
§ Telling `{}` from `null` about it. So
`agpcodec.distinguishes_empty_object_from_null()` answers **true**, and
the test asserts the package's own claim *against `json.type_of`'s
actual behaviour* rather than against a paragraph. schema-nv's note
should be retired when its bodies land.

That is the argument for publishing facts as functions: a stale
sentence in a README is invisible, and a stale assertion fails.

## What this does not do, by name

| | |
| --- | --- |
| **Transports** | stdio, SSE and streamable HTTP are the host's. `agpcodec.to_text` and `of_text` are the serialisation either side of the frame; the frame is not here, which is what lets one state machine serve three transports. |
| **Authorisation** | The OAuth flow the HTTP transport specifies is a `host` concern with `[net]` in it. Its own row. |
| **Completion** | `completion/complete`, the argument autocompletion a client offers while a person fills in a prompt. Named rather than half-declared; it is a small surface and a later revision's. |
| **Elicitation** | A server asking the user a question through the client. `AgpCapabilities.elicitation` carries the flag so a session knows whether the peer offers it; the message types are a row of their own. |
| **Timeouts and retries** | A `core` package has no clock. `agpmeta.ping_request` is the message; *when* to send one is the host's. |
| **A JSON value of its own** | `std.json`'s, deliberately — see the manifest. |

## Dependencies

| | |
| --- | --- |
| `schema-nv ^0.0.1` | A tool's input schema, compiled once and validated against. `check_arguments` answers schema-nv's own `SchOutput`, so a refusal names the instance location, the schema location and the keyword — which is what an agent needs to correct itself and what a boolean would throw away. |

Nothing else. Not an HTTP or a process package: a transport is the
host's. Not a JSON package: the value is the standard library's.

## The layer, and why

`core`, and for this package that is the whole design rather than a
budget it happens to fit. A protocol library that owned a transport
would be a protocol library a caller could only use over that
transport.

**No `@tier(embedded)` claim.** Nothing here is small: every type is a
document, there is not one `@value` struct in the package, and a device
that wanted to speak MCP would want a different, much smaller subset.
`docs/publishing.md` says a device claim is built and not asserted, and
there is no device consumer to build one for.

## The reference implementation

The Model Context Protocol schema itself — `modelcontextprotocol.io`'s
published TypeScript schema and its JSON Schema rendering. Where this
package and the schema disagree about a field, the schema is right.

The behavioural references on this machine are
`compiler/mcp/novo_mcp.ml`, which is a working MCP server, and its
shell tests, which drive a real `initialize` handshake — so the
handshake this package describes is one that already runs here.

## Status

| | |
| --- | --- |
| version | 0.0.1, `stability = "draft"` |
| modules | 8 |
| public functions | 170, every body a `todo()` |
| public types | 16 boxed structs, no `@value` structs, 10 enums with 45 variants |
| tests | 44, 291 assertions, red until bodies land |
| device claim | none, and § The layer says why |
