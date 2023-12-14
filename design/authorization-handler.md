# AuthorizationHandler to process Dynamic Scopes and Rich Authorization Requests (RAR).

* **Status**: Draft #3
* **Github Discussion**: https://github.com/keycloak/keycloak/discussions/8532

## Motivation
 
After considering how Dynamic Scopes and RAR would interact with the system and the possible interactions between them, the team has decided to come up with a mechanism that will abstract the authorization mechanism used in the authorization request.

In this proposal, we’ll be merging the [Dynamic Scopes discussion](https://github.com/keycloak/keycloak/discussions/8486) and the [RAR Design Proposal](https://github.com/keycloak/keycloak-community/pull/266) into a single place.
 
So many considerations, benefits and implementation details of both proposals can be considered true here, as well as some of the ideas proposed in the [Dynamic Scopes vs RAR discussion](https://github.com/keycloak/keycloak/discussions/8488).

## Authorization Handler VS Client scope

In this design, we are introducing a new entity called Authorization Handler, which is supposed to replace existing Client Scope. There are two possible views how these changes can be seen:

* Introducing new entity "Authorization Handler", which is replacing "Client scope"
* Extending existing "Client Scope" entity with more capabilities described in this design (dynamic scopes and RAR requests in addition to just “static scope” requests supported by client scopes in current Keycloak)

The overall output should be the same in both cases. From the implementation and backwards compatibility perspective, it might be probably easier to re-use the existing Client Scope entity under the covers, but we are not 100% sure (this will require us to take a look at the current implementation to decide about the best path).

Regarding migration and backwards compatibility, we will need to make sure that existing Client Scopes would be migrated to the Authorization Handler. So users migrating from Keycloak 15.0.2, who have set up some Client Scope will still have exactly the same functionality in new Keycloak versions as before.

In the next text, it can be assumed that Authorization Handler and Client Scope are the same entity for all purposes. So by Client Scope is meant the "Dynamic" client scope, which will be able to handle dynamic scopes and RAR requests. Not just "Static client scope" as available in Keycloak in version 15.0.2.

**Note**: We need community feedback on what the best path would be, and if we decide to go with the authorization handler, it would also be great to get a better name for it.

## The Client Authorization Handler

A new type of entity will be created in Keycloak that will allow users to configure the behaviour of Static, Dynamic Scopes and RAR in a composable way.

The ClientAuthorizationHandler will be a composite structure consisting of the following pluggable components:

* The ClientAuthorizationContext initiator
* A Compliance processor
* A consent screen enhancer / builder
* Configured Protocol Mappers.


### ClientAuthorizationContext initiator

**NOTE**: The `initial OIDC Authorization-Endpoint request` is the initial request in the default OAuth2/OIDC Authorization-code flow. However various grants and specifications related to OAuth2 and OpenID Connect allows that initial request to be sent in different ways. For example BackchannelAuthenticationEndpoint in case of CIBA, Pushed-Authorization-Request endpoint in case of PAR, DeviceEndpoint in case of OAuth2 Device 
Grant or TokenEndpoint in case of OAuth2 Resource-Owner-Password-Credentials Grant. All those endpoints are entry point for particular authentication use-case and they can accept `scope` or `authorization_details` parameters.  Any mention to the  "Initial Authorization-Endpoint request" can be assumed to be of any of these "initial request" types.

At the initial step, the authorization data sent by the client can be retrieved from the initial Authorization-Endpoint request. This retrieved data will be referred to as AuthorizationRequest, which means the initial request to Keycloak to start the authentication process.

SAML, on the other hand, would still only the support the default client scopes as it does today in Keycloak 15 or earlier.

Additionally to everything said above, the `AuthorizationDetails` object will be validated at the beginning of the request, just like Scopes are validated currently in Keycloak by `AuthorizationEndpointChecker.checkValidScope()`.

### How to parse authorization data

The authorization data is available in the OIDC request by 2 ways:

* In the scope parameter. The scope parameter is already supported in current Keycloak, but rather in static way

* In the authorization_details parameter as defined in the [RAR specification](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-rar-07).

Once Keycloak parses the data from any of these 2 parameters, it will save them in the ClientAuthorizationContext object. Note that same request can contain both `scope` and 
`authorization_details` parameter and hence Keycloak should be able to parse both of them in the same request. The idea is to unify the data in the ClientAuthorizationContext under some structure, which will serve as an abstraction layer over the data that was sent via the scope parameter or authorization_details. However Keycloak will still need to have some flag in ClientAuthorizationContext to track how the data was sent. This may be needed for various reasons. 

For example [Grant Management API specification - section 6.4](https://openid.net/specs/fapi-grant-management-01.html#section-6.4) (not yet supported by Keycloak) allows clients to query Keycloak about the granted consents. Note that the returned structure contains both scope and authorization_details and hence it is needed by KC to track how the particular request was sent.

### ClientAuthorizationContext object

Output of the parsing is ClientAuthorizationContext object, which is a list of AuthorizationDetailsEntry objects. Each AuthorizationDetailsEntry contains data like:

* JSON with the authorization details
* ClientAuthorizationHandler used
* Source - the parsing method (scope parameter or authorization_details parameter)

#### Java pseudo-structure

```java
class ClientAuthorizationContext {
	/**
 	*  List of entries of parsed values from "scope" parameter and    
      * from "authorization_details" .
 	*  Each entry represents single parsed value from "scope"
      * parameter or single entry of "authorization_details"
 	*
 	* There are also entries corresponding to all "default client
     * scopes" for the particular client, which are added
     * automatically.
 	*/
	private List<AuthorizationDetailsEntry> entries;
}

class AuthorizationDetailsEntry {
    	// Client scope linked with this authorizationDetails
    	ClientScopeModel clientScope;

    	// How this entry was added. See enum "Source" below
    	Source source;

    	// AuthorizationDetails data sent by the client. In case that
// this was added by "scope" parameter, the "scope" entry is wrapped
// to the authorizationDetails. See below for the details:
      AuthorizationDetailsJSONRepresentation authzDetails;  	 

}

enum Source {
    	// Entry added with "scope" parameter
    	SCOPE_PARAMETER,

    	// Entry added with "authorization_details" parameter
    	AUTHORIZATION_DETAILS_PARAMETER,

    	// Entry added automatically by default client scope
    	DEFAULT_SCOPE   	 
}
 	 
class AuthorizationDetailsJSONRepresentation {
   	// These fields are from RAR specification
   	private String type;
   	private List<String> locations;
   	private List<String> actions;
   	private List<String> datatypes;
   	private String identifier;
   	private List<String> privileges;
  	 
   	// Custom authorizationDetails data for particular type
   	private Map<String, Object> otherData;
}
```


## Parsing

Assumption of this approach is that there is no limitation on ClientAuthorizationHandler whether it can be added to the request by “scope” parameter or by “authorization_details” parameter.

Requests will go through the chain of parsers. By default, we will have three parsers defined:
* StaticScopeParser
* ScopeParameterParser
* AuthorizationDetailsParameterParser

The **AuthorizationDetailsParameterParser** will go through all values of the “authorization_details” parameter and just use the JSON value as it is. The ClientScope will be determined by the type of authorization_details, which needs to match the name of the client scope.

Note that the `AuthorizationDetailsJSONRepresentation` is supposed to represent `authorization_details` and 
static and dynamic scopes. See below for the examples on what the JSON from the parsed scope representation looks like.

### AuthorizationDetailsParameterParser

A sample RAR payload would look like this:

```json
{
   "type": "payment_initiation",
   "locations": [
  	"https://example.com/payments"
   ],
   "instructedAmount": {
 	"currency": "EUR",
 	"amount": "123.50"
   }
}
```

The AuthorizationDetailsParameterParser will be used to parse it and the `type` will correspond to the **payment_initiation** client scope.

### ScopeParameterParser, 

A ScopeParameterParser will be configured with a regex that will be checked against all scopes to look for a matching value. By default we could consider a `*` parser to allow all scopes.

When a Dynamic Scope is parsed by the ScopeParameterParser, the resulting JSON will only contain the `type` and `identifier` ` properties.

Given an Authorization Handler with a ScopeParameterParser configured with a regexp such as `group:(.*)`. A scope such as `group:123` would match, and the resulting JSON would look like:

```json
{
	"type": "http://keycloak.org/auth-type/group",
	"identifier": "123"
}
```

Given an Authorization Handler with a ScopeParameterParser configured with a regexp such as `tenant:(.*)#group:(.*)`. Scope parameter value like `tenant:123#group:456` would correspond to JSON like this:

```json
{
	"type": "http://keycloak.org/auth-type/tenant-group",
	"identifier": {
    "tenant": "123",
    "group": "456"
  }
}
```

### StaticScopeParser

This parser would work as the default ClientScopes in Keycloak. An Authorization Handler would be created and defined as a Static Scope parser with the scope that will be supported. 

A static OAuth scope such as `address` will be parsed similar to the previous cases, but with the `identifier` field missing:
```json
{
	"type": "address"
}
```

The client scope will be determined again from the type field.

The request will be rejected in case that there are any unknown scope values or unknown authorization_detail types, which don't correspond to any configured authorization handlers in the realm. The error returned to the client is `invalid_authorization_details` as described in the [RAR specification Section 8](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-rar-07#section-8) or `invalid_scope` as it works today and described in the OAuth2 specification.

### ClientAuthorizationContext in AuthenticationSessionModel

The ClientAuthorizationContext can be sent as a field in the AuthenticationSessionModel. Hence it will be available in all the requests of this Authentication session. Existing AuthenticationSession methods “get/setClientScopes” may be deprecated and replaced by “get/setAuthenticationContext”

## Compliance processor

There will be a chain of policies configured in the admin console for a specific Authorization  Handler. The ClientAuthorizationContext can be processed by this policy chain. Each AuthorizationDetailsEntry will be processed by each of the policy and policy will decide if:

* AuthorizationDetails is required to be rendered on the consent screen
* AuthorizationDetails is unsupported/invalid/unauthorized and whole client request should be rejected

Policy can also “enrich” and/or change authorizationDetails entries as sometimes client sent “incomplete” authorization_details and it is up to the AS (Keycloak server) to figure correct values and then in later stages, display them to the user on consent screen or add them to the token. Some use-case described in the [RAR specification Section 7.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-rar-07#section-7.1).

Policy is also responsible for processing the consent screen (The moment when a user submits the consent screen, which can happen multiple times during a single “authentication session” due the fact that more consent screens would be possible with this proposal).

### Naming

Maybe ClientScopesPolicy is ok as a name? IMO any name containing "Authorization" (EG. ClientAuthorizationContextPolicy, AuthorizationDetailsPolicy) can bring the confusion with the authorization services, which also use "Policies"... More wider question if ClientAuthorizationContext itself is good as a name?

### Configuration

Policies can be configured at the client scope (or ClientAuthorizationHandler) level. In addition to the Mappers and Scope tabs, there can be another tab like Policies. By default, each client scope can have only Persistent Consent Policy automatically set (See below for Persistent Consent Policy details).

### When

The policies are processed after the user is authenticated and required actions processed. Hence before the render of consent screen. So policies have access to existing authenticated users and current clients.
In addition, policies are processed after each confirmation of the consent screen (When the user presses “Submit” on the consent screen). This is needed as consent screens in some cases can contain user data and require input from the user (For example there will be a list of accounts on consent screen and users will need to choose one of them).

We can see if we need another way to process at the beginning of the request to AuthorizationEndpoint even before the user is authenticated (EG. For ability to reject request quickly)


### Pseudo-code

```java
interface ClientScopePolicy {

	PolicyVote applyPolicy(ClientScopePolicyContext ctx);

}

public class ClientScopePolicyContext {
    
	// context retrieved from parsing from previous step
	private AuthorizationDetailsContext authzDetailsCtx;

	// authSession. This wraps client (among other things)
	private AuthenticationSessionModel authSession;

	// already existing consent of the user for the client. Null if 
     // no consent exists
	private UserConsentModel existingConsent;

     // Input data confirmed by the user on the consent screen.
     // This is null if policy is processed before consent screen was
     // shown for the first time
     private Map<String, Object> formData; 
    
}

enum PolicyVote {

	// Reject whole request.   
	REJECT,

	// This AuthorizationDetails should be rendered on consent screen  
	RENDER,

	// Skip rendering of this authzDetailsEntry
	SKIP,

	// Policy doesn't vote
	ABSTAIN

}
```

### Default policy implementations


**Persisted Consent Policy**: Will check persisted UserConsentModel and whether the corresponding AuthorizationDetailsEntry was already persisted. When the user confirms the consent screen, it will save the confirmed UserConsentModel to the DB (store).

This corresponds to the same behaviour as the current Keycloak. Besides the fact that it will be able to also persist authorizationDetails to the UserConsentModel (See section UserConsentModel changes below).


**Groups & Tenant Policy**: Builtin Keycloak policies to support multitenancy and other needs. These can be used for example to:
Reject the request if particular group with ID `123` does not exists (In case of scope parameter like `groups:123` was used)
Enrich the authorizationDetails according to Keycloak needs. For example for the scope like `tenant:123#group:456`, the parsed request like this:

```json
{
  "Type": "tenant",
  "Identifier": "123#group:456”
}
```

And policy will further enrich it to something like this:

```json
{
    "type": "http://keycloak.org/auth-type/tenant-group",
    "Locations": [ "https://<keycloak-url>/{realm}/tenants/123/group//456" ],
    "Identifier": {
        "tenant": "123",
        "group": "456"
    }
}
```

Note the policy might be able to change the type and the underlying client scope (or authorization handler)

**Script Policy**: Will be able to read from some script, like for example Javascript (See existing JavascriptPolicy or ScriptAuthenticator for inspiration). This can be useful to achieve more advanced authorization details use-cases. For example this authorization details were already approved by user before:

```json
{
  	"type": "payment_initiation",
  	"locations": [
     	    "https://example.com/payments"
  	],
  	"instructedAmount": {
     	    "currency": "EUR",
     	    "amount": "125.00"
  	}
   }
```

Now when a user sends another request requesting a smaller amount of money (EG. 120 EUR only), the consent can be automatically considered as it was approved before. However for a bigger amount (EG. 130 EUR), the consent is required again for the user as he previously approved that the client can initiate payments just with a smaller amount of 125 EUR.

This requires some domain knowledge for concrete type and hence requires an administrator to implement these policies by himself.

Script policy may also be able to enrich authorization_details. For example to address the RAR use-case described in the [RAR specification Section 7.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-rar-07#section-7.1), it can do:

* Retrieve accounts from the Keycloak DB and add them to the authorization details
* Retrieve accounts from the resource-server and add them to the authorization details. The URLs of the resource-server can be figured from the locations address of the authorization_details object or through some other way.

Then again some custom script may be needed to put data to specified place in the JSON of authorizationDetails.

**Resource policy**: The RAR specification allows users to define a locations array that can be used to query the resource server for extra authorization information.

Users will be able to define a policy that will perform a request to one of the locations in the authorization_details object and extract data from the response with JSON Path/Pointer.

This data can either be added back into the authorization_details object or into the user session notes.

Unlike the Script Policy defined above, it won’t require custom scripting, but it will allow for less complex use cases.

**OperatorPolicy**: A policy that will match a value against others with some predefined operators such as “>, <, =, >=, etc). 

Users will be able to configure a Source and a Target data source which can be any data from the auth session, the user profile, consented scopes and values or the result of previously executed policies.

Again, making use of JSON Pointers / Path to select which specific fields will be compared with the selected operators.

## UserConsentModel changes

Class UserConsentModel represents binding between single user and single client. There is a single consent model per combination of user and client and we will likely keep it like that.

Current UserConsentModel contains only a static list of ClientScopeModel. We will need to extend it to contain all the details of authorization details. Also it will be good to add grantId field to support grant management API specification, which allows (among other things) [querying existing grants for client and user](https://openid.net/specs/fapi-grant-management-01.html#section-6.4) and reference this query by grantId. So newly UserConsentModel may need to contain fields like:

```java
String grantId;
List<AuthorizationDetailsEntry> authorizationDetails;
```

The AuthorizationDetailsEntry object is described above. There is a possibility to remove existing field clientScopes with a list of client scope models. This is replaced by the `authorizationDetails` field.


### Grant management for OAuth 2

In the later implementation milestones of this specification, we can consider adding basic support for [Grant management API](https://openid.net/specs/fapi-grant-management-01.html).  This would mean adding some claims to the token response and support for some claims in the AuthorizationRequest (grant_id, grant_management_action etc). Also possibly adding some new endpoints (like querying consents).

At the first phase of this implementation, Keycloak won’t yet support [Concurrent grants](https://openid.net/specs/fapi-grant-management-01.html#section-3.5). Adding support for them will require more refactoring of models as currently Keycloak always has just a single UserConsentModel per the combination of user and client. Hence this might be postponed for now until the new store is available. Also it will require changes in various other screens in Keycloak including UI. Question is, if we ever need concurrent grants, but that is something to be discussed as an additional step.


## Consent screen enhancer / builder

The consent screen will be reworked to handle more complex use cases as well as to request input from the user for certain dynamic scopes and RAR types.

Right now the consent screen is very static, Keycloak allows the users to define client scopes and a non variable text that will be shown to the user.

With the change to how we handle these new authorization mechanisms, we plan to improve this screen to allow the following functionality:

* Expose variables so users can fully customize the text they show for Dynamic Scopes and RAR.

* Allow requesting data from users when a certain type of authorization data is used. For example: when defining a dynamic scope to enforce certain context in tokens, or to define variables in RAR requests.

* Allow users to select which scopes they want to consent. Right now it’s all or none.

The default behaviour of the consent screen will allow multiple fields to be selected / input in the consent screen itself. Some handlers may let users configure this and detach it from the main consent screen so input selection can be done in its own different screen.

### Exposing variables for text customization

We should also strive for giving more flexibility to define the text that is shown in the consent screen, based on the provided data from the authorization requests.

The server could use either [Json Pointer](https://datatracker.ietf.org/doc/html/rfc6901) or [Json Path](https://jsonpath.com/) to access the authorization_details data in a way that users can obtain certain information to use in the consent text.

Example:

An authorization_details object sent to the server.
```json
{
   "type": "payment_initiation",
   "locations": [
       "https://example.com/payments"
   ],
   "instructedAmount": {
       "currency": "EUR",
       "amount": "125.00"
   }
}
```

If the user wanted to show, say, “Allow the client to send 120.00 EUR?”, the actual string that the user would configure in Keycloak would look like:

"Allow the client to send `${instructedAmount.amount} ${instructedAmount.currency}`?” 

### Composable consent screen


The consent screen will now be composed of multiple HTML templates that can be configured by each authorization handler.

The base template will be defined by the static scopes that are sent and configured in the server, just like it’s being done right now.

ClientAuthorizationHandlers will be able to configure some predefined templates to add to this.

For now, the following default templates could be considered:

* Dropdown selection consent template: These will allow handlers to request a value from a dropdown. These can contain fixed values or values calculated from the system, such as tenants or groups. For example: When sending the group:* dynamic scope, it would contain all the groups for the requesting user.

* Value selection consent template: These will allow handlers to request a value from the users in order to add it to the consent screen. For example: Transfer amount, source/target account etc.

These selections will be subjected to the policies defined by the handler, which will execute every time there’s a new value being added.

**Question**: Should we also consider further consent screen enhancement that may be out of scope from this proposal now that we are improving it? I’m thinking of showing a client image, or name and UX improvements.

Rough implementation pseudo code:

```java
public class ConsentScreenComposer {
	List<ConsentScreenElement> consentScreenElements;

	private List<ConsentScreenElement> buildConsentScreen() {
         List<ConsentScreenElement> computedElements = new <>ArrayList();    
         for(ConsentScreenElement element : consentScreenElements){
             if(element.showInSeparateScreen()) {
                Object userInput = element.show();
			CheckExistingPolicy(userInput); //throws 
   			computedElements.add(new StaticConsentScreenElement(object));  
             } else {
             	computedElements.add(element);
             }
         }
         createConsentScreenView(computedElements);
     }
     
     public void showConsentScreenView() {
         List<ConsentScreenElement> elements = buildConsentScreen();
         // build html page in here
     }
};
```

**Consideration for this question**: There's been some work on the consent screen with a few enhancements (https://github.com/keycloak/keycloak/pull/8092). We should review the work done here and adapt it to the needs of this proposal.

## Configured Protocol Mappers

Protocol Mappers are right now attached to (Static) Client Scopes, and are executed whenever a scope is requested that matches the configured scopes for a client.

When defining a ClientAuthorizationHandler, for either Dynamic Scopes or RAR, we should offer users the ability to also configure Protocol Mappers that will run when the handler is invoked.

These mappers will not be any different from the ones already available, but they will have access to the `ClientAuthorizationContext` object.

Ideally, we should be able to create some generic mappers that will receive configuration on how to handle the data passed in the `ClientAuthorizationContext`, and how to map it to specific claims in the tokens.

Given the amount of use cases that Dynamic Scopes and RAR would cover, this could prove hard to achieve and custom mappers would be necessary, but we should cover as many use cases as possible with built-in mappers.

### Example of a new built-in mapper

A mapper that allows the user to define a few JSON Pointers/Paths from a source (UserSession, ClientAuthorizationContext etc) and another JSON Pointer/Path to claims in the tokens.

For now, we can build a complex JSON Object with json pointers and have Keycloak parse them into the correct format. I.e: `${someClaim.name}` and `${someClaim.lastName}` would be transformed to:

```json
{
   "someClaim": {
        "name": "…", 
        "lastName": "…"
    }
}
```

An evolution of this mapper could let users define a JSON Schema with some JSON
pointers so instead of having a list of mappings, they could define the final structure
of their claim and where to get the data from.

## Successful Authentication Outcomes

In this section, we’re going to showcase how a Dynamic Scope would be parsed and processed by the system.

Keycloak will receive the Dynamic Scope and parse it to the common RAR format, so it’s not necessary to go through successful outcomes for RAR.

### Dynamic Scope successful outcomes

**Scenario 1**: An authorization request sent with the following information:

`GET /auth?scope=openid+groups:123`

The ClientAuthorizationContext object is created, the scope gets parsed to an authorization_details object, resulting in:

```json
{
  "type": "http://keycloak.org/auth-type/group",
  "locations": [
     "https://<keycloak-url>/{realm}/groups/123"
  ],
  "Identifier": "123"
}
```

After a policy pass, there’s no policy that prohibits the execution of this authorization context.

The ClientAuthorizationHandler that matches this scope has configured an embedded consent choice, but as the request is already providing a value, the consent screen just has a text for this specific scope.

Mappers that have been configured for this ClientAuthorizationHandler run, having access to the authorization_details and the ClientAuthorizationContext data.

**Scenario 2**: An authorization request sent with the following information:

`GET /auth?scope=openid+groups:*`

The ClientAuthorizationContext object is created, the scope gets parsed to an authorization_details object, resulting in:

```json
{
  "type": "http://keycloak.org/auth-type/group",
  "locations": [
     "https://<keycloak-url>/{realm}/groups"
  ],
  "Identifier": "*"
}
```

After a policy pass, there’s no policy that prohibits the execution of this authorization context. There may be a policy that allows/denies user interaction for this specific scope.

The ClientAuthorizationHandler that matches this scope has configured an embedded consent choice. As the scope has been sent with a wildcard, the consent screen is built with a dropdown containing all the groups that the user is part of.

After User Selection and Consent Screen acceptance, mappers that have been configured for this ClientAuthorizationHandler run, having access to the authorization_details and the ClientAuthorizationContext data, containing the user choice. 

Scenario 3: An authorization request sent with the following information:

GET /auth?scope=openid+tenants:tenantA#groups:123


The ClientAuthorizationContext object is created, the scope gets parsed to an authorization_details object, resulting in:

	Note: This one may be way more tricky to map to a RAR format, as we have a
  	composite type (tenants/groups). We may need to pre-define types for these
	combinations.

```json
{
  "type": "http://keycloak.org/auth-type/tenant-group",
  "locations": [
 "https://<keycloak-url>/{realm}/tenants/tenant1/groups/123"
  ],
  "Identifier": {
    "tenant": "tenantA",
    "group": "group123"
  }
}
```

After a policy pass, there’s no policy that prohibits the execution of this authorization context.

The ClientAuthorizationHandler that matches this scope has configured an embedded consent choice. But the composite scope contains no wildcards, so no need to ask the user to choose. This ClientAuthorizationHandler has a configured text that uses variables from the RAR request such as:

"Allow the client to get details of group `${identifier.group}` in tenant `${identifier.tenant}`”

After User Selection and Consent Screen acceptance, mappers that have been configured for this ClientAuthorizationHandler run, having access to the authorization_details and the ClientAuthorizationContext data, containing the user choice. 

## Admin UI changes

The current (legacy) Admin UI will not receive any changes for this feature. All changes to the UI will be focused on the new Admin UI that will be coming.

A joint effort of the UI and UX design teams will be required to create a unified new menu item and entity called Authorization Handlers or extend the current Client Scopes with new functionality.

This new entity would handle the following authorization mechanisms:

* Static Scopes
* Dynamis Scopes
* RAR

And any other future OAuth / OIDC authorization mechanism that needs to be handled by Keycloak.

Among the new configurable options, we'll have:

* Allow users to decide which kind of object will be handled (Static or Dynamic Scopes, or RAR).
  * Define specific confugration for each kind
* Allow users to add policies and configure them for this specific object.
* Allow users to define which kind of Consent Screen fragment they want to use, and any custom text for this specific object.
* Allow users to add mappers.