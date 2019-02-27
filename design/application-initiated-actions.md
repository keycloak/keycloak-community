# Application Initiated Actions

* **Status**: Draft
* **Author**: [stianst](https://github.com/stianst)
* **JIRA**: [KEYCLOAK-9366](https://issues.jboss.org/browse/KEYCLOAK-9366)


## Motivation

Keycloak currently has required actions that are used to prompt the user to perform an action associated with their account after authenticating, but prior to being redirected to the application.

Examples include:

* Configure OTP
* Update Profile
* Verify Email

Implementing a custom required action is very easy including the ability to create associated forms with FreeMarker
templates. These can then be deployed to Keycloak as an extension and easily added to a realm.

One issue with required actions is that they require an admin to request a user to perform the action. It is not
currently possible for an application to request an action.

The idea for Application Initiated Actions came around becomes of two things. In the New Account Console we wanted
to make it easy to enable users to register custom two factor authenticators and secondly we have seen a large 
interest in having support for WebAuthn added to Keycloak. With Application Initiated Actions it is possible to
create custom two factor authenticators by implementing the required action SPI and with a FreeMarker form. 

If we wanted to have the same without Application Initiated Actions a custom two factor authenticator would require 
implementing the same, but in addition extend the New Account Console (ReactJS based) as well as extending the Account 
REST API. This would effectively mean implementing the same capabilities twice.

Further, as Application Initiated Actions are driven through the login flows it is possible to for example require a
user to re-authenticate prior to performing the action. This would be harder if the logic was added directly to the
new Account Console.


## Required Action SPI

 A Required Action provider implements the following methods:
 
 * void requiredActionChallenge(RequiredActionContext context)
 * void processAction(RequiredActionContext context)
 
 requiredActionChallenge returns a challenge as a HTML form prompting the user to enter the required values. processAction
 processes the form and applies any relevant changes to the user account.
 
 For Application Initiated Actions we need to add the following methods:
 
 * boolean supportsApplicationInitiation()
 
 This will allow marking what Required Action providers support being initiated by an application. If not specified
 the Required Action will not support application initiation.
 
 RequiredActionContext will also be extended to add isApplicationInitiated() to allow Required Actions to act differently
 depending if they are initated from an application or not. 
 
 
 ## Contract
 
 For a Required Action to be enabled as a Application Initiated Action it is required that the Required Action returns
 a HTML challenge in requiredActionChallenge without making any changes to the user account. In effect this means that
 the user should always be prompted to perform the action before any changes are made.
 
 Depending on the Required Action it can support a cancel button allowing the user to cancel the action and return to
 the application.
 
 
 ## Flows
 
 An application initiates an action through any of the standard OAuth redirect based flows by adding the kc_action 
 query parameter.
 
 For example the application would redirect to 
 ````
 ../realms/myrealm/protocol/openid-connect/auth
    ?response_type=code
    &client_id=myclient
    &redirect_uri=http://example.com
    &kc_action=configure_otp
 ````
 
 In the above if the user is already authenticated the user would be prompted to configure OTP prior to being redirected
 back to the application. The application would now get a new code to obtain new tokens which may be updated depending
 on what action the user performed.
 
 
 ## Client Authentication
 
 As the contract for an Application Initiated Action is that it MUST not make any changes without first prompting the user
 we probably do not need to authenticate the client or control which clients are allowed to initiate actions.
 
 The above is still to be decided and may change.
 
 If we need to authentication clients it poses a challenge as the OAuth flow does not authenticate the client until the
 code is exchanged for a token. There's also public clients to consider.
 
 We can not require the client to include a bearer token either as the auth endpoint is invoked with a web browser
 redirect which does not allow for adding the bearer token as a header and we do not want to include a bearer token
 as a query parameter.  