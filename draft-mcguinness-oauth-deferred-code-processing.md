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

This specification defines deferred processing semantics for OAuth requests that cannot complete synchronously, by introducing a continuation mechanism that applies to both token endpoint and authorization endpoint processing.

An authorization server can defer completion of an OAuth token request or authorization request and return a `deferred_code` representing suspended processing state. Clients later resume processing using the deferred code grant type at the token endpoint, regardless of which endpoint the originating request was submitted to.

This specification also defines optional interaction continuation semantics for deferred requests that require external interaction.

This specification intentionally remains at the OAuth layer and does not define authentication ceremonies, identity semantics, consent semantics, or ID Token processing rules. Higher-layer protocols such as OpenID Connect MAY define profiles using the mechanisms defined by this specification.

--- middle

# Introduction {#introduction}

Traditional OAuth processing assumes that token issuance decisions can be completed synchronously, whether during evaluation of a token request at the token endpoint or during evaluation of an authorization request at the authorization endpoint.

In practice, many deployments require token issuance to pause pending external processing, including:

* policy and risk evaluation, including token exchange policies and external policy engines,
* attestation validation for workloads, devices, or transactions,
* enterprise approval or delegated authorization workflows,
* identity proofing and transaction verification,
* step-up authentication during the front-channel flow,
* cross-domain authorization orchestration.

Existing OAuth and OpenID Connect specifications address portions of this problem for specific protocols or grant types, including the polling model of the OAuth Device Authorization Grant {{RFC8628}} and the backchannel authentication flow of {{CIBA}}. This specification instead defines a generalized deferred execution model that unifies the continuation pattern across origination points: both token endpoint requests and authorization endpoint requests may be deferred, and continuation occurs uniformly at the token endpoint.

This specification separates processing state management from higher-layer identity, authentication, and user interaction semantics.

This specification defines:

* deferred processing semantics for token requests and authorization requests,
* the `deferred_code` continuation reference,
* the deferred code grant type,
* deferred authorization responses delivered through the user agent redirect,
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

Originating Request:
: The OAuth request whose processing has been deferred. The originating request is either a token request submitted to the token endpoint or an authorization request submitted to the authorization endpoint.

Deferred Code:
: An opaque continuation reference representing deferred processing state associated with an originating request.

Deferred Code Grant Type:
: A continuation grant type encoded using the OAuth extension grant mechanism to resume processing of a previously deferred request.

Interaction URI:
: A URI where external interaction associated with a deferred request can occur.

Deferred Request:
: An originating request whose processing has been deferred by the authorization server.

Deferred Authorization Response:
: An authorization endpoint response that conveys a `deferred_code` to the client through the redirection endpoint, as defined in {{authorization-endpoint-extensions}}.

Deferred Processing Response:
: A token endpoint response that conveys a `deferred_code` to the client, as defined in {{deferred-processing-responses}}.

Deferred Processing State:
: Authorization server state associated with a deferred request, including the originating request context, client security context, processing status, expiration, and any implementation-defined asynchronous processing context.

Continuation Request:
: An access token request using the deferred code grant type to resume processing of a deferred request.

Continuation Response:
: A token endpoint response to a continuation request.

# Architectural Model

This specification introduces deferred execution semantics to OAuth processing.

The authorization server can temporarily suspend completion of an originating request while external interaction or asynchronous evaluation occurs. Suspension does not create a new authorization transaction. It preserves the originating request and resumes evaluation of that request later.

This specification uses a continuation architecture with three responsibilities:

* The client submits an ordinary OAuth request to either the token endpoint or the authorization endpoint.
* The authorization server determines that processing cannot complete synchronously and creates deferred processing state. The authorization server returns a deferred processing response (for token endpoint requests) or a deferred authorization response (for authorization endpoint requests).
* The client resumes the suspended processing by presenting the deferred code at the token endpoint using the deferred code grant type.

The continuation interface is the token endpoint in all cases. The originating endpoint determines how the initial `deferred_code` is delivered, but does not affect the continuation polling mechanism, completion vocabulary, or state machine.

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
* allow deferred completion for arbitrary token endpoint and authorization endpoint requests,
* unify the continuation interface at the token endpoint regardless of which endpoint received the originating request,
* avoid client interpretation of server-side processing state,
* support both interactive and non-interactive asynchronous processing,
* allow higher-layer profiles to define user-facing or identity-specific behavior.

## Design Non-Goals

This specification does not define:

* a new authorization endpoint flow distinct from existing authorization request processing,
* a redirect-based interaction protocol,
* a user authentication protocol,
* push delivery of access tokens or identity assertions,
* semantics for the external interaction,
* resource server processing for deferred codes.

The notification mode defined in {{notification}} signals state changes only and does not deliver tokens or identity claims to the client.

## Processing Model

The authorization server processes an originating request according to the rules of the endpoint and (for token requests) the requested grant type. During that processing, the authorization server can reach one of the following outcomes:

* the request fails synchronously and the authorization server returns the appropriate error response for the endpoint;
* the request succeeds synchronously and the authorization server returns the appropriate success response for the endpoint (an access token response, an authorization code, or an access token via redirect);
* the request is valid but cannot complete synchronously, so the authorization server creates deferred processing state and returns either a deferred processing response (for token endpoint originating requests, see {{deferred-processing-responses}}) or a deferred authorization response (for authorization endpoint originating requests, see {{authorization-endpoint-extensions}}).

The deferred processing state is an authorization server internal record. It is referenced by the deferred code but is not exposed to the client. The state MUST preserve the originating request context needed to complete processing. The client MUST NOT resubmit the originating request parameters during continuation, except as defined for the first continuation request following authorization endpoint deferral (see {{authorization-endpoint-pkce}}).

Continuation requests resume evaluation of the originating request. The authorization server uses the deferred code to locate deferred processing state, verifies the continuation request against the binding requirements for that state, and returns one of the continuation responses defined in this specification. When the originating request was made at the authorization endpoint, successful completion returns an access token response directly; no separate authorization code redemption step occurs.

## Deferred Processing State

Deferred processing state has an implementation-defined representation, but it MUST be sufficient to enforce the requirements of this specification.

At minimum, the authorization server MUST associate deferred processing state with:

* the originating request type (token endpoint or authorization endpoint),
* the originating request parameters needed to complete processing,
* the client identity or public-client binding information associated with the originating request,
* sender-constraining or proof-of-possession requirements associated with the originating request,
* the current processing status,
* expiration time or expiration policy,
* any deferred code values currently valid for the state.

For authorization endpoint originating requests, the deferred processing state MUST additionally preserve the validated `redirect_uri`, the requested response type, the requested `scope`, any `code_challenge` and `code_challenge_method` values, and any other authorization request parameters required to complete processing.

The authorization server SHOULD also associate deferred processing state with requested resources, requested scopes, grant-specific input artifacts, policy evaluation inputs, interaction status, and audit information needed for security monitoring.

Deferred processing state MUST NOT be used to expand authorization beyond the originating request. Any authorization, approval, authentication, or policy result obtained during deferred processing can only constrain or complete the originating request.

## State Lifecycle

A deferred request follows this abstract lifecycle:

~~~ ascii-art
 Originating request
 (token endpoint or authorization endpoint)
          |
          v
 +--------------------+
 | Synchronous        |
 | evaluation         |
 +--------------------+
    |        |       |
    |        |       +--> success response
    |        |              (access token / code /
    |        |               token-via-redirect)
    |        |
    |        +----------> error response
    |                       (endpoint-appropriate)
    |
    v
 deferred response
   (processing response   or   authorization response
    at token endpoint)        (redirect to client)
    |
    v
 +--------------------+       continuation request
 | Deferred state     | <-----(token endpoint)----+
 | pending / waiting  |                           |
 +--------------------+                           |
    |        |       |                            |
    |        |       +--> continuation error -----+
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
: Processing completed successfully. The authorization server returns an access token response for the originating request.

Denied:
: Processing completed unsuccessfully because the originating request is not authorized. The authorization server returns `access_denied` when the unsuccessful outcome is the result of an authorization decision reached during deferred processing, or another token endpoint error response when more specific to the reason for failure (see {{unsuccessful-completion}}).

Expired:
: The deferred processing state or deferred code is no longer valid due to expiration. The authorization server returns `expired_token`.

Invalid:
: The deferred code cannot be used because it is malformed, unknown, revoked, not bound to the requesting client, or otherwise invalid. The authorization server returns `invalid_grant`. Deferred processing state terminated by client-initiated cancellation as defined in {{cancellation}} is observed externally as Invalid.

The authorization server MAY transition between Pending and Interaction Required more than once. Complete, Denied, Expired, and Invalid are terminal from the client's perspective.

# Deferred Code Semantics {#deferred-code-semantics}

A `deferred_code` is an opaque continuation reference representing deferred processing state associated with a previously submitted originating request. The originating request is either a token request submitted to the token endpoint or an authorization request submitted to the authorization endpoint.

A `deferred_code`:

* is not an access token,
* is not a refresh token,
* is not an authorization code,
* is not intended for resource server access,
* does not represent authorization delegation by itself,
* is only valid as input to the deferred code grant type at the authorization server token endpoint,
* MUST be treated as opaque by clients.

The authorization server MUST bind the `deferred_code` to the originating request and associated client security context.

The authorization server MAY rotate deferred codes during continuation processing.

If the authorization server issues a replacement deferred code, the client MUST discard the previous value.

If a deferred code is not sender-constrained or otherwise bound to proof-of-possession material, the authorization server MUST rotate the deferred code after each valid continuation request that returns `authorization_pending`, `interaction_required`, or `slow_down`. Authorization servers MUST use one-time-use deferred codes or an equivalent replay-resistant mechanism for deferred requests involving one-time credentials, high-value resources, public clients, or externally exposed interaction URIs.

# Deferred Processing

When processing an originating request, the authorization server MAY defer completion of the request. The originating request may be a token request submitted to the token endpoint or an authorization request submitted to the authorization endpoint.

Deferred processing allows token issuance to continue asynchronously after the originating request has been evaluated.

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

## Client Capability {#client-capability}

A client MAY declare whether it supports receiving deferred responses using the client metadata parameter `deferred_code_processing_supported`. The parameter is a boolean value.

If `true`, the authorization server MAY return deferred responses to this client when other eligibility criteria are met. The capability covers both deferred processing responses at the token endpoint (see {{deferred-processing-responses}}) and deferred authorization responses at the authorization endpoint (see {{authorization-endpoint-extensions}}); a client either understands the deferred code continuation model or does not, and the same metadata signal applies to both originating endpoints.

If `false`, the authorization server MUST NOT defer originating requests from this client and MUST instead complete each originating request synchronously, either by issuing the appropriate success response or by returning a terminal error response.

A client that has not registered this parameter is treated as if the value is `true`. The default reflects that a deferred processing response uses the OAuth token endpoint error response form defined by {{RFC6749}}, and a deferred authorization response uses the OAuth authorization response redirect form: a client unaware of this specification observes a deferred processing response as a token endpoint error and abandons the request, or observes a deferred authorization response as an authorization response that lacks `code` or `access_token` and is rejected by client state-machine validation. In both cases the unaware client experiences a clean failure with no protocol harm, so an authorization server offering deferral does not place an unaware client in a worse position than returning a terminal error today.

A client SHOULD register `deferred_code_processing_supported=false` when it cannot accept a non-synchronous outcome, for example when the calling code path has a hard latency budget, when the client has no facility to retain or resume continuation state, or when client policy requires sync-or-fail token acquisition.

Capability is a static property of the client implementation, expressed in client metadata. This specification does not define a per-request opt-in parameter; the same client cannot meaningfully support deferred processing on one request and not another, and a per-request flag would force one layer of the client stack to assert capability on behalf of another.

Client metadata is registered as defined by {{RFC7591}} or by an authorization server's local registration mechanism.

## Authorization Server Behavior

The authorization server is responsible for deciding whether an originating request is eligible for deferred processing, whether that request is a token request or an authorization request. The authorization server MAY apply local policy, grant-type-specific policy, response-type-specific policy, client policy, client capability declared in client metadata (see {{client-capability}}), resource policy, or higher-layer profile requirements when making that decision.

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

# Authorization Endpoint Extensions {#authorization-endpoint-extensions}

The following sections extend the OAuth authorization endpoint defined in {{RFC6749}} to allow the authorization server to defer completion of an authorization request.

## Deferred Authorization Response

When the authorization server determines that an authorization request cannot complete synchronously, it MAY return a deferred authorization response in place of an authorization response or authorization error response.

The authorization server MUST validate the authorization request to the extent necessary to determine eligibility for deferred processing before creating deferred processing state. This includes validating `client_id`, `redirect_uri`, `response_type`, and any other request parameters whose validation does not depend on completion of the deferred work. The authorization server MUST NOT issue a `deferred_code` for a request whose redirect URI cannot be validated.

The authorization server returns the deferred authorization response by redirecting the user agent to the client's redirection endpoint using the same redirect mechanism it would use for a synchronous authorization response, including any selected `response_mode` such as `query` or `form_post`.

Authorization servers SHOULD NOT return deferred authorization responses with `response_mode=fragment`. The fragment component is accessible to any script running on the redirect page, including third-party scripts and browser extensions, exposing the `deferred_code` to a substantially broader attack surface than the query or `form_post` channels and creating a risk class equivalent to the implicit-flow access token exposure that motivated deprecation of the implicit grant in {{RFC9700}}. Higher-layer profiles MAY permit fragment mode for specific use cases that justify the exposure with additional controls.

The response includes the following parameters:

deferred_code:
: REQUIRED. The deferred code value, as defined in {{deferred-code-semantics}}.

state:
: REQUIRED if the authorization request included a `state` parameter. The exact value received from the client.

expires_in:
: REQUIRED. The remaining lifetime of the deferred code in seconds.

interaction_uri:
: OPTIONAL. A URI identifying where external interaction associated with the deferred request can occur, as defined in {{interaction-uri}}. The `interaction_uri` MAY be omitted when external interaction is not yet known to be required; subsequent continuation responses MAY introduce an `interaction_uri` if the state transitions to Interaction Required.

interval:
: OPTIONAL. The minimum number of seconds the client SHOULD wait before submitting a continuation request.

A deferred authorization response MUST NOT include any authorization response parameter that would indicate synchronous completion of the originating request, including but not limited to the `code` parameter defined by {{RFC6749}} for the authorization code grant, the `access_token` parameter defined by {{RFC6749}} for the implicit grant, and the `id_token` parameter defined by OpenID Connect {{OIDC-CORE}} for the implicit or hybrid response types. Higher-layer profiles that define additional authorization response parameters MUST extend this prohibition to any such parameter that would indicate synchronous completion.

### Example

A deferred authorization response delivered as a redirect:

~~~ http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?
  deferred_code=8N5B2K1&
  state=xyz&
  expires_in=600&
  interaction_uri=https%3A%2F%2Fas.example.com%2Finteract%2F8N5B2K1
~~~

## Client Processing of Deferred Authorization Responses

A client that receives a deferred authorization response MUST verify the `state` parameter exactly as it would for any authorization response, as required by {{RFC6749}} and {{RFC9700}}.

The client MUST NOT treat the deferred authorization response as a successful authorization. The client resumes processing by submitting a continuation request to the token endpoint using the deferred code grant type defined in {{continuation-request}}. The client does not redeem an authorization code; the continuation request completes the originating authorization request and, when processing completes successfully, the token endpoint returns an access token response directly.

A client MAY present the `interaction_uri` to the user. This specification does not require any particular interaction channel.

## PKCE for Deferred Authorization Responses {#authorization-endpoint-pkce}

When the originating authorization request included a `code_challenge` parameter as defined by {{RFC7636}}, the authorization server MUST associate the corresponding `code_challenge` and `code_challenge_method` values with the deferred processing state.

PKCE verification cannot be performed at the authorization endpoint because the client has not yet presented the `code_verifier`. The first continuation request following a deferred authorization response MUST include the `code_verifier` parameter when PKCE was used on the originating authorization request, and the authorization server MUST verify the verifier against the stored `code_challenge` using the stored `code_challenge_method` before completing the deferred processing.

This is an exception to the general prohibition in {{continuation-request}} on presenting `code_verifier` on continuation requests. The exception applies only to the first continuation request following a deferred authorization response, and only when PKCE was used on the originating authorization request. The `code_verifier` MUST NOT be re-presented on subsequent continuation requests; after the first continuation, PKCE verification state is preserved in the deferred processing state.

If PKCE verification fails, the authorization server MUST reject the continuation request with `invalid_grant` and MUST terminate the deferred processing state.

For authorization endpoint deferral of public client authorization requests, PKCE is REQUIRED in accordance with {{RFC9700}}. The `code_verifier` presented on the first continuation request serves as the continuation binding for the originating public client, in combination with any other binding mechanism applicable to the originating request.

When the originating authorization request additionally signals a sender-constraining key, for example via a DPoP key JWK thumbprint mechanism associated with {{RFC9449}} or via mutual-TLS client certificate binding {{RFC8705}} expressed at the authorization endpoint, the authorization server MUST bind the deferred processing state to that key or certificate thumbprint. Each continuation request MUST use the corresponding proof or certificate binding as defined in {{continuation-request}}. The specific mechanism by which the authorization request conveys a sender-constraining key is out of scope for this specification.

## Authorization Endpoint Errors

When the authorization server cannot create deferred processing state for an authorization request, it returns the appropriate authorization endpoint error response defined by {{RFC6749}} in place of a deferred authorization response. The authorization server MUST NOT issue a `deferred_code` to defer reporting of a known-fatal error.

If the `redirect_uri` cannot be validated, the authorization server MUST NOT redirect the user agent to the unvalidated URI and MUST handle the failure as required by {{RFC6749}} for invalid redirect URIs, typically by displaying an error to the resource owner. In particular, the authorization server MUST NOT issue a `deferred_code` to an unvalidated redirect URI.

## Example: Front-Channel Step-Up

A client initiates an authorization code request:

~~~ http
GET /authorize?
  response_type=code&
  client_id=s6BhdRkqt3&
  redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb&
  scope=transfer.execute&
  state=xyz&
  code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
  code_challenge_method=S256 HTTP/1.1
Host: as.example.com
~~~

The authorization server determines that the requested operation requires step-up authentication and a manager approval, and defers the authorization. It redirects the user agent back to the client:

~~~ http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?
  deferred_code=8N5B2K1&
  state=xyz&
  expires_in=900&
  interaction_uri=https%3A%2F%2Fas.example.com%2Finteract%2F8N5B2K1
~~~

The client presents the `interaction_uri` to the user, then submits a continuation request with the PKCE verifier:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred_code&
deferred_code=8N5B2K1&
code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk&
client_id=s6BhdRkqt3
~~~

The authorization server verifies the PKCE verifier, finds processing still in progress, and returns:

~~~ json
{
  "error": "authorization_pending",
  "deferred_code": "9P2K7M3",
  "interval": 5,
  "expires_in": 880
}
~~~

Subsequent continuation requests omit the `code_verifier`. Once the step-up and approval complete, the continuation response returns an access token response directly.

# Token Endpoint Extensions

The following sections extend the OAuth token endpoint defined in {{RFC6749}}.

## Deferred Processing Responses {#deferred-processing-responses}

When the authorization server defers processing of a token request, it returns a token endpoint error response indicating deferred processing state.

Before issuing a deferred code, the authorization server MUST authenticate the client when client authentication is required and MUST validate the token request to the extent necessary to determine eligibility for deferred processing. The authorization server MUST NOT issue a deferred code for a malformed request, for an unauthenticated client that is required to authenticate, or for a request that is known to be unauthorized.

For unauthenticated clients, including public clients, the authorization server MUST NOT issue a deferred code unless the deferred processing state is bound to additional context that the requesting client instance MUST demonstrate on each continuation request. Acceptable bindings include DPoP {{RFC9449}}, mutual-TLS client certificate binding {{RFC8705}}, an authenticated user-agent session, a device-bound context, or an equivalent local mitigation. The binding MUST be sufficient to prevent another client instance that obtains the deferred code from completing the continuation.

For token endpoint originating requests, preserved PKCE {{RFC7636}} verification state alone is NOT a sufficient continuation binding. PKCE verification is performed once during evaluation of the originating token request, the `code_verifier` is consumed at that point, and continuation requests do not re-present it (see {{continuation-request}}). PKCE therefore proves the requestor of the originating token request possessed the verifier but does not authenticate the continuation requestor. PKCE MAY be combined with a sender-constraining or session-binding mechanism listed above to satisfy this requirement. The continuation binding requirements for authorization endpoint originating requests are defined separately in {{authorization-endpoint-pkce}}, where the `code_verifier` is presented for the first time on the first continuation request and can serve as the primary continuation binding.

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
: Deferred processing state exists, but processing of the originating request has not completed.

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

## Deferred Code Parameter

The `deferred_code` parameter is an opaque continuation reference representing deferred processing state.

Clients MUST treat the value as opaque.

The authorization server MUST NOT encode authorization, identity, or policy decisions in a way that requires client interpretation of the value.

## Interaction URI Parameter {#interaction-uri}

The `interaction_uri` parameter identifies a location where required external interaction can occur.

This specification does not define the semantics of the interaction.

The `interaction_uri` parameter MUST be an HTTPS URI. It MUST NOT include a fragment component.

Unless defined otherwise by a higher-layer profile, authorization servers SHOULD generate the `interaction_uri` using an origin trusted for the authorization server. Trust MAY be established through the issuer identifier, authorization server metadata, prior client configuration, or by matching the token endpoint origin. If an `interaction_uri` uses an origin different from both the issuer identifier and the token endpoint origin, the authorization server or higher-layer profile MUST define how clients determine that the origin is trusted for the authorization server.

The authorization server MUST bind the `interaction_uri` to the deferred processing state associated with the returned deferred code. Accessing the `interaction_uri` MUST NOT by itself authorize the originating request or complete deferred processing. Completion occurs only when the authorization server updates the deferred processing state according to local policy or a higher-layer profile.

The authorization server MUST make interaction URIs single-use or otherwise resistant to replay. The authorization server MUST limit the lifetime of interaction URIs to no longer than the lifetime of the associated deferred code.

If interaction at the `interaction_uri` can affect deferred processing state, the authorization server MUST authenticate the actor performing the interaction or bind the interaction to an existing authenticated session, device context, transaction challenge, or higher-layer profile mechanism sufficient for the sensitivity of the originating request.

The authorization server MUST NOT require clients to parse the `interaction_uri` or extract state from it. Clients MUST treat the `interaction_uri` as an opaque URI.

## Interval Parameter

The `interval` parameter indicates the minimum number of seconds the client SHOULD wait before submitting another continuation request.

If the client polls more frequently than permitted by the authorization server, the authorization server MAY return `slow_down` as defined by {{RFC8628}} or another token endpoint error response appropriate to the condition.

## Expires In Parameter

In a deferred processing response and in continuation responses returning `authorization_pending`, `interaction_required`, or `slow_down`, the `expires_in` parameter is REQUIRED and indicates the remaining lifetime of the deferred code in seconds. This usage parallels the `expires_in` parameter in the OAuth Device Authorization Grant {{RFC8628}} response, which describes the device code lifetime rather than an access token lifetime.

In a successful access token response returned upon deferred completion, the `expires_in` parameter retains the meaning defined for the original grant type and describes the lifetime of the issued access token.

Clients MUST NOT infer the lifetime of a deferred code from the lifetime of any access token that might be issued upon successful completion.

# Deferred Code Grant Type

The deferred code grant type uses the OAuth extension grant mechanism defined by {{RFC6749}} to continue processing of a previously submitted token request.

The deferred code grant type is a continuation mechanism. It does not create a new authorization grant from the resource owner, client, authorization server, or external approver. The authorization grant, if any, remains the authorization grant represented by the originating request.

Authorization semantics are derived exclusively from the originating request associated with the deferred code.

The deferred code grant type MUST NOT:

* modify the originating request,
* expand requested authorization,
* change requested resources,
* alter sender-constraining requirements,
* change the authenticated client identity associated with the original request.

The authorization server MUST bind the deferred code to the originating request and its associated security context.

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
: REQUIRED if the client is not authenticating with the authorization server and the originating request included a client identifier. The value MUST identify the same client as the originating request. This follows the unauthenticated client request pattern defined for the token endpoint in {{RFC6749}}.

code_verifier:
: REQUIRED on the first continuation request following a deferred authorization response when the originating authorization request included a `code_challenge` parameter, as defined in {{authorization-endpoint-pkce}}. MUST NOT be included on any other continuation request.

The request MUST NOT include parameters that modify or replace parameters from the originating request, including `scope`, `resource` as defined by {{RFC8707}}, `audience`, `authorization_details` as defined by {{RFC9396}}, `redirect_uri`, `subject_token`, `actor_token`, `assertion`, or grant-type-specific parameters from the originating request. The authorization server MUST reject such requests with `invalid_request`. The `code_verifier` parameter is permitted on continuation requests only as defined above for authorization endpoint deferral.

If the originating request was made by an authenticated client, the continuation request MUST authenticate the same client. If the originating request was made by an unauthenticated client that included a `client_id`, the continuation request MUST include the same `client_id`. If the originating request used sender-constrained client authentication or proof-of-possession material, the authorization server MUST apply equivalent sender-constraining requirements to the continuation request.

When the client uses a client authentication mechanism that requires a fresh credential per request, such as the `client_assertion` and `client_assertion_type` parameters defined by {{RFC7521}}, the continuation request MAY present a freshly minted credential of the same type that authenticates the same client. The authorization server MUST verify that the freshly presented client credential identifies the client bound to the deferred processing state.

If the originating request included a valid DPoP proof {{RFC9449}}, the authorization server MUST bind the deferred processing state to the DPoP public key thumbprint. Each continuation request MUST include a valid DPoP proof for the token endpoint, and the DPoP proof key MUST match the key bound to the deferred processing state. If the DPoP proof is invalid, the authorization server returns the error response defined by {{RFC9449}}. If the proof is valid but does not match the deferred processing state binding, the authorization server MUST reject the continuation request with `invalid_grant`.

If the originating request used mutual-TLS client certificate binding {{RFC8705}}, the authorization server MUST bind the deferred processing state to the certificate or certificate thumbprint used for the original request. Each continuation request MUST use certificate binding that matches the deferred processing state. If the certificate binding does not match, the authorization server MUST reject the continuation request with `invalid_grant`.

If the access token produced by successful completion is sender-constrained, the confirmation or binding material for that access token MUST be derived from the originating request and continuation security context. The authorization server MUST NOT allow a continuation request to replace sender-constraining key material from the originating request.

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

The authorization server MUST verify that the deferred code is valid, unexpired, and bound to the client and security context of the originating request.

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

When processing completes successfully, the authorization server returns an access token response appropriate to the originating request and MAY include any parameters normally returned for the corresponding grant type or response type. For example, a bearer access token response is defined by {{RFC6750}}. For authorization endpoint originating requests, the successful completion response is the access token response defined by {{RFC6749}} for the token endpoint, returned directly without any intermediate authorization code redemption.

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

If the authorization server determines that the originating request cannot be completed successfully, it returns a token endpoint error response appropriate to the reason for failure. The authorization server SHOULD return `access_denied` when the failure results from an authorization decision reached during deferred processing, such as policy, attestation outcome, or approval. Other token endpoint error codes defined by {{RFC6749}}, {{RFC8628}}, or higher-layer profiles MAY be used when more specific. The authorization server MUST NOT return `invalid_grant` for unsuccessful completion of an otherwise valid deferred request; that error is reserved for an invalid or unbound deferred code (see {{invalid-deferred-code}}).

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

The notification mode applies equally to deferred processing state created from a token endpoint originating request and to state created from an authorization endpoint originating request. For authorization endpoint originating requests, notifications can be particularly useful because the client does not otherwise hold an open connection to the authorization server and would otherwise rely entirely on polling.

The notification token defined in this specification is issued by the authorization server and validated by the client. This is the inverse of the `client_notification_token` model defined by {{CIBA}}, in which the client issues the token and the authorization server presents it. Higher-layer profiles that require client-issued notification tokens for compatibility with CIBA MAY layer such a mechanism on top of this specification.

## Client Notification Endpoint

A client that wishes to receive notifications registers a notification endpoint URL using the client metadata parameter `deferred_code_notification_endpoint`. Client metadata is registered as defined by {{RFC7591}} or by an authorization server's local registration mechanism.

The endpoint MUST be an HTTPS URI. It MUST NOT contain a fragment component.

## Notification Token Parameter

When the authorization server defers an originating request from a client that has registered a notification endpoint and intends to deliver notifications for the deferred processing state, continuation responses MAY include a `notification_token` parameter. The deferred processing response for a token endpoint originating request MAY also include a `notification_token`.

For authorization endpoint originating requests, the `notification_token` MUST NOT be conveyed in the deferred authorization response. The redirect query component is observable in browser history, server logs, and referer headers; placing a bearer credential there would expose it to the same disclosure surface as the deferred code itself, weakening notification authentication. The authorization server MUST defer issuance of the `notification_token` to the first continuation response.

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

# Originating Request Applicability

This specification applies to any OAuth token request and to any OAuth authorization request.

When an authorization server defers an originating request, it MUST preserve the security properties of the corresponding grant type or response type. Deferred processing MUST NOT make a one-time credential reusable, extend the validity of an expired credential, bypass proof requirements, or allow the client to substitute different grant inputs during continuation.

The following subsections describe per-grant-type considerations for token endpoint originating requests. Considerations for authorization endpoint originating requests are addressed in {{authorization-endpoint-extensions}}; this section does not duplicate them.

## Authorization Code Grant

An authorization server MAY defer processing during authorization code redemption.

If an authorization server defers authorization code redemption, it MUST treat the authorization code as consumed or otherwise unusable outside the deferred processing state. A later continuation request resumes the deferred redemption; it is not a second authorization code redemption attempt. The authorization server MUST preserve PKCE {{RFC7636}} verification results, redirect URI validation, client binding, and any other authorization code grant checks performed before deferral.

PKCE verification, including comparison of the `code_verifier` against the previously stored `code_challenge` as defined by {{RFC7636}}, MUST occur during evaluation of the originating token request, before the authorization server creates deferred processing state. The `code_verifier` is consumed at that point and MUST NOT be re-presented on continuation requests, consistent with the prohibition on resubmitting originating request parameters in {{continuation-request}}. Continuation requests rely on the PKCE verification result preserved in deferred processing state. This rule applies to deferral of authorization code redemption at the token endpoint; the rules for authorization endpoint deferral are defined in {{authorization-endpoint-pkce}}.

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

When the originating request includes the `authorization_details` parameter defined by {{RFC9396}}, deferred processing state MUST preserve the originally requested authorization details. Continuation requests MUST NOT include `authorization_details`, and the authorization server MUST evaluate completion against the originally requested authorization details. Authorization decisions reached during deferred processing MAY narrow the granted authorization details but MUST NOT broaden them beyond what the original request expressed.

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

## Deferred Code Processing Supported Metadata

The `deferred_code_processing_supported` metadata parameter is a Boolean value indicating support for deferred code processing.

If omitted, the default value is `false`.

This parameter indicates that the authorization server can return deferred processing responses from the token endpoint and accepts the deferred code grant type. It does not indicate that every supported grant type or every token request is eligible for deferred processing.

An authorization server that supports this specification and publishes the `grant_types_supported` metadata parameter defined by {{RFC8414}} MUST include `urn:ietf:params:oauth:grant-type:deferred_code` in that parameter.

## Deferred Code Grant Types Supported Metadata

The `deferred_code_grant_types_supported` metadata parameter is a JSON array containing OAuth grant type values for which the authorization server can return deferred processing responses from the token endpoint.

This parameter is REQUIRED when `deferred_code_processing_supported` is `true`. When the authorization server publishes the `grant_types_supported` metadata parameter defined by {{RFC8414}}, the values of this parameter MUST be a subset of `grant_types_supported`. An empty array indicates that no originating-request grant types are currently eligible for deferred processing at the token endpoint, even though the authorization server otherwise supports the deferred code grant type.

The value `urn:ietf:params:oauth:grant-type:deferred_code` MUST NOT appear in this array because the array describes originating token request grant types, not the continuation grant type.

## Deferred Code Authorization Endpoint Supported Metadata

The `deferred_code_authorization_endpoint_supported` metadata parameter is a Boolean value indicating support for deferred authorization responses at the authorization endpoint, as defined in {{authorization-endpoint-extensions}}.

If omitted, the default value is `false`.

When `true`, the authorization server MAY return a deferred authorization response in place of an authorization response. The authorization server's policy for when authorization endpoint deferral is applied is implementation-defined and MAY be further constrained by higher-layer profiles.

## Deferred Code Notification Supported Metadata

The `deferred_code_notification_supported` metadata parameter is a Boolean value indicating support for the notification mode defined in {{notification}}.

If omitted, the default value is `false`.

When `true`, the authorization server MAY deliver notifications to clients that have registered a `deferred_code_notification_endpoint` and SHOULD include a `notification_token` in deferred processing responses for those clients when notifications will be delivered.

### Example

~~~ json
{
  "issuer": "https://as.example.com",
  "authorization_endpoint":
    "https://as.example.com/authorize",
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
  "deferred_code_authorization_endpoint_supported": true,
  "deferred_code_notification_supported": true
}
~~~

# Security Considerations

Implementations MUST follow the OAuth 2.0 Security Best Current Practice {{RFC9700}} in addition to the requirements in this section.

## Deferred Code Entropy

Deferred codes MUST contain sufficient entropy to resist guessing attacks.

The entropy requirement applies to any value that can be used as a bearer continuation reference at the token endpoint. Authorization servers SHOULD generate deferred codes using a cryptographically secure random source or protect structured values using integrity and confidentiality mechanisms appropriate for bearer artifacts.

## Deferred Code Binding

The authorization server MUST bind the deferred code to the originating request and associated security context.

The binding MUST include the client identity when a client identity is present. It MUST include sender-constraining or proof-of-possession material used by the originating request, including DPoP public key thumbprints {{RFC9449}} or mutual-TLS certificate bindings {{RFC8705}}. It SHOULD include requested resources, requested scopes, grant-type-specific input artifacts, and any other values that affect token issuance.

## Proof-of-Possession Continuity

Deferred processing MUST preserve proof-of-possession and sender-constraining properties from the originating request through every continuation request and the final access token response.

When the originating request used DPoP, each continuation request MUST use a valid DPoP proof for the token endpoint with the same DPoP key bound to the deferred processing state. When the originating request used mutual TLS, each continuation request MUST use certificate binding that matches the deferred processing state.

Authorization servers MUST reject continuation requests that attempt to replace, omit, or weaken proof-of-possession or sender-constraining material from the originating request. If a proof is syntactically or cryptographically invalid, the authorization server returns the error defined by the proof mechanism. If the proof is valid but does not match the deferred processing state, the authorization server returns `invalid_grant`.

When DPoP nonces {{RFC9449}} are used, the authorization server MAY require a fresh DPoP proof bound to a current nonce on each continuation request and MAY return `use_dpop_nonce` with an updated nonce as defined by {{RFC9449}}. A `use_dpop_nonce` response MUST NOT terminate the deferred processing state. The client MUST retry the continuation request with a DPoP proof that incorporates the updated nonce. Nonce rotation is independent of deferred code rotation; either, both, or neither MAY occur on a given continuation response.

## Client Authentication

The authorization server MUST require client authentication equivalent to the originating request when the original request used client authentication.

For unauthenticated clients, including public clients, the authorization server MUST require the continuation request to identify the same client as the originating request when a client identifier was present. Client identification alone does not authenticate a public client or prove possession of the same client instance. Authorization servers MUST bind deferred processing state to additional proof or context that the requesting client instance demonstrates on each continuation request, such as DPoP proof {{RFC9449}}, mutual-TLS client certificate binding {{RFC8705}}, an authenticated user-agent session, device-bound context, or other sender-constraining material applicable to the originating request.

For token endpoint originating requests, preserved PKCE {{RFC7636}} verification state is NOT, by itself, a sufficient continuation binding for public clients: the `code_verifier` is consumed during the originating token request and is not re-presented on continuation requests, so PKCE state cannot authenticate the continuation requestor. PKCE MAY be combined with one of the bindings above but does not substitute for one.

For authorization endpoint originating requests, the `code_verifier` is presented on the first continuation request as defined in {{authorization-endpoint-pkce}}, and that verifier MAY serve as the continuation binding for the originating client instance. Authorization servers MAY still require an additional sender-constraining or session-binding mechanism for high-sensitivity originating requests or for continuation requests after the first.

Authorization servers MUST NOT allow a continuation request to weaken client authentication, sender-constraining, or proof-of-possession requirements from the originating request.

## Deferred Code Replay

Authorization servers SHOULD detect replay of invalidated or rotated deferred codes.

Replay detection is particularly important when a deferred code is returned through an interaction channel, logged by intermediaries, or exposed to user agents.

## Deferred Code Rotation

Authorization servers MAY rotate deferred codes during continuation processing.

Clients MUST discard replaced values.

When rotating deferred codes, authorization servers SHOULD invalidate the previous value promptly after issuing the replacement.

If a deferred code is not sender-constrained or otherwise bound to proof-of-possession material, authorization servers MUST rotate the deferred code after each valid continuation request that returns `authorization_pending`, `interaction_required`, or `slow_down`. Authorization servers MAY omit rotation when the deferred code is sender-constrained or protected by an equivalent replay-resistant mechanism.

## Authorization Endpoint Deferral

A `deferred_code` delivered through a deferred authorization response is conveyed via the user-agent redirect, and is therefore subject to the same disclosure surface as an authorization code, including browser history, referer headers, server access logs, and any user-agent extensions that observe redirect traffic.

Authorization servers MUST treat the `deferred_code` returned from the authorization endpoint with at least the protection applied to authorization codes. The deferred code binding, rotation, and sender-constraining requirements defined in {{deferred-code-semantics}} and the deferred code entropy and binding requirements in this section apply unchanged to authorization endpoint deferral.

When PKCE was used on the originating authorization request, the `code_verifier` presented on the first continuation request serves as the primary continuation binding for the originating client instance. When PKCE was not used, the authorization server MUST require sender-constraining or session-binding on continuation requests sufficient to prevent another client instance that obtains the deferred code from completing the continuation, consistent with the public-client continuation binding requirements in {{deferred-processing-responses}}.

Authorization servers MUST validate the `redirect_uri` of the originating authorization request and complete any other authorization request validation that does not depend on completion of the deferred work before issuing the deferred authorization response. Authorization servers MUST NOT use the deferred authorization response mechanism to defer reporting of an authorization request that is known to be unauthorized or malformed.

The `interaction_uri` returned in a deferred authorization response is subject to the same protections as an `interaction_uri` returned in a deferred processing response.

Authorization servers SHOULD apply more conservative lifetime defaults to deferred codes returned through the authorization endpoint redirect than to deferred codes returned only at the token endpoint, because the redirect channel exposes the value to browser history, server logs, and referer headers that the token endpoint does not.

Authorization servers SHOULD treat deferred codes returned through the authorization endpoint redirect as one-time-use even at short lifetimes, rotating the value on the first continuation response that does not complete the request. The redirect channel exposes the initially returned value to multiple log surfaces; the value rotated on first continuation arrives only at the token endpoint and is not subject to the same disclosure surface.

When a deferred code returned through the authorization endpoint redirect has a lifetime measured in hours or days under the rules of {{lifetime-considerations}}, the authorization server MUST sender-constrain the deferred code through DPoP {{RFC9449}}, mutual-TLS client certificate binding {{RFC8705}}, or an equivalent mechanism. PKCE {{RFC7636}}, while required as a continuation binding for public clients by {{authorization-endpoint-pkce}}, is not by itself sufficient for long-lived front-channel deferred codes: PKCE binds the client instance that originated the request, but a stolen browser session retaining access to the original client instance can complete continuation. Sender-constraining the deferred code itself defends against this case.

Authorization servers SHOULD bind deferred processing state created from an authorization endpoint originating request to enough device or session context to detect when continuation requests arrive from a context inconsistent with the originating request. Inconsistent context includes a different network origin, a different user-agent fingerprint, or other deployment-specific signals available to the authorization server. Detection of inconsistent continuation context is not by itself grounds for rejection (legitimate cross-device flows can produce inconsistent context), but SHOULD be reported as a security event for monitoring and MAY trigger additional risk-based controls defined by local policy or higher-layer profiles.

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

## Deferred Processing Lifetime {#lifetime-considerations}

Authorization servers SHOULD limit deferred processing lifetime and invalidate expired deferred codes.

Long-lived deferred processing state increases replay risk, can retain sensitive request context, and increases the likelihood that completion will evaluate against assumptions that have changed since the originating request, for example policy updates, attestation expiry, certificate revocation, or account state changes. Authorization servers SHOULD choose lifetimes that are no longer than necessary for the expected asynchronous processing or interaction.

For asynchronous policy evaluation, attestation validation, and out-of-band human-in-the-loop approval delivered through push notification, in-app inbox, or similar channels with sub-hour resolution, deferred code lifetimes SHOULD be measured in minutes.

Long-lived deferred processing state, measured in hours or days, is appropriate for use cases including enterprise governance approval workflows, identity verification involving document submission, and autonomous-agent authorization requests pending human review. Authorization servers MAY use such lifetimes when all of the following controls are in place:

* sender-constraining of the deferred code through DPoP {{RFC9449}}, mutual-TLS client certificate binding {{RFC8705}}, or an equivalent mechanism;
* one-time-use deferred codes, with rotation on each continuation response that does not complete the request;
* an explicit user or administrator approval semantic that justifies the lifetime, such that the deferred state corresponds to a real stakeholder decision rather than a passive timeout;
* operational monitoring of long-lived deferred state, including detection of replay, code rotation lag, and unusual completion patterns.

Authorization servers using long-lived deferred codes SHOULD apply protections equivalent to those required for refresh tokens by {{RFC9700}} and SHOULD re-evaluate authorization-relevant inputs (policy, attestation, account state) at completion time rather than relying solely on the snapshot captured when deferred processing state was created.

## Authorization Scope Stability

Continuation processing MUST NOT expand the requested authorization beyond the originating request. Authorization servers MUST evaluate the continuation request using the originating request context and MUST reject attempts to substitute new requested resources, scopes, subject tokens, assertions, or other grant inputs. This requirement applies equally to deferred token endpoint requests and deferred authorization endpoint requests.

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

Authorization servers MUST validate registered notification endpoint URLs at registration time and prior to each delivery. Authorization servers MUST reject endpoints whose host components resolve to loopback, link-local, multicast, broadcast, or other non-public addresses. Higher-layer profiles or explicit deployment policy MAY permit internal destinations, but the base specification requires public destinations. Authorization servers MUST NOT follow HTTP redirects from the notification endpoint.

Authorization servers SHOULD apply rate limits to outgoing notifications, both per deferred processing state (to prevent a runaway state-transition loop from producing many notifications) and per registered notification endpoint (to bound the load placed on any single client endpoint). Outgoing rate-limit enforcement complements the client-side rate limit required below.

The notification endpoint is publicly reachable and receives POST requests from the public internet. Clients MUST validate the inbound bearer credential against the most recent `notification_token` value before treating the notification as authentic. Clients SHOULD apply rate limiting to the notification endpoint to mitigate abuse.

## Notification Token Protection

The `notification_token` is a bearer credential. Authorization servers MUST issue notification tokens with sufficient entropy to resist guessing and MUST transmit them only over TLS-protected responses to the client.

Authorization servers SHOULD limit notification token lifetime to no longer than the lifetime of the associated deferred processing state and SHOULD invalidate notification tokens promptly when the deferred processing state ends.

Authorization servers MUST NOT reuse a `notification_token` value across distinct deferred processing states.

## Notification Information Leakage

The notification payload conveys only the affected `deferred_code`. Authorization servers MUST NOT include access tokens, identity claims, attestation outcomes, policy results, subject identifiers, or other state in the notification body or headers.

Notification timing is observable to anyone with access to the client's notification endpoint traffic, including shared hosting infrastructure, CDN access logs, operational dashboards, and any intermediaries between the authorization server and the client endpoint. The time between deferral and notification, the time between notifications, and the absence of expected notifications all leak information about deferred processing state, including approval latency, attestation evaluation duration, manager-review wait time, and policy evaluation cost.

Authorization servers SHOULD avoid using notification timing or delivery patterns that would reveal sensitive information about deferred processing state, consistent with the oracle resistance requirements above. Where timing observability is a material concern, authorization servers SHOULD apply jitter to notification delivery, deliver notifications on a coarse-grained schedule rather than on every state transition, or rely on polling without notifications.

## Privacy Considerations

Deferred processing can retain information from the originating request, including subject identifiers, requested resources, policy evaluation inputs, attestation evidence, or transaction details. Authorization servers SHOULD minimize retained state, protect it at rest, and delete it when processing completes or expires.

Interaction URIs and deferred codes SHOULD NOT reveal user identifiers, client identifiers, resource identifiers, transaction details, or policy decisions to parties that can observe browser history, logs, referrer headers, or network metadata.

# IANA Considerations

## OAuth Parameters Registry

This specification registers the following parameters in the "OAuth Parameters" registry established by {{RFC6749}}.

Parameter name:
: `deferred_code`

Parameter usage location:
: authorization response, token response, token request

Change controller:
: IETF

Specification document(s):
: This document

Parameter name:
: `interaction_uri`

Parameter usage location:
: authorization response, token response

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
: JSON array containing OAuth grant type values for which the authorization server can return deferred processing responses from the token endpoint.

Change controller:
: IETF

Specification document(s):
: This document

Metadata name:
: `deferred_code_authorization_endpoint_supported`

Metadata description:
: Boolean value indicating support for deferred authorization responses at the authorization endpoint.

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

This specification registers the following parameters in the "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}.

Client Metadata Name:
: `deferred_code_processing_supported`

Client Metadata Description:
: Boolean value indicating whether the client supports receiving deferred processing responses at the token endpoint.

Change Controller:
: IETF

Specification Document(s):
: This document

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

This specification defines a generic Async Token Request layer for OAuth: a single continuation reference, polling state machine, and completion vocabulary that apply uniformly across originating endpoints. The originating request may be a token request at the token endpoint or an authorization request at the authorization endpoint; the continuation interface is the token endpoint in both cases.

## Why a Unified Async Layer

OAuth's existing async story is fragmented. The OAuth Device Authorization Grant {{RFC8628}} defines its own continuation handle (`device_code`) and polling state machine for input-constrained devices. {{CIBA}} defines its own continuation handle (`auth_req_id`) and a separate backchannel authentication endpoint for end-user authentication. {{JWT-GRANT-INTERACTION}} defines interaction semantics specific to JWT bearer requests. {{RFC8693}} token exchange defines its own typed-token response shape. Each addresses a particular origination case with its own vocabulary, its own state machine, and its own error semantics.

This specification factors those patterns into a single substrate. The same `deferred_code`, the same continuation grant type, the same `authorization_pending` / `interaction_required` / `slow_down` / `expired_token` vocabulary, and the same polling state machine apply whether the originating request was a token endpoint request, an authorization endpoint request, or a future deferral case that has not yet been standardized. Existing flows such as Device Authorization Grant and CIBA can be understood as profiles of this substrate that pre-date its generalization.

The substrate framing also makes adding deferral cheap. A new deferral case requires defining only how the initial `deferred_code` is delivered (response shape, conveyance channel) and any case-specific eligibility rules; it inherits the continuation, completion, cancellation, notification, and metadata machinery unchanged.

## Why Token Endpoint Continuation Regardless of Originating Endpoint

The continuation interface is the token endpoint in all cases, even when the originating request arrived at the authorization endpoint. Three reasons:

* The completion artifact is an access token. The token endpoint already defines the access token response shape, error vocabulary, and sender-constraining controls (DPoP, mutual TLS, client authentication). Routing continuation through the token endpoint avoids duplicating those controls at a parallel endpoint.
* The continuation is a back-channel operation. The originating authorization request used the user agent because the resource owner needed to be present; continuation does not, because the resource owner's part is either complete or pending external action. Forcing continuation through the user agent would add latency without adding security.
* The state machine is identical regardless of origin. Pending, Interaction Required, Complete, Denied, Expired, Invalid are the same conditions whether the deferred state was created from a token request or an authorization request. A single endpoint hosting that state machine is simpler than two.

## Why the Error Response Encoding

Deferred processing responses at the token endpoint use the OAuth token endpoint error response form (`400 Bad Request` plus an `error` value) rather than a successful token response carrying a typed deferral artifact.

The error encoding has several properties that the typed-token alternative does not:

* **Uniform applicability across grant types and endpoints.** Every OAuth grant type defines a 400-error response shape with the same structure (`error`, `error_description`, `error_uri`). The error encoding reuses that shape unchanged for any originating flow, including grant types that do not otherwise emit typed-token responses such as `authorization_code`, `client_credentials`, and `refresh_token`. A typed-token deferral encoding would require those grant types to adopt a token-exchange-shaped response solely to recognize deferral.
* **Naïve-client safety.** OAuth client libraries dispatch on HTTP status. Under the 400-error encoding, a client unaware of this specification observes an unrecognized OAuth error and abandons cleanly. Under a 200-encoded typed-token alternative, an unaware client could populate its `access_token` slot with a deferral artifact and attempt to use it as a bearer credential, producing 401 responses at resource servers and potentially driving the client to re-run the original authorization flow.
* **Field separation.** The deferred handle is in a field structurally distinct from `access_token`, eliminating the possibility that a deferral artifact is mistakenly used as a bearer credential.
* **Composition with other 400-encoded controls.** DPoP nonce challenges {{RFC9449}}, mutual-TLS binding mismatches {{RFC8705}}, and PKCE failures {{RFC7636}} all surface as 400 errors. The deferred processing response shares this response pipeline; a single 400-handling path on the client covers all of them.
* **Natural fit for the interaction-required signal.** The `interaction_required` error code is already registered in the OAuth Extensions Error Registry by {{OIDC-CORE}}. Encoding "external interaction is required before processing can continue" as a token endpoint error code aligned with the existing interaction-required vocabulary keeps the interaction signal native to the protocol rather than profile-specific.

Authorization servers operating observability tooling that treats 4xx responses as errors should classify deferred processing responses separately from terminal errors, recognizing that `authorization_pending`, `interaction_required`, and `slow_down` represent ongoing operation rather than failure.

Neither status code is semantically clean for the underlying state: 200 implies a successful issuance that has not occurred, and 400 implies a client error that did not happen. Both are wire-compatibility compromises against existing OAuth precedents. This specification chooses 400 to align with the polling encoding of {{RFC8628}} and to obtain the uniformity, safety, and composition properties listed above.

# Relationship to Existing OAuth Deferred Interaction Mechanisms

OAuth and OpenID Connect already define several mechanisms involving deferred completion, polling, or out-of-band interaction. This specification generalizes the underlying pattern: a continuation reference, a polling state machine, and a completion vocabulary, independent of how the originating request was made. Existing mechanisms can be understood as predecessors that solved specific cases before this generalization existed.

## Relationship to OAuth Device Authorization Grant

The OAuth Device Authorization Grant {{RFC8628}} defines a separate authorization flow designed for input-constrained devices.

In the Device Authorization Grant:

* the client first obtains a device code from a dedicated device authorization endpoint,
* the user completes interaction through a separate verification URI,
* the client polls the token endpoint for completion using the device code.

This specification differs in several ways:

* it does not define a separate device authorization endpoint; deferral applies to existing token endpoint and authorization endpoint requests,
* it applies to arbitrary token requests and authorization requests rather than a single device-oriented flow,
* it allows deferred processing after receipt of an otherwise valid originating request, including non-interactive cases such as policy evaluation or attestation,
* it supports both interaction-based and non-interactive asynchronous processing.

The Device Authorization Grant polling vocabulary (`authorization_pending`, `slow_down`, `expired_token`) is reused unchanged in this specification, with `deferred_code` taking the role that `device_code` plays in the device flow.

## Relationship to OpenID Connect CIBA

This specification is distinct from {{CIBA}}.

CIBA defines an OpenID Connect authentication flow in which a Relying Party initiates end-user authentication by sending an authentication request to a dedicated backchannel authentication endpoint. CIBA returns an `auth_req_id` identifying the authentication request and defines poll, ping, and push modes for obtaining or delivering the authentication result.

This specification instead defines a generic OAuth deferred-processing substrate. It does not define a backchannel authentication endpoint, an authentication request object, push token delivery, or an `auth_req_id`. The deferred code mechanism applies to originating requests that already use existing OAuth endpoints, including the authorization endpoint and the token endpoint.

The key differences are:

* Protocol layer: CIBA is an OpenID Connect authentication flow. This specification is an OAuth substrate for deferred originating requests.
* Initiation point: CIBA starts with a request to a dedicated backchannel authentication endpoint. This specification starts with an ordinary OAuth request to either the authorization endpoint or the token endpoint.
* Continuation handle: CIBA uses an `auth_req_id` for an authentication transaction. This specification uses a `deferred_code` for suspended originating-request processing state.
* Subject requirement: CIBA is centered on authenticating an end-user identified to the OpenID Provider. This specification does not require an end-user and can apply to client credentials, token exchange, assertion grants, refresh tokens, authorization requests, and other originating requests.
* Interaction semantics: CIBA defines an out-of-band end-user authentication and consent model. This specification only signals that external interaction may be required and leaves the interaction semantics to the authorization server or a higher-layer profile.
* Completion model: CIBA defines poll, ping, and push modes. This specification defines token endpoint polling through the deferred code grant type and an OPTIONAL state-change notification mode (see {{notification}}); push delivery of access tokens or identity assertions remains outside its scope.
* Response semantics: CIBA is designed to return OpenID Connect authentication results, including ID Token semantics. This specification returns the access token response appropriate to the originating request's grant type or response type and does not define ID Token processing.
* Applicability: CIBA defines a specific authentication flow. This specification defines a general suspension and continuation pattern for OAuth originating requests.

An authorization server that supports both CIBA and this specification SHOULD use CIBA when the client is initiating an OpenID Connect backchannel authentication flow for an end-user. It SHOULD use this specification when an existing OAuth originating request cannot complete synchronously and the authorization server needs to preserve and later resume evaluation of that request.

Higher-layer profiles MAY combine this specification with OpenID Connect semantics, but doing so does not make the deferred code grant type equivalent to the CIBA grant type. A profile that needs CIBA behavior, including backchannel authentication endpoint processing, `auth_req_id`, delivery modes, or ID Token authentication result semantics, SHOULD use or profile CIBA directly.

## Relationship to OAuth JWT Grant Interaction Response

The OAuth JWT Grant Interaction Response specification {{JWT-GRANT-INTERACTION}} defines interaction semantics for JWT bearer grant requests.

This specification generalizes those concepts across OAuth originating requests and introduces:

* generalized deferred processing semantics applicable to token requests and authorization requests,
* a deferred code grant type for continuation at the token endpoint,
* applicability to arbitrary grant types and authorization response types,
* non-interactive asynchronous processing.

This specification uses `interaction_required` as the error value for deferred processing that is waiting on external interaction, in both token endpoint continuation responses and deferred authorization responses where applicable. That value is aligned with OpenID Connect interaction terminology {{OIDC-CORE}} and the pending-interaction polling state described by {{JWT-GRANT-INTERACTION}}.

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

# End-to-End Examples

This appendix illustrates complete deferred flows. The examples are non-normative.

## Token Endpoint Deferral with Interaction

This example illustrates a deferred authorization code redemption at the token endpoint that includes external interaction.

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

## Authorization Endpoint Deferral with Step-Up

This example illustrates a deferred authorization endpoint flow in which the originating request requires step-up authentication and external approval. No authorization code is ever issued; continuation polling at the token endpoint completes the originating request directly.

A client sends an authorization code request that includes PKCE:

~~~ http
GET /authorize?
  response_type=code&
  client_id=s6BhdRkqt3&
  redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb&
  scope=transfer.execute&
  state=af0ifjsldkj&
  code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
  code_challenge_method=S256 HTTP/1.1
Host: as.example.com
~~~

The authorization server determines that the requested operation requires step-up authentication and a manager approval. Instead of issuing an authorization code, it defers the authorization and redirects the user agent back to the client:

~~~ http
HTTP/1.1 302 Found
Location: https://client.example.com/cb?
  deferred_code=8N5B2K1&
  state=af0ifjsldkj&
  expires_in=900&
  interaction_uri=https%3A%2F%2Fas.example.com%2Finteract%2F8N5B2K1
~~~

The client validates the `state` parameter, then surfaces the `interaction_uri` to the user. While the user completes step-up at the interaction URI, the client submits a first continuation request with the PKCE verifier:

~~~ http
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred_code&
deferred_code=8N5B2K1&
code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk&
client_id=s6BhdRkqt3
~~~

The authorization server verifies the PKCE verifier, finds processing still in progress, rotates the deferred code, and returns `interaction_required` to indicate the manager approval step is still outstanding. It also issues a notification token now that delivery to the redirect was complete:

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
  "notification_token": "9d2f6c4b8a1e7d3f5b9c0a2e4f6d8b1a",
  "interval": 5,
  "expires_in": 880
}
~~~

The client discards the previous `deferred_code` and continues polling with the replacement value. The `code_verifier` is not included on subsequent continuation requests. Once the user completes step-up and the manager approves the request, the authorization server transitions the state to Complete and the next continuation response returns an access token directly:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "SlAV32hkKG",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "8xLOxBtZp8",
  "scope": "transfer.execute"
}
~~~

No authorization code was issued at any point; the deferred code carried continuation state across both the front-channel and back-channel portions of the flow.

# Acknowledgements

This specification builds upon concepts explored in OAuth JWT Grant Interaction Response and OpenID Connect Deferred Transaction Resources.
