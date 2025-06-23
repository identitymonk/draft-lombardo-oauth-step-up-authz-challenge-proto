---
title: "OAuth 2.0 step-up authorization challenge proto"
abbrev: "STEPZ"
category: info

docname: draft-lombardo-oauth-step-up-authz-challenge-proto-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Web Authorization Protocol"
keyword:
 - security
 - trust
venue:
  group: "Web Authorization Protocol"
  type: ""
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "identitymonk/draft-lombardo-oauth-step-up-authz-challenge"
  latest: "https://identitymonk.github.io/draft-lombardo-oauth-step-up-authz-challenge/draft-lombardo-oauth-step-up-authz-challenge-proto.html"

author:
 -
    fullname: Jean-François Lombardo
    nickname: Jeff
    organization: AWS
    country: Canada
    email: jeffsec@amazon.com

 -
    fullname: Alexandre Babeanu
    organization: IndyKite
    country: Canada
    email: alex.babeanu@indykite.com

 -
    fullname: Yaron Zehavi
    organization: Raiffeisen Bank International
    country: Austria
    email: yaron.zehavi@rbinternational.com

 -
    fullname: George Fletcher
    organization: Practical Identity
    country: United States of America
    email: george@practicalidentity.com

normative:
  RFC7519: # JWT
  RFC6749: # OAuth 2.0 Framework
  RFC6750: # OAuth 2.0 Authorization Framework: Bearer Token Usage
  RFC9068: # JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens
  D-OpenID-AuthZEN: #AuthZEN
    title: Authorization API
    target: https://openid.github.io/authzen/
    author:
    - name: Omri GAzitt
      role: editor
      org: Asserto
    - name: DAvid Brossard
      role: editor
      org: Axiomatics
    - name: Atul Tulshibagwale
      role: editor
      org: SGNL
    date: 2025
  RFC8259: #JSON
  RFC9396: # Rich Authorization Request
  RFC9126: # Pushed Authorization Request
  RFC9101: # JWT-Secured Authorization Request
  RFC7517: # JSON Web Key

informative:
  RFC7662: # Introspection
  RFC9470: # OAuth 2.0 Step Up Authentication Challenge Protocol
  RFC9728: # OAuth 2.0 Protected Resource Metadata
  RFC8414: # OAuth2 Authorization Server Metadata
  RFC9449: # DPoP
  I-D.ietf-oauth-v2-1: # OAuth 2.1
  I-D.lombardo-oauth-client-extension-claims: # OAuth2 Client extensions claims
  XACML:
    title: eXtensible Access Control Markup Language (XACML) Version 1.1
    target: https://www.oasis-open.org/committees/xacml/repository/cs-xacml-specification-1.1.pdf
    author:
    - name: Simon Godik
      role: editor
      org: Overxeer
    - name: Tim Moses (Ed.)
      role: editor
      org: Entrust
    date: 2006
  SP.800-162:
    title: Guide to Attribute Based Access Control (ABAC) Definition and Considerations
    target: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-162.pdf
    author:
    - name: Vincent Hu
      role: editor
      org: NIST
    - name: David Ferraiolo
      role: editor
      org: NIST
    - name: Richard Kuhn
      role: editor
      org: NIST
    - name: Adam Schnitzer
      role: editor
      org: BAH
    - name: Kenneth Sandlin
      role: editor
      org: MITRE
    - name: Robert Miller
      role: editor
      org: MITRE
    - name: Karen Scarfone
      role: editor
      org: Scarfone Cybersecurity
    date: 2014
  FAPI2.0-Security-Profiles:
    title: FAPI 2.0 Security Profile
    target: https://openid.net/specs/fapi-2_0-security-02.html
    author:
    - name: Dr Daniel Fett
      role: editor
      org: Authlete
    - name: Dave Tonge
      role: editor
      org: Moneyhub Financial Technology
    - name: Joseph Heenan
      role: editor
      org: Authlete
  hl7.fhir.uv.smart-app-launch:
    title: HL7 FHIR SMART App Launch
    target: https://www.hl7.org/fhir/smart-app-launch/app-launch.html#obtain-authorization-code
  MCP:
    title: Model Context Protocol
    target: https://modelcontextprotocol.io/specification/2025-06-18/basic


--- abstract

It is not uncommon for resource servers to require additional information like details of delegation authorization, or assurance proof of the delegation of authorization mechanism used according to the characteristics of a request. This document introduces a mechanism that resource servers can use to signal to a client that the  data and metadata  associated with
the access token of the current request does not meet its authorization requirements and, further, how to meet them. This document also codifies a taxonomy to guide the client into starting a new request towards the authorization server in order to get issued, if applicable, a new set of tokens matching the requirements.


--- middle

# Introduction

In simple authorization scenarios, an authorization server will determine what claims to embed in the tokens to issue on the basis of aspects such as the scopes requested, the resource, the identity of the client, and other characteristics known a provisioning time. [RFC9470] helped improve the feedback a resource server can provide to a client in case the user authentication method, authentication class, or the freshness of the authentication event did not meet the requirements expexted by the resource server. Although those approaches are viable in many situations, it falls short in several important circumstances, for instance, in [FAPI2.0-Security-Profiles] or [hl7.fhir.uv.smart-app-launch] regulated APIs when they require peculiar client authentication mechanisms to be enforced or transaction specific details to be present in the token. These requirements may depend upon resource access rules or policies implemented at a policy decision point it relies on, using a logic that is opaque to the authorization server.

This document extends the collection of error codes defined by [RFC6750] and by [RFC9470] with a new error codes, `failed_authorization` and `insufficient_authorization`, which can be used by resource servers to signal to clients that the authorization delegation represented by the access token presented with the request does not meet the authorization requirements of the resource server. This document also introduces associated payload definitions. The resource server can use these payloads to explicitly communicate to the client its authorization requirements.

The client can then use this information to reach back to the authorization server with a new authorization request that specifies the additional authorization details required for issuing tokens useable at the resource. This document does not describe any new methods to perform this additional authorization request but will rely on OAuth 2.0 Rich Auhtorization Request [RFC9396], OAuth 2.0 Pushed Authorization Request [RFC9126], or OAuth 2.0 JWT-Secured Authorization Request [RFC9101] for this purpose. These extensions will make it possible to implement interoperable step up authorization flows with minimal work from resource servers, clients, and authorization servers.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses the terms "access token", "authorization server", "authorization endpoint", "authorization request", "client", "protected resource", and "resource server" defined by [RFC6749].

# Protocol Overview

The following is an end-to-end sequence of a typical step up authorization scenario implemented according to this specification. The scenario assumes that the client obtained an access token for the protected resource before the sequence described below takes place.

    +----------+                                          +--------------+
    |          |                                          |              |
    |          |-----------(1) request ------------------>|              |
    |          |                                          |              |
    |          |<---------(2) challenge ------------------|   Resource   |
    |          |                                          |    Server    |
    |  Client  |                                          |              |
    |          |-----------(5) request ------------------>|              |
    |          |                                          |              |
    |          |<-----(6) protected resource -------------|              |
    |          |                                          +--------------+
    |          |
    |          |
    |          |  +-------+                              +---------------+
    |          |->|       |                              |               |
    |          |  |       |--(3) authorization request-->|               |
    |          |  | User  |                              |               |
    |          |  | Agent |<-----------[...]------------>| Authorization |
    |          |  |       |                              |     Server    |
    |          |<-|       |                              |               |
    |          |  +-------+                              |               |
    |          |                                         |               |
    |          |<-------- (4) access token --------------|               |
    |          |                                         |               |
    +----------+                                         +---------------+

_Figure 1: Abstract Protocol Flow_

1. The client requests a protected resource, presenting an access token.
2. The resource server determines that the circumstances in which the presented access token was obtained offer insufficient authorization details, wrong grant flow, or inadequate client authentication mechanism; it therefore denies the request and returns a challenge describing (using a combination of error code and payload details) what authorization requirements must be met to allow the request. It is possible here for the resource server to rely on the responses from an external policy decision point.
3. The client redirects the user agent to the authorization server with an authorization request that includes the authorization details indicated by the resource server in the previous step.
4. A new and adequate authorization sequence takes place between the user agent, the client and the authorization server, resulting in the issuance of a new access token that encapsulates the authorization level, or ceremonies requested by the resource server. The authorization server uses the payload for the resource server's response forwarded by the client to initiate the right grant flow type or to ensure that the authorization details are met. The authorization server may here also contact an external policy decision point to request evaluation of complex business access policies. The newly minted access token contains or references information about the authorization elements required, including but not limited to the claims defined in [I-D.lombardo-oauth-client-extension-claims].
5. The client repeats the request from step 1, presenting the newly obtained access token.
6. The resource server finds that the authorization details, grant flow or client authentication mechanism used during the acquisition of the new access token complies with its requirements and returns the representation of the requested protected resource.

Such protocol flow is coherent with the expectations of [FAPI2.0-Security-Profiles] and section 2.1.10.2.1 of [hl7.fhir.uv.smart-app-launch].

The validation operations mentioned in steps 2 and 6 imply that the resource server has a way of evaluating the authorization requirements that occurred during the ceremonies that led to the issuance of the access token. In the context of this document, the assessment by the resource server of the specific authorization mechanisms used to obtain a token for the requested resource is called an "authorization state".

This document does not describe how the resource server performs this assessment of the authorization state, whether the access token is a JSON Web Token (JWT) [RFC9068] or is validated via introspection [RFC7662] or whether the resource provider is performing this assessment natively or by offloading the assessment to a policy decision point as defined in [D-OpenID-AuthZEN], [XACML] or NIST's ABAC [SP.800-162]. This document rather describes how the resource provider tells the client what type of authorization it needs to get from the authorization server for a request.

The terms "authorization state" and "step up" are metaphors in this specification. These metaphors do not suggest that there is an absolute hierarchy of authorization states expressed in interoperable fashion. The notion of a state emerges from the fact that the resource server may only want to accept certain authorization mechanisms. When presented with a token derived from particuliar authorization mechanisms (i.e., a given authorization state) that it does not want to accept (i.e., below the threshold it will accept), the resource server seeks to step up (i.e., renegotiate) from the current authorization state to one that it may accept. The "step up" metaphor is intended to convey a shift from the original authorization state to one that is acceptable to the resource server.

Although the case in which the new access token supersedes old tokens by virtue of a higher authorization state is common, in line with the connotation of the term "step up authorization", it is important to keep in mind that this might not necessarily hold true in the general case. For example, for a particular request, a resource server might require a higher authorization state and a shorter validity, resulting in a token suitable for one-off calls but leading to frequent prompts: hence, offering a suboptimal user experience if the token is reused for routine operations. In such a scenario, the client would be better served by keeping both the old tokens, which are associated with a lower authorization state, and the new one: selecting the appropriate token for each API call. This is not a new requirement for clients, as incremental consent and least-privilege principles will require similar algorithms for managing access tokens associated with different scopes and permission levels. This document does not recommend any specific token-caching strategy: that choice will be dependent on the characteristics of every particular scenario and remains application-dependent as in the core OAuth cases. Furthermore, OAuth 2.0 [RFC6749] assumes access tokens are treated as opaque by clients. The token format might thus be unreadable to the client or might change at any time to become unreadable. So, during the course of any token-caching strategy, a client must not attempt to inspect the content of the access token to determine the associated authentication information or other details (see Section 6 of [RFC9068] for a more detailed discussion).

# Authorization Requirements Challenge

## HTTP Error Status Code

The resource server responds with a `403` HTTP status code using the Bearer authentication scheme's error parameter (from [RFC6750]) when a request compliant with this specification does not meet its authorization state requirements.

## Error Codes

This specification introduces two new error code values for this challenge and other OAuth authentication schemes, as defined in OAuth 2.0 Demonstrating Proof of Possession (DPoP)[RFC9449] and in OAuth 2.0 Step Up Authentication Challenge Protocol[RFC9470]:

`failed_authorization`:
: The authorization mechanisms used for the issuance of the access token presented with the request do not meet the authorization state requirements of the protected resource. It is up to the client to decide what to perform next; some further implementation-specific details MAY be provided in the payload of the response.

`insufficient_authorization`:
: The authorization mechanisms used for the issuance of the access token presented with the request do not meet the authorization state requirements of the protected resource. The client is provided a detailed standard response that describes exactly which new authorization mechanisms and details SHOULD be used in order to gain access to the requested resource. The client SHOULD then initiate a new ceremony with the authorization server, that comply with the stated resource requirements.

Note: the logic through which the resource server determines that the current request does not meet the authorization state requirements of the protected resource, and associated functionality (such as expressing, deploying and publishing such requirements), is out of scope for this document.

## WWW-Authenticate Header Error Codes And Associated Payloads

Each error code matches different types of HTTP response payloads, which guide the client into initiating the next Authorization Request.

### Response Payloads
This document specifies that all compliant HTTP responses must contain a payload, and that the contents of this payload depends on the error code issued by the resource server.

The payload is expected to be a JSON object, the details of which are provided in the following sections.

### `failed_authorization` Error

If the error code `failed_authorization` is used, the description field of the `WWW-Authenticate` HTTP Header MUST be set to `The authorization level is not met`.

Then the HTTP body payload MUST be formatted as an AuthZEN [D-OpenID-AuthZEN] response for a `decision` that MUST be set to `false` and MUST include as part of the response a `context` object. The `context` object SHOULD then include the following properties:

error_msg:
: _REQUIRED_ - a string representing a human readable justification of the resource provider access control leading to a `deny` of the request.

details:
: _REQUIRED_ - a valid [RFC8259] JSON structure

The `details` JSON structure MUST include only one of the following element:

expected_claims:
:  _OPTIONAL_ - a list of space-delimited, case-sensitive strings representing the claim names that MUST be provided in the access token.

expected_values:
:  _OPTIONAL_ - a valid JSON [RFC8259] structure that will indicate the claim names that MUST be provided as JSON keys and the values, expressed as a JSON array of values.

pdp_message:
: _OPTIONAL_ - a valid JSON [RFC8259] structure representing the error details the format of the policy decision point (PDP) that is the access authority for the resource provider. If the PDP is [D-OpenID-AuthZEN] compliant, then the `pdp_message` MAY relay the PDP response `context` object.

The following is a non-normative example of a WWW-Authenticate header with the error code `failed_authorization` and associated payload:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="failed_authorization", error_description="The authorization level is not met"

{
  "decision": false,
  "context": {
    "error_msg": "The user must belongs to a project to access the resource",
    "details": {
      "expected_values": {
        "project" : ["phoenix", "eagle"]
      }
    }
  }
}
```

The following is a non-normative example of a compliant HTTP response relaying the response of a [D-OpenID-AuthZEN] compliant policy decision point (PDP) to the requesting client:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="failed_authorization", error_description="The authorization level is not met"

{
  "decision": false,
  "context": {
    "error_msg": "Access Policy failure",
    "details": {
      "pdp_message": {
        "id": "0",
        "reason_admin": {
          "en": "Request failed policy C076E82F"
        },
        "reason_user": {
          "en-403": "Insufficient privileges. Contact your administrator",
          "es-403": "Privilegios insuficientes. Póngase en contacto con su administrador"
        }
      }
    }
  }
}
```

The format of the `pdp_message` element is left implementation specific and non-normative; it should be negotiated and agreed upon by the client and resource server. Nevertheless usage of open standards is recommended, for example the usage of [D-OpenID-AuthZEN] payloads wherever possible.

### `insufficient_authorization` Error

If the error code `insufficient_authorization` is used, the description field of the `WWW-Authenticate` HTTP Header MUST be set to `The authorization level requires more details`.

Then the HTTP body payload MUST be formatted as an AuthZEN [D-OpenID-AuthZEN] response for a `decision` that MUST be set to `false` and MUST include as part of the response a `context` object. The `context` object SHOULD then include the following properties:

method:
: _REQUIRED_ - The value of this element MUST be an absolute URI that can be registered with IANA. It SHOULD support present, future or custom values. If IANA registered URIs are used, then their meaning and semantics should be respected and used as defined in the registry. This specification recognizes primarily the values `urn:ietf:params:oauth:grant-ext:rar` defined by the Rich Authorization Request [RFC9396] specification and `urn:ietf:params:oauth:grant-ext:par` defined through the Pushed Authorization Request [RFC9126] specification.

The structure MUST then include one of the following elements:

authorization_details:
: _OPTIONAL_ -  in JSON notation, an array of objects as defined in section 2 of Rich Authorization Request [RFC9396]

jar:
: _OPTIONAL_ - a Request Object as defined in section 2.2 of JWT-Secured Authorization Request [RFC9101]. In this case, the Request Object MUST contain the claim `iss` set to the Resource Provider and the claim `jwks_uri` to expose its signing keys in JSON Web Key format [RFC7517] through a dedicated dedicated URI.

The following is a non-normative example of a compliant response where the resource server requests a PAR authorization ceremony:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_authorization", error_description="The authorization level requires more details"


{
  "decision": false,
  "context": {
    "method": "urn:ietf:params:oauth:grant-ext:par",
    "jar": "eyJraWQiOiI3ZjczNWM5Ni0xMzg5LTQ1ODQtOWM5Zi01ZDg2MWJiYzU5YmIiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwczovL3JwLmV4YW1wbGUuY29tIiwiYXVkIjoiaHR0cHM6Ly9hcy5leGFtcGxlLmNvbSIsInJlc3BvbnNlX3R5cGUiOiJjb2RlIGlkX3Rva2VuIiwiY2xpZW50X2lkIjoiczZCaGRSa3F0MyIsInJlZGlyZWN0X3VyaSI6Imh0dHBzOi8vY2xpZW50LmV4YW1wbGUub3JnL2NiIiwiYXV0aG9yaXphdGlvbl9kZXRhaWxzIjpbeyJ0eXBlIjoiY3VzdG9tZXJfaW5mb3JtYXRpb24iLCJsb2NhdGlvbnMiOlsiaHR0cHM6Ly9leGFtcGxlLmNvbS9jdXN0b21lcnMiXSwiYWN0aW9ucyI6WyJyZWFkIiwid3JpdGUiXSwiZGF0YXR5cGVzIjpbImNvbnRhY3RzIiwicGhvdG9zIl19XSwic2NvcGUiOiJvcGVuaWQgcHJvZmlsZSAiLCJzdGF0ZSI6ImFmMGlmanNsZGtqIiwibm9uY2UiOiJuLTBTNl9XekEyTWoiLCJtYXhfYWdlIjo4NjQwMH0.267q0EAuug-CjjaXXzF25dBaVLuAVR8zFrVgsMhLVqo"
  }
}
```

The following is a non-normative example of a compliant response where the resource server requests pecific authorization details:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_authorization", error_description="The authorization level requires more details"

{
  "decision": false,
  "context": {
    "method": "urn:ietf:params:oauth:grant-ext:rar",
    "authorization_details":  [
        {
          "type": "customer_information",
          "locations": [
              "https://example.com/customers"
          ],
          "actions": [
              "read",
              "write"
          ],
          "datatypes": [
              "contacts",
              "photos"
          ]
        }
    ]
  }
}
```

# Setp-Up Auhtorization Request
A client receiving a step-up Authorization challenge as defined in section 4 of this specification from the resource server MUST take actions appropriate to each use case, as describe in the following sections.

### Case Of `failed_authorization` Error

This case is implementation-specific and the client MUST determine on its own the next course of action. The client MAY for example decide to provide feedback to the end user, or MAY additionally trigger further ceremonies between the user agent and/or authorization server, or MAY follow any other applicable RFC. It is assumed here that an out-of-band contract exists between client and resource server, which enables the client to eventually provide the resource server with the right access token to access the protected resource.

### Case Of `insufficient_authorization` Error

In this case, the client MUST follow the requirements fixed by the Resource Provider. The following situations CAN occur:

|     method                            | authorization_details |          jar          |  Expectation of the client                                                              |
| ------------------------------------- | --------------------- | --------------------- | --------------------------------------------------------------------------------------- |
| `urn:ietf:params:oauth:grant-ext:rar` | Set as specified      |         empty         | Starts an Authorization Request as defined by RAR with the authorization details        |
| `urn:ietf:params:oauth:grant-ext:rar` |         empty         | Set as specified      | Starts an Authorization Request as defined in RAR with the Request Object in JWT format |
| `urn:ietf:params:oauth:grant-ext:par` | Set as specified      |         empty         | Starts an Authorization Request as defined by PAR with the authorization details        |
| `urn:ietf:params:oauth:grant-ext:par` |         empty         | Set as specified      | Starts an Authorization Request as defined in PAR with the Request Object in JWT format |

This specification recognizes that other method URI might be defined in the future. The relevant specifications SHALL define the expected behavior for such new methods.

# Authorization Response

 An authorization server complying with this specification will react to the presence of the presented parameters and react as defined by the associated specification being, without limitations to, the response described in [RFC6749].

# Information conveyed via the Access Token

To evaluate whether an access token meets the protected resource's requirements, the resource server will enforce the conventional token-validation logic before analysing the content of the payload of the token as defined by [RFC7519], [RFC6749], and [RFC9068] as long as any specification that applies to IANA registered claims.

# Deployment  Considerations

This specification facilitates the communication of requirements from a resource server to a client, which, in turn, can enable a more granular and appropriate Authorization Request at the Authorization Server using either an OAuth 2.0 [RFC6749] defined grant flow, a Rich Authorization Request [RFC9396], a Push Authorization Request [RFC9126], or a JWT-Secured Authorization Request [RFC9101]. However, it is important to realize that the user experience achievable in every specific deployment is a function of the policies each resource server and authorization server pair establishes. Imposing constraints on those policies is out of scope for this specification. It is therefore perfectly possible for resource servers and authorization servers to impose requirements that are impossible for subjects or clients to comply with or that lead to an undesirable user-experience outcome.

# Security Considerations

This specification adds to previously defined OAuth mechanisms. Their respective security considerations apply:

- OAuth 2.0 [RFC6749],
- JWT access tokens [RFC9068],
- Bearer WWW-Authenticate [RFC6750],
- token introspection [RFC7662],
- authorization server metadata [RFC8414],
- Rich Authorization Request [RFC9396],
- Push Authorization Request [RFC9126],
- JWT-Secured Authorization Request [RFC9101] and
- AuthZEN [D-OpenID-AuthZEN]

## Scope

This specification does not attempt to define the mechanics by which access control is made by the resource provider and how the result of such access control evaluation should be translated into one of the challenges defined in Section 4.

This specification does not attempt to define the mechanics by which extended authorization requests are processed and vlidated by the authorization server.

## Validation Of Token

For this specification, the resource provider MUST examine the incoming access token and enforce the conventional token-validation logic - be it based on JWT validation, introspection, or any other method - before determining whether or not a challenge should be returned.

## Step-Up Authorization Challenge Payload

Following this document, response from the resource server to the client might unintentionally disclose information about the subject, the resource, the action to be performed, as long as context-specific data such as but not limited to authorization details that an attacker might use to gain knowledge about their target.

Implementers should use care in determining what to disclose in the challenge and in what circumstances.

# IANA Considerations

This document has no IANA actions.

--- back

# Use Cases

## LLM Agent accessing a service via an LLM Tool on behalf of a user

LLM agents, including those based on large language models (LLMs), are designed to manage user context, memory, and interaction state across multi-turn conversations. To perform complex tasks, these agents often integrate with external systems such as SaaS applications, internal services, or enterprise data sources. When accessing these systems, the agent operates on behalf of the end user, and its actions are constrained by the user’s identity, role, and permissions as defined by the enterprise. This ensures that all data access and operations are properly scoped and compliant with organizational access controls.

When using an LLM Tool, the LLM Agent is consumming an external API which has its own access control logic and policies on what conditions should be met for the access to the resources and the generation of the enriched information that the AI Agent can send return to the user who prompted it, with potential support of the LLM. Some access control conditions might require some authorization details as defined into, whithout being limited to, the examples of Rich Authorization Request [RFC9396].

A non normative example of such interaction would be at the functional level:

    +----------+
    |          |                  +--------------+
    |          |---(1) prompt --->|              |                                                                                      +--------------+
    |          |                  |              |-----------------------------------(2) LLM request ---------------------------------->|              |
    |          |                  |              |                                                                                      |              |
    |          |                  |              |<-----------------------------------(3) Tool usage -----------------------------------|              |
    |  User    |                  |              |                        +--------------+                                              |              |
    |          |                  |              |---(4) tool request --->|              |                                              |              |
    |          |                  |              |                        |              |                           +--------------+   |              |
    |          |                  |              |                        |              |---(5) service request --->|              |   |              |
    |          |                  |     LLM      |                        |   LLM Tool   |                           | Service APIs |   |      LLM     |
    |          |                  |    Agent     |                        |              |<--(6) service response ---|              |   |              |
    |          |                  |              |                        |              |                           +--------------+   |              |
    |          |                  |              |<--(7) tool response ---|              |                                              |              |
    |          |                  |              |                        +--------------+                                              |              |
    |          |                  |              |                                                                                      |              |
    |          |                  |              |-----------------------------------(8) LLM request ---------------------------------->|              |
    |          |                  |              |                                                                                      |              |
    |          |                  |              |<----------------------------------(9) LLM outcome -----------------------------------|              |
    |          |                  |              |                                                                                      +--------------+
    |          |<-(10) response --|              |
    |          |                  +--------------+
    +----------+

_Figure 2: Abstract AI Agent Use Case Flow_

### Preconditions

* The LLM Agent has a registered OAuth 2.0 Client (`com.example.llm-agent`) with the Enterprise IdP (`idp.example.com`)
* The LLM Tool has a registered OAuth 2.0 Client (`4960880b83dc9`) with the Enterprise IdP (`idp.example.com`)
* The LLM Tool has a registered OAuth 2.0 Client (`eb1e27d2df8b7`) with External Service IdP (`authorization-server.saas.net`)
* The External Service APIs is protected by the Trust Domain controlled by the External Service IdP (`authorization-server.saas.net`)
* User already authenticated at the Enterprise IdP (`idp.example.com`) and delegated its authorization to the LLM Agent
* The LLM Agent is in possession of an Identity Token, an Access Token, and a Refresh Token issued by the Enterprise IdP (`idp.example.com`)
* We assume that the Access Token is valid for the duration of this example and possess the appropriate scopes and claims to be authorized to call the LLM Tool

### LLM Agent receives a response fromn the LLM

LLM Agent receives a directive to use the LLM Tool with a specific payload and it calls the external LLM Tool provided by an Enterprise internal IT with a valid access token.

```http
POST /Pay
Host: tool.example.com
Authorization: Bearer ejyfewfewfwefwefewf.e.fwefwe.fw.e.fwef

to=DE02100100109307118603&
amount=123.50
```

### LLM Tool receives the request

LLM tool tries to call the service with the External Service APIs with the Access Token received using the JWT Bearer Authentication scheme (5) and is issued an authentication challenge.

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="new_authorization_needed", error_description="A new authorization request is needed"

{
  "decision": false,
  "context": {
    "error_msg": "The user must be authorized to initiate payment for Merchant A ",
    "details": {
      "method": "urn:ietf:params:oauth:grant-ext:rar",
      "authorization_details": [
        {
            "type": "payment_initiation",
            "actions": [
              "initiate",
              "status",
              "cancel"
            ],
            "locations": [
              "https://example.com/payments"
            ],
            "instructedAmount": {
              "currency": "EUR",
              "amount": "123.50"
            },
            "creditorName": "Merchant A",
            "creditorAccount": {
              "iban": "DE02100100109307118603"
            },
            "remittanceInformationUnstructured": "Ref Number Merchant"
        }
      ]
    }
  }
}
```
> Note: How agents discover available tools is out of scope of this specification

LLM Agent fetches the external tool resource's `OAuth 2.0 Protected Resource Metadata` per [RFC9728] to dynamically discover an authorization server that can issue an access token for the resource.

```http
GET /.well-known/oauth-protected-resource
Host: api.saas.net
Accept: application/json

HTTP/1.1 200 Ok
Content-Type: application/json

{
  "resource": "https://api.saas.net/",
  "authorization_servers": [ "https://authorization-server.saas.net" ],
  "bearer_methods_supported": [
    "header",
    "body"
  ],
  "scopes_supported": [
    "agent.tools.read",
    "agent.tools.write"
  ],
  "resource_documentation": "https://idp.saas.net/tools/resource_documentation.html"
}
```

LLM Agent discovers the Authorization Server configuration per {{RFC8414}}.

```http
GET /.well-known/oauth-authorization-server
Host: authorization-server.saas.net
Accept: application/json

HTTP/1.1 200 Ok
Content-Type: application/json

{
  "issuer": "https://authorization-server.saas.net",
  "authorization_endpoint": "https://authorization-server.saas.net/oauth2/authorize",
  "token_endpoint": "https://authorization-server.saas.net/oauth2/token",
  "jwks_uri": "https://authorization-server.saas.net/oauth2/keys",
  "registration_endpoint": "authorization-server.saas.net/oauth2/register",
  "scopes_supported": [
    "agent.read", "agent.write"
  ],
  "response_types_supported": [
    "code"
  ],
  "grant_types_supported": [
    "authorization_code", "refresh_token"
  ]
}
```

LLM Agent has learned all necessary endpoints and supported capabilites to obtain an access token for the external tool.

### LLM Tool obtains a set of token from Authorization Server protecting the API

The LLM tool redirects the LLM Agent for an authorization request:

```http
GET /oauth2/authorize?response_type=code
   &client_id=eb1e27d2df8b7
   &state=af0ifjsldkj
   &redirect_uri=https%3A%2F%2Ftool.example.com%2Fcb
   &code_challenge_method=S256
   &code_challenge=K2-ltc83acc4h0c9w6ESC_rEMTJ3bwc-uCHaoeK1t8U
   &authorization_details=%5B%7B%22type%22:%20%22payment_initiation
   %22,%22actions%22:%20%5B%22initiate%22,%22status%22,%22cancel%22
   %5D,%22locations%22:%20%5B%22https://example.com/payments%22%5D,
   %22instructedAmount%22:%20%7B%22currency%22:%20%22EUR%22,%22amount
   %22:%20%22123.50%22%7D,%22creditorName%22:%20%22Merchant%20A%22,
   %22creditorAccount%22:%20%7B%22iban%22:%20%22DE02100100109307118603
   %22%7D,%22remittanceInformationUnstructured%22:%20%22Ref%20Number
   %20Merchant%22%7D%5D
Host: authorization-server.saas.net
```

> We don't describe the wy the user is authenticated as it follows Rich Authorization Request [RFC9396]

The LLM Tool will receive an Authorization Code that it will be able to exchange for a set of JWTs issued by the Authorization Server protecting the API.

The LLM Tool can then make a new request to the External Service APIs (5). If it can meet the APIs Access Control requirement, the flow will follow with a response (6).

# Acknowledgments
{:numbered="false"}

The authors wants to acknowledge the support and work of the following indivisuals: Grese Hyseni (Raiffeisen Bank International, grese.hyseni@rbinternational.com), Henrik Kroll (Raiffeisen Bank International, henrik.kroll@rbinternational.com).

The authors wants also to recognize the trail blazers and thought leaders that created the ecosystem without which this draft proposal would not be able to solve customer pain points and secure usage of digital services, especially without being limited to: Vittorio Bertocci†, Brian Campbell (Ping Identity), Justin Richer (MongoDB), Aaron Parecki (Okta), Pieter Kasselman (SPRL), Mike Jones (Self-Issued Consulting, LLC).
