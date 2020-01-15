# Client Conformance Profiles

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-11612](https://issues.jboss.org/browse/KEYCLOAK-11612)

## Motivation

To protect APIs that require high level security like providing financial services, there are several API security profiles like Financial-grade API Security Profile.

To enforce the client operating in complying with such the security profile, the client conformance profiles are introduced. 

## Overview

The client conformance profile's tasks are the following.

* Client's Configuration Check

When registering the newly client or configuring the client that has already been registered, the client conformance profile checks whether the configuration of the client complies with the applied security profile.

If not, keycloak rejects this registration/configuration and returns the appropriate error.

* Client's Request Check

When the client sends the request, the client conformance profile checks whether this request complies with the applied security profile.

If not, keycloak rejects this request and returns the appropriate error.

* Security Profile Specific Operation on the Client's Request

Depending on the security profile, keycloak needs to do this profile specific operation when it receives the request from the client.

## Scope

### Target Security Profile

The following security profiles are to be supported.

* [FAPI (Financial-grade API) Read Only API Security Profile](
https://openid.net/specs/openid-financial-api-part-1-ID2.html)
* [FAPI (Financial-grade API) Read and Write API Security Profile](https://openid.net/specs/openid-financial-api-part-2-ID2.html)

The following FAPI based security profiles might be supported in the future.

* UK : [OpenBanking Security Profile](
https://openbanking.atlassian.net/wiki/spaces/DZ/pages/7046134/Open+Banking+Security+Profile+-+Implementer+s+Draft+v1.1.0)
* Australia : [CDR (Consumer Data Rights) Security Profile](
https://consumerdatastandardsaustralia.github.io/standards/#security-profile)


## Design

### Scope of client conformance profile

Keycloak can select and set the client conformance profile per client.

### The number of applicable client conformance profile simultaneously

Keycloak can set only one client conformance profile to the client.

### When and how keycloak selects and sets the client conformance profile to the client

The administrator must be involved in the process of determining and setting the client conformance profile to the client.

It must be prohibited for the client itself to determine and set the client conformance profile by itself, without the administrator's involvement.

The ways of client registration are as follows.

* Dynamic Client Registration Endpoint
    - Keycloak Client Representation
    - OpenID Connect Client Metadata Description

* Client Registration CLI

* Admin Console
    - UI
    - CLI
    - REST API directly
    - Import
 
The following 2 ways of client registrations can only determine and set the client conformance profile to the client.

* Dynamic Client Registration Endpoint
    - OpenID Connect Client Metadata Description
        * Use Initial Access Token

When creating the initial access token on the admin console, keycloak provides the pull-down menu for specifying the client conformance profile. The selected client conformance profile is included onto the initial access token to convey it to dynamic client registration endpoint afterwards.

* Admin Console
    - UI

When creating the client on the admin console, keycloak provides the pull-down menu for specifying the client conformance profile.

### When and how keycloak changes the client conformance profile to the client that has already been registered

The administrator must be involved in the process of changing the client conformance profile to the client.

It must be prohibited for the client itself to change the client conformance profile by itself, without the administrator's involvement.

The ways of client configuration are as follows.

* Dynamic Client Registration Endpoint
    - Keycloak Client Representation
    - OpenID Connect Client Metadata Description

* Client Registration CLI

* Admin Console
    - UI
    - CLI
    - REST API directly
    - Import

The following 1 way of client configuration can change the client conformance profile to the client.

* Admin Console
    - UI

### Client's Configuration Check

Keycloak needs to do Client's Configuration Check at the following points.

* Dynamic Client Registration Endpoint
    - Keycloak Client Representation
    - OpenID Connect Client Metadata Description

* Client Registration CLI

* Admin Console
    - UI
    - CLI
    - REST API directly
    - Import

### Client's Request Check

Keycloak needs to do Client's Request Check at the following points.

* Authorization Request
* Token Request
* Token Refresh Request
* Token Introspection Request
* UserInfo Request
* (Back Channel) Logout Request

### Security Profile Specific Operation on the Client's Request

Depending on applied security profile, keycloak needs to do Security Profile Specific Operation on the Client's Request at the following points.

* Authorization Response
* Token Response
* Token Refresh Response
* Token Introspection Response
* UserInfo Response
* (Back Channel) Logout Response

### Client Conformance Profiles SPI

Depending on applied security profile, keycloak needs to process Client's Configuration Check, Client's Request Check, and Security Profile Specific Operation on the Client's Request.

To realize it, Client Conformance Profile SPI is introduced. Its provider implementations are also provided per client conformance profile. 

## Implementation

### Plan

Ph 1. Introducing Client Conformance Profile Mechanism (SPI Provider)

Ph 2. Supporting FAPI Read Only Security Profile (Implementing Provider)

Ph 3. Supporting FAPI Read and Write Security Profile (Implementing Provider)

Ph (TBD). Supporting OpenBanking Security Profile

Ph (TBD). Supporting CDR Security Profile

### FAPI Read Only API Security Profile

#### Endpoint : Dynamic Client Registration Endpoint

Client's Configuration Check

* Operations :
    - Registering the client
    - Configuring the existing client

##### 1. Client Authentication Method
OpenID Connect Client Metadata - `token_endpoint_auth_method` is one of the following values.
 
 1. `tls_client_auth_subject_dn` (currently, keycloak only supports it for Mutual TLS for OAuth Client Authentication[MTLS])
 2. `client_secret_jwt`
 3. `private_key_jwt`

* Reference:
    - Authz Server / 4, 4-1, 4-2
    - Confidential Client / 1, 1-1, 1-2

Error Response : 400 Bad Request (the detail is TBD)

##### 2. Proof Key for Conde Exchange [PKCE]
Set the client's `code_challenge_method` to `S256` to enforce it [PKCE].

* Reference:
    - Authz Server / 7
    - Public Client / 1, 2

Error Response : -

##### 3. Redirect URI
OpenID Connect Client Metadata - `redirect_uris` includes at least one URI.

All URIs in `redirect_uris` do not include any wildcard, only specifying one URI.

The scheme of all URIs in `redirest_uris` are https.

* Reference:
    - Authz Server / 8
    - Authz Server / 20

Error Response : 400 Bad Request (the detail is TBD)

#### Endpoint : Authorization Endpoint

Client's Request Check

* Operations :
    - handle authorization request

##### 1. Proof Key for Conde Exchange [PKCE]
The authorization request includes `code_challenge` and `code_challenge_method`.

`code_challenge_method` is `S256`.

The value of `code_challenge` follows [PKCE].

* Reference:
    - Authz Server / 7
    - Public Client / 1, 2

Error Response : to the client (the detail follows [PKCE])

##### 2. Redirect URI
The authorization request includes only one `redirect_uri`.

The value of `redirect_uri` exactly matches one of the values in pre-registered `redirect_uris` of OpenID Connect Client Metadata

* Reference:
    - Authz Server / 9, 10

Error Response : to the user's browser (TBD)

##### 3. Authz Request Parameters for Security
The authorization request's `scope` includes `openid`.

If so, the authorization request includes `nonce`.

If not, the authorization request includes `state`.

* Reference:
    - Public Client / 7, 8, 9

Error Response : to the client (the detail is TBD)

#### Endpoint : Token Endpoint

Client's Request Check

* Operations :
    - handle token request

##### 1. Client Authentication Method
The client challenges its authentication by the method pre-registered in `token_endpoint_auth_method` of OpenID Connect Client Metadata.

* Reference:
    - Authz Server / 4, 4-1, 4-2
    - Confidential Client /1, 1-1. 1-2

Error Response : 400 Bad Request (the detail is TBD)

##### 2. Proof Key for Conde Exchange [PKCE]
The token request includes `code_verifier`.

The value of `code_verifier` follows [PKCE].

The `code_verifier` is verified using `code_challenge` by following [PKCE].

* Reference:
    - Authz Server / 7
    - Public Client / 1, 2

Error Response : 400 Bad Request (the detail follows [PKCE])

#### Endpoint : Token Introspection Endpoint

Client's Request Check

* Operations :
    - handle token introspection request

##### 1. Client Authentication Method
The client challenges its authentication by the method pre-registered in `token_endpoint_auth_method` of OpenID Connect Client Metadata.

* Reference:
    - Authz Server / 4, 4-1, 4-2
    - Confidential Client /1, 1-1. 1-2

Error Response : 400 Bad Request (the detail is TBD)

#### Endpoint : Logout Endpoint

Client's Request Check

* Operations :
    - handle (backchannel) logout request

##### 1. Client Authentication Method
The client challenges its authentication by the method pre-registered in `token_endpoint_auth_method` of OpenID Connect Client Metadata.

* Reference:
    - Authz Server / 4, 4-1, 4-2
    - Confidential Client /1, 1-1. 1-2

Error Response : 400 Bad Request (the detail is TBD)

### FAPI Read and Write API Security Profile

Besides following FAPI Read Only Security Profile, FAPI Read and Write API Security requires to satisfy the follwing points additionaly.

#### Endpoint : Dynamic Client Registration Endpoint

Client's Configuration Check

* Operations :
    - Registering the client
    - Configuring the existing client

##### 1. Exclude Public Client
OIDC Client Metadata - `grant_types` does not include `code` and include `token` or `id_token`. (which means implicit flow)

If so, keycloak concludes that the client is Public Client. Keycloak refuses Public Client.

FAPI Read and Write API Security Profile does not exclude the public client. Such the public client needs to support the Holder-of-Key Token. It is plausible for the public client to use OAuth2 Token Binding [OAUTB] to realize Holder-of-Key Token.

However, keycloak does not support [OAUTB] so that keycloak excludes the public client for applying FAPI Read and Write API Security Profile.

* Reference:
    - N/A

Error Response : 400 Bad Request (the detail is TBD)

##### 2. Holder-of-Key Token
OIDC Client Metadata - `tls_client_certificate_bound_access_tokens` is true.

* Reference:
    - Authz Server / 5, 6
    - Public Client / 1
    - Confidential Client / 1

Error Response : 400 Bad Request (the detail is TBD)

##### 3. Excluding PKCE
After confirming that the client to be registered is not Public Client, keycloak exclude the setting of PKCE on this client by removing `code_challenge_method`.

* Reference:
    - Authz Server / 12

Error Response : -

##### 4. Client Authentication Method
OpenID Connect Client Metadata - `token_endpoint_auth_method` is one of the following values.
 
 1. `tls_client_auth_subject_dn` (currently, keycloak only support it for Mutual TLS for OAuth Client Authentication[MTLS])
 2. `private_key_jwt`

* Reference:
    - Authz Server / 14, 14-1, 14-2

Error Response : 400 Bad Request (the detail is TBD)

##### 5. ID Token Signing Algorithm
OpenID Connect Client Metadata - `id_token_signed_response_alg` is not `none`. (not defined is also OK)

* Reference:
    - Public Client / 4

Error Response : 400 Bad Request (the detail is TBD)

##### 6. Secure JWS Algorithm Consideration
OpenID Connect Client Metadata - `id_token_signed_response_alg` is `ES256`, `ES384`, `ES512`, `PS256`, `PS384` or `PS512`. (not defined is also OK)

OpenID Connect Client Metadata - `request_object_signing_alg` is `ES256`, `ES384`, `ES512`, `PS256`, `PS384` or `PS512`. (not defined is also OK)

If OpenID Connect Client Metadata - `token_endpoint_auth_method` is `private_key_jwt`, OpenID Connect Client Metadata - `token_endpoint_auth_signing_alg` is `ES256`, `ES384`, `ES512`, `PS256`, `PS384` or `PS512`. (not defined is also OK)

* Reference:
    - JWS algorithm considerations / 1, 2, 3

Error Response : 400 Bad Request (the detail is TBD)

#### Endpoint : Authorization Endpoint

Client's Request Check

* Operations :
    - handle authorization request

##### 1. Request Object
The authorization request includes `request` or `request_uri`. (none included or both included is NG)

If `request_uri` is included, the scheme of the value of `request_uri` is https.

The request object from `request` or `request_uri` includes JWS signature.

Its signature algorithm is `ES256`, `ES384`, `ES512`, `PS256`, `PS384` or `PS512`.

* Reference:
    - Authz Server / 1
    - Public Client / 2
    - JWS algorithm considerations / 1, 2, 3

Error Response : to the client (the detail is TBD)

##### 2. Response Type (Hybrid)
The authorization request includes `response_type`.

The value of `response_type` is `code id_token` or `code id_token token`.

* Reference:
    - Authz Server / 2

Error Response : to the client (the detail is TBD)

##### 3. Request Object Verification
The request object includes all parameters that the authorization request includes. (except for `request` and `request_uri`).

The request object includes `"exp"` claim.

* Reference:
    - Authz Server / 10
    - Authz Server / 13
    - Public Client / 8, 9

Error Response : to the client (the detail is TBD)

#### Endpoint : Token Endpoint

Client's Request Check

* Operations :
    - handle token request

##### 1. Client Authentication Method
The client challenges its authentication by the method pre-registered in `token_endpoint_auth_method` of OpenID Connect Client Metadata (`tls_client_auth_subject_dn` or `private_key_jwt`).

If `private_key_jwt`, the signature algorithm of JWS Client Assertion is `ES256`, `ES384`, `ES512`, `PS256`, `PS384` or `PS512`.

* Reference:
    - Authz Server / 14, 14-1, 14-2
    - JWS algorithm considerations / 1, 2, 3

Error Response : 400 Bad Request (the detail is TBD)

##### 2. Holder-of-Key Token
In TLS handshake, the client is authenticated by its PKI X.509 Certificate.

Keycloak binds this certificate with an access token and a refresh token by following Mutual-TLS Client Certificate-Bound Access Tokens [MTLS].

* Reference:
    - Authz Server / 5, 6
    - Public Client / 1
    - Confidential Client / 1

Error Response : 400 Bad Request (the detail is TBD)

#### Endpoint : Token Introspection Endpoint

Client's Request Check

* Operations :
    - handle token introspection request

##### 1. Client Authentication Method
The client challenges its authentication by the method pre-registered in `token_endpoint_auth_method` of OpenID Connect Client Metadata (`tls_client_auth_subject_dn` or `private_key_jwt`).

If `private_key_jwt`, the signature algorithm of JWS Client Assertion is `ES256`, `ES384`, `ES512`, `PS256`, `PS384` or `PS512`.

* Reference:
    - Authz Server / 14, 14-1, 14-2
    - JWS algorithm considerations / 1, 2, 3

Error Response : 400 Bad Request (the detail is TBD)

#### Endpoint : Logout Endpoint

Client's Request Check

* Operations :
    - handle (backchannel) logout request

##### 1. Client Authentication Method
The client challenges its authentication by the method pre-registered in `token_endpoint_auth_method` of OpenID Connect Client Metadata (`tls_client_auth_subject_dn` or `private_key_jwt`).

If `private_key_jwt`, the signature algorithm of JWS Client Assertion is `ES256`, `ES384`, `ES512`, `PS256`, `PS384` or `PS512`.

* Reference:
    - Authz Server / 14, 14-1, 14-2
    - JWS algorithm considerations / 1, 2, 3

Error Response : 400 Bad Request (the detail is TBD)

##### 2. Holder-of-Key Token
In TLS handshake, the client is authenticated by its PKI X.509 Certificate.

Keycloak verifies whether the refresh token is bound with this certificate by following Mutual-TLS Client Certificate-Bound Access Tokens [MTLS].

* Reference:
    - Authz Server / 5, 6
    - Public Client / 1
    - Confidential Client / 1

Error Response : 400 Bad Request (the detail is TBD)

#### Endpoint : UserInfo Endpoint

Client's Request Check

* Operations :
    - handle UserInfo request

##### 1. Holder-of-Key Token
In TLS handshake, the client is authenticated by its PKI X.509 Certificate.

Keycloak verifies whether the access token is bound with this certificate by following Mutual-TLS Client Certificate-Bound Access Tokens [MTLS].

* Reference:
    - Authz Server / 5, 6
    - Public Client / 1
    - Confidential Client / 1

Error Response : 400 Bad Request (the detail is TBD)

## TODO

### "acr" (Authentication Context Class Reference) treatment

FAPI security profile includes "acr" related items. However, keycloak has not yet determined what and when "acr" value is set.
It has already been discussed in [KEYCLOAK-3314](https://issues.redhat.com/browse/KEYCLOAK-3314), but still open.

### track update of FAPI Security Profile

FAPI Read Only API Security Profile and FAPI Read and Write API Security profile are still in Implementer's Draft phase. Therefore, we need to track update of them.
