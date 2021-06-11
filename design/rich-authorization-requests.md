# Rich Authorization Requests (RAR)

* **Status**: Notes
* **JIRA**: TBD


## Motivation

[Rich Authorization Requests][1] is designed to allow OAuth Client to include detailed information to an authorization request. Thus, Client can make a specific authorization requirement.

## Specifications

The spec introduces a new parameter "authorization_details" that allows clients to specify their fine-grained authorization, such as `please let me make a payment with the amount of 45 Euros` or `please give me read access to folder A and write access to file X`.

An example of authorization details using
the payment authorization object:

````
[
      {
         "type": "payment_initiation",
         "actions": [
            "initiate",
            "status",
            "cancel"
         ],
         "locations": [
            "https://example.com/payments"
         ],
         "instructedAmount": {
            "currency": "EUR",
            "amount": "123.50"
         },
         "creditorName": "Merchant123",
         "creditorAccount": {
            "iban": "DE02100100109307118603"
         },
         "remittanceInformationUnstructured": "Ref Number Merchant"
      }
   ]
````

The type of data contained in authorization details is infinite and depends on the field (Payment, Health, Tax Data, etc...)

Therefore, it will be more scalable to add the processing of authorization details as SPI (e.g. `RarProcessor.Class`). Then, an extension can come with the specific implementation.

However, there is some default configuration and behavior keycloak should implement:

- Add parameters in Authorization Server Metadata
- Add parameters in Client Metadata
- Accept `authorization_details` parameter in all places where the `scope` parameter is used
- Add authorization details to access tokens and token introspection

### Authorization Server Metadata

This parameter must be added to the .well-known/openid-configuration:

- **authorization_details_types_supported**

The authorization data types supported can be determined using the metadata parameter "authorization_details_types_supported", which is an JSON array.

### Client Metadata

- **authorization_details_types**

Clients announce the authorization data types they use in the new dynamic client registration parameter "authorization_details_types".


### Authorization Request

As per spec,
The `authorization_details` request parameter CAN be used to specify
authorization requirements in all places where the "scope" parameter
is used for the same purpose, examples include:

*  Authorization requests as specified in [RFC6749],

*  Access token requests as specified in [RFC6749], if also used as
   authorization requests, e.g. in the case of assertion grant types
   [RFC7521],

*  Request objects as specified in [I-D.ietf-oauth-jwsreq],

*  Device Authorization Request as specified in [RFC8628],

*  Backchannel Authentication Requests as defined in [OpenID.CIBA].

we will use it first in Authorization requests.

### Token Response

In addition to the token response parameters as defined in [RFC6749],
the authorization server MUST also return the authorization details
as granted by the resource owner and assigned to the respective
access token.

## Implementation details

SPI

````
public interface RarProcessorProvider extends Provider {}

public interface RarProcessorProviderFactory extends ProviderFactory<RarProcessorProvider> { }
````
e.g. implementation in extension

````
public class Psd2RarProcessorProvider implements RarProcessorProvider {}

public class Psd2RarProcessorProviderFactory implements RarProcessorProviderFactory { 

    public static final String ID = "rar-psd2";
    public static final String SUPPORTED_DATA_TYPE = []; //Json array schema definition
}
````

* Support advertisement of supported authorization details types in OAuth server metadata

````
OIDCConfigurationRepresentation config = new OIDCConfigurationRepresentation();
config.setAuthorizationDataTypesSupported(
    getSupportedAuthorizationDataTypes(RarProcessorProvider.class)
    
);
````

*  Check if authorization data type or authorization details is not conforming to the respective type
   definition when authorization_details is available in the authorization requests 

````
rarProcessorProvider.isAuthorizationDetailsCompliant(authorizationDetails)
````
After the user authenticate, rarProcessor should:

*  Determine how authorization details are shown to the user in the user consent
*  Exchange with the resource server if necessary
*  Enrich authorization details in the user consent process (e.g. add selected accounts or set expirations)
*  show the consent page

````
rarProcessorProvider.prepareAuthorizationDetails()
````
   
When the user submits the consent  

*  Merge requested and pre-existing authorization_details (e.g. in case of update)  

* store authorization_details within the grant

````
rarProcessorProvider.processAuthorizationDetails()
````

### Admin UI

AS should allow the possibility to select RarProcessor Implementation Module.

Files/Classes/methods affected:

## Tests
RAR should be properly covered by unit and integration tests.

## Documentation
RAR usage should be properly documented.

Affected documents: Securing Applications and Services Guide

## Resources
* [draft-ietf-oauth-rar][1]

[1]: https://tools.ietf.org/html/draft-ietf-oauth-rar-04
