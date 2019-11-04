# Application Initiated Actions

* **Status**: Final #2
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

Actions always prompt the user prior to making any changes do not need any additional consent.


## Flows

The application uses the standard OpenID Connect Authorization Endpoint to initiate an action in the same way as it would authenticate a user with the addition of a new parameter `kc_action`.

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
https://myclient.com?code=asdfasdfsadf&state=asdfasdfasdf&kc_action_status=success
````

The kc_action_status allows the application to identify if the user completed the action or not. Valid values are success, cancelled or error.

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

2. Support required actions that do require re-authentication.


# Notes

In Final #2 it was believed that an option to require consent would be needed. However, this will be very uncommon and as such it will not be built in to AIAs directly, but rather actions will be required to implenent this themselves if needed.

Re-authentication was implemented in Keycloak 8, but only with a hardcoded re-authenticate after 300 seconds. More flexbility will be added here in the future as we introduce step-up and addaptive authentication in Keycloak.
