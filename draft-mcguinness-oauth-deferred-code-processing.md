---
title: "OAuth 2.0 Deferred Code Processing"
abbrev: oauth-deferred-code
docname: draft-mcguinness-oauth-deferred-code-processing-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Deferred Processing
 - Token Endpoint
 - Token Exchange

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC6750:
  RFC6755:
  RFC7009:
  RFC7521:
  RFC7523:
  RFC7591:
  RFC7636:
  RFC8414:
  RFC8628:
  RFC8693:
  RFC8705:
  RFC8707:
  RFC9396:
  RFC9449:
  RFC9700:

informative:
  CIBA:
    target: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
    title: "OpenID Client Initiated Backchannel Authentication Core"
  DTR:
    target: https://gniero.github.io/oidc-dtr-resources/
    title: "OpenID Connect Deferred Transaction Resources"
  JWT-GRANT-INTERACTION:
    target: https://datatracker.ietf.org/doc/draft-parecki-oauth-jwt-grant-interaction-response/
    title: "OAuth JWT Grant Interaction Response"
  OIDC-CORE:
    target: https://openid.net/specs/openid-connect-core-1_0.html
    title: "OpenID Connect Core 1.0"

--- abstract

This specification defines deferred processing semantics for OAuth token requests that cannot complete synchronously, by introducing a continuation mechanism at the token endpoint.

An authorization server can defer completion of any OAuth token request and return a `deferred_code` representing suspended token processing state. Clients can later resume processing using the deferred code grant type.

This specification also defines optional interaction continuation semantics for deferred requests that require external interaction.

This specification intentionally remains at the OAuth layer and does not define authentication ceremonies, identity semantics, consent semantics, or ID Token processing rules. Higher-layer protocols such as OpenID Connect MAY define profiles using the mechanisms defined by this specification.

--- middle

# Introduction {#introduction}

Traditional OAuth token processing assumes that token issuance decisions can be completed synchronously during evaluation of the token request.

In practice, many deployments require token issuance to pause pending external processing, including:

* policy and risk evaluation, including token exchange policies and external policy engines,
* attestation validation for workloads, devices, or transactions,
* enterprise approval or delegated authorization workflows,
* identity proofing and transaction verification,
* cross-domain authorization orchestration.

Existing OAuth and OpenID Connect specifications address portions of this problem for specific protocols or grant types. This specification instead defines a generalized deferred execution model for OAuth token endpoint processing.

This specification separates token processing state management from higher-layer identity, authentication, and user interaction semantics.

This specification defines:

* deferred processing semantics for token requests,
* the `deferred_code` continuation reference,
* the deferred code grant type,
* optional interaction continuation semantics,
* continuation polling guidance,
* authorization server metadata extensions.

This specification does not define:

* user authentication semantics,
* browser interaction models,
* identity semantics,
* ID Token processing semantics,
* consent ceremony requirements,
* transaction presentation requirements.

Protocols such as OpenID Connect MAY define profiles and additional processing requirements using the mechanisms defined by this specification.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

Deferred Code:
: An opaque continuation reference representing deferred processing state associated with a previously submitted token request.

Deferred Code Grant Type:
: A continuation grant type encoded using the OAuth extension grant mechanism to resume processing of a previously deferred request.

Interaction URI:
: A URI where external interaction associated with a deferred request can occur.

Deferred Request:
: A token request whose processing has been deferred by the authorization server.

Deferred Processing State:
: Authorization server state associated with a deferred request, including the original token request context, client security context, processing status, expiration, and any implementation-defined asynchronous processing context.

Continuation Request:
: An access token request using the deferred code grant type to resume processing of a deferred request.

Continuation Response:
: A token endpoint response to a continuation request.

# Architectural Model

This specification introduces deferred execution semantics to OAuth token processing.

The authorization server can temporarily suspend completion of a token request while external interaction or asynchronous evaluation occurs. Suspension does not create a new authorization transaction. It preserves the original token request and resumes evaluation of that request later.

This specification uses a continuation architecture with three responsibilities:

* The client submits an ordinary OAuth token request.
* The authorization server determines that token processing cannot complete synchronously and creates deferred processing state.
* The client resumes the suspended processing by presenting a deferred code at the token endpoint.

The OAuth layer is responsible only for:

* deferred processing state management,
* continuation correlation,
* deferred completion signaling,
* continuation semantics,
* optional interaction continuation references.

Higher-layer protocols remain responsible for:

* user authentication,
* identity semantics,
* consent semantics,
* authentication ceremony,
* transaction presentation,
* identity proofing,
* assurance evaluation.

This separation allows higher-layer protocols such as OpenID Connect to profile this specification without embedding identity-specific behavior into the OAuth substrate.

## Design Goals

This specification is designed to:

* preserve existing OAuth grant semantics,
* allow deferred completion for arbitrary token endpoint requests,
* keep the continuation interface at the token endpoint,
* avoid client interpretation of server-side processing state,
* support both interactive and non-interactive asynchronous processing,
* allow higher-layer profiles to define user-facing or identity-specific behavior.

## Design Non-Goals

This specification does not define:

* a new authorization endpoint flow,
* a redirect-based interaction protocol,
* a user authentication protocol,
* push delivery of access tokens or identity assertions,
* semantics for the external interaction,
* resource server processing for deferred codes.

The notification mode defined in {{notification}} signals state changes only and does not deliver tokens or identity claims to the client.

## Processing Model

The authorization server processes an initial access token request according to the rules of the requested grant type. During that processing, the authorization server can reach one of the following outcomes:

* the request fails synchronously and the authorization server returns a token endpoint error response;
* the request succeeds synchronously and the authorization server returns an access token response;
* the request is valid but cannot complete synchronously, so the authorization server creates deferred processing state and returns a deferred processing response.

The deferred processing state is an authorization server internal record. It is referenced by the deferred code but is not exposed to the client. The state MUST preserve the original token request context needed to complete token processing. The client MUST NOT resubmit the original grant parameters during continuation.

Continuation requests resume evaluation of the original token request. The authorization server uses the deferred code to locate deferred processing state, verifies the continuation request against the binding requirements for that state, and returns one of the continuation responses defined in this specification.

## Deferred Processing State

Deferred processing state has an implementation-defined representation, but it MUST be sufficient to enforce the requirements of this specification.

At minimum, the authorization server MUST associate deferred processing state with:

* the original token request parameters needed to complete processing,
* the client identity or public-client binding information associated with the original request,
* sender-constraining or proof-of-possession requirements associated with the original request,
* the current processing status,
* expiration time or expiration policy,
* any deferred code values currently valid for the state.

The authorization server SHOULD also associate deferred processing state with requested resources, requested scopes, grant-specific input artifacts, policy evaluation inputs, interaction status, and audit information needed for security monitoring.

Deferred processing state MUST NOT be used to expand authorization beyond the original token request. Any authorization, approval, authentication, or policy result obtained during deferred processing can only constrain or complete the original request.

## State Lifecycle

A deferred request follows this abstract lifecycle:

~~~ ascii-art
 Initial token request
          |
          v
 +--------------------+
 | Access token       |
 | processing         |
 +--------------------+
    |        |       |
    |        |       +--> access token response
    |        |
    |        +----------> token endpoint error response
    |
    v
 deferred processing response
    |
    v
 +--------------------+       continuation request
 | Deferred state     | <----------------------+
 | pending / waiting  |                        |
 +--------------------+                        |
    |        |       |                         |
    |        |       +--> continuation error --+
    |        |
    |        +----------> access token response
    |
    +------------------> terminal error response
~~~

The diagram is descriptive. The authorization server can rotate deferred codes, transition between pending and interaction-required conditions, or terminate processing at any point according to local policy. For clarity, the diagram does not depict every transition defined by this specification: client-initiated cancellation (see {{cancellation}}) terminates deferred processing state from the Pending or Interaction Required conditions, and authorization-server-initiated notifications (see {{notification}}) MAY accompany any state transition.

Deferred processing state ends when the request completes successfully, fails permanently, expires, is revoked, is canceled by the client, or is otherwise terminated by the authorization server.

## Abstract State Status

Authorization servers can use any internal representation for deferred processing state. For interoperability, the externally observable status of that state is mapped to token endpoint responses as follows:

Pending:
: Processing is in progress and no external interaction is required. The authorization server returns `authorization_pending`.

Interaction Required:
: Processing cannot continue until an external interaction occurs. The authorization server returns `interaction_required` and an `interaction_uri`.

Complete:
: Processing completed successfully. The authorization server returns an access token response for the original token request.

Denied:
: Processing completed unsuccessfully because the original token request is not authorized. The authorization server returns `access_denied` when the unsuccessful outcome is the result of an authorization decision reached during deferred processing, or another token endpoint error response when more specific to the reason for failure (see {{unsuccessful-completion}}).

Expired:
: The deferred processing state or deferred code is no longer valid due to expiration. The authorization server returns `expired_token`.

Invalid:
: The deferred code cannot be used because it is malformed, unknown, revoked, not bound to the requesting client, or otherwise invalid. The authorization server returns `invalid_grant`. Deferred processing state terminated by client-initiated cancellation as defined in {{cancellation}} is observed externally as Invalid.

The authorization server MAY transition between Pending and Interaction Required more than once. Complete, Denied, Expired, and Invalid are terminal from the client's perspective.

# Deferred Code Semantics {#deferred-code-semantics}

A `deferred_code` is an opaque continuation reference representing deferred processing state associated with a previously submitted token request.

A `deferred_code`:

* is not an access token,
* is not a refresh token,
* is not intended for resource server access,
* does not represent authorization delegation by itself,
* is only valid at the authorization server token endpoint,
* MUST be treated as opaque by clients.

The authorization server MUST bind the `deferred_code` to the original token request and associated client security context.

The authorization server MAY rotate deferred codes during continuation processing.

If the authorization server issues a replacement deferred code, the client MUST discard the previous value.

If a deferred code is not sender-constrained or otherwise bound to proof-of-possession material, the authorization server MUST rotate the deferred code after each valid continuation request that returns `authorization_pending`, `interaction_required`, or `slow_down`. Authorization servers MUST use one-time-use deferred codes or an equivalent replay-resistant mechanism for deferred requests involving one-time credentials, high-value resources, public clients, or externally exposed interaction URIs.

# Deferred Processing

When processing a token request, the authorization server MAY defer completion of the request.

Deferred processing allows token issuance to continue asynchronously after the initial token request has been evaluated.

A deferred request MAY involve:

* asynchronous policy evaluation,
* external orchestration,
* enterprise workflow execution,
* workload validation,
* user interaction,
* authentication continuation,
* transaction review,
* other implementation-defined processing.

The authorization server communicates deferred processing state to the client through:

* deferred codes,
* continuation responses,
* optional interaction continuation references.

## Client Behavior

A client that receives a deferred processing response MUST NOT treat the response as successful token issuance. The client can either abandon the deferred request or submit continuation requests according to this specification.

The client SHOULD wait at least the number of seconds indicated by the `interval` parameter before submitting a continuation request. If no `interval` value is provided, the client SHOULD use a reasonable default polling interval and apply backoff after repeated pending responses.

The client MUST use the most recent deferred code value returned by the authorization server. If a continuation response includes a replacement deferred code, the client MUST discard the previous value and use the replacement value for subsequent continuation requests.

A client MAY present an `interaction_uri` to a user or another external actor when appropriate for the client application and higher-layer profile. This specification does not require the client to dereference the `interaction_uri`, embed it in a user agent, or use any particular interaction channel.

## Authorization Server Behavior

The authorization server is responsible for deciding whether a token request is eligible for deferred processing. The authorization server MAY apply local policy, grant-type-specific policy, client policy, resource policy, or higher-layer profile requirements when making that decision.

The authorization server MUST make continuation responses depend on the deferred processing state associated with the deferred code, not on new grant parameters supplied by the client.

The authorization server MAY complete deferred processing independently of client polling. For example, external policy evaluation, administrator approval, attestation validation, or user interaction can update the deferred processing state before the next continuation request.

The authorization server SHOULD avoid exposing sensitive policy decisions through differences in whether a request is deferred, denied immediately, or left pending. Where policy confidentiality is important, authorization servers SHOULD normalize response timing, error selection, and polling behavior to reduce probing or oracle attacks.

## Higher-Layer Extension Points

Higher-layer profiles can build on this specification by defining:

* when an authorization server returns `interaction_required`,
* the semantics and user experience associated with an `interaction_uri`,
* additional response parameters for interaction or transaction context,
* completion criteria for external authentication, approval, consent, or review,
* additional notification mechanisms that reduce or eliminate client polling,
* constraints on which grant types, clients, or resources can use deferred processing.

Higher-layer profiles MUST NOT require clients to interpret the `deferred_code` value.

# Token Endpoint Extensions

The following sections extend the OAuth token endpoint defined in {{RFC6749}}.

## Deferred Processing Responses

When the authorization server defers processing of a token request, it returns a token endpoint error response indicating deferred processing state.

Before issuing a deferred code, the authorization server MUST authenticate the client when client authentication is required and MUST validate the token request to the extent necessary to determine eligibility for deferred processing. The authorization server MUST NOT issue a deferred code for a malformed request, for an unauthenticated client that is required to authenticate, or for a request that is known to be unauthorized.

For unauthenticated clients, including public clients, the authorization server MUST NOT issue a deferred code unless the deferred processing state is bound to additional context that the requesting client instance MUST demonstrate on each continuation request. Acceptable bindings include DPoP {{RFC9449}}, mutual-TLS client certificate binding {{RFC8705}}, an authenticated user-agent session, a device-bound context, or an equivalent local mitigation. The binding MUST be sufficient to prevent another client instance that obtains the deferred code from completing the continuation.

Preserved PKCE {{RFC7636}} verification state alone is NOT a sufficient continuation binding. PKCE verification is performed once during evaluation of the original token request, the `code_verifier` is consumed at that point, and continuation requests do not re-present it (see {{continuation-request}}). PKCE therefore proves the original requestor possessed the verifier but does not authenticate the continuation requestor. PKCE MAY be combined with a sender-constraining or session-binding mechanism listed above to satisfy this requirement.

The authorization server MAY return either:

* `authorization_pending`
* `interaction_required`

This specification reuses the `authorization_pending`, `slow_down`, and `expired_token` token endpoint error codes defined by {{RFC8628}} with the generalized semantics defined in {{deferred-error-semantics}}. This specification uses the `interaction_required` error code registered by OpenID Connect {{OIDC-CORE}} and updates its registration to add token endpoint response usage for deferred code processing.

The response MUST include `deferred_code` and `expires_in` parameters. The response SHOULD include an `interval` parameter unless the authorization server has no client polling expectation.

These responses use HTTP status code `400 Bad Request` and the error response encoding defined for token endpoint error responses in {{RFC6749}}, consistent with the polling encoding of {{RFC8628}}. The error response form is used for non-failure conditions such as `authorization_pending`, `interaction_required`, and `slow_down` to preserve wire-level compatibility with existing OAuth token endpoint error parsing.

Deferred processing responses and continuation responses MUST be returned with `Cache-Control: no-store` and `Pragma: no-cache` HTTP response headers, as required for token endpoint responses by {{RFC6749}}. Intermediate caches MUST NOT retain `deferred_code`, `notification_token`, `interaction_uri`, or any associated parameters.

## Deferred Error Semantics {#deferred-error-semantics}

The following token endpoint error values are used by this specification:

authorization_pending:
: Deferred processing state exists, but processing of the original token request has not completed.

interaction_required:
: Deferred processing cannot continue until external interaction associated with the deferred request occurs.

slow_down:
: The client is polling more frequently than permitted by the authorization server. The client MUST increase the polling interval before sending another continuation request.

expired_token:
: The deferred code or deferred processing state has expired and can no longer be used.

invalid_grant:
: The deferred code is malformed, unknown, revoked, not bound to the requesting client, or otherwise invalid.

The use of `authorization_pending`, `slow_down`, and `expired_token` aligns with the token endpoint polling model of the OAuth Device Authorization Grant {{RFC8628}}. In this specification, those errors refer to deferred processing state rather than device authorization state. The error name `expired_token` is inherited verbatim from {{RFC8628}}; its `_token` suffix is part of the registered error code vocabulary and does not refer to the `deferred_code` parameter defined by this specification. The use of `interaction_required` aligns with OpenID Connect interaction terminology, but in this specification it appears at the token endpoint to indicate deferred processing state waiting for external interaction.

## Authorization Pending

The `authorization_pending` error indicates deferred processing remains ongoing.

The client MAY submit a continuation request after waiting at least the number of seconds indicated by the `interval` parameter, if present.

The response MAY include an `error_description` parameter conveying short, human-readable progress information for display to a user, such as a queue position, processing step, or estimated time remaining. Authorization servers MUST NOT include identifiers, policy decisions, attestation outcomes, risk signals, or any other information that would reveal sensitive state or enable oracle-style probing. Clients MUST NOT depend on the format or content of `error_description` for protocol decisions.

### Example

~~~ json
{
  "error": "authorization_pending",
  "deferred_code": "4QFJ3P9",
  "interval": 5,
  "expires_in": 900
}
~~~

## Interaction Required

The `interaction_required` error indicates external interaction is needed before processing can continue.

The response MUST include an `interaction_uri` parameter.

### Example

~~~ json
{
  "error": "interaction_required",
  "deferred_code": "4QFJ3P9",
  "interaction_uri":
    "https://as.example.com/interact/4QFJ3P9",
  "interval": 5,
  "expires_in": 900
}
~~~

## deferred_code

The `deferred_code` parameter is an opaque continuation reference representing deferred processing state.

Clients MUST treat the value as opaque.

The authorization server MUST NOT encode authorization, identity, or policy decisions in a way that requires client interpretation of the value.

## interaction_uri

The `interaction_uri` parameter identifies a location where required external interaction can occur.

This specification does not define the semantics of the interaction.

The `interaction_uri` parameter MUST be an HTTPS URI. It MUST NOT include a fragment component.

Unless defined otherwise by a higher-layer profile, authorization servers SHOULD generate the `interaction_uri` using an origin trusted for the authorization server. Trust MAY be established through the issuer identifier, authorization server metadata, prior client configuration, or by matching the token endpoint origin. If an `interaction_uri` uses an origin different from both the issuer identifier and the token endpoint origin, the authorization server or higher-layer profile MUST define how clients determine that the origin is trusted for the authorization server.

The authorization server MUST bind the `interaction_uri` to the deferred processing state associated with the returned deferred code. Accessing the `interaction_uri` MUST NOT by itself authorize the original token request or complete deferred processing. Completion occurs only when the authorization server updates the deferred processing state according to local policy or a higher-layer profile.

The authorization server MUST make interaction URIs single-use or otherwise resistant to replay. The authorization server MUST limit the lifetime of interaction URIs to no longer than the lifetime of the associated deferred code.

If interaction at the `interaction_uri` can affect deferred processing state, the authorization server MUST authenticate the actor performing the interaction or bind the interaction to an existing authenticated session, device context, transaction challenge, or higher-layer profile mechanism sufficient for the sensitivity of the original token request.

The authorization server MUST NOT require clients to parse the `interaction_uri` or extract state from it. Clients MUST treat the `interaction_uri` as an opaque URI.

## interval

The `interval` parameter indicates the minimum number of seconds the client SHOULD wait before submitting another continuation request.

If the client polls more frequently than permitted by the authorization server, the authorization server MAY return `slow_down` as defined by {{RFC8628}} or another token endpoint error response appropriate to the condition.

## expires_in

In a deferred processing response and in continuation responses returning `authorization_pending`, `interaction_required`, or `slow_down`, the `expires_in` parameter is REQUIRED and indicates the remaining lifetime of the deferred code in seconds. This usage parallels the `expires_in` parameter in the OAuth Device Authorization Grant {{RFC8628}} response, which describes the device code lifetime rather than an access token lifetime.

In a successful access token response returned upon deferred completion, the `expires_in` parameter retains the meaning defined for the original grant type and describes the lifetime of the issued access token.

Clients MUST NOT infer the lifetime of a deferred code from the lifetime of any access token that might be issued upon successful completion.

# Deferred Code Grant Type

The deferred code grant type uses the OAuth extension grant mechanism defined by {{RFC6749}} to continue processing of a previously submitted token request.

The deferred code grant type is a continuation mechanism. It does not create a new authorization grant from the resource owner, client, authorization server, or external approver. The authorization grant, if any, remains the authorization grant represented by the original token request.

Authorization semantics are derived exclusively from the original token request associated with the deferred code.

The deferred code grant type MUST NOT:

* modify the original token request,
* expand requested authorization,
* change requested resources,
* alter sender-constraining requirements,
* change the authenticated client identity associated with the original request.

The authorization server MUST bind the deferred code to the original token request and its associated security context.

## Grant Type

The deferred code grant type value is:

~~~ text
urn:ietf:params:oauth:grant-type:deferred_code
~~~

## Continuation Request {#continuation-request}

The client MAY submit a continuation request using the deferred code grant type to resume processing of a deferred request.

The request MUST use the token endpoint. The request MUST include the following parameters using the `application/x-www-form-urlencoded` format defined by {{RFC6749}}:

grant_type:
: REQUIRED. Value MUST be `urn:ietf:params:oauth:grant-type:deferred_code`.

deferred_code:
: REQUIRED. The deferred code previously returned by the authorization server.

client_id:
: REQUIRED if the client is not authenticating with the authorization server and the original token request included a client identifier. The value MUST identify the same client as the original token request. This follows the unauthenticated client request pattern defined for the token endpoint in {{RFC6749}}.

The request MUST NOT include parameters that modify or replace parameters from the original token request, including `scope`, `resource` as defined by {{RFC8707}}, `audience`, `authorization_details` as defined by {{RFC9396}}, `redirect_uri`, `code_verifier`, `subject_token`, `actor_token`, `assertion`, or grant-type-specific parameters from the original request. The authorization server MUST reject such requests with `invalid_request`.

If the original token request was made by an authenticated client, the continuation request MUST authenticate the same client. If the original token request was made by an unauthenticated client that included a `client_id`, the continuation request MUST include the same `client_id`. If the original token request used sender-constrained client authentication or proof-of-possession material, the authorization server MUST apply equivalent sender-constraining requirements to the continuation request.

When the client uses a client authentication mechanism that requires a fresh credential per request, such as the `client_assertion` and `client_assertion_type` parameters defined by {{RFC7521}}, the continuation request MAY present a freshly minted credential of the same type that authenticates the same client. The authorization server MUST verify that the freshly presented client credential identifies the client bound to the deferred processing state.

If the original token request included a valid DPoP proof {{RFC9449}}, the authorization server MUST bind the deferred processing state to the DPoP public key thumbprint. Each continuation request MUST include a valid DPoP proof for the token endpoint, and the DPoP proof key MUST match the key bound to the deferred processing state. If the DPoP proof is invalid, the authorization server returns the error response defined by {{RFC9449}}. If the proof is valid but does not match the deferred processing state binding, the authorization server MUST reject the continuation request with `invalid_grant`.

If the original token request used mutual-TLS client certificate binding {{RFC8705}}, the authorization server MUST bind the deferred processing state to the certificate or certificate thumbprint used for the original request. Each continuation request MUST use certificate binding that matches the deferred processing state. If the certificate binding does not match, the authorization server MUST reject the continuation request with `invalid_grant`.

If the access token produced by successful completion is sender-constrained, the confirmation or binding material for that access token MUST be derived from the original token request and continuation security context. The authorization server MUST NOT allow a continuation request to replace sender-constraining key material from the original token request.

### Example

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=
urn:ietf:params:oauth:grant-type:deferred_code&
deferred_code=4QFJ3P9&
client_id=s6BhdRkqt3
~~~

The authorization server MUST verify that the deferred code is valid, unexpired, and bound to the client and security context of the original token request.

# Continuation Responses

## Still Pending

If processing remains incomplete, the authorization server returns a token endpoint error response with `authorization_pending`.

The response MUST include an `expires_in` parameter. The response MUST include a `deferred_code` parameter when the authorization server rotates the deferred code, including the cases in which rotation is required by {{deferred-code-semantics}}; the client MUST discard the previous value and use the replacement. Otherwise the response MAY include a `deferred_code` parameter, and if omitted, the client continues to use the deferred code value from the continuation request.

~~~ json
{
  "error": "authorization_pending",
  "deferred_code": "8N5B2K1",
  "interval": 5,
  "expires_in": 895
}
~~~

## Interaction Required

If external interaction is required before processing can continue, the authorization server returns a token endpoint error response with `interaction_required`.

The response MUST include an `expires_in` parameter and an `interaction_uri` parameter. The response MUST include a `deferred_code` parameter when the authorization server rotates the deferred code, including the cases in which rotation is required by {{deferred-code-semantics}}; the client MUST discard the previous value and use the replacement. Otherwise the response MAY include a `deferred_code` parameter, and if omitted, the client continues to use the deferred code value from the continuation request.

~~~ json
{
  "error": "interaction_required",
  "deferred_code": "8N5B2K1",
  "interaction_uri":
    "https://as.example.com/interact/8N5B2K1",
  "interval": 5,
  "expires_in": 895
}
~~~

## Polling Too Frequent

If the client submits continuation requests more frequently than permitted, the authorization server MAY return a token endpoint error response with `slow_down`. Deferred processing state is preserved.

The client MUST increase its polling interval before submitting another continuation request. The new interval MUST be at least the larger of any `interval` value provided in the response and the previously used interval increased by 5 seconds. The unconditional 5-second increase preserves the polling behavior defined by {{RFC8628}}.

The response MUST include an `expires_in` parameter. The response MUST include a `deferred_code` parameter when the authorization server rotates the deferred code, including the cases in which rotation is required by {{deferred-code-semantics}}; the client MUST discard the previous value and use the replacement. Otherwise the response MAY include a `deferred_code` parameter, and if omitted, the client continues to use the deferred code value from the continuation request.

~~~ json
{
  "error": "slow_down",
  "deferred_code": "8N5B2K1",
  "interval": 10,
  "expires_in": 890
}
~~~

## Successful Completion

When processing completes successfully, the authorization server returns an access token response appropriate to the original grant type and MAY include any parameters normally returned for that grant type. For example, a bearer access token response is defined by {{RFC6750}}.

This specification does not define or constrain higher-layer identity response semantics.

### Example

~~~ json
{
  "access_token": "SlAV32hkKG",
  "token_type": "Bearer",
  "expires_in": 3600
}
~~~

The authorization server MUST invalidate the deferred code after successful completion. Any prior deferred code values associated with the same deferred processing state are also invalidated.

## Unsuccessful Completion {#unsuccessful-completion}

If the authorization server determines that the original token request cannot be completed successfully, it returns a token endpoint error response appropriate to the reason for failure. The authorization server SHOULD return `access_denied` when the failure results from an authorization decision reached during deferred processing, such as policy, attestation outcome, or approval. Other token endpoint error codes defined by {{RFC6749}}, {{RFC8628}}, or higher-layer profiles MAY be used when more specific. The authorization server MUST NOT return `invalid_grant` for unsuccessful completion of an otherwise valid deferred request; that error is reserved for an invalid or unbound deferred code (see {{invalid-deferred-code}}).

~~~ json
{
  "error": "access_denied"
}
~~~

## Expired Deferred Code

If the deferred code has expired, the authorization server returns a token endpoint error response.

~~~ json
{
  "error": "expired_token"
}
~~~

## Invalid Deferred Code {#invalid-deferred-code}

If the deferred code is malformed, unknown, revoked, not bound to the requesting client, or otherwise invalid, the authorization server returns a token endpoint error response.

~~~ json
{
  "error": "invalid_grant"
}
~~~

# Cancellation {#cancellation}

A client MAY abandon a deferred request by revoking the deferred code using OAuth Token Revocation {{RFC7009}}.

The revocation request is submitted to the authorization server's revocation endpoint as defined by {{RFC7009}}. The `token` parameter value is the most recently issued deferred code. The request SHOULD include `token_type_hint=deferred_code`.

The client MUST authenticate to the revocation endpoint with credentials equivalent to those required for a continuation request as defined in {{continuation-request}}.

Authorization servers that issue DPoP-bound deferred codes {{RFC9449}} MUST accept and validate DPoP proofs at the revocation endpoint for those codes; the proof key MUST match the key bound to the deferred processing state. Authorization servers that issue mutual-TLS bound deferred codes {{RFC8705}} MUST require certificate binding on the revocation request that matches the deferred processing state. If the binding does not match, the authorization server MUST treat the revocation request as not authorized for that deferred processing state and MUST NOT terminate the state.

Successful revocation terminates the deferred processing state. Subsequent continuation requests MUST be rejected with `invalid_grant`. Revocation does not produce an access token response.

If the deferred code presented for revocation is unknown, expired, already revoked, or has been replaced by a rotated value, the authorization server processes the request as defined by {{RFC7009}}. Authorization servers SHOULD treat any deferred code value previously associated with the same deferred processing state as a valid revocation target.

# Notification {#notification}

This specification defines an OPTIONAL notification mode that allows the authorization server to inform a registered client when deferred processing state changes. The notification mode is a hint mechanism intended to reduce polling latency. Clients MUST continue to obtain processing results by submitting continuation requests as defined in {{continuation-request}}; notifications convey only the affected deferred code, not result data.

The notification token defined in this specification is issued by the authorization server and validated by the client. This is the inverse of the `client_notification_token` model defined by {{CIBA}}, in which the client issues the token and the authorization server presents it. Higher-layer profiles that require client-issued notification tokens for compatibility with CIBA MAY layer such a mechanism on top of this specification.

## Client Notification Endpoint

A client that wishes to receive notifications registers a notification endpoint URL using the client metadata parameter `deferred_code_notification_endpoint`. Client metadata is registered as defined by {{RFC7591}} or by an authorization server's local registration mechanism.

The endpoint MUST be an HTTPS URI. It MUST NOT contain a fragment component.

## notification_token

When the authorization server defers a token request from a client that has registered a notification endpoint and intends to deliver notifications for the deferred processing state, the deferred processing response and continuation responses MAY include a `notification_token` parameter.

The `notification_token` is an opaque bearer credential used to authenticate the authorization server's notification request to the client's notification endpoint. Clients MUST treat the value as opaque.

The authorization server MUST issue notification tokens with sufficient entropy to resist guessing and MUST bind each notification token to a single deferred processing state. The authorization server MAY rotate the notification token. If a continuation response includes a replacement `notification_token`, the client MUST discard the previous value and validate subsequent notifications against the replacement.

### Example

A deferred processing response with a `notification_token`:

~~~ json
{
  "error": "authorization_pending",
  "deferred_code": "8N5B2K1",
  "notification_token": "9d2f6c4b8a1e7d3f5b9c0a2e4f6d8b1a",
  "interval": 5,
  "expires_in": 900
}
~~~

## Notification Request

When the authorization server elects to send a notification, it sends an HTTPS POST request to the client's registered notification endpoint with the following:

* an `Authorization` header containing `Bearer` followed by the current `notification_token` value,
* a `Content-Type` header of `application/json`,
* a JSON body containing the affected `deferred_code`.

~~~ http
POST /deferred-notify HTTP/1.1
Host: client.example.com
Authorization: Bearer 9d2f...
Content-Type: application/json

{"deferred_code": "8N5B2K1"}
~~~

The authorization server MUST NOT include access tokens, identity claims, processing results, or other state in the notification body. The notification conveys only that the deferred processing state has changed.

## Notification Response

The client SHOULD respond with HTTP status `200 OK` or `204 No Content` and MUST NOT include processing results in the response body.

The authorization server MUST NOT follow HTTP redirects from the notification endpoint and MUST treat any 3xx response as a delivery failure.

The authorization server MAY retry notification delivery using an exponential backoff. The authorization server MUST NOT depend on successful notification delivery; deferred processing state remains authoritative and is obtained through continuation requests.

## Client Behavior

A client that receives a notification MUST validate the bearer credential in the `Authorization` header against the most recent `notification_token` issued for the deferred code identified in the notification body. If the credential is missing, malformed, or does not match a known notification token, the client MUST reject the notification.

After validating a notification, the client SHOULD submit a continuation request to obtain the current processing state. The notification itself does not convey result data.

A client MUST NOT rely on receiving a notification. Polling at the interval indicated by the authorization server remains required when no notification has been received.

## Authorization Server Behavior

The authorization server MAY deliver a notification on any state transition, including transitions to Pending, Interaction Required, Complete, Denied, or Expired. The authorization server MUST validate the registered notification endpoint URL prior to delivery.

The authorization server MUST authenticate the notification request using only the issued `notification_token`. The authorization server MUST NOT include client credentials or other long-lived secrets in the notification request.

# Grant Type Applicability

This specification applies to any OAuth token request.

When an authorization server defers a token request, it MUST preserve the security properties of the original grant type. Deferred processing MUST NOT make a one-time credential reusable, extend the validity of an expired credential, bypass proof requirements, or allow the client to substitute different grant inputs during continuation.

## Authorization Code Grant

An authorization server MAY defer processing during authorization code redemption.

If an authorization server defers authorization code redemption, it MUST treat the authorization code as consumed or otherwise unusable outside the deferred processing state. A later continuation request resumes the deferred redemption; it is not a second authorization code redemption attempt. The authorization server MUST preserve PKCE {{RFC7636}} verification results, redirect URI validation, client binding, and any other authorization code grant checks performed before deferral.

PKCE verification, including comparison of the `code_verifier` against the previously stored `code_challenge` as defined by {{RFC7636}}, MUST occur during evaluation of the original token request, before the authorization server creates deferred processing state. The `code_verifier` is consumed at that point and MUST NOT be re-presented on continuation requests, consistent with the prohibition on resubmitting original grant parameters in {{continuation-request}}. Continuation requests rely on the PKCE verification result preserved in deferred processing state.

## Refresh Token Grant

An authorization server MAY require additional evaluation before issuing refreshed tokens.

When refresh token rotation is used, the authorization server SHOULD rotate the refresh token only on successful completion of the deferred request, not at the time of deferral. Issuing a replacement refresh token before the deferred operation completes would create two refresh tokens authorizing the same period, weakening the rotation guarantees described in {{RFC9700}}.

While deferred processing state exists for a refresh request, the authorization server MUST treat the original refresh token as suspended: it MUST NOT honor concurrent refresh requests using the same refresh token while a deferred refresh request derived from it is pending. The authorization server MUST prevent the original refresh token and the deferred processing state from together producing more tokens than a single successful synchronous refresh request would have produced.

## Client Credentials Grant

An authorization server MAY defer issuance pending workload attestation, enterprise policy evaluation, or asynchronous risk analysis.

For client credentials requests, deferred processing state MUST remain bound to the authenticated client and any sender-constraining or proof-of-possession material used by the original request.

## Token Exchange

An authorization server MAY defer processing during token exchange evaluation as defined by {{RFC8693}}.

For token exchange requests, deferred processing state MUST preserve the original `subject_token`, `subject_token_type`, `actor_token`, `actor_token_type`, `requested_token_type`, the requested `resource` values defined by {{RFC8707}}, the requested `audience` values, and `scope` used to evaluate the exchange. Continuation requests MUST NOT substitute replacement tokens or change requested token characteristics.

### Example

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange&
subject_token=eyJhbGciOi...&
subject_token_type=
urn:ietf:params:oauth:token-type:access_token&
resource=https://api.example.com
~~~

~~~ json
{
  "error": "interaction_required",
  "deferred_code": "abc123",
  "interaction_uri":
    "https://as.example.com/interact/abc123",
  "interval": 5,
  "expires_in": 600
}
~~~

## Rich Authorization Requests

When the original token request includes the `authorization_details` parameter defined by {{RFC9396}}, deferred processing state MUST preserve the originally requested authorization details. Continuation requests MUST NOT include `authorization_details`, and the authorization server MUST evaluate completion against the originally requested authorization details. Authorization decisions reached during deferred processing MAY narrow the granted authorization details but MUST NOT broaden them beyond what the original request expressed.

## Assertion Grants

An authorization server MAY defer processing during assertion grant evaluation as defined by {{RFC7521}}.

For assertion grants, deferred processing state MUST preserve assertion validation results and the original assertion value or an equivalent protected reference to it. Deferral MUST NOT extend the validity period of an expired assertion or permit replay of the assertion outside the deferred processing state.

### JWT Bearer Example

The following example shows deferred processing for a JWT bearer authorization grant as defined by {{RFC7523}}.

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=
urn:ietf:params:oauth:grant-type:jwt-bearer&
assertion=eyJhbGciOi...
~~~

~~~ json
{
  "error": "authorization_pending",
  "deferred_code": "jwt-771",
  "interval": 10,
  "expires_in": 300
}
~~~

# Authorization Server Metadata

The following OAuth Authorization Server Metadata parameters are defined.

## deferred_code_processing_supported

Boolean value indicating support for deferred code processing.

If omitted, the default value is `false`.

This parameter indicates that the authorization server can return deferred processing responses from the token endpoint and accepts the deferred code grant type. It does not indicate that every supported grant type or every token request is eligible for deferred processing.

An authorization server that supports this specification and publishes the `grant_types_supported` metadata parameter defined by {{RFC8414}} MUST include `urn:ietf:params:oauth:grant-type:deferred_code` in that parameter.

## deferred_code_grant_types_supported

JSON array containing OAuth grant type values for which the authorization server can return deferred processing responses.

This parameter is REQUIRED when `deferred_code_processing_supported` is `true`. When the authorization server publishes the `grant_types_supported` metadata parameter defined by {{RFC8414}}, the values of this parameter MUST be a subset of `grant_types_supported`. An empty array indicates that no original-request grant types are currently eligible for deferred processing, even though the authorization server otherwise supports the deferred code grant type.

The value `urn:ietf:params:oauth:grant-type:deferred_code` MUST NOT appear in this array because the array describes original token request grant types, not the continuation grant type.

## deferred_code_notification_supported

Boolean value indicating support for the notification mode defined in {{notification}}.

If omitted, the default value is `false`.

When `true`, the authorization server MAY deliver notifications to clients that have registered a `deferred_code_notification_endpoint` and SHOULD include a `notification_token` in deferred processing responses for those clients when notifications will be delivered.

### Example

~~~ json
{
  "issuer": "https://as.example.com",
  "token_endpoint":
    "https://as.example.com/token",
  "revocation_endpoint":
    "https://as.example.com/revoke",
  "grant_types_supported": [
    "authorization_code",
    "client_credentials",
    "urn:ietf:params:oauth:grant-type:token-exchange",
    "urn:ietf:params:oauth:grant-type:deferred_code"
  ],
  "deferred_code_processing_supported": true,
  "deferred_code_grant_types_supported": [
    "authorization_code",
    "client_credentials",
    "urn:ietf:params:oauth:grant-type:token-exchange"
  ],
  "deferred_code_notification_supported": true
}
~~~

# Security Considerations

Implementations MUST follow the OAuth 2.0 Security Best Current Practice {{RFC9700}} in addition to the requirements in this section.

## Deferred Code Entropy

Deferred codes MUST contain sufficient entropy to resist guessing attacks.

The entropy requirement applies to any value that can be used as a bearer continuation reference at the token endpoint. Authorization servers SHOULD generate deferred codes using a cryptographically secure random source or protect structured values using integrity and confidentiality mechanisms appropriate for bearer artifacts.

## Deferred Code Binding

The authorization server MUST bind the deferred code to the original token request and associated security context.

The binding MUST include the client identity when a client identity is present. It MUST include sender-constraining or proof-of-possession material used by the original token request, including DPoP public key thumbprints {{RFC9449}} or mutual-TLS certificate bindings {{RFC8705}}. It SHOULD include requested resources, requested scopes, grant-type-specific input artifacts, and any other values that affect token issuance.

## Proof-of-Possession Continuity

Deferred processing MUST preserve proof-of-possession and sender-constraining properties from the original token request through every continuation request and the final access token response.

When the original token request used DPoP, each continuation request MUST use a valid DPoP proof for the token endpoint with the same DPoP key bound to the deferred processing state. When the original token request used mutual TLS, each continuation request MUST use certificate binding that matches the deferred processing state.

Authorization servers MUST reject continuation requests that attempt to replace, omit, or weaken proof-of-possession or sender-constraining material from the original token request. If a proof is syntactically or cryptographically invalid, the authorization server returns the error defined by the proof mechanism. If the proof is valid but does not match the deferred processing state, the authorization server returns `invalid_grant`.

When DPoP nonces {{RFC9449}} are used, the authorization server MAY require a fresh DPoP proof bound to a current nonce on each continuation request and MAY return `use_dpop_nonce` with an updated nonce as defined by {{RFC9449}}. A `use_dpop_nonce` response MUST NOT terminate the deferred processing state. The client MUST retry the continuation request with a DPoP proof that incorporates the updated nonce. Nonce rotation is independent of deferred code rotation; either, both, or neither MAY occur on a given continuation response.

## Client Authentication

The authorization server MUST require client authentication equivalent to the original token request when the original request used client authentication.

For unauthenticated clients, including public clients, the authorization server MUST require the continuation request to identify the same client as the original token request when a client identifier was present. Client identification alone does not authenticate a public client or prove possession of the same client instance. Authorization servers MUST bind deferred processing state to additional proof or context that the requesting client instance demonstrates on each continuation request, such as DPoP proof {{RFC9449}}, mutual-TLS client certificate binding {{RFC8705}}, an authenticated user-agent session, device-bound context, or other sender-constraining material applicable to the original token request.

Preserved PKCE {{RFC7636}} verification state from the original token request is NOT, by itself, a sufficient continuation binding for public clients: the `code_verifier` is consumed during the original request and is not re-presented on continuation requests, so PKCE state cannot authenticate the continuation requestor. PKCE MAY be combined with one of the bindings above but does not substitute for one.

Authorization servers MUST NOT allow a continuation request to weaken client authentication, sender-constraining, or proof-of-possession requirements from the original token request.

## Deferred Code Replay

Authorization servers SHOULD detect replay of invalidated or rotated deferred codes.

Replay detection is particularly important when a deferred code is returned through an interaction channel, logged by intermediaries, or exposed to user agents.

## Deferred Code Rotation

Authorization servers MAY rotate deferred codes during continuation processing.

Clients MUST discard replaced values.

When rotating deferred codes, authorization servers SHOULD invalidate the previous value promptly after issuing the replacement.

If a deferred code is not sender-constrained or otherwise bound to proof-of-possession material, authorization servers MUST rotate the deferred code after each valid continuation request that returns `authorization_pending`, `interaction_required`, or `slow_down`. Authorization servers MAY omit rotation when the deferred code is sender-constrained or protected by an equivalent replay-resistant mechanism.

## Interaction URI Protection

Interaction URIs MUST use HTTPS.

Authorization servers SHOULD bind interaction state to authenticated sessions where applicable.

Interaction URIs can become bearer references to sensitive continuation state. Authorization servers MUST NOT place sensitive information in URI query components. Authorization servers SHOULD use referrer-policy, cache-control, and logging practices that reduce disclosure of interaction URIs.

Clients MUST NOT assume that accessing an interaction URI completes authorization or authentication. Completion semantics are defined by the authorization server or by a higher-layer profile.

## Concurrent Continuation Requests

Clients SHOULD send at most one outstanding continuation request per deferred processing state. Multiple concurrent continuation requests, including those caused by client retries after network errors, can race with deferred code rotation and yield ambiguous outcomes.

Authorization servers MUST process continuation requests so that successful completion is observed at most once per deferred processing state. Concurrent or retried continuation requests for the same deferred processing state MUST NOT cause more than one access token response to be issued.

When a deferred code has been rotated, continuation requests presenting a previous deferred code value MUST be rejected with `invalid_grant` even if the deferred processing state itself is still valid.

## Continuation Request Rate Limiting

Authorization servers SHOULD rate limit continuation requests.

Clients SHOULD respect the `interval` parameter.

Authorization servers MAY increase the `interval` value when clients poll too frequently.

## Deferred Processing Lifetime

Authorization servers SHOULD limit deferred processing lifetime and invalidate expired deferred codes.

Long-lived deferred processing state increases replay risk and can retain sensitive request context. Authorization servers SHOULD choose lifetimes that are no longer than necessary for the expected asynchronous processing or interaction. Unless a higher-layer profile defines stronger controls, deferred code lifetimes SHOULD be measured in minutes. Authorization servers SHOULD NOT use lifetimes measured in hours or days without sender-constraining, one-time-use deferred codes, explicit user or administrator approval semantics, and operational monitoring.

## Authorization Scope Stability

Continuation processing MUST NOT expand the requested authorization beyond the original token request. Authorization servers MUST evaluate the continuation request using the original request context and MUST reject attempts to substitute new requested resources, scopes, subject tokens, assertions, or other grant inputs.

## Oracle Resistance

Deferred processing can expose information about policy state if attackers can compare immediate denial, deferred processing, polling duration, interaction requirements, and final errors across many requests. Authorization servers SHOULD avoid using externally observable deferred-processing behavior to reveal sensitive policy decisions, user existence, resource existence, risk signals, approval rules, or attestation outcomes.

Authorization servers SHOULD normalize error responses, timing, retry intervals, and expiration behavior where those signals would otherwise disclose sensitive information.

## Security Event Logging

Authorization servers SHOULD record security events for deferred code creation, continuation requests, deferred code rotation, interaction URI creation, interaction completion, successful completion, denial, expiration, revocation, replay detection, and rate-limit enforcement.

Logs SHOULD include enough correlation information to support investigation without recording deferred code values, interaction URI secrets, subject tokens, assertions, authorization codes, refresh tokens, or other sensitive artifacts in cleartext.

## Cancellation Authorization

The authorization server MUST authorize cancellation requests against the deferred processing state with the same rigor applied to continuation requests. An attacker that obtains a deferred code value but does not satisfy sender-constraining or proof-of-possession bindings MUST NOT be able to terminate deferred processing state through revocation.

Authorization servers SHOULD record successful and rejected revocation attempts as security events.

## Notification Endpoint Protection

The notification endpoint registered by a client is dereferenced by the authorization server and is therefore a server-side request forgery (SSRF) attack surface.

Authorization servers MUST validate registered notification endpoint URLs at registration time and prior to each delivery. Authorization servers SHOULD reject endpoints whose host components resolve to loopback, link-local, multicast, broadcast, or other non-public addresses unless an explicit deployment policy permits internal destinations. Authorization servers MUST NOT follow HTTP redirects from the notification endpoint.

The notification endpoint is publicly reachable and receives POST requests from the public internet. Clients MUST validate the inbound bearer credential against the most recent `notification_token` value before treating the notification as authentic. Clients SHOULD apply rate limiting to the notification endpoint to mitigate abuse.

## Notification Token Protection

The `notification_token` is a bearer credential. Authorization servers MUST issue notification tokens with sufficient entropy to resist guessing and MUST transmit them only over TLS-protected responses to the client.

Authorization servers SHOULD limit notification token lifetime to no longer than the lifetime of the associated deferred processing state and SHOULD invalidate notification tokens promptly when the deferred processing state ends.

Authorization servers MUST NOT reuse a `notification_token` value across distinct deferred processing states.

## Notification Information Leakage

The notification payload conveys only the affected `deferred_code`. Authorization servers MUST NOT include access tokens, identity claims, attestation outcomes, policy results, subject identifiers, or other state in the notification body or headers.

Authorization servers SHOULD avoid using notification timing or delivery patterns to reveal sensitive information about deferred processing state, consistent with the oracle resistance requirements above.

## Privacy Considerations

Deferred processing can retain information from the original token request, including subject identifiers, requested resources, policy evaluation inputs, attestation evidence, or transaction details. Authorization servers SHOULD minimize retained state, protect it at rest, and delete it when processing completes or expires.

Interaction URIs and deferred codes SHOULD NOT reveal user identifiers, client identifiers, resource identifiers, transaction details, or policy decisions to parties that can observe browser history, logs, referrer headers, or network metadata.

# IANA Considerations

## OAuth Parameters Registry

This specification registers the following parameters in the "OAuth Parameters" registry established by {{RFC6749}}.

Parameter name:
: `deferred_code`

Parameter usage location:
: token response, token request

Change controller:
: IETF

Specification document(s):
: This document

Parameter name:
: `interaction_uri`

Parameter usage location:
: token response

Change controller:
: IETF

Specification document(s):
: This document

Parameter name:
: `notification_token`

Parameter usage location:
: token response

Change controller:
: IETF

Specification document(s):
: This document

## OAuth Authorization Server Metadata Registry

This specification registers the following parameters in the "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

Metadata name:
: `deferred_code_processing_supported`

Metadata description:
: Boolean value indicating support for deferred code processing.

Change controller:
: IETF

Specification document(s):
: This document

Metadata name:
: `deferred_code_grant_types_supported`

Metadata description:
: JSON array containing OAuth grant type values for which the authorization server can return deferred processing responses.

Change controller:
: IETF

Specification document(s):
: This document

Metadata name:
: `deferred_code_notification_supported`

Metadata description:
: Boolean value indicating support for deferred code state-change notification mode.

Change controller:
: IETF

Specification document(s):
: This document

## OAuth Dynamic Client Registration Metadata Registry

This specification registers the following parameter in the "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}.

Client Metadata Name:
: `deferred_code_notification_endpoint`

Client Metadata Description:
: HTTPS URL where the authorization server delivers deferred code state-change notifications to the client.

Change Controller:
: IETF

Specification Document(s):
: This document

## OAuth URI Registry

This specification registers the following URI in the "OAuth URI" registry established by {{RFC6755}}.

URN:
: `urn:ietf:params:oauth:grant-type:deferred_code`

Common name:
: Deferred Code Grant Type

Change controller:
: IETF

Specification document(s):
: This document

## OAuth Extensions Error Registry

The error names `authorization_pending`, `slow_down`, and `expired_token` are already registered in the "OAuth Extensions Error Registry" established by {{RFC6749}} by the OAuth Device Authorization Grant {{RFC8628}}. This specification reuses those error names at the token endpoint with the generalized deferred-processing semantics defined in {{deferred-error-semantics}}. No registry changes are required for these three error names; the registrations continue to point to {{RFC8628}}, and this document defines how those errors apply to deferred processing state in addition to device authorization state.

The error name `access_denied` is registered by {{RFC6749}} for authorization endpoint use and is extended to token endpoint use by {{RFC8628}}. This specification relies on the {{RFC8628}} extension and applies `access_denied` to deferred processing state that has terminated unsuccessfully. No registry changes are required.

This specification updates the existing `interaction_required` registration in the "OAuth Extensions Error Registry" established by {{RFC6749}}. That error name was originally registered by OpenID Connect {{OIDC-CORE}} for authorization endpoint use. This specification adds token endpoint response usage for deferred code processing.

Error name:
: `interaction_required`

Error usage location:
: authorization endpoint response, token endpoint response

Related protocol extension:
: OpenID Connect, deferred code processing

Change controller:
: OpenID Foundation Artifact Binding Working Group; IETF for token endpoint response usage defined by this document

Specification document(s):
: OpenID Connect Core 1.0; this document

--- back

# Design Rationale

This specification extends token endpoint processing rather than defining a separate authorization protocol or authorization endpoint.

The evaluations enumerated in {{introduction}} frequently occur after receipt of an otherwise valid token request. In such cases, introducing a separate authorization flow or redirect-based interaction model may be inappropriate or impossible: the client has already presented its grant inputs, and any external evaluation acts on that request rather than initiating a new authorization transaction.

Extending token endpoint processing semantics directly preserves existing OAuth grant models, keeps the continuation interface where the client is already prepared to interact, and avoids defining a parallel endpoint with overlapping responsibilities.

# Relationship to Existing OAuth Deferred Interaction Mechanisms

OAuth and OpenID Connect already define several mechanisms involving deferred completion, polling, or out-of-band interaction. This specification generalizes deferred processing semantics across OAuth token requests without introducing a new authorization framework.

## Relationship to OAuth Device Authorization Grant

The OAuth Device Authorization Grant {{RFC8628}} defines a separate authorization flow designed for input-constrained devices.

In the Device Authorization Grant:

* the client first obtains a device code,
* the user completes interaction through a separate verification URI,
* the client polls the token endpoint for completion.

This specification differs in several ways:

* it does not define a new authorization flow,
* it applies to arbitrary token requests,
* it does not require a separate authorization endpoint,
* it allows deferred processing after receipt of an otherwise valid token request,
* it supports both interaction-based and non-interactive asynchronous processing.

## Relationship to OpenID Connect CIBA

This specification is distinct from {{CIBA}}.

CIBA defines an OpenID Connect authentication flow in which a Relying Party initiates end-user authentication by sending an authentication request to a dedicated backchannel authentication endpoint. CIBA returns an `auth_req_id` identifying the authentication request and defines poll, ping, and push modes for obtaining or delivering the authentication result.

This specification instead defines a token endpoint continuation mechanism for OAuth token requests that have already reached the token endpoint. It does not define a backchannel authentication endpoint, an authentication request object, token delivery modes, or an `auth_req_id`.

The key differences are:

* Protocol layer: CIBA is an OpenID Connect authentication flow. This specification is an OAuth token endpoint extension.
* Initiation point: CIBA starts with a request to the backchannel authentication endpoint. This specification starts with an ordinary token request to the token endpoint.
* Continuation handle: CIBA uses an `auth_req_id` for an authentication transaction. This specification uses a `deferred_code` for suspended token endpoint processing state.
* Subject requirement: CIBA is centered on authenticating an end-user identified to the OpenID Provider. This specification does not require an end-user and can apply to client credentials, token exchange, assertion grants, refresh tokens, and other token requests.
* Interaction semantics: CIBA defines an out-of-band end-user authentication and consent model. This specification only signals that external interaction may be required and leaves the interaction semantics to the authorization server or a higher-layer profile.
* Completion model: CIBA defines poll, ping, and push modes. This specification defines token endpoint polling through the deferred code grant type and an OPTIONAL state-change notification mode (see {{notification}}); push delivery of access tokens or identity assertions remains outside its scope.
* Response semantics: CIBA is designed to return OpenID Connect authentication results, including ID Token semantics. This specification returns the access token response appropriate to the original OAuth grant type and does not define ID Token processing.
* Applicability: CIBA defines a specific authentication flow. This specification defines a general suspension and continuation pattern for OAuth token endpoint processing.

An authorization server that supports both CIBA and this specification SHOULD use CIBA when the client is initiating an OpenID Connect backchannel authentication flow for an end-user. It SHOULD use this specification when an existing OAuth token request cannot complete synchronously and the authorization server needs to preserve and later resume evaluation of that request.

Higher-layer profiles MAY combine this specification with OpenID Connect semantics, but doing so does not make the deferred code grant type equivalent to the CIBA grant type. A profile that needs CIBA behavior, including backchannel authentication endpoint processing, `auth_req_id`, delivery modes, or ID Token authentication result semantics, SHOULD use or profile CIBA directly.

## Relationship to OAuth JWT Grant Interaction Response

The OAuth JWT Grant Interaction Response specification {{JWT-GRANT-INTERACTION}} defines interaction semantics for JWT bearer grant requests.

This specification generalizes those concepts for all OAuth token requests and introduces:

* generalized deferred processing semantics,
* a deferred code grant type for continuation,
* applicability to arbitrary grant types,
* non-interactive asynchronous processing.

This specification uses `interaction_required` as the token endpoint error value for deferred processing that is waiting on external interaction. That value is aligned with OpenID Connect interaction terminology {{OIDC-CORE}} and the pending-interaction polling state described by {{JWT-GRANT-INTERACTION}}.

# Relationship to Higher-Layer Protocols

This specification remains at the OAuth layer.

Higher-layer specifications MAY define profiles addressing:

* identity-specific semantics,
* ID Token processing,
* authentication ceremony requirements,
* consent semantics,
* transaction presentation requirements,
* identity proofing requirements,
* callback or notification mechanisms beyond the state-change notification mode defined by this specification.

For example, an OpenID Connect profile MAY define:

* deferred ID Token issuance,
* authentication continuation semantics,
* identity transaction binding,
* transaction assurance semantics,
* Deferred Transaction Resource integration {{DTR}}.

# End-to-End Example

This appendix illustrates a complete deferred authorization code redemption that includes external interaction. The example is non-normative.

A client redeems an authorization code at the token endpoint, presenting a PKCE `code_verifier` and a DPoP proof:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAi...

grant_type=authorization_code&
code=SplxlOBeZQQYbYS6WxSbIA&
redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb&
code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk&
client_id=s6BhdRkqt3
~~~

The authorization server validates the request and PKCE verifier, then defers completion pending an enterprise approval workflow. It returns a deferred processing response:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "error": "authorization_pending",
  "deferred_code": "8N5B2K1",
  "interval": 5,
  "expires_in": 600
}
~~~

The client polls using the deferred code grant type with a fresh DPoP proof:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAi...

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred_code&
deferred_code=8N5B2K1&
client_id=s6BhdRkqt3
~~~

External approval determines that a manager must review the request. The authorization server rotates the deferred code, transitions the state to Interaction Required, and returns an `interaction_uri`:

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "error": "interaction_required",
  "deferred_code": "9P2K7M3",
  "interaction_uri":
    "https://as.example.com/interact/9P2K7M3",
  "interval": 5,
  "expires_in": 540
}
~~~

The client surfaces the `interaction_uri` to a user. After the manager completes the approval at that URI, the authorization server transitions the state to Complete. The next continuation request returns an access token response:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAi...

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred_code&
deferred_code=9P2K7M3&
client_id=s6BhdRkqt3
~~~

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "SlAV32hkKG",
  "token_type": "DPoP",
  "expires_in": 3600,
  "refresh_token": "8xLOxBtZp8"
}
~~~

The deferred code is invalidated; subsequent continuation requests return `invalid_grant`.

# Acknowledgements

This specification builds upon concepts explored in OAuth JWT Grant Interaction Response and OpenID Connect Deferred Transaction Resources.
