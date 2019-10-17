# W3C Web Authentication - Implementation stages - Authenticator only

* **Status**: Draft #2
* **JIRA**: [KEYCLOAK-9358](https://issues.jboss.org/browse/KEYCLOAK-9358), [KEYCLOAK-9359](https://issues.jboss.org/browse/KEYCLOAK-9359), [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360)

## Motivation

As mentioned in [**W3C Web Authentication - Two-Factor**](https://github.com/keycloak/keycloak-community/blob/master/design/web-authn-two-factor.md), it is important to implement WebAuthn support to keycloak. This design document treats the following three issues mentioned in this - Implementation stages - Authenticator only:

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

1. [**Application Initiated Actions (AIA)**](https://github.com/keycloak/keycloak-community/blob/master/design/application-initiated-actions.md)

It relates to how a user registers their public key credential on keycloak.

Before realizing AIA, the user can register their public key credential at their login time by Required Action.

After realizing AIA, the user can also do it by AIA that [User Account Service](https://www.keycloak.org/docs/latest/server_admin/index.html#_account-service) also use it.

2. [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md)

One point of its work is to realize multiple credentials management per user. Therefore, it relates to how many public key credentials a user can manages on keycloak.

Before completing this work, a user can manage single public key credential on keycloak. Please note that **this single public key credential support is tentative and omitted afterward**.

After completing this work, a user can manage multiple public key credentials on keycloak.

It might be possible to support multiple public key credentials per user before completing this work. However, if so, this design document and that work do the same thing for refactoring credential management. To avoid this situation, this design document itself does not treat support for multiple public key credentials per user.

## Implementation Plan
***

The [WebAuthn API specification](https://www.w3.org/TR/webauthn/) covers broader areas, and this design document is affected by others explained just above. Therefore, the WebAuthn support should be started by minimum support.

### First Implementation Phase

The first implementation phase covers the following :

* Support for minimum WebAuthn Registration/Authentication feature - satisfying [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) acceptance criteria
    - Not considering [**AIA**](https://github.com/keycloak/keycloak-community/blob/master/design/application-initiated-actions.md)
        - The user can register their public key credential at their login time by Required Action.
    - Not considering [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md)
        - Single public key credential per user
     - Not support attestation Statement verification
     - Not support WebAuthn authenticator metadata acquisition from external sources

This [merged pull request](https://github.com/keycloak/keycloak/pull/6248/) has realized this phase.

### Second Implementation Phase

The second implementation phase covers the following :

* Support attestation statement verification - satisfying [KEYCLOAK-11372](https://issues.jboss.org/browse/KEYCLOAK-11372) requirements.
  - Support KeyStore file used as Trust Store for Attestation Statement Trustworthiness Validation.
  - Not support using FIDO Alliance Metadata Services(MDS)

### Future Implementation Items

The following additional features should be realized step by step:

#### Credential Management related issue

* Edit WebAuthn authenticator credential's metadata by Admin
  - Managed the existing ticket [**Allow admin to manage two factor authenticators for a user**](https://issues.jboss.org/browse/KEYCLOAK-9694).
  - This issue can be worked on after realizing new Account Console and Account REST API.

* Delete registered WebAuthn authenticator credential by User
  - Managed the existing ticket [**Manage two factor authentication through account management console**](https://issues.jboss.org/browse/KEYCLOAK-9367).
  - This issue can be worked on after realizing new Account Console and Account REST API.

* Manage WebAuthn authenticator's metadata by WebAuthn Credential Provider
  - This issue can be worked on after refactoring Credentials in [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md).

* Support for Multiple Public Key Credentials per User Account
  - This issue can be worked on after refactoring Credentials in [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md).

* Support for public key credential registration by User Account Service
  - This issue can be worked on after realizing new Account Console and Account REST API and AIA

* Support WebAuthn authenticator metadata acquisition from external sources (e.g. FIDO Alliance MDS)

#### Configuration related issue

* Fine-grained scope for WebAuthn Registration and Authentication configuration

#### Authentication related issue

* Support using FIDO Alliance MDS used for Attestation Statement Trustworthiness Validation.

#### Performance related issue

* Avoid checking and updating WebAuthn authenticator credential's counter
  - This issue can be worked on after refactoring Credentials in [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md).

* Cache WebAuthn authenticator's credential
  - This issue can be worked on after refactoring Credentials in [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md).


### Breaking Change between Implementation Phases

This WebAuthn support is still in develomplent. Therefore, there is a chance that registered public key credentials and its related information on some implementation phase can not taken up to the succeeding implementation phase, also the specification and user interfaces may change.

## Public Key Credential
***

In the [WebAuthn API specification](https://www.w3.org/TR/webauthn/), a user's credential is called "Public Key Credential". It is created by the user's WebAuthn authenticator (e.g. Security Key) and stored in RP (namely, keycloak). It is used for WebAuthn user authentication by process.

Theoretically, it is possible that several public key credentials can be registered by one WebAuthn authenticator. However, it seems to be common that one public key credential can be registered by one WebAuthn authenticator. Therefore, in this design document, "register a WebAuthn authenticator" and "register a public key credential" are used as the same meaning interchangeably.

### How Many Public Key Credential a User Can Manage

At first implementation phase, a user can manage only single public key credential. Please note that **this single public key credential support is tentative and omitted afterward**.

At first implementation phase, the features about Public Key Credential (e.g. credential selection on WebAuthn authentication) are implemented the same as the implementation phase when multiple credentials are supported in order that make it easy to work on later implementation phases (e.g. multiple credentials per user supported ).

It seems to be odd for users that the features supposing multiple credentials are implemented at this first implementation phase. But this single public key credential is tentative and not officially supported later.

After realizing [**Managing multi-factor authentication and Step-up authentication in Keycloak**](https://github.com/keycloak/keycloak-community/blob/master/design/multi-factor-admin-and-step-up.md), the user can manage several public key credentials.

### Managed Information on Public Key Credential

Information managed in keycloak as Public Key Credential consists of a public key credential itself and its metadata.

#### Public Key Credential Itself

Public Key Credential the RP(keycloak) stores and manages are as follows :

* [Attested Credential Data](https://www.w3.org/TR/webauthn/#sec-attested-credential-data)
  * [AAGUID](https://fidoalliance.org/specs/fido-v2.0-rd-20180702/fido-metadata-statement-v2.0-rd-20180702.html#authenticator-attestation-guid-aaguid-typedef)
  * [Credential ID](https://www.w3.org/TR/webauthn/#credential-id)
  * [Credential Public Key](https://www.w3.org/TR/webauthn/#credential-public-key)
* [Count](https://www.w3.org/TR/webauthn/#signcount)

Those are information returned from WebAuthn API call (`navigator.credentials.create()`) on WebAuthn registration except Attestation Statement. Those are immutable except for a count.

The reason why it does not include an attestation statement is as follows :

* It is not used when WebAuthn authentication.
* Its size tends to big because it may include a certificate chain. 

#### Metadata

The metadata of public key credential is to be used for a user to select which public key credential is used when WebAuthn user authentication. Therefore, the metadata is understandable for its user.

* Metadata
  * Label : a text the user can write for identifying this public key credential

When registering the public key credential, the user themselves input some texts in order to recognize it afterwards.

##### Metadata candidates consideration

There are two other types of information that seems to be used as the metadata but not used:

* Information returned from WebAuthn API call

After investigating [WebAuthn API Specification](https://www.w3.org/TR/webauthn/#sec-authenticator-data), no information is returned from WebAuthn API call (`navigator.credentials.create()`) that is appropriate as the metadata. Therefore, this type of information is not used as the metadata.

* Information acquired from external sources 

For example, such information can be acquired from [FIDO Alliance Metadata Services](https://fidoalliance.org/metadata/).

After investigating [FIDO's specification about metadata](https://fidoalliance.org/specs/fido-security-requirements-v1.0-fd-20170524/fido-authenticator-metadata-requirements_20170524.html), it seems to be appropriate for the metadata. However, such metadata can not be gotten for any type of WebAuthn authenticators. AAGUID is used as the key for finding its metadata, but there is the case that there is no metadata found.

## Registration
***

### Definition

"Registration" or "WebAuthn registration" here means that a user creates their public key credential by their WebAuthn authenticator and register it onto keycloak.

### How to Realize

It is realized by a required action provider called "WebAuthn Register". It conforms to the following acceptance criteria in [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) : Two factor authentication with W3C Web Authentication:

>    * Admin registers required action for user to register a security key

Also, to make the browser execute WebAuthn API (`navigator.credentials.create()`), a JavaScript and a FreeMarker template are required.

### Actions for Registration

A user without having their user account in keycloak:

* They can register their public key credential along with creating their user account in keycloak.

An existing user having their user account in keycloak:

* They can register their own public key credential at their login time by Required Action.

* After realizing [**AIA**](https://github.com/keycloak/keycloak-community/blob/master/design/application-initiated-actions.md), the user can register their own public key credential it by AIA that User Account Service also use it.

### Registered Public Key Credential's Metadata

As mentioned [before](#metadata), when registering the public key credential, the user can input texts on the single string field to identify this credential afterwards on user authentication.

### Configuration

Registration can be configured by "WebAuthn Policy" policy the same as other credentials like password and OTP.

Basically, this design document follows [WebAuthn API Specification](https://www.w3.org/TR/webauthn/#dictionary-makecredentialoptions) for configuration items and their values.

#### Configuration Scope

The configuration is realized by the policy so that it is adopted per realm.

Note:
* This scope seems to be too much coarse. Therefore more [fine-grained scope](#configuration-related-issue) should be considered in the future implementation phase

#### Configuration Item - Relying Party Entity Name

  - UI : TextBox
  - Default setting : "keycloak"

*Notes:*

* This [setting](https://www.w3.org/TR/webauthn/#dom-publickeycredentialentity-name) is **required** for WebAuthn API (`navigator.credentials.create()`).

#### Configuration Item - Signature Algorithm

  - Item selection : Multiple items can be selected.
  - UI : ComboBox that enables multiple item selection and have a blank item.
    - If a user selects only the blank item, it means no algorithm is selected.
    - If a user selects no items, it also means no algorithm is selected.
    - If a user selects multiple items including the blank item, this blank item selection is ignored. 
  - Supported algorithms : {ES256, ES384, ES512, RS1, RS256, RS384, RS512}
  - Default setting : a blank item selected
  - If no algorithm is selected, {ES256} is adapted.

*Notes:*

* These supported algorithms are ones the WebAuthn protocol processing core library [webauthn4j](https://github.com/webauthn4j/webauthn4j) supported.
* This [setting](https://www.w3.org/TR/webauthn/#dictdef-publickeycredentialparameters) is **required** for WebAuthn API (`navigator.credentials.create()`)
* According to the rules just above, if a user selects only a blank item or no items, it is equal to selecting only {ES256}.
* Keycloak does not support PS256, PS384, PS512 because the core library webauthn4j does not support them. The reason why is told in [WebAuthn4J Reference](https://webauthn4j.github.io/webauthn4j/en/#webauthn4js-ps256ps384ps512-support).

#### Configuration Item - Relying Party ID

  - UI : TextBox
  - Default setting : blank

*Notes:*

* This [setting](https://www.w3.org/TR/webauthn/#dom-publickeycredentialrpentity-id) is optional for WebAuthn API (`navigator.credentials.create()`). If this configuration is left blank, the host part of the base URL of keycloak's server is adapted.

#### Configuration Item - WebAuthn Authenticator Attachment

  - Item selection : Only single item can be selected.
  - UI : ComboBox that enables single item selection and have a blank item.
    - If a user selects only this blank item, that means this configuration is not used.
    - If a user selects no items, it also means this configuration is not used.
  - Supported WebAuthn authenticator attachment options : {platform, cross-platform}
  - Default setting : a blank item selected that means this configuration is not used.

*Notes:*

* This [setting](https://www.w3.org/TR/webauthn/#dom-authenticatorselectioncriteria-authenticatorattachment) is optional for WebAuthn API (`navigator.credentials.create()`). Therefore, this configuration is not used, WebAuthn authenticators are not filtered out by their attachment type.

#### Configuration Item - Require Resident Key

  - Item selection : Only single item can be selected.
  - UI : ComboBox that enables single item selection and have a blank item.
    - If a user selects only this blank item, that means this configuration is not used.
    - If a user selects no items, it also means this configuration is not used.
  - Supported resident key options : {Yes, No}
  - Default setting : a blank item selected that means this configuration is not used.

*Notes:*

* This [setting](https://www.w3.org/TR/webauthn/#dom-authenticatorselectioncriteria-requireresidentkey) is optional for WebAuthn API (`navigator.credentials.create()`). Therefore, this configuration is not used, WebAuthn authenticators are not filtered out by their resident key capability.

#### Configuration Item - User Verification Requirement

  - Item selection : Only single item can be selected.
  - UI : ComboBox that enables single item selection and have a blank item.
    - If a user selects only this blank item, that means this configuration is not used.
    - If a user selects no items, it also means this configuration is not used.
  - Supported user verification options : {required, preferred, discouraged}
  - Default setting : a blank item selected that means this configuration is not used.

*Notes:*

* This [setting](https://www.w3.org/TR/webauthn/#dom-authenticatorselectioncriteria-userverification) is optional for WebAuthn API (`navigator.credentials.create()`). Therefore, this configuration is not used, WebAuthn authenticators are not filtered out by their user verification capability.
* This configuration is also applied for WebAuthn API (`navigator.credentials.get()`).

#### Configuration Item - Attestation Conveyance Preference

  - Item selection : Only single item can be selected.
  - UI : ComboBox that enables single item selection and have a blank item.
    - If a user selects only this blank item, that means this configuration is not used.
    - If a user selects no items, it also means this configuration is not used.
  - Supported attestation conveyance preference : {none, indirect, direct}
 - Default setting : a blank item selected that means this configuration is not used.

*Notes:*

* This [setting](https://www.w3.org/TR/webauthn/#dom-publickeycredentialcreationoptions-attestation) is optional for WebAuthn API (`navigator.credentials.create()`). this configuration item is left blank, it is the same that "none" is specified.

* At the first implementation phase, tthis Attestation Conveyance Preference value is fixed as "none". If so, the WebAuthn API (`navigator.credentials.create()`) returns AAGUID as all 0 filled. Therefore, in this situation, keycloak can not get the metadata from FIDO Alliance Metadata Service.

#### Configuration Item - Timeout

  - UI : Integer from 0 to 31536 (second)
  - Default setting : 0

*Notes:*

* This [setting](https://www.w3.org/TR/webauthn/#dom-publickeycredentialcreationoptions-timeout) is optional for WebAuthn API (`navigator.credentials.create()`). this configuration is set to 0, this configuration itself is not used.
* This configuration is also applied for WebAuthn API (`navigator.credentials.get()`).

#### Configuration Item - Avoid Same WebAuthn Authenticator Registration

  - UI : Switch (ON/OFF)
  - Default setting : OFF

*Notes:*

* If set to "ON", the WebAuthn authenticator that has already been registered can not be newly registered. This is applied to the operation of registering WebAuthn authenticator. The default setting is "OFF".
* This configuration item uses [exclude credentials](https://www.w3.org/TR/webauthn/#dom-publickeycredentialcreationoptions-excludecredentials) to realize this feature.

### Filtering out Users' WebAuthn Authenticators

This section relates to the criteria mentioned in [KEYCLOAK-9358](https://issues.jboss.org/browse/KEYCLOAK-9358) : Investigate W3C Web Authentication libraries.

>    * Attestation to be able to get details about authenticators and limit what authenticators should be supported by a realm

There is such the use case that the administrator enforces users to use only the biometric authenticator (e.g. fingerprint), and they want keycloak to reject the WebAuthn authenticator without biometric authentication capability.

In order to do that, keycloak needs to know the WebAuthn authenticator's capability. It seems to be impossible to do that unless keycloak get the detailed metadata. However, information returned from WebAuthn API (`navigator.credentials.create()`) does not contain such the detailed information.

#### by AAGUID

AAGUID list is set up as a configuration item. Filtering users' WebAuthn authenticators works as follows:

* Get information returned from WebAuthn API (`navigator.credentials.create()`)
* Check whether AAGUID contained in this information matches one in AAGUID list
  * If matches, register a public key credential based on this information.
  * If not, returns an error telling a user .

This AAGUID list works as white list.


#### by Metadata acquired from external sources

At the first step of implementation, it is out of scope.

### Open Issues

User Interface, Logging, Error Handling need to be considered.

## Authentication
***

### Definition

"Authentication" here means that a user uses their public key credential as credential of their user authentication.

### How to Realize

It is realized by an authenticator provider called "WebAuthn Authenticator". It conforms to the following acceptance criteria in [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) : Two factor authentication with W3C Web Authentication:

>    * Admin manually registers Web Authentication authenticator to replace OTP authenticator

It is an authenticator provider. Therefore, by adding this authenticator provider after existing username authenticator provider, it conforms to the following acceptance criteria in [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) : Two factor authentication with W3C Web Authentication:

>    * When user logs in again after registering the security key the user should be prompted to tap the security key after entering username and password

Also, to make the browser execute WebAuthn API, a JavaScript and a FreeMarker template are required.

### Configuration

WebAuthn authentication's configuration uses WebAuthn registration's configuration realized as  "WebAuthn Policy".

Basically, this design document follows [WebAuthn API Specification](https://www.w3.org/TR/webauthn/#assertion-options) for configuration items and their values.

The configuration items in WebAuthn authentication is as follows :

* [User Verification Requirement](#configuration-item---user-verification-requirement)
* [Timeout](#configuration-item---timeout)

### Specifying Public Key Credentials

A user can select which their public key credentials are used for their WebAuthn authentication.

On WebAuthn user authentication:

  * Keycloak shows the list of public key credentials with its metadata that the user has registered.
  * The user can select some of their public key credentials.
  * It is acceptable that the user selects no public key credentials.
  
  - Item selection : Multiple items can be selected.
  - UI : ComboBox that enables multiple item selection and have a blank item.
    - If a user selects only the blank item, it means no public key credentials is selected.
    - If a user selects no items, it also means no public key credentials is selected.
    - If a user selects multiple items including the blank item, this blank item selection is ignored. 
  - Default setting : a blank item selected that means no public key credentials are specified.

*Notes:*

* This [setting](https://www.w3.org/TR/webauthn/#dom-publickeycredentialrequestoptions-allowcredentials) is optional for WebAuthn API (`navigator.credentials.get()`). However, it must be needed if the user has registered their public key credential by the WebAuthn authenticator without [Resident Key](https://www.w3.org/TR/webauthn/#client-side-resident-public-key-credential-source) capability. Such the WebAuthn authenticator needs the information sent by this option. 

### Open Issues

User Interface, Logging, Error Handling need to be considered.

## Credential Management
***

### Definition

"Credential Management" here means that a user and an administrator can do the following things :

* Register the new [public key credential](#public-key-credential-itself) onto keycloak by the user.

* Delete the existing public key credential stored in keycloak by the user and the administrator.

* View information on the existing public key credential stored in keycloak by the user and the administrator.
    - Attested Credential Data
      - AAGUID
      - Credential ID
      - Credential Public Key
    - [Metadata](#metadata)
      - Label

* Edit information on the existing public key credential stored in keycloak by the user and the administrator
    - [Metadata](#metadata)
      - Label

### Management by a user

After realizing AIA, the user can also manage their own public key by [**AIA**](https://github.com/keycloak/keycloak-community/blob/master/design/application-initiated-actions.md) that User Account Service also use it.

### Management by an administrator

The administrator can manage all user's public key credentials on Admin Console 
(Users -> [user] -> Credentials tab) which implies that it can be done by Admin REST API.

### Open Issues

User Interface, Logging, Error Handling need to be considered.

## Attestation Statement Verification
***

As mentioned in the [implementation plan](#implementation-plan), an attestation statement validation is supported in the second implementation phase.

According to the [WebAuthn API Specification](https://www.w3.org/TR/webauthn/#registering-a-new-credential), the attestation statement verification consists of the following two part :

* Attestation statement validation : Step 1 to 14 and 17 to 19 of [Registering a New Credential in WebAuthn API Specification](https://www.w3.org/TR/webauthn/#registering-a-new-credential) 

* Attestation statement trustworthiness validation : Step 15 to 16 of [Registering a New Credential in WebAuthn API Specification](https://www.w3.org/TR/webauthn/#registering-a-new-credential) 

### Attestation Statement Validation 

#### Attestation Statement Format

Keycloak support the following [Attestation Statement Format](https://www.w3.org/TR/webauthn/#attestation-statement-format) :

* [Packed Attestation Statement Format](https://www.w3.org/TR/webauthn/#packed-attestation)
* [TPM Attestation Statement Format](https://www.w3.org/TR/webauthn/#tpm-attestation)
* [Android Key Attestation Statement Format](https://www.w3.org/TR/webauthn/#android-key-attestation)
* [Android SafetyNet Attestation Statement Format](https://www.w3.org/TR/webauthn/#android-safetynet-attestation)
* [FIDO U2F Attestation Statement Format](https://www.w3.org/TR/webauthn/#fido-u2f-attestation)
* [None Attestation Statement Format](https://www.w3.org/TR/webauthn/#none-attestation)

#### Attestation Type

Keycloak support the following [Attestation Type](https://www.w3.org/TR/webauthn/#attestation-type) :

* [Basic Attestation (Basic)](https://www.w3.org/TR/webauthn/#basic-attestation)
* [Self Attestation (Self)](https://www.w3.org/TR/webauthn/#self-attestation)
* [Attestation CA (AttCA)](https://www.w3.org/TR/webauthn/#attestation-ca)
* [No attestation statement (None)](https://www.w3.org/TR/webauthn/#none)

Keycloak does not support [Elliptic Curve based Direct Anonymous Attestation (ECDAA)](https://www.w3.org/TR/webauthn/#elliptic-curve-based-direct-anonymous-attestation) because the core library webauthn4j does not support it. The reason why is told in [WebAuthn4J Reference](https://www.w3.org/TR/webauthn/#elliptic-curve-based-direct-anonymous-attestation).

### Attestation Statement Trustworthiness Validation

Depending on an attestation type, there are three types of Attestation Statement Trustworthiness Validation (Step 16 of  [Registering a New Credential in WebAuthn API Specification](https://www.w3.org/TR/webauthn/#registering-a-new-credential)).

* Self Attestation
* ECDAA
* Certification Path

Keycloak supports Self Attestation and Certification Path. Keycloak does not support ECDAA because the core library webauthn4j does not support it. The reason why is told in [WebAuthn4J Reference](https://www.w3.org/TR/webauthn/#elliptic-curve-based-direct-anonymous-attestation).

#### Trust Anchor

In order to do Attestation Statement Trustworthiness Validation with Certification Path, keycloak needs to get its trust anchor's certificate. It is plausible to get the trust anchor's certificate in the following ways :

* KeyStore file used as Trust Store having CA certificates indicating the holder of the private key used for signing into an attestation statement.

* FIDO Alliance MDS

At [the second phase of implementation](#second-implementation-phase), keycloak only supports KeyStore file used as Trust Store.

*Notes :*
* Keycloak uses existing `Truststore Provider` so that the setting of the KeyStore file used as the trust store follows the [existing keycloak's document about the trust store](https://www.keycloak.org/docs/latest/server_installation/index.html#_truststore). 
* Keycloak does not follow the update of KeyStore file (e.g. adding or deleting CA certificates) while keycloak is running.
* Keycloak does not check the revocation of CA certificates in the trust store(e.g. by CRL or OCSP).

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

[Webauthn4j](https://github.com/webauthn4j/webauthn4j/blob/master/LICENSE.txt) is Apache License 2.0.

>    * A reasonable sized community

[Webauthn4j's community](https://github.com/webauthn4j/) is still a relatively small community, but we library user side and webauthn4j's originator and maintainer [ynojima](https://github.com/ynojima) can communicate smoothly and continue making contributions each other.

>    * Good test coverage

[Webauthn4j](https://sonarcloud.io/component_measures?id=webauthn4j&metric=coverage&view=list) shows relatively high test coverage. In addition to that, webauthn4j has passed the conformance test of all mandatory test cases and optional Android Key attestation test cases of [FIDO2 Test Tools provided by FIDO Alliance](https://fidoalliance.org/certification/functional-certification/conformance/).

>    * Maven build (Graddle is ok, but not ideal)

Webauthn4j is Gradle build.

## Performance Consideration

### Avoid checking and updating WebAuthn authenticator credential's counter

When WebAuthn authentication, keycloak [checks and updates the counter](https://www.w3.org/TR/webauthn/#signature-counter) of the registered public key credential by following the step 17 of [Verifying an Authentication Assertion in the webauthn specification](https://www.w3.org/TR/webauthn/#verifying-assertion). The public key credential is stored on the persistent storage so that this process invokes the disk write access, which might diminish the performance if too many authentication requests happen in the short time.

To manage this matter, such the option should be considered that this counter's check and update is not executed intentionally.

This option is included in WebAuthn Policy. 

**Note**:
* If this option is enabled once and disabled again, the step 17 of [Verifying an Authentication Assertion in the webauthn specification](https://www.w3.org/TR/webauthn/#verifying-assertion) is never passed successfully because the counter value in the public key credential stored in keycloak has been out of sync with the one in the WebAuthn authenticator.

### Cache WebAuthn authenticator's credential

When WebAuthn authentication, keycloak looks up the registered public key credential to execute [Verifying an Authentication Assertion in the webauthn specification](https://www.w3.org/TR/webauthn/#verifying-assertion). The public key credential is stored on the persistent storage so that this process invokes the disk read access, which might diminish the performance if too many authentication requests happen in the short time.

To manage this matter, the public key credential should be cached the same as other credentials like OTP and password (implementing `OnUserCache` interface).

**Note**:
* This cache might be effective when the option of avoiding checking and updating WebAuthn authenticator credential's counter is enabled because if this option is disabled, keycloak needs to do write access to the persistent storage on every authentication. Therefore, this cache should be enabled automatically only if this option is enabled.

## Testing
***

For automated functional tests for the integration, [Web Authentication Testing API](https://docs.google.com/document/d/1bp2cMgjm2HSpvL9-WsJoIQMsBi1oKGQY6CvWD-9WmIQ/edit#) can be used.

It has been confirmed as preliminary analysis that this Web Authentication Testing API can be integrated into keycloak's Arquillian integration testing framework and basic functional tests of registration and authentication flow has been realized.

## Acknowledgements
***

webauthn4j's originator and maintainer [ynojima](https://github.com/ynojima) gave fruitful comments and advice on this design document.

