# W3C Web Authentication - Implementation stages - Authenticator only

* **Status**: Draft
* **Author**: [tnorimat](https://github.com/tnorimat)
* **JIRA**: [KEYCLOAK-9358](https://issues.jboss.org/browse/KEYCLOAK-9358), [KEYCLOAK-9359](https://issues.jboss.org/browse/KEYCLOAK-9359), [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360)

## Motivation

As mentioned in [**W3C Web Authentication - Two-Factor**](https://github.com/keycloak/keycloak-community/blob/master/design/web-authn-two-factor.md), it is important to implement WebAuthn support to keycloak. This design document treats the following three issues mentioned in [**W3C Web Authentication - Two-Factor**](https://github.com/keycloak/keycloak-community/blob/master/design/web-authn-two-factor.md) Implementation stages - Authenticator only:

1. [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) : Two factor authentication with W3C Web Authentication

>  Acceptance criteria:
>
>    * Admin manually registers Web Authentication authenticator to replace OTP authenticator
>    * Admin registers required action for user to register a security key
>    * When user logs in again after registering the security key the user should be prompted to tap the security key after entering username and password

2. [KEYCLOAK-9358](https://issues.jboss.org/browse/KEYCLOAK-9358) : Investigate W3C Web Authentication libraries

>  The selected library should ideally support:
>
>    * MFA and passwordless
>    * Username stored on device (no need to enter username when authenticating)
>    * Attestation to be able to get details about authenticators and limit what authenticators should be supported by a realm
>
>  It should also have:
>
>    * A compatible open source license
>    * A reasonable sized community
>    * Good test coverage
>    * Maven build (Graddle is ok, but not ideal)

3. [KEYCLOAK-9359](https://issues.jboss.org/browse/KEYCLOAK-9359) : Investigate how to test Web Authentication

## Relationship among other design documents
***

This design documents depends on the following two design documents:

1. [**Application Initialted Actions (AIA)**](https://github.com/keycloak/keycloak-community/blob/master/design/application-initiated-actions.md)

It relates to how a user registers their public key credential on keycloak.

Before realizing AIA, the user can register their own public key credential at their login time by Required Action.

After realizing AIA, the user can also do it by AIA that [User Account Service](https://www.keycloak.org/docs/latest/server_admin/index.html#_account-service) also use it.

2. [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md)

One point of its work is to realize multiple credentials management per user. Therefore, it relates to how many public key credentals a user can manages on keycloak.

Before realizing Managing multi-factor authentication and Step-up authentication in Keycloak, a user can manage single public key credential on keycloak. Please note that this single public key credential support is tentative and omitted afterward.

After realizing Managing multi-factor authentication and Step-up authentication in Keycloak, a user can manage multiple public key credentials on keycloak.

It might be possible to support multiple public key credentials per user before realizing Managing multi-factor authentication and Step-up authentication in Keycloak. However, I'm afraid that this design and that do the same thing for refactoring credential management in keycloak. To avoid this situation, this design itself does not treat how to realize managing multiple public key credentials per user.

## Implementation Plan
***

The [WebAuthn API specification](https://www.w3.org/TR/webauthn/) covers broader areas, and this design is affected by other design documents explained just above. Therefore, I'll start by minimun support.

* Support for minimum WebAuthn Registration/Authentication feature - satisfying [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) acceptance criteria
    - Not considering [**AIA**](https://github.com/keycloak/keycloak-community/blob/master/design/application-initiated-actions.md)
        - The user can register their own public key credential at their login time by Required Action.
    - Not considering [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md) 
        - Single public key credential per user
     - Not support attestation Statement verification
     - Not support authenticator metadata aquisition from external sources

After that, I'll gradually implement additional features step by step. I'll consider the following implementation steps:

* Support for public key credential registration by User Account Service after realizing AIA

* Support for Multiple Public Key Credentials per User Account after realizing Managing multi-factor authentication and Step-up authentication in Keycloak

* Support attestation statement verification

* Support authenticator metadata aquisition from external sources (e.g. FIDO Alliance Metadata Services)

## Public Key Credential
***

In the [WebAuthn API specification](https://www.w3.org/TR/webauthn/), a user's credential is called "Public Key Credential". It is created by the user's authenticator (e.g. Security Key) and stored in RP (namely, keycloak). It is used for user authentication.

### How Many Public Key Credential a User Can Manage

At first, a user can manage only single public key credential. Please note that this single public key credential support is tentative and omitted afterward.

After realizing [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md) , the user can manage several public key credentials.

### Managed Information on Public Key Credential

Informanation managed in keycloak as Public Key Credential consists of a public key credential itself and its metadata.

#### Public Key Credential Itself

Public Key Credential itself are as follows :

* Attestation Statement
* Attested Credential Data
  * AAGUID
  * Credential ID
  * Credential Public Key
* Count

Those are all information returned from WebAuthn API call (`navigator.credentials.create()`).
Those are immutable except for Count.

#### Metadata

The metadata of public key credential is to be used for a user to select which public key credential is used for user authentication. Therefore, the metadata is understandable for its user.

* Metadata
  * Label : a text the user can write for identifying this public key credential

When registering the public key credential, the user themselves input some notes in order for them to recoginize it afterwards.

There are two other types of information that seems to be used as the metadata but not used:

* Information returned from WebAuthn API call

After investigating [WebAuthn API Specification](https://www.w3.org/TR/webauthn/#sec-authenticator-data), I found that no information returned from WebAuthn API call (`navigator.credentials.create()`) is appropriate as the metadata. Therefore, this type of information as the metadata.

* Information acquired from external sources 

For example, such information can be acquired from [FIDO Alliance Metadata Services](https://fidoalliance.org/metadata/).

After investigating [FIDO's specification about metadata](https://fidoalliance.org/specs/fido-security-requirements-v1.0-fd-20170524/fido-authenticator-metadata-requirements_20170524.html), it seems to be appropriate for the metadata. However, such metadata can not be gotten for any type of authenticators. AAGUID is used as the key for finding its metadata, but there is the case that there is no metadata found.

## Registration
***

### Definition

"Registration" here means that an user create their public key credential on their authenticator and register it onto keycloak.

### How to Realize

It is realized by a required action provider called "WebAuthn Register". It conforms to the following acceptance criteria in [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) : Two factor authentication with W3C Web Authentication:

>    * Admin registers required action for user to register a security key

Also, to make the browser execute WebAuthenticatin API, javascript and freemarker template are required.

### Actions for Registration

An user without having their user account in keycloak:

* They can register their public key credential along with creating their user account in keycloak.

An existing user having their user account in keycloak:

* They can register their own public key credential at their login time by Required Action.

* After realizing [**AIA**](https://github.com/keycloak/keycloak-community/blob/master/design/application-initiated-actions.md), the user can register their own public key credential it by AIA that User Account Service also use it.

### Registered Public Key Credential's Metadata

As mentioned before, on registering the public key credential, the user can input texts on the single string field to identify this credential afterwards on user authentication.

### Configuration

Registration can be configured by "WebAuthn Register" provider config.

Basically, this design document follows [WebAuthn API Specification](https://www.w3.org/TR/webauthn/#dictionary-makecredentialoptions) for configuration items and their values.

#### Configuration Scope

This configuration is applied to all users using this provider. It suffice to say that this configuration is adopted per realm.

#### Configuration Item - Signature Algorithm

  - Multiple items can be selected.
  - Supported algorithms are the following : {ES256, ES384, ES512, RS1, RS256, RS384, RS512}
  - Default setting is : {ES256}
  - If no algorithm is selected, {ES256} is adapted.

*Notes:*

* These supported algorithms are supported ones in the WebAuthn protocol processing core library [webauthn4j](https://github.com/webauthn4j).
* This setting is **required** for WebAuthn API (`navigator.credentials.create()`)

#### Configuration Item - Authenticator Attachment

  - Have the switch (ON/OFF) to specify whether this configuration is used or not. Its default setting is OFF.
  - Only single item can be selected.
  - Supported authenticator attachment options are the following : {platform, cross-platform}
  - Default setting is : platform

*Notes:*

* This setting is optional for WebAuthn API (`navigator.credentials.create()`). Therefore, if the switch if OFF, the executor of this WebAuthn API does not filter out authenticators by their attachment type.

#### Configuration Item - Require Resident Key

  - Have the switch (ON/OFF) to specify whether this configuration is used or not. Its default setting is OFF.
  - Only single item can be selected.
  - Supported resident key options are the following : {Yes, No}
  - Default setting is : No

*Notes:*

* This setting is optional for WebAuthn API (`navigator.credentials.create()`). Therefore, if the switch if OFF, the executor of this WebAuthn API does not filter out authenticators by their resident key capability.

#### Configuration Item - User Verification Requirement

  - Have the switch (ON/OFF) to specify whether this configuration is used or not. Its default setting is OFF.
  - Only single item can be selected.
  - Supported user verification options are the following : {required, preferred, discouraged}
  - Default setting is : preferred

*Notes:*

* This setting is optional for WebAuthn API (`navigator.credentials.create()`). Therefore, if the switch if OFF, the executor of this WebAuthn API does not filter out authenticators by their user verification capability.

#### Configuration Item - Attestation Conveyance Preference

  - Have the switch (ON/OFF) to specify whether this configuration is used or not. Its default setting is OFF.
  - Only single item can be selected.
  - Supported attestation conveyance preference are the following : {none, indirect, direct}
  - Default setting is : none

*Notes:*

* This setting is optional for WebAuthn API (`navigator.credentials.create()`). If the switch if OFF, the executor of this WebAuthn API does its work as the default setting "none" is specified.

* Unless Support Attestation Statement Verification is implemented, the Attestation Conveyance Preference value is fixed as "none". If so, the executor of this WebAuthn API (`navigator.credentials.create()`) returns AAGUID as all 0 filled. Therefore, in this situation, keycloak can not get the metadata from FIDO Alliance Metadata Service.

### Filtering out Users' Authenticators

This section relates to the criteria mentioned in [KEYCLOAK-9358](https://issues.jboss.org/browse/KEYCLOAK-9358) : Investigate W3C Web Authentication libraries.

>    * Attestation to be able to get details about authenticators and limit what authenticators should be supported by a realm

There is such the use case that the administrator enforce users to use only the biometric authenticator (e.g. fingerprint), and they want keycloak to reject the authenticator without biometric authentication capability.

In order to do that, keycloak needs to know the authenticator's capability. It seems to be impossible to do that unless keycloak get the detailed metadata. However, information returned from WebAuthn API (`navigator.credentials.create()`) does not contain such the detailed information.

#### by AAGUID

AAGUID list is set up as a configuration item. Filtering users' authenticators works as follows:

* Get information returned from WebAuthn API (`navigator.credentials.create()`)
* Check whether AAGUID contained in this information matches one in AAGUID list
  * If matches, register a public key credential based on this information.
  * If not, returns an error telling a user .

This AAGUID list works as white list.


#### by Metadata acquired from extrenal sources

At the first step of implementation, it is out of scope. After realizing support authenticator metadata aquisition from external sources (e.g. FIDO Alliance Metadata Services), I'll consider filtering users' authenticators based on such the metadata.

### Open Issues

User Interface, Logging, Error Handling need to be considered.

## Authentication
***

### Definition

"Authentication" here means that a user use their public key credential as credential of their user authentication.

### How to Realize

It is realized by an authenticator provider called "WebAuthn Authenticator". It conforms to the following acceptance criteria in [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) : Two factor authentication with W3C Web Authentication:

>    * Admin manually registers Web Authentication authenticator to replace OTP authenticator

It is an authenticator provider. Therefore, by adding this authenticator provider after existing username authenticator provider, it conforms to the following acceptance criteria in [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) : Two factor authentication with W3C Web Authentication:

>    * When user logs in again after registering the security key the user should be prompted to tap the security key after entering username and password

Also, to make the browser execute WebAuthenticatin API, javascript and freemarker template are required.

### Configuration

Authentication can be configured by "WebAuthn Authenticator" provider config. 

Basically, this design document follows [WebAuthn API Specification](https://www.w3.org/TR/webauthn/#assertion-options) for configuration items and their values.

#### Configuration Scope

This configuration is applied to all users using this provider. It suffice to say that this configuration is adopted per realm.

#### Configuration Item - User Verification Requirement

  - Have the switch (ON/OFF) to specify whether this configuration is used or not. Its default setting is OFF.
  - Only single item can be selected.
  - Supported user verification options are the following : {required, preferred, discouraged}
  - Default setting is : preferred

*Notes:*

* This setting is optional for WebAuthn API (`navigator.credentials.get()`). Therefore, if the switch if OFF, the executor of this WebAuthn API does not filter out authenticators by their user verification capability.

### Specifying Public Key Credentials

User can select which their public key credentials are used for their authentication.

On user authentication:

  * Keycloak shows the list of public key credentials with its metadata that the user has registered.
  * The user can select some of their public key credentials.
  * It is acceptable that the user selects no public key credentials.

*Notes:*

* This setting is optional for WebAuthn API (`navigator.credentials.get()`). However, it must be needed if the user has registered their public key credential by the authenticator without Resident Key capability. Such the authenticator needs the information sent by this option. 

### Open Issues

User Interface, Logging, Error Handling need to be considered.

## Credential Management
***

### Definition

"Credential Management" here means that a user and administrator can do the following things :

* Register the new public key credential onto keycloak by the user.

* Delete the existing public key credential stored in keycloak by the user and administrator.

* View information on the existing public key credential stored in keycloak by the user and administrator.
    - Attestation Statement
    - Attested Credential Data
      - AAGUID
      - Credential ID
      - Credential Public Key
    - Count
    - Metadata
      - Label : a text the user can write for identifying this public key credential

* Edit information on the existing public key credential stored in keycloak by the user and administrator
    - Metadata
      - Label : a text the user can write for identifying this public key credential

### Management by user

After realizing AIA, the user can also manage their own public key by [**AIA**](https://github.com/keycloak/keycloak-community/blob/master/design/application-initiated-actions.md) that User Account Service also use it.

### Management by administrator

The administrator can manage all user's public key credentials on Admin Console 
(Users -> [user] -> Credentials tab) which implies that it can be done by Admin REST API.

### Open Issues

User Interface, Logging, Error Handling need to be considered.

## Attestation Statement Verification
***

*This feature are out of scope at the first implementation phase.*

In order to verify Attestation Statement, keycloak needs to get its trust anchor's certificate. It is plausible to get the trust anchor's certificate in the following ways :

* obtained from FIDO Alliance Metadata Services.

* obtained from an authenticator's vendor.

When this feature is to be supported, it is needed to consider the above two ways to get the trust anchors' certificates.

## Authenticator Metadata Acquisition from External Sources
***

*This feature are out of scope at the first implementation phase.*

When this feature is to be supported, it is needed to consider getting public key credential's metadata from FIDO Alliance Metadata Services (MDS).

## WebAuthn Extensions
***

Any kind of WebAuthn API Extensions are out of scope on this design.

## Libraries
***

To process WebAuthn protocol, [webauthn4j](https://github.com/webauthn4j/webauthn4j) can be used by considering the following acceptance criteria [KEYCLOAK-9358](https://issues.jboss.org/browse/KEYCLOAK-9358) mentioned:

>  The selected library should ideally support:
>
>    * MFA and passwordless
>    * Username stored on device (no need to enter username when authenticating)
>    * Attestation to be able to get details about authenticators and limit what authenticators should be supported by a realm

These criteria can be satisfied by using [webauthn4j](https://github.com/webauthn4j/webauthn4j). 

>  It should also have:
>
>    * A compatible open source license

[webauthn4j](https://github.com/webauthn4j/webauthn4j/blob/master/LICENSE.txt) is Apache License 2.0.

>    * A reasonable sized community

[webauthn4j's community](https://github.com/webauthn4j/) is still a relatively small community, but we library user side and webauthn4j's originator and maintainer [ynojima](https://github.com/ynojima) can communicate smoothly and continue making contributions each other.

>    * Good test coverage

[Webauthn4j](https://sonarcloud.io/component_measures?id=webauthn4j&metric=coverage&view=list) shows relatively high test coverage. In addition to that, webauthn4j has passed the comformance test of all mandatory test cases and optional Android Key attestation test cases of [FIDO2 Test Tools provided by FIDO Alliance](https://fidoalliance.org/certification/functional-certification/conformance/).

>    * Maven build (Graddle is ok, but not ideal)

[webauthn4j](https://github.com/webauthn4j/webauthn4j/) is Gradle build.

## Testing
***

For automated functional tests for the integration, [Web Authentication Testing API](https://docs.google.com/document/d/1bp2cMgjm2HSpvL9-WsJoIQMsBi1oKGQY6CvWD-9WmIQ/edit#) can be used.

It has been confirmed as preliminary analysis that [Web Authentication Testing API](https://docs.google.com/document/d/1bp2cMgjm2HSpvL9-WsJoIQMsBi1oKGQY6CvWD-9WmIQ/edit#) can be integrated into keycloak's Arquillian integration testing framework and basic functional tests of registration and authentication flow has been realized.

## Acknowledgements
***

webauthn4j's originator and maintainer [ynojima](https://github.com/ynojima) gave fruitful comments and advice on this design document.


