%%%
title = "Response as Push"
abbrev = "rap"
category = "info"
docname = "draft-dwaite-response-as-push-latest"
ipr = "none"
workgroup = "todo"
keyword = ["oauth", "openid", "connect"]

[seriesInfo]
name = "Internet-Draft"
value = "draft-dwaite-response-as-push"
status = "standard"

[venue]
group = "todo"
type = "Working Group"
github = "response-as-push/response-as-push"
latest = "https://response-as-push.github.io/"

stand_alone = "yes"
smart_quotes = "no"

[pi]
toc = "yes"
sortrefs = "yes"
symrefs = "yes"

[[author]]
initials = "D."
surname = "Waite"
fullname = "David Waite"
[author.address]
email = "david+rap@alkaline-solutions.com"

%%%

.# Abstract

TODO Abstract

{mainmatter}

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Pushed Authorization Response Mode (PARM) {#parm}

A common use-case for Self-Issued OPs is requesting and receiving Verifiable Presentations.  Both the authorization request and redirect response containing verifiable presentation data can grow quite large, such that the resulting URIs are longer than popular browsers can support.  The OAuth 2.0 Pushed Authorization Requests ([@!RFC9126]) is a well defined solution for this on the request flow, but doesn't include how a large response could be similarly handled.

To summarize the flow defined by [@!RFC9126]: when the request URI is too large, the Authorization Server (AS) can advertise a supporting endpoint in their metadata with two values: `pushed_authorization_request_endpoint` and `require_pushed_authorization_requests`.  The OAuth Client can then `POST` to this endpoint and receive back a short-lived unguessable URI which is subsequently used in the normal request flow as the `request_uri` parameter.

In order to support this same pattern as PAR, the roles and names need to be updated for the response flow by a Self-Issued OP.  Instead of the AS advertising and hosting the endpoint, it is the Relying Party that provides this.  Instead of the OAuth Client detecting and performing the `POST`, it is the SIOP implementation that acts as the HTTP client.

The definition of PARM that follows is OPTIONAL for implementers.  It borrows heavily from the PAR RFC, focusing on highlighting where the roles and terms diverge.

## PARM Endpoint {#parm_endpoint}

The pushed authorization response mode endpoint is an HTTP API exactly as specified in PAR.

The Relying Party supporting PARM SHOULD include the URL of their endpoint in their Relying Party metadata document using the `pushed_authorization_response_endpoint` parameter as defined in (#parm_rp_metadata).

The endpoint accepts the authorization response parameters defined in [@!RFC6749] for the `redirect_uri` as well as all applicable extensions defined for the the response.

### Response {#parm_response}

SIOP sends the parameters that comprise the authorization response directly to the PARM endpoint. A typical response parameter set might include: `id_token`, `token_type`, `access_token`, and `state`. However, the pushed authorization response can be composed of any of the parameters applicable for use with the `redirect_uri` including those defined in [@!RFC6749] as well as all applicable extensions. The `response_uri` authorization response parameter defined here is one exception, which MUST NOT be provided.

The SIOP implementation constructs the message body of an HTTP `POST` request with `x-www-form-urlencoded` formatted parameters using a character encoding of UTF-8 as described in Appendix B of [@!RFC6749].

The Relying Party MUST process the `POST` parameters identically to how it would process them if received via the `redirect_uri`.

### Successful Response {#parm_success}

The successful processing of the response parameters is handled nearly identically to a PAR endpoint.  The server MUST generate a request URI and provide it in the response with a `201` HTTP status code. The following parameters are included as top-level members in the message body of the HTTP response using the `application/json` media type as defined by [@!RFC8259].

* `response_uri` : The response URI corresponding to the authorization response posted. This URI is a single-use reference to the respective response data in the subsequent redirect request. The way the redirect processing obtains the authorization response data is at the discretion of the Relying Party and out of scope of this specification. There is no need to make the authorization response data available to other parties via this URI.
* `expires_in` : A JSON number that represents the lifetime of the response URI in seconds as a positive integer. The response URI lifetime is at the discretion of the Relying Party but will typically be relatively short (e.g., between 5 and 600 seconds).

The format of the `response_uri` value is at the discretion of the Relying Party but it MUST contain some part generated using a cryptographically strong pseudorandom algorithm such that it is computationally infeasible to predict or guess a valid value (see Section 10.10 of [@!RFC6749] for specifics). The Relying Party MAY construct the `response_uri` value using the form `urn:ietf:params:oauth:response_uri:<reference-value>` with `<reference-value>` as the random part of the URI that references the respective authorization response data.

The `response_uri` value MUST be bound to the response data posted to the endpoint.

The following is an example of such a response:

```
 HTTP/1.1 201 Created
 Content-Type: application/json
 Cache-Control: no-cache, no-store

 {
  "response_uri": 
    "urn:ietf:params:oauth:response_uri:6esc_11ACC5bwc014ltc14eY22c",
  "expires_in": 60
 }
```

### Error Response {#parm_error}

The same error handling and response codes as defined in PAR apply equally to PARM.

## Response URI {#response_uri}

Similarly to the `request_uri` as defined in [@!OpenID.Core], PARM introduces a new `response_uri` parameter that is only valid when used in combination with the `redirect_uri` during a response flow.

`response_uri`
: URI parameter to be added to the `redirect_uri` as defined in {parm_success}. 

## Relying Party Metadata {#parm_rp_metadata}

The following Relying Party metadata parameters are introduced to signal the server's capability and policy with respect to PARM.

`pushed_authorization_response_endpoint`
: The URL of the pushed authorization response endpoint at which an SIOP implementation can `POST` an authorization response to exchange for a `response_uri` value usable in the `redirect_uri`.  The presence of this value is sufficient for SIOP to determine that it may use the PARM flow.

`require_pushed_authorization_responses`
: Boolean parameter indicating whether the Relying Party accepts authorization response data ONLY via PARM. If omitted, the default value is `false`. 

## Cross Device {#parm_cross_device}

Using PARM is also an ideal solution when Self-Issued OPs are used in cross device flows, where the request may be initiated via a scanned QR code, NFC tag, or BLE beacon.  In these flows there may be limited or no ability to return a large response over the same channel.  A Relying Party that supports these flows should consider using PARM to process the responses coming from another device where the `redirect_uri` would not return on the same channel.

There are two important considerations when implementing cross device support:
1. Since the `redirect_uri` will continue the flow in a _different_ browser on the _authenticating_ device and not where the request was initiated, any statefulness needs to be tracked server-side by the relying party and/or embedded into the `state` request and response parameter.
2. The response is not bound to the requesting channel on the original device, leaving it extremely vulnerable to trivial phishing attacks.  When using a cross device flow for authentication the requesting device MUST be managed such that the user cannot have navigated it to potential phishing sites.
3. If the Relying Party is located on an internal network, it may be required to host the PARM endpoint outside of that network and accessible to authenticating devices which may not be on the same network.

## Mobile User Experience {#parm_mobile}

When Self-Issued OP implementations are mobile apps they will be completing the request by asking the OS to load the final `redirect_uri` with the correct parameters.

That redirect URI may already be registered to the OS by the Relying Party's native app, allowing it to load and continue the native experience.  Alternatively, if the app is not installed (such as in a cross-device flow) it will allow the Relying Party to provide a web experience for further instructions or a path to install their app.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


{backmatter}

<reference anchor="OpenID.Core" target="http://openid.net/specs/openid-connect-core-1_0.html"> <front> <title>OpenID Connect Core 1.0 incorporating errata set 1</title> <author initials="N." surname="Sakimura" fullname="Nat Sakimura"> <organization>NRI</organization> </author> <author initials="J." surname="Bradley" fullname="John Bradley"> <organization>Ping Identity</organization> </author> <author initials="M." surname="Jones" fullname="Michael B. Jones"> <organization>Microsoft</organization> </author> <author initials="B." surname="de Medeiros" fullname="Breno de Medeiros"> <organization>Google</organization> </author> <author initials="C." surname="Mortimore" fullname="Chuck Mortimore"> <organization>Salesforce</organization> </author> <date day="8" month="Nov" year="2014"/> </front> </reference>

# Acknowledgments

TODO acknowledge.
