# OpenID Connect Client Initiated Backchannel Authentication Flow (CIBA Flow)

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-12137](https://issues.jboss.org/browse/KEYCLOAK-12137)

## Motivation

The major characteristic of CIBA flow that other OAuth2/OIDC flow does not have is that the end-user authentication and authorization of CIBA flow is decoupled from its token issuing process. Financial industries seem to be interested in CIBA flow for realizing online payment services using end-user's smart devices that can leverage this CIBA flow's decoupled authentication and authorization characteristic.

In order for keycloak to be adopted widely by such industries, it seems to be important to support CIBA flow by keycloak.

## Use-cases

CIBA protocol is utilized by the following security profiles:

* [UK OpenBanking Security Profiles : CIBA Profile](https://standards.openbanking.org.uk/security-profiles/)
* [Financial-grade API (FAPI) : Client Initiated Backchannel Authentication Profile](https://openid.net/specs/openid-financial-api-ciba-ID1.html)

## Scope

This design document does not cover all features defined in [CIBA protocol specification](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html). As the first step toward supporting full-fledged CIBA protocol specification, this design document covers only some parts of features defined in CIBA protocol specification. The details are shown in "Inside CIBA Protocol" section.

This design document covers all features defined in [FAPI CIBA profile](https://openid.net/specs/openid-financial-api-ciba-ID1.html) so as to make keycloak become [Certified FAPI CIBA OpenID Provider](https://openid.net/certification/) in the future.

After accomplishing this first step, additional features support will be treated by the next versions of this design document.

## Milestone

A lot of works need to be done to realize the CIBA flow support even in this time as the first step. Therefore, sub-tasks of this support will be defined and corresponding JIRA tickets be created under [KEYCLOAK-12137](https://issues.jboss.org/browse/KEYCLOAK-12137) .

## Specifications

The actual CIBA flow implementation consists of the following parts :

* Inside CIBA Protocol
  - Backchannel Authentication Request/Response (Backchannel Auth Req/Res)
  - Token Request/Response (Token Req/Res)
* Outside CIBA Protocol
  - Authentication/Authorization by Authentication Device (AD) (AuthN/AuthZ by AD)

In one CIBA flow, these parts are executed basically in sequence as follows :

1. Inside CIBA Protocol  : Backchannel Auth Req/Res
2. Outside CIBA Protocol : AuthN/AuthZ by AD
3. Inside CIBA Protocol  : Token Req/Res

2 and 3 might be executed in reverse order. 3 might be executed several times until 2 finishes.

CIBA protocol specification requires AuthN/AuthZ by AD but does not define how to do it, namely it is out of scope of CIBA Protocol specification.

In Backchannel Auth Req/Res part, by following CIBA protocol, keycloak processes a backchannel authentication request from Consumption Device (CD) and returns its response. Beside that, keycloak invokes AuthN/AuthZ by AD part.

In AuthN/AuthZ by AD part, keycloak itself or other entity requests an AD to authenticate and get consents from an end-user and inform Token Req/Res part of its result. Besides that, it executes all the processes that need to be executed after the end-user logged-in like creating User Session and Authenticated Client Session that are needed to generate tokens in Token Req/Res part.

In Token Req/Res part, by following CIBA protocol, keycloak processes a token request from CD and returns its response based on the information received from AuthN/AuthZ by AD part.

### Inside CIBA Protocol

This section describes which features defined by CIBA protocol specification are supported or not by this version of the design document.

#### Backchannel Authentication Endpoint

Keycloak's Backchannel Authentication Endpoint basically follows [7. Backchannel Authentication Endpoint](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#rfc.section.7) and [13. Authentication Error Response](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#rfc.section.13) .

URI of this endpoint is as follows :

`{scheme}://{host}[:{port}]/auth/realms/{realm}/protocol/openid-connect/backchannelAuthn`

At this endpoint, the following security tasks are done the same as at Token Endpoint :

* CORS
* Check TLS
* Check Realm
* Check Client (Client Authentication)
* Check User (is existed, is activated)
* Prohibit Caching on HTTP Client
  - Cache-Control : no-store
  - Pragma : no-cache

#### Token Endpoint

Keycloak's existing Token Endpoint basically follows [10.1. Token Request Using CIBA Grant Type](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#rfc.section.10.1) and [11. Token Error Response](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#rfc.section.11) .

#### Conveying Information on End-User to be Authenticated from CD

On CIBA Protocol specification Section [7.1. Authentication Request](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#auth_request) :

#### Request Parameters for user info conveyance

- login_hint       : supported
  - User Resolver : keycloak provides the resolver for the following information
    - username
    - email
  - Encryption : this resolver supports encryption for them by using client secret with AES128/192/256bit

- login_hint_token : supported
  - User Resolver : keycloak provides the resolver for the following token
    - Format : JWT
    - Profile : refer to "AuthN/AuthZ User Resolver" section.
    - User Information
      - username
      - email
  - Integrity Protection/Message Authentication : JWS (client signed)
    - optional
    - algorithm : ES256/384/512, PS256/384/512, RS256/384/512
  - Encryption : JWE
    - not supported. (supported in the future)

- id_token_hint    : supported
  - PPID support for "sub"

#### Request Parameters related to user info conveyance

- request (signed authentication request) : supported
- user_code : supported

##### Notes for user info conveyance

There are two types of representation for conveying user to be authenticated from a client (CD) to keycloak :

- public or guessable

  It can be acquired from public sources (e.g. name, phone number).

  Even if cannot be acquired from public sources, it is easy to guess (e.g. email address).

- not public and not guessable

  It cannot be acquired from public source and hard to guess (e.g. UserModel's ID in keycloak internal).

Considering security, we pay attention to the following points :

- Unsolicited backchannel authentication request by a malicious user (impersonation)
  - public or guessable user information is vulnerable to this attack.
  - Regardless of the way of conveying user to be authenticated, User Code seems to be effective countermeasure.
- Unsolicited backchannel authentication request by a malicious other client to attack other victim client (replay, forge legitimate backchannel authentication request)
  - PPID seems to be effective countermeasure.
  - Encryption seems to be effective countermeasure.
  - Signed JWT seems to be effective countermeasure.
  - Regardless of the way of conveying user to be authenticated, User Code seems to be effective countermeasure.
- Unsolicited backchannel authentication request by a malicious client itself (keep user's user_code and other authentication related information once this client tried CIBA flow with this user, forge legitimate backchannel authentication request)
  - No principle effective countermeasure by keycloak seems not to exist (but might be my misunderstanding). Using binding message and the end user being taking extra precaution on it seems to be only the way to prevent it.
- PII leakage on the wire
  - PPID seems to be effective countermeasure.
  - Encryption seems to be effective countermeasure.

#### Backchannel Token Delivery Modes

On CIBA Protocol specification Section [5. Poll, Ping and Push Modes](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#rfc.section.5) :

* poll : supported
* ping : NOT supported at this time but will be supported in the future
* push : NOT supported at this time but will be supported in the future

#### Token Request Throttling

On CIBA Protocol specification Section [7.3. Successful Authentication Request Acknowledgement](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#successful_authentication_request_acknowdlegment) :

* interval : supported

If this "interval" is set to 0, the token request throttling is deactivated.

According to CIBA protocol specification, the access not waiting this interval is incurred the penalty of 5 sec as an additional wait.

#### Authorization

Authorization by AD works the same as the current keycloak's authorization code flow. If the CD's "Consent Required" setting is "Yes", the following scope value are presented to the end-user on the AD's UI:

* scope                : specified by Backchannel Authentication Request's "scope" parameter
* client default scope : set by keycloak Admin Console's Client Settings

#### Binding Message

On CIBA Protocol specification Section [7.1. Authentication Request](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#auth_request), Binding Message is supported.

#### User Code

On CIBA Protocol specification Section [7.1. Authentication Request](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#auth_request), User Code is supported.

#### Signed Authentication Request

On CIBA Protocol specification Section [7.1.1. Signed Authentication Request](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#signed_auth_request), Signed Authentication Request is supported.

#### Pairwise Identifiers

On CIBA Protocol specification Section [4. Registration and Discovery Metadata](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#registration), Poll Modes with Pairwise Identifiers is NOT supported.

#### Requested Expiry

On CIBA Protocol specification Section [7.1. Authentication Request](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#auth_request), Requested Expiry is NOT supported.

#### Token Request Polling

On CIBA Protocol specification Section [10.1. Token Request Using CIBA Grant Type](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html#rfc.section.10.1), Long Polling is not supported at this time but will be supported in the future.

#### Multiple ADs per user

It does not mention explicitly in the CIBA Protocol specification but there might be the case that some use case requires registering multiple ADs. It is not supported at this time but will be supported in the future.

### Outside CIBA Protocol

The process of AuthN/AuthZ by AD is highly use-cases dependent (e.g. Push notification to the end-user's smart device, SMS message to the end-user's smart device, etc.).

To keep the design in use-cases independent, this design document describes the interface between inside and outside CIBA protocol.

However, if keycloak lacks the actual implementation of these interfaces, the user of keycloak does not use CIBA flow unless they implement these interfaces by themselves. It is inconvenient for users.

Therefore, this design document also defines the default implementation for these interfaces.

### Runtime Environment

This version of the design document considers both standalone, clustering environment and multi-cluster(Cross-DC) environment.

### Settings

To configure inside CIBA part's operation, CIBA policy is introduced as part of Admin Console's Realm Settings.

#### CIBA Policy

The following items defined in CIBA protocol specification can be configured per realm as default but overridden per client :

* backchannel_token_delivery_mode : fix "poll"
* expires_in
* interval : if set to 0, the token request throttling is deactivated

### Single Sign On

keycloak cannot realize SSO by CIBA flow.

To realize SSO, keycloak uses Cookies retained in the end-user's browser. In CIBA protocol, the end-user's browser does not appear in the CIBA flow.

## Design and Implementation

This section describes how to realize what Specification section determined.

The following diagram shows components and their interactions to accomplish CIBA flow. Beside three parts mentioned earlier, one cache component "Auth Result Cache" is introduced.

```
+------+               +-----------------------------------------------------------------------+
| CD   |               | keycloak                                                              |
|      |               |  +---------------------+                 +--------------------------+ |
|      |               |  | Inside CIBA         |                 | Outside CIBA             | |
|      |  (1) POST     |  |  +---------------+  |                 |  +--------------------+  | |
|      | ------------------> | Backchannel   |  | (3)             |  | AuthN/AuthZ by AD  |  | |
|      | <-[auth_req_id]---- | Auth Req/Res  | --[Auth Result ID]--> |                    |  | |
|      |              (2) |  +---------------+  |                 |  +--------------------+  | |
|      |               |  |                     |                 |            |             | |
|      |               |  |                     |                 +------------|-------------+ |
|      |               |  |                     |                          (4) |               |
|      |               |  |                     |                       [Auth Result ID]       |
|      |               |  |                     |                              |               |
|      |               |  |                     +------------------------------V-------------+ |
|      |  (5) POST     |  |  +---------------+    (6)                +--------------------+  | |
|      | -[auth_req_id]----> | Token Req/Res | --[Auth Result ID]--> | Auth Result Cache  |  | |
|      | <------------------ |               |                       |                    |  | |
|      |              (7) |  +---------------+                       +--------------------+  | |
|      |               |  +------------------------------------------------------------------+ |
+------+               +-----------------------------------------------------------------------+
```

(1) : CD sends to keycloak Backchannel Auth Request.

(2) : keycloak returns to CD Backchannel Auth Response. It includes "auth_req_id" defined in CIBA protocol specification to identify the CIBA flow invoked by the request (1). "auth_req_id" includes Backchannel Auth Request related context data in order for Token Req/Res part to restore it for generating tokens afterwards. To make "auth_req_id" be tamper-resistant and be verifiable, JWS is applied to "auth_req_id". If it includes the data that must not be revealed to the entity other than CD and keycloak, JWE is applied to "auth_req_id".

(3) : Backchannel Auth Req/Res part sends request to AuthN/AuthZ by AD part with "Auth Result ID" to identify the CIBA flow bound with "auth_req_id" of (2).

(4) : After conducting AuthN/AuthZ by AD, AuthN/AuthZ by AD part pushes its result to Auth Result Cache with "Auth Result ID" to identify the CIBA flow bound with "auth_req_id" of (2).

(5) : CD sends to keycloak Token Request with "auth_req_id" bound with request (1).

(6) : Token Req/Res part retrieves Backchannel Auth Request related context data from "auth_req_id" of (2) and takes the result of AuthN/AuthZ by AD part from Auth Result Cache with "Auth Result ID" bound with "auth_req_id" of (2).

(7) : keycloak returns to CD tokens based on the result of AuthN/AuthZ by AD bound with request (1).

### How to Implement These Three Parts

To support broad spectrum of use-cases flexibly, these parts are implemented separately.

Inside CIBA protocol part is implemented as body of the codes because this phase is invariant with respect to specific use-cases.

AuthN/AuthZ by AD part of outside CIBA protocol part is implemented as detachable provider because this phase is highly dependent on specific use-cases. The developer implements this provider and can load it onto keycloak according to keycloak's [Server Developer Guide - Registering provider implementations
](https://www.keycloak.org/docs/latest/server_development/index.html#registering-provider-implementations). 

### How to Implement Auth Result Cache

Considering clustering environment, Auth Result Cache is implemented as Infinispan distributed cache.

Auth Result Cache can be realized by the existing `actionToken` cache via the provider refactoring `CodeToTokenStoreProvider`. 

Auth Result Cache is for single-use, but considering the following case that the CD requests tokens before the completion of authentication by AD so that keycloak only removes this cache entry after the completion of authentication by AD.

`actionToken` supports Cross-DC so that this CIBA flow also supports Cross-DC.

### Interface between keycloak and CD

Interface between keycloak and CD are defined by CIBA protocol specification. Therefore, keycloak follows it.

Multiple CIBA flows can run between keycloak and CDs simultaneously. According to CIBA protocol specification, "auth_req_id" can be used to identify them.

In keycloak, CIBA flow's Backchannel Authentication Request context information consists of the following :
* User ID   : ID of the end-user whom CD requested to be authenticated by AD
* Client ID : ID of CD who sent Backchannel Authentication request
* Scope     : OAuth2's scope which CD requested to get consent from the end-user

### Interface between Inside and Outside CIBA Protocol

As mentioned earlier, the CIBA flow consists of three parts : Backchannel Auth Req/Res, AuthN/AuthZ by AD and Token Req/Res.

The interface between inside and outside CIBA Protocol are comprised of the following two interfaces :

1. From Backchannel Auth Req/Res part to AuthN/AuthZ by AD part
2. From AuthN/AuthZ by AD part to Token Req/Res part

On Token Req/Res part, Keycloak must issue tokens about the authenticated end-user on AuthN/AuthZ by AD part whom CD requested to authenticate on Backchannel Auth Req/Res part. Keycloak needs to process such several CIBA flows simultaneously.

In order to do it, keycloak needs the information on identifying which communications on each part belongs to which CIBA flow. Such the information is called "Auth Result ID" in this document. At least, all interfaces between inside and outside CIBA Protocol needs to include this information.

CIBA flow is identified by "auth_req_id" defined by CIBA protocol between CD and keycloak (Inside CIBA Protocol).

CIBA flow is identified by "Auth Result ID" between keycloak (Inside CIBA Protocol) and keycloak (Outside CIBA Protocol).

keycloak (Inside CIBA Protocol) manages the relationship between "auth_req_id" and "Auth Result ID".

#### From Backchannel Auth Req/Res part to AuthN/AuthZ by AD part

Without considering use-case dependent specific Authentication/Authorization way in AuthN/AuthZ by AD part, the following tasks need to be accomplished in AuthN/AuthZ by AD part :

* Authentication : Identify who are to be authenticated
* Authorization : Identity which scopes are to be granted
* CD/AD Binding : Show binding message on AD's UI in order for an end-user to recognize that this authentication/authorization was invoked by CD
* Expiration : Terminate authentication/authorization in a finite time frame
* CIBA Flow Binding : Recognize that authentication/authorization belongs to which CIBA flow

To do these tasks, the following information need to be passed from Backchannel Auth Req/Res part to AuthN/AuthZ by AD part

* Backchannel Authentication Request parameters
  - login_hint
  - scope
  - binding_message
* Backchannel Authentication Expiry
  - expires_in (from Backchannel Authentication Response parameters)
* Auth Result ID

#### From AuthN/AuthZ by AD part to Backchannel Auth Req/Res part

In order to return Backchannel Auth Response to CD, keycloak does not need to wait for the completion of AuthN/AuthZ by AD part according to CIBA protocol specification.

Therefore, no information needs to be passed from AuthN/AuthZ by AD part to Backchannel Auth Req/Res part.

#### From AuthN/AuthZ by AD part To Token Req/Res part

Without considering use-case dependent specific Authentication/Authorization way, in Token Req/Res part the following tasks need to be accomplished in this part :

* Token generation : generate token based on the result of AuthN/AuthZ by AD

To do these tasks, the following information need to be passed from AuthN/AuthZ by AD part to To Token Req/Res part

* Auth Result ID
* AuthN/AuthZ Result
  - succeeded : successfully authenticated and authorized
  - unauthorized : successfully authenticated but not refused to grant consents
  - cancelled : authentication and authorization was cancelled
  - failed :  authentication failed
  - different : different user was authenticated
  - expired : authentication and authorization process expired
  - unknown : other unexpected event happened

To generate tokens, the following information is also needed :

* User ID   : ID of the end-user whom CD requested to be authenticated by AD
* Client ID : ID of CD who sent Backchannel Authentication request
* Scope     : OAuth2's scope which CD requested to get consent from the end-user

This information can be retrieved from CIBA flow's context date identified by "auth_req_id" which is related to received "Auth Result ID".

AuthN/AuthZ by AD part does not communicate with Token Req/Res part directly because the latter is invoked by CD's token request. Therefore, keycloak implements Auth Result Cache to hold the result of AuthN/AuthZ by AD. AuthN/AuthZ by AD part pushes this result to this store with "Auth Result ID" as the access key. Token Req/Res part retrieves this result from this store with "Auth Result ID".

#### From Token Req/Res part to AuthN/AuthZ by AD part

CIBA protocol specification does not describe whether Token Request affects AuthN/AuthZ by AD.

Therefore, no information needs to be passed from Token Req/Res part to AuthN/AuthZ by AD part.

### Default Implementation of AuthN/AuthZ by AD part of Outside CIBA Protocol

There are several ways of AuthN/AuthZ by AD (e.g. push notification to an android device, push notification to an iOS device, send SMS message to an android device, send SMS message to an iOS device, make a telephone call, etc.). Each of such the way uses different device and technology so that keycloak cannot cover all of such ways. 

Considering this point, this default implementation delegates AuthN/AuthZ by AD to other entity called here "Decoupled Auth Server" outside keycloak. It assumes that this Decoupled Auth Server is under control by an administrator of keycloak.

```
---------------------------------+                 +-----------+
| keycloak                       |                 | Decoupled |
|  +--------------------------+  |                 | Auth      |
|  | Outside CIBA             |  |                 | Server    |
|  |  +--------------------+  |  | (i) POST        |           |
|  |  | AuthN/AuthZ by AD  | --[Auth Result ID]--> |           |
|  |  |                    | <-[Auth Result ID]--- |           |
|  |  +--------------------+  |  |    (ii) POST    |           |
|  +------------|-------------+  |                 |           | 
|         (iii) |                |                 |           |
|         [Auth Result ID]       |                 |           |
|               V                |                 |           |
|     +--------------------+     |                 |           |
|     | Auth Result Cache  |     |                 |           |
|     |                    |     |                 |           |
|     +--------------------+     |                 |           |
+--------------------------------+                 +-----------+
```

(i)   : keycloak sends to Decoupled Auth Server an AuthN/AuthZ by AD request with "Auth Result ID".

(ii)  : After completion of AuthN/AuthZ by AD, Decoupled Auth Server sends to keycloak the results of AuthN/AuthZ by AD with "Auth Result ID".

(iii) : After receiving the results of AuthN/AuthZ by AD, AuthN/AuthZ by AD part pushes this result to Auth Result Cache with "Auth Result ID".

AuthN/AuthZ by AD part's necessary tasks is as follows :

1. Requests an AD to authenticate and get consents from an end-user
2. Executes all the processes that need to be executed after the end-user logged-in like creating User Session and Authenticated Client Session that are needed to generate tokens in Token Req/Res part.
3. Informing Token Req/Res part of 1's result.

1 is highly dependent on actual use-cases while 2 and 3 seems to be use-case independent. Therefore, The abstract base class implementing 2 and 3 is provided and the class extending this abstract base class is also provided.

#### From Token Req/Res part to AuthN/AuthZ by AD part

##### Endpoint

Not specified by this design document.

##### Method

Only POST method is supported.

##### Content-Type

application/x-www-form-urlencoded

##### Form Data

* auth_result_id

  "Auth Result ID" bound with "auth_req_id" issued to CD.

* binding_message

  "binding_message" sent from CD.

* is_consent_required

  set "true" if CD requires getting consent from an end user via AD. If not, set "false".

* user_info

  indicating who is authenticated by AD so that it must be identifiable by AD.

* scope

  "scope" sent from CD.

#### From AuthN/AuthZ by AD part to Token Req/Res part

##### Endpoint

`{scheme}://{host}[:{port}]/auth/realms/{realm}/protocol/openid-connect/ext/ciba-decoupled-authn-callback/`

##### Method

Only POST method is supported.

##### Authentication

Required by the same manner in Token Endpoint. It means that the decoupled auth server needs to be registered in keycloak as a confidential client in advance.

##### Content-Type

application/x-www-form-urlencoded

##### Form Data

* auth_req_id

  "auth_req_id" issued to CD.

* user_info

  indicating who was authenticated by AD.

* auth_result

  the result of authentication by AD.

  * succeeded : successfully authenticated and authorized
  * unauthorized : successfully authenticated but not refused to grant consents
  * cancelled : authentication and authorization was cancelled
  * failed : authentication failed
  * unknown : other unexpected event happened

### Cache

To prevent entries from not being released forever, both Auth Req Context Cache and Auth Result Cache are implemented as a Infinispan cache with lifespan of its entry.

Infinispan will eventually remove expired entries but not just in time precisely. Therefore, the entry itself has the field showing its expiry and keycloak checks whether this entry expires based on this field.

Both structures of entries of Auth Req Context Cache and Auth Result Cache is so simple that keycloak uses existing cache "actionTokens" for both of them.

### IDs for Identifying CIBA Flow

"auth_req_id" and "Auth Result ID" are version 4 UUID String and it is the key to retrieve the entry from the cache swiftly

These IDs lack integrity protection and source authentication so that do not contain any kind of information on CIBA flow.

### Authentication Flow

Like other flows (Browser Flow, Direct Grant Flow, etc.), the authentication flow for CIBA flow is also needed to execute all the processes that need to be executed after the end-user logged-in like creating User Session and Authenticated Client Session that are needed to generate tokens in Token Req/Res part.

However, authentication and authorization is executed by AuthN/AuthZ by AD part so that it is enough for the authentication flow for CIBA flow to contain only the pass-through authenticator. 

### Event

The following events are newly added :

* AUTH_REQ_ID_TO_TOKEN : On token endpoint, keycloak successfully generate tokens in return to "auth_req_id".
* AUTH_REQ_ID_TO_TOKEN_ERROR : On token endpoint, keycloak fails to generate tokens in return to "auth_req_id".

### Server Metadata

The supported server metadata is as follows :

* "backchannel_token_delivery_modes_supported" : ["poll"]
* "backchannel_authentication_endpoint" : "{scheme}://{host}[:{port}]/auth/realms/{realm}/protocol/openid-connect/backchannelAuthn"
* "backchannel_user_code_parameter_supported" : false

### Client Metadata

The supported client metadata is as follows :

* "backchannel_token_delivery_mode" : only accepts "poll"

### Token Request Throttling

According CIBA protocol specification, keycloak determines whether it refuses the token request based on its accompanying "auth_req_id".

There are three options considered :

1. Utilize existing `DefaultBruteForceProtector`
2. Use Inifinispan cache with lifespan of its entry to share among all nodes in the cluster information on which token request with "auth_req_id" is refused
3. Use some collection with lifespan of its entry on each node, not sharing among all nodes in the cluster information on which token request with "auth_req_id" is refused

1 is too rich for this token request throttling.

2 can satisfies CIBA protocol specification but has performance concern if vast number of token requests happens. When the token request arrives without waiting an interval, keycloak needs to update when this next token request with "auth_req_id" be allowed to be accepted with some additional wait as penalty. Updating the entry of Infinispan cache causes some traffic among nodes in the cluster to sync the status of this entry.

3 can partially satisfies CIBA protocol specification but no traffic is exchanged among nodes in the cluster due to this token request throttling. However, it seems that keycloak does not prepare such collection so that implementing it or introducing some kind of OSS is required.

In this time, 3 is to be realized to avoid using new cache layer and its associated traffic among nodes.

When Token Request arrives, keycloak at first check whether access throttling is applied or not. If not, keycloak decrypt the `auth_req_id`, extract the key for accessing the cache entry of Auth Result Cache, get and remove this entry to find out the result of authentication by AD.

## Extensibility

In order for users to use CIBA flow on their each use-case, the following SPIs are introduced.

### AuthN/AuthZ User Resolver

This provider converts user information in "login_hint", "login_hint_token" and "id_token_hint" to User Model.

CIBA protocol specification defines these parameters to convey information on a user to be authenticated. However, CIBA protocol specification does not define the value itself of them (except for "id_token_hint").

This provider converts CD-dependent user representation in such the parameters to user representation in keycloak, namely User Model.

### Decoupled Authentication Provider

This provider is the implementation of AuthN/AuthZ by AD part of outside CIBA Protocol.

This part is highly dependent of use-cases so that the developer can implement this provider to realize CIBA flow on their use-case.

## Security Consideration

### "auth_req_id" Swapping

"auth_req_id" Swapping is that keycloak sends tokens to the entity other than the one from which keycloak received the backchannel authentication request.

To prevent it, keycloak checks and confirms that CD sending Backchannel Auth Request bound with "auth_req_id" must be equal to CD sending Token Request with the same "auth_req_id". 

### User Swapping

User Swapping is that keycloak sends tokens generated based on the result of the authenticate and be authorized by the end-user whom CD did not intended.

To prevent it, keycloak checks and confirms that the user to be authenticated must be equal to actually authenticated user. 

### Replay Attack/Guessing Attack

"auth_req_id" might be the replay attack and guessing attack target. To prevent this replay attack, the following measure are considered :

* "auth_req_id" is short lived
* "auth_req_id" is one time use (except for the token request before completion of AuthN/AuthZ by AD)
* The generator of "auth_req_id" has enough bits of entropy to guard against these attacks in a certain level(Use a version 4 UUID string having about 112 bits of entropy)

### DoS Attack

 keycloak only aims to minimize CPU usage by Token Request aiming DoS attack.
 keycloak cannot refuse to receive Token Request aiming DoS attack. If want to prevent it, other means is required.

## Privacy Consideration

### PII on the wire

The user information might be conveyed by "login_hint", "login_hint_token", and "id_token_hint".
It might be better that such the information be encoded so that other entity cannot identify the user itself. AuthN/AuthZ User Resolver mentioned just before can be used to convert this encoded information to User Model.

## Performance Consideration

### Sync Auth Result Cache

keycloak needs to share or distribute among nodes cache entries of Auth Result Cache. Therefore, the performance problem might arise if keycloak has vast number of Backchannel Authentication Requests simultaneously or in the short time frame.

### Find User Session Model Bound with "auth_req_id"

In Token Req/Res part, keycloak needs to find User Session Model of the end-user authenticated by AD in a CIBA flow with bound with "auth_req_id".

The one way is that keycloak sets "auth_req_id" to User Session's note when generating this User Session and find it afterwards by linear search. It seems to have serious performance problem especially when keycloak having vast number of User Sessions and distributed among several nodes.

The faster way is using its User Session ID as a search key. Therefore, in Backchannel Auth Req/Res part, keycloak issues the User Session ID in advance and use it when generating User Session in AuthN/AuthZ by AD part. However, this way cannot utilize key affinity feature of Infinispan.

