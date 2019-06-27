# Implemented

List of OAuth, OpenID Connect and related specifications that are currently implemented fully or partially by Keycloak.
This is a work in progress and the list is incomplete.


## OpenID Connect


### [OpenID Connect Core](http://openid.net/specs/openid-connect-core-1_0.html) (Final)

Defines the core OpenID Connect functionality: authentication built on top of OAuth 2.0 and the use of claims to 
communicate information about the End-User.

* Keycloak coverage: To be reviewed


### [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) (Final)

Defines how clients dynamically discover information about OpenID Providers.

* Keycloak coverage: To be reviewed


### [OpenID Connect Dynamic Registration](http://openid.net/specs/openid-connect-registration-1_0.html) (Final)

Defines how clients dynamically register with OpenID Providers.

* Keycloak coverage: To be reviewed


### [OpenID Connect Session Management](http://openid.net/specs/openid-connect-session-1_0.html) (Draft)

Defines how to manage OpenID Connect sessions, including postMessage-based logout functionality.

* Keycloak coverage: To be reviewed


# Non-standard

List of non-standard approaches in Keycloak

* Admin endpoints - manage realms, clients, users, etc.
* Account endpoint - manage user account
* Logout endpoint - custom logout endpoint that is close to, but does not follow any specifications
* Admin adapter endpoints - ability to send not-before, logout and other "admin" events to adapters
* Cluster node registration - ability to register node instances for a client to send logout events to correct node in a cluster
* Permission endpoint - ability to retrive permissions for a user from Keycloak authorization services


# To be considered

List of OAuth, OpenID Connect and related specifications that should be considered implemented by Keycloak.


## OpenID Connect


### [Financial-grade API (FAPI)](https://openid.net/wg/fapi/) (Draft)

Security profiles defining how OpenID Connect, OAuth and related specifications can be used in a highly secure way
for fintech use-cases.

* [Keycloak implementation notes](fapi-notes.md)


### [OpenID Connect Front-Channel Logout](http://openid.net/specs/openid-connect-frontchannel-1_0.html) (Draft)

Defines a front-channel logout mechanism that does not use an OP iframe on RP pages.


### [OpenID Connect Back-Channel Logout](http://openid.net/specs/openid-connect-backchannel-1_0.html) (Draft)

Defines a logout mechanism that uses direct back-channel communication between the OP and RPs being logged out.


### [OpenID Connect Federation](http://openid.net/specs/openid-connect-federation-1_0.html) (Draft)

Defines how sets of OPs and RPs can establish trust by utilizing a Federation Operator


## OAuth


### [OAuth 2.0 Multiple Response Types](http://openid.net/specs/oauth-v2-multiple-response-types-1_0.html) (Final)

Defines several specific new OAuth 2.0 response types


### [OAuth 2.0 Form Post Response Mode](http://openid.net/specs/openid-connect-migration-1_0.html) (Final)

Defines how to return OAuth 2.0 Authorization Response parameters (including OpenID Connect Authentication Response 
parameters) using HTML form values that are auto-submitted by the User Agent using HTTP POST.