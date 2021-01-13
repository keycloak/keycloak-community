# Token Exchange SPI

## Abstract
This document outlines the design of the proposed Token Exchange SPI for Keycloak (TODO: JIRA link).
## Rationale
Keycloak does implement the OAuth 2.0 Token Exchange ([RFC 8693](https://tools.ietf.org/html/rfc8693)), but does that in a peculiar way (Securing Applications and Services Guide, [7. Token Exchange](https://www.keycloak.org/docs/latest/securing_apps/index.html#_token-exchange)):
> Token exchange in Keycloak is a very loose implementation of the OAuth Token Exchange specification at the IETF. We have extended it a little, ignored some of it, and loosely interpreted other parts of the specification.

Current implementation has the following limitations and shortcomings:
* The following (optional) parameters are not supported:
	* `resource`
	* `scope`
	* `actor_token` and `actor_token_type`
* Supporting internal exchange to/from non-OAuth clients requires core Keycloak modification. Currently, only exchange to SAML is supported;
* Exchanging to/from external IdPs is supported for OAuth IdPs only and requires brokering to be set up;
* Token exchange is built into `TokenEndpoint` which is already ~1400 LoC.

Considering the above, we are suggesting the introduction of the Token Exchange SPI.

## Use Cases and Benefits
* Complement Protocol SPI: allow to support exchanging to/from non-OAuth clients without modifying Keycloak
* Complement Identity Provider SPI: allow to support exchanging to/from non-OAuth IdPs without modifying Keycloak
* Non-brokered trust: allow to recognize, validate and exchange tokens issued by 3rd party entities in the cases when the trust cannot be expressed in terms of brokering

The SPI will also allow to clean up `TokenEndpoint` and to improve its maintainability.

## Design
* `TokenExchangeSPI`: SPI class
* `TokenExchangeProviderFactory`: factory interface
* `TokenExchangeProvider`: main (provider) interface; should define the `exchange()` method:
```
AccessTokenResponse exchange(TokenExchangeContext);
```
* `TokenExchangeContext`:
	* should supply the data needed to perform exchange (parsed request parameters);
	* should provide access to `KeycloakSession`;
	* may also contain auxiliary helper objects like `TokenManager`.

### Configuration

#### Provider level
Token exchange providers could be configured using the standard `org.keycloak.Config` API. The configuration will be global, and will be stored in the filesystem (or whichever config store is selected).

#### Instance level
In addition to that, we can provide instance-level configuration, similar to authenticators. The configuration will realm-local, and will be stored in the database. The relevant UI parts will need to be added to the Admin Console.

### Provider selection
Based on the request data, one and only one token exchange provider should be selected. There are several options for that:

#### Provider-side selection
The selection process could be delegated to the providers themselves. For that, `TokenExchangeProvider` could define the additional `supports()` method:
```
boolean supports(TokenExchangeContext);
```
This method alone won't be sufficient, since any two providers can return `true` simultaneously. To resolve the possible conflict, we'll need to introduce deterministic ordering of the providers (e.g. via the `TokenExchangeProviderFactory#getPriority` method).

This option assumes that the provider selection should be the responsibility of the provider developer (and not end user).

#### Policy-based selection
As a different option, the selection process could be decoupled from the provider code completely. For that, we should introduce policies (predicates) that would operate on the request data and return a decision; each policy will be associated with a single provider or a processor (configured provider instance).

This option assumes that the provider selection should be configurable by end users (and thus would require a UI).

## Implementation plan
### Phase 0
(Backbase internal PoC)
### Phase 1
* Add Token Exchange SPI interfaces and classes
* Refactor `TokenEndpoint` to use the new SPI
* Move Keycloak's token exchange implementation to `DefaultTokenExchangeProvider`
* Implement provider-based selection
### Phase 2
* Convert Keycloak's built-in token exchange methods to the new SPI
* Use Grant Type SPI
### Phase 3
* Implement policy-based provider selection
* Implement token exchange processors (configured provider instances)
## Relation to other SPIs
### Grant Type SPI
The RFC says:
> The client makes a token exchange request to the token endpoint with
   an extension grant type using the HTTP "POST" method.

Thus, token exchange is presumed to be implemented as extension grant (see RFC 6749, The OAuth 2.0 Authorization Framework, [4.5. Extension Grants](https://tools.ietf.org/html/rfc6749#section-4.5)). Once Keycloak has Grant Type SPI and proper extension grants support, the whole token exchange feature should be ported to the new SPI.
## Relation to other tasks/epics
[KEYCLOAK-8276: Review Token Exchange](https://issues.redhat.com/browse/KEYCLOAK-8276)