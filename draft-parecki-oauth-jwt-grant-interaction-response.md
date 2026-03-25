---
title: "JWT Authorization Grant Interaction Response"
abbrev: "JAG-IR"
category: info

docname: draft-parecki-oauth-jwt-grant-interaction-response-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - jwt authorization grant
 - interaction
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  github: "aaronpk/draft-parecki-oauth-jwt-grant-interaction-response"
  latest: "https://aaronpk.github.io/draft-parecki-oauth-jwt-grant-interaction-response/draft-parecki-oauth-jwt-grant-interaction-response.html"

author:
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com
 -
    fullname: Brian Campbell
    organization: Ping Identity
    email: bcampbell@pingidentity.com
 -
    fullname: Dapeng Liu
    organization: Alibaba Group
    email: max.ldp@alibaba-inc.com

normative:
  OAUTH-2.1: I-D.draft-ietf-oauth-v2-1
  RFC7523:

informative:
  RFC8628:
  I-D.draft-ietf-oauth-identity-assertion-authz-grant:

...

--- abstract

This document defines an extension to the JWT Authorization Grant {{RFC7523}}
that enables an authorization server to indicate that user interaction is
required in order to complete an authorization request.
Instead of returning an access token or an error, the authorization
server returns a URI that the client launches where the user can interact
with the authorization server, along with a polling interval. The client can
then poll for the access token or wait for a redirect before retrying
the original request.


--- middle

# Introduction

The JWT Authorization Grant [[RFC7523]] and Identity Assertion Grant
[[I-D.draft-ietf-oauth-identity-assertion-authz-grant]]
enable clients to obtain access tokens without direct user
approval at the authorization server. However, certain scenarios
may require explicit user consent, even if the initial authorization
can be obtained without user interaction:

- AI Agent Authorization: Autonomous agents acting on behalf of
  users need explicit consent for specific operations, not just
  identity verification.

- High-Risk Operations: Financial transactions, data deletion, or
  other sensitive operations may require step-up consent.

- Compliance Requirements: Regulatory frameworks (GDPR, etc.) may
  require explicit, verifiable consent for certain activities.

- Policy-Based Authorization: Fine-grained authorization policies
  (e.g., "allow purchases up to $50") require user understanding and approval.

Currently, if user interaction is needed, the authorization server
has no standardized way to communicate this to the client within the
JWT Authorization Grant flow. The client would typically receive an
error response and would need to fall back to a traditional redirect-based
OAuth flow, negating the benefits of using the JWT Authorization Grant.

This specification defines an interaction response that the
authorization server can return in place of an access token. The
interaction response contains a URI that the client opens (typically
in a browser) so the user can interact with the authorization server.
It also includes a polling interval, similar to the Device
Authorization Grant {{RFC8628}}, indicating how frequently the client
should re-request the token.

The client's token request MAY include a `redirect_uri` parameter.
If provided, the authorization server redirects the user's browser
to this URI after the interaction is complete. Unlike the
Authorization Code flow, no authorization code is included in the
redirect. The redirect serves only as a signal to the client that the
user interaction has completed. The client then retries its original
JWT Authorization Grant request to obtain the access token.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Token Request

The client makes a token request to the authorization server's token
endpoint as defined in {{RFC7523}}, with the addition of an OPTIONAL
`redirect_uri` parameter.

`redirect_uri`
: OPTIONAL. The URI to which the authorization server will redirect
  the user's browser after the interaction is complete. The redirect
  URI MUST be previously registered with the authorization server or
  otherwise validated per authorization server policy.

An example request:

~~~ http
POST /token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion=eyJhbGciOi...
&redirect_uri=https://client.example.org/callback
~~~



# Token Endpoint Responses {#token-endpoint-responses}

In addition to the error codes defined in {{Section 3.2.4 of OAUTH-2.1}},
the following error codes are specified for use with the JWT
Authorization Grant Interaction Response in token endpoint responses.

## Interaction Required Response {#interaction-response}

When the authorization server receives a valid JWT Authorization
Grant request but determines that user interaction is required before
an access token can be issued, it responds with an OAuth error
response as defined in {{Section 3.2.4 of OAUTH-2.1}} as defined below:

`error`
: REQUIRED. `interaction_required`. Indicates that user interaction
  is required, and the following additional parameters are defined
  in the response.

`interaction_uri`
: REQUIRED. The URI that the client MUST launch (typically in the
  user's browser) to allow the user to interact with the authorization
  server. The URI MUST use the "https" scheme.

`interval`
: OPTIONAL. The minimum number of seconds that the client SHOULD
  wait between polling requests to the token endpoint. If no value is
  provided, the default is 5 seconds.

`expires_in`
: OPTIONAL. The number of seconds after which the interaction URI
  and the associated authorization session will expire.

The response MUST include a `Content-Type` header field set to
`application/json`.

~~~ http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "interaction_required",
  "interaction_uri": "https://auth.example.com/interact/abc123",
  "interval": 5,
  "expires_in": 600
}
~~~

## Interaction Pending Response {#interaction-pending-response}

When user interaction is pending, and the client makes a subsequent
token endpoint request, the authorization server responds with an
OAuth error response as defined in {{Section 3.2.4 of OAUTH-2.1}}
with one of the values defined below:

`interaction_pending`
: The authorization request is still pending as the end user hasn't
  yet completed the user-interaction steps.  The
  client SHOULD repeat the access token request to the token endpoint.
  Before each new request,
  the client MUST wait at least the number of seconds specified by
  the `interval` parameter defined in {{interaction-response}}, or 5 seconds if none was provided,
  and respect any increase in the polling interval required by the "slow_down" error.

`slow_down`
: A variant of `authorization_pending`, the authorization request is
  still pending and polling should continue, but the interval MUST
  be increased by 5 seconds for this and all subsequent requests.

`access_denied`
: The authorization request was denied.


## Handling the Interaction Response

Upon receiving an interaction response as defined in
{{token-endpoint-responses}}, the client MUST:

1. Launch or redirect a browser to the `interaction_uri`.

2. Wait for the interaction to complete using one or both of the
   following mechanisms:

   * **Polling:** The client re-sends its original token request
     (including the same JWT assertion and parameters) to the token
     endpoint, waiting at least `interval` seconds between each
     request. The authorization server will continue to return the
     interaction response until the user interaction is complete,
     at which point it will return an access token response or an
     error response as defined in {{Section 3.2 of OAUTH-2.1}}.

   * **Redirect notification:** If the client included a
     `redirect_uri` in its original request, it MAY wait for the
     authorization server to redirect the user's browser to that URI.
     Upon receiving the redirect, the client SHOULD immediately retry
     its original token request. No authorization code or other
     parameters are included in the redirect; the redirect serves
     solely as a signal that the interaction is complete.



# Authorization Server Behavior {#as-behavior}

When the authorization server receives a JWT Authorization Grant
request and determines that user interaction is required, it MUST:

1. Generate a unique `interaction_uri` for the user interaction session.

2. Associate the interaction session with the original token request
   parameters (including the JWT assertion claims, requested scope,
   any `redirect_uri` provided by the client, and parameters from any extensions).

3. Return the interaction response as defined in {{interaction-response}}.

While the user interaction is pending, subsequent token requests from
the client with the same JWT assertion SHOULD return the interaction
response or an `interaction_pending` error.

After the user has completed the required interaction, the
authorization server MUST:

1. If a `redirect_uri` was provided, redirect the user's browser to
   that URI. The redirect MUST NOT include an authorization code or
   access token.

2. On the next token request from the client with the same JWT
   assertion, return an access token response as defined in
   {{Section 3.2.3 of OAUTH-2.1}} or an error response as defined in
   {{Section 3.2.4 of OAUTH-2.1}}.


# Complete Flow Diagram

The diagram below is a non-normative example of using this specification
in conjunction with the [[I-D.draft-ietf-oauth-identity-assertion-authz-grant]].

~~~
 +--------+     +--------+     +--------+     +----------+
 |  User  |     | Client |     |  IdP   |     |   AS     |
 |        |     | (Agent)|     |        |     |          |
 +--------+     +--------+     +--------+     +----------+
     |               |               |               |
 (1) | Authenticate  |               |               |
     |-------------->|               |               |
     |               |               |               |
 (2) |               | Token Exchange|               |
     |               | (get ID-JAG)  |               |
     |               |-------------->|               |
     |               |               |               |
 (3) |               | ID-JAG        |               |
     |               |<--------------|               |
     |               |               |               |
 (4) |               | JWT Grant     |               |
     |               | Request       |               |
     |               |------------------------------>|
     |               |               |               |
 (5) |               |               |    Validate   |
     |               |               |    & Decide:  |
     |               |               |    Interaction|
     |               |               |    Required   |
     |               |               |               |
 (6) |               | interaction_required          |
     |               | + interaction_uri             |
     |               |<------------------------------|
     |               |               |               |
 (7) |<--------------| Redirect to   |               |
     |               | interaction   |               |
     |               | page          |               |
     |               |               |               |
 (8) | Review &      |               |               |
     | Approve       |               |               |
     |---------------------------------------------->|
     |               |               |               |
 (9) |               |<------------------------------|
     |               |   Redirect    |               |
     |               | (via browser) |               |
     |               |               |               |
(10) |               | Retry JWT     |               |
     |               | Grant Request |               |
     |               |------------------------------>|
     |               |               |               |
(11) |               | Access Token  |               |
     |               |<------------------------------|
~~~

1. User authenticates to the Client through the IdP, typically via OpenID Connect
2. The Client exchanges the previously-obtained ID Token for an ID-JAG
3. The IdP validates the request against the configured policy and returns an ID-JAG
4. The Client presents the ID-JAG to the AS in a JWT Authorization Request
5. The AS validates the ID-JAG, and determines that further interaction is required
6. The AS returns the `interaction_required` response
7. The client redirects the browser to the specified page
8. The user visits the URL and confirms the request
9. The AS redirects to the Client's `redirect_uri`
10. The Client retries the JWT Authorization Request
11. The AS validates the request, sees the user has confirmed, and issues an access token


# Security Considerations

## Interaction URI Security

The `interaction_uri` MUST be an "https" URI. The URI SHOULD contain
sufficient entropy to prevent guessing or brute-force attacks. The
authorization server SHOULD bind the interaction session to the
client identity from the original token request.

## Redirect URI Validation

If the client provides a `redirect_uri`, the authorization server
MUST validate it against the client's registered redirect URIs,
consistent with {{Section 2.3.1 of OAUTH-2.1}}.



# IANA Considerations

TBD


--- back

# Appendix

## Use Cases

### AI Agent with Browser Access

An AI agent needs to perform operations on behalf of a user at a
third-party service. The agent can redirect the user's browser to
a consent page, then receive a callback when consent is complete.

This differs from the Device Authorization Grant (RFC 8628) in
that no polling is required - the redirect/callback model provides
lower latency.

### High-Risk Transaction Approval

A client presents a valid JWT assertion but requests authorization
for a high-value financial transaction. The AS determines that
step-up consent is required before issuing the token.

### Regulatory Compliance

A client requests access to sensitive data. Regulatory requirements
mandate that explicit, auditable consent must be obtained and
recorded before access is granted.

## Example Sequence




# Acknowledgments
{:numbered="false"}

TODO acknowledge.

