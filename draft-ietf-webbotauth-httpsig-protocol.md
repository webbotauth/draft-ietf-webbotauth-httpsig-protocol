---
title: "HTTP Message Signatures for automated traffic"
abbrev: "HTTP Message Signatures for Bots"
category: std

docname: draft-ietf-webbotauth-httpsig-protocol-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Web Bot Auth"
keyword:
 - not-yet
venue:
  group: "Web Bot Auth"
  type: "Working Group"
  mail: "web-bot-auth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/web-bot-auth/"
  github: "webbotauth/draft-ietf-webbotauth-httpsig-protocol"
  latest: "https://webbotauth.github.io/draft-ietf-webbotauth-httpsig-protocol/draft-ietf-webbotauth-httpsig-protocol.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk
 -
    fullname: Sandor Major
    organization: Google
    email: ietf@sandormajor.com

normative:
  DIGEST-FIELDS: RFC9530
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP-MESSAGE-SIGNATURES-IANA:
    title: HTTP Message Signatures
    target: https://www.iana.org/assignments/http-message-signature/http-message-signature.xhtml
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  HTTP-MORE-STATUS-CODE: RFC6585
  JWK: RFC7517
  JWK-OKP: RFC8037
  JWK-THUMBPRINT: RFC7638
  STRUCTURED-HEADERS: RFC9651
  URI: RFC3986
  WellKnownURIs:
    title: Well-Known URIs
    target: https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml


informative:
  HPACK: RFC7541
  HTTP-BEST-PRACTICES: RFC9205
  OAUTH-BEARER: RFC6750
  QPACK: RFC9204
  RFC8446:
  REGISTRY: I-D.draft-meunier-webbotauth-registry
  OWASP-SSRF:
    title: OWASP Server-Side Request Forgery Prevention Cheat Sheet
    target: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
  USE-CASES: I-D.draft-nottingham-webbotauth-use-cases
  WELLKNOWN-URI: RFC8615

--- abstract

This document describes a protocol for identifying automated
traffic using {{HTTP-MESSAGE-SIGNATURES}}. The goal
is to allow automated HTTP clients to cryptographically sign outbound
requests, allowing HTTP servers to verify their identity with confidence.

It defines the `Signature-Agent` header field for in-band key discovery, a
key directory format based on JWKS, and a well-known URI at which that
directory is served.


--- middle

# Introduction

Agents are increasingly used in business and user workflows, including AI assistants,
search indexing, content aggregation, and automated testing. These agents need to reliably identify
themselves to origins for several reasons:

1. Regulatory compliance requiring transparency of automated systems
2. Origin resource management and access control
3. Protection against impersonation
4. Service level differentiation between human and automated traffic

Current identification methods such as IP allowlisting, User-Agent strings, or shared API keys have
significant limitations in security, scalability, and manageability. This document defines a
protocol enabling agents to cryptographically identify themselves using {{HTTP-MESSAGE-SIGNATURES}}.
It proposes that every request from bots be signed by a private key owned by its provider.
This way, every origin can validate the service identifier. {{trust-model}}
defines what that identifier is and what validation it establishes.

# Motivation

There is an increase in agent traffic on the Internet. Many agents
choose to identify their traffic today via IP Address lists and/or unique
User-Agents. This is often done to demonstrate trust and safety claims, support
allowlisting/denylisting the traffic in a granular manor, and enable sites to
monitor and rate limit per agent operator. However, these mechanisms have drawbacks:

 1. User-Agent, when used alone, can be spoofed meaning anyone may attempt to
    act as that agent. It is also overloaded - an agent may be using Chromium and
    wish to present itself as such to ensure rendering works, yet it still wants to
    differentiate its traffic to the site.
 2. IP blocks alone can present a confusing story. IPs on cloud platforms have
    layers of ownership - the platform owns the IP and registers it in their
    published IP blocks, only to be re-published by the agent with little to bind
    the publication to the actual service provider that may be renting infra. Purchasing
    dedicated IP blocks is expensive, time consuming, and requires significant
    specialist knowledge to set up. These IP blocks may have prior reputation
    history that needs to be carefully inspected and managed before purchase and
    use.
 3. An agent may go to every website on the Internet and share a secret with
    them like a Bearer from {{OAUTH-BEARER}}. This is impractical to scale for any
    agent beyond select partnerships, and insecure, as key rotation is challenging
    and becomes less secure as the consumers scale.

Using well-established cryptography, we can instead define a simple and secure
mechanism that empowers small and large agents to share their identity.

## Objectives and Constraints {#objectives}

This protocol has two objectives:

1. Continuity of bot trust, so that an origin can tell it is dealing with the
   same party it dealt with before.
2. Optional binding to another anchor, such as a domain.

It works under two constraints:

1. Preserve the simplicity of usage for bots, and the simplicity of action for
   websites.
2. Require no pre-established relationship between the two.

The second constraint is what rules out shared secrets and per-site
onboarding. The first is a statement about operational cost on both ends: a
site today greps its logs for an IP address and a User-Agent, and with this
protocol it greps for a handle it can verify.

## HTTP Layer Choice

This protocol operates solely at the HTTP layer.
It allows signatures to be generated and
verified without modifying the transport layer or TLS stack. It enables
flexible deployment across proxies, gateways, and origin servers, and aligns
with existing tooling and infrastructure that already inspect and manipulate
HTTP headers.

Because the signature is embedded in the request itself, it travels with the
message through intermediaries, preserving end-to-end verifiability even when
requests are forwarded or transformed within the HTTP layer.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

**User**
: An entity initiating requests through an agent. May be a human operator or another system.

**Agent**
: An orchestrated user agent (e.g. Chromium, CURL). It implements the HTTP protocol and constructs valid HTTP requests with {{HTTP-MESSAGE-SIGNATURES}} signatures.

**Origin**
: An HTTP server receiving signed requests that implements the HTTP protocol and verifies {{HTTP-MESSAGE-SIGNATURES}} signatures. It acts as a verifier of the signature as defined by {{HTTP-MESSAGE-SIGNATURES}}.

# Identifiers and Trust Model {#trust-model}

This section defines the identifiers produced by this protocol and what a
verifier can conclude from a valid signature.

## The Signature-Agent URL is the identifier {#url-is-identifier}

An Agent identifies itself with the HTTPS URL it publishes its keys at, carried
in `Signature-Agent` ({{signature-agent}}). A verifier resolves that member
value directly as a JWK Set ({{key-distribution-and-discovery}}) and checks the
signature against the keys it returns. The identifier is the URL the verifier
fetched. What the verifier ends up with is a pair: that URL, and a key the URL
provides.
Origins can log, rate limit, allowlist, or block the URL the way they do IP
addresses and User-Agent today.

The URL on its own carries nothing. A client picks the value it sends, so an
unresolved `Signature-Agent` is a claim rather than an identity. It becomes an
identifier once the verifier fetches it and finds that it provides a key that
verifies the request ({{discovery-is-not-trust}}). Until then, verifiers MUST
NOT attach policy to it.

A valid signature over a resolved URL proves that a holder of a key that URL
publishes signed the covered message. The message can still be replayed.
Signature lifetimes and covered components bound the duration and scope of that
replay ({{generating-http-message-signature}}). It says nothing about who
operates the Agent, whether the Agent is benign, or whether the request is
authorized. Those are origin policy.

Nothing stops an Agent from abandoning a URL and standing up another one, and
the protocol does not try to prevent this. It targets honest clients that want
to be recognised across requests.

## Rotation {#rotation}

Because the identifier is the URL and not the key, an Agent can rotate keys
without losing continuity. It publishes the new key alongside the old one, then
drops the old one ({{key-rotation}}). The URL does not change, so a verifier
that recognised it before still recognises it after. No name and no third party
are involved.

`keyid` selects which key verifies a given request. Verifiers cannot use it to
carry continuity across a rotation, as that value is derived from the key
material.

## When No URL Is Available {#no-url}

A signed request MUST carry `Signature-Agent` ({{signature-agent}}). A verifier
can still be left with no URL to attribute to: discovery failed with nothing
cached ({{discovery-failure}}), redistributed material carried no proof
({{redistributed-key-material}}), or the header is absent. The identifier is
then the `keyid` thumbprint defined in
{{generating-http-message-signature}}. A verifier that already holds that key
verifies the request, and attributes it no further than the holder of that key.

This mode has no rotation. A new key is a new identifier, and the verifier has
no way to connect the two.

## What the URL Endorses {#discovery-is-not-trust}

Resolving a `Signature-Agent` URL over TLS establishes that the host named in
the URL served this key set at fetch time.
Whoever controls that URL says this key signs for it. That
is what makes the URL usable as an identifier, and all it gives you. It does
not say that the operator of that URL is honest, or that it is the same party
everyone knows about.

What matters is the association between a URL and the keys published there. A
verifier that already holds the keys does not need to fetch it. A verifier MUST
NOT attribute a request to a `Signature-Agent` URL unless it made this
association. This can be either by resolving the URL itself, at request time or
ahead of it, or from {{redistributed-key-material}}. Verifiers may refetch a URL
to handle key additions and removals, bounded by {{cache-behaviour}}.

Where a verifier obtains the same pair from more than one source, the newer
pair wins, including when it omits a key that older resolution included.
Pairs are ordered by when they were produced, not when the verifier obtained
them: the `created` parameter for a directory response signature
({{origin-binding-appendix}}), and the time of the fetch for a directory the
verifier resolved itself.

## Binding a Key to a Web Origin {#origin-binding}

A well-known URL is a special case of the above. When a
`Signature-Agent` value is the well-known URI registered in {{wkuri-reg}}, the
identifier is still the URL, but that URL now names a domain rather than an
arbitrary path on one. {{WELLKNOWN-URI}} reserves the path, so the domain
operator stands behind the key set.

In practice, this is meant to allow additional information to be carried against
a name. That mechanism lives in {{origin-binding-appendix}}. A verifier that
wants to use this case may also recognise the shape of the URL and apply those
checks itself.

## Out of Scope

This protocol does not authenticate human users, does not provide anonymous
authentication, and does not define authorization or delegation. It does not
define how trust is accrued, held, or exchanged, and it defines no
mechanism for one origin to convey an opinion about an Agent to another. See
{{privacy-considerations}}.

A client has a choice whether to sign its requests, and an origin has a choice
how it treats signed and unsigned requests. Multiple factors could influence
either decision, but the decisions themselves are outside the scope of this
document.

# Protocol Overview

~~~aasvg
+----------------+
| User           |
|     +-------+  |                                 +--------+
|     | Agent +--+----- signed request + URL ----->| Origin |
|     +---+---+  |                                 +---+----+
+---------+------+                                     |
          |                                            |
          |                +-----------+               |
          +-- publishes -->| Directory |<-- resolves --+
                           +-----------+
~~~
{: title="Web Bot Auth architecture"}

A User initiates an action requiring the Agent to perform an HTTP request.
The Agent constructs the request, generates a signature using its signing key,
and includes it in the request as defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}
along with the `Signature-Agent` header for discovery of its verification key.
Upon receiving the request, the Origin ensures it has the verification key for the Agent,
validates the signature, and processes the request if the signature is valid.
How a User directs an Agent is outside the scope of this document.

## Deployment Models

Signature verification can be performed either directly by origins or delegated
to a fronting proxy. Direct verification by origins provides simplicity and
control. Proxy verification offloads processing and enables shared caching across
multiple origins. The choice depends on traffic volume and operational
requirements.

## Generating HTTP Message Signature {#generating-http-message-signature}

{{HTTP-MESSAGE-SIGNATURES}} defines components to be signed.

Agents MUST include at least one of the following components:

`@authority`
: as defined in {{Section 2.2.3 of HTTP-MESSAGE-SIGNATURES}}

`@target-uri`
: as defined in {{Section 2.2.2 of HTTP-MESSAGE-SIGNATURES}}

Agents MUST include the following `@signature-params` as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`created`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`expires`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}. It is RECOMMENDED that `expires` be no more than 24 hours after `created`.

`keyid`
: MUST be a base64url JWK SHA-256 Thumbprint as defined in {{Section 3.2 of JWK-THUMBPRINT}} for RSA and EC, and in {{Appendix A.3 of JWK-OKP}} for ed25519.

`tag`
: MUST be `web-bot-auth`

The signing key is available to the agent at request time. Algorithms should be registered with IANA as part of HTTP Message Signatures Algorithm registry.

The creation of the signature is defined in {{Section 3.1 of HTTP-MESSAGE-SIGNATURES}}.

The components above bind the signature to an authority, not to a request. A
signature covering `@authority` alone verifies against any method, path, or body
sent to that authority until it expires, so anyone who observes one request can
reuse it against the same origin until then.
`expires` bounds how long that lasts; the covered components bound what it
reaches. {{field-compression}} covers what that costs on the wire.

Agents that want to narrow the scope of their signature SHOULD also cover the
following components. A signer that omits them remains conformant.

`@method`
: narrows the signature to one method. Defined in {{Section 2.2.1 of HTTP-MESSAGE-SIGNATURES}}

`@path`
: narrows the signature to one resource. {{example-multiple-signatures}} shows
it in use. Defined in {{Section 2.2.6 of HTTP-MESSAGE-SIGNATURES}}.

`@target-uri`
: narrows the signature to one resource, including the query string. The
signature is invalidated by intermediary reordering, which might happen for
caching purposes for instance. Defined in {{Section 2.2.2 of HTTP-MESSAGE-SIGNATURES}}.

`@query-param`
: narrows the signature to individual query parameters. This is a more robust
alternative to cover query parameters. Defined in {{Section 2.2.8 of HTTP-MESSAGE-SIGNATURES}}.


No component covers the body. An Agent that needs one MUST send and cover
`Content-Digest` {{DIGEST-FIELDS}}. This document does not require it. Most
automated traffic is `GET`, and a mandatory digest would force every Agent to
buffer request bodies it would otherwise stream.

### Signature-Agent {#signature-agent}

`Signature-Agent` is a Dictionary Structured Header as defined in
{{Section 3.2 of STRUCTURED-HEADERS}}. Its member values MUST be String Items
that contain an absolute {{URI}}, whose scheme MUST be `https`. Dictionary
members MUST NOT carry parameters. A verifier MUST ignore a member that carries
parameters. If dictionary values are not valid URIs, the entire header field
MAY be ignored.

Earlier versions of this protocol defined `Signature-Agent` as a bare String,
and deployments still send it ({{example-legacy}}). A verifier MAY accept that
form and treat it as a dictionary with a single member whose key is the label
of the signature covering it. Signers MUST send the dictionary form. The two
are distinguishable on the wire: a String Item begins with a double quote `"`
while a Dictionary member key does not.

A signed request MUST carry the `Signature-Agent` header, as described in {{sending-request}}.
Its member keyed to the signature label MUST be signed as a component as defined in {{Section 2.1 of HTTP-MESSAGE-SIGNATURES}}.
The `Signature-Agent` member identifies where candidate key material can be found.
The key used to verify the signature is selected by the `keyid` parameter of the
corresponding `Signature-Input` member.

This results in the following components to be signed

~~~
("@authority" "signature-agent";key="sig1")
~~~

### Multiple Signatures {#multiple-signatures}

A request MAY contain more than one Web Bot Auth signature. Each signature is
identified by its HTTP Message Signatures label. Each signer MUST provide a
`Signature-Agent` member for its label. A verifier MUST NOT attribute a
signature to a member that signature does not cover.

A signer MAY cover members from another signature label, which preserves
evidence that another signer contributed to the request. A signer that covers
`"signature";key=X` MUST also cover `"signature-input";key=X`, and MUST cover
every component identifier listed in `"signature-input";key=X`.

A signature value on its own does not identify the message it was computed
over, which is why {{Section 7.3.7 of HTTP-MESSAGE-SIGNATURES}} recommends
against signing one. Covering `signature-input` is not sufficient:
it lists component identifiers, whose values resolve against whatever
message the verifier holds. An outer signature that named those identifiers
without covering them would still verify after the whole header set was lifted
onto a different message, the ambiguity
{{Section 7.3.7 of HTTP-MESSAGE-SIGNATURES}} describes. Covering the union
addresses this: the outer signer commits to a message on which the inner signature
is checkable, under which key, and over which validity window.

A signer that cannot cover one of those components, because it changed the
value the inner signature was computed over, MUST NOT cover the inner
`signature` member. It signs the request on its own terms, and the inner
signature is left untouched.

Verifiers MUST validate each signature independently against its own covered
components and its own key. An outer signature that covers an inner one is
evidence that those bytes, over that set of components, were present. It does
not make the inner signature valid, and it does not express authorization,
delegation, or consent. Those meanings are deployment policy, or are carried in
separately signed fields.

### Replay Protection

Replay protection, including use of the `nonce` signature parameter, is
described in {{Section 7.2.2 of HTTP-MESSAGE-SIGNATURES}}. This document defines
no additional nonce requirements.

### Additional Headers

Agents MAY include additional components, such as specific HTTP headers, in the signature.
This can be prompted by the origin requesting additional headers, as described in {{requesting-message-signature}},
or initiated by the agent to provide more information within the signature scope.
For example, an agent might include an HTTP header expressing its intent and sign it.

Origins MAY ignore certain headers at their own discretion,
and request a new signature, as described in {{requesting-message-signature}}.

### Sending a Request {#sending-request}

An Agent SHOULD send a request with the signature generated above.

~~~
NOTE: '\' line wrapping per RFC 8792

GET /path/to/resource HTTP/1.1
Host: origin.example.com
Signature: sig=abc==
Signature-Input: sig=("@authority" "signature-agent";key="sig");\
                 created=1700000000;\
                 expires=1700011111;\
                 keyid="ba3e64==";\
                 tag="web-bot-auth"
Signature-Agent: sig="https://signer.example.com"
~~~

A signed request carries three headers

1. `Signature` defined in {{generating-http-message-signature}}
2. `Signature-Input` defined in {{generating-http-message-signature}}
3. `Signature-Agent` defined in {{signature-agent}}

The Origin learns the URL from the request, so it resolves keys after the
first request rather than before it. Later requests verify from cache
({{cache-behaviour}}).

~~~aasvg
+-------+                   +--------+     +-----------+
| Agent |                   | Origin |     | Directory |
+---+---+                   +---+----+     +-----+-----+
    |                           |                |
    +== Signed request + URL ==>|                |
    |                           |                |
    |                       .-- if keys not resolved --.
    |                       |   |                |     |
    |                       |   +--- Resolve --->|     |
    |                       |   |<---- Keys -----+     |
    |                       |   |                |     |
    |                       '--------------------------'
    |                           |                |
    |<======== Response ========+                |
    |                           |                |
~~~
{: title="Key resolution follows the first request"}

## Requesting a Message Signature {#requesting-message-signature}

{{Section 5 of HTTP-MESSAGE-SIGNATURES}} defines the `Accept-Signature` field which can be used to request a Message Signature from a client by an origin.
An Origin MAY choose to request signatures from clients that did not initially
provide them. A resulting signature MUST satisfy this profile. The Origin MAY
request additional parameters defined by {{HTTP-MESSAGE-SIGNATURES}}, including
`nonce`.
The status code SHOULD be 403 Forbidden as defined in {{Section 15.5.4 of HTTP}}.

An Origin that detects a signature or nonce has been used more often than its
policy permits MAY request a new signature and respond with 429 Too Many
Requests as defined in {{Section 4 of HTTP-MORE-STATUS-CODE}}.

## Validating Message Signature

Upon receiving an HTTP request, the origin has to verify the signature. The algorithm is provided in {{Section 3.2 of HTTP-MESSAGE-SIGNATURES}}.
Similar to a regular User-Agent check, this happens at the HTTP layer, once headers are received.

Additional requirements are placed on this validation:

- During step 1 to 3 included, if the Origin fails to parse the provided `Signature`, `Signature-Input`, or `Signature-Agent` headers, it MAY respond with status code 400 Bad Request as defined in {{Section 15.5.1 of HTTP}}.
- During step 4, the Origin MAY discard signatures for which the `tag` is not set to `web-bot-auth`.
- During step 5, the Origin MAY discard signatures for which it does not know the `keyid` for the `Signature-Agent` URL the signature covers.
- During step 5, if the `keyid` is not known for that URL, the Origin MAY fetch key material as indicated by the `Signature-Agent` header defined in {{signature-agent}}. Fetching key material affects only whether verification is possible, not what a valid signature means ({{discovery-is-not-trust}}).

Key lookup MUST be keyed on the (URL, key) pair, not on the key alone. A
verifier that indexes by `keyid` alone will verify a request as coming from one URL
that provides a key it learned from another, and attribute that request to the URL the client
asserted. The party whose URL is asserted cannot detect or stop this: its own
directory is never fetched, so no rotation or removal has any effect.

## Key Distribution and Discovery {#key-distribution-and-discovery}

This section describes how a verifier resolves a `Signature-Agent` URL to key
material. {{discovery-is-not-trust}} covers what the fetch does and does not
establish.

The reference for discovery is an HTTPS URL, carried in a `Signature-Agent`
member as defined in {{signature-agent}}. A verifier fetches that URL directly
and interprets the response as a JWK Set.

The resource fetched under this section MUST be served with a `200 (OK)` HTTP
status code. A verifier MUST treat all other HTTP status codes as discovery
failures and MUST NOT automatically follow HTTP redirects.

The URL the verifier resolved is the identifier, with any query discarded. The
verifier fetches the member value as sent, so the query is dropped from the
identifier and not from the request. Otherwise one key set would yield an
identifier per spelling, and an Agent could mint them at will.

Identifiers are compared after normalization as described in
{{Section 6.2.2 of URI}} and {{Section 6.2.3 of URI}}. Two identifiers are the
same when their normalized forms are equal octet for octet.

TLS authenticates the host but not the path. An arbitrary path does not name
the host's operator. The well-known URI is reserved, so that URL additionally
names a domain ({{origin-binding}}).

The key is selected using the `keyid` parameter in `Signature-Input`.

Note: when a JWK set is served at the well-known URI registered in
{{wkuri-reg}}, JWK MAY carry a `kid`. In this case, it MUST be set to the
thumbprint defined in {{generating-http-message-signature}}, so a verifier
selects a key by matching `keyid` against `kid`. Deriving `kid` from the key
material keeps it globally unique and lets a verifier check the directory's own
labelling rather than trusting it.

A JWK Set at another URL may serve other consumers, where `kid` is an
operator-chosen label. A verifier that cannot match `keyid` against `kid` there
computes thumbprints instead.

~~~
Signature-Agent: sig1="https://signature-agent.test/jwks.json"
~~~

### Directory Format {#configuration}

The resource MUST be a JSON Web Key Set (JWKS) as defined in {{Section 5 of
JWK}}. A JWK `alg` member, when present, follows {{Section 4.4 of JWK}}. The
`alg` signature parameter, when present, follows the HTTP Signature Algorithms
section of {{HTTP-MESSAGE-SIGNATURES-IANA}}.

The directory MUST be served over HTTPS. A directory served at the well-known
URI registered in {{wkuri-reg}} MUST be served with media type
`application/http-message-signatures-directory+json`.

A verifier SHOULD validate the directory format and reject malformed entries.

~~~
GET /.well-known/http-message-signatures-directory HTTP/1.1
Host: example.com
Accept: application/http-message-signatures-directory+json

HTTP/1.1 200 OK
Content-Type: application/http-message-signatures-directory+json
Cache-Control: max-age=86400
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600
  }]
}
~~~

### Key Rotation {#key-rotation}

Directory operators SHOULD rotate keys by publishing the old and the new key
together, then removing the old one:

1. Add the new key to the directory before its intended use date
2. Continue to include the old key until its expiration date
3. Remove expired keys from the directory

Removing a key from the directory deactivates it. Verifiers stop accepting it
once their cached copy expires, so the directory's cache lifetime bounds how
long a removed key keeps verifying. Verifiers SHOULD cache the directory
contents and refresh upon expiration, as described in {{cache-behaviour}}.

It is not a revocation mechanism, and this document does not define any.

### Redistributed Key Material {#redistributed-key-material}

IP addresses and user-agent have been aggregated and distributed via lists.
This section says what a verifier may conclude from key material it did not
fetch itself.
Defining a format for redistribution is out of scope.

A verifier MUST NOT attribute a request to a `Signature-Agent` URL on the basis
of redistributed key material unless it carries, for the key in question, a
valid directory response signature as described in {{origin-binding-appendix}}
whose `expires` has not passed. Without that proof the material stays usable
for verifying signatures, but it carries no URL, so the identifier falls back
to the key thumbprint ({{no-url}}).

The main requirement is to terminate the TLS connection.
A verifier polling a directory on its own schedule is resolving it. So is a
control plane polling on behalf of the verifiers it serves. None of these options
constitute redistribution. Nor is a list that names directory URLs rather than embedding
keys. For instance, {{REGISTRY}} works that way, and the verifier still resolves them.

## Session Considerations {#sessions}

Per-request signing and verification costs CPU; uncached key discovery adds
latency. For high request rates, an origin can verify a request-specific
signature once and issue a session credential for later requests. This can
amortize asymmetric verification and reduce bytes, but adds the risks of token
theft and replay.

A reused signature already has token semantics until `expires`. A session
established from one extends that window past `expires` unless the credential is
bounded to it: no longer-lived, and no wider in scope than the components the
signature covered. Session establishment and binding are out of scope.

# Security Considerations {#security-considerations}

## Use of TLS

We reassess {{Section 7.1.2 of HTTP-MESSAGE-SIGNATURES}}.
Clients SHOULD use TLS {{RFC8446}}
(https) or equivalent transport security when making requests with
Message signatures. Failing to do so exposes the Message signature to numerous
attacks that could give attackers unintended access.

This include reverse proxy and their consideration presented in {{reverse-proxy}}.

An origin SHOULD refuse Signature headers when communicated over an unsecured channel.

## Performance Impact

Origins should account for the overhead of signature verification in their operations. A local cache of public keys reduces network requests and verification latency. The choice of signing algorithm impacts CPU requirements. Origins should monitor verification latency and set appropriate timeouts to maintain service levels under load.
See {{sessions}}: a session amortizes that cost by replacing verification with a
bearer credential. {{field-compression}} covers the byte cost.

## Key Compromise Response

This document defines no revocation. Removing a compromised key from the
directory is the only remedy, and it takes effect at each verifier on its next
refresh, so the key can keep verifying for as long as {{key-rotation}} allows.
The protocol carries no channel back to verifiers, so an
Agent cannot reach them sooner. Signature lifetimes
({{generating-http-message-signature}}) are the only lever that acts faster.

Agents SHOULD remove a compromised key and publish a replacement immediately.
Origins should support rapid key rotation and monitor for suspicious signature
patterns.

## Shared Secrets Considered Harmful

Implementations MUST NOT use shared HMAC defined in {{Section 3.3.3 of HTTP-MESSAGE-SIGNATURES}}.
Shared secrets break non-repudiation and make auditing
difficult. Each automated client SHOULD use a unique asymmetric keypair to
ensure attribution, support key rotation, and enable effective rotation if
needed.

## Key Reuse Considered Harmful

Implementations SHOULD NOT reuse a signing key for different purposes. For
example, if an agent implementer has two agents they want to differentiate,
these should use distinct signing keys and signing key directories.

## Reverse Proxy Consideration {#reverse-proxy}

An origin may be placed behind a reverse proxy, which means the proxy will see
the `Signature` and `Signature-Agent` headers before the origin does.
A proxy SHOULD NOT strip the `Signature` or `Signature-Agent` headers from
requests.

A proxy SHOULD NOT replay signatures against other reverse proxies used by the
origin, as this allows impersonation of the principal signature agent.

### Signature-Agent Labeling

{{Section 7.2.5 of HTTP-MESSAGE-SIGNATURES}} allows an intermediary to relabel
a signature, because the label of a `Signature` dictionary member is not part
of the signature base. The key of a `Signature-Agent` member is different: when
a signature covers `"signature-agent";key="agent2"`, that key appears in the
signature base, so changing it invalidates the signature. Only the holder of
the signing key can produce a signature over the new member key.

An intermediary MUST NOT alter the key of a `Signature-Agent` member that is
covered by a signature it is not able to recompute. Relabeling the `Signature`
dictionary member remains permitted.

A signer acting as an intermediary on its own signature is not restricted by
this, since it can sign the result.

## Server-Side Request Forgery (SSRF) {#ssrf}

As described in {{key-distribution-and-discovery}}, verifiers may fetch key directories based on
the value conveyed in `Signature-Agent` when included in a request. Since
clients control the `Signature-Agent` header value, this introduces a risk of
server-side request forgery (SSRF) attacks by malicious clients.

Verifiers SHOULD take appropriate precautions as follows:

`Response size`
: a directory can be arbitrarily large. Verifiers SHOULD reject responses
  exceeding a defined byte limit after content decoding.

`Key count`
: a JWKS with many keys forces O(n) key search. Verifiers SHOULD enforce
  a maximum key count.

`Fetch latency`
: no timeout allows slowloris-style exhaustion. Verifiers SHOULD apply
  a wall-clock timeout to directory fetches.

`Redirect chains`
: unbounded HTTP redirects can be used to amplify requests. Verifiers
  SHOULD limit redirect depth.

`Network address ranges`
: no address filtering can target internal services. Verifiers SHOULD
  prevent directory fetches to private, loopback, and link-local
  address ranges.

Further recommendations can be found in the Open Worldwide Application
Security Project (OWASP) SSRF Prevention Cheat Sheet {{OWASP-SSRF}}.

## Test and Demonstration Keys

Test keys, including the example keys in {{HTTP-MESSAGE-SIGNATURES}}, MUST NOT
be used in production. Verifiers SHOULD reject known test keys when they are
detected in key directories or out-of-band configuration.

## Static Signatures

Deployments MUST NOT treat a precomputed Web Bot Auth signature as a long-lived
access credential. A reusable static signature has bearer-token semantics and can
be replayed until the covered signature parameters, key, or verifier policy make
it unusable.

Agents SHOULD generate signatures for the request being sent, with bounded
`created` and `expires` values. Long expiration windows increase replay risk.

## Discovery Failure {#discovery-failure}

Resolving a `Signature-Agent` URL can fail in several ways: the name does not
resolve, the connection or TLS handshake fails, the response is not a directory
or contains no key matching `keyid`, or the fetch is refused by the verifier's
own limits ({{ssrf}}). All have the same outcome for the request in hand. The
verifier holds no association between that URL and the signing key, so under
{{discovery-is-not-trust}} it MUST NOT attribute the request to that URL. It may
still verify the signature if it holds the key by other means, in which case the
identifier is the thumbprint ({{no-url}}); otherwise the request is unverified.

They differ in what they say about cached state, and verifiers MUST keep them
apart. A directory that resolves and does not contain the key is evidence: it is
newer than whatever the verifier holds, and under {{discovery-is-not-trust}}
replaces it. That is how a removed key stops verifying. A directory that fails
to resolve is not evidence and MUST NOT evict a cached entry, or an
operator's outage revokes its keys at every verifier at once.

A failed fetch says nothing about the signer. It does not prove the signer is
malicious, and it does not make the request trusted. What an origin does with
an unverified request is local policy, and treating it as a distinct outcome
rather than as success or failure is discussed in {{verifier-outcomes}}.
Verifiers should also expect failures to be correlated: a single operator's
directory going down takes out every request naming it at once, across every
verifier whose cache expires in the same window.

## Unsigned Requests {#unsigned-requests}

Most HTTP requests carry no signature. A verifier that sees none has learned
nothing about the sender: not that it is automated, not that it is human, not
that it is evading anything. Absence of a signal is not evidence about the
party that did not send it, in the same way that a failed fetch
({{discovery-failure}}) is not evidence about the signer.

What an origin does with a request it cannot attribute is its own decision, as
it was before this protocol existed. This document neither requires an origin
to treat unsigned requests differently nor gives it grounds to.

# Privacy Considerations {#privacy-considerations}

## Public Identity

This protocol assumes that automated clients identify themselves
explicitly using digital signatures. The identity associated with a signing
key is expected to be publicly discoverable for verification purposes. This
reduces anonymity and allows receivers to associate requests with specific
agents. If an agent wishes not to identify itself, this is not the right
choice of protocol for it.

## No Human Correlation

A key tied to a specific human individual exposes personally identifiable
information and makes the key usable for user tracking or profiling. A key that
represents a role, company, or automation identity (e.g., "news-aggregator-bot",
"example-crawler-v1") avoids this.

## Minimizing Tracking Risks

To limit tracking risks, implementations SHOULD avoid long-lived, globally
unique key identifiers unless strictly necessary. Key rotation SHOULD be
supported, and clients SHOULD take care to avoid signing information that
could be used to correlate activity across contexts, especially where
sensitive user data is involved.

## Directory Content and Access Patterns

A key directory should only contain keys actively used for signing. Additional
keys or metadata expose more about the signing service than verification
requires. Verifiers fetching a directory also reveal something about their
verification patterns, so directory servers should avoid logging personally
identifiable information from directory requests.


# IANA Considerations

This section contains considerations for IANA.

## Well-Known 'http-message-signatures-directory' URI {#wkuri-reg}

This document updates the "Well-Known URIs" Registry {{WellKnownURIs}} with the
following values.

| URI Suffix  | Change Controller  | Reference | Status | Related information |
|:------------|:-------------------|:----------|:-------|:--------------------|
| http-message-signatures-directory | IETF | this document | permanent | None |
{: #wellknownuri-values title="'http-message-signatures-directory' Well-Known URI"}

## Media Types

The following entries should be added to the IANA "media types"
registry:

- "application/http-message-signatures-directory+json"

The templates for these entries are listed below and the
reference should be this RFC.

### "application/http-message-signatures-directory+json" Media Type

Type name:

: application

Subtype name:

: http-message-signatures-directory

Required parameters:

: N/A

Optional parameters:

: N/A

Encoding considerations:

: "binary"

Security considerations:

: see {{security-considerations}}

Interoperability considerations:

: N/A

Published specification:

: this specification

Applications that use this media type:

: Services that implement the signer role for HTTP Message
  Signatures and verifiers that interact with the signer for
  the purpose of validating signatures.


Fragment identifier considerations:

: N/A

Additional information:

: <dl spacing="compact">
  <dt>Magic number(s):</dt><dd>N/A</dd>
  <dt>Deprecated alias names for this type:</dt><dd>N/A</dd>
  <dt>File extension(s):</dt><dd>N/A</dd>
  <dt>Macintosh file type code(s):</dt><dd>N/A</dd>
  </dl>

Person and email address to contact for further information:

: see Authors' Addresses section

Intended usage:

: COMMON

Restrictions on usage:

: N/A

Author:

: see Authors' Addresses section

Change controller:

: IETF
{: spacing="compact"}


--- back

# Use Cases and What They Need {#use-cases}

{{USE-CASES}} collects the use cases this group has discussed. Most are served
by the URL alone. The table below records which ones need the domain binding in
{{origin-binding-appendix}}, and why.

| Use case | What the origin does | Needs |
|:---------|:---------------------|:------|
| Mitigating volumetric abuse | Rate limit per URL | URL |
| Controlling access by bots | Set policy per URL | URL |
| Providing different content to bots | Recognise a given URL | URL |
| Auditing bot behaviour | Group logs by URL | URL |
| Classifying traffic | Correlate observed behaviour with a URL | URL |
| IP address mobility and sharing | Nothing: the signature does not depend on the IP | URL |
| Robots.txt alignment | Match the crawler against a name in the file | Domain |
| Conveying contextual information | Read signed headers alongside the identifier | Domain |
{: title="Use cases and the identifier they need"}

The last two are the pattern from {{origin-binding}}. Both consume something
held against a name rather than against the key: a robots.txt file names
crawlers, and contextual assertions are only worth as much as the party making
them. End-user authentication and anonymous authentication are out of scope.

# Validating the Domain Binding {#origin-binding-appendix}

This appendix describes what a verifier checks when it wants the domain a key
is published under, rather than the URL on its own. It applies when the
`Signature-Agent` value is the well-known URI registered in {{wkuri-reg}}.
Verification, rotation, and continuity do not depend on any of it, and a
verifier that only needs the URL as an identifier can skip the whole appendix.

Authority over the domain comes from the TLS connection to the directory.
Nothing below adds to that.

## Possession Proof on the Directory Response

It is RECOMMENDED that a directory server construct and include one HTTP
Message Signature per key with the response, as defined in
{{HTTP-MESSAGE-SIGNATURES}}. Each key SHOULD be used to provide one signature.
These signatures prove possession of the advertised keys and, by covering
`@authority`, prevent the key set from being re-served under a different
authority. This matters for a domain-bound identifier, where the verifier is
about to consume information it holds against the name: it distinguishes a key set
the key holders assembled from one that was copied.

Directory server MUST include the following covered components:

`@authority`
: as defined in {{Section 2.2.3 of HTTP-MESSAGE-SIGNATURES}}. `req` flag defined in {{Section 2.4 of HTTP-MESSAGE-SIGNATURES}} MUST be set.

`content-digest`
: as defined in {{DIGEST-FIELDS}}.

Directory server MUST include the following `@signature-params` as defined in
{{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`created`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

`expires`
: as defined in {{Section 2.3 of HTTP-MESSAGE-SIGNATURES}}

Without them the signature is a permanent assertion that these keys were bound
to this authority at some unstated time, of no use to a verifier
consuming it through {{redistributed-key-material}}.

`keyid`
: MUST be a base64url JWK SHA-256 Thumbprint as defined in {{Section 3.2 of JWK-THUMBPRINT}} for RSA and EC, and in {{Appendix A.3 of JWK-OKP}} for ed25519.

`tag`
: MUST be `http-message-signatures-directory`

A verifier using a corresponding directory response signature as proof MUST
validate it using the key provided by the directory and MUST validate the
`Content-Digest` field against the response body. If either validation fails,
the proof is invalid. A verifier MAY use a key obtained by resolving the
directory itself whether or not it has valid proof. Attribution from
redistributed key material is governed by {{redistributed-key-material}}. A
verifier MUST reject a directory response signature whose `created` is in the
future, as it would a certificate that is not yet valid.
{{discovery-is-not-trust}} orders competing evidence by `created`, so a
future-dated signature would outrank every later fetch.

## What the Binding Attaches To

The binding is not exclusive. Several domains may publish the same key, and the
binding attaches to the pair the verifier validated, not to the key on its own.
A verifier that recognises a key under one domain has learned nothing about the
same key served under another.

# Deployment Guidance

This appendix is operational guidance. It does not define new protocol
requirements.

## Verifier Outcomes {#verifier-outcomes}

Verifiers should keep three outcomes distinct:

`verified`
: the signature and key material validate.

`invalid`
: the signature, covered components, key, or freshness checks fail.

`unverified`
: the verifier cannot obtain enough information to decide, for example because
  directory discovery failed or the key is unknown.

Origins can apply local policy to each outcome. During deployment, treating
`unverified` as one bot-management signal is safer than treating it as either
`verified` or `invalid`.

## Directory Availability

Directory resources are bootstrap material. Operators serving a directory should
make it reachable without requiring Web Bot Auth on the directory request. They
should also avoid bot protection rules that block ordinary verifier fetches of
the well-known resource.

The directory endpoint should support `GET`. Supporting `HEAD`, `ETag`,
`Last-Modified`, `Cache-Control`, and conditional requests can reduce fetch
load. Cache is specifically discussed in {{cache-behaviour}}.

## Bounded Directory Fetches

Verifiers fetch directories named by untrusted requests, and should bound those
fetches as described in {{ssrf}}.

Verifiers should also coalesce concurrent fetches for the same directory and
apply per-directory or per-origin concurrency limits. This avoids a fetch storm
when many requests reference the same uncached directory.

## Cache Behaviour {#cache-behaviour}

Verifiers should use normal HTTP caching semantics {{HTTP-CACHE}} for key
directories. In particular, verifiers should respect `Cache-Control`, `Expires`,
`Date`, `ETag`, and `Last-Modified` when present.

A verifier should not fetch the directory for every request. It should refresh
cached directories when they become stale, and can use background refresh with
jitter to avoid synchronized refetches.

## Negative Caching and Retry

Verifiers can cache unsuccessful discovery outcomes for a short period to reduce
repeated fetches. Negative cache entries should expire after no more than five
minutes. They are operational throttling state, not proof that a signature is
invalid.

Network failures, TLS failures, and `5xx` responses should be treated as
transient unless local policy says otherwise. Verifiers should retry with bounded
exponential backoff and jitter. When a directory response includes
`Retry-After`, verifiers should respect it as described by {{HTTP}} and
{{HTTP-BEST-PRACTICES}}.

## Freshness and Replay

Shorter signature lifetimes reduce replay risk but increase sensitivity to clock
skew and signing failures. Whether a verifier requires and enforces nonces is
deployment policy. Rejecting duplicate nonces requires an atomic check-and-record
across the enforcement scope. Verifiers need to bound retention and admission of
nonce state. If that state is unavailable, the verifier should not treat
signature validation as evidence that replay was detected or prevented. Replay
protection is described in
{{Section 7.2.2 of HTTP-MESSAGE-SIGNATURES}}.

Verifiers should avoid accepting signatures with freshness windows longer than
their risk model permits.

## Field Compression {#field-compression}

Covering per-request components costs bytes when a connection is reused. HPACK
{{HPACK}} and QPACK {{QPACK}} can index a repeated `Signature`,
`Signature-Input`, or `Signature-Agent` value, so a signature reused across
requests on one connection is sent once and referenced afterwards. A per-request
value cannot be referenced; it is sent as a literal every time. Huffman coding
and an indexed field name reduce that literal, they do not replace the
reference.

This is not a reason to widen the covered components. The bytes saved are the
bytes of a credential anyone who observes it can replay until `expires`
({{generating-http-message-signature}}), and one static signature for many
requests is an anti-pattern ({{deployment-anti-patterns}}). An encoder that
treats a signature as a credential may also decline to index it
({{Section 7.1.3 of HPACK}}).

## Directory Response Signature Lifetimes

Where the key set is redistributed, revocation latency is already floored by
how often the redistributor republishes, so a short `expires` on a directory
response signature ({{origin-binding-appendix}}) buys nothing and costs
availability: at expiry every consumer drops that operator's keys to unverified
at once, with no serving stale. Operators should set `expires` well beyond the
republication interval of any list they expect to appear in. The lever for
faster revocation is publishing more often, not signing shorter.

## Rollout and Fallback

Web Bot Auth deployments will coexist with existing bot identification signals
during rollout. Verifiers can continue to use existing methods such as IP-based
checks, forward-confirmed reverse DNS, local allowlists, and reputation systems.

Fallback should not turn an unsupported or unverifiable Web Bot Auth signature
into a trusted identity. It should leave the request in the origin's existing
bot-management path.

## Proxies and Intermediaries

Proxies and intermediaries need to preserve the fields covered by a signature if
the origin will verify that signature. If a proxy rewrites the authority, path,
or signed header fields, the origin may no longer see the message that was
signed.

A deployment can instead verify at the proxy and pass the result to the origin
through a deployment-local trusted channel. That assertion is local policy; it is
not a replacement for the original HTTP Message Signature.

## CORS

Key directories contain public key material. If browser-based verifiers need to
fetch them cross-origin, a directory server can use a permissive CORS policy such
as `Access-Control-Allow-Origin: *` without credentials. CORS is not key
authentication and does not replace signature validation.

## Deployment Anti-Patterns {#deployment-anti-patterns}

Deployments should avoid:

* using test or demonstration keys in production
* issuing one static signature for many requests
* asking users to copy long-lived signatures into third-party tools
* sharing one signing key across unrelated agents or purposes
* relying on manual key rotation as the only revocation mechanism

# Examples

## Delegation and Chaining

Delegation and chaining are out of scope for this document and are expected
to be specified separately. Input is welcome on the associated
[GitHub issue](https://github.com/webbotauth/draft-ietf-webbotauth-httpsig-protocol/issues/27).

## Multiple Signatures with a Remote Browser {#example-multiple-signatures}

This example shows Alice's agent using a remote browser to fetch a resource. The
agent signs selected request fields. The remote browser signs the request it
sends to the origin and also covers the agent's signature fields. The signature
values are illustrative; this is not a test vector.

~~~
NOTE: '\' line wrapping per RFC 8792

GET /resource HTTP/1.1
Host: origin.example
Signature-Agent: agent="https://agent.alice.example",\
 browser="https://browser.example"
Signature-Input: agent=("@method" "@authority" "@path"\
 "signature-agent";key="agent");created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;tag="web-bot-auth",\
 browser=("@method" "@authority" "@path"\
 "signature-agent";key="browser"\
 "signature-agent";key="agent"\
 "signature-input";key="agent"\
 "signature";key="agent");created=1735689601\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;tag="web-bot-auth"
Signature: agent=:YWdlbnQtc2lnbmF0dXJl:,\
 browser=:YnJvd3Nlci1zaWduYXR1cmU=:
~~~

The origin verifies each signature on its own. The `agent` signature covers the
fields selected by Alice's agent. The `browser` signature covers the request
sent by the remote browser, its own `Signature-Agent` member, and all three of
the `agent` label's fields, as {{multiple-signatures}} requires. This records
that the remote browser forwarded a request carrying the agent's signature. It
does not say that Alice's agent authorized the remote browser to act for it.

# Test Vectors

Except where noted, these vectors exercise the minimum this document requires:
`@authority` and a `Signature-Agent` member and nothing else, with an `expires`
far enough out that they do not age. That combination is a parsing and
verification exercise, not a
configuration to copy: as {{generating-http-message-signature}} explains, a
signature covering `@authority` alone is reusable against that authority for
any method, path, and body until it expires. Deployments should cover more and
expire sooner.

## RSASSA-PSS Using SHA-512

The test vectors in this section use the RSA-PSS key defined in {{Appendix B.1.2 of HTTP-MESSAGE-SIGNATURES}}.
This section includes non-normative test vectors that may be used as test cases to validate implementation correctness.

### Signature-Agent Included Present on the Request {#example-signature-agent-included}

This example presents a minimal signature using the rsa-pss-sha512 algorithm over test-request. The request contains
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"signature-agent";key="sig2": "https://signature-agent.test"
"@signature-params": ("@authority" "signature-agent";key="sig2")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=4889289600\
 ;nonce="wcfPQPh7SzkvrIVvhD00vNk9PkxJNY2NVbYl2PVBB4zmUoluSwE7W6bPtF60QA3k8g06FU7PPCD+J58YofY1zg=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig2`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: sig2="https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent";key="sig2")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=4889289600\
 ;nonce="wcfPQPh7SzkvrIVvhD00vNk9PkxJNY2NVbYl2PVBB4zmUoluSwE7W6bPtF60QA3k8g06FU7PPCD+J58YofY1zg=="\
 ;tag="web-bot-auth"
Signature: sig2=:iOCCPgZeV5D5KlfZLXIFQOZR2njqFiCMZIrb5HRRLrFZR5Wtpsn7HwbKfWYHqW3dYU7HV/RohjNg7fuyQ1Mdy2J4mya9JnQCZtCcIcOK/LM9Yc6g6ZTw0onurllrzEzuNQIFFkBTgPJZlA3rHioFrmDmn+nnN7HaC+it/PA2iGfENHYq9D3Ud7ZeDICmF1s5bsMgc+y+MxTPFQHxOEyhbCDjZbT5yeSkGP1Czczg7Cpf0Fz/wuF3mJJ7TL1mzQk3PPizXFr/EnjdUWri1/W8M/f93d389BFF+qbkISzAvOHEk5TqOoyrzzotTn3xmYN6SCGTHG42fK4rA4Ei0qCnnA==:
~~~

### Legacy Signature-Agent, sf-string {#example-legacy}

Retained for implementers migrating to the dictionary form ({{signature-agent}}). Do not copy it into new deployments.

This example presents a minimal signature using the rsa-pss-sha512 algorithm over test-request. The request contains
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"signature-agent": "https://signature-agent.test"
"@signature-params": ("@authority" "signature-agent")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=1735693200\
 ;nonce="XSHtZVCThSIAksXsH9WBs6AtxtXC0eQGiIcUGSoJstFs8lAWakjhrfwzLhyjtme5iXMZvmFWqDEs6cT3Jf+BbQ=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig2`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: "https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent")\
 ;created=1735689600\
 ;keyid="oD0HwocPBSfpNy5W3bpJeyFGY_IQ_YpqxSjQ3Yd-CLA"\
 ;alg="rsa-pss-sha512"\
 ;expires=1735693200\
 ;nonce="XSHtZVCThSIAksXsH9WBs6AtxtXC0eQGiIcUGSoJstFs8lAWakjhrfwzLhyjtme5iXMZvmFWqDEs6cT3Jf+BbQ=="\
 ;tag="web-bot-auth"
Signature: sig2=:I1QWNzGXdP1a4dSvOHLCVOOanEYHDk+ZsVxM9MLX/p4ko69ghKwR5EOtAD96g7g4GWP7lmpM/jFAf9q8EFRDTPLjUXySwMv4YPgabv2LQihTJG2y8a2m6IGltyruwQNiqSJVUuRaG9+b17CGmAMFZh30X6GXLdQJrCARpeTqPwp2DC+a8haDE/VE5EruqzjA5/2mKwvrkzkSqeW5tOVtFwWRRHIOidquf/8Je6kM9mhgkg4arudLA5SL4wyyYE1jURIgcOl8agrfdJ5Def23DIRtiOLRa8jT9cpTLFAuFHN+mrZA/LH9h0gSIg1cPb+0cMASee5uku1KjWcFer7jWA==:
~~~

## EdDSA Using Curve edwards25519

The test vectors in this section use the Ed25519 key defined in {{Appendix B.1.4 of HTTP-MESSAGE-SIGNATURES}}.
This section include non-normative test vectors that may be used as test cases to validate implementation correctness.

### Signature-Agent Included Present on the Request

This example presents a minimal signature using the ed25519 algorithm over test-request. The request contains
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"signature-agent";key="sig2": "https://signature-agent.test"
"@signature-params": ("@authority" "signature-agent";key="sig2")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=4889289600\
 ;nonce="n9p433xm+NJ3ph3upfBIGmsuwHw387YV7Q/F+6BSpGCVjYCqQw6rznNA8PVVLySrAWsv0hQtFioQb6E1YsauiA=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig2`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: sig2="https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent";key="sig2")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=4889289600\
 ;nonce="n9p433xm+NJ3ph3upfBIGmsuwHw387YV7Q/F+6BSpGCVjYCqQw6rznNA8PVVLySrAWsv0hQtFioQb6E1YsauiA=="\
 ;tag="web-bot-auth"
Signature: sig2=:CGokAoY/FrPpyuzTe1Fb2kLTAH9Idw6qSFd1+D77r1firsMRCxeuLu5e6O+ek3zQgDMAzjiXtLVfqE00xnABCQ==:
~~~

### Legacy Signature-Agent, sf-string

Retained for implementers migrating to the dictionary form ({{signature-agent}}). Do not copy it into new deployments.

This example presents a minimal signature using the ed25519 algorithm over test-request. The request contains
a `Signature-Agent` header.

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority": example.com
"signature-agent": "https://signature-agent.test"
"@signature-params": ("@authority" "signature-agent")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=1735693200\
 ;nonce="e8N7S2MFd/qrd6T2R3tdfAuuANngKI7LFtKYI/vowzk4lAZYadIX6wW25MwG7DCT9RUKAJ0qVkU0mEeLElW1qg=="\
 ;tag="web-bot-auth"
~~~

This results in the following Signature-Input and Signature header fields being added to the message under the label `sig2`:

~~~
NOTE: '\' line wrapping per RFC 8792

Signature-Agent: "https://signature-agent.test"
Signature-Input: sig2=("@authority" "signature-agent")\
 ;created=1735689600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;alg="ed25519"\
 ;expires=1735693200\
 ;nonce="e8N7S2MFd/qrd6T2R3tdfAuuANngKI7LFtKYI/vowzk4lAZYadIX6wW25MwG7DCT9RUKAJ0qVkU0mEeLElW1qg=="\
 ;tag="web-bot-auth"
Signature: sig2=:jdq0SqOwHdyHr9+r5jw3iYZH6aNGKijYp/EstF4RQTQdi5N5YYKrD+mCT1HA1nZDsi6nJKuHxUi/5Syp3rLWBA==:
~~~

### Signed Directory Response

This example presents the possession proof described in
{{origin-binding-appendix}}, using the ed25519 algorithm. The directory server
signs its own response with the key it publishes. The signature covers
`@authority` from the request that fetched the directory, so the key set cannot
be re-served under another authority, and `content-digest` over the response
body, so the key set cannot be swapped under a captured signature.

The proof is bound to the request that fetched the directory:

~~~
GET /.well-known/http-message-signatures-directory HTTP/1.1
Host: signature-agent.test
Accept: application/http-message-signatures-directory+json
~~~

The response body is the following JSON Web Key Set, signed exactly as shown,
with no trailing newline:

~~~
NOTE: '\' line wrapping per RFC 8792

{"keys":[{"kty":"OKP","crv":"Ed25519","kid":"poqkLGiymh_W0uP6PZFw-\
 dvez3QJT5SolqXBCW38r0U","x":"JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw\
 3c5D0bs","use":"sig"}]}
~~~

Those bytes give the following `Content-Digest` field value:

~~~
Content-Digest: sha-256=:CADMT2aBdV/rqQr/NIru64ERQkCobVvllA4V0fLFDu0=:
~~~

The corresponding signature base is:

~~~
NOTE: '\' line wrapping per RFC 8792

"@authority";req: signature-agent.test
"content-digest": sha-256=:CADMT2aBdV/rqQr/NIru64ERQkCobVvllA4V0fLFDu0=:
"@signature-params": ("@authority";req "content-digest")\
 ;created=1735689600\
 ;expires=4889289600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;tag="http-message-signatures-directory"
~~~

This results in the following Content-Digest, Signature-Input and Signature
header fields being added to the response under the label `binding`:

~~~
NOTE: '\' line wrapping per RFC 8792

Content-Digest: sha-256=:CADMT2aBdV/rqQr/NIru64ERQkCobVvllA4V0fLFDu0=:
Signature-Input: binding=("@authority";req "content-digest")\
 ;created=1735689600\
 ;expires=4889289600\
 ;keyid="poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U"\
 ;tag="http-message-signatures-directory"
Signature: binding=:l6P8R67tm3kujAxbHWio7ll01qrEZ0dKD/WWlGhNYEmTnFZM8Wt0VQ9zqGfvo7T/UMkBxsigzChM1Gpz7gOVBg==:
~~~

# Implementations

This draft has a couple of public implementations. A demonstration server has been deployed to [https://http-message-signatures-example.research.cloudflare.com/](https://http-message-signatures-example.research.cloudflare.com/).

It uses ed25519 example signing and verifying keys defined in {{Appendix B.1.4 of HTTP-MESSAGE-SIGNATURES}}.

## Clients

draft-meunier-webbotauth-httpsig-protocol-00

* [Chrome MV3](https://github.com/cloudflare/web-bot-auth) (TypeScript)

* [Cloudflare Workers](https://github.com/cloudflare/web-bot-auth) (TypeScript)

* [Rust binaries](https://github.com/cloudflare/web-bot-auth) (Rust)

draft-meunier-web-bot-auth-architecture-03

* [Puppeteer script](https://github.com/stytchauth/web-bot-auth-example) (JavaScript)

* [Guzzle middleware](https://github.com/olipayne/guzzle-web-bot-auth-middleware) (PHP)

* [Python script](https://zenn.dev/oymk/articles/944069e5eddc27) (Python)

* [Bot-Authentication](https://github.com/cyberstormdotmu/bot-authentication) (Python)

* [HTTPie plugin](https://github.com/cloudflare/web-bot-auth) (Python)

* [Web scrapers (scrapy/crawl4ai)](https://github.com/cyberstormdotmu/bot-authentication) (Python)

* [HUMAN Verified AI Agents](https://github.com/HumanSecurity/human-verified-ai-agent) (Python)

* [Linzer](https://github.com/nomadium/linzer/blob/master/spec/integration/cloudflare_example_research_spec.rb) (Ruby)

## Servers

draft-meunier-webbotauth-httpsig-protocol-00

* [Cloudflare Workers](https://github.com/cloudflare/web-bot-auth) (TypeScript)

draft-meunier-web-bot-auth-architecture-03

* [Caddy plugin](https://github.com/cloudflare/web-bot-auth) (Go)

* [Apache module](https://github.com/garyillyes/web-bot-auth-apache) (C)

## Test Vectors

* In [JSON format](https://github.com/cloudflare/web-bot-auth/blob/main/packages/web-bot-auth/test/test_data/web_bot_auth_architecture_v2.json)


# Acknowledgments
{:numbered="false"}

The editor would also like to thank the following individuals (listed in alphabetical order) for feedback, insight, and implementation of this document -
Marwan Fayed,
Maxime Guerreiro,
Scott Hendrickson,
Jonathan Hoyland,
Nikhil Kandoi,
Akshat Mahajan,
Mark Nottingham,
Eugenio Panero,
Lucas Pardue,
Malte Ubl,
Loganaden Velvindron,
Tanya Verma.

{::comment}
Pending permission to acknowledge. Add to the list above once each has agreed.
Richard Barnes,
Max Gerber,
Dick Hardt,
Dennis Jackson,
Blake Morrison,
Kaveh Ranjbar,
Eric Rescorla,
Justin Richer,
Martin Thomson.
{:/comment}

# Changelog
{:numbered="false"}

draft-ietf-webbotauth-httpsig-protocol-01

- Make every `Signature-Agent` value a direct JWK Set URI and prohibit member
  parameters. Remove the `directory`, `jwks_uri`, and `cimd` discovery types.

draft-meunier-webbotauth-httpsig-protocol-02

- Require `Signature-Agent` on every signed request, and require each signature
  to cover the member keyed to its own label. Drop the test vectors that omitted
  the header. Recast the no-URL identifier as a verifier state, not an Agent
  mode.
- Defer `Signature-Key` to draft-hardt-httpbis-signature-key.
- List the components an Agent may cover beyond the required set.
- Drop the 2119 keywords from the human binding guidance in Privacy
  Considerations.
- Require discovery to return `200 (OK)` and prohibit redirects.
- Make directory response signatures optional for directly resolved keys.
- Defer replay protection, including nonce handling, to
  {{HTTP-MESSAGE-SIGNATURES}}.
- Add a signed directory response test vector.
- Redraw the figures. The overview figure now shows the architecture, naming
  who publishes and who resolves, rather than a message order, and places the
  Agent inside the User. The sending figure shows key resolution following the
  first request, conditional on the keys not already being resolved, and the
  request example moves out of it.

draft-meunier-webbotauth-httpsig-protocol-01

- Add an Identifiers and Trust Model section: opaque and domain binding modes.
- Describe the document as a protocol throughout (was: architecture).
- Fold `draft-meunier-webbotauth-httpsig-directory` into this document with its
  IANA registrations, and move to Standards Track.
- Anchor identity on the resolved `Signature-Agent` URL rather than the key.
  Rotation is the same URL serving a new key; the thumbprint identifies only
  when no URL is sent. A `directory` value is an origin, identifiers are
  normalized before comparison, and `kid` equals the key thumbprint.
- Define attribution: lookup on the (URL, key) pair, what redistributed key
  material must carry, which source wins when two disagree, what a failed
  resolution means, a bound on the resolution behind it, and rejection
  of directory response signatures dated in the future.
- When chaining, an outer signature covering an inner `signature` MUST also
  cover its `signature-input`, `signature-agent`, and every component the inner
  signature covered. Verifiers validate each signature independently.
- Say what a signature does not reach: not the body without `Content-Digest`,
  and not the method or path when only `@authority` is covered. Correct the
  relabeling guidance, and note how to migrate from the sf-string form.
- Move the domain binding to an appendix. Add objectives, the relationship with
  anonymous bot authentication, a use case to identifier mapping, what an
  unsigned request tells a verifier, and the worst case for a compromised key.
- Note how field compression treats reused and per-request signatures, and why
  that is not a reason to widen coverage.

draft-meunier-webbotauth-httpsig-protocol-00

- Rename draft from `draft-meunier-web-bot-auth-architecture`.
- Add SSRF guidance for `Signature-Agent` directory fetches.
- Add deployment guidance for verifier outcomes, directory fetches, caching,
  retry, rollout, proxies, CORS, and observability.
- Add guidance for test keys, static signatures, and discovery failures.
- Add multiple Web Bot Auth signatures and an example.
- Add typed `Signature-Agent` discovery examples for `directory`, `jwks_uri`,
  and `cimd`.
- Group implementations by the draft version that added them.
- Clarify that `Signature-Input` `keyid` selects the key and `Signature-Agent`
  points to candidate key material.
- Note `Signature-Key` as an optional discovery header.
- Align examples with published test-vector fixtures.
- Fix typos.

draft-meunier-web-bot-auth-architecture-05

- Add Sandor Major as an author.
- Add session protocol considerations.
- Update HTTP Message Signatures test vectors.
- Keep legacy `Signature-Agent` string examples for implementers migrating to
  dictionary members.

draft-meunier-web-bot-auth-architecture-04

- Change `Signature-Agent` to a Structured Fields dictionary.
- Add a security consideration for intermediaries that relabel
  `Signature-Agent` members.
- Allow `@target-uri` as a replacement for `@authority`.
- Add contributors.
- Add implementations.
- Remove the `purpose` field from the Web Bot Auth example.

draft-meunier-web-bot-auth-architecture-03

- Update the Linzer example URL.
- Fix the section reference and name for status code 429.
- Fix typos.

draft-meunier-web-bot-auth-architecture-02

- Add response status codes.
- Add references for readability.
- Add text about signing extra headers.
- Add TLS guidance to Security Considerations.
- Add RSASSA-PSS examples.
- Update acknowledgments.
- Add PHP, Python, Ruby, and Rust implementations.
- Fix `Signature-Agent` in the architecture diagram to use Structured Fields.
- Fix test vectors to use Structured Fields for `Signature-Agent`.
- Fix typos.

draft-meunier-web-bot-auth-architecture-01

- Require clients to sign `Signature-Agent` when it is present.
- Add test vectors for requests with and without `Signature-Agent`.
- Fix the example diagram.
- Add reverse proxy security considerations.
- Update text about why an origin may request a new signature.
- Update nonce validation wording and uniqueness requirements.
- Add acknowledgments.

draft-meunier-web-bot-auth-architecture-00

- Initial draft.
- Describe how to use HTTP Message Signatures to sign requests.
- Describe signature verification.
- Define the `web-bot-auth` tag.
- Derive `keyid` from the JWK Thumbprint.
- Add initial Security and Privacy Considerations.
