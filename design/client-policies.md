# Client Policies

* **Status**: Draft #2
* **JIRA**: [KEYCLOAK-13933](https://issues.jboss.org/browse/KEYCLOAK-13933), [KEYCLOAK-16137](https://issues.jboss.org/browse/KEYCLOAK-16137)
## Motivation

To make it easy to secure client applications, it is beneficial to realize the following four points in a unified way.

1. Setting policies on what configuration a client can have
2. Validation of client configurations
3. Conformance to a profile such as FAPI
4. Limiting what type of configuration a client can have based on source that is creating the client

1 and 2 have been realized by the current keycloak, but not flexible and comprehensive.

3 will be realized by [Client Conformance Profiles](https://issues.redhat.com/browse/KEYCLOAK-11612) based on this "Client Policies". It discusses which kind of security profiles like FAPI are supported and how to implement them.

4 has not yet been realized by the current keycloak.

To realize these four points in a unified way, here "Client Policies" concept is introduced.

## Use-cases

Client Policies realize the following three points mentioned in Motivation as follows.

### 1. Setting policies on what configuration a client can have

The current keycloak has already supported them as [Client Registration Policies](https://www.keycloak.org/docs/latest/securing_apps/index.html#client-registration-policies).

However, it can only cover OIDC Dynamic Client Registration.

Client Policies cover not only what Client Registration Policies can do but other client registration and configuration ways.

In the future, Client Registration will be replaced by Client Policies.

### 2. Validation of client configurations

The current keycloak has already supported validation on whether the client follows settings like Proof Key for Code Exchange, Request Object Signing Algorithm, Holder-of-Key Token, etc on some endpoints like Authorization Endpoint, Token Endpoint, etc.

These can be specified by each setting item (on Admin Console, switch, pulldown menu, etc). To make the client application secure, user need to set a lot of such settings in appropriate way, which makes it difficult for the user to secure their client application.

Client Policies can do these validation of client configurations mentioned just above. In the future, these each setting items will be replaced by Client Policies. 

The pre-set client policies for following some security profiles (FAPI, SPA, Native App, etc) are provided, which makes it easy for the user to secure their client application.

### 3. Conformance to a profile such as FAPI

It is treated in [Client Conformance Profiles](https://issues.redhat.com/browse/KEYCLOAK-11612) based on this Client Policies. Which kind of security profiles are supported and how to realize them are discussed on 
[Client Conformance Profiles](https://issues.redhat.com/browse/KEYCLOAK-11612).

## Scope

This client policy concept is independent of any specific protocol. However, this document only deals with the implementation for OIDC.

## Implementation details

Client Policies consists of the four building blocks, Condition, Executor, Profile and Policy.

### Condition : which client

A condition determines to which client a policy is adopted. The condition checks whether one specified criteria is satisfied. For example, some condition checks whether the [access type](https://www.keycloak.org/docs/latest/server_admin/index.html#oidc-clients) of the client is confidential.

The client can not be used solely by itself. It can be used in a policy that is described afterwards.

#### How to implement

A condition can be implemented as a provider (`ClientPolicyConditionSpi`, `ClientPolicyConditionFactory`, `ClientPolicyCondition`). Developers can add their own condition by implementing this provider.

#### How to configure

A condition can be configurable the same as other configurable provider. What can be configured depends on each condition's nature.

#### Which kind of conditions are provided

The following conditions are provided:

1. The way of creating/updating a client

* OIDC Dynamic Client Registration
  * Authenticated
  * Anonymous
* Admin REST API (Admin Console, etc.)

E.g. When creating a client, a policy is adopted when this client is created by OIDC Dynamic Client Registration with the access token.

2. Author of a client

* Group
* Role

On OIDC dynamic client registration, an author of a client is the end user who was authenticated to get an access token for generating a new client, not Service Account of the existing client that actually accesses the registration endpoint with the access token.

On registration by Admin REST API, an author of a client is the end user like keycloak's administrator

3. Client

* Client Access Type
  * confidential
  * public
  * bearer-only

E.g. When a client sends an authorization request, a policy is adopted if this client is confidential.

* Client Domain Name
* Client Role
* Client Scope
* Client Host
* Client IP


### Executor : what action

An executor specifies what action is executed on a client to which a policy is adopted. The executor executes one or several specified actions. For example, some executor checks whether the value of the parameter "redirect_uri" in the authorization request matches exactly with one of the pre-registered redirect URIs on Authorization Endpoint and rejects this request if not. 

The executor can not be used solely by itself. It can be used in a profile that is described afterwards.

#### How to implement

An executor can be implemented as a provider (`ClientPolicyExecutorSpi`, `ClientPolicyExecutorFactory`, `ClientPolicyExecutor`). Developers can add their own executor by implementing provider.

#### How to configure

An executor can be configurable the same as other configurable provider. What can be configured depends on each executor's nature.

#### What events

An executor acts on the following events:

* Creating a client
* Updating a client
* Sending an authorization request
* Sending a token request
* Sending a token refresh request
* Sending a token revocation request
* Sending a token introspection request
* Sending a userinfo request
* Sending a logout request with a refresh token

On each event, an executor can work in multiple phases. For example, on creating/updating a client, the executor can modify the client configuration in augment phase. After that, the executor validate this configuration in validation phase.

#### Which kind of executors are provided

One of several purposes for this executor is to realize client conformance profiles discussed in [its design document](https://github.com/keycloak/keycloak-community/pull/44). To do so, following executors are needed:

* Enforce more secure client authentication method when client registration
* Enforce Holder-of-Key Token
* Enforce Proof Key for Code Exchange (PKCE)
* Enforce secure signature algorithm for Signed JWT client authentication (private-key-jwt)
* Enforce HTTPS redirect URI
* Enforce Request Object satisfying high security level
* Enforce Response Type of OIDC Hybrid Flow
* Enforce more secure state and nonce treatment for preventing CSRF
* Enforce more secure signature algorithm when client registration

Also, one of several purposes for this executor is to take over [Client Registration Policy](https://www.keycloak.org/docs/latest/securing_apps/index.html#client-registration-policies).

### Profile : bundle executors to realize a security profile

A profile consists of several executors which can realize a security profile like FAPI.

The profile is referred from the policy just described below and all executors in this profile are executed against the clients which this policy adopts. 

### Policy : how to determine which client and what action

A policy consists of several conditions. The policy can be adopted to clients satisfying all conditions of this policy. The policy refers several profiles and all executors of these profiles execute their task against the client that this policy is adopted.

This policy can be enabled/disabled. The policy works if it is enabled.

Considering both a condition and an executor do their task independently from other conditions and other executors, the administrator can not specify in which order conditions and executors are evaluated.

#### Built-in Policy and Profile

A built-in policy and profile are ones registered in keycloak as default and can not be modified. This built-in profile can be used to enforce basic security level to all clients.

### Endpoint and Operation : when and where policy is adopted

Conditions and executors of a policy should be evaluated on endpoints where a client application interact with keycloak by OAuth2/OIDC protocol. Considering this point, when and where a policy can be adopted to a client is as follows:

##### OIDC Dynamic Client Registration
  * Operation
    * Before Register Client
    * After Registered Client
    * Before Update Client
    * After Updated Client
    * View Client
    * Unregister Client
  * Authentication
    * Authenticated (Using Initial Access Token, Registration Access Token)
    * Anonymous

##### Admin REST API
  * Operation
    * Before Register Client
    * After Registered Client
    * Before Update Client
    * After Updated Client
    * View Client
    * Unregister Client

##### Authorization Endpoint
  * Operation
    * Authorization Request

##### Token Endpoint
  * Operation
    * Token Request
    * Token Refresh Request

##### Token Revocation Endpoint
  * Operation
    * Revocation Request by Refresh Token

##### Token Introspection Endpoint
  * Operation
    * Token Request

##### UserInfo Endpoint
  * Operation
    * UserInfo Request

##### Logout Endpoint
  * Operation
    * Logout Request by Refresh Token

### Configuration

Policies, profiles, conditions, executors can be configured on Admin Console. To do so, add "Client Policies" tab on Realm->Realm Settings, which means an administrator can have client policies per realm.

An administrator can create new profile and add executors.

An administrator can create new policy, add conditions and refer to profiles.

An administrator can update their created policies and profiles while they can not update built-in policies and profiles.

### Backward Compatibility

Client Policies can take over Client Registration Policies. However, Client Registration Policies also still co-exist. In the future, Client Registration Policies will be removed and the existing client registration policies will be migrated into new client policies automatically.

The RH-SSO has already supported Client Registration Policies officially, not tech preview while it will treat newly introduced Client Policies as tech preview at first. Therefore, the keycloak will include both Client Registration Policies and Client Policies. After Client Policies being supported officially by RH-SSO, the keycloak drops Client Registration Policies.

### Extensibility

Condition, Executor, Profile and Policy are implemented as Provider. Therefore, developers can implement their own condition, executor, profile and policy by themselves, which makes it easy to realize some security profiles that may come in the future.

## External Interface - Admin REST API

To add/read/modify/delete conditions/executors/policies/profiles, following Admin REST APIs are supported.

The style of API description conforms to [keycloak's API Documentation](https://www.keycloak.org/docs-api/11.0/rest-api/index.html).

### Design Concept

Admin REST API for Client Policies are designed to satisfy the following requirements.

- keycloak admin can view and update the client policy on the editable textarea UI (editing JSON representation directly).
- keycloak admin can control policies, conditions and executors on the ordinal UI (e.g. like the existing Client Registration Policy).

To satisfy them, the following APIs are prepared.
- API that can control the profile and its containing executors.
- API that can control the policy and its containing conditions and references to profiles.
- API that can not control a condition and an executor one by one. The reason why is that the condition and the executor must be included in just one policy and profile. They can not work by themselves and they can not belong to several policies and profiles.

### Definition

#### Profiles

For example, the profiles can be represented as follows.

```
{
    "profiles": [
    {
        "name": "foo",
        "description" : "TBD"
        "builtin": false,
        "executors": [
            "secure-client-authn-executor": {
                "client-authns": [ "private-key-jwt" ]
            }         
        ]
    },
    {    
         "name": "bar",
         "description" : "also TBD"
         "builtin": false,
         "executors": [
            "pkce-enforce-executor": {},
            "secure-session-enforce-executor": {}   
        ]
    }
  ]
}
```
- "profiles" - It can be represented as Array of the Object "profile"

#### Object "profile"

- "name" - The profile's name. It must be unique. It can be represented as String. It can be modified.
- "description" - The human readable description of the profile. It can be represented as String. It can be modified.
- "builtin" - It shows whether this profile is built-in profile or not. It can be represented as Boolean. If it is true, then  this profile is the built-in profile. It can not be modified.
- "executors" : It can be represented as Array of the Object "executor" 

#### Object "executor"

This object consists of the key and its value. This key stands for the executor's provider ID. This value can be represented as Object which contains configurations that depend on each executor.

#### Policies

The policies can be represented as follows.

```
{
    "policies": [{
        "name": "administrator-connect-from-local-network",
        "description" : "as usual TBD"
        "builtin": false,
        "enable": true,
        "conditions": {
                "clientroles-condition": {
                    "roles", ["admin"]
                },
                "client-ipaddr-condition": {
                    "ipaddr", ["10.20.30.40", "192.168.0.0/16"]
                }                
        }
        "profiles": [ "foo", "bar" ]    
    }]
}
```

- "policies" - It can be represented as Array of the Object "policy"

#### Object "policy"

- "name" - The policy's name. It must be unique. It can be represented as String. It can be modified.
- "description" - The human readable description of the policy. It can be represented as String. It can be modified.
- "builtin" - It shows whether this policy is built-in profile or not. It can be represented as Boolean. If it is true, then  this policy is the built-in policy. It can not be modified.
- "enable" - It shows whether this policy is evaluated or not. It can be represented as Boolean. If it is false, this policy is not evaluated. It can be modified.
- "conditions" : It can be represented as Array of the Object "condition" 
- "profiles" : It specifies which profiles are referred and their containing executors work against clients. Such profiles are specified by their provider IDs. It can be represented as Array of String.

#### Object "condition"

This object consists of the key and its value. This key stands for the condition's provider ID. This value can be represented as Object which contains configurations that depend on each condition.

### Resource

#### Retrieve the profiles

```
GET /auth/admin/realms/{realm}/client-policies/profiles
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|

##### Responses

|Code|Description|Schema|
|---|---|---|
|200|successfully retrieved|profiles|
|400|invalid request|-|

#### Update the profiles

```
PUT /auth/admin/realms/{realm}/client-policies/profiles
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Body|rep|required|profiles representation|profiles|

##### Responses

|Code|Description|Schema|
|---|---|---|
|204|successfully updated|-|
|400|invalid request|-|


#### Create the profile

```
POST /auth/admin/realms/{realm}/client-policies/profiles
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Body|rep|required|profile representation|profile|

##### Responses

|Code|Description|Schema|
|---|---|---|
|201|profile created|-|
|400|invalid request|-|

HTTP Response Header
```
Location: {scheme}://{authority}/auth/admin/realms/{realm}/client-policies/profiles/{profile}
```
|Name|Description|Schema|
|---|---|---|
|realm|realm name|string|
|profile|created profile name*1|string|

*1 : created profile name must be URL safe so that it is encoded in percent encoding.


#### Retrieve the profile

```
GET /auth/admin/realms/{realm}/client-policies/profiles/{profile}
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Path|profile|required|profile name|string|

##### Responses

|Code|Description|Schema|
|---|---|---|
|200|successfully retrieved|profile|
|400|invalid request|-|
|404|profile not found|-|


#### Update the profile

```
PUT /auth/admin/realms/{realm}/client-policies/profiles/{profile}
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Path|profile|required|profile name|string|
|Body|rep|required|profile representation|profile|

##### Responses

|Code|Description|Schema|
|---|---|---|
|204|successfully updated|-|
|400|invalid request|-|
|404|profile not found|-|

#### Delete the profile

```
DELETE /auth/admin/realms/{realm}/client-policies/profiles/{profile}
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Path|profile|required|profile name|string|

##### Responses

|Code|Description|Schema|
|---|---|---|
|204|successfully deleted|-|
|400|invalid request|-|
|404|profile not found|-|

#### Retrieve the policies

```
GET /auth/admin/realms/{realm}/client-policies/policies
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|

##### Responses

|Code|Description|Schema|
|---|---|---|
|200|successfully retrieved|policies|
|400|invalid request|-|


#### Update the policies

```
PUT /auth/admin/realms/{realm}/client-policies/policies
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Body|rep|required|policies representation|policies|

##### Responses

|Code|Description|Schema|
|---|---|---|
|204|successfully updated|-|
|400|invalid request|-|


#### Create the policy

```
POST /auth/admin/realms/{realm}/client-policies/policies
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Body|rep|required|policy representation|policy|

##### Responses

|Code|Description|Schema|
|---|---|---|
|201|policy created|-|
|400|invalid request|-|

HTTP Response Header
```
Location: {scheme}://{authority}/auth/admin/realms/{realm}/client-policies/policies/{policy}
```
|Name|Description|Schema|
|---|---|---|
|realm|realm name|string|
|profile|created policy name*2|string|

*2 : created policy name must be URL safe so that it is encoded in percent encoding.


#### Retrieve the policy

```
GET /auth/admin/realms/{realm}/client-policies/policies/{policy}
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Path|policy|required|policy name|string|

##### Responses

|Code|Description|Schema|
|---|---|---|
|200|successfully retrieved|policy|
|400|invalid request|-|
|404|policy not found|-|


#### Update the policy

```
PUT /auth/admin/realms/{realm}/client-policies/policies/{policy}
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Path|policy|required|policy name|string|
|Body|rep|required|policy representation|policy|

##### Responses

|Code|Description|Schema|
|---|---|---|
|204|successfully updated|-|
|400|invalid request|-|
|404|policy not found|-|


#### Delete the policy

```
DELETE /auth/admin/realms/{realm}/client-policies/policies/{policy}
```

##### Parameters

|Type|Name|Requirement|Description|Schema|
|---|---|---|---|---|
|Path|realm|required|realm name|string|
|Path|policy|required|policy name|string|

##### Responses

|Code|Description|Schema|
|---|---|---|
|204|successfully deleted|-|
|400|invalid request|-|
|404|policy not found|-|


### Implementation Consideration

This section shows the points need to be considered when implementing these Admin REST APIs.

#### External/Internal Representation

Unlike the existing providers, both external and internal representation of client policies are the same, namely JSON representation.

#### How to manage JSON representation of client policies

There are two types of the JSON representation of client policies, Read & Write and Read Only.

Generally, the built-in profiles and policies are Read Only type while others are Read & Write type.

The basic strategy of storing data is that the only Read & Write type data is stored in the DB because we want to avoid DB access as possible as we can in order to improve the performance especially in Cross-DC environment.

##### Managing Read & Write data

The Read & Write type JSON representation is stored as the realm's attribute. The reason why is that we want to avoid changing DB's schema and accompanied DB migration from older version of keycloak.

###### Storing Unit

The profiles are stored as the realm's attribute with the key "client-policies.profiles".

The policies are stored as the realm's attribute with the key "client-policies.policies".

###### Initial State

On the first time of booting keycloak, both attribute's are empty.

##### Managing Read Only data

The Read Only type JSON representation is stored as the file "keycloak-default.json" and enclosed in keycloak-services project's jar file. The reason why is that we want to prevent the keycloak's administrator from modifying Read Only data. If doing so, keycloak would not work properly. Also If we change the built-in profiles and policies in the new version of keycloak, explicit migration process for Client Policies feature from the older version of keycloak is not needed.

##### Managing Half and Half data

The built-in profile is totally Read Only while the built-in policy is not. The reason why is that the built-in policy can also be enabled/disabled by its field "enabled" the same as for the ordinal policy. It can be modified. Therefore, the following strategy is taken.

- The built-in policy is stored and managed in the file. This policy's "enabled" field is fixed as "true".
- Only "name" and "enabled" fields are stored and managed in the realm's attribute. "name" field is read only but it is needed to identify the policy.
- "enabled" field of the build-in policy stored in the realm's attribute overrides the one stored in the file.

#### Exporting Client Policies in JSON formatted file

In exported JSON file for realm setting, only Read and Write type profile and policy (namely, "builtin" field is false) is exported. 

As for the built-in policy, the fields stored on the realm's attribute are only exported (namely, "name" and "enabled" fields).

#### Importing Client Policies in JSON formatted file

Only Read and Write type profile and policy (namely, "builtin" field is false) is imported. 

As for the built-in policy, the fields stored on the realm's attribute can only be imported (namely, "name" and "enabled" fields).

#### Creating Client Policies

Only Read and Write type profile and policy (namely, "builtin" field is false) can be created. The keycloak's administrator can not create the built-in profile and policy.

#### Updating Client Policies

When updating client policies, the field that is not defined on each Admin API specification at the former section is ignored.

Only Read and Write type profile and policy (namely, "builtin" field is false) can be updated.

#### Deleting Client Policies

Only Read and Write type profile and policy (namely, "builtin" field is false) can be deleted. The keycloak's administrator can not delete the built-in profile and policy.




