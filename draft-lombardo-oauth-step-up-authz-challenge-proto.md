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
  latest: "https://identitymonk.github.io/draft-lombardo-oauth-step-up-au   thz-challenge/draft-lombardo-oauth-step-up-authz-challenge-proto.html"

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

informative:
  RFC7662: # Introspection
  RFC9470: # OAuth 2.0 Step Up Authentication Challenge Protocol
  RFC9728: # OAuth 2.0 Protected Resource Metadata
  RFC8414: # OAuth2 Authorization Server Metadata
  RFC9449: # DPoP
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
  MCP:
    title: Model Context Protocol
    target: https://modelcontextprotocol.io/specification/2025-03-26/basic


--- abstract

It is not uncommon for resource servers to require additional information like details of delegation authorization, or assurance proof of the delegation of authorization mechanism used according to the characteristics of a request. This document introduces a mechanism that resource servers can use to signal to a client that the  data and metadata  associated with
the access token of the current request does not meet its authorization requirements and, further, how to meet them. This document also codifies a taxonomy to guide the client into starting a new request towards the authorization server in order to get issued, if applicable, a new set of tokens matching the requirements.


--- middle

# Introduction

In simple authorization scenarios, an authorization server will determine what claims to embed in the tokens to issue on the basis of aspects such as the scopes requested, the resource, the identity of the client, and other characteristics known a provisioning time. Although that approach is viable in many situations, it falls short in several important circumstances. Consider, for instance, a FAPI 2.0 or SMART on FHIR regulated API requiring peculiar client authentication mechanism to be enforced or transaction specific details to be present in the token. These requirements may depend upon resource access rules or policies implemented at a policy decision point it relies on, using a logic that is opaque to the authorization server.

This document extends the collection of error codes defined by [RFC6750] and by [RFC9470] with a new error codes, `insufficient_delegated_authorization` and `requested_authorization`, which can be used by resource servers to signal to clients that the authorization delegation represented by the access token presented with the request does not meet the authorization requirements of the resource server. This document also introduces associated payload definitions. The resource server can use these payloads to explicitly communicate to the client its authorization requirements.

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

The validation operations mentioned in steps 2 and 6 imply that the resource server has a way of evaluating the authorization requirements that occurred during the ceremonies that led to the issuance of the access token. In the context of this document, the assessment by the resource server of the specific authorization mechanisms used to obtain a token for the requested resource is called an "authorization state".

This document does not describe how the resource server performs this assessment of the authorization state, whether the access token is a JSON Web Token (JWT) [RFC9068] or is validated via introspection [RFC7662] or whether the resource provider is performing this assessment natively or by offloading the assessment to a policy decision point as defined in [D-OpenID-AuthZEN], [XACML] or NIST's ABAC [SP.800-162]. This document rather describes how the resource provider tells the client what type of authorization it needs to get from the authorization server for a request.

The terms "authorization state" and "step up" are metaphors in this specification. These metaphors do not suggest that there is an absolute hierarchy of authorization states expressed in interoperable fashion. The notion of a state emerges from the fact that the resource server may only want to accept certain authorization mechanisms. When presented with a token derived from particuliar authorization mechanisms (i.e., a given authorization state) that it does not want to accept (i.e., below the threshold it will accept), the resource server seeks to step up (i.e., renegotiate) from the current authorization state to one that it may accept. The "step up" metaphor is intended to convey a shift from the original authorization state to one that is acceptable to the resource server.

Although the case in which the new access token supersedes old tokens by virtue of a higher authorization state is common, in line with the connotation of the term "step up authorization", it is important to keep in mind that this might not necessarily hold true in the general case. For example, for a particular request, a resource server might require a higher authorization state and a shorter validity, resulting in a token suitable for one-off calls but leading to frequent prompts: hence, offering a suboptimal user experience if the token is reused for routine operations. In such a scenario, the client would be better served by keeping both the old tokens, which are associated with a lower authorization state, and the new one: selecting the appropriate token for each API call. This is not a new requirement for clients, as incremental consent and least-privilege principles will require similar algorithms for managing access tokens associated with different scopes and permission levels. This document does not recommend any specific token-caching strategy: that choice will be dependent on the characteristics of every particular scenario and remains application-dependent as in the core OAuth cases. Furthermore, OAuth 2.0 [RFC6749] assumes access tokens are treated as opaque by clients. The token format might thus be unreadable to the client or might change at any time to become unreadable. So, during the course of any token-caching strategy, a client must not attempt to inspect the content of the access token to determine the associated authentication information or other details (see Section 6 of [RFC9068] for a more detailed discussion).

# Authorization Requirements Challenge

## HTTP Error Status Code

The resource server responds with a `403` HTTP status code using the Bearer authentication scheme's error parameter (from [RFC6750]) when a request compliant with this specification does not meet its authorization state requirements.

## Error Codes

This specification introduces two new error code values for this challenge and other OAuth authentication schemes, as defined in OAuth 2.0 Demonstrating Proof of Possession (DPoP)[RFC9449] and in OAuth 2.0 Step Up Authentication Challenge Protocol[RFC9470]:

`insufficient_delegated_authorization`:
: The authorization mechanisms used for the issuance of the access token presented with the request do not meet the authorization state requirements of the protected resource. It is up to the client to decide what to perform next; some further implementation-specific details are provided in the payload of the response (see below).

`requested_authorization`:
: The authorization mechanisms used for the issuance of the access token presented with the request do not meet the authorization state requirements of the protected resource. The client is provided a detailed standard response that describes exactly which new authorization mechanisms and details SHOULD be used in order to gain access to the requested resource. The client is expected to then initiate new ceremonies with the authorization server, that comply with the stated resource requirements.

Note: the logic through which the resource server determines that the current request does not meet the authorization state requirements of the protected resource, and associated functionality (such as expressing, deploying and publishing such requirements), is out of scope for this document.

## WWW-Authenticate Header Error Codes And Associated Payloads

Each error code matches different types of HTTP response payloads, which guide the client into initiating the next Authorization Request.

### Response Payloads
This document specifies that all compliant HTTP responses must contain a payload, and that the contents of this payload depends on the error code issued by the resource server.

The payload is expected to be a JSON object, the details of which are provided in the following sections.

### `insufficient_delegated_authorization` Error

If the error code `insufficient_delegated_authorization` is used, then the HTTP body payload MUST be formatted as an AuthZEN [D-OpenID-AuthZEN] response for a `decision` that is set to `false` and MUST include as part of the response a `context` object. The `context` object should then include the following properties:

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

The following is a non-normative example of a WWW-Authenticate header with the error code `insufficient_delegated_authorization` and associated payload:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_user_authorization", error_description="A different authorization level is required"

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
WWW-Authenticate: Bearer error="insufficient_user_authorization", error_description="A different authorization level is required"

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

### `requested_authorization` Error

If the error code `requested_authorization` is used, the HTTP payload MUST be a valid [AuthZEN] compliant JSON [RFC8259] structure wich will include:

method:
: _REQUIRED_ - The value of this element MUST be an absolute URI that can be registered with IANA. It SHOULD support present, future or custom values. If IANA registered URIs are used, then their meaning and semantics should be respected and used as defined in the registry. This specification recognizes primarily the values `urn:ietf:params:oauth:grant-ext:rar` defined by the Rich Authorization Request [RFC9396] specification and `urn:ietf:params:oauth:grant-ext:par` defined through the Pushed Authorization Request [RFC9126] specification.

The structure MUST then include one of the following elements:

authorization_details:
: _OPTIONAL_ -  in JSON notation, an array of objects as defined in section 2 of Rich Authorization Request [RFC9396]

jar:
: _OPTIONAL_ - a JSON Web Token (JWT) [RFC7519], which JWT Claims Set holds the JSON-encoded OAuth 2.0 authorization request parameters as defined in section 2.1 of JWT-Secured Authorization Request [RFC9101]

reference:
: _OPTIONAL_ - an absolute URI that references the set of parameters comprising an OAuth 2.0 authorization request in a form of a Request Object as defined in section 2.2 of JWT-Secured Authorization Request [RFC9101]

The following is a non-normative example of a compliant response where the resource server requests a PAR authorization ceremony:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="requested_authorization", error_description="Requires PAR."

{
  "method": "urn:ietf:params:oauth:grant-ext:par",
  "reference": "https://tfp.example.org/request.jwt/GkurKxf5T0Y-mnPFCHqWOMiZi4VS138cQO_V7PZHAdM"
}
```

The following is a non-normative example of a compliant response where the resource server requests pecific authorization details:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="requested_authorization", error_description="Requires RAR."

{
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
```

# Setp-Up Auhtorization Request
A client receiving a step-up Authorization challenge as defined in section 4 of this specification from the resource server MUST take actions appropriate to each use case, as describe in the following sections.

### Case Of `insufficient_delegated_authorization` Error

This case is implementation-specific and the client MUST determine on its own the next course of action. The client MAY for example decide to provide feedback to the end user, or MAY additionally trigger further ceremonies between the user agent and/or authorization server, or MAY follow any other applicable RFC. It is assumed here that an out-of-band contract exists between client and resource server, which enables the client to eventually provide the resource server with the right access token to access the protected resource.

### Case Of `requested_authorization` Error

In this case, the client MUST follow the requirements fixed by the Resource Provider. The following situations CAN occur:

|     method                            | authorization_details |          jar          |       reference       |  Expectation of the client                                                                                                                                           |
| ------------------------------------- | --------------------- | --------------------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `urn:ietf:params:oauth:grant-ext:rar` | Set as specified      |         empty         |         empty         | Starts an Authorization Request as defined by RAR with the authorization details                                                                                     |
| `urn:ietf:params:oauth:grant-ext:rar` |         empty         | Set as specified      |         empty         | Starts an Authorization Request as defined in RAR with the Request Object in JWT format                                                                              |
| `urn:ietf:params:oauth:grant-ext:rar` |         empty         |         empty         | Set as specified      | Starts an Authorization Request as defined in RAR with the Request Object URI (it would mean the Resource Provider will have send the Request Object to AS directly) |
| `urn:ietf:params:oauth:grant-ext:par` | Set as specified      |         empty         |         empty         | Starts an Authorization Request as defined by PAR with the authorization details                                                                                     |
| `urn:ietf:params:oauth:grant-ext:par` |         empty         | Set as specified      |         empty         | Starts an Authorization Request as defined in PAR with the Request Object in JWT format                                                                              |
| `urn:ietf:params:oauth:grant-ext:par` |         empty         |         empty         | Set as specified      | Starts an Authorization Request as defined in PAR with the Request Object URI (it would mean the Resource Provider will have send the Request Object to AS directly) |

This specification recognizes that other method URI might be defined in the future. The relevant specifications SHALL define the expected behavior for such new methods.

# Authorization Response

 An authorization server complying with this specification will react to the presence of the presented parameters and react as defined by the associated specification being, without limitations to, the response described in [RFC6749].

# Information conveyed via the Access Token

To evaluate whether an access token meets the protected resource's requirements, the resource server will enforce the conventional token-validation logic before analysing the content of the payload of the token as defined by [RFC7519], [RFC6749], and [RFC9068] as long as any specification that applies to IANA registered claims.

# Authorization Server Metadata

TODO

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

This specification does not attempt to define the mechanics by which access control is made by the resource provider and how the result of such access control evaluation should be translated into one of the challenges defined in Section 4. Still, such payload might unintentionally disclose information about the subject, the resource, the action to be performed, as long as context-specific data such as but not limited to authorization details  that an attacker might use to gain knowledge about their target. Implementers should use care in determining what to disclose in the challenge and in what circumstances.

For this specification, the resource provider MUST examine the incoming access token and enforce the conventional token-validation logic - be it based on JWT validation, introspection, or any other method - before determining whether or not a challenge should be returned.

As this specification provides a mechanism for the resource server to trigger user interaction, it's important for the authorization server and clients to consider that a malicious resource server might abuse that feature.

# IANA Considerations

This document has no IANA actions.

--- back

# Use Cases

## LLM Agent accessing a service via an LLM Tool on behalf of a user

LLM agents, including those based on large language models (LLMs), are designed to manage user context, memory, and interaction state across multi-turn conversations. To perform complex tasks, these agents often integrate with external systems such as SaaS applications, internal services, or enterprise data sources. When accessing these systems, the agent MAY operate on behalf of an end user; its actions are then constrained by the user’s identity, role, and permissions as defined by the enterprise. This ensures that all data access and operations are properly scoped and compliant with organizational access controls.

In the case of autonomous agents or agents triggered by system or environmental events (i.e., in the cases where agents are not responding to human prompts), their actions should then by constrained by their own identity and entitlements. It is in that case assumed that the agent itself is capable of submitting to standard authorization ceremonies resulting in them holding access tokens compliant to the various OAuth [RFC6749] and [MCP] specifications.

In the present use-case though, a user prompts an agent for some data using natural language. In order to comply, the AI agent determines that it needs to fetch some LLM-grounding data through an external tool exposed by an Model Context protocol [MCP] compliant server. The actual underlying external API has its own access control logic and resource access policies. This non-normative example assumes that the external API responds using the Rich Authorization Request [RAR - RFC9396] method.

The following non normative diagram (Figure 2) depicts an example of the functional interaction between a LLM-powered agent, acting as an MCP client, the MCP Server exposing a Tool of interest to the Agent and the underlying API, the protected resource:


    +-------------------+
    |                   |                       +--------------------+                        +-----------------+
    |                   |        1) Prompt      |                    |      2) /authorize     |                 |
    |                   |---------------------->|                    |----------------------->|                 |
    |                   |                       |                    |  11) /token + MCP-CODE |                 |
    |                   |                       |     MCP Client     |----------------------->|                 |
    |                   |                       |   (LLM-enabled)    |       12) TOKEN        |                 |
    |                   |10) redirect + MCP-CODE|                    |<-----------------------|                 |
    |                   |---------------------->|                    | 13) Call Tool +  TOKEN |                 |
    |   User Agent      |                       |                    |----------------------->|                 |                                 +-------------+
    |                   |                       |                    |                        |                 |                                 |             |
    |                   |                       |                    |   17) HTTP 403         |                 |                                 |             |
    |                   |                       |                    |   WWW-AUTHENTICATE     |                 |                                 |             |
    |                   |                       |                    |<-----------------------|                 |                                 | Backend API |
    |                   |                       |                    |   18) /authorize       |                 |    14) API Call + TOKEN         | (Resource)  |
    |                   |                       |                    |----------------------->|    MCP Server   |-------------------------------->|             |---> 15) (AuthZEN)
    |                   |                       +--------------------+                        |                 |                                 |             |<---
    |                   |                                                                     |  (Exposed TOOL) |                                 |             |
    |                   |                   3) Redirect to External AS                        |                 |       16) HTTP 403              |             |
    |                   |<--------------------------------------------------------------------|                 |<--------------------------------|             |
    |                   |                      6) /token + AS-CODE                            |                 | HTTP/1.1 403 Forbidden          |             |
    |                   |-------------------------------------------------------------------->|                 | WWW-Authenticate:               |             |
    |                   |                9) redirect to MCP Client + MCP-CODE                 |                 | Bearer error="..",              |             |
    |                   |<--------------------------------------------------------------------|                 | error_description="Requires RAR.|             |
    |                   |                                                                     |                 |                                 |             |
    |                   |              19) Redirect to External AS - back to 4)               |                 |                                 +-------------+
    |                   |<------------------------------------------------------------------- |                 |
    |                   |                                                                     |                 |
    |                   |                        +---------------------+                      |                 |
    |                   |    4) /authorize       |                     | 7) /token + AS-CODE  |                 |
    |                   |----------------------> |    External AS      <----------------------|                 |
    |                   |                        |      (IdP)          |                      |                 |
    |                   | 5) redirect + AS-CODE  |                     |      8) TOKEN        |                 |
    |                   |<---------------------- |                     ---------------------->|                 |
    |                   |                        |                     |                      |                 |
    +-------------------+                        +---------------------+                      +-----------------+

_Figure 2: Abstract AI Agent MCP Use Case Flow with External AS_

As per section 2.10 Third-Party Authorization Flow of the [MCP] specification, the MCP server acts as an authorization server, but proxies an external AS. In our use-case,

### Preconditions

* The LLM Agent (MCP Client) has registered as an OAuth 2.0 Client (`com.example.llm-agent`) with the MCP Server(`mcp.example.com`)
* The LLM Tool (MCP Server) has registered as an OAuth 2.0 Client (`4960880b83dc9`) with the external Enterprise IdP (`idp.example.com`)
* The External Service API is protected by the Trust Domain controlled by the External Service IdP (`idp.example.com`)
* We assume that the Access Token is valid for the duration of this example and possess the appropriate scopes and claims to be authorized to call the LLM Tool and underlying API.
* The backend API and the MCP Server both use the same external AS, they can use the same access token which carries both it in `audience` claim.

### Flow details
The following flow is based on the [MCP] authorization flow described in its section 2.10, and augments it with a Step-Up Authorization request as described in this document:

1) The user initiates a prompt request on the MCP Client from the User Agent.
2) The user is not yet authenticated, the MCP Client therefore redirects the User agent to the MCP Server's `/authorize` endpoint.
3) The MCP Server relays the `authorize` request to the external AS, and redirects the user Agent there with an `authorization_code` grant flow.
4) The User authenticates with the external AS and authorizes the MCP Client.
5) The AS returns the `authorization_code` and redirects the User Agent to the MCP Server's callback.
6) The User agent requests a token with the Code provided by the external AS.
7) The MCP server's `/token` endpoint relays the token request to the external AS's `/token` endpoint.
8) The MCP Server gets the `access_token` in response. It caches the `access_token` and generates a new MCP-CODE, different from the initial code provided by the external AS.
9) The MCP Server redirects the user-agent to the MCP Client with the MCP-CODE.
10) The User Agent presents the MCP-CODE to the MCP Client.
11) The MCP Client requests the `access_token` from the MCP Server's `/token` endpoint.
12) The MCP Server responds with the external AS's `access_token`, which it cached in step 8.
13) Once ready, the MCP Client can now invoke the right MCP Server Tool, passing the `access_token` along with the request.
14) The MCP Server makes a backend call to the underlying API. The API validates the `access_token` with the same external AS.
15) The API resource _may_ use an external Policy Decision Point via [AuthZen] to evaluate the request.
16)  The API resource determines that it requires an elevated Authorization State, and expresses the requirement through a RAR payload response.
17)  The MCP Server relays the API Resource's response back to the MCP CLient.
18)  The MCP Client initiates a new `authorize` PAR request with the MCP Server's `/authorize` endpoint.
19)  The MCP Server relays the request to the external AS by redirecting the User Agent there. The process starts again from step 4, this time following a RAR flow.

### MCP Client receives a response from the LLM

Step 13 details: MCP Client receives a directive to use the MCP Tool with a specific payload and it calls the external MCP Server Tool with a valid access token.

```http
POST /Pay
Host: tool.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.KMUFsIDTnFmyG3nMiGM6H9FNFUROf3wh7SmqJp-QV30

to=DE02100100109307118603&amount=123.50
```

### LLM Tool receives the request

Step 15 details: MCP Server tool tries to call the backend Service APIs with the Access Token received using the JWT Bearer Authentication scheme and is issues an authentication challenge.

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="requested_authorization", error_description="A new authorization request is needed"

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

> Note: The methods used by MCP Clients and Agents to discover the right Tool(s) to use is out of scope of this specification

The LLM Agent has learned all necessary endpoints and supported capabilites to obtain an access token for the external tool.

### LLM Tool obtains the access_token from the Authorization Server protecting the API

Step 7 details:

```http
GET /oauth2/authorize?response_type=code&client_id=eb1e27d2df8b7&state=af0ifjsldkj&redirect_uri=https%3A%2F%2Ftool.example.com%2Fcb&code_challenge_method=S256&code_challenge=K2-ltc83acc4h0c9w6ESC_rEMTJ3bwc-uCHaoeK1t8U&authorization_details=%5B%7B%22type%22:%20%22payment_initiation%22,%22actions%22:%20%5B%22initiate%22,%22status%22,%22cancel%22%5D,%22locations%22:%20%5B%22https://example.com/payments%22%5D,%22instructedAmount%22:%20%7B%22currency%22:%20%22EUR%22,%22amount%22:%20%22123.50%22%7D,%22creditorName%22:%20%22Merchant%20A%22,%22creditorAccount%22:%20%7B%22iban%22:%20%22DE02100100109307118603%22%7D,%22remittanceInformationUnstructured%22:%20%22Ref%20Number%20Merchant%22%7D%5D
Host: idp.example.com
```

> The User's authentication follows the flows described in the Rich Authorization Request [RFC9396] specification.

The MCP Server Tool will receive an Authorization Code that it will be able to exchange for a set of JWTs issued by the Authorization Server protecting the API.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
