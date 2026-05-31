# Revisable Deferred Authorization for OAuth Asynchronous Request Processing

## Status

Proposal. This document defines a higher-layer profile of [draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md). It is presented to validate that specification's extension model for request revision, externally-observable states, and profile-defined out-of-band mechanisms. It is not itself on a publication path; a companion specification (e.g., Mission-Bound OAuth) would carry the final text.

## Abstract

This proposal defines a revisable deferred authorization profile for OAuth asynchronous request processing. The profile consists of the `revisable` value for the `grant_mode` parameter, the Revision Required externally-observable state, the `revision_required` token endpoint error, clarification response parameters, and a Pushed Authorization Requests (PAR) based revision submission mechanism. It allows an authorization server, when it determines that an originating request cannot be granted as stated but a narrowed version could be, to invite the client to push a narrowed revision and then continue polling with the existing deferred-code continuation mechanism rather than abandoning the request outright.

The motivating use case is Mission-Bound OAuth, where an autonomous agent proposes a "mission" (set of permissions and authorization details) and a human reviewer may approve a narrowed subset. Without this mechanism, the agent must abandon the original request and submit a new one, losing the deferred processing state and any preceding work.

## Scope

This proposal is intentionally narrow. It defines the protocol machinery required to revise an existing deferred processing state by narrowing the originating request. It does not define mission semantics, consent rendering, reviewer user experience, policy language, or authorization-server decision criteria.

The proposal is also not a general request-editing facility. Revisions can only reduce or clarify the originating request according to profile-defined comparison rules. The deferred code grant type remains unchanged, and continuation requests never carry revised authorization parameters.

## Relation to the Base Specification

This proposal uses the following extension surfaces defined by the base spec:

| Extension surface | Use |
|---|---|
| OAuth Grant Mode Values Registry (Specification Required policy) | Registers `revisable` |
| §Higher-Layer Extension Points: additional externally-observable states | Defines the Revision Required state |
| §Higher-Layer Extension Points: profile-defined response parameters for out-of-band mechanisms | Defines `clarification_handle`, `rejected_scope`, and `rejected_authorization_details` response parameters |
| §Continuation Request: profile-defined mechanisms for updating preserved parameters | Defines the PAR-based revision submission mechanism, subject to the base spec's narrowing-only constraint |

This proposal also touches the following IANA registries outside the base spec's own:

- OAuth Extensions Error Registry: registers `revision_required`
- OAuth Parameters Registry: registers `clarification_handle`, `rejected_scope`, and `rejected_authorization_details`
- Pushed Authorization Requests (RFC 9126) is the profile-chosen submission mechanism

The base specification's §Continuation Request includes a generic carve-out permitting "higher-layer profiles MAY define mechanisms outside the deferred code grant type itself for updating the parameters preserved in the deferred processing state" subject to a narrowing-only constraint. This proposal exercises that carve-out by specifying PAR as the concrete submission mechanism, the `clarification_handle` as the binding between PAR submissions and the deferred processing state, and the required narrowing checks for revised request parameters.

## Defined Value

**revisable**
: Client accepts a revisable grant. The authorization server MAY return a Revision Required response inviting the client to push a narrowed revision via PAR, in addition to or in place of denying the originating request outright. Revision Required is itself a deferred-processing condition and is observed through token endpoint responses carrying a `deferred_code`.

## Relationship to `grant_mode=deferred`

The `revisable` value authorizes only the Revision Required state and the clarification handshake defined by this proposal. It does not by itself indicate that the client accepts ordinary asynchronous deferral for reasons unrelated to revision, such as policy evaluation or external approval.

A client that accepts both ordinary deferral and revisable grants SHOULD send both values:

~~~ text
grant_mode=deferred revisable
~~~

A request containing only `grant_mode=revisable` allows the authorization server to return `revision_required` when the request cannot be granted as stated but could be granted after narrowing. If the same request requires ordinary asynchronous processing unrelated to revision, the authorization server MUST NOT infer support for that outcome from `revisable` alone.

## Defined State: Revision Required

When the authorization server determines that the originating request cannot be granted as stated, but a sufficiently narrowed version could be, and the client signaled `grant_mode=revisable`, the authorization server returns a Revision Required response in place of `access_denied`.

The Revision Required response is a token endpoint error response. It can be returned as the initial token endpoint response to an originating token request, or as a continuation response for an existing deferred request. For authorization endpoint originating requests, the authorization server first returns a deferred authorization response as defined by the base specification; Revision Required is then observed at the token endpoint during continuation.

### Wire Shape

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "error": "revision_required",
  "deferred_code": "8N5B2K1",
  "clarification_handle": "ch_4QFJ3P9",
  "rejected_scope": "crm:write",
  "rejected_authorization_details": [
    {"type": "transfer", "limit": "10000"}
  ],
  "expires_in": 540,
  "interval": 5
}
~~~

The response MUST include `error`, `deferred_code`, `clarification_handle`, and `expires_in`. The `interval` parameter follows the base specification's retained polling interval semantics: when present, it establishes or updates the retained polling interval for the deferred processing state, and subsequent responses do not need to repeat it.

The `clarification_handle` is bound to the deferred processing state and authorizes the client to push a revision via PAR. It is not a bearer authorization grant, access token, refresh token, or continuation handle; it is valid only as input to the revision submission mechanism defined by this proposal.

The `rejected_*` parameters are OPTIONAL. They provide the agent orchestrator with information about which aspects of the originating request were refused, allowing it to plan a narrowed proposal without needing additional out-of-band interaction. Authorization servers MAY omit either parameter when disclosure would reveal sensitive policy state or when the client can compute a narrowed request through other means.

## Client Processing

A client receiving `revision_required` MUST treat the originating request as not yet authorized. The client MUST NOT use the `rejected_scope`, `rejected_authorization_details`, or `clarification_handle` values as evidence of granted authorization.

The client SHOULD either submit a narrowed revision using the `clarification_handle`, continue polling according to the retained interval while waiting for a new state transition, or cancel the deferred request using the base specification's cancellation mechanism.

The client MUST use the most recent `deferred_code` returned by the authorization server. If a later Revision Required response rotates either the `deferred_code` or the `clarification_handle`, the client MUST discard the previous value and use the replacement.

## Revision Submission

The client submits a revised authorization request to the PAR endpoint, including the `clarification_handle` as an additional parameter:

~~~ http
POST /par HTTP/1.1
Host: as.example.com
Authorization: Basic czZCaGRSa3F0Mzo3RmpmcDBaQnIxS3REUmJuZlZkbUl3
Content-Type: application/x-www-form-urlencoded

response_type=code&
client_id=s6BhdRkqt3&
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb&
scope=financial%3Aread&
state=xyz&
code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
code_challenge_method=S256&
clarification_handle=ch_4QFJ3P9
~~~

This proposal uses PAR as an authenticated request-submission endpoint for revised authorization request parameters. The returned PAR `request_uri`, if any, is an artifact of the PAR protocol and is not used to drive the deferred-code continuation.

The authorization server:

1. Validates that `clarification_handle` is bound to an existing deferred processing state in the Revision Required condition
2. Validates that `clarification_handle` is unexpired, single-use, and bound to the same client and sender-constraining context as the deferred processing state
3. Validates that the revised request narrows the originating request: every requested scope, resource, audience, and authorization detail in the revised request MUST be no broader than the corresponding value in the originating request, and at least one refused dimension MUST be narrowed unless the profile explicitly allows metadata-only clarification
4. Validates that client identity matches the original request
5. Validates that sender-constraining material (DPoP key, mTLS certificate) matches the deferred processing state
6. Invalidates the `clarification_handle`
7. Updates the deferred processing state with the revised parameters
8. Transitions the state back to Pending (or Interaction Required, if the AS re-presents the narrowed request to the same reviewer)

If validation succeeds, the PAR response is a standard PAR success response. The returned `request_uri` exists only because the revision is submitted through the PAR endpoint; it is not a new continuation handle, and the client MUST NOT use it to initiate a separate authorization transaction. The current `deferred_code` remains the continuation handle. The client continues polling that `deferred_code` at the token endpoint to observe the re-reviewed state.

If validation fails (e.g., revision attempts to expand authorization), the PAR endpoint returns an appropriate PAR error response (`invalid_request` or a profile-defined error), and the deferred processing state remains in the Revision Required condition. Because each `clarification_handle` is single-use, a retry requires the client to obtain a new `clarification_handle` from a subsequent continuation response. The authorization server SHOULD issue a new handle in the next `revision_required` response unless local policy terminates the deferred request. The client MAY retry with a different revision only after receiving a new handle.

## Worked Example

An agent requests read and write access to a CRM resource so that it can update customer records after reviewing support tickets:

~~~ http
GET /authorize?
  response_type=code&
  client_id=s6BhdRkqt3&
  redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb&
  scope=crm%3Aread%20crm%3Awrite&
  state=xyz&
  code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
  code_challenge_method=S256&
  grant_mode=deferred%20revisable HTTP/1.1
Host: as.example.com
~~~

The authorization server cannot approve write access without human review, so it starts deferred processing and returns a deferred authorization response:

~~~ http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?
  deferred_code=dc_7M2R4&
  state=xyz&
  expires_in=600
~~~

The client begins continuation polling at the token endpoint. Because the originating authorization request used PKCE, the first continuation request includes the verifier as required by the base specification:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred_code&
deferred_code=dc_7M2R4&
code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk&
client_id=s6BhdRkqt3
~~~

During review, the reviewer determines that the agent may read CRM records but may not write them. Instead of denying the request outright, the authorization server returns `revision_required` as the continuation response:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "error": "revision_required",
  "deferred_code": "dc_9P2K7",
  "clarification_handle": "ch_4QFJ3P9",
  "rejected_scope": "crm:write",
  "expires_in": 420,
  "interval": 5
}
~~~

The client treats the request as not authorized. It constructs a narrowed request that removes `crm:write` and submits it to the PAR endpoint with the clarification handle:

~~~ http
POST /par HTTP/1.1
Host: as.example.com
Authorization: Basic czZCaGRSa3F0Mzo3RmpmcDBaQnIxS3REUmJuZlZkbUl3
Content-Type: application/x-www-form-urlencoded

response_type=code&
client_id=s6BhdRkqt3&
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb&
scope=crm%3Aread&
state=xyz&
code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
code_challenge_method=S256&
clarification_handle=ch_4QFJ3P9
~~~

The authorization server validates that the revised request is no broader than the originating request, invalidates `ch_4QFJ3P9`, updates the existing deferred processing state, and returns a standard PAR success response:

~~~ http
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store

{
  "request_uri": "urn:ietf:params:oauth:request_uri:6esc_11ACC5bwc014ltc14eY22c",
  "expires_in": 90
}
~~~

The client discards the returned `request_uri` for purposes of deferred processing. It continues polling the token endpoint with the current deferred code and the retained interval. Because PKCE verification occurred on the first continuation request, the `code_verifier` is not repeated:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred_code&
deferred_code=dc_9P2K7&
client_id=s6BhdRkqt3
~~~

If the narrowed request is approved, the authorization server completes the existing deferred processing state and returns a token response:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "crm:read"
}
~~~

## Applicability of PAR

The PAR-based revision mechanism is most natural when the originating request or the profile's revision vocabulary is authorization-request-shaped. Profiles applying revisable grants to token endpoint originating requests, such as token exchange or assertion grants, MUST define how token-request parameters map into a PAR revision request or MUST define a dedicated revision endpoint instead.

Profiles MUST NOT use the PAR `request_uri` returned from revision submission to create a new authorization transaction unless they explicitly terminate the deferred processing state and start a separate OAuth flow. The normal behavior of this proposal is to update the existing deferred processing state and continue using the existing `deferred_code`.

## State Machine Extension

~~~ ascii-art
... Pending or Interaction Required
        |
        v
+-----------------------+
|  Revision Required    |
+-----------------------+
        |
        +---(PAR with clarification_handle)--->
              |
              v
        Pending / Interaction Required
        (re-review of revised request)
              |
              v
        Complete / Denied / Expired
~~~

The Revision Required → Pending transition may occur multiple times (each revision triggers re-review). Profiles SHOULD bound the number of revision cycles per deferred processing state to prevent abuse.

If the authorization server determines that no acceptable narrowed revision remains possible, or if the client exceeds the profile-defined revision cycle bound, the authorization server SHOULD transition the deferred processing state to Denied and return `access_denied` on subsequent continuation requests.

## Security Considerations

- **Narrowing only.** Revisions MUST NOT expand the originating request. Authorization servers MUST validate narrowing per parameter (scope subset, resource subset, audience subset, authorization_details subset), allowing unchanged dimensions only when they remain no broader than the originating request. The narrowing comparison rules for `authorization_details` follow the inclusion semantics defined by RFC 9396 §6.
- **Clarification handle is single-use.** Each `clarification_handle` value MUST be invalidated after one PAR submission, whether the submission succeeded or failed validation. A new clarification handle is issued on subsequent Revision Required transitions.
- **Clarification handle lifetime.** The `clarification_handle` lifetime MUST NOT exceed the remaining lifetime of the deferred code. Authorization servers SHOULD use substantially shorter lifetimes when the handle is exposed to agent orchestration layers or other components outside the OAuth client.
- **Sender-constraining continuity.** The clarification handle MUST be sender-constrained equivalently to the `deferred_code`. An attacker that obtains the handle without the corresponding DPoP key or mTLS certificate MUST NOT be able to push a revision.
- **Request URI misuse.** The PAR `request_uri` returned after a successful revision submission MUST NOT become a second path to authorization. Clients discard it, and authorization servers SHOULD reject attempts to use it at the authorization endpoint unless a profile explicitly defines separate-flow behavior.
- **Policy disclosure.** The `rejected_scope` and `rejected_authorization_details` parameters can reveal policy boundaries, resource existence, approval rules, or reviewer decisions. Authorization servers SHOULD return only the minimum refusal detail needed for the client to construct a narrowed request and SHOULD omit these parameters when policy confidentiality is more important than automated revision.
- **Revision cycle bounding.** Authorization servers SHOULD bound the number of Revision Required → Pending transitions per deferred processing state (suggested: 3-5). Excessive cycles SHOULD be logged as a security event.
- **Audit logging.** Each revision submission MUST be logged with sufficient detail to support security investigation, including the originating request parameters, the revised parameters, and the narrowing comparison outcome.
- **Stale consent.** If the re-reviewed request is presented to a human reviewer, profiles SHOULD compute a fresh `consent_rendering_hash` or equivalent so that the reviewer's prior consent does not transfer to the revised request without re-acknowledgement.
- All other security considerations of the base spec apply unchanged.

## IANA Considerations

This proposal would register the following:

**OAuth Grant Mode Values registry** (established by the base spec):

| Value | Description | Specification |
|---|---|---|
| `revisable` | Client accepts a revisable grant inviting PAR-based narrowed revisions | (this proposal) |

**OAuth Extensions Error Registry** (established by RFC 6749):

| Name | Usage Location | Specification |
|---|---|---|
| `revision_required` | token endpoint response | (this proposal) |

**OAuth Parameters Registry** (established by RFC 6749):

| Name | Usage Location | Specification |
|---|---|---|
| `clarification_handle` | token endpoint response, PAR request | (this proposal) |
| `rejected_scope` | token endpoint response | (this proposal) |
| `rejected_authorization_details` | token endpoint response | (this proposal) |

## Extensibility Validation Summary

This proposal exercises the base specification's extension surfaces without requiring base-spec amendment:

1. **Grant Mode value registration.** `revisable` is added via the Specification Required policy already defined by the base spec.
2. **Profile-defined state.** Revision Required is added via the §Higher-Layer Extension Points hook for "additional externally-observable states extending the abstract state lifecycle." Mapped to a token endpoint error code (`revision_required`) per the base spec's pattern for distinguishing profile-defined states.
3. **Profile-defined response parameters.** `clarification_handle`, `rejected_scope`, and `rejected_authorization_details` are added via the §Higher-Layer Extension Points hook for "additional response parameters carrying handles or references used by profile-defined out-of-band mechanisms."
4. **Profile-defined parameter-update mechanism.** PAR-based revision submission is permitted by the base spec's §Continuation Request carve-out for "mechanisms outside the deferred code grant type itself for updating the parameters preserved in the deferred processing state," subject to the narrowing-only constraint. This proposal supplies the concrete mechanism (PAR) and the binding (`clarification_handle`) layered on top of the generic carve-out.
5. **Continuation polling unchanged.** The client polls the deferred_code at the token endpoint as defined by the base spec; revision adds a parallel back-channel for state updates but does not modify the polling state machine.
6. **Security model preserved.** Sender-constraining, deferred_code rotation, lifetime, replay, and oracle-resistance rules from the base spec apply unchanged; this proposal adds narrowing enforcement and handle single-use rules on top.

**Verdict: the extensibility model is sufficient.** Every extension dimension exercised by this proposal lands on a generic base-spec hook. The base spec does not name PAR, revision flows, the `revisable` value, or the Revision Required state; the proposal supplies all of those on top of the generic hooks the base spec provides for new states, new response parameters, and profile-defined parameter-update mechanisms.

## Relation to Mission-Bound OAuth

Mission-Bound OAuth (forthcoming) builds on this proposal by adding:

- Consent rendering hash computation for revised requests
- Agent orchestration patterns for planning narrowed revisions
- Mission-level abstractions over the underlying scope/resource/authorization_details vocabulary
- Multi-party approval workflows

None of these require changes to either this proposal or the base spec; they layer on top using the same extension surfaces.
