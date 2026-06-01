---
title: "OAuth 2.0 Interim Partial Completion and Grant Mode Parameter"
abbrev: oauth-interim-grant-mode
docname: draft-mcguinness-oauth-interim-grant-mode-latest
category: std

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"

keyword:
 - OAuth
 - Deferred Processing
 - Partial Completion
 - Interim
 - Grant Mode
 - OpenID Connect

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. McGuinness
    name: Karl McGuinness
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC7521:
  RFC8126:
  DEFERRED:
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-deferred-code-processing
    title: "OAuth 2.0 Deferred Request Processing"

informative:
  RFC9396:
  OIDC-CORE:
    target: https://openid.net/specs/openid-connect-core-1_0.html
    title: "OpenID Connect Core 1.0"

--- abstract

This specification defines the `grant_mode` request parameter and establishes the OAuth Grant Mode Values registry. A value in `grant_mode` signals that the client accepts the named *shape* or *handling* mode for the grant produced by the authorization server in response to the request. The parameter is orthogonal to the `completion_mode` parameter defined in "OAuth 2.0 Deferred Request Processing" {{DEFERRED}}; `completion_mode` carries values about *how* the request is completed, while `grant_mode` carries values about *what* the produced grant looks like or *how* the originating request is handled.

This specification also defines the first value registered in that registry, `interim`, and the interim partial-completion replacement semantics it signals for token endpoint responses. A client signaling `grant_mode=interim` accepts an initial response artifact (such as an OpenID Connect ID Token populated with currently-verified claims) issued together with a `deferred_code` for continuation; the upgraded artifact replaces the interim version when deferred processing finishes.

The combined approach of defining the `grant_mode` parameter alongside its first value follows the pattern of {{RFC9396}}, which defined the `authorization_details` mechanism alongside its initial values. Future specifications that define additional `grant_mode` values reuse the parameter and registry established by this document.

--- middle

# Introduction

The OAuth 2.0 Deferred Request Processing specification {{DEFERRED}} (the "base specification" or "substrate") defines a generic substrate for asynchronous completion of OAuth requests. It defines the `completion_mode` request parameter for clients to signal acceptance of deferred completion modes (how a request's result reaches the client) and a partial completion wire shape for responses that issue an initial artifact together with a `deferred_code` for asynchronous upgrade.

The substrate does not define a per-request signal for the *shape* of the produced grant or the *handling* of the originating request, as distinct from how the request is completed. Examples of such shape and handling modes include:

* A client accepting a partial-completion artifact populated with currently-available data (an *interim* representation) that will be upgraded asynchronously to a complete representation.
* A client accepting an authorization server invitation to push a narrowed revision of the originating request before the outcome is decided (a *revisable* handshake; defined in a separate profile).

This specification fills that gap by defining the `grant_mode` request parameter and establishing the OAuth Grant Mode Values registry, and by registering the first value, `interim`, with the partial-completion semantics that motivate it.

The motivating use case for `interim` is OpenID Connect identity verification flows where document review can take hours or days but the OpenID Provider can return preliminary claims (such as a verified email address and name) immediately while extended verification proceeds asynchronously. The substrate's partial completion wire shape carries the initial artifact and the `deferred_code`; this specification's `interim` value signals the client's acceptance of that pattern and defines the replacement semantics on continuation.

This specification:

* defines the `grant_mode` request parameter;
* defines `grant_mode` syntax, semantics, multi-value composition rules, and orthogonality with `completion_mode`;
* establishes the OAuth Grant Mode Values registry;
* registers the `interim` value and defines the partial-completion replacement semantics it carries;
* defines security considerations for both the parameter framework and the `interim` value.

This specification does not:

* define semantics for any `grant_mode` value other than `interim`;
* define OpenID Connect claim verification policy, identity-proofing rules, or relying-party trust decisions for interim artifacts;
* modify the deferred-code lifecycle, sender-constraining rules, or other parts of the substrate security model.

Profile specifications that consume the `interim` value MUST define which artifacts can be issued on an interim basis, how the interim nature is marked, and how the interim artifact is invalidated or constrained after continuation completes unsuccessfully. Future specifications that need to register additional `grant_mode` values reuse the parameter and registry established here.

## Relationship to the Base Specification's Publication

This specification has a normative dependency on {{DEFERRED}} (the base specification's `completion_mode` parameter, partial completion wire shape, and substrate security model) and is intended to advance to publication together with that specification. The dependency is asymmetric: the base specification stands on its own and does not require this document, but this document cannot be implemented without the base specification's partial completion wire shape and deferred-code continuation mechanism. Working groups considering adoption of this specification SHOULD coordinate with the adoption schedule of {{DEFERRED}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Substrate-level terminology used in this specification (such as "originating request", "deferred code", and "completion mode") is defined in {{DEFERRED}}. Terminology specific to this specification is defined in the next section.

# Terminology

This specification adopts the terminology defined in {{DEFERRED}}, including the terms "originating request", "deferred code", "deferred processing state", "continuation request", "completion mode", and the abstract state vocabulary. Additional terms specific to this specification:

Grant Mode:
: A named *shape* or *handling* mode for the grant produced by the authorization server, signaled by a value in the `grant_mode` request parameter defined in {{grant-mode-parameter}}.

Interim Representation:
: A response artifact issued together with a `deferred_code` in a partial completion response, populated with currently-available data and intended to be replaced asynchronously by a complete representation through deferred-code continuation.

# Relation to the Base Specification

This specification depends on the OAuth 2.0 Deferred Request Processing specification {{DEFERRED}} for:

* the partial completion wire shape (200 OK token response with an issued artifact, `deferred_code`, and `deferred_code_expires_in`);
* the deferred-code continuation mechanism and continuation grant type;
* the `completion_mode` parameter, which is composable with `grant_mode` on the same request (a client accepting both deferred completion and interim partial-completion signals `completion_mode=deferred grant_mode=interim`);
* the substrate security model (sender-constraining continuity, originating-request immutability, deferred-code lifetime, replay protection, and oracle resistance), which applies unchanged to the interim case;
* the §Profile-Defined Advisory Delivery Channels hook, available for profiles consuming `interim` that wish to deliver upgrade notifications through advisory channels rather than relying solely on polling.

This specification does not require modifications to the substrate's continuation grant type, deferred-code lifecycle, sender-constraining rules, or state machine.

# The grant_mode Parameter {#grant-mode-parameter}

This specification defines the `grant_mode` request parameter. A value in `grant_mode` signals that the client accepts the named *shape* or *handling* mode for the grant produced by the authorization server in response to the request that carries the parameter. The parameter applies to both authorization endpoint requests and token endpoint requests as defined in {{RFC6749}}.

The parameter is informed by OAuth's existing multi-valued request parameters (`scope`, `response_type`, `prompt`, `acr_values`); reviewers familiar with those parameters will recognize the structure even though each parameter has its own selection and composition rules.

## Syntax

The `grant_mode` parameter is OPTIONAL. Its value is a space-delimited, case-sensitive list of one or more grant mode values registered in the "OAuth Grant Mode Values" registry established by this specification.

If the parameter is omitted, the authorization server processes the request as if no grant mode was expressed.

## Semantics

A value in `grant_mode` is a request-level signal that the client accepts a grant produced in the named shape or handling mode for the request that carries the parameter. The authorization server MAY produce a grant exhibiting any listed mode, subject to its own policy and the request's eligibility. Presence of a value does not guarantee that the named mode will be applied.

When multiple values are present, the values are interpreted disjunctively: the client accepts a grant produced in any one of the listed modes. The authorization server MAY honor any of the listed values, in any combination compatible with the originating request and authorization server policy.

The `grant_mode` parameter does not declare durable client capability. Specifications defining `grant_mode` values SHOULD also define a corresponding client metadata field for durable capability, following the precedent of `response_type` and `response_types` in {{RFC6749}}.

The authorization server MUST NOT use the `grant_mode` parameter to expand authorization beyond what other request parameters express.

## Composition with completion_mode

The `grant_mode` parameter is orthogonal to the `completion_mode` parameter defined in {{DEFERRED}}. A client MAY signal both on the same request.

For example, a request carrying `completion_mode=deferred grant_mode=interim` indicates that the client accepts deferred completion AND accepts an interim partial-completion artifact shape. The authorization server MAY produce either outcome, both, or neither, depending on its policy and the request's eligibility.

Specifications defining new `grant_mode` values MUST specify whether and how their values interact with other values in the same parameter and with `completion_mode` values that may accompany them.

## Relationship to Other OAuth Request Parameters

The `grant_mode` parameter is modeled on OAuth's pattern of space-delimited, registry-backed request parameters. It is structurally similar to the following:

* **`scope`** (defined by {{RFC6749}}): the client requests a set of access scopes; the authorization server can grant a subset.
* **`response_type`** (defined by {{RFC6749}}): the client identifies the authorization response artifact set it is prepared to receive.
* **`prompt`** (defined by {{OIDC-CORE}}): the client lists authentication-flow behavior values; the authorization server applies the prompts.
* **`acr_values`** (defined by {{OIDC-CORE}}): the client lists authentication context class references.
* **`completion_mode`** (defined by {{DEFERRED}}): the client lists completion modes it accepts.
* **`grant_mode`** (defined by this specification): the client lists grant shape or handling modes it accepts.

The `grant_mode` parameter joins the existing `grant_*` family alongside `grant_type` defined by {{RFC6749}}:

* `grant_type` expresses the kind of grant flow being executed at the token endpoint.
* `grant_mode` expresses shape or handling modes the client accepts for the grant produced.

The `_type`/`_mode` distinction mirrors the same distinction in the `response_*` family: `_type` names the kind, `_mode` names the manner.

# The interim Value {#interim-value}

This specification registers the first value in the OAuth Grant Mode Values registry established above.

**interim**
: Client accepts an interim grant. The authorization server MAY return an initial response artifact populated with currently-available data together with a `deferred_code` for continuation. When deferred processing completes, the continuation response returns the complete artifact, replacing the interim version.

The `interim` value is a grant *shape* (what the produced grant looks like), as distinct from a completion *mode* (how the result reaches the client).

## Relationship to completion_mode=deferred

The `interim` value authorizes the partial-completion behavior defined in this section. It does not by itself indicate that the client accepts deferred completion with no immediately issued artifact. Deferred completion is signaled separately through the `completion_mode` parameter defined in {{DEFERRED}}.

A client that accepts either an interim artifact or ordinary deferred completion SHOULD send both signals on the same request:

~~~ text
completion_mode=deferred grant_mode=interim
~~~

A request containing only `grant_mode=interim` allows the authorization server to return an interim partial-completion response. If the same request cannot produce an interim artifact and requires ordinary deferred completion, the authorization server MUST NOT infer support for deferred completion from `interim` alone.

## Endpoint Applicability

The `grant_mode` parameter is registered for both authorization endpoint requests and token endpoint requests so future grant-mode values can define endpoint-specific behavior without redefining the parameter. The `interim` value defined by this specification is narrower: it defines only token endpoint partial-completion responses.

If an authorization endpoint request contains `grant_mode=interim`, this specification alone does not authorize the authorization server to return an interim authorization endpoint response. The authorization server MAY ignore the value for authorization endpoint processing, or MAY apply profile-defined behavior from another specification that explicitly defines authorization endpoint semantics for `interim` or for a more specific grant-mode value. Such a profile MUST define the response shape, client validation rules, and continuation behavior for that endpoint.

## Wire Shape

When a client sends `grant_mode=interim` at the token endpoint and the authorization server elects to produce an interim grant, the authorization server returns a partial completion response as defined in §Partial Completion of {{DEFERRED}}:

* A standard 200 OK token response carrying the interim representation (for example, an OpenID Connect `id_token`) populated with currently-available data.
* `deferred_code` and `deferred_code_expires_in` parameters for continuation.

A profile consuming this grant mode MUST define how the interim nature of the representation is marked, either using a response parameter or a marker inside the artifact itself. This specification does not mandate a specific marker; consuming profiles MUST specify one.

When the continuation completes successfully, the upgraded representation replaces the interim representation. The authorization server MUST invalidate the interim representation no later than when the continuation response returns the complete representation. When the partial completion response includes multiple credentials (for example, both an `access_token` and an `id_token`), the consuming profile MUST specify which of those credentials are interim and replaced by the upgraded response, and which (if any) remain valid independently of the upgrade. The authorization server MUST invalidate credentials marked as interim no later than when the continuation response issues their replacements, consistent with the substrate's cancellation and revocation rules.

If deferred processing completes unsuccessfully, the authorization server returns the appropriate token endpoint error response defined by {{DEFERRED}}. The profile MUST define whether the interim representation remains valid until its own expiration, is revoked immediately, or remains valid only for a narrowed purpose. If the unsuccessful completion invalidates any claim or authorization expressed by the interim representation, the authorization server MUST revoke or otherwise invalidate the interim representation.

## Continuation

The client polls the token endpoint using the deferred code grant type defined in {{DEFERRED}}. When the continuation response indicates completion, it returns a fresh 200 OK token response with the complete artifact replacing the interim version. The `deferred_code` is invalidated on completion as required by the base spec.

## Example

Token request signaling interim acceptance:

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

The `id_token` carries the verified-now claim set (for example, `email_verified` and `name`); the `deferred_code` provides the continuation handle for the upgrade.

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

The completion response carries the upgraded `id_token` in a token endpoint success response. The upgraded `id_token` replaces the interim `id_token`. Access token handling on completion follows the consuming profile's token issuance policy.

# Security Considerations

## Parameter Framework

The `grant_mode` parameter is a client-supplied signal of acceptance. The authorization server MUST NOT treat presence of a value as authorization to perform any action the originating request does not already authorize.

Specifications defining additional `grant_mode` values are responsible for the security analysis of their specific behavior. This specification defines security considerations only for the `interim` value defined here and for the parameter mechanics.

Specifications defining `grant_mode` values SHOULD also define a corresponding client metadata field for durable capability so that authorization servers can decide on profile applicability without relying solely on per-request signaling.

## Interim Representations

Interim representations carry data that is valid now and eligible to appear in the complete representation if deferred processing completes successfully. Profiles consuming the `interim` value MUST mark the interim representation as such so the consuming party can apply appropriate trust decisions. The substrate's §Higher-Layer Extension Points hook supports profile-defined response parameters for this purpose; profiles MAY also define the marker as part of the artifact itself (for example, a claim in an interim ID Token).

Interim data MUST NOT include claims, attributes, or authorization characteristics that would be absent from a successful complete representation for the same request. The interim representation is a valid partial representation, not a preview of unverified data. Profiles MUST define which subsets are eligible for interim issuance.

Profiles MUST define whether and how an interim representation is invalidated when deferred processing completes unsuccessfully. The default behavior assumed by this specification is that the interim representation remains valid until its own expiration when the unsuccessful completion does not invalidate the interim data; profiles MAY define stricter behavior (such as immediate revocation on any unsuccessful completion).

The `deferred_code` lifetime SHOULD match expected verification SLAs. For document verification this is typically measured in hours; for manual review it may be measured in days. For long-lifetime deferred codes, the §Deferred Processing Lifetime guidance of {{DEFERRED}} applies, including the required sender-constraining, one-time-use, explicit stakeholder approval semantic, and operational monitoring.

## Composition with Other Profiles

When a `grant_mode` value defined by this or another specification is used in combination with `completion_mode` values or other `grant_mode` values, profile authors MUST analyze whether the combination produces a different security posture than either value in isolation. In particular, combining `interim` with a profile that delivers the upgraded artifact through an advisory channel (such as a webhook or SSE stream) requires that the advisory delivery rules in {{DEFERRED}} §Profile-Defined Advisory Delivery Channels apply to the upgraded artifact.

## Substrate Security Model

All other security considerations of {{DEFERRED}} apply unchanged, including sender-constraining continuity, deferred-code rotation, replay protection, and oracle resistance. This specification does not weaken or modify those rules.

# IANA Considerations

## OAuth Parameters Registry

This specification registers the following parameter in the "OAuth Parameters" registry established by {{RFC6749}}:

Parameter name:
: `grant_mode`

Parameter usage location:
: authorization request, token request

Change controller:
: IETF

Specification document(s):
: This specification

## OAuth Grant Mode Values Registry

This specification establishes the "OAuth Grant Mode Values" registry. The registry contains values used in the `grant_mode` parameter defined in {{grant-mode-parameter}}.

The registration policy for new values is Specification Required as defined by {{RFC8126}}.

Initial registry contents:

Value:
: `interim`

Description:
: Client accepts an interim grant with continuation upgrade, as defined in {{interim-value}}.

Change controller:
: IETF

Specification document(s):
: This specification

--- back

# Design Rationale

## Why Bundle the Parameter and the First Value in One Specification

This specification defines the `grant_mode` parameter framework (parameter, registry, syntax, semantics, composition rules) alongside the first value registered in the registry (`interim`). It does not separate the framework into a standalone parameter-only specification.

The bundling pattern follows {{RFC9396}}, which defined the `authorization_details` parameter mechanism alongside its initial use case (Rich Authorization Requests for OAuth). Subsequent specifications register additional `authorization_details` types without redefining the parameter mechanics, and the framework spec's adoption was motivated by a concrete use case rather than as standalone scaffolding.

The same reasoning applies here. A standalone `grant_mode` parameter specification with no values would be infrastructure without a concrete user, which is hard to motivate for IETF adoption. Bundling the framework with a real value (`interim` for OpenID Connect identity verification flows) ensures that working group review is grounded in a concrete use case while still producing reusable infrastructure for future specifications.

Future specifications that define additional `grant_mode` values (for example, a profile defining a `revisable` value for request-revision handshakes) reference this specification for the parameter mechanics and register their values in the registry established here. The framework does not need to be re-litigated for each subsequent value.

## Why grant_mode is Distinct from completion_mode

The `completion_mode` parameter defined in {{DEFERRED}} carries values about *how* the request is completed: synchronously, asynchronously through deferred-code continuation, or through profile-defined channels such as push or streaming delivery. The `grant_mode` parameter defined here carries values about *what* the produced grant looks like, or *how* the originating request is handled.

A single combined parameter would force values from these two semantic categories to share a name. `completion_mode=interim` would be a category error (interim is not a mode of completion; it is the shape of the artifact). Conversely, `grant_mode=push` would also be a category error (push is delivery, not grant shape).

Two parameters with disjoint semantic responsibility allow each parameter's name to match what its values express. The parameters are otherwise structurally identical (space-delimited, registry-backed, disjunctive multi-value semantics) so the cognitive load of having two parameters is small.

# Acknowledgements

This specification builds upon the OAuth 2.0 Deferred Request Processing specification {{DEFERRED}}, including its partial completion wire shape and `completion_mode` parameter. The pattern of bundling a parameter framework with a first value definition is borrowed from {{RFC9396}}.
