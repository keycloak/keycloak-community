# Client Policies

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-13933](https://issues.jboss.org/browse/KEYCLOAK-13933)

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

Client Policies realize the following three points mentioned in Motivataion as follows.

### 1. Setting policies on what configuration a client can have

The current keycloak has already supported them as [Client Registration Policies](https://www.keycloak.org/docs/latest/securing_apps/index.html#client-registration-policies).

However, it can only cover OIDC Dynamic Client Registration.

Client Policies cover not only what Client Registration Policies can do but other client registration and configuration ways.

In the future, Client Registration will be replaced by Client Policies.

### 2. Validation of client configurations

The current keycloak has already supported validation on whether the client follows settings like Proof Key for Code Exchange, Request Object Signing Algorithm, Holder-of-Key Token, etc on some endpoints like Authorization Endpoint, Token Endpoint, etc.

These can be specified by each setting item (on Admin Console, switch, pulldown menu, etc). To make the client application secure, user need to set a lot of such settings in appropriate way, which makes it difficult for the user to secure their client application.

Client Policies can do these validation of client configurations mentioned just above. In the futre, these each setting items will be replaced by Client Policies. 

The pre-set client policies for following some security profiles (FAPI, SPA, Native App, etc) are provided, which makes it easy for the user to secure their client application.

### 3. Conformance to a profile such as FAPI

It is treated in [Client Conformance Profiles](https://issues.redhat.com/browse/KEYCLOAK-11612) based on this Client Policies. Which kind of security profiles are supported and how to realize them are discussed on 
[Client Conformance Profiles](https://issues.redhat.com/browse/KEYCLOAK-11612).

## Scope

This client policy concept is independent of any specific protocol. However, this document only deals with the implementation for OIDC.

## Implementation details

Client Policies consists of the four building blocks, Policy, Condition, and Executor.

### Condition : which client

A condition determins to which client a policy is adopted. The condition checks whether one specified criteria is satisfied. Such the task is that checking whether the [access type](https://www.keycloak.org/docs/latest/server_admin/index.html#oidc-clients) of the client is confidential.

#### How to implement

A condition can be implemented as a provider (`ClientPolicyConditionSpi`, `ClientPolicyConditionFactory`, `ClientPolicyCondition`). Developers can add their own condition by implementing this provider.

#### How to configure

A condition can be configuable the same as other configurable provider. What can be configured depends on each condition's nature.

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

An executor specifies what action is executed on a client to which a policy is adopted. The executor executes one or several specified actions. Such the action is checking whether the value of the parameter "redirect_uri" in the authorization request matches exactly with one of the pre-registered redirect URIs on Authorization Endpoint and rejecting this request if not. 

#### How to implement

An executor can be implemented as a provider (`ClientPolicyExecutorSpi`, `ClientPolicyExecutorFactory`, `ClientPolicyExecutor`). Developers can add their own executor by implementing provider.

#### How to configure

An executor can be configuable the same as other configurable provider. What can be configured depends on each executor's nature.

#### What events

An executor acts on the follwing events:

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


### Policy : how to determine which client and what action

A policy consists of several conditions and executors. A policy can be adpoted to a client satisfying all conditions of this policy. All executors execute their task against this client.

Considering both a condition and an executor do their task independently from other conditions and other executors, the administrator can not specify in which order conditions and executors are evaluated.

#### Pre-set Policy

A pre-set policy is a client policy that have registered in keycloak as default.
This pre-set policy can be used to enforce basic security level to all clients.

### Endpoint and Opeation : when and where policy is adopted

Conditions and executors of a policy should be evaluated on endpoints where a client application interact with keycloak by OAuth2/OIDC protocol. Considering this point, when and where a policy can be adopted to a client is as follows:

##### OIDC Dynamic Client Registration
  * Opeation
    * Register
    * Update
  * Authentication
    * Authenticated (Using Initial Access Token, Registration Access Token)
    * Anonymous

##### Admin REST API
  * Opeation
    * Register
    * Update

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

Policies, conditions, executors can be configured on Admin Console. To do so, add "Client Policies" tab on Realm->Realm Settings, which means an administrator can have client policies per realm.

An administrator can create new policy and add conditions and executors.
An administrator can update their created policies by add/modify/delete conditions and executors while the administrator can not update pre-set policies.

### Backward Compatibility

Client Policies can take over Client Registration. However, Client Registration Policies also still co-exist. In the future, Client Registration Policies will be removed and the existing client registration policies will be migrated into new client policies automatically.

### Extensibility

Condition, Executor and Policy are implemented as Provider. Therefore, developers can implement their own condition, executor and policy by themselves, which makes it easy to realize some security profiles that may come in the future.

