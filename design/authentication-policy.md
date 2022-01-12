# Authentication Policy
* **Status**: Draft #1

## Motivation
Configuring the factors used in an authentication process requires changing the authentication flows for a realm, which means applying an advanced feature to common use cases. It also means these requirements cannot vary within a realm without creating custom authenticator implementations, an even more advanced feature. Likewise, identity providers are configured at the realm level and cannot be exposed selectively without extensions.

The goal of authentication policy is to expose this configuration in easy to use menus and allow it to be associated with entities like groups and roles so these settings can be applied conditionally.

## Scope
This first pass at authentication policy is intended to establish the internal framework for how policies can be identified and applied based on user properties and cover two crucial areas:
* Which second factors are required and how required they are
* Identity provider restrictions and automatic redirects (see related discussion on automatic redirects based on user properties: https://github.com/keycloak/keycloak/discussions/8839)

Future implementation phases may expand what can be controlled by policy beyond the items mentioned above. Some candidates include:
* Which first factors, including non-interactive ones like Kerberos, may be used
* Fine-grained configuration of authentication factors (ex: having different OTP or WebAuthn policies for subsets of users)

## Design Proposal
An authentication policy is a set of criteria (the entities to which the policy applies) and a collection of policy elements - the rules - that form the policy content. Each of these rules consumes some configuration and applies it to authentication flows and other Keycloak behaviors in an opinionated way so that a user doesn't have to configure details that can be implied (see the tables in the Policy Application section for examples.) Advanced users will also be able to create new rule types to address cases not accounted for by built-in rule types.

Steps of an authentication flow opt in to being controlled by policy by using a new POLICY_BASED requirement level that indicates to Keycloak that a policy must be evaluated before that step can be processed. The result of the policy evaluation will replace the requirement level and may also alter the config map passed to the authenticator.

If there are policies for a realm that reference user data (as most policies will) and a user is not yet known when the first POLICY_BASED authenticator is reached, the user will be prompted to identity themselves. Exactly how this will work has not been specified, but it may reuse an existing form (ex: username form) or introduce a new one.

To get this going, there are three main areas of concern: how authentication policies are specified, how Keycloak identifies which policy applies to a given situation, and how the content of that policy affects Keycloak’s behavior.

### Policy Definition
Authentication policies are a new entity that will be accessible and mutable via the administrative REST API and exposed in the administrative UI with both form-based and JSON editors. Policies will belong to a realm and can additionally be connected to other entities within that realm to control when they apply. Additionally, conflicts between policies (ex: when multiple policies apply at a given moment) will be resolved by priority order (see Policy Selection below.)

The following properties are common to all policies, and many will be used directly by the policy engine:

Name|Value(s)|Description
-|-|-
name|Text|The name of the policy
description|Text|A more detailed description of the policy
enabled|On/Off|Whether or not this policy should be in effect
group|(Optional) Name of the associated group|This policy will be a candidate (see Policy Selection below) for any user who is a member of the associated group. 
role|(Optional) Name of the associated role|This policy will be a candidate (see Policy Selection below) for any user who is a member of the associated role
email_domain|(Optional) The domain portion of an email address (ex: acme.com) to match against.|Using this option in any policy requires an email address to be known (ex: with an early prompt) during policy selection. This policy will be a candidate for any user with an email address @ the given domain. (See Policy Selection section below.)
priority|Signed integer|Controls the precedence order when multiple policies apply to a given situation (see Policy Selection.) Lower numbers take precedent.
rules|Object array|Each element must have a “type” field which will be used to resolve the policy implementation at runtime. The rest of the element must conform to that policy type’s specification. Builtin types are described below.

Additionally, the configuration available to a given rule/policy type can extend these properties. The initial builtin types follow:

Idp Policy - allows users to configure which (if any) identity providers are available in a policy and whether or not users must log in with them:
Name|Value(s)|Description
-|-|-
requirement|ALLOWED / REQUIRED / DISABLED|See: Policy Application > Idp Policies below
idps|Array of strings|The aliases of the identity providers included in the policy. An empty array indicates all of the identity providers for the realm.

Two-Factor Policy - allows users to configure which second factors are available and whether users are required to use them:
Name|Value(s)|Description
-|-|-
requirement|ALLOWED / REQUIRED / DISABLED|See: Policy Application > Two-Factor Policies below
factors|Array of objects, each with an type property referring to a the name of a credential type|The credential types controlled by this policy

Data-Driven Policy - allows users to override an authentication flow for a subset of users. This provides an alternative way to add dynamic behavior to a flow without creating a new policy type unless the extra complexity is warranted (ex: if the policy type should affect several authenticators in some complex way based on relatively little configuration.)

Name|Value(s)|Description
-|-|-
authenticators|Array of objects (authenticator alias, requirement, and config map)|If an authenticator matches the given alias, its requirement is set to the given requirement, and the items in the config map upsert into the authenticator's config for this execution only

For example, a policy that requires members of the admin role to use or enroll in OTP could be described with the following json:
```json
[
 {
   "name":"Admin role two factor",
   "description": "Two factor for acme group",
   "role": "admin",
   "rules": [
     {
       "type":"two-factor",
       "requirement": "REQUIRED",
       "factors":[{"type":"OTP"}]
     }
   ],
   "priority":1
 }
]
```

### Policy Selection
Once policies have been defined for a realm, Keycloak needs to be able to identify which policy applies to a given decision. A policy is a *candidate* if it's the default or has a relationship to the current user (ex: group membership.) 

If there are multiple candidates, the policy with the lowest priority value (greatest importance) applies.

### Policy Application
Policies must be simple to configure while having the capability to apply that configuration to a wide variety of situations and cases. This creates a need for a separation between the user's configuration surface (the configuration input described above) and the internal SPI where the configuration is applied.

To control authentication flows, any policy type will be able to change the requirement of an authenticator it applies to and will be able to inject variable configuration data for the authenticator's config map. This allows a lot of fine-grained control over the authentication flow, which should be encapsulated within the policy type itself. The exception to this is the data-driven policy type, which allows users to supply the configuration they want verbatim.

Other SPI interfaces that extend the authentication policy interface will likely be necessary to expose specific behaviors outside of authentication flows. Consider the requirements for the idp policy described below or the need to control access to managing different credential types based on applicable policy. Some of these details are still being worked on separately from this proposal.

#### Identity Provider Policies
The goal of an identity provider (idp) policy is to describe the relationship between a set of users and a set of identity providers. This relationship is defined by its requirement level:
* ALLOWED: users may use one of the specified idps to log in. Other first factors are permitted per realm configuration.
* REQUIRED: users must use one of the specified idps to log in. No other first factors are permitted. If only one idp is specified, users will be redirected to that idp as soon as their username is known (whether via identity-first, a login hint, or otherwise.)
* DISABLED: users must not use any idp to log in. Other first factors are permitted per realm configuration.

Additionally, an idp policy may specify a subset of the identity providers of the realm to ALLOW/REQUIRE (see Policy Definition above.)

This policy would then be evaluated by and affect various parts of the login process and of Keycloak’s behavior.
* Users with exactly one REQUIRED idp will be redirected to that idp immediately after identification.
* The login page will show only a subset of options depending on policy
    * If there are REQUIRED idps, those idps will be shown but no password or password reset options
    * If there are ALLOWED idps, those idps will be shown alongside the password and password reset options
    * If the idp policy is DISABLED, no idps will be shown, but password and password reset options still will.
* When attempting to link to an existing user (first broker login flow), only users who are ALLOWED or REQUIRED to log in with the given broker are valid options
* After login (post broker login flow), the user’s policy will be checked to ensure..
    * They are still allowed to use that idp (consider idp-initiated login after a policy has been changed to disallow that provider)
    * Any existing idp links that are no longer allowed by the policy are deleted
* When requesting password resets
    * If an identity broker is present (first broker login flow), password resets will only be allowed against users who are ALLOWED or REQUIRED to log in with that provider
    * If no identity broker is present (browser flow), password resets will only be allowed against users who are not REQUIRED to log in with a provider.
* When working with the account console, only idps that are ALLOWED or REQUIRED are shown. If idp policy is DISABLED, no Connected Accounts section is shown.

#### Two-Factor Policies
Two-Factor policies are intended to describe the relationship between a set of users and potential second-factors. This allows subsets of a realm’s factors to be exposed to subsets of users and to automatically compel a subset of users to enroll. As with idp policies, the nature of this relationship is defined by its requirement level:
* ALLOWED: the users may enroll in the referenced factors and, having done so, will be challenged on them on subsequent logins.
* REQUIRED: the users must enroll if they haven’t already and, having done so, will be challenged to use them on subsequent logins.
* DISABLED: the users will not be able to enroll in, nor will they be challenged on previously enrolled, factors.

This is represented in the following table:

||Login - Enrolled|Login - Not Enrolled|Account Console
-|-|-|-
ALLOWED|Challenge|Skip|Allow enrollment and removal
REQUIRED|Challenge|Enroll|Allow enrollment but cannot remove all indicated factors
DISABLED|Skip|Skip|Do not allow any management/do not show tab

This policy must be evaluated during a login flow when determining which, if any, 2nd factors to challenge/enroll and in the account console when determining which 2nd factors can be managed and how.

## Use cases
* Users in group acme are REQUIRED to login with acme.com idp.
* Users in group acme are REQUIRED  to authenticate with any second factor.
* ALL users are ALLOWED to authenticate with any second factor.(Keycloak default behavior)
* Users from email domain acme.com are REQUIRED to login with acme idp.
* Users with admin role are REQUIRED to authenticate with OTP second factor.

## Examples

Users in group acme are REQUIRED to login with acme.com idp.

```json
[
 {
   "name":"Acme Idp",
   "description": "Acme company idp",
   "group": "acme",
   "rules": [
     {
       "type": "idp",
       "requirement": "REQUIRED",
       "idps":[{"alias":"acme.com"}]
     }
   ],
   "priority":0
 }
]
```

Users in group acme are REQUIRED  to authenticate with any second factor.

```json
[
 {
   "name":"Acme two factor",
   "description": "Two factor for acme group",
   "groups": "acme",
   "rules": [
     {
       "type":"two-factor",
       "requirement": "REQUIRED",
       "factors":[{"type":"OTP"},{"type":"WebAuthn"}]
     }
   ],

   "priority":0
 }
]
```

ALL users are ALLOWED to authenticate with any second factor.(Keycloak default behavior)
```json
[
 {
   "name":"Default Two factor Realm Policy",
   "description": "Default two factor realm policy",
   "rules": [
     {
       "type":"two-factor",
       "requirement": "ALLOWED",
       "factors":[]
     }
   ]
 }
]
```

Users with email domain acme.com are REQUIRED to authenticate with acme idp.

```json
[
 {
   "name":"Acme Idp",
   "description": "Acme company idp",
   "email_domain": "acme.com",
   "rules": [
     {
       "type":"idp",
       "requirement": "REQUIRED",
       "idps":[{"alias":"acme.com"}]
     }
   ],
   "priority":0
 }
]
```

Users with admin role are REQUIRED to authenticate with OTP second factor

```json
[
 {
   "name":"Admin role two factor",
   "description": "Two factor for acme group",
   "role": "admin",
   "authenticators": [
     {
       "type":"two-factor",
       "requirement": "REQUIRED",
       "factors":[{"type":"OTP"}]
     }
   ],
   "priority":1
 }
]
```

Sample authentication policy

```json
{
 "Name": "Acme Policy",
 "Description": "Policy for Acme Organization",
 "group": "acme",
 "rules": [
   {
     "type": "two-factor",
     "requirement": "REQUIRED",
     "factors": [
       {
         "type": "OTP"
       }
     ]
   },
   {
     "type": "idp",
     "requirement": "ALLOWED",
     "idps": [
       {
         "alias": "acme.com"
       }
     ]
   }
 ]
}
```