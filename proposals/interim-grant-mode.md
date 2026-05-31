# Interim Partial Completion for OAuth Asynchronous Request Processing

## Status

Proposal. This document defines a higher-layer profile of [draft-mcguinness-oauth-deferred-code-processing](../draft-mcguinness-oauth-deferred-code-processing.md). It is presented to validate that specification's extension model for partial completion, grant mode registration, and continuation polling. It is not itself on a publication path; a companion specification (e.g., an OpenID Connect profile) would carry the final text.

## Abstract

This proposal defines an interim partial-completion profile for OAuth asynchronous request processing. The profile consists of the `interim` value for the `grant_mode` parameter and replacement semantics for a partial response artifact returned together with a `deferred_code`. It allows a client to accept an initial response artifact (typically an OpenID Connect ID Token populated with currently-verified claims) and use the substrate's continuation polling mechanism to obtain the complete representation later.

The motivating use case is OpenID Connect identity verification flows where document review can take hours or days but the OpenID Provider can return preliminary claims (verified email, name) immediately while extended verification proceeds asynchronously.

## Scope

This proposal is intentionally narrow. It defines interim replacement semantics on top of the base specification's partial completion wire shape. It does not define OpenID Connect claim verification policy, identity-proofing rules, the marker used to identify an interim artifact, or relying-party trust decisions.

The proposal is not a general partial-issuance framework. Profiles that consume this proposal MUST define which artifacts can be issued on an interim basis, how the interim nature is marked, and how the interim artifact is invalidated or constrained after continuation completes unsuccessfully.

## Relation to the Base Specification

This proposal uses the following extension surfaces defined by the base spec:

| Extension surface | Use |
|---|---|
| OAuth Grant Mode Values Registry (Specification Required policy) | Registers `interim` |
| §Partial Completion | Defines the interim semantics on top of the partial completion wire shape (200 OK with issued artifact + `deferred_code` + `deferred_code_expires_in`) |
| §Higher-Layer Extension Points: additional response parameters | Allows a consuming profile to define a profile-specific marker indicating interim status |
| Notification mode | Signals when extended verification completes |

This proposal does NOT require:
- New externally-observable states (uses Pending only)
- New error codes
- Modifications to the continuation grant type
- Modifications to the deferred code lifecycle, sender-constraining rules, or security model

## Defined Value

**interim**
: Client accepts an interim grant. The authorization server MAY return an initial response artifact populated with currently-available claims together with a `deferred_code` for continuation. When deferred processing completes, the continuation response returns the complete artifact, replacing the interim version.

## Relationship to `grant_mode=deferred`

The `interim` value authorizes the partial-completion behavior defined by this proposal. It does not by itself indicate that the client accepts full deferral with no immediately issued artifact.

A client that accepts either an interim artifact or ordinary asynchronous deferral SHOULD send both values:

~~~ text
grant_mode=interim deferred
~~~

A request containing only `grant_mode=interim` allows the authorization server to return an interim partial-completion response. If the same request cannot produce an interim artifact and requires ordinary asynchronous processing, the authorization server MUST NOT infer support for full deferral from `interim` alone.

## Wire Shape

### Initial Response with Interim Representation

When a client sends `grant_mode=interim` at the token endpoint and the authorization server elects to produce an interim grant, the authorization server returns a partial completion response as defined in §Partial Completion of the base specification:

- A standard 200 OK token response carrying the interim representation (e.g., `id_token`) populated with currently-verified claims.
- `deferred_code` and `deferred_code_expires_in` parameters for continuation.

A profile using this grant mode MUST define how the interim nature of the representation is marked, either using a response parameter or a marker inside the artifact itself. This proposal does not mandate a specific marker; profiles that consume this proposal MUST specify one.

This proposal defines partial-completion semantics for the interim case: when the continuation completes successfully, the upgraded representation replaces the interim representation. The authorization server MUST invalidate the interim representation no later than when the continuation response returns the complete representation.

If deferred processing completes unsuccessfully, the authorization server returns the appropriate token endpoint error response. The profile MUST define whether the interim representation remains valid until its own expiration, is revoked immediately, or remains valid only for a narrowed purpose. If the unsuccessful completion invalidates any claim or authorization expressed by the interim representation, the authorization server MUST revoke or otherwise invalidate the interim representation.

### Continuation

The client polls the token endpoint using the deferred code grant type defined in the base spec. The response when complete returns a fresh 200 OK with the complete artifact replacing the interim version. The deferred_code is invalidated on completion as required by the base spec.

### Example

Token request signalling interim acceptance:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=SplxlOBeZQQYbYS6WxSbIA&
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb&
code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk&
grant_mode=interim&
client_id=s6BhdRkqt3
~~~

Interim response (verification in progress):

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "SlAV32hkKG",
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "eyJhbGciOiJSUzI1NiJ9...",
  "deferred_code": "8N5B2K1",
  "deferred_code_expires_in": 82800,
  "interval": 60
}
~~~

The `id_token` carries the verified-now claim set (e.g., email_verified, name); the `deferred_code` provides the continuation handle for the upgrade.

Continuation request:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred_code&
deferred_code=8N5B2K1&
client_id=s6BhdRkqt3
~~~

Polling response while verification continues:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "error": "authorization_pending",
  "deferred_code": "8N5B2K1",
  "interval": 60,
  "expires_in": 82800
}
~~~

Completion response (extended verification finished):

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "9u1Zq7...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "eyJhbGciOiJSUzI1NiJ9... (complete claims) ..."
}
~~~

The completion response carries the upgraded `id_token` in a token endpoint success response. The upgraded `id_token` replaces the interim `id_token`; access token handling follows the profile's token issuance policy.

## Security Considerations

- Interim representations carry a real subset of what will eventually be verified. Profiles using this mode MUST mark the interim representation as such so the relying party can apply appropriate trust decisions.
- Interim claims MUST NOT include claims that will not be present in the complete representation. Profiles MUST define which claims may appear in the interim representation.
- Profiles MUST define whether and how an interim representation is invalidated when deferred processing completes unsuccessfully.
- The `deferred_code` lifetime SHOULD match expected verification SLAs (typically measured in hours for document verification, days for manual review).
- For long-lifetime deferred codes, the §Deferred Processing Lifetime guidance of the base spec applies, including the required sender-constraining, one-time-use, explicit stakeholder approval semantic, and operational monitoring.
- All other security considerations of the base spec apply unchanged.

## IANA Considerations

This proposal would register a single value in the OAuth Grant Mode Values registry (established by the base spec):

| Value | Description | Specification |
|---|---|---|
| `interim` | Client accepts an interim grant with continuation upgrade | (this proposal) |

No other registry actions are required.

## Extensibility Validation Summary

This proposal demonstrates the base specification's extensibility model:

1. **Single new value registered via the existing Specification Required policy** — no base spec change required.
2. **State machine reused without modification** — interim flows use Pending and Complete states already defined by the base spec.
3. **Wire shape provided by §Partial Completion** — the base spec defines the 200 OK + issued artifact + `deferred_code` + `deferred_code_expires_in` pattern; this proposal supplies the interim semantics (artifact replacement on completion and the requirement for a profile-defined marker convention) on top of that pattern.
4. **Continuation grant type unchanged** — the client polls with the standard `urn:ietf:params:oauth:grant-type:deferred_code`.
5. **Security model unchanged** — sender-constraining, lifetime, replay, and oracle-resistance rules apply as defined in the base spec.

**Verdict: the extensibility model is sufficient.** No protocol-level changes to the base spec are required to support interim grants. Profile-specific concerns (which claims are interim-eligible, what marker indicates interim-ness on the artifact) live at the profile layer.
