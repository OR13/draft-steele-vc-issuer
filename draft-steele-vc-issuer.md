---
title: "VC Issuer"
category: info

docname: draft-steele-vc-issuer-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
keyword:
 - verifiable credentials
 - decentralized identifiers
 - http apis
venue:
  github: "OR13/draft-steele-vc-issuer"
  latest: "https://OR13.github.io/draft-steele-vc-issuer/draft-steele-vc-issuer.html"

author:
 -
    fullname: "Orie Steele"
    organization: Transmute
    email: "orie@transmute.industries"

normative:
  RFC7518:
  RFC9562:
  RFC7519:
  RFC7800:
  RFC9449:

informative:
  W3C-VC:
    title: "Verifiable Credentials Data Model v2.0"
    target: https://www.w3.org/TR/vc-data-model-2.0/
  Secure-VC:
    title: "Securing Verifiable Credentials using JOSE and COSE"
    target: https://www.w3.org/TR/vc-jose-cose/
  OpenID4VC:
    author:
      org: "OpenID Foundation"
    title: "OpenID for Verifiable Credential Presentation"
    target: "https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0-ID1.html"
---

--- abstract

This document describes a protocol for issuing Verifiable Credentials using HTTP.

--- middle

# Introduction

The Verifiable Credentials (VC) data model is described in {{W3C-VC}}, and securing mechanisms based on JOSE and COSE are described in {{Secure-VC}}.

Although HTTP based protocols exist for issuing Verifiable Credentials, such as {{OpenID4VC}}, they are encumbered by the conventions and history of OAuth and the experience needed to deploy OpenID successfully.

This document describes an HTTP API for VC Issuers, that provides interoperability while limiting normative dependencies, optionality and excessive cryptographic agility.

The API this document specifies is designed with priority for use by organizations or businesses that need to request credentials from other organizations or governments.

Although this API describes HTTP resources which require authentication, a specific authentication mechanism is not specified.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The terms issuer, subject and holder are defined in {{W3C-VC}}.

The terms `iat`, `exp`, `iss`, `sub`, `aud`, are defined in {{RFC7519}}.

The term `nonce` is defined in {{RFC9449}}.

The term `cnf` is defined in {{RFC7800}}.

The term `ES256` is a digital signature algorithm described in {{Section 3.4 of RFC7518}}.

This document uses "..." to ellide text in examples for readability.

# Mandatory to Implement

Although {{Secure-VC}} describes several different media types, and each media type can be secured with many different signing or encryption algorithms, this documents specifies the following as mandatory to implement:

The following media types MUST be supported as JWT Claim Sets in the request body of the issuance API:

- application/vc

The following media types MUST be supported in the response body of the issuance and read APIs:

- application/vc+jwt

The following JSON Web Signature algorithms MUST be supported:

- ES256

This document might be updated in the future to support additional media types or signing algorithms.

# Issuer Resources

## Credentials

The `/credentials` resource describes the collection of verifiable credentials associated with an issuer.

This collection supports creating, and retrieving resources.

### Create

Creating a verifiable credential requires several individual steps to be completed.

- The server MUST validate the request body, ensuring the JSON object is conformant to {{W3C-VC}}.
- The server MAY reject credentials that do not match specific schemas.
- The server SHOULD confirm any relationships between the claims requested and the identity of the subject initiating the request.
  For example, if Bob is requesting a name credential, the server should ensure that Bob is not able to receive a credential with the name Alice.
  A detailed analysis and set of recommendations regarding binding claims about subjects to credentials is beyond the scope of this document.
- The server MAY retrieve or compute additional claims related to the validity period of the credential and its status update mechanisms.
  For example, a Driver's License credential has a fixed validity period with a start and end date, and can be suspended or revoked.
  Other credentials might never expire, or might not support any status update claims.
- The server MUST validate the credential after applying all server side changes, to ensure it remains conformant to {{W3C-VC}}.
- The server MUST sign the response token with ES256.
- The server MAY overwrite any member of the request body.
- The server MAY inject additional JWT claims, such as `iat`, `exp`, `iss`, `sub`, and `cnf`, before securing the credential as a JWT.
- The server SHOULD assign an `id` value that is globally unique, such as a UUID as described in {{Section 5.4 of RFC9562}}.
- The server MAY create credentials related to the issuance request, such as credentials which support status assertions or status lists.

The following informative example demonstrates how HTTP resources and media types are related to the credential issuance process.

~~~

Request:

POST /credentials HTTP/1.1
Host: example.gov
Authorization: Bearer ey...
Content-Type: application/vc
Accept: application/vc+jwt

{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
  ],
  "type": [
    "VerifiableCredential",
    "ExampleCredentialType"
  ],
  "validFrom": "2019-12-11T03:50:55Z",
  "issuer": {
    "type": [
      "Organization"
    ],
    "id": "https://example.gov/issuer/42",
  },
  ...
}

Response:

HTTP/1.1 200 OK
Content-Type: application/vc+jwt

eyJhbGciOiJFZE...

~~~
{: #example-create-request align="left" title="Issuing a Credential"}


## Confirmation Methods

An important step in the issuance of a credential is for the issuer to confirm the subject of the credential controls a public key or cryptographic identifier the credential will be bound to.

Confirmation methods are described in detail in {{RFC7800}}.

This section describes a simple HTTP API for convincing an issuer to accept a confirmation claim in a requested credential issuance.

The holder first obtains a `nonce` from the issuer, and then presents a confirmation token, that proves possession of a given key or cryptographic identifier, to a given audience, as part of a credential issuance request.

The following informative example demonstrates one way this can be accomplished:

~~~

Request:

POST /nonce HTTP/1.1
Host: example.gov
Content-Length: 0
Authorization: Bearer ey...
Accept: application/json

Response:

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "c_nonce": "wKI4LT17ac15ES9bw8ac4",
  "c_nonce_expires_in": 120
}

~~~
{: #example-nonce-request align="left" title="Requesting a Nonce"}

To create the confirmation token, the holder signs a key or key reference in the protected header, and the nonce and an audience in the payload:

~~~
{
  "typ": "subject-confirmation+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "nUWAoAv3XZith8E7i19OdaxOLYFOwM-Z2EuM02TirT4",
    "y": "HskHU8BjUi1U9Xqi7Swmj8gwAK_0xkcDjEW_71SosEY"
  }
}.{
  "aud": "https://example.gov",
  "iat": 1701960444,
  "nonce": "LarRGSbmUPYtRYO6BQ4yn8"
}
~~~
{: #example-confirmation-jwt align="left" title="Confirmation Token"}

Note that the `typ` value in the protected header can be different or absent.

The `aud`, `iat` and `nonce` claims MUST be present in the payload.

Additional claims MAY be present, and MUST be ignored when not understood.

The confirmation token is then passed as a query parameter in the credential issuance request us the `cnft` query parameter.

~~~
Request:

POST /credentials?cnft=ey... HTTP/1.1
Host: example.gov
Authorization: Bearer ey...
Content-Type: application/vc
Accept: application/vc+jwt

{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
  ],
  "cnf": {
    "jkt":"0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I"
  },
  ...
}

Response:

HTTP/1.1 200 OK
Content-Type: application/vc+jwt

eyJhbGciOiJFZE...

~~~
{: #example-issuance-with-confirnmation-request align="left" title="Issuing Credential with Subject Confirmation"}

A server processes the `cnft` query parameter by verifying the signature on the confirmation token in the protected header, and then validating the `nonce`.
Validating the `nonce` can be handled different ways, including lookups in a stateful database or checking digital signatures.
A detailed description of nonce production and consumption is beyond the scope of this document, see {{OpenID4VC}} for a description of one way that nonces can be used during credential issuance.
Once the issuer is satisfied that the nonce is sufficiently fresh, the issuer checks that either the public key used to sign the confirmation token, or its thumbprint, is included in the credential.

The server MUST return a 400 error in the case that a `cnft` query parameter is sent, but no corresponding confirmation claim exists in the request body.

### Read

Retrieving a verifiable credential requires several steps to be completed.
The server MUST determine if the credential with the given `id` is available.
Some credentials types SHOULD require authorization, and other credential types SHOULD support anonymous access.
If the credential does not support anonymous access, the server MUST ensure the client making the request is authorized to read the credential.
A detailed analys of credential types and their associated requirements is beyond the scope of this document.

The following informative example demonstrates how HTTP resources and media types are related to resolving or retrieving credentials.

~~~

Request:

GET /credentials/:id HTTP/1.1
Host: example.gov
Authorization: Bearer ey...
Accept: application/vc+jwt

Response:

HTTP/1.1 200 OK
Content-Type: application/vc+jwt

eyJhbGciOiJFZE...

~~~
{: #example-read-request align="left" title="Reading a Credential"}

# Security Considerations

## Private Keys

It is important to protect the signing key used to issue credentials, and well as any confirmation keys associated with the credentials.
It is RECOMMENDED that all private keys be initialized such that they cannot be exported.

## Authorization

HTTP Clients that request credential issuance MUST be authenticated.
A client may request credentials related to or bound to many different subjects.
A full description of policy for issuing credentials for subjects is beyond the scope of this document.

## HTTPS

HTTPS MUST be used with all resources described in this document.
Issuers MUST support at least TLS 1.3 or a more recent version.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
