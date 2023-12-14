# OAuth 2.0 Demonstrating Proof-of-Possession at the Application Layer (DPoP)

* **Status**: Notes
* **JIRA**: [KEYCLOAK-15169](https://issues.redhat.com/browse/KEYCLOAK-15169)

## Motivation

DPoP is a mechanism to prevent illegal API calls from succeeding only with a stolen access token. This is done via augmenting the calls to OAuth 2.0 endpoints as well as resource servers with a special DPoP header that proves possession of a private key, and binding the issued tokens to that key. Please refer to the Resources section for more info on DPoP.    

## Implementation Details

DPoP is a cross-cutting concept that would affect multiple Keycloak components, namely core libraries, OAuth/OIDC endpoints, adapters, models and admin UI.

It is suggested that DPoP should be set up on a per-client basis, with the three options available that should be interpreted as follows:

|**Value** | **Keycloak Server** | **Keycloak Adapters** |
| --- | --- | --- |
| Required | OAuth endpoints (where applicable) should require a DPoP header and a DPoP-bound token; should return an error otherwise. | Protected resources should require a DPoP header and a DPoP-bound token, or should return an error otherwise. |
| Optional | _Token Endpoint_: if DPoP header is present, the returned token should be DPoP-bound and should contain a `cnf` claim with a JWK thumbprint; should be a regular token otherwise  <br/><br/>_UserInfo endpoint_: if the access token is DPoP-bound, a matching DPoP header should be required; no header should be required otherwise. | Protected resources should require a matching DPoP header if the supplied access token is DPoP-bound; should ignore the header otherwise. |
| Disabled | DPoP headers should be ignored; issued tokens shouldn’t contain JWK thumbprints. | DPoP headers and token JWK thumbprints should be ignored. |

Other DPoP configuration options should include:

| **Option** | **Description** |
| --- | --- |
| DPoP Proof Lifetime | Time window in which the respective DPoP proof JWT would be accepted |
| DPoP Allowed Clock Skew | To accommodate for clock offsets, the server MAY accept DPoP proofs that carry an `iat` time in the reasonably near future (e.g., a few seconds in the future). |

_([8.1. DPoP Proof Replay][4])_

### Endpoints

#### Authorization Endpoint

Binding tokens issued directly from the authorization endpoint has been intentionally considered out of scope for the main DPoP draft, and is defined by a separate draft (see Resources).
In order to support DPoP in implicit and hybrid flows, authorization endpoint must honor the `dpop` request parameter which would contain a DPoP proof.

Classes/methods affected:

* org.keycloak.protocol.oidc.endpoints.AuthorizationEndpoint

#### Token Endpoint

With DPoP enabled, token endpoint must understand the DPoP header and perform binding between the keypair presented thereby and the issued token.

As per the spec,

> To request an access token that is bound to a public key using DPoP,
   the client MUST provide a valid DPoP proof JWT in a "DPoP" header
   when making an access token request to the authorization server's
   token endpoint.  This is applicable for all access token requests
   regardless of grant type (including, for example, the common
   "authorization\_code" and "refresh\_token" grant types but also
   extension grants such as the JWT authorization grant \[RFC7523\]).

_([5.5. DPoP Access Token Request][6])_

First and foremost, the `authorization_code` and `refresh_token` grant types must be supported; support for other grant types (ROPC, client credentials, token exchange, extension grants) could be added later.

Classes/methods affected:

* org.keycloak.protocol.oidc.endpoints.TokenEndpoint
  * processGrantRequest()

#### Introspection Endpoint

For DPoP-bound tokens, token introspection endpoint must return an additional `cnf` claim with the `jkt` member containing JWK thumbprint (a SHA-256 hash).

_([6.2. JWK Thumbprint Confirmation Method in Token Introspection][8])_

Classes/methods affected:

* org.keycloak.protocol.oidc.AccessTokenIntrospectionProvider
  * introspect()

#### Metadata Endpoint

With DPoP enabled, a new parameter, `dpop_signing_alg_values_supported`, must be included into the authorization server metadata.

_([5.1. Authorization Server Metadata][10])_

Classes/methods affected:

* org.keycloak.protocol.oidc.OIDCWellKnownProvider
	* getConfig()
* org.keycloak.protocol.oidc.representations.OIDCConfigurationRepresentation

#### UserInfo Endpoint

The spec doesn’t directly mention the UserInfo endpoint, but it clearly falls into the category of bearer token protected resources:

> The UserInfo Endpoint is an OAuth 2.0 Protected Resource that returns Claims about the authenticated End-User. To obtain the requested Claims about the End-User, the Client makes a request to the UserInfo Endpoint using an Access Token obtained through OpenID Connect Authentication.

_(OpenID Connect Core 1.0,_ [_5.3. UserInfo Endpoint_][11]_)_

Therefore, UserInfo endpoint must also be made DPoP-aware.

Classes/methods affected:

* org.keycloak.protocol.oidc.endpoints.UserInfoEndpoint
	* issueUserInfo()

#### Logout Endpoint

The `POST` variant of logout invocation uses refresh token, which could be DPoP-bound. Therefore, Logout endpoint must also be made DPoP-aware.

Classes/methods affected:

* org.keycloak.protocol.oidc.endpoints.LogoutEndpoint
	* logoutToken()

#### Other Endpoints

The following endpoints in Keycloak are public:

* Server Discovery
* Server JWK set
* Authorization

The following endpoints don’t use bearer tokens, but rather client credentials:

* Token Introspection
* Token Revocation

The following endpoints use cookie authentication:

* Check Session iframe

Therefore, no other endpoints should be affected.

#### Client Registration
Client registration should support DPoP-related parameters inside client metadata.

Classes/methods affected:

* org.keycloak.services.clientregistration.oidc.DescriptionConverter
* org.keycloak.representations.oidc.OIDCClientRepresentation

### Models

#### Access Token

New property, `jkt`, should be recognized as a member of the `cnf` claim.

Classes/methods affected:

* org.keycloak.representations.AccessToken.CertConf
	* rename to Conf or Confirmation; introduce `keyThumbprint` property

#### Client

There are the following DPoP-related client settings:

* DPoP mode (enabled/optional/disabled)
* DPoP proof lifetime (integer)
* DPoP clock skew (integer)

Those could be backed by client attributes, hence no changes to the client model and DB schema.

Classes/methods affected:

* org.keycloak.protocol.oidc.OIDCAdvancedConfigWrapper

### Adapters

#### Java Adapters

In the Bearer only mode, Java adapters must:

* enforce DPoP proof validation in the “Enabled/Required” mode;
* allow for DPoP validation of DPoP-bound tokens in the “Optional” mode;
* ignore DPoP proofs in the “Disabled” mode.

The following options should be exposed in the adapter configuration:

* DPoP Mode: required/optional/disabled
* DPoP Proof Lifetime
* DPoP Allowed Clock Skew

> :question: Should we support DPoP for the “traditional” (monolithic) web applications, where adapters themselves handle Authorization Code flow, and the tokens are not exposed to the user agent?

Classes/methods affected:

*   org.keycloak.adapters.\*

##### Application Clustering

As per the spec,

> Servers SHOULD store, in the context of the request URI, the `jti` value of each DPoP proof for the time window in which the respective DPoP proof JWT would be accepted and decline HTTP requests to the same URI for which the `jti` value has been seen before.

> :question: How do we implement DPoP proof replay protection in a clustered environment?
>
> Currently, only SAML adapters for Wildfly/AS/JBossWeb support distributed sessions via Infinispan.

#### JavaScript Adapter

In addition to DPoP handling proper, JavaScript adapter should also expose a method that would allow developers to obtain the DPoP proof header, since the latter would be required to access protected resources. As DPoP proof is bound to a particular HTTP method and URL, it should be recomputed for every protected resource request.

Authors of [axios-keycloak][12], [keycloak-angular][13] and similar HTTP interceptors should be notified so that DPoP support could be introduced in the respective projects.

Files affected:

*   adapters/oidc/js/src/main/resources/keycloak.d.ts

##### Keypair lifetime and refresh tokens
As per the spec, refresh tokens are also considered DPoP-bound. This implies that the entire chain of refresh tokens issued by Keycloak during the session would be bound to the same DPoP keypair. Hence, the keypair should be persisted in the user agent for the duration of the session; the same would be relevant for offline tokens. It should be emphasized that the longer the lifetime of the keypair, the higher the probability of it being leaked. For long-lived keypairs, secure storage should be preferred (e.g. non-extractable keys stored on a smartcard and accessed via WebCrypto API).

#### Other Adapters

Support in Node.JS adapter and mod\_authn\_oidc could be also considered.

### Admin UI

The following configuration options should be exposed in the Admin UI for OIDC clients:

* DPoP Mode: required/optional/disabled
* DPoP Proof Lifetime
* DPoP Allowed Clock Skew

Files/classes affected:

* org.keycloak.services.resources.admin.ClientResource
* org.keycloak.representations.idm.ClientRepresentation
* themes/src/main/resources/theme/base/admin/resources/partials/client-detail.html
* themes/src/main/resources/theme/base/admin/resources/js/controllers/clients.js

### Core

Methods should be added to support [JWK thumbprint (RFC 7638)][14] computation.

Classes affected:

* org.keycloak.jose.jwk.{JWK,RSAPublicJWK,ECPublicJWK}
	* add `getThumbprint()` method

### Tests

DPoP should be properly covered by unit and integration tests.

### Documentation

DPoP usage should be properly documented.

Affected documents:

*   Securing Applications and Services Guide

## Relation to other token binding mechanisms

In Keycloak, we already have [MTLS Holder-of-Key][16] token binding mechanism. Other mechanisms are emerging, like e.g. [OAuth Proof of Possession Tokens with HTTP Message Signatures][17].

Those mechanisms have a lot in common with DPoP, thus we should consider introducing Token Binding SPI, in order to avoid code duplication and make adding mechanisms easier.

Also it makes sense to allow user to choose only one token binding mechanism per client. This mutually-exclusive behavior would be also easier to implement with Token Binding SPI in place.

## Milestones
### M1 Core DPoP
* Models, libraries, endpoints, admin UI
* `authorization_code` and `refresh_token` grants

### M2 Adapters
* support in Java adapters
* support in JavaScript adapter

### M3 Token Binding SPI
* introduce Token Binding SPI
* port MTLS-HoK to the SPI
* port DPoP to the SPI

### M4 Advanced DPoP
* support for additional grant types:
	* client credentials
	* ROPC
	* token exchange
	* extension grants
	* UMA
* support DPoP optional mode
* support for hybrid/implicit flows
* support for WebCrypto API in JavaScript adapter

## Open Questions

1.  Which OAuth grant types should be DPoP-enabled?
2.  Should we support Java adapters' redirect mode (for monolithic webapps)?
3.  How do we implement adapter-side DPoP proof replay protection in a clustered environment?
4.  Should we generate a DPoP keypair per session, or should it be persisted in the user agent (or both)?
    1.  Should we use Web Crypto API to support non-extractable keys?
5.  What is the relationship between DPoP and UMA?

## Resources
* [OAuth 2.0 Demonstrating Proof-of-Possession at the Application Layer (DPoP)][1]
* [OAuth 2.0 DPoP for the Implicit Flow][15]
* [Illustrated DPoP (OAuth Access Token Security Enhancement)][2]

[1]: https://datatracker.ietf.org/doc/draft-ietf-oauth-dpop/
[2]: https://darutk.medium.com/illustrated-dpop-oauth-access-token-security-enhancement-801680d761ff
[4]: https://tools.ietf.org/id/draft-ietf-oauth-dpop-02.html#name-dpop-proof-replay
[6]: https://tools.ietf.org/id/draft-ietf-oauth-dpop-02.html#name-dpop-access-token-request
[8]: https://tools.ietf.org/id/draft-ietf-oauth-dpop-02.html#name-jwk-thumbprint-confirmation-
[10]: https://tools.ietf.org/id/draft-ietf-oauth-dpop-02.html#name-authorization-server-metada
[11]: https://openid.net/specs/openid-connect-core-1_0.html#UserInfo
[12]: https://github.com/herrmannplatz/axios-keycloak
[13]: https://github.com/mauriciovigolo/keycloak-angular
[14]: https://www.rfc-editor.org/rfc/rfc7638.html
[15]: https://datatracker.ietf.org/doc/html/draft-jones-oauth-dpop-implicit-00
[16]: https://issues.redhat.com/browse/KEYCLOAK-6771
[17]: https://datatracker.ietf.org/doc/draft-richer-oauth-httpsig/