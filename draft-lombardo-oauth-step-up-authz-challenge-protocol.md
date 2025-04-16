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
    fullname: Jean-François "Jeff" Lombardo
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
  RFC9396: # Rich Authorization Request
  RFC9126: # Pushed Authorization Request
  RFC9101: # JWT-Secured Authorization Request



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

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
