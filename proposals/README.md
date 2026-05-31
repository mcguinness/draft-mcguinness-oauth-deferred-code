# Proposals

This directory contains profile/extension proposals that build on [draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md), currently titled "OAuth 2.0 Deferred Request Processing". They exist to validate that the base specification's extension surfaces are sufficient to support real downstream profiles without modification.

These proposals are not on a publication path. They exercise every declared extension surface of the base spec to test whether the substrate gets out of profile authors' way, not to claim that the substrate makes any of these proposals small. Each proposal is responsible for its own complexity; the substrate's contribution is that none required a base-spec amendment.

## Proposals in this directory

### [interim-grant-mode.md](./interim-grant-mode.md)

Defines `grant_mode=interim` and interim replacement semantics for the base specification's partial completion wire shape. Allows the authorization server to return an initial response artifact (typically an OpenID Connect ID Token with currently-verified claims) together with a `deferred_code` for continuation, replacing the artifact with the complete version when extended verification finishes.

Exercises:
- OAuth Grant Mode Values Registry (registration)
- §Partial Completion (profile-defined artifact replacement semantics)
- §Higher-Layer Extension Points (profile-defined marker for the interim artifact)
- Continuation polling state machine (reused unchanged)

### [revisable-grant-mode.md](./revisable-grant-mode.md)

Defines `grant_mode=revisable`, the Revision Required externally-observable state, and a PAR-based clarification handshake mechanism. Allows the authorization server to invite a client to push a narrowed revision of the originating request via Pushed Authorization Requests when the original cannot be granted as stated but a narrowed version could be.

Exercises:
- OAuth Grant Mode Values Registry (registration)
- §Higher-Layer Extension Points: additional externally-observable states (defines Revision Required state)
- §Higher-Layer Extension Points: profile-defined response parameters (`clarification_handle`, `rejected_scope`, `rejected_authorization_details`)
- §Continuation Request: profile-defined mechanisms for updating preserved parameters (PAR-based revision submission, subject to the base spec's narrowing-only constraint)
- OAuth Extensions Error Registry (new error code `revision_required`)
- OAuth Parameters Registry (new parameters)

### [webhook-result-delivery.md](./webhook-result-delivery.md)

Defines `grant_mode=push` and a webhook delivery profile that pushes a completion event to a client-registered HTTPS endpoint. When the issued access token is sender-constrained (DPoP-bound or mTLS-bound), the webhook MAY carry the access token response itself; otherwise it carries only non-authoritative result metadata. Polling at the token endpoint is always available as an alternative authoritative completion path.

Exercises:
- OAuth Grant Mode Values Registry (registration)
- §Profile-Defined Advisory Delivery Channels (generic hook), including the credential-delivery scope for sender-constrained credentials
- OAuth Authorization Server Metadata Registry (new `deferred_code_push_delivery_supported`)
- OAuth Dynamic Client Registration Metadata Registry (new `deferred_code_push_delivery_endpoint`)
- OAuth Parameters Registry (new `deferred_code_push_delivery_token`)
- HTTP-response-as-confirmation, cancellation-aware revocation for delivered-but-unconfirmed credentials, polling-equivalence

### [sse-streaming.md](./sse-streaming.md)

Defines `grant_mode=stream` and a Server-Sent Events streaming profile that streams deferred processing state transitions and a final completion event over a long-lived HTTPS connection. When the issued access token is sender-constrained, the `complete` event MAY carry the access token response itself; otherwise it carries only non-authoritative result metadata. Polling at the token endpoint is always available as an alternative authoritative completion path.

Exercises:
- OAuth Grant Mode Values Registry (registration)
- §Profile-Defined Advisory Delivery Channels (generic hook), including the credential-delivery scope and the use of a profile-defined authorization-server endpoint
- OAuth Authorization Server Metadata Registry (new `deferred_code_streaming_endpoint`, `deferred_code_streaming_supported`)
- OAuth Parameters Registry (new `deferred_code_stream_handle`)
- State vocabulary reuse: SSE event payloads reuse the substrate's `authorization_pending` / `interaction_required` / `slow_down` vocabulary
- Clean-connection-close-as-confirmation, cancellation-aware revocation for delivered-but-unconfirmed credentials, polling-equivalence

## How to read these proposals

Each proposal includes an "Extensibility Validation Summary" section that catalogs which base-spec extension surfaces it exercises and confirms whether base-spec changes would be required to advance the proposal toward publication.

All four proposals land entirely on existing base-spec extension surfaces; no base-spec changes are required for any of them. The proposals illustrate four distinct extension dimensions the substrate supports:

- **interim-grant-mode** — adds a new grant_mode value plus profile-defined artifact-replacement semantics on top of the partial completion wire shape.
- **revisable-grant-mode** — adds a new grant_mode value, a profile-defined externally-observable state, profile-defined response parameters, and a profile-defined parameter-update mechanism via the generic carve-out in §Continuation Request.
- **webhook-result-delivery** — adds a new grant_mode value plus a profile-defined advisory delivery channel (HTTPS POST to client-registered endpoint) that delivers sender-constrained credentials directly when applicable, via the credential-delivery scope of §Profile-Defined Advisory Delivery Channels.
- **sse-streaming** — adds a new grant_mode value, a short-lived `deferred_code_stream_handle`, and a profile-defined advisory delivery channel that uses a profile-defined authorization-server endpoint and delivers sender-constrained credentials in the terminal `complete` event when applicable, also via the credential-delivery scope of the advisory-delivery hook.

The base spec provides generic extension hooks (new states, new response parameters, profile-defined out-of-band parameter-update mechanisms, profile-defined advisory delivery channels) with security floors but without naming any specific extension or anticipating any specific use case. The proposals layer their concrete designs on top of those generic hooks.

## Composition

The four proposals each exercise one extension dimension. Real higher-layer profiles will often combine dimensions, and the base spec's composition rules (the grant_mode parameter's multi-value semantics, and the higher-layer extension points' obligation that profiles MUST specify how their parameters compose) are the load-bearing hand-off when they do.

The following is a **sketch**, not a worked proposal. It is included to motivate what a composed profile would look like, not to specify one. An OpenID Connect deferred ID Token issuance profile using both interim partial completion and webhook delivery would, at minimum:

- register a single profile-level grant_mode value (e.g., `interim-id-token`) and accept `grant_mode=deferred interim-id-token push` to signal that ordinary deferral, interim ID Token replacement, and webhook delivery are all acceptable for the request;
- inherit the interim wire shape (200 OK with `id_token`, `deferred_code`, `deferred_code_expires_in`) from §Partial Completion and the interim-grant-mode proposal;
- inherit webhook delivery for the upgraded ID Token from the webhook proposal under §Profile-Defined Advisory Delivery Channels, requiring that the upgraded ID Token carry a `cnf` confirmation-method binding so that webhook credential delivery applies rather than the preview-only fallback;
- specify, in its own text, the composition rules: that an interim artifact is delivered synchronously in the partial completion response, that the upgrade is delivered through webhook when the client has registered a push delivery endpoint and the upgraded credentials are sender-constrained, and that polling at the token endpoint remains available as an alternative authoritative completion path if the webhook delivery is missed.

The substrate does not write that profile; it provides the wire shapes, the grant_mode value space, and the always-available polling path that the profile would compose against. A composed profile of this kind has not been written up. Every proposal in this directory is currently single-dimension; whether the substrate's composition rules are sufficient in practice will only be answered once a composed profile is written.

## Future proposals

Additional profile proposals MAY be added here as they are explored. Candidates that the base spec is designed to support:

- Additional notification mechanisms beyond the state-change hint defined in the base spec
- Profile-specific deferred-code lifetime classes (short polling, long governance)

Notes on candidates that were considered and not pursued:

- Partial-scope grants with continuation upgrade are supported directly by the base spec via §Partial Completion; they do not require a separate `grant_mode` value. A profile may still register a `grant_mode` value to mark requests that prefer partial completion over full deferral, but the wire shape and lifecycle are defined by the base spec.
- A `grant_mode=non-interactive` value was considered for grants that must complete without user interaction (server-to-server batch flows, autonomous agents without a present user) but was not pursued. The OpenID Connect `prompt=none` parameter covers most such use cases at the authorization endpoint, and pure OAuth grants (`client_credentials`, `refresh_token`, `jwt-bearer`, `token-exchange`) have no user to prompt by definition. Two parallel non-interactive vocabularies would add confusion without commensurate benefit.

None of the remaining candidates have been written up; they are listed here as known candidate use cases that the extensibility model is expected to accommodate.
