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

The JWT Authorization Grant {{RFC7523}} allows a client to present a
JWT assertion to an authorization server's token endpoint in exchange
for an access token. In some scenarios, however, the authorization
server cannot immediately issue an access token because user
interaction is required -- for example, to obtain consent, collect
additional information, or satisfy other policy requirements.

Currently, if user interaction is needed, the authorization server
has no standardized way to communicate this to the client within the
JWT Authorization Grant flow. The client would typically receive an
error response and would need to fall back to a different OAuth flow
entirely.

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


# Interaction Response {#interaction-response}

When the authorization server receives a valid JWT Authorization
Grant request but determines that user interaction is required before
an access token can be issued, it responds with an HTTP 200 response
containing a JSON object with the following parameters:

interaction_uri
: REQUIRED. The URI that the client MUST launch (typically in the
  user's browser) to allow the user to interact with the authorization
  server. The URI MUST use the "https" scheme.

interval
: OPTIONAL. The minimum number of seconds that the client SHOULD
  wait between polling requests to the token endpoint. If no value is
  provided, the default is 5 seconds.

expires_in
: OPTIONAL. The number of seconds after which the interaction URI
  and the associated authorization session will expire.

The response MUST include a `Content-Type` header field set to
`application/json`.

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "interaction_uri": "https://auth.example.com/interact/abc123",
  "interval": 5,
  "expires_in": 600
}
~~~

## Interaction Pending Response {#interaction-pending-response}

In addition to the error codes defined in {{Section 3.2.3 of OAUTH-2.1}},
the following error codes are specified for use with the JWT
Authorization Grant Interaction Response in token endpoint responses:

interaction_pending
: The authorization request is still pending as the end user hasn't
  yet completed the user-interaction steps.  The
  client SHOULD repeat the access token request to the token endpoint.
  Before each new request,
  the client MUST wait at least the number of seconds specified by
  the `interval` parameter defined in {{interaction-response}}, or 5 seconds if none was provided,
  and respect any increase in the polling interval required by the "slow_down" error.

slow_down
: A variant of `authorization_pending`, the authorization request is
  still pending and polling should continue, but the interval MUST
  be increased by 5 seconds for this and all subsequent requests.

access_denied
: The authorization request was denied.



# Client Behavior {#client-behavior}

## Token Request

The client makes a token request to the authorization server's token
endpoint as defined in {{RFC7523}}, with the addition of an OPTIONAL
`redirect_uri` parameter.

redirect_uri
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

## Handling the Interaction Response

Upon receiving an interaction response as defined in
{{interaction-response}}, the client MUST:

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

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

