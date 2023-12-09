# OpenID Verifiable for Credential Issuance

## Observerability

* **Status**: Notes
* **JIRA**: None
* **Discussion**: [OpenID for Verifiable Credential Issuance #17616](https://github.com/keycloak/keycloak/discussions/17616?sort=new)


## Motivation
OpenID Verifiable Credential Issuance ([OID4VC](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html)) has been discussed a lot in the Self-Soverin Identity (SSI) especially due to European Commission having released a [Framework](https://digital-strategy.ec.europa.eu/en/library/european-digital-identity-wallet-architecture-and-reference-framework) for eIDASv2. The later tends to chose OID4VC as a protocol to issue Verifiable Credential (VC) and to present Verifiable Presentation (VP).

For simplicity, Verifiable Credential (VC) is a token generally either JSON-LD / JWT format issued by OpenID Provider to the User whereas a Verifiable Presentation (VP) is a token presented by User to Relying Party for authentication and authorizations purposes. A VP can be the result of an aggregation of multiples VCs or simply a subset of attributes of a single VC. [OID4VP](https://openid.net/specs/openid-connect-4-verifiable-presentations-1_0-07.html) and [SIOPv2](https://openid.net/specs/openid-connect-self-issued-v2-1_0.html) have been introduced for the purposes of presenting Verifiable Presentation from USER to Relying Party.

Currently, they are many competing standards in the world of VC data formats:
* [Verifiable Credentials based on SD-JWT](https://www.ietf.org/archive/id/draft-terbu-oauth-sd-jwt-vc-00.html#name-verifiable-credentials-base)
* [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/)
* [ISO MDOC - MDLs](https://www.iso.org/standard/69084.html)
* [JWT-VC](https://github.com/decentralized-identity/did-jwt-vc)

We will be focussing on implementing the first format [SD-JWT VC](https://www.ietf.org/archive/id/draft-terbu-oauth-sd-jwt-vc-00.html#name-verifiable-credentials-base) in this document.

This format build on top of the IETF Draft: [Selective Disclosure for JWTs](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-selective-disclosure-jwt-06)

## Intention of this Document
This document outlines the design specifications for integrating OID4VC (including OID4VCI and OID4VP) into Keycloak. The implementation aims to ensure interoperability with the GAIN POC and the HAIP profile, while also incorporating additional features relevant to organizational use within Keycloak. This will be pursued as long as contributions are thoroughly reviewed and accepted by the Keycloak maintenance team.

## Interoperability Matrix
The following table displays some competing initiatives, all implementing OID4VC. This table is managed by OIDF GAIN PoC working group at [OID4VC Profiles](https://docs.google.com/spreadsheets/d/1s6REK5eNAb3GSElID0J02_TtbuI2Exd9z-CLdLx0emk/edit?usp=sharing). There is a attempt to define a baseline profile, that will be supported by all initiatives.

Keycloak being the largest SSO open source software, we want to use this initiative to secure implementation of OID4VC into Keycloak.

## PoCs

* there is an [existing POC from FIWARE](https://github.com/FIWARE/keycloak-vc-issuer)

We are calling for any other contribution out there.

# Visual 
The following map from [Chris](https://github.com/coxchrisw) provides an [OID4VC Breakdown](/design/img/oid4vc_mapping_cox.png) of all the functionalities for each participant type in the verifiable credential world.

# Feature Set

## Credential Issuance - Issuer

This part addresses the functionality associated with the issuer of a verifiable credential. Keycloak will play both the role of an authorization server and a credential issuer. All yellow cards are relevant for the first steps as outlined in [Visuals](#visual). Cads with a red square are future or optional features.

In this first step, we will extend Keycloak to act as a VC Issuer. We will implement the following flows:

* The [pre-authorize code flow](./oid4vci/preauthorize-code-flow.md)
* The [authorization code flow](./oid4vci/authorization-code-flow.md)

## Verification of Presentation - Verifier
In the second step, we will equip keycloak with verifier functionality to fit into many federation use cases currently in use with Keycloak. To approach this second step, we will need a detailed design suggestion that outlines where to start, which includes new endpoints and the existing affected endpoints.

## Verifiable Presentation - Web Wallet
At the moment, it is not within the scope of Keycloak to operate as a web wallet, even though this seems like a natural extension of an identity provider. If use cases justify the integration of a web wallet functionality into Keycloak, the corresponding rationales and design suggestions will be added to this architectural document.

## Verifiable Presentation - Edge Wallet
The issuance of verifiable presentation will generally be performed by edge wallets. We will be working on wallet prototypes to test and demo use cases implemented into keycloak. 

# Technology Outlook
For the implmentation of an interoperable OID4VC solution, we will be dealing with following technologies

## Selective Disclosure: SD-JWT
All emerging SSI standards are including selective disclosure as a core feature to address privacy. Selective disclosure is an essential requirement in the eIDAS2.0 directive of the EU. 

For the purpose of the work, [SD-JWT VC](https://www.ietf.org/archive/id/draft-terbu-oauth-sd-jwt-vc-00.html#name-verifiable-credentials-base) defines a mechanism for selective disclosure of individual elements of a JSON object used as the payload of a JSON Web Signature (JWS) structure.

**Index Sites**
* A collection of implementations can be found in the [IETF OAuth Working Group - SD-JWT](https://github.com/oauth-wg/oauth-selective-disclosure-jwt).
* A couple of repositories at the [open wallet foundation](https://github.com/orgs/openwallet-foundation-labs/repositories?q=sd-jwt) are also addressing the topic. We have no java implementation there yet.

**Other Sites**
* This [waltid kotlin implementation](https://github.com/walt-id/waltid-sd-jwt) was archived and is being redesigned
* This is a [Java Implementation from Authlete](https://github.com/authlete/sd-jwt).

**ToDo: SD-JWT lib**
For the purpose of this work, we will have to decide on how to setup and implement (or select and use) an SD-JWT library that corelates with the governance model of keycloak components.

## Credential Format: SD-JWT-VC
[SD-JWT-based Verifiable Credentials](https://www.ietf.org/archive/id/draft-terbu-oauth-sd-jwt-vc-00.html) describes data formats as well as validation and processing rules to express Verifiable Credentials with JSON payloads based on the Selective Disclosure for JWTs (SD-JWT)

**Index Sites**
As for existing implementations, a pure context free implementation of SD-JWT VC is not to be found in public repositories.

**ToDo: SD-JWT-VC lib**
For the purpose of this work, we will have to decide on how to setup and implement an SD-JWT-VC library that corelates with the governance model of keycloak components.

## Credential Format: VCDM
There is a lot of implementation out there with the [VCDM]((https://www.w3.org/TR/vc-data-model/)) credential format. But many technical implementation have run into complexity trying to implement selective disclosure on top of VCDM.

**ToDo: VCDM Support**
In this work, we will have to discuss on wether VCDM is to be supported, and if yes, if the VCDM support will include any type of selective disclosure (annoncred, ...). We will also be discussing on the need to include this part and the existance of stake holders. For the time beeing, eIDAS2.0 is leading toward SD-JWT and ISO mDLs.

## Credential Format: mDLs - ISO/IEC 18013-5:2021
[IDO mDL](https://www.iso.org/standard/69084.html) defines the technical specifications for implementing a driving licence on a mobile device. It's essentially a blueprint for creating "mobile driving licences" (mDLs) that function securely and consistently across different platforms.

This technology is being prototyped in many US states and is also included into the list of format supported by the eIDAS2.0 directive.

mDLs also provides selective disclosure.

**Index Sites**

* The [Open Wallet Foundation - Identity Credential](https://github.com/openwallet-foundation-labs/identity-credential) is a repository of Java libraries addressing different aspects of ISO mDLs.

**Other Sites**

* This [Waltid kotlin implementation](https://github.com/walt-id/waltid-mdoc) was archived and is being redesigned
* This is a [Java Implementation from Authlete](https://github.com/authlete/cbor).

**ToDo: SD-JWT lib**
For the purpose of this work, we will have to decide on how to setup and implement (or select and use) an mdoc library that corelates with the governance model of keycloak components.

# Cryptographic Work

## Signature Types

**Purpose**

Digital signature is used in following area of the OID4VC:
* For Credential Issuance: An verifiable credential is allways signed by it issuer.
* For Credential Presentation, when key binding is included in the presented VC. 
 
**SD-JWT-VC and/or mDLs**

* Signature Type: ES256 (both for JOSE and COSE)

**ToDo: Keycloak Integration**

For the purpose of this work, we will have be:
* Analyzing how to reuse the ES256 crypto implementation libraries allready available in keycloak for the production and verification of signatures.
* Design **Credential Format** and **Selective Disclosure** libs, such as to have them share tose libs already in use by keycloak.

## Expression of key Binding
In case the issuer wants to binds the verifiable credential to a key controlled by the holder, this key (or a reference thereof) must be included into the produced credential. For the purpose of interoperability, implementations will have to decide on where and how to proceed.

**SD-JWT-VC and/or mDLs**

* For the GAIN/HAIP POCs, we will be expecting the key to be a jwk in the cnf claim of the JWT, or
* In the common base profile in negotiation, it can be a kid (with did:jwk) in the cnf claim of the JWT.

## Issuer Key Resolution / Validation
A party validating a VC or VP has to be able to retrieve and validate the key used by the issuer to produce the presentation. For this reason, we need to define how to embed or reference the issuer key in the VC.

* For the GAIN/HAIP POCs, we are looking at:
    * jwt-issuer specification from the OIDF.
    * X509 certificate chains (e.g x5t)
* For  the baseline profile in negotiation, did:web and jwt-issuer will both be supported.

In matter of trust anchors, we might endup having some static list of x509 root certificates (incl. eIDAS Certificates) or even using OpenId Federation like described in these [slides from our coleague Giuseppe](https://docs.google.com/presentation/d/13FvGbd-pxoKw4VMCrdjOE8MlGLAs6m04eRPHc0N_kRM/edit#slide=id.p). But nothing is decide yet and most POC will be run with peer Issuer Key exchanges.

