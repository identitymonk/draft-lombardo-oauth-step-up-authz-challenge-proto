---
title: "OAuth2 Step-Up Authorization Challenge"
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
  github: "identitymonk/draft-ietf-oauth-step-up-authz-challenge"
  latest: "https://identitymonk.github.io/draft-ietf-oauth-step-up-authz-challenge/draft-ietf-oauth-step-up-authz-challenge-protocol.html"

author:
 -
    fullname: Jeff Lombardo
    organization: AWS
    email: jeff@authnopuz.xyz
 -
    fullname: Alex Babeanu
    organization: IndieKite
    email: alex.babeanu@indykite.com

normative:
  RFC6749: # OAuth2
  RFC7519: # JSON Web Token
  RFC9470: # Step Up Authentication Challenge

informative:


--- abstract

It is not uncommon for resource servers to require more details information the characteristics of a request. This document introduces a mechanism that resource servers can use to signal to a client that the details about the Subject, Action, Resource, and Context do not meet its authorization requirements and, further, how to meet them. This document also codifies a mechanism for guiding a client into shaping a rich or pushed authorization request towards the authorization server in order to acquire, if authorized, a new set of tokens that could match the expectations of the resource provider.

--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
