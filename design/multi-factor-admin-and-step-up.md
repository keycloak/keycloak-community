# Managing multi-factor authentication and Step-up authentication in Keycloak

* **Status**: Draft
* **Author**: [Alistair Doswald](https://github.com/AlistairDoswald)
* **JIRAs**: [KEYCLOAK-9693](https://issues.jboss.org/browse/KEYCLOAK-9693), [KEYCLOAK-9694](https://issues.jboss.org/browse/KEYCLOAK-9694) and [KEYCLOAK-847](https://issues.jboss.org/browse/KEYCLOAK-847)

## Motivation

The [W3C Web Authentication - Two-Factor](https://github.com/Keycloak/Keycloak-community/blob/master/design/web-authn-two-factor.md) document specifies as sub tasks
the administration of two authentication factors ([KEYCLOAK-9693](https://issues.jboss.org/browse/KEYCLOAK-9693)) and of multiple authenticators for users
([KEYCLOAK-9694](https://issues.jboss.org/browse/KEYCLOAK-9694)).

The purpose of this design document is to define how this is to be handled in the
administration console, and therefore by extension to how Keycloak handles authentication
on a logical level. The design may also specify functions, data structures and screen
mockups as the design progresses.

The design of the step up authentication ([KEYCLOAK-847](https://issues.jboss.org/browse/KEYCLOAK-847)) process is also defined in this proposal as it is linked to the multi-factor
authentication (MFA) and the mechanisms that are defined in the one will have an impact
on the other.

The design also has the following constraints:
* The design must be simple to use, but can be customised for more complex use cases. This is true for both the administrator when setting up the realm and the user when logging in with Keycloak.
* In addition to two factor authentication, the design must take into account the use with passwordless authentication (i.e. a single factor authentication that is not a password).
* The system must be easily extensible for groups that wish to add their own authenticators within Keycloak.

For [KEYCLOAK-9693](https://issues.jboss.org/browse/KEYCLOAK-9693) this means changes
to the authentication logic, changes to the authentication part of the admin console,
changes to the authentication screens that the user sees during login, changes to the
REST API, and changes to the database.

For [KEYCLOAK-9694](https://issues.jboss.org/browse/KEYCLOAK-9694) this means
changes to the *users > credentials* part of the administration console and to
the REST API.

For [KEYCLOAK-847](https://issues.jboss.org/browse/KEYCLOAK-847) this means changes
to the authentication logic, changes to the authentication part of the admin console,
changes to the client part of the admin console, extending Keycloak's comprehension
of the OIDC and SAML protocols, and some small changes to the REST API.

What this design proposal does _not_ cover is the specific handling of the WebAuthn
and modifications to the account console, as other design proposals and JIRAs already
specify those modifications.

## Design proposal for [KEYCLOAK-9693](https://issues.jboss.org/browse/KEYCLOAK-9693)

In this section we discuss modifications necessary to allow an admin to manage two factor
authentication, and the design of the admin GUI for this task. We will also be using the following concepts:
* **Authentication factor**: A new type of execution in the authentication flow, to which only authentication executions can be assigned. This is designed to allow N factors to be added to an authentication flow.
* **Authentication execution**: An execution that can only be set within an authentication factor. This is the actual code that is run to authenticate a user. Examples of authentication executions in Keycloak would be the classes OTPFormAuthenticator, UsernamePasswordForm or ValidateOTP.
* **Authentication category**: A logical grouping between authentication executions and credentials. This design is to allow for multi-tokens.
* **Credential**: The credentials that are used by the user to log in. Keycloak stores shared secret that is used to log in with, for example hash for passwords, symmetric key for OTP and public key for Fido2. In this document, **credential** will be used for both the representation in Keycloak and the user's possession/knowledge.

### Authentication logic

#### Authentication factors

The current authentication flow system should be kept, but simplified. Authenticators categories -
that is elements that provide identity (e.g. password, cookie, kerberos, otp, fido, ...) - should
be separate from other executions like "reset password".

Thus, instead of directly adding "cookie" or "password" as alternatives in the authentication flow, the administrator can add the execution "authentication factor", and under
authentication factor he can add only the valid authenticator executions. Each of the
authenticator executions under the authentication factor are considered valid alternatives,
and are evaluated in order. The authentication stops being automatically evaluated at the
first authentication execution that requires user input (alternative: all non-user input
authentication executions are evaluated before the first user-input authentication execution).

Adding a 2nd "authentication factor" execution sets the 2nd factor, and it must then have
authentication executions assigned. To have an authenticator execution be valid for both
authentication factors it must be set under both 1st and 2nd authentication factor. For
example, kerberos could be set for both 1st and 2nd factor, which would mean that the user
skips both factors if he's registered with kerberos, but it could also be just set for the
first factor, in which case he'd still have to input the 2nd factor. To handle optional 2nd
factors there could either be a "optional authentication factor type" or have an
"optional" flag in the options of the "authentication factor".

Potentially, this system could also allow a company that's really security
conscientious/paranoiac to set N factors.
By setting a single authentication factor with an authentication executions that are not
passwords (e.g. Fido2), this system also allows passwordless login.

Authentication factors are treated as a bloc in the evaluation of alternatives. That is, if in
a flow there's "Identity provider redirector", "authentication factor 1", "authentication
factor 2", entering the authentication factor 1 flow will automatically cause "authentication
factor 2" to be executed after.

To make things perhaps a little more clear, the current default "Browser" authentication
would become for example:

```
Auth type
------------------------------------------------------------
Identity Provider Redirector
Authentication factor (1)
  |- Cookie
  |- Kerberos
  |- Username Password Form
Authentication factor (2) [OPTIONAL]
  |- Cookie
  |- Kerberos
  |- OTP Form
```
And if we wanted to have an mandatory 2nd factor, with either OTP or a FIDO2 configured:

```
Auth type
------------------------------------------------------------
Identity Provider Redirector
Authentication factor (1)
  |- Cookie
  |- Kerberos
  |- Username Password Form
Authentication factor (2)
  |- Cookie
  |- Kerberos
  |- OTP Form
  |- FIDO-2
```
Potentially we could also introduce another type of authentication execution, we could call it "Authenticated on", to simplify the authenticators that bypass the authentication factors. We could then have:

```
Auth type
------------------------------------------------------------
Identity Provider Redirector
Authenticated on
  |- Cookie
  |- Kerberos
Authentication factor (1)
  |- Username Password Form
Authentication factor (2)
  |- OTP Form
  |- FIDO-2
```

#### Authentication categories

Authentication categories allow the link between the actual execution within the flow and the
credentials of a certain type.

For example, currently there is the OTPFormAuthenticator for the browser flow and ValidateOTP for the direct grant flow.
However, both these authentication executions will accept input from the same OTP credentials. This means that schematically, we would have:
```
-----------------------         ----------------------
| OTP Credential 1    |---------|                    |    ------------------------
-----------------------         |                    |----| OTPFormAuthenticator |
-----------------------         | OTP Authentication |    ------------------------
| OTP Credential 2    |---------|      Category      |    ---------------
-----------------------         |                    |----| ValidateOTP |
-----------------------         |                    |    ---------------
| OTP Credential 3    |---------|                    |
-----------------------         ----------------------
```

This schema serves as a base for all credentials and allows them to be
used for the appropriate authentication executions. This link allows Keycloak to present
alternative credentials for an execution during login.

To simplify selection in the GUI, it would also be useful to do some refactoring to ensure
that the execution code for the `Browser` / `Registration Flow` / `Direct Grant Flow` /
`Reset Credentials` / `Client Authentication` Bindings can only be selected as executions
for their binding categories. This already seems to be the case for `Client Authentication`,
and would simplify the selection of authentication executions within a authentication
factor.

It may also allow a simplification of the naming when selecting an authentication
category within an authentication factor. For example, we could just
have OTP instead of OTP Form (for `Browser` binding) and OTP (for `Direct Grant`)

In addition to its function, an authentication category will also have a set of
data/metadata that describes it. This can be a general field that will be found
in all authenticator categories, for example description information (internationalised),
an icon or something more functional like data for OIDC's amr claims (see section on
step-up) or whether the authentication category can have one or many authenticators
assigned to it. This information would not be held in the database, but rather
directly in Keycloak as static information in the code or as resources.

### Authentication section in the admin console

#### Flows

The schema described in the logic would be implemented
in the _Authentication > flows_ section. Some work can be done on the UX to make
the process more intuitive, but this will be proposed at a later state in the
design process.

##### Default Flows

Keycloak should have the same default flows as is currently the case, except that two
authentication factors are used, with preset values for the authenticator executions:
rather than adding and removing authenticator executions, the user can enable or disable
them.
This is only UX, because behind Keycloak sets or removes the **disabled** flag within the
authentication execution's config. The 2nd authentication factor can also be disabled, and
if either the 2nd factor is disabled, or if all authentication executions are disabled in
the second factor, then the login will be single factor only.

For example, for the browser flow this would be:
```
Auth type                         | Options
---------------------------------------------------------------------------
Identity Provider Redirector
Authenticated on               
  |- Cookie
  |- Kerberos                      [ ] enabled  [x] disabled
Authentication factor (1)
  |- Username Password Form
Authentication factor (2)          [x] enabled  [ ] disabled  [ ] optional
  |- OTP Form                      [x] enabled  [ ] disabled
  |- FIDO-2                        [x] enabled  [ ] disabled
```
And for the Direct Grant the flow would be for example:
```
Auth type                         | Options
---------------------------------------------------------------------------
Username Validation
Authentication factor (1)
  |- Password
Authentication factor (2)          [x] enabled  [ ] disabled  [ ] optional
  |- OTP                           [x] enabled  [ ] disabled
  |- Other                         [x] enabled  [ ] disabled
```

##### Configurable flows

Configurable flows work as is currently the case in Keycloak with the possibility
to add new authentication flows and either replace the default flow in _Bindings_
or in the _Client > [client] > Authentication Flow Overrides_. Here the normal configuration of
an authentication factor is to add (or remove) authenticator executions. However, within an
authentication execution's configuration, it is possible to set a **disable** flag.
This is for when an admin doesn't want to keep an execution active, but doesn't want it's
configuration to be deleted. An **optional** flag is present in the configuration of the
authentication factor, meaning that it will only be required for users which have at least
one credential set for an execution that is within that authentication factor.

For UX purposes, the GUI shows directly in the flow when an authentication factor is set
to optional, or if an authentication execution is disabled.

#### Bindings

No changes are necessary here.

#### Required Actions

Current required actions are a bit too limited to support multi-tokens. While it is currently
possible to require a user to, for example, register an OTP device, it must be possible for the
administrator to require "register a valid 1st factor" and "register a valid 2nd factor".

To do this, it must be possible to register a required action multiple times, and to configure
required actions. It must also be possible to remove required actions from the list. The code
of the required action should define if it can be registered multiple times, and how it can
be configured. However, all required actions should be able to be removed, and removing a
required action should also remove it from all users for which it is currently required.

The purpose of this is to be able to have a "register credential" required action in which
authentication categories can be set. For example, if an admin were to configure a 2nd factor
with OTP and Fido2, he could then set a "register credential" with the OTP and Fido2
authentication categories. A user, when confronted with that required action, would only have
to register one of the two.


#### Policies

The password policy for the realm should remain, but the OTP policy should disappear, as a user
could have several alternative OTP devices with different settings. As such, the information
about the OTP settings should probably move to information associated with the credential.

### Authentication screens for the user

When the user is redirected towards Keycloak, unless he's using some system that
bypasses the login actions (e.g. session cookie, kerberos,...) he will arrive at a
login prompt that corresponds to his authentication category.

The rule for selecting which credential is chosen for a user is
1. Authentication category that is configured on the client or asked for by the client (see section on step up)
  1. Credential of the asked for category that the user has selected as preferred (see
    section on the handling of the users' credentials)
  1. Credential of the asked for category that is the first in the list of the user
1. Credential that is among the allowed categories of the authentication factor and that is selected as preferred by the user.
1. Credential that is first in the list of the user that is of the first authentication category in the authentication flow

The pseudo code for this selection would be:
```
if (Client asks for a set of categories OR a set of categories is set for the client) {
  if (user has a preferred credential in an allowed category) {
    Ask for a login with that preferred credential
  } else {
    Ask for a login with the first credential in the user's list that corresponds to the first allowed category
  }
} else if (user has a preferred credential in an allowed category) {
  Ask for a login with that preferred credential
} else {
  Ask for a login with the first credential in the user's list that corresponds to the first allowed category
}
```

During an authentication factor, the user can select any authenticator he has registered for
one of the authorised authentication categories. For the browser flow, this means that the
displayed login page may change when switching between authentication categories, for
example when changing from OTP to FIDO-2. The UI must make it clear what is happening to the
user. While the UI could show authentication categories for the user's credentials, it
shouldn't list any authentication categories which are empty for the user (that the user
hasn't registered a credential for).

### REST API

There's no major change here, aside from updating the scheme to support the separation
between authentication executions and other executions, and adding instructions to edit which
authentication categories are assigned to an authentication factor.

### Database modifications

Authentication executions must be easily separable from other executions in order
for them to be linked to actual authenticator categories. One necessary modification is
to have the authentication category as information in the database for an authentication
execution, as the field may be selected upon.

Authentication categories themselves should not be in the database however, as they
are fixed entities set at development time. The only requirement is that both
authentication executions and credentials must be findable by their authentication categories.

## Design proposal for [KEYCLOAK-9694](https://issues.jboss.org/browse/KEYCLOAK-9694)

### Changes to the users > Credential menu

Instead of manage Password we have "manage credentials", with a list of credentials
for a user. The credentials are grouped by authentication category, and could indicate which
authentication flows and factors they are currently valid for. The administrator should
be able to edit / remove a credential, and in some cases maybe even create a credential.

#### Editing

The administrator should be able to visualise the data of the credential (except any
secret data) and edit metadata about a device. Data would be any data in the fields of
the credential that describe immutable data about the credential (secret or not). The
administrator (and user) would be able to set a label for the device to identify it.
They would also be able to set a "preferred order" for the credentials. The credential used
by default during login will be the "most preferred" credential belonging to a valid
authentication category, at each authentication step.

A note for passwords: Unlike other credentials there should not be more than one password
that can be configured. Also, among its edition option it should be possible to reset
(temporarily or not) as is currently the case.

#### Deleting

Deleting credentials can lead to a user being unable to authenticate himself.
While a user shouldn't be able to put himself in this situation, an admin must be
able to remove any or all credentials for a user if they have been compromised.
Considering the problems that may result from incorrectly removing a credential,
the admin GUI should ask for a validation (e.g. "Are you sure?") before doing
the delete.

If the administrator and user can remove credentials, the possibility to disable
credentials may no longer be useful. However, there would also be no problem in keeping it.

#### Creating

For some authentication categories, an admin may be able to create a credential.
The most obvious of these is the password, as is currently the case. The admin may
create a new password for a user (temporary or not). If one already exists, it
overwrites the previous one.

For other authentication categories, it is up to them to determine if a credential
can be created, and to determine how it can be created.

### Modifications to the REST API

Currently there's no way to get credentials with the REST API. This should change with these
modifications to reflect the new options for the administrator. The API should function in the
same manner as the admin console: Credentials can be exposed (except for secret
values), the metadata edited and the credentials deleted with the same restrictions as described in the previous section.

### Database modifications

The current system has multiple fields to store information about a crendiential. Some
are common, others are not. For example, a password's secret is the hash of the password,
an otp's secret is the value shared between Keycloak and the device. However, some information
is only relevant for a credential of a specific authentication category, for example,
password credentials contain the salt and the hash iterations, while TOTP may have the
number of digits and the period. Other authenticators will likely have their own set of
fixed metadata. This data doesn't need to be queried nor modified, therefore the new
credential table columns could be:
```
-----------------------------
| ID                        |
-----------------------------
| authenticator_category    |
-----------------------------
| user_label                |
-----------------------------
| secret_data               |
-----------------------------
| credential_data           |
-----------------------------
```

With `ID` the primary key of the credential, `authenticator_category` a string
set during the creation that must reference an existing authenticator category,
`user_label` being the editable name of the credential by the user (a varchar(N)),
`secret_data` containing a static json with the information that cannot be transmitted
outside of Keycloak (and potentially crypted for security reasons) and the `credential_data`
containing a json with the static information of the credential that can be shared in the
admin console or via the REST API. It is up to the code linked with an authentication
execution to be able to interpret the `secret_data` and `credential_data` correctly.

## Design proposal for [KEYCLOAK-847](https://issues.jboss.org/browse/KEYCLOAK-847)

In this section we discuss mechanisms of step up authentication, the logic of the
implementation within Keycloak, how the administrator can configure the step up
within the admin console (what this implies for flows and clients), and the
implications for users.

### Step up mechanisms

Step-up is the ability to allow access to clients or resources based on an authentication
level of a user. It can be described by two example use cases:
1. A user has a session in Keycloak for a *client A* after having logged in with username and password. However, when connecting to a *client B* which demands greater security, the user is asked for a second authentication factor before proceeding. Had the user connected initially to *client B*, he would have had to provide username/password **and** the second factor.
1. A user has a session in Keycloak with a *client C*, that was logged in with username and password. However, when doing an operation with more risk *client C* asks Keycloak to provide a second factor authentication for the user. Note that the client may also ask for a full 2 factor authentication: username/password + 2nd factor. A good example for this case would be a bank that allows a user to browse his transaction history with only username/password, but which requires a 2nd factor to access the payment functionality.

In many ways, step-up can be viewed as an authorization which is enabled by stronger
authentication to be granted. Use case 1 can be handled only through internal configurations
within Keycloak but the use case 2 requires support from both Keycloak and the client.
Conceptually, the mechanisms to support both use cases must be present within Keycloak.

Use case 2 is supported by both the SAML and OIDC protocols. Within these protocols
there are three concepts that are represented that enable step-up:
* Level of Authentication (LoA)
* Specifying authentication categories
* Forcing authentication

#### Level of Authentication

This defines an agreed upon level of authentication (also called level of assurance)
between an IDP and client. There is no single formal document that specifies the
requirements for an authentication.
For example [NIST's SP 800-63-3](https://pages.nist.gov/800-63-3/) has three different
categories of three levels each, while
[ISO/IEC 29115](https://www.iso.org/standard/45138.html) defines 4 assurance levels.
The basic idea is the manner in which an authentication is done or which authenticators
are used causes lower or greater assurances about the identity of the user. This level
can then be used to determine if a user can or not perform a requested action.

OIDC supports LoA with the optional Authentication Context Class `acr` claim,
or with the acr_values parameter as defined in the
[core specification](https://openid.net/specs/openid-connect-core-1_0.html#acrSemantics).

SAML supports LoA with the [Identity Assurance Profiles](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-assurance-profile.html) part of the protocol, and with the
`<RequestedAuthnContext>` element.

#### Specifying authenticator categories

A client may also specify exactly which category of authenticator it wishes to use.

For SAML this is done with the [Authentication Context for the OASIS Security Assertion Markup Language](http://docs.oasis-open.org/security/saml/v2.0/saml-authn-context-2.0-os.pdf)
part of the protocol, and with the `<RequestedAuthnContext>` element.

For OIDC there is no way to do this. The `amr` parameter can describe which authenticators
are used with the Authentication Method Reference `amr` Claim, which is defined within the
[core specification](https://openid.net/specs/openid-connect-core-1_0.html) with some
extra information in [rfc8176](https://tools.ietf.org/html/rfc8176). However, there
is no described way to use this to ask for specific authentication categories.

#### Forcing authentication

To fully support the step-up, a client should be able to force re-authentication even
when Keycloak considers that a user is authenticated to an acceptable level.

For OIDC this is done with the `prompt=login` optional parameter.

For SAML this is done with the `ForceAuthn="true"` attribute of the `AuthnRequest`

### Logic within Keycloak

Implementing the step-up authentication in Keycloak means implementing both the LoA
and the request of authenticator categories. These capabilities are build upon the new
authentication mechanisms described previously in the document.

#### Level of Authentication

The LoA is specified as markers within the authentication flow, specifically,
this is set as a configuration option within an authentication factor. For Keycloak
this is internally set as a positive numerical value. Any correspondence to an
external definition of an LoA is handled through configuration.

Passing an authentication factor within an authentication flow sets that level for a user.
User sessions record both the current highest authentication level used, as well as the
authentication categories used (possibly which credentials have been used
instead of the authentication categories).

By default, clients do not require a LoA, which means that it will have to do the full
authentication flow. For this, any negative value is used, meaning "No LoA requirement":
the full flow is used if a user is not currently connected to Keycloak, and normal single
sign on behaviour if the user is already logged in. However a required LoA can be set within
the client, meaning that during an authentication flow, a client may only require a single
factor, even if the flow specifies multiple factors without the optional parameter.
In this sense, setting an LoA to an authentication factor automatically marks it as
optional under certain conditions.

When a client is logged on with a sufficient factor, it can be directly accessed. When a
client is not logged with a sufficient factor, the extra authentication factor (or factors)
are asked for. A client may at any time use the LoA protocol mechanisms to ask for an
authentication level. If it requires the full re-authentication for the level,
no matter the current state of the user's session, it must use the force parameter.
When a client is using a protocol to ask for LoA (forced or not), Keycloak must respect the
protocol with respect to how it acts and how it responds.

A conflict situation may arise conceptually when an admin specifies several
flows, sets different LoA levels to each, and assigns the flows to different clients.
However, the rule is always the same: if a user has a certain level, it needs only have that
level to connect to a client. It's up to the admin to make sure that the LoA is
coherent.

A note for LoA configuration: for a given LoA for a specification such as [ISO/IEC 29115](https://www.iso.org/standard/45138.html), many of the factors for different
LoA levels are not dependent on MFA, but on organisational or other technical factors
(e.g. password strength, nature of an OTP device, ...), so it is up to the administrator
to evaluate the actual LoA that is transmitted to a realm's clients.

#### Specifying authenticator categories

This only makes sense for SAML clients, which can use protocol mechanisms to request
specific authenticator categories, and in this case, Keycloak must act and respond as
defined by the protocol. If the user is already authenticated with the requested
authenticator categories, Keycloak can simply return the requested token, but if not or
if the request uses the force parameter it must perform the extra authentication(s).

This can also be handled on a configuration level for clients, but it is simply
a matter of creating a specific authentication flow, with the desired number of
authentication factors, and the required authenticator categories, and then setting
the authentication flow to a client.

For both SAML and OIDC, however, this will require being able to define respectively
[authentication context classes](https://docs.oasis-open.org/security/saml/v2.0/saml-authn-context-2.0-os.pdf) and [authentication method reference values](https://tools.ietf.org/html/rfc8176) to an authentication category. This is necessary to be able to transmit a
value within an `<AuthContext>` (for SAML) or an `amr` claim (for OIDC). It is also necessary
for Keycloak to be able to present during an authentication with a `<RequestedAuthnContext>`
the user with the correct choice of authenticators.

### Modifications to the admin console

#### Authentication > flows section

Authentication factors must be able to be configured with an LoA, which can be
any positive value. Subsequent authentication factors must have their values set
to equal or increasing values. A string value may also be set as an alias for the
LoA, to be used for example when the LoA is described as an URI. If the alias is
not set, the numeral value is sent to the client in the assertion, and if it
is set, the alias is used. For ease of information, the interface should show
that the authentication factor is being used as a LoA, and at which level.

#### Client

In the client configuration it must be possible to set a LoA before being able to
have access to the client. Since the LoA is attached to the flow, it should be within
the same GUI section as the choice of authentication flow, and for coherence, it should
be only among the choices of the flow. However, behind the scenes this only sets
the level for the client. If the configuration of the authentication flow is changed
and the value of the LoA modified, this does not change the value within the client. Within the
configuration of the client, it may show that the current LoA is not relevant to the
configured authentication flow as a visual aid.

Note that for OIDC clients, this means that a client can have a different LoA
requirements for browser and direct grant flows.

### Modifications to the REST API

There are only minimal modifications to the REST API, and they mirror the changes
to the GUI:
* Possibility to set the LoA and alias when setting or modifying an authentication factor
* Possibility to set the LoA for the browser or direct grant flow when creating or editing a client

### Extending protocol implementation within Keycloak

The following protocols or optional parts of protocols must be implemented for OIDC:
* Handling of the `acr` claim, `acr_values` parameter and `prompt` parameter as
described in the [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
* Implementation of the [RFC 8176: Authentication Method Reference Values](https://tools.ietf.org/html/rfc8176)

And for SAML:
* [Authentication Context for the OASIS Security Assertion Markup Language (SAML) V2.0](https://docs.oasis-open.org/security/saml/v2.0/saml-authn-context-2.0-os.pdf), although only context classes that actually match an authenticator need be implemented
* [SAML V2.0 Identity Assurance Profiles](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-assurance-profile.html)
* Handling of the `<RequestedAuthnContext>` element and of the `ForceAuthn` parameter
as described in the [SAML core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf).
