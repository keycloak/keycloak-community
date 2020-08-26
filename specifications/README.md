Specifications
==============

## Implemented

List of OAuth, OpenID Connect and related specifications that are currently implemented fully or partially by Keycloak.
This is a work in progress and the list is incomplete.


### OpenID Connect


#### [OpenID Connect Core](http://openid.net/specs/openid-connect-core-1_0.html)

> This specification defines the core OpenID Connect functionality: authentication built on top of OAuth 2.0 and the use of Claims to communicate information about the End-User. It also describes the security and privacy considerations for using OpenID Connect.

Keycloak coverage: To be reviewed


#### [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html)

> This specification defines a mechanism for an OpenID Connect Relying Party to discover the End-User's OpenID Provider and obtain information needed to interact with it, including its OAuth 2.0 endpoint locations.

Keycloak coverage: To be reviewed


#### [OpenID Connect Dynamic Registration](http://openid.net/specs/openid-connect-registration-1_0.html)

> This specification defines how an OpenID Connect Relying Party can dynamically register with the End-User's OpenID Provider, providing information about itself to the OpenID Provider, and obtaining information needed to use it, including the OAuth 2.0 Client ID for this Relying Party.

Keycloak coverage: To be reviewed


#### [OpenID Connect Session Management](http://openid.net/specs/openid-connect-session-1_0.html)

> This document describes how to manage sessions for OpenID Connect, including when to log out the End-User.

Keycloak coverage: To be reviewed

Notes:
* Initiate logout endpoint does not follow specification, and can also be used in a non-standard way. Will be covered as 
  in [KEYCLOAK-12133](https://issues.redhat.com/browse/KEYCLOAK-12133)


### OAuth 2.0

#### [RFC 7662 Token Introspection](https://tools.ietf.org/html/rfc7662)

> This specification defines a method for a protected resource to query
   an OAuth 2.0 authorization server to determine the active state of an
   OAuth 2.0 token and to determine meta-information about this token.
   OAuth 2.0 deployments can use this method to convey information about
   the authorization context of the token from the authorization server
   to the protected resource.

Keycloak coverage: To be reviewed


#### [RFC 7009 Token Revocation](https://tools.ietf.org/html/rfc7009)

> This document proposes an additional endpoint for OAuth authorization
     servers, which allows clients to notify the authorization server that
     a previously obtained refresh or access token is no longer needed.
     This allows the authorization server to clean up security
     credentials.  A revocation request will invalidate the actual token
     and, if applicable, other tokens based on the same authorization
     grant.


#### [The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://tools.ietf.org/html/rfc6750)

> This specification describes how to use bearer tokens in HTTP
   requests to access OAuth 2.0 protected resources.  Any party in
   possession of a bearer token (a "bearer") can use it to get access to
   the associated resources (without demonstrating possession of a
   cryptographic key).  To prevent misuse, bearer tokens need to be
   protected from disclosure in storage and in transport.

### Misc

#### [JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)

> JSON Web Token (JWT) is a compact, URL-safe means of representing
     claims to be transferred between two parties.  The claims in a JWT
     are encoded as a JSON object that is used as the payload of a JSON
     Web Signature (JWS) structure or as the plaintext of a JSON Web
     Encryption (JWE) structure, enabling the claims to be digitally
     signed or integrity protected with a Message Authentication Code
     (MAC) and/or encrypted.

## Non-standard

List of non-standard approaches in Keycloak

* Admin endpoints - manage realms, clients, users, etc.
* Account endpoint - manage user account
* Logout endpoint - custom logout endpoint that is close to, but does not follow any specifications
* Admin adapter endpoints - ability to send not-before, logout and other "admin" events to adapters
* Cluster node registration - ability to register node instances for a client to send logout events to correct node in a cluster
* Permission endpoint - ability to retrive permissions for a user from Keycloak authorization services


## To be considered

List of OAuth, OpenID Connect and related specifications that should be considered implemented by Keycloak.


### OpenID Connect


#### [OpenID Connect Front-Channel Logout](http://openid.net/specs/openid-connect-frontchannel-1_0.html) (Draft)

> This specification defines a logout mechanism that uses front-channel communication via the User Agent between the OP and RPs being logged out that does not need an OpenID Provider iframe on Relying Party pages. Other protocols have used HTTP GETs to RP URLs that clear login state to achieve this. This specification does the same thing. It also reuses the RP-initiated logout functionality specified in Section 5 of OpenID Connect Session Management 1.0 (RP-Initiated Logout).

[KEYCLOAK-2939](https://issues.jboss.org/browse/KEYCLOAK-2939)


#### [OpenID Connect Back-Channel Logout](http://openid.net/specs/openid-connect-backchannel-1_0.html) (Draft)

> This specification defines a logout mechanism that uses direct back-channel communication between the OP and RPs being logged out; this differs from front-channel logout mechanisms, which communicate logout requests from the OP to RPs via the User Agent.

[KEYCLOAK-2940](https://issues.jboss.org/browse/KEYCLOAK-2940)


#### [OpenID Connect Client Initiated Backchannel Authentication Flow](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html) (Draft)

> OpenID Connect Client Initiated Backchannel Authentication Flow is an authentication flow like OpenID Connect. However, unlike OpenID Connect, there is direct Relying Party to OpenID Provider communication without redirects through the user's browser. This specification has the concept of a Consumption Device (on which the user interacts with the Relying Party) and an Authentication Device (on which the user authenticates with the OpenID Provider and grants consent). This specification allows a Relying Party that has an identifier for a user to obtain tokens from the OpenID Provider. The user starts the flow with the Relying Party at the Consumption Device, but authenticates and grants consent on the Authentication Device.


#### [OpenID Connect Federation](http://openid.net/specs/openid-connect-federation-1_0.html) (Draft)

> The OpenID Connect standard specifies how a Relying Party (RP) can discover metadata about an OpenID Provider (OP), and then register to obtain RP credentials. The discovery and registration process does not involve any mechanisms of dynamically establishing trust in the exchanged information, but instead rely on out-of-band trust establishment.
>  
> In an identity federation context, this is not sufficient. The participants of the federation must be able to trust information provided about other participants in the federation. OpenID Connect Federations specifies how trust can be dynamically obtained by resolving trust from a common trusted third party.
>  
> While this specification is primarily targeting OpenID Connect, it is designed to allow for re-use by other protocols and in other use cases.


### [Financial-grade API (FAPI)](https://openid.net/wg/fapi/)

> In many cases, Fintech services such as aggregation services use screen scraping and stores user passwords. This model is both brittle and insecure. To cope with the brittleness, it should utilize an API model with structured data and to cope with insecurity, it should utilize a token model such as OAuth [RFC6749, RFC6750].
>
>This working group aims to rectify the situation by developing a REST/JSON model protected by OAuth. Specifically, the FAPI WG aims to provide JSON data schemas, security and privacy recommendations and protocols to:
> * enable applications to utilize the data stored in the financial account,
> * enable applications to interact with the financial account, and
> * enable users to control the security and privacy settings.
>
> Both commercial and investment banking account as well as insurance, and credit card accounts are to be considered.

[Keycloak implementation notes](fapi-notes.md)

##### [Part 1: Read Only API Security Profile](https://openid.net/specs/openid-financial-api-part-1-ID2.html)

##### [Part 2: Read & Write API Security Profile](https://openid.net/specs/openid-financial-api-part-2-ID2.html)


### OAuth 2.0

#### [OAuth 2.0 Security Best Current Practice](https://oauth.net/2/oauth-best-practice/)

> OAuth 2.0 Security Best Current Practice describes security requirements and other recommendations for clients and servers implementing OAuth 2.0.

Keycloak supports everything laid out in the best practices, but documentation and adapters do not
make it easy for users to follow / guarantee that their use of Keycloak follows the best practices.


#### [OAuth 2.0 for Browser-Based Apps](https://oauth.net/2/browser-based-apps/)

> OAuth 2.0 for Browser-Based Apps describes security requirements and other recommendations for SPAs and browser-based applications using OAuth 2.0.
> 
> Among other things, it recommends using the Authorization Code flow with the PKCE extension instead of using the Implicit flow.

Keycloak should make it easier for users that secure browser-based apps using Keycloak to make
sure they are doing it according to best practices. 


#### [OAuth 2.0 for Native Apps](https://tools.ietf.org/html/rfc8252)

>    OAuth 2.0 authorization requests from native apps should only be made
     through external user-agents, primarily the user's browser.  This
     specification details the security and usability reasons why this is
     the case and how native apps and authorization servers can implement
     this best practice.

Keycloak should make it easier for users that secure native apps using Keycloak to make
sure they are doing it according to best practices. 


#### [RFC 8626 Device Authorization Grant](https://oauth.net/2/device-flow/)

> The OAuth 2.0 Device Authorization Grant (formerly known as the Device Flow) is an OAuth 2.0 extension that enables devices with no browser or limited input capability to obtain an access token. This is commonly seen on Apple TV apps, or devices like hardware encoders that can stream video to a YouTube channel.

[KEYCLOAK-7675](https://issues.jboss.org/browse/KEYCLOAK-7675)

Design proposal: [https://github.com/keycloak/keycloak-community/pull/6]


#### [OAuth 2.0 Multiple Response Types](http://openid.net/specs/oauth-v2-multiple-response-types-1_0.html) (Final)

> This specification provides guidance on the proper encoding of responses to OAuth 2.0 Authorization Requests in which the request uses a Response Type value that includes space characters. Furthermore, this specification registers several new Response Type values in the OAuth Authorization Endpoint Response Types registry.
> 
> This specification also defines a Response Mode Authorization Request parameter that informs the Authorization Server of the mechanism to be used for returning Authorization Response parameters from the Authorization Endpoint.


#### [OAuth 2.0 Form Post Response Mode](http://openid.net/specs/openid-connect-migration-1_0.html) (Final)

> This specification defines how an OpenID Authentication 2.0 Relying Party can migrate the user from OpenID 2.0 identifier to OpenID Connect Identifier by using an ID Token that includes the OpenID 2.0 verified claimed ID. In this specification, the method to request such an additional claim and the method for the verification of the resulting ID Token is specified.


#### [JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens](https://tools.ietf.org/html/draft-ietf-oauth-access-token-jwt-07)

> This specification defines a profile for issuing OAuth 2.0 access
     tokens in JSON web token (JWT) format.  Authorization servers and
     resource servers from different vendors can leverage this profile to
     issue and consume access tokens in interoperable manner.


#### [OAuth 2.0 Authorization Server Metadata](https://tools.ietf.org/html/rfc8414)

> This specification defines a metadata format that an OAuth 2.0 client
     can use to obtain the information needed to interact with an
     OAuth 2.0 authorization server, including its endpoint locations and
     authorization server capabilities.


### Misc


#### [JSON Web Token Best Current Practices](https://tools.ietf.org/html/draft-ietf-oauth-jwt-bcp-07)

> JSON Web Tokens, also known as JWTs, are URL-safe JSON-based security
     tokens that contain a set of claims that can be signed and/or
     encrypted.  JWTs are being widely used and deployed as a simple
     security token format in numerous protocols and applications, both in
     the area of digital identity, and in other application areas.  The
     goal of this Best Current Practices document is to provide actionable
     guidance leading to secure implementation and deployment of JWTs.


#### [It's Time for OAuth 2.1](https://aaronparecki.com/2019/12/12/21/its-time-for-oauth-2-dot-1)

> Trying to understand OAuth often feels like being trapped inside a maze of specs, trying to find your way out, before you can finally do what you actually set out to do: build your application.


#### [OAuth 3](https://oauth.net/3/)

> This specification and its extensions are being developed within the IETF TXAuth Working Group. Questions, suggestions and protocol changes should be discussed on the TXAuth mailing list.
> 
> Read the design details and examples of OAuth 3 at oauth.xyz.
> 
> This specification is very much in progress, and interested readers are encouraged to participate in its development by joining the IETF Working Group or attending OAuth events.
