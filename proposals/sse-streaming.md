# Server-Sent Events Streaming for OAuth Deferred Request Processing

## Status

Proposal. This document defines a candidate higher-layer profile of [draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md). It is presented to validate the substrate's advisory-delivery extension model. It is not itself on a publication path.

## Abstract

This proposal defines a Server-Sent Events (SSE) streaming profile that lets a client open a long-lived HTTPS connection to receive deferred processing state transitions and a final completion event as a stream. When the issued access token is sender-constrained (DPoP-bound or mTLS-bound), the `complete` event MAY carry the access token response itself; otherwise it carries only non-authoritative result metadata, and the client obtains the issued credentials by polling. Continuation polling at the token endpoint is always available as an alternative authoritative completion path. The profile is intended for clients with stable connectivity that prefer near-real-time state updates, particularly for medium-duration interactive flows where Pending -> Interaction Required -> Complete may transition rapidly.

## Scope

This proposal defines:

- A new `grant_mode` value, `stream`, by which clients signal acceptance of SSE streaming.
- A new HTTPS endpoint (the SSE streaming endpoint) at which clients open SSE connections using a short-lived stream handle bound to an existing deferred processing state.
- The SSE event format for state transitions, slow-down signals, and final completion indications.
- Authentication and binding of the SSE connection to the deferred processing state.
- Polling fallback when SSE is unavailable or the connection drops.

This proposal does not define:

- Webhook completion delivery — see the [Webhook Result Delivery proposal](./webhook-result-delivery.md) for that case.
- WebSocket-based streaming. SSE is chosen for its HTTP/1.1 compatibility, simple text protocol, and unidirectional server-to-client semantics that match the substrate's pull-and-receive model.

## Relation to the Base Specification

This proposal uses the following extension surfaces defined by the base spec:

| Extension surface | Use |
|---|---|
| OAuth Grant Mode Values Registry (Specification Required policy) | Registers `stream` |
| §Profile-Defined Advisory Delivery Channels (generic hook) | Defines SSE streaming as a concrete instance, including the profile-defined endpoint permitted by the hook |
| OAuth Authorization Server Metadata Registry | Registers `deferred_code_streaming_endpoint` and `deferred_code_streaming_supported` |
| OAuth Parameters Registry | Registers `deferred_code_stream_handle` |
| Abstract State Status | The SSE event vocabulary maps directly to the substrate's Pending / Interaction Required / Complete / Denied / Expired / Invalid states |
| Continuation polling state machine | Reused as fallback when SSE is unavailable; polling remains authoritative |

This proposal lands entirely on the base spec's generic advisory delivery channel hook; no base-spec amendment is required. The hook explicitly permits profile-defined advisory channels to use new authorization-server endpoints distinct from the token endpoint (with the constraints that such endpoints are published in AS metadata, authenticate using credentials and sender-constraining proof equivalent to a continuation request, and do not host continuation processing). The streaming endpoint defined here is one such profile-defined endpoint.

## Defined Value

**stream**
: Client accepts SSE streaming of deferred processing state transitions. The authorization server MAY make the SSE streaming endpoint available for clients that have signaled `stream`. Combining with `deferred` signals acceptance of both deferred processing and streaming delivery: `grant_mode=deferred stream`.

## Wire Shape

### Stream Handle

An authorization server that supports this profile MAY issue a `deferred_code_stream_handle` parameter in a deferred processing response or continuation response for a client that signaled `stream`. The `deferred_code_stream_handle` is an opaque, single-use handle used only to establish an SSE connection. It is bound to the same client and sender-constraining context as the deferred processing state, MUST have a lifetime no longer than the associated `deferred_code`, and SHOULD have a substantially shorter lifetime.

For authorization endpoint originating requests, the `deferred_code_stream_handle` MUST NOT be conveyed in the deferred authorization response. If the authorization server issues a stream handle for such a request, it does so no earlier than the first continuation response, after the client has authenticated to the token endpoint.

Example continuation response carrying a stream handle:

~~~ json
{
  "error": "authorization_pending",
  "deferred_code": "8N5B2K1",
  "deferred_code_stream_handle": "sh_7M2R4",
  "interval": 5,
  "expires_in": 900
}
~~~

### Establishing the Stream

After receiving a `deferred_code_stream_handle`, a client that has signaled `stream` MAY open an SSE connection to the streaming endpoint:

~~~ http
GET /deferred-stream?deferred_code_stream_handle=sh_7M2R4 HTTP/1.1
Host: as.example.com
Authorization: Bearer <client-credentials-or-DPoP-token>
Accept: text/event-stream
~~~

The authorization server validates the client identity, sender-constraining proof, stream handle freshness, and stream handle binding to the deferred processing state using rules equivalent to those that apply to continuation requests. On successful validation, the authorization server consumes the stream handle and returns:

~~~ http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-store

event: state
data: {"status":"authorization_pending","interval":5,"expires_in":900}

~~~

Note the trailing blank line; SSE events are separated by `\n\n` per the SSE specification.

### State Transition Events

Each state transition produces an event:

~~~
event: state
data: {"status":"interaction_required","interaction_uri":"https://as.example.com/interact/8N5B2K1","expires_in":540}

event: state
data: {"status":"authorization_pending","expires_in":300}
~~~

The `status` values reuse the substrate's vocabulary (`authorization_pending`, `interaction_required`, `slow_down`). The `data` payload uses the same parameter names as the equivalent token endpoint responses.

If the authorization server rotates the `deferred_code` while the stream is open, the next event MUST include the replacement `deferred_code`. The client MUST discard the previous value and use the replacement value for future continuation requests and, if needed, stream reestablishment:

~~~
event: state
data: {"status":"authorization_pending","deferred_code":"P9Q4R2S","expires_in":300}
~~~

### Completion or Terminal Error

When deferred processing terminates successfully, the authorization server sends a final event and closes the stream. The form of the `complete` event depends on whether the issued access token is sender-constrained.

**Credential delivery.** When the issued access token is sender-constrained (DPoP-bound or mTLS-bound) and any included refresh token and ID Token meet the requirements stated below, the `complete` event MAY carry the access token response as the `result`:

~~~
event: complete
data: {"event_id":"f9c1...","issued_at":1714073760,"status":"complete","result":{"access_token":"SlAV32hkKG","token_type":"DPoP","expires_in":3600,"scope":"transfer.execute","refresh_token":"8xLOxBtZp8"}}

~~~

The `access_token` MUST be sender-constrained. If the originating request established DPoP, the issued access token is bound to the DPoP key thumbprint and `token_type` is `DPoP` per {{RFC9449}}. If the originating request established mTLS client certificate binding, the issued token is bound to the certificate thumbprint per {{RFC8705}}. If a `refresh_token` is included, it MUST be sender-constrained by the same mechanism. If an `id_token` is included, it MUST carry a confirmation-method binding (such as a `cnf` claim) tying it to the client-held key or certificate.

A delivered access token in the `complete` event is authoritative on confirmed receipt (see Confirmation and Cancellation below). The same access token response would be returned by continuation polling for the same deferred processing state until polling-equivalence is broken by cancellation or expiration.

**Preview-only delivery.** When the issued access token is not sender-constrained, the `complete` event MUST omit the access token, refresh token, ID Token, and authorization code, and MUST instead carry only non-authoritative result metadata:

~~~
event: complete
data: {"event_id":"f9c1...","issued_at":1714073760,"status":"complete","result_preview":{"token_type":"Bearer","expires_in":3600,"scope":"transfer.execute"}}

~~~

The `result_preview` member MUST NOT contain access tokens, refresh tokens, ID Tokens, authorization codes, assertions, or other credentials. The client MUST NOT use any previewed value as granted authorization and MUST poll the token endpoint to obtain the issued credentials.

Authorization servers MUST select credential delivery only when all of the following hold:

* the issued access token is sender-constrained at issuance through DPoP, mTLS, or an equivalent mechanism the resource server verifies on every use;
* if a refresh token is included, it is sender-constrained by the same mechanism;
* if an ID Token is included, it carries a confirmation-method binding.

If any of these conditions is not met, the authorization server MUST use preview-only delivery.

**Terminal error.** For terminal errors:

~~~
event: error
data: {"event_id":"a1b2...","issued_at":1714073760,"error":"access_denied"}

~~~

After the `complete` or `error` event, the authorization server closes the SSE connection.

### Confirmation and Cancellation

When the `complete` event carries credential delivery (`result`), the client confirms receipt by closing the SSE connection cleanly after processing the `complete` event. Clean connection close by the client after the `complete` event is delivery confirmation; an abrupt drop, network failure, or absence of close before a configurable timeout is unconfirmed delivery.

If the deferred processing state is cancelled per the base specification's §Cancellation before delivery is confirmed, the authorization server MUST:

- terminate the SSE connection without further delivery,
- revoke the delivered access token and refresh token, if any, that were carried in a delivered-but-unconfirmed `complete` event, equivalently to revoking tokens obtained through polling.

Once delivery is confirmed, the delivered credentials are committed and cancellation does not retroactively invalidate them; credentials in the client's possession are revoked only through their own mechanisms.

When the `complete` event carries `result_preview` only, no credentials have been delivered through SSE; the credentials are obtained through subsequent polling and are subject to the cancellation rules in the base specification's §Cancellation directly.

### Fallback to Polling

If the SSE connection drops or the streaming endpoint is unavailable, the client MUST fall back to polling the `deferred_code` at the token endpoint per the substrate's continuation grant type. The substrate's polling state machine is authoritative in all cases; SSE is a delivery optimization, not a replacement. A client that wants to reestablish streaming after a dropped connection obtains a new `deferred_code_stream_handle` from a continuation response. A previously consumed `deferred_code_stream_handle` MUST NOT be reused.

## Client Processing

A client opening an SSE connection MUST authenticate using the same credentials and sender-constraining proof required for a continuation request. The authorization server MUST validate this on connection establishment.

The client MUST treat `state` events as advisory hints; they MAY be used to update local UI or scheduling decisions but MUST NOT be used as granted authorization. The client MUST reject any event with an `issued_at` value outside the client's configured freshness window and MUST detect replay of an `event_id` already accepted for the same deferred processing state.

When the `complete` event carries a `result` containing a sender-constrained access token response, the client MAY treat the response as authoritative on receipt. The client MUST verify that the issued access token is sender-constrained as expected (for example, bound to the DPoP key the client used on the originating request); if the credential is not bound to the client's key or certificate as expected, the client MUST reject the delivery and SHOULD obtain the result by polling instead. The client confirms receipt by closing the SSE connection cleanly after processing the `complete` event.

When the `complete` event carries `result_preview` instead, the client MUST NOT treat the preview as an access token response and MUST poll the token endpoint to obtain the issued credentials.

If connection establishment fails before the authorization server consumes the `deferred_code_stream_handle`, the client MAY retry connecting once with exponential backoff. If an established stream drops before a terminal event, the client MUST poll the token endpoint and obtain a new `deferred_code_stream_handle` before reconnecting.

## Authorization Server Behavior

The authorization server MAY make the SSE streaming endpoint available for deferred processing states whose clients signaled `stream`. The authorization server MUST NOT depend on successful streaming; deferred processing state remains observable through continuation requests until completion is confirmed or the state ends.

The authorization server SHOULD apply per-state and per-client limits on simultaneous SSE connections (a deferred processing state SHOULD support at most one active SSE connection at a time).

The authorization server MUST validate sender-constraining and client authentication on connection establishment. If the connection is established and the client's binding context changes (for example, DPoP nonce rotation requires a fresh proof), the authorization server MAY close the stream and require the client to poll for a new `deferred_code_stream_handle` before reestablishing.

When the `complete` event carries credential delivery, the authorization server MUST track the confirmation state of the delivery and MUST handle the cancellation race as defined above: revoke delivered-but-unconfirmed credentials when cancellation occurs, treat confirmed credentials as committed.

## Security Considerations

- The SSE endpoint URL contains a `deferred_code_stream_handle` query parameter. The stream handle is sensitive because it attaches a stream to deferred processing state. Authorization servers MUST make stream handles opaque, high-entropy, short-lived, single-use, and bound to the same client and sender-constraining context as the associated deferred processing state. Authorization servers and clients SHOULD minimize logging of stream handles.
- The SSE connection is long-lived. Authorization servers SHOULD enforce a maximum connection duration aligned with the `deferred_code` lifetime, and MUST close the connection when the deferred processing state terminates.
- The `complete` event, when carrying a sender-constrained access token response, is sensitive metadata even though the credential is useless without the client's bound key or certificate. TLS protects the value in transit. Authorization servers SHOULD minimize server-side logging of SSE payloads and SHOULD consider end-to-end encryption (for example, JWE) for high-sensitivity deployments.
- SSE events MUST NOT contain bearer access tokens, bearer refresh tokens, authorization codes, assertions, or ID Tokens that lack a confirmation-method binding. The authorization server MUST verify that the credentials selected for delivery satisfy these requirements before sending the `complete` event with `result`. If the requirements are not met, the authorization server MUST use preview-only delivery instead.
- Replay: each SSE event MUST include `event_id` and `issued_at` for client-side replay detection. The deferred_code rotation rules from the base spec apply. If a deferred_code is rotated mid-stream, the SSE event MUST carry the new value and the client MUST discard the previous value. The sender-constraining proof at the resource server protects a delivered credential against replay by parties that do not hold the bound key.
- Cancellation race: delivered-but-unconfirmed credentials (where the SSE connection has not been cleanly closed by the client after the `complete` event) MUST be revoked if cancellation is received before confirmation. Failure to enforce this leaves a window in which a cancelled request can still produce usable credentials.
- All other security considerations of the base spec apply unchanged.

## IANA Considerations

This proposal would register:

**OAuth Grant Mode Values:**
| Value | Description |
|---|---|
| `stream` | Client accepts SSE streaming of deferred state transitions and completion indications |

**OAuth Authorization Server Metadata:**
| Name | Description |
|---|---|
| `deferred_code_streaming_supported` | Boolean indicating support for SSE streaming |
| `deferred_code_streaming_endpoint` | HTTPS URL of the SSE streaming endpoint |

**OAuth Parameters Registry:**
| Name | Usage |
|---|---|
| `deferred_code_stream_handle` | Token response, to convey a short-lived handle used to establish an SSE stream for a deferred processing state |

## Extensibility Validation Summary

This proposal exercises the following base-spec extension surfaces:

1. **Grant Mode value registration.** `stream` is added via the Specification Required policy.
2. **AS metadata.** Two new metadata fields registered.
3. **State vocabulary reuse.** The SSE event payloads reuse the substrate's status vocabulary (`authorization_pending`, `interaction_required`, `slow_down`) and parameter names without modification.
4. **Profile-defined response parameter.** `deferred_code_stream_handle` carries a short-lived stream-establishment handle without making the `deferred_code` itself part of the streaming endpoint URL.
5. **Advisory delivery channel carrying sender-constrained credentials.** The `complete` event MAY carry an access token response when the credentials are sender-constrained, exercising the credential-delivery scope of §Profile-Defined Advisory Delivery Channels.
6. **Polling-equivalence and polling-availability preserved.** Polling at the token endpoint remains available as an alternative authoritative completion path and returns the same credentials.

**Verdict: no base-spec amendment required.** The base spec's generic advisory-delivery hook in §Profile-Defined Advisory Delivery Channels permits this proposal as a concrete instance, including the use of a new authorization-server endpoint and credential delivery for sender-constrained tokens. The hook's requirements (authentication, freshness, cancellation handling, polling-equivalence, substrate-invariant preservation, profile-defined endpoints don't host continuation processing, and the bearer-credential exclusions) are satisfied by this proposal's specific design.

## Relationship to the Notification Mechanism

The base specification's §Notification mechanism is a single state-change hint per state transition. SSE streams every state transition over a long-lived connection. Notification and SSE are not mutually exclusive: a client MAY register a notification endpoint (for events when no SSE connection is open) and use SSE when actively waiting for completion. The two mechanisms use distinct credentials and address different latency requirements.
