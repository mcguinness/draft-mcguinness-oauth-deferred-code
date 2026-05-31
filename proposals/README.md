# Proposals

This directory contains profile/extension proposals that build on [draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md). They exist to validate that the base specification's extension surfaces are sufficient to support real downstream profiles without modification.

These proposals are not on a publication path. They are exercises that demonstrate the extensibility model by exercising every declared extension surface of the base spec.

## Proposals in this directory

### [interim-grant-mode.md](./interim-grant-mode.md)

Defines `grant_mode=interim`. Allows the authorization server to return an initial response artifact (typically an OpenID Connect ID Token with currently-verified claims) together with a `deferred_code` for continuation, replacing the artifact with the complete version when extended verification finishes.

Exercises:
- OAuth Grant Mode Values Registry (registration)
- §Higher-Layer Extension Points (additional response parameters in the 200 OK token response)
- Continuation polling state machine (reused unchanged)

### [revisable-grant-mode.md](./revisable-grant-mode.md)

Defines `grant_mode=revisable`, the Revision Required abstract state, and a PAR-based clarification handshake mechanism. Allows the authorization server to invite a client to push a narrowed revision of the originating request via Pushed Authorization Requests when the original cannot be granted as stated but a narrowed version could be.

Exercises:
- OAuth Grant Mode Values Registry (registration)
- §Abstract State Status profile-extension hook (defines Revision Required state)
- §Higher-Layer Extension Points (profile-defined response parameters: `clarification_handle`, `rejected_scope`, `rejected_authorization_details`)
- §Continuation Request carve-out (PAR-based revision mechanism outside the continuation grant type)
- OAuth Extensions Error Registry (new error code `revision_required`)
- OAuth Parameters Registry (new parameters)

## How to read these proposals

Each proposal includes an "Extensibility Validation Summary" section that catalogs which base-spec extension surfaces it exercises and asserts whether base-spec changes would be required to support it. The intent is that an honest reviewer can confirm "no base-spec change required" for each proposal by walking the checklist.

If a proposal cannot be expressed without modifying the base spec, the proposal documents the gap and the base spec is amended. The two proposals here both pass the test: the base spec, as currently drafted on the `async-token-request-layer` branch, is sufficient to support both extensions without modification.

## Future proposals

Additional profile proposals MAY be added here as they are explored. Candidates that the base spec is designed to support:

- Additional notification mechanisms beyond the state-change hint defined in the base spec
- Profile-specific deferred-code lifetime classes (short polling, long governance)

Notes on candidates that were considered and not pursued:

- Partial-scope grants with continuation upgrade are supported directly by the base spec via §Partial Completion; they do not require a separate `grant_mode` value. A profile may still register a `grant_mode` value to mark requests that prefer partial completion over full deferral, but the wire shape and lifecycle are defined by the base spec.
- A `grant_mode=non-interactive` value was considered for grants that must complete without user interaction (server-to-server batch flows, autonomous agents without a present user) but was not pursued. The OpenID Connect `prompt=none` parameter covers most such use cases at the authorization endpoint, and pure OAuth grants (`client_credentials`, `refresh_token`, `jwt-bearer`, `token-exchange`) have no user to prompt by definition. Two parallel non-interactive vocabularies would add confusion without commensurate benefit.

None of the remaining candidates have been written up; they are listed here as known candidate use cases that the extensibility model is expected to accommodate.
