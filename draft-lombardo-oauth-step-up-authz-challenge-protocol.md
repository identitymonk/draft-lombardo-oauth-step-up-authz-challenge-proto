---
title: "OAuth 2.0 step-up authorization challenge protocol"
abbrev: "TODO - Abbreviation"
category: info

docname: draft-lombardo-oauth-step-up-authz-challenge-protocol-latest
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
  latest: "https://identitymonk.github.io/draft-lombardo-oauth-step-up-authz-challenge/draft-lombardo-oauth-step-up-authz-challenge.html"

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
  RFC6749: # OAuth 2.0 Framework
  I-D.lombardo-oauth-client-extension-claims: # OAuth2 Client extensions claims

informative:
  RFC9470: # OAuth 2.0 Step Up Authentication Challenge Protocol
  RFC9396: # Rich Authorization Request
  RFC9126: # Pushed Authorization Request
  RFC9101: # JWT-Secured Authorization Request
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



--- abstract

It is not uncommon for resource servers to require additional information like details of delegation authorization, or assurance proof of the delegation of authorization mechanism used according to the characteristics of a request. This document introduces a mechanism that resource servers can use to signal to a client that the  data and metadata  associated with
the access token of the current request does not meet its authorization requirements and, further, how to meet them. This document also codifies a taxonomy to guide the client into starting a new request towards the authorization server in order to get issued, if applicable, a new set of tokens matching the requirements.


--- middle

# Introduction

In simple API authorization scenarios, an authorization server will determine what claims to embed in the tokkens to issue on the basis of aspects such as the scopes requested, the resource, the identity of the client, and other characteristics known a provisioning time. Although that approach is viable in many situations, it falls short in several important circumstances. Consider, for instance, a FAPI 2.0 or SMART on FHIR regulated API requiring peculiar client authentication mechanism to be enforced or transaction specific details to be present in the token depending on whether the resource being accessed need to meet some rules estimated by the API itself, or a policy decision point it relies on, using a logic that is opaque to the authorization server.

This document extends the collection of error codes defined by [RFC6750] and by [RFC9470] with a new error codes, `insufficient_delegated_authorization` and `new_authorization_needed`, which can be used by resource servers to signal to clients that the authorization delegation represented by the access token presented with the request does not meet the authorization requirements of the resource server. This document also introduces associated payload definitions. The resource server can use these payloads to explicitly communicate to the client the required authorization details required.

The client can use that information to reach back to the authorization server with through a new authorization request that specifies the additional authorization details required to be described in the tokens to be issued. This document does not describe new methods to perform the new authorization request but will rely on OAuth 2.0 Rich Auhtorization Request [RFC9396], OAuth 2.0 Pushed Authorization Request [RFC9126], or OAuth 2.0 JWT-Secured Authorization Request [RFC9101].

Those extensions will make it possible to implement interoperable step up authorization with minimal work from resource servers, clients, and authorization servers.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This specification uses the terms "access token", "authorization server", "authorization endpoint", "authorization request", "client", "protected resource", and "resource server" defined by [RFC6749].

# Protocol Overview

The following is an end-to-end sequence of a typical step up authorization scenario implemented according to this specification. The scenario assumes that, before the sequence described below takes place, the client already obtained an access token for the protected resource.

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
2. The resource server determines that the circumstances in which the presented access token was obtained offer insufficient authorization details, wrong grant flow, or inadequate client authentication mechanism; hence, it denies the request and returns a challenge describing (using a combination of error code and payload details) what authorization details  requirements must be met for the resource server to allow a request.
3. The client directs the user agent to the authorization server with an authorization request that includes the authorization details indicated by the resource server in the previous step.
4. Whatever sequence required by the grant of choice plays out; this will include the necessary steps to authenticate the client in accordance with the playoad, to use the required grant flow type, or to validate that the authorization details are met. Then, the authorization server returns a new access token to the client. The new access token contains or references information about the elements required, including but not limited to what [I-D.lombardo-oauth-client-extension-claims] defines.
5. The client repeats the request from step 1, presenting the newly obtained access token.
6. The resource server finds that the authorization details, grant flow, or client authentication mechanism used during the acquisition of the new access token complies with its requirements and returns the representation of the requested protected resource.

The validation operations mentioned in steps 2 and 6 imply that the resource server has a way of evaluating, on top of the requirements of authentication as defined in [RFC9470], the authorization requirements that occurred during the process by which the access token was obtained. In the context of this document, the assessment by the resource server of the specific authorization mechanisms used to obtain a token for the requested resource is called an "authorization state".

This document will not describe how the resource server can perform this assessment of the authorization state whatever the access token is a JSON Web Token (JWT) [RFC9068] or is validated via introspection [RFC7662] and whatever the resource provider is performing this assessment natively or by offloading the assessment to a policy decision point as defined in [XACML] and NIST's ABAC [SP.800-162] Still, this document describes how the resource provider can signal to the client any insatisfying authorization  is expected to be obtained from the authorization server if applicable.

The terms "authorization state" and "step up" are metaphors in this specification. These metaphors do not suggest that there is an absolute hierarchy of authorization states expressed in interoperable fashion. The notion of a state emerges from the fact that the resource server may only want to accept certain authorization mechanisms. When presented with a token derived from particuliar authorization mechanisms (i.e., a given authorization state) that it does not want to accept (i.e., below the threshold it will accept), the resource server seeks to step up (i.e., renegotiate) from the current authorization state to one that it may accept. The "step up" metaphor is intended to convey a shift from the original authorization state to one that is acceptable to the resource server.

Although the case in which the new access token supersedes old tokens by virtue of a higher authorization state is common, in line with the connotation of the term "step up authorization", it is important to keep in mind that this might not necessarily hold true in the general case. For example, for a particular request, a resource server might require a higher authorization state and a shorter validity, resulting in a token suitable for one-off calls but leading to frequent prompts: hence, offering a suboptimal user experience if the token is reused for routine operations. In such a scenario, the client would be better served by keeping both the old tokens, which are associated with a lower authorization state, and the new one: selecting the appropriate token for each API call. This is not a new requirement for clients, as incremental consent and least-privilege principles will require similar heuristics for managing access tokens associated with different scopes and permission levels. This document does not recommend any specific token-caching strategy: that choice will be dependent on the characteristics of every particular scenario and remains application-dependent as in the core OAuth cases. Also recall that OAuth 2.0 [RFC6749] assumes access tokens are treated as opaque by clients. The token format might be unreadable to the client or might change at any time to become unreadable. So, during the course of any token-caching strategy, a client must not attempt to inspect the content of the access token to determine the associated authentication information or other details (see Section 6 of [RFC9068] for a more detailed discussion).

# Authorization Requirements Challenge

## HTTP Error Status Code

When a request compliant with this specification does not meet the authorization state requirements, the resource server responds using the `403` HTTP status code.

## Error Codes

This specification introduces two new error code value for the challenge of the Bearer authentication scheme's error parameter (from [RFC6750]) and other OAuth authentication schemes, such as those seen in [RFC9449], which use the same error parameter:

`insufficient_delegated_authorization`:
: The authorization mechansisms used for the issuance of the access token presented with the request does not meet the authorization state requirements of the protected resource. It is up to the client to decide what to perform next.

`new_authorization_needed`:
: The authorization mechansisms used for the issuance of the access token presented with the request does not meet the authorization state requirements of the protected resource. The client is guided on why new authorization mechanisms and details SHOULD be used through a new authorization request to the authorization server.

Note: the logic through which the resource server determines that the current request does not meet the authorization state requirements of the protected resource, and associated functionality (such as expressing, deploying and publishing such requirements), is out of scope for this document.

## WWW-Authenticate Parameters And Payload

### Case Of `insufficient_delegated_authorization`

### Case Of `new_authorization_needed`

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
