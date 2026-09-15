# agent-proto-nv

The Model Context Protocol (MCP) is an open protocol that standardizes how
applications provide context to large language models. It is specified at
[modelcontextprotocol.io](https://modelcontextprotocol.io/specification),
and it carries every message in a
[JSON-RPC 2.0](https://www.jsonrpc.org/specification) envelope. This
package brings the protocol's messages, and the rule about the order they
may be sent in, to novo-lang. It is built on
[schema-nv](https://novo-lang.org/packages/schema-nv), for the JSON Schema
a tool declares its arguments with.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What MCP is

Three parties appear in the specification. The **host** is the application
a person uses. The **client** lives inside the host and holds one
connection. The **server** is the separate program on the other end of
that connection, offering the host something it did not have. One host
runs many clients, and each client talks to one server.

A server offers three kinds of thing, and the difference between them is
about who decides. **Tools** are actions the model may choose to call, such
as running a query or sending a message. **Resources** are data the
application chooses to put in front of the model, addressed by a URI.
**Prompts** are templates a person chooses to run, surfaced as a slash
command or a menu item. A client that offered prompts to a model as if they
were tools would hand it a set of actions nobody chose.

Traffic also runs the other way. **Sampling** is a server asking the client
to run a model on its behalf, so a server can summarise or classify without
holding an API key. **Roots** are the client telling the server which
directories it may work in. **Elicitation** is a server asking the person a
question through the client.

Every message is one of four shapes, and JSON-RPC 2.0 fixes all four. A
**request** carries an id and expects an answer. A **result** and an
**error** carry that id back, and never both. A **notification** carries no
id and expects nothing at all. Sending a notification where a request
belongs is a wait for an answer that was never coming. Sending a request
where a notification belongs leaves an id the peer will never close.

A connection opens with the **initialize handshake**, described on the Base
Protocol's Lifecycle page. The client sends `initialize` with the protocol
revision it proposes and the **capabilities** it offers. The server answers
with the revision it settled on and its own capabilities. The client then
sends `notifications/initialized`, and only after that may ordinary traffic
flow. A capability is a named feature a peer declares: a method whose
capability the peer did not declare is a method that peer will never
answer.

A protocol revision is a date, such as `2024-11-05`. The client proposes
one and the server may answer with an older one. Methods and fields differ
between revisions, so the revision that was agreed decides what may be sent
and what a message means.

MCP runs over three transports: standard input and output, server-sent
events, and streamable HTTP. They frame messages differently and have one
message format and one lifecycle between them. This package contains no
transport. It performs no input or output, opens no socket, spawns no
process and consults no clock. Every function turns values into values.

| Quantity | Value |
| --- | --- |
| Message shapes in the envelope | 4 |
| JSON-RPC protocol version string on every message | `2.0` |
| Revision the standard library's `McpClient` speaks | `2024-11-05` |
| Log levels, ordered | 8 |
| Reserved JSON-RPC error code range | −32768 to −32000 |
| Parse error, invalid request, method not found | −32700, −32600, −32601 |
| Invalid params, internal error | −32602, −32603 |

## Install

```
novo pkg add agent-proto-nv
```

## Example

```novo
use agpsession
use agpcodec
use agpwire

fn main() [io]
    // What this client offers a server. A client that answers a
    // request for its roots says so here.
    let own = agpsession.with_roots(agpsession.no_capabilities(), false)

    // A fresh session. Nothing has been sent on this connection yet.
    let s = agpsession.client_session(own)

    // Ask whether a tool listing may go out now, before any byte is
    // written. It may not, and the answer says which of the two
    // reasons it is.
    match agpsession.may_send(s, "tools/list")
        AgpAllow           => println("tools/list may be sent")
        AgpNotYet(state)   => println("not yet: the session is ${state}")
        AgpNotOffered(cap) => println("this peer has no ${cap}")

    // The one request a fresh session may send, as JSON text. Framing
    // that text and writing it is the transport's work, not this
    // package's.
    let me = AgpPeerInfo { name: "example-client", version: "1.0", title: "" }
    println(agpcodec.to_text(agpsession.initialize_request(s, AgpIdNum(1), me)))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: agent-proto-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `agpwire` | The JSON-RPC 2.0 envelope: the four message shapes, the two spellings of an id, batches, and the table of method names with the side each one comes from. |
| `agpsession` | A connection's protocol state as a value: the six states, the capabilities as named fields, the revision rules, and the two calls that say what may be sent and received. |
| `agptool` | Tools: what a server offers, the content a tool answers with, and the check of an argument object against the tool's input schema. |
| `agpres` | The four things addressed by a URI: resources, resource templates, subscriptions and roots. |
| `agpprompt` | Prompts a person runs, the turns they expand to, and sampling requests a server sends to a client. |
| `agpmeta` | The traffic about the conversation: the eight log levels, progress tokens, cancellation and ping. |
| `agpcodec` | Messages as JSON text, with no framing, and the decode and encode that take the negotiated revision into account. |
| `agperror` | The peer's error object and this package's own refusals, as two separate types, with the standard JSON-RPC codes. |

## How to choose an entry point

There are three ways to turn text into a message, and they differ in how
much context they check.

**`agpcodec.of_text` reads one message and checks only its shape.** Use it
in a test, in a capture reader, and anywhere no session exists.

**`agpcodec.decode_for` also takes the negotiated revision.** It refuses a
method that revision does not have, which is a peer sending something it
agreed not to. Use it when you track the revision yourself.

**`agpcodec.decode_in` takes the session.** It reads the revision off the
session and also applies `agpsession.may_receive`, so a message that is
well formed and arrived at the wrong moment is refused here rather than
further up. Use it in any program that owns a connection.
`agpcodec.encode_in` is the same call outbound, and it applies
`agpsession.may_send`.

**`agpcodec.any_of_text` is a receiver's first call.** JSON-RPC allows an
array of messages, so an object and an array can both arrive on one
connection. This answers a batch either way, with one message in it for the
single case, so the caller writes one loop.

The message builders in `agptool`, `agpres`, `agpprompt`, `agpmeta` and
`agpsession` each produce an `AgpMessage`. Build the struct by hand and
call `agpwire.request` directly when you need a field a builder does not
take.

## The rules a user needs

1. **Nothing but `initialize` may be sent on a fresh connection, and
   nothing ordinary until the handshake is finished.** The Base Protocol's
   Lifecycle page gives the three steps. `agpsession.may_send` answers
   before a byte is written, and `agpsession.sent` refuses a message
   `may_send` would have refused, so a caller that skipped the check still
   cannot reach a state the protocol does not have.
2. **The two refusals mean opposite things.** `AgpNotYet` carries the
   state, and the caller finishes the handshake and tries again.
   `AgpNotOffered` carries a capability name, and the caller stops asking
   this peer. A boolean would collapse the two.
3. **`notifications/initialized` is a step a client owes.** The server is
   entitled to send nothing until it arrives, so a session that looks
   connected and receives nothing is usually this. `handshake_incomplete`
   answers true in exactly that state.
4. **An id that is a number is not the same id as the string of that
   number.** JSON-RPC 2.0 section 4 allows both spellings.
   `AgpIdNum(3)` and `AgpIdStr("3")` are different ids, and `agpwire.id_eq`
   is the comparison that keeps them apart.
5. **`agpwire.expects_response` decides whether the sender waits.** It is
   true for a request and false for a notification (JSON-RPC 2.0 section
   4.1). A batch of notifications alone gets no response at all, not an
   empty array, and `agpwire.batch_expects_response` says so before the
   caller starts waiting.
6. **A tool that failed answers a result, not an error.** A command that
   exited non-zero or a file that was absent comes back as a normal
   JSON-RPC result whose `isError` is true, so the model can see the
   failure and react to it. `AgpToolResult.is_error` carries it.
   `agptool.error_result` builds one, so `is_error` is not the field a
   caller forgot.
7. **A JSON-RPC error object is the peer talking, and an `AgpFault` never
   leaves this machine.** They are two types. A caller that retried on both
   would retry the one that can never succeed. `agperror.is_fatal` says
   which faults end a session.
8. **Check a tool's arguments before spending a model turn on them.**
   `agptool.compile_input` compiles the tool's declared JSON Schema once,
   and `agptool.check_arguments` answers schema-nv's own `SchOutput`, which
   names the instance location, the schema location and the keyword. The
   alternative is a round trip and the error code −32602.
9. **A tool's `read_only`, `idempotent`, `destructive` and `open_world`
   fields are hints and not guarantees.** The Server Features Tools page
   says so. An agent that treats them as guarantees runs a destructive tool
   it was told not to.
10. **Cancellation is a notification, so the answer may still arrive.** The
    Base Protocol's Cancellation utility gives no acknowledgement, and the
    server may have sent its response before the notification reached it.
    `agpsession.received` reports that late answer as `AgpUnknownId`, which
    is not fatal, and the right move is to drop it.
    `agpmeta.response_may_still_arrive` is that rule as a value.
11. **`initialize` may not be cancelled.** `agpmeta.may_cancel` answers
    false for it. Cancelling it leaves a session in a state neither side
    can name.
12. **A progress token travels in a request's `_meta`, not in its params.**
    A request that carried none gets no progress notifications even from a
    server that would have sent them. `agpmeta.with_progress_token` puts it
    where the protocol keeps it.
13. **Log levels are ordered, and the order is not alphabetical.** The
    eight levels come from syslog. `agpmeta.level_at_least` is the
    comparison. A client that wants nothing sets the most serious level
    rather than filtering a stream it still receives.
14. **A resource template is a URI with holes, and expanding one can be
    refused.** `agpres.expand` answers a `Result`, because a value
    containing `..` or a slash changes the URI's structure rather than
    filling it in. `agpres.value_is_safe` is that check on its own.
15. **Every field of a sampling request is a preference.** The Client
    Features Sampling page puts the choice of model with the client. The
    specification also says the client should show the person the request
    before it runs, and `agpprompt.requires_approval` is that obligation as
    a value.
16. **Pagination cursors are opaque and the server defines them.** Pass
    back the cursor a previous page answered, or an empty string for the
    first page. A constructed cursor is an invented one.
17. **A large integer id may not survive the round trip.**
    `agpcodec.number_is_exact` answers whether a number of that magnitude
    is carried exactly. A client that needs globally unique ids uses
    `AgpIdStr`.

## What is not included

- **The transports.** Standard input and output, server-sent events and
  streamable HTTP each frame messages their own way. `agpcodec.to_text` and
  `of_text` are the serialisation either side of a frame, and keeping the
  frame out is what lets one state machine serve all three.
- **Authorisation.** The OAuth flow the HTTP transport specifies needs
  network access, and this package has no effects at all.
- **Elicitation messages.** `AgpCapabilities.elicitation` carries the flag,
  so a session knows whether the peer offers it. The message types are not
  declared.
- **Completion.** `completion/complete`, the argument autocompletion a
  client offers while a person fills in a prompt, is not declared.
- **Timeouts, retries and scheduling.** The session records which requests
  are outstanding so a response can be matched and a duplicate id refused.
  It does not decide what to send next. Those need a clock this package
  does not have.
- **A JSON value of its own.** Documents are the standard library's
  `std.json` values. A second JSON type would mean every caller converting
  a document before it could be sent.
- **A microcontroller build.** No module here is declared to build for a
  device with no heap allocator. Every type in the package is a document
  and none is a `@value` struct.

## Related packages

- [schema-nv](https://novo-lang.org/packages/schema-nv) is JSON Schema
  2020-12 validation. A tool's input schema is compiled and checked with
  it, and `agptool.check_arguments` answers schema-nv's own output value
  rather than a boolean of its own.
- [llm-client-nv](https://novo-lang.org/packages/llm-client-nv) speaks to a
  hosted model API. That is the other half of an agent: this package says
  what a server offers, and that one asks a model what to do with it.
- [prompt-nv](https://novo-lang.org/packages/prompt-nv) builds the prompt
  text a model is sent. An MCP prompt expands to turns, and prompt-nv is
  where turns become a string for a local model.
- `std.mcp` in the standard library is `McpClient`: a real connection to a
  server over a stdio subprocess, with the `[io, mcp]` effect row. It owns
  the transport and answers a tool catalogue as a list of strings. This
  package owns the types and the ordering rule and owns no connection. A
  program that wants to talk to a server today uses `std.mcp`.
- `std.llm` in the standard library runs inference, including tool calling
  and embeddings. It is what a host does with what a server offered.
- `std.json` is the document type every message here carries.

## Tests

```bash
novo test tests/agpwire_tests.nv      #  8 tests: the envelope, ids and batches
novo test tests/agpsession_tests.nv   #  8 tests: the states, versions and refusals
novo test tests/agptool_tests.nv      #  9 tests: tools, schemas and results
novo test tests/agpcodec_tests.nv     #  7 tests: text, revisions and round trips
novo test tests/agpsurface_tests.nv   # 12 tests: every message this package builds
```

The reference is the Model Context Protocol's published schema, in
TypeScript and in its JSON Schema rendering. Where this package and that
schema disagree, the schema is right. The envelope's rules come from the
JSON-RPC 2.0 specification.

`agpsurface_tests.nv` builds every message the package can build and reads
each one back through `agpwire`, so a signature that does not compose with
the next one fails to compile rather than failing in a consumer. It also
asserts each builder against `agpwire.originator_of`, which is how a
consumer finds out it has been sending a server's message from a client.
`agpsession_tests.nv` asserts that `agpsession.stdlib_version` and the
revision `std.mcp` sends are the same string, so the day one moves the test
fails until the other does.

The tests compile today and fail at run, each on the
`not implemented: agent-proto-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `agpwire.jsonrpc_version`, `.request`, `.result_for`, `.error_for`, `.notification` | no |
| `agpwire.id_of`, `.method_of`, `.expects_response`, `.is_response`, `.id_eq`, `.id_text` | no |
| `agpwire.encode`, `.decode`, `.batch`, `.encode_batch`, `.decode_batch`, `.is_batch` | no |
| `agpwire.batch_expects_response`, `.batch_response_count` | no |
| `agpwire.known_methods`, `.is_known_method`, `.is_notification_method`, `.originator_of` | no |
| `agpsession.client_session`, `.server_session`, `.no_capabilities`, the six `with_*` calls | no |
| `agpsession.has_capability`, `.capability_names` | no |
| `agpsession.versions_supported`, `.latest_version`, `.stdlib_version`, `.supports_version` | no |
| `agpsession.negotiate`, `.method_in_version` | no |
| `agpsession.may_send`, `.may_receive`, `.allowed_methods` | no |
| `agpsession.sent`, `.received`, `.closing`, `.closed` | no |
| `agpsession.is_ready`, `.state_name`, `.outstanding_count`, `.is_outstanding` | no |
| `agpsession.initialize_request`, `.initialize_result`, `.initialized_notification`, `.handshake_incomplete` | no |
| `agpsession.capabilities_of`, `.peer_info_of` | no |
| `agptool.list_request`, `.tools_of`, `.next_cursor_of`, `.list_result`, `.find`, `.names_of` | no |
| `agptool.compile_input`, `.check_arguments`, `.arguments_fault` | no |
| `agptool.required_arguments`, `.argument_names` | no |
| `agptool.call_request`, `.result_of`, `.call_result`, `.text_result`, `.error_result` | no |
| `agptool.text_of`, `.content_kinds`, `.has_non_text`, `.decode_data`, `.list_changed_notification` | no |
| `agpres.list_request`, `.resources_of`, `.next_cursor_of`, `.read_request`, `.contents_of`, `.read_result` | no |
| `agpres.is_text`, `.decode_blob` | no |
| `agpres.templates_request`, `.templates_of`, `.variables_of`, `.expand`, `.value_is_safe`, `.matches` | no |
| `agpres.subscribe_request`, `.unsubscribe_request`, `.updated_notification`, `.updated_uri`, `.list_changed_notification` | no |
| `agpres.roots_request`, `.roots_of`, `.roots_result`, `.roots_changed_notification`, `.is_within_roots`, `.root_of` | no |
| `agpprompt.list_request`, `.prompts_of`, `.next_cursor_of`, `.get_request`, `.messages_of`, `.description_of`, `.get_result` | no |
| `agpprompt.required_arguments`, `.arguments_complete`, `.is_user_facing`, `.list_changed_notification` | no |
| `agpprompt.no_preferences`, `.sampling_request`, `.sampling_of`, `.sampling_result`, `.model_of`, `.stop_reason_of` | no |
| `agpprompt.requires_approval`, `.describe_request`, `.crosses_servers` | no |
| `agpmeta.set_level_request`, `.log_notification`, `.level_of`, `.logger_of` | no |
| `agpmeta.level_name`, `.level_of_name`, `.level_at_least`, `.levels` | no |
| `agpmeta.with_progress_token`, `.progress_token_of`, `.progress_notification`, `.progress_token_in` | no |
| `agpmeta.progress_of`, `.progress_total_of`, `.has_total`, `.token_eq` | no |
| `agpmeta.cancel_notification`, `.cancelled_id`, `.cancel_reason`, `.response_may_still_arrive`, `.may_cancel` | no |
| `agpmeta.ping_request`, `.ping_result`, `.is_ping` | no |
| `agpcodec.to_text`, `.of_text`, `.batch_to_text`, `.any_of_text` | no |
| `agpcodec.decode_for`, `.encode_for`, `.decode_in`, `.encode_in` | no |
| `agpcodec.round_trips`, `.message_eq`, `.distinguishes_empty_object_from_null`, `.number_is_exact` | no |
| `agperror.code_parse_error`, `.code_invalid_request`, `.code_method_not_found`, `.code_invalid_params`, `.code_internal_error` | no |
| `agperror.reserved_code_range`, `.is_standard_code`, `.standard_message` | no |
| `agperror.as_rpc_error`, `.is_fatal`, `.describe` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
