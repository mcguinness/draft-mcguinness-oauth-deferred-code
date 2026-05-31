# Proposals

This directory contains profile/extension proposals that build on [draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md), currently titled "OAuth 2.0 Asynchronous Request Processing". They exist to validate that the base specification's extension surfaces are sufficient to support real downstream profiles without modification.

These proposals are not on a publication path. They are exercises that demonstrate the extensibility model by exercising every declared extension surface of the base spec.

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

## How to read these proposals

Each proposal includes an "Extensibility Validation Summary" section that catalogs which base-spec extension surfaces it exercises and confirms whether base-spec changes would be required to advance the proposal toward publication.

The intent is that an honest reviewer can confirm "no base-spec change required" for each proposal by walking the checklist. The two proposals here both pass the test: the base spec is sufficient to support both extensions without modification. The base spec does this by providing generic extension hooks (new states, new response parameters, profile-defined out-of-band mechanisms with narrowing-only constraints) without naming any specific extension or anticipating any specific use case. The proposals layer their concrete designs on top of the generic hooks.

## Future proposals

Additional profile proposals MAY be added here as they are explored. Candidates that the base spec is designed to support:

- Additional notification mechanisms beyond the state-change hint defined in the base spec
- Profile-specific deferred-code lifetime classes (short polling, long governance)

Notes on candidates that were considered and not pursued:

- Partial-scope grants with continuation upgrade are supported directly by the base spec via §Partial Completion; they do not require a separate `grant_mode` value. A profile may still register a `grant_mode` value to mark requests that prefer partial completion over full deferral, but the wire shape and lifecycle are defined by the base spec.
- A `grant_mode=non-interactive` value was considered for grants that must complete without user interaction (server-to-server batch flows, autonomous agents without a present user) but was not pursued. The OpenID Connect `prompt=none` parameter covers most such use cases at the authorization endpoint, and pure OAuth grants (`client_credentials`, `refresh_token`, `jwt-bearer`, `token-exchange`) have no user to prompt by definition. Two parallel non-interactive vocabularies would add confusion without commensurate benefit.

None of the remaining candidates have been written up; they are listed here as known candidate use cases that the extensibility model is expected to accommodate.
