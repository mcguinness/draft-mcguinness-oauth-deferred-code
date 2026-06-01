# Webhook Result Delivery for OAuth Deferred Request Processing

## Status

Proposal. This document defines a candidate higher-layer profile of [draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md). It is presented to validate the substrate's advisory-delivery extension model, including the credential-delivery option permitted for sender-constrained credentials. It is not itself on a publication path.

## Abstract

This proposal defines a webhook delivery profile that pushes a completion event to a client-registered HTTPS endpoint. When the issued access token is sender-constrained (DPoP-bound or mTLS-bound), the webhook payload MAY carry the access token response itself; otherwise the payload carries only non-authoritative result metadata, and the client obtains issued credentials by polling. Continuation polling at the token endpoint is always available as an alternative authoritative completion path. The profile is intended for clients that prefer event-driven completion to long-running polling, particularly for governance workflows with hours-to-days deferred lifetimes.

## Scope

This proposal defines:

- A new `completion_mode` value, `push`, by which clients signal they accept webhook completion delivery.
- An AS-initiated HTTPS POST to a client-registered endpoint carrying a completion event with either a sender-constrained access token response or a non-authoritative preview, depending on whether the issued credentials are sender-constrained.
- A delivery-confirmation mechanism through the webhook's HTTP response, and a cancellation race rule that distinguishes confirmed and unconfirmed deliveries.
- AS metadata advertising support and client metadata registering the endpoint.
- Authentication of the webhook request, binding to the deferred processing state, and freshness protection against replay.

This proposal does not define:

- Streaming protocols (SSE, WebSocket) for completion delivery — see the [Server-Sent Events Streaming proposal](./sse-streaming.md) for that case.
- Result delivery via the existing `deferred_code_notification_endpoint`. The base specification's notification mechanism is intentionally a state-change hint that carries no result data; this proposal does not change that. Webhook result delivery uses a separate endpoint.
- Push delivery of bearer credentials. The base spec's §Profile-Defined Advisory Delivery Channels rules forbid bearer access tokens, bearer refresh tokens, authorization codes, assertions, and `cnf`-less ID Tokens in advisory payloads. A use case requiring push delivery of a bearer access token SHOULD use CIBA push mode or another mechanism designed for that case.

## Relation to the Base Specification

This proposal uses the following extension surfaces defined by the base spec:

| Extension surface | Use |
|---|---|
| OAuth Completion Mode Values Registry (Specification Required policy) | Registers `push` |
| §Profile-Defined Advisory Delivery Channels (generic hook) | Defines the webhook channel as a concrete instance, subject to the bearer-credential exclusions, additional requirements for credential delivery, and polling-availability rule |
| OAuth Authorization Server Metadata Registry | Registers `deferred_code_push_delivery_supported` |
| OAuth Dynamic Client Registration Metadata Registry | Registers `deferred_code_push_delivery_endpoint` |
| Continuation polling state machine | Reused unchanged; polling remains available as an authoritative completion path |

This proposal lands entirely on the base spec's generic advisory delivery channel hook; no base-spec amendment is required. The hook permits profile-defined advisory channels to carry sender-constrained credentials subject to authentication, freshness, cancellation handling, and polling-equivalence. This proposal supplies the concrete design (HTTPS POST to client-registered endpoint, bearer credential authentication of the webhook, JSON body shape, HTTP-response-as-confirmation) layered on those requirements.

## Defined Value

**push**
: Client accepts webhook completion delivery to the registered `deferred_code_push_delivery_endpoint`. The authorization server MAY deliver a completion event to that endpoint when deferred processing completes. When the issued access token is sender-constrained (DPoP-bound or mTLS-bound), the completion event MAY carry the access token response. Combining with `deferred` signals acceptance of both deferred completion and push completion delivery: `completion_mode=deferred push`.

The `push` value does not by itself authorize ordinary deferred-code processing. A client that accepts webhook delivery for a deferred request SHOULD send `completion_mode=deferred push`. If a request contains `completion_mode=push` without `deferred`, the authorization server MUST NOT infer acceptance of deferred completion from `push` alone; it MAY use push delivery only when deferred processing is otherwise authorized by client metadata or by a profile that explicitly defines `push`-only semantics.

## Wire Shape

### Push Delivery Token

When the authorization server intends to deliver webhook events for a deferred processing state, a deferred processing response or continuation response includes a `deferred_code_push_delivery_token` parameter:

~~~ json
{
  "error": "authorization_pending",
  "deferred_code": "8N5B2K1",
  "deferred_code_push_delivery_token": "9d2f6c4b8a1e7d3f5b9c0a2e4f6d8b1a",
  "interval": 5,
  "expires_in": 900
}
~~~

For authorization endpoint originating requests, the `deferred_code_push_delivery_token` MUST NOT be conveyed in the deferred authorization response. If the authorization server issues a push delivery token for such a request, it does so no earlier than the first continuation response, after the client has authenticated to the token endpoint.

The `deferred_code_push_delivery_token` is an opaque bearer credential used only to authenticate webhook delivery from the authorization server to the client's registered push delivery endpoint. The authorization server MAY rotate the token in later continuation responses. If a continuation response includes a replacement value, the client MUST discard the previous value and validate subsequent webhook deliveries against the replacement.

### Webhook Request: Credential Delivery

When deferred processing completes successfully, the client has signaled `push` and registered an endpoint, and the issued access token is sender-constrained (DPoP-bound or mTLS-bound), the authorization server MAY send an HTTPS POST carrying the access token response as the `result`:

~~~ http
POST /completion-webhook HTTP/1.1
Host: client.example.com
Authorization: Bearer 9d2f6c4b8a1e7d3f5b9c0a2e4f6d8b1a
Content-Type: application/json

{
  "deferred_code": "8N5B2K1",
  "event_id": "2b7f0e6a-0c74-4e3e-8a7a-2efcb6c5b7d9",
  "issued_at": 1714073760,
  "status": "complete",
  "result": {
    "access_token": "SlAV32hkKG",
    "token_type": "DPoP",
    "expires_in": 3600,
    "refresh_token": "8xLOxBtZp8",
    "scope": "transfer.execute"
  }
}
~~~

The `result` member carries the access token response that a continuation request for the same deferred processing state would return. The `access_token` MUST be sender-constrained: if the originating request established DPoP, the issued access token is bound to the DPoP key thumbprint, and `token_type` is `DPoP` per {{RFC9449}}; if the originating request established mTLS client certificate binding, the issued token is bound to the certificate thumbprint per {{RFC8705}}. If a `refresh_token` is included, it MUST be sender-constrained by the same mechanism. If an `id_token` is included, it MUST carry a confirmation-method binding (such as a `cnf` claim) tying it to the client-held key or certificate.

A delivered access token is authoritative on confirmed delivery (see Confirmation and Cancellation below). The same access token response would be returned by continuation polling for the same deferred processing state until polling-equivalence is broken by cancellation or expiration.

### Webhook Request: Preview-Only Delivery

When the issued access token is not sender-constrained, the webhook payload MUST omit the access token, refresh token, ID Token, and authorization code, and instead carry only non-authoritative result metadata. The client confirms the terminal state and obtains issued credentials by polling the token endpoint.

~~~ http
POST /completion-webhook HTTP/1.1
Host: client.example.com
Authorization: Bearer 9d2f6c4b8a1e7d3f5b9c0a2e4f6d8b1a
Content-Type: application/json

{
  "deferred_code": "8N5B2K1",
  "event_id": "2b7f0e6a-0c74-4e3e-8a7a-2efcb6c5b7d9",
  "issued_at": 1714073760,
  "status": "complete",
  "result_preview": {
    "token_type": "Bearer",
    "expires_in": 3600,
    "scope": "transfer.execute"
  }
}
~~~

The `result_preview` member is a non-authoritative summary of the access token response and MUST NOT contain access tokens, refresh tokens, ID Tokens, authorization codes, assertions, or other credentials. The client MUST NOT use any previewed value as granted authorization.

Authorization servers MUST select credential delivery only when all of the following hold:

* the issued access token is sender-constrained at issuance through DPoP, mTLS, or an equivalent mechanism the resource server verifies on every use;
* if a refresh token is included, it is sender-constrained by the same mechanism;
* if an ID Token is included, it carries a confirmation-method binding (such as a `cnf` claim) tying it to the client-held key or certificate.

If any of these conditions is not met, the authorization server MUST use preview-only delivery.

### Terminal Error

For terminal errors, `status` is set to `error` and the payload includes `error`:

~~~ json
{
  "deferred_code": "8N5B2K1",
  "event_id": "d33cf8fd-e57e-4a9d-9f1c-74d9c3904db6",
  "issued_at": 1714073760,
  "status": "error",
  "error": "access_denied"
}
~~~

### Confirmation and Cancellation

The webhook's HTTP response serves as the delivery confirmation:

- A 200 OK or 204 No Content response confirms delivery. Delivered credentials, if any, are committed and the deferred processing state is closed.
- Any other response, a network error, or timeout is unconfirmed delivery. The authorization server MAY retry delivery using exponential backoff or treat the delivery as failed and rely on the client polling.

If the deferred processing state is cancelled per the base specification's §Cancellation before delivery is confirmed, the authorization server MUST:

- suppress any pending or retry delivery,
- revoke the delivered access token and refresh token, if any, that were carried in a delivered-but-unconfirmed webhook payload, equivalently to revoking tokens obtained through polling.

Once delivery is confirmed, the delivered credentials are committed and cancellation does not retroactively invalidate them; cancellation may still prevent further continuation observations of the deferred processing state, but credentials in the client's possession are revoked only through their own mechanisms.

## Client Processing

A client receiving a webhook MUST validate the bearer credential against the most recent `deferred_code_push_delivery_token` issued for the identified `deferred_code`. The client MUST reject webhook deliveries with an `issued_at` value outside the client's configured freshness window and MUST detect replay of an `event_id` already accepted for the same deferred processing state.

When the webhook carries a `result` containing a sender-constrained access token response, the client MAY treat the response as authoritative on receipt. The client MUST verify that the issued access token is sender-constrained as expected (for example, that the access token is bound to the DPoP key the client used on the originating request); if the credential is not bound to the client's key or certificate as expected, the client MUST reject the delivery and SHOULD obtain the result by polling instead. The client returns 200 or 204 to confirm acceptance.

When the webhook carries `result_preview` instead, the client MUST NOT treat the preview as an access token response and MUST poll the token endpoint to obtain the issued credentials.

If push delivery fails (network error, validation failure, client rejection), the client SHOULD continue polling the `deferred_code` at the token endpoint per the substrate's continuation grant type. Polling is always available as an authoritative completion path.

## Authorization Server Behavior

The authorization server MAY deliver completion events via webhook to clients that have registered a `deferred_code_push_delivery_endpoint` and signaled `completion_mode=push`, provided deferred processing for the request is also authorized by `completion_mode=deferred`, client metadata, or an explicit profile rule. The authorization server MUST NOT depend on successful delivery; deferred processing state remains observable through continuation requests until delivery is confirmed or the state ends.

The authorization server MUST issue `deferred_code_push_delivery_token` values with sufficient entropy and bind each to a single deferred processing state, similar to the base specification's `notification_token` requirements.

The authorization server SHOULD apply rate limits to outgoing webhook deliveries and SHOULD validate the registered endpoint URL at registration time and prior to each delivery (SSRF protection per the base spec's notification endpoint requirements).

The authorization server MUST track the confirmation state of each delivery and MUST handle the cancellation race as defined above: revoke delivered-but-unconfirmed credentials when cancellation occurs, treat confirmed credentials as committed.

## Security Considerations

- The `deferred_code_push_delivery_token` is a bearer credential. Authorization servers MUST issue with high entropy, transmit only over TLS, bind to a single deferred processing state, and rotate or invalidate when the state ends.
- The webhook payload, when carrying a sender-constrained access token response, is sensitive metadata even though the credential is useless without the client's bound key or certificate. Authorization servers MUST ensure TLS to the registered endpoint and SHOULD consider end-to-end encryption (for example, JWE) for high-sensitivity deployments. Authorization servers SHOULD avoid logging webhook payloads in cleartext.
- Webhook payloads MUST NOT contain bearer access tokens, bearer refresh tokens, authorization codes, assertions, or ID Tokens that lack a confirmation-method binding. The authorization server MUST verify that the credentials selected for delivery satisfy these requirements before issuing the webhook.
- SSRF: the registered endpoint URL MUST be validated at registration time and prior to each delivery; the same loopback/link-local/non-public rejection rules apply as for the substrate's notification endpoint.
- Replay: delivery MUST include sufficient context (the `deferred_code`, the bound `deferred_code_push_delivery_token`, `event_id`, and `issued_at`) for the client to validate that the result is fresh, has not already been accepted, and corresponds to a request the client initiated. The sender-constraining proof at the resource server protects the delivered credential against replay by parties that do not hold the bound key.
- Cancellation race: delivered-but-unconfirmed credentials MUST be revoked if cancellation is received before the client confirms delivery. Failure to enforce this leaves a window in which a cancelled request can still produce usable credentials.
- All other security considerations of the base spec apply unchanged.

## IANA Considerations

This proposal would register:

**OAuth Completion Mode Values:**
| Value | Description |
|---|---|
| `push` | Client accepts webhook completion delivery |

**OAuth Authorization Server Metadata:**
| Name | Description |
|---|---|
| `deferred_code_push_delivery_supported` | Boolean indicating support for webhook completion delivery |

**OAuth Dynamic Client Registration Metadata:**
| Name | Description |
|---|---|
| `deferred_code_push_delivery_endpoint` | HTTPS URL where the authorization server delivers completion events |

**OAuth Parameters Registry:**
| Name | Usage |
|---|---|
| `deferred_code_push_delivery_token` | Token response, in continuation responses, to convey the bearer credential the AS will use for webhook authentication |

## Extensibility Validation Summary

This proposal exercises the following base-spec extension surfaces:

1. **Completion Mode value registration.** `push` is added via the Specification Required policy.
2. **AS metadata.** `deferred_code_push_delivery_supported` registered.
3. **Client metadata.** `deferred_code_push_delivery_endpoint` registered.
4. **Advisory delivery channel carrying sender-constrained credentials.** The webhook payload MAY carry an access token response when the credentials are sender-constrained, exercising the credential-delivery scope of §Profile-Defined Advisory Delivery Channels.
5. **Polling-equivalence and polling-availability preserved.** Polling at the token endpoint remains available as an alternative authoritative completion path and returns the same credentials.

**Verdict: no base-spec amendment required.** The base spec's generic advisory-delivery hook in §Profile-Defined Advisory Delivery Channels permits this proposal as a concrete instance, including credential delivery for sender-constrained tokens. The hook's requirements (authentication, freshness, cancellation handling, polling-equivalence, substrate-invariant preservation, and the bearer-credential exclusions) are satisfied by this proposal's specific design (bearer credential authentication of the webhook itself, event identifier and timestamp replay checks, HTTP-response-as-confirmation, sender-constrained credentials only, and polling fallback).

## Relationship to the Notification Mechanism

The base specification's §Notification mechanism pushes state-change hints (carrying only the `deferred_code`) to clients to reduce polling latency. This proposal is distinct: it pushes a completion event that MAY carry a sender-constrained access token response, not merely a state-change hint.

A client MAY register both endpoints. The notification endpoint receives state-change hints throughout deferred processing; the push delivery endpoint receives a final completion event. The two mechanisms use different bearer credentials (`notification_token` vs. `deferred_code_push_delivery_token`) to permit independent rotation and revocation.

The base specification's notification mechanism explicitly forbids carrying result data; this proposal does not change that. Result data (and, when sender-constrained, credential delivery) is carried only on the push delivery endpoint defined here.
