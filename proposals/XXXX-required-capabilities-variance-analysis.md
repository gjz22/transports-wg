# Tool-List Variance: Where Should Each Dimension Be Handled?

> **Working document.** This is a temporary decision aid accompanying
> [PR #31](https://github.com/modelcontextprotocol/transports-wg/pull/31)
> (SEP-XXXX: Required Capabilities for Tools).  It is not part of the
> SEP and is expected to be removed or folded in once a principle is
> agreed.

## Purpose

Review feedback on the SEP observed that "client capabilities may
change the available tools" is a subset of a broader problem: "the
available tools may change," whether because of authorization, client
capabilities, protocol version, language, server configuration, or a
deployment.  Deciding where capability-based variance is handled
(server-side or client-side) therefore sets a precedent for the other
dimensions.

This document lists the dimensions along which a tool list (or a tool
definition) can vary, how each one changes tools, how MCP handles it
today, and how HTTP handles the analogous problem.  The goal is to
agree on a **principle** for which kinds of variance are handled
server-side and which client-side, and why, rather than deciding one
dimension at a time.

## The Two Models

HTTP already names and analyzes both models in
[RFC 9110 §12 (Content Negotiation)](https://www.rfc-editor.org/rfc/rfc9110#section-12):

- **Server-side** corresponds to
  [proactive ("server-driven") negotiation](https://www.rfc-editor.org/rfc/rfc9110#section-12.1):
  the client sends signals on the request, the server selects or
  tailors the response, and caches must key on those signals
  ([`Vary`](https://www.rfc-editor.org/rfc/rfc9110#section-12.5.5)).
- **Client-side** corresponds to
  [reactive ("agent-driven") negotiation](https://www.rfc-editor.org/rfc/rfc9110#section-12.2):
  the server publishes the available alternatives and the client
  selects among them.

RFC 9110 lists the "serious disadvantages" of proactive negotiation.
They closely mirror the arguments in the SEP:

> - It is impossible for the server to accurately determine what might
>   be "best" for any given user, since that would require complete
>   knowledge of both the capabilities of the user agent and the
>   intended use for the response …
> - Having the user agent describe its capabilities in every request
>   can be both very inefficient … and a potential risk to the user's
>   privacy;
> - It complicates the implementation of an origin server and the
>   algorithms for generating responses to a request; and,
> - It limits the reusability of responses for shared caching.

It also says when reactive negotiation is preferable:

> Reactive negotiation is advantageous when the response would vary
> over commonly used dimensions (such as type, language, or encoding),
> when the origin server is unable to determine a user agent's
> capabilities from examining the request, and generally when public
> caches are used to distribute server load and reduce network usage.

It is also explicit about the costs of reactive negotiation: the list
of alternatives has to be transmitted, and RFC 9110 "does not define a
mechanism for supporting automatic selection."  In MCP terms, those
costs are the size of the annotations and the need for the client to
define matching rules.

## Summary Table

| Dimension | How it can change tools | MCP today | HTTP analogue | Where HTTP handles it |
| --- | --- | --- | --- | --- |
| **Authorization / user identity** | Which tools are visible; which operations succeed | Servers filter per user in practice (not specified); runtime [`insufficient_scope`](https://modelcontextprotocol.io/specification/draft/basic/authorization#runtime-insufficient-scope-errors) challenge; `cacheScope: "private"` ([SEP-2549](https://modelcontextprotocol.io/seps/2549-TTL-for-list-results)) | `Authorization`; `Cache-Control: private` | **Server.**  Shared caches must not reuse authenticated responses ([RFC 9111 §3.5](https://www.rfc-editor.org/rfc/rfc9111#section-3.5)) |
| **Protocol version** | Shape of tool definitions (which fields exist); tools relying on newer protocol features | Per-request [`protocolVersion`](https://modelcontextprotocol.io/specification/draft/basic/versioning#protocol-version-negotiation); server rejects unsupported versions with `-32022` listing supported ones; [`server/discover`](https://modelcontextprotocol.io/specification/draft/server/discover) advertises `supportedVersions` | HTTP version via ALPN ([RFC 7301](https://www.rfc-editor.org/rfc/rfc7301)) | **Negotiated at the connection**, not a content dimension.  The server must emit a format the client can parse |
| **Client implementation identity** | Workarounds for specific clients | `clientInfo` is self-reported; implementations **SHOULD NOT** use it to change behavior ([`_meta`](https://modelcontextprotocol.io/specification/draft/basic#_meta)) | [`User-Agent`](https://www.rfc-editor.org/rfc/rfc9110#section-10.1.5) | **Server, but discouraged.**  Fingerprinting risk; the web moved to [feature detection](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Browser_detection_using_the_user_agent) |
| **Language / i18n** | Tool `title` and `description` text; elicitation messages | [SEP-2792](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2792) (open): per-request `acceptLanguage`, server translates, cache key **MUST** include it | [`Accept-Language`](https://www.rfc-editor.org/rfc/rfc9110#section-12.5.4) + `Vary` | **Server** (proactive), with `Vary: Accept-Language` |
| **Client capabilities** (core: elicitation, `sampling.tools`, …) | Tool availability (hard requirements); possibly descriptions or behavior | Per-request `clientCapabilities` ("relevant to this request"); `-32021` at call time; **this SEP** proposes client-side annotations | `Accept` (media types); [Client Hints](https://www.rfc-editor.org/rfc/rfc8942); feature detection; `srcset` | **Mixed.**  Server-side via `Accept` and opt-in Client Hints; client-side via feature detection and [responsive images](https://developer.mozilla.org/en-US/docs/Web/HTML/Guides/Responsive_images) |
| **Extensions** (tasks, MCP Apps, …) | Availability (task-only or app-only tools); definition tailoring (`_meta.ui`, description) | Declared in `capabilities.extensions`; [MCP Apps](https://github.com/modelcontextprotocol/ext-apps/blob/ea3e9d0e624a20bc1c51310292f09d9b8ed17161/specification/draft/apps.mdx?plain=1#L2238-L2309) says servers **SHOULD** check client capabilities before registering UI tools | Same as client capabilities | Same as client capabilities |
| **Server configuration / deployment** | Tool set changes over time | [`notifications/tools/list_changed`](https://modelcontextprotocol.io/specification/draft/server/tools); `ttlMs` (SEP-2549) | Validators (`ETag`), freshness (`max-age`) | **Server.**  Not client-dependent |

## Dimension Notes

### Authorization / User Identity

Always server-side.  A client cannot be trusted to hide tools a user is
not authorized to use, and publishing those tools to every client would
reveal that they exist.  HTTP treats authenticated responses as
non-shareable by default
([RFC 9111 §3.5](https://www.rfc-editor.org/rfc/rfc9111#section-3.5)).
MCP's `cacheScope: "private"` in
[SEP-2549](https://modelcontextprotocol.io/seps/2549-TTL-for-list-results)
covers this axis for caching.

Per-user tool filtering is common practice, but the MCP specification
does not describe it.  The specification does define a call-time
signal,
[`insufficient_scope`](https://modelcontextprotocol.io/specification/draft/basic/authorization#scope-challenge-handling)
with step-up authorization.  That is the authorization counterpart of
`MissingRequiredClientCapabilityError` (`-32021`).

### Protocol Version

Server-side by necessity: the protocol version determines the wire
format of the tool definitions themselves, so a client cannot "filter"
a representation it cannot parse.  Selecting the version, however, is
closer to reactive negotiation.  The client declares a version on each
request.  If the server does not support it, the server responds with
[`UnsupportedProtocolVersionError`](https://modelcontextprotocol.io/specification/draft/basic/versioning#protocol-version-negotiation)
listing its `supported` versions, and the client picks one and retries.
[`server/discover`](https://modelcontextprotocol.io/specification/draft/server/discover)
lets the client learn the supported versions up front.

On the HTTP comparison: the HTTP version is negotiated per connection
via ALPN ([RFC 7301](https://www.rfc-editor.org/rfc/rfc7301)) and does
not change response content, so it is not a close analogue.
`User-Agent` is closer to MCP's `clientInfo` (next section) than to the
protocol version.

### Client Implementation Identity

[RFC 9110 §10.1.5](https://www.rfc-editor.org/rfc/rfc9110#section-10.1.5)
acknowledges that servers use `User-Agent` to "tailor responses to
avoid particular user agent limitations," but warns about
fingerprinting.  The web has largely moved from user-agent sniffing to
[feature detection](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Browser_detection_using_the_user_agent),
which is a client-side decision.  MCP has already taken a position
here: `clientInfo` and `serverInfo` are self-reported, and
implementations "**SHOULD NOT** use them to change the behavior of the
client or server"
([basic `_meta`](https://modelcontextprotocol.io/specification/draft/basic#_meta)).

### Language / Internationalization

[SEP-2792](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2792)
(open; sponsored by @pja-ant) takes the server-side route.  The client
sends `io.modelcontextprotocol/acceptLanguage` in request `_meta`, and
a server that honors the preference translates tool titles and
descriptions.  Any cache that reuses a response **MUST** include the
language preference in its cache key, and Streamable HTTP uses
`Vary: Accept-Language`.  This mirrors HTTP's
[`Accept-Language`](https://www.rfc-editor.org/rfc/rfc9110#section-12.5.4)
proactive negotiation.

Server-side works well here for two reasons:

- **The preference is the same across requests.**  The language the
  user wants for `tools/list` is the language they want for
  `tools/call`, so no cross-request coordination is needed.
- **Shipping every translation to every client would be wasteful.**

SEP-2792 accepts the extra cache dimension as the cost.

### Client Capabilities

This is the dimension this SEP addresses, and the one where prior art
is genuinely mixed.

- **HTTP, server-side:** `Accept` declares media types the client can
  handle, and the server selects.
  [Client Hints (RFC 8942)](https://www.rfc-editor.org/rfc/rfc8942)
  let servers adapt content to device capabilities.  But RFC 8942
  exists largely to fix the problems of sending capabilities
  proactively.  It makes disclosure **opt-in**: the server must ask
  via `Accept-CH`
  ([§3.1](https://www.rfc-editor.org/rfc/rfc8942#section-3.1)), "to
  mitigate privacy concerns of passive fingerprinting."  It also
  requires `Vary` on cacheable responses
  ([§3.2](https://www.rfc-editor.org/rfc/rfc8942#section-3.2)).
- **Web, client-side:** feature detection, and `srcset` / `<picture>`,
  where the server lists variants and "the browser will choose" among
  them
  ([MDN: Responsive images](https://developer.mozilla.org/en-US/docs/Web/HTML/Guides/Responsive_images)).
- **LSP, server-side, declared up front:** MCP's design draws on LSP,
  which exchanges capabilities once in the
  [`initialize`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize)
  request
  ([`ClientCapabilities`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#clientCapabilities)).
  Servers tailor responses to them.  For example, a client lists its
  supported hover formats in `contentFormat`
  ([Hover](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_hover)),
  and the server chooses markdown or plain text.  This works in LSP
  because capabilities are session-scoped.  Under
  [SEP-2575](https://modelcontextprotocol.io/specification/draft/basic#statelessness),
  MCP capabilities are request-scoped.

Unlike the dimensions above, the client capabilities on a `tools/list`
request can legitimately differ from those on a later `tools/call`.
The specification defines `clientCapabilities` as "relevant to this
request."  Server-side handling therefore needs an explicit
cross-request mechanism that the other dimensions do not.

#### What LSP Says About the Tradeoff

LSP never discusses server-side vs. client-side handling of
capabilities, because it never needed to.  Capabilities are exchanged
once and hold for the whole session, so the server always knows what
the client supports and simply tailors its behavior.  The
cross-request problem cannot arise in LSP.  SEP-2575 created it for
MCP by removing the session.  LSP does have three related pieces of
prior art:

- **Completion filtering is client-side by design, with stated
  reasons.**  The
  [Completion](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_completion)
  section says:

  > to achieve consistency across languages and to honor different
  > clients usually the client is responsible for filtering and
  > sorting.  This has also the advantage that client can experiment
  > with different filter and sorting models.  However servers can
  > enforce different behavior by setting a `filterText` / `sortText`
  >
  > for speed clients should be able to filter an already received
  > completion list if the user continues typing.  Servers can opt out
  > of this using a `CompletionList` and mark it as `isIncomplete`.

  The reasoning mirrors the SEP's: client flexibility in how results
  are presented, and reuse of an already-received list.  The server
  keeps an override (`filterText`) and an opt-out (`isIncomplete`).
  The caveat is that completion items are filtered by what the user
  has typed, not by client capabilities.  This is precedent for
  *where* to filter, not for capabilities specifically.
- **Changes to available features are server-pushed and stateful.**
  [`client/registerCapability`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#client_registerCapability)
  and `client/unregisterCapability` let servers add or remove features
  after `initialize`.  The 3.0 changelog says servers "can more
  dynamically react to client features" and "react to settings or
  configuration changes without a restart."  A server supporting both
  static and dynamic registration "needs to check the client
  capability in the initialize request," which is server-side
  tailoring that depends on the session.  The MCP counterpart is
  `notifications/tools/list_changed`.
- **Unknown metadata is ignored.**
  [Capabilities](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#capabilities):
  "Clients should ignore server capabilities they don't understand."
  This is the same pattern as MCP Apps' `_meta.ui`, which clients
  without MCP Apps support ignore.

In short, LSP is prior art for server-side tailoring, but only
because its capabilities are scoped to a session, which MCP's no
longer are.  Where LSP does weigh client-side against server-side
filtering (completion), it defaults to the client.

### Extensions

Extensions are declared inside client capabilities
(`capabilities.extensions`), so they are the same axis as client
capabilities rather than a separate one.  Two extensions illustrate the
two kinds of variance:

- **Tasks:** a tool that only works as a long-running task is an
  *availability* question.
- **MCP Apps:** the
  [MCP Apps specification](https://github.com/modelcontextprotocol/ext-apps/blob/ea3e9d0e624a20bc1c51310292f09d9b8ed17161/specification/draft/apps.mdx?plain=1#L2238-L2309)
  is prior art for server-side handling.  Clients advertise
  `io.modelcontextprotocol/ui` with supported `mimeTypes`.  Servers
  "**SHOULD** check client capabilities before registering UI-enabled
  tools" and "**MAY** register different tool variants based on host
  capabilities"; the specification's example registers a UI version
  and a text-only version of the same tool with different
  descriptions.  It describes this capability exchange as happening in
  the `initialize` request.  That text predates SEP-2575 and has not
  yet been updated for per-request capabilities, so it is unclear
  which request's capabilities a server should check today.

MCP Apps is also prior art for client-side handling.  Tools carry
`_meta.ui.visibility`, and "Host **MUST NOT** include tools in the
agent's tool list when their visibility does not include `"model"`"
([Visibility](https://github.com/modelcontextprotocol/ext-apps/blob/ea3e9d0e624a20bc1c51310292f09d9b8ed17161/specification/draft/apps.mdx?plain=1#L414-L421)).
This is a per-tool `_meta` annotation that the client uses to filter.
Most UI tools are hybrid: "If host does not support MCP Apps, tool
behaves as standard tool (text-only fallback)"
([Behavior](https://github.com/modelcontextprotocol/ext-apps/blob/ea3e9d0e624a20bc1c51310292f09d9b8ed17161/specification/draft/apps.mdx?plain=1#L405-L412)).
App-only tools are a real availability case, though.  A host without
MCP Apps support does not understand `visibility` and would otherwise
show them to the model.

### Server Configuration / Deployment

Not dependent on the client, so not a candidate for client-side
handling.  It is handled through invalidation and freshness
(`notifications/tools/list_changed`, `ttlMs`) rather than through
negotiation.  It is listed here for completeness, because it
contributes to the cache key.

## Observations

1. **The dimensions fall into four groups.**
   - **Security:** authorization.  Must be server-side.
   - **Wire format:** protocol version.  Server-side by necessity,
     with version selection negotiated.
   - **User preference:** language.  Server-side; consistent across
     requests.
   - **Client functionality:** capabilities and extensions.
     Contested.

   Client identity (`clientInfo`) is already settled: do not vary on
   it.
2. **Client functionality is the only dimension where the list-time
   and call-time signals can legitimately differ.**  Every other
   dimension is the same on `tools/list` and `tools/call`, so
   server-side handling needs no cross-request mechanism for them.
3. **Availability and tailoring are different problems.**  Client-side
   annotations handle availability (show or hide a tool).  Tailoring
   (a different description or schema) needs either variants the
   client selects between, which requires selection rules, or
   metadata that unsupporting clients ignore, as MCP Apps mostly does.
4. **HTTP's own guidance leans reactive for capabilities.**  RFC 9110
   favors reactive negotiation "when the origin server is unable to
   determine a user agent's capabilities from examining the request"
   and for shared caching.  RFC 8942 shows that server-side capability
   negotiation is workable, but only with opt-in disclosure and
   explicit `Vary`.
5. **LSP's server-side tailoring relies on session-scoped
   capabilities.**  MCP no longer has those.  Where LSP does choose
   where to filter (completion), it defaults to the client, for reasons
   similar to the SEP's.

## Candidate Principles (for Discussion)

- **A. Client-side for client functionality.**  For any variance
  driven by what the client can do (capabilities and extensions), the
  server publishes one tool list annotated with requirements, and the
  client selects.  Authorization, protocol version, and language stay
  server-side.
- **B. Server-side for everything.**  All variance is server-side.
  The specification defines a cross-request capability declaration
  (e.g., the SEP's Alternative 1a) and a `Vary`-like cache mechanism
  that can express a capabilities dimension.
- **C. Split by kind of variance.**  Availability is client-side
  (annotations); tailoring a definition is server-side, using the
  capabilities relevant to the list request.

## Open Questions

- How often do capability-dependent tools need a *different
  definition*, as opposed to simply being unavailable?  Collecting
  real examples (GitHub's MCP Apps metadata, task-only tools,
  elicitation-only tools) would ground the choice.
- Should cache invariants (a `Vary`-like extension of `cacheScope`) be
  defined first, in parallel, or after?  Every option except A needs
  them for client functionality; all options need them for language.
- If B or C is chosen, which request's capabilities does a server use
  when deciding on `tools/list`?  MCP Apps needs an answer to this
  regardless, since `initialize` no longer exists.
