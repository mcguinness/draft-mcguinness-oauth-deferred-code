# Proposals

This directory contains profile/extension proposals that build on [draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md) (the base specification, currently titled "OAuth 2.0 Deferred Request Processing") and, where applicable, on [draft-mcguinness-oauth-interim-grant-mode](../draft-mcguinness-oauth-interim-grant-mode.md) (the companion specification that defines the `grant_mode` parameter and its registry alongside the `interim` value).

These proposals exist to validate that the published specifications' extension surfaces are sufficient to support real downstream profiles without modification. They are not themselves on a publication path. They demonstrate extensibility, not simplicity: a profile can fit the substrate and still require substantial profile-specific security and composition rules.

The published specifications in this repository are:

- **[draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md)** — the substrate. Defines the deferred processing mechanism, `deferred_code`, polling state machine, partial completion wire shape, `completion_mode` parameter and its registry, and the substrate's other extension hooks.
- **[draft-mcguinness-oauth-interim-grant-mode](../draft-mcguinness-oauth-interim-grant-mode.md)** — the companion. Defines the `grant_mode` request parameter and the OAuth Grant Mode Values registry, and registers the first value, `interim`. Following the RFC 9396 model, the parameter framework is bundled with its first value rather than published as a standalone framework spec.

## Proposals in this directory

### [revisable-grant-mode.md](./revisable-grant-mode.md)

Defines `grant_mode=revisable`, the Revision Required externally-observable state, and a PAR-based clarification handshake mechanism. Allows the authorization server to invite a client to push a narrowed revision of the originating request via Pushed Authorization Requests when the original cannot be granted as stated but a narrowed version could be. Depends on the base spec and on [draft-mcguinness-oauth-interim-grant-mode](../draft-mcguinness-oauth-interim-grant-mode.md) for the `grant_mode` parameter and the OAuth Grant Mode Values registry.

Exercises:
- OAuth Grant Mode Values Registry (registers `revisable`, in the registry established by draft-mcguinness-oauth-interim-grant-mode)
- §Higher-Layer Extension Points (base spec): additional externally-observable states (defines Revision Required state)
- §Higher-Layer Extension Points (base spec): profile-defined response parameters (`clarification_handle`, `rejected_scope`, `rejected_authorization_details`)
- §Continuation Request (base spec): profile-defined mechanisms for updating preserved parameters (PAR-based revision submission, subject to the base spec's narrowing-only constraint)
- OAuth Extensions Error Registry (new error code `revision_required`)
- OAuth Parameters Registry (new parameters)

### [webhook-result-delivery.md](./webhook-result-delivery.md)

Defines `completion_mode=push` and a webhook delivery profile that pushes a completion event to a client-registered HTTPS endpoint. When the issued access token is sender-constrained (DPoP-bound or mTLS-bound), the webhook MAY carry the access token response itself; otherwise it carries only non-authoritative result metadata. Polling at the token endpoint is always available as an alternative authoritative completion path. Depends on the base spec only.

Exercises:
- OAuth Completion Mode Values Registry (registration)
- §Profile-Defined Advisory Delivery Channels (generic hook), including the credential-delivery scope for sender-constrained credentials
- OAuth Authorization Server Metadata Registry (new `deferred_code_push_delivery_supported`)
- OAuth Dynamic Client Registration Metadata Registry (new `deferred_code_push_delivery_endpoint`)
- OAuth Parameters Registry (new `deferred_code_push_delivery_token`)
- HTTP-response-as-confirmation, cancellation-aware revocation for delivered-but-unconfirmed credentials, polling-equivalence

### [sse-streaming.md](./sse-streaming.md)

Defines `completion_mode=stream` and a Server-Sent Events streaming profile that streams deferred processing state transitions and a final completion event over a long-lived HTTPS connection. When the issued access token is sender-constrained, the `complete` event MAY carry the access token response itself, but this uses weaker successful-write-as-commit semantics rather than explicit acknowledgement; profiles that do not accept that tradeoff use non-authoritative result metadata and polling. Polling at the token endpoint is always available as an alternative authoritative completion path. Depends on the base spec only.

Exercises:
- OAuth Completion Mode Values Registry (registration)
- §Profile-Defined Advisory Delivery Channels (generic hook), including the credential-delivery scope and the use of a profile-defined authorization-server endpoint
- OAuth Authorization Server Metadata Registry (new `deferred_code_streaming_endpoint`, `deferred_code_streaming_supported`)
- OAuth Parameters Registry (new `deferred_code_stream_handle`)
- State vocabulary reuse: SSE event payloads reuse the substrate's `authorization_pending` / `interaction_required` / `slow_down` vocabulary
- Successful-write-as-commit, cancellation-before-write suppression, polling-equivalence

## How to read these proposals

Each proposal includes an "Extensibility Validation Summary" section that catalogs which extension surfaces it exercises and confirms whether changes to the base spec or the interim-grant-mode spec would be required to advance the proposal toward publication. None of the proposals require such changes, but that does not mean every design choice in the proposal is equally strong or equally reusable.

The proposals illustrate distinct extension dimensions:

- **revisable-grant-mode** — registers a `grant_mode` value, defines a profile-defined externally-observable state, profile-defined response parameters, and a profile-defined parameter-update mechanism via the generic carve-out in §Continuation Request.
- **webhook-result-delivery** — adds a new `completion_mode` value plus a profile-defined advisory delivery channel (HTTPS POST to client-registered endpoint) that delivers sender-constrained credentials directly when applicable, via the credential-delivery scope of §Profile-Defined Advisory Delivery Channels.
- **sse-streaming** — adds a new `completion_mode` value, a short-lived `deferred_code_stream_handle`, and a profile-defined advisory delivery channel that uses a profile-defined authorization-server endpoint. It is strongest as a state-streaming and preview-delivery profile; its optional sender-constrained credential delivery exercises the edge of the advisory-delivery hook.

The base spec provides generic extension hooks (new states, new response parameters, profile-defined out-of-band parameter-update mechanisms, profile-defined advisory delivery channels, the `completion_mode` parameter and its registry) with security floors but without naming any specific extension or anticipating any specific use case. The interim-grant-mode spec provides the `grant_mode` parameter and its registry. The proposals layer their concrete designs on top of those generic mechanisms.

## Composition

The three proposals each exercise one extension dimension. Real higher-layer profiles will often combine dimensions, and the composition rules — the `completion_mode` multi-value semantics defined by the base spec, the `grant_mode` multi-value semantics defined by draft-mcguinness-oauth-interim-grant-mode, the orthogonality of the two parameters, and the higher-layer extension points' obligation that profiles MUST specify how their parameters compose — are the load-bearing hand-off when they do.

The following is a **sketch**, not a worked proposal. It is included to motivate what a composed profile would look like, not to specify one. An OpenID Connect deferred ID Token issuance profile using both interim partial completion and webhook delivery would, at minimum:

- depend on the base spec, draft-mcguinness-oauth-interim-grant-mode (for both `grant_mode` parameter mechanics and `interim` semantics), and the webhook-result-delivery proposal;
- reuse `grant_mode=interim` from draft-mcguinness-oauth-interim-grant-mode and define, within this profile, which artifact in the partial completion response is interim (the `id_token`) and the marker that identifies it as such (for example, an `interim` claim in the ID Token). The profile accepts `completion_mode=deferred push grant_mode=interim` on the request to signal that ordinary deferred completion, webhook delivery, and interim ID Token replacement are all acceptable;
- inherit the interim wire shape (200 OK with `id_token`, `deferred_code`, `deferred_code_expires_in`) from draft-mcguinness-oauth-interim-grant-mode and the base spec's §Partial Completion;
- inherit webhook delivery for the upgraded ID Token from the webhook proposal under §Profile-Defined Advisory Delivery Channels, requiring that the upgraded ID Token carry a `cnf` confirmation-method binding so that webhook credential delivery applies rather than the preview-only fallback;
- specify, in its own text, the composition rules: that an interim artifact is delivered synchronously in the partial completion response, that the upgrade is delivered through webhook when the client has registered a push delivery endpoint and the upgraded credentials are sender-constrained, and that polling at the token endpoint remains available as an alternative authoritative completion path if the webhook delivery is missed.

The substrate provides the wire shapes, the `completion_mode` value space, and the always-available polling path that the profile composes against. draft-mcguinness-oauth-interim-grant-mode provides the `grant_mode` parameter and registry. A composed profile of this kind has not been written up. Every proposal in this directory is currently single-dimension; whether the composition rules are sufficient in practice will only be answered once a composed profile is written.

## Future proposals

Additional profile proposals MAY be added here as they are explored. Candidates that the published specifications are designed to support:

- Additional notification mechanisms beyond the state-change hint defined in the base spec
- Profile-specific deferred-code lifetime classes (short polling, long governance)

Notes on candidates that were considered and not pursued:

- Partial-scope grants with continuation upgrade are supported directly by the base spec via §Partial Completion; they do not require any new per-request signal beyond what the substrate already defines. A profile may still register a `grant_mode` value in the registry established by draft-mcguinness-oauth-interim-grant-mode to mark requests that prefer partial completion over full deferral, but the wire shape and lifecycle are defined by the base spec.
- A `non-interactive` mode value was considered for grants that must complete without user interaction (server-to-server batch flows, autonomous agents without a present user) but was not pursued. The OpenID Connect `prompt=none` parameter covers most such use cases at the authorization endpoint, and pure OAuth grants (`client_credentials`, `refresh_token`, `jwt-bearer`, `token-exchange`) have no user to prompt by definition. Two parallel non-interactive vocabularies would add confusion without commensurate benefit.

None of the remaining candidates have been written up; they are listed here as known candidate use cases that the extensibility model is expected to accommodate.
