# [Financial-grade API - Part 1: Read-Only API Security Profile](https://openid.net/specs/openid-financial-api-part-1.html)

Keycloak mostly covers what is required by the FAPI read-only security profile. However, it heavily relies on clients
to be configured correctly and there is no validation that clients are doing the correct thing.


## Validation of client configuration

One idea would be to introduce client profiles that can validate clients are configured correctly. Further, client 
profiles could be configured to be blocking client registration and requests that do not pass validation, or to
simply warn the administrator.

As an example a FAPI client profile could validate things like the following when clients are registered:

* Confidential client is configured with MTLS or JWT based authentication
* Key sizes
* Wildcard not included in redirect_uri
* redirect_uri only permits https

Further, it could validate things like following when the clients initiate authentication requests:

* PKCE is enabled using S256
* HTTPS is enabled
* Includes openid in scope value, nonce, state and code_verifier

Validation for client requests may be better done by setting explicit config options on client (such as PKCE required),
rather than having a client profile invoked on requests.


## Notes from reviewing specification

* Should review [MTLS](https://tools.ietf.org/html/draft-ietf-oauth-mtls) and coverage
* Should review [JWT authentication](https://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication) and coverage
* Should review [X.1254](http://www.itu.int/rec/dologin_pub.asp?lang=e&id=T-REC-X.1252-201004-I!!PDF-E&type=items) with regards to LoA 2 requirement  
* Do not currently validate key sizes
* Verify if redirect_uri is required in authorization request. We may allow omitting if client has a single redirect_uri registered
* redirect_uri in Keycloak can contain wildcards, while it explicitly states it should be an exact match
* Check if we return the list of granted scopes with the issued access token
* No support for opaque tokens currently
* https can be optional depending on realm ssl setting
* Review if Keycloak supports elliptic curve with JWT client authentication
* Review REST endpoints provided by Keycloak with regards to recommendations in the spec