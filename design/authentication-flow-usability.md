# Authentication sessions improvements

* **Status**: Draft #1
* **Author**: [Marek Posolda](https://github.com/mposolda)
* **JIRAs**: [KEYCLOAK-5179](https://issues.jboss.org/browse/KEYCLOAK-5179), Related issue is also [KEYCLOAK-6839](https://issues.jboss.org/browse/KEYCLOAK-6839)

## Current state & issues

In some scenario, there is the message "You took too long to login" displayed to the end users during login to Keycloak. Details and steps to reproduce
are in the description of [KEYCLOAK-5179](https://issues.jboss.org/browse/KEYCLOAK-5179).

This design tries to fix this bug as well as generally improve the usability of the authentication and fix some other potential usability issues.
The design is more technical and contains some references to java classes and internal Keycloak implementation, because there are various corner cases to handle
and implementor of this should take them into account and be aware of the implementation details anyway.

The design starts with the brief introduction of authentication session model and some description of what happens in Keycloak during authentication flow.
You can skip this part if you are already familiar with it.

The [next part](#the-you-are-already-logged-in-issue) is the description of the issue and 2 designs to address this.
[Minimal design](#minimal-approach) describes the basic solution, which should fix the bug and address the issue for the majority of cases.
[Advanced design](#advanced-approach) describes more complex solution, which should fix some other corner-cases, improve usability of authentication
in general and decrease the amount of memory. 

## Authentication sessions - current model

The authenticationSession is the kind of session, which encapsulates the informations about the authentication state. It
is valid during the authentication of user. There are 2 model classes:

* `RootAuthenticationSessionModel` - This encapsulates informations related to the authentication of user within whole
browser. The browser can have more screens/tabs with the Keycloak login forms opened. So rootAuthenticationSession contains map of 
AuthenticationSessionModels where each AuthenticationSessionModel represents one browser tab.

* `AuthenticationSessionModel` - This encapsulates informations related to the authentication of user within single browser tab. The informations here could be divided to 2 basic groups:

    * Client data: Data sent from the OIDC/SAML client, which initiated authentication. This is needed, so that Keycloak server knows what to do after authentication (EG. redirect to OIDC client with appropriate parameters)

    * Authentication data: Data related to the authentication process itself. It tracks what authenticator/requiredActions where triggered, informations attached by individual authenticators etc.



## Authentication flow - how it currently works


Authentication flow starts when OIDC/SAML client redirects to Keycloak server to request login. What happens under the covers is:
1. Keycloak looks if there is cookie AUTH_SESSION_ID in the browser.

    * If yes, it lookups the root authentication session by ID.

    * If not, or if lookup of root session failed, it creates new Root authentication session and adds the AUTH_SESSION_ID cookie with it's ID.

2. New Authentication session is created and added to current root session. The `tab_id` is randomly generated and used as a key in the map to lookup auth session within root authentication session.

3. There can be various login/required actions/consent forms shown during authentication. The parameters used by the login forms are:

    * `code`, `execution` - Those are used to track current state of authentication and are not important for this design.

    * `tab_id` - Used to lookup the authSession within the root authentication session

    * `client_id` - Used just for the error cases. See below in step 5

4. When the request arrives to Keycloak during authentication (either actions triggered from some authenticator or browser
refresh when some authentication form is shown), the Keycloak is able to lookup Root authentication session based on cookie AUTH_SESSION_ID and then the authentication session based on the `tab_id` . In case that rootAuthSession + authSession is found, the flow can continue.

5. In case that root authentication session is not found, the error page is shown. This is typically page with the error message and link "Back to application" . The application can be determined due the client_id parameter and points to the "base URL" of the client. The error message is usually _You are already logged-in_ in case that we have user session with
same ID as current authentication session (See below) or _You took to long to login_ if user session doesn't exists.  

6. After authentication is finished, the userSession is created with same ID as the current authentication session and the root authentication session is removed. The cookie AUTH_SESSION_ID is still kept to preserve session stickiness

7. When new browser tab is opened and login requested, then we go to step 1. Typically user is logged-in due the SSO and rootAuthSession + authSession is created and removed in very same request

### Logout

Authentication session is also used to track logout state. This is needed during frontchannel logout flow for OIDC/SAML as SSO logout can involve multiple redirects to clients and back. So it is needed to track what clients were already logged-out etc. The authentication session used for
logout is currently removed after logout together with root authentication session. This may be changed after fix of [KEYCLOAK-6839](https://issues.jboss.org/browse/KEYCLOAK-6839).


### KC_RESTART cookie

KC_RESTART is the cookie, which is created at the beginning of the authentication flow (Step 1 above). It contains client
informations encoded in the signed JWT token. The cookie is useful in case that root authentication session expires (Step 5 from
above), it can be used to re-create the new authentication session from the client informations supplied in the cookie. But
overally the cookie is not so useful as it contains informations just from single client and doesn't work properly with more
browser tabs.


## The "You are already logged-in" issue

The issue is described in the [JIRA](https://issues.jboss.org/browse/KEYCLOAK-5179)  in the description. Example how it probably often happens in real life is: 

1. User is logged in some javascript applications (EG. KC admin console) in many browser tabs

2. User clicks "Logout" in the KC admin console in one of the tabs. He is logged-out in current tab and then in all the other tabs within 5 seconds due the session iframe used by JS adapter.

3. Login form with username/password is displayed in all the tabs now

4. User opens the tab1 and successfully authenticate to the admin console with username/password

5. User clicks to tab2 when the username/password is already shown as described in the step 2. He fills the username/password now. ATM the
RootAuthenticationSession doesn't exists (was deleted in previous step), but userSession exists. Hence message `You are already logged-in`

Note there is no issue if after step 4 user directly opens the URL of the application like `http://localhost/auth/admin` as this triggers SSO login and will automatically authenticate user.


## Solution

There may be 2 possible ways to improve this. The minimalistic approach can be small amount of work, with handling most of the cases, but
still some potential usability issues. Advanced approach will improve usability of authentication in general in even more
cases than `You are already logged-in`, but is much more work.


### Minimal approach

1. In addition to parameters used in the authenticators. See [authentication flow above](#authentication-flow---how-it-currently-works), we will 
add new parameter `redirect_uri` containing the redirect_uri parameter. In addition to that, we will need to add `state` containing the `state` parameter (in case of OIDC) or `relay_state` (in case of SAML).
The easiest will be to encode all the parameters like base64(c=client-id&r=redirect-uri&s=state) and add into parameter like `client_data`.
In case of OIDC, we may also need to add the `response_mode` . In other words `client_data` may contain protocol specific data.
Probably new method may need to be added to `LoginProtocol` interface for this. 

2. When RootAuthSession is not found after confirm username/password form (Step 5 from [above](#the-you-are-already-logged-in-issue)), then instead of showing
`You are already logged-in` page, we will redirect user back to the application.
We can use the redirect_uri supplied from the `client_data` parameter for that and we can use the other data (state etc) to
properly transfer error to the client. Again, maybe new method may need to be added to `LoginProtocol` interface.

3. In most of the cases, when the application receives the error, it will immediatelly create new OIDC/SAML request and redirect again to the Keycloak

4. Now since it is new request, Keycloak will usually just authenticate user due the SSO and redirect back to the application as authenticated user.

5. Currently we remove whole root authentication session after authentication is finished.We can potentially change this behaviour and 
remove just the auth session corresponding to current tab_id and remove root auth session just if there are no more auth sessions (browser tabs).
This may fix lots of the `You are already logged-in` cases as auth session will be available in the root auth session.
However it will have impact on memory and at the same time, there will be still cases when the root
authentication session is expired and hence redirect to the application will be needed.


#### Minimal approach - challenges

1. From the user point of view, when the username/password screen is shown in tab2 in step 5 from [above](#the-you-are-already-logged-in-issue),
the user will be just automatically redirected as authenticated to the application after confirm username/password. Few browser redirects is "implementation detail", which most of the users won't notice.
However there could be some usability confusions. For example:

    * User will be authenticated even if he provides invalid username/password or even he clicks "Submit" without providing any username/password. As username/password verification didn't really happen in tab2, even if it looks to user as it happened. IMO this is not major issue and same behaviour can be seen by Twitter for example.
    
    * After confirm username/password screen, user won't see TOTP (or any other 2 factor authenticators) even if he potentially has 2 factor enabled. He will be directly redirected to the application. I am not sure about usability of this and if it's confusing to users or if it's ok.

2. Another potential challenge is handling of the error on the adapter side.

    * In OAuth2/OIDC, there doesn't seem to be any nice error code for this purpose. See https://openid.net/specs/openid-connect-core-1_0.html#AuthError and https://tools.ietf.org/html/rfc6749#section-4.1.2. It seems best error code might be "server_error" and we may add the custom error_description, so that our adapters can understand it and handle it.

3. In our adapters, we may need to doublecheck if behaviour is correct and user will be redirected back to KC as described in
the step 3 [above](#minimal-approach). For servlet application, I suppose it won't be an issue as redirect_uri is usually "secured" URI and hence automatically intercepted.
I think it may be similar for other server adapters (node.js, gatekeeper etc) but will be good to doublecheck.
For the keycloak.js, I think "login_required" will work fine and automatically redirect to KC, but for the applications with "check_sso" or without any "onLoad" value used,
(For example _js-console_ application from the deprecated keycloak-examples), the adapter won't automatically redirect to Keycloak. So some changes in the adapter
may be needed to achieve this.

4. For SAML, will be good to doublecheck the behaviour. I suppose that SAML adapters are server-side and hence is not an
issue to redirect back to the application.



### Advanced approach

Advanced approach would be more work, but will help with some more advanced use-cases described in the [minimalistic approach](##minimal-approach)
and would mean some more improvements regarding usability of authentication flows and memory consumption. Especially:

* Mitigate the usability issues above and improve usability of authentication in general

* Minimalize "error" redirects to the application

* Minimalize impact on the memory and ensure that most of the state is saved in the browser

The disadvantage is more refactoring and more work needed and the risk of some more regressions.


#### Advanced approach - solution proposal

From the high level point of view, advanced approach would mean that "authentication data" (See [above](#authentication-sessions---current-model) for what it means) are
ideally shared per whole browser - per all browser tabs. Client data are not tracked anymore in the authentication session on
the server, but in the browser - either in the request parameters sent across all requests or per browser cookie.

1. Every page with the authenticators will contain some small JS code. This JS code will be activated when the particular browser tab is shown and gains focus.
It may be possible to use the `onFocus` function on the body element for example:
```
<body onfocus="onFocus()">
```

2. In case that focus is gained on the browser tab, the page will be automatically refreshed and moved to the latest state. For example if user confirms username/password
in browser tab1, which moved to the TOTP screen, then tab2 will be moved in the background to the TOTP screen as well when it gains focus.
In details, when javascript recognize the change in the focus, then
tab2 will:
    * Send the browser "refresh" request. In the meantime, there can be some message on the screen like "Refreshing the page. Please wait", so user
    is not tempted to fill username/password form, which is not valid anymore. Or it could be just a shading full screen div or whatever.
    
    * Authentication session will be shared per whole browser (with some exceptions described below in section [different flows in same browser](#different-flows-in-same-browser).
    So on KC side, we will check the latest state from AuthenticationSession and update the page accordingly including URLs. This will allow all browser tabs will be updated to TOTP after confirm username/password in tab1
    
    * The JS code will need to be added somewhen to the HTML template, which is used in all the forms for Keycloak built-in authenticators,
    required-actions and consent screens. This may not work for custom authentication pages added by user themselves, in case
    they don't use the generic Keycloak freemarker template. We can't do much here and JS won't be triggered in that case. We
    just need to document this in the server developer guide and encourage users that it is recommended to inherit their
    authenticators from the common Keycloak freemarker template. Also we need to doublecheck that our
    authenticator/requiredActions quickstarts work with that.
    
    * In case that user triggers new authentication request in browser tab2 when there is already authentication in progress
    in tab1, KC will be able to lookup current authenticationSession and directly redirect user to the latest step. So for
    example assuming that user is already on the TOTP screen in tab1 and he starts new authentication request to OIDC endpoint
    in tab2, then he will be redirected to TOTP screen in tab2.
    
    * This also means that after authentication completed in tab1, KC redirects to the client application in particular tab as in
    current Keycloak master. In that case, authentication session will be removed (See step 7 below). When tab2, which shared 
    same authentication session, gains focus and the authentication session was already removed, it will start authentication from the
     beginning and create new authentication session. This will in most cases do SSO login. So in most cases, user will be just directly redirected to the application
     when particular browser tab2 gains focus.


3. As mentioned above, the client data, which are specific to each browser tab, won't be saved in the authentication session
anymore, but will be saved in the browser itself. This will be either in the request parameter sent across authentication requests or in the cookie, which will need to be "namespaced" to be specific per each browser tab. Few details to this:
    
    * The client data will be encoded in the signed JWT. It can be pretty similar to the current KC_RESTART cookie - in other words, some JSON data signed with the realm secret key.
    
    * In ideal case, the token can be added as separate parameter to the authentication requests. In other words, instead of
    parameters `code`, `execution`, `client_data` and `tab_id`, we will have `code`, `execution`, `tab_id` and `client_data_full` where
    `client_data_full` will contain this encoded token. No need for `client_data` parameter, which has just few of client data.
    
    * There is a URL limit of 2000 characters (excluding host, but including the path, query params and fragment). So to be
    on safe side, we can't include `client_data_full` as request parameter in case that is too long. I suggest to use 1200
    characters as a limit to be on safe side and have enough space for the other parameters. In this case, we will need to
    fallback to use cookie.
    
    * In case that cookie fallback is used, we may need to use the parameters like `code`, `execution`, `tab_id`, `client_data` and `cookie_suffix` as
    before. The `cookie_suffix` will be used just as a namespace, so
    that every browser tab can have it's own cookie. In other words in case that tab1 will use `cookie_suffix=MNBVCXZ`, then there
    will be cookie like CLIENT_DATA_MNBVCXZ containing the signed JWT for this particular tab. The `client_data` will be again
    used just for the error purpose as cookie can be expired/removed in some cases, so we will redirect to the application
    with the error in that case as described in the [minimalistic design](#minimal-approach).
    
    * NOTE: The even better approach will be to add some "boilerplate" suffix to the URL path and scope the cookie CLIENT_DATA to
    that particular URL path. This will have advantage that browser won't always send all the cookies like CLIENT_DATA_*, but
    just one of the cookies specific to particular path. However this approach will be more work and likely not needed as
    cookie is just a fallback...

4. The `client_data_full` will involve those client-related data, which are currently saved mostly in the authentication session:
    * redirect_uri
    * client_id
    * protocol
    * client_notes
    * client_scopes
    * auth_session_id

  All the mention data can be likely removed from the AuthenticationSessionModel.

  The `auth_session_id` will be included and KC server will always check if this match the authSessionId from the AUTH_SESSION_ID cookie.
  This is for security purposes, so the potential attacker can't easily supply to user his `client_data_full` to simulate
  redirect to some random client, which user didn't initiated. In case of non-matching AUTH_SESSION_ID and authSessionId from
  the parameter, we may need to redirect to the client with the error.

5. In case that authenticationSession is expired, we can re-create it easily based on the parameters 

6. We can get rid of KC_RESTART cookie entirely. It won't be needed anymore.

7. We can always remove authSession from the rootSession after authentication finished and remove rootAuthSession just in case there are not any more auth sessions in it.
This will be usually the case as authSession will be now "usually" shared among all browser tabs (See below in the section [different flows in same browser](#different-flows-in-same-browser) when this may not be true).

8. Regarding logout, there is nothing, which needs to be changed. We will still have authSession inside rootSession used to
track front-channel logout and remove just this particular authSession from the rootAuthSession (assuming the fix for https://issues.jboss.org/browse/KEYCLOAK-6839 is done).

##### Action tokens


Regarding action token use-cases (typically email involved use-cases like _Forget password_ etc), there is already action token,
which is sent as the parameter in the URL, which is sent in the emails. So due to this, it will be good to always use
"fallback to cookie" approach to avoid URL be too long. So the action tokens URL will still contain the parameters like `client_data`, 
`tab_id` and `cookie_suffix`.

* In case that action-token URL is opened in same browser where it was initiated, we will have AUTH_SESSION_ID cookie as well
as CLIENT_DATA_${cookie_suffix} available and it would be simply possible to continue.

* In case of different browser, the AUTH_SESSION_ID as well as CLIENT_DATA_${cookie_suffix} won't be there. The KC can still lookup
the authenticationSession by the ID from the actionToken and eventually continue with the particular flow. Full client data won't
be available, but those are not needed anyway because we don't want to redirect to client at the end of the flow, but
displaying some message with the URL "Back to the application" . This will be still available due the `client_data` parameter.

In shortcut, action tokens should work just fine :)


##### Different flows in same browser

Sharing authentication data described above may not be always possible due some corner cases when we have different flows
within same browser. This involves cases like:

1) Authentication flow override for particular client: The overriden authentication flow can have completely different
executions than the default flow etc.

2) Different flows in different tabs: For example tab1 may have "firstBrokerLogin" flow after successfully redirected from the
IDP when tab2 has browser flow.

3) Parameters like `max_age`, `prompt`, `kc_idp_hint` and maybe others. For example user is already authenticated, but triggers
request with `prompt=login` in tab2, so username/password screen is displayed here. In case that later in tab3 user opens
authentication request, he should be directly authenticated through SSO, not switched to username/password screen because
tab2 requested `prompt=login`

4) Step-up authentication: Assuming that tab2 wants some stronger LoA (level of assurance), user may be required to provide
some additional authentication steps (TOTP, WebAuthn etc) even if he was already authenticated. Other browser tabs shouldn't
be affected if they are fine with the default LoA and don't need TOTP.

Due the above, we may still need the more authenticationSessions in single browser and hence still have rootAuthSession.
The difference is that authentication session in most cases can be shared among browser tabs and hence have same `tab_id`
parameter in browser tabs where it can be shared.

What can be done is to compute `tab_id` at the beginning of the
authentication instead of always randomly generating it. Tab ID can be computed as a hash of the following things:

* Some random value, which is shared per whole rootAuthSession and saved at the rootAuthSession level

* flow_id: This will handle case 1 and 2

* Some OIDC/SAML parameters, which affects how is user authenticated. This will handle case 3 and 4. We may need
callback method on `LoginProtocol` interface, which will allow to specify, which protocol parameters affect authentication.
In case of OIDC it can be `prompt`, `max_age`, `kc_idp_hint`, `acr_values` and maybe others. In case of SAML, there is also
something similar to `prompt=login` and other similar things...

In most cases, the flow_id will be same in more browser tabs (pointing usually to realm browser flow) and there won't be
parameters like `prompt`, so `tab_id` will be same across browser tabs and allow sharing of authentication sessions.