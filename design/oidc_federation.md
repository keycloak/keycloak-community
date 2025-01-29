# OpenID Connect Federation

- **Status**: Draft #1
- **JIRA**: [KEYCLOAK-16574](https://issues.redhat.com/browse/KEYCLOAK-16574)

## Motivation

The [OpenID Connect Federation](https://openid.net/specs/openid-connect-federation-1_0.html) standard will enable flexible and scalable trust establishment among entities and will represent a key enabling technology to either build identity federations on top of OpenID Connect or securely integrate with existing ones. 
Our goal is to support the OpenID connect federation standard and demonstrate interoperability with other components/libraries currently in development in the context of the OpenID foundation.
In this context, we will extend Keycloak to be able to be configured as an Openid Provider and/or as a Relying Party, which can participate in OpenID Connect Federation(s). 

## Common Implementation Parts 

This section describes the extensions required for configuring Keycloak as either an OpenID Provider or a Relying Party that can participate in an OpenID Connect Federation.

### Representation entities

- [Entity Statement](https://openid.net/specs/openid-connect-federation-1_0.html#entity-statement)
  - [Entity Metadata](https://openid.net/specs/openid-connect-federation-1_0.html#metadata), included in Entity Statement
     - [RP Metadata](https://openid.net/specs/openid-connect-federation-1_0.html#RP_metadata) extends OIDCClientRepresentation, included in Entity Metadata 
     - [OP  Metadata](https://openid.net/specs/openid-connect-federation-1_0.html#OP_metadata) extends `OIDCConfigurationRepresentation`, included in Entity Metadata 
     - [Federation Entity] (https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.3.6) included in Entity Metadata 
  - Metadata Policy, included in Entity Statement
     - OIDCFedClientRepresentationPolicy, metadata_policy for RP Metadata included in Metadata Policy
     - OIDCFedConfigurationRepresentationPolicy, metadata_policy for OP Metadata included in Metadata Policy
         - [Policy](https://openid.net/specs/openid-connect-federation-1_0.html#PolicyLanguage) for single fields included in OIDCFederationClientRepresentationPolicy/ OIDCFedConfigurationRepresentationPolicy
         - [PolicyList](https://openid.net/specs/openid-connect-federation-1_0.html#PolicyLanguage) for list fields included in OIDCFederationClientRepresentationPolicy/ OIDCFedConfigurationRepresentationPolicy


### Trust chain 

- An entity to hold the required information to describe a trust chain
- An algorithm to compute all possible trust chains from a leaf node to a trust anchor, as described [here](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.2.2)
- An algorithm to validate a given trust chain (integrity, signing validity, parent-child consistency, etc)
- In a trust chain, Metadata Policies should be combined as described [here](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.4.1.3)

## OpenID Provider (OP)

This section is related to the extensions required to support configuring Keycloak as an Openid Provider that can participate in an OpenID Connect Federation.

We want to support both [automatic registration](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.9.1) and [explicit registration](https://openid.net/specs/openid-connect-federation-1_0.html#explicit).

### Realm Configuration

The realm admin can add needed configuration for this realm being an Openid Provider that participates in an OpenID Connect Federation. Fields configured are :
- `Authority hints`, which are entity identifiers of intermediate entities or trust anchor(s) that Keycloak OP belongs to. See about authority_hints in [specification](https://openid.net/specs/openid-connect-federation-1_0.html#entity-statement)
- `trust anchor ids`
- `registration type` supported in this realm. Values are automatic, explicit or both. Default value is ‘both’. 
- entity statement `expiration time`

All these fields are mandatory for participating in an OpenID Connect Federation.
The configuration above will be a separate tab in the ‘Client Registration’ tab of the Realm settings.
If no authority hint has been configured in the realm, the endpoints will return an error response.

### Database changes

Introduce `OIDCFedConfig` model class, containing all needed configuration for the Federated OP (authority hints, trust anchors, registration type, expiration time) is stored into a single table with the schema (realmID, configuration), where the realmID is a foreign key to the existing Realm (Entity) ID and the configuration is a json serialized string of a `OIDCFedConfig` class instance.

### Well-known OIDC federation endpoint for OP 

As described in [Obtaining Federation Entity Configuration Information](https://openid.net/specs/openid-connect-federation-1_0.html#federation_configuration), Keycloak needs to expose `.well-known/openid-federation endpoint` - part of [Well-Known URIs](https://tools.ietf.org/html/rfc8615) specification. For implementing this, we need to create `OIDCFedOPWellKnownProvider extends OIDCWellKnownProvider` and `OIDCFedOPWellKnownProviderFactory implements WellKnownProviderFactory`. The `getConfig` method of `OIDCFedOPWellKnownProvider` will return a signed Keycloak Entity Statement, in the form of a [JSON Web Signature (JWS)](https://tools.ietf.org/html/rfc7515). Response content type MUST be set to `application/jose`. We need to remove @Produces(MediaType.APPLICATION_JSON) .
[OP Metadata](https://openid.net/specs/openid-connect-federation-1_0.html#OP_metadata) entity will extend `OIDCConfigurationRepresentation` with the following values:
- `client_registration_types_supported` =`registration type` of Realm Configuration
- `federation_registration_endpoint` = url of [Federation Registration Endpoint](#federation-registration-endpoint). Value will be shown if Explicit Registration is supported for this realm.
- `client_registration_authn_methods_supported` = {"ar": ["request_object"]}. This allows the use of authorization request, which has already been supported by Keycloak, in Automatic Registration. 

### Automatic Registration

[Specification](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.9.1)
 
According to the specification, the authentication request can be performed by passing a request object by value as described in section 6.1 in [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) or using pushed authorization as described in [Pushed Authorization Requests](https://tools.ietf.org/id/draft-ietf-oauth-par-04.html). 
We propose to use authorization request in Automatic Registration which is already supported by Keycloak. 
In order to support Automatic Registration, we need a separate method in AuthorizationEndpoint. 
This requires extending the `checkClient` method of `AuthorizationEndpoint` as follows: if Client Id does not exist in the client database and this realm supports automatic registration, Keycloak will perform an automatic registration process as described [here](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.9.1.1.1).
Note that in Automatic Registration, the RP will not be saved in Keycloak’s client database.

### Explicit Registration

[Specification](https://openid.net/specs/openid-connect-federation-1_0.html#explicit)

#### Federation Registration Endpoint 

[Specification](https://openid.net/specs/openid-connect-federation-1_0.html#Clireg).
POST web service with body being a signed RP Entity Statement ( application/jose).
[Actions](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.9.2.1.2.1) :
1. Check for valid and not expired signed entity statement
2. aud field must contain Keycloak OP entity Identifier (http(s)://host:port/{basepath}/realms/{realm-name} ) 
3. Find at least one valid [trust chain](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.7)
4. Find one trust chain with accepted Metadata Policies. Enforce Metadata Policies to RP Metadata.
5. Check that no existing Client Id matches the entity identifier in this realm.
6. Save RP Metadata as Keycloak does in [Dynamic Client Registration](https://www.keycloak.org/docs/latest/securing_apps/index.html#openid-connect-dynamic-client-registration). The RP’s Entity Identifier is saved as the Client Id. 
7. RP Metadata is changed based on the Client saved in the database and RP Metadata Policies used are added to the Entity Statement. 
Moreover, the trust anchor of the accepted trust chain and the authority hint of Keycloak Federated OP for this trust chain are added. The updated signed Entity Statement is returned.

If the realm does not support explicit registration, an error Response will be returned.

#### [Expired Client](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.9.2.2.2)

Based on the specification, a Client imported with Explicit Registration MUST expire when the signed entity statement expires. In order to support this, OIDC Federation Clients need to have an expiration time field set upon registration (its value MUST be no later than the trust chain's expiration time). A Runnable Task will be executed in specified time intervals to remove any Clients which have expired.

## Relying Party (RP)
This section is related to the extensions required to support configuring Keycloak as a Relying Party that can participate in an OpenID Connect Federation.

### OIDC Federation enabled Identity Provider

We need to extend the current OpenID Connect 1.0 Identity provider model (`OIDCIdentityProviderConfig`) with some additional information as required by the [specification](https://openid.net/specs/openid-connect-federation-1_0.html#RP_metadata): 
- `client_registration_types` (required): the federation registration type with one of {explicit, automatic} value. 
- `organization_name` (optional): A human readable name representing the organization owning the RP (Keycloak).
- `authority_hints` (required): the entity identifier(s) of intermediate entities or trust anchor(s) that Keycloak RP belongs to. This information is intended to be included in the self-signed entity statement of the RP. 
- `expired`: Entity statement expiration time. The client registration will expire at this time. In the case of the explicit registration, Keycloak will need to periodically renew the registration (see [this](#explicit-registration-1) for details).
- `trust_anchor_ids` (required): List containing the entity identifiers of the trust anchors. 
- `op_entity_identifier` : OP entity identifier. Required only for explicit registration.


The Client Id will be automatically set to the entity identifier ( `http(s)://host:port/{basepath}/realms/{realm-name}/{rp_alias}`). The Client Secret will not be set. If `client_registration_types` is set to explicit, Identity Provider will be disabled. See [this](#explicit-registration-1) for details

Moreover, `OIDCFedIdentityProviderFactory` needs to extend `AbstractIdentityProviderFactory` and `FederationOIDCIdentityProvider` needs to extend `AbstractIdentityProvider`. The authentication process will be different depending on client_registration_types - see [section](https://openid.net/specs/openid-connect-federation-1_0.html#client_registration). That’s why some methods of OIDCFedIdentityProvider will be different based on client_registration_types. Automatic registration requires a different implementation of the `createAuthorizationUrl` method. Therefore, `OIDCFedIdentityProvider` can not extend AbstractOAuth2IdentityProvider.

### Database changes

NO database changes are needed. Extra fields will be saved in IDENTITY_PROVIDER_CONFIG.

### [Automatic Registration](https://openid.net/specs/openid-connect-federation-1_0.html#rfc.section.9.1)

Automatic registration will allow Keycloak to act as an RP that can send authorization requests to an OP without first registering with the OP. We propose to perform the request by passing a request object by value as described in section 6.1 in OpenID Connect Core 1.0, which is already supported by Keycloak.
To support this, the request parameter value is a JWT whose Claims are the request parameters specified in Section 3.1.2 in OpenID Connect Core 1.0. The JWT MUST be signed and MAY be encrypted. 


#### Well-known OIDC federation endpoint for RP {#rp-well-know}

The OIDC federation specification introduces the [.well-known/openid-federation](https://openid.net/specs/openid-connect-federation-1_0.html#federation_configuration) endpoint also for the RPs that support automatic registration - which provides a JWT self-signed entity statement for the RP. 
Thus, keycloak should have an additional endpoint available only for OIDC Federation IdP that supports automatic registration under each tenant (realm), with the alias name of the idp prepended. 
The relative path could follow the format: http(s)://host:port/{basepath}/realms/{realm-name}/{idp_alias}/.well-known/openid-federation - http(s)://host:port/{basepath}/realms/{realm-name}/{idp_alias} will be entity identifier, which eventually uniquely identifies RP within the whole federation. 
This .well-known endpoint is mandatory for a successful automatic registration process. Response content type MUST be set to application/jose.RP Metadata need to be constructed from the defined Federation OIDC Identity Provider.


### [Explicit Registration](https://openid.net/specs/openid-connect-federation-1_0.html#explicit){#rp-explicit}
The Keycloak RP needs to be registered with the Federated OP in order to be enabled. We propose to show a register RP button after saving the explicit Federation OIDC Identity Provider.
If the realm admin presses this button, keycloak will send a POST request with the body containing its signed entity Statement to Federation OP federation_registration_endpoint. Entity Statement value is same as described in [Well-known OIDC federation endpoint for RP](##rp-well-know). 
If the Federated OP returns a success response, Keycloak should parse the returned Entity Statement and apply the required configuration changes to the Federation OIDC identity Provider. 
The returned Entity Statement will contain at least Client Secret (in the case of client secret post/basic auth/jwt client authentication) and Client Id (the OP may choose this value to be different from the RP’s entity identifier). 
The OP will choose the trust anchor id of RP registration. `trust_anchor_ids` will contain only the returned trust anchor id.
Without a successful registration process, the Federated OIDC identity Provider configuration will be saved with `enabled` set to `false`. A Runnable Task will be executed periodically to renew the registration(s) with federated OIDC Providers.


## Testing

We can implement test code using an existing test framework.

## Documentation

We need to add explanation about supporting the spec into [keycloak-documentation](https://github.com/keycloak/keycloak-documentation). [Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/index.html) looks like a better place to explain it.

## Implementation proposal

**Source:** https://github.com/eosc-kc/keycloak/tree/openid-federation-op

**Current limitations:**
Current implementation is related to supporting configuring Keycloak as an Openid Provider that can participate in an OpenID Connect Federation. 
[Expired Client](#expired-client) and [Automatic Registration](#automatic-registration-1) of Openid Provider (OP) has not been implemented yet.
More tests are needed and implementation may be added as Keycoak extension.
