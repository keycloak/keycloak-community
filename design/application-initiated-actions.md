# Application Initiated Actions

* **Status**: Draft #2
* **Author**: [stianst](https://github.com/stianst)
* **JIRA**: [KEYCLOAK-9366](https://issues.jboss.org/browse/KEYCLOAK-9366)


## Motivation

Keycloak currently has required actions. With required actions it is possible to require a user to perform actions associated with their account after authenticating with Keycloak prior to being redirected to the application. Examples of actions can be requiring the user to update the profile or configuring OTP.

Implementing a custom required action is fairly straightforward requiring implementing one method that creates a form using FreeMarker templates and another method that processes the submitted form. Both the action and the FreeMarker templates can be deployed to Keycloak as an extension, where the template can be included alongside the code through theme resources.

One shortcoming with required actions is that they require the admin to register the action with the user. It is not currently possible for an application to request an action nor is it possible for a user to intiate the action for example through the account console.

As part of re-designing the account console which will be a SPA application backed by a new Account REST API one requirement was to not have to implement relatively complex operations like registering a OTP both as a required action and also as a REST endpoint and corresponding HTML/JS within the account console. Further, the introduction of the Account REST API opens up for developing completely custom account management applications or integrating into existing applications.

With that in mind it is wanted to be able to allow users to easily invoke such actions from the operation, but have the actual logic driven through the required actions mechanism.


## Application Initiated Actions

Application Initiated Actions is the proposed solution to allow applications and the Keycloak Account console to initiate actions that the user should perform on their account such as registering an OTP.

Leveraging the Required Action SPI and a standard OpenID Connect redirect based flow the application can relatively easily add links to initiate an action and will after the user have performed the action receive new tokens that may be updated with updated details from the user or use the Account REST API to obtain the new details.


## Action Consent

A required action by default is not available as an application initated action, but can be extended to permit being an initiated action. There are two types of initiated actions. Those that always prompt the user prior to making any changes to the users account and those that do not.

An example of an action that always prompt the user is updating the profile. In this case the user is presented with a form to enter new profile information and no changes are made until after the user has updated and submitted the form.

An example of an action that does not always prompt the user prior to updating is an action that links the users account to an additional social provider. For this action the user is automatically redirected to the social provider.

Actions that always prompt the user prior to making any changes do not need any additional consent. Actions that do not always prompt the user will need consent from the user, or alternatively the application can include an id_token_hint with the request that proves the application does not need consent from the user. This is done by checking if the client the id token was issued to has a scope on the manage_account role. If the id_token_hint is not included for such actions Keycloak will present a consent screen to the user prior to initiating the action.


## Flows

The application uses the standard OpenID Connect Authorization Endpoint to initiate an action in the same way as it would authenticate a user with the addition of a new parameter `kc_action` and optionally include the `id_token_hint` parameter.

As an example an application that wants the user to update the profile would redirect the user to the following URL:

````
 ../realms/myrealm/protocol/openid-connect/auth
    ?response_type=code
    &client_id=myclient
    &redirect_uri=https://myclient.com
    &kc_action=update_profile
````

When this URL is loaded Keycloak would first authenticate the user if not already authenticated then it would execute the `update_profile` action, which displays the update profile form to the user.

After the user has updated and submitted the form the user is then redirected back to the application. In the example above as the response_type is set to code this would mean Keycloak would redirect the user to the following URL:

````
https://myclient.com?code=asdfasdfsadf&state=asdfasdfasdf
````

After exchanging the code for tokens using the regular OpenID Connect protocol the application would then be able to see the update profile in the ID token. The application is also able to use the Account REST API if it has permissions to do so.


## Required Action SPI

 A Required Action provider implements the following methods:
 
 ````
 void requiredActionChallenge(RequiredActionContext context)
 
 void processAction(RequiredActionContext context)
 ````
 
`requiredActionChallenge` returns a challenge as a HTML form prompting the user to enter the required values. `processAction`
 processes the form and applies any relevant changes to the user account.
 
 For Application Initiated Actions we need to add the following methods:
 
 ````
 default InitiatedActionSupport initiatedActionSupport() {
     return InitiatedActionSupport.NOT_SUPPORTED;
}
 ````
 
As this is a default method, Required Actions do not need to be updated unless they should support initating by applications. To support initiating by applications the required action should override this default method and return either `InitiatedActionSupport.SUPPORTED` or `InitiatedActionSupport.CONSENT_REQUIRED`.
 
`InitiatedActionSupport.SUPPORTED` means the required action will always prompt the user prior to making any changes and Keycloak will initate the required action by invoking the `requiredActionChallenge` method on the action.

`InitiatedActionSupport.CONSENT_REQUIRED` means the required action will not prompt the user prior to making any changes. In this case Keycloak will display a consent form asking the user to permit the application to initiate the action prior to executing the required action. An application can be permitted to not require this consent if it has a scope on the manage_user role, where Keycloak will identify the application by looking at the aud field of the ID token supplied through the `id_token_hint` parameter.

A further optional method will be added to required actions that will allow the action to require re-authentication prior to executing the action:

````
default boolean initiatedActionRequiresReAuth() {
    return false;
}
````

To require re-authentication prior to initiating the action the required action should override this method to return true.

For actions that require re-authentication under the covers Keycloak will do the equivalent of the application passing prompt=login.


### Implementation Phases

Application Initated Actions can be broken into 3 implementation phases:

1. Only supporting required actions that will always prompt the user prior to making any changes (`InitiatedActionSupport.SUPPORTED`) and do not require re-authentication.

2. Support required actions that do not always prompt the user prior to making any changes (`InitiatedActionSupport.CONSENT_REQUIRED`).

3. Support required actions that do require re-authentication.


# Notes on id_token_hint

When parameters are passed as query parameters there is a risk that they are logged in the web servers logs. This is not an issue as it is mitigated by:

* id_token_hint is already an established parameter in OpenID Connect, which should mean it is safe to use 
* HTTPS is required and query parameters are encrypted with https
* Logs should not include details like this and/or kept secure
* The ID token is short lived
* The ID token can not usually be used even if obtained. This is due to the fact that applications do only receive ID tokens directly from Keycloak to authenticate users and don't accept them from other sources. Applications that for some reason do accept these for examples from a cookie in case rhe application is stateless should not pass the ID token as a query parameter. They can instead use post based flows.
* Keycloak will only skip the consent if the ID token is associated with the current session for the user. This further means the ID token can only be used on the same machine that it was issued for, which again reduces the risk of an ID token being missused if somehow obtained.
